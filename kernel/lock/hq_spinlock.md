
# Hierarchical Queued (HQ) Spinlock

* **Hierarchical Queued (HQ) Spinlock**：NUMA 感知自旋锁，旨在减少高竞争场景下的跨 NUMA 节点缓存行流量，从而提高锁吞吐量。其设计结合了 **Cohort Locking**（Dave Dice）和 Linux 内核原生 **Queued Spinlock (qspinlock)** 的思想，采用两级队列结构，并支持动态模式切换。

## 1. 核心设计思想

- **两级队列**  
  每个 NUMA 节点维护一个本地 FIFO 队列（`numa_queue`），存放该节点上的等待线程。所有非空的节点队列再通过一个全局的 **节点间链表** 串联起来，保证节点之间的 FIFO 顺序。

- **动态元数据绑定**  
  为了避免为每个锁静态预分配 NUMA 队列（开销过大），HQ spinlock 按需分配 `lock_metadata` 结构，通过锁的指针哈希值索引，实现锁与 NUMA 队列的动态绑定。

- **两阶段交接**  
  - **本地交接**：在同一 NUMA 节点的等待队列内，使用 MCS 锁机制交接。  
  - **远程交接**：当本地队列为空时，沿节点间链表找到下一个非空节点队列，将锁交接给该节点的队首线程。

- **可切换模式**  
  锁可以在普通 `qspinlock` 模式和 NUMA‑aware 模式之间动态切换。低竞争时使用普通模式以降低开销，高竞争时自动升级为 HQ 模式。

## 2. 关键数据结构

```c
struct numa_qnode {
    struct mcs_spinlock mcs;   // MCS 节点，内嵌 locked、next
    u16 lock_id;               // 锁的哈希 ID（元数据索引）
    u16 wrong_fallback_tail;   // 模式切换时的辅助字段
    u16 general_handoffs;      // 全局交接计数（用于公平性阈值）
    u16 numa_node;             // 所属 NUMA 节点（1..N）
};

struct numa_queue {
    struct numa_qnode *head;   // 该节点队列的队首
    u64 seq_counter_tail;      // 低位：队列尾索引；高位：元数据序列号
    u16 next_node;             // 节点间链表中的下一个节点 ID
    u16 prev_node;             // 节点间链表中的上一个节点 ID
    u16 handoffs_not_head;     // 远程交接计数（防饥饿）
};

struct lock_metadata {
    atomic_t seq_counter;       // 序列号，标识元数据的“代”
    struct qspinlock *lock_ptr; // 指向拥有该元数据的锁
    union {
        u32 nodes_tail;         // 编码节点链表的头/尾
        struct {
            u16 tail_node;
            u16 head_node;
        };
    };
};
```

- **全局元数据池**：`meta_pool[LOCK_ID_MAX]`，每个锁通过 `hash_ptr(lock, LOCK_ID_BITS)` 获得 `lock_id`，尝试占用对应的 `lock_metadata`。
- **每节点队列表**：`queue_table[MAX_NUMNODES][LOCK_ID_MAX]`，为每个可能的锁在每个 NUMA 节点上预分配一个 `numa_queue`。

## 3. 动态元数据管理（锁绑定）

- **`grab_lock_meta()`**：尝试将 `lock_metadata` 绑定到当前锁。若 `meta->lock_ptr` 为 NULL，则通过 `cmpxchg_acquire` 将其指向当前锁；若已指向同一锁，则共享；若指向其他锁（哈希冲突），则放弃 NUMA 模式，回退到普通 qspinlock。
- **`setup_lock_mode()`**：根据锁当前状态，决定切换到 HQ 模式或保持普通模式。成功绑定后，通过 `set_mode_hqlock()` 在锁的原子值中设置 `_Q_LOCKTYPE_HQ` 和 `_Q_LOCK_MODE_QSPINLOCK_VAL` 标志，告知后续等待者使用 NUMA 感知路径。
- **`release_lock_meta()`**：当锁完全释放且所有节点队列都为空时，清除元数据中的 `lock_ptr`，使该元数据可被其他锁重用。

## 4. 加锁慢路径流程（`queued_spin_lock_slowpath` 的 HQ 扩展）

当锁处于 HQ 模式时，慢路径调用 `hqlock_xchg_tail()` 代替普通的 `xchg_tail()`：

```c
old = hqlock_xchg_tail(lock, tail, node, &numa_awareness_on);
```

- **若锁处于普通模式**：调用 `try_update_tail_qspinlock_mode()`，执行标准 qspinlock 的 `xchg_tail`，若发现 `_Q_LOCK_INVALID_TAIL`（表示正在切换模式），则进行补偿逻辑。
- **若锁处于 HQ 模式或无模式**：调用 `try_update_tail_hqlock_mode()`：
  1. 通过 `setup_lock_mode()` 获得或建立元数据绑定，并读取元数据的 `seq_counter`。
  2. 找到当前 CPU 对应的 `numa_queue`（通过 `lock_id` 和 `numa_node`）。
  3. 使用 `try_cmpxchg` 更新该队列的 `seq_counter_tail`（低位为队列尾索引，高位为 `seq_counter`），若失败说明元数据已被释放或切换，则重试。
  4. 如果是该节点队列的第一个等待者，则调用 `append_node_queue()` 将本节点插入全局节点链表，并返回之前的尾节点（用于后续 MCS 链接）。

- **特殊情况**：当 `old == Q_NEW_NODE_QUEUE` 且 `numa_awareness_on` 为真时，跳过 MCS 自旋，直接进入头部等待（因为第一个入队者需要全局自旋）。

## 5. 解锁与交接流程（`hqlock_clear_tail_handoff`）

锁释放时，当前锁持有者调用 `hqlock_try_clear_tail()` 判断下一步动作：

- **本地队列非空**：返回 `HQLOCK_HANDOFF_LOCAL`，随后 `hqlock_handoff()` 执行本地交接：将锁传给本节点队列的下一个 `numa_qnode`，并增加 `general_handoffs` 计数器。
- **本地队列空，但有其他节点队列**：
  - 若节点链表中有后继节点，返回后继节点 ID（>0），执行远程交接。
  - 若当前节点是链表尾且链表头不是自身（即只有自己一个节点但可能有新节点正在加入），返回 `HQLOCK_HANDOFF_REMOTE_HEAD`，表示需要交接给链表头节点。
- **若所有队列都空**：调用 `release_lock_meta()` 释放元数据，并将锁切回普通模式（清除 HQ 标志）。

