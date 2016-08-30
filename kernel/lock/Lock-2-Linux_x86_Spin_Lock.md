# Linux x86自旋锁的实现

## 原理
* `struct raw_spinlock`的`raw_lock`成员分为`head`和`tail`两部分
  * 如果处理器个数不超过 256，则`head`使用低位（0-7 位），`tail`使用高位（ 8-15 位）；
  * 如果处理器个数超过 256，则`head`使用低 16 位，`tail`使用高 16 位。
  * 可见排队自旋锁最多支持 2^16=65536 个处理器。
* 排队自旋锁初始化时`raw_lock`被置为 0，即`head`和`tail`置为 0。
* 内核执行线程申请自旋锁时，原子地将`tail`域加 1，表示下一个持锁者的票据号将会是多少（*Next*），并将`tail`原值返回作为自己的票据序号（*Ticket*）。
* 同时，原子地取出`head`域，表示当前持锁者的票据号应是多少（*Owner*）。
* `head`与`tail`相等时，表明锁处于未使用状态（此时也无人申请该锁）。
  * 如果返回的票据序号 *Ticket* 等于申请时的 *Owner* 值，说明自旋锁处于未使用状态，则直接获得锁；
  * 否则，该线程忙等待检查 *Owner* 是否等于自己持有的票据序号 *Ticket*，一旦相等，则表明锁轮到自己获取。
* 线程释放锁时，原子地将 *Owner* 加 1 即可，下一个持有该 *Ticket* 的线程将会发现这一变化，从忙等待状态中退出，继续执行后续代码。
* 线程将严格地按照申请顺序依次获取排队自旋锁，从而完全解决了“不公平”问题。

## 实现

### 数据结构
* include/linux/spinlock_types.h
```c
typedef struct raw_spinlock {
    arch_spinlock_t raw_lock;
#ifdef CONFIG_GENERIC_LOCKBREAK
    unsigned int break_lock;
#endif
#ifdef CONFIG_DEBUG_SPINLOCK
    unsigned int magic, owner_cpu;
    void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
} raw_spinlock_t;
...
typedef struct spinlock {
    union {
        struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
        struct {
            u8 __padding[LOCK_PADSIZE];
            struct lockdep_map dep_map;
        };  
#endif
    };  
} spinlock_t;
```

