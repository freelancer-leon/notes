# CPU 负载均衡的发起

* 负载均衡主要是 CFS 的 pull 模型：目标 CPU 在 tick、即将 idle、NOHZ idle 或 `wake/fork/exec` 选核时，判断自己是否应该参与均衡，然后找最忙的 `sched_group`、最忙的 `rq`，再从源 `runqueue` 拉任务过来
* NOHZ 更新时间
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
```
* 周期性更新时间
```c
//kernel/time/tick-common.c
tick_setup_device()
   if (tick_device_uses_broadcast(newdev, cpu))
      -> dev->event_handler = tick_handle_periodic;
      return;
   if (td->mode == TICKDEV_MODE_PERIODIC)
      tick_setup_periodic(newdev, 0);
         //kernel/time/tick-broadcast.c
      -> tick_set_periodic_handler(dev, broadcast)
            if (!broadcast)
               dev->event_handler = tick_handle_periodic;
            else
               dev->event_handler = tick_handle_periodic_broadcast;
   else
      tick_setup_oneshot(newdev, handler, next_event);

//kernel/time/tick-common.c
tick_handle_periodic()
-> tick_periodic(cpu)
      //kernel/time/timer.c
   -> update_process_times(user_mode(get_irq_regs()))
```
* NOHZ 和周期性 tick 触发负载均衡的路径
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

## NOHZ 空闲负载均衡
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

#### `idle_rq()`：判断运行队列“真正空闲”
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
        //在开启了 CONFIG_NO_HZ_FULL 的系统中，某些 CPU 被隔离用于用户态低延迟任务，它们不允许被用作 ILB，
        //必须排除在 housekeeping 之外
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
* “内务处理”（Housekeeping）CPU 是那些没有被隔离、可以执行各种常规内核工作的 CPU
  * `HK_TYPE_KERNEL_NOISE` 是内务处理 CPU 的一种类型，它代表了没有被配置为 `nohz_full` 的 CPU 集合，可以安全地执行“内核噪声”类工作

### `check_cpu_capacity()` 检查 CPU 算力是否“受损”
* `check_cpu_capacity()` 比较的是 **当前剩余可用算力** 与 **理论最大原生算力** 的比值
  * `arch_scale_cpu_capacity(cpu)`：该 CPU 的 **原生最大算力**（基于最高频率和微架构，通常归一化为 `1024`）
  * `rq->cpu_capacity`：该 CPU **当前实时可用算力**。它会因为以下 “侧活动（Side Activity）” 而动态衰减：
    * 运行了实时任务（RT/DL 类）抢占了大片时间片
    * 处理硬件中断（IRQ）或软中断
    * 遭遇温度限制（Thermal Throttling）导致降频
    * 被 cgroup 的 `cpu.max` 限制
  * `sd->imbalance_pct`：容忍阈值（通常为 `117`，即 `117%`）。用于防止算力因微小波动而频繁触发误判
* 数学逻辑与判断条件（换算成百分比）移项后等价于：
   ```c
  rq->cpu_capacity / arch_scale_cpu_capacity < 100 / sd->imbalance_pct
   ```
* 当 `imbalance_pct = 117` 时，`100 / 117 ≈ 85.4%`
* **结论**：如果 *当前可用算力* 低于 *原生最大算力* 的 `85%`，函数返回 `true`，认为该 CPU 的算力“明显受损”

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
```
##### 设计意图
* 在负载均衡中，如果某个 CPU 自身算力受损（比如被 RT 任务吃掉 `20%` 算力），调度器会放弃将新任务或重任务迁移给它，否则该任务会被“饿死”
* 同时，如果算力受损的 CPU 上正跑着任务，它也会主动将这些任务推给其他算力健全的 CPU

### `check_misfit_status()` 检查是否存在“不匹配”的任务
* 什么是 **“Misfit Task”（不匹配任务）**？
  * 在异构（大小核）架构中，小核（LITTLE / E-core）算力有限，大核（big / P-core）算力充足
  * 当一个任务的 PELT 平均利用率（`util_avg`）超过了当前 CPU 的最大算力（`cpu_capacity`）时，该任务就被标记为“**Misfit**”
  * 形象比喻：让小核（只能扛 400 算力）去运行一个需要 600 算力的重负载游戏，就是“小牛拉大车”，属于“不匹配”
* 谁来设置 `rq->misfit_task_load`？
  * 在 CFS 调度类的 `update_misfit_status()` 函数中，每当 **任务入队** 或 **进行 PELT 更新** 时，会检查当前 CPU 上运行的任务的 `util_avg`
  * 如果它大于该 CPU 的 `cpu_capacity`，`rq->misfit_task_load` 就被赋值为该任务的负载值，否则为 `0`
#### 设计意图：
* 一旦某个 CPU 上存在“Misfit Task”，调度器必须在负载均衡时采取紧急行动（Active Balance），将该任务迁移到算力更高的大核上去，以确保任务的性能（避免严重掉帧或延迟）
* 这是智能手机和现代 PC 混合架构保证流畅度的核心机制
```cpp
/* Check if the rq has a misfit task */
static inline bool check_misfit_status(struct rq *rq)
{
        return rq->misfit_task_load;
}
```

### 更新不匹配任务的状态
#### 前置工具函数：如何衡量“算力损耗”与“是否合适”
* **`hw_load_avg()`** 读取硬件（如 Intel HWP 或 ARM DSU）反馈的“平均硬件负载”。如果硬件认为该 CPU 长期处于高负载，会主动限制最大频率
  * kernel/sched/pelt.h
```cpp
static inline u64 hw_load_avg(struct rq *rq)
{
    return READ_ONCE(rq->avg_hw.load_avg);
}
```
* **`cpufreq_get_pressure()`** 读取因 DVFS（动态电压频率调整）或温度限制（Thermal）导致的性能压力值
  * include/linux/cpufreq.h
```cpp
DECLARE_PER_CPU(unsigned long, cpufreq_pressure);
static inline unsigned long cpufreq_get_pressure(int cpu) 
{
    return READ_ONCE(per_cpu(cpufreq_pressure, cpu));
}
```
* **实际可用算力** = 原生最大算力 - 两者中的较大者
  * 这意味着只要硬件降频或过热，`get_actual_cpu_capacity()` 的值会明显变小
  * kernel/sched/fair.c
```cpp
static inline unsigned long get_actual_cpu_capacity(int cpu)
{
    unsigned long capacity = arch_scale_cpu_capacity(cpu);

    capacity -= max(hw_load_avg(cpu_rq(cpu)), cpufreq_get_pressure(cpu));

    return capacity;
}
```
* `fits_capacity()` 留有余地的“合身”判断
  * 移项得：`cap < max * 0.8`
  * 即：只有当当前利用率（`cap`）低于 CPU 最大算力（`max`）的 `80%` 时，才认为这个任务“合身”
  * 这留出了 `20%` 的缓冲余量，避免任务刚一放上去就因为微小的算力波动而触发迁移
```cpp
/*
 * The margin used when comparing utilization with CPU capacity.
 *
 * (default: ~20%)
 */
#define fits_capacity(cap, max) ((cap) * 1280 < (max) * 1024)
```

#### `util_fits_cpu()` 判断给定利用率是否“适合”在指定CPU上运行
* 该函数是调度器“任务-CPU适配性”检查的核心，主要服务于：
   - 负载均衡中判断是否应将任务迁移到该 CPU（特别是异构大小核系统）
   - 更新 `misfit` 状态时判断任务是否需要紧急搬移
* 核心设计思想：
1. 基础适配性：使用 `fits_capacity()` 宏检查原始 `util` 是否低于 CPU 容量（`capacity_of(cpu)`）的 `80%`，预留 `20%` 余量以避免频繁抖动
2. UClamp 影响：
   * `uclamp_max`：若任务被“上限”钳制（即不允许超过某个性能点），则只要该上限值不大于 **CPU 原生算力**（`arch_scale_cpu_capacity()`），就认为适配，忽略实际压力。因为设置了上限意味着任务不想跑太快，可以接受性能损失
   * `uclamp_min`：若任务被“下限”提升（即必须达到最低性能），则必须保证 CPU 在当前硬件 / CPUFreq 压力下的 **实际可用算力**（`get_actual_cpu_capacity()`）不低于 `uclamp_min`，否则返回 `-1`（坚决拒绝）
3. 区分“原生算力”与“当前实际可用算力”：
   - `capacity_of()` 包含因 RT/IRQ 等“软压力”导致的衰减，用于判断普通 `util` 是否适配
   - `arch_scale_cpu_capacity()` 是 **硬件原生最大算力**（不受软压力影响），用于 UClamp 比较，因为 UClamp 是用户对性能的硬性需求，不应因临时软压力而缩水
   - `get_actual_cpu_capacity()` 进一步 **减去硬件/CPUFreq 压力**，用于 `uclamp_min` 的强制检查，因为这些压力直接导致 OPP下降，影响实际性能天花板
```cpp
/*
 * @util:       任务的 PELT 平均利用率（原始值，未经 UClamp 修正）
 * @uclamp_min: 任务的 UClamp 最小性能要求（[0..SCHED_CAPACITY_SCALE]）
 * @uclamp_max: 任务的 UClamp 最大性能限制（[0..SCHED_CAPACITY_SCALE]）
 * @cpu:        目标 CPU 编号
 *
 * 返回值：
 *   1  — 适合：任务可以在该 CPU 上良好运行
 *   0  — 不适合：任务在该 CPU 上会超载或无法满足需求
 *  -1  — 特别不适合：由于 UClamp_min 要求高于该 CPU 的实际可用算力（考虑硬件/CPUFreq压力），
 *        即使任务当前利用率很低，也必须拒绝该 CPU
 */
static inline int util_fits_cpu(unsigned long util, 
                unsigned long uclamp_min,
                unsigned long uclamp_max,
                int cpu)
{
    unsigned long capacity = capacity_of(cpu); //该 CPU 最大算力（受软压力影响）
    unsigned long capacity_orig;
    bool fits, uclamp_max_fits;
    //检查真实的利用率是否合适没有应用任何 uclamp 限制或提升能力下的 CPU 最大算力
    /*    
     * Check if the real util fits without any uclamp boost/cap applied.
     */
    fits = fits_capacity(util, capacity);
    //未启用 UClamp 时，直接使用基础判断
    if (!uclamp_is_used())
        return fits;
    //UClamp 启用后，需要区分“原生容量”和“当前可用容量”。对于 uclamp_min/max 的比较，使用
    //原生容量（不受软压力影响），因为软压力（如 RT 任务）是临时的，不应影响用户设定的性能要求
    /*    
     * We must use arch_scale_cpu_capacity() for comparing against uclamp_min and
     * uclamp_max. We only care about capacity pressure (by using
     * capacity_of()) for comparing against the real util.
     * 在与 uclamp_min、uclamp_max 做比对时，必须使用 arch_scale_cpu_capacity()。
     * 只有在和真实利用率做比对时，才需要通过 capacity_of() 考量算力压力。
     *
     * If a task is boosted to 1024 for example, we don't want a tiny
     * pressure to skew the check whether it fits a CPU or not.
     * 举例来说，如果一个任务被提升至 1024，我们不希望微小的算力压力干扰该任务是否适合运行在某颗 CPU 上的判断。
     *
     * Similarly if a task is capped to arch_scale_cpu_capacity(little_cpu), it
     * should fit a little cpu even if there's some pressure.
     * 同理，如果任务被限制为 arch_scale_cpu_capacity(little_cpu)，即便存在一定算力压力，该任务也应当可以运行在小核 CPU 上。
     *
     * Only exception is for HW or cpufreq pressure since it has a direct impact
     * on available OPP of the system.
     * 唯一例外是硬件压力或调频子系统压力，这类压力会直接影响系统可用的 OPP 性能档位。
     *
     * We honour it for uclamp_min only as a drop in performance level
     * could result in not getting the requested minimum performance level.
     * 仅针对 uclamp_min 我们会考虑该压力因素：性能档位下降，可能导致无法达到任务要求的最低性能等级。
     *
     * For uclamp_max, we can tolerate a drop in performance level as the
     * goal is to cap the task. So it's okay if it's getting less.
     * 对于 uclamp_max，可以容忍性能档位下降，因为该约束的目的是限制任务性能上限，实际得到的性能低于上限是可接受的。
     */
    capacity_orig = arch_scale_cpu_capacity(cpu);
    //处理 uclamp_max（上限钳制）
    //若任务被限制在某个性能点以下，则只要该上限不超过原生容量，就认为该 CPU 是“适配”的
    //（忽略软压力，因为软压力只会让它跑得更慢，但任务本身就允许慢）
    //特殊情况：若 CPU 已是最大容量（capacity_orig == SCHED_CAPACITY_SCALE）且 uclamp_max
    //也是最大，则没有限制意义，不应强制适配，保持原有 fits 不变
    /*
     * We want to force a task to fit a cpu as implied by uclamp_max.
     * But we do have some corner cases to cater for..
     * 我们希望依据 uclamp_max 的约束，强制让任务能够适配某颗 CPU。但仍需要处理一些边界场景。
     *
     *                                 C=z
     *   |                             ___
     *   |                  C=y       |   |
     *   |_ _ _ _ _ _ _ _ _ ___ _ _ _ | _ | _ _ _ _ _  uclamp_max
     *   |      C=x        |   |      |   |
     *   |      ___        |   |      |   |
     *   |     |   |       |   |      |   |    (util somewhere in this region)
     *   |     |   |       |   |      |   |
     *   |     |   |       |   |      |   |
     *   +----------------------------------------
     *         CPU0        CPU1       CPU2
     *
     *   In the above example if a task is capped to a specific performance
     *   point, y, then when:
     *   以上示例中，若任务被限制在特定性能点 y，则：
     *   * util = 80% of x then it does not fit on CPU0 and should migrate
     *     to CPU1
     *     当利用率为 x 的 80% 时，该任务无法在 CPU0 上运行，应当迁移至 CPU1
     *   * util = 80% of y then it is forced to fit on CPU1 to honour
     *     uclamp_max request.
     *     当利用率为 y 的 80% 时，为满足 uclamp_max 的配置要求，强制该任务适配 CPU1
     *  
     *   which is what we're enforcing here. A task always fits if
     *   uclamp_max <= capacity_orig. But when uclamp_max > capacity_orig,
     *   the normal upmigration rules should withhold still.
     *   这正是此处要实现的逻辑。当 uclamp_max <= capacity_orig 时，任务始终可以适配 CPU。
     *   但当 uclamp_max > capacity_orig 时，常规的向上迁移规则仍应当生效。
     * 
     *   Only exception is when we are on max capacity, then we need to be
     *   careful not to block overutilized state. This is so because:
     *   唯一的例外：当 CPU 处于最大算力状态时，需要注意不能阻塞超饱和状态。原因如下：
     *     1. There's no concept of capping at max_capacity! We can't go
     *        beyond this performance level anyway.
     *        最大算力下不存在限流的概念！系统本身就无法超越该性能等级。
     *     2. The system is being saturated when we're operating near
     *        max capacity, it doesn't make sense to block overutilized.
     *        系统运行在接近最大算力时已经处于饱和状态，阻塞超饱和状态没有实际意义。
     */
    uclamp_max_fits = (capacity_orig == SCHED_CAPACITY_SCALE) && (uclamp_max == SCHED_CAPACITY_SCALE);
    uclamp_max_fits = !uclamp_max_fits && (uclamp_max <= capacity_orig);
    fits = fits || uclamp_max_fits;
    //处理 uclamp_min（最低性能保障）
    //仅当任务被“boost”（即 util < uclamp_min）时需要额外检查
    //此时，必须确保即使存在硬件/CPUFreq 压力，实际可用算力仍能达到 uclamp_min
    //若不能，返回 -1 表示“坚决拒绝”，因为用户要求的最低性能无法保证
    //注意：这里使用 get_actual_cpu_capacity()，它减去了硬件负载和 CPUFreq 压力
    /*
     *
     *                                 C=z
     *   |                             ___       (region a, capped, util >= uclamp_max)
     *   |                  C=y       |   |
     *   |_ _ _ _ _ _ _ _ _ ___ _ _ _ | _ | _ _ _ _ _ uclamp_max
     *   |      C=x        |   |      |   |
     *   |      ___        |   |      |   |      (region b, uclamp_min <= util <= uclamp_max)
     *   |_ _ _|_ _|_ _ _ _| _ | _ _ _| _ | _ _ _ _ _ uclamp_min
     *   |     |   |       |   |      |   |
     *   |     |   |       |   |      |   |      (region c, boosted, util < uclamp_min)
     *   +----------------------------------------
     *         CPU0        CPU1       CPU2
     *
     * a) If util > uclamp_max, then we're capped, we don't care about
     *    actual fitness value here. We only care if uclamp_max fits
     *    capacity without taking margin/pressure into account.
     *    See comment above.
     *    若 util > uclamp_max，则任务处于限流状态，此处不关注实际适配度。
     *    仅判断 uclamp_max 是否适配 CPU 算力，不考虑余量与算力压力。
     *
     * b) If uclamp_min <= util <= uclamp_max, then the normal
     *    fits_capacity() rules apply. Except we need to ensure that we
     *    enforce we remain within uclamp_max, see comment above.
     *    若 uclamp_min ≤ util ≤ uclamp_max，则执行标准 fits_capacity() 判断逻辑。
     *    但必须保证任务运行不超出 uclamp_max 限制，参见上文注释。
     *
     * c) If util < uclamp_min, then we are boosted. Same as (b) but we
     *    need to take into account the boosted value fits the CPU without
     *    taking margin/pressure into account.
     *    若 util < uclamp_min，则任务处于提频增压状态。逻辑同 (b)，但需要基于增压后的
     *    数值判断是否适配 CPU，不考虑余量与算力压力。
     *
     * Cases (a) and (b) are handled in the 'fits' variable already. We
     * just need to consider an extra check for case (c) after ensuring we
     * handle the case uclamp_min > uclamp_max.
     * 场景 (a) 与 (b) 已由 fits 变量完成处理。只需在处理完 uclamp_min > uclamp_max 的
     * 边界条件后，额外增加对场景 (c) 的判断。
     */
    uclamp_min = min(uclamp_min, uclamp_max); //合法性修正：min 不能大于 max
    if (fits && (util < uclamp_min) &&
        (uclamp_min > get_actual_cpu_capacity(cpu)))
        return -1;
    //返回最终适配结果
    return fits;
}
```
* 注意：当 `fits && util < uclamp_min && uclamp_min < get_actual_cpu_capacity` 成立时，可以返回 `true`（合适），理由是：
  * CPU 不仅当下的空闲算力足够应付任务现状（`fits`），而且其物理硬件能力（扣除降频压力后）完全有能力兜住任务未来的最低性能需求（`uclamp_min`）
  * 既然既能保底，又能舒适，当然是最佳选择
  * 这个逻辑确保了 “提升（Boost）” 场景下的任务总能得到一块“不会因为未来负载突增而掉链子”的 CPU
