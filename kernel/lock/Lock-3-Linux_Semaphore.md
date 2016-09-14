# Linux信号量（Semaphore）的实现

## 头文件及结构体定义

* include/linux/semaphore.h
```c
/*
 * Copyright (c) 2008 Intel Corporation
 * Author: Matthew Wilcox <willy@linux.intel.com>
 *
 * Distributed under the terms of the GNU GPL, version 2
 *
 * Please see kernel/semaphore.c for documentation of these functions
 */
#ifndef __LINUX_SEMAPHORE_H
#define __LINUX_SEMAPHORE_H

#include <linux/list.h>
#include <linux/spinlock.h>

/* Please don't access any members of this structure directly */
struct semaphore {
    raw_spinlock_t      lock;      /*操作信号量计数使需用自旋锁保护*/
    unsigned int        count;     /*信号量计数*/
    struct list_head    wait_list; /*等待任务列表*/
};

#define __SEMAPHORE_INITIALIZER(name, n)                \
{                                   \
    .lock       = __RAW_SPIN_LOCK_UNLOCKED((name).lock),    \
    .count      = n,                        \
    .wait_list  = LIST_HEAD_INIT((name).wait_list),     \
}

#define DEFINE_SEMAPHORE(name)  \
    struct semaphore name = __SEMAPHORE_INITIALIZER(name, 1)

static inline void sema_init(struct semaphore *sem, int val)
{
    static struct lock_class_key __key;
    *sem = (struct semaphore) __SEMAPHORE_INITIALIZER(*sem, val);
    lockdep_init_map(&sem->lock.dep_map, "semaphore->lock", &__key, 0);
}

extern void down(struct semaphore *sem);
extern int __must_check down_interruptible(struct semaphore *sem);
extern int __must_check down_killable(struct semaphore *sem);
extern int __must_check down_trylock(struct semaphore *sem);
extern int __must_check down_timeout(struct semaphore *sem, long jiffies);
extern void up(struct semaphore *sem);

#endif /* __LINUX_SEMAPHORE_H */
```

* include/linux/spinlock_types.h
```c
...
#define __RAW_SPIN_LOCK_INITIALIZER(lockname)   \
    {                   \
    .raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,  \
    SPIN_DEBUG_INIT(lockname)       \
    SPIN_DEP_MAP_INIT(lockname) }
...
```

* arch/x86/include/asm/spinlock_types.h
```c
...
#define __ARCH_SPIN_LOCK_UNLOCKED   { { 0 } }
...
```

## 实现
* 成员`.lock`是自旋锁，用来控制对信号量其他成员的访问。
* 成员`.count`是信号量计数，表示有多少任务正持有锁。
* 如果`.count`等于**0**，意味着有许多任务在等待列表上。
* `down()`一族的函数会引起睡眠，因此不能在中断上下文中被调用，但是`down_trylock()`除外，因为其无论得不得的到锁都不会引起睡眠，因此可以在中断上下文中使用。
* 不像互斥量，`up()`可能在任何上下文中被调用，甚至是被之前未调用过`down()`的任务调用。
* 这样一来，会出现中断上下文与进程上下文共享数据（`.count`）的问题。设想这过程中未禁止中断，在进程上下文拿了自旋锁（`.lock`）后操作信号量计数，此期间如果被中断打断，并且中断对该信号量调用`up()`函数，那么`up()`拿自旋锁的时候会自旋，中断未能返回造成死锁。因此，在操作信号量计数时必须得禁止中断。
* 而大部分时候，在不确定中断状态的情况下，应该用`spin_lock_irqsave()`和`spin_lock_irqrestore()`一类的函数。


