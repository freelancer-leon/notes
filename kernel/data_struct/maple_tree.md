# Maple Tree

## 概述

* Maple Tree 是一种 B Tree 类型的数据结构，专为存储 **非重叠区间**（包括大小为 `1` 的区间）而优化。
  * 该树的设计目标是使用简便，无需用户自行编写搜索方法。
  * 它支持以 **高效利用缓存** 的方式遍历指定区间内的条目，并可跳转到前一个或后一个条目。
  * Maple Tree 还可配置为 **RCU 安全模式**，支持读写并发操作。
    * 写操作必须通过锁进行同步，该锁可以是默认的自旋锁，用户也可将其设置为其他类型的外部锁。
* Maple Tree 保持较小的内存占用，并针对现代处理器的 cache 进行了高效设计。
* 大多数用户可直接使用 **基础 API**，而 **高级 API** 则适用于更复杂的场景。
* Maple Tree 最重要的应用场景是**虚拟内存区域（VMA）的跟踪管理**。
* Maple Tree 可存储 `0` 至 `ULONG_MAX` 之间的数值。
* Maple Tree 将 **低两位为 '`10`' 且数值小于 `4096`**（即 2、6、10……4094）的数值保留为内部使用。
  * 若条目可能使用这些保留值，用户可通过 `xa_mk_value()` 进行转换，并通过 `xa_to_value()` 恢复原值。
  * 若用户需要使用保留值，可在高级 API 中进行转换，但基础 API 会对此进行限制。
* Maple Tree 还可配置为支持 **搜索指定大小（或更大）的空闲区间**。
* 高级API还支持**节点预分配**。这对于那些必须在无法执行内存分配的代码段中，确保存储操作成功的用户非常有用。
* Maple Tree 节点的分配大小相对较小，约为 `256` 字节。

## 基础 API（Normal API）
* 首先需要初始化Maple Tree：静态分配的 Maple Tree 使用 `DEFINE_MTREE()`，动态分配的则使用 `mt_init()`。
* 新初始化的 Maple Tree 在区间 `0 ~ ULONG_MAX` 内存储的是 `NULL` 指针。
* 目前支持两种类型的 Maple Tree ：**分配树（allocation tree）** 与 **常规树（regular tree）**。
  * 常规树的内部节点具有更高的分支因子；
  * 分配树的分支因子较低，但允许用户从 `0` 向上或从 `ULONG_MAX` 向下搜索指定大小（或更大）的空闲区间。
* 初始化时传入 `MT_FLAGS_ALLOC_RANGE` 标志即可使用分配树。
* 之后可以通过 `mtree_store()` 或 `mtree_store_range()` 设置条目：
  * `mtree_store()` 会用新条目覆盖已有条目，成功返回 `0`，失败返回错误码。
  * `mtree_store_range()` 功能类似，但作用于一个区间。
* 使用 `mtree_load()` 可以获取指定索引处存储的条目。
* 可以使用 `mtree_erase()` 删除整个区间（只需知道区间内任意一个值即可）；也可以通过向 `mtree_store()` 传入 `NULL` 条目，来部分删除一个或多个区间。
* 如果只想在某个区间（或索引）当前为 `NULL` 时才存入新条目，可以使用 `mtree_insert_range()` 或 `mtree_insert()`。若区间非空，它们会返回 `-EEXIST`。
* 可以使用 `mt_find()` 从指定索引向上搜索条目。
* 可以通过 `mt_for_each()` 遍历指定区间内的每个条目，调用时必须提供一个临时变量来存储游标。
* 若要遍历整棵树，可将区间设为 `0` 和 `ULONG_MAX`。
* 如果调用者在遍历期间会一直持有锁，建议参考高级 API 部分的 `mas_for_each()`。
* 有时需要确保下一次对 Maple Tree 的存储操作不进行内存分配，这种场景请参见高级 API。
* 可以使用 `mtree_dup()` 复制整棵 Maple Tree ，这比逐个插入元素到新树中效率更高。
* 最后，可以通过 `mtree_destroy()` 清空 Maple Tree 的所有条目。
  * 如果 Maple Tree 中的条目是指针，建议先释放这些条目。

## 分配节点（Allocating Nodes）
* 分配由内部树代码来处理

