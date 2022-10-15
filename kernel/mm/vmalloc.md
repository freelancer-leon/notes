# vmalloc

## x86-64
* x86-64 不再是通过缺页时将主内核页表同步到缺页进程的内核页表了，而是在 `vmalloc()` 分配时就同步好了<sup>[1][2]</sup>
* 但随后又被移除了，被新的方式取代<sup>[3]</sup>，但该版本有 bug，被以下 commit fix，最后这种新的方式得以回归<sup>[4]</sup>
  ```cpp
  commit 995909a4e22bc7b3ea3a71388cbb62ffebd76e7b
  Author: Joerg Roedel <jroedel@suse.de>
  Date:   Fri Aug 7 10:40:13 2020 +0200

      x86/mm/64: Do not dereference non-present PGD entries
  ```
### 预分配的方式
* 内核启动时，不管有没有用到 `vmalloc()`，都先将 `VMALLOC_START`~`VMALLOC_END` 范围在第二级的页表分配好，主要工作由 `preallocate_vmalloc_pages()` 来完成
  * 因为同步时需要迭代`pgd_list`，这个过程需要拿`pgd_lock`，为了避免拿锁，预先在启动时就将会用到的第二级的页表项分配好
  * 比如 5 级页表 `vmalloc()` 有 `12.5PB` 的范围，那就预先分配好 `12.5PB / 256TB = 50` 个 `p4d` 级的页表项（顶级页表此时已经有了，无需分配）
  * 比如 4 级页表 `vmalloc()` 有 `32TB` 的范围，那就预先分配好 `32TB / 512GB = 64` 个 `pud` 级的页表项
    ```cpp
    start_kernel()
    -> mm_init()
       -> mem_init()
          -> preallocate_vmalloc_pages()
       -> vmalloc_init()
    ```
* 注意，内核页表在所有进程之间是共享的，这句话怎么理解
  * 在创建进程过程中会有 `clone_pgd_range(pgd + KERNEL_PGD_BOUNDARY, swapper_pg_dir + KERNEL_PGD_BOUNDARY, KERNEL_PGD_PTRS)`，这行语句的意思是将新进程的顶级页目录中映射内核地址空间部分的条目从主内核页表拷贝一份
    * 每个进程自然是有自己的顶级页表，而顶级页表映射内核地址空间的那一半内容是相同的，即从第二级页表开始共享页表；
    * 另一半映射则拷贝自各自的父进程，需要时建立 COW
* 因为 `vmalloc()` 会用到的第二级页表已经事先分配好了，而内核页表从第二级开始就是共享的，因此 `vmalloc()` 用到的下面各级页表项就不再需要专门同步了
* 之前是 ondemand 的，由于启动时主内核页表没有预映射`vmalloc()`区域，所以有缺页时 `vmalloc_fault() -> vmalloc_sync_one()` 来同步页表
* 采用新的方式后，x86-64 中发生内核缺页的原因又少了一种 :-)


```cpp
vmalloc(unsigned long size)
-> __vmalloc_node(size, 1, GFP_KERNEL, NUMA_NO_NODE, __builtin_return_address(0))
   -> __vmalloc_node_range()
   -> area = __get_vm_area_node()
      -> area = kzalloc_node()
      -> va = alloc_vmap_area(size, align, start, end, node, gfp_mask)
      -> setup_vmalloc_vm(area, va, flags, caller)
   -> __vmalloc_area_node(area, gfp_mask, prot, shift, node)
         /* Please note that the recursion is strictly bounded. */
         if (array_size > PAGE_SIZE) {
            area->pages = __vmalloc_node(array_size, 1, nested_gfp, node,
                          area->caller);
         } else {
            area->pages = kmalloc_node(array_size, nested_gfp, node);
         }
         //alloc_pages 并填写上面分配好空间的 area->pages 数组
      -> area->nr_pages = vm_area_alloc_pages(gfp_mask | __GFP_NOWARN, node, page_order, nr_small_pages, area->pages)
      -> vmap_pages_range(addr, addr + size, prot, area->pages, page_shift)
         -> vmap_pages_range_noflush()
               for (i = 0; i < nr; i += 1U << (page_shift - PAGE_SHIFT)) {
                   err = vmap_range_noflush(addr, addr + (1UL << page_shift),
                             __pa(page_address(pages[i])), prot, page_shift);
                   if (err)
                      return err;
                   addr += 1UL << page_shift;
               }
```

## x86-32
* x86-32 经过短暂改变又回到了旧的方式，即需要从主内核页表同步
* `vmalloc()` 分配好并填充好当前进程页表后，会根据之前遍历的过程知道是否在某一级页表发生了变化，如果发生了变化就需要同步到其他进程的页表
* `pgd_list` 是一个全局链表，上面挂着所有进程的`pgd`
* 以下是 fork 时挂 `pgd_list` 链表的路径
```cpp
copy_process()
-> copy_mm()
   -> dup_mm()
      -> mm = allocate_mm()
      -> mm_init()
         -> mm_alloc_pgd(mm)
            -> mm->pgd = pgd_alloc(mm)
               -> pgd = _pgd_alloc()
               -> mm->pgd = pgd;
               -> pgd_ctor(mm, pgd)
                     -> clone_pgd_range(pgd + KERNEL_PGD_BOUNDARY,
                        swapper_pg_dir + KERNEL_PGD_BOUNDARY, KERNEL_PGD_PTRS)
                     if (!SHARED_KERNEL_PMD) {
                     -> pgd_set_mm(pgd, mm)
                     -> pgd_list_add(pgd)
                           struct page *page = virt_to_page(pgd);
                        -> list_add(&page->lru, &pgd_list)
                     }
```

## References

- [1] [[RFC PATCH 0/7] mm: Get rid of vmalloc_sync_(un)mappings()](https://lore.kernel.org/lkml/20200508144043.13893-1-joro@8bytes.org/)
- [2] [[PATCH v3 0/7] mm: Get rid of vmalloc_sync_(un)mappings()](https://lore.kernel.org/all/20200515140023.25469-8-joro@8bytes.org/T/#md6fc597dd4d3e025eb5ffe4688cd4da74898bcb1)
- [3] [[PATCH v3 0/3] x86/mm/64: Remove vmalloc/ioremap pgtable synchronization](https://lore.kernel.org/all/20200721095953.6218-1-joro@8bytes.org/)
- [4] [[PATCH 0/2] x86: Retry to remove vmalloc/ioremap synchronzation](https://lore.kernel.org/all/20200814151947.26229-1-joro@8bytes.org/)
