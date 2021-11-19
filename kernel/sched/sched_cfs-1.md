
## CFS调度器相关的`sched_latency_ns`、`sched_min_granularity_ns`和 `sched_wakeup_granularity_ns`参数

### 初始化
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
```sh
$ sysctl kernel.sched_tunable_scaling
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
* 简单的说，就是随着 CPU 数目的增多，调度的“有效延迟”肯定会减小，但减小的幅度却不可能是线性的。
* 可以想象，到后来即使加入更多的 CPU，调度因此而获得的收益会愈不明显，所以也就没有必要再返回更大的 factor 了。
* 另外，在 CPU 很多的情况下，如果是线性增长而得到一个较大最小调度粒度和调度延迟，也不是期望的结果，太大的最小调度粒度不利于抢占的发生。
* kernel/sched/fair.c
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

### 设置

* 而设置这几个参数则通过回调函数`sched_proc_update_handler()`修改
```c
int sched_proc_update_handler(struct ctl_table *table, int write,
                void __user *buffer, size_t *lenp,
                loff_t *ppos)
{
        int ret = proc_dointvec_minmax(table, write, buffer, lenp, ppos);
        unsigned int factor = get_update_sysctl_factor();

        if (ret || !write)
                return ret;

        sched_nr_latency = DIV_ROUND_UP(sysctl_sched_latency,
                                        sysctl_sched_min_granularity);

#define WRT_SYSCTL(name) \
        (normalized_sysctl_##name = sysctl_##name / (factor))
        WRT_SYSCTL(sched_min_granularity);
        WRT_SYSCTL(sched_latency);
        WRT_SYSCTL(sched_wakeup_granularity);
#undef WRT_SYSCTL

        return 0;
}
#endif
...__```
```
* 注意，全局变量`sched_nr_latency`除了初始化外，唯一被赋值的地方就是这里，公式为：
  ```c
  sched_nr_latency = ceil( sysctl_sched_latency / sysctl_sched_min_granularity )
  ```

### 微调
* 关于调节`sched_latency_ns`和`sched_min_granularity_ns`会对系统，或者说抢占，造成的影响，主要考察`sched_latency_ns`的计算公式和以下几个函数：
  ```c
  check_preempt_tick()
    -> sched_slice()
       -> __sched_period()
  ```
#### 计算调度周期__sched_period
* 首先是看计算 **调度周期**（也叫 *目标延迟* 或者 *调度延迟*）的函数`__sched_period()`
* 该延迟使得 CFS 不必每个 tick 都去检查是否需要调度切换，而是延迟到一定程度再去检查
* 在调度延迟这个时间片内，`cfs_rq`中的每个进程以求优先级为权重瓜分时间
```c
/*
 * The idea is to set a period in which each task runs once.
 *
 * When there are too many tasks (sched_nr_latency) we have to stretch
 * this period because otherwise the slices get too small.
 *
 * p = (nr <= nl) ? l : l*nr/nl
 */
static u64 __sched_period(unsigned long nr_running)
{
        if (unlikely(nr_running > sched_nr_latency))
                return nr_running * sysctl_sched_min_granularity;
        else
                return sysctl_sched_latency;
}
```
* `nr_running`通常是就绪队列上的进程数，由此可以看到，调度周期同时受`nr_running`,`sysctl_sched_min_granularity`,`sysctl_sched_latency`三者影响，`sched_nr_latency`是根据后二者计算出来的
* 我们将这两个分支称为 **条件1** 和 **条件2**
1. **条件1**: `nr_running > sched_nr_latency`：排队的就绪进程较多，每个进程分得的时间按权重比例划分`nr_running * sysctl_sched_min_granularity`
2. **条件2**:`nr_running < sched_nr_latency`：排队的就绪进程较少，每个进程分得的时间按权重比例划分`sysctl_sched_latency`

```c
/*
 * We calculate the wall-time slice from the period by taking a part
 * proportional to the weight.
 *
 * s = p*P[w/rw]
 */
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
        u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);

        for_each_sched_entity(se) {
                struct load_weight *load;
                struct load_weight lw;

                cfs_rq = cfs_rq_of(se);
                load = &cfs_rq->load;

                if (unlikely(!se->on_rq)) {
                        lw = cfs_rq->load;

                        update_load_add(&lw, se->load.weight);
                        load = &lw;
                }    
                slice = __calc_delta(slice, se->load.weight, load);
        }    
        return slice;
}
...*```
```
* `sched_slice()`是按权重比例划分调度周期为时间片的过程
* 注意，这里的时间片是墙上时间（实际时间），不是虚拟时间
* `check_preempt_tick()`详见 [周期性调度检查check_preempt_tick](#sched_cfs.md#周期性调度检查check_preempt_tick)

#### `sysctl_sched_latency`不变，只减小`sysctl_sched_min_granularity`
* 根据公式，`sched_nr_latency`会比较大，因此容易进入 **条件2**
* 有可能用更长的调度周期`sysctl_sched_latency`，而不是`nr_running * sysctl_sched_min_granularity`
* 虽然调度最小粒度比较小，但比改动前的理想运行时间会更长
* 在`check_preempt_tick()`时会因`if (delta_exec > ideal_runtime)` 不易达成变得不易被抢占
#### `sysctl_sched_min_granularity`不变，只减小`sysctl_sched_latency`
* 根据公式，`sched_nr_latency`会比较小，因此容易进入 **条件1**
* 以调度最小粒度换算后进行调度，即`ideal_runtime`会变小
* 由于调度周期变短了，因此`check_preempt_tick()`的以下条件会比较容易达成，易于发生抢占
  ```c
  if (delta_exec > ideal_runtime) {
      resched_curr(rq_of(cfs_rq));
      ...
      return;
  }
  ```
* 在`check_preempt_tick()`时，即使之前那个条件未达成，以下检查条件与之前相当，对抢占没影响
  ```c
  if (delta_exec < sysctl_sched_min_granularity)
      return;
  ```
* 但由于调度周期变短，以下条件会容易达成，易于抢占
  ```c
  if (delta > ideal_runtime)
      resched_curr(rq_of(cfs_rq));
  ```
#### 同时减小`sysctl_sched_latency`和`sysctl_sched_min_granularity`
* 根据公式，`sched_nr_latency`不变，因此容易进入 **条件1** 和 **条件2** 的机会和调整之前相当
* 然而，不论是进入哪个条件，调度周期都变短了，见`__sched_period()`
* 因此，理想运行时间也变短了，见`sched_slice()`
* 在`check_preempt_tick()`时，
  * `if (delta_exec > ideal_runtime)` 容易达成，易被抢占
  * `if (delta_exec < sysctl_sched_min_granularity)` 不易达成，易被抢占
  * `if (delta > ideal_runtime)` 容易达成，易被抢占
