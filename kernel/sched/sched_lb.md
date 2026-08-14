# CPU 负载均衡

* 负载均衡主要是 CFS 的 pull 模型：目标 CPU 在 tick、即将 idle、NOHZ idle 或 `wake/fork/exec` 选核时，判断自己是否应该参与均衡，然后找最忙的 `sched_group`、最忙的 `rq`，再从源 `runqueue` 拉任务过来

```c
apic_timer_interrupt()
-> smp_apic_timer_interrupt()
   -> local_apic_timer_interrupt()
      -> hrtimer_interrupt()
         -> __run_hrtimer()
               //kernel/time/tick-sched.c
            -> tick_nohz_handler()
               -> tick_sched_handle()
                     //kernel/time/timer.c
                  -> update_process_times(user_mode(regs))

//kernel/time/tick-common.c
tick_handle_periodic()
-> tick_periodic(cpu)
      //kernel/time/timer.c
   -> update_process_times(user_mode(get_irq_regs()))
```

```c
//kernel/time/timer.c
update_process_times()
   //kernel/sched/core.c
-> sched_tick()
      rq->idle_balance = idle_cpu(cpu)
      //kernel/sched/fair.c
   -> sched_balance_trigger(rq)
         //周期均衡不是每个 tick 都执行，rq->next_balance 是后面 sched_balance_domains()
         //根据各层 domain 的最早下次均衡时间更新出来的
         if (time_after_eq(jiffies, rq->next_balance))
         -> raise_softirq(SCHED_SOFTIRQ) //kick SCHED_SOFTIRQ 软中断，调用回调 sched_balance_softirq()
      -> nohz_balancer_kick(rq)
```

### `nohz_balance_exit_idle()`：CPU 退出空闲状态
* `nohz_balance_exit_idle(struct rq *rq)` 是 CPU 从空闲状态退出的“出口”函数，它会执行一系列清理工作
```cpp
void nohz_balance_exit_idle(struct rq *rq)
{
        WARN_ON_ONCE(rq != this_rq());
        //如果为 0，说明 CPU 的 tick 没有被停止（即并未处于 NOHZ 空闲状态），函数直接返回
        if (likely(!rq->nohz_tick_stopped))
                return;
        //置为 0，标志着该 CPU 不再处于 tick 停止的空闲状态
        rq->nohz_tick_stopped = 0;
        //将当前 CPU 从全局的记录了系统中所有处于 NOHZ 空闲状态的 CPU 掩码中清除
        cpumask_clear_cpu(rq->cpu, nohz.idle_cpus_mask);
        //将当前 CPU 所在的 LLC 调度域标记为繁忙
        set_cpu_sd_state_busy(rq->cpu);
}
```
* 调用的场景：
```c
kernel/sched/core.c|8673| <<sched_cpu_deactivate>> nohz_balance_exit_idle(rq);
kernel/sched/fair.c|12758| <<nohz_balancer_kick>> nohz_balance_exit_idle(rq);
```
* **场景一**：CPU 从空闲中被唤醒
  * 当一个处于 NOHZ 空闲状态的 CPU 被新任务唤醒时，`nohz_balance_exit_idle()` 会被调用，以更新其状态
* **场景二**：CPU 被热拔出（Hot-removal）
  * 在 CPU 热插拔的销毁路径上，为了将该 CPU 从负载均衡的候选集中移除，也会调用此函数
* **反操作**：当 CPU 进入 NOHZ 空闲时，会调用 `nohz_balance_enter_idle()`，它最终会调用 `set_cpu_sd_state_idle()`，将 `sd->nohz_idle` 置 `1`，并递减 `sd->shared->nr_busy_cpus`
* `nohz.idle_cpus_mask` 是一个全局的 CPU 位图，让调度器能快速、简单地知道“哪些 CPU 的 tick 是停止的”
* `sd->nohz_idle` 和 `sd->shared->nr_busy_cpus` 是调度域级别的精细信息。它让调度器在决定是否对某个 LLC 域进行负载均衡时，能评估该域的整体繁忙程度，从而做出更精准的决策