**交接细节**：
- **本地交接** (`handoff_local`)：更新 `queue->head` 为下一个 MCS 节点，调用 `arch_mcs_spin_unlock_contended()` 唤醒下一个线程。同时将 `general_handoffs` 传递给后继。
- **远程交接** (`handoff_remote`)：通过 `lock_metadata` 找到链表头节点（或指定节点），获取其 `head` 线程，直接唤醒它。同时更新 `handoffs_not_head` 计数器以防止头部饥饿。

## 6. 公平性与饥饿避免

- **交接阈值**：每个锁维护一个 `general_handoffs` 计数器，记录该锁已被交接的次数。当该值超过 `hqlock_fairness_threshold`（可配置）时，即使本地队列非空，也强制进行一次远程交接，将锁交给其他节点上的等待者。
- **头部节点保护**：节点链表中的头部节点如果长期未被服务，其 `handoffs_not_head` 会递增。当该值达到在线节点数时，下次远程交接会强制交给头部节点，防止尾部节点“插队”。

## 7. 动态模式切换

- **进入 HQ 模式**：在锁释放时，如果 `determine_contention_qspinlock_mode()` 检测到高竞争（`general_handoffs` 超过预设阈值 `hqlock_general_handoffs_turn_numa`），则在 `low_contention_try_clear_tail()` 中设置 `_Q_LOCK_INVALID_TAIL`，使后续加锁者进入 HQ 路径，并尝试绑定元数据。
- **退出 HQ 模式**：当锁完全释放且所有队列为空时，`release_lock_meta()` 会清除锁中的 HQ 标志，将锁恢复为普通 qspinlock。

## 8. 初始化与配置

- **内核配置**：`CONFIG_HQSPINLOCKS` 依赖 `NUMA` 和 `QUEUED_SPINLOCKS`，不支持大端架构。
- **启动参数**：`numa_spinlock=auto/on/off` 控制是否启用 HQ spinlock。
- **元数据分配**：在 `hq_configure_spin_lock_slowpath()` 中调用 `hqlock_alloc_global_queues()`，使用 `memblock_alloc_range_nid()` 为元数据池和每节点队列表预留内存。
- **锁初始化**：提供 `spin_lock_init_hq()` / `DEFINE_SPINLOCK_HQ()` 宏，初始化的锁会带有 `_Q_LOCKTYPE_HQ | _Q_LOCK_MODE_QSPINLOCK_VAL` 标志，表示其支持 HQ 模式但初始处于普通模式。

## 9. 与现有 CNA 锁的差异

- **CNA (Compact NUMA-aware)** 使用简单的双队列（主队列+辅助队列），HQ spinlock 则为每个 NUMA 节点维护独立队列，形成更精细的层次结构。
- **元数据动态绑定**：CNA 需要为每个锁预留额外的 `cna_node` 空间（通过 `struct qnode` 扩充），而 HQ 通过全局池按需绑定，理论上可以服务更多锁。
- **交接阈值**：CNA 通过 `intra_node_threshold` 控制是否切回主队列，HQ 使用 `general_handoffs` 和 `handoffs_not_head` 双重机制。

## 总结

Hierarchical Queued Spinlock 通过两级队列、动态元数据绑定和智能交接策略，在保持 qspinlock 低开销的同时，显著减少了高竞争下的跨 NUMA 缓存一致性流量。其设计兼顾了**局部性**（优先本地交接）、**公平性**（阈值强制远程交接）和**可伸缩性**（按需元数据分配），是对 Linux 内核自旋锁机制的一次重要 NUMA 优化尝试。

## 函数解析

### `set_lock_mode()`

```c
/*
 * set_lock_mode - 尝试更新锁的原子值，以设置 HQ 模式或普通 qspinlock 模式
 * @lock:      目标锁指针
 * @__val:     锁当前的原子值（传入前由调用者读取）
 * @lock_id:   锁的元数据索引（若 lock_id == LOCK_ID_NONE 表示要设置为普通模式）
 *
 * 返回值：
 *   LOCK_MODE_HQLOCK    - 成功设置为 HQ 模式
 *   LOCK_MODE_QSPINLOCK - 成功设置为普通 qspinlock 模式
 *   LOCK_NO_MODE        - 设置失败（因锁值在循环中被其他 CPU 改变，或 pending 位未清除）
 *
 * 核心逻辑：
 *   1. 等待锁的 pending 位被清除（如果 pending 被设置，不能直接修改锁值，
 *      否则可能破坏锁状态）。
 *   2. 根据 lock_id 决定要写入的新值：若启用 HQ 模式则保留锁尾并设置模式位；
 *      若关闭 HQ 模式则清除 _Q_LOCK_INVALID_TAIL 并清除模式位。
 *   3. 使用原子 CAS 尝试更新锁值，成功后返回对应模式。
 *   4. 若 CAS 失败或检测到锁模式已被其他 CPU 设置，则返回 LOCK_NO_MODE 让调用者重试。
 *
 * 注：该函数在 setup_lock_mode() 中被调用，用于实际发布模式切换。
 */
static inline hqlock_mode_t set_lock_mode(struct qspinlock *lock, int __val, u16 lock_id)
{
    u32 val = (u32)__val;
    u32 new_val = 0;
    u32 lock_mode = encode_lock_mode(lock_id);

    /* 循环直到成功更新锁值，或遇到无法继续的情况 */
    while (!(val & _Q_LOCK_MODE_MASK)) {
        /*
         * 等待 pending 位被清除。
         * 如果锁的 pending 位被设置（表示有线程正在通过 pending 快速路径获取锁），
         * 我们不能直接修改锁值，否则可能清除 pending 位并破坏锁状态。
         * 必须等到 pending 位被清除后，才能安全地修改锁的模式位。
         */
        if (val & _Q_PENDING_VAL) {
            /* 条件读取：等待 pending 位被清除，同时可能锁模式已被其他 CPU 设置 */
            val = atomic_cond_read_relaxed(&lock->val, !(VAL & _Q_PENDING_VAL));

            /* 如果在等待期间锁模式已被设置（其他 CPU 抢先），则放弃本次尝试 */
            if (val & _Q_LOCK_MODE_MASK)
                return LOCK_NO_MODE;
        }

        /*
         * 构造新的锁值：
         * - 如果要启用 NUMA 感知（lock_id != LOCK_ID_NONE）：
         *   保留锁的原有值（包括 tail、locked、pending），并加上 HQ 模式标志。
         * - 如果要切换回普通模式（lock_id == LOCK_ID_NONE）：
         *   清除 _Q_LOCK_INVALID_TAIL 标志（如果有），并清除模式位，
         *   同时保留 locked/pending/tail 原有值。
         */
        if (lock_id != LOCK_ID_NONE)
            new_val = val | lock_mode;
        else
            new_val = (val & ~_Q_LOCK_INVALID_TAIL) | lock_mode;

        /*
         * 如果要启用 HQ 模式，需要确保元数据中的 seq_counter 更新（在分配元数据时已递增）
         * 已经被全局可见，之后才发布锁的模式位。
         * 此写屏障与 setup_lock_mode() 中的 smp_rmb() 配对，保证其他 CPU
         * 在读取到锁模式为 HQ 之前，能够看到 seq_counter 的正确值。
         */
        if (lock_id != LOCK_ID_NONE)
            smp_wmb();

        /* 原子尝试更新锁的值，使用 relaxed 顺序（因为成功后的顺序由其他操作保证） */
        bool updated = atomic_try_cmpxchg_relaxed(&lock->val, &val, new_val);

        if (updated) {
            /* 成功更新，根据 lock_id 返回对应的模式 */
            return (lock_id == LOCK_ID_NONE) ?
                LOCK_MODE_QSPINLOCK : LOCK_MODE_HQLOCK;
        }

        /*
         * CAS 失败，说明在循环过程中锁的 val 已被其他 CPU 修改。
         * 此时 val 已被 atomic_try_cmpxchg_relaxed 更新为最新的锁值，
         * 继续循环，重新检查条件。
         */
    }

    /* 如果退出循环时 val 已经包含 _Q_LOCK_MODE_MASK，说明模式已被其他 CPU 设置 */
    return LOCK_NO_MODE;
}
```

