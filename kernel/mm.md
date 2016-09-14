# 内存管理

# 页（Pages）

* 物理页为内存管理的最基本单元。
* 内存管理单元（MMU)——管理内存并把虚拟地址转化为物理地址的硬件。
* MMU以 **页（page）** 为单位进行处理，以页大小为单位管理系统中的页表。
* 从虚拟内存角度来看，*页* 就是最小单位。
* 体系结构不同，支持的页大小也不尽相同。32位支持4K的页，64位支持8K的页。

## page结构

### page结构成员
* include/linux/mm_types.h
```c
struct page {
    unsigned long         flags;
    atomic_t              _count;
    atomic_t              _mapcount;
    unsigned long         private;
    struct address_space *mapping;
    pgoff_t               index;
    struct list_head      lru;
    void                 *virtual;
};
```
* `flags` 存放页的状态，至少可表示32种，参考`include/linux/page-flags.h`
* `_count` 页的引用计数。
  * 当计数值为 **-1** 时说明当前内核没有引用这一页，在新的分配中可以使用它。
  * 不应当直接检查该域，而是调用`page_count()`
    * 返回 **0** 表示页空闲
    * 返回 *正整数* 表示页正在使用
  * 一个页可以由 *页缓存* 使用，此时`mapping`域指向和这个页关联的`address_space`对象
  * 页作为 *私有数据* 时，由`private`指向
  * 页可作为 *进程页表中的映射*
* `virtual` 页的 *虚拟地址*
  * *高端内存* 并不永久映射到内核地址空间上，此时这个域的值为 *NULL*，需要时，动态映射

### page结构的意义
* `page`结构与 *物理页* 相关，而非与虚拟页相关。因此该结构对页的描述是短暂的。
* 即使页中包含的数据继续存在，由于交换的原因，它们也可能并不再和同一个`page`结构相关联。
* 内核仅仅用这个数据结构来描述 *当前时刻* 在相关物理页上存放的东西。
* 这种数据结构的目的在于描述 *物理内存本身*，而不是描述 *包含在其中的数据*。
* 内核用`page`结构来管理系统中所有的页，内核需要知道：
  * 一个页是否空闲（有没有被分配）
  * 页被分配给谁
    * 用户空间进程
    * 动态分配的内核数据
    * 静态内核代码
    * 页高速缓存

# 区（Zones）
* 由于硬件的限制，内核不能对所有的页一视同仁。
* 有些页位于内存中特定的物理地址上，所以不能将其用于一些特定的任务。
* 故内核把页划分为不同的 **区（zone）**，对具有相似特性的页进行分组。
* Linux必须处理如下两种 **由于硬件存在缺陷而引起的内存寻址问题**：
  * 一些硬件只能用某种特定的内存地址来执行DMA（直接内存访问）。
  * 一些体系结构的内存的物理寻址范围比虚拟寻址范围大得多，有一些内存不能永久地映射到内核空间上。
* 因为存在这些制约条件，Linux主要使用四种区：
  * ZONE_DMA —— 这个区的页能用来执行DMA操作
  * ZONE_DMA32 —— 和ZONE_DMA类似，用来执行DMA操作。不同之处在于，这些页面只能被32为设备访问。在某些体系结构中，该区将比ZONE_DMA更大
  * ZONE_NORMAL —— 该区包含能正常映射的页
  * ZONE_HIGHMEM —— 该区包含 *“高端内存”*，其中的页不能永久映射到内存
  * include/linux/mmzone.h
```c
...
enum zone_type {
#ifdef CONFIG_ZONE_DMA
    /*
     * ZONE_DMA is used when there are devices that are not able
     * to do DMA to all of addressable memory (ZONE_NORMAL). Then we
     * carve out the portion of memory that is needed for these devices.
     * The range is arch specific.
     *
     * Some examples
     *
     * Architecture     Limit
     * ---------------------------
     * parisc, ia64, sparc  <4G
     * s390         <2G
     * arm          Various
     * alpha        Unlimited or 0-16MB.
     *
     * i386, x86_64 and multiple other arches
     *          <16M.
     */
    ZONE_DMA,
#endif
#ifdef CONFIG_ZONE_DMA32
    /*
     * x86_64 needs two ZONE_DMAs because it supports devices that are
     * only able to do DMA to the lower 16M but also 32 bit devices that
     * can only do DMA areas below 4G.
     */
    ZONE_DMA32,
#endif
    /*
     * Normal addressable memory is in ZONE_NORMAL. DMA operations can be
     * performed on pages in ZONE_NORMAL if the DMA devices support
     * transfers to all addressable memory.
     */
    ZONE_NORMAL,
#ifdef CONFIG_HIGHMEM
    /*
     * A memory area that is only addressable by the kernel through
     * mapping portions into its own address space. This is for example
     * used by i386 to allow the kernel to address the memory beyond
     * 900MB. The kernel will set up special mappings (page
     * table entries on i386) for each page that the kernel needs to
     * access.
     */
    ZONE_HIGHMEM,
#endif
    ZONE_MOVABLE,
#ifdef CONFIG_ZONE_DEVICE
    ZONE_DEVICE,
#endif
__MAX_NR_ZONES

};
...
```

