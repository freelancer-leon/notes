# 时间管理

# 概念

* 内核需要在硬件的帮助下才能计算和管理时间
* **系统定时器** 是一种可编程硬件芯片，以固定频率产生中断。该中断即所谓的 **定时器中断**。
* **定时器中断处理程序** 负责 **更新系统时间**，也负责 **执行需要周期性运行的任务**。
* **动态定时器** 一种用来推迟执行的工具。
* 定时器以某种频率自行触发（击中（hitting）或射中（popping）），该频率可通过编程预定，称作 **节拍率（tick rate）**。
* 因为预编的节拍率对内核来说是可知的，所以内核知道连续两次时钟中断的间隔时间，该间隔时间成为 **节拍（tick）**。
* 节拍 = 1/节拍率 秒，tick = 1/(tick rate) 秒，可见，*节拍是一个时间概念*。
* 系统定时器频率（节拍率 tick rate）是通过静态预处理定义的，即 **HZ（赫兹）**。
  * 在系统启动时，按照HZ值对硬件进行设置。
  * 体系架构不同，HZ的值也不同。
  * 如x86体系结构中，系统定时器频率默认值为100，时钟中断频率就为100HZ，即该处理器上每秒时钟中断100次（即10ms一次）。
  * 注意，HZ的值是可调的。不要在代码中hard-coded HZ的值。

# HZ

## 理想的HZ值
* 更高的时钟中断解析度（resolution）可提高时间驱动事件的解析度
* 提高了时间驱动事件的准确度（accuracy）

## 高HZ的优劣
* 依赖定时执行的系统调用，如 poll() 和 select()，能够以更高的精度运行
* 对诸如资源消耗和系统运行时间的测量会有更精细的解析度
* 提高进程抢占的准确度

## 高HZ的劣势
* 节拍率高 -> 时钟中断频率高 -> 系统负担越重
* 中断处理程序占用时间越多，处理器处理其他工作的时间就越少
* 频繁打乱处理器的高速缓存并增加耗电

## 动态时钟
* 启用 **动态时钟** 的系统也称为 **无时钟系统（tickless system）**
* 注意：不要和后面要说的 *动态定时器* 混淆
* 内核配置选项为`CONFIG_NO_HZ`
* 动态时钟只有在有些任务需要实际执行时才激活时钟周期。否则会临时禁用时钟周期。
* 内核如何判定系统无事可做？
  * 运行队列没有活动进程，内核将选取一个特别的idle进程（调度器类为idle_task）来运行。
  * 每当选中idle进程运行时，都将禁用周期时钟，直至如下情况再重新启用时钟周期：
    * 下一个定时器即将到器为止
    * 或者有中断发生
  * 其间CPU进入不受打扰的睡眠状态。

### 注意以下几点
* 只有经典定时器需要考虑此用法。高分辨率定时器不绑定到时钟频率，也并非基于周期时钟实现。
* 单触发时钟是实现动态时钟的先决条件。纯粹的周期性的定时器根本就不是适用于该机制。

# jiffies

* 全局变量`jiffies`记录自系统启动以来产生的节拍的总数。
  * 启动时初始化为0。
  * 每次时钟中断处理程序会增加该变量的值（因此可以认为其单位为（次）），意味着时间过去了一个节拍（tick）。
  * 每秒内`jiffies`的值会增加`HZ`。
  * 系统的运行时间为`jiffies/HZ`秒（`HZ`的单位为（次/秒））。

* include/linux/jiffies.h
```c
/* some arch's have a small-data section that can be accessed register-relative
 * but that can only take up to, say, 4-byte variables. jiffies being part of
 * an 8-byte variable may not be correctly accessed unless we force the issue
 */
#define __jiffy_data  __attribute__((section(".data")))

/*
 * The 64-bit value is not atomic - you MUST NOT read it
 * without sampling the sequence number in jiffies_lock.
 * get_jiffies_64() will do this for you as appropriate.
 */
extern u64 __jiffy_data jiffies_64;
extern unsigned long volatile __jiffy_data jiffies;
/*对32位平台和64位平台，获取jiffies_64的值方法不一样，64位平台直接返回即可*/
#if (BITS_PER_LONG < 64)
u64 get_jiffies_64(void);
#else
static inline u64 get_jiffies_64(void)
{
    return (u64)jiffies;
}
#endif
...
```

* kernel/time/jiffies.c
```c
__cacheline_aligned_in_smp DEFINE_SEQLOCK(jiffies_lock);

#if (BITS_PER_LONG < 64)
u64 get_jiffies_64(void)
{
    unsigned long seq;
    u64 ret;

    do {
        seq = read_seqbegin(&jiffies_lock);
        ret = jiffies_64;
    } while (read_seqretry(&jiffies_lock, seq));
    return ret;
}   
EXPORT_SYMBOL(get_jiffies_64);
#endif

EXPORT_SYMBOL(jiffies);
```
* 32位体系结构不能原子地一次访问64位变量中的两个32位数值。因此在读取`jiffies`时需用seq锁对变量`jiffies`进行锁定。
* 这里用`jiffies_64`变量的初值覆盖`jiffies`变量
  * arch/x86/kernel/vmlinux.lds.S
  ```c
  #ifdef CONFIG_X86_32
  OUTPUT_ARCH(i386)
  ENTRY(phys_startup_32)
  jiffies = jiffies_64;
  #else
  OUTPUT_ARCH(i386:x86-64)
  ENTRY(phys_startup_64)
  jiffies_64 = jiffies;
  #endif
  ```
* `jiffies`取整个64位`jiffies_64`变量的低32位，时间管理代码使用整个64位，来避免整个64位的溢出。
* 访问`jiffies`的代码仅会读取`jiffies_64`的低32位。通过`get_jiffies_64()`读取整个64位数值。

