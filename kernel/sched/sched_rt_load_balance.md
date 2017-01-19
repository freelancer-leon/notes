# 实时调度负载均衡
***
# 目录

- [Root Domain](#Root-Domain)
- [CPU优先级管理](#CPU优先级管理)
- [balance_callback()调用回调函数](#balance_callback()调用回调函数)
- [参考资料](#参考资料)

# Root Domain
* 实时调度器需要几个全局的，或者说系统范围的资源作出调度决定，以及 CPU 数量的增加而出现的可伸缩性瓶颈（由于锁保护的这些资源的竞争）。
* *Root Domain* 引入的目的就是为了减少这样的竞争以改善可伸缩性。
* cpuset 提供了一个把 CPU 分成子集被一个进程或者或一组进程使用的机制。
* 几个 cpuset 可以重叠。
* 一个 cpuset 被称为“*互斥的（exclusive）*”，如果没有其他的 cpuset 包含重叠的 CPU。
* 每个互斥的 cpuset 定义了一个与其他 cpuset 或 CPU 分离的 **孤岛域（isolated domain，也叫作 **root domain）**。
* 与每个 root domian 有关的信息存在 `struct root_domain`结构（对象）中：
* kernel/sched/sched.h
```c
#ifdef CONFIG_SMP

/*
 * We add the notion of a root-domain which will be used to define per-domain
 * variables. Each exclusive cpuset essentially defines an island domain by
 * fully partitioning the member cpus from any other cpuset. Whenever a new
 * exclusive cpuset is created, we also create and attach a new root-domain
 * object.
 *
 */
struct root_domain {
        atomic_t refcount;
        atomic_t rto_count;
        struct rcu_head rcu;
        cpumask_var_t span;
        cpumask_var_t online;

        /* Indicate more than one runnable task for any CPU */
        bool overload;

        /*
         * The bit corresponding to a CPU gets set here if such CPU has more
         * than one runnable -deadline task (as it is below for RT tasks).
         */
        cpumask_var_t dlo_mask;
        atomic_t dlo_count;
        struct dl_bw dl_bw;
        struct cpudl cpudl;

        /*
         * The "RT overload" flag: it gets set if a CPU has more than
         * one runnable RT task.
         */
        cpumask_var_t rto_mask;
        struct cpupri cpupri;

        unsigned long max_cpu_capacity;
};

extern struct root_domain def_root_domain;

#endif /* CONFIG_SMP */
```
* `refcount` root domain 的引用计数，当 root domain 被运行队列引用时加一，反之减一。
* `span` *属于该 root domain 的运行队列* 的 *可用 CPU* 的范围，`cpumask_var_t`掩码。
* `overload` 表明该 root domain 有任一 CPU 有多于一个的可运行任务。
* `rto_mask` 某 CPU 有多于一个的可运行实时任务，对应的位被设置，`cpumask_var_t`掩码。
* `rto_count` 过载的（overload）的 CPU 的数目。
* `cpupri` 包含在 root domain 中的 *CPU 优先级管理* 结构成员，详见下文。
* 这些 root domain 被用于减小 per-domain 变量的全局变量的范围。
* 无论何时一个互斥 cpuset 被创建，一个新 root domain 对象也会被创建，信息来自 CPU 成员。
* 缺省情况下，一个单独高层的 root domain 被创建，并把所有 CPU 作为成员。
* 所有的实时调度决定只在一个 root domain 的范围内作出决定。

# CPU优先级管理
* CPU 优先级管理（CPU Priority Management）跟踪系统中每个 CPU 的优先级，为了让进程迁移的决定更有效率。
* CPU 优先级有 102 个:

cpupri | prio
---|---
CPUPRI_INVALID (-1) | -1
CPUPRI_IDLE (0) | MAX_PRIO (140)
CPUPRI_NORMAL (1) | MAX_RT_PRIO ~ MAX_PRIO-1 (100~139)
2~101 | 99~0

* `prio`转`cpupri`的函数如下：
  * kernel/sched/cpupri.c
```c
/* Convert between a 140 based task->prio, and our 102 based cpupri */
static int convert_prio(int prio)
{
        int cpupri;

        if (prio == CPUPRI_INVALID)
                cpupri = CPUPRI_INVALID;
        else if (prio == MAX_PRIO)
                cpupri = CPUPRI_IDLE;
        else if (prio >= MAX_RT_PRIO)
                cpupri = CPUPRI_NORMAL;
        else
                cpupri = MAX_RT_PRIO - prio + 1;

        return cpupri;
}
```

* 处于`CPUPRI_INVALID`状态的 CPU 没有资格参与 task routing。
* `cpupri`在 root domain 级别的范围内。每个互斥的 cpuset 由一个含有 cpupri 数据的 root momain 组成。
* 系统从两个维度的位映射来维护这些 CPU 状态：
  1. 不同的优先级
  2. 在某个优先级上的 CPU（CPU 的优先级等于 `rq->rt.highest_prio`）
* kernel/sched/cpupri.h
```c
#define CPUPRI_NR_PRIORITIES    (MAX_RT_PRIO + 2)
...
struct cpupri_vec {
        atomic_t        count;
        cpumask_var_t   mask;
};

struct cpupri {
        struct cpupri_vec pri_to_cpu[CPUPRI_NR_PRIORITIES];
        int *cpu_to_pri;
};
```
#### `struct cpupri`
* `cpu_to_pri` 指示一个 *CPU 的优先级*。
* `pri_to_cpu` 持有关于一个 cpuset 在某个特定的优先级上的所有 CPU 的信息。

#### `struct cpupri_vec`
* `count` 在这个优先级上的 CPU 的数量。
* `mask` 在这个优先级上的 CPU 位码。

# balance_callback()调用回调函数
* `queue_push_tasks()`和`queue_pull_task()`可以将实时调度负载均衡函数`push_rt_tasks()`和`pull_rt_task()`放入`rq->balance_callback`链表。
* 负载均衡函数会等到调用`balance_callback()`时执行均衡负载。
* kernel/sched/rt.c
```c
static inline int has_pushable_tasks(struct rq *rq)
{
        return !plist_head_empty(&rq->rt.pushable_tasks);
}

static DEFINE_PER_CPU(struct callback_head, rt_push_head);
static DEFINE_PER_CPU(struct callback_head, rt_pull_head);

static void push_rt_tasks(struct rq *);
static void pull_rt_task(struct rq *);

static inline void queue_push_tasks(struct rq *rq)
{
        if (!has_pushable_tasks(rq))
                return;

        queue_balance_callback(rq, &per_cpu(rt_push_head, rq->cpu), push_rt_tasks);
}

static inline void queue_pull_task(struct rq *rq)
{
        queue_balance_callback(rq, &per_cpu(rt_pull_head, rq->cpu), pull_rt_task);
}
```

* kernel/sched/sched.h
```c
static inline void
queue_balance_callback(struct rq *rq,
                       struct callback_head *head,
                       void (*func)(struct rq *rq))
{
        lockdep_assert_held(&rq->lock);

        if (unlikely(head->next))
                return;

        head->func = (void (*)(struct callback_head *))func;
        head->next = rq->balance_callback;
        rq->balance_callback = head;
}
```

* kernel/sched/core.c
```c
/* rq->lock is NOT held, but preemption is disabled */
static void __balance_callback(struct rq *rq)
{
        struct callback_head *head, *next;
        void (*func)(struct rq *rq);
        unsigned long flags;

        raw_spin_lock_irqsave(&rq->lock, flags);
        head = rq->balance_callback;
        rq->balance_callback = NULL;
        while (head) {
                func = (void (*)(struct rq *))head->func;
                next = head->next;
                head->next = NULL; /*这个操作很重要，能确保链表元素指向自身时不会出现死循环*/
                head = next;

                func(rq);
        }
        raw_spin_unlock_irqrestore(&rq->lock, flags);
}

static inline void balance_callback(struct rq *rq)
{       /*先判断一下 rq->balance_callback 指针是不是空*/
        if (unlikely(rq->balance_callback))
                __balance_callback(rq);
}
...
```

# 参考资料
- [Real-Time Linux Kernel Scheduler](http://www.linuxjournal.com/magazine/real-time-linux-kernel-scheduler)
- [Linux Kernel 排程機制介紹](http://blog.csdn.net/hlchou/article/details/7425416)