* 区的实际使用和分布与体系结构相关。
  * 某些体系结构在内存的任何地址上执行DMA都没有问题。此时，`ZONE_DMA`为空，`ZONE_NORMAL`可直接用于分配。
  * x86体系结构上的ISA设备就不能在整个32位地址空间中执行DMA，只能访问物理内存的前16MB。因此，`ZONE_DMA`在x86上包含的页都在0~16MB的内存范围里。
* `ZONE_HIGHMEM`能否直接映射也取决于体系结构。
  * 在32位x86系统上，`ZONE_HIGHMEM`为高于 896MB 的所有物理内存。
  * 在其他体系结构上，由于所有内存都被直接映射，所以`ZONE_HIGHMEM`为空。
* **高端内存（high memory）** `ZONE_HIGHMEM`所在的内存为高端内存。
* **低端内存（low memory）** 系统的其余内存为低端内存。
* 除x86架构外，在其他体系结构上`ZONE_NORMAL`是所有的可用物理内存。

#### Zones on x86-32
Zone | Description | Physical Memory
---|---|---
ZONE_DMA | DMA-able pages | < 16MB
ZONE_NORMAL | Normally addressable pages | 16–896MB
ZONE_HIGHMEM | Dynamically mapped pages | > 896MB

* 注意：**区的划分没有任何物理意义**，这只不过是内核为了管理页而采取的一种逻辑上的分组。
* 某些分配可能需要从特定的区中获取页，而另一些分配则可以从多个区中获取页。
  * 用于DMA的内存必须从`ZONE_DMA`中进行分配
  * 一般用途的内存即能从`ZONE_DMA`也能从`ZONE_NORMAL`中分配
  * 但不能同时从两个区分配，因为 **分配是不能跨区界限的**。
* 不是所有体系结构都定义了全部区。
  * 如Intel x86-64体系结构可以映射和处理64位的地址空间，所以x86-64没有`ZONE_HIGHMEM`，所有的物理内存都处于`ZONE_DMA`和`ZONE_NORMAL`。

* include/linux/mmzone.h
```c
enum zone_watermarks {
    WMARK_MIN,
    WMARK_LOW,
    WMARK_HIGH,
    NR_WMARK
};
...
struct zone {
  /* zone watermarks, access with *_wmark_pages(zone) macros */
  unsigned long            watermark[NR_WMARK];
  unsigned long            lowmem_reserve[MAX_NR_ZONES];
  struct per_cpu_pageset   pageset[NR_CPUS];
  /* Write-intensive fields used from the page allocator */
  spinlock_t               lock;
  struct free_area         free_area[MAX_ORDER];
  spinlock_t               lru_lock;
  struct zone_lru {
    struct list_head       list;
    unsigned long          nr_saved_scan;
  } lru[NR_LRU_LISTS];
  struct zone_reclaim_stat reclaim_stat;
  unsigned long            pages_scanned;
  unsigned long            flags;
  atomic_long_t            vm_stat[NR_VM_ZONE_STAT_ITEMS];
  int                      prev_priority;
  unsigned int             inactive_ratio;
  wait_queue_head_t        *wait_table;
  unsigned long            wait_table_hash_nr_entries;
  unsigned long            wait_table_bits;
  struct pglist_data       *zone_pgdat;
  unsigned long            zone_start_pfn;
  unsigned long            spanned_pages;
  unsigned long            present_pages;
  const char               *name;
};
```
* `lock` 防止`zone`结构的并发访问。不是保护页的。
* `watermark` 数组持有该区的最小，最低和最高水位值。内核使用水位为每个zone设置合适的内存消耗基准。该水位随空闲内存的多少而变化。
* `name` 以`NULL`结束的字符串表示这个区的名字。启动时初始化。

# 获得页

## 物理页分配函数
* 分配2<sup>order</sup>个 **连续的** **物理页**，返回一个指针，指向第一个页的`page`结构体。
  ```c
  static inline struct page * alloc_pages(gfp_t gfp_mask, unsigned int order)
  ```