* arch/x86/include/asm/spinlock_types.h
```c
#ifdef CONFIG_PARAVIRT_SPINLOCKS
#define __TICKET_LOCK_INC   2
#define TICKET_SLOWPATH_FLAG    ((__ticket_t)1)
#else
#define __TICKET_LOCK_INC   1
#define TICKET_SLOWPATH_FLAG    ((__ticket_t)0)
#endif

/*如果处理器个数不超过 256，则 head 使用低位（0-7 位），tail 使用高位（ 8-15 位）；
  如果处理器个数超过 256，则 head 使用低 16 位，tail使用高 16 位。
  可见排队自旋锁最多支持 2^16=65536 个处理器。*/
#if (CONFIG_NR_CPUS < (256 / __TICKET_LOCK_INC))
typedef u8  __ticket_t;
typedef u16 __ticketpair_t;
#else
typedef u16 __ticket_t;
typedef u32 __ticketpair_t;
#endif

#define TICKET_LOCK_INC ((__ticket_t)__TICKET_LOCK_INC)

#define TICKET_SHIFT    (sizeof(__ticket_t) * 8)
...
#ifdef CONFIG_QUEUED_SPINLOCKS
#include <asm-generic/qspinlock_types.h>
#else
typedef struct arch_spinlock {
    union {
        __ticketpair_t head_tail;
        struct __raw_tickets {
            __ticket_t head, tail;
        } tickets;
    };
} arch_spinlock_t;
#define __ARCH_SPIN_LOCK_UNLOCKED   { { 0 } }
#endif /* CONFIG_QUEUED_SPINLOCKS */
```
### 封装方法
* include/linux/spinlock.h
```c
#define raw_spin_trylock(lock)  __cond_lock(lock, _raw_spin_trylock(lock))

#define raw_spin_lock(lock) _raw_spin_lock(lock)
...
#define raw_spin_lock_bh(lock)      _raw_spin_lock_bh(lock)
#define raw_spin_unlock(lock)       _raw_spin_unlock(lock)
...

static __always_inline void spin_lock(spinlock_t *lock)
{
    raw_spin_lock(&lock->rlock);
}

static __always_inline void spin_lock_bh(spinlock_t *lock)
{
    raw_spin_lock_bh(&lock->rlock);
}

static __always_inline int spin_trylock(spinlock_t *lock)
{
    return raw_spin_trylock(&lock->rlock);
}
...
static __always_inline void spin_unlock(spinlock_t *lock)
{
    raw_spin_unlock(&lock->rlock);
}

```
* 根据`CONFIG_INLINE_SPIN_LOCK`选项是否配置，`_raw_spin_lock()`的实现稍有不同，但都会调用`__raw_spin_lock()`。
* include/linux/spinlock_api_smp.h
```c
#ifdef CONFIG_INLINE_SPIN_LOCK
#define _raw_spin_lock(lock) __raw_spin_lock(lock)
#endif

#ifdef CONFIG_INLINE_SPIN_LOCK_BH
#define _raw_spin_lock_bh(lock) __raw_spin_lock_bh(lock)
#endif
...
#ifdef CONFIG_INLINE_SPIN_TRYLOCK
#define _raw_spin_trylock(lock) __raw_spin_trylock(lock)
#endif
...
#ifndef CONFIG_UNINLINE_SPIN_UNLOCK
#define _raw_spin_unlock(lock) __raw_spin_unlock(lock)
#endif
...

static inline void __raw_spin_lock_bh(raw_spinlock_t *lock)
{   /*先禁的下半部，同时，禁下半部也意味着禁止抢占*/
    __local_bh_disable_ip(_RET_IP_, SOFTIRQ_LOCK_OFFSET);
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    /*再去竞争锁*/
    LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}

static inline int __raw_spin_trylock(raw_spinlock_t *lock)
{   /*注意：自旋锁会禁止抢占*/
    preempt_disable();
    if (do_raw_spin_trylock(lock)) {
        spin_acquire(&lock->dep_map, 0, 1, _RET_IP_);
        return 1;
    }   
    preempt_enable();
    return 0;
}
...
static inline void __raw_spin_lock(raw_spinlock_t *lock)
{   /*注意：自旋锁会禁止抢占*/
    preempt_disable();
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}

static inline void __raw_spin_unlock(raw_spinlock_t *lock)
{
    spin_release(&lock->dep_map, 1, _RET_IP_);
    do_raw_spin_unlock(lock);
    preempt_enable();
}

```
* kernel/locking/spinlock.c
```c
#ifndef CONFIG_INLINE_SPIN_TRYLOCK
int __lockfunc _raw_spin_trylock(raw_spinlock_t *lock)
{
    return __raw_spin_trylock(lock);
}
EXPORT_SYMBOL(_raw_spin_trylock);
#endif
...
#ifndef CONFIG_INLINE_SPIN_LOCK
void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
{
    __raw_spin_lock(lock);
}
EXPORT_SYMBOL(_raw_spin_lock);
...
#ifndef CONFIG_INLINE_SPIN_LOCK_BH
void __lockfunc _raw_spin_lock_bh(raw_spinlock_t *lock)
{
    __raw_spin_lock_bh(lock);
}   
EXPORT_SYMBOL(_raw_spin_lock_bh);
#endif

#ifdef CONFIG_UNINLINE_SPIN_UNLOCK
void __lockfunc _raw_spin_unlock(raw_spinlock_t *lock)
{
    __raw_spin_unlock(lock);
}
EXPORT_SYMBOL(_raw_spin_unlock);
#endif
...
#endif
```