## jiffies的回绕
* `jiffies`有可能会发生回绕（wrap around），最好用提供的宏来进行判断。
* include/linux/typecheck.h
```c
...
/*
 *  These inlines deal with timer wrapping correctly. You are
 *  strongly encouraged to use them
 *  1. Because people otherwise forget
 *  2. Because if the timer wrap changes in future you won't have to
 *     alter your driver code.
 *                                                                                                                                                 
 * time_after(a,b) returns true if the time a is after time b.
 *
 * Do this with "<0" and ">=0" to only test the sign of the result. A
 * good compiler would generate better code (and a really good compiler
 * wouldn't care). Gcc is currently neither.
 */
#define time_after(a,b)     \
    (typecheck(unsigned long, a) && \
     typecheck(unsigned long, b) && \
     ((long)((b) - (a)) < 0))
#define time_before(a,b)    time_after(b,a)

#define time_after_eq(a,b)  \
    (typecheck(unsigned long, a) && \
     typecheck(unsigned long, b) && \
     ((long)((a) - (b)) >= 0))
#define time_before_eq(a,b) time_after_eq(b,a)
...
```
* 注意，`time_after`和`time_before`并不能完全解决回绕（wrap around）问题：
  * 例如，想用`time_after(a,b)`来判断`a`时间是否在`b`之后，假设，`b`的值是 2<sup>31</sup>-1 = 2147483647，HZ=1000，即一个tick是1ms，则经过 2147483649 个tick之后，`jiffies`回绕，此时`(long)(b-a)>0`，判断结果是`a`在`b`之前，然而事实上`a`确实已经是在`b`之后。
  * 再假如，想用`time_before(a,b)`来判断`a`时间是否在`b`之前，假设，`b`的值是 0，HZ=1000，即一个tick是1ms，则经过 2<sup>31</sup>-1 = 2147483648 个tick之后，`jiffies`回绕，此时`(long)(a-b)<0`，判断结果是`a`在`b`之前，然而事实上`a`确实已经是在`b`之后很久了。
  * 可见，该宏的使用前提是`a`和`b`的逻辑差值在2<sup>31</sup>（如果HZ是1000则大概是24.86天）之内，否则有可能会因为回绕而不再准确。

## 用户空间和HZ
* 如果在内核中修改了HZ的定义值，用户空间并不知道新HZ值，这就打破了用户空间的常量关系，从而导致问题。
* 内核定义`USER_HZ`来代表用户空间看到的HZ值。
* 在需要把以节拍数/秒为单位的`jiffies`值导出到用户空间时，需要用`jiffies_to_clock_t()`或`jiffies_64_to_clock_t()`，将一个由`HZ`表示的节拍计数转换成用`USER_HZ`表示的节拍计数：
* include/linux/jiffies.h
```c
/* TICK_NSEC is the time between ticks in nsec assuming SHIFTED_HZ */
/*TICK_NSEC是两个tick之间逝去的纳秒数。
  由于整数除法小数点后的数会被截断，这里加上HZ/2是为了四舍五入。
  整型数除法四舍五入的做法就是被除数加上除数的1/2后再去做除法运算。
  例如：HZ为60，如果采用浮点数，那么结果是：
  (1000000000+60/2)/60 = 16666667.166667
  如果用的整数运算：
  （1000000000+60/2)/60 = 16666667
  1000000000/60 = 16666666
 */
#define TICK_NSEC ((NSEC_PER_SEC+HZ/2)/HZ)
```
* kernel/time/time.c
```c
...
/*
 * Convert jiffies/jiffies_64 to clock_t and back.
 */
clock_t jiffies_to_clock_t(unsigned long x)
{
/*TICK_NSEC是内核一个tick的纳秒时间，(NSEC_PER_SEC / USER_HZ)是用户空间一个tick的纳秒时间*/
#if (TICK_NSEC % (NSEC_PER_SEC / USER_HZ)) == 0
/*当HZ与USER_HZ是整倍数关系时，逝去的时间 T 是一样的，于是有：
  T = x / HZ = y / USER_HZ
  则用户空间的节拍计数 y 为：
  y = x * USER_HZ / HZ*/
# if HZ < USER_HZ
    return x * (USER_HZ / HZ);
# else
    /*HZ > USER_HZ时，整数除法 USER_HZ / HZ == 0，故需要变换一下方程*/
    return x / (HZ / USER_HZ);
# endif
#else
    /*用户空间每个tick的时间以ns为单位为：NSEC_PER_SEC*1/USER_HZ，
      内核在某段时间里的用去的时间以ns为单位为：(u64)x * TICK_NSEC，
      故二者的商为用户空间在某段时间里的节拍计数。*/
    return div_u64((u64)x * TICK_NSEC, NSEC_PER_SEC / USER_HZ);
#endif
}
EXPORT_SYMBOL(jiffies_to_clock_t);
...
```

# 硬时钟和定时器

## 实时时钟
* **RTC**，用来持久存放系统时间的设备，系统关闭后也可靠主板上的微型电池供电，保持系统的计时。
* PC架构中，RTC与CMOS集成在一起，RTC的运行与BIOS的保存设置都是通过同一电池供电。
* 系统启动时，内核通过读取RTC来初始化墙上时间，该时间存放在`xtime`变量中。这是它最主要的作用。

## 系统定时器
* 系统定时器的根本思想——提供一种周期性触发中断机制
* 对电子晶振进行分频来实现；或者提供一个衰减测量器（decrementer）来实现（设置测量器初始值，该值以固定频率衰减，衰减到0时触发一个中断）
* x86主要采用可编程中断时钟（PIT）
* x86其他时钟资源还包括本地的APIC和时间戳（TSC）等

# 时钟中断处理程序
可以划分为体系结构相关部分和与体系结构无关部分。1/HZ秒发生一次。

### 体系结构相关
* 作为系统定时器的中断处理程序而注册到内核中。
* 获得`xtime_lock`锁，以便对访问`jiffies_64`和墙上时间`xtime`进行保护。
* 需要时，应答或重新设置系统时钟。
* 周期性地使用 *墙上时间* 更新 *实时时钟*。
* 调用体系结构无关的时钟例程`time_periodic()`。

### 体系结构无关
* 给`jiffies_64`变量增加 **1**（这个操作即使在32位体系结构上也是安全的，因为前面已经获得了`xtime_lock`锁）。
* 更新 *资源消耗的统计值*，比如当前进程所消耗的系统时间和用户时间。
* 执行已经到期的 *动态定时器*。
* 执行`scheduler_tick()`函数。
* 更新 *墙上时间*，该时间存放在`xtime`变量中。
* 计算 *平均负载值*。

* kernel/time/tick-common.c
```c
...
/*  
 * Periodic tick
 */
static void tick_periodic(int cpu)
{
    if (tick_do_timer_cpu == cpu) {
        write_seqlock(&jiffies_lock);

        /* Keep track of the next tick event */ /*记录下一个节拍事件*/
        tick_next_period = ktime_add(tick_next_period, tick_period);

        do_timer(1);
        write_sequnlock(&jiffies_lock);
        update_wall_time(); /*更新墙上时间*/
    }   
    /*通过查看系统寄存器来查看该tick是花费在用户空间还是内核空间*/
    update_process_times(user_mode(get_irq_regs()));
    profile_tick(CPU_PROFILING);  /*内核profile入口点*/
}   
...
```
* `jiffies_lock`是seq锁，之前讲`get_jiffies_64()`时提到过。

