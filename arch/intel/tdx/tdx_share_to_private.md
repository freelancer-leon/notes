# TDX 共享和私有内存转换
* TD 可以通过 `tdvmcall(TDG.VP.VMCALL<MapGPA>)` 将一段内存在共享和私有之间进行转换
* 对于目前 kernel + Qemu 的实现来说，`MapGPA` 不是必须的
  * 当前 KVM 的设计是用一个 xarray 来追踪每个 GPA 的内存属性是 private 还是 shared（`NULL`）
  * 即使没有主动 tdcall `MapGPA`，也会因为 guest 访问时被动地发生缺页 TD-exit 到 KVM，由于 `fault->is_private` 与内存属性不一致而 upcall 到 Qemu
  * 对于 Qemu 来说，这里的处理和主动 `MapGPA` 的函数是一样的，都是 `kvm_convert_memory()`，即
    * 如果这个 fault 的 GPA 属于一个有 gmem backend 的 slot，那么就转换其内存属性，并 punch hole（意味着会 zap 在原有 EPT 中的映射）
    * 如果这个 fault 的 GPA 没有 gmem 作为 benkend，例如 PCI 设备的 BAR 空间的对应的 slot，那么 Qemu 就退出
      * 比如 guest 访问某个 BAR 之前未经过 `ioremap()` 去设置页表中的 shared bit，导致一次 `fault->is_private = true` 的 EPT violation
  * 因此，更关键的是 TD guest 必须正确地设置页表中的 shared bit，不要以私有的方式访问共享页面
  * 另外 shared -> private 转换时，`tdx_accept_memory()` 是必须的，重新 augment 进来的私有内存必须被 accepted 才能使用
  * 但是主动提前将一段 GPA 通过 tdcall `MapGPA` 进行转换，肯定要好过被动地在这段 GPA 中一次次地走上述过程节省开销

## TD Guest 发起转换
* TDX 不允许 VMM 直接访问 guest 私有内存，必须显式共享与 VMM 通信所需的任何内存
* 相同的规则适用于任何进出 TDX 客户机的 DMA，所有 DMA 页面都必须标记为 *共享页面*
* 在不对设备驱动程序进行任何更改的情况下实现此目的的通用方法是使用 SWIOTLB 框架
* 这只是其中一个发起 *共享 <=> 私有* 转换的路径
```c
start_kernel()
-> mm_init()
-> mem_encrypt_init()
      if (!cc_platform_has(CC_ATTR_MEM_ENCRYPT)) //TD VM 是有这个属性的，见 intel_cc_platform_has()
         return; //所以不会返回，而是往下走，转换 SWIOTLB buffer 为共享内存
   -> swiotlb_update_mem_attributes()
         struct io_tlb_mem *mem = &io_tlb_default_mem;
         vaddr = phys_to_virt(mem->start);
         bytes = PAGE_ALIGN(mem->nslabs << IO_TLB_SHIFT);
      -> set_memory_decrypted((unsigned long)vaddr, bytes >> PAGE_SHIFT);
         -> __set_memory_enc_dec(addr, numpages, false) //第三个参数 false 表示解密
            -> __change_page_attr_set_clr(&cpa, 1) //修改页表条目，设置 shared bit，这样 GVA 翻译过来的 GPA 就是 shared
               -> __change_page_attr(cpa, primary) //CPA 是 change page attribute 的缩写
            -> x86_platform.guest.enc_status_change_finish(addr, numpages, enc)
            => tdx_enc_status_changed()
               -> _tdx_hypercall(TDVMCALL_MAP_GPA, start, end - start, 0, 0)
         mem->vaddr = swiotlb_mem_remap(mem, bytes);
```
* kernel/dma/swiotlb.c 的 `setup_io_tlb_npages()` 解析到 `swiotlb=fore` 的 kernel command line，会将 `swiotlb_force_bounce = true`
* 如果 kernel command line 不设置，TDX guest 也会因为 `cc_platform_has(CC_ATTR_GUEST_MEM_ENCRYPT)` 在以下路径强制使能 SWIOTLB `x86_swiotlb_flags |= SWIOTLB_FORCE`，并传递到 `io_tlb_mem` 的 `force_bounce` 域
```c
start_kernel()
-> mm_init()
   -> mem_init()
      -> pci_iommu_alloc()
         -> pci_swiotlb_detect()
               if (cc_platform_has(CC_ATTR_GUEST_MEM_ENCRYPT)) {
                  x86_swiotlb_enable = true;
                  x86_swiotlb_flags |= SWIOTLB_FORCE;
               }
         -> swiotlb_init(x86_swiotlb_enable, x86_swiotlb_flags)
            -> swiotlb_init_remap(addressing_limit, flags, NULL)
                  struct io_tlb_mem *mem = &io_tlb_default_mem;
               -> swiotlb_init_io_tlb_mem()
                     mem->force_bounce = swiotlb_force_bounce || (flags & SWIOTLB_FORCE);
```
* 这样在类似 `dma_map_single()` 的时候就会根据 `force_bounce` 域要求该 DMA 采用 SWIOTLB 作为 bounce

