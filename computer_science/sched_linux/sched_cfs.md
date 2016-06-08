# 完全公平调度(Completely Fair Scheduler)
***

#### CFS调度实体 sched_entity
* include/linux/sched.h::sched_entity
```c
struct sched_entity {
    struct load_weight  load;       /* for load-balancing */
    struct rb_node      run_node;
    struct list_head    group_node;
    unsigned int        on_rq;

    u64         exec_start;
    u64         sum_exec_runtime;
    u64         vruntime;
    u64         prev_sum_exec_runtime;
...
};
```

#### 创建新进程时的place_entity()

* `task_fork_fair()`函数是这么调用的
  * kernel/sched/fair.c
```c
static void task_fork_fair(struct task_struct *p)
{
...
  cfs_rq = task_cfs_rq(current);
  curr = cfs_rq->curr;
...
  if (curr)
      se->vruntime = curr->vruntime;
  place_entity(cfs_rq, se, 1);
...
}
```
如果当前进程所在队列不为空，将所在队列当前进程的vruntime作为新进程vruntime的基础。

* `place_entity()`函数我们在此只关心`initial = 1`的部分。
  * kernel/sched/fair.c
```c
static void
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
    u64 vruntime = cfs_rq->min_vruntime;

    /*   
     * The 'current' period is already promised to the current tasks,
     * however the extra weight of the new task will slow them down a
     * little, place the new task so that it fits in the slot that
     * stays open at the end.
     */
    if (initial && sched_feat(START_DEBIT))
        vruntime += sched_vslice(cfs_rq, se);
...
    /* ensure we never gain time by being placed backwards. */
    se->vruntime = max_vruntime(se->vruntime, vruntime);
}
```
* 新进程的初始vruntime值就以它所在运行队列的min_vruntime为基础来设置，与老进程保持在合理的差距范围内。

* sched_features是控制调度器特性的开关，每个bit表示调度器的一个特性。在`kernel/sched/features.h`文件中记录了全部的特性。见《Professional Linux Kernel Architecture》
> Note that the real kernel sources will execute portions of the code depending on outcomes of sched_feature queries. The CF
scheduler supports some ‘‘configurable’’ features, but they can only be turned on and off in debugging mode — otherwise, the set
of features is fixed.

* `START_DEBIT`是其中之一，如果打开这个特性，表示给新进程的vruntime初始值要设置得比默认值更大一些，这样会推迟它的运行时间，以防进程通过不停的fork来获得cpu。

  * kernel/sched/features.h
```c
/*
 * Place new tasks ahead so that they do not starve already running
 * tasks
 */
SCHED_FEAT(START_DEBIT, true)
```
* `sched_vslice()`用于计算将要插入的进程的虚拟时间片，仅在`place_entity()`被调用。
  * kernel/sched/fair.c::sched_vslice()
```c
/*
 * We calculate the vruntime slice of a to-be-inserted task.
 *
 * vs = s/w
 */
static u64 sched_vslice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    return calc_delta_fair(sched_slice(cfs_rq, se), se);
}
```
* 如果`START_DEBIT`位被设置，则通过加上一个调度周期内的虚拟时间片将新进程推后执行。
* 否则新进程与父进程的vruntime一致，如果`sysctl_sched_child_runs_first`参数没有设成**1**，子进程与父进程谁先被执行还不确定。

### 避而不谈的\__calc_delta()

很多关于kernel的书籍和讲CFS的文章在讲到\__calc_delta()函数的时候都选择了略过，可能是由于这一部分变动的比较大，但这里有多有趣的细节值得推敲，这里根据v4.5的代码讲一下。

