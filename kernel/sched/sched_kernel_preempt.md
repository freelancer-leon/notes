# 内核抢占
* 内核抢占会发生：
  * 中断处理程序正在执行，且返回内核空间之前。
  * 内核代码再一次具有可抢占性的时候（例如，调用`preempt_enable()`的时候）。
  * 如果内核中的任务显示调用`schedule()`。
  * 如果内核中的任务阻塞（这同样也会导致调用`schedule()`）。

* 是否能被抢占？
  * include/linux/preempt.h
```c
...
#ifdef CONFIG_PREEMPT_COUNT

#define preempt_disable() \
do { \
    preempt_count_inc(); \
    barrier(); \
} while (0)
...
#define preemptible()   (preempt_count() == 0 && !irqs_disabled())

#ifdef CONFIG_PREEMPT
#define preempt_enable() \
do { \
    barrier(); \
    if (unlikely(preempt_count_dec_and_test())) \
        __preempt_schedule(); \
} while (0)

...
#else /* !CONFIG_PREEMPT */
#define preempt_enable() \
do { \
    barrier(); \
    preempt_count_dec(); \
} while (0)

...
#endif /* CONFIG_PREEMPT */
...
#else /* !CONFIG_PREEMPT_COUNT */
/*
 * Even if we don't have any preemption, we need preempt disable/enable
 * to be barriers, so that we don't have things like get_user/put_user
 * that can cause faults and scheduling migrate into our preempt-protected
 * region.
 */
#define preempt_disable()           barrier()
...
#define preempt_enable()            barrier()
...
#define preemptible()               0

#endif /* CONFIG_PREEMPT_COUNT */
```
* `___preempt_schedule()`的实现x86平台进行了重载，其他平台则采用通用的宏替换成`preempt_schedule()`。
* 内核抢占前会先通过`preemptible()`判断能否进行抢占。
* kernel/sched/core.c
```c
/*
 * this is the entry point to schedule() from in-kernel preemption
 * off of preempt_enable. Kernel preemptions off return from interrupt
 * occur there and call schedule directly.
 */
asmlinkage __visible void __sched notrace preempt_schedule(void)
{
    /*
     * If there is a non-zero preempt_count or interrupts are disabled,
     * we do not want to preempt the current task. Just return..
     */
    if (likely(!preemptible()))
        return;

    preempt_schedule_common();
}
NOKPROBE_SYMBOL(preempt_schedule);
EXPORT_SYMBOL(preempt_schedule);
```

# 抢占计数preempt_count

* `preempt_count_add()`和`preempt_count_sub()`调用可重载的`__preempt_count_add()`和`__preempt_count_sub()`
* 目前就x86重载`__preempt_count_add()`和`__preempt_count_sub()`的实现
* 非Debug版本的`preempt_count_add()`和`preempt_count_sub()`
  * include/linux/preempt.h
```c
#if defined(CONFIG_DEBUG_PREEMPT) || defined(CONFIG_PREEMPT_TRACER)
extern void preempt_count_add(int val);
extern void preempt_count_sub(int val);
...
#else
#define preempt_count_add(val)  __preempt_count_add(val)
#define preempt_count_sub(val)  __preempt_count_sub(val)
...
#endif
```

* Debug版本的`preempt_count_add()`和`preempt_count_sub()`
  * kernel/sched/core.c
```c
#if defined(CONFIG_PREEMPT) && (defined(CONFIG_DEBUG_PREEMPT) || \
                defined(CONFIG_PREEMPT_TRACER))

void preempt_count_add(int val)
{
#ifdef CONFIG_DEBUG_PREEMPT
    /*
     * Underflow?
     */
    if (DEBUG_LOCKS_WARN_ON((preempt_count() < 0)))
        return;
#endif
    __preempt_count_add(val);
#ifdef CONFIG_DEBUG_PREEMPT
    /*
     * Spinlock count overflowing soon?
     */
    DEBUG_LOCKS_WARN_ON((preempt_count() & PREEMPT_MASK) >=
                PREEMPT_MASK - 10);
#endif
    if (preempt_count() == val) {
        unsigned long ip = get_lock_parent_ip();
#ifdef CONFIG_DEBUG_PREEMPT
        current->preempt_disable_ip = ip;
#endif
        trace_preempt_off(CALLER_ADDR0, ip);
    }
}   
EXPORT_SYMBOL(preempt_count_add);
NOKPROBE_SYMBOL(preempt_count_add);

void preempt_count_sub(int val)
{   
#ifdef CONFIG_DEBUG_PREEMPT
    /*
     * Underflow?
     */
    if (DEBUG_LOCKS_WARN_ON(val > preempt_count()))
        return;
    /*
     * Is the spinlock portion underflowing?
     */
    if (DEBUG_LOCKS_WARN_ON((val < PREEMPT_MASK) &&
            !(preempt_count() & PREEMPT_MASK)))
        return;
#endif

    if (preempt_count() == val)
        trace_preempt_on(CALLER_ADDR0, get_lock_parent_ip());
    __preempt_count_sub(val);
}
EXPORT_SYMBOL(preempt_count_sub);
NOKPROBE_SYMBOL(preempt_count_sub);
...
```