#### 关键设计要点说明

1. **pending 位等待**  
   当锁的 pending 位被设置时，表示有一个线程正在通过“pending 快速路径”获取锁（该路径在锁仅有锁持有者而无等待者时使用）。此时如果直接修改锁的 `val`（例如设置模式位），可能会错误地清除 pending 位或导致锁状态不一致。因此必须等待 pending 位被清除后才能安全更新。

2. **_Q_LOCK_INVALID_TAIL 的处理**  
   当锁要从普通模式切换到 HQ 模式时，之前可能有一些线程在普通模式下已经设置了 `_Q_LOCK_INVALID_TAIL` 标志（该标志表示“tail 无效，请走 HQ 路径”）。切换回普通模式时，需要清除该标志，以便后续普通模式的 `xchg_tail` 能够正常工作。

3. **内存屏障的放置**  
   `smp_wmb()` 仅在启用 HQ 模式时使用，确保 `seq_counter` 的写入（在 `grab_lock_meta` 中完成）在锁的模式位发布之前对其它 CPU 可见。这是因为其他 CPU 在看到锁模式变为 HQ 后，会立即读取元数据的 `seq_counter` 来验证队列的有效性；如果 `seq_counter` 尚未传播，可能导致错误的验证失败。

4. **CAS 失败处理**  
   `atomic_try_cmpxchg_relaxed` 在失败时会自动将 `val` 更新为当前锁值，因此循环可以继续。如果循环条件 `!(val & _Q_LOCK_MODE_MASK)` 不满足（即锁模式已经被其他 CPU 设置），则函数返回 `LOCK_NO_MODE`，让调用者（`setup_lock_mode`）重试或回退。


### `setup_lock_mode()`

```c
/*
 * setup_lock_mode - 尝试将锁设置为 HQ 模式（NUMA 感知）或回退到普通 qspinlock 模式
 * @lock:          目标锁指针
 * @lock_id:       通过哈希计算的锁元数据索引（0 到 LOCK_ID_MAX-1）
 * @meta_seq_counter: 输出参数，返回元数据当前的序列号（用于后续队列操作）
 *
 * 返回值：
 *   LOCK_MODE_HQLOCK    - 成功进入 NUMA 感知模式
 *   LOCK_MODE_QSPINLOCK - 使用普通 qspinlock 模式（因哈希冲突或配置原因）
 *   LOCK_NO_MODE        - 临时状态，需要重试（实际循环中不会返回此值）
 *
 * 核心逻辑：
 *   1. 如果锁已经处于 HQ 模式，验证元数据仍归属于本锁（通过 seq_counter 检查），
 *      若验证通过则直接返回 HQ 模式。
 *   2. 如果锁处于普通 qspinlock 模式，直接返回该模式。
 *   3. 否则锁处于“无模式”状态（_Q_LOCK_MODE_MASK == 0），此时需要竞争元数据
 *      所有权，并尝试将锁切换到 HQ 模式；若元数据冲突则回退到普通模式。
 */
static inline
hqlock_mode_t setup_lock_mode(struct qspinlock *lock, u16 lock_id, u32 *meta_seq_counter)
{
    hqlock_mode_t mode;

    do {
        enum meta_status status;
        int val = atomic_read(&lock->val);

        /* ---------- 情况1：锁已经处于 NUMA 感知模式 ---------- */
        if (is_mode_hqlock(val)) {
            struct lock_metadata *lock_meta = get_meta(lock_id);
            /*
             * 锁当前处于 LOCK_MODE_HQLOCK，但我们必须确保关联的元数据
             * 没有被其他锁占用（即没有发生哈希冲突）。
             *
             * 可能发生的几种情况：
             * [情况1] 另一个锁正在使用同一个元数据（哈希冲突）
             * [情况2] 元数据在刚被释放后又重新分配给了本锁（同一锁的不同生命周期）
             * [情况3] 元数据空闲，无人使用
             * [情况4] 锁模式已被切换回 LOCK_MODE_QSPINLOCK
             */

            /* 读取元数据的当前序列号（该序列号在元数据分配时递增） */
            int seq_counter = atomic_read(&lock_meta->seq_counter);

            /*
             * 需要保证“seq_counter”的读取发生在“lock->val”再次读取之前。
             * 否则，如果另一个 CPU 刚刚释放了元数据并清除了锁的 HQ 标志，
             * 我们可能读到旧的 seq_counter 但新的锁值，造成错误判断。
             * 此读屏障与 set_lock_mode() 中的 smp_wmb() 配对，
             * 确保在 HQ 标志发布之前，seq_counter 的更新已经全局可见。
             */
            smp_rmb();
            val = atomic_read(&lock->val);

            /* 再次检查锁是否仍处于 HQ 模式 */
            if (is_mode_hqlock(val)) {
                /* 模式仍然有效，将序列号返回给调用者（用于后续入队验证） */
                *meta_seq_counter = (u32)seq_counter;
                return LOCK_MODE_HQLOCK;
            }
            /*
             * 如果走到这里，说明锁模式已经不是 HQ 模式了，可能有两种情况：
             *   1. 元数据空闲，无人使用，需要重新尝试获取所有权并发布 HQ 模式。
             *   2. 锁已经切换回 LOCK_MODE_QSPINLOCK 模式。
             * 无论哪种，都跳出当前分支，继续循环（重新读取 lock->val）。
             */
            continue;
        }
        /* ---------- 情况2：锁已经处于普通 qspinlock 模式 ---------- */
        else if (is_mode_qspinlock(val)) {
            return LOCK_MODE_QSPINLOCK;
        }

        /* ---------- 情况3：锁处于“无模式”状态（尚未决定使用哪种模式） ---------- */
        /*
         * 尝试获取元数据的“弱”所有权（即临时绑定）。
         * grab_lock_meta 可能返回：
         *   META_GRABBED   - 成功获取所有权（此前 lock_ptr 为 NULL）
         *   META_SHARED    - 元数据已被同一锁的其他竞争者占用（lock_ptr == lock）
         *   META_CONFLICT  - 元数据被其他锁占用（哈希冲突）
         */
        status = grab_lock_meta(lock, lock_id, meta_seq_counter);
        if (status == META_SHARED) {
            /*
             * 已有其他线程为本锁占用了元数据，并且可能正在尝试发布 HQ 模式。
             * 我们可以快速循环几次，等待锁的 val 中出现 HQ 标志，
             * 或者最终该元数据被释放（极少情况）。直接 continue 重试即可。
             */
            continue;
        }

        /* 根据元数据获取结果，尝试设置锁的模式（HQ 或回退到 qspinlock） */
        if (status == META_GRABBED)
            mode = set_mode_hqlock(lock, val, lock_id);
        else if (status == META_CONFLICT)
            mode = set_mode_qspinlock(lock, val);
        else
            BUG_ON(1);   /* 未预期的状态 */

        /*
         * 如果我们成功获取了元数据（status == META_GRABBED），但最终
         * set_mode_hqlock() 未能将锁切换到 HQ 模式（例如因为锁值在此期间被改变，
         * 或由于竞争导致模式设置失败），则我们需要释放刚刚占用的元数据，
         * 以便其他锁可以使用它。
         */
        if (status == META_GRABBED && mode != LOCK_MODE_HQLOCK) {
            smp_store_release(&meta_pool[lock_id].lock_ptr, NULL);
#ifdef CONFIG_HQSPINLOCKS_DEBUG
            atomic_dec(&cur_buckets_in_use);
#endif
        }
        /* 如果 mode == LOCK_NO_MODE，循环继续尝试；否则退出循环 */
    } while (mode == LOCK_NO_MODE);

    return mode;
}
```

