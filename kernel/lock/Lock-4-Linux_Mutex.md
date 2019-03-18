# Linux Mutex的实现

## mutex结构体及初始化

### mutex结构体

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
    /* 1: unlocked, 0: locked, negative: locked, possible waiters */
    atomic_t        count;          /*使用计数*/
    spinlock_t      wait_lock;      /*结构体访问控制自旋锁*/
    struct list_head    wait_list;  /*等待进程列表*/
#if defined(CONFIG_DEBUG_MUTEXES) || defined(CONFIG_MUTEX_SPIN_ON_OWNER)
    struct task_struct  *owner;
#endif
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    struct optimistic_spin_queue osq; /* Spinner MCS lock */
#endif
#ifdef CONFIG_DEBUG_MUTEXES
    void            *magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
#endif
};
```
### mutex结构体初始化

* kernel/locking/mutex.c
```c
void
__mutex_init(struct mutex *lock, const char *name, struct lock_class_key *key)
{
    atomic_set(&lock->count, 1);   /*使用计数直接被初始化成 1 */
    spin_lock_init(&lock->wait_lock);
    INIT_LIST_HEAD(&lock->wait_list);
    mutex_clear_owner(lock);
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    osq_lock_init(&lock->osq);
#endif

    debug_mutex_init(lock, name, key);
}

EXPORT_SYMBOL(__mutex_init);
```

## mutex加锁操作

### mutex锁调试的影响

* 对于锁调试没开启的情况有优化，如果`__mutex_fastpath_lock()`能拿到锁，就不用走`__mutex_lock_slowpath()`去尝试了。
* 这里可以比较一下不同分支调用`__mutex_lock_common()`所传的参数有什么不同。

* include/linux/mutex.h
```c
/*
 * See kernel/locking/mutex.c for detailed documentation of these APIs.
 * Also see Documentation/locking/mutex-design.txt.
 */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
extern void mutex_lock_nested(struct mutex *lock, unsigned int subclass);
...
#define mutex_lock(lock) mutex_lock_nested(lock, 0)
...
#else
extern void mutex_lock(struct mutex *lock);
...
# define mutex_lock_nested(lock, subclass) mutex_lock(lock)
...
#endif

```

* kernel/locking/mutex.c
```c
#ifndef CONFIG_DEBUG_LOCK_ALLOC
/*
 * We split the mutex lock/unlock logic into separate fastpath and
 * slowpath functions, to reduce the register pressure on the fastpath.
 * We also put the fastpath first in the kernel image, to make sure the
 * branch is predicted by the CPU as default-untaken.
 */
__visible void __sched __mutex_lock_slowpath(atomic_t *lock_count);

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
 * ( The CONFIG_DEBUG_MUTEXES .config option turns on debugging
 *   checks that will enforce the restrictions and will also do
 *   deadlock debugging. )
 *
 * This function is similar to (but not equivalent to) down().
 */
void __sched mutex_lock(struct mutex *lock)
{
    might_sleep();
    /*
     * The locking fastpath is the 1->0 transition from
     * 'unlocked' into 'locked' state.
     */
    __mutex_fastpath_lock(&lock->count, __mutex_lock_slowpath);
    mutex_set_owner(lock);
}

EXPORT_SYMBOL(mutex_lock);
#endif

...

#ifdef CONFIG_DEBUG_LOCK_ALLOC
/*锁调试开启分支*/
void __sched
mutex_lock_nested(struct mutex *lock, unsigned int subclass)
{
    might_sleep();
    __mutex_lock_common(lock, TASK_UNINTERRUPTIBLE,
                subclass, NULL, _RET_IP_, NULL, 0);
}

EXPORT_SYMBOL_GPL(mutex_lock_nested);
...
#endif

...
#ifndef CONFIG_DEBUG_LOCK_ALLOC
/*锁调试禁止分支*/
...
__visible void __sched
__mutex_lock_slowpath(atomic_t *lock_count)
{
    struct mutex *lock = container_of(lock_count, struct mutex, count);

    __mutex_lock_common(lock, TASK_UNINTERRUPTIBLE, 0,
                NULL, _RET_IP_, NULL, 0);
}
...
#endif
```

* arch/x86/include/asm/mutex_64.h
```c
/**
 * __mutex_fastpath_lock - decrement and call function if negative
 * @v: pointer of type atomic_t
 * @fail_fn: function to call if the result is negative
 *
 * Atomically decrements @v and calls <fail_fn> if the result is negative.
 */
