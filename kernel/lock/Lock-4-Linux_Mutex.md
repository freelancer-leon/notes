# Linux Mutex 的实现

## mutex 结构体及初始化

### mutex 结构体

* include/linux/mutex.h

```c
/*
 * Simple, straightforward mutexes with strict semantics:
 *
 * - only one task can hold the mutex at a time
 * - only the owner can unlock the mutex
 * - multiple unlocks are not permitted
 * - recursive locking is not permitted
 * - a mutex object must be initialized via the API
 * - a mutex object must not be initialized via memset or copying
 * - task may not exit with mutex held
 * - memory areas where held locks reside must not be freed
 * - held mutexes must not be reinitialized
 * - mutexes may not be used in hardware or software interrupt
 *   contexts such as tasklets and timers
 *
 * These semantics are fully enforced when DEBUG_MUTEXES is
 * enabled. Furthermore, besides enforcing the above rules, the mutex
 * debugging code also implements a number of additional features
 * that make lock debugging easier and faster:
 *
 * - uses symbolic names of mutexes, whenever they are printed in debug output
 * - point-of-acquire tracking, symbolic lookup of function names
 * - list of all locks held in the system, printout of them
 * - owner tracking
 * - detects self-recursing locks and prints out all relevant info
 * - detects multi-task circular deadlocks and prints out all affected
 *   locks and tasks (and only those tasks)
 */
struct mutex {
    atomic_long_t       owner;
    raw_spinlock_t      wait_lock;
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    struct optimistic_spin_queue osq; /* Spinner MCS lock */
#endif
    struct list_head    wait_list;
#ifdef CONFIG_DEBUG_MUTEXES
    void            *magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
#endif
};
```

* `owner`被划分为两个域
  * task field：`63 bit`至`3 bit`存放锁的持有者的`task_struct`地址
  * flags field：`2 bit`至`0 bit`存放锁的状态，其中
    * `bit 0`表示`wait_list`是否为空；不为`0`时解锁需唤醒等待者
    * `bit 1`表示释放锁的时候需要把锁交给第一等待者
    * `bit 2`表示 handoff 已经完成，等待着`task`域的进程来占用锁

```c
/*
 * @owner: contains: 'struct task_struct *' to the current lock owner,
 * NULL means not owned. Since task_struct pointers are aligned at
 * at least L1_CACHE_BYTES, we have low bits to store extra state.
 *
 * Bit0 indicates a non-empty waiter list; unlock must issue a wakeup.
 * Bit1 indicates unlock needs to hand the lock to the top-waiter
 * Bit2 indicates handoff has been done and we're waiting for pickup.
 */
#define MUTEX_FLAG_WAITERS  0x01
#define MUTEX_FLAG_HANDOFF  0x02
#define MUTEX_FLAG_PICKUP   0x04

#define MUTEX_FLAGS     0x07
```

#### 什么是乐观自旋？

> 举例，不带乐观自旋的 mutex 版本中，若 A 线程持有锁且正在运行，此时 B 线程想拿锁，发现锁正被 A 拿着，B 将立即进入睡眠状态，在 A 线程释放锁后再去唤醒 B 线程来拿锁。但是我们通常认为持有锁的 A 若是在运行的情况下，通常会很快的退出临界区并释放锁，那么 B 去睡眠则是不必要的，毕竟睡眠和唤醒也是有代价的，完全可以多等一会即可。那么当 mutex 开启乐观自旋的 feature 后，若 B 拿不到锁，且发现锁的持有者 A 仍占用着 CPU 运行时，则不再去睡眠，而是像自旋锁一样进行自旋等待，直到 A 释放锁，但期间若 A 失去 CPU（即`p->on_cpu`为`0`），那么 B 将不会继续傻傻的自旋等待，而是进入睡眠。

#### 什么是 handoff 机制？

> 当 mutex 支持乐观自旋时，那么会存在这样的情况：`wait_list`中有一些进程在睡眠并等待被唤醒拿锁，同时还有一些进程不在`wait_list`中且不断的自旋等锁或乐观自旋等锁。`wait_list`上的进程大概率是抢不过自旋拿锁的进程的，这是因为调度延时的存在。当被唤醒的进程真正获得 CPU 时，锁早就被自旋等锁的进程给偷走了(偷锁)，这会在一定程度的造成`wait_list`的等待者长时间“饥饿”。

### mutex 结构体初始化
* 非实时抢占版本的`mutex_init()`
* include/linux/mutex.h
```c
#define mutex_init(mutex)               \
do {                            \
    static struct lock_class_key __key;     \
                            \
    __mutex_init((mutex), #mutex, &__key);      \
} while (0)
```
* kernel/locking/mutex.c
```c
void
__mutex_init(struct mutex *lock, const char *name, struct lock_class_key *key)
{
    atomic_long_set(&lock->owner, 0);
    raw_spin_lock_init(&lock->wait_lock);
    INIT_LIST_HEAD(&lock->wait_list);
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    osq_lock_init(&lock->osq);
#endif

    debug_mutex_init(lock, name, key);
}
EXPORT_SYMBOL(__mutex_init);
```

## mutex 加锁
* Mutex 必须由给它加锁的同一个进程释放，**递归加锁是不允许的**
* include/linux/mutex.h