#### 关键设计要点说明

1. **元数据序列号（seq_counter）的作用**  
   每次元数据被重新分配（绑定到一个新锁）时，`seq_counter` 会原子递增。每个 NUMA 节点的队列头中保存了该序列号。当线程入队时，会检查队列中保存的序列号是否与当前元数据的序列号匹配，若不匹配则说明元数据已被释放并重新分配给其他锁，此时必须重新执行绑定流程，从而避免了使用过期的队列结构。

2. **内存屏障配对**  
   - `smp_rmb()` 在 `setup_lock_mode` 中用于确保读取 `seq_counter` 之后才读取 `lock->val`，防止读到过期的 HQ 标志。  
   - `set_lock_mode()` 中的 `smp_wmb()` 在更新 `lock->val` 之前，确保元数据中 `seq_counter` 的递增已经全局可见。两者配合保证了元数据代次与锁模式的一致性。

3. **META_SHARED 的快速路径**  
   当 `grab_lock_meta` 返回 `META_SHARED` 时，说明元数据已经被当前锁的其他等待者占用，且很可能锁的 `val` 即将变为 HQ 模式。此时无需再竞争元数据，直接重试读取 `lock->val` 即可，减少了原子操作开销。

4. **回退到普通 qspinlock 的条件**  
   元数据哈希冲突（`META_CONFLICT`）或 `set_mode_hqlock` 失败时，会调用 `set_mode_qspinlock` 将锁切换到普通模式。此后，该锁的生命周期内将不再尝试 NUMA 感知，直到锁完全释放后重新进入“无模式”状态。

5. **并发释放元数据的处理**  
   在 `status == META_GRABBED` 但未能成功设置 HQ 模式时，代码通过 `smp_store_release` 将 `lock_ptr` 置为 NULL，并递减使用计数。这保证了后续竞争者能够重新获取该元数据，不会因为残留指针而导致死锁。

### `try_update_tail_hqlock_mode()`