#### `set_cpu_sd_state_busy()`：标记调度域为“繁忙”
* `set_cpu_sd_state_busy(int cpu)` 将一个 CPU 所在的 **最后一级缓存（LLC）调度域** 标记为“繁忙”状态
```cpp
static void set_cpu_sd_state_busy(int cpu)
{
        struct sched_domain *sd;
        
        rcu_read_lock();
        sd = rcu_dereference_all(per_cpu(sd_llc, cpu)); //获取该 CPU 对应的 LLC 调度域
        //检查该域是否被标记为空闲
        if (!sd || !sd->nohz_idle)
                goto unlock; //如果已经处于繁忙状态了，跳过
        sd->nohz_idle = 0;   //如果处于空闲状态，则将其清除，表示该域不再处于完全空闲状态
        //这个计数器反映了该 LLC 域中繁忙 CPU 的数量，是负载均衡决策的重要依据，递增它
        atomic_inc(&sd->shared->nr_busy_cpus);
unlock:
        rcu_read_unlock();
}
```

### 找到 NOHZ 空闲负载均衡（ILB）

#### “真正空闲”判断 `idle_rq()`
* `idle_rq(struct rq *rq)` 判断一个运行队列是否处于绝对空闲状态，三个条件缺一不可：
1. `rq->curr == rq->idle`：当前正在执行的进程是 idle 进程（`PID=0`）
2. `!rq->nr_running`：运行队列中没有任何可运行的任务
3. `!rq->ttwu_pending`：没有待处理的“远程唤醒”（Try To Wake Up, TTWU）中断
* kernel/sched/sched.h
```cpp
static inline bool idle_rq(struct rq *rq)
{
        return rq->curr == rq->idle && !rq->nr_running && !rq->ttwu_pending;
}
```
##### 为什么需要判断 `rq->ttwu_pending`？
* 当一个任务在其他 CPU 上被唤醒（`try_to_wake_up()`）时，该任务可能被排入当前 CPU 的队列
* 但唤醒操作是通过 IPI（处理器间中断） 异步进行的。如果在 IPI 还未到达并处理 `ttwu_queue` 之前，我们仅检查 `nr_running`，会误以为 CPU 空闲，从而错误地将其选为 ILB 或让该 CPU 进入更深度的休眠
* `ttwu_pending` 标志保证了即使任务还没真正入队，我们也知道它“即将到来”，因此不认为 CPU 是空闲的
#### `idle_cpu()`：CPU 当前是空闲的吗？
* 返回 `1` 表示 CPU 确实空闲，`0` 表示繁忙或有任务待处理
* kernel/sched/syscalls.c
```cpp
/**
 * idle_cpu - is a given CPU idle currently?
 * @cpu: the processor in question.
 *
 * Return: 1 if the CPU is currently idle. 0 otherwise.
 */
int idle_cpu(int cpu)
{
        return idle_rq(cpu_rq(cpu));
}
```
#### `find_new_ilb()`：选举空闲负载均衡器（ILB）
* `find_new_ilb(void)` 的目标是：从所有空闲的 CPU 中，选出一个来担任“代理均衡器”，替繁忙的 CPU 执行 NOHZ 均衡任务
* NOHZ 空闲负载均衡（ILB）细节：
  * 当某个繁忙 CPU 检测到可能需要执行空闲重均衡时，会触发空闲负载均衡器，由该均衡器为所有空闲 CPU 执行空闲负载均衡
```cpp
/*
 * NOHZ idle load balancing (ILB) details:
 *
 * - When one of the busy CPUs notices that there may be an idle rebalancing
 *   needed, they will kick the idle load balancer, which then does idle
 *   load balancing for all the idle CPUs.
 */
static inline int find_new_ilb(void)
{       //当前正在运行的 CPU（通常是时钟中断触发了软中断的繁忙 CPU）
        int this_cpu = smp_processor_id();
        const struct cpumask *hk_mask;
        int ilb_cpu;
        //在开启了 CONFIG_NO_HZ_FULL 的系统中，某些 CPU 被隔离用于用户态低延迟任务
        //（HK_TYPE_KERNEL_NOISE），它们不允许被用作 ILB，必须排除在 housekeeping 之外
        hk_mask = housekeeping_cpumask(HK_TYPE_KERNEL_NOISE); //获取“内务处理（Housekeeping）” 掩码
        //遍历所有 tick 已停止的空闲 CPU 与允许做内核工作的 CPU（housekeeping）的交集
        for_each_cpu_and(ilb_cpu, nohz.idle_cpus_mask, hk_mask) {
                //当前 CPU 是因为繁忙（有 tick）才触发查找 ILB 的，绝不能让自己去充当 ILB，
                //否则会打破“让繁忙 CPU 专注做任务”的设计初衷，且容易产生递归均衡
                if (ilb_cpu == this_cpu)
                        continue;
                //虽然 CPU 在 nohz.idle_cpus_mask 中，但在遍历瞬间它可能刚好被唤醒（退出了空闲状态）
                //这里调用 idle_cpu() 做最终实时校验，只有确认它现在依然真的空闲，才返回其编号
                if (idle_cpu(ilb_cpu))
                        return ilb_cpu;
        }     

        return -1;
}
```
### NOHZ 空闲负载均衡的跨核唤醒
* `nohz_flags(cpu)` 是一个 per CPU 的原子位掩码（`atomic_t`），用于记录该 CPU 上待处理的 NOHZ 均衡请求类型（如普通均衡 `NOHZ_BALANCE_KICK`、统计更新 `NOHZ_STATS_KICK` 等）
  * 它充当了一个无锁工作队列，用于在多个繁忙 CPU 同时想唤醒同一个空闲 CPU 时进行仲裁
  * kernel/sched/sched.h