* kernel/time/timekeeping.c
```c
...
/*
 * Must hold jiffies_lock
 */
void do_timer(unsigned long ticks)
{
    jiffies_64 += ticks;     /*给`jiffies_64`变量增加*/
    calc_global_load(ticks); /*计算平均负载值*/
}
...
```

* kernel/time/timer.c
```c
/*
 * Called from the timer interrupt handler to charge one tick to the current
 * process.  user_tick is 1 if the tick is user time, 0 for system.
 */
void update_process_times(int user_tick)
{
    struct task_struct *p = current;

    /* Note: this timer irq context must be accounted for as well. */
    account_process_tick(p, user_tick); /*更新资源消耗的统计值*/
    run_local_timers();                 /*触发softirq去执行已经到期的动态定时器*/
    rcu_check_callbacks(user_tick);     /*TODO:rcu*/
#ifdef CONFIG_IRQ_WORK
    if (in_irq())
        irq_work_tick();                /*TODO:irq_work*/
#endif
    scheduler_tick();                   /*进入周期性调度的入口点*/
    run_posix_cpu_timers(p);
}
...
/*
 * Called by the local, per-CPU timer interrupt on SMP.
 */
void run_local_timers(void)
{
    hrtimer_run_queues();          /*TODO:hrtimer*/
    raise_softirq(TIMER_SOFTIRQ);  /*触发定时器softirq*/
}
...
```
* arch/x86/include/asm/ptrace.h
```c
/*
 * user_mode(regs) determines whether a register set came from user
 * mode.  On x86_32, this is true if V8086 mode was enabled OR if the
 * register set was from protected mode with RPL-3 CS value.  This
 * tricky test checks that with one comparison.
 *
 * On x86_64, vm86 mode is mercifully nonexistent, and we don't need
 * the extra check.
 */
 static inline int user_mode(struct pt_regs *regs)
{
#ifdef CONFIG_X86_32
    return ((regs->cs & SEGMENT_RPL_MASK) | (regs->flags & X86_VM_MASK)) >= USER_RPL;
#else
    return !!(regs->cs & 3);
#endif
}
/*用户空间返回 1，内核空间返回 0*/
...
```

* kernel/sched/cputime.c
```c
/*
 * Account a single tick of cpu time.
 * @p: the process that the cpu time gets accounted to
 * @user_tick: indicates if the tick is a user or a system tick
 */
void account_process_tick(struct task_struct *p, int user_tick)
{       
    cputime_t one_jiffy_scaled = cputime_to_scaled(cputime_one_jiffy);
    struct rq *rq = this_rq();

    if (vtime_accounting_cpu_enabled())
        return;

    if (sched_clock_irqtime) {
        irqtime_account_process_tick(p, user_tick, rq, 1);
        return;
    }

    if (steal_account_process_tick())
        return;
    /*用户空间用时、内核空间用时和空间时间分别计算*/
    if (user_tick)
        account_user_time(p, cputime_one_jiffy, one_jiffy_scaled);
    else if ((p != rq->idle) || (irq_count() != HARDIRQ_OFFSET))
        account_system_time(p, HARDIRQ_OFFSET, cputime_one_jiffy,
                    one_jiffy_scaled);
    else
        account_idle_time(cputime_one_jiffy);
}
```

* TODO：大概过程，尚需细琢。
```
x86_64_start_kernel()
        |
        V
x86_64_start_reservations()
        |
        +-----------------------------------------------+
        |                                               |
        V                                               |
x86_intel_mid_early_setup(){                            |
  x86_init.timers.timer_init = intel_mid_time_init;     |
}                                       |               |
        +-------------------------------+               |
        |                                               V
        |                                         start_kernel()
        V
intel_mid_time_init()
{
  x86_init.timers.setup_percpu_clockev = setup_boot_APIC_clock;
}                                                 |
        +-----------------------------------------+
        |
        V
setup_boot_APIC_clock()
        |
        V
setup_APIC_timer()
        |
        V
clockevents_register_device()
        |
        V
tick_check_new_device()
        |
        V
tick_setup_device()
        |
        V
tick_setup_periodic()
        |
        V
tick_set_periodic_handler()
{
  if (!broadcast)
      dev->event_handler = tick_handle_periodic;
  else                              |
      dev->event_handler = tick_handle_periodic_broadcast;
}                                   |
        +---------------------------+
        |
        V
tick_handle_periodic()
        |
        V
tick_periodic()
```