## 锁（Locking）
* 不需要担心锁
* Maple Tree 使用 RCU 和内部的自旋锁来同步访问
* 持有 RCU 读锁
  * mtree_load()
  * mt_find()
  * mt_for_each()
  * mt_next()
  * mt_prev()
* 内部持有 `ma_lock`
  * mtree_store()
  * mtree_store_range()
  * mtree_insert()
  * mtree_insert_range()
  * mtree_erase()
  * mtree_dup()
  * mtree_destroy()
  * mt_set_in_rcu()
  * mt_clear_in_rcu()
* 如果你想利用内部锁来保护存储在 Maple Tree 中的数据结构，可以在调用 `mtree_load()` 之前先调用 `mtree_lock()`，然后在调用 `mtree_unlock()` 之前对找到的对象增加引用计数。
  * 这样可以防止在 *查找对象* 和 *增加引用计数* 之间，*存储操作* 将该对象从树中移除。
  * 你也可以使用 RCU 来避免解引用已释放的内存，但相关说明不在本文档范围内。

## 高级 API（Advanced API）
* 高级 API 提供了更强的灵活性和更优的性能，但代价是接口更难使用，且内置的安全保障更少。
* 使用高级 API 时，您必须自行处理锁机制。可以使用 `ma_lock`、RCU 或外部锁进行保护。
* 只要锁机制兼容，您可以在同一棵树上混合使用高级操作和普通操作。普通 API 是基于高级 API 实现的。
* 高级 API 围绕 `ma_state` 结构体构建，这也是 `mas_` 前缀的由来。
* `ma_state` 结构体用于跟踪树的操作状态，方便树的内部和外部用户使用。
* Maple Tree 的初始化方式与普通 API 相同，请参见上文。
* Maple state 通过 `mas->index` 和 `mas->last` 分别跟踪区间的起始和结束。
* `mas_walk()` 会遍历树，定位到 `mas->index` 所在的位置，并根据该条目的区间设置 `mas->index` 和 `mas->last`。
* 可以使用 `mas_store()` 设置条目。`mas_store()` 会用新条目覆盖已有条目，并返回第一个被覆盖的已有条目。
  * 区间通过 maple state 的成员 `index` 和 `last` 传入。
* 可以使用 `mas_erase()` 删除整个区间，只需将 maple state 的 `index` 和 `last` 设置为待删除的目标区间即可。
  * 该函数会删除该范围内找到的第一个区间，将 maple state 的 `index` 和 `last` 设置为被删除的区间，并返回该位置原有的条目。
* 可以使用 `mas_for_each()` 遍历指定区间内的每个条目。若要遍历整棵树，可将区间设为 `0` 和 `ULONG_MAX`。
* 如果需要在遍历期间周期性地释放锁，请参见锁相关章节的 `mas_pause()`。
* 使用 maple state 可以让 `mas_next()` 和 `mas_prev()` 像操作链表一样工作。
* 由于 Maple Tree 的分支因子很高，缓存优化带来的收益超过了这种操作带来的平均性能损耗。
* `mas_next()` 会返回 `index` 所在条目之后的下一个条目。
* `mas_prev()` 会返回 `index` 所在条目之前的上一个条目。
* `mas_find()` 在首次调用时，会找到大于或等于 `index` 的第一个条目；后续每次调用则返回下一个条目。
* `mas_find_rev()` 在首次调用时，会找到小于或等于 `last` 的第一个条目；后续每次调用则返回上一个条目。
* 如果用户需要在操作过程中主动让出锁，必须使用 `mas_pause()` 暂停 maple state。
* 使用分配树时，还提供了一些额外接口：
* 如果需要在区间内搜索空闲区间，可以使用 `mas_empty_area()` 或 `mas_empty_area_rev()`。
* `mas_empty_area()` 从给定的最低索引开始，向上搜索至区间上限，寻找空闲区间。
* `mas_empty_area_rev()` 从给定的最高索引开始，向下搜索至区间下限，寻找空闲区间。

## 高级节点分配
* 节点分配通常由树结构内部处理，但如果需要在写入操作发生前预先分配内存，可以调用 `mas_expected_entries()`。
* 该函数会根据要插入的区间数量，按最坏情况分配所需的节点数，并使树进入**批量插入模式**。插入操作完成后，对枫树状态调用 `mas_destroy()` 即可释放未使用的分配空间。