```cpp
#define nohz_flags(cpu)         (&cpu_rq(cpu)->nohz_flags)
```
* `rq->nohz_csd`：一个 `call_single_data` 结构体，通过 `INIT_CSD()` 初始化为绑定 `nohz_csd_func` 回调。它是 SMP 异步 IPI 的载体
```cpp
void __init sched_init(void)
{
...
                 INIT_CSD(&rq->nohz_csd, nohz_csd_func, rq);
...
}
```

#### 发起端：`kick_ilb()` 踢醒空闲 CPU
* 若已到执行时机，则通过跨 CPU 的 SMP 函数调用（IPI）唤醒一个 CPU 来执行 NOHZ 负载均衡
* 我们选取 `HK_TYPE_KERNEL_NOISE` 管家集合中的第一个空闲 CPU（如果存在）
  * kernel/sched/fair.c
```cpp
/*
 * Kick a CPU to do the NOHZ balancing, if it is time for it, via a cross-CPU
 * SMP function call (IPI).
 *
 * We pick the first idle CPU in the HK_TYPE_KERNEL_NOISE housekeeping set
 * (if there is one).
 */
static void kick_ilb(unsigned int flags)
{
        int ilb_cpu;
        //仅当触发完整 ILB 时才递增 nohz.next_balance，仅更新统计信息时不做该递增
        /*
         * Increase nohz.next_balance only when if full ilb is triggered but
         * not if we only update stats.
         */
        if (flags & NOHZ_BALANCE_KICK)
                nohz.next_balance = jiffies+1;
        //选举 ILB（空闲负载均衡器）CPU
        ilb_cpu = find_new_ilb();
        if (ilb_cpu < 0)
                return;
        //若 ilb_cpu 不存在新的 NOHZ 均衡工作项，例如 flags 中的所有位在 ilb_cpu 上均已置位，则无需处理
        /*
         * Don't bother if no new NOHZ balance work items for ilb_cpu,
         * i.e. all bits in flags are already set in ilb_cpu.
         */     
        if ((atomic_read(nohz_flags(ilb_cpu)) & flags) == flags)
                return; 
        //对 rq::nohz_csd 的访问由 NOHZ_KICK_MASK 做串行保护；第一个置位标志的调用方持有该对象；
        //由 nohz_csd_func () 完成清除操作
        /*
         * Access to rq::nohz_csd is serialized by NOHZ_KICK_MASK; he who sets
         * the first flag owns it; cleared by nohz_csd_func().
         */
        flags = atomic_fetch_or(flags, nohz_flags(ilb_cpu));
        //如果返回的旧值中已经含有 KICK 标志，说明目标 CPU 上已有未处理的 IPI 在排队
        //为了避免 IPI 风暴，直接放弃本次发送
        if (flags & NOHZ_KICK_MASK)
                return;
        //通过该方式，我们向处于空闲状态的目标 CPU 发送 IPI，执行 NOHZ 空闲负载均衡的软中断会在 IPI 返回前完成运行
        /*
         * This way we generate an IPI on the target CPU which
         * is idle, and the softirq performing NOHZ idle load balancing
         * will be run before returning from the IPI.
         */
        smp_call_function_single_async(ilb_cpu, &cpu_rq(ilb_cpu)->nohz_csd);
}
```
* 使用 `atomic_fetch_or()` 做“分布式锁”：
  * 多个繁忙 CPU 可能同时选中同一个空闲 CPU
  * 通过原子操作，**第一个成功设置 `NOHZ_KICK_MASK` 的 CPU** 获得发送 IPI 的权利，其他 CPU 看到旧值中已有 `NOHZ_KICK_MASK` 后直接返回
  * 这避免了多个 IPI 涌向同一个空闲 CPU