* 注意：如果前面是因为 `fits = false`，`uclamp_max_fits = true` 而使 `fits = fits || uclamp_max_fits = true`，`fits && util < uclamp_min && uclamp_min < get_actual_cpu_capacity` 成立，这是一种什么样的场景？
  * 这是一种 “CPU 正被临时事务（软中断/RT任务）骚扰，但物理硬件本身完好无损，且任务被‘限流（Capped）’了” 的场景
  * CPU 因为临时噪声（软中断）显得很忙，但物理硬件过硬（没降频），且任务被严格限速（`uclamp_max` 较低）。虽然表面挤了点，但实际塞得下，且能保证任务的最低要求（`uclamp_min`），因此是合适的
  * 就是防止调度器在软压力（瞬时的 RT/IRQ）下过度反应，错误地将“被限流”的任务迁移走，从而节省功耗和迁移开销。只有当硬压力（降频）威胁到 `uclamp_min` 时，它才返回 `-1` 采取紧急行动
#### `update_misfit_status()` 更新不匹配任务的状态
```cpp
static inline int task_fits_cpu(struct task_struct *p, int cpu)
{
        unsigned long uclamp_min = uclamp_eff_value(p, UCLAMP_MIN);
        unsigned long uclamp_max = uclamp_eff_value(p, UCLAMP_MAX);
        unsigned long util = task_util_est(p);
        /*
         * Return true only if the cpu fully fits the task requirements, which
         * include the utilization but also the performance hints.
         */
        return (util_fits_cpu(util, uclamp_min, uclamp_max, cpu) > 0);
}

static inline void update_misfit_status(struct task_struct *p, struct rq *rq)
{
    int cpu = cpu_of(rq);
    //系统不是异构架构（没有大小核区分），不需要考虑任务匹配的问题
    if (!sched_asym_cpucap_active())
        return;
    //1. !p：没有正在运行的任务
    //2. p->nr_cpus_allowed == 1：该任务被绑核了（CPU亲和性限制），只能在这一个 CPU 上跑，根本无法迁移，报了也没用
    //3. arch_scale_cpu_capacity(cpu) == p->max_allowed_capacity：当前 CPU 已经是该任务允许的最高性能核（比如已经在大核上了），没必要再往上迁
    //4. task_fits_cpu(p, cpu)：调用 util_fits_cpu() 判定，如果返回 true，说明任务在当前 CPU 上跑得很舒服（利用率低于 80%），不需要挪窝
    /*    
     * Affinity allows us to go somewhere higher?  Or are we on biggest
     * available CPU already? Or do we fit into this CPU ?
     */
    if (!p || (p->nr_cpus_allowed == 1) || 
        (arch_scale_cpu_capacity(cpu) == p->max_allowed_capacity) ||
        task_fits_cpu(p, cpu)) {
        //认为当前没有“不匹配任务”
        rq->misfit_task_load = 0;
        return;
    }
    //能走到这里说明这个任务确实“小马拉大车”了
    /*    
     * Make sure that misfit_task_load will not be null even if
     * task_h_load() returns 0.
     */
    rq->misfit_task_load = max_t(unsigned long, task_h_load(p), 1);
}
```
* `task_h_load(p)`：计算该任务在层次化调度（cgroups）下的真实负载权重
* `max_t(unsigned long, task_h_load(p), 1)` 防止 `task_h_load()` 返回 `0`（虽然极少发生，但在极端轻载或特定 cgroup 配置下可能为 `0`）
  * 如果负载值为 `0`，`check_misfit_status()` 会认为没有 misfit（因为 `0` 被视为 `false`）
  * 这里强制设为 `1`，是为了“占位”，确保即使负载值为 `0`，Misfit 标志也能被正确识别，从而触发迁移

### NOHZ 空闲负载均衡的决策中心
* `nohz_balancer_kick()`：当系统中存在空闲 CPU 时，用于触发空闲负载均衡器的现行决策点
```cpp
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
        //如果当前 CPU 正在执行 idle_balance（即它本身就是被唤醒来干活的 ILB），
        //则直接返回，避免自己踢自己造成死循环
        if (unlikely(rq->idle_balance))
                return;
        //将当前 CPU 从 nohz.idle_cpus_mask 中移除（因为它此刻正在运行 tick，不算严格意义上的空闲 CPU 了
        /*
         * We may be recently in ticked or tickless idle mode. At the first
         * busy tick after returning from idle, we will update the busy stats.
         * 我们近期可能处于有 ticked 或 tickless 的空闲模式。从空闲状态退出后的第一个
         * 忙 tick 时刻，会更新忙状态统计信息。
         */
        nohz_balance_exit_idle(rq);
        //nohz.has_blocked_load：系统中有某些 CPU 的 PELT 阻塞负载已经积压太久，需要被清理
        //time_after(now, nohz.next_blocked)：检查是否已经到了清理这些阻塞负载的时间点
        if (READ_ONCE(nohz.has_blocked_load) &&
            time_after(now, READ_ONCE(nohz.next_blocked)))
                flags = NOHZ_STATS_KICK; //注意：此时只置位了统计更新标志，并没有置位负载均衡标志

        /*
         * Most of the time system is not 100% busy. i.e nohz.nr_cpus > 0
         * Skip the read if time is not due.
         * 大多数情况下系统不会处于 100% 繁忙状态，也就是 nohz.nr_cpus > 0。
         * 若尚未到达执行时机，则跳过本次读取操作
         * 
         * If none are in tickless mode, there maybe a narrow window
         * (28 jiffies, HZ=1000) where flags maybe set and kick_ilb called.
         * But idle load balancing is not done as find_new_ilb fails.
         * That's very rare. So read nohz.nr_cpus only if time is due.
         * 如果没有 CPU 处于 tickless 模式，会存在一个很短的时间窗口（28 个 jiffies，HZ=1000）：
         * 该场景下标志位可能被置位，同时调用 kick_ilb。
         * 但由于 find_new_ilb 执行失败，空闲负载均衡并不会实际执行。该情况发生概率极低。
         * 因此仅当到达执行时机时，才读取 nohz.nr_cpus
         */
        if (time_before(now, nohz.next_balance)) //如果还没到下一次负载均衡的时间
                goto out; //先不急着决策（只处理可能的统计更新）
        //如果当前系统中压根没有处于 NOHZ 空闲状态的 CPU（nohz.idle_cpus_mask 为空），
        //那么没必要折腾 ILB，直接返回。因为没有空闲 CPU 可以用来帮忙干活。
        /*
         * None are in tickless mode and hence no need for NOHZ idle load
         * balancing
         */
        if (unlikely(cpumask_empty(nohz.idle_cpus_mask)))
                return;
        //进入核心决策区
        //条件 1：当前 CPU 自身过载（最朴素的原因）
        if (rq->nr_running >= 2) { //当前这个繁忙 CPU 上排队了至少 2 个任务
                flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                goto out; //既然有任务在排队，说明这个 CPU 忙不过来了，赶紧唤醒一个空闲 CPU（ILB）来“分担火力”
        }

        rcu_read_lock();
        //条件 2：当前 CPU 算力受损（容量衰减）
        sd = rcu_dereference_all(rq->sd);
        if (sd) {
                /*
                 * If there's a runnable CFS task and the current CPU has reduced
                 * capacity, kick the ILB to see if there's a better CPU to run on:
                 */
                //当前 CPU 上有 CFS 任务在运行，且该 CPU 的可用算力因 RT/IRQ 干扰或硬件降频而严重缩水
                if (rq->cfs.h_nr_runnable >= 1 && check_cpu_capacity(rq, sd)) {
                        flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                        goto unlock; //既然它变弱了，就赶紧把任务迁走
                }
        }
        //条件 3：非对称封装（ASYM_PACKING）—— 优先核心闲置
        sd = rcu_dereference_all(per_cpu(sd_asym_packing, cpu));
        if (sd) {
                /*
                 * When ASYM_PACKING; see if there's a more preferred CPU
                 * currently idle; in which case, kick the ILB to move tasks
                 * around.
                 * 开启 ASYM_PACKING 时：检查是否存在优先级更高且当前处于空闲状态的 CPU；
                 * 如果存在，则唤醒空闲负载均衡器（ILB），迁移任务。
                 * When balancing between cores, all the SMT siblings of the
                 * preferred CPU must be idle.
                 * 在核心间做负载均衡时，目标优选 CPU 对应的所有 SMT 逻辑线程必须全部空闲。
                 */
                for_each_cpu_and(i, sched_domain_span(sd), nohz.idle_cpus_mask) {
                        //常见于 x86 架构（Intel ITMT），某些核心频率更高（“优先核心”）。
                        //如果系统中存在一个比当前 CPU 更优（频率更高）的核心，且那个核心正好处于空闲状态
                        if (sched_asym(sd, i, cpu)) {
                                flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                                goto unlock; //既然有更好的空闲核心在“摸鱼”，赶紧把当前 CPU 上的任务踢过去，让任务跑得更快
                        }
                }
        }
        //条件 4：异构算力（ASYM_CPUCAPACITY）—— 检测到 Misfit 任务
        sd = rcu_dereference_all(per_cpu(sd_asym_cpucapacity, cpu));
        if (sd) {
                /*
                 * When ASYM_CPUCAPACITY; see if there's a higher capacity CPU
                 * to run the misfit task on.
                 * 当前 CPU 上存在一个“Misfit Task”（即 rq->misfit_task_load 非零，重任务跑在小核上
                 */
                if (check_misfit_status(rq)) {
                        flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                        goto unlock;
                }

                /*
                 * For asymmetric systems, we do not want to nicely balance
                 * cache use, instead we want to embrace asymmetry and only
                 * ensure tasks have enough CPU capacity.
                 * 对于非对称系统，我们不追求对 cache 使用做均衡调度；相反，要顺应系统的非对称特性，仅保证任务能够获得充足的 CPU 算力
                 * Skip the LLC logic because it's not relevant in that case.
                 * 此时跳过最后一级缓存（LLC）相关逻辑，因为该场景下缓存均衡逻辑不再适用
                 */
                goto unlock;
        }
        //条件 5：LLC 域（sd_llc_shared，见 sched_topo.md）负载不均
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
                 * 若各 LLC 域之间存在负载不均衡（也就是我们有机会提升整体 cache 利用率），
                 * 就需要找到一个负载更低的 LLC 域，从当前域迁出部分负载。同理，也可能需要在当前
                 * LLC 域内部打散负载（例如 SMT 核被任务占满，但同域其他 CPU 处于空闲）。
                 * 在当前代码位置无法获知其他域的繁忙状态 —— 因此只要判断该 LLC 域存在可迁移任务，
                 * 就触发一次 NOHZ 负载均衡。
                 */
                nr_busy = atomic_read(&sds->nr_busy_cpus);
                if (nr_busy > 1) {//当前 CPU 所在的 LLC 域中，有超过 1 个繁忙的 CP（且有空闲 CPU 存在）
                        flags = NOHZ_STATS_KICK | NOHZ_BALANCE_KICK;
                        goto unlock; //说明缓存资源可能没有被充分利用，或者 SMT 兄弟核之间负载不均，需要通过 ILB 来整理一下
                }
        }
unlock:
        rcu_read_unlock();
out:
        if (READ_ONCE(nohz.needs_update)) //如果系统需要更新 NOHZ 统计数据
                flags |= NOHZ_NEXT_KICK;  //强制追加 NOHZ_NEXT_KICK 标志

        if (flags)
                kick_ilb(flags);
}
```

#### `nohz.has_blocked_load`
* `nohz.has_blocked_load` 表示 `nohz.idle_cpus_mask` 里的 tick-stopped idle CPU 可能还有 blocked PELT load/util 需要继续衰减更新
  * 因为这些 CPU 停了 scheduler tick，不能靠本地 tick 周期性更新 blocked averages，所以需要别的 CPU 触发 `NOHZ_STATS_KICK`，代它们跑 `sched_balance_update_blocked_averages()`