```c
/**
 * mutex_lock - acquire the mutex
 * @lock: the mutex to be acquired
 *
 * Lock the mutex exclusively for this task. If the mutex is not
 * available right now, it will sleep until it can get it.
 *
 * The mutex must later on be released by the same task that
 * acquired it. Recursive locking is not allowed. The task
 * may not exit without first unlocking the mutex. Also, kernel
 * memory where the mutex resides must not be freed with
 * the mutex still locked. The mutex must first be initialized
 * (or statically defined) before it can be locked. memset()-ing
 * the mutex to 0 is not allowed.
 *
 * (The CONFIG_DEBUG_MUTEXES .config option turns on debugging
 * checks that will enforce the restrictions and will also do
 * deadlock debugging)
 *
 * This function is similar to (but not equivalent to) down().
 */
void __sched mutex_lock(struct mutex *lock)
{
    might_sleep(); // 如果在原子上下文，运行到这里会给出告警
    // 如果没人持锁，这里走的是快速路径；否则走的慢速路径
    if (!__mutex_trylock_fast(lock))
        __mutex_lock_slowpath(lock);
}
```

### 加锁的快速路径
* kernel/locking/mutex.c

```c
/*
 * Lockdep annotations are contained to the slow paths for simplicity.
 * There is nothing that would stop spreading the lockdep annotations outwards
 * except more code.
 */

/*
 * Optimistic trylock that only works in the uncontended case. Make sure to
 * follow with a __mutex_trylock() before failing.
 */
static __always_inline bool __mutex_trylock_fast(struct mutex *lock)
{
    unsigned long curr = (unsigned long)current;
    unsigned long zero = 0UL;

    if (atomic_long_try_cmpxchg_acquire(&lock->owner, &zero, curr))
        return true;

    return false;
}
```
* `atomic_long_try_cmpxchg_acquire(mem, old, new)`是一个封装的原子操作
  * 先是`cmpxchg`的语义：
    * `mem`里的内容和`old`进行比较，如果相等则将`mem`里的内容更新为`new`，（无论成功与否）返回`mem`里原来的值。
    * 比较`mem`原来的值与`old`的值，如果一样，更新成功；如果不一样，说明`mem`的值被其他 CPU 修改了，更新失败。
  * 加了个`try`表示，如果更新成功，返回`true`；如果更新失败，返回`false`。
  * **注意关键的一点**，即便`cmpxchg`失败，该函数也会把`old`的指更新为`cmpxchg`的返回值，即`mem`在原子操作时的值。
* `owner`域为 **空** 表示没人持有锁，所以这里把`owner`域和`0`比较，如果成功拿到锁，`owner`更新为当前任务的`struct taske_struct`指针

### 加锁的慢速路径
* Mutex 涉及三个队列：
  * **OSQ（optimistic_spin_queue）**：`struct optimistic_spin_queue osq`，锁的拥有者在运行，新的竞争者挂入这个队列。队列的第一个任务持有 OSQ 锁后在`owner`上自旋，其他任务在自己的`locked`域上睡眠
  * **等待队列**：`struct list_head wait_list`，锁的拥有者去睡眠，新出现的竞争者会加入这个队列；而 OSQ 上的竞争者发现这个事情后也会结束自旋，逐个移入到这个队列（不排除移到一半锁的拥有者醒来的情况，导致 OSQ 上的任务又进入自旋状态）
  * **唤醒队列**：`wake_q`在释放锁时慢速路径的栈上分配，上面挂着可以被唤醒的进程

#### 尝试拿锁 __mutex_trylock_common()
* `__mutex_trylock()`调用`__mutex_trylock_common()`时`handoff`为`false`，所以就三种结果：
  1. `owner`域的`task`位为空，说明锁已经被释放，可直接拿到锁
  2. `owner`域的`task`位非空，判断`task`是否指向自己，若不指向自己则说明锁正被用着呢，拿锁失败
  3. 若`task`指向自己，说明这个锁正在通过 handoff 机制把锁交给自己，拿锁一定成功
* 只有`__mutex_trylock_or_handoff()`调用`__mutex_trylock_common()`时`handoff`有可能为`true`
  * 如果是等待队列上的第一个进程调用，`handoff`为`true`，能拿到锁则拿，拿不到也要把`HANDOFF`标志设上，表示我们发现锁被偷了；对应地，锁拥有者在释放锁时会检查这个标志位，如果已经设上了，则清除`HANDOFF`标志，设置`PICKUP`标志并把它所看到的第一等待者设为`task`
  * 如果不是等待队列上的第一个进程调用，`handoff`为`false`，效果和调用`__mutex_trylock()`一样