# 实际时间
* 追踪一下`sys_gettimeofday`系统调用的实现来看一下实际时间是怎么得到的：
* kernel/time/time.c
```c
/*
 * The timezone where the local system is located.  Used as a default by some
 * programs who obtain this value by using gettimeofday.
 */
struct timezone sys_tz;

EXPORT_SYMBOL(sys_tz);
...
SYSCALL_DEFINE2(gettimeofday, struct timeval __user *, tv,
        struct timezone __user *, tz)
{
    if (likely(tv != NULL)) {
        struct timeval ktv;
        do_gettimeofday(&ktv);
        if (copy_to_user(tv, &ktv, sizeof(ktv)))
            return -EFAULT;
    }
    if (unlikely(tz != NULL)) {
        if (copy_to_user(tz, &sys_tz, sizeof(sys_tz)))
            return -EFAULT;
    }
    return 0;
}
...__
```
* kernel/time/timekeeping.c
```c
#ifdef CONFIG_DEBUG_TIMEKEEPING
...
static inline cycle_t timekeeping_get_delta(struct tk_read_base *tkr)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    cycle_t now, last, mask, max, delta;
    unsigned int seq;

    /*
     * Since we're called holding a seqlock, the data may shift
     * under us while we're doing the calculation. This can cause
     * false positives, since we'd note a problem but throw the
     * results away. So nest another seqlock here to atomically
     * grab the points we are checking with.
     */
    do {
        seq = read_seqcount_begin(&tk_core.seq);
        now = tkr->read(tkr->clock);
        last = tkr->cycle_last;
        mask = tkr->mask;
        max = tkr->clock->max_cycles;
    } while (read_seqcount_retry(&tk_core.seq, seq));

    delta = clocksource_delta(now, last, mask);

    /*
     * Try to catch underflows by checking if we are seeing small
     * mask-relative negative values.
     */
    if (unlikely((~delta & mask) < (mask >> 3))) {
        tk->underflow_seen = 1;
        delta = 0;
    }

    /* Cap delta value to the max_cycles values to avoid mult overflows */
    if (unlikely(delta > max)) {
        tk->overflow_seen = 1;
        delta = tkr->clock->max_cycles;
    }

    return delta;
}
#else
...
static inline cycle_t timekeeping_get_delta(struct tk_read_base *tkr)
{
    cycle_t cycle_now, delta;

    /* read clocksource */
    cycle_now = tkr->read(tkr->clock);

    /* calculate the delta since the last update_wall_time */
    delta = clocksource_delta(cycle_now, tkr->cycle_last, tkr->mask);

    return delta;
}
#endif
...
static inline s64 timekeeping_delta_to_ns(struct tk_read_base *tkr,
                      cycle_t delta)
{
    s64 nsec;

    nsec = delta * tkr->mult + tkr->xtime_nsec;
    nsec >>= tkr->shift;

    /* If arch requires, add in get_arch_timeoffset() */
    return nsec + arch_gettimeoffset();
}

static inline s64 timekeeping_get_ns(struct tk_read_base *tkr)
{
    cycle_t delta;

    delta = timekeeping_get_delta(tkr);
    return timekeeping_delta_to_ns(tkr, delta);
}
...
/**
 * __getnstimeofday64 - Returns the time of day in a timespec64.
 * @ts:     pointer to the timespec to be set
 *
 * Updates the time of day in the timespec.
 * Returns 0 on success, or -ve when suspended (timespec will be undefined).
 */
int __getnstimeofday64(struct timespec64 *ts)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    unsigned long seq;
    s64 nsecs = 0;

    do {
        seq = read_seqcount_begin(&tk_core.seq);

        ts->tv_sec = tk->xtime_sec;
        nsecs = timekeeping_get_ns(&tk->tkr_mono);

    } while (read_seqcount_retry(&tk_core.seq, seq));

    ts->tv_nsec = 0;
    timespec64_add_ns(ts, nsecs);

    /*
     * Do not bail out early, in case there were callers still using
     * the value, even in the face of the WARN_ON.
     */
    if (unlikely(timekeeping_suspended))
        return -EAGAIN;
    return 0;
}
EXPORT_SYMBOL(__getnstimeofday64);

/**
 * getnstimeofday64 - Returns the time of day in a timespec64.
 * @ts:     pointer to the timespec64 to be set
 *
 * Returns the time of day in a timespec64 (WARN if suspended).
 */
void getnstimeofday64(struct timespec64 *ts)
{
    WARN_ON(__getnstimeofday64(ts));
}
EXPORT_SYMBOL(getnstimeofday64);
...
/**
 * do_gettimeofday - Returns the time of day in a timeval
 * @tv:     pointer to the timeval to be set
 *
 * NOTE: Users should be converted to using getnstimeofday()
 */
void do_gettimeofday(struct timeval *tv)
{
    struct timespec64 now;

    getnstimeofday64(&now);
    tv->tv_sec = now.tv_sec;
    tv->tv_usec = now.tv_nsec/1000;
}   
EXPORT_SYMBOL(do_gettimeofday);
...
```

* kernel/time/timekeeping_internal.h
```c
...
#ifdef CONFIG_CLOCKSOURCE_VALIDATE_LAST_CYCLE
static inline cycle_t clocksource_delta(cycle_t now, cycle_t last, cycle_t mask)
{
    cycle_t ret = (now - last) & mask;

    /*
     * Prevent time going backwards by checking the MSB of mask in
     * the result. If set, return 0.
     */
    return ret & ~(mask >> 1) ? 0 : ret;
}   
#else
static inline cycle_t clocksource_delta(cycle_t now, cycle_t last, cycle_t mask)
{
    return (now - last) & mask;
}
#endif
...
```

* include/linux/time64.h
```c
/**
 * timespec64_add_ns - Adds nanoseconds to a timespec64
 * @a:      pointer to timespec64 to be incremented
 * @ns:     unsigned nanoseconds value to be added
 *
 * This must always be inlined because its used from the x86-64 vdso,
 * which cannot call other kernel functions.
 */
static __always_inline void timespec64_add_ns(struct timespec64 *a, u64 ns)
{
    a->tv_sec += __iter_div_u64_rem(a->tv_nsec + ns, NSEC_PER_SEC, &ns);
    a->tv_nsec = ns;
}
/*.tv_sec存经过的秒数，所以加上商；.tv_nsec记录的是自上一秒开始经过的ns数，所以用余数。*/
...
```
* include/linux/math64.h
```c
...
/* 64位除法，通过一轮循环减法，得到商和余数。
   dividend：被除数
   divisor：除数
   remainder：余数
   返回值：商
 */
static __always_inline u32
__iter_div_u64_rem(u64 dividend, u32 divisor, u64 *remainder)
{
    u32 ret = 0;

    while (dividend >= divisor) {
        /* The following asm() prevents the compiler from
           optimising this loop into a modulo operation.  */
        asm("" : "+rm"(dividend)); /*防止编译器把该循环优化成求模运算*/

        dividend -= divisor;
        ret++;
    }
    /*余数*/
    *remainder = dividend;
    /*商*/
    return ret;
}
...
```
* 系统调用`settimeofday`需要具有`CAP_SYS_TIME`权能。

# 定时器

* 定时器，动态定时器，内核定时器 —— 管理内核流逝时间的基础。使工作在指定的时间点上执行。
* 使用定时器：
  * 初始化，设置超时，指定超时发生后执行的函数
  * 激活定时器
* 超时后定时器自动撤销。动态定时器不断地创建和撤销，运行次数也不受限制。

## 使用定时器
* 定时器结构
* include/linux/timer.h
```c
struct timer_list {
    /*
     * All fields that change during normal runtime grouped to the
     * same cacheline
     */
    struct hlist_node   entry;    /*定时器链表的入口*/
    unsigned long       expires;  /*以jiffies为单位的定时器值*/
    void            (*function)(unsigned long); /*定时器处理函数*/
    unsigned long       data;     /*传给处理函数的长整型参数*/
    u32         flags;
    int         slack;
    ...
};
...
```
* 使用示例
```c
struct timer_list my_timer;
init_timer(&my_timer);
my_timer.expires = jiffies + delay; /* timer expires in delay ticks */
my_timer.data = 0;                  /* zero is passed to the timer handler */
my_timer.function = my_function;    /* function to run when timer expires */
add_timer(&my_timer);               /*激活定时器*/
...
/*更改已激活的定时器的超时时间。
  如果定时器未激活，该操作会激活它，且设置新的超时时间。
  如果定时器未被激活，该函数返回 0，否则返回 1。
 */
mod_timer(&my_timer, jiffies + new_delay);
...
/*在定时器超时前停止定时器。
  如果定时器未被激活，该函数返回 0，否则返回 1。
  不需要为已超时的定时器调用该函数，因为它们会自动删除。
 */
del_timer(&my_timer);
...
/*在多处理器上定时器中断有可能已经在其他处理上运行了，所以删除操作要等待定时器处理程序都退出。
  这时应该使用del_timer_sync()。
 */
del_timer_sync(&my_timer);
```
* 一般定时器都在超时后马上执行，但也有可能推迟到下一个tick，所以定时器不能用来实现任何硬实时任务。

