# slub

## 可回收的slab
* slab cache是否可回收可查看文件`/sys/kernel/slab/<slab name>/reclaim_account`。
* include/linux/slab.h

  ```c
  /* The following flags affect the page allocator grouping pages by mobility */
  #define SLAB_RECLAIM_ACCOUNT    0x00020000UL        /* Objects are reclaimable */
  #define SLAB_TEMPORARY      SLAB_RECLAIM_ACCOUNT    /* Objects are short-lived */
  ```

* mm/slub.c
  ```c
  static ssize_t reclaim_account_show(struct kmem_cache *s, char *buf)
  {
      return sprintf(buf, "%d\n", !!(s->flags & SLAB_RECLAIM_ACCOUNT));
  }

  static ssize_t reclaim_account_store(struct kmem_cache *s,
                  const char *buf, size_t length)
  {
      s->flags &= ~SLAB_RECLAIM_ACCOUNT;
      if (buf[0] == '1')
          s->flags |= SLAB_RECLAIM_ACCOUNT;
      return length;
  }
  SLAB_ATTR(reclaim_account);
  ```
* 由此可见，该标志位可以通过后期写文件的方式修改：

  ```
  # echo 1 > /sys/kernel/slab/kmalloc-4096/reclaim_account
  ```

* 该标志在`slabinfo`工具的`Flg`列对应的标志是小写`a`。

# slub调试

## 内核启动参数

```
slub_debug                              # Enable full debugging for all slabs.
slub_debug=<Debug-Options>              # Enable options for all slabs
slub_debug=<Debug-Options>,<slab name>  # Enable options only for select slabs

Possible debug options are
    F       Sanity checks on (enables SLAB_DEBUG_CONSISTENCY_CHECKS Sorry SLAB legacy issues)
    Z       Red zoning
    P       Poisoning (object and padding)
    U       User tracking (free and alloc)
    T       Trace (please only use on single slabs)
    A       Toggle failslab filter mark for the cache
    O       Switch debugging off for caches that would have caused higher minimum slab orders
    -       Switch all debugging off (useful if the kernel is configured with CONFIG_SLUB_DEBUG_ON)
```

* 系统启动后也可以调整slub调试功能，但比较有限。例如，打开某个slab的trace功能：
```
echo 1 > /sys/kernel/slab/<slab name>/trace
```

#### 为什么有的slab不能打开trace？
* 例如：
  ```
  # echo 1 > /sys/kernel/slab/kmalloc-4096/trace
  -sh: echo: write error: Invalid argument
  ```

* trace功能跟`SLAB_TRACE`标志位有关
  * include/linux/slab.h
  ```c
  #define SLAB_TRACE      0x00200000UL    /* Trace allocations and frees */
  ```
* 跟踪一个合并的cache会造成令人困惑的结果，并且会引起其他问题，比如会把一个可合并的的cache变得不可合并了

  * mm/slub.c
  ```c
  #define __CMPXCHG_DOUBLE    0x40000000UL /* Use cmpxchg_double */
  /* ... */
  static void trace(struct kmem_cache *s, struct page *page, void *object,
                                  int alloc)
  {
      if (s->flags & SLAB_TRACE) {
          pr_info("TRACE %s %s 0x%p inuse=%d fp=0x%p\n",
              s->name,
              alloc ? "alloc" : "free",
              object, page->inuse,
              page->freelist);

          if (!alloc)
              print_section("Object ", (void *)object,
                      s->object_size);

          dump_stack();
      }    
  }
  /* ... */
  static ssize_t trace_show(struct kmem_cache *s, char *buf)
  {
      return sprintf(buf, "%d\n", !!(s->flags & SLAB_TRACE));
  }   

  static ssize_t trace_store(struct kmem_cache *s, const char *buf,
                              size_t length)
  {    
      /*
       * Tracing a merged cache is going to give confusing results
       * as well as cause other issues like converting a mergeable
       * cache into an umergeable one.
       */
      /**/
      if (s->refcount > 1)
          return -EINVAL;

      s->flags &= ~SLAB_TRACE;
      if (buf[0] == '1') {
          s->flags &= ~__CMPXCHG_DOUBLE;
          s->flags |= SLAB_TRACE;
      }   
      return length;
  }   
  SLAB_ATTR(trace);
  /* ... */
  ```

##### 如何检查`s->refcount`?
```
# cat /sys/kernel/slab/kmalloc-4096/aliases
1
```
* 即是`aliases`文件的内容
* mm/slub.c
```c
static ssize_t aliases_show(struct kmem_cache *s, char *buf)
{
    return sprintf(buf, "%d\n", s->refcount < 0 ? 0 : s->refcount - 1);
}
SLAB_ATTR_RO(aliases);
```