### TD Guest 的 DMA 内存共享
* 当使能 `CONFIG_INTEL_TDX_GUEST` 的时候会使能 `CONFIG_X86_MEM_ENCRYPT`
```c
config INTEL_TDX_GUEST
    bool "Intel TDX (Trust Domain Extensions) - Guest Support"
    depends on X86_64 && CPU_SUP_INTEL
    depends on X86_X2APIC
    select ARCH_HAS_CC_PLATFORM
    select X86_MEM_ENCRYPT
    select X86_MCE
```
* 这会导致 arch/x86/mm/mem_encrypt.c 被编译，从而让 `mem_encrypt_init()` 覆盖 init/main.c 中定义的 `void __init __weak mem_encrypt_init(void) { }`
  * arch/x86/mm/Makefile
```makefile
obj-$(CONFIG_X86_MEM_ENCRYPT)   += mem_encrypt.o
```
* 这样 TD guest 的 swiotlb 会被强制使能
* 对于 TD host，由于 `tdx_early_init()` 的时候通过 `cpuid 0x21` 返回值不是 `"IntelTDX    "` 得知自己不是 TD guest，不会设置 `vendor` 为 `cc_set_vendor(CC_VENDOR_INTEL)`。因此，`cc_platform_has()` 总是返回 `false`

## VMM 处理 `tdcall(TDG.VP.VMCALL<MapGPA>)`
* TD 调用 `vmcall (function)/tdcall (instruction)` 会导致 VM-exit 从 SEAM non-root operation 退出到 KVM 进行处理
```cpp
vcpu_enter_guest()
-> static_call(kvm_x86_handle_exit)(vcpu, exit_fastpath)
=> vt_handle_exit()
      if (is_td_vcpu(vcpu))
         return tdx_handle_exit(vcpu, fastpath);
            switch (exit_reason.basic)
            case EXIT_REASON_TDCALL:
               return handle_tdvmcall(vcpu);
                  switch (tdvmcall_leaf(vcpu))
                  case TDG_VP_VMCALL_MAP_GPA:
                     return tdx_map_gpa(vcpu);
                        if (kvm_slot_can_be_private(slot))
                           return tdx_vp_vmcall_to_user(vcpu);
```
* KVM 在 `tdx_map_gpa()` 中会检查转换范围的合法性，如果没问题就转交给 Qemu 处理