```c
/*
 * try_update_tail_hqlock_mode - 在 NUMA 感知（HQ）模式下将当前线程入队到本地队列
 * @lock:       目标锁指针
 * @lock_id:    锁的元数据索引（哈希值）
 * @qnode:      当前线程的 numa_qnode（内嵌在 mcs_spinlock 中）
 * @tail:       编码了 CPU 和节点索引的 tail 值（由 encode_tail 生成）
 * @next_tail:  输入/输出参数，可能包含之前失败的普通模式 tail 值
 * @old_tail:   输出参数，返回本地队列之前的状态（用于链接 MCS 节点）
 *
 * 返回值：
 *   true  - 成功将当前线程加入本地 NUMA 队列（包括可能是第一个入队者）
 *   false - 需要回退到普通 qspinlock 模式（锁已切换到该模式）
 *
 * 核心逻辑：
 *   1. 调用 setup_lock_mode() 确保锁处于 HQ 模式，并获得元数据序列号。
 *   2. 获取当前 CPU 所属 NUMA 节点的本地队列（numa_queue）。
 *   3. 通过 CAS 更新本地队列的 seq_counter_tail，将当前线程的 tail 索引写入队列尾。
 *      CAS 同时验证队列的序列号与元数据序列号一致，防止使用已释放的元数据。
 *   4. 如果之前发生过模式切换时遗留的普通模式 tail，则记录在 qnode->wrong_fallback_tail。
 *   5. 根据更新前队列是否为空，决定是否需要初始化队列或链接到全局节点链表。
 *   6. 返回旧的队列状态（old_tail）供调用者建立 MCS 链接。
 */
static inline bool try_update_tail_hqlock_mode(struct qspinlock *lock, u16 lock_id,
				struct numa_qnode *qnode, u32 tail, u32 *next_tail, u32 *old_tail)
{
	u32 meta_seq_counter;
	hqlock_mode_t mode;

	struct numa_queue *queue;
	u64 old_counter_tail;
	bool updated_queue_tail = false;

re_setup:
	/* 获取/验证锁的模式，并读取元数据的当前序列号 */
	mode = setup_lock_mode(lock, lock_id, &meta_seq_counter);

	/* 如果锁已经处于普通 qspinlock 模式，则无法使用 HQ 路径，返回 false */
	if (mode == LOCK_MODE_QSPINLOCK)
		return false;

	/* 获取当前 CPU 所在 NUMA 节点的本地队列（每个 lock_id 在每个 NUMA node 上有一个队列） */
	queue = get_local_queue(qnode);

	/*
	 * 本地队列的 seq_counter_tail 字段是一个 64 位值：
	 *   - 低 32 位：队列尾索引（与 qspinlock 的 tail 编码相同）
	 *   - 高 32 位：元数据序列号（seq_counter）
	 * 每次队列操作前，必须确保队列中保存的序列号与元数据当前的序列号匹配，
	 * 否则说明元数据已被释放并重新分配给其他锁，当前队列已无效，需要重试。
	 */
	old_counter_tail = READ_ONCE(queue->seq_counter_tail);

	/* 循环尝试更新本地队列尾，直到成功或发现序列号不匹配 */
	while (!updated_queue_tail &&
		   decode_tc_counter(old_counter_tail) == meta_seq_counter) {
		/*
		 * 构造新的 seq_counter_tail 值：
		 *   - 尾索引：(*next_tail) >> _Q_TAIL_OFFSET（通常是当前 CPU 的编码）
		 *   - 序列号：meta_seq_counter（保持不变）
		 */
		updated_queue_tail =
			try_cmpxchg_relaxed(&queue->seq_counter_tail, &old_counter_tail,
				encode_tc((*next_tail) >> _Q_TAIL_OFFSET, meta_seq_counter));
	}

	/* 如果 CAS 失败（old_counter_tail 的序列号已改变），说明元数据被重置，必须重新开始 */
	if (!updated_queue_tail)
		goto re_setup;

	/*
	 * 特殊情况：*next_tail != tail 表示我们在进入此函数之前，
	 * 曾经尝试在普通 qspinlock 模式下执行 xchg_tail，但锁模式在那一刻被切换为 HQ。
	 * 那些在普通模式下已经排队（设置了全局 tail）的线程需要被“收养”。
	 * 记录它们的 tail 到 qnode->wrong_fallback_tail，以便在交接时通知它们
	 * 它们实际上已经被移入本 NUMA 节点的本地队列中。
	 */
	if (unlikely(*next_tail != tail))
		qnode->wrong_fallback_tail = *next_tail >> _Q_TAIL_OFFSET;

	/* 获取更新前的本地队列尾索引（低 32 位） */
	*old_tail = decode_tc_tail(old_counter_tail);

	/* 如果之前本地队列为空（old_tail == 0），需要初始化队列并可能加入全局节点链表 */
	if (!(*old_tail)) {
		u16 prev_node_id;

		/* 初始化当前 NUMA 队列（设置 head 指针、清除 handoffs 计数等） */
		init_queue(qnode);
		/*
		 * 将本节点追加到全局节点链表的尾部，返回之前的尾节点 ID。
		 * 如果之前没有其他节点，prev_node_id == 0；否则为非零。
		 */
		prev_node_id = append_node_queue(lock_id, qnode->numa_node);
		/*
		 * 编码 old_tail 以表示“新节点队列”：
		 *   - 如果 prev_node_id != 0，则返回 Q_NEW_NODE_QUEUE（一个特殊标志值），
		 *     通知调用者本线程是第一个入队者且全局链表已有其他节点。
		 *   - 否则返回 0，表示当前锁完全没有等待者（包括其他节点）。
		 */
		*old_tail = prev_node_id ? Q_NEW_NODE_QUEUE : 0;
	} else {
		/*
		 * 本地队列非空，old_tail 是之前的尾索引（CPU 编码），
		 * 需要左移 _Q_TAIL_OFFSET 位，以符合调用者期望的全局 tail 编码格式，
		 * 这样调用者可以将其作为“上一个节点”的指针来链接 MCS 节点。
		 */
		*old_tail <<= _Q_TAIL_OFFSET;
	}

	return true;
}
```

#### 关键设计要点说明

1. **序列号验证机制**  
   每个 `numa_queue` 的 `seq_counter_tail` 高位存储元数据的序列号。当元数据被释放并重新分配给另一个锁时，序列号会递增。如果线程在入队前读取到的队列序列号与当前元数据的序列号不一致，说明该队列已经属于另一个锁，必须重新执行 `setup_lock_mode` 获取新的元数据绑定，从而避免错误地将自己加入其他锁的队列。

2. **`Q_NEW_NODE_QUEUE` 特殊值**  
   * 当线程是本地队列的第一个入队者，且全局节点链表中已有其他节点（即 `prev_node_id != 0`）时，`*old_tail` 被设置为 `Q_NEW_NODE_QUEUE`（值为 1）。
   * 这个特殊值在 `queued_spin_lock_slowpath` 中被检测到（`if (numa_awareness_on && old == Q_NEW_NODE_QUEUE) goto mcs_spin;`），表示该线程需要直接进入 MCS 自旋等待，而不是再次尝试通过 `xchg_tail` 链接，因为锁已经被其他节点上的线程持有或等待。

3. **处理模式切换时的遗留 tail**  
   当锁从普通模式切换到 HQ 模式时，可能已经有多个线程通过普通模式的 `xchg_tail` 设置了全局 tail，并看到了 `_Q_LOCK_INVALID_TAIL`。这些线程会调用 `try_update_tail_qspinlock_mode` 进行补偿，并将它们收集到的 tail 传递给 `try_update_tail_hqlock_mode`。这里的 `wrong_fallback_tail` 机制确保了这些“迷失”的线程最终能被正确纳入本地队列，并在锁释放时得到唤醒。

4. **`old_tail` 的编码一致性**  
   调用者（`hqlock_xchg_tail`）期望 `old_tail` 要么是 0（无前驱），要么是左移后的 tail 值（可以解码为 MCS 节点指针）。
   * 因此当本地队列非空时，函数将 `old_tail` 左移 `_Q_TAIL_OFFSET` 位；
   * 当队列为空且没有其他节点时返回 `0`；
   * 当队列为空但有其他节点时返回 `Q_NEW_NODE_QUEUE`，调用者会特殊处理这个值。

### `try_update_tail_qspinlock_mode()`

