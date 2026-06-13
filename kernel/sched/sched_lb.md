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

* `numa_distance[i, j]` 是记录 NUMA 节点距离的二维数组
  * `numa_distance_cnt` 是第一维和第二维的最大索引数
* `numa_node_dist()` 函数给出一、二维的索引 `i` 和 `j`，返回这两个节点间的距离
  * 当创建 NUMA 级别调度域时，一些特定的体系结构可以修改它的 NUMA 距离，修改 NUMA 节点组和 NUMA 的级别
```cpp
//mm/numa_memblks.c
int __node_distance(int from, int to) 
{
        if (from >= numa_distance_cnt || to >= numa_distance_cnt)
                return from == to ? LOCAL_DISTANCE : REMOTE_DISTANCE;
        return numa_distance[from * numa_distance_cnt + to];
}
//arch/x86/include/asm/topology.h
#define node_distance(a, b) __node_distance(a, b)
//kernel/sched/topology.c
/*
 * An architecture could modify its NUMA distance, to change
 * grouping of NUMA nodes and number of NUMA levels when creating
 * NUMA level sched domains.
 *
 * A NUMA level is created for each unique
 * arch_sched_node_distance.
 */     
static int numa_node_dist(int i, int j)
{
        return node_distance(i, j);
}
```
* `slit_cluster_symmetric()` 测试以节点 N 起始的 on-trace cluster 是否对称
  * 使用上三角迭代避免明显重复的比较
```cpp
/*
 * Test if the on-trace cluster at (N,N) is symmetric.
 * Uses upper triangle iteration to avoid obvious duplicates.
 *
 * @N: 当前 cluster 内第一个 NUMA 节点的索引（cluster 起始点）
 * @return: true 表示 cluster 内所有节点对的距离对称，false 表示存在不对称
 */
static bool slit_cluster_symmetric(int N)
{
        int u = topology_num_nodes_per_package(); //每个物理封装内的节点数（cluster 大小）
        //遍历 cluster 内的所有节点对（包括自身，即 k==l 时距离应为 0）
        for (int k = 0; k < u; k++) {
                //内层从 k 开始，仅检查上三角（含对角线），避免重复检查对称对
                for (int l = k; l < u; l++) {
                        // 比较正向距离和反向距离是否相等
                        if (node_distance(N + k, N + l) !=
                            node_distance(N + l, N + k))
                                return false;   //发现不对称，直接返回失败
                }
        }
        //所有节点对的距离都对称
        return true;
}
```

* `slit_cluster_package()` 检查 cluster 内所有节点的每一个 CPU 都是否拥有相同的封装 ID
```cpp
/*
 * Return the package-id of the cluster, or ~0 if indeterminate.
 * Each node in the on-trace cluster should have the same package-id.
 *
 * 返回该 cluster 的物理封装 ID（package-id），若无法确定则返回 ~0。
 * 要求 cluster 内所有节点的每一个 CPU 都拥有相同的封装 ID。
 *
 * @N: 当前 cluster 内第一个 NUMA 节点的索引
 * @return: cluster 的 package-id（一致）或 ~0（不一致/无效）
 */
static u32 slit_cluster_package(int N)
{
        int u = topology_num_nodes_per_package(); //每个物理封装内的节点数（cluster 大小）
        u32 pkg_id = ~0; //初始化为无效值（全 1）
        //遍历 cluster 内的每一个 NUMA 节点
        for (int n = 0; n < u; n++) {
                const struct cpumask *cpus = cpumask_of_node(N + n); //获取归属于该节点的所有 CPU 的掩码
                int cpu;
                // 遍历该节点上的每一个 CPU
                for_each_cpu(cpu, cpus) {
                        u32 id = topology_logical_package_id(cpu); //获取该 CPU 的逻辑封装 ID
                        //第一次设置：记录第一个遇到的封装 ID
                        if (pkg_id == ~0)
                                pkg_id = id;
                        if (pkg_id != id)      //发现不一致的封装 ID，说明该 cluster 跨越了不同的物理封装
                                return ~0;     //返回无效值
                }
        }
        //所有 CPU 的封装 ID 一致，返回该 ID
        return pkg_id;
}
```
* 验证 SLIT 表是否符合 SNC（子 NUMA 集群）所期望的格式，具体要求：
  - 每个 on-trace （即同一个物理封装内的 SNC cluster）内部距离对称
  - 每个 on-trace 拥有一个唯一的物理封装 ID（package-id）