## Qemu 处理 `tdcall(TDG.VP.VMCALL<MapGPA>)`
* 当 KVM 将请求转到 Qemu 后，进行如下处理
```cpp
kvm_cpu_exec()
-> kvm_vcpu_ioctl(cpu, KVM_RUN, 0)
     switch (run->exit_reason)
     default:
     -> kvm_arch_handle_exit()
           case KVM_EXIT_TDX:
        -> tdx_handle_exit()
              switch (tdx_exit->type)
              case KVM_EXIT_TDX_VMCALL:
              -> tdx_handle_vmcall(cpu, &tdx_exit->u.vmcall)
                 switch (vmcall->subfunction)
                 case TDG_VP_VMCALL_MAP_GPA:
                 -> tdx_handle_map_gpa(cpu, vmcall)
                    -> kvm_convert_memory(gpa, size, private)
                          if (memory_region_can_be_private(section.mr))
                          -> kvm_encrypt_reg_region(start, size, shared_to_private)
                                attr.attributes = reg_region ? KVM_MEMORY_ATTRIBUTE_PRIVATE : 0;
                             -> kvm_vm_ioctl(kvm_state, KVM_SET_MEMORY_ATTRIBUTES, &attr)
                             -> ram_block_convert_range(rb, offset, size, shared_to_private)
                                -> ram_block_discard_range_fd(rb, start, length, fd)
                                   -> fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE, start, length)
```

