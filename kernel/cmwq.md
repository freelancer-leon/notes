# CMWQ（Concurrency Managed Workqueue）

## 工作队列涉及的几个角色
* 粗略地说，工作队列中有以下几个角色：
* 一个 `struct workqueue_struct` 实例定义了一个工作队列
* 对于某工作队列 `struct workqueue_struct` 来说，会对应到多个 CPU 相关的工作队列池 `struct pool_workqueue`
* 一个工作队列池 `struct pool_workqueue` 上排着多项工作 `struct work_struct`
* 一个工作队列池 `struct pool_workqueue` 又关联到 CPU 对应的公用工作者池 `struct worker_pool`
* 工作者池 `struct worker_pool` 是工作者 `struct worker` 的集合
* 工作者 `struct worker` 的本质上是内核线程的，负责执行工作 `struct work_struct`

## 工作者池的分类
* 按照运行特性，工作者池主要分为 CPU bound 和 unbound 两类

### CPU bound worker pool
* 绑定特定 CPU，其管理的 worker 都运行在该 CPU 上
* 根据优先级分为 normal pool 和 high priority pool，后者管理高优先级的 worker
* Linux 会为每个 online CPU 都创建 1 个 normal pool 和 1 个 high priority pool，并在命名上进行区分
  * 比如 `[kworker/1:3]` 表示 CPU 1 上 normal pool 的第 3 个 worker ，而 `[kworker/2:0H]` 表示 CPU 2 上 high priority pool 的第 0 个 worker

### CPU unbound worker pool
* 其管理的 worker 可以运行在任意的 CPU 上
* 比如 `[kworker/u32:2]` 表示 unbound pool 32 的第 2 个 worker 进程
* unbound 类型的 workqueue 的属性 `struct workqueue_attrs` 可以修改，而 bound 的则不行
* unbound 类型的 workqueue 会有与 CPU 数量相当的 `pwq` 加一个缺省的 `pwq`，由以下 commit 引入：
  * [commit 636b927eba5b ("workqueue: Make unbound workqueues to use per-cpu pool_workqueues")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=636b927eba5bc633753f8eb80f35e1d5be806e51)
  * 在此之前，`pwq` 的数量与 NUMA 的数量相关，现 `struct workqueue_struct.numa_pwq_tbl[]` 域已被删除
  * 这会带来一个问题：一个 unbound workqueue 由原来的对应到 `N` numa_node 变为 `M+1` cpu 个 `pool_workqueue`，本意是想提高 unbound workqueue 的 thoughtput，但副作用是在 CPU core 很多的机器上 flush 该 workqueue 时获取锁的次数剧增，因为 `flush_workqueue_prep_pwqs()` 函数是以 `pwq` 的个数（`for_each_pwq(pwq, wq)`）争用 `pwq->pool->lock` 这个自旋锁（即 `struct workqueue_struct *wq` 的 `struct pool_workqueue *pwq` 的 `struct worker_pool *pool` 的 `raw_spinlock_t lock`），于是获取锁的次数由 `N` 增加到 `M+1`，在有的机器上会导致性能下降
    * 注意：这个锁不是标准的公用的 per-CPU 的工作者池 `struct worker_pool` 的锁，因为工作队列是 unbound 类型的，工作者池是根据属性计算出的哈希值按需分配的。而每个工作队列的 per-CPU 的工作队列池 `struct pool_workqueue` 并没有独立的锁
    * 可以将 `struct workqueue_attrs` 的 `.order` 属性导出，例如 `/sys/devices/virtual/workqueue/xfs-cil!ram[n]/ordered`，控制其顺序处理工作。这样内核在创建工作队列时 `struct pool_workqueue` 就只会有一个缺省的 `dfl_pwq`，而其余的 Per-CPU 的 `pool_workqueue` 会全指向这个缺省的 `dfl_pwq`，牺牲了吞吐量，保证了工作的顺序性