* 如果在 SNC 之上再使用 NUMA_EMU（NUMA 仿真），后果自负。
```cpp
/*
 * Validate the SLIT table is of the form expected for SNC, specifically:
 *
 *  - each on-trace cluster should be symmetric,
 *  - each on-trace cluster should have a unique package-id.
 *
 * If you NUMA_EMU on top of SNC, you get to keep the pieces.
 */
static bool slit_validate(void)
{       //每个物理封装内的 NUMA 节点数（SNC 划分的 cluster 内节点个数）
        int u = topology_num_nodes_per_package();
        u32 pkg_id, prev_pkg_id = ~0;             //当前 cluster 的物理封装 ID 以及上一个 cluster 的 ID
        //遍历每一个物理封装（package）
        for (int pkg = 0; pkg < topology_max_packages(); pkg++) {
                int n = pkg * u;   //当前封装内第一个 NUMA 节点的索引
                //确保 on-trace cluster 内部距离对称，且每个 cluster 拥有不同的封装 ID。
                /*
                 * Ensure the on-trace cluster is symmetric and each cluster
                 * has a different package id.
                 *
                 */
                if (!slit_cluster_symmetric(n))   //检查以节点 n 起始的簇是否内部对称
                        return false;
                pkg_id = slit_cluster_package(n); //获取该簇的物理封装 ID
                if (pkg_id == ~0)                 //无效的 package ID
                        return false;
                if (pkg && pkg_id == prev_pkg_id) //不同封装却得到相同的 ID（冲突）
                        return false;
                //记录当前封装 ID，用于下一次比较
                prev_pkg_id = pkg_id;
        }
        //所有验证通过
        return true; 
}
```
* 当启用 SNC（如 Intel 的 Sub-NUMA Clustering）时，一个物理 CPU 封装内会被划分为多个 NUMA 节点组（cluster）
* 为了保证调度器的正确性，SLIT 表需要满足特定条件：
  * 每个 cluster 内部的 NUMA 节点之间的访问距离必须对称（即 `distance[i][j] == distance[j][i]）`
  * 并且每个 cluster 必须拥有唯一的物理封装标识