## VMM 的处理 `ioctl(fd, KVM_SET_MEMORY_ATTRIBUTES, &attr)`
* `ioctl(fd, KVM_SET_MEMORY_ATTRIBUTES, &attr)` 将一个范围的内存属性 **从共享转为私有** 的处理路径：
```c
kvm_vm_ioctl()
   case KVM_SET_MEMORY_ATTRIBUTES:
   -> copy_from_user(&attrs, argp, sizeof(attrs))
   -> kvm_vm_ioctl_set_mem_attributes(kvm, &attrs)
   -> copy_to_user(argp, &attrs, sizeof(attrs))
```
* `kvm_vm_ioctl_set_mem_attributes()` 设置指定内存范围的的属性
```cpp
static int kvm_vm_ioctl_set_mem_attributes(struct kvm *kvm,
                       struct kvm_memory_attributes *attrs)
{
    gfn_t start, end;
    unsigned long i;
    void *entry;
    int idx;
    u64 supported_attrs = kvm_supported_mem_attributes(kvm); //对于 TD VM 仅返回 KVM_MEMORY_ATTRIBUTE_PRIVATE

    /* 'flags' is currently not used. */
    if (attrs->flags)
        return -EINVAL;  //当前未使用该域
    if (attrs->attributes & ~supported_attrs)
        return -EINVAL;  //不允许有不支持的属性
    if (attrs->size == 0 || attrs->address + attrs->size < attrs->address)
        return -EINVAL;  //防止无效大小和大小太大导致了回绕
    if (!PAGE_ALIGNED(attrs->address) || !PAGE_ALIGNED(attrs->size))
        return -EINVAL;  //地址和大小必须对齐到页

    start = attrs->address >> PAGE_SHIFT; //得到起始 GFN
    end = (attrs->address + attrs->size - 1 + PAGE_SIZE) >> PAGE_SHIFT; //得到结束 GFN
    //对于私有内存地址，xarray 的 entry 内容为 KVM_MEMORY_ATTRIBUTE_PRIVATE
    entry = attrs->attributes ? xa_mk_value(attrs->attributes) : NULL;
    //如果是 TD VM，开始 invalidate，将起始和结束 GFN 加入 invalidate 范围
    if (kvm_arch_has_private_mem(kvm)) {
        KVM_MMU_LOCK(kvm);
        kvm_mmu_invalidate_begin(kvm);
        kvm_mmu_invalidate_range_add(kvm, start, end);
        KVM_MMU_UNLOCK(kvm);
    }
    //GFN 范围加入 per VM 的内存属性 xarray，key 为 GFN。对于私有内存 GFN，value 为 KVM_MEMORY_ATTRIBUTE_PRIVATE；对于共享内存 GFN，value 为 NULL
    mutex_lock(&kvm->lock);
    for (i = start; i < end; i++)
        if (xa_err(xa_store(&kvm->mem_attr_array, i, entry,
                    GFP_KERNEL_ACCOUNT)))
            break;
    mutex_unlock(&kvm->lock);
    //如果是 TD VM，清除实际 GFN 范围的映射，结束 invalidate
    if (kvm_arch_has_private_mem(kvm)) {
        idx = srcu_read_lock(&kvm->srcu);
        KVM_MMU_LOCK(kvm);
        if (i > start)
            kvm_unmap_mem_range(kvm, start, i, attrs->attributes);
        kvm_mmu_invalidate_end(kvm);
        KVM_MMU_UNLOCK(kvm);
        srcu_read_unlock(&kvm->srcu, idx);
    }
    //修正实际操作的地址和大小
    attrs->address = i << PAGE_SHIFT;
    attrs->size = (end - i) << PAGE_SHIFT;

    return 0;
}
```
* `kvm_unmap_mem_range()` 清除给定范围内存的映射，再设置该范围的内存属性
```cpp
static void kvm_unmap_mem_range(struct kvm *kvm, gfn_t start, gfn_t end,
                unsigned long attrs)
{
    struct kvm_gfn_range gfn_range;
    struct kvm_memory_slot *slot;
    struct kvm_memslots *slots;
    struct kvm_memslot_iter iter;
    int i;
    int r = 0;

    gfn_range.attrs = attrs;
    gfn_range.may_block = true;
    gfn_range.flags = KVM_GFN_RANGE_FLAGS_SET_MEM_ATTR;
    //遍历地址空间，另一个地址空间是 SMM
    for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
        slots = __kvm_memslots(kvm, i); //得到该地址空间的内存插槽
        //遍历每个内存插槽以覆盖要修改内存属性的地址范围
        kvm_for_each_memslot_in_gfn_range(&iter, slots, start, end) {
            slot = iter.slot;
            gfn_range.start = max(start, slot->base_gfn);
            gfn_range.end = min(end, slot->base_gfn + slot->npages);
            if (gfn_range.start >= gfn_range.end) //跳过大小为 0 的 slot
                continue;
            gfn_range.slot = slot;
            //清除本次迭代 GFN 范围的映射
            r |= kvm_unmap_gfn_range(kvm, &gfn_range);
            //设置内存的属性为指定的属性
            kvm_arch_set_memory_attributes(kvm, slot, attrs,
                               gfn_range.start,
                               gfn_range.end);
        }
    }

    if (r)
        kvm_flush_remote_tlbs(kvm);
}
```
* `kvm_unmap_gfn_range()` 取消给定范围内存的映射，它的调用者为：
  * `kvm_restrictedmem_invalidate_begin()`，`range->flags` 为 `KVM_GFN_RANGE_FLAGS_RESTRICTED_MEM`
    * 在 PUNCH_HOLE（`fallocate(PUNCH_HOLE)` or truncate）的时候走到这个分支，因为 restricted memory 意味着 slot 可以私有，因此不符合条件会触发警告
    * 该场景需要清除页面映射时，包括清除私有页面的映射，因此 `zap_private = true`
  * `kvm_unmap_mem_range()`，`range->flags` 为 `KVM_GFN_RANGE_FLAGS_SET_MEM_ATTR`
    * **共享转私有** 时，欲设置范围内存的属性是 `KVM_MEMORY_ATTRIBUTE_PRIVATE`，**不要** 清除私有页面的映射，`zap_private = false` 
    * **私有转共享** 时，欲设置范围内存的属性是 `0`，清除私有页面的映射，`zap_private = true` 