## 定时器竞争条件
* 不要尝试删除-修改-添加timer的方式来代替`mod_timer()`，无法在SMP情况下保证安全。
* 一般情况下用`del_timer_sync()`，而不是`del_timer()`。
* 因为内核异步执行中断处理程序，所以要重点保护 *定时器处理程序* 中的共享数据（即中断与软中断间共享了数据）。
* 注意：和`del_timer()`不同，`del_timer_sync()`不能用于中断上下文，可能导致死锁。

## 定时器的实现（调用）
* 之前在`run_local_timers()`中已经看到了每个tick会调用`raise_softirq(TIMER_SOFTIRQ)`激活定时器softirq。
* `start_kernel()`会调用`init_timers()`
  * kernel/time/timer.c
```c
void __init init_timers(void)
{
    init_timer_cpus();
    init_timer_stats();
    timer_register_cpu_notifier();
    open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
}
```
* 定时器softirq的处理函数为`run_timer_softirq()`
  * kernel/time/timer.c
```c
/*
 * per-CPU timer vector definitions:
 */
#define TVN_BITS (CONFIG_BASE_SMALL ? 4 : 6)
#define TVR_BITS (CONFIG_BASE_SMALL ? 6 : 8)
#define TVN_SIZE (1 << TVN_BITS)
#define TVR_SIZE (1 << TVR_BITS)
#define TVN_MASK (TVN_SIZE - 1)
#define TVR_MASK (TVR_SIZE - 1)
#define MAX_TVAL ((unsigned long)((1ULL << (TVR_BITS + 4*TVN_BITS)) - 1))
...
struct tvec_base {
    spinlock_t lock;
    struct timer_list *running_timer;
    unsigned long timer_jiffies;
    unsigned long next_timer;
    unsigned long active_timers;
    unsigned long all_timers;
    int cpu;
    bool migration_enabled;
    bool nohz_active;
    struct tvec_root tv1;
    struct tvec tv2;
    struct tvec tv3;
    struct tvec tv4;
    struct tvec tv5;
} ____cacheline_aligned;
...
static int cascade(struct tvec_base *base, struct tvec *tv, int index)
{
    /* cascade all the timers from tv up one level */
    struct timer_list *timer;
    struct hlist_node *tmp;
    struct hlist_head tv_list;

    hlist_move_list(tv->vec + index, &tv_list);

    /*
     * We are removing _all_ timers from the list, so we
     * don't have to detach them individually.
     */
    hlist_for_each_entry_safe(timer, tmp, &tv_list, entry) {
        /* No accounting, while moving them */
        __internal_add_timer(base, timer);
    }

    return index;
}

static void call_timer_fn(struct timer_list *timer, void (*fn)(unsigned long),
              unsigned long data)
{
    int count = preempt_count();

#ifdef CONFIG_LOCKDEP
    /*
     * It is permissible to free the timer from inside the
     * function that is called from it, this we need to take into
     * account for lockdep too. To avoid bogus "held lock freed"
     * warnings as well as problems when looking into
     * timer->lockdep_map, make a copy and use that here.
     */
    struct lockdep_map lockdep_map;

    lockdep_copy_map(&lockdep_map, &timer->lockdep_map);
#endif
    /*
     * Couple the lock chain with the lock chain at
     * del_timer_sync() by acquiring the lock_map around the fn()
     * call here and in del_timer_sync().
     */
    lock_map_acquire(&lockdep_map);

    trace_timer_expire_entry(timer);
    fn(data);
    trace_timer_expire_exit(timer);

    lock_map_release(&lockdep_map);

    if (count != preempt_count()) {
        WARN_ONCE(1, "timer: %pF preempt leak: %08x -> %08x\n",
              fn, count, preempt_count());
        /*
         * Restore the preempt count. That gives us a decent
         * chance to survive and extract information. If the
         * callback kept a lock held, bad luck, but not worse
         * than the BUG() we had.
         */
        preempt_count_set(count);
    }
}

#define INDEX(N) ((base->timer_jiffies >> (TVR_BITS + (N) * TVN_BITS)) & TVN_MASK)

/**
 * __run_timers - run all expired timers (if any) on this CPU.
 * @base: the timer vector to be processed.
 *
 * This function cascades all vectors and executes all expired timer
 * vectors.
 */
static inline void __run_timers(struct tvec_base *base)
{
    struct timer_list *timer;
    /*base指向的是将要被处理的定时器向量，这里先禁中断的方式锁住该向量__*/
    spin_lock_irq(&base->lock);

    while (time_after_eq(jiffies, base->timer_jiffies)) {
        struct hlist_head work_list;
        struct hlist_head *head = &work_list;
        int index;
        /*如果该向量上没有定时器，则更新timer_jiffies后退出循环*/
        if (!base->all_timers) {
            base->timer_jiffies = jiffies;
            break;
        }
        /*TVR_MASK为6位或8位，这里计算得到索引*/
        index = base->timer_jiffies & TVR_MASK;

        /*
         * Cascade timers:
         */
        if (!index &&
            (!cascade(base, &base->tv2, INDEX(0))) &&
                (!cascade(base, &base->tv3, INDEX(1))) &&
                    !cascade(base, &base->tv4, INDEX(2)))
            cascade(base, &base->tv5, INDEX(3));
        ++base->timer_jiffies;
        hlist_move_list(base->tv1.vec + index, head);
        while (!hlist_empty(head)) {
            void (*fn)(unsigned long);
            unsigned long data;
            bool irqsafe;

            timer = hlist_entry(head->first, struct timer_list, entry);
            fn = timer->function;
            data = timer->data;
            irqsafe = timer->flags & TIMER_IRQSAFE;

            timer_stats_account_timer(timer);

            base->running_timer = timer;
            detach_expired_timer(timer, base);
            if (irqsafe) {
                spin_unlock(&base->lock);
                call_timer_fn(timer, fn, data);
                spin_lock(&base->lock);
            } else {
                spin_unlock_irq(&base->lock);
                call_timer_fn(timer, fn, data);
                spin_lock_irq(&base->lock);
            }
        }
    }
    base->running_timer = NULL;
    spin_unlock_irq(&base->lock);
}
...
/*
 * This function runs timers and the timer-tq in bottom half context.
 */
static void run_timer_softirq(struct softirq_action *h)
{
    struct tvec_base *base = this_cpu_ptr(&tvec_bases);

    if (time_after_eq(jiffies, base->timer_jiffies))
        __run_timers(base);
}
...*
```

