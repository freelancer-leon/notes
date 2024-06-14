
# 复合页面
* **复合页面（compound page）** 是将两个或多个物理上连续的页面组合成一个单元，在许多方面可以将其视为一个更大的页面
  * 它们最常用于创建大页，在 hugetlbfs 或透明巨页面子系统中使用，但它们也出现在其他上下文中
* 复合页可以用作匿名内存或用作内核中的缓冲区；但是，它们不能出现在 page cache 中，page cache 只用于处理单例页面
* 可以调用一个普通的内存分配函数分配一个复合页面，比如`alloc_pages()`设置了`__GFP_COMP`分配标志和至少为`1`的阶（order）
  * 由于复合页面的实现方式，无法创建 *零阶（单页）* 的复合页面
  * 分配的“阶”是要分配的页数的以`2`为底的对数；因此，零对应于单页、一对应到两页等
* 请注意，复合页面不同于从正常的高阶分配请求返回的页面，例如以下语句将返回四个物理上连续的页面，但它们不是复合页面：
  ```c
  pages = alloc_pages(GFP_KERNEL, 2);  /* no __GFP_COMP */
  ```
  * 不同之处在于，创建复合页面涉及创建大量元数据；很多时候，不需要元数据，因此可以避免创建元数据的开销
* 大部分元数据存储在`page`结构中
  * 复合页面中的第一个（普通）页面称为“**head page**”；设置了`PG_head`标志
  * 所有其他页面都是“**tail pages**”；设置了`PG_tail`标志
* 无论使用哪种约定，如果传入的页面是复合页面，对`PageCompound()`的调用都将返回`true`
* 如果需要，可以使用`PageHead()`和`PageTail()`来区分 head page 和 tail pages
* 每个 tail pages 都有一个指向 head page 的指针，存储在`struct page`的`first_page`字段中，该和以下场景中的字段占用相同的存储空间：
  * `private`字段
  * 当页面保存页表条目时使用的自旋锁
  * 当页面由 slab 分配器拥有时使用的`slab_cache`指针
* `compound_head()`辅助函数可用于查找与任何 tail pages 关联的 head page
* 需要有一些信息描述了整个复合页面，比如：页面的 order（大小），以及用于在不再需要页面时将页面返回给系统的析构函数
  * 我们可能首先想到将该信息存储在 head page 的`struct page`中，但那里已经没有空间了
  * 相反，*order* 存储在第一个 tail page 的`struct page`结构的`lru.prev`字段中
    * 虽然`struct page`中的许多 union 字段被覆盖使用，但这里的 order 只是在存储在指针字段之前转换为指针类型
  * 类似地，*指向析构函数的指针* 存储在第一个 tail page 的`struct page`结构的`lru.next`字段中
  * 将复合页面元数据扩展到第二个`page`结构是复合页面必须至少包含两个页面的原因
* 内核中只声明了两个复合页面析构函数
  * 默认情况下，使用`free_compound_page()`；它所做的只是将内存返回给页面分配器
  * 不过，hugetlbfs 子系统使用`free_huge_page()`来保持其记帐是最新的
* 在大多数情况下，复合页面是不必要的，可以使用普通分配；（普通分配的）调用代码需要记住它分配了多少页，否则就不需要存储在复合页中的元数据
  * 但是，有时将页面组作为一个整体来处理很重要，即使有人只是引用其中的单个页面，也会指示出它属于一个复合页面
  * 透明巨页就是一个典型的例子；如果用户空间试图更改巨页的一部分的保护，则需要定位和分解整个巨页
  * 各种各样的驱动程序还使用复合页面来简化一些较大的缓冲区的管理

# Folio
* 复合页面的机制使得从 tail page 的`page`结构到复合页面的 head page 变得容易
* 内核中的许多接口都使用了该功能，但它产生了一个基本的歧义：如果一个函数被传递一个指向 tail page 的`page`结构的指针，那它是期望作用在该 tail page 还是整个复合页上？
* 具有`struct page`参数的函数可能需要一个 head page 或 base page，如果给定一个 tail page，则会出现 BUG
  * 它可能适用于任何类型的页面并以`PAGE_SIZE`字节操作
  * 它可以与任何类型的页面一起工作，如果给定一个 head page，它可以以`page_size()`字节进行操作，但是如果给定一个 base page 或 tail page，它以`PAGE_SIZE`字节操作
  * 如果传递了 head page 或 tail page，它可能以`page_size()`字节进行操作
  * `PAGE_SIZE`是 base page 的大小，而`page_size()`返回一个（可能是复合的）页面的完整大小
* 为了澄清这种情况，Wilcox 提出了“page filio”的概念，它实际上只是保证不是 tail page 的`page`结构
  * 任何接受 folio 的函数都将在完整的复合页面（如果确实是复合页面）上运行，而不会产生歧义
  * 结果是内核的内存管理子系统更加清晰；由于函数被转换为以 folios 作为参数，因此很明显它们不打算在 tail page 上运行
* 如果一个函数可能被传入一个 tail page，但它必须处理整个复合页，那么它必须调用内联函数`compound_head()`获取复合页的 head page 的`struct page`的地址
  * 在函数之间传递 tail page，每个函数都调用内联函数`compound_head()`造成的后果是内核变大和运行变慢
* 类型`struct folio`被定义为一个简单的包装结构体，如下：
  * include/linux/mm_types.h
  ```cpp
  struct folio {
    /* private: don't document the anon union */
    union {
      struct {
    /* public: */
        unsigned long flags;
        struct list_head lru;
        struct address_space *mapping;
        pgoff_t index;
        void *private;
        atomic_t _mapcount;
        atomic_t _refcount;
  #ifdef CONFIG_MEMCG
        unsigned long memcg_data;
  #endif
    /* private: the union with struct page is transitional */
      };
      struct page page;
    };
  };
  ```
* 在`struct folio`结构体的基础上建立一套新的基础设施，例如`folio_get()`和`folio_put()`将会像`get_page()`和`put_page()`一样管理 folio 的引用计数，但是不需要调用`compound_head()`

# References
* [LWN - An introduction to compound pages](https://lwn.net/Articles/619514/)
* [LWN - Clarifying memory management with page folios](https://lwn.net/Articles/849538/)