* *页地址* 转 *逻辑地址*
  ```c
  static inline void *page_address(const struct page *page)
  ```
* 分配连续物理页，并返回逻辑地址
  ```c
  /*
   * Common helper functions.
   */
  unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order)
  {
      struct page *page;

      /*
       * __get_free_pages() returns a 32-bit address, which cannot represent
       * a highmem page
       */
      VM_BUG_ON((gfp_mask & __GFP_HIGHMEM) != 0);

      page = alloc_pages(gfp_mask, order);
      if (!page)
          return 0;
      return (unsigned long) page_address(page);
  }
  EXPORT_SYMBOL(__get_free_pages);
  ```

* 分配单个页面
  ```c
  struct page * alloc_page(gfp_t gfp_mask)
  unsigned long __get_free_page(gfp_t gfp_mask)
  ```

### Low-Level Page Allocation Methods
Flag | Description
---|---
alloc_page(gfp_mask) | Allocates a single page and returns a pointer to its
alloc_pages(gfp_mask,order) | Allocates 2 <sup>order</sup> pages and returns a pointer to the first page’s page structure
__get_free_page(gfp_mask) | Allocates a single page and returns a pointer to its logical address
__get_free_pages(gfp_mask,order)  | Allocates 2 <sup>order</sup> pages and returns a pointer to the first page’s logical address
get_zeroed_page(gfp_mask) | Allocates a single page, zero its contents and returns a pointer to its logical address

# 释放页
* 释放页函数
  ```c
  void __free_pages(struct page *page, unsigned int order)
  void free_pages(unsigned long addr, unsigned int order)
  void free_page(unsigned long addr)
  ```
* 释放页时要谨慎，只能释放属于你的页。传递了错误的`struct page`或地址，用了错误的`order`值，这些都可能导致系统崩溃。
* 调用分配函数之后要注意进行错误检查。内核分配可能失败。

# kmalloc()
* 以 *字节* 为单位分配。返回的内存块 *至少* 要有`size`大小。
  ```c
  static __always_inline void *kmalloc(size_t size, gfp_t flags)
  ```
> kmalloc() may allocate more than you asked, although you have no way of knowing how much more! Because at its heart the kernel allocator is page-based, some allocations may be rounded up to fit within the available memory. The kernel never returns less memory than requested. If the kernel is unable to find at least the requested amount, the allocation fails and the function returns NULL.

* 分配的内存区域（region）在 **物理上是连续的**。
* 注意检查返回值是不是`NULL`，如果是，要适当地处理错误。

## gfp_mask标志

### 标志分类
* include/linux/gfp.h

#### 行为修饰符（How）
* 表示内核应当如何分配所需的内存。
* 某些特定的情况下，只能使用某些特定的方法分配内存（例如，在中断处理程序中分配内存的过程中不能睡眠）。

##### Action Modifiers

Flag | Description
---|---
__GFP_WAIT | The allocator can sleep.
__GFP_HIGH | The allocator can access emergency pools.
__GFP_IO | The allocator can start disk I/O.
__GFP_FS | The allocator can start filesystem I/O.
__GFP_COLD | The allocator should use cache cold pages.
__GFP_NOWARN | The allocator does not print failure warnings.
__GFP_REPEAT | The allocator repeats the allocation if it fails, but the allocation can potentially fail.
__GFP_NOFAIL | The allocator indefinitely repeats the allocation. The allocation cannot fail.
__GFP_NORETRY | The allocator never retries if the allocation fails.
__GFP_NOMEMALLOC | The allocator does not fall back on reserves.
__GFP_HARDWALL | The allocator enforces “hardwall” cpuset boundaries.
__GFP_RECLAIMABLE | The allocator marks the pages reclaimable.
__GFP_COMP | The allocator adds compound page metadata (used internally by the hugetlb code).

#### 区修饰符（Where）
* 表示从哪儿分配内存。
* 内核优先从`ZONE_NORMAL`区开始，为确保其他区在需要时有足够的空闲页可用。
* 不能给`__get_free_pages()`或`kmalloc()`指定`ZONE_HIGHMEM`。
  * 返回的是逻辑地址，而不是`page`结构；
  * 这两个函数分配的内存可能还没映射到内核的虚拟地址空间，因此，可能根本就没有逻辑地址。
* 只有`alloc_pages()`才能分配高端内存`ZONE_HIGHMEM`。

##### Zone Modifiers
Flag | Description
---|---
__GFP_DMA | Allocates only from ZONE_DMA
__GFP_DMA32 | Allocates only from ZONE_DMA32
__GFP_HIGHMEM | Allocates from ZONE_HIGHMEM or ZONE_NORMAL

