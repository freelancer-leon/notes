# CPU 负载均衡

```c
start_kernel()
  -> rest_init()
       -> kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);

kernel_init()
  -> kernel_init_freeable()
     -> smp_init()
        -> idle_threads_init()
     -> sched_init_smp()
        -> sched_init_numa()
        -> init_sched_domains(cpu_active_mask)
           -> alloc_sched_domains(ndoms_cur = 1)
           -> build_sched_domains(doms_cur[0], NULL)
              -> __visit_domain_allocation_hell()
                 -> __sdt_alloc(cpu_map)
           -> register_sched_domain_sysctl()
  -> run_init_process()
```
* arch/x86/include/asm/topology.h
```cpp
/* Topology information */
enum x86_topology_domains {
        TOPO_SMT_DOMAIN,
        TOPO_CORE_DOMAIN,
        TOPO_MODULE_DOMAIN,
        TOPO_TILE_DOMAIN,
        TOPO_DIE_DOMAIN,
        TOPO_DIEGRP_DOMAIN,
        TOPO_PKG_DOMAIN,
        TOPO_MAX_DOMAIN,
};
```

* include/linux/sched/topology.h
```cpp
struct sd_data {
        struct sched_domain *__percpu *sd;
        struct sched_group *__percpu *sg;
        struct sched_group_capacity *__percpu *sgc;
};

struct sched_domain_topology_level {
        sched_domain_mask_f mask;
        sched_domain_flags_f sd_flags;
        int                 numa_level;
        struct sd_data      data;
        char                *name;
};
...
#define SDTL_INIT(maskfn, flagsfn, dname) ((struct sched_domain_topology_level) \
            { .mask = maskfn, .sd_flags = flagsfn, .name = #dname })
```

* 缺省 domain level 的定义 `default_topology[]`
  * kernel/sched/core.c
```cpp
/*
 * Topology list, bottom-up.
 */
static struct sched_domain_topology_level default_topology[] = {
#ifdef CONFIG_SCHED_SMT
        SDTL_INIT(tl_smt_mask, cpu_smt_flags, SMT),
#endif

#ifdef CONFIG_SCHED_CLUSTER
        SDTL_INIT(tl_cls_mask, cpu_cluster_flags, CLS),
#endif

#ifdef CONFIG_SCHED_MC
        SDTL_INIT(tl_mc_mask, cpu_core_flags, MC),
#endif
        SDTL_INIT(tl_pkg_mask, NULL, PKG),
        { NULL, },
};

static struct sched_domain_topology_level *sched_domain_topology =
        default_topology;
static struct sched_domain_topology_level *sched_domain_topology_saved;

#define for_each_sd_topology(tl)                        \
        for (tl = sched_domain_topology; tl->mask; tl++)
```
* 每个级别的调度域和调度组会在 `__sdt_alloc()` 用 `kzalloc_node()` 分配出来

### 调度级别
#### SMT
* `SMT` 对应到 **Core** 一级的调度
```cpp
//include/linux/topology.h
#if defined(CONFIG_SCHED_SMT) && !defined(cpu_smt_mask)
static inline const struct cpumask *cpu_smt_mask(int cpu)
{
        return topology_sibling_cpumask(cpu);
}
#endif
//kernel/sched/topology.c
const struct cpumask *tl_smt_mask(struct sched_domain_topology_level *tl, int cpu) 
{
        return cpu_smt_mask(cpu);
}
```

#### Cluster
* 由 `cpu_l2c_shared_mask()` 可知，`Cluster` 对应的是共享 L2 cache 一级的调度
* `Cluster` 一级在 x86 平台中没有，在一些 ARM 处理器上会出现
```cpp
//arch/x86/kernel/smpboot.c
const struct cpumask *cpu_clustergroup_mask(int cpu)
{               
        return cpu_l2c_shared_mask(cpu);
}

//kernel/sched/topology.c
const struct cpumask *tl_mc_mask(struct sched_domain_topology_level *tl, int cpu)
{
        return cpu_coregroup_mask(cpu);
}
```

#### MC
* 由 `cpu_llc_shared_mask()` 可知，这一级对应的是共享 LLC cache 一级的调度
* 对于 AMD，是 CCX（core compute complex）一级
```cpp
//arch/x86/kernel/smpboot.c
const struct cpumask *cpu_coregroup_mask(int cpu)
{
        return cpu_llc_shared_mask(cpu);
}

//kernel/sched/topology.c
const struct cpumask *tl_mc_mask(struct sched_domain_topology_level *tl, int cpu)
{
        return cpu_coregroup_mask(cpu);
}
```

#### Node

#### NUMA

#### Package

```cpp
//include/linux/topology.h
static inline const struct cpumask *cpu_node_mask(int cpu)
{
        return cpumask_of_node(cpu_to_node(cpu));
}
//kernel/sched/topology.c
const struct cpumask *tl_pkg_mask(struct sched_domain_topology_level *tl, int cpu)
{
        return cpu_node_mask(cpu);
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