* 我们看`CONFIG_LOCK_STAT`选项没开启时`LOCK_CONTENDED`的实现
  * include/linux/lockdep.h
```c
#ifdef CONFIG_LOCK_STAT
...
#else /* CONFIG_LOCK_STAT */

#define lock_contended(lockdep_map, ip) do {} while (0)
#define lock_acquired(lockdep_map, ip) do {} while (0)

#define LOCK_CONTENDED(_lock, try, lock) \
    lock(_lock)

#endif /* CONFIG_LOCK_STAT */

```

* include/linux/spinlock.h
```c
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)
{   
    __acquire(lock);
    arch_spin_lock(&lock->raw_lock);
}
...
static inline int do_raw_spin_trylock(raw_spinlock_t *lock)
{
    return arch_spin_trylock(&(lock)->raw_lock);
}

static inline void do_raw_spin_unlock(raw_spinlock_t *lock) __releases(lock)
{
    arch_spin_unlock(&lock->raw_lock);
    __release(lock);
}
...
```

### 实现细节

* arch/x86/include/asm/spinlock.h

```c
static inline int  __tickets_equal(__ticket_t one, __ticket_t two)
{
    /*我们观察TICKET_SLOWPATH_FLAG为 0 的情况；
      如果one和two相等，异或结果为 0，取非后函数返回true；
      如果one和two不等，异或结果不为0，与TICKET_SLOWPATH_FLAG取反的结果进行与操作也不为0，取非后返回false。*/
    return !((one ^ two) & ~TICKET_SLOWPATH_FLAG);
}

static inline void __ticket_check_and_clear_slowpath(arch_spinlock_t *lock,
                            __ticket_t head)
{
    if (head & TICKET_SLOWPATH_FLAG) {
        arch_spinlock_t old, new;

        old.tickets.head = head;
        new.tickets.head = head & ~TICKET_SLOWPATH_FLAG;
        old.tickets.tail = new.tickets.head + TICKET_LOCK_INC;
        new.tickets.tail = old.tickets.tail;

        /* try to clear slowpath flag when there are no contenders */
        cmpxchg(&lock->head_tail, old.head_tail, new.head_tail);
    }
}

...

/*
 * Ticket locks are conceptually two parts, one indicating the current head of
 * the queue, and the other indicating the current tail. The lock is acquired
 * by atomically noting the tail and incrementing it by one (thus adding
 * ourself to the queue and noting our position), then waiting until the head
 * becomes equal to the the initial value of the tail.
 *
 * We use an xadd covering *both* parts of the lock, to increment the tail and
 * also load the position of the head, which takes care of memory ordering
 * issues and should be optimal for the uncontended case. Note the tail must be
 * in the high part, because a wide xadd increment of the low part would carry
 * up and contaminate the high part.
 */
static __always_inline void arch_spin_lock(arch_spinlock_t *lock)
{
    /*TICKET_LOCK_INC为 1，加锁时tail+1，表示下一个获得锁的线程应持有的Ticket*/
    register struct __raw_tickets inc = { .tail = TICKET_LOCK_INC };

    /*xadd将相加后的值存入指定内存地址，并返回指定地址里原来的值。
      实现时会在之前插入lock 指令，表示lock的下一条指令执行期间锁内存，于是下一条指令就是原子操作。
      这样，下面一条语句的效果就是：
      1. 锁的tail域加 1，即更新了 Next（锁的tail域）
      2. 同时读取 Owner（锁的 head域）
      3. 本地变量inc 的 tail 域存的是 Ticket，head 域存的是 Owner
     */
    inc = xadd(&lock->tickets, inc);
    /*如果 Owner 与 Ticket 相等表示获得锁，退出*/
    if (likely(inc.head == inc.tail))
        goto out;

    /*否则，进入自旋过程*/
    for (;;) {
        unsigned count = SPIN_THRESHOLD;

        do {
            /*不停读取 Owner，即锁对象的 head 域，更新到inc.head*/
            inc.head = READ_ONCE(lock->tickets.head);
            /*Owner 等于 inc.tail存的 Ticket，获得锁，退出*/
            if (__tickets_equal(inc.head, inc.tail))
                goto clear_slowpath;
            /*让CPU歇一会，后面会详细讲怎么歇*/
            cpu_relax();
        } while (--count);
        /*未配置CONFIG_PARAVIRT_SPINLOCKS选项，下面函数为空函数*/
        __ticket_lock_spinning(lock, inc.tail);
    }
clear_slowpath:
    /*未配置CONFIG_PARAVIRT_SPINLOCKS选项，下面函数啥也不做*/
    __ticket_check_and_clear_slowpath(lock, inc.head);
out:
    barrier();  /* make sure nothing creeps before the lock is taken */
}

static __always_inline int arch_spin_trylock(arch_spinlock_t *lock)
{
    arch_spinlock_t old, new;

    /*取出 Owner 与 Next */
    old.tickets = READ_ONCE(lock->tickets);
    /*Owner（.head）与Next（.tail）不等，锁已被其他线程持有，返回 0 */
    if (!__tickets_equal(old.tickets.head, old.tickets.tail))
        return 0;
    /*Owner与Next相等，开始尝试持有锁，仅仅是开始，这里操作的是栈上的变量new。
      Next（.tail）放在高位，故要将锁增量TICKET_LOCK_INC左移到高位再相加。
     */
    new.head_tail = old.head_tail + (TICKET_LOCK_INC << TICKET_SHIFT);
    new.head_tail &= ~TICKET_SLOWPATH_FLAG;

    /* cmpxchg is a full barrier, so nothing can move before it */
    /*cmpxchg族指令实现原子地比较并交换，由于前面插入了lock指令，因此cmpxchg指令
      执行时锁内存总线，别的线程无法更新锁。
      cmpxchg 比较 old.head_tail 与 lock->head_tail地址里的值：
      如果相等，将 new.head_tail 的值存入 lock->head_tail地址指向的内存，返回原来 lock->head_tail 地址里的值。
      故返回值如果等于 old.head_tail 表示成功。执行线程由此获得锁，返回true。
      此举原子地完成了比较和更新 Next （占用锁）的过程。
     */
    return cmpxchg(&lock->head_tail, old.head_tail, new.head_tail) == old.head_tail;
}

static __always_inline void arch_spin_unlock(arch_spinlock_t *lock)
{
    if (TICKET_SLOWPATH_FLAG &&
        static_key_false(&paravirt_ticketlocks_enabled)) {
        __ticket_t head;

        BUILD_BUG_ON(((__ticket_t)NR_CPUS) != NR_CPUS);

        head = xadd(&lock->tickets.head, TICKET_LOCK_INC);

        if (unlikely(head & TICKET_SLOWPATH_FLAG)) {
            head &= ~TICKET_SLOWPATH_FLAG;
            __ticket_unlock_kick(lock, (head + TICKET_LOCK_INC));
        }
    } else
       /*解锁只需将 Owner（.head）原子地加 1 即可*/
        __add(&lock->tickets.head, TICKET_LOCK_INC, UNLOCK_LOCK_PREFIX);
}

static inline int arch_spin_is_locked(arch_spinlock_t *lock)
{
    struct __raw_tickets tmp = READ_ONCE(lock->tickets);
    /*判断是否能锁已被持有只需判断Next（.tail）与Owner（.head）是否一样即可*/
    return !__tickets_equal(tmp.tail, tmp.head);
}
```

