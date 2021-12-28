# 乐观自旋锁 osq_lock

## optimistic_spin_node 结构体
* include/linux/osq_lock.h
```c
/*
 * An MCS like lock especially tailored for optimistic spinning for sleeping
 * lock implementations (mutex, rwsem, etc).
 */
struct optimistic_spin_node {
    struct optimistic_spin_node *next, *prev;
    int locked; /* 1 if lock acquired */
    int cpu; /* encoded CPU # + 1 value */
};

struct optimistic_spin_queue {
    /*
     * Stores an encoded value of the CPU # of the tail node in the queue.
     * If the queue is empty, then it's set to OSQ_UNLOCKED_VAL.
     */
    atomic_t tail;
};
```
* `locked`：**1** 表示已经获得锁
* `tail`：存储队列中末尾节点编码过的 CPU 号。如果队列为空，将其设置为`OSQ_UNLOCKED_VAL`

## Per CPU `osq_node`
* `osq_lock`是经过裁减的类 MCS 锁，特地为睡眠锁（mutex，rwsem 等）实现了乐观自旋
* `osq_node`是`struct optimistic_spin_node`类型的 Per CPU 变量，使用一个单独的 mcs node 是安全的，因为睡眠锁不会在中断上下文被调用，并且我们在自旋的时候关闭了抢占
* kernel/locking/osq_lock.c
```c
/*
 * An MCS like lock especially tailored for optimistic spinning for sleeping
 * lock implementations (mutex, rwsem, etc).
 *
 * Using a single mcs node per CPU is safe because sleeping locks should not be
 * called from interrupt context and we have preemption disabled while
 * spinning.
 */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct optimistic_spin_node, osq_node);
```

## 编码 CPU
* 当 CPU 的值为 **0** 表示没有 CPU，因此编码`osq_node.cpu`的时候把实际 CPU 号加一
```c
/*
 * We use the value 0 to represent "no CPU", thus the encoded value
 * will be the CPU number incremented by 1.
 */
static inline int encode_cpu(int cpu_nr)
{
	return cpu_nr + 1;
}

static inline int node_cpu(struct optimistic_spin_node *node)
{
	return node->cpu - 1;
}

static inline struct optimistic_spin_node *decode_cpu(int encoded_cpu_val)
{
	int cpu_nr = encoded_cpu_val - 1;

	return per_cpu_ptr(&osq_node, cpu_nr);
}
```

## osq_wait_next()
* `osq_wait_next()`目的是为`unlock()`或者`unqueue()`获得一个稳定的`node->next`指针
1. 如果我们不是队列的最后一个节点
   * 等待我们后面的节点把前驱的后继（即我们的`node->next`）设置完
   * 然后我们再把`node->next`设置为`NULL`（别忘了，我们是在`unlock()/unqueue()`的路径上，见后面精解）
   * 接着再返回指向后面的节点指针