## 高级锁机制
* Maple Tree 默认使用自旋锁，但也可以使用外部锁来进行树结构更新。
* 要使用外部锁，必须在初始化树时传入 `MT_FLAGS_LOCK_EXTERN` 标志，这通常通过宏 `MTREE_INIT_EXT()` 实现，该宏接收一个外部锁作为参数。

## 有趣的 Maple Tree 实现细节
* Maple Tree 的每种节点类型，都配备了若干 **条目槽位**（slots for entries）和若干 **枢轴值槽位**（slots for pivots）。
* 对于高密度节点，其 pivot 的值并非显式存储，而是由所在位置隐式确定，计算方式十分简单：槽位索引 + 该节点的最小值即可得到对应 pivot 的值。
  ```c
  slot index + the minimum of the node = pivot
  ```
* 按常规 B 树的术语定义，这里的 pivot 的值其实就是 B 树中的 **键（key）**；而 Maple Tree 中特意使用 “**枢轴值（pivot）**” 这一术语，是为了凸显该树结构 **专为描述地址区间设计** 的核心特性。
* 二者的本质区别在于：
  * Maple Tree 的 pivot 的值可出现在子树中，且会关联一个对应数值的条目；
  * 而传统 B Tree 中的键，是树内特定位置的 **唯一标识**，仅用于定位单个节点/元素。
* 此外需要注意，Maple Tree 中 **pivot 的值的取值范围，会包含同一索引对应的 slot**。
* 以下展示了一个 range64 节点的 slot 与 pivot 的布局。
```sh
Slots -> | 0 | 1 | 2 | ... | 12 | 13 | 14 | 15 |
         ┬   ┬   ┬   ┬     ┬    ┬    ┬    ┬    ┬
         │   │   │   │     │    │    │    │    └─ Implied maximum
         │   │   │   │     │    │    │    └─ Pivot 14
         │   │   │   │     │    │    └─ Pivot 13
         │   │   │   │     │    └─ Pivot 12
         │   │   │   │     └─ Pivot 11
         │   │   │   └─ Pivot 2
         │   │   └─ Pivot 1
         │   └─ Pivot 0
         └─  Implied minimum
```
* Slot 的内容：
  * **内部（非叶子）节点** 包含指向其他节点的指针。
  * **叶子节点** 包含条目。
* 我们所关注的位置通常被称为 **偏移量（offset）**。
  * 所有偏移量都对应一个 slot，但最后一个偏移量的 pivot 由上层节点隐式提供（根节点则为 `UINT_MAX`）。
* 区间特性会使某些写入操作变得复杂。
  * 在修改任何 B Tree 变体时，通常只会添加或删除一个条目。
  * 而在修改 Maple Tree 时，一次存储操作可能会覆盖整个数据集、树的一半，或树的中间一半。

## Maple Tree 数据结构
* 已分配的节点在插入树中之前均为可变状态；一旦插入树中，其类型便无法修改，除非将节点从树中移除并等待一个 RCU 宽限期结束。
* *被移除的节点* 会将自身的 `->parent` 指针设为指向自己。
* RCU 读端操作在使用从 *槽位数组（slots array）* 中读取的值之前，会先检查 `->parent` 指针的状态。
  * 这一设计允许我们将 slots array 复用为 RCU head 结构。
* 树中的节点均会指向其父节点，除非节点的 `->parent` 指针 `bit 0` 被置 `1`。

```cpp
#if defined(CONFIG_64BIT) || defined(BUILD_VDSO32_64)
/* 64bit sizes */
#define MAPLE_NODE_SLOTS    31  /* 256 bytes including ->parent */
#define MAPLE_RANGE64_SLOTS 16  /* 256 bytes */
#define MAPLE_ARANGE64_SLOTS    10  /* 240 bytes */
#define MAPLE_ALLOC_SLOTS   (MAPLE_NODE_SLOTS - 1)
#else
/* 32bit sizes */
#define MAPLE_NODE_SLOTS    63  /* 256 bytes including ->parent */
#define MAPLE_RANGE64_SLOTS 32  /* 256 bytes */
#define MAPLE_ARANGE64_SLOTS    21  /* 240 bytes */
#define MAPLE_ALLOC_SLOTS   (MAPLE_NODE_SLOTS - 2)
#endif /* defined(CONFIG_64BIT) || defined(BUILD_VDSO32_64) */

#define MAPLE_NODE_MASK     255UL
```