## x86的preempt_count
* x86的`thread_info`结构里并没有`preempt_count`成员，而是通过Per-CPU变量`__preempt_count`存储的。
* arch/x86/include/asm/thread_info.h
```c
struct thread_info {
    struct task_struct  *task;      /* main task structure */
    __u32           flags;      /* low level flags */
    __u32           status;     /* thread synchronous flags */
    __u32           cpu;        /* current CPU */
    mm_segment_t        addr_limit;
    unsigned int        sig_on_uaccess_error:1;
    unsigned int        uaccess_err:1;  /* uaccess failed */
};
```

* x86重载的`preempt_count`相关的实现
* arch/x86/include/asm/preempt.h
```c
DECLARE_PER_CPU(int, __preempt_count);
...
/*
 * We mask the PREEMPT_NEED_RESCHED bit so as not to confuse all current users
 * that think a non-zero value indicates we cannot preempt.
 */
static __always_inline int preempt_count(void)
{   /*返回抢占计数*/
    return raw_cpu_read_4(__preempt_count) & ~PREEMPT_NEED_RESCHED;
}

static __always_inline void preempt_count_set(int pc)
{   /*设置抢占计数*/
    raw_cpu_write_4(__preempt_count, pc);
}
...
/*
 * The various preempt_count add/sub methods
 */

static __always_inline void __preempt_count_add(int val)
{
    raw_cpu_add_4(__preempt_count, val);
}

static __always_inline void __preempt_count_sub(int val)
{
    raw_cpu_add_4(__preempt_count, -val);
}
...
```

## ARM的preempt_count

* arch/arm/include/asm/thread_info.h
```c
/*
 * low level task data that entry.S needs immediate access to.
 * __switch_to() assumes cpu_context follows immediately after cpu_domain.
 */
struct thread_info {
    unsigned long       flags;      /* low level flags */
    int         preempt_count;  /* 0 => preemptable, <0 => bug */
    mm_segment_t        addr_limit; /* address limit */
    struct task_struct  *task;      /* main task structure */
    __u32           cpu;        /* cpu */
    __u32           cpu_domain; /* cpu domain */
    struct cpu_context_save cpu_context;    /* cpu context */
    __u32           syscall;    /* syscall number */
    __u8            used_cp[16];    /* thread used copro */
    unsigned long       tp_value[2];    /* TLS registers */
#ifdef CONFIG_CRUNCH
    struct crunch_state crunchstate;
#endif
    union fp_state      fpstate __attribute__((aligned(8)));
    union vfp_state     vfpstate;
#ifdef CONFIG_ARM_THUMBEE
    unsigned long       thumbee_state;  /* ThumbEE Handler Base register */
#endif
};
...
/*
 * how to get the current stack pointer in C
 */
register unsigned long current_stack_pointer asm ("sp");

/*
 * how to get the thread information struct from C
 */
static inline struct thread_info *current_thread_info(void) __attribute_const__;

static inline struct thread_info *current_thread_info(void)
{
    return (struct thread_info *)
        (current_stack_pointer & ~(THREAD_SIZE - 1));
}
...
```

* ARM用的是通用的`__preempt_count_add()`和`__preempt_count_sub()`实现
* include/asm-generic/preempt.h
```c
static __always_inline int preempt_count(void)
{   /*返回抢占计数*/
    return current_thread_info()->preempt_count;
}

static __always_inline int *preempt_count_ptr(void)
{
    return &current_thread_info()->preempt_count;
}

static __always_inline void preempt_count_set(int pc)
{   /*设置抢占计数*/
    *preempt_count_ptr() = pc;
}
...
/*
 * The various preempt_count add/sub methods
 */

static __always_inline void __preempt_count_add(int val)
{
    *preempt_count_ptr() += val;
}

static __always_inline void __preempt_count_sub(int val)
{
    *preempt_count_ptr() -= val;
}
```

* `preempt_count_ptr()`仅在以下函数中被调用，x86`thread_info`没有`preempt_count`成员，因此以下函数也是独立实现的：
  * `preempt_count_set()`
  * `__preempt_count_add()`
  * `__preempt_count_sub()`
  * `__preempt_count_dec_and_test()`