2. 如果我们队列的最后一个节点，则我们更新`queue->tail`为`old`（`prev`），把锁往回移，此情况返回`NULL`
```c
/*
 * Get a stable @node->next pointer, either for unlock() or unqueue() purposes.
 * Can return NULL in case we were the last queued and we updated @lock instead.
 */
static inline struct optimistic_spin_node *
osq_wait_next(struct optimistic_spin_queue *lock,
	      struct optimistic_spin_node *node,
	      struct optimistic_spin_node *prev)
{
	struct optimistic_spin_node *next = NULL;
	int curr = encode_cpu(smp_processor_id());
	int old;
	// 如果有一个 prev 节点在队列中，那么 old 的值要么是 prev 节点的 CPU 编号，要么是 OSQ_UNLOCKED_VAL，
	// 因为如果我们是当前队列的最后一个，那么队列接下来会变为空。
	/*
	 * If there is a prev node in queue, then the 'old' value will be
	 * the prev node's CPU #, else it's set to OSQ_UNLOCKED_VAL since if
	 * we're currently last in queue, then the queue will then become empty.
	 */
	old = prev ? prev->cpu : OSQ_UNLOCKED_VAL;

	for (;;) {
		if (atomic_read(&lock->tail) == curr &&
		    atomic_cmpxchg_acquire(&lock->tail, curr, old) == curr) {
			/*
			 * We were the last queued, we moved @lock back. @prev
			 * will now observe @lock and will complete its
			 * unlock()/unqueue().
			 */
			break; // 如果我们是最后一个入队的，则把锁往回移。prev 将会观察到锁然后完成 unlock()/unqueue()
		}

		/*
		 * We must xchg() the @node->next value, because if we were to
		 * leave it in, a concurrent unlock()/unqueue() from
		 * @node->next might complete Step-A and think its @prev is
		 * still valid.
		 *
		 * If the concurrent unlock()/unqueue() wins the race, we'll
		 * wait for either @lock to point to us, through its Step-B, or
		 * wait for a new @node->next from its Step-C.
		 */
		if (node->next) { // 1.2
			next = xchg(&node->next, NULL); // 1.2.2
			if (next)  // 1.2.2.1
				break; // 1.2.2.1.2
		}
		// 1.2.1 或 1.2.2.1.1
		cpu_relax();
	}

	return next;
}
```
* `for`循环里的逻辑如下：
```c
1. queue->tail 是否指向我们？
   1.1 是，说明我们是最后一个节点，cmpxchg() 把 queue->tail 往前移
       1.1.1 成功，（因为我们是最后一个节点）返回 next (= NULL)
       1.1.2 失败，说明有新节点插入到我们后面，前进到 1.2
   1.2 否，node->next 是否为空？
       1.2.1 是，新节点插入还没完成，还未设置好前驱的后继，cpu_relax() 后回到 1
       1.2.2 否，xchg() 把我们的 node->next 置为 NULL，返回旧 next（应指向新插入的节点）
             1.2.2.1 旧 next 是否为空？
                     1.2.2.1.1 是，（插入的新节点 unqueue 了？我们可能又是最后一个？）回到 1
                     1.2.2.1.2 否，说明新插入的节点设置好了前驱的后继，即旧 next 已经稳定，返回旧 next
```
### 1.1.2 的 xchg()
```c
/*
 * We must xchg() the @node->next value, because if we were to
 * leave it in, a concurrent unlock()/unqueue() from
 * @node->next might complete Step-A and think its @prev is
 * still valid.
 *
 * If the concurrent unlock()/unqueue() wins the race, we'll
 * wait for either @lock to point to us, through its Step-B, or
 * wait for a new @node->next from its Step-C.
 */
if (node->next) { // 1.2
        next = xchg(&node->next, NULL); // 1.2.2
        if (next)  // 1.2.2.1
                break; // 1.2.2.1.2
}
// 1.2.1 或 1.2.2.1.1
```
* 这里假设节点 A 和 B，B 是 A 的前驱，A 在 unqueue，B 在 unqueue()/unlock() 时调用`osq_wait_next()`
* **情况 1**： 如果 A 在 B 走到`1.2`和`1.2.2`之间时，先走完 *步骤 A*，会导致 B 从`1.2.2.1.1`再次循环直到它认为`next`变得稳定
   * A 是最后一个节点，走到 *步骤 B* 让 B 成为最后一个节点
   * 或者 A 后面还有节点，走到 *步骤 C* 让 B 的`next`指向 A 后面的节点
* **情况 2**： 如果 B 先`xchg`成功，那么 A 会在 *步骤 A* 的`cmpxchg`失败，再次循环直到`prev`变得稳定
   * 如果 B 是 unlock 操作，A 会看能否在 *步骤 A* 拿到锁
   * 如果 B 是 unqueue 操作，A 会重复在 *步骤 A* 的循环
* 再看个`osq_wait_next()`里没有调`xchg()`的例子，A、B、C 三者关系为：
  ```c
  C --prev--> A --prev--> B
  ```
  * 竞争场景如下：