* 根节点的 `node->parent` 指针 `bit 0` 被置 `1`，指针的其余部分则指向树结构本身。该指针中已无可用的额外位（在 m68k 架构下，此数据结构仅能按 `2` 字节对齐）。
* 非根内部节点仅能以 `maple_range_*` 类型节点作为父节点。
* 所有树节点均按 `256B` 对齐，父节点指针也遵循此规则。
  * 译注：因此，节点的 `->parent` 指针的低 `8` 位（`bit 0-7`）可以被拿来复用做其他目的，比如类型和偏移量。
  * 存储 `32 bits` 或 `64 bits` 数值时，偏移量可容纳在 `4` 个比特位中；
  * 而存储 `16 bits` 数值时，需要额外 `1` 个比特位来存放偏移量，该额外位通过复用节点类型的最后一个比特位实现。
  * 具体可通过 `bit 1` 来标识：`bit 2` 究竟属于 *节点类型* 的组成部分，还是用于表示 *slot 偏移*。
* 节点类型确定后，其是 allocation range 类型还是 range 类型，将通过检查 Maple tree 中不可变的标志位 —— 即 `MT_FLAGS_ALLOC_RANGE` 标志来判定。
* 节点类型:
  * `0b??1 = Root`（根节点的 `bit 0` 被置 `1`）
  * `0b?00 = 16 bit nodes`（`bit 1` 为零 `bit 2` 复用作额外的 slot 偏移）
  * `0b010 = 32 bit nodes`（`bit 1` 置位 `bit 2` 为节点类型，此时 `bit 2` 为 `0` 表示 `32 bit` 节点）
  * `0b110 = 64 bit nodes`（`bit 1` 置位 `bit 2` 为节点类型，此时 `bit 2` 为 `1` 表示 `64 bit` 节点）
* 在父节点指针中的 slot 大小与位置：

类型    | slot 位置
--------|----------
`0b??1` | 根节点
`0b?00` | `16 bit` 值，类型在 `bit 0-1`，slot 偏移在 `bit 2-6`
`0b010` | `32 bit` 值，类型在 `bit 0-2`，slot 偏移在 `bit 3-6`
`0b110` | `64 bit` 值，类型在 `bit 0-2`，slot 偏移在 `bit 3-6`

### 节点结构

* `struct maple_metadata` 元数据用于优化空闲区间更新相关代码，同时也为 *反向搜索空闲区间* 或其他需要 *查找数据末尾* 的代码提供支持。
```cpp
struct maple_metadata {
    unsigned char end;  /* 数据末尾位置 */
    unsigned char gap;  /* 最大空闲区间的偏移量 */
};
```

* ==**叶节点** 不存储指向其他节点的指针==，而是存储用户数据。用户几乎可以存储任意位模式（bit pattern）的数据。
* 如上所述，对于低两位设为 `10` 的数据，无法实现将条目存储在根指针 `0` 位置的优化。
* 我们还将小于 `4096` 且低两位设为 `10` 的数值（即 2、6、10……4094）预留出来，供内部使用。
* 部分接口会将错误码以负数值右移两位、低两位设为 `10` 的形式返回，尽管选择将这类数值存储在数组中并非错误操作，但如果使用 `mas_is_err()` 检测错误，可能会造成判断混淆。
* **非叶节点** 会存储指向节点的类型（由 `bits 3-6` 表示，对应 `maple_type` 枚举类型），`bit 2` 为保留位。
  * 目前 `bits 0-1` 暂未使用。
* 按常规 B Tree 的术语定义，枢轴值（pivots）也被称作键（keys）。
* Maple Tree 中使用 “**枢轴值（pivots）**” 这一术语，是为了凸显该树结构专为描述区间而设计的特性。
  * Pivots 可出现在子树中，且会关联一个对应数值的条目；
  * 而传统 B Tree 中的键是树内特定位置的唯一标识。
  * Maple Tree 中，pivot 的值的取值范围包含同一索引对应的 slot。