* kernel/locking/mutex.c
```c
static inline struct task_struct *__owner_task(unsigned long owner)
{
    return (struct task_struct *)(owner & ~MUTEX_FLAGS); // 返回 owner 域的 task 位
}
...
static inline unsigned long __owner_flags(unsigned long owner)
{
    return owner & MUTEX_FLAGS; // 返回 owner 域的 flags 位
}

static inline struct task_struct *__mutex_trylock_common(struct mutex *lock, bool handoff)
{
    unsigned long owner, curr = (unsigned long)current;

    owner = atomic_long_read(&lock->owner);
    for (;;) { /* must loop, can race against a flag */
        unsigned long flags = __owner_flags(owner); // 得到 owner 域的 flags 位
        unsigned long task = owner & ~MUTEX_FLAGS;  // 得到 owner 域的 task 位
        // owner 域的信息分开放到 flags 和 task 里（即使原子操作失败，owner 也会被更新为 lock->owner 的新值）
        if (task) {
            if (flags & MUTEX_FLAG_PICKUP) { // 释放锁那边 handoff 已经完成，等待被移交的进程来拿锁
                if (task != curr) // 来尝试的进程是不是被移交的进程？
                    break; // 不是，本函数返回 task，由于 task 不为空，调用本函数的 trylock 返回 false，尝试拿锁失败
                // 是，清除 pickup 标志；理论上，下面的原子操作一定能拿锁成功，且返回 NULL，调用本函数的 trylock 返回 true，尝试拿锁成功
                flags &= ~MUTEX_FLAG_PICKUP;
            } else if (handoff) { // 当我们是等待队列上第一个进程时，尝试拿锁或设置 HANDOFF 标志
                if (flags & MUTEX_FLAG_HANDOFF)
                    break; // 锁的 HANDOFF 已经被设过了，调用本函数的 trylock 返回 false，尝试失败
                flags |= MUTEX_FLAG_HANDOFF; // 我是第一个等待者，由我来负责设置 HANDOFF 标志
                // 我们在下面的原子操作试一试：
                // 1. 原子操作成功且 task == curr 成立，这个情况不太可能，我们是等待者，必然不是 owner
                // 2. 原子操作成功，但 task != curr，在不改变 owner task 的情况下设置了 HANDOFF 标志，
                //    期望 owner 释放锁的时候需要把锁交给我，虽然尝试拿锁失败，但也不是无功而返
                // 3. 原子操作失败，锁被偷了，我们通过循环再次尝试设置 HANDOFF 标志
            } else {    // 当前锁已经被占用了，且没在等待被移交的进程来拿锁，我们也不是等待队列上的第一个进程
                break;  // 让调用本函数的 trylock 返回 false，尝试拿锁失败
            }
        } else {
            MUTEX_WARN_ON(flags & (MUTEX_FLAG_HANDOFF | MUTEX_FLAG_PICKUP)); // 此时不应该有这两个标志位
            // owner 的 task 位已经空了，我们将通过下面的原子操作尝试拿锁：
            // 1. 原子操作成功，理论上此时 task == curr 一定成立，我们拿到锁
            // 2. 原子操作失败，owner 刷新为 lock->owner 的新值，会进入到上面的 else 分支，拿锁失败
            task = curr;
        }
        // 原子操作 + try 尝试失败，还有重试的机会，注意，失败还会刷新 owner 的值为现在 lock->owner 的值
        if (atomic_long_try_cmpxchg_acquire(&lock->owner, &owner, task | flags)) {
            if (task == curr) // 原子操作成功且 task == curr 成立，调用本函数的 trylock 返回 true，尝试拿锁成功
                return NULL;  // 拿锁成功
            break; // 原子操作成功但 task == curr 不成立，调用本函数的 trylock 返回 false，尝试拿锁失败
        }
    }

    return __owner_task(owner);
}

/*
 * Trylock or set HANDOFF
 */
static inline bool __mutex_trylock_or_handoff(struct mutex *lock, bool handoff)
{   // 当进程是等待队列的第一个进程时 handoff 为 true
    return !__mutex_trylock_common(lock, handoff);
}

/*
 * Actual trylock that will work on any unlocked state.
 */
static inline bool __mutex_trylock(struct mutex *lock)
{   // __mutex_trylock_common() 需要返回 NULL，我们才能认为 trylock 成功
    return !__mutex_trylock_common(lock, false);
}
...
/**
 * mutex_trylock - try to acquire the mutex, without waiting
 * @lock: the mutex to be acquired
 *
 * Try to acquire the mutex atomically. Returns 1 if the mutex
 * has been acquired successfully, and 0 on contention.
 *
 * NOTE: this function follows the spin_trylock() convention, so
 * it is negated from the down_trylock() return values! Be careful
 * about this when converting semaphore users to mutexes.
 *
 * This function must not be used in interrupt context. The
 * mutex must be released by the same task that acquired it.
 */
int __sched mutex_trylock(struct mutex *lock)
{
    bool locked;

    MUTEX_WARN_ON(lock->magic != lock);

    locked = __mutex_trylock(lock);
    if (locked)
        mutex_acquire(&lock->dep_map, 0, 1, _RET_IP_);

    return locked;
}
```

#### mutex 乐观自旋

##### mutex_spin_on_owner()
* `mutex_spin_on_owner()`在给的`owner`上自旋，直到以下三个条件之一达成：
  1. 如果`owner`不在运行了，返回`false`
  2. 我们需要被调度了，返回`false`
  3. 如果`owner`发生了变化，返回`true`