* 对应的路径如下：
```c
alloc_workqueue()
-> __alloc_workqueue()
      if (flags & WQ_UNBOUND)
      -> wq->unbound_attrs = alloc_workqueue_attrs()
            if (wq->flags & __WQ_ORDERED)
            -> apply_workqueue_attrs_locked(wq, ordered_wq_attrs[highpri])
               -> apply_wqattrs_prepare(wq, attrs, wq_unbound_cpumask)
                  -> wq_calc_pod_cpumask(new_attrs, cpu) //计算和产生工作队列的 cpumask，会改变一些属性
                  -> alloc_unbound_pwq() //根据 odered 属性确定是只分配一个还是 M+1 cpu 个工作队列池
                     -> pool = get_unbound_pool(attrs) //获取工作队列池对应的工作者池
                        -> u32 hash = wqattrs_hash(attrs) //根据 struct workqueue_attrs 属性计算出哈希
                        -> hash_for_each_possible(unbound_pool_hash, pool, hash_node, hash)
                             if (wqattrs_equal(pool->attrs, attrs)) //比较 hash，查找 unbound_pool_hash 里是否已经有工作者池
                                 return pool; //返回工作者池
                        -> pool = kzalloc_node(sizeof(*pool), GFP_KERNEL, node) //如果找不到则分配一个工作者池
                        -> init_worker_pool(pool)
                           -> raw_spin_lock_init(&pool->lock) //初始化工作者池锁
                        -> create_worker(pool) //创建工作者
                           -> worker = alloc_worker(pool->node) //分配工作者
                           -> format_worker_id(id_buf, sizeof(id_buf), worker, pool) //格式化 kworker 的名字
                              -> scnprintf(buf, size, "kworker/u%d:%d", pool->id, worker->id) //对于 unbound 类型
                           -> worker->task = kthread_create_on_node(worker_thread, worker, pool->node, "%s", id_buf) //创建 kworker 内核线程
                           -> kthread_bind_mask(worker->task, pool_allowed_cpus(pool)) //kworker 绑定在 pool 指定的 CPU 上
                           -> worker_attach_to_pool(worker, pool) //把工作者 attach 在工作者池上
                        -> hash_add(unbound_pool_hash, &pool->hash_node, hash) //工作者池加入 unbound_pool_hash
                     -> pwq = kmem_cache_alloc_node(pwq_cache, GFP_KERNEL, pool->node)
                     -> init_pwq(pwq, wq, pool)
                           pwq->pool = pool; //把分配的工作者池与工作队列池关联起来
```
* kernel/workqueue.c
```cpp
/* allocate the attrs and pwqs for later installation */
static struct apply_wqattrs_ctx * apply_wqattrs_prepare(struct workqueue_struct *wq,
                      const struct workqueue_attrs *attrs,
                      const cpumask_var_t unbound_cpumask)
{
...
        //分配一个缺省的 pool_workqueue
        ctx->dfl_pwq = alloc_unbound_pwq(wq, new_attrs);
...
        //如果没设置 ordered 属性，则分配 CPU 个数对应的 pool_workqueue；否则仅链接到缺省的 pool_workqueue
        for_each_possible_cpu(cpu) {
                if (new_attrs->ordered) {
                        ctx->dfl_pwq->refcnt++;
                        ctx->pwq_tbl[cpu] = ctx->dfl_pwq;
                } else {
                        wq_calc_pod_cpumask(new_attrs, cpu);
                        ctx->pwq_tbl[cpu] = alloc_unbound_pwq(wq, new_attrs);
                        if (!ctx->pwq_tbl[cpu])
                                goto out_free;
                }
        }
...
}
```

## 工作队列相关结构体的定义