```c
   A                                B
   ---------------------            ---------------------
1. osq_lock()                       unlock()
2. 更新 lock->tail
3.                                  释放锁的快速路径失败，xchg() 读到的 next 为空
4.                                  进入 osq_wait_next() 等待 next，即 B，变得稳定
5. 设置 B->next 为 A
6. need_resched(), unqueue
7.                                  发现 node->next 有值，为 A，unlock 可以继续了，返回 next = A
8. Step A，把 C->next 设为 NULL
9. Step B，等待 C 变得稳定
10.Step C，断开链接，C -prev-> B, C <-next- B
11.                                 B 把锁交给 A，但 A 已经 unqueue 了
```
* 问题就出在`osq_wait_next()`没有调用`xchg(&node->next, NULL)`导致 A 在 *步骤 A* 误认为`prev`已经稳固了
* 如果`xchg`发生在 **8 之前**，那就是上面的 **情况 2**，A 需要在 *步骤 A* 等待 B 变得稳定
* 如果`xchg`发生在 **8 之后**，那就是上面的 **情况 1**，B 需要在`osq_wait_next()`等待 A 变得稳定

## osq_lock()
加锁操作的总体逻辑如下：
1. 先改`lock->tail`，因为我们是新加入的锁，所以队尾应该指向我们
2. 设置我们的前驱节点
3. 设置前驱节点的后继为我们
4. 在自己的`locked`域上自旋，直至获得锁，返回成功，或者需要被调度出去
5. 如果需要被调度出去，就进入了 unqueue 路径，要做的事情就是回退之前的操作
6. 步骤 A，等待前一个节点变得稳定，有可能还能获得锁
7. 步骤 B，等待（如果有）后一个节点变得稳定
8. 步骤 C，我们从链表上断开
9. 返回加锁失败
```c
bool osq_lock(struct optimistic_spin_queue *lock)
{
	struct optimistic_spin_node *node = this_cpu_ptr(&osq_node);
	struct optimistic_spin_node *prev, *next;
	int curr = encode_cpu(smp_processor_id());
	int old;

	node->locked = 0;
	node->next = NULL;
	node->cpu = curr;

	/*
	 * We need both ACQUIRE (pairs with corresponding RELEASE in
	 * unlock() uncontended, or fastpath) and RELEASE (to publish
	 * the node fields we just initialised) semantics when updating
	 * the lock tail.
	 */
	old = atomic_xchg(&lock->tail, curr); // 原子操作将自己添加到队尾
	if (old == OSQ_UNLOCKED_VAL) // 原来队列是否为空？
		return true; // 是空的，我们获得锁。注意，无需专门设置 node->locked 为 1
	// 如果队列不为空
	prev = decode_cpu(old); // 得到前一个节点
	node->prev = prev; // 设置当前节点的前驱节点

	/*
	 * osq_lock()			unqueue
	 *
	 * node->prev = prev		osq_wait_next()
	 * WMB				MB
	 * prev->next = node		next->prev = prev // unqueue-C
	 *
	 * Here 'node->prev' and 'next->prev' are the same variable and we need
	 * to ensure these stores happen in-order to avoid corrupting the list.
	 */
	smp_wmb(); // 屏障1)

	WRITE_ONCE(prev->next, node); // 设置前驱节点的后继为当前节点
	// 通常 prev 在上面的写操作之后不能再去碰它；因为在那一刻解锁操作能从栈中处理和清扫元素节点。
	// 然而，因为我们的节点是静态存储的 per-cpu 变量，这就保证了他们的存在 -- 这允许我们应用 cmpxchg 来尝试撤销入队
	/*
	 * Normally @prev is untouchable after the above store; because at that
	 * moment unlock can proceed and wipe the node element from stack.
	 *
	 * However, since our nodes are static per-cpu storage, we're
	 * guaranteed their existence -- this allows us to apply
	 * cmpxchg in an attempt to undo our queueing.
	 */
	// 自旋等待 locked 变为 1，获得锁，或争用锁被取消。注意，如果 IPI 是用 monitor-wait 来实现的，
	// 那么来自它的 need_resched() 会唤醒 smp_cond_load_relaxed()
	/*
	 * Wait to acquire the lock or cancellation. Note that need_resched()
	 * will come with an IPI, which will wake smp_cond_load_relaxed() if it
	 * is implemented with a monitor-wait. vcpu_is_preempted() relies on
	 * polling, be careful.
	 */
	if (smp_cond_load_relaxed(&node->locked, VAL || need_resched() ||
				  vcpu_is_preempted(node_cpu(node->prev))))
		return true; // 加入队列
	// 如果是因为 need_resched() 而退出自旋，则会走到 unqueue() 路径
	/* unqueue */
	/*
	 * Step - A  -- stabilize @prev
	 *
	 * Undo our @prev->next assignment; this will make @prev's
	 * unlock()/unqueue() wait for a next pointer since @lock points to us
	 * (or later).
	 */
	// 步骤 A -- prev 变得稳定。撤销我们给 prev->next 的赋值；
	// 这会使前一个节点的 unlock()/unqueue() 操作等待 next 指针，因为当前锁正（或随后）指向我们
	for (;;) {
		/*
		 * cpu_relax() below implies a compiler barrier which would
		 * prevent this comparison being optimized away.
		 */
		// data_race()给一些工具提示此处有蓄意的数据竞争。如果此前我们把 prev->next 设成 node,那我们要回退该操作
		if (data_race(prev->next) == node &&
		    cmpxchg(&prev->next, node, NULL) == node) // 如果是我们之前让前驱的后继指向我们，现在让它指向 NULL
			break; // 回退成功，退出循环
		// 这里导致的回退失败的唯一可能结果是与 unlock() 竞争失败的结果，unlock() 提前把 prev->next 设为 NULL
		// 在这种情况下我们应该可以观察到 node->locked 变成 1
		/*
		 * We can only fail the cmpxchg() racing against an unlock(),
		 * in which case we should observe @node->locked becoming
		 * true.
		 */
		if (smp_load_acquire(&node->locked))
			return true; // 与 unlock() 竞争导致回退失败，但它却把锁传给了我们，这种情况下我们幸运地获得锁
		// 如果我们的前一个节点也在 unqueue() 且在步骤 B 的 osq_wait_next() 的 1.2.2 设置了自己的 next 为 NULL，会导致以上两个条件都失败
		cpu_relax();
		// 前一节点并发的 unqueue() 在 C.1 处 next 就是我们，在那里我们的 prev 被设置为前前节点，C.2 处把前前节点的 next 设置为我们
		/*
		 * Or we race against a concurrent unqueue()'s step-B, in which
		 * case its step-C will write us a new @node->prev pointer.
		 */
		prev = READ_ONCE(node->prev);
	}
	// 步骤 B -- next 稳定。与 unlock 相似，
	// 1. 有新的节点插入到了我们后面，等待它把 node->next 指向它自己，这种情况下返回的 next 指向新插入的节点
	// 2. 或者把锁从 node 移回到 prev（lock->tail 的指向从 node 变为 prev），返回 next 值为 NULL
	/*
	 * Step - B -- stabilize @next
	 *
	 * Similar to unlock(), wait for @node->next or move @lock from @node
	 * back to @prev.
	 */

	next = osq_wait_next(lock, node, prev);
	if (!next) // osq_wait_next() 返回空，表明我们是最后一个节点且回退成功
		return false; // lock->tail 回退成功，返回争锁失败，别忘了我们是在 unqueue() 路径
	// 步骤 C -- 断开链接。prev 是稳定的，因为它正等待着 prev->next 被设置，next 也是稳定的，
	// 因为我们的 next 指针指向 NULL 且它会在步骤 A 等待。
	/*
	 * Step - C -- unlink
	 *
	 * @prev is stable because its still waiting for a new @prev->next
	 * pointer, @next is stable because our @node->next pointer is NULL and
	 * it will wait in Step-A.
	 */
	// 能走到这里，是因为我们后面还有新插入的节点，next 指向它
	WRITE_ONCE(next->prev, prev); // C.1 设置新插入节点的前驱为我们的前驱
	WRITE_ONCE(prev->next, next); // C.2 设置我们的前驱的后继为新插入的节点
	// 我们从链表上断开了，unqueue() 结束，加锁失败
	return false;
}
```
* `smp_cond_load_relaxed(ptr, cond_expr)`语义为，自旋直至条件表达式`cond_expr`成立
* `smp_load_acquire(p)`语义为带屏障的方式读出地址`p`中的内容
* **C.1**：我们的下一个节点是稳固的，即便最坏的情况也是，因为我们的`next`是`NULL`，下一个节点会在 *步骤 A* 等待我们在这里设置它的`prev`
* **C.2**：我们的前一个节点是稳固的，即便最坏的情况也是，前一个节点在`osq_wait_next()`等待我们设置它的`next`