* 注意：`flags = NOHZ_STATS_KICK` 只置位了统计更新标志，并没有置位负载均衡标志，这是因为
  * PELT 的负载平均值会随时间指数衰减。如果一个空闲 CPU 停了 tick（进入 NOHZ 模式），它上面的阻塞负载（blocked load）就无法被及时更新
  * 如果长期不更新，调度器拿到的负载数据会是“过时且偏高”的。这会导致：
    * `find_busiest_group()` 误判某些 CPU 依然很繁忙（其实是历史负载残留）
    * 触发不必要的负载均衡
  * 因此，调度器强制规定：哪怕没有任务需要迁移，到了时间（`next_blocked`）也必须强制唤醒 ILB CPU，去各个空闲 CPU 上把 PELT 数据“打扫”一遍（即更新 `rq->avg_rt`、`rq->avg_irq` 等）
  * 此时 `flags` 只有 `NOHZ_STATS_KICK`，`kick_ilb()` 虽然发了 IPI，但 ILB 收到后只会更新统计，不会拉取任务

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
        //由 nohz_csd_func() 完成清除操作
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

### `SCHED_SOFTIRQ` 软中断处理函数 `sched_balance_softirq()`
* `SCHED_SOFTIRQ` 软中断在两个地方触发：
1. 为做周期性的负载均衡由直接由本地的 `sched_tick() -> sched_balance_trigger()` 触发
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

### 进行 NOHZ 空闲负载均衡
* 有两条路径会走到 `_nohz_idle_balance()`，我们这里看的是第二条路径
```c
//kernel/sched/idle.c
do_idle()
   //kernel/sched/fair.c
-> nohz_run_idle_balance(cpu)
   -> _nohz_idle_balance(cpu_rq(cpu), NOHZ_STATS_KICK)

//kernel/sched/fair.c
sched_balance_softirq()
-> nohz_idle_balance(this_rq, idle)
   -> _nohz_idle_balance(this_rq, flags)
```
* NOHZ 空闲负载均衡入口函数 `nohz_idle_balance()`，被踢醒了后起来负责干活
```cpp
/*
 * In CONFIG_NO_HZ_COMMON case, the idle balance kickee will do the
 * rebalancing for all the CPUs for whom scheduler ticks are stopped.
 */
static bool nohz_idle_balance(struct rq *this_rq, enum cpu_idle_type idle) 
{
        unsigned int flags = this_rq->nohz_idle_balance;
        //flags 是由 nohz_csd_func() 在 IPI 中断中存入 rq->nohz_idle_balance 的
        if (!flags)
                return false;           //没有待处理的 NOHZ 请求

        this_rq->nohz_idle_balance = 0; //清除标志，防止重复执行
        //只有 idle == CPU_IDLE（严格空闲状态）才允许执行。如果 CPU 已经被唤醒去跑别的任务，就不适合再替别人做均衡了
        if (idle != CPU_IDLE)           //只有真正处于 CPU_IDLE 状态才干活
                return false;

        _nohz_idle_balance(this_rq, flags);
        //告诉上层 sched_balance_softirq()：“我已经执行过 NOHZ 均衡了，你不要再走常规负载均衡路径”
        return true; 
}
```
* `_nohz_idle_balance()` 是内部函数，用于对所有空闲 CPU 执行负载均衡。根据标志位的不同，本次负载均衡可以仅做阻塞负载的简易更新，或是执行包含任务迁移的完整负载均衡
```cpp
/*
 * Internal function that runs load balance for all idle CPUs. The load balance
 * can be a simple update of blocked load or a complete load balance with
 * tasks movement depending of flags.
 */
static void _nohz_idle_balance(struct rq *this_rq, unsigned int flags)
{
	/* Earliest time when we have to do rebalance again */
	unsigned long now = jiffies;
	unsigned long next_balance = now + 60*HZ;
	bool has_blocked_load = false;
	int update_next_balance = 0;
	int this_cpu = this_rq->cpu;
	int balance_cpu;
	struct rq *rq;
	//NOHZ_BALANCE_KICK 通常总是和 NOHZ_STATS_KICK 一起被设置，如果 NOHZ_BALANCE_KICK 单独出现，说明状态机出了问题
	WARN_ON_ONCE((flags & NOHZ_KICK_MASK) == NOHZ_BALANCE_KICK);
	//我们假定本次更新完成后不再存在空闲负载，并清除 has_blocked_load 标志。
	//如果在此期间有 CPU 进入空闲状态，该 CPU 会置位 has_blocked_load 标志，触发新一轮空闲负载更新。
	//由于 CPU 变为空闲时，会先加入 idle_cpus_mask 掩码，再置位该标志，因此我们可以保证不会误清状态，
	//也不会遗漏对空闲 CPU 的负载检查。
	//idle_cpus_mask 与 needs_update 之间的逻辑同样遵循该规则。
	/*
	 * We assume there will be no idle load after this update and clear
	 * the has_blocked_load flag. If a cpu enters idle in the mean time, it will
	 * set the has_blocked_load flag and trigger another update of idle load.
	 * Because a cpu that becomes idle, is added to idle_cpus_mask before
	 * setting the flag, we are sure to not clear the state and not
	 * check the load of an idle cpu.
	 *
	 * Same applies to idle_cpus_mask vs needs_update.
	 */
	if (flags & NOHZ_STATS_KICK)
		WRITE_ONCE(nohz.has_blocked_load, 0);
	if (flags & NOHZ_NEXT_KICK)
		WRITE_ONCE(nohz.needs_update, 0);
	//内存屏障确保上面的清零操作对其他 CPU 可见
	/*
	 * Ensures that if we miss the CPU, we must see the has_blocked_load
	 * store from nohz_balance_enter_idle().
	 * 如果我们漏掉了某个刚进入空闲的 CPU，我们一定能看到它在 nohz_balance_enter_idle() 中对 has_blocked_load 的写入
	 */
	smp_mb();
	//从 this_cpu 的下一个 CPU 开始遍历，让我们以 this_cpu 结尾，给其他空闲 CPU 一个拉取负载的机会
	/*
	 * Start with the next CPU after this_cpu so we will end with this_cpu and let a
	 * chance for other idle cpu to pull load.
	 */
	for_each_cpu_wrap(balance_cpu,  nohz.idle_cpus_mask, this_cpu+1) {
		if (!idle_cpu(balance_cpu)) //虽然 balance_cpu 在 nohz.idle_cpus_mask 中，但可能刚好被唤醒
			continue;           //跳过它，避免干扰正在恢复繁忙的 CPU
		//场景：正在替别人干活的这个 CPU（this_cpu），自己突然被唤醒了（!idle_cpu(this_cpu)），
		//并且有更高优先级的任务需要调度（need_resched()）
		//动作：立即中止均衡工作。但在中止前，必须保存状态
		/*
		 * If this CPU gets work to do, stop the load balancing
		 * work being done for other CPUs. Next load
		 * balancing owner will pick it up.
		 * 如果该 CPU 忽然来活了，停止为其他 CPU 所作的负载均衡工作。下一个负载均衡的 owner 会继续。
		 */
		if (!idle_cpu(this_cpu) && need_resched()) {
			if (flags & NOHZ_STATS_KICK)//如果还有统计更新任务没做完，把 has_blocked_load 重新置位
				has_blocked_load = true; //告诉系统“还有活没干完，下次再来”
			if (flags & NOHZ_NEXT_KICK)
				WRITE_ONCE(nohz.needs_update, 1);
			goto abort; //宁可让空闲 CPU 的均衡工作延迟，也不能耽误当前 CPU 自己的调度任务
		}

		rq = cpu_rq(balance_cpu);
		//刷新目标 CPU 的 PELT 阻塞负载统计（即 rq->avg_rt、rq->avg_irq、cfs_rq->avg 等）
		//has_blocked_load |= 累计标记。只要任何一个 CPU 有阻塞负载，最终就会重新设置全局的 nohz.has_blocked_load，确保下一轮还会来更新
		if (flags & NOHZ_STATS_KICK)
			has_blocked_load |= update_nohz_stats(rq);
		//只有当目标 CPU 的 next_balance 到期时，才进行任务迁移
		/*
		 * If time for next balance is due,
		 * do the balance.
		 */
		if (time_after_eq(jiffies, rq->next_balance)) {
			struct rq_flags rf;

			rq_lock_irqsave(rq, &rf);
			update_rq_clock(rq); //更新该 CPU 的时钟（因为它的 tick 停了，rq->clock 可能过时）
			rq_unlock_irqrestore(rq, &rf);
			//只有 NOHZ_BALANCE_KICK 标志存在时才做迁移。如果只有 NOHZ_STATS_KICK，则跳过
			if (flags & NOHZ_BALANCE_KICK)
				sched_balance_domains(rq, CPU_IDLE);
		}
		//目的：在所有被遍历的 CPU 中，找出最早需要下一次均衡的时间点
		//next_balance 初始化为 now + 60*HZ（60 秒后）。如果某个 CPU 的 next_balance 更早，就更新这个值
		if (time_after(next_balance, rq->next_balance)) {
			next_balance = rq->next_balance; //将这个最早的时间写入 nohz.next_balance，作为全局的下一次 NOHZ 均衡触发时间
			update_next_balance = 1;
		}
	}
	//只有确实找到了更早的 next_balance 时间才更新
	/*
	 * next_balance will be updated only when there is a need.
	 * When the CPU is attached to null domain for ex, it will not be
	 * updated.
	 */
	if (likely(update_next_balance))
		nohz.next_balance = next_balance;
	//设置下一次统计更新时间。这保证了即使系统完全空闲，PELT 统计也会定期被刷新
	if (flags & NOHZ_STATS_KICK)
		WRITE_ONCE(nohz.next_blocked,
			   now + msecs_to_jiffies(LOAD_AVG_PERIOD));
	//如果还有 CPU 的阻塞负载没更新完（或被中止），重新置位全局标志，确保下一轮 nohz_balancer_kick 会再次触发统计更新
abort:
	/* There is still blocked load, enable periodic update */
	if (has_blocked_load)
		WRITE_ONCE(nohz.has_blocked_load, 1);
}
```
* `update_nohz_stats()` 是 NOHZ 空闲负载均衡中负责按需更新指定 CPU 阻塞负载统计的辅助函数
  * 被 `_nohz_idle_balance()` 在遍历所有空闲 CPU 时调用，用于决定“这个 CPU 的 PELT 阻塞负载现在要不要刷新？刷新后还有没有残留？”
  * 只在“有负载、仍空闲、且到期”时才真正刷新 PELT 统计，并通过返回值精确告知调用者“这个 CPU 是否还需要后续的统计更新”
```cpp
static bool update_nohz_stats(struct rq *rq)
{       //简单取出目标 CPU 编号，后续用于掩码检查和实际更新
        unsigned int cpu = rq->cpu;
        //在 nohz_balance_enter_idle() 中被置为 1，表示该 CPU 上有未清理的 PELT 阻塞负载
        if (!rq->has_blocked_load) //为 0，说明没有需要更新的东西
                return false;      //直接返回 false，避免无谓开销
        //虽然 _nohz_idle_balance() 是从 nohz.idle_cpus_mask 中遍历出来的，但在遍历过程中，
        //目标 CPU 可能刚好被唤醒并退出了空闲（nohz_balance_exit_idle() 会清除掩码）
        if (!cpumask_test_cpu(cpu, nohz.idle_cpus_mask))
                return false; //如果它已经不再空闲，就不应该再替它更新统计
        //rq->last_blocked_load_update_tick 记录该 CPU 上次更新阻塞负载的 jiffies 时间
        //能走到这里 rq->has_blocked_load 仍然为 1，且 CPU 仍在空闲掩码中，说明确实存在阻塞负载，只是还不到刷新时间
        if (!time_after(jiffies, READ_ONCE(rq->last_blocked_load_update_tick)))
                return true;//告诉调用者：“这个 CPU 还有阻塞负载，需要以后继续关注”，从而让全局的 nohz.has_blocked_load 保持置位，安排下一轮更新
        //时间到期后，调用这个函数真正刷新该 CPU 的 PELT 阻塞负载
        sched_balance_update_blocked_averages(cpu); //可能将 rq->has_blocked_load 清 0（如果更新后没有残留）
        //更新完成后，再次读取 rq->has_blocked_load
        //如果更新后已经没有阻塞负载了，返回 false；如果仍有（例如更新期间又有新的阻塞负载加入），返回 true
        //这样调用者就能准确知道该 CPU 是否还需要后续关注
        return rq->has_blocked_load;
}
```

### CPU 进入 NOHZ 空闲状态时设置的负载均衡相关信息
```cpp
static void set_cpu_sd_state_idle(int cpu)
{
        struct sched_domain *sd;
        sd = rcu_dereference_all(per_cpu(sd_llc, cpu)); //返回该 CPU 所属的最后一级缓存（LLC）调度域

        /* See set_cpu_sd_state_busy(): nohz_idle is only used with sd->shared. */
        if (!sd || !sd->shared || sd->nohz_idle)
                return;
        sd->nohz_idle = 1; //表示该 LLC 域进入“至少有一个 CPU 空闲”的状态
        //这个计数器是 nohz_balancer_kick() 中判断“LLC 域内是否有多个繁忙 CPU”的依据
        atomic_dec(&sd->shared->nr_busy_cpus);
}
//该例程将会记录 CPU 将要以停止 tick 的方式进入 idle。该信息会在将来被用于执行空闲负载均衡
/*
 * This routine will record that the CPU is going idle with tick stopped.
 * This info will be used in performing idle load balancing in the future.
 */
void nohz_balance_enter_idle(int cpu)
{
        struct rq *rq = cpu_rq(cpu);
        //这个函数只能在 CPU 自己的上下文（即将进入空闲）中调用
        WARN_ON_ONCE(cpu != smp_processor_id());
        //如果 CPU 正在下线（热拔出），直接返回，不做任何状态记录
        /* If this CPU is going down, then nothing needs to be done: */
        if (!cpu_active(cpu))
                return;
        //因为这个 CPU 即将停掉 tick，它的 PELT 阻塞负载（blocked load）将无法被及时更新。
        //标记这个标志，是为了告诉系统“这个 CPU 上有未清理的负载”
        /*
         * Can be set safely without rq->lock held
         * If a clear happens, it will have evaluated last additions because
         * rq->lock is held during the check and the clear
         * 这里不需要持有 rq->lock，因为如果后续有清除操作，它一定会在持有 rq->lock 的情况下检查并清除，此时会看到这个写入
         */
        rq->has_blocked_load = 1;
        //因为 nohz.idle_cpus_mask 和 sd->nohz_idle 等状态已经设置过了（这个函数之前走进来过？），
        //不需要重复设置。但负载可能在 tick 停止期间被添加，所以需要重新触发一次检查
        /*
         * The tick is still stopped but load could have been added in the
         * meantime. We set the nohz.has_blocked_load flag to trig a check of the
         * *_avg. The CPU is already part of nohz.idle_cpus_mask so the clear
         * of nohz.has_blocked_load can only happen after checking the new load
         * 
         * Tick 仍然处于停止状态，但在此期间可能新增负载。我们置位 nohz.has_blocked_load 标志，
         * 触发对各类 *_avg 负载统计值的检查。该 CPU 已经在 nohz.idle_cpus_mask 掩码中，因此
         * nohz.has_blocked_load 标志只能在完成新负载检查之后才会被清除
         */
        if (rq->nohz_tick_stopped)
                goto out;
         //如果这个 CPU 没有关联任何调度域（完全隔离的 CPU，如 isolcpus 或 nohz_full 隔离的 CPU），则直接返回
        /* If we're a completely isolated CPU, we don't play: */
        if (on_null_domain(rq)) //这些 CPU 不参与负载均衡，不需要被记录到 NOHZ 空闲 CPU 集合中
                return;
        //标记该 CPU 的 tick 已停止
        rq->nohz_tick_stopped = 1;
        //将该 CPU 加入全局 NOHZ 空闲 CPU 掩码。从此以后，find_new_ilb() 和 _nohz_idle_balance() 都会遍历到这个 CPU
        cpumask_set_cpu(cpu, nohz.idle_cpus_mask);
        //屏障的作用：确保 idle_cpus_mask 的设置（原子操作）先于后续的 has_blocked_load 和 needs_update 写入对其他 CPU 可见
        //如果 _nohz_idle_balance() 没有观察到 idle_cpus_mask 的写入（即没遍历到这个 CPU），
        //那么它一定能观察到后续的 has_blocked_load 和 needs_update 写入。这样保证了这个 CPU
        //不会“两头落空”——要么被遍历到，要么被标志位唤醒
        /*
         * Ensures that if nohz_idle_balance() fails to observe our
         * @idle_cpus_mask store, it must observe the @has_blocked_load
         * and @needs_update stores.
         */
        smp_mb__after_atomic();
        //将 LLC 调度域标记为空闲，递减 nr_busy_cpus
        set_cpu_sd_state_idle(cpu);
        //告诉系统“有空闲 CPU 状态变化了，需要更新”
        WRITE_ONCE(nohz.needs_update, 1);
out:    //无论走哪条路径，最终都会执行下面这条语句。这是因为每个进入空闲的 CPU 都可能带来新的阻塞负载，必须触发一次 PELT 统计更新
        /*
         * Each time a cpu enter idle, we assume that it has blocked load and
         * enable the periodic update of the load of idle CPUs
         */
        WRITE_ONCE(nohz.has_blocked_load, 1);
}
```