* 异步发送（`_async` 后缀）CSD：繁忙 CPU 不需要等待空闲 CPU 处理完均衡再返回，避免了繁忙 CPU 因 IPI 响应而陷入阻塞延迟

#### 接收端：`nohz_csd_func()` 空闲 CPU 被唤醒后做什么
```cpp
static void nohz_csd_func(void *info)
{
        struct rq *rq = info; 
        int cpu = cpu_of(rq);
        unsigned int flags;
        //清除本 CPU 的 nohz_flags 中的 KICK 位（表示“正在处理”）
        /*
         * Release the rq::nohz_csd.
         */
        flags = atomic_fetch_andnot(NOHZ_KICK_MASK | NOHZ_NEWILB_KICK, nohz_flags(cpu));
        WARN_ON(!(flags & NOHZ_KICK_MASK));
        //再次检查当前 CPU 是否真的还处于空闲状态
        rq->idle_balance = idle_cpu(cpu);
        if (rq->idle_balance) {
                rq->nohz_idle_balance = flags; //如果确实仍空闲，将 flags 暂存到 rq 中
                __raise_softirq_irqoff(SCHED_SOFTIRQ); //然后触发调度软中断
        }
}
```
* 为什么必须二次检查 `idle_cpu`？
  * 从繁忙 CPU 调用 `find_new_ilb()` 选出 `ilb_cpu`，到它发送 IPI，再到 IPI 真正抵达目标 CPU，这之间存在时间窗口
  * 在这个窗口内，目标 CPU 可能已经被其他任务唤醒（`ttwu_pending` 或 `nr_running` 增加）
  * 如果目标 CPU 不再空闲，则绝不能在当前上下文中触发 `SCHED_SOFTIRQ`，否则会在一个正在繁忙执行任务的 CPU 上强行插入负载均衡软中断，既浪费算力，也破坏调度公平