### 屏障1)
```c
node->prev = prev; // 设置当前节点的前驱节点
/*
 * osq_lock()			unqueue
 *
 * node->prev = prev		osq_wait_next()
 * WMB				MB
 * prev->next = node		next->prev = prev // unqueue-C
 *
 * Here 'node->prev' and 'next->prev' are the same variable and we need
 * to ensure these stores happen in-order to avoid corrupting the list.
 */
smp_wmb(); // 屏障1)

WRITE_ONCE(prev->next, node); // 设置前驱节点的后继为当前节点
```
* 这里有个竞争条件需要 **写屏障**`smp_wmb()` 来保证不会出现 store-store 引起的乱序
* 假设有节点 A、B、C，B 是 A 的前驱节点，C 是 B 的前驱节点；A 在通过`osq_lock()`获取锁，B 在并发地 unqueue，
  * A 节点发生乱序，先执行了`prev->next = node`，这会导致 B 在其 *步骤 B* 的`osq_wait_next()`能走到`1.2.2.1.2`，返回一个指向 A 节点的指针
  * 随后在 B 节点的 *步骤 C* 中，将 A 节点的`prev`指向 C，这没什么问题
  * 问题出在，A 节点随后再执行乱序的`node->prev = prev`语句，把 A 节点的`prev`又指回了 B，然而 B 已经 unqueue 了，并不应该出现在链表上