* arch/x86/include/asm/cmpxchg.h
```c
/*
 * An exchange-type operation, which takes a value and a pointer, and
 * returns the old value.
 */
#define __xchg_op(ptr, arg, op, lock)                   \
    ({                              \
            __typeof__ (*(ptr)) __ret = (arg);          \
        switch (sizeof(*(ptr))) {               \
        case __X86_CASE_B:                  \
            asm volatile (lock #op "b %b0, %1\n"        \
                      : "+q" (__ret), "+m" (*(ptr)) \
                      : : "memory", "cc");      \
            break;                      \
        case __X86_CASE_W:                  \
            asm volatile (lock #op "w %w0, %1\n"        \
                      : "+r" (__ret), "+m" (*(ptr)) \
                      : : "memory", "cc");      \
            break;                      \
        case __X86_CASE_L:                  \
            asm volatile (lock #op "l %0, %1\n"     \
                      : "+r" (__ret), "+m" (*(ptr)) \
                      : : "memory", "cc");      \
            break;                      \
        case __X86_CASE_Q:                  \
            asm volatile (lock #op "q %q0, %1\n"        \
                      : "+r" (__ret), "+m" (*(ptr)) \
                      : : "memory", "cc");      \
            break;                      \
        default:                        \
            __ ## op ## _wrong_size();          \
        }                           \
        __ret;                          \
    })
...

/*
 * xadd() adds "inc" to "*ptr" and atomically returns the previous
 * value of "*ptr".
 *
 * xadd() is locked when multiple CPUs are online
 * xadd_sync() is always locked
 * xadd_local() is never locked
 */
#define __xadd(ptr, inc, lock)  __xchg_op((ptr), (inc), xadd, lock)
#define xadd(ptr, inc)      __xadd((ptr), (inc), LOCK_PREFIX)
...
#define __add(ptr, inc, lock)                       \
    ({                              \
            __typeof__ (*(ptr)) __ret = (inc);          \
        switch (sizeof(*(ptr))) {               \
        case __X86_CASE_B:                  \
            asm volatile (lock "addb %b1, %0\n"     \
                      : "+m" (*(ptr)) : "qi" (inc)  \
                      : "memory", "cc");        \
            break;                      \
        case __X86_CASE_W:                  \
            asm volatile (lock "addw %w1, %0\n"     \
                      : "+m" (*(ptr)) : "ri" (inc)  \
                      : "memory", "cc");        \
            break;                      \
        case __X86_CASE_L:                  \
            asm volatile (lock "addl %1, %0\n"      \
                      : "+m" (*(ptr)) : "ri" (inc)  \
                      : "memory", "cc");        \
            break;                      \
        case __X86_CASE_Q:                  \
            asm volatile (lock "addq %1, %0\n"      \
                      : "+m" (*(ptr)) : "ri" (inc)  \
                      : "memory", "cc");        \
            break;                      \
        default:                        \
            __add_wrong_size();             \
        }                           \
        __ret;                          \
    })
...
```

