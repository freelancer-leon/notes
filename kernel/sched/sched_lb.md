# CPU 负载均衡

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
