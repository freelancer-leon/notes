# slab层（Slab Layer）

* slab分配器扮演了 **通用数据结构缓存** 的角色。
* slab分配器试图在几个基本原则之间寻求一种平衡：
  * 频繁使用的数据结构也会频繁分配和释放，因此应当缓存它们。
  * 频繁地分配和回收必然会导致内存碎片（难以找到大块连续可用的内存）。为了避免这种现象，空闲链表的缓存会连续地存放。因为已经释放的数据结构又会放回空闲链表，因此不会导致碎片。
  * 回收的对象可以立即投入下一次分配，因此，对于频繁的分配和释放，空闲链表能够提高其性能。
  * 如果分配器知道对象大小，页大小和总的高速缓存的大小这样的概念，它会作出更明智的决策。
  * 如果让部分缓存专属于单个处理器（对系统上的每个处理器独立而唯一），那么，分配和释放就可以在不加SMP锁的情况下运行。
  * 如果分配器是与NUMA相关的，它就可以从相同的内存结点为请求者进行分配。
  * 对存放的对象进行着色（color），以防止多个对象映射到相同的高速缓存行（cache line）。

## slab层的设计
![mm_slab](pic/mm_slab_1.gif)
* 不同对象划分为 **高速缓存组**，其中每个高速缓存组都存放 **不同类型** 的对象。**每种对象类型** 对应一个高速缓存。
* `kmalloc()`接口建立在slab层之上，使用了一组通用高速缓存。
* 高速缓存被划分为 **slab**，slab由一个或多个物理上连续的页组成。
* 一般情况下，slab也就仅仅由一页组成。每个高速缓存可以由多个slab组成。
* 每个slab都包含一些被缓存的数据结构。
* 每个slab处于三种状态之一：满，部分满，空。
* 当内核某一部分需要一个新对象时：
  * 先从 *部分满* 的slab中进行分配。
  * 没有 *部分满*，则从 *空* slab中进行分配。
  * 没有 *空* slab，则创建一个slab。
* 注意 slabs_empty 链表中的 slab 是进行 **回收（reaping）** 的主要备选对象。正是通过此过程，slab 所使用的内存被返回给操作系统供其他用户使用。
* slab 链表中的每个 slab 都是一个连续的内存块（一个或多个连续页），它们被划分成一个个对象。这些对象是从特定缓存中进行分配和释放的基本元素。
* 注意 slab 是 slab 分配器进行操作的最小分配单位，因此如果需要对 slab 进行扩展，这也就是所扩展的最小值。
* 由于对象是从 slab 中进行分配和释放的，因此单个 slab 可以在 slab 链表之间进行移动。
  * 例如，当一个 slab 中的所有对象都被使用完时，就从 slabs_partial 链表中移动到 slabs_full 链表中。
  * 当一个 slab 完全被分配并且有对象被释放后，就从 slabs_full 链表中移动到 slabs_partial 链表中。
  * 当所有对象都被释放之后，就从 slabs_partial 链表移动到 slabs_empty 链表中。

### slab 背后的动机
与传统的内存管理模式相比， slab 缓存分配器提供了很多优点。
* 首先，内核通常依赖于对小对象的分配，它们会在系统生命周期内进行无数次分配。slab 缓存分配器通过对类似大小的对象进行缓存而提供这种功能，从而避免了常见的碎片问题。
* slab 分配器还支持通用对象的初始化，从而避免了为同一目而对一个对象重复进行初始化。
* 最后，slab 分配器还可以支持硬件缓存对齐和着色，这允许不同缓存中的对象占用相同的缓存行，从而提高缓存的利用率并获得更好的性能。