#ifdef CC_HAVE_ASM_GOTO
static inline void __mutex_fastpath_lock(atomic_t *v,
                     void (*fail_fn)(atomic_t *))
{
    asm_volatile_goto(LOCK_PREFIX "   decl %0\n"
              "   jns %l[exit]\n"
              : : "m" (v->counter)
              : "memory", "cc"
              : exit);
    fail_fn(v);
exit:
    return;
}
#else
#define __mutex_fastpath_lock(v, fail_fn)           \
do {                                \
    unsigned long dummy;                    \
                                \
    typecheck(atomic_t *, v);               \
    typecheck_fn(void (*)(atomic_t *), fail_fn);        \
                                \
    asm volatile(LOCK_PREFIX "   decl (%%rdi)\n"        \
             "   jns 1f     \n"         \
             "   call " #fail_fn "\n"           \
             "1:"                   \
             : "=D" (dummy)             \
             : "D" (v)                  \
             : "rax", "rsi", "rdx", "rcx",      \
               "r8", "r9", "r10", "r11", "memory"); \
} while (0)
#endif
```

### mutex加锁和锁争用的实现

* 在修改mutex锁内部数据期间，抢占被禁止，但没有禁止中断。所以mutex绝对不能用在中断上下文。
* 问题：为什么修改锁数据期间要禁止抢占？
  * A：禁止抢占的原因是，mutex锁内部数据属于进程间共享数据，用自旋锁修改数据期间如果被别的进程抢占并获取同一mutex时会先在自旋锁上死锁。
* 如果争不到锁，用的是`schedule_preempt_disabled()`来重新调度，原因如上所述。
* 调用`schedule_preempt_disabled()`之前先解开自旋锁，调用`schedule_preempt_disabled()`之后立即获取自旋锁。还是那句：持有自旋锁期间不允许调度。

* kernel/locking/mutex.c
```c
/*
 * Lock a mutex (possibly interruptible), slowpath:
 */
static __always_inline int __sched
__mutex_lock_common(struct mutex *lock, long state, unsigned int subclass,
            struct lockdep_map *nest_lock, unsigned long ip,
            struct ww_acquire_ctx *ww_ctx, const bool use_ww_ctx)
{
    struct task_struct *task = current;
    struct mutex_waiter waiter;
    unsigned long flags;
    int ret;

    preempt_disable();
    mutex_acquire_nest(&lock->dep_map, subclass, 0, nest_lock, ip);

    if (mutex_optimistic_spin(lock, ww_ctx, use_ww_ctx)) {
        /* got the lock, yay! */
        preempt_enable();
        return 0;
    }

    spin_lock_mutex(&lock->wait_lock, flags);

    /*
     * Once more, try to acquire the lock. Only try-lock the mutex if
     * it is unlocked to reduce unnecessary xchg() operations.
     */
    if (!mutex_is_locked(lock) &&
        (atomic_xchg_acquire(&lock->count, 0) == 1))
        goto skip_wait;

    debug_mutex_lock_common(lock, &waiter);
    debug_mutex_add_waiter(lock, &waiter, task_thread_info(task));

    /* add waiting tasks to the end of the waitqueue (FIFO): */
    list_add_tail(&waiter.list, &lock->wait_list);
    waiter.task = task;

    lock_contended(&lock->dep_map, ip);

    for (;;) {
        /*
         * Lets try to take the lock again - this is needed even if
         * we get here for the first time (shortly after failing to
         * acquire the lock), to make sure that we get a wakeup once
         * it's unlocked. Later on, if we sleep, this is the
         * operation that gives us the lock. We xchg it to -1, so
         * that when we release the lock, we properly wake up the
         * other waiters. We only attempt the xchg if the count is
         * non-negative in order to avoid unnecessary xchg operations:
         */
        if (atomic_read(&lock->count) >= 0 &&
            (atomic_xchg_acquire(&lock->count, -1) == 1))
            break;

        /*
         * got a signal? (This code gets eliminated in the
         * TASK_UNINTERRUPTIBLE case.)
         */
        if (unlikely(signal_pending_state(state, task))) {
            ret = -EINTR;
            goto err;
        }

        if (use_ww_ctx && ww_ctx->acquired > 0) {
            ret = __ww_mutex_lock_check_stamp(lock, ww_ctx);
            if (ret)
                goto err;
        }

        __set_task_state(task, state);

        /* didn't get the lock, go to sleep: */
        spin_unlock_mutex(&lock->wait_lock, flags);
        schedule_preempt_disabled();
        spin_lock_mutex(&lock->wait_lock, flags);
    }
    __set_task_state(task, TASK_RUNNING);

    mutex_remove_waiter(lock, &waiter, current_thread_info());
    /* set it to 0 if there are no waiters left: */
    if (likely(list_empty(&lock->wait_list)))
        atomic_set(&lock->count, 0);
    debug_mutex_free_waiter(&waiter);

skip_wait:
    /* got the lock - cleanup and rejoice! */
    lock_acquired(&lock->dep_map, ip);
    mutex_set_owner(lock);

    if (use_ww_ctx) {
        struct ww_mutex *ww = container_of(lock, struct ww_mutex, base);
        ww_mutex_set_context_slowpath(ww, ww_ctx);
    }

    spin_unlock_mutex(&lock->wait_lock, flags);
    preempt_enable();
    return 0;
err:
    mutex_remove_waiter(lock, &waiter, task_thread_info(task));
    spin_unlock_mutex(&lock->wait_lock, flags);
    debug_mutex_free_waiter(&waiter);
    mutex_release(&lock->dep_map, 1, ip);
    preempt_enable();
    return ret;
}
...__```
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