##### `s->refcount`何时会增加？
* 在创建一个新的slab高速缓存时，调用的`kmem_cache_create()`会调用`__kmem_cache_alias()`
* `__kmem_cache_alias()`会查看将要创建的slab是否能合并到已有的slab，如果可以，则在`/sys/kernel/slab/`目录下建立一个别名就可以了，同时会增加`s->refcount`。
* mm/slub.c
```c
struct kmem_cache *
__kmem_cache_alias(const char *name, size_t size, size_t align,
           unsigned long flags, void (*ctor)(void *))
{
    struct kmem_cache *s, *c;

    s = find_mergeable(size, align, flags, name, ctor);
    if (s) {
        s->refcount++;
        /*...*/
        /*增加引用计数后，在sysfs下创建别名；
          如果不成功会导致kmem_cache_create()创建一个采用真名的slab*/
        if (sysfs_slab_alias(s, name)) {
            s->refcount--;
            s = NULL;
        }
    }

    return s;
}  
```

### slub合并/别名

#### 怎么样的slab是可以合并的？

```c
/*
 * Set of flags that will prevent slab merging
 */
#define SLAB_NEVER_MERGE (SLAB_RED_ZONE | SLAB_POISON | SLAB_STORE_USER | \
                SLAB_TRACE | SLAB_DESTROY_BY_RCU | SLAB_NOLEAKTRACE | \
                SLAB_FAILSLAB)

#define SLAB_MERGE_SAME (SLAB_DEBUG_FREE | SLAB_RECLAIM_ACCOUNT | \
                SLAB_CACHE_DMA | SLAB_NOTRACK)

struct kmem_cache *find_mergeable(size_t size, size_t align,
                unsigned long flags, const char *name, void (*ctor)(void *))
{    
        struct kmem_cache *s;

        if (slab_nomerge || (flags & SLAB_NEVER_MERGE))
                return NULL;

        if (ctor)
                return NULL;

        size = ALIGN(size, sizeof(void *));
        align = calculate_alignment(flags, align, size);
        size = ALIGN(size, align);
        flags = kmem_cache_flags(size, flags, name, NULL);

        list_for_each_entry_reverse(s, &slab_caches, list) {
                if (slab_unmergeable(s))
                        continue;

                if (size > s->size)
                        continue;

                if ((flags & SLAB_MERGE_SAME) != (s->flags & SLAB_MERGE_SAME))
                        continue;
                /*
                 * Check if alignment is compatible.
                 * Courtesy of Adrian Drzewiecki
                 */
                if ((s->size & ~(align - 1)) != s->size)
                        continue;

                if (s->size - size >= sizeof(void *))
                        continue;

                if (IS_ENABLED(CONFIG_SLAB) && align &&
                        (align > s->align || s->align % align))
                        continue;

                return s;
        }
        return NULL;
}
```

#### 怎么样的slab是不能合并的？

#### 怎么样可以让原来合并的slub不合并？
* 内核启动参数`slab_nomerge`和`slub_nomerge`可以全局地控制是否合并相同大小的slab。
```
slab_nomerge    [MM]
                Disable merging of slabs with similar size. May be
                necessary if there is some reason to distinguish
                allocs to different slabs. Debug options disable
                merging on their own.
                For more information see Documentation/vm/slub.txt.

slub_nomerge    [MM, SLUB]
                Same with slab_nomerge. This is supported for legacy.
                See slab_nomerge for more information.
```

* mm/slab_common.c
```c
/*
 * Merge control. If this is set then no merging of slab caches will occur.
 * (Could be removed. This was introduced to pacify the merge skeptics.)
 */
static int slab_nomerge;

static int __init setup_slab_nomerge(char *str)
{
    slab_nomerge = 1;
    return 1;
}

#ifdef CONFIG_SLUB
__setup_param("slub_nomerge", slub_nomerge, setup_slab_nomerge, 0);
#endif

__setup("slab_nomerge", setup_slab_nomerge);
```
* 当然，也可以通过内核参数的方式，**只让指定slub名称的方式让特定的slub变成unmergable状态**：
  ```
  slub_debug=P,request_sock_TCP
  ```
