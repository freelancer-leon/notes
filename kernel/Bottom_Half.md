# 目录
- [Bottom Half的类型](#Bottom%5fHalf的类型)
- [Softirq](#Softirq)
- [tasklet](#tasklet)
- [工作队列（work queue）](#工作队列（work%5fqueue）)
- [下半部之间加锁](#下半部之间加锁)
- [禁止下半部](#禁止下半部)

# Bottom Half的类型

- **BH**：每个BH都在全局范围内进行同步。即使分属于不同的处理器，也不允许任何两个bottom half同时执行。
- **任务队列**：定义一组队列，其中每个队列都包含一个由等待调用的函数组成的链表。

***

- **softirqs**：一组**静态**定义的下半部接口，32个，可以在所有处理器上同时执行 -- 即使两个类型相同也可以。
- **tasklet**：基于softirq实现的灵活性强，**动态** 创建的下半部实现机制。不同类型的tasklet可以在不同的处理器上同时执行，但类型相同的tasklet不能同时执行。

***

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
* 在一个核上，一个软中断不会抢占另外一个软中断
* 实际上，唯一可以抢占软中断的是中断处理程序
* 其他软中断（包括同类型的软中断）可以在其他处理器上同时执行
* **软中断和 tasklet 都是运行在中断上下文中**，它们与任一进程无关，没有支持的进程完成重新调度。所以软中断和 tasklet 不能睡眠、不能阻塞，它们的代码中不能含有导致睡眠的动作，如减少信号量、从用户空间拷贝数据或手工分配内存等。
* 也正是由于它们运行在中断上下文中，所以它们在同一个 CPU 上的执行是串行的，这样就不利于实时多媒体任务的优先处理。


## 执行软中断

* **触发软中断**（raising the softirq）一个注册的软中断必须在被标记后才会被执行。
* 通常中断处理程序会在返回前标记它的软中断，使其在稍后被执行。
* 软中断被检查和执行的地方
  * ~~在那些显式检查和执行待处理软中断的代码中，如网络子系统中（当`ksoftirqd`可运行时不会同步处理软中断）~~
  * 从一个硬件中断代码处返回时
  * 在`ksoftirqd`内核线程中
  * 在`ktimers/%u`内核线程中`run_ktimerd() -> __do_softirq()`
  * 禁止下半部后再次使能软中断的时候
* 不管用什么方法唤起，软中断都要在`handle_softirqs()`中执行。

### 禁止下半部后再次使能软中断
* kernel/softirq.c::`__local_bh_enable_ip()` -> `__do_softirq()`

### ~~直接调用`do_softirq()`处理软中断~~
* net/core/dev.c::`netif_rx_ni()` -> `do_softirq()`
* 该调用点已于 commit baebdf48c360 ("net: dev: Makes sure netif_rx() can be invoked in any context.") 被删除，并通过增加关闭和开启下半部，将对软中断的处理归并到上一种情况
  ```c
  do_softirq()
   -> do_softirq_own_stack()
      -> __do_softirq()
         -> handle_softirqs(false)
            -> softirq_vec->action()
  ```
* kernel/softirq.c
```cpp
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
    unsigned long end = jiffies + MAX_SOFTIRQ_TIME; /*默认为 2ms*/
    unsigned long old_flags = current->flags;
    int max_restart = MAX_SOFTIRQ_RESTART; /*默认为 10 次*/
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
    /*开启之前关闭的中断，如run_ksoftirqd()。因此软中断处理程序期间中断是打开的。*/
    local_irq_enable();
    //指向软中断向量数组的起始地址
    h = softirq_vec;
    //按软中断向量的优先级逐级处理挂起的软中断
    while ((softirq_bit = ffs(pending))) {  /* ffs - find first set bit in word */
        unsigned int vec_nr;
        int prev_count;
        //h 指向软中断向量中有软中断正在挂起的数组元素
        h += softirq_bit - 1;
        //vec_nr 为正在处理的软中断向量号
        vec_nr = h - softirq_vec;
        prev_count = preempt_count();

        kstat_incr_softirqs_this_cpu(vec_nr);
        //处理挂起的软中断
        trace_softirq_entry(vec_nr);
        h->action(h);
        trace_softirq_exit(vec_nr);
        if (unlikely(prev_count != preempt_count())) {
            pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
                   vec_nr, softirq_to_name[vec_nr], h->action,
                   prev_count, preempt_count());
            preempt_count_set(prev_count);
        }
        h++; //h 指向软中断向量中下一个正在挂起软中断的数组元素
        pending >>= softirq_bit;
    }

    rcu_bh_qs();
    local_irq_disable(); /*返回之前恢复中断的关闭状态。*/
    /*以上为在软中断上下文中调用软中断处理函数的部分，下面是考虑是否还要继续在软中断上下文中处理软中断*/
    pending = local_softirq_pending(); /*如果此时还有未处理完的软中断*/
    if (pending) { /*软中断处理未超过 2ms，且没有进程需要调度，且以上过程未重复执行超过 10次*/
        if (time_before(jiffies, end) && !need_resched() &&
            --max_restart)
            goto restart; /*仍然在软中断上下文中执行软中断*/
        /*否则唤醒 ksoftirqd 去执行剩下的软中断*/
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
    /*如果有挂起的软中断，且 ksoftirqd 并未运行，则处理挂起的软中断*/
    if (pending && !ksoftirqd_running())
        do_softirq_own_stack();

    local_irq_restore(flags);
}
```
* 注意：由此可见，调用`do_softirq()`并不意味着一定会调用`__do_softirq()`，让挂起的软中断同步得到处理。
* 对于大部分体系结构，`do_softirq_own_stack()`的实现就是调用`__do_softirq()`
* 不同的是 `CONFIG_SOFTIRQ_ON_OWN_STACK` 选项可以控制软中断处理使用的是被打断的当前栈还是中断栈
* 通用的 `CONFIG_SOFTIRQ_ON_OWN_STACK` 关闭时的处理：
  * include/asm-generic/softirq_stack.h
  ```c
  ...
  #ifdef CONFIG_SOFTIRQ_ON_OWN_STACK
  void do_softirq_own_stack(void);
  #else
  static inline void do_softirq_own_stack(void)
  {
      __do_softirq();
  }
  #endif
  ...
  ```
* x86 开启 `CONFIG_SOFTIRQ_ON_OWN_STACK` 时 `do_softirq_own_stack()` 就通过汇编 `call_on_irqstack` 切换到中断栈上，然后在处理软中断
  * arch/x86/include/asm/irq_stack.h
  ```c
  #ifdef CONFIG_SOFTIRQ_ON_OWN_STACK
  /*
   * Macro to invoke __do_softirq on the irq stack. This is only called from
   * task context when bottom halves are about to be reenabled and soft
   * interrupts are pending to be processed. The interrupt stack cannot be in
   * use here.
   */
  #define do_softirq_own_stack()                                          \
  {                                                                       \
          __this_cpu_write(hardirq_stack_inuse, true);                    \
          call_on_irqstack(__do_softirq, ASM_CALL_ARG0);                  \
          __this_cpu_write(hardirq_stack_inuse, false);                   \
  }

  #endif
  ```

### 中断退出时处理软中断
* 从`irq_exit()`开始看起。退出中断上下文，如果需要且可能的话，执行软中断处理。
* kernel/softirq.c
```c

static inline void invoke_softirq(void)
{       /*如果本 CPU 上软中断处理进程已经在运行状态了，则直接返回*/
        if (ksoftirqd_running())
                return;
        /*如果没有启用中断线程化，则直接处理软中断（同步方式）*/
        if (!force_irqthreads) {
#ifdef CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK
                /*
                 * We can safely execute softirq on the current stack if
                 * it is the irq stack, because it should be near empty
                 * at this stage.
                 */
                /*如果开启 CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK，在中断处理函数使用过的
                  栈上继续执行软中断，这很安全，因为此时它几乎是空的*/
                __do_softirq();
#else
                /*
                 * Otherwise, irq_exit() is called on the task stack that can
                 * be potentially deep already. So call softirq in its own stack
                 * to prevent from any overrun.
                 */
                /*否则，irq_exit() 会在任务栈上调用，而该栈可能已经很深了。
                  因此，在 softirq 自己的栈中调用它，以防止出现溢出。*/
                do_softirq_own_stack();
#endif
        } else {
                /*如果启用中断线程化，且中断处理进程并未运行，在这里唤醒它（异步方式）*/
                wakeup_softirqd();
        }
}
...
/*
 * Exit an interrupt context. Process softirqs if needed and possible:
 */
void irq_exit(void)
{
#ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
        local_irq_disable();
#else
        WARN_ON_ONCE(!irqs_disabled());
#endif
        /*在 irq 退出时进行记账*/
        account_irq_exit_time(current);
        preempt_count_sub(HARDIRQ_OFFSET); /*递减抢占计数中用于 HARDIRQ 的计数*/
        /*如果不处于中断上下文，包括：NMI，硬中断，软中断或者禁止下半部的情形；
          并且有挂起的软中断待处理，调用软中断处理函数*/
        if (!in_interrupt() && local_softirq_pending())
                invoke_softirq();

        tick_irq_exit();
        rcu_irq_exit();
        trace_hardirq_exit(); /* must be last! */
}
```

### `ksoftirqd`中断处理进程
* 每个CPU一个的辅助处理 softirq（和 tasklet）的内核线程
* 引入 ksoftirqd 的原因：
  * 在中断处理函数返回时处理 softirq 是最常见的 softirq 处理时机
  * softirq 触发的频率有时很高，而有的 softirq 还会重新触发自己以便得到再次执行
  * 为防止用户空间进程饥饿，作为折中的方案，内核不会立即处理重新触发的 softirq
  * 当大量 softirq 出现时，内核会唤醒一组内核线程来处理这些负载，即 **ksoftirqd**
* ksoftirqd 每次迭代都会最终调用`schedule()`让其他进程有机会得到处理
  ```c
  early_initcall(spawn_ksoftirqd)

  start_kernel()
   ...
   -> spawn_ksoftirqd()
      -> smpboot_register_percpu_thread(&softirq_threads)
         -> smpboot_register_percpu_thread_cpumask()
            -> __smpboot_create_thread()
               -> kthread_create_on_cpu(smpboot_thread_fn,...)

  smpboot_thread_fn()
   -> ht->thread_fn(td->cpu)
  ```

* kernel/softirq.c
  ```c
  static int ksoftirqd_should_run(unsigned int cpu)
  {
          return local_softirq_pending();
  }

  static void run_ksoftirqd(unsigned int cpu)
  {
          local_irq_disable(); /*先关闭中断，在进入软中断处理函数前一刻才开启它*/
          if (local_softirq_pending()) { /*如果有软中断挂起*/
                  /*
                   * We can safely run softirq on inline stack, as we are not deep
                   * in the task stack here.
                   */
                  __do_softirq();        /*进行软中断处理*/
                  local_irq_enable();    /*开启之前关闭的中断*/
                  cond_resched_rcu_qs();
                  return;
          }
          local_irq_enable(); /*开启之前关闭的中断*/
  }
  ...
  static struct smp_hotplug_thread softirq_threads = {
          .store                  = &ksoftirqd,
          .thread_should_run      = ksoftirqd_should_run,
          .thread_fn              = run_ksoftirqd, /*软中断处理进程函数*/
          .thread_comm            = "ksoftirqd/%u",
  };

  static __init int spawn_ksoftirqd(void)
  {
          cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL,
                                    takeover_tasklets);
          BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

          return 0;
  }
  early_initcall(spawn_ksoftirqd); /*通过 initcall 的方式来注册软中断处理进程函数*/
  ```

## 使用软中断

### 注册 softirq

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
      /*例如，这里注册了 tasklet 和 HI_SOFTIRQ 的软中断处理函数*/
      open_softirq(TASKLET_SOFTIRQ, tasklet_action);
      open_softirq(HI_SOFTIRQ, tasklet_hi_action);
  }
  ```
* 软中断处理程序执行时允许响应中断（即开中断），但不能睡眠
* 在一个软中断处理程序在运行时，当前处理器的软中断被禁止，但其他处理器并不禁止软中断
* 因此其他处理器可以运行软中断处理程序，甚至是同一处理程序
* 这意味着共享数据要加锁，但加锁又影响性能，失去软中断的意义，因此尽量用单处理器数据或其他技巧避免加锁
* 还有就是改用 tasklet

### 触发 softirq
* `raise_softirq()` 是对 `raise_softirq_irqoff()` 的封装，会确保触发软中断前后开启和关闭中断
* `raise_softirq_irqoff()` 是对 `__raise_softirq_irqoff()` 的封装
  * 必须在关中断的情况下调用，否则会唤醒 `ksoftirqd` 确保软中断会被尽快处理
  * 如果知道中断已经被禁，比如中断上下文，可以直接调
* `__raise_softirq_irqoff()` 是最底层的调用
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
          wakeup_softirqd(); /*唤醒 ksoftirqd 软中断处理进程*/
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

# tasklet

* 用 softirq 实现，因此本质上也是 softirq
* `tasklet_struct`结构
  ```c
  struct tasklet_struct
  {
      struct tasklet_struct *next;
      unsigned long state;
      atomic_t count;
      void (*func)(unsigned long);
      unsigned long data;
  };
  ```
* `count` 成员是某个 tasklet 的引用计数，是关于这个 tasklet 的开关
  * 不为 `0`，被禁止；
  * 为 `0`，激活，被设为挂起状态才能执行
* `state` 的 `TASKLET_STATE_RUN` 则是用来检查当前 tasklet 是否已经在某个处理器上执行
* tasklet state
  ```c
  enum
  {
      TASKLET_STATE_SCHED,    /* Tasklet is scheduled for execution */
      TASKLET_STATE_RUN   /* Tasklet is running (SMP only) */
  };
  ```
* 分为 `HI_SOFTIRQ` 和 `TASKLET_SOFTIRQ` 两类，`HI_SOFTIRQ` 优先于 `TASKLET_SOFTIRQ` 执行
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

## 调度 tasklet
* `tasklet_schedule(t)` 和 `tasklet_hi_schedule(t)`
  * 如果 `t.state` 没置位，置为 `TASKLET_STATE_SCHED`，否则什么也不做
  * 把 `tasklet_struct` 加入 per-CPU 链表
  * 触发 `TASKLET_SOFTIRQ` softirq
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
    raise_softirq_irqoff(TASKLET_SOFTIRQ);  /*触发 TASKLET_SOFTIRQ softirq*/
    local_irq_restore(flags);
}
EXPORT_SYMBOL(__tasklet_schedule);
```

## 处理 tasklet
* `tasklet_action()`和`tasklet_hi_action()`，tasklet的softirq处理函数，在`softirq_init()`时注册
* 需要通过测试并设置`TASKLET_STATE_RUN`来保障同一类型的tasklet不能同时在其他CPU上运行
  * include/linux/interrupt.h
```cpp
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
        /*先看看该类型 tasklet 是否在其他 CPU 上执行。
          如果没有，则原子地置位 RUN，开始执行；否则跳过该 tasklet*/
        if (tasklet_trylock(t)) {
            if (!atomic_read(&t->count)) { /*检查 tasklet 是否被禁止，count == 0 表示激活*/
                if (!test_and_clear_bit(TASKLET_STATE_SCHED,
                            &t->state)) //我们置的 RUN，必须由我们来清 SCHED
                    BUG();              //如果 SCHED 已经被别人清了，必然是 bug
                t->func(t->data);  /*执行 tasklet*/
                tasklet_unlock(t); /*清除 RUN 标志*/
                continue;          /*接着处理下一个tasklet*/
            }
            tasklet_unlock(t);
        }
        /*跳过的 tasklet 放回链表，并触发 softirq 让它下回再处理*/
        local_irq_disable();
        t->next = NULL;
        *__this_cpu_read(tasklet_vec.tail) = t;
        __this_cpu_write(tasklet_vec.tail, &(t->next));
        __raise_softirq_irqoff(TASKLET_SOFTIRQ);
        local_irq_enable();
    }
}
```

## 使用 tasklet

### 声明 tasklet

* 静态创建 tasklet
```c
#define DECLARE_TASKLET(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }

#define DECLARE_TASKLET_DISABLED(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(1), func, data }
```

* 动态创建 tasklet
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

### 实现 tasklet 处理函数的注意事项
* 因为是靠 softirq 实现的，所以 tasklet 处理函数不能睡眠
  * 不能用信号量或阻塞式函数
* 注意，此期间允许响应中断
* 注意保护不同 tasklet 间的共享数据，或者 tasklet 与其他 softirq 共享的数据

### tasklet 接口函数
* `tasklet_schedule()`
* `tasklet_disable()`和`tasklet_disable_nosync()`，增加 `tasklet_struct.count`，不为 `0` 表示禁止
* `tasklet_enable()`，减小 `tasklet_struct.count`，为 `0` 表示激活
* `tasklet_kill()`从链表中移出某个 tasklet
  * 会等待正在执行的 tasklet 执行完毕
  * 注意，其他地方仍有可能将 tasklet 再度放回链表
  * 可能会引起休眠，**禁止在中断上下文使用**

# 工作队列（work queue）
* 现在工作队列的实现是 [CMWQ（Concurrency Managed Workqueue）](cmwq.md)
* 工作队列是将工作推后执行的一种方式，可以用来实现中断下半部
* 工作队列在**进程上下文**，允许在工作中重新调度或睡眠。
  * 需要大量内存
  * 获取信号量
  * 阻塞式I/O

## 工作队列的实现
* **工作者线程**（worker thread）是内核线程，由工作队列子系统提供的接口创建，负责执行由内核其他部分排到队列里的任务。
* 驱动程序可以创建一个专门的 worker thread 来处理要推后的工作。
* 工作队列子系统也提供缺省的 worker thread，每个处理器对应一个。
* 创建属于自己的 worker thread 的情况：
  * 要在 worker thread 执行大量的处理操作
  * CPU 密集型和性能要求严格的任务。可减轻缺省 worker thread 的负担，避免饥饿

### worker thread 结构体
* 每种类型的 work thread 都对应到一个 `workqueue_struct` 结构体。
* `workqueue_struct` 结构体包含一个 Per CPU 的 `cpu_workqueue_struct` 结构体。
* 每种类型的 work thread 在每个 CPU 上有一个 thread。
* 因此每个 work thread 可以用一个 `cpu_workqueue_struct`。
* `cpu_workqueue_struct` 结构的成员 `worklist` 为链表头，链表成员是`work_struct` 类型的任务。

# 下半部之间加锁
* 在使用下半部机制时，即使是在单处理器系统下，避免共享数据被同时访问也是至关重要的。
* 一个下半部实际上可能在任何时候执行。
* 同类型tasklet间共享的数据无需考虑同步问题。
* 两个不同类型的tasklet共享同一数据需要加锁。
* 为了本地和SMP的保护并且防止死锁出现：
  * 如果**进程上下文**和一个**下半部**共享数据，在访问这些数据之前，需要**禁止下半部**且获得锁的使用权。
  * 如果**中断上下文**和一个**下半部**共享数据，在访问数据之前，需要**禁止中断**且获得锁的使用权。
* 任何在工作队列中被共享的数据需要使用锁机制。
* Q：为什么共享数据除了需要锁的保护，还需要禁止中断或禁止下半部？是因为加锁时也有可能被中断打断吗？
  * A：禁止中断或禁止下半部的原因是，中断会抢占下半部的执行，下半部会打断进程的执行。
    *  假设进程持有锁，而打断它的下半部竞争的是同一把锁，则下半部会因为无法得到进程持有的锁而挂住，造成死锁。
    *  如果进程禁止了下半部再去拿锁，则不会被下半部打断，直到它再次开启下半部。
    *  中断与下半部之间共享数据同理。

# 禁止下半部
* 禁止下半部主要指禁止**softirq**和**tasklet**。
* 保护数据的常见的做法是**先拿到锁，再禁下半部** 吗？
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

#define PREEMPT_OFFSET  (1UL << PREEMPT_SHIFT)	/*抢占计数偏移*/
#define SOFTIRQ_OFFSET  (1UL << SOFTIRQ_SHIFT)	/*软中断偏移*/
#define HARDIRQ_OFFSET  (1UL << HARDIRQ_SHIFT)	/*硬中断偏移*/
#define NMI_OFFSET  (1UL << NMI_SHIFT)					/*NMI中断偏移*/

#define SOFTIRQ_DISABLE_OFFSET  (2 * SOFTIRQ_OFFSET)  /*禁止下半部偏移，为了与处理软中断区别开*/

/* We use the MSB mostly because its available */
#define PREEMPT_NEED_RESCHED    0x80000000
...__```
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
	...*```
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
...__```
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

* 关于`SOFTIRQ_DISABLE_OFFSET`，`in_serving_softirq()`更多信息见该 commit `75e1056f5c57050415b64cb761a3acc35d91f013`: sched: Fix softirq time accounting