* kernel/sched/fair.c
```c
#define WMULT_CONST (~0U)
#define WMULT_SHIFT 32

static void __update_inv_weight(struct load_weight *lw)
{
    unsigned long w;

    if (likely(lw->inv_weight))  /* 如果当前的inv_weight有预设值，则直接采用预设值 */
        return;

    w = scale_load_down(lw->weight); /* 否则以当前权重为基础重新计算 */

    if (BITS_PER_LONG > 32 && unlikely(w >= WMULT_CONST))
        lw->inv_weight = 1;           /* 当64位系统且权重超过2^32，inv_weight一律看作1 */
    else if (unlikely(!w))            /* 当权重值为0 */
        lw->inv_weight = WMULT_CONST; /* inv_weight值为2^32-1 */
    else /* inv_weight为(2^32-1)/权重，当32位系统，或64位系统但权重未超过2^32，且权重不为0 */
        lw->inv_weight = WMULT_CONST / w;
}

/*
 * delta_exec * weight / lw.weight
 *   OR
 * (delta_exec * (weight * lw->inv_weight)) >> WMULT_SHIFT
 *
 * Either weight := NICE_0_LOAD and lw \e prio_to_wmult[], in which case
 * we're guaranteed shift stays positive because inv_weight is guaranteed to
 * fit 32 bits, and NICE_0_LOAD gives another 10 bits; therefore shift >= 22.
 *
 * Or, weight =< lw.weight (because lw.weight is the runqueue weight), thus
 * weight/lw.weight <= 1, and therefore our shift will also be positive.
 */
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
    u64 fact = scale_load_down(weight); /* 当前可以认为没有对权重weight进行比例伸缩 */
    int shift = WMULT_SHIFT;  /* WMULT_SHIFT为32 */

    __update_inv_weight(lw);  /* 给lw的inv_weight一个合理的值，且在 1 ~ 2^32-1 的范围内 */

    /* 当权重>=2^32时，尝试不断缩小权重(每次折半)，并用shift记录，直至权重小于2^32，
       即经过此步骤 0 < fact < 2^32-1 */
    if (unlikely(fact >> 32)) {
        while (fact >> 32) {
            fact >>= 1;
            shift--;
        }
    }

    /* hint to use a 32x32->64 mul */
    /* 之前的步骤已经确保fact和lw->inv_weight的值都不会超过2^32-1，
       因此它们的乘积也不会超过2^64。
       1) 当__calc_delta()用于计算当前进程消耗的vruntime时，
          即 weight := NICE_0_LOAD, lw := &se->load 时,
          其实就是在计算：
          fact = （2^32-1） * NICE_0_LOAD / se->load.weight
          这里先乘个2^32-1能保证fact的结果不会因为NICE_0_LOAD < se->load.weight
          而导致的整形除法商为0的情况。
      2) 当__calc_delta()用于计算按权重比例分得的调度周期内墙上时钟时间片时，
         即被sched_slice()调用，weight是调度实体的权重，而lw是cfs_rq队列上权重和。
         这里计算的就是当前调度实体的权重占所在队列的权重总和的比例，并放大2^32-1倍。
     */
    fact = (u64)(u32)fact * lw->inv_weight;

    /* 再次缩小上面的乘操作可能导致的fact > 2^32-1 */
    while (fact >> 32) {
        fact >>= 1;
        shift--;
    }

    /* vruntime = (delta_exec * 扩大后的权重比) >> shift
       之前__update_inv_weight(lw)放大的倍数会在这里通过右移shift位缩小回来。
     */
    return mul_u64_u32_shr(delta_exec, fact, shift);
}
```

* 这里可以考虑计算`vruntime`时（即`weight := NICE_0_LOAD`）的两种极限情况：
  1. 当`lw->weight`特别小，比如为1时，`__update_inv_weight(lw)`会把`lw->inv_weight`置为`WMULT_CONST`（即`0x ff ff ff ff`）, `NICE_0_LOAD`等于1024相当于将`0x ff ff ff ff`再右移10位。因此，第二个while循环后`shift`的值会变为22。`mul_u64_u32_shr(delta_exec, fact, shift)`相当于`delta_exec * (2^32-1) / 2^22`变换成`delta_exec * (2^10 - 1/2^22)`，算下来应该与`delta_exec * 1024`的值差不多。
  2. 当`lw->weight`特别大，甚至超过`WMULT_CONST`（即`0x ff ff ff ff`）时，`__update_inv_weight(lw)`会把`lw->inv_weight`置为**1**，两次while循环都不会产生`fact`右移的情况。`fact`仍然是`NICE_0_LOAD`，而`shift`还是32，故`mul_u64_u32_shr(delta_exec, fact, shift)`相当与`delta_exec >> 22`。考虑`2^22 = 4,194,304`，最小调度粒度设为4ms（即4,000,000ns）也不是没有道理的。