### 工作队列 `struct workqueue_struct`
```c
/*
 * The externally visible workqueue.  It relays the issued work items to
 * the appropriate worker_pool through its pool_workqueues.
 */
struct workqueue_struct {
    struct list_head    pwqs;       /* WR: all pwqs of this wq */
    struct list_head    list;       /* PR: list of all workqueues */
    ...
    struct workqueue_attrs  *unbound_attrs; /* PW: only for unbound wqs */
    struct pool_workqueue __rcu *dfl_pwq;   /* PW: only for unbound wqs */
    ...
    struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
    ...
};
```
### 工作队列池 `struct pool_workqueue`
```c
/*
 * The per-pool workqueue.  While queued, the lower WORK_STRUCT_FLAG_BITS
 * of work_struct->data are used for flags and the remaining high bits
 * point to the pwq; thus, pwqs need to be aligned at two's power of the
 * number of flag bits.
 */
struct pool_workqueue {
    struct worker_pool  *pool;      /* I: the associated pool */
    struct workqueue_struct *wq;        /* I: the owning workqueue */
    ...
    struct list_head    pwqs_node;  /* WR: node on wq->pwqs */
    ...
} __aligned(1 << WORK_STRUCT_FLAG_BITS);
```
### 工作者池 `struct worker_pool`
```c
struct worker_pool {
    spinlock_t      lock;       /* the pool lock */
    int         cpu;        /* I: the associated cpu */
    int         node;       /* I: the associated node ID */
    int         id;     /* I: pool ID */
    ...
    struct list_head    worklist;   /* L: list of pending works */
    int         nr_workers; /* L: total number of workers */
    ...
    /* see manage_workers() for details on the two manager mutexes */
    struct mutex        manager_arb;    /* manager arbitration */
    struct worker       *manager;   /* L: purely informational */
    struct mutex        attach_mutex;   /* attach/detach exclusion */
    struct list_head    workers;    /* A: attached workers */
    ...
    int         refcnt;     /* PL: refcnt for unbound pools */

    /*
     * The current concurrency level.  As it's likely to be accessed
     * from other CPUs during try_to_wake_up(), put it in a separate
     * cacheline.
     */
    atomic_t        nr_running ____cacheline_aligned_in_smp;
    ...
} ____cacheline_aligned_in_smp;
```
* 系统默认有两个标准的 per-CPU `struct worker_pool` 工作者池
  * `bh_worker_pools` 用于下半部的工作
  * `cpu_worker_pools` 用于 per-CPU 的工作
  * kernel/workqueue.c
```cpp
/* the BH worker pools */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS], bh_worker_pools);

/* the per-cpu worker pools */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS], cpu_worker_pools);
```
* 在以下路径被初始化
```cpp
start_kernel()
-> workqueue_init_early()
      /* initialize BH and CPU pools */
   -> for_each_possible_cpu(cpu) {
              struct worker_pool *pool;
              i = 0;
              for_each_bh_worker_pool(pool, cpu) {
                      init_cpu_worker_pool(pool, cpu, std_nice[i]);
                      pool->flags |= POOL_BH;
                      init_irq_work(bh_pool_irq_work(pool), irq_work_fns[i]);
                      i++;
              }
              i = 0;
              for_each_cpu_worker_pool(pool, cpu)
                      init_cpu_worker_pool(pool, cpu, std_nice[i++]);
      }
```
* 此外，对于 unbounded 的工作队列会动态分配工作者池，路径在上面已经展示过了
  * 需要注意一点的是，unbounded 的工作队列的工作者池不是 per-CPU 的，而是按需分配的