* kernel/locking/mutex.c
```c
/* 小心！"owner" 是一个完全推测的指针访问且并不可靠
 * Look out! "owner" is an entirely speculative pointer access and not
 * reliable.
 * “noinline” 是为了能在 Perf 剖析中被显示出来
 * "noinline" so that this function shows up on perf profiles.
 */
static noinline
bool mutex_spin_on_owner(struct mutex *lock, struct task_struct *owner,
             struct ww_acquire_ctx *ww_ctx, struct mutex_waiter *waiter)
{
    bool ret = true;
    // 不停读取 mutex 当前的 owner 还是不是我们期望的 owner；如果不是，说明锁曾经被释放过
    rcu_read_lock();
    while (__mutex_owner(lock) == owner) {
        /*
         * Ensure we emit the owner->on_cpu, dereference _after_
         * checking lock->owner still matches owner. If that fails,
         * owner might point to freed memory. If it still matches,
         * the rcu_read_lock() ensures the memory stays valid.
         */
        barrier();
        // 下面的条件成立意味着争用者要去睡眠
        /*
         * Use vcpu_is_preempted to detect lock holder preemption issue.
         */
        if (!owner->on_cpu || need_resched() ||
                vcpu_is_preempted(task_cpu(owner))) {
            ret = false; // owner 已经不在运行了或我们需要被调度，我们要去睡眠了
            break;
        }

        if (ww_ctx && !ww_mutex_spin_on_owner(lock, ww_ctx, waiter)) {
            ret = false;
            break;
        }

        cpu_relax();
    }
    rcu_read_unlock();

    return ret;
}
// 检查 mutex 能否在 owner 上乐观自旋
/*
 * Initial check for entering the mutex spinning loop
 */
static inline int mutex_can_spin_on_owner(struct mutex *lock)
{
    struct task_struct *owner;
    int retval = 1;
    // 如果当前准备乐观自旋争锁的进程 TIF_NEED_RESCHED 标志被设置，不要乐观自旋
    if (need_resched())
        return 0;

    rcu_read_lock();
    owner = __mutex_owner(lock);
    // 得到当前锁的 owner
    /*
     * As lock holder preemption issue, we both skip spinning if task is not
     * on cpu or its cpu is preempted
     */
    if (owner) // 锁的持有者在 CPU 上运行才自旋争锁，vcpu_is_preempted() 为虚拟化相关的判断
        retval = owner->on_cpu && !vcpu_is_preempted(task_cpu(owner));
    rcu_read_unlock();
    // 如果锁没有 owner，说明 mutex 已经被释放，尝试在自旋争锁的路径上去拿锁，这样更快，而不是回到慢速路径上去争锁
    /*
     * If lock->owner is not set, the mutex has been released. Return true
     * such that we'll trylock in the spin path, which is a faster option
     * than the blocking slow path.
     */
    return retval;
}
```
##### mutex_optimistic_spin()
* 当我们发现锁的持有者当前正运行在一个（不同的）CPU 上时，我们尽可能以自旋的方式去争用锁，我们并不需要被调度出去。这么做的理由是，如果锁的持有者正在运行，它很可能会很快释放锁
* 拿到锁时返回 **true**，否则返回 **false**，意味着我们需要转到慢速路径并且睡眠
* 如果自旋者在等待队列上，`waiter`标志位被设为`true`，这种情况下就不要加到乐观自旋锁的队列上了，而是在锁的 owner 上自旋
```c
/*
 * Optimistic spinning.
 *
 * We try to spin for acquisition when we find that the lock owner
 * is currently running on a (different) CPU and while we don't
 * need to reschedule. The rationale is that if the lock owner is
 * running, it is likely to release the lock soon.
 *
 * The mutex spinners are queued up using MCS lock so that only one
 * spinner can compete for the mutex. However, if mutex spinning isn't
 * going to happen, there is no point in going through the lock/unlock
 * overhead.
 *
 * Returns true when the lock was taken, otherwise false, indicating
 * that we need to jump to the slowpath and sleep.
 *
 * The waiter flag is set to true if the spinner is a waiter in the wait
 * queue. The waiter-spinner will spin on the lock directly and concurrently
 * with the spinner at the head of the OSQ, if present, until the owner is
 * changed to itself.
 */
static __always_inline bool
mutex_optimistic_spin(struct mutex *lock, struct ww_acquire_ctx *ww_ctx,
              struct mutex_waiter *waiter)
{
    if (!waiter) {
        /*
         * The purpose of the mutex_can_spin_on_owner() function is
         * to eliminate the overhead of osq_lock() and osq_unlock()
         * in case spinning isn't possible. As a waiter-spinner
         * is not going to take OSQ lock anyway, there is no need
         * to call mutex_can_spin_on_owner().
         */
        // 这里调用 mutex_can_spin_on_owner() 的目的是，当自旋不可能拿到锁时，消除 osq_lock() 和 osq_unlock() 的潜在开销
        if (!mutex_can_spin_on_owner(lock))
            goto fail;
        // 为了避免突然惊跑正在尝试获取锁的 mutex 自旋者，自旋者在 owner 域上自旋前，需要先获取 MCS 排队锁
        /*
         * In order to avoid a stampede of mutex spinners trying to
         * acquire the mutex all at once, the spinners need to take a
         * MCS (queued) lock first before spinning on the owner field.
         */
        if (!osq_lock(&lock->osq))
            goto fail; // 获取乐观自旋锁失败
    }    

    for (;;) {
        struct task_struct *owner;

        /* Try to acquire the mutex... */
        owner = __mutex_trylock_or_owner(lock); // 再次争锁，如果失败，返回锁的 owner
        if (!owner) // 争锁是否成功？上面的函数返回 NULL 表示成功
           break; // 成功了，能走到下面的 return true
        // 争锁失败，锁已经有 owner 了，自旋在 lock->owner 上，等待上面提到的三个条件
        /*
         * There's an owner, wait for it to either
         * release the lock or go to sleep.
         */
        if (!mutex_spin_on_owner(lock, owner, ww_ctx, waiter))
            goto fail_unlock; // 条件 1 或 2 达成，我们要去睡眠了
        // 条件 3 达成，lock->owner 发生变化了，可以回到循环开始再次重试拿锁
        /*
         * The cpu_relax() call is a compiler barrier which forces
         * everything in this loop to be re-loaded. We don't need
         * memory barriers as we'll eventually observe the right
         * values at the cost of a few extra spins.
         */
        cpu_relax();
    }

    if (!waiter)
       osq_unlock(&lock->osq);

    return true;


fail_unlock:
    if (!waiter)
       osq_unlock(&lock->osq);

fail:
    /*
     * If we fell out of the spin path because of need_resched(),
     * reschedule now, before we try-lock the mutex. This avoids getting
     * scheduled out right after we obtained the mutex.
     */
    // 如果我们是因为需要被重新调度的原因而从自旋路径上掉出来的，那现在就重新调度吧，
    // 别去尝试竞争 mutex 锁了。这样避免我们一拿到锁就又要被调度出去。否则返回争锁失败。
    if (need_resched()) {
       /*
        * We _should_ have TASK_RUNNING here, but just in case
        * we do not, make it so, otherwise we might get stuck.
        */
       __set_current_state(TASK_RUNNING); // 防患于未然，能走到这通常不在任何等待队列上
       schedule_preempt_disabled(); // [抢占点 1]
    }

    return false;
}
```