* 该屏障保证，当 B 节点看到自己的`next`被 A 设置时，A 已经将其`prev`指针指向自己，从而避免了链表出现错误

## osq_unlock()
* 对于没有竞争者的快速路径，如果锁队列的对尾`lock->tail`正指向我，那么将其设为`OSQ_UNLOCKED_VAL`即可释放锁
* 如果锁是有竞争者的
  * `xchg()`将本节点的`next`域置为`NULL`，脱离队列，并且用本地变量`next`记录下下一个节点
  * 如果下一个节点 **不为空**，将其锁字`locked`域置为`1`表示把锁递交给它，然后返回
  * 如果下一个节点 **为空**，但前面的快速路径释放锁是失败的，说明下一个节点正处于不稳定的状态，调用`osq_wait_next()`等它变得稳定
    * 如果稳定后下一个节点不为空，再将其锁字`locked`域置为`1`，把锁递交给它
    * 如果稳定后下一个节点为空，说明锁队列已经空了，不需要再做什么了
* 可见释放锁如需传递锁，只需设置下一个节点的锁字和并设置本节点的`next`为`NULL`，不需要操作下一个节点的`prev`和`next`指针
```c
void osq_unlock(struct optimistic_spin_queue *lock)
{
	struct optimistic_spin_node *node, *next;
	int curr = encode_cpu(smp_processor_id());

	/*
	 * Fast path for the uncontended case.
	 */
	if (likely(atomic_cmpxchg_release(&lock->tail, curr,
					  OSQ_UNLOCKED_VAL) == curr))
		return; // 队列上没有其他锁的情况，设置 tail 为 OSQ_UNLOCKED_VAL 即可

	/*
	 * Second most likely case.
	 */
	node = this_cpu_ptr(&osq_node);
	next = xchg(&node->next, NULL);
	if (next) {
		WRITE_ONCE(next->locked, 1);
		return;
	}
	// 我们是锁的持有者，所以可以认为 prev 指针是 NULL
	next = osq_wait_next(lock, node, NULL);
	if (next)
		WRITE_ONCE(next->locked, 1);
}
```
