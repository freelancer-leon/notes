# CPU 负载均衡
* `sg->sgc`：指向 `sched_group_capacity`，它是调度组对应的容量结构，用于描述该组内 CPU 的总算力
* `sgc->cpumask`：这个掩码包含了 **可用于向上负载均衡的 CPU**
```cpp
static inline struct cpumask *group_balance_mask(struct sched_group *sg) 
{
        return to_cpumask(sg->sgc->cpumask);
}
```
* 比如在 [之前的例子](sched_topo.md#例子可重叠调度组) 中，`cpu0` 在 `NUMA-2` 这一级，根据 `sched_verbose` 开启后的启动打印信息，有 3 个调度组 `sg {0,3,5},{2},{4}`
  * 第一个调度组包括 Node `{0, 3, 5}` 的 CPU，但只有 Node `0` 的 CPU 是均衡 CPU，因此有 `mask=0-15`
    * 打印函数 `sched_domain_debug_one(): !cpumask_equal(group_balance_mask(group), sched_group_span(group)))`
  * 第二个调度组包括 Node `2` 的 CPU，且它们都是均衡 CPU，因此没有 `mask=` 打印
  * 第三个调度组与二类似，`build_overlap_sched_groups()` 的时候就已经沉降到 Node `4` CPU 所在的 `NUMA-1` 那一级
    * 通过 `*per_cpu_ptr(sdd->sd, i)` 找到的 `sibling` 是范围为 Node `{1, 4-5}` 的调度域，它的子调度域 `sched_domain_span(sibling->child)` 为 Node `4`
    * 因此有 `build_balance_mask(): cpumask_equal(sg_span, sched_domain_span(sibling->child) -> cpumask_set_cpu(i, mask)`
```c
domain-3: span=0-15,32-95 level=NUMA [15] node=0,2-5
 groups: 0:{ span=0-15,48-63,80-95 mask=0-15 cap=48707 }, 32:{ span=32-47 cap=16383 }, 64:{ span=64-79 cap=16383 }  
                  0, 3, 5                                           2                            4
```
* 由此可见，`sg->cpumask` 与 `sg->sgc->cpumask` 是两个不同，后者是前者的子集
* 花括号前的数字是 `sg->sgc->id`，由于 `sg->sgc` 指向同一个 `sgc`（即负载均衡 CPU 的第一个 CPU 所属的 `sgc`，见 `init_overlap_sched_group()`），因此该 `id` 与 `group_balance_cpu()` 的返回值相等
```c
/*
 * Return the canonical balance CPU for this group, this is the first CPU
 * of this group that's also in the balance mask.
 *
 * The balance mask are all those CPUs that could actually end up at this
 * group. See build_balance_mask().
 *
 * Also see should_we_balance().
 */                 
int group_balance_cpu(struct sched_group *sg)
{                               
        return cpumask_first(group_balance_mask(sg));
}
```
```c
static int should_we_balance(struct lb_env *env)
{
        struct cpumask *swb_cpus = this_cpu_cpumask_var_ptr(should_we_balance_tmpmask);
        struct sched_group *sg = env->sd->groups;
        int cpu, idle_smt = -1;

        /*
         * Ensure the balancing environment is consistent; can happen
         * when the softirq triggers 'during' hotplug.
         */
        if (!cpumask_test_cpu(env->dst_cpu, env->cpus))
                return 0;

        /*
         * In the newly idle case, we will allow all the CPUs
         * to do the newly idle load balance.
         *
         * However, we bail out if we already have tasks or a wakeup pending,
         * to optimize wakeup latency.
         */
        if (env->idle == CPU_NEWLY_IDLE) {
                if (env->dst_rq->nr_running > 0 || env->dst_rq->ttwu_pending)
                        return 0;
                return 1;
        }

        cpumask_copy(swb_cpus, group_balance_mask(sg));
        /* Try to find first idle CPU */
        for_each_cpu_and(cpu, swb_cpus, env->cpus) {
                if (!idle_cpu(cpu))
                        continue;

                /*
                 * Don't balance to idle SMT in busy core right away when
                 * balancing cores, but remember the first idle SMT CPU for
                 * later consideration.  Find CPU on an idle core first.
                 */
                if (sched_smt_active() &&
                    !(env->sd->flags & SD_SHARE_CPUCAPACITY) &&
                    !is_core_idle(cpu)) {
                        if (idle_smt == -1)
                                idle_smt = cpu;
                        /*
                         * If the core is not idle, and first SMT sibling which is
                         * idle has been found, then its not needed to check other
                         * SMT siblings for idleness:
                         */
                        cpumask_andnot(swb_cpus, swb_cpus, cpu_smt_mask(cpu));
                        continue;
                }

                /*
                 * Are we the first idle core in a non-SMT domain or higher,
                 * or the first idle CPU in a SMT domain?
                 */
                return cpu == env->dst_cpu;
        }

        /* Are we the first idle CPU with busy siblings? */
        if (idle_smt != -1)
                return idle_smt == env->dst_cpu;

        /* Are we the first CPU of this group ? */
        return group_balance_cpu(sg) == env->dst_cpu;
}
```

```cpp
static inline unsigned long cpu_util_irq(struct rq *rq)
{
        return READ_ONCE(rq->avg_irq.util_avg);
}

static inline
unsigned long scale_irq_capacity(unsigned long util, unsigned long irq, unsigned long max)
{
        util *= (max - irq);
        util /= max;
        
        return util;

}

static inline unsigned long cpu_util_rt(struct rq *rq)
{
        return READ_ONCE(rq->avg_rt.util_avg);
}

static inline unsigned long cpu_util_dl(struct rq *rq)
{
        return READ_ONCE(rq->avg_dl.util_avg);
}

static unsigned long scale_rt_capacity(int cpu)
{
        unsigned long max = get_actual_cpu_capacity(cpu);
        struct rq *rq = cpu_rq(cpu);
        unsigned long used, free;
        unsigned long irq;

        irq = cpu_util_irq(rq);

        if (unlikely(irq >= max))
                return 1;

        /*
         * avg_rt.util_avg and avg_dl.util_avg track binary signals
         * (running and not running) with weights 0 and 1024 respectively.
         */
        used = cpu_util_rt(rq);
        used += cpu_util_dl(rq);

        if (unlikely(used >= max))
                return 1;

        free = max - used;

        return scale_irq_capacity(free, irq, max);
}

static void update_cpu_capacity(struct sched_domain *sd, int cpu)
{
        unsigned long capacity = scale_rt_capacity(cpu);
        struct sched_group *sdg = sd->groups;

        if (!capacity)
                capacity = 1;

        cpu_rq(cpu)->cpu_capacity = capacity;
        trace_sched_cpu_capacity_tp(cpu_rq(cpu));

        sdg->sgc->capacity = capacity;
        sdg->sgc->min_capacity = capacity;
        sdg->sgc->max_capacity = capacity;
}

void update_group_capacity(struct sched_domain *sd, int cpu)
{
        struct sched_domain *child = sd->child;
        struct sched_group *group, *sdg = sd->groups;
        unsigned long capacity, min_capacity, max_capacity;
        unsigned long interval;

        interval = msecs_to_jiffies(sd->balance_interval);
        interval = clamp(interval, 1UL, max_load_balance_interval);
        sdg->sgc->next_update = jiffies + interval;

        if (!child) {
                update_cpu_capacity(sd, cpu);
                return;
        }

        capacity = 0;
        min_capacity = ULONG_MAX;
        max_capacity = 0;

        if (child->flags & SD_NUMA) {
                /*
                 * SD_NUMA domains cannot assume that child groups
                 * span the current group.
                 */

                for_each_cpu(cpu, sched_group_span(sdg)) {
                        unsigned long cpu_cap = capacity_of(cpu);

                        capacity += cpu_cap;
                        min_capacity = min(cpu_cap, min_capacity);
                        max_capacity = max(cpu_cap, max_capacity);
                }
        } else  {
                /*
                 * !SD_NUMA domains can assume that child groups
                 * span the current group.
                 */

                group = child->groups;
                do {
                        struct sched_group_capacity *sgc = group->sgc;

                        capacity += sgc->capacity;
                        min_capacity = min(sgc->min_capacity, min_capacity);
                        max_capacity = max(sgc->max_capacity, max_capacity);
                        group = group->next;
                } while (group != child->groups);
        }

        sdg->sgc->capacity = capacity;
        sdg->sgc->min_capacity = min_capacity;
        sdg->sgc->max_capacity = max_capacity;
}
```

```cpp
/*
 * Remove and clamp on negative, from a local variable.
 *
 * A variant of sub_positive(), which does not use explicit load-store
 * and is thus optimized for local variable updates.
 */
#define lsub_positive(_ptr, _val) do {                          \
        typeof(_ptr) ptr = (_ptr);                              \
        *ptr -= min_t(typeof(*ptr), *ptr, _val);                \
} while (0)

static inline long sibling_imbalance(struct lb_env *env,
                                    struct sd_lb_stats *sds,
                                    struct sg_lb_stats *busiest,
                                    struct sg_lb_stats *local)
{
        int ncores_busiest, ncores_local;
        long imbalance;

        if (!env->idle || !busiest->sum_nr_running)
                return 0;

        ncores_busiest = sds->busiest->cores;
        ncores_local = sds->local->cores;

        if (ncores_busiest == ncores_local) {
                imbalance = busiest->sum_nr_running;
                lsub_positive(&imbalance, local->sum_nr_running);
                return imbalance;
        }

        /* Balance such that nr_running/ncores ratio are same on both groups */
        imbalance = ncores_local * busiest->sum_nr_running;
        lsub_positive(&imbalance, ncores_busiest * local->sum_nr_running);
        /* Normalize imbalance and do rounding on normalization */
        imbalance = 2 * imbalance + ncores_local + ncores_busiest;
        imbalance /= ncores_local + ncores_busiest;

        /* Take advantage of resource in an empty sched group */
        if (imbalance <= 1 && local->sum_nr_running == 0 &&
            busiest->sum_nr_running > 1)
                imbalance = 2;

        return imbalance;
}

static inline bool
sched_reduced_capacity(struct rq *rq, struct sched_domain *sd)
{
        /*
         * When there is more than 1 task, the group_overloaded case already
         * takes care of cpu with reduced capacity
         */
        if (rq->cfs.h_nr_runnable != 1)
                return false;

        return check_cpu_capacity(rq, sd);
}
```

```cpp
static unsigned long cpu_load(struct rq *rq)
{                                     
        return cfs_rq_load_avg(&rq->cfs);
}

/**
 * cpu_util() - Estimates the amount of CPU capacity used by CFS tasks.
 * @cpu: the CPU to get the utilization for
 * @p: task for which the CPU utilization should be predicted or NULL
 * @dst_cpu: CPU @p migrates to, -1 if @p moves from @cpu or @p == NULL
 * @boost: 1 to enable boosting, otherwise 0
 *
 * The unit of the return value must be the same as the one of CPU capacity
 * so that CPU utilization can be compared with CPU capacity.
 *
 * CPU utilization is the sum of running time of runnable tasks plus the
 * recent utilization of currently non-runnable tasks on that CPU.
 * It represents the amount of CPU capacity currently used by CFS tasks in
 * the range [0..max CPU capacity] with max CPU capacity being the CPU
 * capacity at f_max.
 *
 * The estimated CPU utilization is defined as the maximum between CPU
 * utilization and sum of the estimated utilization of the currently
 * runnable tasks on that CPU. It preserves a utilization "snapshot" of
 * previously-executed tasks, which helps better deduce how busy a CPU will
 * be when a long-sleeping task wakes up. The contribution to CPU utilization
 * of such a task would be significantly decayed at this point of time.
 *
 * Boosted CPU utilization is defined as max(CPU runnable, CPU utilization).
 * CPU contention for CFS tasks can be detected by CPU runnable > CPU
 * utilization. Boosting is implemented in cpu_util() so that internal
 * users (e.g. EAS) can use it next to external users (e.g. schedutil),
 * latter via cpu_util_cfs_boost().
 *
 * CPU utilization can be higher than the current CPU capacity
 * (f_curr/f_max * max CPU capacity) or even the max CPU capacity because
 * of rounding errors as well as task migrations or wakeups of new tasks.
 * CPU utilization has to be capped to fit into the [0..max CPU capacity]
 * range. Otherwise a group of CPUs (CPU0 util = 121% + CPU1 util = 80%)
 * could be seen as over-utilized even though CPU1 has 20% of spare CPU
 * capacity. CPU utilization is allowed to overshoot current CPU capacity
 * though since this is useful for predicting the CPU capacity required
 * after task migrations (scheduler-driven DVFS).
 *
 * Return: (Boosted) (estimated) utilization for the specified CPU.
 */
static unsigned long
cpu_util(int cpu, struct task_struct *p, int dst_cpu, int boost)
{
        bool add_task = p && task_cpu(p) != cpu && dst_cpu == cpu;
        bool sub_task = p && task_cpu(p) == cpu && dst_cpu != cpu;
        struct cfs_rq *cfs_rq = &cpu_rq(cpu)->cfs;
        unsigned long util = READ_ONCE(cfs_rq->avg.util_avg);
        unsigned long runnable;

        /*
         * If @dst_cpu is -1 or @p migrates from @cpu to @dst_cpu remove its
         * contribution. If @p migrates from another CPU to @cpu add its
         * contribution. In all the other cases @cpu is not impacted by the
         * migration so its util_avg is already correct.
         */
        if (add_task)
                util += task_util(p);
        else if (sub_task)
                lsub_positive(&util, task_util(p));

        if (boost) {
                runnable = READ_ONCE(cfs_rq->avg.runnable_avg);
                if (add_task)
                        runnable += READ_ONCE(p->se.avg.runnable_avg);
                else if (sub_task)
                        lsub_positive(&runnable,
                                      READ_ONCE(p->se.avg.runnable_avg));
                util = max(util, runnable);
        }

        if (sched_feat(UTIL_EST)) {
                unsigned long util_est;

                util_est = READ_ONCE(cfs_rq->avg.util_est);

                /*
                 * During wake-up @p isn't enqueued yet and doesn't contribute
                 * to any cpu_rq(cpu)->cfs.avg.util_est.
                 * If @dst_cpu == @cpu add it to "simulate" cpu_util after @p
                 * has been enqueued.
                 *
                 * During exec (@dst_cpu = -1) @p is enqueued and does
                 * contribute to cpu_rq(cpu)->cfs.util_est.
                 * Remove it to "simulate" cpu_util without @p's contribution.
                 *
                 * Despite the task_on_rq_queued(@p) check there is still a
                 * small window for a possible race when an exec
                 * select_task_rq_fair() races with LB's detach_task().
                 *
                 *   detach_task()
                 *     deactivate_task()
                 *       p->on_rq = TASK_ON_RQ_MIGRATING;
                 *       -------------------------------- A
                 *       dequeue_task()                    \
                 *         dequeue_task_fair()              + Race Time
                 *           util_est_dequeue()            /
                 *       -------------------------------- B
                 *
                 * The additional check "current == p" is required to further
                 * reduce the race window.
                 */
                if (dst_cpu == cpu)
                        util_est += _task_util_est(p);
                else if (p && unlikely(task_on_rq_queued(p) || current == p))
                        lsub_positive(&util_est, _task_util_est(p));

                util = max(util, util_est);
        }

        return min(util, arch_scale_cpu_capacity(cpu));
}

unsigned long cpu_util_cfs(int cpu)
{
        return cpu_util(cpu, NULL, -1, 0);
}

unsigned long cpu_util_cfs_boost(int cpu)
{
        return cpu_util(cpu, NULL, -1, 1);
}
```

```cpp
/*
 * The margin used when comparing CPU capacities.
 * is 'cap1' noticeably greater than 'cap2'
 *
 * (default: ~5%)
 */
#define capacity_greater(cap1, cap2) ((cap1) * 1024 > (cap2) * 1078)

static unsigned long cpu_runnable(struct rq *rq)
{     
        return cfs_rq_runnable_avg(&rq->cfs);
}
```

```cpp
static inline unsigned long uclamp_rq_get(struct rq *rq,
                                          enum uclamp_id clamp_id)
{       
        return READ_ONCE(rq->uclamp[clamp_id].value);
}

static inline bool cpu_overutilized(int cpu)
{
        unsigned long rq_util_max;
 
        if (!sched_energy_enabled())
                return false;

        rq_util_max = uclamp_rq_get(cpu_rq(cpu), UCLAMP_MAX);

        /* Return true only if the utilization doesn't fit CPU's capacity */
        return !util_fits_cpu(cpu_util_cfs(cpu), 0, rq_util_max, cpu);
}
```

```cpp
static inline bool sched_asym_prefer(int a, int b)
{
        return arch_asym_cpu_priority(a) > arch_asym_cpu_priority(b);
}

/**
 * sched_use_asym_prio - Check whether asym_packing priority must be used
 * @sd:         The scheduling domain of the load balancing
 * @cpu:        A CPU
 *
 * Always use CPU priority when balancing load between SMT siblings. When
 * balancing load between cores, it is not sufficient that @cpu is idle. Only
 * use CPU priority if the whole core is idle.
 *
 * Returns: True if the priority of @cpu must be followed. False otherwise.
 */
static bool sched_use_asym_prio(struct sched_domain *sd, int cpu)
{
        if (!(sd->flags & SD_ASYM_PACKING)) 
                return false;

        if (!sched_smt_active())
                return true;

        return sd->flags & SD_SHARE_CPUCAPACITY || is_core_idle(cpu);
}
        
static inline bool sched_asym(struct sched_domain *sd, int dst_cpu, int src_cpu)
{
        /*
         * First check if @dst_cpu can do asym_packing load balance. Only do it
         * if it has higher priority than @src_cpu.
         */
        return sched_use_asym_prio(sd, dst_cpu) &&
                sched_asym_prefer(dst_cpu, src_cpu);
}

/**
 * sched_group_asym - Check if the destination CPU can do asym_packing balance
 * @env:        The load balancing environment
 * @sgs:        Load-balancing statistics of the candidate busiest group
 * @group:      The candidate busiest group
 *
 * @env::dst_cpu can do asym_packing if it has higher priority than the
 * preferred CPU of @group.
 *      
 * Return: true if @env::dst_cpu can do with asym_packing load balance. False
 * otherwise.
 */             
static inline bool 
sched_group_asym(struct lb_env *env, struct sg_lb_stats *sgs, struct sched_group *group)
{                       
        /*
         * CPU priorities do not make sense for SMT cores with more than one
         * busy sibling.
         */             
        if ((group->flags & SD_SHARE_CPUCAPACITY) &&
            (sgs->group_weight - sgs->idle_cpus != 1))
                return false;
                        
        return sched_asym(env->sd, env->dst_cpu, READ_ONCE(group->asym_prefer_cpu));
}
```

```cpp
static inline bool smt_balance(struct lb_env *env, struct sg_lb_stats *sgs,
                               struct sched_group *group)
{
        if (!env->idle)
                return false; 

        /*
         * For SMT source group, it is better to move a task
         * to a CPU that doesn't have multiple tasks sharing its CPU capacity.
         * Note that if a group has a single SMT, SD_SHARE_CPUCAPACITY
         * will not be on.
         */
        if (group->flags & SD_SHARE_CPUCAPACITY &&
            sgs->sum_h_nr_running > 1)
                return true;

        return false;
}
```

```cpp
/*
 * The margin used when comparing LLC utilization with CPU capacity.
 * It determines the LLC load level where active LLC aggregation is
 * done.                            
 * Derived from fits_capacity().
 *
 * (default: ~50%, tunable via debugfs)
 */
static bool fits_llc_capacity(unsigned long util, unsigned long max)
{               
        u32 aggr_pct = llc_overaggr_pct;

        /*
         * For single core systems, raise the aggregation
         * threshold to accommodate more tasks.
         */ 
        if (cpu_smt_num_threads == 1)
                aggr_pct = (aggr_pct * 3 / 2);

        return util * 100 < max * aggr_pct;
}

/*
 * The margin used when comparing utilization.
 * is 'util1' noticeably greater than 'util2'
 * Derived from capacity_greater().
 * Bias is in perentage.
 */                     
/* Allows dst util to be bigger than src util by up to bias percent */
#define util_greater(util1, util2) \
        ((util1) * 100 > (util2) * (100 + llc_imb_pct))

static __maybe_unused bool get_llc_stats(int cpu, unsigned long *util,
                                         unsigned long *cap)
{
        struct sched_domain_shared *sd_share;

        sd_share = rcu_dereference_all(per_cpu(sd_llc_shared, cpu));
        if (!sd_share)
                return false;

        *util = READ_ONCE(sd_share->util_avg);
        *cap = READ_ONCE(sd_share->capacity);

        return true;
}

/*
 * Decision matrix according to the LLC utilization. To
 * decide whether we can do task aggregation across LLC.
 *
 * By default, 50% is the threshold for treating the LLC
 * as busy. The reason for choosing 50% is to avoid saturation
 * of SMT-2, and it is also a safe cutoff for other SMT-n
 * platforms. SMT-1 has higher threshold because it is
 * supposed to accommodate more tasks, see fits_llc_capacity().
 *
 * 20% is the utilization imbalance percentage to decide
 * if the preferred LLC is busier than the non-preferred LLC.
 * 20 is a little higher than the LLC domain's imbalance_pct
 * 17. The hysteresis is used to avoid task bouncing between the
 * preferred LLC and the non-preferred LLC, and it will
 * be turned into tunable debugfs.
 *
 * 1. moving towards the preferred LLC, dst is the preferred
 *    LLC, src is not.
 *
 * src \ dst      30%  40%  50%  60%
 * 30%            Y    Y    Y    N
 * 40%            Y    Y    Y    Y
 * 50%            Y    Y    G    G
 * 60%            Y    Y    G    G
 *
 * 2. moving out of the preferred LLC, src is the preferred
 *    LLC, dst is not:
 *
 * src \ dst      30%  40%  50%  60%
 * 30%            N    N    N    N
 * 40%            N    N    N    N
 * 50%            N    N    G    G
 * 60%            Y    N    G    G
 *
 * src :      src_util
 * dst :      dst_util
 * Y :        Yes, migrate
 * N :        No, do not migrate
 * G :        let the Generic load balance to even the load.
 *
 * The intention is that if both LLCs are quite busy, cache aware
 * load balance should not be performed, and generic load balance
 * should take effect. However, if one is busy and the other is not,
 * the preferred LLC capacity(50%) and imbalance criteria(20%) should
 * be considered to determine whether LLC aggregation should be
 * performed to bias the load towards the preferred LLC.
 */

/* migration decision, 3 states are orthogonal. */
enum llc_mig {
        mig_forbid = 0,         /* N: Don't migrate task, respect LLC preference */
        mig_llc,                /* Y: Do LLC preference based migration */
        mig_unrestricted        /* G: Don't restrict generic load balance migration */
};

/*
 * Check if task can be moved from the source LLC to the
 * destination LLC without breaking cache aware preferrence.
 * src_cpu and dst_cpu are arbitrary CPUs within the source
 * and destination LLCs, respectively.
 */
static enum llc_mig can_migrate_llc(int src_cpu, int dst_cpu,
                                    unsigned long tsk_util,
                                    bool to_pref)
{
        unsigned long src_util, dst_util, src_cap, dst_cap;

        if (!get_llc_stats(src_cpu, &src_util, &src_cap) ||
            !get_llc_stats(dst_cpu, &dst_util, &dst_cap))
                return mig_unrestricted;

        src_util = src_util < tsk_util ? 0 : src_util - tsk_util;
        dst_util = dst_util + tsk_util;

        if (!fits_llc_capacity(dst_util, dst_cap) &&
            !fits_llc_capacity(src_util, src_cap))
                return mig_unrestricted;

        if (to_pref) {
                /*
                 * Don't migrate if we will get preferred LLC too
                 * heavily loaded and if the dest is much busier
                 * than the src, in which case migration will
                 * increase the imbalance too much.
                 */
                if (!fits_llc_capacity(dst_util, dst_cap) &&
                    util_greater(dst_util, src_util))
                        return mig_forbid;
        } else {
                /*
                 * Don't migrate if we will leave preferred LLC
                 * too idle, or if this migration leads to the
                 * non-preferred LLC falls within sysctl_aggr_imb percent
                 * of preferred LLC, leading to migration again
                 * back to preferred LLC.
                 */
                if (fits_llc_capacity(src_util, src_cap) ||
                    !util_greater(src_util, dst_util))
                        return mig_forbid;
        }
        return mig_llc;
}
```

```cpp
/*
 * Do LLC balance on sched group that contains LLC, and have tasks preferring
 * to run on LLC in idle dst_cpu.
 */
static inline bool llc_balance(struct lb_env *env, struct sg_lb_stats *sgs,
                               struct sched_group *group)
{
        if (!sched_cache_enabled())
                return false;

        if (env->sd->flags & SD_SHARE_LLC)
                return false;

        /*
         * On asymmetric domains, group_misfit_task_load
         * should be prioritized to move tasks to CPU that fit them
         * over aggregating tasks to their preferred LLC.
         */
        if ((env->sd->flags & SD_ASYM_CPUCAPACITY) &&
            sgs->group_misfit_task_load)
                return false;

        /*
         * Skip cache aware tagging if nr_balanced_failed is sufficiently high.
         * Threshold of cache_nice_tries is set to 1 higher than nr_balance_failed
         * to avoid excessive task migration at the same time.
         */
        if (env->sd->nr_balance_failed >= env->sd->cache_nice_tries + 1)
                return false;

        if (sgs->nr_pref_dst_llc &&
            can_migrate_llc(cpumask_first(sched_group_span(group)),
                            env->dst_cpu, 0, true) == mig_llc)
                return true;

        return false;
}
```

```cpp
/*
 * Group imbalance indicates (and tries to solve) the problem where balancing
 * groups is inadequate due to ->cpus_ptr constraints.
 *
 * Imagine a situation of two groups of 4 CPUs each and 4 tasks each with a
 * cpumask covering 1 CPU of the first group and 3 CPUs of the second group.
 * Something like:
 *
 *      { 0 1 2 3 } { 4 5 6 7 }
 *              *     * * *
 *
 * If we were to balance group-wise we'd place two tasks in the first group and
 * two tasks in the second group. Clearly this is undesired as it will overload
 * cpu 3 and leave one of the CPUs in the second group unused.
 *
 * The current solution to this issue is detecting the skew in the first group
 * by noticing the lower domain failed to reach balance and had difficulty
 * moving tasks due to affinity constraints.
 *
 * When this is so detected; this group becomes a candidate for busiest; see
 * update_sd_pick_busiest(). And calculate_imbalance() and
 * sched_balance_find_src_group() avoid some of the usual balance conditions to allow it
 * to create an effective group imbalance.
 *
 * This is a somewhat tricky proposition since the next run might not find the
 * group imbalance and decide the groups need to be balanced again. A most
 * subtle and fragile situation. 
 */       

static inline int sg_imbalanced(struct sched_group *group)
{
        return group->sgc->imbalance;
}

/*
 * group_has_capacity returns true if the group has spare capacity that could
 * be used by some tasks.
 * We consider that a group has spare capacity if the number of task is
 * smaller than the number of CPUs or if the utilization is lower than the
 * available capacity for CFS tasks.
 * For the latter, we use a threshold to stabilize the state, to take into
 * account the variance of the tasks' load and to return true if the available
 * capacity in meaningful for the load balancer.
 * As an example, an available capacity of 1% can appear but it doesn't make
 * any benefit for the load balance.
 */                       
static inline bool
group_has_capacity(unsigned int imbalance_pct, struct sg_lb_stats *sgs)
{               
        if (sgs->sum_nr_running < sgs->group_weight)
                return true;
                
        if ((sgs->group_capacity * imbalance_pct) <
                        (sgs->group_runnable * 100))
                return false;

        if ((sgs->group_capacity * 100) >
                        (sgs->group_util * imbalance_pct))
                return true;

        return false;
}

/*
 *  group_is_overloaded returns true if the group has more tasks than it can
 *  handle.
 *  group_is_overloaded is not equals to !group_has_capacity because a group
 *  with the exact right number of tasks, has no more spare capacity but is not
 *  overloaded so both group_has_capacity and group_is_overloaded return
 *  false.      
 */
static inline bool
group_is_overloaded(unsigned int imbalance_pct, struct sg_lb_stats *sgs)
{
        /* 
         * With EAS and uclamp, 1 CPU in the group must be overutilized to
         * consider the group overloaded.
         */
        if (sched_energy_enabled() && !sgs->group_overutilized)
                return false;
        
        if (sgs->sum_nr_running <= sgs->group_weight)
                return false;

        if ((sgs->group_capacity * 100) <
                        (sgs->group_util * imbalance_pct))
                return true;

        if ((sgs->group_capacity * imbalance_pct) <
                        (sgs->group_runnable * 100))
                return true;

        return false;
}


static inline enum
group_type group_classify(unsigned int imbalance_pct,
                          struct sched_group *group,
                          struct sg_lb_stats *sgs)
{
        if (group_is_overloaded(imbalance_pct, sgs))
                return group_overloaded;

        if (sgs->group_llc_balance)
                return group_llc_balance;

        if (sg_imbalanced(group))
                return group_imbalanced;

        if (sgs->group_asym_packing)
                return group_asym_packing;

        if (sgs->group_smt_balance)
                return group_smt_balance;

        if (sgs->group_misfit_task_load)
                return group_misfit_task;

        if (!group_has_capacity(imbalance_pct, sgs))
                return group_fully_busy;

        return group_has_spare;
}
```

```cpp
/*
 * Record the statistics for this scheduler group for later
 * use. These values guide load balancing on aggregating tasks
 * to a LLC.
 */
static void record_sg_llc_stats(struct lb_env *env,
                                struct sg_lb_stats *sgs,
                                struct sched_group *group)
{
        struct sched_domain_shared *sd_share;
        int cpu;        

        if (!sched_cache_enabled() || env->idle == CPU_NEWLY_IDLE)
                return; 

        /* Only care about sched domain spanning multiple LLCs */
        if (env->sd->child != rcu_dereference_all(per_cpu(sd_llc, env->dst_cpu)))
                return;
                        
        /*
         * At this point we know this group spans a LLC domain.
         * Record the statistic of this group in its corresponding
         * shared LLC domain.
         * Note: sd_share cannot be obtained via sd->child->shared,
         * because the latter refers to the domain that covers the
         * local group. Instead, sd_share should be located using
         * the first CPU of the LLC group.
         */                     
        cpu = cpumask_first(sched_group_span(group));
        sd_share = rcu_dereference_all(per_cpu(sd_llc_shared, cpu));
        if (!sd_share)
                return;

        if (READ_ONCE(sd_share->util_avg) != sgs->group_util)
                WRITE_ONCE(sd_share->util_avg, sgs->group_util);

        if (unlikely(READ_ONCE(sd_share->capacity) != sgs->group_capacity))
                WRITE_ONCE(sd_share->capacity, sgs->group_capacity);
}
```

```cpp
/**
 * update_sg_lb_stats - Update sched_group's statistics for load balancing.
 * @env: The load balancing environment.
 * @sds: Load-balancing data with statistics of the local group.
 * @group: sched_group whose statistics are to be updated.
 * @sgs: variable to hold the statistics for this group.
 * @sg_overloaded: sched_group is overloaded
 */
static inline void update_sg_lb_stats(struct lb_env *env,
                                      struct sd_lb_stats *sds,
                                      struct sched_group *group,
                                      struct sg_lb_stats *sgs,
                                      bool *sg_overloaded)
{
        int i, nr_running, local_group, sd_flags = env->sd->flags;
        bool balancing_at_rd = !env->sd->parent;

        memset(sgs, 0, sizeof(*sgs));

        local_group = group == sds->local;

        for_each_cpu_and(i, sched_group_span(group), env->cpus) {
                struct rq *rq = cpu_rq(i);
                unsigned long load = cpu_load(rq);

                sgs->group_load += load;
                sgs->group_util += cpu_util_cfs(i);
                sgs->group_runnable += cpu_runnable(rq);
                sgs->sum_h_nr_running += rq->cfs.h_nr_runnable;

                nr_running = rq->nr_running;
                sgs->sum_nr_running += nr_running;

                if (cpu_overutilized(i))
                        sgs->group_overutilized = 1;

#ifdef CONFIG_SCHED_CACHE
                if (sched_cache_enabled()) {
                        struct sched_domain *sd_tmp;
                        int dst_llc;

                        dst_llc = llc_id(env->dst_cpu);
                        if (llc_id(i) != dst_llc) {
                                sd_tmp = rcu_dereference_all(rq->sd);
                                if (sd_tmp && (unsigned int)dst_llc < sd_tmp->llc_max)
                                        sgs->nr_pref_dst_llc += sd_tmp->llc_counts[dst_llc];
                        }
                }
#endif

                /*
                 * No need to call idle_cpu() if nr_running is not 0
                 */
                if (!nr_running && idle_cpu(i)) {
                        sgs->idle_cpus++;
                        /* Idle cpu can't have misfit task */
                        continue;
                }

                /* Overload indicator is only updated at root domain */
                if (balancing_at_rd && nr_running > 1)
                        *sg_overloaded = 1;

#ifdef CONFIG_NUMA_BALANCING
                /* Only fbq_classify_group() uses this to classify NUMA groups */
                if (sd_flags & SD_NUMA) {
                        sgs->nr_numa_running += rq->nr_numa_running;
                        sgs->nr_preferred_running += rq->nr_preferred_running;
                }
#endif
                if (local_group)
                        continue;

                if (sd_flags & SD_ASYM_CPUCAPACITY) {
                        if (rq->misfit_task_load) {
                                /*
                                 * Always mark the root domain overloaded so big
                                 * CPUs can pick up misfit tasks via newly idle
                                 * balance.
                                 */
                                if (balancing_at_rd)
                                        *sg_overloaded = 1;

                                /*
                                 * Only account misfit load if @dst_cpu can
                                 * help; otherwise, the group may be classified
                                 * as misfit_task and update_sd_pick_busiest()
                                 * will skip it.
                                 */
                                if (capacity_greater(capacity_of(env->dst_cpu),
                                                     group->sgc->max_capacity) &&
                                    (sgs->group_misfit_task_load < rq->misfit_task_load))
                                        sgs->group_misfit_task_load = rq->misfit_task_load;
                        }
                } else if (env->idle && sched_reduced_capacity(rq, env->sd)) {
                        /* Check for a task running on a CPU with reduced capacity */
                        if (sgs->group_misfit_task_load < load)
                                sgs->group_misfit_task_load = load;
                }
        }

        sgs->group_capacity = group->sgc->capacity;

        sgs->group_weight = group->group_weight;

        if (!local_group) {
                /* Check if dst CPU is idle and preferred to this group */
                if (env->idle && sgs->sum_h_nr_running &&
                    sched_group_asym(env, sgs, group))
                        sgs->group_asym_packing = 1;

                /* Check for loaded SMT group to be balanced to dst CPU */
                if (smt_balance(env, sgs, group))
                        sgs->group_smt_balance = 1;

                /* Check for tasks in this group can be moved to their preferred LLC */
                if (llc_balance(env, sgs, group))
                        sgs->group_llc_balance = 1;
        }

        sgs->group_type = group_classify(env->sd->imbalance_pct, group, sgs);

        record_sg_llc_stats(env, sgs, group);
        /* Computing avg_load makes sense only when group is overloaded */
        if (sgs->group_type == group_overloaded)
                sgs->avg_load = (sgs->group_load * SCHED_CAPACITY_SCALE) /
                                sgs->group_capacity;
}
```

```cpp
/* One group has more than one SMT CPU while the other group does not */
static inline bool smt_vs_nonsmt_groups(struct sched_group *sg1,
                                    struct sched_group *sg2)
{
        if (!sg1 || !sg2)
                return false;

        return (sg1->flags & SD_SHARE_CPUCAPACITY) !=
                (sg2->flags & SD_SHARE_CPUCAPACITY);
}

static bool update_llc_busiest(struct lb_env *env,
                               struct sg_lb_stats *busiest,
                               struct sg_lb_stats *sgs)
{               
        /*
         * There are more tasks that want to run on dst_cpu's LLC.
         */
        return sgs->nr_pref_dst_llc > busiest->nr_pref_dst_llc;
}

/**
 * update_sd_pick_busiest - return 1 on busiest group
 * @env: The load balancing environment.
 * @sds: sched_domain statistics
 * @sg: sched_group candidate to be checked for being the busiest
 * @sgs: sched_group statistics
 *
 * Determine if @sg is a busier group than the previously selected
 * busiest group.
 *         
 * Return: %true if @sg is a busier group than the previously selected
 * busiest group. %false otherwise.
 */
static bool update_sd_pick_busiest(struct lb_env *env,
                                   struct sd_lb_stats *sds,
                                   struct sched_group *sg,
                                   struct sg_lb_stats *sgs)
{
        struct sg_lb_stats *busiest = &sds->busiest_stat;

        /* Make sure that there is at least one task to pull */
        if (!sgs->sum_h_nr_running)
                return false;

        /*
         * Don't try to pull misfit tasks we can't help.
         * We can use max_capacity here as reduction in capacity on some
         * CPUs in the group should either be possible to resolve
         * internally or be covered by avg_load imbalance (eventually).
         *
         * When SMT is active, only pull a misfit to dst_cpu if it is on a
         * fully idle core; otherwise the effective capacity of the core is
         * reduced and we may not actually provide more capacity than the
         * source.
         */
        if ((env->sd->flags & SD_ASYM_CPUCAPACITY) &&
            (sgs->group_type == group_misfit_task) &&
            (!env->dst_core_idle ||
             !capacity_greater(capacity_of(env->dst_cpu), sg->sgc->max_capacity) ||
             sds->local_stat.group_type != group_has_spare))
                return false;

        /*
         * Candidate sg has no more than one task per CPU and has higher
         * per-CPU capacity. Migrating tasks to less capable CPUs may harm
         * throughput. Maximize throughput, power/energy consequences are not
         * considered.
         */
        if ((env->sd->flags & SD_ASYM_CPUCAPACITY) &&
            (sgs->group_type <= group_fully_busy) &&
            (capacity_greater(sg->sgc->min_capacity, capacity_of(env->dst_cpu))))
                return false;

        if (sgs->group_type > busiest->group_type)
                return true;

        if (sgs->group_type < busiest->group_type)
                return false;

        /*
         * The candidate and the current busiest group are the same type of
         * group. Let check which one is the busiest according to the type.
         */

        switch (sgs->group_type) {
        case group_overloaded:
                /* Select the overloaded group with highest avg_load. */
                return sgs->avg_load > busiest->avg_load;

        case group_llc_balance:
                /* Select the group with most tasks preferring dst LLC */
                return update_llc_busiest(env, busiest, sgs);

        case group_imbalanced:
                /*
                 * Select the 1st imbalanced group as we don't have any way to
                 * choose one more than another.
                 */
                return false;

        case group_asym_packing:
                /* Prefer to move from lowest priority CPU's work */
                return sched_asym_prefer(READ_ONCE(sds->busiest->asym_prefer_cpu),
                                         READ_ONCE(sg->asym_prefer_cpu));

        case group_misfit_task:
                /*
                 * If we have more than one misfit sg go with the biggest
                 * misfit.
                 */
                return sgs->group_misfit_task_load > busiest->group_misfit_task_load;

        case group_smt_balance:
                /*
                 * Check if we have spare CPUs on either SMT group to
                 * choose has spare or fully busy handling.
                 */
                if (sgs->idle_cpus != 0 || busiest->idle_cpus != 0)
                        goto has_spare;

                fallthrough;

        case group_fully_busy:
                /*
                 * Select the fully busy group with highest avg_load. In
                 * theory, there is no need to pull task from such kind of
                 * group because tasks have all compute capacity that they need
                 * but we can still improve the overall throughput by reducing
                 * contention when accessing shared HW resources.
                 *
                 * XXX for now avg_load is not computed and always 0 so we
                 * select the 1st one, except if @sg is composed of SMT
                 * siblings.
                 */

                if (sgs->avg_load < busiest->avg_load)
                        return false;

                if (sgs->avg_load == busiest->avg_load) {
                        /*
                         * SMT sched groups need more help than non-SMT groups.
                         * If @sg happens to also be SMT, either choice is good.
                         */
                        if (sds->busiest->flags & SD_SHARE_CPUCAPACITY)
                                return false;
                }

                break;

        case group_has_spare:
                /*
                 * Do not pick sg with SMT CPUs over sg with pure CPUs,
                 * as we do not want to pull task off SMT core with one task
                 * and make the core idle.
                 */
                if (smt_vs_nonsmt_groups(sds->busiest, sg)) {
                        if (sg->flags & SD_SHARE_CPUCAPACITY && sgs->sum_h_nr_running <= 1)
                                return false;
                        else
                                return true;
                }
has_spare:

                /*
                 * Select not overloaded group with lowest number of idle CPUs
                 * and highest number of running tasks. We could also compare
                 * the spare capacity which is more stable but it can end up
                 * that the group has less spare capacity but finally more idle
                 * CPUs which means less opportunity to pull tasks.
                 */
                if (sgs->idle_cpus > busiest->idle_cpus)
                        return false;
                else if ((sgs->idle_cpus == busiest->idle_cpus) &&
                         (sgs->sum_nr_running <= busiest->sum_nr_running))
                        return false;

                break;
        }

        return true;
}
```

```cpp
static inline enum fbq_type fbq_classify_group(struct sg_lb_stats *sgs)
{       
        if (sgs->sum_h_nr_running > sgs->nr_numa_running)
                return regular;
        if (sgs->sum_h_nr_running > sgs->nr_preferred_running)
                return remote;
        return all;
}

static inline enum fbq_type fbq_classify_rq(struct rq *rq)
{       
        if (rq->nr_running > rq->nr_numa_running)
                return regular;
        if (rq->nr_running > rq->nr_preferred_running)
                return remote;
        return all;
}
```

```cpp
static inline void set_rd_overloaded(struct root_domain *rd, int status)
{
        if (get_rd_overloaded(rd) != status)
                WRITE_ONCE(rd->overloaded, status);
}

/*
 * overutilized value make sense only if EAS is enabled
 */
static inline bool is_rd_overutilized(struct root_domain *rd)
{       
        return !sched_energy_enabled() || READ_ONCE(rd->overutilized);
}

static inline void set_rd_overutilized(struct root_domain *rd, bool flag)
{       
        if (!sched_energy_enabled())
                return;

        WRITE_ONCE(rd->overutilized, flag);
        trace_sched_overutilized_tp(rd, flag);
}

static void update_idle_cpu_scan(struct lb_env *env,
                                 unsigned long sum_util)
{
        struct sched_domain_shared *sd_share;
        struct sched_domain *sd = env->sd;
        int llc_weight, pct;
        u64 x, y, tmp;
        /*
         * Update the number of CPUs to scan in LLC domain, which could
         * be used as a hint in select_idle_cpu(). The update of sd_share
         * could be expensive because it is within a shared cache line.
         * So the write of this hint only occurs during periodic load
         * balancing, rather than CPU_NEWLY_IDLE, because the latter
         * can fire way more frequently than the former.
         */
        if (!sched_feat(SIS_UTIL) || env->idle == CPU_NEWLY_IDLE)
                return;
                
        sd_share = sd->shared;
        if (!sd_share)
                return;
         
        /*
         * The number of CPUs to search drops as sum_util increases, when
         * sum_util hits 85% or above, the scan stops.
         * The reason to choose 85% as the threshold is because this is the
         * imbalance_pct(117) when a LLC sched group is overloaded.
         *
         * let y = SCHED_CAPACITY_SCALE - p * x^2                       [1]
         * and y'= y / SCHED_CAPACITY_SCALE
         *
         * x is the ratio of sum_util compared to the CPU capacity:
         * x = sum_util / (llc_weight * SCHED_CAPACITY_SCALE)
         * y' is the ratio of CPUs to be scanned in the LLC domain,
         * and the number of CPUs to scan is calculated by:
         *
         * nr_scan = llc_weight * y'                                    [2]
         *
         * When x hits the threshold of overloaded, AKA, when
         * x = 100 / pct, y drops to 0. According to [1],
         * p should be SCHED_CAPACITY_SCALE * pct^2 / 10000
         *
         * Scale x by SCHED_CAPACITY_SCALE:
         * x' = sum_util / llc_weight;                                  [3]
         *
         * and finally [1] becomes:
         * y = SCHED_CAPACITY_SCALE -
         *     x'^2 * pct^2 / (10000 * SCHED_CAPACITY_SCALE)            [4]
         *
         */
        /* equation [3] */
        x = sum_util;
        llc_weight = sd->span_weight;
        do_div(x, llc_weight);

        /* equation [4] */
        pct = sd->imbalance_pct;
        tmp = x * x * pct * pct;
        do_div(tmp, 10000 * SCHED_CAPACITY_SCALE);
        tmp = min_t(long, tmp, SCHED_CAPACITY_SCALE);
        y = SCHED_CAPACITY_SCALE - tmp;

        /* equation [2] */
        y *= llc_weight;
        do_div(y, SCHED_CAPACITY_SCALE);
        if ((int)y != sd_share->nr_idle_scan)
                WRITE_ONCE(sd_share->nr_idle_scan, (int)y);
}
```

```cpp
/**     
 * update_sd_lb_stats - Update sched_domain's statistics for load balancing.
 * @env: The load balancing environment.
 * @sds: variable to hold the statistics for this sched_domain.
 */     

static inline void update_sd_lb_stats(struct lb_env *env, struct sd_lb_stats *sds)
{
        struct sched_group *sg = env->sd->groups;
        struct sg_lb_stats *local = &sds->local_stat;
        struct sg_lb_stats tmp_sgs;
        unsigned long sum_util = 0;
        bool sg_overloaded = 0, sg_overutilized = 0;
          
        env->dst_core_idle = !sched_smt_active() || is_core_idle(env->dst_cpu);
        
        do {
                struct sg_lb_stats *sgs = &tmp_sgs;
                int local_group;

                local_group = cpumask_test_cpu(env->dst_cpu, sched_group_span(sg));
                if (local_group) {
                        sds->local = sg;
                        sgs = local;

                        if (env->idle != CPU_NEWLY_IDLE ||
                            time_after_eq(jiffies, sg->sgc->next_update))
                                update_group_capacity(env->sd, env->dst_cpu);
                }

                update_sg_lb_stats(env, sds, sg, sgs, &sg_overloaded);

                if (!local_group && update_sd_pick_busiest(env, sds, sg, sgs)) {
                        sds->busiest = sg;
                        sds->busiest_stat = *sgs;
                }

                sg_overutilized |= sgs->group_overutilized;

                /* Now, start updating sd_lb_stats */
                sds->total_load += sgs->group_load;
                sds->total_capacity += sgs->group_capacity;

                sum_util += sgs->group_util;
                sg = sg->next;
        } while (sg != env->sd->groups);

        /*
         * Indicate that the child domain of the busiest group prefers tasks
         * go to a child's sibling domains first. NB the flags of a sched group
         * are those of the child domain.
         */
        if (sds->busiest)
                sds->prefer_sibling = !!(sds->busiest->flags & SD_PREFER_SIBLING);


        if (env->sd->flags & SD_NUMA)
                env->fbq_type = fbq_classify_group(&sds->busiest_stat);

        if (!env->sd->parent) {
                /* update overload indicator if we are at root domain */
                set_rd_overloaded(env->dst_rq->rd, sg_overloaded);

                /* Update over-utilization (tipping point, U >= 0) indicator */
                set_rd_overutilized(env->dst_rq->rd, sg_overutilized);
        } else if (sg_overutilized) {
                set_rd_overutilized(env->dst_rq->rd, sg_overutilized);
        }

        update_idle_cpu_scan(env, sum_util);
}
```

```cpp
#define NUMA_IMBALANCE_MIN 2
                
static inline long
adjust_numa_imbalance(int imbalance, int dst_running, int imb_numa_nr)
{
        /*      
         * Allow a NUMA imbalance if busy CPUs is less than the maximum
         * threshold. Above this threshold, individual tasks may be contending
         * for both memory bandwidth and any shared HT resources.  This is an
         * approximation as the number of running tasks may not be related to
         * the number of busy CPUs due to sched_setaffinity.
         */
        if (dst_running > imb_numa_nr)
                return imbalance;
   
        /*
         * Allow a small imbalance based on a simple pair of communicating
         * tasks that remain local when the destination is lightly loaded.
         */
        if (imbalance <= NUMA_IMBALANCE_MIN)
                return 0;

        return imbalance;
}

/**
 * calculate_imbalance - Calculate the amount of imbalance present within the
 *                       groups of a given sched_domain during load balance.
 * @env: load balance environment
 * @sds: statistics of the sched_domain whose imbalance is to be calculated.
 */
static inline void calculate_imbalance(struct lb_env *env, struct sd_lb_stats *sds)
{
        struct sg_lb_stats *local, *busiest;

        local = &sds->local_stat;
        busiest = &sds->busiest_stat;

        if (busiest->group_type == group_misfit_task) {
                if (env->sd->flags & SD_ASYM_CPUCAPACITY) {
                        /* Set imbalance to allow misfit tasks to be balanced. */
                        env->migration_type = migrate_misfit;
                        env->imbalance = 1;
                } else {
                        /*
                         * Set load imbalance to allow moving task from cpu
                         * with reduced capacity.
                         */
                        env->migration_type = migrate_load;
                        env->imbalance = busiest->group_misfit_task_load;
                }
                return;
        }

        if (busiest->group_type == group_asym_packing) {
                /*
                 * In case of asym capacity, we will try to migrate all load to
                 * the preferred CPU.
                 */
                env->migration_type = migrate_task;
                env->imbalance = busiest->sum_h_nr_running;
                return;
        }

        if (busiest->group_type == group_smt_balance) {
                /* Reduce number of tasks sharing CPU capacity */
                env->migration_type = migrate_task;
                env->imbalance = 1;
                return;
        }

#ifdef CONFIG_SCHED_CACHE
        if (busiest->group_type == group_llc_balance) {
                /* Move a task that prefer local LLC */
                env->migration_type = migrate_llc_task;
                env->imbalance = 1;
                return;
        }
#endif

        if (busiest->group_type == group_imbalanced) {
                /*
                 * In the group_imb case we cannot rely on group-wide averages
                 * to ensure CPU-load equilibrium, try to move any task to fix
                 * the imbalance. The next load balance will take care of
                 * balancing back the system.
                 */
                env->migration_type = migrate_task;
                env->imbalance = 1;
                return;
        }

        /*
         * Try to use spare capacity of local group without overloading it or
         * emptying busiest.
         */
        if (local->group_type == group_has_spare) {
                if ((busiest->group_type > group_fully_busy) &&
                    !(env->sd->flags & SD_SHARE_LLC)) {
                        /*
                         * If busiest is overloaded, try to fill spare
                         * capacity. This might end up creating spare capacity
                         * in busiest or busiest still being overloaded but
                         * there is no simple way to directly compute the
                         * amount of load to migrate in order to balance the
                         * system.
                         */
                        env->migration_type = migrate_util;
                        env->imbalance = max(local->group_capacity, local->group_util) -
                                         local->group_util;

                        /*
                         * In some cases, the group's utilization is max or even
                         * higher than capacity because of migrations but the
                         * local CPU is (newly) idle. There is at least one
                         * waiting task in this overloaded busiest group. Let's
                         * try to pull it.
                         */
                        if (env->idle && env->imbalance == 0) {
                                env->migration_type = migrate_task;
                                env->imbalance = 1;
                        }

                        return;
                }

                if (busiest->group_weight == 1 || sds->prefer_sibling) {
                        /*
                         * When prefer sibling, evenly spread running tasks on
                         * groups.
                         */
                        env->migration_type = migrate_task;
                        env->imbalance = sibling_imbalance(env, sds, busiest, local);
                } else {

                        /*
                         * If there is no overload, we just want to even the number of
                         * idle CPUs.
                         */
                        env->migration_type = migrate_task;
                        env->imbalance = max_t(long, 0,
                                               (local->idle_cpus - busiest->idle_cpus));
                }

#ifdef CONFIG_NUMA
                /* Consider allowing a small imbalance between NUMA groups */
                if (env->sd->flags & SD_NUMA) {
                        env->imbalance = adjust_numa_imbalance(env->imbalance,
                                                               local->sum_nr_running + 1,
                                                               env->sd->imb_numa_nr);
                }
#endif

                /* Number of tasks to move to restore balance */
                env->imbalance >>= 1;

                return;
        }

        /*
         * Local is fully busy but has to take more load to relieve the
         * busiest group
         */
        if (local->group_type < group_overloaded) {
                /*
                 * Local will become overloaded so the avg_load metrics are
                 * finally needed.
                 */

                local->avg_load = (local->group_load * SCHED_CAPACITY_SCALE) /
                                  local->group_capacity;

                /*
                 * If the local group is more loaded than the selected
                 * busiest group don't try to pull any tasks.
                 */
                if (local->avg_load >= busiest->avg_load) {
                        env->imbalance = 0;
                        return;
                }

                sds->avg_load = (sds->total_load * SCHED_CAPACITY_SCALE) /
                                sds->total_capacity;

                /*
                 * If the local group is more loaded than the average system
                 * load, don't try to pull any tasks.
                 */
                if (local->avg_load >= sds->avg_load) {
                        env->imbalance = 0;
                        return;
                }

        }

        /*
         * Both group are or will become overloaded and we're trying to get all
         * the CPUs to the average_load, so we don't want to push ourselves
         * above the average load, nor do we wish to reduce the max loaded CPU
         * below the average load. At the same time, we also don't want to
         * reduce the group load below the group capacity. Thus we look for
         * the minimum possible imbalance.
         */
        env->migration_type = migrate_load;
        env->imbalance = min(
                (busiest->avg_load - sds->avg_load) * busiest->group_capacity,
                (sds->avg_load - local->avg_load) * local->group_capacity
        ) / SCHED_CAPACITY_SCALE;
}
```

```cpp
/*
 * Decision matrix according to the local and busiest group type:
 *
 * busiest \ local has_spare fully_busy misfit asym imbalanced overloaded
 * has_spare        nr_idle   balanced   N/A    N/A  balanced   balanced
 * fully_busy       nr_idle   nr_idle    N/A    N/A  balanced   balanced
 * misfit_task      force     N/A        N/A    N/A  N/A        N/A
 * asym_packing     force     force      N/A    N/A  force      force
 * imbalanced       force     force      N/A    N/A  force      force
 * overloaded       force     force      N/A    N/A  force      avg_load
 *
 * N/A :      Not Applicable because already filtered while updating
 *            statistics.
 * balanced : The system is balanced for these 2 groups.
 * force :    Calculate the imbalance as load migration is probably needed.
 * avg_load : Only if imbalance is significant enough.
 * nr_idle :  dst_cpu is not busy and the number of idle CPUs is quite
 *            different in groups.
 */

/**
 * sched_balance_find_src_group - Returns the busiest group within the sched_domain
 * if there is an imbalance.
 * @env: The load balancing environment.
 *
 * Also calculates the amount of runnable load which should be moved
 * to restore balance.
 *
 * Return:      - The busiest group if imbalance exists.
 */     
static struct sched_group *sched_balance_find_src_group(struct lb_env *env)
{
        struct sg_lb_stats *local, *busiest;
        struct sd_lb_stats sds;

        init_sd_lb_stats(&sds);

        /*
         * Compute the various statistics relevant for load balancing at
         * this level.
         */
        update_sd_lb_stats(env, &sds);

        /* There is no busy sibling group to pull tasks from */
        if (!sds.busiest)
                goto out_balanced;

        busiest = &sds.busiest_stat;

        /* Misfit tasks should be dealt with regardless of the avg load */
        if (busiest->group_type == group_misfit_task)
                goto force_balance;

        if (!is_rd_overutilized(env->dst_rq->rd) &&
            rcu_dereference_all(env->dst_rq->rd->pd))
                goto out_balanced;

        /* ASYM feature bypasses nice load balance check */
        if (busiest->group_type == group_asym_packing)
                goto force_balance;

        /*
         * If the busiest group is imbalanced the below checks don't
         * work because they assume all things are equal, which typically
         * isn't true due to cpus_ptr constraints and the like.
         */
        if (busiest->group_type == group_imbalanced)
                goto force_balance;

        local = &sds.local_stat;
        /*
         * If the local group is busier than the selected busiest group
         * don't try and pull any tasks.
         */
        if (local->group_type > busiest->group_type)
                goto out_balanced;

        /*
         * When groups are overloaded, use the avg_load to ensure fairness
         * between tasks.
         */
        if (local->group_type == group_overloaded) {
                /*
                 * If the local group is more loaded than the selected
                 * busiest group don't try to pull any tasks.
                 */
                if (local->avg_load >= busiest->avg_load)
                        goto out_balanced;

                /* XXX broken for overlapping NUMA groups */
                sds.avg_load = (sds.total_load * SCHED_CAPACITY_SCALE) /
                                sds.total_capacity;

                /*
                 * Don't pull any tasks if this group is already above the
                 * domain average load.
                 */
                if (local->avg_load >= sds.avg_load)
                        goto out_balanced;

                /*
                 * If the busiest group is more loaded, use imbalance_pct to be
                 * conservative.
                 */
                if (100 * busiest->avg_load <=
                                env->sd->imbalance_pct * local->avg_load)
                        goto out_balanced;
        }

        /*
         * Try to move all excess tasks to a sibling domain of the busiest
         * group's child domain.
         */
        if (sds.prefer_sibling && local->group_type == group_has_spare &&
            (busiest->group_type == group_llc_balance ||
            sibling_imbalance(env, &sds, busiest, local) > 1))
                goto force_balance;

        if (busiest->group_type != group_overloaded) {
                if (!env->idle) {
                        /*
                         * If the busiest group is not overloaded (and as a
                         * result the local one too) but this CPU is already
                         * busy, let another idle CPU try to pull task.
                         */
                        goto out_balanced;
                }

                if (busiest->group_type == group_smt_balance &&
                    smt_vs_nonsmt_groups(sds.local, sds.busiest)) {
                        /* Let non SMT CPU pull from SMT CPU sharing with sibling */
                        goto force_balance;
                }

                if (busiest->group_weight > 1 &&
                    local->idle_cpus <= (busiest->idle_cpus + 1)) {
                        /*
                         * If the busiest group is not overloaded
                         * and there is no imbalance between this and busiest
                         * group wrt idle CPUs, it is balanced. The imbalance
                         * becomes significant if the diff is greater than 1
                         * otherwise we might end up to just move the imbalance
                         * on another group. Of course this applies only if
                         * there is more than 1 CPU per group.
                         */
                        goto out_balanced;
                }

                if (busiest->sum_h_nr_running == 1) {
                        /*
                         * busiest doesn't have any tasks waiting to run
                         */
                        goto out_balanced;
                }
        }

force_balance:
        /* Looks like there is an imbalance. Compute it */
        calculate_imbalance(env, &sds);
        return env->imbalance ? sds.busiest : NULL;

out_balanced:
        env->imbalance = 0;
        return NULL;
}
```

```cpp
/*
 * sched_balance_find_src_rq - find the busiest runqueue among the CPUs in the group.
 */
static struct rq *sched_balance_find_src_rq(struct lb_env *env,
                                     struct sched_group *group)
{
        struct rq *busiest = NULL, *rq;
        unsigned long busiest_util = 0, busiest_load = 0, busiest_capacity = 1;
        unsigned int __maybe_unused busiest_pref_llc = 0;
        struct sched_domain __maybe_unused *sd_tmp;
        unsigned int busiest_nr = 0;
        int __maybe_unused dst_llc;
        int i;  

        for_each_cpu_and(i, sched_group_span(group), env->cpus) {
                unsigned long capacity, load, util;
                unsigned int nr_running;
                enum fbq_type rt;

                rq = cpu_rq(i);
                rt = fbq_classify_rq(rq);

                /*
                 * We classify groups/runqueues into three groups:
                 *  - regular: there are !numa tasks
                 *  - remote:  there are numa tasks that run on the 'wrong' node
                 *  - all:     there is no distinction
                 *      
                 * In order to avoid migrating ideally placed numa tasks,
                 * ignore those when there's better options.
                 *      
                 * If we ignore the actual busiest queue to migrate another
                 * task, the next balance pass can still reduce the busiest
                 * queue by moving tasks around inside the node.
                 *
                 * If we cannot move enough load due to this classification
                 * the next pass will adjust the group classification and
                 * allow migration of more tasks.
                 *
                 * Both cases only affect the total convergence complexity.
                 */
                if (rt > env->fbq_type)
                        continue;

                nr_running = rq->cfs.h_nr_runnable;
                if (!nr_running)
                        continue;

                capacity = capacity_of(i);

                /*
                 * For ASYM_CPUCAPACITY domains, don't pick a CPU that could
                 * eventually lead to active_balancing high->low capacity.
                 * Higher per-CPU capacity is considered better than balancing
                 * average load.
                 */
                if (env->sd->flags & SD_ASYM_CPUCAPACITY &&
                    nr_running == 1) {
                        bool cluster_equal_cap = static_branch_unlikely(&sched_cluster_active) &&
                                                 (get_actual_cpu_capacity(env->dst_cpu) ==
                                                  get_actual_cpu_capacity(i));
                        bool smt_degraded_cap = sched_smt_active() && !is_core_idle(i);

                        /*
                         * Busy SMT siblings reduce the capacity of CPU @i. Do
                         * not skip it in this case.
                         *
                         * CONFIG_SCHED_CLUSTER requires balancing load across
                         * clusters of identical capacity, accounting for
                         * hardware and cpufreq pressure.
                         */
                        if (!smt_degraded_cap && !cluster_equal_cap &&
                            !capacity_greater(capacity_of(env->dst_cpu), capacity))
                                continue;
                }

                /*
                 * Make sure we only pull tasks from a CPU of lower priority
                 * when balancing between SMT siblings.
                 *
                 * If balancing between cores, let lower priority CPUs help
                 * SMT cores with more than one busy sibling.
                 */
                if (sched_asym(env->sd, i, env->dst_cpu) && nr_running == 1)
                        continue;

                switch (env->migration_type) {
                case migrate_load:
                        /*
                         * When comparing with load imbalance, use cpu_load()
                         * which is not scaled with the CPU capacity.
                         */
                        load = cpu_load(rq);

                        if (nr_running == 1 && load > env->imbalance &&
                            !check_cpu_capacity(rq, env->sd))
                                break;

                        /*
                         * For the load comparisons with the other CPUs,
                         * consider the cpu_load() scaled with the CPU
                         * capacity, so that the load can be moved away
                         * from the CPU that is potentially running at a
                         * lower capacity.
                         *
                         * Thus we're looking for max(load_i / capacity_i),
                         * crosswise multiplication to rid ourselves of the
                         * division works out to:
                         * load_i * capacity_j > load_j * capacity_i;
                         * where j is our previous maximum.
                         */
                        if (load * busiest_capacity > busiest_load * capacity) {
                                busiest_load = load;
                                busiest_capacity = capacity;
                                busiest = rq;
                        }
                        break;

                case migrate_util:
                        util = cpu_util_cfs_boost(i);

                        /*
                         * Don't try to pull utilization from a CPU with one
                         * running task. Whatever its utilization, we will fail
                         * detach the task.
                         */
                        if (nr_running <= 1)
                                continue;

                        if (busiest_util < util) {
                                busiest_util = util;
                                busiest = rq;
                        }
                        break;

                case migrate_task:
                        if (busiest_nr < nr_running) {
                                busiest_nr = nr_running;
                                busiest = rq;
                        }
                        break;

                case migrate_misfit:
                        /*
                         * For ASYM_CPUCAPACITY domains with misfit tasks we
                         * simply seek the "biggest" misfit task.
                         */
                        if (rq->misfit_task_load > busiest_load) {
                                busiest_load = rq->misfit_task_load;
                                busiest = rq;
                        }

                        break;

                case migrate_llc_task:
#ifdef CONFIG_SCHED_CACHE
                        sd_tmp = rcu_dereference_all(rq->sd);
                        dst_llc = llc_id(env->dst_cpu);

                        if (sd_tmp && (unsigned)dst_llc < sd_tmp->llc_max) {
                                unsigned int this_pref_llc =
                                        sd_tmp->llc_counts[dst_llc];

                                if (busiest_pref_llc < this_pref_llc) {
                                        busiest_pref_llc = this_pref_llc;
                                        busiest = rq;
                                }
                        }
#endif
                        break;

                }
        }

        return busiest;
}
```

```cpp
static inline unsigned long cfs_rq_runnable_avg(struct cfs_rq *cfs_rq)
{
        return cfs_rq->avg.runnable_avg;
}

static inline unsigned long cfs_rq_load_avg(struct cfs_rq *cfs_rq)
{               
        return cfs_rq->avg.load_avg; 
}

/*
 * Compute the hierarchical load factor for cfs_rq and all its ascendants.
 * This needs to be done in a top-down fashion because the load of a child
 * group is a fraction of its parents load.
 */
static void update_cfs_rq_h_load(struct cfs_rq *cfs_rq)
{
        struct sched_entity *se = cfs_rq_se(cfs_rq);
        unsigned long now = jiffies;
        unsigned long load;

        if (cfs_rq->last_h_load_update == now)
                return;

        WRITE_ONCE(cfs_rq->h_load_next, NULL);
        for_each_sched_entity(se) {
                cfs_rq = cfs_rq_of(se);
                WRITE_ONCE(cfs_rq->h_load_next, se);
                if (cfs_rq->last_h_load_update == now)
                        break;
        }

        if (!se) {      
                cfs_rq->h_load = cfs_rq_load_avg(cfs_rq);
                cfs_rq->last_h_load_update = now;
        }

        while ((se = READ_ONCE(cfs_rq->h_load_next)) != NULL) {
                load = cfs_rq->h_load;
                load = div64_ul(load * se->avg.load_avg,
                                cfs_rq_load_avg(cfs_rq) + 1);
                cfs_rq = group_cfs_rq(se);
                cfs_rq->h_load = load;
                cfs_rq->last_h_load_update = now;
        }
}

static unsigned long task_h_load(struct task_struct *p)
{
        struct cfs_rq *cfs_rq = task_cfs_rq(p);

        update_cfs_rq_h_load(cfs_rq);
        return div64_ul(p->se.avg.load_avg * cfs_rq->h_load,
                        cfs_rq_load_avg(cfs_rq) + 1);
}
```

```cpp
/*
 * can_migrate_task - may task p from runqueue rq be migrated to this_cpu?
 */
static
int can_migrate_task(struct task_struct *p, struct lb_env *env)
{
        long degrades, hot;

        lockdep_assert_rq_held(env->src_rq);
        if (p->sched_task_hot)
                p->sched_task_hot = 0;

        /*
         * We do not migrate tasks that are:
         * 1) delayed dequeued unless we migrate load, or
         * 2) target cfs_rq is in throttled hierarchy, or
         * 3) cannot be migrated to this CPU due to cpus_ptr, or
         * 4) running (obviously), or
         * 5) are cache-hot on their current CPU, or
         * 6) are blocked on mutexes (if SCHED_PROXY_EXEC is enabled)
         */
        if ((p->se.sched_delayed) && (env->migration_type != migrate_load))
                return 0;

        if (lb_throttled_hierarchy(p, env->dst_cpu))
                return 0;

        /*
         * We want to prioritize the migration of eligible tasks.
         * For ineligible tasks we soft-limit them and only allow
         * them to migrate when nr_balance_failed is non-zero to
         * avoid load-balancing trying very hard to balance the load.
         */
        if (!env->sd->nr_balance_failed &&
            task_is_ineligible_on_dst_cpu(p, env->dst_cpu))
                return 0;

        /* Disregard percpu kthreads; they are where they need to be. */
        if (kthread_is_per_cpu(p))
                return 0;

        if (task_is_blocked(p))
                return 0;

        if (!cpumask_test_cpu(env->dst_cpu, p->cpus_ptr)) {
                int cpu;

                schedstat_inc(p->stats.nr_failed_migrations_affine);

                env->flags |= LBF_SOME_PINNED;

                /*
                 * Remember if this task can be migrated to any other CPU in
                 * our sched_group. We may want to revisit it if we couldn't
                 * meet load balance goals by pulling other tasks on src_cpu.
                 *
                 * Avoid computing new_dst_cpu
                 * - for NEWLY_IDLE
                 * - if we have already computed one in current iteration
                 * - if it's an active balance
                 */
                if (env->idle == CPU_NEWLY_IDLE ||
                    env->flags & (LBF_DST_PINNED | LBF_ACTIVE_LB))
                        return 0;

                /* Prevent to re-select dst_cpu via env's CPUs: */
                cpu = cpumask_first_and_and(env->dst_grpmask, env->cpus, p->cpus_ptr);

                if (cpu < nr_cpu_ids) {
                        env->flags |= LBF_DST_PINNED;
                        env->new_dst_cpu = cpu;
                }

                return 0;
        }

        /* Record that we found at least one task that could run on dst_cpu */
        env->flags &= ~LBF_ALL_PINNED;

        if (task_on_cpu(env->src_rq, p) ||
            task_current_donor(env->src_rq, p)) {
                schedstat_inc(p->stats.nr_failed_migrations_running);
                return 0;
        }

        /*
         * Aggressive migration if:
         * 1) active balance
         * 2) destination numa is preferred
         * 3) task is cache cold, or
         * 4) too many balance attempts have failed.
         */
        if (env->flags & LBF_ACTIVE_LB)
                return !migrate_llc_task_wrong_dst(p, env);

        degrades = migrate_degrades_locality(p, env);
        if (!degrades) {
                /*
                 * If the NUMA locality is not broken,
                 * further check if migration would hurt
                 * LLC locality.
                 */
                if (migrate_degrades_llc(p, env)) {
                        /*
                         * If regular load balancing fails to pull a task
                         * due to LLC locality, this is expected behavior
                         * and we set LBF_LLC_PINNED so we don't increase
                         * nr_balance_failed unecessarily.
                         */
                        if (env->migration_type != migrate_llc_task)
                                env->flags |= LBF_LLC_PINNED;

                        return 0;
                }

                hot = task_hot(p, env);
        } else {
                hot = degrades > 0;
        }

        if (!hot || env->sd->nr_balance_failed > env->sd->cache_nice_tries) {
                if (hot)
                        p->sched_task_hot = 1;
                return 1;
        }

        schedstat_inc(p->stats.nr_failed_migrations_hot);
        return 0;
}


/*
 * detach_task() -- detach the task for the migration specified in env
 */
static void detach_task(struct task_struct *p, struct lb_env *env)
{
        lockdep_assert_rq_held(env->src_rq);

        if (p->sched_task_hot) {
                p->sched_task_hot = 0;
                schedstat_inc(env->sd->lb_hot_gained[env->idle]);
                schedstat_inc(p->stats.nr_forced_migrations);
        }

        WARN_ON(task_current(env->src_rq, p));
        WARN_ON(task_current_donor(env->src_rq, p));

        deactivate_task(env->src_rq, p, DEQUEUE_NOCLOCK);
        set_task_cpu(p, env->dst_cpu);
}


/*
 * detach_one_task() -- tries to dequeue exactly one task from env->src_rq, as
 * part of active balancing operations within "domain".
 *
 * Returns a task if successful and NULL otherwise.
 */
static struct task_struct *detach_one_task(struct lb_env *env)
{
        struct task_struct *p;

        lockdep_assert_rq_held(env->src_rq);

        list_for_each_entry_reverse(p,
                        &env->src_rq->cfs_tasks, se.group_node) {
                if (!can_migrate_task(p, env))
                        continue;

                detach_task(p, env);

                /*
                 * Right now, this is only the second place where
                 * lb_gained[env->idle] is updated (other is detach_tasks)
                 * so we can safely collect stats here rather than
                 * inside detach_tasks().
                 */
                schedstat_inc(env->sd->lb_gained[env->idle]);
                return p;
        }
        return NULL;
}

/*
 * detach_tasks() -- tries to detach up to imbalance load/util/tasks from
 * busiest_rq, as part of a balancing operation within domain "sd".
 *
 * Returns number of detached tasks if successful and 0 otherwise.
 */
static int detach_tasks(struct lb_env *env)
{
        struct list_head *tasks = &env->src_rq->cfs_tasks;
        unsigned long util, load;
        struct task_struct *p;
        int detached = 0;

        lockdep_assert_rq_held(env->src_rq);

        /*
         * Source run queue has been emptied by another CPU, clear
         * LBF_ALL_PINNED flag as we will not test any task.
         */
        if (env->src_rq->nr_running <= 1) {
                env->flags &= ~LBF_ALL_PINNED;
                return 0;
        }

        if (env->imbalance <= 0)
                return 0;

        while (!list_empty(tasks)) {
                /*
                 * We don't want to steal all, otherwise we may be treated likewise,
                 * which could at worst lead to a livelock crash.
                 */
                if (env->idle && env->src_rq->nr_running <= 1)
                        break;

                env->loop++;
                /* We've more or less seen every task there is, call it quits */
                if (env->loop > env->loop_max)
                        break;

                /* take a breather every nr_migrate tasks */
                if (env->loop > env->loop_break) {
                        env->loop_break += SCHED_NR_MIGRATE_BREAK;
                        env->flags |= LBF_NEED_BREAK;
                        break;
                }

                p = list_last_entry(tasks, struct task_struct, se.group_node);

                if (!can_migrate_task(p, env))
                        goto next;

                switch (env->migration_type) {
                case migrate_load:
                        /*
                         * Depending of the number of CPUs and tasks and the
                         * cgroup hierarchy, task_h_load() can return a null
                         * value. Make sure that env->imbalance decreases
                         * otherwise detach_tasks() will stop only after
                         * detaching up to loop_max tasks.
                         */
                        load = max_t(unsigned long, task_h_load(p), 1);

                        if (sched_feat(LB_MIN) &&
                            load < 16 && !env->sd->nr_balance_failed)
                                goto next;

                        /*
                         * Make sure that we don't migrate too much load.
                         * Nevertheless, let relax the constraint if
                         * scheduler fails to find a good waiting task to
                         * migrate.
                         */
                        if (shr_bound(load, env->sd->nr_balance_failed) > env->imbalance)
                                goto next;

                        env->imbalance -= load;
                        break;

                case migrate_util:
                        util = task_util_est(p);

                        if (shr_bound(util, env->sd->nr_balance_failed) > env->imbalance)
                                goto next;

                        env->imbalance -= util;
                        break;

                case migrate_task:
                        env->imbalance--;
                        break;

                case migrate_misfit:
                        /* This is not a misfit task */
                        if (task_fits_cpu(p, env->src_cpu))
                                goto next;

                        env->imbalance = 0;
                        break;

                case migrate_llc_task:
                        env->imbalance--;
                        break;
                }

                detach_task(p, env);
                list_add(&p->se.group_node, &env->tasks);

                detached++;

#ifdef CONFIG_PREEMPTION
                /*
                 * NEWIDLE balancing is a source of latency, so preemptible
                 * kernels will stop after the first task is detached to minimize
                 * the critical section.
                 */
                if (env->idle == CPU_NEWLY_IDLE)
                        break;
#endif

                /*
                 * We only want to steal up to the prescribed amount of
                 * load/util/tasks.
                 */
                if (env->imbalance <= 0)
                        break;

                continue;
next:
                if (p->sched_task_hot)
                        schedstat_inc(p->stats.nr_failed_migrations_hot);

                list_move(&p->se.group_node, tasks);
        }

        /*
         * Right now, this is one of only two places we collect this stat
         * so we can safely collect detach_one_task() stats here rather
         * than inside detach_one_task().
         */
        schedstat_add(env->sd->lb_gained[env->idle], detached);

        return detached;
}
```

```cpp
/*
 * attach_task() -- attach the task detached by detach_task() to its new rq.
 */
static inline void attach_task(struct rq *rq, struct task_struct *p)
{
        lockdep_assert_rq_held(rq);

        WARN_ON_ONCE(task_rq(p) != rq);
        activate_task(rq, p, ENQUEUE_NOCLOCK);
        wakeup_preempt(rq, p, 0);
}

/*
 * attach_one_task() -- attaches the task returned from detach_one_task() to
 * its new rq.
 */
static inline void attach_one_task(struct rq *rq, struct task_struct *p)
{
        guard(rq_lock)(rq);
        update_rq_clock(rq);
        attach_task(rq, p);
}

/*
 * attach_tasks() -- attaches all tasks detached by detach_tasks() to their
 * new rq.
 */
static void attach_tasks(struct lb_env *env)
{
        struct list_head *tasks = &env->tasks;
        struct task_struct *p;
        struct rq_flags rf;

        rq_lock(env->dst_rq, &rf);
        update_rq_clock(env->dst_rq);

        while (!list_empty(tasks)) {
                p = list_first_entry(tasks, struct task_struct, se.group_node);
                list_del_init(&p->se.group_node);

                attach_task(env->dst_rq, p);
        }

        rq_unlock(env->dst_rq, &rf);
}
```

```cpp
static void update_lb_imbalance_stat(struct lb_env *env, struct sched_domain *sd,
                                     enum cpu_idle_type idle) 
{
        if (!schedstat_enabled())
                return;

        switch (env->migration_type) {
        case migrate_load:
                __schedstat_add(sd->lb_imbalance_load[idle], env->imbalance);
                break;
        case migrate_util:
                __schedstat_add(sd->lb_imbalance_util[idle], env->imbalance);
                break;
        case migrate_task:
                __schedstat_add(sd->lb_imbalance_task[idle], env->imbalance);
                break;
        case migrate_misfit:
                __schedstat_add(sd->lb_imbalance_misfit[idle], env->imbalance);
                break;
        case migrate_llc_task:
                break;
        }     
}
```

```cpp
/*    
 * migration_type is checked elsewhere to decide migration policy, so
 * it shouldn't be repurposed just to flag an LLC-directed active
 * balance across the stopper. Pick the callback here instead.
 */
static inline cpu_stop_fn_t alb_stop_fn(struct lb_env *env)
{   
        if (env->migration_type == migrate_llc_task)
                return active_load_balance_llc_cpu_stop;

        return active_load_balance_cpu_stop;
}
```

```cpp
static inline bool task_misfits_asym_cpu(struct lb_env *env, struct task_struct *p)
{
        /*
         * On asymmetric CPU capacity domains, do not let cache-aware
         * balancing pull the task onto a destination CPU that cannot
         * accommodate it. Doing so would turn the task into a misfit on
         * the destination, trading a cache-locality gain for a capacity
         * loss. If the task already does not fit its source CPU, the move
         * cannot make things worse, so let the LLC preference decide.
         */
        if ((env->sd->flags & SD_ASYM_CPUCAPACITY) && p &&
            !task_fits_cpu(p, env->dst_cpu) &&
            task_fits_cpu(p, env->src_cpu))
                return true;

        return false;
}
```

```cpp
/*
 * Check if active load balance breaks LLC locality in
 * terms of cache aware load balance. The load level and
 * imbalance do not warrant breaking LLC preference per
 * the can_migrate_llc() policy. Here, the benefit of
 * LLC locality outweighs the power efficiency gained from
 * migrating the only runnable task away.
 */
static inline bool
alb_break_llc(struct lb_env *env)
{
        if (!sched_cache_enabled())
                return false;

        if (cpus_share_cache(env->src_cpu, env->dst_cpu))
                return false;
        /*
         * All tasks prefer to stay on their current CPU.
         * Do not pull a task from its preferred CPU if:
         * 1. It is the only task running and does not exceed
         *    imbalance allowance; OR
         * 2. Migrating it away from its preferred LLC would violate
         *    the cache-aware scheduling policy.
         */
        if (env->src_rq->nr_pref_llc_running &&
            env->src_rq->nr_pref_llc_running == env->src_rq->cfs.h_nr_runnable) {
                unsigned long util = 0;
                struct task_struct *cur;

                /*
                 * Migrating misfit tasks from current CPU
                 * to CPU with a better fit.
                 * Prioritize that over LLC preference.
                 */
                if (env->migration_type == migrate_misfit)
                        return false;

                if (env->src_rq->nr_running <= 1)
                        return true;

                cur = rcu_dereference_all(env->src_rq->curr);
                if (cur && cur->sched_class == &fair_sched_class)
                        util = task_util(cur);

                if (task_misfits_asym_cpu(env, cur) ||
                    can_migrate_llc(env->src_cpu, env->dst_cpu,
                                    util, false) == mig_forbid)
                        return true;
        }

        return false;
}
```

```cpp
static inline bool
asym_active_balance(struct lb_env *env)
{
        /*
         * ASYM_PACKING needs to force migrate tasks from busy but lower
         * priority CPUs in order to pack all tasks in the highest priority
         * CPUs. When done between cores, do it only if the whole core if the
         * whole core is idle.
         * 
         * If @env::src_cpu is an SMT core with busy siblings, let
         * the lower priority @env::dst_cpu help it. Do not follow
         * CPU priority.
         */     
        return env->idle && sched_use_asym_prio(env->sd, env->dst_cpu) &&
               (sched_asym_prefer(env->dst_cpu, env->src_cpu) ||
                !sched_use_asym_prio(env->sd, env->src_cpu));
}

static inline bool
imbalanced_active_balance(struct lb_env *env)
{       
        struct sched_domain *sd = env->sd;

        /*
         * The imbalanced case includes the case of pinned tasks preventing a fair
         * distribution of the load on the system but also the even distribution of the
         * threads on a system with spare capacity
         */
        if ((env->migration_type == migrate_task) &&
            (sd->nr_balance_failed > sd->cache_nice_tries+2))
                return 1;

        return 0;
}
```

```cpp
static int need_active_balance(struct lb_env *env)
{
        struct sched_domain *sd = env->sd;

        if (alb_break_llc(env))
                return 0;

        if (asym_active_balance(env))
                return 1;

        if (imbalanced_active_balance(env))
                return 1;

        /*
         * The dst_cpu is idle and the src_cpu CPU has only 1 CFS task.
         * It's worth migrating the task if the src_cpu's capacity is reduced
         * because of other sched_class or IRQs if more capacity stays
         * available on dst_cpu.
         */
        if (env->idle &&
            (env->src_rq->cfs.h_nr_runnable == 1)) {
                if ((check_cpu_capacity(env->src_rq, sd)) &&
                    (capacity_of(env->src_cpu)*sd->imbalance_pct < capacity_of(env->dst_cpu)*100))
                        return 1;
        }
        
        if (env->migration_type == migrate_misfit ||
            env->migration_type == migrate_llc_task)
                return 1;

        return 0;
}
```

```cpp
/*
 * This flag serializes load-balancing passes over large domains
 * (above the NODE topology level) - only one load-balancing instance
 * may run at a time, to reduce overhead on very large systems with
 * lots of CPUs and large NUMA distances.
 *
 * - Note that load-balancing passes triggered while another one
 *   is executing are skipped and not re-tried. 
 *
 * - Also note that this does not serialize rebalance_domains()
 *   execution, as non-SD_SERIALIZE domains will still be
 *   load-balanced in parallel.
 */
static atomic_t sched_balance_running = ATOMIC_INIT(0); 

/*
 * Check this_cpu to ensure it is balanced within domain. Attempt to move
 * tasks if there is an imbalance.
 */
static int sched_balance_rq(int this_cpu, struct rq *this_rq,
                        struct sched_domain *sd, enum cpu_idle_type idle,
                        int *continue_balancing)
{
        int ld_moved, cur_ld_moved, active_balance = 0;
        struct sched_domain *sd_parent = sd->parent;
        struct sched_group *group;
        struct rq *busiest;
        struct rq_flags rf;
        struct cpumask *cpus = this_cpu_cpumask_var_ptr(load_balance_mask);
        struct lb_env env = {
                .sd             = sd,
                .dst_cpu        = this_cpu,
                .dst_rq         = this_rq,
                .dst_grpmask    = group_balance_mask(sd->groups),
                .idle           = idle,
                .loop_break     = SCHED_NR_MIGRATE_BREAK,
                .cpus           = cpus,
                .fbq_type       = all,
                .tasks          = LIST_HEAD_INIT(env.tasks),
        };
        bool need_unlock = false;

        cpumask_and(cpus, sched_domain_span(sd), cpu_active_mask);

        schedstat_inc(sd->lb_count[idle]);

redo:
        if (!should_we_balance(&env)) {
                *continue_balancing = 0;
                goto out_balanced;
        }

        if (!need_unlock && (sd->flags & SD_SERIALIZE)) {
                int zero = 0;
                if (!atomic_try_cmpxchg_acquire(&sched_balance_running, &zero, 1))
                        goto out_balanced;

                need_unlock = true;
        }

        group = sched_balance_find_src_group(&env);
        if (!group) {
                schedstat_inc(sd->lb_nobusyg[idle]);
                goto out_balanced;
        }

        busiest = sched_balance_find_src_rq(&env, group);
        if (!busiest) {
                schedstat_inc(sd->lb_nobusyq[idle]);
                goto out_balanced;
        }

        WARN_ON_ONCE(busiest == env.dst_rq);

        update_lb_imbalance_stat(&env, sd, idle);

        env.src_cpu = busiest->cpu;
        env.src_rq = busiest;

        ld_moved = 0;
        /* Clear this flag as soon as we find a pullable task */
        env.flags |= LBF_ALL_PINNED;
        if (busiest->nr_running > 1) {
                /*
                 * Attempt to move tasks. If sched_balance_find_src_group has found
                 * an imbalance but busiest->nr_running <= 1, the group is
                 * still unbalanced. ld_moved simply stays zero, so it is
                 * correctly treated as an imbalance.
                 */
                env.loop_max  = min(sysctl_sched_nr_migrate, busiest->nr_running);

more_balance:
                rq_lock_irqsave(busiest, &rf);
                update_rq_clock(busiest);

                /*
                 * cur_ld_moved - load moved in current iteration
                 * ld_moved     - cumulative load moved across iterations
                 */
                cur_ld_moved = detach_tasks(&env);

                /*
                 * We've detached some tasks from busiest_rq. Every
                 * task is masked "TASK_ON_RQ_MIGRATING", so we can safely
                 * unlock busiest->lock, and we are able to be sure
                 * that nobody can manipulate the tasks in parallel.
                 * See task_rq_lock() family for the details.
                 */

                rq_unlock(busiest, &rf);

                if (cur_ld_moved) {
                        attach_tasks(&env);
                        ld_moved += cur_ld_moved;
                }

                local_irq_restore(rf.flags);

                if (env.flags & LBF_NEED_BREAK) {
                        env.flags &= ~LBF_NEED_BREAK;
                        goto more_balance;
                }

                /*
                 * Revisit (affine) tasks on src_cpu that couldn't be moved to
                 * us and move them to an alternate dst_cpu in our sched_group
                 * where they can run. The upper limit on how many times we
                 * iterate on same src_cpu is dependent on number of CPUs in our
                 * sched_group.
                 *
                 * This changes load balance semantics a bit on who can move
                 * load to a given_cpu. In addition to the given_cpu itself
                 * (or a ilb_cpu acting on its behalf where given_cpu is
                 * nohz-idle), we now have balance_cpu in a position to move
                 * load to given_cpu. In rare situations, this may cause
                 * conflicts (balance_cpu and given_cpu/ilb_cpu deciding
                 * _independently_ and at _same_ time to move some load to
                 * given_cpu) causing excess load to be moved to given_cpu.
                 * This however should not happen so much in practice and
                 * moreover subsequent load balance cycles should correct the
                 * excess load moved.
                 */
                if ((env.flags & LBF_DST_PINNED) && env.imbalance > 0) {

                        /* Prevent to re-select dst_cpu via env's CPUs */
                        __cpumask_clear_cpu(env.dst_cpu, env.cpus);

                        env.dst_rq       = cpu_rq(env.new_dst_cpu);
                        env.dst_cpu      = env.new_dst_cpu;
                        env.flags       &= ~LBF_DST_PINNED;
                        env.loop         = 0;
                        env.loop_break   = SCHED_NR_MIGRATE_BREAK;

                        /*
                         * Go back to "more_balance" rather than "redo" since we
                         * need to continue with same src_cpu.
                         */
                        goto more_balance;
                }

                /*
                 * We failed to reach balance because of affinity.
                 */
                if (sd_parent) {
                        int *group_imbalance = &sd_parent->groups->sgc->imbalance;

                        if ((env.flags & LBF_SOME_PINNED) && env.imbalance > 0)
                                *group_imbalance = 1;
                }

                /* All tasks on this runqueue were pinned by CPU affinity */
                if (unlikely(env.flags & LBF_ALL_PINNED)) {
                        __cpumask_clear_cpu(cpu_of(busiest), cpus);
                        /*
                         * Attempting to continue load balancing at the current
                         * sched_domain level only makes sense if there are
                         * active CPUs remaining as possible busiest CPUs to
                         * pull load from which are not contained within the
                         * destination group that is receiving any migrated
                         * load.
                         */
                        if (!cpumask_subset(cpus, env.dst_grpmask)) {
                                env.loop = 0;
                                env.loop_break = SCHED_NR_MIGRATE_BREAK;
                                goto redo;
                        }
                        goto out_all_pinned;
                }
        }

        if (ld_moved) {
                sd->nr_balance_failed = 0;
                goto out_unbalanced;
        }

        schedstat_inc(sd->lb_failed[idle]);
        /*
         * Increment the failure counter only on periodic balance.
         * We do not want newidle balance, which can be very
         * frequent, pollute the failure counter causing
         * excessive cache_hot migrations and active balances.
         *
         * Similarly for migration_misfit which is not related to
         * load/util migration, don't pollute nr_balance_failed.
         *
         * The same for cache aware scheduling's allowance for
         * load imbalance. If regular load balance does not
         * migrate task due to LLC locality, it is a expected
         * behavior and don't pollute nr_balance_failed.
         * See can_migrate_task().
         */
        if (idle != CPU_NEWLY_IDLE &&
            env.migration_type != migrate_misfit &&
            !(env.flags & LBF_LLC_PINNED))
                sd->nr_balance_failed++;

        if (!need_active_balance(&env))
                goto out_unbalanced;

        scoped_guard (raw_spin_rq_lock_irqsave, busiest) {
                /*
                 * Don't kick the active_load_balance_cpu_stop,
                 * if the curr task on busiest CPU can't be
                 * moved to this_cpu:
                 */
                if (!cpumask_test_cpu(this_cpu, busiest->curr->cpus_ptr))
                        goto out_one_pinned;

                /* Record that we found at least one task that could run on this_cpu */
                env.flags &= ~LBF_ALL_PINNED;

                /*
                 * ->active_balance synchronizes accesses to
                 * ->active_balance_work.  Once set, it's cleared
                 * only after active load balance is finished.
                 */
                if (busiest->active_balance)
                        goto out_unbalanced;

                /*
                 * @busiest dropped its rq_lock in the middle of
                 * scheduling out its ->curr task (->on_rq := 0), no
                 * need to forcefully punt it away with active balance.
                 */
                if (!busiest->curr->on_rq)
                        goto out_unbalanced;

                busiest->active_balance = 1;
                busiest->push_cpu = this_cpu;
                active_balance = 1;
                preempt_disable();
        }
        if (active_balance) {
                stop_one_cpu_nowait(cpu_of(busiest),
                                    alb_stop_fn(&env), busiest,
                                    &busiest->active_balance_work);
        }
        preempt_enable();

out_unbalanced:
        /* We were unbalanced, so reset the balancing interval */
        sd->balance_interval = sd->min_interval;
        goto out;

out_balanced:
        /*
         * We reach balance although we may have faced some affinity
         * constraints. Clear the imbalance flag only if other tasks got
         * a chance to move and fix the imbalance.
         */
        if (sd_parent && !(env.flags & LBF_ALL_PINNED)) {
                int *group_imbalance = &sd_parent->groups->sgc->imbalance;

                if (*group_imbalance)
                        *group_imbalance = 0;
        }

out_all_pinned:
        /*
         * We reach balance because all tasks are pinned at this level so
         * we can't migrate them. Let the imbalance flag set so parent level
         * can try to migrate them.
         */
        schedstat_inc(sd->lb_balanced[idle]);

        sd->nr_balance_failed = 0;

out_one_pinned:
        ld_moved = 0;

        /*
         * sched_balance_newidle() disregards balance intervals, so we could
         * repeatedly reach this code, which would lead to balance_interval
         * skyrocketing in a short amount of time. Skip the balance_interval
         * increase logic to avoid that.
         *
         * Similarly misfit migration which is not necessarily an indication of
         * the system being busy and requires lb to backoff to let it settle
         * down.
         */
        if (env.idle == CPU_NEWLY_IDLE ||
            env.migration_type == migrate_misfit)
                goto out;

        /* tune up the balancing interval */
        if ((env.flags & LBF_ALL_PINNED &&
             sd->balance_interval < MAX_PINNED_INTERVAL) ||
            sd->balance_interval < sd->max_interval)
                sd->balance_interval *= 2;
out:
        if (need_unlock)
                atomic_set_release(&sched_balance_running, 0);

        return ld_moved;
}
```

```cpp
/* Runqueue only has SCHED_IDLE tasks enqueued */
static int sched_idle_rq(struct rq *rq)
{
        return unlikely(rq->nr_running == rq->cfs.h_nr_idle &&
                        rq->nr_running);
}
```

```cpp
static inline void update_newidle_stats(struct sched_domain *sd, unsigned int success)
{
        sd->newidle_call++;
        sd->newidle_success += success;

        if (sd->newidle_call >= 1024) {
                u64 now = sched_clock();
                s64 delta = now - sd->newidle_stamp;
                sd->newidle_stamp = now;
                int ratio = 0;

                if (delta < 0)
                        delta = 0;

                if (sched_feat(NI_RATE)) {
                        /*
                         * ratio  delta   freq
                         *
                         * 1024 -  4  s -  128 Hz
                         *  512 -  2  s -  256 Hz
                         *  256 -  1  s -  512 Hz
                         *  128 - .5  s - 1024 Hz
                         *   64 - .25 s - 2048 Hz
                         */
                        ratio = delta >> 22;
                }

                ratio += sd->newidle_success;

                sd->newidle_ratio = min(1024, ratio);
                sd->newidle_call /= 2;
                sd->newidle_success /= 2;
        }
}

static inline bool
update_newidle_cost(struct sched_domain *sd, u64 cost, unsigned int success)
{
        unsigned long next_decay = sd->last_decay_max_lb_cost + HZ;
        unsigned long now = jiffies;

        if (cost)
                update_newidle_stats(sd, success);

        if (cost > sd->max_newidle_lb_cost) {
                /*
                 * Track max cost of a domain to make sure to not delay the
                 * next wakeup on the CPU.
                 */
                sd->max_newidle_lb_cost = cost;
                sd->last_decay_max_lb_cost = now;

        } else if (time_after(now, next_decay)) {
                /*
                 * Decay the newidle max times by ~1% per second to ensure that
                 * it is not outdated and the current max cost is actually
                 * shorter.
                 */
                sd->max_newidle_lb_cost = (sd->max_newidle_lb_cost * 253) / 256;
                sd->last_decay_max_lb_cost = now;
                return true;
        }

        return false;
}
```

```cpp
static inline unsigned long
get_sd_balance_interval(struct sched_domain *sd, int cpu_busy)
{
        unsigned long interval = sd->balance_interval;

        if (cpu_busy)
                interval *= sd->busy_factor;

        /* scale ms to jiffies */
        interval = msecs_to_jiffies(interval);

        /*    
         * Reduce likelihood of busy balancing at higher domains racing with
         * balancing at lower domains by preventing their balancing periods
         * from being multiples of each other.
         */
        if (cpu_busy)
                interval -= 1; 

        interval = clamp(interval, 1UL, max_load_balance_interval);

        return interval;
}
```

```cpp
static inline void
update_next_balance(struct sched_domain *sd, unsigned long *next_balance)
{
        unsigned long interval, next;

        /* used by idle balance, so cpu_busy = 0 */
        interval = get_sd_balance_interval(sd, 0);
        next = sd->last_balance + interval;

        if (time_after(*next_balance, next))
                *next_balance = next;
}
```

```cpp
static inline void rq_modified_begin(struct rq *rq, const struct sched_class *class)
{
        if (sched_class_above(rq->next_class, class))
                rq->next_class = class;
}

static inline bool rq_modified_above(struct rq *rq, const struct sched_class *class)
{
        return sched_class_above(rq->next_class, class);
}
```

```cpp
/*
 * It checks each scheduling domain to see if it is due to be balanced,
 * and initiates a balancing operation if so.
 *
 * Balancing parameters are set up in init_sched_domains.
 */
static void sched_balance_domains(struct rq *rq, enum cpu_idle_type idle) 
{
        int continue_balancing = 1;
        int cpu = rq->cpu;
        int busy = idle != CPU_IDLE && !sched_idle_rq(rq);
        unsigned long interval;
        struct sched_domain *sd;
        /* Earliest time when we have to do rebalance again */
        unsigned long next_balance = jiffies + 60*HZ;
        int update_next_balance = 0;
        int need_decay = 0;
        u64 max_cost = 0;

        rcu_read_lock();
        for_each_domain(cpu, sd) { 
                /*    
                 * Decay the newidle max times here because this is a regular
                 * visit to all the domains.
                 */
                need_decay = update_newidle_cost(sd, 0, 0);
                max_cost += sd->max_newidle_lb_cost;

                /*
                 * Stop the load balance at this level. There is another
                 * CPU in our sched group which is doing load balancing more
                 * actively.
                */
                if (!continue_balancing) {
                        if (need_decay)
                                continue;
                        break;
                }

                interval = get_sd_balance_interval(sd, busy);
                if (time_after_eq(jiffies, sd->last_balance + interval)) {
                        if (sched_balance_rq(cpu, rq, sd, idle, &continue_balancing)) {
                                /*
                                 * The LBF_DST_PINNED logic could have changed
                                 * env->dst_cpu, so we can't know our idle
                                 * state even if we migrated tasks. Update it.
                                 */
                                idle = idle_cpu(cpu);
                                busy = !idle && !sched_idle_rq(rq);
                        }
                        sd->last_balance = jiffies;
                        interval = get_sd_balance_interval(sd, busy);
                }
                if (time_after(next_balance, sd->last_balance + interval)) {
                        next_balance = sd->last_balance + interval;
                        update_next_balance = 1;
                }
        }
        if (need_decay) {
                /*
                 * Ensure the rq-wide value also decays but keep it at a
                 * reasonable floor to avoid funnies with rq->avg_idle.
                 */
                rq->max_idle_balance_cost =
                        max((u64)sysctl_sched_migration_cost, max_cost);
        }
        rcu_read_unlock();

        /*
         * next_balance will be updated only when there is a need.
         * When the cpu is attached to null domain for ex, it will not be
         * updated.
         */
        if (likely(update_next_balance))
                rq->next_balance = next_balance;

}
```

```cpp
static void nohz_newidle_balance(struct rq *this_rq)
{
        int this_cpu = this_rq->cpu;

        /* Will wake up very soon. No time for doing anything else*/
        if (this_rq->avg_idle < sysctl_sched_migration_cost)
                return;

        /* Don't need to update blocked load of idle CPUs*/
        if (!READ_ONCE(nohz.has_blocked_load) ||
            time_before(jiffies, READ_ONCE(nohz.next_blocked)))
                return;

        /*
         * Set the need to trigger ILB in order to update blocked load
         * before entering idle state.
         */
        atomic_or(NOHZ_NEWILB_KICK, nohz_flags(this_cpu));
}
```

```cpp
/*
 * sched_balance_newidle is called by schedule() if this_cpu is about to become
 * idle. Attempts to pull tasks from other CPUs.
 *
 * Returns:
 *   < 0 - we released the lock and there are !fair tasks present
 *     0 - failed, no new tasks
 *   > 0 - success, new (fair) tasks present
 */
static int sched_balance_newidle(struct rq *this_rq, struct rq_flags *rf)
        __must_hold(__rq_lockp(this_rq))
{
        unsigned long next_balance = jiffies + HZ; 
        int this_cpu = this_rq->cpu;
        int continue_balancing = 1;
        u64 t0, t1, curr_cost = 0;
        struct sched_domain *sd;
        int pulled_task = 0;

        update_misfit_status(NULL, this_rq);

        /*
         * There is a task waiting to run. No need to search for one.
         * Return 0; the task will be enqueued when switching to idle.
         */
        if (this_rq->ttwu_pending)
                return 0;

        /*
         * We must set idle_stamp _before_ calling sched_balance_rq()
         * for CPU_NEWLY_IDLE, such that we measure the this duration
         * as idle time.
         */
        this_rq->idle_stamp = rq_clock(this_rq);

        /*
         * Do not pull tasks towards !active CPUs...
         */
        if (!cpu_active(this_cpu))
                return 0;

        /*
         * This is OK, because current is on_cpu, which avoids it being picked
         * for load-balance and preemption/IRQs are still disabled avoiding
         * further scheduler activity on it and we're being very careful to
         * re-start the picking loop.
         */
        rq_unpin_lock(this_rq, rf);

        sd = rcu_dereference_sched_domain(this_rq->sd);
        if (!sd)
                goto out;

        if (!get_rd_overloaded(this_rq->rd) ||
            this_rq->avg_idle < sd->max_newidle_lb_cost) {

                update_next_balance(sd, &next_balance);
                goto out;
        }

        /*
         * Include sched_balance_update_blocked_averages() in the cost
         * calculation because it can be quite costly -- this ensures we skip
         * it when avg_idle gets to be very low.
         */
        t0 = sched_clock_cpu(this_cpu);
        __sched_balance_update_blocked_averages(this_rq);

        rq_modified_begin(this_rq, &fair_sched_class);
        raw_spin_rq_unlock(this_rq);

        for_each_domain(this_cpu, sd) {
                u64 domain_cost;

                update_next_balance(sd, &next_balance);

                if (this_rq->avg_idle < curr_cost + sd->max_newidle_lb_cost)
                        break;

                if (sd->flags & SD_BALANCE_NEWIDLE) {
                        unsigned int weight = 1;

                        if (sched_feat(NI_RANDOM) && sd->newidle_ratio < 1024) {
                                /*
                                 * Throw a 1k sided dice; and only run
                                 * newidle_balance according to the success
                                 * rate.
                                 */
                                u32 d1k = sched_rng() % 1024;
                                weight = 1 + sd->newidle_ratio;
                                if (d1k > weight) {
                                        update_newidle_stats(sd, 0);
                                        continue;
                                }
                                weight = (1024 + weight/2) / weight;
                        }

                        pulled_task = sched_balance_rq(this_cpu, this_rq,
                                                   sd, CPU_NEWLY_IDLE,
                                                   &continue_balancing);

                        t1 = sched_clock_cpu(this_cpu);
                        domain_cost = t1 - t0;
                        curr_cost += domain_cost;
                        t0 = t1;

                        /*
                         * Track max cost of a domain to make sure to not delay the
                         * next wakeup on the CPU.
                         */
                        update_newidle_cost(sd, domain_cost, weight * !!pulled_task);
                }

                /*
                 * Stop searching for tasks to pull if there are
                 * now runnable tasks on this rq.
                 */
                if (pulled_task || !continue_balancing)
                        break;
        }

        raw_spin_rq_lock(this_rq);

        if (curr_cost > this_rq->max_idle_balance_cost)
                this_rq->max_idle_balance_cost = curr_cost;

        /*
         * While browsing the domains, we released the rq lock, a task could
         * have been enqueued in the meantime. Since we're not going idle,
         * pretend we pulled a task.
         */
        if (this_rq->cfs.h_nr_queued && !pulled_task)
                pulled_task = 1;

        /* If a higher prio class was modified, restart the pick */
        if (rq_modified_above(this_rq, &fair_sched_class))
                pulled_task = -1;

out:
        /* Move the next balance forward */
        if (time_after(this_rq->next_balance, next_balance))
                this_rq->next_balance = next_balance;

        if (pulled_task)
                this_rq->idle_stamp = 0;
        else
                nohz_newidle_balance(this_rq);

        rq_repin_lock(this_rq, rf);

        return pulled_task;
}
```