```c
/*
 * try_update_tail_qspinlock_mode - 在普通 qspinlock 模式下尝试更新锁的全局 tail
 * @lock:      目标锁指针
 * @tail:      当前线程编码后的 tail 值（标准 qspinlock 格式，已包含 CPU 和节点索引）
 * @old_tail:  输出参数，返回更新前锁的 tail 值（若成功）或 0（若模式切换后直接成功）
 * @next_tail: 输入/输出参数，最初等于 tail；若因模式切换导致 xchg_tail 返回 _Q_LOCK_INVALID_TAIL，
 *             则可能更新为从锁值中读取到的有效 tail，供后续重试使用。
 *
 * 返回值：
 *   true  - 成功在普通模式下完成 tail 更新（或成功处理了模式切换，锁已变为普通模式）
 *   false - 需要回退到 HQ 模式路径（锁当前处于 HQ 模式，且未能恢复 _Q_LOCK_INVALID_TAIL）
 *
 * 核心逻辑：
 *   1. 尝试执行标准的 xchg_tail() 操作，将当前线程的 tail 写入锁的全局 tail 字段。
 *   2. 如果返回的旧值不是 _Q_LOCK_INVALID_TAIL（特殊标志，表示锁正处于模式切换中），
 *      则正常返回旧 tail，表示成功入队到全局队列。
 *   3. 如果返回 _Q_LOCK_INVALID_TAIL，说明锁当前不在普通模式（可能正在切换到 HQ 模式，
 *      或已经处于 HQ 模式）。此时必须：
 *       a) 读取锁当前值，判断锁模式。
 *       b) 如果锁已经变为普通模式，则直接返回成功（旧 tail 为 0，表示没有前驱）。
 *       c) 否则，通过 CAS 将锁值中的 _Q_LOCK_INVALID_TAIL 重新设置（保持该标志），
 *          同时保留锁的其他位（locked、pending、模式位等）。
 *       d) 如果在 CAS 过程中发现锁值变化（可能模式切换已完成），则记录新的 tail 到 *next_tail，
 *          并返回 false，让调用者重试 HQ 路径。
 *
 * 注：此函数主要用于处理锁从普通模式向 HQ 模式切换时的过渡期，避免多个 CPU 同时看到
 *     _Q_LOCK_INVALID_TAIL 导致混乱。
 */
static inline bool try_update_tail_qspinlock_mode(struct qspinlock *lock, u32 tail, u32 *old_tail, u32 *next_tail)
{
    /*
     * next_tail 初始通常等于 tail，但如果之前有过一次失败的调用，
     * 它可能保存了从锁中读取到的其他 CPU 的 tail（极端情况）。
     * 使用 *next_tail 执行 xchg_tail，避免丢失已经排队的线程。
     */
    u32 xchged_tail = xchg_tail(lock, *next_tail);

    /* 常见情况：返回的不是 _Q_LOCK_INVALID_TAIL，说明锁处于普通模式，入队成功 */
    if (likely(xchged_tail != _Q_LOCK_INVALID_TAIL)) {
        *old_tail = xchged_tail;
        return true;
    }

    /*
     * 返回 _Q_LOCK_INVALID_TAIL，表示锁不在普通模式（可能是 HQ 模式或正在切换）。
     * 此时需要进一步处理，避免后续线程无限重试。
     */
    u32 val = atomic_read(&lock->val);
    bool fixed = false;

    /* 循环直到确定锁的模式，并恢复 _Q_LOCK_INVALID_TAIL 标志（如果仍需保持 HQ 模式） */
    while (!fixed) {
        /* 如果锁已经变为普通模式（可能其他 CPU 完成了模式切换），直接返回成功 */
        if (decode_lock_mode(val) == LOCK_MODE_QSPINLOCK) {
            *old_tail = 0;   /* 没有前驱，调用者应视为自己是队列第一个 */
            return true;
        }

        /*
         * 锁仍处于 HQ 模式或“无模式”状态，需要重新设置 _Q_LOCK_INVALID_TAIL，
         * 以防止后续普通模式的 xchg_tail 继续错误地返回该标志。
         * 构造新值：保留锁的 locked、pending、以及锁类型/模式位（_Q_LOCK_TYPE_MODE_MASK），
         * 并将 tail 字段设置为 _Q_LOCK_INVALID_TAIL。
         *
         * 使用 CAS 是为了防止在读取 val 之后，锁模式又变回普通模式。
         * 如果 CAS 成功，则标志恢复完成；如果失败，则 val 被更新为最新值，循环继续。
         */
        fixed = atomic_try_cmpxchg_relaxed(&lock->val, &val,
                _Q_LOCK_INVALID_TAIL | (val & (_Q_LOCKED_PENDING_MASK | _Q_LOCK_TYPE_MODE_MASK)));
    }

    /*
     * 如果从锁值中读取到的 tail（低 32 位 tail 字段）与传入的 tail 不同，
     * 说明在本次调用过程中，有其他 CPU 已经成功更新了 tail。
     * 将那个 tail 保存到 *next_tail 中，以便调用者在下一次尝试（可能走 HQ 路径）
     * 时能够保留这个已经排队的线程信息，避免丢失。
     */
    if ((val & _Q_TAIL_MASK) != tail)
        *next_tail = val & _Q_TAIL_MASK;

    /* 返回 false，通知调用者当前无法在普通模式下完成入队，需要尝试 HQ 路径 */
    return false;
}
```

#### 关键设计要点说明

1. **`_Q_LOCK_INVALID_TAIL` 标志的作用**  
   当锁从普通模式向 HQ 模式切换时，锁的 tail 字段会被设置为 `_Q_LOCK_INVALID_TAIL`（一个全 1 的特殊值，在普通 qspinlock 中不会出现）。这告诉所有后续试图通过 `xchg_tail` 入队的线程：“锁已不在普通模式，不要继续用普通逻辑”。这些线程会进入本函数，看到返回值是 `_Q_LOCK_INVALID_TAIL`，从而转去执行 HQ 路径。

2. **恢复 `_Q_LOCK_INVALID_TAIL` 的必要性**  
   当多个线程几乎同时调用 `xchg_tail` 并返回 `_Q_LOCK_INVALID_TAIL` 时，第一个线程可能已经将锁的 tail 字段修改为其他值（例如通过 HQ 路径的 `append_node_queue`）。其他线程需要重新将锁的 tail 设置为 `_Q_LOCK_INVALID_TAIL`，以确保后续新来的普通模式线程仍然能够检测到模式切换。这就是 CAS 循环的作用。

3. **保留锁的其他位**  
   在恢复 `_Q_LOCK_INVALID_TAIL` 时，必须保留锁的 `locked`、`pending` 以及锁类型/模式位（`_Q_LOCK_TYPE_MODE_MASK`），否则可能破坏锁的状态（例如错误地清除 `locked` 位导致多个线程同时获得锁）。

