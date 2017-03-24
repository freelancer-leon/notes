
### CFS调度器相关的`sched_latency_ns`、`sched_min_granularity_ns` 和 `sched_wakeup_granularity_ns`

* 与 CFS 调度器工作密切相关的几个参数值在不同机器上看到的值可能会不一样，但是看内核源代码，内核的`.config`，`/etc/sysctl.conf`，`/etc/sysctl.d/*`以及其他能想到会配置`sysctl`的地方却找不到有哪里修改了它们。
* 这几个变量在内核中的缺省值：
* kernel/sched/fair.c
```c
unsigned int sysctl_sched_latency = 6000000ULL;
unsigned int normalized_sysctl_sched_latency = 6000000ULL;

unsigned int sysctl_sched_min_granularity = 750000ULL;
unsigned int normalized_sysctl_sched_min_granularity = 750000ULL;

unsigned int sysctl_sched_wakeup_granularity = 1000000UL;
unsigned int normalized_sysctl_sched_wakeup_granularity = 1000000UL;
```

* 但请注意另一个参数 `sysctl_sched_tunable_scaling`
  * Online CPU 的数目和这个参数的值最终决定了`sched_latency_ns`、`sched_min_granularity_ns` 和 `sched_wakeup_granularity_ns`会是多少。
  * 它们会作为计算这些值的公式的因子。
```c
/*
 * The initial- and re-scaling of tunables is configurable
 * (default SCHED_TUNABLESCALING_LOG = *(1+ilog(ncpus))
 *
 * Options are:
 * SCHED_TUNABLESCALING_NONE - unscaled, always *1
 * SCHED_TUNABLESCALING_LOG - scaled logarithmical, *1+ilog(ncpus)
 * SCHED_TUNABLESCALING_LINEAR - scaled linear, *ncpus
 */
enum sched_tunable_scaling sysctl_sched_tunable_scaling
        = SCHED_TUNABLESCALING_LOG;
```

* 你可以通过`sysctl`命令查看`sysctl_sched_tunable_scaling`的值
```
> sysctl kernel.sched_tunable_scaling
kernel.sched_tunable_scaling = 1
```

* 绝大部分系统采用缺省值`SCHED_TUNABLESCALING_LOG` = 1，采用如下公式：
```
sched_latency_ns = normalized_sysctl_sched_latency * (1 + log(ncpus))
```

例如，在 8 核 CPU 的机器上，`1 + log(8) = 4`，所以会有如下值：
```
kernel.sched_latency_ns = 24000000
kernel.sched_min_granularity_ns = 3000000
kernel.sched_tunable_scaling = 1
kernel.sched_wakeup_granularity_ns = 4000000
```

* 为什么不采用固定值，或者随着 CPU 数目线性增长，而是将`log(ncpus)`作为因子？
* 简单的说，就是随着 CPU 性能的增多，调度的“有效延迟”肯定会减小，但减小的幅度却不可能是线性的。
* 可以想象，到后来即使加入更多的 CPU，调度因此而获得的收益会愈不明显，所以也就没有必要再返回更大的 factor 了。
```c
/*
 * Increase the granularity value when there are more CPUs,
 * because with more CPUs the 'effective latency' as visible
 * to users decreases. But the relationship is not linear,
 * so pick a second-best guess by going with the log2 of the
 * number of CPUs.
 *
 * This idea comes from the SD scheduler of Con Kolivas:
 */
static int get_update_sysctl_factor(void)
{
        unsigned int cpus = min_t(int, num_online_cpus(), 8);
        unsigned int factor;

        switch (sysctl_sched_tunable_scaling) {
        case SCHED_TUNABLESCALING_NONE:
                factor = 1;
                break;
        case SCHED_TUNABLESCALING_LINEAR:
                factor = cpus;
                break;
        case SCHED_TUNABLESCALING_LOG:
        default:
                factor = 1 + ilog2(cpus);
                break;
        }

        return factor;
}

static void update_sysctl(void)
{
        unsigned int factor = get_update_sysctl_factor();

#define SET_SYSCTL(name) \
        (sysctl_##name = (factor) * normalized_sysctl_##name)
        SET_SYSCTL(sched_min_granularity);
        SET_SYSCTL(sched_latency);
        SET_SYSCTL(sched_wakeup_granularity);
#undef SET_SYSCTL
}
```

函数`get_update_sysctl_factor()`会在系统启动时被调用，调用栈大致如下：
```c
start_kernel()
  -> rest_init()
    -> kernel_thread(kernel_init, NULL, CLONE_FS)
                      +---> kernel_init()
                            -> kernel_init_freeable()
                               -> sched_init_smp()
                                  -> sched_init_granularity()
                                     -> update_sysctl()
                                       -> get_update_sysctl_factor()
```