#### 慢速路径 __mutex_lock_common()
* kernel/locking/mutex.c
```c
/*
 * Lock a mutex (possibly interruptible), slowpath:
 */
static __always_inline int __sched
__mutex_lock_common(struct mutex *lock, unsigned int state, unsigned int subclass,
            struct lockdep_map *nest_lock, unsigned long ip,
            struct ww_acquire_ctx *ww_ctx, const bool use_ww_ctx)
{
    struct mutex_waiter waiter;
    struct ww_mutex *ww;
    int ret;
    // 如果未 Wound/Wait Mutexes 就先不考虑 ww_ctx
    if (!use_ww_ctx)
        ww_ctx = NULL;
    // 如果在原子上下文，运行到这里会给出告警
    might_sleep();
    // 未开启锁调试时以下宏为空，否则比较 magic 与 lock
    MUTEX_WARN_ON(lock->magic != lock);
    // 根据 struct ww_mutex 的 base 域的地址，即 struct mutex *lock 得到锁所属的 struct ww_mutex 实例的地址
    ww = container_of(lock, struct ww_mutex, base);
    if (ww_ctx) {
        if (unlikely(ww_ctx == READ_ONCE(ww->ctx)))
            return -EALREADY;

        /*
         * Reset the wounded flag after a kill. No other process can
         * race and wound us here since they can't have a valid owner
         * pointer if we don't have any locks held.
         */
        if (ww_ctx->acquired == 0)
            ww_ctx->wounded = 0;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
        nest_lock = &ww_ctx->dep_map;
#endif
    }
    // 关闭抢占
    preempt_disable();
    mutex_acquire_nest(&lock->dep_map, subclass, 0, nest_lock, ip); // lock_dep 相关
    // 先尝试拿锁，拿不到再以乐观自旋的方式争锁
    if (__mutex_trylock(lock) ||
        mutex_optimistic_spin(lock, ww_ctx, NULL)) {
        /* got the lock, yay! */
        lock_acquired(&lock->dep_map, ip);
        if (ww_ctx)
            ww_mutex_set_context_fastpath(ww, ww_ctx);
        preempt_enable();
        return 0; // 我们拿到锁了，返回 0
    }
    // 乐观自旋的方式都拿不到锁，那我们要被挂上等待队列了
    raw_spin_lock(&lock->wait_lock);
    /*
     * After waiting to acquire the wait_lock, try again.
     */
    if (__mutex_trylock(lock)) { // 在等待拿等待队列自旋锁的时间，可能锁已经释放了，再试试
       if (ww_ctx)
          __ww_mutex_check_waiters(lock, ww_ctx);
       // 很幸运，拿到锁了，不需要继续挂上等待队列的操作
       goto skip_wait;
    }
    // 还是没拿到锁
    debug_mutex_lock_common(lock, &waiter);
    waiter.task = current;
    if (use_ww_ctx)
        waiter.ww_ctx = ww_ctx;

    lock_contended(&lock->dep_map, ip);

    if (!use_ww_ctx) {
        /* add waiting tasks to the end of the waitqueue (FIFO): */
        // 把进程挂上等待队列，如果是挂队列上第一个进程的，还要设置 mutex owner 的 WAITERS 位
        __mutex_add_waiter(lock, &waiter, &lock->wait_list);
    } else {
        /*
         * Add in stamp order, waking up waiters that must kill
         * themselves.
         */
        ret = __ww_mutex_add_waiter(&waiter, lock, ww_ctx);
        if (ret)
            goto err_early_kill;
    }

    set_current_state(state); // 设置进程的状态 TASK_KILLABLE 或 TASK_UN/INTERRUPTIBLE
    for (;;) {
        bool first;
        // 一旦我们持有等待队列的锁，我们会被 mutex_unlock() 把锁 handoff 给我们的操作和我们加入等待队列的
        // 操作序列化，所以我们这里可以一试，确保在我们测试错误条件前能拾起 handoff 给我们的锁
        /*
         * Once we hold wait_lock, we're serialized against
         * mutex_unlock() handing the lock off to us, do a trylock
         * before testing the error conditions to make sure we pick up
         * the handoff.
         */
        if (__mutex_trylock(lock))
            goto acquired; // 还真拿到了
        // 检查争用等待队列锁时收到的信号。这确保取消锁操作和 mutex_unlock()操作是按顺序的，且不会错过被唤醒
        /*
         * Check for signals and kill conditions while holding
         * wait_lock. This ensures the lock cancellation is ordered
         * against mutex_unlock() and wake-ups do not go missing.
         */
        if (signal_pending_state(state, current)) {
            ret = -EINTR;
            goto err; // 没抢到锁，因为信号的原因而被唤醒
        }

        if (ww_ctx) {
            ret = __ww_mutex_check_kill(lock, &waiter, ww_ctx);
            if (ret)
                goto err;
        }

        raw_spin_unlock(&lock->wait_lock);
        schedule_preempt_disabled(); // [抢占点 2]

        first = __mutex_waiter_is_first(lock, &waiter); // 记录我们是否等待队列上第一个

        set_current_state(state); // 还没拿到锁，可能会回到[抢占点 2]再次睡眠
        /*
         * Here we order against unlock; we must either see it change
         * state back to RUNNING and fall through the next schedule(),
         * or we must see its unlock and acquire.
         */
        // 1. 如果我们是等待队列的第一个，
        //    1）我们就尝试拿锁或设上 HANDOFF 标志，期待被 handoff，
        //      拿锁成功，跳出循环；拿锁失败，进入下一条件。
        //    2）如果以上尝试失败，不要加到乐观自旋锁的队列（OSQ）上了，而是在锁的 owner 上自旋，
        //      拿锁成功，跳出循环；拿锁失败，回到 for 循环的开头进行新的一轮重试。
        // 2. 如果不是等待队列的第一个，尝试拿锁。拿锁成功，跳出循环，拿锁失败回到 for 循环的开头进行新的一轮重试
        if (__mutex_trylock_or_handoff(lock, first) ||
            (first && mutex_optimistic_spin(lock, ww_ctx, &waiter)))
            break; // 拿锁成功，跳出循环
        // 新的一轮重试，可能在[抢占点 2]再次睡眠
        raw_spin_lock(&lock->wait_lock);
    }
    raw_spin_lock(&lock->wait_lock);
acquired: // 拿锁成功的路径
    __set_current_state(TASK_RUNNING);

    if (ww_ctx) {
        /*
         * Wound-Wait; we stole the lock (!first_waiter), check the
         * waiters as anyone might want to wound us.
         */
        if (!ww_ctx->is_wait_die &&
            !__mutex_waiter_is_first(lock, &waiter))
            __ww_mutex_check_waiters(lock, ww_ctx);
    }
    // 从等待队列中移除
    __mutex_remove_waiter(lock, &waiter);

    debug_mutex_free_waiter(&waiter);

skip_wait:
    /* got the lock - cleanup and rejoice! */
    lock_acquired(&lock->dep_map, ip);

    if (ww_ctx)
        ww_mutex_lock_acquired(ww, ww_ctx);

    raw_spin_unlock(&lock->wait_lock); // 释放保护等待队列的自旋锁
    preempt_enable(); // 开抢占
    return 0; // 拿锁成功

err: // 因错误而退出争锁
    __set_current_state(TASK_RUNNING);
    __mutex_remove_waiter(lock, &waiter);
err_early_kill:
    raw_spin_unlock(&lock->wait_lock);
    debug_mutex_free_waiter(&waiter);
    mutex_release(&lock->dep_map, ip);
    preempt_enable();
    return ret;
}
...
static int __sched
__mutex_lock(struct mutex *lock, unsigned int state, unsigned int subclass,
         struct lockdep_map *nest_lock, unsigned long ip)
{
    return __mutex_lock_common(lock, state, subclass, nest_lock, ip, NULL, false);
}
...
static noinline void __sched
__mutex_lock_slowpath(struct mutex *lock)
{
    __mutex_lock(lock, TASK_UNINTERRUPTIBLE, 0, NULL, _RET_IP_);
}
```
* 等待队列上的第一个进程从 *[抢占点 2]* 被唤醒到`__mutex_trylock_or_handoff(lock, first)`成功设置`HANDOFF`标志之间，还是有多个进程可以多次拿锁的，但设置`HANDOFF`标志并不会因为锁被占用而阻塞。一旦设上了，就会通过`mutex_optimistic_spin()`自旋在`owner`上（而且不用加到 OSQ 上，因为我们需要被立即被 handoff），等待着锁的拥有者设置`PICKUP`标志并将`task`位指向我们后释放锁。
* *[抢占点 1]* 发生的调度是内核抢占，进程仍然是`TASK_RUNNING`；而 *[抢占点 2]* 则是睡眠