#### 类型标志
* 组合了 **行为修饰符** 和 **区修饰符**，将各种可能用到的组合归纳为不同的类型，简化了修饰符的使用。

##### Type Flags

Flag | Description
---|---
GFP_ATOMIC | The allocation is high priority and **must not sleep**. This is the flag to use in interrupt handlers, in bottom halves, while holding a spin-lock, and in other situations where you cannot sleep.
GFP_NOWAIT | Like `GFP_ATOMIC` , except that the call **will not fallback on emergency memory pools**. This increases the liklihood of the memory allocation failing.
GFP_NOIO | This allocation **can block, but must not initiate disk I/O**. This is the flag to use in block I/O code when you cannot cause more disk I/O, which might lead to some unpleasant recursion.
GFP_NOFS | This allocation can block and can initiate disk I/O, if it must, but it **will not initiate a filesystem operation**. This is the flag to use in filesystem code when you cannot start another filesystem operation.
GFP_KERNEL | This is a normal allocation and might block. This is the flag to use in process context code when it is safe to sleep. The kernel will do whatever it has to do to obtain the memory requested by the caller. This flag should be your **default choice**.
GFP_USER | This is a normal allocation and might block. This flag is used to allocate memory **for user-space processes**.
GFP_HIGHUSER | This is an allocation from **`ZONE_HIGHMEM` and might block**. This flag is used to allocate memory **for user-space processes**.
GFP_DMA | This is an allocation from `ZONE_DMA`. Device drivers that **need DMA-able memory** use this flag, usually in combination with one of the preceding flags.

* 当前代码（如中断处理程序，softirq，tasklet）不能睡眠时，只能选择`GFP_ATOMIC`。
* 不管那种分配类型，都必须进行检查，并对错误进行处理。

#### Which Flag to Use When
Situation | Solution
---|---
Process context, can sleep | Use `GFP_KERNEL`.
Process context, cannot sleep | Use `GFP_ATOMIC`, or perform your allocations with `GFP_KERNEL` at an earlier or later point when you can sleep.
Interrupt handler | Use `GFP_ATOMIC`.
Softirq | Use `GFP_ATOMIC`.
Tasklet | Use `GFP_ATOMIC`.
Need DMA-able memory, can sleep | Use `(GFP_DMA | GFP_KERNEL)`.
Need DMA-able memory, cannot sleep | Use `(GFP_DMA | GFP_ATOMIC)`, or perform your allocation at an earlier point when you can sleep.

## kfree()
* `kfree()`释放`kmalloc()`分配出来的内存块
  ```c
  void kfree(const void *);
  ```
* 释放的内存不是`kmalloc()`分配的，或者释放的内存早被释放了，调用该函数会导致严重后果。
* 调用`kfree(NULL)`是安全的。

# vmalloc()
* `vmalloc()`分配的 **物理地址无需连续**，当然，虚拟地址得是连续的。
* 对比`kmalloc()`，其确保分配的页在物理地址上是连续的（虚拟地址自然也是连续的）。
* `vmalloc()`通过分配非连续的物理内存块，再“修正”页表，把内存映射到逻辑地址空间的连续区域中，就能做到这点。
* 大多数情况下，只有硬件设备需要得到物理地址连续的内存。
  * 在很多体系结构上，硬件存在于MMU以外，根本不理解什么是虚拟地址。
  * 此时，硬件设备上用到的任何内存区都必须是物理上连续的块。
* 仅供软件使用的内存块就可以使用只有虚拟地址连续的内存块。
* 对内核而言，所有内存看起来都是逻辑上连续的。
* 很多代码用`kmalloc()`而不是`vmalloc()`主要是处于性能的考虑（`kmalloc()比`vmalloc()`快）。
  * `vmalloc()`把物理上不连续的页转换为虚拟地址空间上连续的页，必须专门建立页表项。
  * 通过`vmalloc()`获得的页必须一个一个地进行映射（因为物理上不连续），会导致比直接内存映射大得多的TLB抖动。
* 因此`vmalloc()`仅在不得已时才会用——典型的就是为了获得大块内存时，如，模块被动态插入到内核。
* `vmalloc()/vfree()`可能睡眠，不能用于中断上下文，也不能用于其他不允许阻塞的情况。

> **TLB (translation lookside buffer)** 是一种 **硬缓冲区**，很多体系结构用它来缓冲 **虚拟地址到物理地址的映射关系**。它极大地提高了系统性能，因为大多数内存都要进行虚拟寻址。

# slab层（Slab Layer）