## PELT 阻塞负载（blocked load）更新

### PELT 阻塞负载（blocked load）更新的入口
* `decayed`：本次更新中，是否有负载发生了时间衰减（即值变小了）
* `done`：是否 **所有** 阻塞负载都已处理干净（没有残留）
  * 初始为 `true`，只要有任何残留会被子函数置为 `false`
* `__update_blocked_others()` 这里的 `other`（非 CFS）指的是：RT（实时）、DL（截止时间）、IRQ（中断）、Thermal（热压力） 这些非 CFS 类的 PELT 负载
  * 传入 `&done`：如果 RT/DL/IRQ 中有阻塞负载没清干净，会把 `done` 置为 `false`
  * 返回值赋给 `decayed`：如果任何一项负载发生了衰减（比如 RT 任务迁移走了，`rq->avg_rt` 应该随之下降），就返回 `true`
* `cpufreq_update_util()` 通知 CPUFreq：
  * CPUFreq 的 **schedutil 调速器** 使用 PELT 的 `util_avg` 来决定 CPU 频率
  * 如果阻塞负载发生了衰减（即 CPU 实际需求变小了），但调速器还基于旧的 `util_avg` 维持高频，就会浪费功耗
  * 所以一旦检测到衰减，立即通知 schedutil 重新评估，及时降频
```c
static void __sched_balance_update_blocked_averages(struct rq *rq)
{   
    bool decayed = false, done = true;
    //先记录“本次更新发生了”，供后续节流判断使用
    update_blocked_load_tick(rq);
    //更新 RT/DL/IRQ/Thermal 的 PELT 阻塞负载
    decayed |= __update_blocked_others(rq, &done);
    decayed |= __update_blocked_fair(rq, &done); //更新 CFS 的 PELT 阻塞负载
    //如果本次更新确认“没有残留的阻塞负载”，就把 rq->has_blocked_load 清零
    update_has_blocked_load_status(rq, !done); //如果还有残留，保持 1 让 update_nohz_stats() 下一轮继续来处理
    if (decayed) //只有确实发生衰减才通知，避免无意义的调用
        cpufreq_update_util(rq, 0); //通知 schedutil 降频
}

static void sched_balance_update_blocked_averages(int cpu)
{   
    struct rq *rq = cpu_rq(cpu);
    
    guard(rq_lock_irqsave)(rq);
    update_rq_clock(rq); //更新队列的时间，见 sched_pelt.md
    __sched_balance_update_blocked_averages(rq);
}
```

#### 记录更新阻塞负载的时间
* `rq->last_blocked_load_update_tick` 记录该 CPU 上次更新阻塞负载的 `jiffies` 时间
* `update_blocked_load_tick()` 把当前 `jiffies` 写入 `rq->last_blocked_load_update_tick` 更新记录
* 之前 `update_nohz_stats()` 用到这个记录：如果还没到下一个更新周期，就跳过实际刷新，只返回“还有负载”让上层稍后再来
```c
static inline void update_blocked_load_tick(struct rq *rq)
{
    WRITE_ONCE(rq->last_blocked_load_update_tick, jiffies);
}
```
#### `update_has_blocked_load_status()` 按需清除阻塞负载标志
* 注意方向性：这里只会清 `0`，不会置 `1`
  * 置 `1` 的责任在 `nohz_balance_enter_idle()`（CPU 进入空闲时）路径
  * 这体现了 **悲观置位，乐观清零** 的策略：只要可能还有残留，就保持 `1`，直到一次完整更新确认干净了才清 `0`
```c
static inline void update_has_blocked_load_status(struct rq *rq, bool has_blocked_load)
{
    if (!has_blocked_load)
        rq->has_blocked_load = 0;
}
```

### `__update_blocked_others()` 非 CFS 阻塞负载更新
* `__update_blocked_others()` 是 PELT 阻塞负载更新流程中的 “非 CFS 前置处理站”
  * 它先刷新 RT/DL/HW/IRQ 四类负载的统计值，汇报是否发生衰减
  * 并标记是否还有残留，为后续 CFS 更新和 CPUFreq 调频提供准确的数据基础