4. **`*next_tail` 的更新**  
   如果在处理过程中发现锁的 `tail` 字段已经包含了某个 CPU 的 `tail`（不是传入的 `tail`），说明已经有另一个普通模式的线程成功更新了 `tail`（可能发生在锁刚刚切回普通模式的一瞬间）。将该 `tail` 返回给调用者，调用者（`hqlock_xchg_tail()`）会将其传递给 `try_update_tail_hqlock_mode()`，最终通过 `qnode->wrong_fallback_tail` 记录，确保那个线程不会被遗漏。


### `hqlock_xchg_tail()`

```c
/*
 * hqlock_xchg_tail - 根据锁当前模式，将当前线程入队到合适的队列
 * @lock:              目标锁指针
 * @tail:              当前线程的编码 tail（包含 CPU 和节点索引）
 * @node:              当前线程的 mcs_spinlock 节点（实际类型为 numa_qnode）
 * @numa_awareness_on: 输入/输出标志，指示当前锁是否处于 NUMA 感知模式
 *
 * 返回值：
 *   成功入队后，返回锁/队列之前的 tail 值（编码格式与标准 qspinlock 一致），
 *   用于链接 MCS 节点。特殊值 Q_NEW_NODE_QUEUE (1) 表示当前线程是第一个
 *   入队者且已有其他 NUMA 节点在等待，需要特殊处理。
 *
 * 核心逻辑（状态机）：
 *   锁可以处于三种模式之一：
 *     1. LOCK_MODE_QSPINLOCK - 普通模式（低竞争或元数据冲突）
 *     2. LOCK_NO_MODE        - 无模式（曾经有竞争但队列已空，首个入队者需尝试启用 HQ）
 *     3. LOCK_MODE_HQLOCK    - HQ 模式（高竞争，启用 NUMA 感知）
 *
 *   根据当前观察到的模式和历史标志，依次尝试：
 *     - 如果 numa_awareness_on == false（之前看到的是普通模式），先尝试普通入队。
 *     - 计算 lock_id（锁的哈希），然后尝试 NUMA 感知入队。
 *     - 若 NUMA 入队失败（元数据冲突等），回退到普通入队。
 *     - 极端情况（模式在尝试过程中突变）则重试整个流程。
 */
static inline u32 hqlock_xchg_tail(struct qspinlock *lock, u32 tail,
				 struct mcs_spinlock *node, bool *numa_awareness_on)
{
	struct numa_qnode *qnode = (struct numa_qnode *)node;

	u16 lock_id;
	u32 old_tail;
	u32 next_tail = tail;

	/*
	 * 第一阶段：如果调用者之前观察到的锁模式是普通模式（即 slowpath 入口
	 * 看到的是 LOCK_MODE_QSPINLOCK），则优先尝试普通 qspinlock 入队。
	 * 这可以避免不必要的 NUMA 路径开销，尤其是在竞争不激烈时。
	 *
	 * try_update_tail_qspinlock_mode 会处理 _Q_LOCK_INVALID_TAIL 标志，
	 * 如果锁正在切换到 HQ 模式，该函数会返回 false 并可能修改 next_tail。
	 */
	if (*numa_awareness_on == false &&
		try_update_tail_qspinlock_mode(lock, tail, &old_tail, &next_tail))
		return old_tail;

	/*
	 * 第二阶段：计算当前锁的哈希 ID（用于索引元数据池）。
	 * 每个锁在首次进入 NUMA 路径时确定 lock_id，并保存在 qnode 中，
	 * 后续重试时直接使用已计算的值。
	 */
	qnode->lock_id = lock_id = hash_ptr(lock, LOCK_ID_BITS);

try_again:
	/*
	 * 尝试 NUMA 感知入队（适用于锁处于 LOCK_NO_MODE 或 LOCK_MODE_HQLOCK 的情况）。
	 * 该函数会：
	 *   - 调用 setup_lock_mode() 确保锁处于 HQ 模式（或确认已是 HQ 模式），
	 *   - 将当前线程加入本地 NUMA 队列，
	 *   - 如果本地队列为空，则将本节点追加到全局节点链表。
	 * 成功时返回 true，并将 old_tail 设置为：
	 *     - 0                : 本地队列为空且无其他节点
	 *     - Q_NEW_NODE_QUEUE : 本地队列为空但有其他节点
	 *     - 非零（左移后）   : 本地队列非空，即前驱 MCS 节点的 tail 编码
	 */
	if (try_update_tail_hqlock_mode(lock, lock_id, qnode, tail, &next_tail, &old_tail)) {
		*numa_awareness_on = true;   /* 后续慢路径知道锁处于 NUMA 模式 */
		return old_tail;
	}

	/*
	 * 第三阶段：NUMA 入队失败（通常因为元数据哈希冲突，导致锁被迫回退到
	 * LOCK_MODE_QSPINLOCK）。此时再次尝试普通入队。
	 *
	 * 注意：这里传入的 next_tail 可能已被 try_update_tail_qspinlock_mode
	 * 或 try_update_tail_hqlock_mode 修改过（例如，记录下因模式切换而丢失的 tail）。
	 */
	if (try_update_tail_qspinlock_mode(lock, tail, &old_tail, &next_tail)) {
		*numa_awareness_on = false;  /* 标记为普通模式 */
		return old_tail;
	}

	/*
	 * 极少数情况：在执行上述步骤期间，锁的 tail 被其他 CPU 清除（例如
	 * 最后一个持有者释放锁并清空了所有队列），导致上述两次尝试均失败。
	 * 此时需要从头重试整个流程。
	 */
	goto try_again;
}
```

#### 关键设计要点

1. **模式记忆与自适应**  
   `numa_awareness_on` 是一个调用者维护的标志，记录了上一次慢路径观察到的锁模式。这允许函数优先尝试上一次成功使用的路径，减少不必要的元数据操作。例如，如果一个锁曾经进入 HQ 模式，后续的竞争者会直接走 NUMA 路径；反之，低竞争时锁会保持在普通模式。

2. **`next_tail` 的传递**  
   在模式切换的过渡期，某些线程可能已经在普通模式下执行了 `xchg_tail` 但看到了 `_Q_LOCK_INVALID_TAIL`。这些线程的 `tail` 值通过 `next_tail` 参数在不同尝试之间传递，最终被记录到 `qnode->wrong_fallback_tail` 中，确保它们不会丢失。

3. **重试与回退**  
   - 普通入队失败（返回 `false`）意味着锁正处于模式切换中，此时立即尝试 NUMA 入队。  
   - NUMA 入队失败通常是因为元数据哈希冲突，此时锁已自动切回普通模式，因此再次尝试普通入队。  
   - 极端情况下的 `goto try_again` 保证了算法的终止性。