* 返回值指示要不要 flush TLB
```cpp
bool kvm_unmap_gfn_range(struct kvm *kvm, struct kvm_gfn_range *range)
{
    bool flush = false;
    //清除该范围内存的反向映射
    if (kvm_memslots_have_rmaps(kvm))
        flush = kvm_handle_gfn_range(kvm, range, kvm_zap_rmap);
    //如果 VM 使能了 TDP MMU
    if (is_tdp_mmu_enabled(kvm)) {
        bool zap_private;

        if (range->flags & KVM_GFN_RANGE_FLAGS_RESTRICTED_MEM) {
            WARN_ON_ONCE(!kvm_slot_can_be_private(range->slot));
            /*
             * For private slot, the callback is triggered
             * via PUNCH_HOLE (fallocate(PUNCH_HOLE) or truncate).
             * private-shared conversion is done by
             * KVM_SET_MEMORY_ATTRIBUTES.
             */
            zap_private = true;
        } else if (range->flags & KVM_GFN_RANGE_FLAGS_SET_MEM_ATTR)
            zap_private = !(range->attrs & KVM_MEMORY_ATTRIBUTE_PRIVATE);
        else
            /*
             * For now private pages are pinned during VM's life
             * time.
             */
            zap_private = false;
        flush = kvm_tdp_mmu_unmap_gfn_range(kvm, range, flush, zap_private);
    }

    return flush;
}
```
* `kvm_tdp_mmu_unmap_gfn_range()` 就是把参数展开调用 `kvm_tdp_mmu_zap_leafs()`
* `kvm_tdp_mmu_zap_leafs()` 就是遍历 VM 的根页表 `kvm->arch.tdp_mmu_roots`，然后从根页表开始清除指定范围的影子页表 `tdp_mmu_zap_leafs()`
* `zap_private && is_private_sp(root)` 这个表达式：
  * 请求是 **私有转共享**，入参 `zap_private = true`
    * 根页表是私有的，传入 `zap_private = true` 给 `tdp_mmu_zap_leafs()` 会去清除私有页面
    * 根页表是共享的，传入 `zap_private = false` 给 `tdp_mmu_zap_leafs()` 会去清除共享页面
      * 防止 `tdp_mmu_zap_leafs()` 中 `WARN_ON_ONCE(zap_private && !is_private_sp(root))` 的触发
  * 请求是 **共享转私有**，入参 `zap_private = false`
    * 根页表是私有的，传入 `zap_private = false` 给 `tdp_mmu_zap_leafs()` 会发现无事可做，直接返回
    * 根页表是共享的，传入 `zap_private = false` 给 `tdp_mmu_zap_leafs()` 会去清除共享页面
  * 可以看到，无论怎么转，范围内的共享页面总会被清除；而 **共享转私有** 时，原来私有的页面映射不清除

