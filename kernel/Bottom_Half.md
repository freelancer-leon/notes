# 目录
- [ Bottom Half的类型 ](#Bottom%5fHalf的类型)
- [Softirq](#Softirq)
- [tasklet](#tasklet)
- [工作队列（work queue）](#工作队列（work%5fqueue）)
- [下半部之间加锁](#下半部之间加锁)
- [禁止下半部](#禁止下半部)

# Bottom Half的类型

- **BH**：每个BH都在全局范围内进行同步。即使分属于不同的处理器，也不允许任何两个bottom half同时执行。
- **任务队列**：定义一组队列，其中每个队列都包含一个由等待调用的函数组成的链表。
---
- **softirqs**：一组**静态**定义的下半部接口，32个，可以在所有处理器上同时执行 -- 即使两个类型相同也可以。
- **tasklet**：基于softirq实现的灵活性强，**动态** 创建的下半部实现机制。不同类型的tasklet可以在不同的处理器上同时执行，但类型相同的tasklet不能同时执行。
---
- **工作队列** 是将工作推后执行的一种方式。
- **内核定时器** 也可以将工作推后执行。

# Softirq

```c
/* PLEASE, avoid to allocate new softirqs, if you need not _really_ high
   frequency threaded job scheduling. For almost all the purposes
   tasklets are more than enough. F.e. all serial device BHs et
   al. should be converted to tasklets, not to softirqs.
 */

enum
{
    HI_SOFTIRQ=0,
    TIMER_SOFTIRQ,
    NET_TX_SOFTIRQ,
    NET_RX_SOFTIRQ,
    BLOCK_SOFTIRQ,
    IRQ_POLL_SOFTIRQ,
    TASKLET_SOFTIRQ,
    SCHED_SOFTIRQ,
    HRTIMER_SOFTIRQ, /* Unused, but kept as tools rely on the
                numbering. Sigh! */
    RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

    NR_SOFTIRQS
};

static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;

DEFINE_PER_CPU(struct task_struct *, ksoftirqd);

const char * const softirq_to_name[NR_SOFTIRQS] = {
    "HI", "TIMER", "NET_TX", "NET_RX", "BLOCK", "BLOCK_IOPOLL",
    "TASKLET", "SCHED", "HRTIMER", "RCU"
};

struct softirq_action
{
    void    (*action)(struct softirq_action *);
};
```
* 一个软中断不会抢占另外一个软中断
* 实际上，唯一可以抢占软中断的是中断处理程序
* 其他软中断（包括同类型的软中断）可以在其他处理器上同时执行
* **软中断和tasklet都是运行在中断上下文中**，它们与任一进程无关，没有支持的进程完成重新调度。所以软中断和tasklet不能睡眠、不能阻塞，它们的代码中不能含有导致睡眠的动作，如减少信号量、从用户空间拷贝数据或手工分配内存等。
* 也正是由于它们运行在中断上下文中，所以它们在同一个CPU上的执行是串行的，这样就不利于实时多媒体任务的优先处理。


## 执行软中断

* **触发软中断**（raising the softirq）一个注册的软中断必须在被标记后才会被执行。
* 通常中断处理程序会在返回前标记它的软中断，使其在稍后被执行。
* 软中断被检查和执行的地方
  * 从一个硬件中断代码处返回时
  * 在ksoftirqd内核线程中
  * 在那些显式检查和执行待处理软中断的代码中，如网络子系统中。
* 不管用什么方法唤起，软中断都要在`do_softirq()`中执行。
  * `do_softirq()` --> `do_softirq_own_stack()` --> `__do_softirq()` --> `softirq_vec->action()`
  * kernel/softirq.c
```c
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
    unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
    unsigned long old_flags = current->flags;
    int max_restart = MAX_SOFTIRQ_RESTART;
    struct softirq_action *h;
    bool in_hardirq;
    __u32 pending;
    int softirq_bit;

    /*
     * Mask out PF_MEMALLOC s current task context is borrowed for the
     * softirq. A softirq handled such as network RX might set PF_MEMALLOC
     * again if the socket is related to swap
     */
    current->flags &= ~PF_MEMALLOC;

    pending = local_softirq_pending();
    account_irq_enter_time(current);

    __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
    in_hardirq = lockdep_softirq_start();

restart:
    /* Reset the pending bitmask before enabling irqs */
    set_softirq_pending(0);

    local_irq_enable();

    h = softirq_vec;

    while ((softirq_bit = ffs(pending))) {  /* ffs - find first set bit in word */
        unsigned int vec_nr;
        int prev_count;

        h += softirq_bit - 1;

        vec_nr = h - softirq_vec;
        prev_count = preempt_count();

        kstat_incr_softirqs_this_cpu(vec_nr);

        trace_softirq_entry(vec_nr);
        h->action(h);
        trace_softirq_exit(vec_nr);
        if (unlikely(prev_count != preempt_count())) {
            pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
                   vec_nr, softirq_to_name[vec_nr], h->action,
                   prev_count, preempt_count());
            preempt_count_set(prev_count);
        }
        h++;
        pending >>= softirq_bit;
    }

    rcu_bh_qs();
    local_irq_disable();

    pending = local_softirq_pending();
    if (pending) {
        if (time_before(jiffies, end) && !need_resched() &&
            --max_restart)
            goto restart;

        wakeup_softirqd();
    }

    lockdep_softirq_end(in_hardirq);
    account_irq_exit_time(current);
    __local_bh_enable(SOFTIRQ_OFFSET);
    WARN_ON_ONCE(in_interrupt());
    tsk_restore_flags(current, old_flags, PF_MEMALLOC);
}

asmlinkage __visible void do_softirq(void)
{
    __u32 pending;
    unsigned long flags;
    /*如果别处已经禁止硬中断或软中断（包括local_bh_disable）则此处直接返回*/
    if (in_interrupt())  /*软中断不能进入到硬中断部分*/
        return;

    local_irq_save(flags);

    pending = local_softirq_pending();

    if (pending)
        do_softirq_own_stack();

    local_irq_restore(flags);
}
```
* arch/x86/kernel/irq_32.c
```c
void do_softirq_own_stack(void)
{   
    struct thread_info *curstk;
    struct irq_stack *irqstk;
    u32 *isp, *prev_esp;

    curstk = current_stack();
    irqstk = __this_cpu_read(softirq_stack);

    /* build the stack frame on the softirq stack */
    isp = (u32 *) ((char *)irqstk + sizeof(*irqstk));

    /* Push the previous esp onto the stack */
    prev_esp = (u32 *)irqstk;
    *prev_esp = current_stack_pointer();

    call_on_stack(__do_softirq, isp);
}
```

## 使用软中断

### 注册softirq

* kernel/softirq.c
```c
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
    softirq_vec[nr].action = action;
}
```
### 注册的例子
* kernel/sched/fair.c
```c
__init void init_sched_fair_class(void)
{
#ifdef CONFIG_SMP
    open_softirq(SCHED_SOFTIRQ, run_rebalance_domains);
...
#endif /* SMP */
}
```
* kernel/softirq.c
```c
void __init softirq_init(void)
{
    int cpu;

    for_each_possible_cpu(cpu) {
        per_cpu(tasklet_vec, cpu).tail =
            &per_cpu(tasklet_vec, cpu).head;
        per_cpu(tasklet_hi_vec, cpu).tail =
            &per_cpu(tasklet_hi_vec, cpu).head;
    }
    /*注意，这里注册了tasklet的softirq处理函数*/
    open_softirq(TASKLET_SOFTIRQ, tasklet_action);
    open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```

* 软中断处理程序执行时允许响应中断（即开中断），但不能睡眠
* 在一个处理程序在运行时，当前处理器的软中断被禁止，但其他处理器并不禁止软中断
* 因此其他处理器可以运行软中断处理程序，甚至是同一处理程序
* 这意味着共享数据要加锁，但加锁又影响性能，失去软中断的意义，因此尽量用单处理器数据或其他技巧避免加锁
* 还有就是改用tasklet

### 触发softirq

* kernel/softirq.c
```c
/*
 * This function must run with irqs disabled!
 */
inline void raise_softirq_irqoff(unsigned int nr)
{
    __raise_softirq_irqoff(nr);

    /*
     * If we're in an interrupt or softirq, we're done
     * (this also catches softirq-disabled code). We will
     * actually run the softirq once we return from
     * the irq or softirq.
     *
     * Otherwise we wake up ksoftirqd to make sure we
     * schedule the softirq soon.
     */
    if (!in_interrupt())
        wakeup_softirqd(); /*唤醒软中断处理程序*/
}

void raise_softirq(unsigned int nr)
{
    unsigned long flags;

    local_irq_save(flags);
    raise_softirq_irqoff(nr);
    local_irq_restore(flags);
}       

void __raise_softirq_irqoff(unsigned int nr)
{
    trace_softirq_raise(nr);
    or_softirq_pending(1UL << nr);  /*置位*/
}
```

* 如果知道中断已经被禁，可以直接调`raise_softirq_irqoff()`。


# tasklet

* 用softirq实现，因此本质上也是softirq
* 分为`HI_SOFTIRQ`和`TASKLET_SOFTIRQ`两类，`HI_SOFTIRQ`优先于`TASKLET_SOFTIRQ`执行
* `count`成员是某个tasklet的引用计数，是关于这个tasklet的开关
* `state`的`TASKLET_STATE_RUN`则是用来检查当前tasklet是否已经在某个处理器上执行
* `tasklet_struct`结构
```c
struct tasklet_struct
{
    struct tasklet_struct *next;
    unsigned long state;
    atomic_t count;  /*不为0，被禁止；为0，激活，被设为挂起状态才能执行*/
    void (*func)(unsigned long);
    unsigned long data;
};
```
* tasklet state
```c
enum
{
    TASKLET_STATE_SCHED,    /* Tasklet is scheduled for execution */
    TASKLET_STATE_RUN   /* Tasklet is running (SMP only) */
};
```
* 两个per-CPU链表`tasklet_vec`和`tasklet_hi_vec`
```c
/*
 * Tasklets
 */
struct tasklet_head {
    struct tasklet_struct *head;
    struct tasklet_struct **tail;
};

static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);

```

## 调度tasklet
* `tasklet_schedule(t)`和`tasklet_hi_schedule(t)`
  * 把`tasklet_struct`加入per-CPU链表
  * 触发softirq
  * include/linux/interrupt.h
```c
extern void __tasklet_schedule(struct tasklet_struct *t);

static inline void tasklet_schedule(struct tasklet_struct *t)
{
    if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))  /*状态置位*/
        __tasklet_schedule(t);
}
```
  * kernel/softirq.c
```c
void __tasklet_schedule(struct tasklet_struct *t)
{
    unsigned long flags;

    local_irq_save(flags);
    t->next = NULL;
    *__this_cpu_read(tasklet_vec.tail) = t;  /*加入链表*/
    __this_cpu_write(tasklet_vec.tail, &(t->next));
    raise_softirq_irqoff(TASKLET_SOFTIRQ);  /*触发softirq*/
    local_irq_restore(flags);
}
EXPORT_SYMBOL(__tasklet_schedule);
```

## 处理tasklet
* `tasklet_action()`和`tasklet_hi_action()`，tasklet的softirq处理函数，在`softirq_init()`时注册
* 需要通过测试并设置`TASKLET_STATE_RUN`来保障同一类型的tasklet不能同时在其他CPU上运行
  * include/linux/interrupt.h
```c
#ifdef CONFIG_SMP
static inline int tasklet_trylock(struct tasklet_struct *t)
{
    return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
}

static inline void tasklet_unlock(struct tasklet_struct *t)
{
    smp_mb__before_atomic();
    clear_bit(TASKLET_STATE_RUN, &(t)->state);
}

static inline void tasklet_unlock_wait(struct tasklet_struct *t)
{
    while (test_bit(TASKLET_STATE_RUN, &(t)->state)) { barrier(); }
}
#else
#define tasklet_trylock(t) 1
#define tasklet_unlock_wait(t) do { } while (0)
#define tasklet_unlock(t) do { } while (0)
#endif
```
  * kernel/softirq.c
```c
static void tasklet_action(struct softirq_action *a)
{
    struct tasklet_struct *list;
    /*先记录下表头，然后清空链表*/
    local_irq_disable();
    list = __this_cpu_read(tasklet_vec.head);
    __this_cpu_write(tasklet_vec.head, NULL);
    __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
    local_irq_enable();

    while (list) {
        struct tasklet_struct *t = list;

        list = list->next;
        /*先看看该类型tasklet是否在其他CPU上执行。
          如果没有，则置位RUN，开始执行；
          否则跳过该tasklet
         */
        if (tasklet_trylock(t)) {
            if (!atomic_read(&t->count)) { /*检查tasklet是否被禁止*/
                if (!test_and_clear_bit(TASKLET_STATE_SCHED,
                            &t->state))
                    BUG();
                t->func(t->data);  /*执行tasklet*/
                tasklet_unlock(t); /*清除RUN标志*/
                continue;          /*接着处理下一个tasklet*/
            }
            tasklet_unlock(t);
        }
        /*跳过的tasklet放回链表，并激活softirq让它下回再处理*/
        local_irq_disable();
        t->next = NULL;
        *__this_cpu_read(tasklet_vec.tail) = t;
        __this_cpu_write(tasklet_vec.tail, &(t->next));
        __raise_softirq_irqoff(TASKLET_SOFTIRQ);
        local_irq_enable();
    }
}
```

## 使用tasklet

### 声明tasklet

* 静态创建tasklet
```c
#define DECLARE_TASKLET(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }

#define DECLARE_TASKLET_DISABLED(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(1), func, data }
```

* 动态创建tasklet
```c
void tasklet_init(struct tasklet_struct *t,                                                                                
          void (*func)(unsigned long), unsigned long data)
{                                                                                         
    t->next = NULL;
    t->state = 0;
    atomic_set(&t->count, 0);
    t->func = func;
    t->data = data;
}
EXPORT_SYMBOL(tasklet_init);
```

### tasklet处理函数
* 因为是靠softirq实现的，所以tasklet处理函数不能睡
  * 不能用信号量或阻塞式函数
* 注意，此期间允许响应中断
* 注意保护不同tasklet间的共享数据，或者tasklet与其他softirq共享的数据


### tasklet函数
* `tasklet_schedule()`
* `tasklet_disable()`和`tasklet_disable_nosync()`
* `tasklet_enable()`
* `tasklet_kill()`从链表中移出某个tasklet
  * 会等待正在执行的tasklet执行完毕
  * 注意，其他地方仍有可能将tasklet再度放回链表
  * 可能会引起休眠，**禁止在中断上下文使用**

## ksoftirqd
* 每个CPU一个的辅助处理softirq（和tasklet）的内核线程
* 引入ksoftirqd的原因：
  * 在中断处理函数返回时处理softirq是最常见的softirq处理时机
  * softirq触发的频率有时很高，而有的softirq还会重新触发自己以便得到再次执行
  * 为防止用户空间进程饥饿，作为折中的方案，内核不会立即处理重新触发的softirq
  * 当大量softirq出现时，内核会唤醒一组内核线程来处理这些负载，即**ksoftirqd**
* ksoftirqd每次迭代都会最终调用`schedule()`让其他进程有机会得到处理
* kernel/softirq.c
```c
static int ksoftirqd_should_run(unsigned int cpu)
{
    return local_softirq_pending();
}

static void run_ksoftirqd(unsigned int cpu)
{
    local_irq_disable();
    if (local_softirq_pending()) {
        /*
         * We can safely run softirq on inline stack, as we are not deep
         * in the task stack here.
         */
        __do_softirq();
        local_irq_enable();
        cond_resched_rcu_qs();
        return;
    }
    local_irq_enable();
}
...

static struct smp_hotplug_thread softirq_threads = {
    .store          = &ksoftirqd,
    .thread_should_run  = ksoftirqd_should_run,
    .thread_fn      = run_ksoftirqd,
    .thread_comm        = "ksoftirqd/%u",
};

static __init int spawn_ksoftirqd(void)
{
    register_cpu_notifier(&cpu_nfb);

    BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

    return 0;
}
early_initcall(spawn_ksoftirqd);
```


# 工作队列（work queue)

* 工作队列是将工作推后执行的一种方式，可以用来实现中断下半部
* 工作队列在**进程上下文**，允许在工作中重新调度或睡眠。
  * 需要大量内存
  * 获取信号量
  * 阻塞式I/O

## 工作队列的实现
* **工作者线程**（worker thread）是内核线程，由工作队列子系统提供的接口创建，负责执行由内核其他部分排到队列里的任务。
* 驱动程序可以创建一个专门的worker thread来处理要推后的工作。
* 工作队列子系统也提供缺省的worker thread，每个处理器对应一个。
* 创建属于自己的worker thread的情况：
  * 要在worker thread执行大量的处理操作
  * CPU密集型和性能要求严格的任务。可减轻缺省worker thread的负担，避免饥饿

### worker thread结构体
* 每种类型的work thread都对应到一个`workqueue_struct`结构体。
* `workqueue_struct`结构体包含一个Per CPU的`cpu_workqueue_struct`结构体。
* 每种类型的work thread在每个CPU上有一个thread。
* 因此每个work thread可以用一个`cpu_workqueue_struct`。
* `cpu_workqueue_struct`结构的成员`worklist`为链表头，链表成员是`work_struct`类型的任务。

#### 新的结构体定义
```c
/*
 * The externally visible workqueue.  It relays the issued work items to
 * the appropriate worker_pool through its pool_workqueues.
 */
struct workqueue_struct {
    struct list_head    pwqs;       /* WR: all pwqs of this wq */
    struct list_head    list;       /* PR: list of all workqueues */
    ...
    struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
    ...
};

/*
 * The per-pool workqueue.  While queued, the lower WORK_STRUCT_FLAG_BITS
 * of work_struct->data are used for flags and the remaining high bits
 * point to the pwq; thus, pwqs need to be aligned at two's power of the
 * number of flag bits.
 */
struct pool_workqueue {
    struct worker_pool  *pool;      /* I: the associated pool */
    struct workqueue_struct *wq;        /* I: the owning workqueue */
    ...
    struct list_head    pwqs_node;  /* WR: node on wq->pwqs */
    ...
} __aligned(1 << WORK_STRUCT_FLAG_BITS);

struct worker_pool {
    spinlock_t      lock;       /* the pool lock */
    int         cpu;        /* I: the associated cpu */
    int         node;       /* I: the associated node ID */
    int         id;     /* I: pool ID */
    ...
    struct list_head    worklist;   /* L: list of pending works */
    int         nr_workers; /* L: total number of workers */
    ...
    /* see manage_workers() for details on the two manager mutexes */
    struct mutex        manager_arb;    /* manager arbitration */
    struct worker       *manager;   /* L: purely informational */
    struct mutex        attach_mutex;   /* attach/detach exclusion */
    struct list_head    workers;    /* A: attached workers */
    ...
    int         refcnt;     /* PL: refcnt for unbound pools */

    /*   
     * The current concurrency level.  As it's likely to be accessed
     * from other CPUs during try_to_wake_up(), put it in a separate
     * cacheline.
     */
    atomic_t        nr_running ____cacheline_aligned_in_smp;
    ...
} ____cacheline_aligned_in_smp;

struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func;
#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;
#endif
};

```

## 使用工作队列
1. 创建推后的工作
* 静态创建
   ```c
   DECLARE_WORK(name, void (*func)(void *), void *data);
   ```
* 动态创建
   ```c
   INIT_WORK(struct work_struct *work, void (*func)(void *), void *data);
   ```
2. 工作队列handler
   ```c
   void work_handler(void *data)
   ```
* 注意：工作队列处理函数不能访问用户空间。因为它是在内核线程中执行的，而内核线程在用户空间没有相关的内存映射。
  * Q：什么时候内核可以访问用户空间？
  * A： 通常发生系统调用时，内核会代表用户空间的进程运行，此时会映射用户空间的内存。
3. 对工作进行调度
   ```c
   schedule_work(&work);
   schedule_delayed_work(&work, delay);
   ```
4. 刷新工作队列
   ```c
   void flush_scheduled_work(void);
   ```
* 该函数会一直等待，直到队列中所有队列中所有对象都被执行完后才返回。
* 等待过程中该函数会休眠，所以只能在进程上下文中使用。
* 该函数不取消任何延迟执行的工作。
* 取消延迟工作应调用：
   ```c
   int cancel_delayed_work(struct work_struct *work);
   ```
5. 创建新的工作队列
* 创建
```c
struct workqueue_struct *create_workqueue(const char *name);
```
* 调度
```c
int queue_work(struct workqueue_struct *wq, struct work_struct *work);
int queue_delayed_work(struct workqueue_struct *wq,
      struct work_struct *work,
      unsigned long delay);
```
* 刷新
```c
void flush_workqueue(struct workqueue_struct *wq);
```

# 下半部之间加锁
* 在使用下半部机制时，即使是在单处理器系统下，避免共享数据被同时访问也是置关重要的。
* 一个下半部实际上可能在任何时候执行。
* 同类型tasklet间共享的数据无需考虑同步问题。
* 两个不同类型的tasklet共享同一数据需要加锁。
* 为了本地和SMP的保护并且防止死锁出现：
  * 如果**进程上下文**和一个**下半部**共享数据，在访问这些数据之前，需要**禁止下半部**且获得锁的使用权。
  * 如果**中断上下文**和一个**下半部**共享数据，在访问数据之前，需要**禁止中断**且获得锁的使用权。
* 任何在工作队列中被共享的数据需要使用锁机制。
* Q：为什么共享数据除了需要锁的保护，还需要禁止中断或禁止下半部？
  * T：难道是因为加锁时也有可能被中断打断？
  * A：禁止中断或禁止下半部的原因是，中断会抢占下半部的执行，下半部会打断进程的执行。
    *  假设进程持有锁，而打断它的下半部竞争的是同一把锁，则下半部会因为无法得到进程持有的锁而挂住，造成死锁。
    *  如果进程禁止了下半部再去拿锁，则不会被下半部打断，直到它再次开启下半部。
    *  中断与下半部之间共享数据同理。

# 禁止下半部
* 禁止下半部主要指禁止**softirq**和**tasklet**。
* 保护数据的常见的做法是**先拿到锁，再禁下半部**？？
  * 这个说法有问题，应该是**先禁下半部，再取竞争锁**，原因上面讨论了。
  * 事实上`spin_lock_bh()`最后的实现`__raw_spin_lock_bh()`也是先禁下半部，再取竞争锁。
* 相关函数为：
  ```c
  static inline void local_bh_disable(void);
  static inline void local_bh_enable(void);
  ```
* 前缀`local_`表明禁的是本地的softirq和tasklet。
* 可以嵌套，因此最后一次调用`local_bh_enable()`时下半部才被激活。
* 这些函数不能禁止工作队列的执行。
  * 因为工作队列运行在进程上下文，不会涉及异步执行的问题，所以没有必要禁止。
  * 对工作队列来说，它保护共享数据所做的工作和其任何进程上下文中所做的差不多。

## 禁止中断，软中断，抢占的内幕

* include/linux/preempt.h
```c
/*
 * We put the hardirq and softirq counter into the preemption
 * counter. The bitmask has the following meaning:
 *
 * - bits 0-7 are the preemption count (max preemption depth: 256)
 * - bits 8-15 are the softirq count (max # of softirqs: 256)
 *
 * The hardirq count could in theory be the same as the number of
 * interrupts in the system, but we run all interrupt handlers with
 * interrupts disabled, so we cannot have nesting interrupts. Though
 * there are a few palaeontologic drivers which reenable interrupts in
 * the handler, so we need more than one bit here.
 *
 *         PREEMPT_MASK:    0x000000ff
 *         SOFTIRQ_MASK:    0x0000ff00
 *         HARDIRQ_MASK:    0x000f0000
 *             NMI_MASK:    0x00100000
 * PREEMPT_NEED_RESCHED:    0x80000000
 */
#define PREEMPT_BITS    8
#define SOFTIRQ_BITS    8
#define HARDIRQ_BITS    4
#define NMI_BITS    1

#define PREEMPT_SHIFT   0
#define SOFTIRQ_SHIFT   (PREEMPT_SHIFT + PREEMPT_BITS) /*8*/
#define HARDIRQ_SHIFT   (SOFTIRQ_SHIFT + SOFTIRQ_BITS) /*16*/
#define NMI_SHIFT   (HARDIRQ_SHIFT + HARDIRQ_BITS)     /*20*/

#define __IRQ_MASK(x)   ((1UL << (x))-1)

#define PREEMPT_MASK    (__IRQ_MASK(PREEMPT_BITS) << PREEMPT_SHIFT)
#define SOFTIRQ_MASK    (__IRQ_MASK(SOFTIRQ_BITS) << SOFTIRQ_SHIFT)
#define HARDIRQ_MASK    (__IRQ_MASK(HARDIRQ_BITS) << HARDIRQ_SHIFT)
#define NMI_MASK    (__IRQ_MASK(NMI_BITS)     << NMI_SHIFT)

#define PREEMPT_OFFSET  (1UL << PREEMPT_SHIFT)
#define SOFTIRQ_OFFSET  (1UL << SOFTIRQ_SHIFT)        /*1<<8 = 256*/
#define HARDIRQ_OFFSET  (1UL << HARDIRQ_SHIFT)
#define NMI_OFFSET  (1UL << NMI_SHIFT)

#define SOFTIRQ_DISABLE_OFFSET  (2 * SOFTIRQ_OFFSET)  /*1<<9 = 512*/

/* We use the MSB mostly because its available */
#define PREEMPT_NEED_RESCHED    0x80000000

```
* 以下标志位共用`preempt_count`是因为前几种情形都不允许发生进程抢占：
  * 抢占计数
  * 软中断
  * 硬中断
  * NMI

* 判断是否可以抢占`preemptible()`宏
  * include/linux/preempt.h
  ```c
  #define preemptible()   (preempt_count() == 0 && !irqs_disabled())
  ```
* `TIF_NEED_RESCHED`是`need_resched()`检查的标志位，它存在每个进程的`thread_info`结构的`flags`里。
  * include/linux/sched.h
  ```c
  static __always_inline bool need_resched(void)
  {
      return unlikely(tif_need_resched());
  }
  ```
  * include/linux/thread_info.h
  ```c
  #define test_thread_flag(flag) \
      test_ti_thread_flag(current_thread_info(), flag)

  #define tif_need_resched() test_thread_flag(TIF_NEED_RESCHED)
  ```
  * arch/x86/include/asm/thread_info.h
  ```c
  /*
   * thread information flags
   * - these are process state flags that various assembly files
   *   may need to access
   * - pending work-to-be-done flags are in LSW
   * - other flags in MSW
   * Warning: layout of LSW is hardcoded in entry.S
   */
  #define TIF_SYSCALL_TRACE   0   /* syscall trace active */
  #define TIF_NOTIFY_RESUME   1   /* callback before returning to user */
  #define TIF_SIGPENDING      2   /* signal pending */
  #define TIF_NEED_RESCHED    3   /* rescheduling necessary */
  #define TIF_SINGLESTEP      4   /* reenable singlestep on user return*/
  #define TIF_SYSCALL_EMU     6   /* syscall emulation active */
  #define TIF_SYSCALL_AUDIT   7   /* syscall auditing active */
  #define TIF_SECCOMP     8   /* secure computing */
  #define TIF_USER_RETURN_NOTIFY  11  /* notify kernel of userspace return */
  #define TIF_UPROBE      12  /* breakpointed or singlestepping */
  #define TIF_NOTSC       16  /* TSC is not accessible in userland */
  #define TIF_IA32        17  /* IA32 compatibility process */
  #define TIF_FORK        18  /* ret_from_fork */
  #define TIF_NOHZ        19  /* in adaptive nohz mode */
  #define TIF_MEMDIE      20  /* is terminating due to OOM killer */
  #define TIF_POLLING_NRFLAG  21  /* idle is polling for TIF_NEED_RESCHED */
  #define TIF_IO_BITMAP       22  /* uses I/O bitmap */
  #define TIF_FORCED_TF       24  /* true if TF in eflags artificially */
  #define TIF_BLOCKSTEP       25  /* set when we want DEBUGCTLMSR_BTF */
  #define TIF_LAZY_MMU_UPDATES    27  /* task is updating the mmu lazily */
  #define TIF_SYSCALL_TRACEPOINT  28  /* syscall tracepoint instrumentation */
  #define TIF_ADDR32      29  /* 32-bit address space on 64 bits */
  #define TIF_X32         30  /* 32-bit native x86-64 binary */
  ...
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

* `PREEMPT_NEED_RESCHED`目前只有x86在用。

* 禁用下半部实际上是增加`SOFTIRQ_DISABLE_OFFSET`到`preempt_count`
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
...
static inline void local_bh_disable(void)
{
    __local_bh_disable_ip(_THIS_IP_, SOFTIRQ_DISABLE_OFFSET);
}

static inline void local_bh_enable_ip(unsigned long ip)
{
    __local_bh_enable_ip(ip, SOFTIRQ_DISABLE_OFFSET);
}
```
* 开启下半部的细节
  * kernel/softirq.c
```c
void __local_bh_enable_ip(unsigned long ip, unsigned int cnt)
{
    WARN_ON_ONCE(in_irq() || irqs_disabled()); /*在硬中断处理过程中开启下半部会引发警告*/
#ifdef CONFIG_TRACE_IRQFLAGS
    local_irq_disable();
#endif
    /*
     * Are softirqs going to be turned on now:
     */
    if (softirq_count() == SOFTIRQ_DISABLE_OFFSET)
        trace_softirqs_on(ip);  /*在嵌套调用的场景，只有最后一次enable才会被跟踪到*/
    /*
     * Keep preemption disabled until we are done with
     * softirq processing:
     */
    preempt_count_sub(cnt - 1); /*可以理解为减完SOFTIRQ_DISABLE_OFFSET后加1，保持禁止抢占*/

    if (unlikely(!in_interrupt() && local_softirq_pending())) {
        /*
         * Run softirq if any pending. And do it in its own stack
         * as we may be calling this deep in a task call stack already.
         */
        do_softirq();  /*如果不在任何中断上下文中且本地有softirq在等待执行，则先执行softirq*/
    }

    preempt_count_dec(); /*减去之前加上的1，允许抢占了*/
#ifdef CONFIG_TRACE_IRQFLAGS
    local_irq_enable();
#endif
    preempt_check_resched();  /*看看会否发生内核抢占*/
}
EXPORT_SYMBOL(__local_bh_enable_ip);
```
