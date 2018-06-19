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

* kernel/sched/core.c
```c
struct sd_data {
        struct sched_domain **__percpu sd;
        struct sched_domain_shared **__percpu sds;
        struct sched_group **__percpu sg;
        struct sched_group_capacity **__percpu sgc;
};

struct sched_domain_topology_level {
        sched_domain_init_f init;
        sched_domain_mask_f mask;
        int                 flags;
        int                 numa_level;
        struct sd_data      data;
};
...
/*
 * Topology list, bottom-up.
 */
static struct sched_domain_topology_level default_topology[] = {
#ifdef CONFIG_SCHED_SMT
        { sd_init_SIBLING, cpu_smt_mask, },
#endif
#ifdef CONFIG_SCHED_MC
        { sd_init_MC, cpu_coregroup_mask, },
#endif
#ifdef CONFIG_SCHED_BOOK
        { sd_init_BOOK, cpu_book_mask, },
#endif
        { sd_init_CPU, cpu_cpu_mask, },
        { NULL, },
};

static struct sched_domain_topology_level *sched_domain_topology = default_topology;

#define for_each_sd_topology(tl)                        \
        for (tl = sched_domain_topology; tl->mask; tl++)
...**```
```
* 每个级别的调度域和调度组会在`__sdt_alloc()`用`kzalloc_node()`分配出来
* `register_sched_domain_sysctl()` 建立如下 `sysctl` 控制项
```
proc/sys/kernel/sched_domain/
├── cpu0
│   ├── domain0
│   │   ├── busy_factor
│   │   ├── busy_idx
│   │   ├── cache_nice_tries
│   │   ├── flags
│   │   ├── forkexec_idx
│   │   ├── idle_idx
│   │   ├── imbalance_pct
│   │   ├── max_interval
│   │   ├── min_interval
│   │   ├── name
│   │   ├── newidle_idx
│   │   └── wake_idx
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