* arch/x86/include/asm/cmpxchg.h
```c
/*
 * Atomic compare and exchange.  Compare OLD with MEM, if identical,
 * store NEW in MEM.  Return the initial value in MEM.  Success is
 * indicated by comparing RETURN with OLD.
 */
#define __raw_cmpxchg(ptr, old, new, size, lock)            \
({                                  \
    __typeof__(*(ptr)) __ret;                   \
    __typeof__(*(ptr)) __old = (old);               \
    __typeof__(*(ptr)) __new = (new);               \
    switch (size) {                         \
    case __X86_CASE_B:                      \
    {                               \
        volatile u8 *__ptr = (volatile u8 *)(ptr);      \
        asm volatile(lock "cmpxchgb %2,%1"          \
                 : "=a" (__ret), "+m" (*__ptr)      \
                 : "q" (__new), "0" (__old)         \
                 : "memory");               \
        break;                          \
    }                               \
    case __X86_CASE_W:                      \
    {                               \
        volatile u16 *__ptr = (volatile u16 *)(ptr);        \
        asm volatile(lock "cmpxchgw %2,%1"          \
                 : "=a" (__ret), "+m" (*__ptr)      \
                 : "r" (__new), "0" (__old)         \
                 : "memory");               \
        break;                          \
    }                               \
    case __X86_CASE_L:                      \
    {                               \
        volatile u32 *__ptr = (volatile u32 *)(ptr);        \
        asm volatile(lock "cmpxchgl %2,%1"          \
                 : "=a" (__ret), "+m" (*__ptr)      \
                 : "r" (__new), "0" (__old)         \
                 : "memory");               \
        break;                          \
    }                               \
    case __X86_CASE_Q:                      \
    {                               \
        volatile u64 *__ptr = (volatile u64 *)(ptr);        \
        asm volatile(lock "cmpxchgq %2,%1"          \
                 : "=a" (__ret), "+m" (*__ptr)      \
                 : "r" (__new), "0" (__old)         \
                 : "memory");               \
        break;                          \
    }                               \
    default:                            \
        __cmpxchg_wrong_size();                 \
    }                               \
    __ret;                              \
})

#define __cmpxchg(ptr, old, new, size)                  \
    __raw_cmpxchg((ptr), (old), (new), (size), LOCK_PREFIX)
...
#define cmpxchg(ptr, old, new)                      \
    __cmpxchg(ptr, old, new, sizeof(*(ptr)))\
...
```