* 内存屏障隐含：`atomic_fetch_andnot` 和 `atomic_fetch_or` 本身包含完整的内存屏障（full barrier），确保了 `nohz_flags` 的修改与 `rq` 其他状态的可见性顺序，是多核并发下的安全保障
```cpp
/*      
 * Check whether the capacity of the rq has been noticeably reduced by side
 * activity. The imbalance_pct is used for the threshold.
 * Return true is the capacity is reduced
 */
static inline int
check_cpu_capacity(struct rq *rq, struct sched_domain *sd)
{
        return ((rq->cpu_capacity * sd->imbalance_pct) <
                                (arch_scale_cpu_capacity(cpu_of(rq)) * 100));
}

/* Check if the rq has a misfit task */
static inline bool check_misfit_status(struct rq *rq)
{
        return rq->misfit_task_load;
}
```
```c
/*
 * Current decision point for kicking the idle load balancer in the presence
 * of idle CPUs in the system.
 */
static void nohz_balancer_kick(struct rq *rq)
{
        unsigned long now = jiffies;
        struct sched_domain_shared *sds;
        struct sched_domain *sd;
        int nr_busy, i, cpu = rq->cpu;
        unsigned int flags = 0;

        if (unlikely(rq->idle_balance))
                return;

        /*
         * We may be recently in ticked or tickless idle mode. At the first
         * busy tick after returning from idle, we will update the busy stats.
         */
        nohz_balance_exit_idle(rq);

        if (READ_ONCE(nohz.has_blocked_load) &&
            time_after(now, READ_ONCE(nohz.next_blocked)))
                flags = NOHZ_STATS_KICK;

        /*
         * Most of the time system is not 100% busy. i.e nohz.nr_cpus > 0
         * Skip the read if time is not due.
         *
         * If none are in tickless mode, there maybe a narrow window
         * (28 jiffies, HZ=1000) where flags maybe set and kick_ilb called.
         * But idle load balancing is not done as find_new_ilb fails.
         * That's very rare. So read nohz.nr_cpus only if time is due.
         */
        if (time_before(now, nohz.next_balance))
                goto out;

        /*
         * None are in tickless mode and hence no need for NOHZ idle load
         * balancing
         */
        if (unlikely(cpumask_empty(nohz.idle_cpus_mask)))
                return;

        if (rq->nr_running >= 2) {
                flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                goto out;
        }

        rcu_read_lock();

        sd = rcu_dereference_all(rq->sd);
        if (sd) {
                /*
                 * If there's a runnable CFS task and the current CPU has reduced
                 * capacity, kick the ILB to see if there's a better CPU to run on:
                 */
                if (rq->cfs.h_nr_runnable >= 1 && check_cpu_capacity(rq, sd)) {
                        flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                        goto unlock;
                }
        }

        sd = rcu_dereference_all(per_cpu(sd_asym_packing, cpu));
        if (sd) {
                /*
                 * When ASYM_PACKING; see if there's a more preferred CPU
                 * currently idle; in which case, kick the ILB to move tasks
                 * around.
                 *
                 * When balancing between cores, all the SMT siblings of the
                 * preferred CPU must be idle.
                 */
                for_each_cpu_and(i, sched_domain_span(sd), nohz.idle_cpus_mask) {
                        if (sched_asym(sd, i, cpu)) {
                                flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                                goto unlock;
                        }
                }
        }

        sd = rcu_dereference_all(per_cpu(sd_asym_cpucapacity, cpu));
        if (sd) {
                /*
                 * When ASYM_CPUCAPACITY; see if there's a higher capacity CPU
                 * to run the misfit task on.
                 */
                if (check_misfit_status(rq)) {
                        flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                        goto unlock;
                }

                /*
                 * For asymmetric systems, we do not want to nicely balance
                 * cache use, instead we want to embrace asymmetry and only
                 * ensure tasks have enough CPU capacity.
                 *
                 * Skip the LLC logic because it's not relevant in that case.
                 */
                goto unlock;
        }

        sds = rcu_dereference_all(per_cpu(sd_llc_shared, cpu));
        if (sds) {
                /*
                 * If there is an imbalance between LLC domains (IOW we could
                 * increase the overall cache utilization), we need a less-loaded LLC
                 * domain to pull some load from. Likewise, we may need to spread
                 * load within the current LLC domain (e.g. packed SMT cores but
                 * other CPUs are idle). We can't really know from here how busy
                 * the others are - so just get a NOHZ balance going if it looks
                 * like this LLC domain has tasks we could move.
                 */
                nr_busy = atomic_read(&sds->nr_busy_cpus);
                if (nr_busy > 1) {
                        flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                        goto unlock;
                }
        }
unlock:
        rcu_read_unlock();
out:
        if (READ_ONCE(nohz.needs_update))
                flags |= NOHZ_NEXT_KICK;

        if (flags)
                kick_ilb(flags);
}
```

## `SCHED_SOFTIRQ` 软中断处理函数 `sched_balance_softirq()`
* `SCHED_SOFTIRQ` 软中断在两个地方触发：
1. 为做周期性的负载均衡由直接由本地的 `sched_tick()` 触发
2. 为做 NOHZ idle 均衡通过 SMP cross-call `nohz_csd_func()` 间接由远程的 `sched_tick()` 触发
* 如果该 CPU 有一个挂起的 `NOHZ_BALANCE_KICK`，那么代替其他 idle CPU 做负载均衡，因为它们的 tick 已经停止了
  * 在 `sched_balance_domains()` 之前做 `nohz_idle_balance()`，给 idle CPU 一个做负载均衡的机会
  * 否则，我们可能只在本地调度域层级中做负载均衡，且一旦拉取到部分负载，就会直接终止 `nohz_idle_balance()`
```cpp
/*
 * This softirq handler is triggered via SCHED_SOFTIRQ from two places:
 *
 * - directly from the local sched_tick() for periodic load balancing
 *
 * - indirectly from a remote sched_tick() for NOHZ idle balancing
 *   through the SMP cross-call nohz_csd_func()
 */
static __latent_entropy void sched_balance_softirq(void)
{
        struct rq *this_rq = this_rq();
        enum cpu_idle_type idle = this_rq->idle_balance;
        /*    
         * If this CPU has a pending NOHZ_BALANCE_KICK, then do the
         * balancing on behalf of the other idle CPUs whose ticks are
         * stopped. Do nohz_idle_balance *before* sched_balance_domains to
         * give the idle CPUs a chance to load balance. Else we may
         * load balance only within the local sched_domain hierarchy
         * and abort nohz_idle_balance altogether if we pull some load.
         */
        if (nohz_idle_balance(this_rq, idle))
                return;
        //常规负载均衡
        /* normal load balance */
        sched_balance_update_blocked_averages(this_rq->cpu);
        sched_balance_domains(this_rq, idle);
}
```