```cpp
/*
 * Zap leaf SPTEs for the range of gfns, [start, end), for all roots. Returns
 * true if a TLB flush is needed before releasing the MMU lock, i.e. if one or
 * more SPTEs were zapped since the MMU lock was last acquired.
 */
bool kvm_tdp_mmu_zap_leafs(struct kvm *kvm, int as_id, gfn_t start, gfn_t end,
               bool can_yield, bool flush, bool zap_private)
{
    struct kvm_mmu_page *root;

    for_each_tdp_mmu_root_yield_safe(kvm, root, as_id)
        flush = tdp_mmu_zap_leafs(kvm, root, start, end, can_yield, flush,
                      zap_private && is_private_sp(root));

    return flush;
}
...
/* Used by mmu notifier via kvm_unmap_gfn_range() */
bool kvm_tdp_mmu_unmap_gfn_range(struct kvm *kvm, struct kvm_gfn_range *range,
                 bool flush, bool zap_private)
{
    return kvm_tdp_mmu_zap_leafs(kvm, range->slot->as_id, range->start,
                     range->end, range->may_block, flush,
                     zap_private);
}
```
* `tdp_mmu_zap_leafs()` 从根页表开始清除指定范围的影子页表
```cpp
/*
 * If can_yield is true, will release the MMU lock and reschedule if the
 * scheduler needs the CPU or there is contention on the MMU lock. If this
 * function cannot yield, it will not release the MMU lock or reschedule and
 * the caller must ensure it does not supply too large a GFN range, or the
 * operation can cause a soft lockup.
 */
static bool tdp_mmu_zap_leafs(struct kvm *kvm, struct kvm_mmu_page *root,
                  gfn_t start, gfn_t end, bool can_yield, bool flush,
                  bool zap_private)
{
    struct tdp_iter iter;
    //将 TDP MMU walk 限定在 host.MAXPHYADDR
    end = min(end, tdp_mmu_max_gfn_exclusive());

    lockdep_assert_held_write(&kvm->mmu_lock);
    //根页表是共享的，请求是清除私有页表页，发出警告
    WARN_ON_ONCE(zap_private && !is_private_sp(root));
    if (!zap_private && is_private_sp(root)) //根页表是私有的，请求是不清除私有页表页
        return false;                        //没事可做，返回吧

    /*
     * start and end doesn't have GFN shared bit.  This function zaps
     * a region including alias.  Adjust shared bit of [start, end) if the
     * root is shared.
     */
    start = kvm_gfn_for_root(kvm, root, start);
    end = kvm_gfn_for_root(kvm, root, end);

    rcu_read_lock();

    for_each_tdp_pte_min_level(iter, root, PG_LEVEL_4K, start, end) {
        if (can_yield &&
            tdp_mmu_iter_cond_resched(kvm, &iter, flush, false)) {
            flush = false;
            continue;
        }
        //跳过未映射的 SPTE 或非叶子 SPTE
        if (!is_shadow_present_pte(iter.old_spte) ||
            !is_last_spte(iter.old_spte, iter.level))
            continue;
        //只将最后一级 SPTE 清除映射，改动会被传播到 TDX modules
        tdp_mmu_set_spte(kvm, &iter, SHADOW_NONPRESENT_VALUE);
        flush = true;
    }

    rcu_read_unlock();

    /*
     * Because this flow zaps _only_ leaf SPTEs, the caller doesn't need
     * to provide RCU protection as no 'struct kvm_mmu_page' will be freed.
     */
    return flush;
}
```
### `kvm_arch_set_memory_attributes()` 更新 slot 的 `lpage_info` 数组元素的标志位
* `kvm_arch_set_memory_attributes()` 对可以私有化的 slot 调用 `kvm_update_lpage_private_shared_mixed()` 更新 lpage_info 中是否有私有和共享页混合的情况对应的标志位
```cpp
static void kvm_update_lpage_private_shared_mixed(struct kvm *kvm,
                          struct kvm_memory_slot *slot,
                          unsigned long attrs,
                          gfn_t start, gfn_t end)
{
    unsigned long pages, mask;
    gfn_t gfn, gfn_end, first, last;
    int level;
    bool mixed;

    /*
     * The sequence matters here: we set the higher level basing on the
     * lower level's scanning result.
     */
    for (level = PG_LEVEL_2M; level <= KVM_MAX_HUGEPAGE_LEVEL; level++) {
        pages = KVM_PAGES_PER_HPAGE(level); //这一级一个页覆盖的页帧数，比如 2MB 的大页含有 512 个页帧
        mask = ~(pages - 1);     //比如 2MB 的大页这个值就是 511
        first = start & mask;    //范围在该级的起始 GFN
        last = (end - 1) & mask; //范围在该级的结束 GFN
        //只检查范围起始页和结束页，因为中间的页要么是加密要么是不加密，不可能是混合着的
        /*
         * We only need to scan the head and tail page, for middle pages
         * we know they will not be mixed.
         */
        gfn = max(first, slot->base_gfn); //范围在该 slot 的起始 GFN
        gfn_end = min(first + pages, slot->base_gfn + slot->npages); //范围在该 slot 的结束 GFN
        mixed = mem_attrs_mixed(kvm, slot, level, attrs, gfn, gfn_end); //范围在该 slot 内是否有属性混合
        linfo_set_mixed(gfn, slot, level, mixed); //设置该 slot 里的 linfo 的混合标志位
        //如果扫描范围结束，返回
        if (first == last)
            return;
        //中间的页要么是加密要么是不加密，不可能是混合着的
        for (gfn = first + pages; gfn < last; gfn += pages)
            linfo_set_mixed(gfn, slot, level, false);
        //检查并处理结束页面
        gfn = last;
        gfn_end = min(last + pages, slot->base_gfn + slot->npages);
        mixed = mem_attrs_mixed(kvm, slot, level, attrs, gfn, gfn_end);
        linfo_set_mixed(gfn, slot, level, mixed);
    }
}

void kvm_arch_set_memory_attributes(struct kvm *kvm,
                    struct kvm_memory_slot *slot,
                    unsigned long attrs,
                    gfn_t start, gfn_t end)
{
    if (kvm_slot_can_be_private(slot))
        kvm_update_lpage_private_shared_mixed(kvm, slot, attrs,
                              start, end);
}
```
* `is_expected_attr_entry()` 用于判断传入的 `kvm->mem_attr_array` xarray 内存条目属性是否是期盼的属性
```cpp
static bool is_expected_attr_entry(void *entry, unsigned long expected_attrs)
{   //变量表明期盼的属性是否是私有
    bool expect_private = expected_attrs & KVM_MEMORY_ATTRIBUTE_PRIVATE;

    if (xa_to_value(entry) & KVM_MEMORY_ATTRIBUTE_PRIVATE) { //条目的属性是私有
        if (!expect_private)   //不期盼条目是私有
            return false;      //条目不是期盼的属性
    } else if (expect_private) //条目的属性是共享，且期盼的属性是私有
        return false;          //条目不是期盼的属性
    //不满足以上情况的条目是期盼的属性
    return true;
}

static bool mem_attrs_mixed(struct kvm *kvm, struct kvm_memory_slot *slot,
                int level, unsigned long attrs,
                gfn_t start, gfn_t end)
{
    unsigned long gfn;
    //判断 2M 这一级的页表下的页是否有属性混合
    if (level == PG_LEVEL_2M)
        return mem_attrs_mixed_2m(kvm, attrs, start, end);

    for (gfn = start; gfn < end; gfn += KVM_PAGES_PER_HPAGE(level - 1))
        if (linfo_is_mixed(lpage_info_slot(gfn, slot, level - 1)) ||
            !is_expected_attr_entry(xa_load(&kvm->mem_attr_array, gfn),
                        attrs))
            return true;
    return false;
}
```
### `lpage_info[]` 数组
* `lpage_info[]` 数组其实是个二维数组
  * 第一维是该 slot 所能支持大页的种类
  * 第二维是该 slot 与每个该种类的大页对应的一个 `struct kvm_lpage_info` 类型的元素
