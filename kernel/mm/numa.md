# NUMA

## numastat
* `numastat` 数据来自 `/sys/devices/system/node/node*/numastat`

```sh
# numastat 
                           node0           node1
numa_hit                17069127         2721282
numa_miss                      0         2947026
numa_foreign             2947026               0
interleave_hit             23894           23955
local_node              17015692         2677985
other_node                 53435         2990323
```
* `numa_hit`、`numa_miss`、`numa_foreign` 这三个计数器，用于反映进程能否从 **首选（prefer）节点** 成功分配内存
* **numa_hit**：成功从该节点分配到的内存页数。从首选节点分配内存 **成功**，增加该计数
* **numa_miss**：本意是希望从首选节点（别的节点）分配，失败后退而求其次，**实际上** 是从该节点分配的内存页数
* **numa_foreign**：从首选节点分配内存 **失败**，增加该计数。与 `numa_miss` 互为影子，每个节点的 `numa_miss` 都来自另一个节点的 `numa_foreign`
  * 比如 node1 上由于内存紧缺分配失败到 node0 上去分配的内存页，计入 node0 的 `numa_miss`，且计入 node1 的 `numa_foreign` 
* **interleave_hit**：无 NUMA 节点偏好的内存请求，此时会均匀分配自各个节点（interleave）
* **local_node**：分配给运行在同一节点的进程的内存页数
* **other_node**：跑在其他节点上的进程在本节点分配的内存页数。
* `local_node + other_node = numa_hit + numa_miss`，比如：
  * node0 上运行的进程 A 首选节点是 node0，
    * node0 上如果内存充足，在 node0 上成功分配，计入 node0 的 `numa_hit`，并且还计入 node0 的 `local_node`
    * node0 上如果内存紧缺，回退到 node1 上去分配的内存页，计入 node1 的 `numa_miss`，并且还计入 node1 的 `other_node`
  * 将 node0 上运行的进程 B 首选节点配置到 node1，B 分配内存时优先从 node1 上去分配，
    * 分配成功的内存页计入 node1 的 `numa_hit`，并且还计入 node1 的 `other_node`
    * 分配失败回退到 node0 上去分配，此时若分配成功计入 node0 的 `numa_miss`，并且还计入 node0 的 `local_node`
  
### 相关代码解释
* `numa_node_id()` 函数返回调用该函数时所在的 NUMA node
  * 如果开启了 `CONFIG_USE_PERCPU_NUMA_NODE_ID`，把每个 CPU 所属的 NUMA node 存储在 per-CPU 变量 `numa_node` 中
  * 如果没开启 `CONFIG_USE_PERCPU_NUMA_NODE_ID`，把每个 CPU 所属的 NUMA node 存储在 per-CPU 变量 `x86_cpu_to_node_map` 中，通过 `cpu_to_node(cpu)` 接口返回
* include/linux/topology.h
```cpp
#ifdef CONFIG_USE_PERCPU_NUMA_NODE_ID
DECLARE_PER_CPU(int, numa_node);

#ifndef numa_node_id
/* Returns the number of the current Node. */
static inline int numa_node_id(void)
{
        return raw_cpu_read(numa_node);
}
#endif
...
#else   /* !CONFIG_USE_PERCPU_NUMA_NODE_ID */

/* Returns the number of the current Node. */
#ifndef numa_node_id
static inline int numa_node_id(void)
{
        return cpu_to_node(raw_smp_processor_id());
}
#endif

#endif  /* [!]CONFIG_USE_PERCPU_NUMA_NODE_ID */
```
* 更新这几个计数的函数是 `zone_statistics()`
  * mm/page_alloc.c
```cpp
/*
 * Update NUMA hit/miss statistics
 */
static inline void zone_statistics(struct zone *preferred_zone, struct zone *z,
                                   long nr_account)
{
#ifdef CONFIG_NUMA
        enum numa_stat_item local_stat = NUMA_LOCAL; //默认 local_stat 是 NUMA_LOCAL

        /* skip numa counters update if numa stats is disabled */
        if (!static_branch_likely(&vm_numa_stat_key))
                return;
        //如果实际分配的 node 与进程当前所在 node 不一致，以下条件为 true，增加实际分配 node 的 other_node 计数
        if (zone_to_nid(z) != numa_node_id())
                local_stat = NUMA_OTHER;
        //如果实际分配的 node 与进程当前所在 node 一致，以上条件为 false，增加实际分配 node 的 local_node 计数
        if (zone_to_nid(z) == zone_to_nid(preferred_zone))
                //如果实际分配的 node 与进程首选分配的 node 一致，增加实际分配 node 的 numa_hit 计数
                __count_numa_events(z, NUMA_HIT, nr_account);
        else {
                //如果实际分配的 node 与进程首选分配的 node 不一致，增加实际分配 node 的 numa_miss 计数和首选分配 node 的 numa_foreign 计数
                __count_numa_events(z, NUMA_MISS, nr_account);
                __count_numa_events(preferred_zone, NUMA_FOREIGN, nr_account);
        }    
        __count_numa_events(z, local_stat, nr_account);
#endif
}
```