## mutex 解锁
* 不能用于中断上下文
* 不允许用于一个未加锁的 mutex
```c
/**
 * mutex_unlock - release the mutex
 * @lock: the mutex to be released
 *
 * Unlock a mutex that has been locked by this task previously.
 *
 * This function must not be used in interrupt context. Unlocking
 * of a not locked mutex is not allowed.
 *
 * This function is similar to (but not equivalent to) up().
 */
void __sched mutex_unlock(struct mutex *lock)
{
#ifndef CONFIG_DEBUG_LOCK_ALLOC
    if (__mutex_unlock_fast(lock))
        return;
#endif
    __mutex_unlock_slowpath(lock, _RET_IP_);
}
```
### 解锁的快速路径
* 对于没有争用的 mutex，`owner`域的低位的 flags 都是 0，如果正好是当前进程，则快速完成解锁
```c
static __always_inline bool __mutex_unlock_fast(struct mutex *lock)
{
    unsigned long curr = (unsigned long)current;

    return atomic_long_try_cmpxchg_release(&lock->owner, &curr, 0UL);
}
```
### 解锁的慢速路径

```c
/*
 * Release the lock, slowpath:
 */
static noinline void __sched __mutex_unlock_slowpath(struct mutex *lock, unsigned long ip)
{
    struct task_struct *next = NULL;
    DEFINE_WAKE_Q(wake_q);
    unsigned long owner;

    mutex_release(&lock->dep_map, ip);
    // 除非 HANDOFF，在那种情况下我们必须不清 owner 域，而是把它设成第一等待者。
    /*
     * Release the lock before (potentially) taking the spinlock such that
     * other contenders can get on with things ASAP.
     *
     * Except when HANDOFF, in that case we must not clear the owner field,
     * but instead set it to the top waiter.
     */
    owner = atomic_long_read(&lock->owner);
    for (;;) {
        MUTEX_WARN_ON(__owner_task(owner) != current); // 释放锁的不是当前进程？！
        MUTEX_WARN_ON(owner & MUTEX_FLAG_PICKUP); // 这个标志是由偷锁者在下面设的，不应该出现在这

        if (owner & MUTEX_FLAG_HANDOFF) // 是否有等待者需要 handoff？
            break; // 对于 HANDOFF 我们需要设置 owner 为第一等待者，处理在下面
        // 没有需要 handoff 的等待者，__owner_flags() 只保留标志位，所以原子操作成功，owner 域的 task 位被清除
        if (atomic_long_try_cmpxchg_release(&lock->owner, &owner, __owner_flags(owner))) {
            if (owner & MUTEX_FLAG_WAITERS) // 等待队列上是否还有等待者？
                break; // 还有，在下面唤醒它们
            // 等待队列上没有等待者了，释放锁结束
            return;
        }
        // 循环前读到的锁拥有者和原子操作时读到的锁拥有者不一致时会导致再次循环，很可能是非 mutex 的拥有者在释放锁
        // 即便原子操作失败，owner 的值也会更新为 lock->owner 现在的值，以此开始下次循环
    }

    raw_spin_lock(&lock->wait_lock);
    debug_mutex_unlock(lock);
    if (!list_empty(&lock->wait_list)) { // 等待队列是否为空？
        /* get the first entry from the wait-list: */
        struct mutex_waiter *waiter =
            list_first_entry(&lock->wait_list,
                     struct mutex_waiter, list);

        next = waiter->task; // 指向第一个等待者的 task_struct

        debug_mutex_wake_waiter(lock, waiter);
        wake_q_add(&wake_q, next); // 第一个等待者加入唤醒队列
    }
    // 第一等待者发觉了锁被偷了，设置了 HANDOFF 标志，所以偷锁者解锁时要保证把锁交给它
    if (owner & MUTEX_FLAG_HANDOFF)
        __mutex_handoff(lock, next);

    raw_spin_unlock(&lock->wait_lock); // 释放保护等待队列的锁

    wake_up_q(&wake_q); // 唤醒等待队列上的进程
}
```