4. **与标准 qspinlock 的兼容性**  
   如果锁从未进入 NUMA 路径（`numa_awareness_on` 始终为 `false`），则函数行为退化为单次 `xchg_tail` 调用（通过 `try_update_tail_qspinlock_mode`），与原生 qspinlock 几乎相同，开销仅多了一次条件判断。

### `hqlock_clear_tail_handoff()`

```c
/*
 * hqlock_clear_tail_handoff - 释放锁时，根据锁模式决定交接策略
 * @lock:      目标锁指针
 * @val:       锁当前的原子值（包含 locked、pending、tail 等信息）
 * @tail:      当前线程的编码 tail（与入队时相同）
 * @node:      当前线程的 mcs_spinlock 节点（实际类型为 numa_qnode）
 * @next:      当前线程在 MCS 队列中的下一个节点（可能为 NULL）
 * @prev:      当前线程在 MCS 队列中的上一个节点（用于普通模式交接）
 * @is_numa_lock: 指示锁是否处于 NUMA 感知模式（由调用者根据锁的 val 判断）
 *
 * 核心逻辑：
 *   该函数在锁持有者即将释放锁时调用，负责：
 *     1. 如果锁处于 NUMA 模式（is_numa_lock == true）或存在“错误回退”标记
 *        （wrong_fallback_tail），则调用 hqlock_try_clear_tail() 进行 NUMA 感知的
 *        队列清理和交接决策。
 *     2. 否则，按普通 qspinlock 模式处理：如果当前线程是全局队列尾且无竞争，
 *        直接清除 tail；否则设置 locked 位，并将锁交接给 MCS 队列中的下一个节点。
 *
 * 注意：此函数在锁已被当前线程持有的上下文中执行，无需再次获取锁。
 */
static inline void hqlock_clear_tail_handoff(struct qspinlock *lock, u32 val,
				    u32 tail,
				    struct mcs_spinlock *node,
				    struct mcs_spinlock *next,
				    struct mcs_spinlock *prev,
				    bool is_numa_lock)
{
	int handoff_info;
	struct numa_qnode *qnode = (void *)node;

	/*
	 * qnode->wrong_fallback_tail 非零表示：当前线程在入队时，曾尝试普通模式
	 * 但遇到 _Q_LOCK_INVALID_TAIL，然后被某个“先驱”线程纳入其本地队列。
	 * 这种情况通常发生在锁从普通模式切换到 HQ 模式的过渡期。
	 * 如果存在该标记，即使锁当前不处于 NUMA 模式，也必须走 NUMA 交接路径，
	 * 因为当前线程实际存在于某个 NUMA 节点的本地队列中。
	 */
	if (is_numa_lock || qnode->wrong_fallback_tail) {
		/*
		 * 关键：在 NUMA 模式下，我们不能先清除 tail 再设置 locked，
		 * 否则可能导致两个线程同时认为自己拥有锁。
		 * 因此先通过 set_locked() 将锁的 locked 位设置为 1，
		 * 确保在清理队列期间锁处于“锁定”状态，防止其他线程偷锁。
		 */
		set_locked(lock);

		/*
		 * 尝试清除当前 NUMA 队列的 tail，并判断需要何种 handoff。
		 * hqlock_try_clear_tail 返回值：
		 *   true  - 当前线程是最后一个等待者，锁已完全释放（无需交接）
		 *   false - 存在其他等待者，需要执行交接
		 * 同时通过 handoff_info 输出具体的交接类型：
		 *   HQLOCK_HANDOFF_LOCAL (0)        - 交接给同 NUMA 节点的下一个线程
		 *   HQLOCK_HANDOFF_REMOTE_HEAD (-1) - 交接给其他节点的队列头部
		 *   >0                              - 交接给指定节点 ID 的队列头部
		 */
		if (hqlock_try_clear_tail(lock, val, tail, node, &handoff_info))
			return;   /* 没有等待者，直接返回（锁已释放） */

		/* 存在等待者，执行实际交接（本地或远程） */
		hqlock_handoff(lock, node, next, tail, handoff_info);
	} else {
		/*
		 * 普通 qspinlock 模式（低竞争或 fallback 路径）
		 */

		/* 如果当前线程是全局队列的尾（即队列中只有自己），尝试直接清除 tail */
		if ((val & _Q_TAIL_MASK) == tail) {
			if (low_contention_try_clear_tail(lock, val, node))
				return;   /* 成功清除 tail，锁已释放 */
		}

		/* 否则，需要先将锁的 locked 位置 1，保证在交接期间锁处于锁定状态 */
		set_locked(lock);

		/* 如果下一个节点尚未观察到，则自旋等待它被链接 */
		if (!next)
			next = smp_cond_load_relaxed(&node->next, (VAL));

		/* 执行普通 MCS 锁交接：唤醒下一个线程，并传递 handoff 计数 */
		low_contention_mcs_lock_handoff(node, next, prev);
	}
}
```

#### 关键设计要点

1. **NUMA 模式下的先 `set_locked` 策略**  
   在 NUMA 模式下，锁的 tail 可能是一个 NUMA 节点 ID，而不是直接的 CPU 索引。如果先清除 tail（将锁值设为 `_Q_LOCKED_VAL`），可能导致其他 CPU 通过 `queued_spin_trylock` 误以为锁已释放而偷锁。因此必须先调用 `set_locked` 将 locked 位置 1，再安全地清理队列。

2. **`wrong_fallback_tail` 的处理**  
   该标记的存在意味着当前线程实际上位于某个 NUMA 节点的本地队列中，而不是全局 MCS 队列。即使锁的 `val` 显示为普通模式（例如因为元数据冲突导致临时 fallback），也必须走 NUMA 交接路径，否则会丢失正确的队列信息。

3. **`low_contention_try_clear_tail` 的作用**  
   在普通模式下，如果当前线程是全局队列的唯一元素，尝试直接清除 tail 并释放锁。如果清除成功（即没有其他等待者），则直接返回；如果失败（说明有新的等待者在此期间加入），则继续走普通交接路径。该函数还会根据 `determine_contention_qspinlock_mode` 判断是否应当设置 `_Q_LOCK_INVALID_TAIL` 来触发模式切换。

4. **与 `hqlock_handoff` 的协作**  
   `hqlock_handoff` 根据 `handoff_info` 决定是进行本地交接（`handoff_local`）还是远程交接（`handoff_remote`），同时处理公平性阈值（`general_handoffs`）和饥饿避免（`handoffs_not_head`）。交接完成后，锁的所有权转移给下一个线程，当前线程不再持有锁。

5. **内存顺序考虑**  
   `set_locked` 通常通过 `smp_store_release` 或 `atomic_or` 实现，确保之前对队列结构的写操作在锁的 locked 位发布前完成，避免重排序导致的并发问题。