* kernel/locking/semaphore.c
```c
/*
 * Copyright (c) 2008 Intel Corporation
 * Author: Matthew Wilcox <willy@linux.intel.com>
 *
 * Distributed under the terms of the GNU GPL, version 2
 *
 * This file implements counting semaphores.
 * A counting semaphore may be acquired 'n' times before sleeping.
 * See mutex.c for single-acquisition sleeping locks which enforce
 * rules which allow code to be debugged more easily.
 */

/*
 * Some notes on the implementation:
 *
 * The spinlock controls access to the other members of the semaphore.
 * down_trylock() and up() can be called from interrupt context, so we
 * have to disable interrupts when taking the lock.  It turns out various
 * parts of the kernel expect to be able to use down() on a semaphore in
 * interrupt context when they know it will succeed, so we have to use
 * irqsave variants for down(), down_interruptible() and down_killable()
 * too.
 *
 * The ->count variable represents how many more tasks can acquire this
 * semaphore.  If it's zero, there may be tasks waiting on the wait_list.
 */

#include <linux/compiler.h>
#include <linux/kernel.h>
#include <linux/export.h>
#include <linux/sched.h>
#include <linux/semaphore.h>
#include <linux/spinlock.h>
#include <linux/ftrace.h>

static noinline void __down(struct semaphore *sem);
static noinline int __down_interruptible(struct semaphore *sem);
static noinline int __down_killable(struct semaphore *sem);
static noinline int __down_timeout(struct semaphore *sem, long timeout);
static noinline void __up(struct semaphore *sem);

/**
 * down - acquire the semaphore
 * @sem: the semaphore to be acquired
 *
 * Acquires the semaphore.  If no more tasks are allowed to acquire the
 * semaphore, calling this function will put the task to sleep until the
 * semaphore is released.
 *
 * Use of this function is deprecated, please use down_interruptible() or
 * down_killable() instead.
 */
void down(struct semaphore *sem)
{
	unsigned long flags;

	raw_spin_lock_irqsave(&sem->lock, flags);
	if (likely(sem->count > 0))
		sem->count--;
	else
		__down(sem);
	raw_spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(down);

/**
 * down_interruptible - acquire the semaphore unless interrupted
 * @sem: the semaphore to be acquired
 *
 * Attempts to acquire the semaphore.  If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the sleep is interrupted by a signal, this function will return -EINTR.
 * If the semaphore is successfully acquired, this function returns 0.
 */
int down_interruptible(struct semaphore *sem)
{
	unsigned long flags;
	int result = 0;

	raw_spin_lock_irqsave(&sem->lock, flags);
	if (likely(sem->count > 0))
		sem->count--;
	else
		result = __down_interruptible(sem);
	raw_spin_unlock_irqrestore(&sem->lock, flags);

	return result;
}
EXPORT_SYMBOL(down_interruptible);

/**
 * down_killable - acquire the semaphore unless killed
 * @sem: the semaphore to be acquired
 *
 * Attempts to acquire the semaphore.  If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the sleep is interrupted by a fatal signal, this function will return
 * -EINTR.  If the semaphore is successfully acquired, this function returns
 * 0.
 */
int down_killable(struct semaphore *sem)
{
	unsigned long flags;
	int result = 0;

	raw_spin_lock_irqsave(&sem->lock, flags);
	if (likely(sem->count > 0))
		sem->count--;
	else
		result = __down_killable(sem);
	raw_spin_unlock_irqrestore(&sem->lock, flags);

	return result;
}
EXPORT_SYMBOL(down_killable);

/**
 * down_trylock - try to acquire the semaphore, without waiting
 * @sem: the semaphore to be acquired
 *
 * Try to acquire the semaphore atomically.  Returns 0 if the semaphore has
 * been acquired successfully or 1 if it it cannot be acquired.
 *
 * NOTE: This return value is inverted from both spin_trylock and
 * mutex_trylock!  Be careful about this when converting code.
 *
 * Unlike mutex_trylock, this function can be used from interrupt context,
 * and the semaphore can be released by any task or interrupt.
 */
int down_trylock(struct semaphore *sem)
{
	unsigned long flags;
	int count;

	raw_spin_lock_irqsave(&sem->lock, flags);
	count = sem->count - 1;
	if (likely(count >= 0))
		sem->count = count;
	raw_spin_unlock_irqrestore(&sem->lock, flags);

	return (count < 0);
}
EXPORT_SYMBOL(down_trylock);

/**
 * down_timeout - acquire the semaphore within a specified time
 * @sem: the semaphore to be acquired
 * @timeout: how long to wait before failing
 *
 * Attempts to acquire the semaphore.  If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the semaphore is not released within the specified number of jiffies,
 * this function returns -ETIME.  It returns 0 if the semaphore was acquired.
 */
int down_timeout(struct semaphore *sem, long timeout)
{
	unsigned long flags;
	int result = 0;

	raw_spin_lock_irqsave(&sem->lock, flags);
	if (likely(sem->count > 0))
		sem->count--;
	else
		result = __down_timeout(sem, timeout);
	raw_spin_unlock_irqrestore(&sem->lock, flags);

	return result;
}
EXPORT_SYMBOL(down_timeout);

/**
 * up - release the semaphore
 * @sem: the semaphore to release
 *
 * Release the semaphore.  Unlike mutexes, up() may be called from any
 * context and even by tasks which have never called down().
 */
void up(struct semaphore *sem)
{
	unsigned long flags;

	raw_spin_lock_irqsave(&sem->lock, flags);
	if (likely(list_empty(&sem->wait_list)))
		sem->count++;
	else
	    /*如果在该信号量上的等待队列不为空，那么处于队列中等待的任务会被唤醒*/
		__up(sem);
	raw_spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(up);

/* Functions for the contended case */

struct semaphore_waiter {
	struct list_head list;
	struct task_struct *task;
	bool up;
};

/*
 * Because this function is inlined, the 'state' parameter will be
 * constant, and thus optimised away by the compiler.  Likewise the
 * 'timeout' parameter for the cases without timeouts.
 */
static inline int __sched __down_common(struct semaphore *sem, long state,
								long timeout)
{
	struct task_struct *task = current;
	struct semaphore_waiter waiter;

    /*将任务放到因争用这个信号量而等待的链表*/
	list_add_tail(&waiter.list, &sem->wait_list);
	waiter.task = task;
	waiter.up = false;

	for (;;) {
	    /*如果是因为信号的原因被唤醒，返回值得是-EINTR*/
		if (signal_pending_state(state, task))
			goto interrupted;
		if (unlikely(timeout <= 0))
			goto timed_out;
		__set_task_state(task, state);
		/*下面马上要调度了，不能在持有自旋锁时调度，此处需先解锁*/
		raw_spin_unlock_irq(&sem->lock);
		/*拿不到信号量，会被调度出去*/
		timeout = schedule_timeout(timeout);
		/*调度完成后再加锁，直至上层函数操作完后再解锁*/
		raw_spin_lock_irq(&sem->lock);
		if (waiter.up)
			return 0;
	}
 /*因信号或超时的原因苏醒的进程需要多一个移出链表的动作。
   对于正常的因信号量释放而唤醒，在up()时已经将进程移出链表了。*/
 timed_out:
	list_del(&waiter.list);
	return -ETIME;

 interrupted:
	list_del(&waiter.list);
	return -EINTR;
}

static noinline void __sched __down(struct semaphore *sem)
{
	__down_common(sem, TASK_UNINTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}

static noinline int __sched __down_interruptible(struct semaphore *sem)
{
	return __down_common(sem, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}

static noinline int __sched __down_killable(struct semaphore *sem)
{
	return __down_common(sem, TASK_KILLABLE, MAX_SCHEDULE_TIMEOUT);
}

static noinline int __sched __down_timeout(struct semaphore *sem, long timeout)
{
	return __down_common(sem, TASK_UNINTERRUPTIBLE, timeout);
}

static noinline void __sched __up(struct semaphore *sem)
{ /*将睡眠在这个信号量的等待队列上的第一个等待者取出*/
	struct semaphore_waiter *waiter = list_first_entry(&sem->wait_list,
						struct semaphore_waiter, list);
  /*将等待者移出队列*/
	list_del(&waiter->list);
	waiter->up = true;
	/*唤醒这个等待者*/
	wake_up_process(waiter->task);
}
```