```cpp
struct maple_range_64 {
    struct maple_pnode *parent;                   //指向父节点的指针
    unsigned long pivot[MAPLE_RANGE64_SLOTS - 1]; //一个节点有 15 个 pivots/keys
    union {
        void __rcu *slot[MAPLE_RANGE64_SLOTS];    //一个节点有 16 个子女
        struct {
            void __rcu *pad[MAPLE_RANGE64_SLOTS - 1];
            struct maple_metadata meta;           //元数据可用于优化
        };
    };
};
```
* 译注：注意两点：
  * 对于 64 bit 系统来说，`struct maple_range_64` 的大小为 `256B`
  * `struct maple_pnode` 没有原型的定义，必须当作指针来使用，用来保存地址
* 在创建树时，用户可以选择 **以减少树中可存储的条目数量为代价**，换取在每个节点中存储更多信息。
* Maple Tree 支持记录当前节点内可用的 **最大 NULL 条目区间**（也称为 **空闲区间/gap**）。
  * 这一特性为 **区间分配** 场景提供了优化。
```cpp
struct maple_arange_64 {
    struct maple_pnode *parent;                    //指向父节点
    unsigned long pivot[MAPLE_ARANGE64_SLOTS - 1]; //15 个 keys
    void __rcu *slot[MAPLE_ARANGE64_SLOTS];        //16 个子女
    unsigned long gap[MAPLE_ARANGE64_SLOTS];       //16 个空闲区间
    struct maple_metadata meta;                    //元数据
};

struct maple_alloc {
    unsigned long total;
    unsigned char node_count;
    unsigned int request_count;
    struct maple_alloc *slot[MAPLE_ALLOC_SLOTS];
};

struct maple_topiary {
    struct maple_pnode *parent;
    struct maple_enode *next; /* Overlaps the pivot */
};
```

### 树的类型枚举
```cpp
enum maple_type {
    maple_dense,     //密集型
    maple_leaf_64,   //叶子
    maple_range_64,  //范围型
    maple_arange_64, //分配范围型
};
//存储的类型
enum store_type {
    wr_invalid,
    wr_new_root,
    wr_store_root,
    wr_exact_fit,
    wr_spanning_store,
    wr_split_store,
    wr_rebalance,
    wr_append,
    wr_node_store,
    wr_slot_store,
};
```
* Maple Tree 的标志
```c
#define MT_FLAGS_ALLOC_RANGE    0x01   //跟踪该树中的空闲区
#define MT_FLAGS_USE_RCU        0x02   //在 RCU 模式下操作
#define MT_FLAGS_HEIGHT_OFFSET  0x02   //树的高度在标志位中的（偏移）位置
#define MT_FLAGS_HEIGHT_MASK    0x7C   //Maple Tree 高度值的掩码
#define MT_FLAGS_LOCK_MASK      0x300  //mt_lock 的使用方式（覆盖以下两种方式）
#define MT_FLAGS_LOCK_IRQ       0x100  //以中断安全（irq‑safe）方式获取锁
#define MT_FLAGS_LOCK_BH        0x200  //以软中断安全（bh‑safe）方式获取锁
#define MT_FLAGS_LOCK_EXTERN    0x300  //不使用内部 mt_lock（使用外部锁）
#define MT_FLAGS_ALLOC_WRAPPED  0x0800
```
* Maple Tree 能够被存储的最大高度
```c
#define MAPLE_HEIGHT_MAX    31 //能够被存储的最大高度
```

### `struct maple_tree` 结构
* 如果树在 *索引 `0`* 处只包含一个条目，该条目通常存放在 `tree->ma_root` 中。
* 为了对页缓存（page cache）进行优化，以 `00`、`01` 或 `11` 结尾的条目会存放在根节点中，而以 `10` 结尾的条目则会存放在普通节点中。
* Bits `3～6` 用于存放 `enum maple_type` 类型。
* `ma_flags` 域的作用：
  * 存储上面的 Maple Tree 的标志位，既用于存放关于该树的一些不可变信息（在树创建时设置），也用于在自旋锁保护下设置动态信息。
  * 标志位的另一个用途是指示树的全局状态，`MT_FLAGS_USE_RCU` 标志就是如此，它表示该树当前处于 RCU 模式。
    * 加入该模式是为了让树在单用户场景下能够复用节点，而不必重新分配节点并通过 RCU 机制释放节点。
