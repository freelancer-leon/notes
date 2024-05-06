# 直接映射

基于[原文：kernel_physical_mapping_init](https://zhuanlan.zhihu.com/p/592840968)做了一些修改

## PAGE_OFFSET
![物理地址的线性映射](https://pic2.zhimg.com/80/v2-2c6ff05812de6570014eca023cb1c9b9_1440w.webp)
* 直接映射在没有开启 `CONFIG_RANDOMIZE_MEMORY` 的情况下，`__PAGE_OFFSET` 就等于 `0xffff_8880_0000_0000`
* 如果 `CONFIG_RANDOMIZE_MEMORY` 开启，则 `__PAGE_OFFSET` 等于一个全局变量 `page_offset_base` 的值
  * 如果开启了 5 级页表，该值会变为 `__PAGE_OFFSET_BASE_L5` 即 `0xff11_0000_0000_0000`
```cpp
//arch/x86/include/asm/page_64_types.h
#define __PAGE_OFFSET_BASE_L5   _AC(0xff11000000000000, UL)
#define __PAGE_OFFSET_BASE_L4   _AC(0xffff888000000000, UL)

#ifdef CONFIG_DYNAMIC_MEMORY_LAYOUT
#define __PAGE_OFFSET           page_offset_base
#else
#define __PAGE_OFFSET           __PAGE_OFFSET_BASE_L4
#endif /* CONFIG_DYNAMIC_MEMORY_LAYOUT */

//arch/x86/include/asm/page_types.h
#define PAGE_OFFSET     ((unsigned long)__PAGE_OFFSET)
```
## 几个填充页表条目的函数
* 定义 `set_p*_init()` 一族函数的宏原型如下，就是多了 `init` 的判断，如果 `init = true` 那么用 `set_p*_safe()` 一族的函数：
  * arch/x86/mm/init_64.c
```cpp
#define DEFINE_ENTRY(type1, type2, init)            \
static inline void set_##type1##_init(type1##_t *arg1,      \
            type2##_t arg2, bool init)      \
{                               \
    if (init)                       \
        set_##type1##_safe(arg1, arg2);         \
    else                            \
        set_##type1(arg1, arg2);            \
}

DEFINE_ENTRY(p4d, p4d, init)
DEFINE_ENTRY(pud, pud, init)
DEFINE_ENTRY(pmd, pmd, init)
DEFINE_ENTRY(pte, pte, init)
```
* `set_p*_safe()` 一族的函数定义如下：
  * include/linux/pgtable.h
```cpp
/*
 * Use set_p*_safe(), and elide TLB flushing, when confident that *no*
 * TLB flush will be required as a result of the "set". For example, use
 * in scenarios where it is known ahead of time that the routine is
 * setting non-present entries, or re-setting an existing entry to the
 * same value. Otherwise, use the typical "set" helpers and flush the
 * TLB.
 */
#define set_pte_safe(ptep, pte) \
({ \
    WARN_ON_ONCE(pte_present(*ptep) && !pte_same(*ptep, pte)); \
    set_pte(ptep, pte); \
})

#define set_pmd_safe(pmdp, pmd) \
({ \
    WARN_ON_ONCE(pmd_present(*pmdp) && !pmd_same(*pmdp, pmd)); \
    set_pmd(pmdp, pmd); \
})

#define set_pud_safe(pudp, pud) \
({ \
    WARN_ON_ONCE(pud_present(*pudp) && !pud_same(*pudp, pud)); \
    set_pud(pudp, pud); \
})

#define set_p4d_safe(p4dp, p4d) \
({ \
    WARN_ON_ONCE(p4d_present(*p4dp) && !p4d_same(*p4dp, p4d)); \
    set_p4d(p4dp, p4d); \
})

#define set_pgd_safe(pgdp, pgd) \
({ \
    WARN_ON_ONCE(pgd_present(*pgdp) && !pgd_same(*pgdp, pgd)); \
    set_pgd(pgdp, pgd); \
})
```
* 当确信由于“set”而不需要 TLB 刷新时，使用 `set_p*_safe()` 并省略 TLB 刷新。
  * 例如，在提前知道例程 *正在设置不存在的条目* 或 *将现有条目重新设置为相同值* 的情况下使用。否则，使用典型的“set” helpers 并刷新 TLB。

## 分割映射范围
* `struct map_range` 结构，描述一个映射的范围
```cpp
struct map_range {
    unsigned long start;
    unsigned long end;
    unsigned page_size_mask;
};
```
* `split_mem_range()` 将一段物理地址范围 `[start, end)` 分割到 `struct map_range[]` 描述的数组
  * arch/x86/mm/init.c
```cpp
static int __meminit split_mem_range(struct map_range *mr, int nr_range,
                     unsigned long start,
                     unsigned long end)
{
    unsigned long start_pfn, end_pfn, limit_pfn;
    unsigned long pfn; //我们将 pfn 称为计算页帧，它随着计算的过程不停地往前移动
    int i;
    //结束物理地址向下取整转成限制物理页帧
    limit_pfn = PFN_DOWN(end);
    //起始物理地址向下取整转成起始物理页帧和计算页帧
    /* head if not big page alignment ? */
    pfn = start_pfn = PFN_DOWN(start);
#ifdef CONFIG_X86_32
...
#else /* CONFIG_X86_64 */
    end_pfn = round_up(pfn, PFN_DOWN(PMD_SIZE)); //计算页帧向上以 2M 对齐作为结束物理页帧
#endif
    if (end_pfn > limit_pfn)   //结束页帧向上取整如果操作了限制页帧
        end_pfn = limit_pfn;   //将结束页帧调整为限制页帧
    if (start_pfn < end_pfn) { //保存这个范围，nr_range 会加 1 后返回
        nr_range = save_mr(mr, nr_range, start_pfn, end_pfn, 0); //最后一个参数是 0，这个范围不能用大页映射
        pfn = end_pfn;         //递进计算页帧为刚才的结束页帧
    }
    //后移起始页帧到计算页帧，此处需要 2M 对齐
    /* big page (2M) range */
    start_pfn = round_up(pfn, PFN_DOWN(PMD_SIZE));
#ifdef CONFIG_X86_32
...
#else /* CONFIG_X86_64 */
    end_pfn = round_up(pfn, PFN_DOWN(PUD_SIZE)); //计算页帧向上以 1G 对齐作为新的结束页帧
    if (end_pfn > round_down(limit_pfn, PFN_DOWN(PMD_SIZE))) //新的结束页帧如果超过了限制页帧向下对齐到 2M
        end_pfn = round_down(limit_pfn, PFN_DOWN(PMD_SIZE)); //将结束页帧调整为限制页帧向下对齐到 2M
#endif
    //保存这个 1G 范围
    if (start_pfn < end_pfn) {
        nr_range = save_mr(mr, nr_range, start_pfn, end_pfn,
                page_size_mask & (1<<PG_LEVEL_2M)); //这个范围可以使用 2M 的大页映射
        pfn = end_pfn; //递进计算页帧为刚才的结束页帧
    }
    //后移起始页帧到计算页帧，此处需要 1G 对齐
#ifdef CONFIG_X86_64
    /* big page (1G) range */
    start_pfn = round_up(pfn, PFN_DOWN(PUD_SIZE)); //计算页帧向上以 1G 对齐作为新的起始页帧
    end_pfn = round_down(limit_pfn, PFN_DOWN(PUD_SIZE)); //限制页帧向下以 1G 对齐作为新的结束页帧
    if (start_pfn < end_pfn) { //保存这个 1G 范围
        nr_range = save_mr(mr, nr_range, start_pfn, end_pfn,
                page_size_mask &
                 ((1<<PG_LEVEL_2M)|(1<<PG_LEVEL_1G))); //这个范围可以使用 2M 或 1G 的大页映射
        pfn = end_pfn; //递进计算页帧为刚才的结束页帧
    }
    //因为上面把限制页帧向下对齐到了 1G 作为结束页帧，剩下的页帧肯定不足 1G
    /* tail is not big page (1G) alignment */
    start_pfn = round_up(pfn, PFN_DOWN(PMD_SIZE)); //计算页帧向上以 2M 对齐作为新的起始页帧
    end_pfn = round_down(limit_pfn, PFN_DOWN(PMD_SIZE)); //限制页帧向下以 2M 对齐作为新的结束页帧
    if (start_pfn < end_pfn) { //保存这个 2M 范围
        nr_range = save_mr(mr, nr_range, start_pfn, end_pfn,
                page_size_mask & (1<<PG_LEVEL_2M)); //这个范围可以使用 2M 的大页映射
        pfn = end_pfn; //递进计算页帧为刚才的结束页帧
    }
#endif
    //因为上面把限制页帧向下对齐到了 2M 作为结束页帧，剩下的页帧肯定不足 2M
    /* tail is not big page (2M) alignment */
    start_pfn = pfn;     //后移起始页帧到计算页帧，不足 2M 不需要再对齐了
    end_pfn = limit_pfn; //限制页帧作为新的结束页帧
    nr_range = save_mr(mr, nr_range, start_pfn, end_pfn, 0); //保存 4K 范围
    //如果启动内存还没设置完成，为小的范围调整 page_size_mask，如果附近也有 ram 则采用大页面大小，而不是小页面大小。
    if (!after_bootmem)
        adjust_range_page_size_mask(mr, nr_range);
    //尽可能将有连续且有相同页大小的范围合并
    /* try to merge same page size and continuous */
    for (i = 0; nr_range > 1 && i < nr_range - 1; i++) {
        unsigned long old_start;
        if (mr[i].end != mr[i+1].start ||                   //如果范围不连续，
            mr[i].page_size_mask != mr[i+1].page_size_mask) //或连续的两个范围可用的页大小不一致
            continue;                                       //跳过这个范围
        /* move it */             //如果两个范围连续且可用的页大小一致
        old_start = mr[i].start;  //先记录下第一个范围的起始页帧
        memmove(&mr[i], &mr[i+1],
            (nr_range - 1 - i) * sizeof(struct map_range)); //将第二个范围及其以后的所有范围移动到第一个范围的位置
        //移动后第一个范围的起始地址刚才被覆盖了，之前的用记录恢复回来；
        //且 i 需减一以抵消 for 迭代一次后的加一操作，让下次迭代仍在该范围操作
        mr[i--].start = old_start;
        nr_range--; //合并了一个范围，范围数组计算减一
    }
    //打印拆分后的范围数组
    for (i = 0; i < nr_range; i++)
        pr_debug(" [mem %#010lx-%#010lx] page %s\n",
                mr[i].start, mr[i].end - 1,
                page_size_string(&mr[i]));

    return nr_range;
}
```

## 初始化内存映射
* `init_memory_mapping()` 在 `PAGE_OFFSET` 设置物理内存的直接映射。
  * 它在 bootmem 初始化之前运行，并直接从物理内存获取页面。为了访问它们，它们被临时映射。
```cpp
/*
 * Setup the direct mapping of the physical memory at PAGE_OFFSET.
 * This runs before bootmem is initialized and gets pages directly from
 * the physical memory. To access them they are temporarily mapped.
 */
unsigned long __ref init_memory_mapping(unsigned long start,
                    unsigned long end, pgprot_t prot)
{
    struct map_range mr[NR_RANGE_MR];
    unsigned long ret = 0;
    int nr_range, i;

    pr_debug("init_memory_mapping: [mem %#010lx-%#010lx]\n",
           start, end - 1);
    //分割内存范围到一个 struct map_range mr[] 数组中，实际有 nr_range 个元素
    memset(mr, 0, sizeof(mr));
    nr_range = split_mem_range(mr, 0, start, end);
    //分别为这些范围建立直接映射
    for (i = 0; i < nr_range; i++)
        ret = kernel_physical_mapping_init(mr[i].start, mr[i].end,
                           mr[i].page_size_mask,
                           prot);

    add_pfn_range_mapped(start >> PAGE_SHIFT, ret >> PAGE_SHIFT);

    return ret >> PAGE_SHIFT;
}
```

### 内核初始化对物理地址的直接映射
* `__kernel_physical_mapping_init()` 为给定的物理地址范围 `[paddr_start, paddr_end)` 建立到虚拟地址 `[__va(paddr_start), __va(paddr_end))` 的映射
* 假设采用的是 *5 级分页* 来分析，这样 `p4d` 一级不会被忽略
```cpp
static unsigned long __meminit
__kernel_physical_mapping_init(unsigned long paddr_start,
                   unsigned long paddr_end,
                   unsigned long page_size_mask,
                   pgprot_t prot, bool init)
{
    bool pgd_changed = false;
    unsigned long vaddr, vaddr_start, vaddr_end, vaddr_next, paddr_last;

    paddr_last = paddr_end;
    vaddr = (unsigned long)__va(paddr_start);  //由起始物理地址得到计算虚拟地址
    vaddr_end = (unsigned long)__va(paddr_end);//由结束物理地址得到结束虚拟地址
    vaddr_start = vaddr;                       //计算虚拟地址赋值给起始虚拟地址
    //从起始 VA 遍历到结束 VA，步长是 256 T
    for (; vaddr < vaddr_end; vaddr = vaddr_next) {
        pgd_t *pgd = pgd_offset_k(vaddr); //根据 VA 的 56~48 位的值得到它对应的 PGD 条目的指针
        p4d_t *p4d;
        //VA 向下对齐到 PGD 一级的起始地址，再加上 1 个 PGD 条目能管理的地址范围作为下一次迭代的 VA
        vaddr_next = (vaddr & PGDIR_MASK) + PGDIR_SIZE; //5 级分页一个 PGD 条目可覆盖 256 TB
        //如果该 PGD 条目已经有映射了
        if (pgd_val(*pgd)) {
            p4d = (p4d_t *)pgd_page_vaddr(*pgd); //该 PGD 条目的内容即一个 p4d 页表页的物理地址，再转成虚拟地址
            paddr_last = phys_p4d_init(p4d, __pa(vaddr), //创建 p4d 及 p4d 以下级别的页表，返回下一个要映射的物理地址
                           __pa(vaddr_end), //传入结束 PA，返回的最后处理的 PA 可能会小于结束 PA
                           page_size_mask,
                           prot, init);
            continue;
        }
        //如果该 PGD 条目未映射，则分配一个页面作为 p4d 页表页
        p4d = alloc_low_page();
        //创建 p4d 及 p4d 以下级别的页表，返回最后处理的 PA。传入结束 PA，返回的最后处理的 PA 可能会小于结束 PA
        paddr_last = phys_p4d_init(p4d, __pa(vaddr), __pa(vaddr_end),
                       page_size_mask, prot, init);

        spin_lock(&init_mm.page_table_lock);
        if (pgtable_l5_enabled()) //如果启用了 5 级页表
            pgd_populate_init(&init_mm, pgd, p4d, init); //填充该 PGD 条目为 p4d 页表页的物理地址
        else
            p4d_populate_init(&init_mm, p4d_offset(pgd, vaddr),
                      (pud_t *) p4d, init);

        spin_unlock(&init_mm.page_table_lock);
        pgd_changed = true; //设置 PGD 被修改的变量
    }
    //如果 PGD 被修改过，将修改同步到全局的 PGD 链表上的 PGD
    if (pgd_changed)
        sync_global_pgds(vaddr_start, vaddr_end - 1);
    //返回下一个要映射的物理地址
    return paddr_last;
}
```

### `phys_p4d_init()`
* `phys_p4d_init()` 在 `p4d` 级别初始化对物理地址的直接映射，基本思路如下：
1. 对于已经存在映射关系的 `p4d` 条目，目前还不存在 `512 G` 大页映射，调用 `phys_pud_init()` 安排 `pud` 级别的映射
2. 对于还没有建立映射关系 `p4d` 条目，目前还不存在 `512 G` 大页映射，调用 `phys_pud_init()` 来建立 `pud` 一级的页面映射
* 根据调用者 `__kernel_physical_mapping_init()` 可知入参 `p4d_page` 是 `p4d` 一级的页表页的地址
```cpp
static unsigned long __meminit
phys_p4d_init(p4d_t *p4d_page, unsigned long paddr, unsigned long paddr_end,
          unsigned long page_size_mask, pgprot_t prot, bool init)
{
    unsigned long vaddr, vaddr_end, vaddr_next, paddr_next, paddr_last;

    paddr_last = paddr_end;
    vaddr = (unsigned long)__va(paddr); //VA 来自起始 PA
    vaddr_end = (unsigned long)__va(paddr_end);
    //如果未开启 5 级页表，p4d 一级别折叠，直接传递到 pud 级的处理函数
    if (!pgtable_l5_enabled())
        return phys_pud_init((pud_t *) p4d_page, paddr, paddr_end,
                     page_size_mask, prot, init);
    //从起始 VA 遍历到结束 VA，步长是 512 G
    for (; vaddr < vaddr_end; vaddr = vaddr_next) {
        p4d_t *p4d = p4d_page + p4d_index(vaddr); //p4d 一级的页表页地址加 VA 在此级的索引得到 VA 在 p4d 一级的页表条目的 VA
        pud_t *pud;
        //VA 向下对齐到 p4d 一级的起始地址，再加上 1 个 p4d 条目能管理的地址范围作为下一次迭代的 VA
        vaddr_next = (vaddr & P4D_MASK) + P4D_SIZE; //一个 p4d 条目可覆盖 512 GB
        paddr = __pa(vaddr);
        //paddr 跟随着 VA 每次步进 512 G，直到 vaddr 超过 vaddr_end；事实上 for 循环 vaddr < vaddr_end 的条件致使下面的条件不可能成立
        //如果真的能让 paddr 达到或超过 paddr_end 后，则需要判断 paddr 对应的物理页面是否与 e820 的映射范围有交集？
        //如果有，那么可以认为是空洞。处于空洞的物理地址，对应的 p4d 条目填 0
        //setup_arch-> ... ->kernel_physical_mapping_init 这条调用路径上, after_bootmem = 0
        if (paddr >= paddr_end) {
            paddr_next = __pa(vaddr_next);
            if (!after_bootmem &&
                !e820__mapped_any(paddr & P4D_MASK, paddr_next,
                         E820_TYPE_RAM) &&
                !e820__mapped_any(paddr & P4D_MASK, paddr_next,
                         E820_TYPE_RESERVED_KERN) &&
                !e820__mapped_any(paddr & P4D_MASK, paddr_next,
                         E820_TYPE_ACPI))
                set_p4d_init(p4d, __p4d(0), init);
            continue;
        }
        //1. 对于已经存在映射关系的 p4d 条目，目前还不存在 512 G 大页映射，调用 phys_pud_init() 安排 pud 级别的映射
        if (!p4d_none(*p4d)) { //如果该 p4d 条目已经有映射了
            pud = pud_offset(p4d, 0); //取出下一级 pud 页表页的地址
            //在 pud 页表页的范围内进行映射，最多映射 512 G 的空间，返回值是最后映射的物理地址
            paddr_last = phys_pud_init(pud, paddr, __pa(vaddr_end),
                    page_size_mask, prot, init);
            continue;
        }
        //2. 对于还未建立映射关系的 pu4 条目，目前还不存在 512 G 大页映射，调用 phys_pud_init() 来建立 pud 一级的页面映射
        pud = alloc_low_page(); //分配一个页面作为 pud 页表页
        paddr_last = phys_pud_init(pud, paddr, __pa(vaddr_end),
                       page_size_mask, prot, init); //建立 pud 一级的页面映射
        //填充该 p4d 条目为 pud 页表页的物理地址
        spin_lock(&init_mm.page_table_lock);
        p4d_populate_init(&init_mm, p4d, pud, init);
        spin_unlock(&init_mm.page_table_lock);
    }

    return paddr_last;
}
```
* 这里的逻辑和 `__kernel_physical_mapping_init()` 比较类似，其实最开始引入时和 `phys_pud_init()` 逻辑比较类似，但遇到了问题，于是由以下 commit 做出修正：
```c
commit 432c833218dd0f75e7b56bd5e8658b72073158d2
Author: Kirill A. Shutemov <kirill@shutemov.name>
Date:   Mon Jun 24 15:31:50 2019 +0300

    x86/mm: Handle physical-virtual alignment mismatch in phys_p4d_init()
```
- https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=432c833218dd0f75e7b56bd5e8658b72073158d2
* 修改前，比如说开启 KASLR 使得 `PAGE_OFFSET` 值为 `0xff4227ff40000000`，这里只需保证对齐到 `PUD_SIZE - 1GB` 即可
  * 由 commit b569c1843498 ("x86/mm/KASLR: Reduce randomization granularity for 5-level paging to 1GB") 引入的
* 迭代 1
  * 对于 `paddr = 0x03 3fe0 0000`，`vaddr = (unsigned long)__va(paddr) = 0xff4228027fe00000`
  * 该 `vaddr` 在 `p4d` 页表页中的索引为 `p4d_index(vaddr) = 0x50 = 80`
  * 对于 `paddr_next = (paddr & P4D_MASK) + P4D_SIZE = 0x80 0000 0000`
* 迭代 2
  * 下一次迭代 `paddr = paddr_next = 0x80 0000 0000`
  * `vaddr = (unsigned long)__va(paddr) = 0xff42287f40000000`
  * 该 `vaddr` 在 `p4d` 页表页中的索引 **仍为**  `p4d_index(vaddr) = 0x50 = 80`
  * 对于 `paddr_next = (paddr & P4D_MASK) + P4D_SIZE = 0x90 0000 0000`
* 该代码假定条目严格对齐，并无条件地将索引递增到 `P4D` 表中，这会创建错误的重复条目。一旦索引到达末尾，页表中的最后一个条目就没处理。
  * 虽然也迭代了 `512 - 80 = 432` 次，但第二次迭代其实重复迭代了索引 `80`，页表最后一个条目并没有迭代到。
  * 改成用虚拟地址迭代就没这个问题，比如 `paddr = 0x03 3fe0 0000` 对应的 `vaddr = 0xff422802_7fe0_0000` 第一次迭代时就算出 `vaddr_next = (vaddr & P4D_MASK) + P4D_SIZE = 0xff422880_0000_0000`，并且算出的索引就是 `80` 而不会重复迭代这个索引
* 该 commit 认为，抛开 `paddr >= paddr_end` 条件可能会计算错误从而导致一条 `P4D` 条目被错误地清除。
  * 事实上 `for` 循环 `vaddr < vaddr_end` 的条件致使该条件不可能成立

### `phys_pud_init()`
![phys_pud_init](https://pic3.zhimg.com/80/v2-348a3393b795c42c665f00c3678483ea_720w.webp)
* `phys_pud_init()` 为物理地址创建 `pud` 级页表映射。
  * 虚拟地址和物理地址不必在此级别对齐。KASLR 可以将虚拟地址随机化到这个级别。
  * 它返回映射的最后一个物理地址。
* `phys_pud_init()` 函数尝试将 `[paddr, paddr_end)` 区间的物理页面映射到 `pud_page` 指向的 `pud` 页表页上。
  * 1 个 `pud_page` 上有 512 个 `pud_entry`，1 个 `pud_entry` 能映射 1G 的空间，所以一个 `pud_page` 最多能映射 512G 的地址空间；
  * `[paddr, paddr_end)` 地址区间如果超过了 512G，那么只会映射该区间的部分地址。
  * `paddr_last` 作为返回值交给调用者，表示自己已经映射了一部分地址，下一个要映射的地址是 `paddr_last`，请调用者继续完成剩余区间的映射。
* 基本思路如下：
1. 对于已经存在映射关系的 `pud` 条目：
   1.1 如果之前不是 1G 巨页映射，调用 `phys_pmd_init()` 安排 `pmd` 级别（2M）的映射
   1.2 如果之前是 1G 映射，`page_size_mask` 指示本次仍然可以按照 1G 映射，那么沿用之前的映射
   1.3 如果之前是 1G 映射，`page_size_mask` 指示本次 **不可以** 按照 1G 映射，调用 `phys_pmd_init()` 来建立 `pmd` 一级的页面映射
2. 对于还没有建立映射关系 `pud` 条目：
   2.1 `page_size_mask` 指示本次可以按照 1G 映射，那么进行 1G 大小的页面映射
   2.2 `page_size_mask` 指示本次 **不可以** 按照 1G 映射，调用 `phys_pmd_init()` 来建立 `pmd` 一级的页面映射
```cpp
/*
 * Create PUD level page table mapping for physical addresses. The virtual
 * and physical address do not have to be aligned at this level. KASLR can
 * randomize virtual addresses up to this level.
 * It returns the last physical address mapped.
 */
static unsigned long __meminit
phys_pud_init(pud_t *pud_page, unsigned long paddr, unsigned long paddr_end,
          unsigned long page_size_mask, pgprot_t _prot, bool init)
{
    unsigned long pages = 0, paddr_next;
    unsigned long paddr_last = paddr_end;
    unsigned long vaddr = (unsigned long)__va(paddr);
    int i = pud_index(vaddr); //根据 VA 得到在 pud 一级页表页中的索引
    //从起始索引遍历到第 512 个索引，PA 递进的步长是 1G
    for (; i < PTRS_PER_PUD; i++, paddr = paddr_next) {
        pud_t *pud;
        pmd_t *pmd;
        pgprot_t prot = _prot;

        vaddr = (unsigned long)__va(paddr); //刷新递进后的 VA
        pud = pud_page + pud_index(vaddr);  //pud 一级的页表页地址加 VA 在此级的索引得到 VA 在 pud 一级的页表条目的 VA
        paddr_next = (paddr & PUD_MASK) + PUD_SIZE; //先把下一次迭代将要递进的 PA 算出来，步长是 1G
        //paddr 每次增长 1G，直到增长到 512G 地址边界；当 paddr 超过函数传递的 paddr_end 参数后，需要
        //判断 paddr 对应的物理页面是否在 e820 的映射范围，如果不是 e820 的映射，那么可以认为是空洞。
        //处于空洞的物理地址，对应的 pud 条目填写 0
        //setup_arch->... ->kernel_physical_mapping_init 这条调用路径上，after_bootmem = 0
        if (paddr >= paddr_end) {
            if (!after_bootmem &&
                !e820__mapped_any(paddr & PUD_MASK, paddr_next,
                         E820_TYPE_RAM) &&
                !e820__mapped_any(paddr & PUD_MASK, paddr_next,
                         E820_TYPE_RESERVED_KERN) &&
                !e820__mapped_any(paddr & PUD_MASK, paddr_next,
                         E820_TYPE_ACPI))
                set_pud_init(pud, __pud(0), init);
            continue;
        }
        //1. 对于已经存在映射关系的 pud 条目：
        if (!pud_none(*pud)) {
            if (!pud_leaf(*pud)) { //1.1 如果之前不是 1G 巨页映射
                pmd = pmd_offset(pud, 0); //取出下一级 pmd 页表页的地址
                //在 pmd 页表页的范围内进行映射，最多映射 1G 的空间，返回值是最后映射的物理地址
                paddr_last = phys_pmd_init(pmd, paddr,
                               paddr_end,
                               page_size_mask,
                               prot, init);
                continue;
            }
            /* 如果走到这里, 说明 pud 条目非空，并且之前已经建立过巨页映射
             * If we are ok with PG_LEVEL_1G mapping, then we will
             * use the existing mapping.
             *
             * Otherwise, we will split the gbpage mapping but use
             * the same existing protection  bits except for large
             * page, so that we don't violate Intel's TLB
             * Application note (317080) which says, while changing
             * the page sizes, new and old translations should
             * not differ with respect to page frame and
             * attributes.
             */ //1.2 如果之前是 1G 映射，page_size_mask 指示本次仍然可以按照 1G 映射，那么沿用之前的映射
            if (page_size_mask & (1 << PG_LEVEL_1G)) {
                if (!after_bootmem)
                    pages++;
                paddr_last = paddr_next;
                continue;
            }
            //1.3 如果之前是 1G 映射，page_size_mask 指示本次不可以按照 1G 映射，后面需要调用 phys_pmd_init() 来建立 pmd 一级的页面映射
            prot = pte_pgprot(pte_clrhuge(*(pte_t *)pud)); //清除页表项中的 PSE 标记
        }
        //2. 对于还没有建立映射关系 pud 条目；或者如果之前是 1G 映射，page_size_mask 指示本次不可以按照 1G 映射，这里同样进不来
        if (page_size_mask & (1<<PG_LEVEL_1G)) { //2.1 page_size_mask 指示本次可以按照 1G 映射，那么进行 1G 大小的页面映射
            pages++;
            spin_lock(&init_mm.page_table_lock);
            set_pud_init(pud,
                     pfn_pud(paddr >> PAGE_SHIFT, prot_sethuge(prot)),
                     init); //填充 pud 条目为被映射的 1G 物理地址
            spin_unlock(&init_mm.page_table_lock);
            paddr_last = paddr_next;
            continue;
        }
        //2.2 page_size_mask 指示本次不可以按照 1G 映射，调用 phys_pmd_init() 来建立 pmd 一级的页面映射
        pmd = alloc_low_page(); //分配一个页面作为 pmd 页表页
        paddr_last = phys_pmd_init(pmd, paddr, paddr_end,
                       page_size_mask, prot, init);
        //填充该 pud 条目为 pmd 页表页的物理地址
        spin_lock(&init_mm.page_table_lock);
        pud_populate_init(&init_mm, pud, pmd, init);
        spin_unlock(&init_mm.page_table_lock);
    }
    //更新 DirectMap 中 1G 映射的统计
    update_page_count(PG_LEVEL_1G, pages);

    return paddr_last;
}
```

### `phys_pmd_init()`
![phys_pmd_init](https://pic4.zhimg.com/80/v2-0efe1f076cb1021d148aba1eb45e3d57_720w.webp)
* `phys_pmd_init()` 为物理地址创建 `pmd` 级页表映射。
  * 虚拟地址和物理地址必须在此级别对齐。
  * 返回映射的最后一个物理地址。
* `phys_pmd_init()` 函数尝试映射 `[paddr, paddr_end)` 区间的物理页面到 `pmd_page` 指向的 `pmd` 页表页上。
  * `pmd_page` 这个页表最多只能映射 1G 的地址空间；
  * `[paddr, paddr_end)` 地址区间如果超过了 1G，那么只会映射部分该区间的部分地址。
  * `paddr_last` 作为返回值交给调用者，表示自己已经映射了一部分地址，下一个要映射的地址是 `paddr_last`，请调用者继续完成剩余区间的映射。
* 基本思路如下：
1. 对于已经存在映射关系的 `pmd` 条目：
   1.1 如果之前不是 2M 大页映射，调用 `phys_pte_init()` 安排 `pte` 级别（4K）的映射
   1.2 如果之前是 2M 映射，`page_size_mask` 指示本次仍然可以按照 2M 映射，那么沿用之前的映射
   1.3 如果之前是 2M 映射，`page_size_mask` 指示本次 **不可以** 按照 2M 映射，调用 `phys_pte_init()` 来建立 `pte` 一级的页面映射
2. 对于还没有建立映射关系 `pmd` 条目：
   2.1 `page_size_mask` 指示本次可以按照 2M 映射，那么进行 2M 大小的页面映射
   2.2 `page_size_mask` 指示本次 **不可以** 按照 2M 映射，调用 `phys_pte_init()` 来建立 `pte` 一级的页面映射
```cpp
/*
 * Create PMD level page table mapping for physical addresses. The virtual
 * and physical address have to be aligned at this level.
 * It returns the last physical address mapped.
 */
static unsigned long __meminit
phys_pmd_init(pmd_t *pmd_page, unsigned long paddr, unsigned long paddr_end,
          unsigned long page_size_mask, pgprot_t prot, bool init)
{
    unsigned long pages = 0, paddr_next;
    unsigned long paddr_last = paddr_end;
    //因为虚拟地址和物理地址在此级别对齐，所以可以根据 PA 得到在 pmd 一级页表页中的索引
    int i = pmd_index(paddr);
    //从起始索引遍历到第 512 个索引，PA 递进的步长是 2M
    for (; i < PTRS_PER_PMD; i++, paddr = paddr_next) {
        pmd_t *pmd = pmd_page + pmd_index(paddr); //pmd 一级的页表页地址（VA）加 paddr 在此级的索引得到 paddr 在 pmd 一级的页表条目的 VA
        pte_t *pte;
        pgprot_t new_prot = prot;
        //先把下一次迭代将要递进的 PA 算出来，步长是 2M
        paddr_next = (paddr & PMD_MASK) + PMD_SIZE;
        //paddr 每次增长 2M，直到增长到 1G 地址边界；当 paddr 超过函数传递的 paddr_end 参数后，需要
        //判断 paddr 对应的物理页面是否在 e820 的映射范围，如果不是 e820 的映射，那么可以认为是空洞。
        //处于空洞的物理地址，对应的 pmd 条目填写 0
        //setup_arch->... ->kernel_physical_mapping_init 这条调用路径上，after_bootmem = 0
        if (paddr >= paddr_end) {
            if (!after_bootmem &&
                !e820__mapped_any(paddr & PMD_MASK, paddr_next,
                         E820_TYPE_RAM) &&
                !e820__mapped_any(paddr & PMD_MASK, paddr_next,
                         E820_TYPE_RESERVED_KERN) &&
                !e820__mapped_any(paddr & PMD_MASK, paddr_next,
                         E820_TYPE_ACPI))
                set_pmd_init(pmd, __pmd(0), init);
            continue;
        }
        //1. 对于已经存在映射关系的 pmd 条目：
        if (!pmd_none(*pmd)) {
            if (!pmd_leaf(*pmd)) { //1.1 如果之前不是 2M 大页映射
                spin_lock(&init_mm.page_table_lock);
                pte = (pte_t *)pmd_page_vaddr(*pmd); //取出下一级 pte 页表页的地址
                paddr_last = phys_pte_init(pte, paddr,
                               paddr_end, prot,
                               init); //在 pte 页表页的范围内进行映射，最多映射 2M 的空间，返回值是最后映射的物理地址
                spin_unlock(&init_mm.page_table_lock);
                continue;
            }
            /* 如果走到这里, 说明 pmd 条目非空，并且之前已经建立过大页映射
             * If we are ok with PG_LEVEL_2M mapping, then we will
             * use the existing mapping,
             *
             * Otherwise, we will split the large page mapping but
             * use the same existing protection bits except for
             * large page, so that we don't violate Intel's TLB
             * Application note (317080) which says, while changing
             * the page sizes, new and old translations should
             * not differ with respect to page frame and
             * attributes.
             */ //1.2 如果之前是 2M 映射，page_size_mask 指示本次仍然可以按照 2M 映射，那么沿用之前的映射
            if (page_size_mask & (1 << PG_LEVEL_2M)) {
                if (!after_bootmem)
                    pages++;
                paddr_last = paddr_next;
                continue;
            }
            //1.3 如果之前是 2M 映射，page_size_mask 指示本次不可以按照 2M 映射，后面需要调用 phys_pte_init() 来建立 pte 一级的页面映射
            new_prot = pte_pgprot(pte_clrhuge(*(pte_t *)pmd));
        }
        //2. 对于还没有建立映射关系 pmd 条目；或者如果之前是 2M 映射，page_size_mask 指示本次不可以按照 2M 映射，这里同样进不来
        if (page_size_mask & (1<<PG_LEVEL_2M)) { //2.1 page_size_mask 指示本次可以按照 2M 映射，那么进行 2M 大小的页面映射
            pages++;
            spin_lock(&init_mm.page_table_lock);
            set_pmd_init(pmd,
                     pfn_pmd(paddr >> PAGE_SHIFT, prot_sethuge(prot)),
                     init); //填充 pmd 条目为被映射的 2M 物理地址
            spin_unlock(&init_mm.page_table_lock);
            paddr_last = paddr_next;
            continue;
        }
        //2.2 page_size_mask 指示本次不可以按照 2M 映射，调用 phys_pte_init() 来建立 pte 一级的页面映射
        pte = alloc_low_page(); //分配一个页面作为 pte 页表页
        paddr_last = phys_pte_init(pte, paddr, paddr_end, new_prot, init);
        //填充该 pmd 条目为 pte 页表页的物理地址
        spin_lock(&init_mm.page_table_lock);
        pmd_populate_kernel_init(&init_mm, pmd, pte, init);
        spin_unlock(&init_mm.page_table_lock);
    }
    update_page_count(PG_LEVEL_2M, pages); //更新 DirectMap 中 2M 映射的统计
    return paddr_last;
}
```

### `phys_pte_init()`
![phys_pte_init](https://pic2.zhimg.com/80/v2-1cbe7f63d866368a59bbcbbd6becd085_720w.webp)
* `phys_pte_init()` 为物理地址创建 `pte` 级页表映射。返回映射的最后一个物理地址。
* `phys_pte_init()` 从 `i` 到 `511`，逐个填充 `pte` 的页表项。
  * 有些物理地址没有在 e820 的映射里面，这部分的物理地址称作空洞，需要在页表项里面填写 `0`。
* 这是页表映射的最后一级了，所以没有之前的向下递进的处理。并且，对于之前已有的映射直接重用。
```cpp
/*
 * Create PTE level page table mapping for physical addresses.
 * It returns the last physical address mapped.
 */
static unsigned long __meminit
phys_pte_init(pte_t *pte_page, unsigned long paddr, unsigned long paddr_end,
          pgprot_t prot, bool init)
{
    unsigned long pages = 0, paddr_next;
    unsigned long paddr_last = paddr_end;
    pte_t *pte;
    int i;
    //因为虚拟地址和物理地址已在 pmd 级别对齐，所以根据 PA 可得到在 pte 一级页表页中的索引
    pte = pte_page + pte_index(paddr);
    i = pte_index(paddr);
    //从起始索引遍历到第 512 个索引，PA 递进的步长是 4K
    for (; i < PTRS_PER_PTE; i++, paddr = paddr_next, pte++) {
        paddr_next = (paddr & PAGE_MASK) + PAGE_SIZE; //先把下一次迭代将要递进的 PA 算出来，步长是 4K
        //因为 paddr 会一直遍历到 2M 边界，所以当超过函数传递的 paddr_end 参数后，需要判断
        //paddr 对应的物理页面是否在 e820 的映射范围，如果不是 e820 的映射，那么可以认为是空洞。
        //处于空洞的物理地址，对应的 pte 条目填写 0
        //setup_arch->... ->kernel_physical_mapping_init 这条调用路径上，after_bootmem = 0
        if (paddr >= paddr_end) {
            if (!after_bootmem &&
                !e820__mapped_any(paddr & PAGE_MASK, paddr_next,
                         E820_TYPE_RAM) &&
                !e820__mapped_any(paddr & PAGE_MASK, paddr_next,
                         E820_TYPE_RESERVED_KERN) &&
                !e820__mapped_any(paddr & PAGE_MASK, paddr_next,
                         E820_TYPE_ACPI))
                set_pte_init(pte, __pte(0), init);
            continue;
        }
        //对于已经存在映射关系的 pte 条目，不做修改，只统计 4K 映射的页面数
        /*
         * We will re-use the existing mapping.
         * Xen for example has some special requirements, like mapping
         * pagetable pages as RO. So assume someone who pre-setup
         * these mappings are more intelligent.
         */
        if (!pte_none(*pte)) {
            if (!after_bootmem)
                pages++;
            continue;
        }
        //对于不存在映射关系的 pte 条目，填写 pte 条目为 PA，建立映射关系。这是页表映射的最后一级了
        if (0)
            pr_info("   pte=%p addr=%lx pte=%016lx\n", pte, paddr,
                pfn_pte(paddr >> PAGE_SHIFT, PAGE_KERNEL).pte);
        pages++;
        set_pte_init(pte, pfn_pte(paddr >> PAGE_SHIFT, prot), init);
        paddr_last = (paddr & PAGE_MASK) + PAGE_SIZE; //记录最后一个设置的 pte，函数最后会返回该值
    }
    //更新 DirectMap 中 4K 映射的统计
    update_page_count(PG_LEVEL_4K, pages);

    return paddr_last;
}
```

## References
- 原文地址：[kernel_physical_mapping_init - 知乎](https://zhuanlan.zhihu.com/p/592840968)