![https://hammertux.github.io/img/SLAB-DS.png](pic/SLAB-DS.png)

### 高速缓存结构 kmem_cache

* 每个高速缓存都用`kmem_cache`结构表示。
* include/linux/slab_def.h
```c
/*
 * Definitions unique to the original Linux SLAB allocator.
 */

struct kmem_cache {
    /*per-cpu数据，每次分配/释放期间都会访问*/
    struct array_cache __percpu *cpu_cache;

/* 1) Cache tunables. Protected by slab_mutex */
    unsigned int batchcount;
    unsigned int limit;
    unsigned int shared;

    unsigned int size;
    struct reciprocal_value reciprocal_buffer_size;
/* 2) touched by every alloc & free from the backend */

    unsigned int flags;     /* constant flags */
    unsigned int num;       /* # of objs per slab */

/* 3) cache_grow/shrink */
    /* order of pgs per slab (2^n) */
    unsigned int gfporder;

    /* force GFP flags, e.g. GFP_DMA */
    gfp_t allocflags;

    size_t colour;          /* cache colouring range */
    unsigned int colour_off;    /* colour offset */
    struct kmem_cache *freelist_cache;
    unsigned int freelist_size;

    /* constructor func */
    void (*ctor)(void *obj);

/* 4) cache creation/removal */
    const char *name;
    struct list_head list;
    int refcount;
    int object_size;
    int align;

/* 5) statistics */
...

    struct kmem_cache_node *node[MAX_NUMNODES];
};
...__```
```
### Slab 空闲对象数组
![https://flylib.com/books/4/454/1/html/2/images/04fig08.jpg](pic/slab_bufctl.jpg)
* slab cache 在创建的时候会通过`calculate_slab_order() -> cache_estimate()`计算出`cachep->num`，即每个 slab 可存放的 object 数目
* slab 管理数据除了`struct slab`结构外，还包含一个有`cachep->num`个元素的数组来指示 slab object 的使用情况
* 这个数组原来放在 `struct slab`结构之后，着色区之前
### 三个 slab 链表和 Per-CPU 的 array_cache

* mm/slab.h
```c
/*
 * struct array_cache
 *
 * Purpose:
 * - LIFO ordering, to hand out cache-warm objects from _alloc
 * - reduce the number of linked list operations
 * - reduce spinlock operations
 *
 * The limit is stored in the per-cpu structure to reduce the data cache
 * footprint.
 *
 */
struct array_cache {
    unsigned int avail;
    unsigned int limit;
    unsigned int batchcount;
    unsigned int touched;
    void *entry[];  /*
             * Must have this definition in here for the proper
             * alignment of array_cache. Also simplifies accessing
             * the entries.
             */
};
...
/*
 * The slab lists for all objects.
 */
struct kmem_cache_node {
    spinlock_t list_lock;

#ifdef CONFIG_SLAB
    struct list_head slabs_partial; /* partial list first, better asm code */
    struct list_head slabs_full;
    struct list_head slabs_free;
    unsigned long free_objects;
    unsigned int free_limit;
    unsigned int colour_next;   /* Per-node cache coloring */
    struct array_cache *shared; /* shared per node */
    struct alien_cache **alien; /* on other nodes */
    unsigned long next_reap;    /* updated without locking */
    int free_touched;       /* updated without locking */
#endif

#ifdef CONFIG_SLUB
    unsigned long nr_partial;
    struct list_head partial;
#ifdef CONFIG_SLUB_DEBUG
    atomic_long_t nr_slabs;
    atomic_long_t total_objects;
    struct list_head full;
#endif
#endif

};
...*```
```

* `kmem_getpages()`创建新的slab。
  * `kmem_getpages()`和`alloc_pages()`都会调用`__alloc_pages_nodemask()`来分配页。
* `kmem_freepages()`释放内存。
* slab 层只有当给定高速缓存部分中即 **没有满** 也 **没有空** 的 slab 的时才会调用页分配函数。
* 只有在下列情况下才会调用 *释放函数*：
  * 当可用内存变得紧缺时，系统试图释放出更多的内存以供使用；
  * 当高速缓存显示撤销时。
* slab 层的管理是在每个高速缓存的基础上，通过提供给整个系统一个简单的接口来完成的。通过接口就可以：
  * 创建和撤销新的高速缓存。
  * 在高速缓存内分配和释放对象。
* 创建一个高速缓存后，slab层所起的作用就像一个专用的分配器，可以为具体的对象类型进行分配。
#### array_cache
* 每个 slab 会建立一个 Per-CPU 的`array_cache`，`kmem_cache`的`cpu_cache`域指向这个Per-CPU变量
  * 每个 Per-CPU 变量`array_cache`里又含有一个数组，数组有若干条目，指向该 slab 的对象
* 分配和释放对象都优先在`array_cache`里进行，减小开销和锁的使用
  * 通常分配会使`ac->avail`会减小
  * 通常释放会使`ac->avail`会增大
* 当请求 **分配 slab 对象** 的时候，如果`array_cache`里没有足够的空闲对象时，调用`cache_alloc_refill()`重填`array_cache`
  * 在 *部分使用* 和 *空闲* 链表上找到`batchcount`个对象，把对象的地址填到`array_cache`的数组里
  * 如果连 *空闲* 链表上都没有空闲对象，调用`cache_grow()`分配新的 shab
  * 也就是说，重填完后`array_cache`会有`batchcount-1`个空闲对象可用于下次分配，那减去的一个对象在本次分配返回时被使用
  * `cache_alloc_refill()`完成后`ac->avail`增加了（非通常情况）
* 当 **释放 slab 对象** 的时候，`array_cache`里的可用条目`ac->avail`比限定的`ac->limit`多（或等于）
  * 调用`cache_flusharray()`归还 **前`batchcount`** 个空闲对象到各自的 slab （LIFO，故前面的对象没那么热）
    * 先将该空闲对象所属的 slab 从链表上取下
    * 归还空闲对象给 slab
    * 如果 slab 全为空闲对象，根据情况看是要销毁 slab 还是将 slab 挂回到 *空闲链表* 上
    * 否则挂回 *部分使用链表* 上
    * 将空闲条目数目`ac->avail`减小`batchcount`（因为已经还回去了）
    * 将（减小后的）`ac->avail`个空闲条目移到前面
  * 然后再把要 free 的对象放到`array_cache`条目上
  * 所以`cache_flusharray()`完成后`ac->avail`反而减小了（非通常情况）

#### __SetPageSlab()宏
* include/linux/page-flags.h
```c
enum pageflags {
  PG_locked,      /* Page is locked. Don't touch. */
  PG_error,
  PG_referenced,
  PG_uptodate,
  PG_dirty,
  PG_lru,
  PG_active,
  PG_slab,
...
};
...
static inline struct page *compound_head(struct page *page)
{
    unsigned long head = READ_ONCE(page->compound_head);

    if (unlikely(head & 1))
        return (struct page *) (head - 1);
    return page;
}

static __always_inline int PageTail(struct page *page)
{
    return READ_ONCE(page->compound_head) & 1;
}
...
/*
 * Page flags policies wrt compound pages
 *
 * PF_ANY:
 *     the page flag is relevant for small, head and tail pages.
 *
 * PF_HEAD:
 *     for compound page all operations related to the page flag applied to
 *     head page.
 *
 * PF_NO_TAIL:
 *     modifications of the page flag must be done on small or head pages,
 *     checks can be done on tail pages too.
 *
 * PF_NO_COMPOUND:
 *     the page flag is not relevant for compound pages.
 */
#define PF_ANY(page, enforce)   page
#define PF_HEAD(page, enforce)  compound_head(page)
#define PF_NO_TAIL(page, enforce) ({                    \
        VM_BUG_ON_PGFLAGS(enforce && PageTail(page), page); \
        compound_head(page);})
#define PF_NO_COMPOUND(page, enforce) ({                \
        VM_BUG_ON_PGFLAGS(enforce && PageCompound(page), page); \
        page;})

/*
 * Macros to create function definitions for page flags
 */
#define TESTPAGEFLAG(uname, lname, policy)              \
static __always_inline int Page##uname(struct page *page)       \
    { return test_bit(PG_##lname, &policy(page, 0)->flags); }
...
#define __SETPAGEFLAG(uname, lname, policy)             \
static __always_inline void __SetPage##uname(struct page *page)     \
    { __set_bit(PG_##lname, &policy(page, 1)->flags); }

#define __CLEARPAGEFLAG(uname, lname, policy)               \
static __always_inline void __ClearPage##uname(struct page *page)   \
    { __clear_bit(PG_##lname, &policy(page, 1)->flags); }
...
#define __PAGEFLAG(uname, lname, policy)                \
    TESTPAGEFLAG(uname, lname, policy)              \
    __SETPAGEFLAG(uname, lname, policy)             \
    __CLEARPAGEFLAG(uname, lname, policy)

...
__PAGEFLAG(Slab, slab, PF_NO_TAIL)
```
* 宏展开后的函数定义
```c
/*__ arch/x86/include/asm/bitops.h*/
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

static __always_inline void __SetPageSlab(struct page *page)
    { __set_bit(PG_slab, &({
      do {
        if (unlikely(1 && PageTail(page))) {
            dump_page(page, "VM_BUG_ON_PAGE(" __stringify(1 && PageTail(page))")");
            BUG();
        }
      } while (0);
      compound_head(page);})->flags); }
...___```
```

## slab分配器的接口

* `kmem_cache_create()`创建一个新的高速缓存。
  * `SLAB_HWCACHE_LINE` slab 层把一个 slab 内的所有对象和硬件 cache line 对齐。可以提高性能，但增加了内存开销，空间换时间。
  * `SLAB_POISON` 内存毒化标志，用已知的值填充 slab。
  * `SLAB_RED_ZONE` 在已分配的内存周围插入“红色警界区”以探测缓冲越界。
  * `SLAB_PANIC` 分配失败时提醒 slab 层。这在要求分配只能成功的时候非常有用。
  * `SLAB_CACHE_DMA` 命令 slab 层用 *可以执行 DMA 的内存* 给每个 slab 分配空间。
    * 只在 *分配的对象用于 DMA*，且 *必须驻留在`ZONE_DMA`区`时* 才需要这个标志。
    * 否则不需要，也不应该设置。
  * **不能用于中断上下文**，会睡眠。
* `kmem_cache_destory()`撤销给定的高速缓存。
  * **不能用于中断上下文**，会睡眠。
  * 除此之外还需确保：
    * 高速缓存中所有的 slab 都必须为空。
    * 在调用`kmem_cache_destory()`过程中（更别提之后了）不再访问这个高速缓存。调用者必须确保同步。
* `kmem_cache_alloc()`从给定的高速缓存中分配对象。
  * 如果高速缓存中所有的 slab 中都没有空闲对象，slab 层必须通过`kmem_getpages()`获取新的页。
* `kmem_cache_free()`释放一个对象，并返还给原先的 slab。
* slab层负责内存紧缺情况下所有底层的对齐，着色，分配，释放，回收等。
* 如果要频繁创建很多相同类型的对象，应该考虑使用 slab 高速缓存，而不是自己实现空闲链表。

## 查看slab信息
* [`cat /proc/slabinfo`](http://man7.org/linux/man-pages/man5/slabinfo.5.html)
  * `limit` 字段表示每个 CPU 可以缓存的对象的最大数量。
  * `batchcount` 字段是当缓存为空时，转换到每个 CPU 缓存中全局缓存对象的最大数量。
  * `shared` 参数说明了 SMP 系统的共享行为。
* [`slbtop`](http://man7.org/linux/man-pages/man1/slabtop.1.html)

# 参考资料
* [The Slab Allocator in the Linux kernel](https://hammertux.github.io/slab-allocator)
* [Section 4.4. Slab Allocator _ The Linux Kernel Primer. A Top-Down Approach for x86 and PowerPC Architectures](https://flylib.com/books/en/4.454.1.55/1/)