```cpp
struct maple_tree {
    union {
        spinlock_t      ma_lock;
#ifdef CONFIG_LOCKDEP
        struct lockdep_map  *ma_external_lock;
#endif
    };  
    unsigned int    ma_flags;
    void __rcu      *ma_root;
};
```
### `struct maple_node` 结构
* Maple Tree 会在多处位置巧妙地压缩各类比特位，这些位置不一定直观。
  * 通常的做法是利用指针按 `N-byte` 对齐这一特性，因此其低侧 `log<sub>2</sub>(N)` 个比特位可以被复用。
  * 我们 **不会** 使用指针的高位来存储额外信息，因为无法确定在任意给定架构上哪些高位是未被使用的。
  * 节点大小为 `256` 字节，并且同样按 `256` 字节对齐，这为我们提供了 `8` 个低位比特位可供自行使用。
* 节点目前分为 4 种类型：
  1. 单指针节点（范围为 `0-0`）
  2. 非叶子分配范围（Non-leaf Allocation Range）节点
  3. 非叶子范围（Non-leaf Rang）节点
  4. 叶子范围节点，所有节点都由若干个节点槽位、枢轴值以及一个父指针组成。
```cpp
struct maple_node {
    union {
        struct {
            struct maple_pnode *parent;
            void __rcu *slot[MAPLE_NODE_SLOTS];
        };
        struct {
            void *pad;
            struct rcu_head rcu;
            struct maple_enode *piv_parent;
            unsigned char parent_slot;
            enum maple_type type;
            unsigned char slot_len;
            unsigned int ma_flags;
        };
        struct maple_range_64 mr64;
        struct maple_arange_64 ma64;
        struct maple_alloc alloc;
    };
};
```
### `struct ma_topiary` 结构
* 更复杂的存储操作可能会导致两个节点合并为一个，或者一个节点分裂为三个，并有可能改变树的高度。
  * 此时可能需要对树的某一半相对于另一半进行重新平衡。
  * `ma_topiary` 结构体用于跟踪哪些节点已从树中 “摘除”，以便后续可以安全地完成变更操作。这一设计是为了支持 RCU。
```cpp
struct ma_topiary {                     
    struct maple_enode *head;
    struct maple_enode *tail;                                            
    struct maple_tree *mtree;                                            
};  
```

## 高级 API

### Maple State 状态枚举

```cpp
enum maple_status {
    ma_active,    //表示 maple state 已指向某个节点及偏移量，可继续对树进行操作
    ma_start,     //表示尚未对树执行搜索操作
    ma_root,      //表示已完成树搜索，且找到的条目位于树的根节点（即索引为 0、长度为 1，且是树中唯一条目）
    ma_none,      //表示已完成树搜索，但树中不存在对应条目
    ma_pause,     //表示 maple state 中的数据可能已失效，需重新开始操作
    ma_overflow,  //表示搜索已达到搜索上限
    ma_underflow, //表示搜索已达到搜索下限
    ma_error,     //表示发生错误，需通过节点检查错误码
};
```
### `struct ma_state` 结构
* **maple state（maple tree 状态）** 定义在 `ma_state` 结构体中，用于在操作过程中跟踪记录相关信息；即便使用高级 API 时，也可在多次操作之间保留该状态。
```cpp
struct ma_state {
    struct maple_tree *tree;    /* 当前操作所属的 maple tree */
    unsigned long index;        /* 操作对应的索引 - 区间起始值 */
    unsigned long last;         /* 操作对应的最后一个索引 - 区间结束值 */
    struct maple_enode *node;   /* 包含本条目的节点 */
    unsigned long min;          /* 该节点的最小索引 - 隐含的枢轴最小值 */
    unsigned long max;          /* 该节点的最大索引 - 隐含的枢轴最大值 */
    struct slab_sheaf *sheaf;   /* 本次操作分配的节点集合 */
    struct maple_node *alloc;   /* 用于快速路径写入的单个已分配节点 */
    unsigned long node_request; /* 本次操作需要分配的节点数量 */
    enum maple_status status;   /* 状态标识（活跃、初始、无匹配等） */
    unsigned char depth;        /* 写操作期间树的遍历深度 */
    unsigned char offset;       /* 节点内偏移量 */
    unsigned char mas_flags;    /* ma_state 标志位 */
    unsigned char end;          /* 节点的结束位置 */
    enum store_type store_type; /* 本次操作所需的存储类型 */
};  
```