* `update_other_load_avgs()` 见 [PELT/其他负载更新函数 `update_other_load_avgs()` 函数](sched_pelt.md#其他负载更新函数-update_other_load_avgs-函数)
* 注释解释了顺序依赖：
  1. `update_other_load_avgs()` 内部可能调用 `cpufreq_update_util()`：当 RT/DL/IRQ 负载发生衰减时，会立即通知 schedutil 调速器重新评估频率
  2. schedutil 计算频率时，依赖所有信号（包括 CFS）的准确值：如果先更新 CFS 再更新 RT，那么在 CFS 触发 `cpufreq_update_util()` 时，RT/DL/IRQ 的信号还是旧的，会导致频率计算偏低或偏高
  3. 因此，必须先更新 RT/DL/IRQ/HW，让这些信号先“归位”，再更新 CFS。这样 CFS 触发的 `cpufreq_update_util()` 就能基于完整的、最新的信号做出正确的频率决策
```c
static bool __update_blocked_others(struct rq *rq, bool *done)
{
    bool updated;
 
    /*
     * update_load_avg() can call cpufreq_update_util(). Make sure that RT,
     * DL and IRQ signals have been updated before updating CFS.
     */
    updated = update_other_load_avgs(rq);
    //RT/DL/HW/IRQ 中还有非零值，就把 *done 置为 false，表示还有残留阻塞负载
    if (others_have_blocked(rq))
        *done = false;
    //如果某个信号（如 RT）从非零衰减到 0，会返回 true  
    return updated;
}
```
#### `others_have_blocked()` 探测是否还有残留的其他阻塞负载
* 该函数判断是否还有残留的其他阻塞负载
  * 只要任何一项非零，就返回 `true`，表示 **有残留**
  * 全部为零才返回 `false`，表示 **这一轮清干净了**
* 注意：这里判断的是“是否大于 `0`”，而不是“是否发生了衰减”。即使负载很小，只要没衰减到 `0`，就认为还有残留
```c
static inline bool others_have_blocked(struct rq *rq)
{   //该 CPU 上 RT（实时）任务 的 PELT 平均利用率。非零表示还有 RT 负载残留
    if (cpu_util_rt(rq))
        return true;
    //该 CPU 上 DL（Deadline）任务 的 PELT 平均利用率
    if (cpu_util_dl(rq))
        return true;
    //硬件反馈负载（如 Intel HWP、ARM DSU），反映硬件层面的压力
    if (hw_load_avg(rq))
        return true;
    //IRQ（中断） 造成的 PELT 平均利用率
    if (cpu_util_irq(rq))
        return true;

    return false;
}
```

### `__update_blocked_fair()` 阻塞负载更新
* `__update_blocked_fair()` 是 PELT 阻塞负载更新流程中的 “CFS 处理站”
  * 它用扣除 throttled 时间后的 PELT 时钟
  * 更新整个 CFS 层级（含 cgroup）的负载平均值，汇报是否发生衰减
  * 并标记是否还有残留负载，为 `has_blocked_load` 状态和 CPUFreq 调频提供准确的 CFS 侧数据

#### 1. `cfs_rq_clock_pelt()` 计算 CFS 运行队列的 PELT 时钟
* `cfs_rq->pelt_clock_throttled`：布尔标志，表示当前 `cfs_rq` **正处于 throttled 状态**
* `cfs_rq->throttled_clock_pelt_time`：**累计** 被 throttled 的 PELT 时间总量（用于从主时钟中减去）
* `cfs_rq->throttled_clock_pelt`：throttled **开始时刻** 的 PELT 时钟值（用于计算当前这次 throttled 已持续多久）
* **情况 A**：正在被 throttled（`pelt_clock_throttled == 1`）
  * 进入 throttled 那一刻的时钟 `throttled_clock_pelt` **减去** 之前累计的 throttled 时间 `throttled_clock_pelt_time`，得到扣掉历史 throttled 时间后的 PELT 时钟
  * 注意：**当前** 这次 throttled 的时间没有被 **减掉** —— 因为在 throttled 期间，`cfs_rq` 不应该发生 PELT 衰减。用 throttle 开始时刻的时钟，正好“冻结”住时间轴
* **情况 B**：正常运行（`pelt_clock_throttled == 0`）
  * 用运行队列的实时 PELT 时钟，减去累计的 throttled 时间
  * 得到的就是 **这个 `cfs_rq` 真正可运行的时间**
* 减去 PELT 累计时间的原因：
  * cgroup 带宽限制（`cpu.max`）会让 `cfs_rq` 在超出配额后被“强制休眠”一段时间
  * 这段时间里，任务并没有真的在等待 CPU —— 是被人为限制的。如果把它算进 PELT，会导致负载值虚高，误导调度器
  * 所以 PELT 时钟必须“跳过”throttled 期间，只统计真正可运行的时间
```c
/* rq->task_clock normalized against any time this cfs_rq has spent throttled */
static inline u64 cfs_rq_clock_pelt(struct cfs_rq *cfs_rq)
{
    if (unlikely(cfs_rq->pelt_clock_throttled)) //正在被 throttled
        return cfs_rq->throttled_clock_pelt - cfs_rq->throttled_clock_pelt_time;
    
    return rq_clock_pelt(rq_of(cfs_rq)) - cfs_rq->throttled_clock_pelt_time;
}
```
#### 2. `cfs_rq_has_blocked_load()` 探测 CFS 是否有残留负载
* `cfs_rq->avg.load_avg`：CFS 运行队列的 **负载平均值**（权重加权）。非零表示还有阻塞任务的负载残留
* `cfs_rq->avg.util_avg`：CFS 运行队列的 **利用率平均值**（算力占用）。非零表示还有实际算力需求残留
* 与 `others_have_blocked()` 的逻辑一致：只要有任何非零值，就认为 **还有残留**，返回 `true`
* 为什么只检查 `load_avg` 和 `util_avg`？
  * CFS 的核心指标就是这两个：`load_avg` 反映“有多少权重在等”，`util_avg` 反映“实际用了多少算力”
  * `runnable_avg` 不单独检查，因为它要么被 `util_avg` 覆盖，要么已经通过 `load_avg` 体现
```c
static inline bool cfs_rq_has_blocked_load(struct cfs_rq *cfs_rq)
{
    if (cfs_rq->avg.load_avg)
        return true;

    if (cfs_rq->avg.util_avg)
        return true;

    return false;
}
```
#### 3. `__update_blocked_fair()` CFS 阻塞负载更新入口

##### `__update_blocked_fair()` 的非组调度版本
* [`update_cfs_rq_load_avg()` 是 CFS 负载的核心更新函数](#update_cfs_rq_load_avg-更新-cfs-运行队列负载平均值)。它会：
  * 对 `cfs_rq->avg.load_avg`、`util_avg`、`runnable_avg` 等做时间衰减
  * 递归更新所有 `leaf_cfs_rq_list` 上的子 `cfs_rq`（cgroup 层级）
  * 返回 `true` 表示发生了衰减（负载值变小了）
  * 注意：虽然这里只传入了根 `cfs_rq`，但 `update_cfs_rq_load_avg()` 内部会通过 `leaf_cfs_rq_list` 遍历整个 cgroup 层级，所以所有子 `cfs_rq` 都会被更新到
```c
static bool __update_blocked_fair(struct rq *rq, bool *done)
{
    struct cfs_rq *cfs_rq = &rq->cfs;
    bool decayed;
    //更新根 cfs_rq 的 PELT 负载
    decayed = update_cfs_rq_load_avg(cfs_rq_clock_pelt(cfs_rq), cfs_rq);
    if (cfs_rq_has_blocked_load(cfs_rq)) //检查根 cfs_rq 是否还有残留负载
        *done = false; //告诉上层 “CFS 这边阻塞负载还没清干净”
    //返回衰减状态给上层，聚合后最终决定是否调用 cpufreq_update_util() 通知 schedutil 调速器降频
    return decayed;
}
```

##### `__update_blocked_fair()` 的组调度版本

* 组调度版的 `__update_blocked_fair()` 是一个自底向上的 cgroup 层级清理器，它遍历所有叶子 `cfs_rq`：
  * [逐个更新 PELT 负载](#update_cfs_rq_load_avg-更新-cfs-运行队列负载平均值)
  * 差分传播到任务组
  * 向父层传播组实体负载
  * 并清理已完全衰减的节点
  * 最终汇报是否还有残留负载和是否发生了影响调频的衰减

###### 1）`update_tg_load_avg()`：差分更新任务组负载
* 该函数确保：
  ```c
  tg->load_avg = 该任务组下所有 cfs_rq 的 avg.load_avg 之和
  ```
* 然而 `tg->load_avg` 是全局共享的（同一个任务组的所有 CPU 都会访问）。如果每次更新都遍历所有 `cfs_rq` 重新求和，开销巨大（`O(n)` 且跨 CPU 缓存行竞争）。
* 所以采用 **差分更新（differential update）**：
  * `cfs_rq->tg_load_avg_contrib` 记录“上次传播时，这个 `cfs_rq` 贡献了多少”
  * `delta = 当前 load_avg - 上次贡献值`，就是需要增量更新的量
  * 只把 `delta` 加到 `tg->load_avg` 上
```cpp
/**
 * update_tg_load_avg - update the tg's load avg
 * @cfs_rq: the cfs_rq whose avg changed
 *
 * This function 'ensures': tg->load_avg := \Sum tg->cfs_rq[]->avg.load.
 * However, because tg->load_avg is a global value there are performance
 * considerations.
 *
 * In order to avoid having to look at the other cfs_rq's, we use a
 * differential update where we store the last value we propagated. This in
 * turn allows skipping updates if the differential is 'small'.
 *
 * Updating tg's load_avg is necessary before update_cfs_share().
 */
static inline void update_tg_load_avg(struct cfs_rq *cfs_rq)
{
        long dl, dr;
        u64 now;
        //根任务组的 load_avg 不用于份额计算，跳过
        /*
         * No need to update load_avg for root_task_group as it is not used.
         */
        if (cfs_rq->tg == &root_task_group)
                return;
        //CPU 已下线，不再贡献负载，跳过
        /* rq has been offline and doesn't contribute to the share anymore: */
        if (!cpu_active(cpu_of(rq_of(cfs_rq))))
                return;
        //限流：迁移密集场景下，tg->load_avg 的访问可能没有边界（限制？），限制最多每毫秒更新一次
        /*
         * For migration heavy workloads, access to tg->load_avg can be
         * unbound. Limit the update rate to at most once per ms.
         */
        now = rq_clock(rq_of(cfs_rq));
        if (now - cfs_rq->last_update_tg_load_avg < NSEC_PER_MSEC)
                return;
        //差分阈值：只有变化量超过上次贡献值的 1/64（约 1.56%）时才真正更新
        dl = cfs_rq->avg.load_avg - cfs_rq->tg_load_avg_contrib; //需要增量更新的量
        dr = cfs_rq->avg.runnable_avg - cfs_rq->tg_runnable_avg_contrib;
        if (abs(dl) > cfs_rq->tg_load_avg_contrib / 64 || 
            abs(dr) > cfs_rq->tg_runnable_avg_contrib / 64) { 
                atomic_long_add(dl, &cfs_rq->tg->load_avg); //加到任务组总负载上
                atomic_long_add(dr, &cfs_rq->tg->runnable_avg);
                cfs_rq->tg_load_avg_contrib = cfs_rq->avg.load_avg; //记录新的贡献值
                cfs_rq->tg_runnable_avg_contrib = cfs_rq->avg.runnable_avg;
                cfs_rq->last_update_tg_load_avg = now; //记录新的贡献值时间戳，供下次差分使用
        }
}
```

###### 2）`update_idle_cfs_rq_clock_pelt()`：记录空闲 `cfs_rq` 的 throttled 时间
* 当一个 `cfs_rq` 变为空闲（`nr_queued == 0`）时，记录它的 throttled 状态到 `throttled_pelt_idle`
* CPU 进入空闲后 tick 停止，`rq->clock_pelt` 不再更新，但真实时间（`sched_clock_cpu()`）继续流逝。当这个 CPU 上的任务迁移走时，需要回答一个问题：
  * 这个 `cfs_rq` 在变空闲那一刻，它的有效 PELT 时钟是多少？从那一刻到现在，真实时间又流逝了多少？
  * `throttled_pelt_idle` 就是回答第一个问题的关键参数之一：它记录了变空闲时刻的累计 throttled 时间，用于从 `rq->clock_pelt_idle` 中扣除，得到 `cfs_rq` 视角的有效 PELT 时钟
  * 延申阅读 [PELT - 迁移前 PELT 衰减任务负载 - 考虑 CFS 带宽限制](sched_pelt.md#考虑-cfs-带宽限制throttled)
```cpp
static inline void update_idle_cfs_rq_clock_pelt(struct cfs_rq *cfs_rq)
{
        u64 throttled;
        //如果当前 cfs_rq 正在被 throttled，用一个特殊值 U64_MAX 标记，
        //表示“这个 cfs_rq 的空闲时间暂时不可用”（因为 throttled 期间的时间不计入 PELT）
        if (unlikely(cfs_rq->pelt_clock_throttled))
                throttled = U64_MAX;
        else //否则记录累计的 throttled 时间
                throttled = cfs_rq->throttled_clock_pelt_time;
        //迁移前 PELT 衰减任务负载 migrate_se_pelt_lag() 函数会用到这个值
        u64_u32_store(cfs_rq->throttled_pelt_idle, throttled);
}
```

###### 3）`skip_blocked_update()`：能否跳过衰减？
* 该函数检查我们是否需要更新一个阻塞的组调度实体的负载和利用率
  * 返回 `false` 必须调用 `update_load_avg()` 做衰减或传播
  * 返回 `true` 则跳过衰减
```cpp
/*
 * Check if we need to update the load and the utilization of a blocked
 * group_entity:
 */
static inline bool skip_blocked_update(struct sched_entity *se)
{
        struct cfs_rq *gcfs_rq = group_cfs_rq(se);//该组调度实体对应的子 cfs_rq
        //如果调度实体有非零的负载或利用率，我们不得不衰减它
        /*    
         * If sched_entity still have not zero load or utilization, we have to
         * decay it:
         */
        if (se->avg.load_avg || se->avg.util_avg)
                return false;
        //子 cfs_rq 有待传播的变化，我们不得不更新调度实体的负载和利用率
        /*    
         * If there is a pending propagation, we have to update the load and
         * the utilization of the sched_entity:
         */
        if (gcfs_rq->propagate)
                return false;
        //否则，调度实体的负载和利用率已经是零且无待传播的变化，尝试衰减它就是浪费时间，跳过
        /*    
         * Otherwise, the load and the utilization of the sched_entity is
         * already zero and there is no pending propagation, so it will be a
         * waste of time to try to decay it:
         */
        return true; 
}
```

###### 4）`load_avg_is_decayed()`：检查 `sched_avg` 是否完全衰减
* `load_sum`、`util_sum`、`runnable_sum` 全为 `0` 才算衰减完毕
```cpp
static inline bool load_avg_is_decayed(struct sched_avg *sa)
{
        if (sa->load_sum)
                return false;

        if (sa->util_sum)
                return false;

        if (sa->runnable_sum)
                return false;
        //数学上 avg = sum / divider，如果 sum == 0 但 avg != 0，说明 PELT 数学关系被破坏（舍入误差累积或并发问题），触发告警
        /*
         * _avg must be null when _sum are null because _avg = _sum / divider
         * Make sure that rounding and/or propagation of PELT values never
         * break this.
         */
        WARN_ON_ONCE(sa->load_avg ||
                      sa->util_avg ||
                      sa->runnable_avg);

        return true;
}
```

###### 5）`child_cfs_rq_on_list()`：检查子 `cfs_rq` 是否还在链表上
* `list_add_leaf_cfs_rq()` 保证：**子 `cfs_rq` 总是紧挨着放在父 `cfs_rq` 之前**，且从链表移除 `cfs_rq` 的时候是自底向上移除的。所以链表中某个 `cfs_rq` 的前一个元素，要么是它的子节点，要么是无关节点
```cpp
/*
 * Because list_add_leaf_cfs_rq always places a child cfs_rq on the list
 * immediately before a parent cfs_rq, and cfs_rqs are removed from the list
 * bottom-up, we only have to test whether the cfs_rq before us on the list
 * is our child.
 * If cfs_rq is not on the list, test whether a child needs its to be added to
 * connect a branch to the tree  * (see list_add_leaf_cfs_rq() for details).
 */
static inline bool child_cfs_rq_on_list(struct cfs_rq *cfs_rq)
{
        struct cfs_rq *prev_cfs_rq;
        struct list_head *prev;
        struct rq *rq = rq_of(cfs_rq);

        if (cfs_rq->on_list) { //在链表上，直接在链表中查前一个元素
                prev = cfs_rq->leaf_cfs_rq_list.prev;
        } else { //不在链表上，但可能有一个“孤立分支”通过 rq->tmp_alone_branch 指向它
                prev = rq->tmp_alone_branch;
        }
        //如果前一个元素是链表头，说明没有子节点，返回 false
        if (prev == &rq->leaf_cfs_rq_list)
                return false;
        //否则取出前一个 cfs_rq
        prev_cfs_rq = container_of(prev, struct cfs_rq, leaf_cfs_rq_list);
        //检查它的父任务组是否就是当前 cfs_rq 的任务组。如果是，说明有子节点还在链表上，返回 true
        return (prev_cfs_rq->tg->parent == cfs_rq->tg);
}
```
###### 6）`cfs_rq_is_decayed()`：判断 `cfs_rq` 是否完全衰减
* 该函数判断 `cfs_rq` 是否完全衰减，只有四个条件全不满足，这个 `cfs_rq` 才能从 `leaf_cfs_rq_list` 中移除
```cpp
static inline bool cfs_rq_is_decayed(struct cfs_rq *cfs_rq)
{       //还有排队任务的权重，未完全衰减
        if (cfs_rq->load.weight)
                return false;
        //load_sum、util_sum、runnable_sum 不全为 0，未完全衰减
        if (!load_avg_is_decayed(&cfs_rq->avg))
                return false;
        //有子 cfs_rq 还在链表上（否则子节点还需要处理），不能从 leaf_cfs_rq_list 中移除
        if (child_cfs_rq_on_list(cfs_rq))
                return false;
        //对任务组的负载贡献还未归零（tg->load_avg 还包含它的旧值）
        if (cfs_rq->tg_load_avg_contrib)
                return false;
        //四个条件全不满足才算“完全衰减”
        return true;
}
```

###### 7）`__update_blocked_fair()` CFS 阻塞负载更新主函数
* 遍历 `rq->leaf_cfs_rq_list` 上的 **所有叶子 `cfs_rq`**，完成以下步骤：
1. 更新 `cfs_rq` 负载，用扣除 throttled 的 PELT 时钟更新负载，返回 `true` 表示发生了衰减或负载移除
2. 如果发生了变化，传播到任务组
   * 只在 `update_cfs_rq_load_avg()` 返回 `true` 时才调用，避免无谓的差分计算
   * 内部有限流和阈值过滤
3. 如果 `cfs_rq` 空闲，记录 throttled 时间
4. 标记根 `cfs_rq` 的衰减
   * **只有根 cfs_rq（`rq->cfs`）的衰减** 才会让 `decayed` 为 `true`
   * 为什么？因为根 `cfs_rq` 的负载变化才真正影响 **整个 CPU 的 CFS 负载**，才需要通知 CPUFreq 调频。子 `cfs_rq` 的衰减只影响 cgroup 内部的份额分配
5. 向父 cfs_rq 传播组实体的负载
   * `skip_blocked_update(se)`：如果 `se` 本身已无负载且无待传播变化，跳过
   * 否则，`update_load_avg(cfs_rq_of(se), se, UPDATE_TG)` 更新组实体的 PELT 负载，并带 `UPDATE_TG` 标志触发向更上层任务组的传播
   * **这是自底向上传播的关键步骤**：子 `cfs_rq` 的负载变化，通过组实体 `se` 反映到父 `cfs_rq` 的负载中
6. 清理完全衰减的 `cfs_rq`
   * 四个条件全不满足，`cfs_rq` 才能从 `leaf_cfs_rq_list` 中移除
   * 目的：避免大量空闲 cgroup 的 `cfs_rq` 长期挂在链表上，导致每次遍历都做无用功
   * `_safe` 遍历保证了删除当前节点后，`pos` 仍然有效
7. 检查残留负载
   * 只要任何一个 `cfs_rq` 还有残留负载，就把 `*done` 置为 `false`
   * 上层 `__sched_balance_update_blocked_averages()` 据此决定是否保持 `rq->has_blocked_load = 1`
```cpp
#ifdef CONFIG_FAIR_GROUP_SCHED

static bool __update_blocked_fair(struct rq *rq, bool *done)
{
        struct cfs_rq *cfs_rq, *pos;
        bool decayed = false;
        //自底向上：由于 list_add_leaf_cfs_rq() 把子节点放在父节点之前，遍历顺序天然是从子到父
        //_safe 后缀：pos 保存下一个节点，允许在遍历中安全地删除当前节点（list_del_leaf_cfs_rq）
        /*    
         * Iterates the task_group tree in a bottom up fashion, see
         * list_add_leaf_cfs_rq() for details.
         */
        for_each_leaf_cfs_rq_safe(rq, cfs_rq, pos) {
                struct sched_entity *se;

                if (update_cfs_rq_load_avg(cfs_rq_clock_pelt(cfs_rq), cfs_rq)) {
                        update_tg_load_avg(cfs_rq);
                        //只对空闲的 cfs_rq 记录 throttled_pelt_idle，供后续 migrate_se_pelt_lag() 使用
                        if (cfs_rq->nr_queued == 0) 
                                update_idle_cfs_rq_clock_pelt(cfs_rq);
                        //标记根 cfs_rq 的衰减
                        if (cfs_rq == &rq->cfs)
                                decayed = true; 
                }     
                //更新组实体的 PELT 负载，并带 UPDATE_TG 标志触发向更上层任务组的传播
                /* Propagate pending load changes to the parent, if any: */
                se = cfs_rq_se(cfs_rq);
                if (se && !skip_blocked_update(se))
                        update_load_avg(cfs_rq_of(se), se, UPDATE_TG);
                //能够有许多 idle 的 CPU cgroup。不要让完全衰减的 cfs_rqs 在链表上逗留
                /*    
                 * There can be a lot of idle CPU cgroups.  Don't let fully
                 * decayed cfs_rqs linger on the list.
                 */
                if (cfs_rq_is_decayed(cfs_rq))
                        list_del_leaf_cfs_rq(cfs_rq);
                //一旦负载/平均利用率为空，不需要再周期性的衰减了
                /* Don't need periodic decay once load/util_avg are null */
                if (cfs_rq_has_blocked_load(cfs_rq))
                        *done = false;
        }

        return decayed;
}
...
#endif
```

### 更新 CFS 运行队列负载平均值
* `__signed_scalar_typeof()` 声明一个 **有符号** 的标量类型，无符号的标量类型转为有符号的，而非标量类型声明保持不变
```c
/*
 * __signed_scalar_typeof(x) - Declare a signed scalar type, leaving
 *                 non-scalar types unchanged.
 */

#define __scalar_type_to_signed_cases(type)             \
        unsigned type:  (signed type)0,             \
        signed type:    (signed type)0

#define __signed_scalar_typeof(x) typeof(               \
        _Generic((x),                       \
             char:  (signed char)0,             \   
             __scalar_type_to_signed_cases(char),       \
             __scalar_type_to_signed_cases(short),      \   
             __scalar_type_to_signed_cases(int),        \   
             __scalar_type_to_signed_cases(long),       \   
             __scalar_type_to_signed_cases(long long),  \
             default: (x)))
```
* `add_positive()` 宏做有符号的加法，并做下溢保护
```c
/*
 * Signed add and clamp on underflow.
 *
 * Explicitly do a load-store to ensure the intermediate value never hits
 * memory. This allows lockless observations without ever seeing the negative
 * values.
 */
#define add_positive(_ptr, _val) do {                           \
    typeof(_ptr) ptr = (_ptr);                              \
    __signed_scalar_typeof(*ptr) val = (_val);              \
    typeof(*ptr) res, var = READ_ONCE(*ptr);                \
                                \
    res = var + val;                                        \
                                \
    if (val < 0 && res > var)                               \
        res = 0;                                        \
                                \
    WRITE_ONCE(*ptr, res);                                  \
} while (0)
```
1. `add_positive(&(sa)->name##_avg, delta_avg)` 把增量加到平均值上。`add_positive()` 内部会做 `*ptr += delta` 并处理溢出保护
2. `add_positive(&(sa)->name##_sum, delta_sum)` 把增量加到总和上
3. 下界保护 `max_t()` 强制保证 `sum >= avg * PELT_MIN_DIVIDER`
```c
/*
 * Because of rounding, se->util_sum might ends up being +1 more than
 * cfs->util_sum. Although this is not a problem by itself, detaching
 * a lot of tasks with the rounding problem between 2 updates of
 * util_avg (~1ms) can make cfs->util_sum becoming null whereas
 * cfs_util_avg is not.
 *  
 * Check that util_sum is still above its lower bound for the new
 * util_avg. Given that period_contrib might have moved since the last
 * sync, we are only sure that util_sum must be above or equal to
 *    util_avg * minimum possible divider
 */     
#define __update_sa(sa, name, delta_avg, delta_sum) do {    \
    add_positive(&(sa)->name##_avg, delta_avg);     \
    add_positive(&(sa)->name##_sum, delta_sum);     \
    (sa)->name##_sum = max_t(typeof((sa)->name##_sum),  \
                   (sa)->name##_sum,        \
                   (sa)->name##_avg * PELT_MIN_DIVIDER); \
} while (0)
```
#### 为什么需要下界保护？
> 由于舍入误差，单个调度实体（`se`）的 `util_sum` 可能比 `cfs_rq` 的 `util_sum` 多出 `1`。这本身不是问题，但如果在两次 `util_avg` 更新之间（约 `1ms`）连续分离大量有舍入问题的任务，就会导致 `cfs->util_sum` 变成 `0`，而 cfs_util_avg 却不为 `0`
> 需要检查 `util_sum` 对于新的 `util_avg` 而言仍高于其下界。考虑到自上次同步后 `period_contrib` 可能已发生变化，我们仅能保证：`util_sum` 必须大于或等于 `util_avg * 最小可行除数`
* 具体场景：
  * 假设 1000 个小任务同时出队，每个任务的 `util_sum` 都因舍入比理论值大 `1`
  * 累加扣除时，`cfs_rq->util_sum` 被多扣了 1000
  * 结果 `util_sum` 变成负数或 `0`，但 `util_avg` 还是正值
  * 这会让 PELT 的数学关系崩溃（`avg = sum / divider` 不再成立）
* 解决方案：
  * 每次更新后，强制 `sum >= avg * PELT_MIN_DIVIDER`
  * `PELT_MIN_DIVIDER` 是 PELT 除数的最小可能值（对应最小时间窗口）
  * 这样即使舍入误差累积，`sum` 也不会低于理论下界

#### `add_tg_cfs_propagate()` 标记任务组需要传播
* CFS 的 cgroup 层级中，每个 `cfs_rq` 的负载变化需要向上汇总到父任务组。propagate 机制避免每次变化都立即向上传播，而是攒到一定量后统一处理
* `cfs_rq->propagate = 1` 标记这个 `cfs_rq` 的负载发生了变化，需要向上传播到 **任务组（task group）** 层级（即 cgroup 的 `tg_load_avg`）
```c
static inline void add_tg_cfs_propagate(struct cfs_rq *cfs_rq, long runnable_sum)
{
    cfs_rq->propagate = 1;
    cfs_rq->prop_runnable_sum += runnable_sum; //累加需要传播的 runnable_sum 增量
}
```
#### `update_cfs_rq_load_avg()` 更新 CFS 运行队列负载平均值
* 当任务从一个 `cfs_rq` 迁移到另一个 `cfs_rq` 时，它的负载不能立即从源 `cfs_rq` 扣除（因为 PELT 更新可能不在同一时刻）
  * Linux 采用延迟扣除：把要扣除的量暂存在 `cfs_rq->removed` 中，等下一次 `update_cfs_rq_load_avg()` 时统一处理
* 为什么用 runnable 估算 `load_sum`？
  * 因为 `load_sum` 需要乘任务权重，计算复杂
  * `runnable_sum` 是无权重的，计算简单，且与 `load_sum` 成比例关系。
  * 用比例估算可以在不增加计算量的前提下，保证任务组负载传播的近似正确
* `u64_u32_store_copy()` 与 `u64_u32_load_copy()` 配对使用，这是 32 位系统上实现 64 位无锁原子读的经典技巧
  * 在 32 位架构上，`u64` 的读和写都不是原子的，会拆成两条 32 位指令
    * 如果读者在写者写到一半时读取，可能读到“高 32 位是新值、低 32 位是旧值”的撕裂值（torn read） 
  * 用“写两次、读两次、比较重试”代替锁
  * 用内存屏障保证写者“先写主后写副”与读者“先读副后读主”的交叉配对，从而在任何并发场景下都不会返回撕裂值
* `last_update_time` 是 PELT 计算衰减的基准时间，如果这个值被读成撕裂值，时间差可能变成一个巨大或极小的数，导致：
  * 负载被错误地瞬间衰减到 `0`
  * 或者负载被错误地放大到天文数字
  * 这会直接摧毁 PELT 的数学正确性，进而影响负载均衡、CPUFreq 调频、任务放置等所有依赖 PELT 的决策
* `last_update_time` 的写入频率很低（每次 PELT 更新才写一次，约 `1ms` 一次），所以重试概率极低，这种无锁方案非常高效
```c
/**
 * update_cfs_rq_load_avg - update the cfs_rq's load/util averages
 * @now: current time, as per cfs_rq_clock_pelt()
 * @cfs_rq: cfs_rq to update
 *
 * The cfs_rq avg is the direct sum of all its entities (blocked and runnable)
 * avg. The immediate corollary is that all (fair) tasks must be attached.
 *
 * cfs_rq->avg is used for task_h_load() and update_cfs_share() for example.
 *
 * Return: true if the load decayed or we removed load.
 *
 * Since both these conditions indicate a changed cfs_rq->avg.load we should
 * call update_tg_load_avg() when this function returns true.
 */
static inline int 
update_cfs_rq_load_avg(u64 now, struct cfs_rq *cfs_rq)
{   
    unsigned long removed_load = 0, removed_util = 0, removed_runnable = 0;
    struct sched_avg *sa = &cfs_rq->avg;
    int decayed = 0;

    if (cfs_rq->removed.nr) {
        unsigned long r;
        u32 divider = get_pelt_divider(&cfs_rq->avg); //得到 PELT 除数，用于把 avg 换算成 sum
        //用 swap() 原子地把 .removed 中的值取出到局部变量，同时把 .removed 清零
        raw_spin_lock(&cfs_rq->removed.lock);
        swap(cfs_rq->removed.util_avg, removed_util);
        swap(cfs_rq->removed.load_avg, removed_load);
        swap(cfs_rq->removed.runnable_avg, removed_runnable);
        cfs_rq->removed.nr = 0;
        raw_spin_unlock(&cfs_rq->removed.lock);
        //对 load、util、runnable 三个指标，分别调用 __update_sa() 扣除（第 3 和 4 个参数都是负的）
        r = removed_load;
        __update_sa(sa, load, -r, -r*divider);

        r = removed_util;
        __update_sa(sa, util, -r, -r*divider);

        r = removed_runnable;
        __update_sa(sa, runnable, -r, -r*divider);
        //removed_runnable 是 removed_load 的无权重版本（runnable 不乘任务权重，load 乘）
        //用 removed_runnable * divider 再右移 SCHED_CAPACITY_SHIFT（通常是 10），估算出 removed_load_sum 的量级
        //调用 add_tg_cfs_propagate() 标记需要向上传播（负值表示扣除）
        /*
         * removed_runnable is the unweighted version of removed_load so we
         * can use it to estimate removed_load_sum.
         */
        add_tg_cfs_propagate(cfs_rq,
            -(long)(removed_runnable * divider) >> SCHED_CAPACITY_SHIFT);
        //只要处理了 removed 路径，就认为发生了负载变化，decayed 置 1
        decayed = 1;
    }
    //根据时间差 now - sa->last_update_time，对 load_avg、util_avg、runnable_avg 及其 sum 做指数衰减，见 sched_pelt.md
    decayed |= __update_load_avg_cfs_rq(now, cfs_rq);
    u64_u32_store_copy(sa->last_update_time,
               cfs_rq->last_update_time_copy,
               sa->last_update_time);
    return decayed; //返回 true 表示本次更新发生了衰减或负载移除
}
```

#### 通知 CPU 调速器
* schedutil 调速器在初始化时（`sugov_start()` / `sugov_register()`）通过 `cpufreq_set_update_util_data()` 把 `sugov_update_single_freq` 或 `sugov_update_shared` 等回调函数挂到 `cpufreq_update_util_data` 这个 per-CPU 指针上
* 调度器不需要知道具体是哪个调速器（schedutil、ondemand、performance……），只需要调用一个函数指针。这是一种策略与机制分离的设计——调度器提供“利用率变化”这个机制，调速器决定“如何调频”这个策略
* `cpufreq_update_util()` 通知（记录）CPU 利用率的改变
```cpp
DECLARE_PER_CPU(struct update_util_data __rcu *, cpufreq_update_util_data);

/**
 * cpufreq_update_util - Take a note about CPU utilization changes.
 * @rq: Runqueue to carry out the update for.
 * @flags: Update reason flags.
 *
 * This function is called by the scheduler on the CPU whose utilization is
 * being updated.
 * 调度器会在利用率发生变更的那个 CPU 上调用。
 * 
 * It can only be called from RCU-sched read-side critical sections.
 * 本函数仅允许在 RCU-sched 读端临界区内调用。
 * 
 * The way cpufreq is currently arranged requires it to evaluate the CPU
 * performance state (frequency/voltage) on a regular basis to prevent it from
 * being stuck in a completely inadequate performance level for too long.
 * That is not guaranteed to happen if the updates are only triggered from CFS
 * and DL, though, because they may not be coming in if only RT tasks are
 * active all the time (or there are RT tasks only).
 * 当前 cpufreq 框架的实现机制，需要定期评估 CPU 的性能状态（频率 / 电压），防止 CPU 长时间卡在完全不匹配负载需求的性能档位。
 * 但如果仅依靠 CFS 和 DL 调度类触发更新，并不能保证这个评估一定会执行：倘若系统中一直只有 RT 任务在运行，就不会产生这类更新事件。
 * 
 * As a workaround for that issue, this function is called periodically by the
 * RT sched class to trigger extra cpufreq updates to prevent it from stalling,
 * but that really is a band-aid.  Going forward it should be replaced with
 * solutions targeted more specifically at RT tasks.
 * 作为该问题的临时补救方案，RT 调度类会周期性调用本函数，额外触发 cpufreq 更新，避免调频逻辑停滞，但这只是一个权宜之计（创可贴）。
 * 后续应当替换为专门针对 RT 任务设计的方案。
 */
static inline void cpufreq_update_util(struct rq *rq, unsigned int flags)
{
        struct update_util_data *data;

        data = rcu_dereference_sched(*per_cpu_ptr(&cpufreq_update_util_data,
                                                  cpu_of(rq)));
        if (data)
                data->func(data, rq_clock(rq), flags);
}
```
* 为什么只有根 cfs_rq（`&rq->cfs`）的利用率变化才通知调速器？
  * CPU 频率是整个 CPU 共享的资源，不是某个 cgroup 私有的。调速器需要的是“这个 CPU 上所有 CFS 任务的总利用率”，而这个聚合值只存在于根 cfs_rq 的 `avg.util_avg` 中
  * 子 cfs_rq（cgroup 层级）的利用率变化，会通过组实体 `se` 向上传播，最终反映到根 cfs_rq 的 `util_avg` 上。在传播到根之前，通知调速器是没有意义的——调速器看到的还是旧值。
* `cfs_rq_util_change()` 函数会忽略的三个边界情况：
1. 进入 idle 时不会触发
   * 因为 idle 线程属于 `!fair` 调度类，不经过 CFS 的 `cfs_rq_util_change()`。所以当 CPU 变空闲时，CFS 的 `util_avg` 衰减到 `0`，但调速器不会收到通知，可能维持高频一段时间
   * 实际上，idle 路径有其他的调频触发机制，比如 `cpufreq_update_util()` 在 idle 进入/退出时的显式调用，但 CFS 这条路径确实覆盖不到
2. `util` 不包含 RT 任务
   * `cfs_rq->avg.util_avg` 只统计 CFS 任务的利用率。如果 CPU 上跑着 RT 任务，CFS 的 `util` 可能是 `0`，但 CPU 实际很忙
   * 这就是为什么 RT 调度类需要自己调用 `cpufreq_update_util()` 并带 `SCHED_CPUFREQ_RT` 标志
3. `util` 不是频率不变的（not freq-invariant）
   * PELT 的 `util_avg` 是在当前频率下测得的。如果 CPU 降频，任务跑得慢，`util_avg` 反而会升高（因为任务占用的时间比例变大了）
   * 要实现真正的“频率不变”，需要 `arch_scale_freq_capacity()` 对 `util_avg` 做归一化
   * 注释说“we'd have to implement arch_scale_freq_capacity() for that” — 这取决于具体架构是否提供了这个钩子
```cpp
static inline void cfs_rq_util_change(struct cfs_rq *cfs_rq, int flags)
{
        struct rq *rq = rq_of(cfs_rq);

        if (&rq->cfs == cfs_rq) {
                /*
                 * There are a few boundary cases this might miss but it should
                 * get called often enough that that should (hopefully) not be
                 * a real problem.
                 *
                 * It will not get called when we go idle, because the idle
                 * thread is a different class (!fair), nor will the utilization
                 * number include things like RT tasks.
                 *
                 * As is, the util number is not freq-invariant (we'd have to
                 * implement arch_scale_freq_capacity() for that). 
                 *
                 * See cpu_util_cfs().
                 */
                cpufreq_update_util(rq, flags);
        }
}
```
#### 进出 `cfs_rq` 的操作
* 为什么 `load_sum` 要乘 `se_weight(se)`？
  * 这是 PELT 中 load 和 util/runnable 的关键区别：
  * `util_sum / runnable_sum`：是无权重的，只反映“时间占比”。直接加减即可
  * `load_sum`：是有权重的。`se->avg.load_sum` 存的是“未乘权重的原始值”，而 cfs_rq 的 `load_sum` 需要的是“乘上任务权重后的值”
  * 因此，在把 `se` 的 `load` 加到 cfs_rq 时，必须乘以 `se_weight(se)`（任务权重，即 `se->load.weight`）
```cpp
enqueue_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
        __update_sa(&cfs_rq->avg, load, se->avg.load_avg,
                    se_weight(se) * se->avg.load_sum);
}

static inline void
dequeue_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
        __update_sa(&cfs_rq->avg, load, -se->avg.load_avg,
                    se_weight(se) * -se->avg.load_sum);
}
```
##### `attach_entity_load_avg()`：把 `se` 挂到 `cfs_rq` 上
* 这个函数在任务迁移到新 CPU / 新 cgroup，或新建任务首次入队时调用
  * 它的目标：让 `se` 的 PELT 状态与 `cfs_rq` 对齐，然后把它加到 `cfs_rq` 的聚合统计中
* 调用者必须先调用 `update_cfs_rq_load_avg()` 更新 `cfs_rq` 的 PELT，保证 `cfs_rq->avg.last_update_time` 是当前时刻。否则对齐会基于过时的时间戳
###### 为什么必须重算 `*_sum` ？
* 因为 `period_contrib` 变了。`sum` 和 `avg` 的关系是 `avg = sum / divider`，而 `divider` 依赖于 `period_contrib`。旧的 `sum` 是基于旧的 `period_contrib` 算出来的，直接拿到新窗口里用会不一致
  * `util_sum / runnable_sum`：直接 `avg * divider`，因为它们是 **无权重** 的
  * `load_sum`：先 `avg * divider` 得到加权值，再 **除以** `se_weight(se)` 反推回无权重值（因为 `se` 的 `load_sum` 存的是无权重值）
  * `if (se_weight(se) < se->avg.load_sum)` 的保护：
     * 如果 `load_sum` 太小（小于权重），除法会得到 `0`，导致后续 `avg = sum * weight / divider` 算出 `0`
     * 此时强制设为 `1`，保证 `load_sum` 至少为 `1`，避免负载被错误清零
     * 这是一个防御性下界，与 `__update_sa()` 的下界保护异曲同工
* 重算 `_sum` 会引入截断误差，不是严格精确的，但此时实体不在 PELT 层级中（刚附着，还没参与衰减），**误差会被后续的正常衰减吸收，影响可忽略**
```cpp
/**
 * attach_entity_load_avg - attach this entity to its cfs_rq load avg
 * @cfs_rq: cfs_rq to attach to
 * @se: sched_entity to attach
 *
 * Must call update_cfs_rq_load_avg() before this, since we rely on
 * cfs_rq->avg.last_update_time being current.
 */
static void attach_entity_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
        /*
         * cfs_rq->avg.period_contrib can be used for both cfs_rq and se.
         * See ___update_load_avg() for details.
         */
        u32 divider = get_pelt_divider(&cfs_rq->avg);
        //- last_update_time 对齐：让 se 和 cfs_rq 的 PELT 时间轴从同一时刻开始
        //- period_contrib 对齐：PELT 的衰减是按固定周期（LOAD_AVG_PERIOD）进行的，period_contrib 记录“当前周期已经过了多少”
        //  如果 se 和 cfs_rq 的 period_contrib 不同，它们的衰减相位就不同，直接加减会导致误差
        //- 不同步会导致负载值在迁移后出现非物理的跳变
        /*
         * When we attach the @se to the @cfs_rq, we must align the decay
         * window because without that, really weird and wonderful things can
         * happen.
         *
         * XXX illustrate
         */
        se->avg.last_update_time = cfs_rq->avg.last_update_time;
        se->avg.period_contrib = cfs_rq->avg.period_contrib;

        /*    
         * Hell(o) Nasty stuff.. we need to recompute _sum based on the new
         * period_contrib. This isn't strictly correct, but since we're
         * entirely outside of the PELT hierarchy, nobody cares if we truncate
         * _sum a little.
         */
        se->avg.util_sum = se->avg.util_avg * divider;

        se->avg.runnable_sum = se->avg.runnable_avg * divider;

        se->avg.load_sum = se->avg.load_avg * divider;
        if (se_weight(se) < se->avg.load_sum)
                se->avg.load_sum = div_u64(se->avg.load_sum, se_weight(se));
        else
                se->avg.load_sum = 1;
        //注意顺序：先对齐、重算 _sum，再累加。如果先累加再重算，cfs_rq 的聚合值会被污染
        enqueue_load_avg(cfs_rq, se);
        cfs_rq->avg.util_avg += se->avg.util_avg;
        cfs_rq->avg.util_sum += se->avg.util_sum;
        cfs_rq->avg.runnable_avg += se->avg.runnable_avg;
        cfs_rq->avg.runnable_sum += se->avg.runnable_sum;
        //告诉 cfs_rq：“我的负载变了，需要向上传播到任务组（tg->load_avg）”
        add_tg_cfs_propagate(cfs_rq, se->avg.load_sum);
        //如果这是根 cfs_rq，通知 CPUFreq 调速器“利用率变了，请重新评估频率”
        cfs_rq_util_change(cfs_rq, 0);

        trace_pelt_cfs_tp(cfs_rq);
}
```
##### `detach_entity_load_avg()`：把 `se` 从 `cfs_rq` 上摘下来
###### 为什么 detach 不需要重算 `*_sum`？
* attach 时需要重算，是因为 `period_contrib` 刚被对齐，旧的 `_sum` 与新窗口不匹配
* detach 时，`se` 已经在 `cfs_rq` 的窗口里待了一段时间，它的 `_sum` 和 `cfs_rq` 的窗口是一致的。直接扣除 `se->avg.util_sum` 等即可
* 但扣除仍然走 `__update_sa()`，享受下界保护，防止舍入误差累积导致 `cfs_rq` 的 `_sum` 崩溃
```cpp
/**
 * detach_entity_load_avg - detach this entity from its cfs_rq load avg
 * @cfs_rq: cfs_rq to detach from
 * @se: sched_entity to detach
 *
 * Must call update_cfs_rq_load_avg() before this, since we rely on
 * cfs_rq->avg.last_update_time being current.
 */
static void detach_entity_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
        dequeue_load_avg(cfs_rq, se);
        __update_sa(&cfs_rq->avg, util, -se->avg.util_avg, -se->avg.util_sum);
        __update_sa(&cfs_rq->avg, runnable, -se->avg.runnable_avg, -se->avg.runnable_sum);

        add_tg_cfs_propagate(cfs_rq, -se->avg.load_sum);

        cfs_rq_util_change(cfs_rq, 0);

        trace_pelt_cfs_tp(cfs_rq);
}
```

##### task group（cgroup）层级中 PELT 负载的自底向上传播

```cpp
/*
 * When on migration a sched_entity joins/leaves the PELT hierarchy, we need to
 * propagate its contribution. The key to this propagation is the invariant
 * that for each group:
 * 当调度实体（sched_entity）在迁移时加入 / 退出 PELT 层级树，我们需要传播它的统计贡献。这个传播逻辑的核心不变量是：对于每一个调度组：
 * 
 *   ge->avg == grq->avg                                                (1)
 *
 * _IFF_ we look at the pure running and runnable sums. Because they
 * represent the very same entity, just at different points in the hierarchy.
 * 前提条件：我们只看纯粹的运行态与可运行态统计和。因为二者描述的是同一个对象，只是处在层级树里不同位置上。
 *
 * Per the above update_tg_cfs_util() and update_tg_cfs_runnable() are trivial
 * and simply copies the running/runnable sum over (but still wrong, because
 * the group entity and group rq do not have their PELT windows aligned).
 * 基于上面这条不变量，update_tg_cfs_util() 和 update_tg_cfs_runnable() 的逻辑看起来很简单，直接拷贝运行态 / 可运行态统计和即可。
 * 但这种做法依然是错的：组调度实体（group entity）和组运行队列（group rq）的 PELT 时间窗口并没有对齐。
 *
 * However, update_tg_cfs_load() is more complex. So we have:
 * 而 update_tg_cfs_load() 的逻辑则更加复杂。我们已知：
 *
 *   ge->avg.load_avg = ge->load.weight * ge->avg.runnable_avg          (2)
 *
 * And since, like util, the runnable part should be directly transferable,
 * the following would _appear_ to be the straight forward approach:
 * 和利用率 util 类似，可运行部分看上去应该可以直接传递，于是很自然会想到下面这个式子：
 *
 *   grq->avg.load_avg = grq->load.weight * grq->avg.runnable_avg       (3)
 *
 * And per (1) we have:
 * 再结合式 (1) 的推论：
 *
 *   ge->avg.runnable_avg == grq->avg.runnable_avg
 *
 * Which gives:
 * 代入得到：
 *
 *                      ge->load.weight * grq->avg.load_avg
 *   ge->avg.load_avg = -----------------------------------             (4)
 *                               grq->load.weight
 *
 * Except that is wrong!
 * 但是，这个推导是错误的！
 *
 * Because while for entities historical weight is not important and we
 * really only care about our future and therefore can consider a pure
 * runnable sum, runqueues can NOT do this.
 * 原因在于：对于调度实体来说，历史权重并不重要，我们只关心未来的负载表现，因此可以只用纯粹的可运行态统计和来计算；但运行队列不能这么做。
 *
 * We specifically want runqueues to have a load_avg that includes
 * historical weights. Those represent the blocked load, the load we expect
 * to (shortly) return to us. This only works by keeping the weights as
 * integral part of the sum. We therefore cannot decompose as per (3).
 * 我们特意要求运行队列的 load_avg 要包含历史权重信息。这部分对应的是阻塞负载（blocked load），也就是预期很快会重新回到本队列的负载。
 * 这个统计成立的前提就是把权重作为统计和中不可拆分的一部分，因此不能像公式 (3) 那样做分解。
 *
 * Another reason this doesn't work is that runnable isn't a 0-sum entity.
 * Imagine a rq with 2 tasks that each are runnable 2/3 of the time. Then the
 * rq itself is runnable anywhere between 2/3 and 1 depending on how the
 * runnable section of these tasks overlap (or not). If they were to perfectly
 * align the rq as a whole would be runnable 2/3 of the time. If however we
 * always have at least 1 runnable task, the rq as a whole is always runnable.
 * 该方案行不通还有另一个理由：可运行态不是零和统计量。
 * 举个例子：一个运行队列上有两个任务，每个任务平均有 2/3 的时间处于可运行状态。
 * 那么整个队列整体处于可运行的时间占比，会在 2/3 ~ 1 之间浮动，取决于两个任务的可运行时间段是否重叠：
 * - 如果二者可运行区间完全重合，整个队列的可运行占比就是 2/3；
 * - 如果任意时刻至少有一个任务是可运行的，那么整个队列就永远处于可运行状态。
 *
 * So we'll have to approximate.. :/
 * 所以我们只能做近似处理.. :/
 *  * Given the constraint:
 * 给定约束条件：
 * 
 *   ge->avg.running_sum <= ge->avg.runnable_sum <= LOAD_AVG_MAX
 *
 * We can construct a rule that adds runnable to a rq by assuming minimal
 * overlap.
 * 我们可以构造一条规则：向运行队列新增可运行负载时，假设任务之间的可运行区间重叠最小；
 * On removal, we'll assume each task is equally runnable; which yields:
 * 移除任务时，则假设所有任务的可运行占比均等，由此得到：
 *
 *    grq->avg.runnable_sum = grq->avg.load_sum / grq->load.weight
 *
 * XXX: only do this for the part of runnable > running ?
 * 是否只对 runnable > running 的这部分负载采用该近似？
 */
```
###### util/runnable 为什么可以直接复制？
* 因为 `util_sum` 是无权重的，代表纯时间占比。`se` 和 `gcfs_rq` 代表同一个实体，`util` 值应该完全一致
* 唯一的差异是 PELT 窗口（`period_contrib`）可能不同，所以需要按父窗口重算 `sum`，但 `avg` 本身可以直接复制
```cpp
static inline void
update_tg_cfs_util(struct cfs_rq *cfs_rq, struct sched_entity *se, struct cfs_rq *gcfs_rq)
{       //计算 delta：gcfs_rq 的新 util_avg 减去 se 的旧 util_avg，就是需要传播的变化量
        long delta_sum, delta_avg = gcfs_rq->avg.util_avg - se->avg.util_avg;
        u32 new_sum, divider;
        //如果 delta 为 0：直接返回，无事可做
        /* Nothing to update */
        if (!delta_avg)
                return;
        //获取 divider：用父 cfs_rq 的 divider（因为要把 se 的统计对齐到父窗口）
        /*
         * cfs_rq->avg.period_contrib can be used for both cfs_rq and se.
         * See ___update_load_avg() for details.
         */
        divider = get_pelt_divider(&cfs_rq->avg);

        /* Set new sched_entity's utilization */
        se->avg.util_avg = gcfs_rq->avg.util_avg; //直接复制
        new_sum = se->avg.util_avg * divider;     //按父窗口重算 sum
        delta_sum = (long)new_sum - (long)se->avg.util_sum; //计算 sum 的增量
        se->avg.util_sum = new_sum;               //更新 sum
        //更新父 cfs_rq：把 (delta_avg, delta_sum) 加到父 cfs_rq 的 util 上，享受下界保护
        /* Update parent cfs_rq utilization */
        __update_sa(&cfs_rq->avg, util, delta_avg, delta_sum);
}

static inline void
update_tg_cfs_runnable(struct cfs_rq *cfs_rq, struct sched_entity *se, struct cfs_rq *gcfs_rq)
{
        long delta_sum, delta_avg = gcfs_rq->avg.runnable_avg - se->avg.runnable_avg;
        u64 new_sum;
        u32 divider;

        /* Nothing to update */
        if (!delta_avg)
                return;

        /*
         * cfs_rq->avg.period_contrib can be used for both cfs_rq and se.
         * See ___update_load_avg() for details.
         */
        divider = get_pelt_divider(&cfs_rq->avg);

        /* Set new sched_entity's runnable */
        se->avg.runnable_avg = gcfs_rq->avg.runnable_avg;
        new_sum = (u64)se->avg.runnable_avg * divider;
        delta_sum = (long)new_sum - (long)se->avg.runnable_sum;
        se->avg.runnable_sum = new_sum;

        /* Update parent cfs_rq runnable */
        __update_sa(&cfs_rq->avg, runnable, delta_avg, delta_sum);
}
```
###### load 为什么不能直接复制？
* 对比 util/runnable 的直接复制，load 必须走近似，因为：
  * **权重必须保留**：父 `rq` 的 `load_sum` 包含历史权重（blocked load），不能分解成 `weight * runnable_sum`
  * **runnable 非 0-sum**：多个任务的 runnable 不能简单相加，必须用“最小重叠”/“均等 runnable”的假设来近似
  * **`prop_runnable_sum` 是无权重的变化量**：它只告诉我们“runnable 时间占比变了多少”，要转换成加权 `load`，必须乘 `se_weight(se)`
```cpp
static inline void
update_tg_cfs_load(struct cfs_rq *cfs_rq, struct sched_entity *se, struct cfs_rq *gcfs_rq)
{       //gcfs_rq->prop_runnable_sum 是 gcfs_rq 累积的“待传播 runnable 变化量”，由 add_tg_cfs_propagate() 在任务入队/出队/迁移时累加
        long delta_avg, running_sum, runnable_sum = gcfs_rq->prop_runnable_sum;
        unsigned long load_avg;
        u64 load_sum = 0;
        s64 delta_sum;
        u32 divider;

        if (!runnable_sum)
                return;
        //处理完立即清零，避免重复传播
        gcfs_rq->prop_runnable_sum = 0;

        /*
         * cfs_rq->avg.period_contrib can be used for both cfs_rq and se.
         * See ___update_load_avg() for details.
         */
        divider = get_pelt_divider(&cfs_rq->avg);
        //正数表示净增加，负数表示净减少
        if (runnable_sum >= 0) {
                //把新增量加到 se 当前的 load_sum（无权重）上。
                //clip 在 divider（即 LOAD_AVG_MAX）：这反映了“CPU 饱和之前，running == runnable”。
                //如果累加超过最大值，说明 CPU 已经饱和，runnable 不可能超过 100% 时间。
                //这对应注释中的近似规则：假设最小重叠（新增任务尽量填补空闲，让 rq 更 runnable）
                /*
                 * Add runnable; clip at LOAD_AVG_MAX. Reflects that until
                 * the CPU is saturated running == runnable.
                 */
                runnable_sum += se->avg.load_sum;
                runnable_sum = min_t(long, runnable_sum, divider);
        } else {
                //用 gcfs_rq->avg.load_sum / gcfs_rq->load.weight 估算每个任务的平均无权重 runnable（假设所有任务 equally runnable）
                //取 min(se->avg.load_sum, load_sum)：
                //- 不能超过 se 当前的无权重 load_sum（否则会“膨胀” se 的 runnable，违背减少的语义）
                //- 也不能超过估算出的每个任务平均 runnable
                //这对应注释中的近似规则：移除时假设每个任务 equally runnable
                /*
                 * Estimate the new unweighted runnable_sum of the gcfs_rq by
                 * assuming all tasks are equally runnable.
                 */
                if (scale_load_down(gcfs_rq->load.weight)) {
                        load_sum = div_u64(gcfs_rq->avg.load_sum,
                                scale_load_down(gcfs_rq->load.weight));
                }

                /* But make sure to not inflate se's runnable */
                runnable_sum = min(se->avg.load_sum, load_sum);
        }
        //util_sum 是 running_sum 按 SCHED_CAPACITY_SHIFT（通常是 10）放大后的值，右移还原到与 runnable_sum 同量级
        /*
         * runnable_sum can't be lower than running_sum
         * Rescale running sum to be in the same range as runnable sum
         * running_sum is in [0 : LOAD_AVG_MAX <<  SCHED_CAPACITY_SHIFT]
         * runnable_sum is in [0 : LOAD_AVG_MAX]
         */
        running_sum = se->avg.util_sum >> SCHED_CAPACITY_SHIFT;
        //物理约束：running 是 runnable 的子集，所以 runnable_sum >= running_sum
        //用 max 强制满足这个约束，防止近似算法产生非物理结果
        runnable_sum = max(runnable_sum, running_sum);
        //runnable_sum 是无权重的，乘上 se_weight(se) 得到父 cfs_rq 视角的加权 load_sum
        load_sum = se_weight(se) * runnable_sum;
        load_avg = div_u64(load_sum, divider); //avg = 加权sum / divider，新的加权负载平均值
        //avg 增量：新 avg 减旧 avg
        delta_avg = load_avg - se->avg.load_avg;
        if (!delta_avg) //但注意此时 se->avg.load_sum 可能仍需更新？
                return; //注释中这里直接 return，意味着 avg 不变时 sum 的微小变化被忽略，这是可接受的近似
        //sum 增量：新加权 sum 减旧加权 sum（注意 se->avg.load_sum 存的是无权重值，所以要先乘 se_weight(se)）
        delta_sum = load_sum - (s64)se_weight(se) * se->avg.load_sum;

        se->avg.load_sum = runnable_sum; //存无权重值
        se->avg.load_avg = load_avg;     //存加权平均
        __update_sa(&cfs_rq->avg, load, delta_avg, delta_sum); //更新父 cfs_rq
}
```
##### `propagate_entity_load_avg()`：传播的调度器
```cpp
/* Update task and its cfs_rq load average */
static inline int propagate_entity_load_avg(struct sched_entity *se)
{
        struct cfs_rq *cfs_rq, *gcfs_rq;
        //普通任务（task entity）没有子 cfs_rq，不需要传播，只有组实体（group entity）才代表一个 cgroup，才有 gcfs_rq
        if (entity_is_task(se))
                return 0;
        //gcfs_rq->propagate 由 add_tg_cfs_propagate() 置位，表示“这个 cfs_rq 有负载变化待传播”
        gcfs_rq = group_cfs_rq(se);
        if (!gcfs_rq->propagate)
                return 0; //如果没有，直接返回
        //有则先清零，防止重复传播
        gcfs_rq->propagate = 0;

        cfs_rq = cfs_rq_of(se);
        //把 gcfs_rq 的 prop_runnable_sum 累加到父 cfs_rq 的 prop_runnable_sum 上
        //这样父 cfs_rq 也被标记为“需要传播”，在它自己的 propagate_entity_load_avg() 被调用时，会继续向上传播
        //这是递归传播链的关键：一次变化可以从最底层 cgroup 一直传播到根 cfs_rq
        add_tg_cfs_propagate(cfs_rq, gcfs_rq->prop_runnable_sum);
        //依次更新三个维度，顺序：util → runnable → load
        //因为 update_tg_cfs_load() 会用到 se->avg.util_sum（计算 running_sum），所以 util 必须先更新
        update_tg_cfs_util(cfs_rq, se, gcfs_rq);
        //runnable 在 load 之前更新，因为 load 的近似会用到 runnable 的信息
        update_tg_cfs_runnable(cfs_rq, se, gcfs_rq);
        update_tg_cfs_load(cfs_rq, se, gcfs_rq);

        trace_pelt_cfs_tp(cfs_rq);
        trace_pelt_se_tp(se);
        //返回 1 表示“确实传播了”，调用者（update_load_avg()）据此决定是否需要继续处理父层
        return 1;
}
```
#### `util_est_update()`：利用率估计
* 函数 `util_est_update()` 是 Linux 调度器中 **利用率估计（utilization estimation，简称 util_est）** 机制的核心更新逻辑
* 它在任务 **出队（dequeue，即任务睡眠）** 时被调用，目的是用一个 **指数加权移动平均（Exponential Weighted Moving Average，EWMA）** 来平滑任务的利用率，从而解决 PELT 在任务频繁唤醒/睡眠场景下的 **低估问题**
##### 背景：为什么需要 util_est？
###### PELT 的“低估”问题
* PELT 的 `util_avg` 是任务利用率的 **历史平均**。考虑一个周期性任务：
  * 它每 16ms 唤醒一次，运行 2ms，然后睡眠 14ms
  * 每次唤醒时，PELT 的 `util_avg` 需要时间爬升（因为它是指数衰减的）
  * 如果任务在 `util_avg` 爬升到真实需求之前就再次睡眠，调度器会低估这个任务的需求
  * 结果：调度器可能把这个任务放在小核上，导致性能不足；或者 CPUFreq 降频，导致卡顿
###### util_est 的解法
* util_est 在每次任务 **出队** 时，记录当时的 `util_avg` 作为一次“采样”，然后维护一个 EWMA：
  ```c
  util_est(t) = w * util_avg(t) + (1-w) * util_est(t-1)
  ```
* `w` 是新样本的权重（内核中配置为 `1/4`，即 `UTIL_EST_WEIGHT_SHIFT = 2`），因此 EWMA 的平滑函数为
  ```c
  ewma = 0.25*dequeued + 0.75*ewma
  ```
* 这个 EWMA 能“记住”任务在上次激活期间的真实利用率，即使它很快又睡眠了
* 调度器在放置任务、计算 CPU 需求时，会取 `max(util_avg, util_est)` 作为任务的利用率估计，从而避免低估

##### `UTIL_AVG_UNCHANGED` 标志的含义
* `UTIL_AVG_UNCHANGED` 标志的含义：自上次出队以来，这个 `se` 的 PELT 值（`util_avg`）**没有发生变化**
* 什么情况下会这样？任务被唤醒后，还没来得及运行任何时间（或者运行时间极短，不足以让 PELT 更新），就又睡眠了
  * 此时 `util_avg` 还是旧值，拿它做 EWMA 采样没有意义，直接跳过
* 这个标志由 `util_est_enqueue()` 在入队时检查并设置：如果入队时发现 `util_avg` 与上次出队时相同，就保留 `UTIL_AVG_UNCHANGED` 标志，告诉出队路径“这次不用更新了”
* 由 [`__update_load_avg_se()` 更新调度实体负载的时候清除](sched_pelt.md#清除标志cfs_se_util_change)
```cpp
static inline void util_est_update(struct sched_entity *se)
{
        unsigned int ewma, dequeued, last_ewma_diff;
        //如果未启用 UTIL_EST 调度特性开关（sched_feat），直接返回，零开销
        if (!sched_feat(UTIL_EST))
                return;
        //读取当前的 EWMA 估计值。注意：这个字段的最低位被用作标志位 UTIL_AVG_UNCHANGED，所以 ewma 的值实际上包含了标志位
        /* Get current estimate of utilization */
        ewma = READ_ONCE(se->avg.util_est);
        //跳过“PELT 未变化”的情况
        /*
         * If the PELT values haven't changed since enqueue time,
         * skip the util_est update.
         */
        if (ewma & UTIL_AVG_UNCHANGED)
                return;
        //获取任务出队那一刻的 PELT 利用率，作为本次 EWMA 的“新样本”
        /* Get utilization at dequeue */
        dequeued = READ_ONCE(se->avg.util_avg);
        //利用率上升时，重置 EWMA：如果当前 EWMA 估计值小于等于本次采样值，说明任务的需求在上升
        //策略：EWMA 只用于平滑下降，不用于平滑上升
        //- 上升时直接重置为 dequeued，让调度器立即感知到任务变重了，快速做出反应（比如迁到大核）
        //- 如果也用 EWMA 平滑上升，会导致调度器对任务变重的反应滞后，影响性能
        /*
         * Reset EWMA on utilization increases, the moving average is used only
         * to smooth utilization decreases.
         */
        if (ewma <= dequeued) {
                ewma = dequeued;
                goto done;
        }
        //跳过“差异过小”的情况：
        //last_ewma_diff = ewma - dequeued：EWMA 与本次采样的差值（此时已确保 ewma > dequeued，即需求在下降）
        //UTIL_EST_MARGIN 是一个很小的阈值（约 1% 的容量，即 SCHED_CAPACITY_SCALE / 100 左右）
        //如果差值小于这个阈值，说明变化太小，不值得更新 EWMA，直接跳过
        //目的：避免微小波动导致 EWMA 频繁抖动，减少无意义的计算
        /* 
         * Skip update of task's estimated utilization when its members are
         * already ~1% close to its last activation value.
         */
        last_ewma_diff = ewma - dequeued;
        if (last_ewma_diff < UTIL_EST_MARGIN)
                goto done;
        //关键保护：跳过“CPU 时间不足”的情况：这是整个函数中最重要的保护逻辑
        //runnable_avg：任务“想要运行”的时间占比（包括实际运行和被抢占等待的时间）
        //dequeued（即 util_avg）：任务“实际运行”的时间占比
        //如果 runnable_avg 明显大于 util_avg（超过一个 margin），说明任务在这段时间里被其他任务抢占了 CPU，它想要运行但没得到足够的时间
        //此时 util_avg 低估了任务的真实需求。如果拿这个低估的值去更新 EWMA，会让 EWMA 也偏低，进一步加剧低估
        //策略：跳过本次更新，等任务下次获得足够 CPU 时间时再采样
        /*
         * To avoid underestimate of task utilization, skip updates of EWMA if
         * we cannot grant that thread got all CPU time it wanted.
         */
        if ((dequeued + UTIL_EST_MARGIN) < READ_ONCE(se->avg.runnable_avg))
                goto done;
        //完全等价于标准 EWMA 公式。用移位代替浮点乘除，是为了在调度器热路径上保持高效
        /*
         * Update Task's estimated utilization
         *
         * When *p completes an activation we can consolidate another sample
         * of the task size. This is done by using this value to update the
         * Exponential Weighted Moving Average (EWMA):
         *
         *  ewma(t) = w *  task_util(p) + (1-w) * ewma(t-1)
         *          = w *  task_util(p) +         ewma(t-1)  - w * ewma(t-1)
         *          = w * (task_util(p) -         ewma(t-1)) +     ewma(t-1)
         *          = w * (      -last_ewma_diff           ) +     ewma(t-1)
         *          = w * (-last_ewma_diff +  ewma(t-1) / w)
         *
         * Where 'w' is the weight of new samples, which is configured to be
         * 0.25, thus making w=1/4 ( >>= UTIL_EST_WEIGHT_SHIFT)
         */
        ewma <<= UTIL_EST_WEIGHT_SHIFT; //ewma = 4 * ewma(t-1) 
        ewma  -= last_ewma_diff;        //ewma = 4 * ewma(t-1) - last_ewma_diff
        ewma >>= UTIL_EST_WEIGHT_SHIFT; //ewma = (4 * ewma(t-1) - last_ewma_diff) / 4
                                        //     = ewma(t-1) - last_ewma_diff / 4
                                        //     = ewma(t-1) + w * (dequeued - ewma(t-1))
                                        //     = w * dequeued + (1-w) * ewma(t-1)
done:
        ewma |= UTIL_AVG_UNCHANGED; //设置标志位，表示“这个 EWMA 值已经与当前的 PELT 状态同步了”
        WRITE_ONCE(se->avg.util_est, ewma); //更新利用率估计，保证写入的原子性和可见性

        trace_sched_util_est_se_tp(se);
}
```
#### `update_load_avg()`：PELT 负载追踪的总调度枢纽
* `update_load_avg()` 是 CFS 中 PELT 负载追踪的总调度枢纽
  * 它被任务入队、出队、周期性 tick、任务迁移等多条路径调用
  * 负责协调“单个调度实体（`se`）更新 → `cfs_rq` 聚合更新 → 向父层传播 → 附着/分离 → 通知调频 → 更新 `util_est`”这一整条链路
  * 整个流程的顺序严格遵循以上依赖链，确保每一层 PELT 统计都基于最新、最准确的数据
* 工作过程可以概况为：
1. 它先用统一的时间基准更新个体 se 的负载
2. 再更新所属 `cfs_rq` 的聚合值
3. 然后向父层传播组实体变化
4. 接着根据 `flags` 走 attach（迁移入）/ detach（迁移出）/ 普通更新三条互斥分支
5. 最后在需要时更新 `util_est`
```cpp
/* Update task and its cfs_rq load average */
static inline void update_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{       //获取当前 cfs_rq 的 PELT 时钟（已扣除 throttled 时间）
        u64 now = cfs_rq_clock_pelt(cfs_rq); //这个 now 会贯穿整个函数，作为所有 PELT 更新的时间基准
        int decayed;
        //se->avg.last_update_time：如果为 0，说明这个 se 是新建的或刚从其他 CPU 迁移过来（尚未在本地更新过），没有历史 PELT 状态需要老化
        //!(flags & SKIP_AGE_LOAD)：调用者明确要求跳过老化时，不做这一步
        /*
         * Track task load average for carrying it to new CPU after migrated, and
         * track group sched_entity load average for task_h_load calculation in migration
         */
        if (se->avg.last_update_time && !(flags & SKIP_AGE_LOAD))
                __update_load_avg_se(now, cfs_rq, se);
        //更新 cfs_rq 聚合负载，注意：这里更新的是 se 所属的 cfs_rq 本身，不是父层
        decayed  = update_cfs_rq_load_avg(now, cfs_rq);
        //向父层传播组实体负载，只对组实体（group entity）有效：如果 se 是普通任务，函数内部直接返回 0
        //propagate 在 update_cfs_rq_load_avg 之后，因为传播会修改 cfs_rq->avg，必须在 cfs_rq 自身更新完之后做，否则会被覆盖
        decayed |= propagate_entity_load_avg(se);

        if (!se->avg.last_update_time && (flags & DO_ATTACH)) {
                //语义：任务刚迁移到新 CPU，第一次入队。需要把 se 的 PELT 统计“挂到”新 cfs_rq 上
                //!se->avg.last_update_time：se 没有 PELT 历史（新建或刚从其他 CPU 迁移过来）
                //flags & DO_ATTACH：调用者来自 enqueue_entity()
                /*    
                 * DO_ATTACH means we're here from enqueue_entity().
                 * !last_update_time means we've passed through
                 * migrate_task_rq_fair() indicating we migrated.
                 *
                 * IOW we're enqueueing a task on a new CPU.
                 */
                attach_entity_load_avg(cfs_rq, se);
                update_tg_load_avg(cfs_rq);

        } else if (flags & DO_DETACH) {
                //语义：任务要从当前 CPU 迁移走，需要把 se 的 PELT 统计从当前 cfs_rq 上“摘下来”
                /*
                 * DO_DETACH means we're here from dequeue_entity()
                 * and we are migrating task out of the CPU.
                 * flags & DO_DETACH：调用者来自 dequeue_entity()，且任务正在迁出当前 CPU
                 */
                detach_entity_load_avg(cfs_rq, se);
                update_tg_load_avg(cfs_rq);
        } else if (decayed) { //既不是 attach 也不是 detach，但 decayed 为 true（发生了衰减或负载变化）
                cfs_rq_util_change(cfs_rq, 0); //如果这是根 cfs_rq，通知 CPUFreq 调速器“利用率变了”
                //传播到任务组
                if (flags & UPDATE_TG)
                        update_tg_load_avg(cfs_rq);
        }
        //放在最后，是因为 util_est 的更新依赖前面已经完成的所有 PELT 更新（尤其是 util_avg 和 runnable_avg 的最新值）
        if (flags & UPDATE_UTIL_EST) //如果调用者要求更新 util_est（通常是在出队路径）
                util_est_update(se);
}
```
* 为什么 attach/detach 分支不调用 `cfs_rq_util_change()`？
  * 因为 `attach_entity_load_avg()` 和 `detach_entity_load_avg()` 内部已经调用了 `cfs_rq_util_change()`。如果这里再调一次，会重复通知。
* 为什么普通分支只在 decayed 时才通知？
  * 如果没有发生衰减或负载变化，通知调速器没有意义，避免无谓开销

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