### 工作者 `struct worker`
```c
/*
 * The poor guys doing the actual heavy lifting.  All on-duty workers are
 * either serving the manager role, on idle list or on busy hash.  For
 * details on the locking annotation (L, I, X...), refer to workqueue.c.
 *
 * Only to be used in workqueue and async.
 */
struct worker {
    /* on idle list while idle, on busy hash table while busy */
    union {
        struct list_head    entry;  /* L: while idle */
        struct hlist_node   hentry; /* L: while busy */
    };

    struct work_struct  *current_work;  /* K: work being processed and its */
    work_func_t     current_func;   /* K: function */
    struct pool_workqueue   *current_pwq;   /* K: pwq */
    u64         current_at; /* K: runtime at start or last wakeup */
    unsigned int        current_color;  /* K: color */

    int         sleeping;   /* S: is worker sleeping? */

    /* used by the scheduler to determine a worker's last known identity */
    work_func_t     last_func;  /* K: last work's fn */

    struct list_head    scheduled;  /* L: scheduled works */

    struct task_struct  *task;      /* I: worker task */
    struct worker_pool  *pool;      /* A: the associated pool */
                        /* L: for rescuers */
    struct list_head    node;       /* A: anchored at pool->workers */
                        /* A: runs through worker->node */

    unsigned long       last_active;    /* K: last active timestamp */
    unsigned int        flags;      /* L: flags */
    int         id;     /* I: worker id */

    /*
     * Opaque string set with work_set_desc().  Printed out with task
     * dump for debugging - WARN, BUG, panic or sysrq.
     */
    char            desc[WORKER_DESC_LEN];

    /* used only by rescuers to point to the target workqueue */
    struct workqueue_struct *rescue_wq; /* I: the workqueue to rescue */
};
```

### 工作项 `struct work_struct`
```c
struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func;
#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;
#endif
};
```

## 使用工作队列
1. 创建推后的工作
* 静态创建
   ```c
   DECLARE_WORK(name, void (*func)(void *), void *data);
   ```
* 动态创建
   ```c
   INIT_WORK(struct work_struct *work, void (*func)(void *), void *data);
   ```
2. 工作队列 handler
   ```c
   void work_handler(void *data)
   ```
* 注意：工作队列处理函数不能访问用户空间。因为它是在内核线程中执行的，而内核线程在用户空间没有相关的内存映射。
  * Q：什么时候内核可以访问用户空间？
  * A： 通常发生系统调用时，内核会代表用户空间的进程运行，此时会映射用户空间的内存。
3. 对工作进行调度
   ```c
   schedule_work(&work);
   schedule_delayed_work(&work, delay);
   ```
4. 刷新工作队列
   ```c
   void flush_scheduled_work(void);
   ```
* 请停止使用该接口，在不久的将来该函数会被移除
* 该函数会一直等待，直到队列中所有队列中所有对象都被执行完后才返回。
* 等待过程中该函数会休眠，所以只能在进程上下文中使用。
* 该函数不取消任何延迟执行的工作。
* 取消延迟工作应调用：
   ```c
   int cancel_delayed_work(struct work_struct *work);
   ```
5. 创建新的工作队列
* 创建
  ```c
  struct workqueue_struct *create_workqueue(const char *name);
  ```
* 调度
  ```c
  int queue_work(struct workqueue_struct *wq, struct work_struct *work);
  int queue_delayed_work(struct workqueue_struct *wq,
        struct work_struct *work,
        unsigned long delay);
  ```
* 刷新
  ```c
  void flush_workqueue(struct workqueue_struct *wq);
  ```

## 系统接口
* `/sys/devices/virtual/workqueue` 目录下可以看到系统中的工作队列，每个工作队列对应一个子目录，通过其下面的文件可以观察和对工作队列进行一些调整

## References
* [Linux Workqueue 机制分析 - 博客 - binsite](https://www.binss.me/blog/analysis-of-linux-workqueue/)
* [Linux Kernel cmwq 介绍](https://daybreakgx.github.io/2016/09/16/Linux_workqueue_generic/)
* [Linux Kernel cmwq 代码分析](https://daybreakgx.github.io/2016/09/21/Linux_workqueue_code_analysis/)
* [Linux Workqueue - 魅族内核团队](https://kernel.meizu.com/2016/08/21//linux-workqueue.html/)
* [内核工作队列修炼日记3-CMWQ](https://www.modb.pro/db/236435)
* [linux-workqueues-1.pdf](https://kernelmeetup.wordpress.com/wp-content/uploads/2023/11/linux-workqueues-1.pdf)