* `KVM_NR_PAGE_SIZES` 为所支持页大小的种类的数目，比如说最大支持 512 GB 的大页，那么就有 512 GB、1 GB、2 MB、4 KB 四种类型的页
```cpp
struct kvm_lpage_info {
    int disallow_lpage;
};

struct kvm_arch_memory_slot {
    struct kvm_rmap_head *rmap[KVM_NR_PAGE_SIZES];
    struct kvm_lpage_info *lpage_info[KVM_NR_PAGE_SIZES - 1];
    unsigned short *gfn_track[KVM_PAGE_TRACK_MAX];
};
```
* 那么 `lpage_info[]` 数组的元素个数是所能支持大页的种类，比如最大支持 512 GB 的大页，那就是 512 GB、1 GB 和 2 MB 三种类型的大页，则此数组有三个元素
* 每个 slot 的起始和结束页帧未必能对应到所支持大页类型要求的起始页帧，因此首尾可能不允许该级的大页，用 `struct kvm_lpage_info` 的 `disallow_lpage` 域表示
* 分配 `lpage_info[]` 数组的路径如下：
```cpp
kvm_set_memslot()
-> kvm_prepare_memory_region()
   -> kvm_arch_prepare_memory_region()
      -> kvm_alloc_memslot_metadata()
```
* 分配 `lpage_info[]` 数组的函数如下：
```cpp
static int kvm_alloc_memslot_metadata(struct kvm *kvm,
                      struct kvm_memory_slot *slot)
{
    unsigned long npages = slot->npages;
    int i, r;

    /*
     * Clear out the previous array pointers for the KVM_MR_MOVE case.  The
     * old arrays will be freed by __kvm_set_memory_region() if installing
     * the new memslot is successful.
     */
    memset(&slot->arch, 0, sizeof(slot->arch));
    //如使能 TDP MMU 不分配 rmap[] 数组
    if (kvm_memslots_have_rmaps(kvm)) {
        r = memslot_rmap_alloc(slot, npages);
        if (r)
            return r;
    }
    //因为处理的是大页，所以第一层的 4K 页不算，索引从 1 开始
    for (i = 1; i < KVM_NR_PAGE_SIZES; ++i) {
        struct kvm_lpage_info *linfo;
        unsigned long ugfn;
        int lpages;
        int level = i + 1;
        //该 slot 上能存放几个这一级的大页
        lpages = __kvm_mmu_slot_lpages(slot, npages, level);
        //这一级的每个大页一个 linfo 类型的数组元素
        linfo = __vcalloc(lpages, sizeof(*linfo), GFP_KERNEL_ACCOUNT);
        if (!linfo)
            goto out_free;
        //lpage_info 数组指向每个大页一个 linfo 类型的数组，形成一个二维数组结构
        slot->arch.lpage_info[i - 1] = linfo;
        //只需要检测 slot 的两头是不是允许大页就可以了，中间肯定是放得下大页的，比如 2MB 大页只需检查首尾 GFN 是否对齐到 512 即可
        if (slot->base_gfn & (KVM_PAGES_PER_HPAGE(level) - 1))
            linfo[0].disallow_lpage = 1;
        if ((slot->base_gfn + npages) & (KVM_PAGES_PER_HPAGE(level) - 1))
            linfo[lpages - 1].disallow_lpage = 1;
        ugfn = slot->userspace_addr >> PAGE_SHIFT; //slot 的 HVA 转 HVFN
        if (kvm_slot_can_be_private(slot))
            ugfn |= slot->restricted_offset >> PAGE_SHIFT; //restricted_offset 是限制文件的起始位置在 slot 中的偏移
        /*
         * If the gfn and userspace address are not aligned wrt each
         * other, disable large page support for this slot.
         */
        //如果 GFN 和用户空间的地址（HVFN），包括 restricted 文件页面，不对齐，禁用该 slot 在该级的大页支持
        if ((slot->base_gfn ^ ugfn) & (KVM_PAGES_PER_HPAGE(level) - 1)) {
            unsigned long j;

            for (j = 0; j < lpages; ++j)
                linfo[j].disallow_lpage = 1;
        }
    }
    //初始化 gfn_track[] 数组，目前就 KVM_PAGE_TRACK_WRITE 一个元素
    if (kvm_page_track_create_memslot(kvm, slot, npages))
        goto out_free;

    return 0;

out_free:
    memslot_rmap_free(slot);

    for (i = 1; i < KVM_NR_PAGE_SIZES; ++i) {
        kvfree(slot->arch.lpage_info[i - 1]);
        slot->arch.lpage_info[i - 1] = NULL;
    }
    return -ENOMEM;
}
```

## KVM: x86/tdp_mmu: implement MapGPA hypercall for TDX
```c
commit cf25c146de54c6248623b6e0a709342a2b84f6a3
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Tue Jan 11 17:34:17 2022 -0800

    KVM: x86/tdp_mmu: implement MapGPA hypercall for TDX

    The TDX Guest-Hypervisor communication interface(GHCI) specification
    defines MapGPA hypercall for guest TD to request the host VMM to map given
    GPA range as private or shared.

    It means the guest TD uses the GPA as shared (or private).  The GPA
    won't be used as private (or shared).  VMM should enforce GPA usage. VMM
    doesn't have to map the GPA on the hypercall request.

    - Zap the aliased region.
      If shared (or private) GPA is requested, zap private (or shared) GPA
      (modulo shared bit).
    - Record the request GPA is shared (or private) by kvm.mem_attr_array.
    - Don't map GPA. The GPA is mapped on the next EPT violation.
```