## 负载均衡统计
* 需要开启内核选项 `CONFIG_SCHEDSTATS` 开启负载均衡统计计数功能
  * **注意**：开启负载均衡统计会带来少量的额外性能开销
* 通过 kernel command line `schedstats=enable`（`setup_schedstats()`）或者设置以下 `sysctl` 参数（`sysctl_schedstats()`）开启负载均衡统
```sh
echo 1 > /proc/sys/kernel/sched_schedstats
```
* 开启负载均衡统计后可以从 `/proc/schedstat` 文件中读到负载均衡的相关统计计数
* 开启 `CONFIG_LATENCYTOP` 选项可以使用 LatencyTOP 工具，用于找出哪些哪些用户空间程序正阻塞在什么内核操作上
  * 它会创建 `/proc/latency_stats` 文件和注册 `/proc/sys/kernel/latencytop` 接口
  * `echo 1 > /proc/sys/kernel/latencytop` 会导致强制开启负载均衡统计（`force_schedstat_enabled()`）
### 统计计数相关代码

#### 控制统计计数相关代码
* 控制相关的代码如下，主要是通过修改 static key `sched_schedstats` 来控制
  * kernel/sched/core.c
```cpp
#ifdef CONFIG_SCHEDSTATS
//sched_schedstats static key
DEFINE_STATIC_KEY_FALSE(sched_schedstats);
//修改 sched_schedstats static key 的统一接口
static void set_schedstats(bool enabled)
{
        if (enabled)
                static_branch_enable(&sched_schedstats);
        else
                static_branch_disable(&sched_schedstats);
}
//开启 /proc/sys/kernel/latencytop 会用到的强制开启负载均衡统计函数
void force_schedstat_enabled(void)
{
        if (!schedstat_enabled()) {
                pr_info("kernel profiling enabled schedstats, disable via kernel.sched_schedstats.\n");
                static_branch_enable(&sched_schedstats);
        }
}
//kernel command line 的 "schedstats=enable|disable" 控制接口
static int __init setup_schedstats(char *str)
{
        int ret = 0;
        if (!str)
                goto out;

        if (!strcmp(str, "enable")) {
                set_schedstats(true);
                ret = 1;
        } else if (!strcmp(str, "disable")) {
                set_schedstats(false);
                ret = 1;
        }
out:
        if (!ret)
                pr_warn("Unable to parse schedstats=\n");

        return ret;
}
__setup("schedstats=", setup_schedstats);
//sysctl 的 /proc/sys/kernel/sched_schedstats 控制接口
#ifdef CONFIG_PROC_SYSCTL
static int sysctl_schedstats(const struct ctl_table *table, int write, void *buffer,
                size_t *lenp, loff_t *ppos)
{
        struct ctl_table t;
        int err;
        int state = static_branch_likely(&sched_schedstats);

        if (write && !capable(CAP_SYS_ADMIN))
                return -EPERM;

        t = *table;
        t.data = &state;
        err = proc_dointvec_minmax(&t, write, buffer, lenp, ppos);
        if (err < 0)
                return err;
        if (write)
                set_schedstats(state);
        return err;
}
#endif /* CONFIG_PROC_SYSCTL */
#endif /* CONFIG_SCHEDSTATS */
```
#### 修改统计计数相关宏
* 可见如果不开启统计计数的功能，修改统计计数不会有效果
  * kernel/sched/stats.h
```cpp
#define   schedstat_enabled()           static_branch_unlikely(&sched_schedstats)
#define __schedstat_inc(var)            do { var++; } while (0)
#define   schedstat_inc(var)            do { if (schedstat_enabled()) { var++; } } while (0)
#define __schedstat_add(var, amt)       do { var += (amt); } while (0)
#define   schedstat_add(var, amt)       do { if (schedstat_enabled()) { var += (amt); } } while (0)
#define __schedstat_set(var, val)       do { var = (val); } while (0)
#define   schedstat_set(var, val)       do { if (schedstat_enabled()) { var = (val); } } while (0)
#define   schedstat_val(var)            (var)
#define   schedstat_val_or_zero(var)    ((schedstat_enabled()) ? (var) : 0)
```
#### 显示负载均衡统计计数
* 通过 `/proc/schedstat` 文件中读负载均衡的相关统计计数的代码如下：
  * include/linux/sched/idle.h