```cpp
/*
 * Compute a sanitized SLIT table for SNC; notably SNC-3 can end up with
 * asymmetric off-trace clusters, reflecting physical assymmetries. However
 * this leads to 'unfortunate' sched_domain configurations.
 *
 * 为 SNC（子 NUMA 集群）计算一个经过清理的 SLIT 表；尤其是 SNC-3 会导致 off-trace cluster 的不对称性，
 * 反映了物理上的不对称。然而这会导致调度域配置出现不理想的情况。
 *
 * For example dual socket GNR with SNC-3:
 * 例如双路 GNR（Granite Rapids）处理器配置 SNC-3 时的节点距离矩阵示例：
 *
 * node distances:
 * node     0    1    2    3    4    5
 *     0:   10   15   17   21   28   26
 *     1:   15   10   15   23   26   23
 *     2:   17   15   10   26   23   21
 *     3:   21   28   26   10   15   17
 *     4:   23   26   23   15   10   15
 *     5:   26   23   21   17   15   10
 *
 * Fix things up by averaging out the off-trace clusters; resulting in:
 * 通过对 off-trace clusters 的距离进行平均化处理来修正，得到如下对称的距离矩阵：
 *
 * node     0    1    2    3    4    5
 *     0:   10   15   17   24   24   24
 *     1:   15   10   15   24   24   24
 *     2:   17   15   10   24   24   24
 *     3:   24   24   24   10   15   17
 *     4:   24   24   24   15   10   15
 *     5:   24   24   24   17   15   10
 */
static int slit_cluster_distance(int i, int j)
{
        static int slit_valid = -1;   //静态标志，记录 SLIT 表是否满足 SNC 修正的条件（-1:未初始化，0:无效，1:有效）
        int u = topology_num_nodes_per_package(); //每个物理 CPU 封装内的 NUMA 节点数（即 SNC 划分的簇内节点数）
        long d = 0;                   //用于累加距离和的临时变量
        int x, y;

        if (slit_valid < 0) {         //首次调用时进行验证
                slit_valid = slit_validate(); //检查 SLIT 表是否符合 SNC 修正所需的格式（例如对称性、跨 cluster 距离不等）
                if (!slit_valid)
                        pr_err(FW_BUG "SLIT table doesn't have the expected form for SNC -- fixup disabled!\n");
                else
                        pr_info("Fixing up SNC SLIT table.\n");
        }
        //这两个节点是否属于同一个物理封装内的同一个 SNC cluster？
        /*
         * Is this a unit cluster on the trace?
         */
        if ((i / u) == (j / u) || !slit_valid)
                return node_distance(i, j);  // 同 cluster 或 SLIT 表无效时，直接返回原始距离
        //不同 cluster（Off-trace）的情况。
        /*
         * Off-trace cluster.
         *
         * Notably average out the symmetric pair of off-trace clusters to
         * ensure the resulting SLIT table is symmetric.
         * 注意：对两个 off-trace clusters 之间的对称对进行平均，以保证修正后的 SLIT 表是对称的。
         */
        x = i - (i % u);  //节点 i 所在 cluster 的起始节点编号
        y = j - (j % u);  //节点 j 所在 cluster 的起始节点编号
        //遍历 cluster x 和 cluster y 中的所有节点对（双向），累加所有距离
        for (i = x; i < x + u; i++) {
                for (j = y; j < y + u; j++) {
                        d += node_distance(i, j); //累加正向距离
                        d += node_distance(j, i); //累加反向距离（确保对称性）
                }
        }
        //返回平均距离：总距离除以节点对总数（2 * u * u，因为每个方向都加了一次）
        return d / (2*u*u);
}
```

* `arch_sched_node_distance()`是 架构相关的 NUMA 节点距离计算函数，供调度器使用
  * 该函数主要用于处理特定 Intel CPU 上 SNC（Sub-NUMA Clustering）导致的不对称距离问题 
  * 对于普通系统，直接返回 ACPI SLIT 表中的原始距离
```cpp
/*
 * @from: 源节点编号
 * @to:   目标节点编号
 *
 * 返回值：节点间的距离值，数值越小表示越“近”，调度器会优先选择近的节点。
 */
int arch_sched_node_distance(int from, int to)
{
        int d = node_distance(from, to); //获取 BIOS 提供的原始 NUMA 距离

        // 根据 CPU 型号决定是否需要修正
        switch (boot_cpu_data.x86_vfm) {
        case INTEL_GRANITERAPIDS_X:   //Intel Granite Rapids X 系列
        case INTEL_ATOM_DARKMONT_X:   //Intel Atom Darkmont X 系列
                //如果系统只有一个物理封装，或者每个封装内的节点数小于 3（即 SNC-2 或未启用 SNC-3），
                //则直接返回原始距离，因为不存在需要修正的非对称性。
                if (topology_max_packages() == 1 || 
                    topology_num_nodes_per_package() < 3) 
                        return d;
                /*
                 * Handle SNC-3 asymmetries.
                 * 处理 SNC-3 导致的不对称距离问题。
                 */
                return slit_cluster_distance(from, to); // 调用上述平均化修正函数
        }
        return d; // 其他 CPU 型号不做修正
}
```

