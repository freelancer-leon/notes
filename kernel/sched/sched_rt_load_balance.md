# 实时调度负载均衡
***
# 目录

- [Root Domain](#Root-Domain)
- [CPU优先级管理](#CPU优先级管理)
- [推调度算法](#推调度算法)
- [拉调度算法](#拉调度算法)
- [balance_callback()调用回调函数](#balance_callback()调用回调函数)
- [参考资料](#参考资料)

# Root Domain
* 实时调度器需要几个全局的，或者说系统范围的资源作出调度决定，以及 CPU 数量的增加而出现的可伸缩性瓶颈（由于锁保护的这些资源的竞争）。
* *Root Domain* 引入的目的就是为了减少这样的竞争以改善可伸缩性。
* cpuset 提供了一个把 CPU 分成子集被一个进程或者或一组进程使用的机制。
* 几个 cpuset 可以重叠。
* 一个 cpuset 被称为“*互斥的（exclusive）*”，如果没有其他的 cpuset 包含重叠的 CPU。
* 每个互斥的 cpuset 定义了一个与其他 cpuset 或 CPU 分离的 **孤岛域（isolated domain，也叫作 root domain）**。
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
* `cpupri` 数值越大表示优先级越高（用的是减法）。
* 处于`CPUPRI_INVALID`状态的 CPU 没有资格参与 task routing。
* `cpupri` 属于 root domain level 的范围。每个互斥的 cpuset 由一个含有 cpupri 数据的 root momain 组成。
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
* `pri_to_cpu` 持有关于一个 cpuset 在 *某个特定的优先级上的* 所有 CPU 的信息。
* `cpu_to_pri` 指示一个CPU 的优先级。

#### `struct cpupri_vec`
* `count` 在这个优先级上的 CPU 的数量。
* `mask` 在这个优先级上的 CPU 位码。

## cpupri_find()
* `cpupri_find()` 查找系统（root domain）中最佳（优先级最低）的 CPU。
* kernel/sched/cpupri.c
```c
/**
 * cpupri_find - find the best (lowest-pri) CPU in the system
 * @cp: The cpupri context
 * @p: The task
 * @lowest_mask: A mask to fill in with selected CPUs (or NULL)
 *
 * Note: This function returns the recommended CPUs as calculated during the
 * current invocation.  By the time the call returns, the CPUs may have in
 * fact changed priorities any number of times.  While not ideal, it is not
 * an issue of correctness since the normal rebalancer logic will correct
 * any discrepancies created by racing against the uncertainty of the current
 * priority configuration.
 *
 * Return: (int)bool - CPUs were found
 */
int cpupri_find(struct cpupri *cp, struct task_struct *p,
                struct cpumask *lowest_mask)
{
        int idx = 0;
        int task_pri = convert_prio(p->prio); /*将任务的有效优先级转为 CPU 优先级*/
        /*给定任务的 CPU 优先级不能超过 CPU 优先级的合理范围*/
        BUG_ON(task_pri >= CPUPRI_NR_PRIORITIES);
        /*从最低优先级开始检查，因为跑着低优先级任务的 CPU 更适合被 routing。
          为什么是从 0 开始？
          还记得 idle 状态的 CPU 和运行普通任务的 CPU 优先级分别是：
          CPUPRI_IDLE (0)、CPUPRI_NORMAL (1)。
          所以这个查找涵盖了空闲的 CPU 和运行普通任务的 CPU。*/
        for (idx = 0; idx < task_pri; idx++) {
                struct cpupri_vec *vec  = &cp->pri_to_cpu[idx];
                int skip = 0;
                /*跳过在某个优先级上没有 CPU 的向量*/
                if (!atomic_read(&(vec)->count))
                        skip = 1;
                /*
                 * When looking at the vector, we need to read the counter,
                 * do a memory barrier, then read the mask.
                 *
                 * Note: This is still all racey, but we can deal with it.
                 *  Ideally, we only want to look at masks that are set.
                 *
                 *  If a mask is not set, then the only thing wrong is that we
                 *  did a little more work than necessary.
                 *
                 *  If we read a zero count but the mask is set, because of the
                 *  memory barriers, that can only happen when the highest prio
                 *  task for a run queue has left the run queue, in which case,
                 *  it will be followed by a pull. If the task we are processing
                 *  fails to find a proper place to go, that pull request will
                 *  pull this task if the run queue is running at a lower
                 *  priority.
                 */
                smp_rmb();

                /* Need to do the rmb for every iteration */
                if (skip)
                        continue;
                /*任务亲和的 CPU 掩码与向量的掩码 “与” 运算，看有没有合适的 CPU。
                  如果没有 CPU 被设置，cpumask_any_and()结果 >= nr_cpu_ids*/
                if (cpumask_any_and(tsk_cpus_allowed(p), vec->mask) >= nr_cpu_ids)
                        continue;

                if (lowest_mask) {
                        /*任务亲和的 CPU 掩码与向量的掩码 “与” 运算赋给 lowest_mask*/
                        cpumask_and(lowest_mask, tsk_cpus_allowed(p), vec->mask);

                        /*
                         * We have to ensure that we have at least one bit
                         * still set in the array, since the map could have
                         * been concurrently emptied between the first and
                         * second reads of vec->mask.  If we hit this
                         * condition, simply act as though we never hit this
                         * priority level and continue on.
                         */
                        /*再次确保 lowest_mask 非空，
                          如果 lowest_mask 为空，cpumask_any()返回结果 >= nr_cpu_ids，进行下一次循环；
                          如果 lowest_mask 不为空，则通过下面的 return 跳出循环并使函数返回。*/
                        if (cpumask_any(lowest_mask) >= nr_cpu_ids)
                                continue;
                }

                return 1;
        }

        return 0;
}
```

## cpupri_set()

# 推调度算法
* **推任务** 搜索一个运行队列，转移它的一个任务到另一个运行队列的操作。
* `push_rt_task()` 算法着眼于运行队列上优先级最高的不在运行的可运行实时任务，考虑所有运行队列，找到一个它能运行的 CPU。
* 它搜索一个优先级更低的队列，就是当前正在运行的任务可以被 *正被推送的任务* 抢占的队列。
* 如前所述，CPU 优先级管理基础结构就是被用于找到一个有最低优先级运行队列的 CPU 掩码。从所有的候选者中选择唯一的最佳 CPU 很重要。
  * 该算法最先考虑的是把任务给最后在上面执行的 CPU，由于它的 cache 很可能还是热的。
  * 如果不行，考虑调度域映射，找到一个逻辑上接近上次运行 CPU 的那一个。
  * 如果也失败了，从掩码中随机选择一个 CPU。
* 推操作被执行直至：
  * 一个实时任务迁移失败
  * 或者没有任务可以被推送
* 因为算法总是选择优先级最高的非运行任务来推送，那这个假设就是，如果它不能迁移，那么最有可能的低一点的实时任务也不能被迁移，于是搜索被中断。
* 当扫描最低优先级运行队列的时候是不加锁的。
  * 当目标运行队列被找到时，只锁定那个队列，之后做一个检查来验证它是否仍旧是一个推任务的候选（由于目标队列可能被一个在其他 CPU 上并行的调度操作修改）。
  * 如果不再合适被推送，重复搜索最多三次，之后搜索被中断。
* `push_rt_task()`函数会在以下时间点被调用：
  1. 非正在运行的普通进程变成实时进程时（比如通过`sched_setscheduler`系统调用）；
  2. 发生调度之后（这时候可能有一个实时进程被更高优先级的实时进程抢占了）；
  3. 实时进程被唤醒之后，如果不能马上在当前 CPU 上运行（它不是当前 CPU 上优先级最高的进程）；

## 推任务的实现
* 先从`push_rt_tasks()`的实现看起。
* kernel/sched/rt.c
```c
/* Only try algorithms three times */
#define RT_MAX_TRIES 3
...
static DEFINE_PER_CPU(cpumask_var_t, local_cpu_mask);
/*返回值为 CPU id，出错时返回 -1*/
static int find_lowest_rq(struct task_struct *task)
{
        struct sched_domain *sd;
        struct cpumask *lowest_mask = this_cpu_cpumask_var_ptr(local_cpu_mask);
        int this_cpu = smp_processor_id(); /*当前 CPU*/
        int cpu      = task_cpu(task);     /*task 所在 CPU*/

        /* Make sure the mask is initialized first */
        if (unlikely(!lowest_mask))
                return -1;
        /*task 设置为只允许在一个 CPU 上运行，对于其他目标队列来说是不能迁移的，返回 -1。*/
        if (tsk_nr_cpus_allowed(task) == 1)
                return -1; /* No other targets possible */
        /*用之前提到的 cpupri_find() 在 task 所属的 root domain 中找优先级最低的 CPU*/
        if (!cpupri_find(&task_rq(task)->rd->cpupri, task, lowest_mask))
                return -1; /* No targets found */

        /*
         * At this point we have built a mask of cpus representing the
         * lowest priority tasks in the system.  Now we want to elect
         * the best one based on our affinity and topology.
         *
         * We prioritize the last cpu that the task executed on since
         * it is most likely cache-hot in that location.
         */
        /*此时，我们建立了一个系统中运行最低优先级任务的 CPU 的掩码。现在我们想根据我们的
          亲和性和拓扑结构选出最佳的那个。
          我们优先选择上一次执行任务的 CPU，因为它的 cache 很可能还是热的。*/
        if (cpumask_test_cpu(cpu, lowest_mask)) /*测试 task 的 CPU 是否在掩码中*/
                return cpu; /*如果在掩码中，优先选择该 CPU*/

        /*
         * Otherwise, we consult the sched_domains span maps to figure
         * out which cpu is logically closest to our hot cache data.
         */
        /*否则，我们根据调度域的范围映射来找出那个 CPU 在逻辑上最接近我们的热缓存数据。*/
        if (!cpumask_test_cpu(this_cpu, lowest_mask))
                this_cpu = -1; /* Skip this_cpu opt if not among lowest */
        /*如果当前 CPU 不在掩码给出的 CPU 里，标记成 -1 跳过它。*/
        rcu_read_lock(); /*domain tree 有 RCU quiescent state transition 的保护*/
        for_each_domain(cpu, sd) { /*由下至上遍历调度域，优先选择邻近的*/
                if (sd->flags & SD_WAKE_AFFINE) {
                        int best_cpu;

                        /*
                         * "this_cpu" is cheaper to preempt than a
                         * remote processor.
                         */
                        /*抢占 "this_cpu" 比抢占一个远程的处理器开销更低。
                          当前 CPU 如果在之前的最低优先级掩码里，且在同一调度域的分支上，
                          则有可能被选中，原因如上所述*/
                        if (this_cpu != -1 &&
                            cpumask_test_cpu(this_cpu, sched_domain_span(sd))) {
                                rcu_read_unlock();
                                return this_cpu;
                        }
                        /*如果当前 CPU 不再同一调度域的分支上，在 "最低优先级掩码" 与
                          "调度域的 CPU span" 的交集中选第一个 CPU 作为最佳 CPU*/
                        best_cpu = cpumask_first_and(lowest_mask,
                                                     sched_domain_span(sd));
                        if (best_cpu < nr_cpu_ids) {
                                rcu_read_unlock();
                                return best_cpu;
                        }
                }
        }
        rcu_read_unlock();

        /*
         * And finally, if there were no matches within the domains
         * just give the caller *something* to work with from the compatible
         * locations.
         */
        /*最后，如果在调度域里没有匹配 lowest_mask 的 CPU，那么就把这个在 lowest_mask
          掩码里，但不在同一调度域分支上的当前 CPU 返回。
          这意味此时当前 CPU （就调度域划分而言）是远程的。*/
        if (this_cpu != -1)
                return this_cpu;
        /*如果当前 CPU 既不在 lowest_mask 掩码里，也不在同一调度域分支上。
          从最低优先级掩码给定的 CPU 中随机选择一个 CPU 返回。*/
        cpu = cpumask_any(lowest_mask);
        if (cpu < nr_cpu_ids)
                return cpu;
        return -1; /*这样的 CPU 如果是非法值，那只能说找不到了。*/
}

/* Will lock the rq it finds */
/*返回值为指向运行队列的指针 - *rq，找不到时返回 NULL*/
static struct rq *find_lock_lowest_rq(struct task_struct *task, struct rq *rq)
{
        struct rq *lowest_rq = NULL; /*指向要返回的优先级最低队列*/
        int tries;
        int cpu;
        /*目前只尝试三次*/
        for (tries = 0; tries < RT_MAX_TRIES; tries++) {
                cpu = find_lowest_rq(task);
                /*如果找不到一个合适的队列，或者找到的队列就是当前队列，则跳出循环，用已有的
                  lowest_rq 的值。*/
                if ((cpu == -1) || (cpu == rq->cpu))
                        break;
                /*否则认为找到了优先级最低队列，更新 lowest_rq 的值为该 CPU 所属 rq*/
                lowest_rq = cpu_rq(cpu);
                if (lowest_rq->rt.highest_prio.curr <= task->prio) {
                        /*
                         * Target rq has tasks of equal or higher priority,
                         * retrying does not release any lock and is unlikely
                         * to yield a different result.
                         */
                        /*如果目标运行队列所记录的最高优先级高于或等于要推走的任务，重试不
                          释放任何锁，因此不可能导致不同的结果。
                          所以这里设置为无法找到队列就跳出循环了。
                          记住，highest_prio记录的是该队列上的最高优先级，但不一定是该
                          队列上正在运行的任务，有可能是该加入队列的还未被调度的高优先级
                          任务，因此是有可能于 CPU 优先级的记录不一致的。*/
                        lowest_rq = NULL;
                        break;
                }

                /* if the prio of this runqueue changed, try again */
                /*因为要在当前队列和目标队列之间迁移，因此需要同时锁定这两个队列*/
                if (double_lock_balance(rq, lowest_rq)) {
                        /*   
                         * We had to unlock the run queue. In
                         * the mean time, task could have
                         * migrated already or had its affinity changed.
                         * Also make sure that it wasn't scheduled on its rq.
                         */
                        /*double_lock_balance()会有个先解锁再加锁的过程，这个间隙有可
                          能会发生一些改变，因此需要重新做以下检查：
                          - 之前从 rq 里选出来的 task 现在其 rq 域不再指向 rq 了
                          - 进程的亲和性被修改导致不再在最低优先级队列的 CPU 里了
                          - 当前队列正在运行的任务就是该task
                          - 任务的调度器不再是实时调度器
                          - 任务不在运行队列上了
                          以上任何一种情况出现都被认为查找失败。*/
                        if (unlikely(task_rq(task) != rq ||
                                     !cpumask_test_cpu(lowest_rq->cpu,
                                                       tsk_cpus_allowed(task)) ||
                                     task_running(rq, task) ||
                                     !rt_task(task) ||
                                     !task_on_rq_queued(task))) {

                                double_unlock_balance(rq, lowest_rq);
                                lowest_rq = NULL;
                                break;
                        }    
                }    
                /* If this rq is still suitable use it. */
                /*如果能锁定成功且能通过以上一系列检查，这时才认为该运行队列是合适被推给任务
                  的，可以跳出循环了，此时返回值 lowest_rq 具备有效值。
                  解除 double_unlock_balance(rq, lowest_rq) 的地方在 push_rt_task()，
                  因为在这期间要一直锁定着两个队列。*/
                if (lowest_rq->rt.highest_prio.curr > task->prio)
                        break;

                /* try again */
                /*否则进行重试*/
                double_unlock_balance(rq, lowest_rq);
                lowest_rq = NULL;
        }

        return lowest_rq;
}

static struct task_struct *pick_next_pushable_task(struct rq *rq)
{
        struct task_struct *p;

        if (!has_pushable_tasks(rq))
                return NULL;
        /*plist是按照优先级由高到底进行排序的，所以 first entry 是链表里优先级最高的*/
        p = plist_first_entry(&rq->rt.pushable_tasks,
                              struct task_struct, pushable_tasks);
        /*返回前检查一些不该存在的错误*/
        BUG_ON(rq->cpu != task_cpu(p)); /*任务 CPU 与 队列 CPU 不是同一个*/
        BUG_ON(task_current(rq, p)); /*选出的任务竟然是当前任务*/
        BUG_ON(tsk_nr_cpus_allowed(p) <= 1); /*选出的任务仅允许在一个CPU上运行*/

        BUG_ON(!task_on_rq_queued(p)); /*选出的任务不在可运行队列上*/
        BUG_ON(!rt_task(p)); /*选出的任务不是实时任务*/

        return p;
}

/*
 * If the current CPU has more than one RT task, see if the non
 * running task can migrate over to a CPU that is running a task
 * of lesser priority.
 */
/*如果当前 CPU 有超过一个实时任务，看有没有非运行态任务可以被迁移到另一个运行着优先级低一些的
  任务的 CPU*/
static int push_rt_task(struct rq *rq)
{
        struct task_struct *next_task;
        struct rq *lowest_rq;
        int ret = 0;
        /*如果该队列没有过载，即没有一个可以被迁移的实时任务，则没有实时任务可以被推走*/
        if (!rq->rt.overloaded)
                return 0;
        /*首先在该队列的可推链表里找有没有合适的任务，如果没有则返回。
          之前任务入列的时候会通过 enqueue_pushable_task() 把可能的任务加进可推队列。*/
        next_task = pick_next_pushable_task(rq);
        if (!next_task)
                return 0;

retry:
        if (unlikely(next_task == rq->curr)) { /*选出的任务竟然是 rq 的当前任务*/
                WARN_ON(1);
                return 0; /*这种情况不是期望的行为，但也没可以推走的任务*/
        }

        /*
         * It's possible that the next_task slipped in of
         * higher priority than current. If that's the case
         * just reschedule current.
         */
        /*从可推队列里选出的任务可能是一个刚溜进的来的高优先级任务，如果是这种情况，重新调度
          当前任务即可，也不需要推给别人了。*/
        if (unlikely(next_task->prio < rq->curr->prio)) {
                resched_curr(rq);
                return 0;
        }

        /* We might release rq lock */
        get_task_struct(next_task);

        /* find_lock_lowest_rq locks the rq if found */
        /*在比选出来的任务优先级低的队列中，找到优先级最低的队列。
          如果找到的话，当前队列 rq 和最低优先级队列 lowest_rq 都会处于锁定状态*/
        lowest_rq = find_lock_lowest_rq(next_task, rq);
        if (!lowest_rq) { /*如果没找到优先级最低的队列*/
                struct task_struct *task;
                /*
                 * find_lock_lowest_rq releases rq->lock
                 * so it is possible that next_task has migrated.
                 *
                 * We need to make sure that the task is still on the same
                 * run-queue and is also still the next task eligible for
                 * pushing.
                 */
                /*因为 find_lock_lowest_rq() 会释放 rq->lock 锁，所以之前选中的
                  next_task 有已经被迁移的可能。*/
                task = pick_next_pushable_task(rq);
                if (task_cpu(next_task) == rq->cpu && task == next_task) {
                        /*
                         * The task hasn't migrated, and is still the next
                         * eligible task, but we failed to find a run-queue
                         * to push it to.  Do not retry in this case, since
                         * other cpus will pull from us when ready.
                         */
                        /*next_task 的 CPU 仍然是当前队列的 CPU，还没有被迁移，且仍然是
                         下一个具备被迁移资格的任务，但我们找不到一个合适的运行队列来接纳它。
                         这种情况不需要重试，直到其他 CPU 在合适的时候拉走它。*/
                        goto out;
                }

                if (!task)
                        /* No more tasks, just exit */
                        /*找不到优先级最低的队列，也没有其他任务可以推，直接退出*/
                        goto out;

                /*
                 * Something has shifted, try again.
                 */
                /*找不到优先级最低的队列，且之前选中的 next_task 已经被迁移走了，那么把刚
                  选出的 task 作为新的 next_task 进行下一次重试*/
                put_task_struct(next_task); /*对应到之前的 get_task_struct()*/
                next_task = task;
                goto retry;
        }
        /*如果找到优先级最低的队列，开始迁移 next_task*/
        /*选中的任务从当前队列 rq 中出列*/
        deactivate_task(rq, next_task, 0);
        /*选中的任务 cpu 设置为将要迁移至的最低优先级队列的 cpu*/
        set_task_cpu(next_task, lowest_rq->cpu);
        /*选中的任务进入最低优先级队列*/
        activate_task(lowest_rq, next_task, 0);
        ret = 1; /*返回值设置为 1 表示成功*/
        /*把最低优先级队列的当前任务设置为“需要被立即重新调度”*/
        resched_curr(lowest_rq);
        /*同时解锁当前队列 rq 和最低优先级队列 lowest_rq*/
        double_unlock_balance(rq, lowest_rq);

out:
        put_task_struct(next_task); /*对应到之前的 get_task_struct()*/

        return ret;
}

static void push_rt_tasks(struct rq *rq)
{
        /* push_rt_task will return true if it moved an RT */
        /*push_rt_task()如果移动了一个实时任务将会返回 1。
          这个循环会一直执行到给定队列没有任务可以推走*/
        while (push_rt_task(rq))
                ;
}
...`_
```

# 拉调度算法
* `pull_rt_task()`算法着眼于一个 root domain 中所有过载的运行队列，检查它们是否有一个实时任务能运行在目标运行队列（就是说，目标 CPU 在 `task->cpus_allowed_mask` 中）且其优先级高于将要被调度的运行队列的任务。
* 如果符合条件，将任务排在目标运行队列。
* 该搜索只在扫描 root domain 中所有过载的运行队列之后中断。因此，拉操作可能拉多于一个任务到目标运行队列。
* 如果算法在第一遍的时候只选择一个要被拉的候选任务，那么在第二遍的时候才实际的去拉，有一种可能是，被选择的最高优先级任务不再是候选（由于在其他 CPU 上的并行调度操作）。
* 为了避免这种在 *查找最高优先级队列* 和 *当实际去执行拉操作时最高优先级任务仍然在运行队列上* 的竞争，拉操作会继续去拉任务。
* 最坏的情况是，这可能导致许多被拉到目标运行队列的任务之后有可能被推到其他 CPU，导致任务弹跳。任务弹跳被任务是一种稀有事件。

## 拉任务的实现
* 先从`pull_rt_task()`的实现看起。
* kernel/sched/rt.c
```c
static void pull_rt_task(struct rq *this_rq)
{
	int this_cpu = this_rq->cpu, cpu;
	bool resched = false;
	struct task_struct *p;
	struct rq *src_rq;

	if (likely(!rt_overloaded(this_rq)))
		return;

	/*
	 * Match the barrier from rt_set_overloaded; this guarantees that if we
	 * see overloaded we must also see the rto_mask bit.
	 */
	smp_rmb();

#ifdef HAVE_RT_PUSH_IPI
	if (sched_feat(RT_PUSH_IPI)) {
		tell_cpu_to_push(this_rq);
		return;
	}
#endif

	for_each_cpu(cpu, this_rq->rd->rto_mask) {
		if (this_cpu == cpu)
			continue;

		src_rq = cpu_rq(cpu);

		/*
		 * Don't bother taking the src_rq->lock if the next highest
		 * task is known to be lower-priority than our current task.
		 * This may look racy, but if this value is about to go
		 * logically higher, the src_rq will push this task away.
		 * And if its going logically lower, we do not care
		 */
		if (src_rq->rt.highest_prio.next >=
		    this_rq->rt.highest_prio.curr)
			continue;

		/*
		 * We can potentially drop this_rq's lock in
		 * double_lock_balance, and another CPU could
		 * alter this_rq
		 */
		double_lock_balance(this_rq, src_rq);

		/*
		 * We can pull only a task, which is pushable
		 * on its rq, and no others.
		 */
		p = pick_highest_pushable_task(src_rq, this_cpu);

		/*
		 * Do we have an RT task that preempts
		 * the to-be-scheduled task?
		 */
		if (p && (p->prio < this_rq->rt.highest_prio.curr)) {
			WARN_ON(p == src_rq->curr);
			WARN_ON(!task_on_rq_queued(p));

			/*
			 * There's a chance that p is higher in priority
			 * than what's currently running on its cpu.
			 * This is just that p is wakeing up and hasn't
			 * had a chance to schedule. We only pull
			 * p if it is lower in priority than the
			 * current task on the run queue
			 */
			if (p->prio < src_rq->curr->prio)
				goto skip;

			resched = true;

			deactivate_task(src_rq, p, 0);
			set_task_cpu(p, this_cpu);
			activate_task(this_rq, p, 0);
			/*
			 * We continue with the search, just in
			 * case there's an even higher prio task
			 * in another runqueue. (low likelihood
			 * but possible)
			 */
		}
skip:
		double_unlock_balance(this_rq, src_rq);
	}

	if (resched)
		resched_curr(this_rq);
}
```

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