* init/Kconfig
```sh
config BASE_FULL
    default y
    bool "Enable full-sized data structures for core" if EXPERT
    help
      Disabling this option reduces the size of miscellaneous core
      kernel data structures. This saves memory on small machines,
      but may reduce performance.
...
config BASE_SMALL
    int
    default 0 if BASE_FULL
    default 1 if !BASE_FULL      
...
```

# 延迟执行
* 延迟执行（Delaying Execution）——除了 *定时器* 与 *下半部* 之外的来推迟执行任务的方法。

## 忙等待（Busy Looping）
* 事实上，所有延迟方法在进程上下文中使用得很好，因为中断处理程序都应该尽可能快地执行。
* 延迟执行不管在哪种情况下，都不应该在持有锁时或者禁止中断时发生。
* 示例
  ```c
  unsigned long delay = jiffies + 5*HZ; /*延迟5秒*/

  while (time_before(jiffies, delay))
    cond_resched();
  ```

## 短延迟（Small Delays）
* 利用忙循环的方式延迟指定的时间。
```c
void udelay(unsigned long usecs);
void ndelay(unsigned long nsecs);
void mdelay(unsigned long msecs);
```
* BogoMIPS —— Its name is a contraction of bogus (that is, fake) and MIPS (million of instructions per second)
* **BogoMIPS** 值记录处理器在给定时间内忙循环执行的次数。
  * 存放在变量`loops_per_jiffy`中。
  * 可从`/proc/cpuinfo`文件中读到。
  * 延迟循环函数使用`loops_per_jiffy`值来计算（相当准确）为提供精确延迟而需要进行多少次循环。
* `loops_per_jiffy`在启动时由`calibrate_delay()`计算。
* `loops_per_jiffy`可以用kernel命令行参数`lpj=n`进行设置。
  * init/calibrate.c
```c
...
unsigned long preset_lpj;
static int __init lpj_setup(char *str)
{
    preset_lpj = simple_strtoul(str,NULL,0);
    return 1;
}

__setup("lpj=", lpj_setup);
...*...
/*
 * This is the number of bits of precision for the loops_per_jiffy.  Each
 * time we refine our estimate after the first takes 1.5/HZ seconds, so try
 * to start with a good estimate.
 * For the boot cpu we can skip the delay calibration and assign it a value
 * calculated based on the timer frequency.
 * For the rest of the CPUs we cannot assume that the timer frequency is same as
 * the cpu frequency, hence do the calibration for those.
 */
#define LPS_PREC 8

static unsigned long calibrate_delay_converge(void)
{
    /* First stage - slowly accelerate to find initial bounds */
    unsigned long lpj, lpj_base, ticks, loopadd, loopadd_base, chop_limit;
    int trials = 0, band = 0, trial_in_band = 0;

    lpj = (1<<12);

    /* wait for "start of" clock tick */
    ticks = jiffies;
    while (ticks == jiffies)
        ; /* nothing */
    /* Go .. */
    ticks = jiffies;
    do {
        if (++trial_in_band == (1<<band)) {
            ++band;
            trial_in_band = 0;
        }
        __delay(lpj * band);
        trials += band;
    } while (ticks == jiffies);
    /*
     * We overshot, so retreat to a clear underestimate. Then estimate
     * the largest likely undershoot. This defines our chop bounds.
     */
    trials -= band;
    loopadd_base = lpj * band;
    lpj_base = lpj * trials;


recalibrate:
    lpj = lpj_base;
    loopadd = loopadd_base;

    /*      
     * Do a binary approximation to get lpj set to
     * equal one clock (up to LPS_PREC bits)
     */
    chop_limit = lpj >> LPS_PREC;
    while (loopadd > chop_limit) {
        lpj += loopadd;
        ticks = jiffies;
        while (ticks == jiffies)
            ; /* nothing */
        ticks = jiffies;
        __delay(lpj);
        if (jiffies != ticks)   /* longer than 1 tick */
            lpj -= loopadd;
        loopadd >>= 1;
    }
    /*
     * If we incremented every single time possible, presume we've
     * massively underestimated initially, and retry with a higher
     * start, and larger range. (Only seen on x86_64, due to SMIs)
     */
    if (lpj + loopadd * 2 == lpj_base + loopadd_base * 2) {
        lpj_base = lpj;
        loopadd_base <<= 2;
        goto recalibrate;
    }

    return lpj;
}
...
void calibrate_delay(void)
{
    unsigned long lpj;
    static bool printed;
    int this_cpu = smp_processor_id();

    if (per_cpu(cpu_loops_per_jiffy, this_cpu)) {
        /*当前CPU的lpj已经被设置过了，无需再次校准*/
        lpj = per_cpu(cpu_loops_per_jiffy, this_cpu);
        if (!printed)
            pr_info("Calibrating delay loop (skipped) "
                "already calibrated this CPU");
    } else if (preset_lpj) {
        /*lpj通过kernel命令行参数配置指定过了，无需校准*/
        lpj = preset_lpj;
        if (!printed)
            pr_info("Calibrating delay loop (skipped) "
                "preset value.. ");
    } else if ((!printed) && lpj_fine) {
        /*lpj已由体系架构时钟相关的初始化过程校准过了，此处无需校准*/
        lpj = lpj_fine;
        pr_info("Calibrating delay loop (skipped), "
            "value calculated using timer frequency.. ");
    } else if ((lpj = calibrate_delay_is_known())) {
        /*用体系架构相关的CPU校准延迟，有的CPU提供该数据*/
        /*
         * Check if cpu calibration delay is already known. For example,
         * some processors with multi-core sockets may have all cores
         * with the same calibration delay.
         *
         * Architectures should override this function if a faster calibration
         * method is available.
         */
        ;
    } else if ((lpj = calibrate_delay_direct()) != 0) {
        /*用体系架构相关的read_current_timer()来计算得到校准延迟*/
        /* This routine uses the read_current_timer() routine and gets the
         * loops per jiffy directly, instead of guessing it using delay().
         * Also, this code tries to handle non-maskable asynchronous events
         * (like SMIs)
         */
        if (!printed)
            pr_info("Calibrating delay using timer "
                "specific routine.. ");
    } else {
        if (!printed)
            pr_info("Calibrating delay loop... ");
        lpj = calibrate_delay_converge();
    }
    /*将lpj数据存入Per-CPU变量cpu_loops_per_jiffy*/
    per_cpu(cpu_loops_per_jiffy, this_cpu) = lpj;
    if (!printed)
        pr_cont("%lu.%02lu BogoMIPS (lpj=%lu)\n",
            lpj/(500000/HZ),
            (lpj/(5000/HZ)) % 100, lpj);

    loops_per_jiffy = lpj; /*将lpj数据存入loops_per_jiffy全局变量*/
    printed = true; /*静态变量置为true，表示已打印过了，之后不再打印*/

    calibration_delay_done();
}
...
```
#### 注意事项
* `udelay()`只用在小延迟中，因为在快速机器上可能导致溢出。超过1ms不要用`udelay()`。
* 除非绝对必要，这些函数最好不要用。
* 持锁忙等或禁止中断是一种粗鲁的做法，影响系统响应时间和性能。
* 忙等函数主要用在延迟小的地方，通常在微秒范围内。