#### 交接锁 __mutex_handoff()
* 当`task = NULL`时，等同于常规解锁
* handoff 时设置`PICKUP`标志，清除`HANDOFF`标志，保留`WAITERS`标志
* 像常规解锁一样提供`RELEASE`语义，`__mutex_trylock()`为 handoff 提供一个对应的`ACQUIRE`语义
```c
/*
 * Give up ownership to a specific task, when @task = NULL, this is equivalent
 * to a regular unlock. Sets PICKUP on a handoff, clears HANDOFF, preserves
 * WAITERS. Provides RELEASE semantics like a regular unlock, the
 * __mutex_trylock() provides a matching ACQUIRE semantics for the handoff.
 */     
static void __mutex_handoff(struct mutex *lock, struct task_struct *task)
{               
    unsigned long owner = atomic_long_read(&lock->owner);

    for (;;) {
        unsigned long new;

        MUTEX_WARN_ON(__owner_task(owner) != current); // 释放锁的不是当前进程？！
        MUTEX_WARN_ON(owner & MUTEX_FLAG_PICKUP); // 该标志由我们在下面设上的，怎么这就已经有了？

        new = (owner & MUTEX_FLAG_WAITERS); // 保留 owner 的 waiter 位
        new |= (unsigned long)task; // 设置 task 位为要递交锁的进程
        if (task) // 要被递交锁的进程是否为空？
            new |= MUTEX_FLAG_PICKUP; // 设置 pickup 位，不管 handoff 位就等于把它清除了
        // 更新锁的 owner 域
        if (atomic_long_try_cmpxchg_release(&lock->owner, &owner, new))
            break; // 原子操作成功，交接完成
        // 即便原子操作失败，owner 的值也会更新为 lock->owner 现在的值，以此开始下次循环
    }
}
```

