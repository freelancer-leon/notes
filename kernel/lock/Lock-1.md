# 目录
- [理论](#理论)
- [原子操作](#原子操作)
- [自旋锁（Spin Lock）](#自旋锁（Spin%5fLock）)
- [读-写自旋锁](#读-写自旋锁)
- [信号量（Semaphores）](#信号量（Semaphores）)
- [读-写信号量（Reader-Writer Semaphores）](#读-写信号量（Reader-Writer%5fSemaphores）)
- [互斥体（Mutexes）](#互斥体（Mutexes）)
- [Completion变量](#Completion变量)
- [BLK大内核锁](#BLK大内核锁)
- [顺序锁（Sequential Locks）](#顺序锁（Sequential%5fLocks）)
- [禁止抢占](#禁止抢占)
- [顺序和屏障（Barriers）](#顺序和屏障（Barriers）)

# 理论

## 概念
* **临界区**（Critical regions)
* **竞争条件**（Race conditions）：如果两个执行线程可能处于同一临界区中同时执行。
* **同步**（synchronization）：避免并发和防止竞争条件。
* **伪并发**（pseudo-concurrency）：有可能产生竞争条件的并发操作并不是真正的同时发生，但它们相互交叉进行。
* **真并发**（true concurrency）：在SMP机器中，两个进程可以真正地在临界区中同时执行。
* **锁的争用**（lock contention）：指当锁正在被占用时，其他线程试图获得该锁。
* 中断安全代码（interrupt-safe）在中断处理程序中能避免并发访问的安全代码。
* SMP安全代码（SMP-safe）在对称多处理的机器中能避免并发访问的安全代码。
* 抢占安全代码（preempt-safe）在内核抢占时能避免并发访问的安全代码。

### 并发的原因
- 中断
- 软中断包括tasklet
- 内核抢占
- 睡眠及与用户空间的同步
- 对称多处理（Symmetrical multiprocessing）

### 加锁的一般要考虑的问题

* 最开始设计代码的时候就要考虑加锁，而不是事后才想到。
* 要给数据而不是代码加锁。
* 当锁争用严重时,加锁太粗会降低可伸缩性（scalability）。
* 当锁争用不明显时，加锁太细加大系统开销，造成浪费。
* 精髓：**Start simple and grow in complexity only as needed. Simplicity is key.**。

### 关于编写内核代码是否加锁时考虑的问题
* 这个数据是不是**全局**的？除了当前线程外，其他线程能不能访问它？
* 这个数据会不会在进程上下文和**中断上下文**中共享？它是不是要在两个不同的中断处理程序中共享？
* 进程在访问数据时可不可能被**抢占**？被调度的新程序会不会访问同一数据？
* 当进程是不是会**睡眠**（阻塞）在某些资源上，如果是，它会让共享数据处于何种状态？
* 怎样防止数据失控？
* 如果这个函数又在**另一个处理器**上被调用将会发生什么？
* 如何确保代码远离**并发**威胁？

### 加锁的简单规则
* 按顺序加锁
* 防止发生饥饿
* 不要重复请求同一个锁
* 设计应力求简单


# 原子操作
* 内核提供两组针对原子操作的接口，分别针对**整数**或**单独的位**进行操作。
* 大多数体系结构都会提供支持原子操作的简单算术指令。
* 缺少简单的原子操作指令的体系结构，也为单步执行提供了锁内存总线的指令，确保其他改变内存的操作不能同时发生。
* 在大部分体系结构上，读取一个字本身就是原子操作，即在对一个字进行写入操作期间不可能完成对该字的读取。

## 原子整数操作
* 针对**整数**的原子操作只能对`atomic_t`类型的数据进行处理。
* 不用`int`类型的原因：
  * 防止混用，该类型数据不会被传递给任何非原子函数
  * 防止编译器优化，原子操作最终接收到正确的内存地址
  * 屏蔽不同体系架构的实现差异


### Atomic Integer Methods
Atomic Integer Operation | Description
---|---
ATOMIC_INIT(int i) | At declaration, initialize to i.
int atomic_read(atomic_t *v) | Atomically read the integer value of v.
void atomic_set(atomic_t *v, int i) | Atomically set v equal to i.
void atomic_add(int i, atomic_t *v) | Atomically add i to v.
void atomic_sub(int i, atomic_t *v) | Atomically subtract i from v.
void atomic_inc(atomic_t *v) | Atomically add one to v.
void atomic_dec(atomic_t *v) | Atomically subtract one from v.
int atomic_sub_and_test(int i, atomic_t *v) | Atomically subtract i from v and return true if the result is zero; otherwise false.
int atomic_add_negative(int i, atomic_t *v) | Atomically add i to v and return true if the result is negative; otherwise false.
int atomic_add_return(int i, atomic_t *v) | Atomically add i to v and return the result.
int atomic_sub_return(int i, atomic_t *v) | Atomically subtract i from v and return the result.
int atomic_inc_return(int i, atomic_t *v) | Atomically increment v by one and return the result.
int atomic_dec_return(int i, atomic_t *v) | Atomically decrement v by one andreturn the result.
int atomic_dec_and_test(atomic_t *v) | Atomically decrement v by one and return true if zero; false otherwise.
int atomic_inc_and_test(atomic_t *v) | Atomically increment v by one and return true if the result is zero; false otherwise.

* include/linux/types.h
```c
typedef struct {
    int counter;
} atomic_t;

#ifdef CONFIG_64BIT
typedef struct {
    long counter;
} atomic64_t;
#endif
```

* arch/x86/include/asm/atomic.h
```c
/**
 * atomic_add - add integer to atomic variable
 * @i: integer value to add
 * @v: pointer of type atomic_t
 *
 * Atomically adds @i to @v.
 */
static __always_inline void atomic_add(int i, atomic_t *v)
{
    asm volatile(LOCK_PREFIX "addl %1,%0"
             : "+m" (v->counter)
             : "ir" (i));
}
```

* 所有64位体系结构都提供了`atomic64_t`类型，以及一组对应的算数操作方法。

### 原子性与顺序性的比较

#### 原子性
  * 一个字长的 *读取* 总是原子地发生，绝不可能对同一个字交错地进行写。
  * 读总是返回一个完整的字，或者在写操作之前，或者之后，绝不可能发生在写的过程中。
  * 原子性确保指令执行期间不被打断，要么全部执行完，要么根本不执行。

#### 顺序性
  * 有时代码的需求不属于原子性要求，而是顺序性要求。比如，读必须在待定的写之前发生。
  * 顺序性确保即使两条或多条指令出现在独立的执行线程中，甚至独立的处理器上，它们本该执行的顺序却依然要保持。
  * 顺序性通过屏障（barrier）来实施。

## 原子位操作
* 原子位操作是对普通内存的地址进行操作。
  * 因此没有特殊的数据类型。
  * 对位号的范围也没有限制。

### Atomic Bitwise Methods
Atomic Bitwise Operation | Description
---|---
void set_bit(int nr, void *addr) | Atomically set the nr-th bit starting from addr.
void clear_bit(int nr, void *addr) | Atomically clear the nr-th bit starting from addr.
void change_bit(int nr, void *addr) | Atomically flip the value of the nr-th bit starting from addr.
int test_and_set_bit(int nr, void *addr) | Atomically set the nr-th bit starting from addr and return the previous value.
int test_and_clear_bit(int nr, void *addr) | Atomically clear the nr-th bit starting from addr and return the previous value.
int test_and_change_bit(int nr, void *addr) | Atomically flip the nr-th bit starting from addr and return the previous value.
int test_bit(int nr, void *addr) | Atomically return the value of the nr-th bit starting from addr.

* arch/x86/include/asm/bitops.h
```c
/*
 * These have to be done with inline assembly: that way the bit-setting
 * is guaranteed to be atomic. All bit operations return 0 if the bit
 * was cleared before the operation and != 0 if it was not.
 *
 * bit 0 is the LSB of addr; bit 32 is the LSB of (addr+1).
 */

#if __GNUC__ < 4 || (__GNUC__ == 4 && __GNUC_MINOR__ < 1)
/* Technically wrong, but this avoids compilation errors on some gcc
   versions. */
#define BITOP_ADDR(x) "=m" (*(volatile long *) (x))
#else
#define BITOP_ADDR(x) "+m" (*(volatile long *) (x))
#endif

#define ADDR                BITOP_ADDR(addr)

/*
 * We do the locked ops that don't return the old value as
 * a mask operation on a byte.
 */
#define IS_IMMEDIATE(nr)        (__builtin_constant_p(nr))
#define CONST_MASK_ADDR(nr, addr)   BITOP_ADDR((void *)(addr) + ((nr)>>3))
#define CONST_MASK(nr)          (1 << ((nr) & 7))

/**
 * set_bit - Atomically set a bit in memory
 * @nr: the bit to set
 * @addr: the address to start counting from
 *
 * This function is atomic and may not be reordered.  See __set_bit()
 * if you do not require the atomic guarantees.
 *
 * Note: there are no guarantees that this function will not be reordered
 * on non x86 architectures, so if you are writing portable code,
 * make sure not to rely on its reordering guarantees.
 *
 * Note that @nr may be almost arbitrarily large; this function is not
 * restricted to acting on a single-word quantity.
 */
static __always_inline void
set_bit(long nr, volatile unsigned long *addr)
{
    if (IS_IMMEDIATE(nr)) {
        asm volatile(LOCK_PREFIX "orb %1,%0"
            : CONST_MASK_ADDR(nr, addr)
            : "iq" ((u8)CONST_MASK(nr))
            : "memory");
    } else {
        asm volatile(LOCK_PREFIX "bts %1,%0"
            : BITOP_ADDR(addr) : "Ir" (nr) : "memory");
    }
}

/**
 * __set_bit - Set a bit in memory
 * @nr: the bit to set
 * @addr: the address to start counting from
 *
 * Unlike set_bit(), this function is non-atomic and may be reordered.
 * If it's called on the same region of memory simultaneously, the effect
 * may be that only one operation succeeds.
 */
static __always_inline void __set_bit(long nr, volatile unsigned long *addr)
{
    asm volatile("bts %1,%0" : ADDR : "Ir" (nr) : "memory");
}
...__
```
* 查找第一个被设置（或未被设置）的位
```c
int find_first_bit(unsigned long *addr, unsigned int size);
int find_first_zero_bit(unsigned long *addr, unsigned int size);
```
* arch/x86/include/asm/bitops.h
```c
static __always_inline unsigned long __ffs(unsigned long word);
static __always_inline unsigned long __fls(unsigned long word);
static __always_inline int ffs(int x);
static __always_inline int fls(int x);
static __always_inline unsigned long ffz(unsigned long word);
```

### 非原子位操作
* 原子位操作前加上两个下划线前缀即位非原子位操作
* 非原子位操作不保证原子性
* 非原子位操作比对应的原子位操作可能会执行的快一些
* 非原子位操作用于已经有锁保护的场景
* 原子操作保证了操作一定会发生，这一点在顺序性开始起作用，或者操作硬件寄存器的时候很重要。


# 自旋锁（Spin Lock）

* 自旋锁最多只能被一个可执行线程持有
* 试图获得一个被持有的自旋锁的线程，会一直进行忙循环-旋转-等待锁重新可用
* 在任意时刻，自旋锁都可以防止多于一个的执行线程进入临界区
* 自旋锁使得请求它的线程在等待锁重新可用时**自旋**（特别浪费处理器时间），这是自旋锁的要点
* 故自旋锁不应被长时间地持有，其设计初衷就是**短时间内轻量级加锁**
* 与会引起睡眠的锁相比，自旋锁不会引起进程的上下文切换，只是少量的几行代码
* 持有自旋锁的时间最好小于两次上下文切换的时间

## 自旋锁方法
* 自旋锁为多处理器机器提供防止并发访问的保护机制
* 单处理器（UP）编译的时候不会加入自旋锁。如果禁止内核抢占，则在编译时自旋锁会被完全剔除
* 注意：Linux的自旋锁实现是**不可递归**的！所以千万小心不要自死锁
* 自旋锁可用于中断处理程序（中断上下文）中
* 在**中断处理程序中使用自旋锁时，一定要在获取锁之前首先禁止本地中断**（在当前处理器上的中断请求）。
  * 原因：（发生在本地的）中断处理程序可能会打断正持有锁的内核代码，又去争用同一个自旋锁，造成死锁。
  * 注意，需关闭的只是本地中断。如果中断发生在其他的处理器上，即使中断处理程序在同一锁上自旋，也不会妨碍锁的持有者（在不同处理器上）最终释放锁。
* 自旋锁使用示例
```c
DEFINE_SPINLOCK(mr_lock);
spin_lock(&mr_lock);
/* critical region ... */
spin_unlock(&mr_lock);

/*禁中断同时请求锁，之前说了,中断处理程序中使用自旋锁时，
  一定要在获取锁之前首先禁止本地中断，*/
DEFINE_SPINLOCK(mr_lock);
unsigned long flags;
spin_lock_irqsave(&mr_lock, flags);
/* critical region ... */
spin_unlock_irqrestore(&mr_lock, flags);

/*无条件禁中断或开中断版本，不提倡使用*/
DEFINE_SPINLOCK(mr_lock);
spin_lock_irq(&mr_lock);
/* critical section ... */
spin_unlock_irq(&mr_lock);
```

* 在UP系统上使用`spin_lock_irqsave()`的情形，尽管编译时抛弃掉了锁机制，但仍需关中断。为了防止中断处理程序访问共享数据，即伪并发场景。
* 加锁会禁止内核抢占`preempt_disable()`；解自旋锁会允许内核抢占`preempt_enable()`。
* 自旋锁调试配置选项
  * `CONFIG_DEBUG_SPINLOCK`：调试自旋锁
  * `CONFIG_DEBUG_LOCK_ALLOC`：全程调试自旋锁

### Spin Lock Methods
Method | Description
---|---
spin_lock() | Acquires given lock
spin_lock_irq() | Disables local interrupts and acquires given lock
spin_lock_irqsave() | Saves current state of local interrupts, disables local interrupts, and acquires given lock
spin_unlock() | Releases given lock
spin_unlock_irq() | Releases given lock and enables local interrupts
spin_unlock_irqrestore() | Releases given lock and restores local interrupts to given previous state
spin_lock_init() | Dynamically initializes given spinlock_t
spin_trylock() | Tries to acquire given lock; if unavailable, returns nonzero
spin_is_locked() | Returns nonzero if the given lock is currently acquired, otherwise it returns zero

### 自旋锁与下半部

* 如果**进程上下文**和一个**下半部**共享数据，在访问这些数据之前，需要**禁止下半部**且获得锁的使用权。
* 如果**中断上下文**和一个**下半部**共享数据，在访问数据之前，需要**禁止中断**且获得锁的使用权。
* Q：为什么共享数据除了需要锁的保护，还需要禁止中断或禁止下半部？
  * A：禁止中断或禁止下半部的原因是，中断会抢占下半部的执行，下半部会打断进程的执行。
    *  假设进程持有锁，而打断它的下半部竞争的是同一把锁，则下半部会因为无法得到进程持有的锁而挂住，造成死锁。
    *  如果进程禁止了下半部再去拿锁，则不会被下半部打断，直到它再次开启下半部。
    *  中断与下半部之间共享数据同理。
* 同类tasklet间共享数据不需加锁，因为同类tasklet不能同时运行。
* 不同类tasklet间共享数据需要加自旋锁，因为不同类tasklet可能会在别的CPU上运行。但不需要禁止下半部，因为同一处理器不会有tasklet相互抢占。
* 软中断间共享数据（无论是否同类），都需要加锁，因为同不同类softirq都有可能会在不同的CPU上运行。但不需要禁止下半部，因为同一处理器不会有softirq相互抢占。

#### 关于自旋锁与下半部的总结

数据如何共享 | 是否需要锁 | 是否需要禁止下半部/中断
---|---|---
进程与下半部间 | 是 | 禁止下半部
中断与下半部间 | 是 | 禁止中断
同类tasklet | 否 | 否
不同tasklet | 是 | 否
softirq间 | 是 | 否

* **原则**：先屏蔽干扰再加锁。

### 自旋锁与原子上下文

#### 问题 1：持有自旋锁的代码能否睡眠？
* 持有自旋锁的代码不允许睡眠，因为有可能会造成死锁。
* 睡眠会引起调度，假设线程 A 在持有锁的情况下睡眠，则其他某个线程 B 试图获得相同的锁的时候：
  * 最好的情况，线程 B 会在处理器上自旋很长时间，直到持有锁的线程 A 被唤醒并释放锁；
  * 最坏的情况：
    * 持有锁的线程 A 被移出运行队列，当唤醒的条件达成 A 被重新调度（B 即是 A 这种情况），有可能再次申请自己持有的锁，造成自死锁。
    * 能够完成唤醒条件的线程 B 也申请的同一个自旋锁，A 始终无法被唤醒，造成ABBA死锁。
* **拥有自旋锁的代码，相关处理器上的抢占会被禁止**（可见代码实现）。
* 在单处理器系统上，为避免竞态，抢占被禁止。

#### 问题 2：持有自旋锁是否一定不能放弃处理器（除了不允许被抢占，还不允许被中断）？
* 如果持锁线程与中断处理程序/下半部不共享资源，则可以被中断处理程序抢占，无需禁止中断。
* 如果持锁线程与中断处理程序/下半部，或中断处理程序之间共享资源，则需禁中断再持锁。这问题之前讨论过了。

#### 问题 3：为什么自旋锁在实现里禁止相关处理器上的内核抢占？
* 如果无共享数据，持有自旋锁是无需禁止中断的。
* 这也意味着持有自旋锁的进程有可能被中断打断，或者中断处理程序被更高级的中断打断。
* 回忆内核抢占，当中断处理程序返回的时候，是引发内核抢占的一个点。
* 内核抢占意味着会发生调度，有可能会运行别的进程，也就是说会出现“准并发”（quasi-concurrent）。
* 因此在自旋锁的实现里，无论是SMP还是UP的代码，持有锁都会调用`preempt_disable()`禁止抢占。
* 这样的话，中断处理程序返回后，内核会检查是否允许内核抢占，如果不允许则会返回原来中断执行的地方继续运行。
* 然而 *禁止内核抢占* 并不能 *阻止代码自己调用`schedule()`进行调度*，`preempt_schedule()`和`preempt_schedule_notrace()`才会检查`preemptible()`。

# 读-写自旋锁

* 读/写锁，共享/排斥锁，并发/排斥锁。
* 不能将读锁*升级*为写锁，即持有读锁还想持有写锁，会造成自死锁。
* 如果确实需要写操作，则一开始就请求写锁。
* 如不能清晰区分读写，就应使用自旋锁，而不是读/写锁。
* 即使一个线程递归地获得同一读锁也不会造成自死锁。
* 如果中断处理程序中对某共享数据只有读操作而没有写操作，可以混合使用“interrupt disabling”锁，使用`read_lock()`而不是`read_lock_irqsave()`对读进行保护。
* 有写操作的中断处理程序还是需要用`write_lock_irqsave()`禁止中断，否则中断里的读操作就有可能锁死在写锁上。
  * 假如读者正在进行操作，包含写操作的中断发生，由于读锁没全部被释放，所以写操作会自旋，然而释放读锁的操作只能等到包含写操作的中断返回后才能进行，于是发生ABBA死锁。
* 注意：使用Linux的读/写锁时，谨记该机制**对读操作比较照顾**。
  * 当读锁被持有时，写锁只能等待。并且可以有新的读者。
  * 如此时有大量的读者，有可能造成写者的饥饿。

## Reader-Writer Spin Lock Methods
Method | Description
---|---
read_lock() | Acquires given lock for reading
read_lock_irq() | Disables local interrupts and acquires given lock for reading
read_lock_irqsave() | Saves the current state of local interrupts, disables local interrupts, and acquires the given lock for reading
read_unlock() | Releases given lock for reading
read_unlock_irq() | Releases given lock and enables local interrupts
read_unlock_irqrestore() | Releases given lock and restores local interrupts to the given previous state
write_lock() | Acquires given lock for writing
write_lock_irq() | Disables local interrupts and acquires the given lock for writing
write_lock_irqsave() | Saves current state of local interrupts, disables local interrupts, and acquires the given lock for writing
write_unlock() | Releases given lock
write_unlock_irq() | Releases given lock and enables local interrupts
write_unlock_irqrestore() | Releases given lock and restores local interrupts to given previous state
write_trylock() | Tries to acquire given lock for writing; if unavailable, returns nonzero
rwlock_init() | Initializes given rwlock_t

# 信号量（Semaphores）

* Linux的信号量是一种睡眠锁。
* 信号量开销比自旋锁大。
* 争用信号量的进程会在等待锁可用时进入睡眠，所以信号量适合锁会被长时间持有的情况。
* 相反，锁被持有的时间比较短就不太适合使用信号量。因为 *睡眠 + 维护等待队列 + 唤醒* 的开销可能比占用锁的全部时间还长。
* 由于在锁争用时会引起睡眠，所以信号量只能用在进程上下文，因为在中断上下文中不能调度，所以一旦睡眠，无法唤醒。
* 占用信号量的同时不能占用自旋锁。因为等待信号量会引起睡眠，而[持有自旋锁时不允许睡眠](#自旋锁与原子上下文)。
* 注意：`down_trylock()`和`up()`可以在中断上下文中被调用，它们并不会引起睡眠。
* 不同于自旋锁，信号量不会禁止内核抢占。

## 计数信号量与二值信号量
* 信号量可以同时允许任意数量的锁持有者，而自旋锁在一个时刻最多允许一个任务持有它。
* 二值信号量（*binary semaphore*），互斥信号量（*mutex - mutual exclusion*）
* 计数信号量（*counting semaphore*）
* `P()`和`V()`, *Proberen* 和 *Verhogen*，`donw()`和`up()`。
* 原理：
  * 当信号量计数的值大于0时，任务获得锁，否则进入等待队列。
  * 任务释放锁，信号量计数的值增加。
  * 如果在该信号量上的等待队列不为空，那么处于队列中等待的任务会被唤醒，同时获得该信号量。

## Semaphore Methods
Method | Description
---|---
sema_init(struct semaphore *, int) | Initializes the dynamically created semaphore to the given count
down_interruptible (struct semaphore *) | Tries to acquire the given semaphore and enter interruptible sleep if it is contended
down(struct semaphore *) | Tries to acquire the given semaphore and enter uninterruptible sleep if it is contended
down_trylock(struct semaphore *) | Tries to acquire the given semaphore and immediately return nonzero if it is contended
up(struct semaphore *) | Releases the given semaphore and wakes a waiting task, if any

# 读-写信号量（Reader-Writer Semaphores）

* 所有读-写信号量都是互斥信号量，它们的引用计数等于**1**。
* 它们只对写者互斥，不对读者。
* 只要没有写者，并发持有读锁的读者不限。
* 只能有唯一的写者（在没有读者时）可以获得写锁。
* 所有读-写信号量的睡眠都不会被信号打断，所以只有一个版本的`down`。
* 特有操作`downgrade_write()`，动态将获取的写锁转为读锁。
* 除非代码中的读和写可以明白无误地分割开来，否则最好不使用它。

```c
/*动态版本为init_rwsem(struct rw_semaphore *sem)*/
static DECLARE_RWSEM(mr_rwsem);
/* attempt to acquire the semaphore for reading ... */
down_read(&mr_rwsem);
/* critical region (read only) ... */
/* release the semaphore */
up_read(&mr_rwsem);
/* ... */
/* attempt to acquire the semaphore for writing ... */
down_write(&mr_rwsem);
/* critical region (read and write) ... */
/* release the semaphore */
up_write(&mr_sem);
```

# 互斥体（Mutexes）

- [ ] 任何时刻中只有一个任务可以持有mutex，即mutex的使用计数永远是**1**。
- [ ] 给mutex上锁者必须负责给其再解锁。
  * 不能在一个上下文锁定一个mutex，而在另一个上下文中给它解锁。
  * 这个限制使得mutex不适合内核同用户空间复杂的同步场景。
  * 最常见的使用方式：在同一上下文中上锁和解锁。
- [ ] 不允许递归上锁和解锁。
- [ ] 持有mutex时进程不可以退出。
- [ ] mutex不能在中断或者下半部中使用，即使是`mutex_trylock()`也不行。
- [ ] mutex只能通过官方API管理：只能用提供的方法初始化，不可被拷贝，手动初始化或者重复初始化。

* 打开调试选项`CONFIG_DEBUG_MUTEXES`，内核可以采用编程方式检查和警告任何践踏其约束法则的不老实行为。

## Mutex Methods
Method | Description
---|---
mutex_lock(struct mutex *) | Locks the given mutex; sleeps if the lock is unavailable
mutex_unlock(struct mutex *) | Unlocks the given mutex
mutex_trylock(struct mutex *) | Tries to acquire the given mutex; returns one if successful and the lock is acquired and zero otherwise
mutex_is_locked (struct mutex *) | Returns one if the lock is locked and zero otherwise

## Semaphores，Mutexes 与 Spin locks

#### 如何在 Semaphores 和 Mutexes 之间作出选择
  * 除非**mutex**的某个约束妨碍你使用，否则优先使用**mutex**。
  * 只有碰到特殊场合（一般是很底层的代码）才会需要使用**semaphore**。
  * 这两者之间，首选应是**mutex**。

#### 如何在 Spin Locks 和 Mutexes/Semaphores 之间作出选择
Requirement | Recommended Lock
---|---
Low overhead | locking Spin lock is preferred.
Short lock hold time | Spin lock is preferred.
Long lock hold time | Mutex is preferred.
Need to lock from interrupt context | Spin lock is required.
Need to sleep while holding lock | Mutex is required.


# Completion变量

* **Completion变量**（completion variable）：在内核中一个任务需要发出信号通知另一任务发生了某个特定事件，使两个任务得以同步的简单方法。
* 通常用法：
  1. 将completion variable作为数据结构中的一项动态创建。
  2. 完成数据结构初始化工作的内核代码将调用`wait_for_completion()`进行等待。
  3. 初始化完成后调用`complete()`唤醒在等待的内核任务。
* 实现比较简单，主要就是由保护`completion`结构体成员的自旋锁，等待队列，和`schedule_timeout()`组成。

#### Completion Variable Methods

Method | Description
---|---
init_completion(struct completion *) | Initializes the given dynamically created completion variable
wait_for_completion(struct completion *) | Waits for the given completion variable to be signaled
complete(struct completion *) | Signals any waiting tasks to wake up


# BLK大内核锁

* BLK（大内核锁）是一个全局自旋锁，使用它主要是为了方便实现从Linux最初的SMP过渡到细粒度加所机制。
* 仅做了解，不要用。
* Sample
```c
lock_kernel();
/*
* Critical section, synchronized against all other BKL users...
* Note, you can safely sleep here and the lock will be transparently
* released. When you reschedule, the lock will be transparently
* reacquired. This implies you will not deadlock, but you still do
* not want to sleep if you need the lock to protect data here!
*/
unlock_kernel();
```

#### BKL Methods

Function | Description
---|---
lock_kernel () | Acquires the BKL.
unlock_ kernel() | Releases the BKL.
kernel_ locked() | Returns nonzero if the lock is held and zero otherwise. (UP always returns nonzero.)

# 顺序锁（Sequential Locks）

* Seq锁：对**写者**有利的锁。
* Seq锁实现原理：
  * 主要依靠一个序列计数器。
  * 当有疑义的数据写入时，会得到一个锁，并且序列值会增加。
  * 在读取数据之前和之后，序列值都被读取。
  * 如果读取的序列值相同，说明读操作过程中没有被写打断过。
  * 如果读取的值是偶数，那就表明写操作没有发生。
* 只要没有其他写者，写锁总是能够被成功获得。
* 读者不会影响写锁。
* 挂起的写者会不断地使得读操作循环，直到不再有任何写者持有锁为止。
* 选用Seq锁的理想场景：
  * 数据存在很多读者
  * 数据写者很少
  * 虽然写者很少，但是希望**写优先于读**，而且不允许读者让写者饥饿。
  * 数据很简单，如简单结构，甚至是简单的整型。在某些场合是不能使用原子变量的。

### Seq锁使用实例

##### 数据结构

* include/linux/seqlock.h
```c
...
typedef struct {
    struct seqcount seqcount;
    spinlock_t lock;
} seqlock_t;

/*
 * These macros triggered gcc-3.x compile-time problems.  We think these are
 * OK now.  Be cautious.
 */
#define __SEQLOCK_UNLOCKED(lockname)            \
    {                       \
        .seqcount = SEQCNT_ZERO(lockname),  \
        .lock = __SPIN_LOCK_UNLOCKED(lockname)  \
    }

#define seqlock_init(x)                 \
    do {                        \
        seqcount_init(&(x)->seqcount);      \
        spin_lock_init(&(x)->lock);     \
    } while (0)

#define DEFINE_SEQLOCK(x) \
        seqlock_t x = __SEQLOCK_UNLOCKED(x)
...__
```

##### 读Seq锁
* kernel/time/jiffies.c
```c
..
__cacheline_aligned_in_smp DEFINE_SEQLOCK(jiffies_lock);

#if (BITS_PER_LONG < 64)
u64 get_jiffies_64(void)
{
    unsigned long seq;
    u64 ret;

    do {
        seq = read_seqbegin(&jiffies_lock);
        ret = jiffies_64;
    } while (read_seqretry(&jiffies_lock, seq));
    return ret;
}
EXPORT_SYMBOL(get_jiffies_64);
#endif
...
```

##### 写Seq锁
* kernel/time/timekeeping.c
```c
...
/*
 * Must hold jiffies_lock
 */
void do_timer(unsigned long ticks)
{
    jiffies_64 += ticks;
    calc_global_load(ticks);
}
...
```

* kernel/time/tick-common.c
```c
...
/*
 * Periodic tick
 */
static void tick_periodic(int cpu)
{
    if (tick_do_timer_cpu == cpu) {
        write_seqlock(&jiffies_lock);  /*加Seq写锁*/

        /* Keep track of the next tick event */
        tick_next_period = ktime_add(tick_next_period, tick_period);

        do_timer(1);
        write_sequnlock(&jiffies_lock); /*解Seq写锁*/
        update_wall_time();
    }   

    update_process_times(user_mode(get_irq_regs()));
    profile_tick(CPU_PROFILING);
}
...
```

# 禁止抢占

* 内核中抢占可能在任何时刻发生。一个任务与被抢占任务可能会在同一个临界区内运行。为避免这种情况，**自旋锁总是会禁止抢占**。
* 如果自旋锁被持有，内核便不能进行抢占（之前已经讨论过[为什么自旋锁在实现里禁止相关处理器上的内核抢占？](#为什么自旋锁在实现里禁止相关处理器上的内核抢占？)）。
* SMP-safe也意味着preempt-safe，因为它们面对相同的并发问题。
* Per-CPU数据不需要自旋锁保护，因为它们对每个处理器都是唯一的。但是需要禁止抢占来保护，因为会有伪并发的场景。
* 即使是UP场景，Per-CPU数据也会以伪并发（pseudo-concurrently）的方式被访问。
* `preempt_disable()`可以嵌套调用，但调用数目需与`preempt_enable()`一一对应。
* Per-CPU使用示例
  ```c
  int cpu;
  /* disable kernel preemption and set “cpu” to the current processor */
  cpu = get_cpu();
  /* manipulate per-processor data ... */
  /* reenable kernel preemption, “cpu” can change and so is no longer valid */
  put_cpu();
  ```

#### Kernel Preemption-Related Methods

Function | Description
---|---
preempt_disable() | Disables kernel preemption by incrementing the preemption counter
preempt_enable() | Decrements the preemption counter and checks and services any pending reschedules if the count is now zero
preempt_enable_no_resched() | Enables kernel preemption but does not check for any pending reschedules
preempt_count() | Returns the preemption count

* 注意：`preempt_enable()`是触发内核抢占的一个点，因为此时内核抢占再次可用。


# 顺序和屏障（Barriers）

* **barrier**：可能重新排序和写的处理器提供了机器指令来确保顺序要求，这些确保顺序的指令称为 *屏障*。
* `volatile`关键字可以防止编译器的优化，主要体现在以下三个方面：
  * 不会从寄存器中读取变量，而是重新从内存中取。
  * 不要对变量进行各种激进的优化，甚至将变量直接消除，保证程序员写在代码中的指令，一定会被执行。
  * `volatile`变量间的操作，不会被编译器交换顺序。
* 即便是用`volatile`声明来防止编译器对变量操作的重新排序，仍然有可能会因为处理器优化pipeline的原因对指令重新排序，因此barrier仍然是很重要的。
* 对于SMP，`smp_rmb()` , `smp_wmb()` , `smp_mb()` 和 `smp_read_barrier_depends()`是常用的 *内存屏障*（如x86的`mfence`指令）；对于UP，则被定义为 *编译器屏障*，如gcc的`#define barrier() __asm__ __volatile__("": : :"memory")`。
* *内存屏障* 可以完成 *编译器屏障* 的功能，但编译器屏障要比内存屏障 **轻量**（它实际上是轻快的）得多。
* 实际上，编译器屏障几乎是空闲的，因为它只防止编译器可能的重排指令。
* 不同体系架构，屏障的实际效果差别很大。如x86不会打乱存储，`wmb()`就什么也不做。
* 为最坏的情况（即排序能力最弱的处理器）使用恰当的内存屏障，代码才能在编译时执行针对体系结构的优化。


#### Memory and Compiler Barrier Methods

Barrier | Description
---|---
rmb() | Prevents loads from being reordered across the barrier
read_barrier_depends() | Prevents data-dependent loads from being reordered across the barrier
wmb() | Prevents stores from being reordered across thebarrier
mb() | Prevents load or stores from being reordered across the barrier
smp_rmb() | Provides an rmb() on SMP, and on UP provides a barrier()
smp_read_barrier_depends() | Provides a read_barrier_depends() on SMP, and provides a barrier() on UP
smp_wmb() | Provides a wmb() on SMP, and provides a barrier() on UP
smp_mb() | Provides an mb() on SMP, and provides a barrier() on UP
barrier() | Prevents the compiler from optimizing stores or loads across the barrier
