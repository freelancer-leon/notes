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
  * 在此之前，`pwq` 的数量与 NUMA 的数量相关

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

# References
* [Linux Workqueue 机制分析 - 博客 - binsite](https://www.binss.me/blog/analysis-of-linux-workqueue/)
* [Linux Kernel cmwq 介绍](https://daybreakgx.github.io/2016/09/16/Linux_workqueue_generic/)
* [Linux Kernel cmwq 代码分析](https://daybreakgx.github.io/2016/09/21/Linux_workqueue_code_analysis/)
* [Linux Workqueue - 魅族内核团队](https://kernel.meizu.com/2016/08/21//linux-workqueue.html/)
* [内核工作队列修炼日记3-CMWQ](https://www.modb.pro/db/236435)
* [linux-workqueues-1.pdf](https://kernelmeetup.wordpress.com/wp-content/uploads/2023/11/linux-workqueues-1.pdf)