* `sched_record_numa_dist()` 记录并提取 NUMA 节点间的所有 **唯一距离值**
  * 遍历所有在线 NUMA 节点对，收集节点间距离值，利用 bitmap 去重，最终得到一组从小到大排列的、不重复的距离值，用于构建 NUMA 调度层级。
  * 例如，典型值可能为 `[10, 20, 30]` 分别代表本地、同一 NUMA 域、跨域等。
```cpp
//include/linux/topology.h
#define DISTANCE_BITS           8
//kernel/sched/topology.c
#define NR_DISTANCE_VALUES (1 << DISTANCE_BITS)

/*
 * sched_record_numa_dist - 记录并提取 NUMA 节点间的所有唯一距离值
 * @offline_node: 需要忽略的离线节点（通常为 NUMA_NO_NODE 或特定离线节点）
 * @n_dist:       用于获取两个节点间距离的回调函数，返回整型距离值
 * @dist:         输出参数：动态分配的数组，存储所有升序排列的唯一距离值
 * @levels:       输出参数：唯一距离值的个数（即 NUMA 距离等级数）
 *
 * 返回值：0 表示成功，负错误码表示失败（-ENOMEM 或 -EINVAL）
 */
static int sched_record_numa_dist(int offline_node, int (*n_dist)(int, int),
                                  int **dist, int *levels)
{
        unsigned long *distance_map __free(bitmap) = NULL; //位图，用于标记哪些距离值已出现（内存由 __free 自动释放）
        int i, j;
        int *distances;

        /*
         * O(nr_nodes^2) de-duplicating selection sort -- in order to find the
         * unique distances in the node_distance() table.
         *
         * 采用 O(节点数^2) 的去重选择排序 —— 目的是找出 node_distance() 表中所有唯一的距离值。
         * 这里使用 bitmap 代替实际排序：因为距离值范围有限（NR_DISTANCE_VALUES 一般为 32 或 64），
         * 只需将出现的距离值在位图中对应位置标记，最后顺序扫描即可得到排序后的唯一值。
         */
        distance_map = bitmap_alloc(NR_DISTANCE_VALUES, GFP_KERNEL); //分配位图，大小等于可能距离值的总数（如 0~63）
        if (!distance_map)
                return -ENOMEM;
        //清空位图
        bitmap_zero(distance_map, NR_DISTANCE_VALUES);
        /* 遍历所有在线节点对（跳过 offline_node） */
        for_each_cpu_node_but(i, offline_node) {           //外层循环：节点 i
                for_each_cpu_node_but(j, offline_node) {   //内层循环：节点 j
                        int distance = n_dist(i, j);       //获取节点 i 与 j 之间的 NUMA 距离
                        /* 校验距离值是否在合法范围内 [LOCAL_DISTANCE, NR_DISTANCE_VALUES) */
                        if (distance < LOCAL_DISTANCE || distance >= NR_DISTANCE_VALUES) {
                                sched_numa_warn("Invalid distance value range");
                                return -EINVAL;
                        }
                        //将该距离值在位图中对应的比特位置 1（去重标记）
                        bitmap_set(distance_map, distance, 1);
                }
        }
        /*
         * We can now figure out how many unique distance values there are and
         * allocate memory accordingly.
         *
         * 现在可以知道有多少个唯一距离值，并据此分配内存。
         */
        nr_levels = bitmap_weight(distance_map, NR_DISTANCE_VALUES); //统计位图中置 1 的位数 = 唯一距离值的个数
        //为存储唯一距离值分配数组空间
        distances = kzalloc_objs(int, nr_levels);
        if (!distances)
                return -ENOMEM;
        /* 按升序遍历位图，将所有置 1 位的索引（即距离值）取出，存入 distances 数组 */
        for (i = 0, j = 0; i < nr_levels; i++, j++) {
                j = find_next_bit(distance_map, NR_DISTANCE_VALUES, j); //找到下一个置 1 的比特位
                distances[i] = j;   //将该比特位的索引（即距离值）保存
        }

        *dist = distances;   //通过输出参数返回距离数组
        *levels = nr_levels; //返回等级数量

        return 0;
}
```