# schedule_timeout()
* `schedule_timeout()`：让需要延迟执行的任务睡眠到指定的时间耗尽后再重新运行。
* 当指定时间到期后，内核唤醒被延迟的任务并将其重新放回运行队列。
* 在调用`schedule_timeout()`前需将进程状态设置为`TASK_UNINTERRUPTIBLE`或`TASK_INTERRUPTIBLE`，否则任务不会睡眠。
* 注意：该函数会引起睡眠，须处于进程上下文，且不能持有锁。

* kernel/time/timer.c
```c
static void process_timeout(unsigned long data)
{
    wake_up_process((struct task_struct * )data);
}

/**
 * schedule_timeout - sleep until timeout
 * @timeout: timeout value in jiffies
 *
 * Make the current task sleep until @timeout jiffies have
 * elapsed. The routine will return immediately unless
 * the current task state has been set (see set_current_state()).
 *
 * You can set the task state as follows -
 *
 * %TASK_UNINTERRUPTIBLE - at least @timeout jiffies are guaranteed to
 * pass before the routine returns. The routine will return 0
 * 保证在例程返回前至少经过了@timeout jiffies。
 * %TASK_INTERRUPTIBLE - the routine may return early if a signal is
 * delivered to the current task. In this case the remaining time
 * in jiffies will be returned, or 0 if the timer expired in time
 * 可能会早于约定的时间返回，比如说因为信号的缘故。剩余的jiffies会返回，如果超时则返回0。
 * The current task state is guaranteed to be TASK_RUNNING when this
 * routine returns.
 * 当例程返回时保证任务状态是TASK_RUNNING。
 * Specifying a @timeout value of %MAX_SCHEDULE_TIMEOUT will schedule
 * the CPU away without a bound on the timeout. In this case the return
 * value will be %MAX_SCHEDULE_TIMEOUT.
 * 超时值设为%MAX_SCHEDULE_TIMEOUT会立即调度。
 * In all cases the return value is guaranteed to be non-negative.
 * 所有情况下都保证返回值非负。
 */
signed long __sched schedule_timeout(signed long timeout)
{
    struct timer_list timer;
    unsigned long expire;
    /*这里先要处理两种异常情况*/
    switch (timeout)
    {
    case MAX_SCHEDULE_TIMEOUT: /*#define MAX_SCHEDULE_TIMEOUT    LONG_MAX*/
        /*
         * These two special cases are useful to be comfortable
         * in the caller. Nothing more. We could take
         * MAX_SCHEDULE_TIMEOUT from one of the negative value
         * but I' d like to return a valid offset (>=0) to allow
         * the caller to do everything it want with the retval.
         */
        schedule();
        goto out;
    default:
        /*
         * Another bit of PARANOID. Note that the retval will be
         * 0 since no piece of kernel is supposed to do a check
         * for a negative retval of schedule_timeout() (since it
         * should never happens anyway). You just have the printk()
         * that will tell you if something is gone wrong and where.
         */
        if (timeout < 0) {
          printk(KERN_ERR "schedule_timeout: wrong timeout "
              "value %lx\n", timeout);
          dump_stack();
          current->state = TASK_RUNNING;
          goto out;
      }
  }
  /*当前jiffies加上期望的超时经过的jiffies数为超时发生时的jiffies值。*/
  expire = timeout + jiffies;
  /*设置定时器处理函数为process_timeout(current)，意即，将来唤醒的进程是自己*/
  setup_timer_on_stack(&timer, process_timeout, (unsigned long)current);
  __mod_timer(&timer, expire, false, TIMER_NOT_PINNED); /*激活定时器*/
  schedule(); /*发起调度，暂时离开例程*/
  del_singleshot_timer_sync(&timer); /*再次调度回来后，删除定时器。TODO：自动删除？*/

  /* Remove the timer from the object tracker */
  /*当CONFIG_DEBUG_OBJECTS_TIMERS配置项打开的时候，会由额外的object产生，这里需要销毁。*/
  destroy_timer_on_stack(&timer);

  timeout = expire - jiffies; /*如果是提前唤醒，需要返回剩余的jiffies*/

out:
  return timeout < 0 ? 0 : timeout;
}
EXPORT_SYMBOL(schedule_timeout);

/*以下是相关的辅助函数*/
/*
 * We can use __set_current_state() here because schedule_timeout() calls
 * schedule() unconditionally.
 */
signed long __sched schedule_timeout_interruptible(signed long timeout)
{
    __set_current_state(TASK_INTERRUPTIBLE);
    return schedule_timeout(timeout);
}
EXPORT_SYMBOL(schedule_timeout_interruptible);

signed long __sched schedule_timeout_killable(signed long timeout)
{
    __set_current_state(TASK_KILLABLE);
    return schedule_timeout(timeout);
}
EXPORT_SYMBOL(schedule_timeout_killable);

signed long __sched schedule_timeout_uninterruptible(signed long timeout)
{
    __set_current_state(TASK_UNINTERRUPTIBLE);
    return schedule_timeout(timeout);
}
EXPORT_SYMBOL(schedule_timeout_uninterruptible);

/*
 * Like schedule_timeout_uninterruptible(), except this task will not contribute
 * to load average.
 */
signed long __sched schedule_timeout_idle(signed long timeout)
{
    __set_current_state(TASK_IDLE);
    return schedule_timeout(timeout);
}
EXPORT_SYMBOL(schedule_timeout_idle);
...
```