* 对于 `state->node`： 
  * 第 `0` 位（`bit 0`）被置位，则其指向的是树中非节点的位置（例如根节点特殊区域）；
  * 第 `1` 位（`bit 1`）被置位，则剩余比特位存储的是一个负的错误码（errno）；
  * 第 `2` 位（即 “未分配槽位（unallocated slots）” 位）始终为 `0`；
  * 第 `3` 至 `6` 位用于标识节点类型。
* `state->alloc` 字段要么 **存储申请的节点数量**，要么 **指向一个已分配的节点**：
  * 若 `state->alloc` 存储的是申请的节点数量，则其第 `1` 位会被置位（值为 `0x1`），剩余比特位为具体数值；
  * 若 `state->alloc` 指向一个节点，则该节点的类型为 `maple_alloc`。
* **`maple_alloc`** 类型节点包含以下内容：
  * 预留 `MAPLE_NODE_SLOTS - 1` 个位置用于存储更多已分配节点；
  * 记录已分配节点的总数；
  * 记录当前节点内的 `node_count`（本节点中已分配的节点数量）。
  * 当需要存储的节点数量超过 `MAPLE_NODE_SLOTS - 1` 时，会将额外节点存入 `state->alloc->slot[0]` 指向的节点中。
  * 从 `state->alloc` 中 **取用** 节点的逻辑：
    * 持续从 `state->alloc` 节点中移除节点，直到 `state->alloc->node_count` 变为 `1`；
    * 此时会返回当前 `state->alloc`，并将 `state->alloc->slot[0]` 升级为新的 `state->alloc`。
  * 向 `state->alloc` 中 **添加** 节点的逻辑：
    * 将当前 `state->alloc` 存入待添加节点的 `slot[0]` 中，完成节点入栈。
* 该状态结构体还包含以下核心字段：
  * `state->node` 对应的隐含 **最小/最大值（implied min/max）**：
    * 若节点 **非根节点**，则该值继承自父节点；
    * 若为 **根节点**，则范围是 `0` 到无穷大（`0-∞`）；
  * **搜索深度（depth）**：每向下/向上遍历一个节点，深度值相应加 1/减 1；
  * **偏移量（offset）**：表示当前关注的节点内槽位/枢轴（slot/pivot）位置，用于读写操作。
  * 返回值时，`maple state` 的 **`index`** 和 **`last`** 字段分别存储条目的区间 **起始值** 和 **结束值**。
* Maple Tree 中的区间均为 **闭区间（包含边界值）**。
* 状态的 **`status`** 字段用于决定后续操作如何处理该状态：
  * 若 `status` 为 `ma_start`，则后续操作需从树根开始向下遍历；
  * 若 `status` 为 `ma_pause`，则当前节点数据可能已失效，需丢弃；
  * 若 `status` 为 `ma_overflow`，则上一次操作已触及搜索上限。

```cpp
struct ma_wr_state {
    struct ma_state *mas;            /* 指向 ma_state 的指针 */
    struct maple_node *node;         /* 解码后的 mas->node 节点 */
    unsigned long r_min;             /* 区间最小值 */
    unsigned long r_max;             /* 区间最大值 */
    enum maple_type type;            /* mas->node 的节点类型 */
    unsigned char offset_end;        /* 写操作结束的偏移位置 */
    unsigned long *pivots;           /* 指向 mas->node->pivots 的指针 */
    unsigned long end_piv;           /* 偏移结束位置对应的枢轴值 */
    void __rcu **slots;              /* 指向 mas->node->slots 的指针 */
    void *entry;                     /* 待写入的条目 */
    void *content;                   /* 被覆盖的现有条目 */
    unsigned char vacant_height;     /* 存在空闲空间的最低节点高度 */
    unsigned char sufficient_height; /* 满足最小容量 +1 个节点的最低节点高度 */
};
```

## References

* [Maple Tree — The Linux Kernel documentation](https://docs.kernel.org/core-api/maple_tree.html)
* [How to get rid of mmap_sem - LWN.net](https://lwn.net/Articles/893906/)
* [Introducing maple trees - LWN.net](https://lwn.net/Articles/845507/)
* [The ongoing search for mmap_lock scalability](https://lwn.net/Articles/893906/)
* [The next steps for the maple tree - LWN.net](https://lwn.net/Articles/974860/)
* [The Maple Tree, A Modern Data Structure for a Complex Problem](https://blogs.oracle.com/linux/the-maple-tree-a-modern-data-structure-for-a-complex-problem)