* 调用过程如下：
```c
kmem_cache_create()
  |
  +-->__kmem_cache_alias()
  |       |
  |       +-->find_mergeable()
  |             |
  |             +-->kmem_cache_flags()
  |
  +-->do_kmem_cache_create()
        |
        +-->__kmem_cache_create()
              |
              +-->kmem_cache_open()
                    |
                    +-->kmem_cache_flags()
```
* 它们都会调用`kmem_cache_flags()`看是否需要根据内核参数`slub_debug=`新增指定的flags：
* mm/slub.c
```c
#define DEBUG_DEFAULT_FLAGS (SLAB_DEBUG_FREE | SLAB_RED_ZONE | \
                                SLAB_POISON | SLAB_STORE_USER)
/*...*/
static int __init setup_slub_debug(char *str)
{
        slub_debug = DEBUG_DEFAULT_FLAGS;
        if (*str++ != '=' || !*str)
                /*
                 * No options specified. Switch on full debugging.
                 */
                goto out;

        if (*str == ',')
                /*
                 * No options but restriction on slabs. This means full
                 * debugging for slabs matching a pattern.
                 */
                goto check_slabs;

        slub_debug = 0;
        if (*str == '-')
                /*
                 * Switch off all debugging measures.
                 */
                goto out;

        /*
         * Determine which debug features should be switched on
         */
        for (; *str && *str != ','; str++) {
                switch (tolower(*str)) {
                case 'f':
                        slub_debug |= SLAB_DEBUG_FREE;
                        break;
                case 'z':
                        slub_debug |= SLAB_RED_ZONE;
                        break;
                case 'p':
                        slub_debug |= SLAB_POISON;
                        break;
                case 'u':
                        slub_debug |= SLAB_STORE_USER;
                        break;
                case 't':
                        slub_debug |= SLAB_TRACE;
                        break;
                case 'a':
                        slub_debug |= SLAB_FAILSLAB;
                        break;
                case 'o':
                /*
                 * Avoid enabling debugging on caches if its minimum
                 * order would increase as a result.
                 */
                disable_higher_order_debug = 1;
                break;
        default:
                pr_err("slub_debug option '%c' unknown. skipped\n",
                       *str);
        }
}

check_slabs:
        if (*str == ',')
                slub_debug_slabs = str + 1;
out:
        return 1;
}

__setup("slub_debug", setup_slub_debug);

unsigned long kmem_cache_flags(unsigned long object_size,
        unsigned long flags, const char *name,
        void (*ctor)(void *))
{
/*
 * Enable debugging if selected on the kernel commandline.
 */
if (slub_debug && (!slub_debug_slabs || (name &&
        !strncmp(slub_debug_slabs, name, strlen(slub_debug_slabs)))))
        flags |= slub_debug;

return flags;
}
```
* 之前的`request_sock_TCP` slub的可合并状态就是在这个过程中被改变的。
* 然而，对于有些slub，这种方式是无效的，比如：
  ```
  slub_debug=P,kmalloc-64
  ```
* 原因是，`kmalloc-*`一族的slub分配方式比较特殊，基本过程是
  * 先以匿名方式创建一组`kmalloc_caches[]`
  * 然后在分别给这个数组中的slub分别赋予`kmalloc-*`的名字
  * 然而`kmem_cache_flags()`函数会在这之前会被调用，当时这些slub还没有名字，当然不会与内核参数里指定的slub匹配。

```c
start_kernel()
  |
  +-->mm_init()
        |
        +-->kmem_cache_init()
              |
              +-->create_kmalloc_caches()
                    |
                    +-->create_kmalloc_cache()
                    |     |
                    |     +-->create_boot_cache()
                    |           |
                    |           +-->__kmem_cache_create()
                    |                 |
                    |                 +-->kmem_cache_open()
                    |                       |
                    |                       +-->kmem_cache_flags()
                    |
                    +-->for (i = 0; i <= KMALLOC_SHIFT_HIGH; i++) {
                            struct kmem_cache *s = kmalloc_caches[i];
                            char *n;
                            if (s) {
                               n = kasprintf(GFP_NOWAIT, "kmalloc-%d", kmalloc_size(i));

                               BUG_ON(!n);
                               s->name = n;
                            }
                        }
```

* 这种情况下，只能通过一些hack的方式才能使特定的`kmalloc-*`独立出来：
```c
diff --git a/mm/slab_common.c b/mm/slab_common.c
@@ -58,6 +58,16 @@ __setup_param("slub_nomerge", slub_nomerge, setup_slab_nomerge, 0);

 __setup("slab_nomerge", setup_slab_nomerge);

+static int kmalloc_nomerge = 64;  /* kmalloc-64 is unmergable by default */
+
+static int __init setup_kmalloc_nomerge(char *str)
+{
+   get_option(&str, &kmalloc_nomerge);
+   return 1;
+}
+
+__setup("kmalloc_nomerge=", setup_kmalloc_nomerge);
+
 /*
  * Determine the size of a slab object
  */
@@ -835,8 +845,13 @@ void __init create_kmalloc_caches(unsigned long flags)
    }
    for (i = KMALLOC_SHIFT_LOW; i <= KMALLOC_SHIFT_HIGH; i++) {
        if (!kmalloc_caches[i]) {
-           kmalloc_caches[i] = create_kmalloc_cache(NULL,
-                           1 << i, flags);
+           if ((1 << i) != kmalloc_nomerge) {
+               kmalloc_caches[i] = create_kmalloc_cache(NULL,
+                               1 << i, flags);
+           } else { /* don't use SLAB_NEVER_MERGE, that will enable lots of flags */
+               kmalloc_caches[i] = create_kmalloc_cache(NULL,
+                               1 << i, flags | SLAB_POISON);
+           }
        }

        /*

```

## 参考资料
- [怎样诊断SLAB泄露问题](http://linuxperf.com/?p=148)