## Mutex 锁调试

```c
#include "mutex-debug.h"

/*
 * Must be called with lock->wait_lock held.
 */
/*调用于加锁之时，还未获得锁之前*/
void debug_mutex_lock_common(struct mutex *lock, struct mutex_waiter *waiter)
{
	memset(waiter, MUTEX_DEBUG_INIT, sizeof(*waiter)); //poison 栈上的变量 waiter
	waiter->magic = waiter; //设置锁调试特定域 magic
	INIT_LIST_HEAD(&waiter->list); //该 waiter 稍后会被挂在 lock->wait_list 上
}
/*调用于解锁之时，未调用 wake_q_add()之前*/
void debug_mutex_wake_waiter(struct mutex *lock, struct mutex_waiter *waiter)
{
	SMP_DEBUG_LOCKS_WARN_ON(!spin_is_locked(&lock->wait_lock)); //mutex 的等锁队列未上锁！
	DEBUG_LOCKS_WARN_ON(list_empty(&lock->wait_list)); //因为该函数在等锁队列不为空的分支被调用，等锁队列为空意味着有问题！
	DEBUG_LOCKS_WARN_ON(waiter->magic != waiter); //waiter 的 magic 域被修改！
	DEBUG_LOCKS_WARN_ON(list_empty(&waiter->list)); //waiter 刚从等锁队列上被找到，还未拿下来，此时 waiter->list 为空必为异常！waiter->list 要一直等到锁争用成功后，在 mutex_remove_waiter() 里从等锁队列里删除
}
/*调用于加锁之时，获得锁之后，mutex_remove_waiter()之后*/
void debug_mutex_free_waiter(struct mutex_waiter *waiter)
{
	DEBUG_LOCKS_WARN_ON(!list_empty(&waiter->list));
	memset(waiter, MUTEX_DEBUG_FREE, sizeof(*waiter));
}
/*调用于加锁之时，还未获得锁之前，紧跟 debug_mutex_lock_common()*/
void debug_mutex_add_waiter(struct mutex *lock, struct mutex_waiter *waiter,
			    struct task_struct *task)
{
	SMP_DEBUG_LOCKS_WARN_ON(!spin_is_locked(&lock->wait_lock)); //未获得 mutex 等锁队列的锁！

	/* Mark the current thread as blocked on the lock: */
	task->blocked_on = waiter; /*在 task_struct 里记录导致任务阻塞的 mutex*/
}
/*调用于加锁之时，获得锁之后，debug_mutex_free_waiter()之前。
  注意：无论 mutex 锁调试是否打开，该函数都会被调用到*/
void mutex_remove_waiter(struct mutex *lock, struct mutex_waiter *waiter,
			 struct task_struct *task)
{
	DEBUG_LOCKS_WARN_ON(list_empty(&waiter->list)); //争锁前无论是否获得锁，都会先把 waiter 挂于 lock->wait_list 上，因此该条件不成立肯定是有问题
	DEBUG_LOCKS_WARN_ON(waiter->task != task); //争锁前无论是否获得锁，都会先把 waiter-> task = current，争锁成功后发现该条件不成立，肯定有问题
	DEBUG_LOCKS_WARN_ON(task->blocked_on != waiter);//争锁成功后，任务阻塞的 waiter 与之前不一致，肯定有问题！mutex锁调试未打开时，task->blocked_on 恒等于 NULL
	task->blocked_on = NULL;

	list_del_init(&waiter->list); //争锁成功后，waiter 需从 lock->wait_list 上删除
	waiter->task = NULL; //争锁成功后，waiter 不再有意义，task 域置为 NULL
}
/*调用于解锁之时，调用 debug_mutex_wake_waiter()之前*/
void debug_mutex_unlock(struct mutex *lock)
{
	if (likely(debug_locks)) {
		DEBUG_LOCKS_WARN_ON(lock->magic != lock); //将要解的锁被破坏了！
		DEBUG_LOCKS_WARN_ON(!lock->wait_list.prev && !lock->wait_list.next); //将要解的锁的等锁队列异常！
	}
}
/*调用于 __mutex_init()*/
void debug_mutex_init(struct mutex *lock, const char *name,
		      struct lock_class_key *key)
{
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	/*
	 * Make sure we are not reinitializing a held lock:
	 */
	debug_check_no_locks_freed((void *)lock, sizeof(*lock));
	lockdep_init_map(&lock->dep_map, name, key, 0);
#endif
	lock->magic = lock; //mutex 锁调试特定域赋初值
}
/*mutex 锁调试被关闭时，该函数定义为空函数*/
/***
 * mutex_destroy - mark a mutex unusable
 * @lock: the mutex to be destroyed
 *
 * This function marks the mutex uninitialized, and any subsequent
 * use of the mutex is forbidden. The mutex must not be locked when
 * this function is called.
 */
void mutex_destroy(struct mutex *lock)
{
	DEBUG_LOCKS_WARN_ON(mutex_is_locked(lock));
	lock->magic = NULL;
}

EXPORT_SYMBOL_GPL(mutex_destroy);
```