```cpp
enum cpu_idle_type {
        __CPU_NOT_IDLE = 0,
        CPU_IDLE,
        CPU_NEWLY_IDLE,
        CPU_MAX_IDLE_TYPES
};
```
  * kernel/sched/stats.c
```cpp
/*
 * Current schedstat API version.
 *
 * Bump this up when changing the output format or the meaning of an existing
 * format, so that tools can adapt (or abort)
 */
#define SCHEDSTAT_VERSION 17

static int show_schedstat(struct seq_file *seq, void *v)
{
        int cpu;

        if (v == (void *)1) {
                seq_printf(seq, "version %d\n", SCHEDSTAT_VERSION);
                seq_printf(seq, "timestamp %lu\n", jiffies);
        } else {
                struct rq *rq;
                struct sched_domain *sd;
                int dcount = 0;
                cpu = (unsigned long)(v - 2);
                rq = cpu_rq(cpu);

                /* runqueue-specific stats */
                seq_printf(seq,
                    "cpu%d %u 0 %u %u %u %u %llu %llu %lu",
                    cpu, rq->yld_count,
                    rq->sched_count, rq->sched_goidle,
                    rq->ttwu_count, rq->ttwu_local,
                    rq->rq_cpu_time,
                    rq->rq_sched_info.run_delay, rq->rq_sched_info.pcount);

                seq_printf(seq, "\n");

                /* domain-specific stats */
                rcu_read_lock();
                for_each_domain(cpu, sd) {
                        enum cpu_idle_type itype;

                        seq_printf(seq, "domain%d %s %*pb", dcount++, sd->name,
                                   cpumask_pr_args(sched_domain_span(sd)));
                        for (itype = 0; itype < CPU_MAX_IDLE_TYPES; itype++) {
                                seq_printf(seq, " %u %u %u %u %u %u %u %u %u %u %u",
                                    sd->lb_count[itype],
                                    sd->lb_balanced[itype],
                                    sd->lb_failed[itype],
                                    sd->lb_imbalance_load[itype],
                                    sd->lb_imbalance_util[itype],
                                    sd->lb_imbalance_task[itype],
                                    sd->lb_imbalance_misfit[itype],
                                    sd->lb_gained[itype],
                                    sd->lb_hot_gained[itype],
                                    sd->lb_nobusyq[itype],
                                    sd->lb_nobusyg[itype]);
                        }
                        seq_printf(seq,
                                   " %u %u %u %u %u %u %u %u %u %u %u %u\n",
                            sd->alb_count, sd->alb_failed, sd->alb_pushed,
                            sd->sbe_count, sd->sbe_balanced, sd->sbe_pushed,
                            sd->sbf_count, sd->sbf_balanced, sd->sbf_pushed,
                            sd->ttwu_wake_remote, sd->ttwu_move_affine,
                            sd->ttwu_move_balance);
                }
                rcu_read_unlock();
        }
        return 0;
}
...
static const struct seq_operations schedstat_sops = {
        .start = schedstat_start,
        .next  = schedstat_next,
        .stop  = schedstat_stop,
        .show  = show_schedstat,
};

static int __init proc_schedstat_init(void)
{
        proc_create_seq("schedstat", 0, NULL, &schedstat_sops);
        return 0;
}
subsys_initcall(proc_schedstat_init);
```

## 调节负载均衡参数
* 开启调度器的详细模式 `sched_verbose_write()`
  ```sh
  echo 1 > /sys/kernel/debug/sched/verbose
  ```
* `update_sched_domain_debugfs()` 在 `debugfs` 建立负载均衡控制项，例如
```sh
/sys/kernel/debug/sched/domains
├── cpu0
│   ├── domain0
│   │   ├── busy_factor
│   |   ├── cache_nice_tries
│   |   ├── flags
│   |   ├── groups_flags
│   |   ├── imbalance_pct
│   |   ├── level
│   |   ├── max_interval
│   |   ├── max_newidle_lb_cost
│   |   ├── min_interval
│   |   └── name
│   └── domain1
├── cpu1
│   ├── domain0
│   └── domain1
├── cpu2
│   ├── domain0
│   └── domain1
├── cpu3
│   ├── domain0
│   └── domain1
└── cpu4
    ├── domain0
    └── domain1
```