* 在\__calc_delta()用于计算时间片的时候（即被sched_slice()），`lw->weight`是就绪队列权重的总和，虽然`lw->weight`有可能会非常的大，但根据`sched_slice()`的实现，`delta_exec`是调度周期，当就绪队列里任务非常多时，调度周期也会相应地增大。


* include/asm-generic/bitsperlong.h
```c
#ifdef CONFIG_64BIT
#define BITS_PER_LONG 64
#else
#define BITS_PER_LONG 32
#endif /* CONFIG_64BIT */
```

* include/linux/math64.h
```c
#if defined(CONFIG_ARCH_SUPPORTS_INT128) && defined(__SIZEOF_INT128__)

#ifndef mul_u64_u32_shr
static inline u64 mul_u64_u32_shr(u64 a, u32 mul, unsigned int shift)
{
    return (u64)(((unsigned __int128)a * mul) >> shift);
}
#endif /* mul_u64_u32_shr */

#else

#ifndef mul_u64_u32_shr
static inline u64 mul_u64_u32_shr(u64 a, u32 mul, unsigned int shift)
{
    u32 ah, al;
    u64 ret;       /* u64在32位和64位平台上长度都是 8Byte */

    al = a;
    ah = a >> 32;  /* a的高32位右移用来计算，防止溢出u64 */

    ret = ((u64)al * mul) >> shift; /* cast成u64防止两个u32相乘上溢造成的截断 */
    /* 如果a的高32位不为空，则需拼接高32位的运算结果。由于低32位的结果右移了shift，
       所以在拼接时高32位的运算结果需要左移的位数是32与shift的差值。 */
    if (ah)
        ret += ((u64)ah * mul) << (32 - shift);

    return ret;
}
#endif /* mul_u64_u32_shr */

#endif
```

* [128-bit Integers (__int128)](https://gcc.gnu.org/onlinedocs/gcc-6.1.0/gcc/_005f_005fint128.html#g_t_005f_005fint128)

> As an extension the integer scalar type __int128 is supported for targets which have an integer mode wide enough to hold 128 bits. Simply write __int128 for a signed 128-bit integer, or unsigned __int128 for an unsigned 128-bit integer. There is no support in GCC for expressing an integer constant of type __int128 for targets with long long integer less than 128 bits wide.

* init/Kconfig
```sh
config ARCH
    string
    option env="ARCH"
...
#
# For architectures that know their GCC __int128 support is sound
#
config ARCH_SUPPORTS_INT128
    bool
...
```

* arch/x86/Kconfig
```sh
# Select 32 or 64 bit
config 64BIT
    bool "64-bit kernel" if ARCH = "x86"
    default ARCH != "i386"
    ---help---
      Say yes to build a 64-bit kernel - formerly known as x86_64
      Say no to build a 32-bit kernel - formerly known as i386

config X86_32
    def_bool y
    depends on !64BIT

config X86_64
    def_bool y
    depends on 64BIT

### Arch settings
config X86
    def_bool y
    ...
    select ARCH_SUPPORTS_INT128     if X86_64
    ...
```

* include/asm-generic/int-ll64.h
```c
#ifndef __ASSEMBLY__

typedef signed char s8;
typedef unsigned char u8;

typedef signed short s16;
typedef unsigned short u16;

typedef signed int s32;
typedef unsigned int u32;

typedef signed long long s64;
typedef unsigned long long u64;
...
#else /* __ASSEMBLY__ */
...
#endif /* __ASSEMBLY__ */
```
#### 结论
* `__calc_delta()`函数需要保证无论是32位平台还是64位平台，都能返回一个u64类型的值。且需要解决好潜在的上溢问题。
* 在用于计算`vruntime`时可能会由于调度实体的权重比`NICE_0_LOAD`大导致它们的商为0，因为它们用的都是整数类型，故`__calc_delta()`需要解决这个问题。

### 调度器的一些features
* kernel/sched/features.h