#### 让CPU歇一会

* arch/x86/include/asm/processor.h
```c
/* REP NOP (PAUSE) is a good thing to insert into busy-wait loops. */
static __always_inline void rep_nop(void)
{
    asm volatile("rep; nop" ::: "memory");
}

static __always_inline void cpu_relax(void)
{
    rep_nop();
}
```

* 参考Stack Overflow上一篇文章 [What does “rep; nop;” mean in x86 assembly?](http://stackoverflow.com/questions/7086220/what-does-rep-nop-mean-in-x86-assembly) 提几个问题：

##### rep; nop; 是什么意思？
   * `rep`指令的释义：Repeats a string instruction the number of times specified in the count register or until the indicated condition of the ZF flag is no longer met. 操作码为`F3`。
     * [REP/REPE/REPZ/REPNE/REPNZ—Repeat String Operation Prefix](http://www.felixcloutier.com/x86/REP:REPE:REPZ:REPNE:REPNZ.html)
   * `nop`指令的释义：One byte no-operation instruction. 操作码为`90`。
     * [NOP — No Operation](http://www.felixcloutier.com/x86/NOP.html)

##### 为什么不是 pause 指令？
   * `pause`指令的释义：Gives hint to processor that improves performance of spin-wait loops。
   * 注意：`pause`指令的操作码为`F3 90`
   * [PAUSE — Spin Loop Hint](http://www.felixcloutier.com/x86/PAUSE.html)

> **Description**
>
> Improves the performance of spin-wait loops. When executing a “spin-wait loop,” processors will suffer a severe performance penalty when exiting the loop because it detects a possible memory order violation. The PAUSE instruction provides a hint to the processor that the code sequence is a spin-wait loop. The processor uses this hint to avoid the memory order violation in most situations, which greatly improves processor performance. For this reason, it is recommended that a PAUSE instruction be placed in all spin-wait loops.
>
> An additional function of the PAUSE instruction is to reduce the power consumed by a processor while executing a spin loop. A processor can execute a spin-wait loop extremely quickly, causing the processor to consume a lot of power while it waits for the resource it is spinning on to become available. Inserting a pause instruction in a spin-wait loop greatly reduces the processor’s power consumption.
>
> This instruction was introduced in the Pentium 4 processors, but is backward compatible with all IA-32 processors. In earlier IA-32 processors, the PAUSE instruction operates like a NOP instruction. The Pentium 4 and Intel Xeon processors implement the PAUSE instruction as a delay. The delay is finite and can be zero for some processors. This instruction does not change the architectural state of the processor (that is, it performs essentially a delaying no-op operation).
>
> This instruction’s operation is the same in non-64-bit modes and 64-bit mode.

* 可见`pause`指令实现自旋等待的效果更好，原因在于：
  * 给处理器一个提示，我这里想要自旋，处理器籍此优化其性能。
  * `pause`指令实现的自旋等待循环，处理器会减少能源消耗。

* 对于不支持超线程的处理器，用`pause`指令也是有益的。
  * 现代的处理器多为[Superscalar processor](https://en.wikipedia.org/wiki/Superscalar_processor)，这意味着它会尝试同时并行地预取，译码和执行多条指令。
  * 然而在自旋等待的场景，这么做并不会提高执行速度。
  * 采用`pause`指令可以被认为是限制在流水线上的（不必要的）指令数。
  * 参考 [http://stackoverflow.com/questions/7086220/what-does-rep-nop-mean-in-x86-assembly](http://stackoverflow.com/questions/7086220/what-does-rep-nop-mean-in-x86-assembly)


* 代码里写的`rep; nop;`，实际上它们的操作码与`pause`的操作码是相同的，这是为了向后兼容。
  * `pause`指令是Pentium 4处理器引入的，这样实现向后兼容所有的IA-32处理器。
  * 早期的IA-32处理器没有上面的优化效果，但依然会自旋等待（因为`rep; nop;`指令是支持的）。
  * [此处](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2013-February/007473.html)也有人讨论过。

##### 为什么不单独用 nop 指令？

* 见讨论 [rep; Nop / -asm pause](https://software.intel.com/en-us/forums/watercooler-catchall/topic/309231)

> NOP instruction can be between 0.4-0.5 clocks and PAUSE instruction can consume 38-40 clocks. Please refer to the whitepaper on how to measure the latency and throughput of various instructions. The REPE instruction comes in various flavors and the latency/throughput of each of them varies. Please also see below for the sample code to measure the average clocks.

* 简单地说，比起用`pause`指令，用`nop`指令让CPU歇的时间更短，能耗必然会增高。

### spin_lock_bh()是先禁下半部还是先竞争锁？

* 之前讲下半部加锁时就讨论过了，应该是先禁下半部。
* 可以参考上面`__raw_spin_lock_bh()`的实现。
* include/linux/bottom_half.h
```c
#ifdef CONFIG_TRACE_IRQFLAGS
extern void __local_bh_disable_ip(unsigned long ip, unsigned int cnt);
#else
static __always_inline void __local_bh_disable_ip(unsigned long ip, unsigned int cnt)
{
    preempt_count_add(cnt);
    barrier();
}
#endif
...__
```

## 示例

动作 | Next (lock.tail) | Owner (lock.head) | Ticket (local_variable.tail) | 描述
---|---|---|---|---
初始状态 | 0 | 0 | - | -
P1加锁 | 1 | 0 | 0 | Next + 1，Owner == Ticket == 0，P1获得锁
P2加锁 | 2 | 0 | 1 | Next + 1 == 2，Ticket = Old Next = 1，Owner != Ticket，P2自旋
P3加锁 | 3 | 0 | 2 | Next + 1 == 3，Ticket = Old Next = 2，Owner != Ticket，P3自旋
P1解锁 | 3 | 1 | - | Owner + 1 == 1，P1释放锁
P2得锁 | 3 | 1 | 1 | Owner == Ticket == 1，P2获得锁
p4加锁 | 4 | 1 | 3 | Next + 1 == 4，Ticket = Old Next = 3，Owner != Ticket，P4自旋
P2解锁 | 4 | 2 | - | Owner + 1 == 2，P2释放锁
P3得锁 | 4 | 2 | 2 | Owner == Ticket == 2，P3获得锁
P3解锁 | 4 | 3 | - | Owner + 1 == 3，P3释放锁
P4得锁 | 4 | 3 | 3 | Owner == Ticket == 3，P4获得锁
P4解锁 | 4 | 4 | - | Owner + 1 == 4，P4释放锁