* `sched_init_numa()` 初始化调度器所需的 NUMA 拓扑信息，该函数完成以下主要工作：
  1. 从 SLIT 表中记录节点间的原始 NUMA 距离，并存储唯一距离值；
  2. 若架构提供了修正后的距离函数（如 SNC 场景），则记录修正后的距离；
  3. 为每个 NUMA 距离等级构建对应的 CPU 掩码（表示距当前节点该跳数以内的所有 CPU）；
  4. 扩展默认的调度域拓扑，追加 NODE 层和 NUMA 层；
  5. 更新全局的调度域拓扑指针，使调度器能够使用 NUMA 感知的负载均衡。
```cpp
/*
 * @offline_node: 需要排除的离线节点（通常为 NUMA_NO_NODE）
 */
void sched_init_numa(int offline_node)
{
        struct sched_domain_topology_level *tl; 
        int nr_levels, nr_node_levels;
        int i, j;
        int *distances, *domain_distances;
        struct cpumask ***masks;
        //从 SLIT 表中记录原始的 NUMA 距离（使用 node_distance 函数）
        /* Record the NUMA distances from SLIT table */
        if (sched_record_numa_dist(offline_node, numa_node_dist, &distances,
                                   &nr_node_levels))
                return;  //记录失败（内存不足或距离值非法），直接返回

        /* Record modified NUMA distances for building sched domains */
        /* 
         * 若架构提供了修正后的距离函数（例如处理 SNC 不对称），
         * 则重新记录修正后的距离值，否则直接使用原始距离。
         */
        if (modified_sched_node_distance()) {
                if (sched_record_numa_dist(offline_node, arch_sched_node_distance,
                                           &domain_distances, &nr_levels)) {
                        kfree(distances);  // 修正距离记录失败，释放已分配的原始距离数组
                        return;
                }
        } else {
                domain_distances = distances;   // 无修正，直接共享原始距离数组
                nr_levels = nr_node_levels;
        }

        /* 将原始距离数组暴露给全局（供调试或其它模块使用） */
        rcu_assign_pointer(sched_numa_node_distance, distances);
        /* 记录最大的 NUMA 距离值（即最远的等级） */
        WRITE_ONCE(sched_max_numa_distance, distances[nr_node_levels - 1]); 
        /* 记录原始距离的唯一等级数量 */
        WRITE_ONCE(sched_numa_node_levels, nr_node_levels);

        /*
         * 'nr_levels' contains the number of unique distances
         *
         * The sched_domains_numa_distance[] array includes the actual distance
         * numbers.
         *
         * nr_levels 存储了唯一距离值的个数（即 NUMA 层级数）
         * sched_domains_numa_distance[] 数组存放具体的距离数值
         */

        /*
         * Here, we should temporarily reset sched_domains_numa_levels to 0.
         * If it fails to allocate memory for array sched_domains_numa_masks[][],
         * the array will contain less then 'nr_levels' members. This could be
         * dangerous when we use it to iterate array sched_domains_numa_masks[][]
         * in other functions.
         *
         * We reset it to 'nr_levels' at the end of this function.
         *
         * 这里先将 sched_domains_numa_levels 临时置为 0，以避免在后续分配失败时
         * 其他函数遍历未完全初始化的 masks 数组造成危险。在函数末尾会将其恢复为 nr_levels。
         */
        rcu_assign_pointer(sched_domains_numa_distance, domain_distances);
        sched_domains_numa_levels = 0;

        /* 为每个 NUMA 等级分配一级指针数组（每个等级对应一个节点的掩码列表） */
        masks = kzalloc(sizeof(void *) * nr_levels, GFP_KERNEL);
        if (!masks)
                return;

        /*
         * Now for each level, construct a mask per node which contains all
         * CPUs of nodes that are that many hops away from us.
         *
         * 对于每一个 NUMA 距离等级，为每个节点构造一个 cpumask，
         * 该 mask 包含所有距离当前节点 <= 该等级距离的节点上的所有 CPU。
         * 即：对于节点 j，等级 i 的 mask 包含所有满足
         *   arch_sched_node_distance(j, k) <= sched_domains_numa_distance[i]
         * 的节点 k 上的全部 CPU。
         */
        for (i = 0; i < nr_levels; i++) {
                /* 为第 i 级分配一个指针数组，长度为 nr_node_ids，每个元素指向一个 cpumask */
                masks[i] = kzalloc(nr_node_ids * sizeof(void *), GFP_KERNEL);
                if (!masks[i])
                        return;

                /* 遍历每一个在线节点（跳过 offline_node） */
                for_each_cpu_node_but(j, offline_node) {
                        struct cpumask *mask = kzalloc(cpumask_size(), GFP_KERNEL);
                        int k;

                        if (!mask)
                                return;

                        masks[i][j] = mask;   // 存储该节点 j 在第 i 级的 cpumask

                        /* 遍历所有可能的其他节点 k，将符合条件的节点 CPU 加入 mask */
                        for_each_cpu_node_but(k, offline_node) {
                                /* 调试模式：检查距离对称性（若不对称则告警） */
                                if (sched_debug() &&
                                    (arch_sched_node_distance(j, k) !=
                                     arch_sched_node_distance(k, j)))
                                        sched_numa_warn("Node-distance not symmetric");

                                /* 若节点 k 到节点 j 的距离大于当前等级的距离阈值，则跳过 */
                                if (arch_sched_node_distance(j, k) >
                                    sched_domains_numa_distance[i])
                                        continue;

                                /* 将节点 k 上的所有 CPU 并入当前 mask */
                                cpumask_or(mask, mask, cpumask_of_node(k));
                        }
                }
        }
        /* 将构造好的 masks 数组发布到全局 */
        rcu_assign_pointer(sched_domains_numa_masks, masks);

        /* Compute default topology size */
        /* 计算默认调度域拓扑的条目数（以空 mask 为终止标记） */
        for (i = 0; sched_domain_topology[i].mask; i++);

        /* 分配新的拓扑数组：原有默认条目 + NUMA 增加的 NODE 层 + (nr_levels-1) 个 NUMA 层 + 终止符 */
        tl = kzalloc((i + nr_levels + 1) *
                        sizeof(struct sched_domain_topology_level), GFP_KERNEL);
        if (!tl)
                return;

        /*
         * Copy the default topology bits..
         * 复制默认的拓扑层级（如 SMT, MC, PKG 等）
         */
        for (i = 0; sched_domain_topology[i].mask; i++)
                tl[i] = sched_domain_topology[i];

        /*
         * Add the NUMA identity distance, aka single NODE.
         * 添加 NUMA 身份距离层，即一个节点自身（对应距离为 LOCAL_DISTANCE）
         */
        tl[i++] = SDTL_INIT(sd_numa_mask, NULL, NODE);

        /*
         * .. and append 'j' levels of NUMA goodness.
         * 追加剩余的 NUMA 层级（从第 1 级到第 nr_levels-1 级）
         * 注意：第 0 级（本地距离）已经由 NODE 层覆盖，所以从 j=1 开始。
         */
        for (j = 1; j < nr_levels; i++, j++) {
                tl[i] = SDTL_INIT(sd_numa_mask, cpu_numa_flags, NUMA);
                tl[i].numa_level = j;   // 记录该层对应的 NUMA 距离等级索引
        }
        /* 此时 tl 数组最后一个元素由 kzalloc 隐式填充为 { NULL, } */

        /* 保存原有的拓扑指针，以便恢复（目前仅用于调试或热插拔重置） */
        sched_domain_topology_saved = sched_domain_topology;
        /* 将新的拓扑指针发布到全局，后续调度域构建将使用此拓扑 */
        sched_domain_topology = tl;

        /* 最后将 sched_domains_numa_levels 恢复为真实的 NUMA 层级数 */
        sched_domains_numa_levels = nr_levels;

        /* 根据 NUMA 拓扑类型（如扁平、分层等）设置全局 numa_topology_type */
        init_numa_topology_type(offline_node);
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