# Profiling
* 时钟中断处理程序调用profile驱动程序的中断处理函数，根据pc（程序计数器）所在函数的范围，使对应的内部计数器加1。
* 没有计入花在执行时钟中断处理程序和屏蔽时钟级中断的代码上的时间。

# Debug
## /proc/timer_list
* 源码 `kernel/time/timer_list.c`
* active timers 的格式

timer 索引 | timer 地址 | timer 到期时执行的回调函数名 | S:timer 当前状态
---|---|---|---
4 | `<c0923c98>` | sched_rt_period_timer | S:01

expires at | 到期的软时刻 | - | 到期的时刻 | nsecs | [in | 到期的软时间 | to | 到期的时间 | nsecs]
---|---|---|---|---|---|---|---|---|---
expires at | 69210000000000 | - | 69210000000000 | nsecs | [in | 624168272 | to | 624168272 | nsecs]

### `/proc/timer_list`例子
```# cat /proc/timer_list
Timer List Version: v0.6
HRTIMER_MAX_CLOCK_BASES: 2
now at 69209375831728 nsecs

cpu: 0
 clock 0:
  .base:       c2a5daf8
  .index:      0
  .resolution: 1 nsecs
  .get_time:   ktime_get_real
  .offset:     1509435077983117381 nsecs
active timers:
 clock 1:
  .base:       c2a5db28
  .index:      1
  .resolution: 1 nsecs
  .get_time:   ktime_get
  .offset:     0 nsecs
active timers:
#0: <c2a5db90>, tick_sched_timer, S:01
# expires at 69209376000000-69209376000000 nsecs [in 168272 to 168272 nsecs]
#1: <eac95aa8>, hrtimer_wakeup, S:01
# expires at 69209549034720-69209549284718 nsecs [in 173202992 to 173452990 nsecs]
#2: <eb055a08>, hrtimer_wakeup, S:01
# expires at 69209905092186-69209905697621 nsecs [in 529260458 to 529865893 nsecs]
#3: <eb2f1e88>, hrtimer_wakeup, S:01
# expires at 69209961466124-69209961516124 nsecs [in 585634396 to 585684396 nsecs]
#4: <c0923c98>, sched_rt_period_timer, S:01
# expires at 69210000000000-69210000000000 nsecs [in 624168272 to 624168272 nsecs]
#5: <eb2897c0>, sched_rt_period_timer, S:01
# expires at 69210000000000-69210000000000 nsecs [in 624168272 to 624168272 nsecs]
...
.expires_next   : 69209376000000 nsecs
.hres_active    : 1
.nr_events      : 17692832
.nr_retries     : 168
.nr_hangs       : 0
.max_hang_time  : 0 nsecs
.nohz_mode      : 0
.idle_tick      : 0 nsecs
.tick_stopped   : 0
.idle_jiffies   : 0
.idle_calls     : 0
.idle_sleeps    : 0
.idle_entrytime : 0 nsecs
.idle_waketime  : 0 nsecs
.idle_exittime  : 0 nsecs
.idle_sleeptime : 0 nsecs
.last_jiffies   : 0
.next_jiffies   : 0
.idle_expires   : 0 nsecs
jiffies: 17227343


Tick Device: mode:     1
Per CPU device: 0
Clock Event Device: decrementer
max_delta_ns:   42950102504
min_delta_ns:   1000
mult:           214746217
shift:          32
mode:           3
next_event:     69209376000000 nsecs
set_next_event: decrementer_set_next_event
set_mode:       decrementer_set_mode
event_handler:  hrtimer_interrupt
retries:        0
```

## /proc/timer_stats
* 内核选项 `CONFIG_TIMER_STATS`
* 源码 `kernel/time/timer_stats.c`
* 格式

timer 调用的次数 | 启动 timer 的 PID | 启动 timer 的进程名 | 启动 timer 的函数名 | (timer到期时执行的回调函数名)
---|---|---|---|---
223|27|watchdog/1|start_bandwidth_timer|(sched_rt_period_timer)

### `/proc/timer_stats`例子
```
# echo 1 > /proc/timer_stats
# cat /proc/timer_stats
Timer Stats Version: v0.3
Sample period: 347.695 s
Collection: active
 2844,     0 swapper/7        hrtimer_start_range_ns (ehci_hrtimer_func)
 19379,  3037 chromium-browse  schedule_hrtimeout_range_clock (hrtimer_wakeup)
    1,     0 swapper/4        hrtimer_start_range_ns (tick_sched_timer)
 8060,     0 swapper/2        hrtimer_start_range_ns (tick_sched_timer)
 4375,     0 swapper/5        hrtimer_start_range_ns (tick_sched_timer)
 9519,     7 rcu_sched        rcu_gp_kthread (process_timeout)
 6925,  3475 chromium-browse  hrtimer_start_range_ns (hrtimer_wakeup)
 20862,  3156 SoftwareVsyncTh  hrtimer_start_range_ns (hrtimer_wakeup)
 7652,     0 swapper/3        hrtimer_start_range_ns (tick_sched_timer)
3176D,    99 kworker/5:1      mod_delayed_work_on (delayed_work_timer_fn)
  223,    27 watchdog/1       start_bandwidth_timer (sched_rt_period_timer)
...
254916 total events, 733.159 events/sec

```

# References
- [Linux时间子系统之一：clock source（时钟源）](http://blog.csdn.net/droidphone/article/details/7975694)
- [Linux时间子系统之二：表示时间的单位和结构](http://blog.csdn.net/DroidPhone/article/details/7979295)
- [Linux时间子系统之三：时间的维护者：timekeeper](http://blog.csdn.net/droidphone/article/details/7989566)
- [Linux时间子系统之四：定时器的引擎：clock_event_device](http://blog.csdn.net/droidphone/article/details/8017604)
- [Linux时间子系统之五：低分辨率定时器的原理和实现](http://blog.csdn.net/droidphone/article/details/8051405)
- [Linux时间子系统之六：高精度定时器（HRTIMER）的原理和实现](http://blog.csdn.net/droidphone/article/details/8074892)
- [Linux时间子系统之七：定时器的应用--msleep()，hrtimer_nanosleep()](http://blog.csdn.net/DroidPhone/article/details/8104433)
- [Linux时间子系统之八：动态时钟框架（CONFIG_NO_HZ、tickless）](http://blog.csdn.net/DroidPhone/article/details/8112948)
