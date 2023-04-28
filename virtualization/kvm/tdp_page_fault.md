# KVM Page Fault

## 对事件的反应

### Guest 缺页（或 EPT violation）
* 这是最复杂的事件。页面错误的原因可能是：
  * 真正的 guest 错误（guest 翻译不允许访问）
    * 不适用于直接模式
  * 访问页面翻译不存在
  * 访问受保护的翻译
    * 当 logging 脏页时，内存被写保护
    * 同步影子页面被写保护
      * 不适用于直接模式
  * 访问不可翻译的内存（MMIO）

#### 缺页处理如下
* 如果设置了 error code 的 `RSV` 位，则缺页是由于 guest 访问 MMIO 引起的，缓存的 MMIO 信息可用
  * 遍历影子页表
  * 检查 spte 中的有效 gerneration 号（参见下面的“MMIO sptes 的快速失效”）
  * 将信息缓存到 `vcpu->arch.mmio_gva`、`vcpu->arch.mmio_access` 和 `vcpu->arch.mmio_gfn`，调用 emulator
* 如果 error code 的 `P` 位和 `R/W` 位都被设置，这可能会被处理为“快速页面错误”（在不获取 MMU 锁的情况下修复）
* 如果需要，遍历 guest 页表以确定 guest 的翻译（gva->gpa 或 ngpa->gpa）
  * 如果权限不足，将 fault 反馈给 guest
* 确定 host 页面
  * 如果这是一个 MMIO 请求，则没有 host 页面；将信息缓存到 `vcpu->arch.mmio_gva`、`vcpu->arch.mmio_access` 和 `vcpu->arch.mmio_gfn`
* 遍历影子页表以找到翻译的 SPTE，根据需要实例化缺失的中间页表
  * 如果这是一个 MMIO 请求，将 MMIO 信息缓存到 SPTE 并在 SPTE 上设置一些保留位（参见 `kvm_mmu_set_mmio_spte_mask` 的调用者）
* 尝试取消同步页面
  * 如果成功，我们可以让 guest 继续并修改 GPTE
* 模拟指令
  * 如果失败，unshadow 页面并让 guest 继续
* 更新指令修改的任何翻译

## union kvm_mmu_page_role
* 联合体 `kvm_mmu_page_role`
* arch/x86/include/asm/kvm_host.h
```cpp
/*
 * kvm_mmu_page_role tracks the properties of a shadow page (where shadow page
 * also includes TDP pages) to determine whether or not a page can be used in
 * the given MMU context.  This is a subset of the overall kvm_cpu_role to
 * minimize the size of kvm_memory_slot.arch.gfn_track, i.e. allows allocating
 * 2 bytes per gfn instead of 4 bytes per gfn.
 *
 * Upper-level shadow pages having gptes are tracked for write-protection via
 * gfn_track.  As above, gfn_track is a 16 bit counter, so KVM must not create
 * more than 2^16-1 upper-level shadow pages at a single gfn, otherwise
 * gfn_track will overflow and explosions will ensure.
 *
 * A unique shadow page (SP) for a gfn is created if and only if an existing SP
 * cannot be reused.  The ability to reuse a SP is tracked by its role, which
 * incorporates various mode bits and properties of the SP.  Roughly speaking,
 * the number of unique SPs that can theoretically be created is 2^n, where n
 * is the number of bits that are used to compute the role.
 *
 * But, even though there are 19 bits in the mask below, not all combinations
 * of modes and flags are possible:
 *
 *   - invalid shadow pages are not accounted, so the bits are effectively 18
 *
 *   - quadrant will only be used if has_4_byte_gpte=1 (non-PAE paging);
 *     execonly and ad_disabled are only used for nested EPT which has
 *     has_4_byte_gpte=0.  Therefore, 2 bits are always unused.
 *
 *   - the 4 bits of level are effectively limited to the values 2/3/4/5,
 *     as 4k SPs are not tracked (allowed to go unsync).  In addition non-PAE
 *     paging has exactly one upper level, making level completely redundant
 *     when has_4_byte_gpte=1.
 *
 *   - on top of this, smep_andnot_wp and smap_andnot_wp are only set if
 *     cr0_wp=0, therefore these three bits only give rise to 5 possibilities.
 *
 * Therefore, the maximum number of possible upper-level shadow pages for a
 * single gfn is a bit less than 2^13.
 */
union kvm_mmu_page_role {
    u32 word;
    struct {
        unsigned level:4;
        unsigned has_4_byte_gpte:1;
        unsigned quadrant:2;
        unsigned direct:1;
        unsigned access:3;
        unsigned invalid:1;
        unsigned efer_nx:1;
        unsigned cr0_wp:1;
        unsigned smep_andnot_wp:1;
        unsigned smap_andnot_wp:1;
        unsigned ad_disabled:1;
        unsigned guest_mode:1;
        unsigned passthrough:1;
        unsigned :5;

        /*
         * This is left at the top of the word so that
         * kvm_memslots_for_spte_role can extract it with a
         * simple shift.  While there is room, give it a whole
         * byte so it is also faster to load it from memory.
         */
        unsigned smm:8;
    };
};
```
* `level`: 此影子页面在其所属的影子分页层次结构中的级别。`1=4k` sptes，`2=2M` sptes，`3=1G` sptes，等等
* `direct`：如果设为 `1`，从该页面可到达的叶子节点是一个线性范围
   * 例如，包含实模式在内的地址翻译，后备是小的 host 页面的 guest 大页，以及 NPT 或 EPT 被激活时的 gpa->hpa 的转换。
   * 线性范围从 (`gfn << PAGE_SHIFT`) 开始，其大小由 `role.level` 决定（一级 `2MB`，二级 `1GB`，三级 `0.5TB`，四级 `256TB`）
   * 如果设为 `0`，则此页面对应于 `gfn` 域表示的 guest 页表
* `quadrant`：象限
* `root_role` 的初始化路径
```cpp
kvm_vm_ioctl()
case KVM_CREATE_VCPU:
     kvm_vm_ioctl_create_vcpu(kvm, arg)
     -> kvm_arch_vcpu_create(vcpu)
        -> kvm_init_mmu(vcpu)
           else if (tdp_enabled)
           -> init_kvm_tdp_mmu(vcpu, cpu_role)
                 struct kvm_mmu *context = &vcpu->arch.root_mmu;
              -> union kvm_mmu_page_role root_role = kvm_calc_tdp_mmu_root_page_role(vcpu, cpu_role)
                 context->root_role.word = root_role.word;
```
* `word` 域和匿名 `struct` 是联合体，所以通过结构体赋值的方式初始化了`vcpu->arch.root_mmu.root_role`

### `root_role` 的初始值
* arch/x86/kvm/mmu/mmu.c
```cpp
static inline int kvm_mmu_get_tdp_level(struct kvm_vcpu *vcpu)
{
    /* tdp_root_level is architecture forced level, use it if nonzero */
    if (tdp_root_level)
        return tdp_root_level;

    /* Use 5-level TDP if and only if it's useful/necessary. */
    if (max_tdp_level == 5 && cpuid_maxphyaddr(vcpu) <= 48)
        return 4;

    return max_tdp_level;
}

static union kvm_mmu_page_role
kvm_calc_tdp_mmu_root_page_role(struct kvm_vcpu *vcpu,
                union kvm_cpu_role cpu_role)
{
    union kvm_mmu_page_role role = {0};

    role.access = ACC_ALL;
    role.cr0_wp = true;
    role.efer_nx = true;
    role.smm = cpu_role.base.smm;
    role.guest_mode = cpu_role.base.guest_mode;
    role.ad_disabled = !kvm_ad_enabled();
    role.level = kvm_mmu_get_tdp_level(vcpu);
    role.direct = true;
    role.has_4_byte_gpte = false;

    return role;
}
```
* 其中 `role.level` 的值由 `tdp_root_level` 和 `max_tdp_level` 决定，而这两个值由虚拟机配置和 VMX 所能支持的 EPT 级数决定
  * arch/x86/kvm/vmx/capabilities.h
  ```cpp
  static inline bool cpu_has_vmx_ept_5levels(void)
  {
      return vmx_capability.ept & VMX_EPT_PAGE_WALK_5_BIT;
  }
  ```
  * arch/x86/kvm/vmx/vmx.c
  ```cpp
  static int vmx_get_max_tdp_level(void)
  {
      if (cpu_has_vmx_ept_5levels())
          return 5;
      return 4;
  }
  
  void kvm_configure_mmu(bool enable_tdp, int tdp_forced_root_level,
                 int tdp_max_root_level, int tdp_huge_page_level)
  {
      tdp_enabled = enable_tdp;
      tdp_root_level = tdp_forced_root_level;
      max_tdp_level = tdp_max_root_level;
      ...
  }
  static __init int hardware_setup(void)
  {
      ...
      kvm_configure_mmu(enable_ept, 0, vmx_get_max_tdp_level(),
                ept_caps_to_lpage_level(vmx_capability.ept));
      ...
  }
  ```

## struct kvm_mmu_page
* `struct kvm_mmu_page`是影子页面的主要数据结构
  * 影子页面包含 512 个 spte，可以是叶 spte，也可以是非叶 spte
  * 影子页面可能包含叶子和非叶子的混合
* arch/x86/kvm/mmu/mmu_internal.h
```cpp
struct kvm_mmu_page {
    /*
     * Note, "link" through "spt" fit in a single 64 byte cache line on
     * 64-bit kernels, keep it that way unless there's a reason not to.
     */
    struct list_head link;
    struct hlist_node hash_link;

    bool tdp_mmu_page;
    bool unsync;
    u8 mmu_valid_gen;
    bool lpage_disallowed; /* Can't be replaced by an equiv large page */

    /*
     * The following two entries are used to key the shadow page in the
     * hash table.
     */
    union kvm_mmu_page_role role;
    gfn_t gfn;

    u64 *spt;

    /*
     * Stores the result of the guest translation being shadowed by each
     * SPTE.  KVM shadows two types of guest translations: nGPA -> GPA
     * (shadow EPT/NPT) and GVA -> GPA (traditional shadow paging). In both
     * cases the result of the translation is a GPA and a set of access
     * constraints.
     *
     * The GFN is stored in the upper bits (PAGE_SHIFT) and the shadowed
     * access permissions are stored in the lower bits. Note, for
     * convenience and uniformity across guests, the access permissions are
     * stored in KVM format (e.g.  ACC_EXEC_MASK) not the raw guest format.
     */
    u64 *shadowed_translation;

    /* Currently serving as active root */
    union {
        int root_count;
        refcount_t tdp_mmu_root_count;
    };
    unsigned int unsync_children;
    union {
        struct kvm_rmap_head parent_ptes; /* rmap pointers to parent sptes */
        tdp_ptep_t ptep;
    };
    union {
        DECLARE_BITMAP(unsync_child_bitmap, 512);
        struct {
            struct work_struct tdp_mmu_async_work;
            void *tdp_mmu_async_data;
        };
    };

    struct list_head lpage_disallowed_link;
#ifdef CONFIG_X86_32
    /*
     * Used out of the mmu-lock to avoid reading spte values while an
     * update is in progress; see the comments in __get_spte_lockless().
     */
    int clear_spte_count;
#endif

    /* Number of writes since the last time traversal visited this page.  */
    atomic_t write_flooding_count;

#ifdef CONFIG_X86_64
    /* Used for freeing the page asynchronously if it is a TDP MMU page. */
    struct rcu_head rcu_head;
#endif
};
```
* `link`：`link` 将该结构链接到 `kvm->arch.active_mmu_pages` 和 `invalid_list` 上，标注该页结构不同的状态
* `hash_link`：`hash_link` 将该结构链接到 `kvm->arch.mmu_page_hash` 哈希表上，以便进行快速查找，hash key 由接下来的 `gfn` 和 `role` 决定
* `gfn`：含义与 `.role.direct` 相关：
   * `.role.direct = 1`：在直接映射时，保存的是线性映射的基页帧；来自于 `tdp_mmu_init_sp(child_sp, iter->sptep, iter->gfn, role)` 中的 `iter->gfn`
   * `.role.direct = 0`：在非直接映射时，保存的是 guest page table，该页表包含了由该页映射的 translation
* `role`：该页的“角色”，详细参见上文对 `union kvm_mmu_page_role` 的说明
* `spt`：对应的 SPT/EPT 页帧的地址（HVA），SPT/EPT 页的 `struct page` 结构中 `page->private` 域会反向指向该 `struct kvm_mmu_page`
  * 该域可以指向一个 lower-level shadow page，也可以指向真正的数据 page
  * 见 `tdp_mmu_init_sp(struct kvm_mmu_page *sp, ...) -> set_page_private(virt_to_page(sp->spt), (unsigned long)sp)`
* `parent_ptes`：指向上一级 `spt`，表示有哪些上一级页表页的页表项指向该页表页
* `unsync`：该域只对页结构的叶子节点有效，可以执行该页的翻译是否与 guest 的翻译一致
  * 如果为 `false`，则可能出现修改了该页中的 pte 但没有更新 TLB，而 guest 读取了 TLB 中的数据，导致了不一致
* `root_count`：该页被多少个 vcpu 作为根页结构
* `unsync_children`：记录该页结构下面有多少个子节点是 unsync 状态的
* `mmu_valid_gen`：该页的 generation number
   * KVM维护了一个全局的的 gen number（`kvm->arch.mmu_valid_gen`），如果该域与全局的 gen number 不相等，则将该页标记为 invalid page
   * 该结构用来快速的碾压掉 KVM 的 MMU paging structure
     * 例如，如果想废弃掉当前所有的MMU页结构，需要处理掉所有的MMU页面和对应的映射；
     * 但是通过该结构，可以直接将 `kvm->arch.mmu_valid_gen` 加 `1`，那么当前所有的 MMU 页结构都变成了 invalid，而处理掉页结构的过程可以留给后面的过程（如内存不够时）再处理，可以加快废弃所有 MMU 页结构的速度。
     * 当 `mmu_valid_gen` 值达到最大时，可以调用 `kvm_mmu_invalidate_zap_all_pages` 手动废弃掉所有的 MMU 页结构
* `unsync_child_bitmap`：记录了 unsync 的子结构的位图
* `clear_spte_count`：仅针对 `32` 位 host 有效，具体作用参考函数 `__get_spte_lockless` 的注释
* `write_flooding_count`：在写保护模式下，对于任何一个页的写都会导致 KVM 进行一次 emulation
   * 对于叶子节点（真正指向数据页的节点），可以使用 unsync 状态来保护频繁的写操作不会导致大量的 emulation
   * 但是对于非叶子节点（paging structure节点）则不行。对于非叶子节点的写 emulation 会修改该域，如果写 emulation 非常频繁，KVM 会 unmap 该页以避免过多的写 emulation
### 转换函数 `to_shadow_page()` 和 `sptep_to_sp()`
* 两个频繁用到的转换函数 `to_shadow_page()` 和 `sptep_to_sp()`
* `to_shadow_page()` 用于将 HPA 转为 KVM 中指向其对应的 `struct kvm_mmu_page` 结构实例的指针，利用了 `struct page` 结构中 `page->private` 域会反向指向该`struct kvm_mmu_page` 的特性
* `sptep_to_sp()` 用于将 sptep（HVA）转为 KVM 中指向其对应的 `struct kvm_mmu_page` 结构实例的指针
* arch/x86/kvm/mmu/mmu_internal.h
```cpp
static inline struct kvm_mmu_page *to_shadow_page(hpa_t shadow_page)
{
    struct page *page = pfn_to_page(shadow_page >> PAGE_SHIFT);

    return (struct kvm_mmu_page *)page_private(page);
}

static inline struct kvm_mmu_page *sptep_to_sp(u64 *sptep)
{
    return to_shadow_page(__pa(sptep));
}
```

## Page Fault Handling
```cpp
vcpu_run()
-> vcpu_enter_guest()
   -> kvm_mmu_reload(vcpu)
      -> kvm_mmu_load(vcpu)
         -> mmu_topup_memory_caches(vcpu, !vcpu->arch.mmu->root_role.direct)
         -> mmu_alloc_special_roots(vcpu)
            if (vcpu->arch.mmu->root_role.direct)
            -> mmu_alloc_direct_roots(vcpu)
                  if (is_tdp_mmu_enabled(vcpu->kvm)) {
                     root = kvm_tdp_mmu_get_vcpu_root_hpa(vcpu);
                               union kvm_mmu_page_role role = vcpu->arch.mmu->root_role;
                               struct kvm_mmu_page *root;
                            -> root = tdp_mmu_alloc_sp(vcpu)
                               -> sp = kvm_mmu_memory_cache_alloc(&vcpu->arch.mmu_page_header_cache)
                               -> sp->spt = kvm_mmu_memory_cache_alloc(&vcpu->arch.mmu_shadow_page_cache)
                            -> tdp_mmu_init_sp(root, NULL, 0, role)
                               -> set_page_private(virt_to_page(sp->spt), (unsigned long)sp)
                                  sp->tdp_mmu_page = true; //表明这是一个 TDP MMU 页面而不是 shadow 页面
                            -> list_add_rcu(&root->link, &kvm->arch.tdp_mmu_roots)
                            -> return __pa(root->spt); //返回的是页面的物理地址
                     mmu->root.hpa = root;
                  }
         -> kvm_mmu_sync_roots(vcpu)
         -> kvm_mmu_load_pgd(vcpu)
               root_hpa = vcpu->arch.mmu->root.hpa;
            -> static_call(kvm_x86_load_mmu_pgd)(vcpu, root_hpa, vcpu->arch.mmu->root_role.level)
         -> static_call(kvm_x86_flush_tlb_current)(vcpu)
   -> exit_fastpath = static_call(kvm_x86_vcpu_run)(vcpu)
   => vmx_vcpu_run(vcpu)
   -> static_call(kvm_x86_handle_exit)(vcpu, exit_fastpath)
   => vmx_handle_exit()
      -> __vmx_handle_exit()
         -> kvm_vmx_exit_handlers[exit_handler_index](vcpu)
         => handle_ept_violation()
            -> kvm_mmu_page_fault(vcpu, gpa, error_code, NULL, 0)
               -> kvm_mmu_do_page_fault()
                  -> kvm_tdp_page_fault(vcpu, &fault)
                     -> direct_page_fault(vcpu, fault)
                           bool is_tdp_mmu_fault = is_tdp_mmu(vcpu->arch.mmu);
                           fault->gfn = fault->addr >> PAGE_SHIFT;
                        -> fault->slot = kvm_vcpu_gfn_to_memslot(vcpu, fault->gfn)
                        -> fast_page_fault(vcpu, fault)
                        -> mmu_topup_memory_caches(vcpu, false)
                        -> kvm_faultin_pfn(vcpu, fault)
                           -> fault->pfn = __gfn_to_pfn_memslot(slot, fault->gfn, false, &async,
                                             fault->write, &fault->map_writable, &fault->hva)
                                           -> addr = __gfn_to_hva_many(slot, gfn, NULL, write_fault)
                                              *hva = addr;
                                           -> hva_to_pfn(addr, atomic, async, write_fault, writable)
                                              -> hva_to_pfn_fast()
                                                 -> get_user_page_fast_only()
                                              -> hva_to_pfn_slow()
                                              -> hva_to_pfn_remapped()
                        -> handle_abnormal_pfn(vcpu, fault, ACC_ALL)
                        -> make_mmu_pages_available(vcpu)
                           if (is_tdp_mmu_fault)
                              r = kvm_tdp_mmu_map(vcpu, fault)
                           else
                              r = __direct_map(vcpu, fault)
```

## TDP Iterator
* TDP Iterator 以先序遍历迭代所映射范围为 `[start, end)` 的 GFN 里的每个 SPTE
* 基本算法如下：
1. 如果当前 SPTE 并非最下一级 SPTE，则向下步入它指向的页表
2. 如果迭代器不能向下步入（说明它正处于最下一级页表），它会尝试步进到分页结构的当前页中的下一条 SPTE
3. 如果迭代器不能步进到当前页面的下一个条目，它会尝试回退到父分页结构页面。在这种情况下，该 SPTE 已经被访问过，因此迭代器也必须再次步进到下一条目
* arch/x86/kvm/mmu/tdp_mmu.c
```cpp
#define tdp_mmu_for_each_pte(_iter, _mmu, _start, _end)     \
    for_each_tdp_pte(_iter, to_shadow_page(_mmu->root.hpa), _start, _end)
```
* 虚拟 MMU 的 `root.hpa` 用 `to_shadow_page(_mmu->root.hpa)` 转为指向其对应的 `struct kvm_mmu_page` 实例的指针
* arch/x86/kvm/mmu/tdp_iter.h
```cpp
/*
 * Iterates over every SPTE mapping the GFN range [start, end) in a
 * preorder traversal.
 */
#define for_each_tdp_pte_min_level(iter, root, min_level, start, end) \
    for (tdp_iter_start(&iter, root, min_level, start); \
         iter.valid && iter.gfn < end;           \
         tdp_iter_next(&iter))

#define for_each_tdp_pte(iter, root, start, end) \
    for_each_tdp_pte_min_level(iter, root, PG_LEVEL_4K, start, end)
```
* `struct tdp_iter` 是迭代器的核心数据结构
  * `gfn_t next_last_level_gfn`：遍历的起始 GFN；如果遍历被中断过，恢复后从这个 GFN 开始继续遍历（尽管 `tdp_iter_restart()` 把 `iter->level` 重设回了 `iter->root_level`）
  * `tdp_ptep_t sptep`：指向当前遍历到的 SPTE 的指针，当然是个虚拟地址
  * `u64 old_spte`：`sptep` 指向的条目里的内容的快照，页表条目里的内容当然是物理地址了
  * `gfn_t gfn`：当前 SPTE 所在页表页的最低 GFN，会通过 `tdp_mmu_init_sp()` 赋值给 `struct kvm_mmu_page` 的 `gfn` 域
  * `tdp_ptep_t pt_path[PT64_ROOT_MAX_LEVEL]`：存储遍历到当前 SPTE 所经过的页表页的虚拟地址
* `tdp_iter_next()` 在分页结构中以先序遍历进入下一个 SPTE。为了到达下一个 SPTE，迭代器
  * 要么向下转向目标 GFN（如果当前是非最后一级 SPTE）
  * 要么转到映射更高 GFN 的 SPTE
#### GFN 对齐函数 `round_gfn_for_level()`
* `round_gfn_for_level()` 用于取对齐到不同层级的 GFN，分段为 9-9-9-9-9
```cpp
static gfn_t round_gfn_for_level(gfn_t gfn, int level)
{
    return gfn & -KVM_PAGES_PER_HPAGE(level);
}
```
* 对于不同的 `level`，`-KVM_PAGES_PER_HPAGE(level)` 的值分别是：
  * `5`：`0xfffffff000000000`
  * `4`：`0xfffffffff8000000`
  * `3`：`0xfffffffffffc0000`
  * `2`：`0xfffffffffffffe00`
  * `1`：`0xffffffffffffffff`
* 最后一级相当于不做对齐，这就是 `try_step_down()` 在最后一级时能让 `iter->gfn` 为 `fault->gfn` 的原因
  * 见 `iter->gfn = round_gfn_for_level(iter->next_last_level_gfn, iter->level)`
  * 故而随后的 `tdp_iter_refresh_sptep(iter)` 能让 `iter->sptep` 指向 `fault->gfn` 对应的 SPTE

#### SPTE 转页表函数 `spte_to_child_pt()`
* 该函数入参为 SPTE 的内容（PA）和该 SPTE 的层级，返回该条目所指向的子页表页的虚拟地址
  * 因为期望返回的是页表页，所以当 PTE 条目不存在或已是最后一级（即叶子节点，数据页）时，返回 `NULL`
```cpp
/*
 * Given an SPTE and its level, returns a pointer containing the host virtual
 * address of the child page table referenced by the SPTE. Returns null if
 * there is no such entry.
 */
tdp_ptep_t spte_to_child_pt(u64 spte, int level)
{
    /*
     * There's no child entry if this entry isn't present or is a
     * last-level entry.
     */
    if (!is_shadow_present_pte(spte) || is_last_spte(spte, level))
        return NULL;
    //PA -> KVM PFN -> 对齐的 PA -> VA，通过这个 VA 操作页表
    return (tdp_ptep_t)__va(spte_to_pfn(spte) << PAGE_SHIFT);
}
```

## 建立 TDP MMU 映射 `kvm_tdp_mmu_map()`
* arch/x86/kvm/mmu/tdp_mmu.c
```cpp
static void tdp_mmu_init_sp(struct kvm_mmu_page *sp, tdp_ptep_t sptep,
                gfn_t gfn, union kvm_mmu_page_role role)
{   //这里设置了 page->private 回指 kvm_mmu_page 结构
    set_page_private(virt_to_page(sp->spt), (unsigned long)sp);

    sp->role = role; //子继承父的 role
    sp->gfn = gfn; //新页表页的起始页帧
    sp->ptep = sptep; //设置页表页的父页表页的虚拟地址，由于是新分配的，必然只有一个父页表页
    sp->tdp_mmu_page = true;

    trace_kvm_mmu_get_page(sp, true);
}

static void tdp_mmu_init_child_sp(struct kvm_mmu_page *child_sp,
                  struct tdp_iter *iter)
{
    struct kvm_mmu_page *parent_sp;
    union kvm_mmu_page_role role;
    //根据父页表页的虚拟地址得到其对应的 kvm_mmu_page 指针
    parent_sp = sptep_to_sp(rcu_dereference(iter->sptep));

    role = parent_sp->role; //子继承父的 role
    role.level--; //往下走一级
    //初始化子页表页对应的 kvm_mmu_page 实例的各个域
    tdp_mmu_init_sp(child_sp, iter->sptep, iter->gfn, role);
}
...
/*
 * Installs a last-level SPTE to handle a TDP page fault.
 * (NPT/EPT violation/misconfiguration)
 */
static int tdp_mmu_map_handle_target_level(struct kvm_vcpu *vcpu,
					  struct kvm_page_fault *fault,
					  struct tdp_iter *iter)
{
	struct kvm_mmu_page *sp = sptep_to_sp(rcu_dereference(iter->sptep));
	u64 new_spte;
	int ret = RET_PF_FIXED;
	bool wrprot = false;
	//能走到这里目标级别和当前影子叶子页表页的级别应该是一致的，否则发出警告
	if (WARN_ON_ONCE(sp->role.level != fault->goal_level))
		return RET_PF_RETRY;
	//new_spte 是新页表项的内容，期望是一个物理地址
	if (unlikely(!fault->slot))
		new_spte = make_mmio_spte(vcpu, iter->gfn, ACC_ALL);
	else //创建叶子页表页的 SPTE 的内容
		wrprot = make_spte(vcpu, sp, fault->slot, ACC_ALL, iter->gfn,
					 fault->pfn, iter->old_spte, fault->prefetch, true,
					 fault->map_writable, &new_spte);
	//将新页表项的内容 new_spte 填充到 iter->sptep 指向的页表项
	if (new_spte == iter->old_spte)
		ret = RET_PF_SPURIOUS;
	else if (tdp_mmu_set_spte_atomic(vcpu->kvm, iter, new_spte))
		return RET_PF_RETRY;
	else if (is_shadow_present_pte(iter->old_spte) &&
		 !is_last_spte(iter->old_spte, iter->level))
		kvm_flush_remote_tlbs_with_address(vcpu->kvm, sp->gfn,
						   KVM_PAGES_PER_HPAGE(iter->level + 1));

	/*
	 * If the page fault was caused by a write but the page is write
	 * protected, emulation is needed. If the emulation was skipped,
	 * the vCPU would have the same fault again.
	 */
	if (wrprot) {
		if (fault->write)
			ret = RET_PF_EMULATE;
	}

	/* If a MMIO SPTE is installed, the MMIO will need to be emulated. */
	if (unlikely(is_mmio_spte(new_spte))) {
		vcpu->stat.pf_mmio_spte_created++;
		trace_mark_mmio_spte(rcu_dereference(iter->sptep), iter->gfn,
				     new_spte);
		ret = RET_PF_EMULATE;
	} else {
		trace_kvm_mmu_set_spte(iter->level, iter->gfn,
				       rcu_dereference(iter->sptep));
	}

	return ret;
}

/*
 * tdp_mmu_link_sp - Replace the given spte with an spte pointing to the
 * provided page table.
 *
 * @kvm: kvm instance
 * @iter: a tdp_iter instance currently on the SPTE that should be set
 * @sp: The new TDP page table to install.
 * @shared: This operation is running under the MMU lock in read mode.
 *
 * Returns: 0 if the new page table was installed. Non-0 if the page table
 *          could not be installed (e.g. the atomic compare-exchange failed).
 */
static int tdp_mmu_link_sp(struct kvm *kvm, struct tdp_iter *iter,
			   struct kvm_mmu_page *sp, bool shared)
{	//创建一个非叶子页表页的 SPTE 内容，由子页表页的虚拟地址转换为物理地址作为主要内容
	u64 spte = make_nonleaf_spte(sp->spt, !kvm_ad_enabled());
	int ret = 0;
	//设置父页表页（从 iter->sptep 得到）的页表项内容为物理地址 spte
	if (shared) {
		ret = tdp_mmu_set_spte_atomic(kvm, iter, spte);
		if (ret)
			return ret;
	} else {
		tdp_mmu_set_spte(kvm, iter, spte);
	}

	tdp_account_mmu_page(kvm, sp);

	return 0;
}
...
/*
 * Handle a TDP page fault (NPT/EPT violation/misconfiguration) by installing
 * page tables and SPTEs to translate the faulting guest physical address.
 */
int kvm_tdp_mmu_map(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault)
{
	struct kvm_mmu *mmu = vcpu->arch.mmu;
	struct kvm *kvm = vcpu->kvm;
	struct tdp_iter iter;
	struct kvm_mmu_page *sp;
	int ret = RET_PF_RETRY;

	kvm_mmu_hugepage_adjust(vcpu, fault);

	trace_kvm_mmu_spte_requested(fault);

	rcu_read_lock();
	//以先序遍历的方式逐级建立映射
	tdp_mmu_for_each_pte(iter, mmu, fault->gfn, fault->gfn + 1) {
		int r;
		//如果使能了不可执行巨页的 workaround（见后面简介）需要对巨页的缺页处理进行拆分，用小的页代替巨页
		if (fault->nx_huge_page_workaround_enabled)
			disallowed_hugepage_adjust(fault, iter.old_spte, iter.level);
		//如果 SPTE 已经被另一个线程冻结，直接放弃重试，避免不必要的页表分配和释放
		/*
		 * If SPTE has been frozen by another thread, just give up and
		 * retry, avoiding unnecessary page table allocation and free.
		 */
		if (is_removed_spte(iter.old_spte))
			goto retry;
		//到了目标层级，跳出循环；否则继续建立目标层级以上逐级的页表映射
		if (iter.level == fault->goal_level)
			goto map_target_level;
		//如果下一级映射存在且不是大页，步进到下一级
		/* Step down into the lower level page table if it exists. */
		if (is_shadow_present_pte(iter.old_spte) &&
		    !is_large_pte(iter.old_spte))
			continue;
		//SPTE 映射不存在则分配新 sp，或者 SPTE 指向一个巨页则分割巨页
		/*
		 * The SPTE is either non-present or points to a huge page that
		 * needs to be split.
		 */
		sp = tdp_mmu_alloc_sp(vcpu); //分配准备用于子页表页的 kvm_mmu_page 实例及其要管理的页表页，用 spt 域（HVA）指向它
		tdp_mmu_init_child_sp(sp, &iter); //将子页表页链接到父页表页，设置子页表页 struct page->private 回指 sp
		//传递是否“不允许创建巨页”的属性给新 sp
		sp->nx_huge_page_disallowed = fault->huge_page_disallowed;
		//SPTE 映射指向一个巨页，需要分割巨页
		if (is_shadow_present_pte(iter.old_spte))
			r = tdp_mmu_split_huge_page(kvm, &iter, sp, true);//子页表页的每个条目指向分割好的数据页
		else //对于子页表页不存在的情况，将父页表页（从 iter->sptep 得到）的 SPTE 内容设为新分配的子页表页（由 sp->spt 转换得到）的物理地址
			r = tdp_mmu_link_sp(kvm, &iter, sp, true);
		//如果安装上层 SPTE 失败，则强制 guest 重试，例如，因为不同的任务修改了 SPTE
		/*
		 * Force the guest to retry if installing an upper level SPTE
		 * failed, e.g. because a different task modified the SPTE.
		 */
		if (r) {
			tdp_mmu_free_sp(sp);
			goto retry;
		}
		//如果缺页不允许创建巨页，添加 sp 到 possible_nx_huge_page_link 链表，将来 zap 该页的时候有可能恢复成不可执行的巨页
		if (fault->huge_page_disallowed &&
		    fault->req_level >= iter.level) {
			spin_lock(&kvm->arch.tdp_mmu_pages_lock);
			if (sp->nx_huge_page_disallowed)
				track_possible_nx_huge_page(kvm, sp);
			spin_unlock(&kvm->arch.tdp_mmu_pages_lock);
		}
	}

	/*
	 * The walk aborted before reaching the target level, e.g. because the
	 * iterator detected an upper level SPTE was frozen during traversal.
	 */
	WARN_ON_ONCE(iter.level == fault->goal_level);
	goto retry;
	//前面的准备工作已经完成，数据页 fault->pfn 在前面 kvm_faultin_pfn() 已经分配了，此处需将它填充最后一级页表页的 SPTE 完成映射
map_target_level:
	ret = tdp_mmu_map_handle_target_level(vcpu, fault, &iter);

retry:
	rcu_read_unlock();
	return ret;
}
```
* `tdp_mmu_set_spte_atomic()` 调用了 `__handle_changed_spte()`，如果修改过程中需要从分页结构中删除旧的子树，即满足调用 `handle_removed_pt()` 的情况，则递归处理子页表
```cpp
tdp_mmu_set_spte_atomic()
-> __handle_changed_spte()
   -> handle_removed_pt(kvm, spte_to_child_pt(old_spte, level), shared)
         for (i = 0; i < SPTE_ENT_PER_PAGE; i++) {
            -> kvm_tdp_mmu_write_spte_atomic(sptep, REMOVED_SPTE)
            // or
            -> kvm_tdp_mmu_write_spte(sptep, old_spte, REMOVED_SPTE, level)
         }
      -> handle_changed_spte(kvm, kvm_mmu_page_as_id(sp), gfn, old_spte, REMOVED_SPTE, level, shared)
         -> __handle_changed_spte()
```
* `__handle_changed_spte()` 的 `shared` 参数表明此操作可能不会在独占使用 MMU 锁的情况下运行，并且该操作必须与可能正在修改 SPTE 的其他线程同步
* 不再有 `kvm->arch.tdp_mmu_pages` 链表了
```c
commit d25ceb9264364dca0683748b301340097fdab6c7
Author: Sean Christopherson <seanjc@google.com>
Date:   Wed Oct 19 16:56:15 2022 +0000

    KVM: x86/mmu: Track the number of TDP MMU pages, but not the actual pages
```

#### 页的回收
* 在拆除 VM 时，zap root 会在通过走到 `handle_removed_pt()`，该函数会递归删除和释放子页表页
```c
...
   vm-6434    [059] ....   234.587042: kvm_tdp_mmu_spte_changed: as id 0 gfn ffc00 level 2 old_spte 800000010bc97907 new_spte 80000000000005a0
   vm-6434    [059] ....   234.587043: kvm_mmu_prepare_zap_page: sp gen 0 gfn ffc00 l1 8-byte q0 direct wux nxe ad root 0 sync
   vm-6434    [059] ....   234.587044: <stack trace>
 => trace_event_raw_event_kvm_mmu_page_class
 => __handle_changed_spte
 => __handle_changed_spte
 => __tdp_mmu_set_spte
 => __tdp_mmu_zap_root
 => tdp_mmu_zap_root
 => kvm_tdp_mmu_zap_all
 => kvm_mmu_zap_all
 => kvm_mmu_notifier_release
 => __mmu_notifier_release
 => exit_mmap
 => mmput
 => vhost_detach_mm.isra.32
 => vhost_vsock_dev_release
 => __fput
 => task_work_run
 => do_exit
 => do_group_exit
 => get_signal
 => arch_do_signal
 => exit_to_user_mode_prepare
 => syscall_exit_to_user_mode
 => entry_SYSCALL_64_after_hwframe
   vm-6434    [059] ....   234.587044: kvm_tdp_mmu_spte_changed: as id 0 gfn ffc00 level 1 old_spte 86000001848dbb77 new_spte 80000000000005a0
   vm-6434    [059] ....   234.587048: kvm_tdp_mmu_spte_changed: as id 0 gfn ffc01 level 1 old_spte 86000001848dcb77 new_spte 80000000000005a0
...
```
* 然而数据页并不在此时释放，需要等到销毁虚拟 memory slot（用户态 `unmmap()`）的时候才最终返回给 buddy system

#### `kvm.nx_huge_pages` 和 `KVM_CAP_VM_DISABLE_NX_HUGE_PAGES`
* 为了缓解 iTLB Multihit CPU 漏洞（Documentation/admin-guide/hw-vuln/multihit.rst），KVM 可以用 module 参数 `nx_huge_pages=force|auto` 来强制软件不允许使用可执行的巨页
  * 如果缓解已启用，在这种情况下，缓解措施在 Linux 内核 KVM 模块中实现了不可执行的大页。EPT 中的所有大页都被标记为 **不可执行**
  * 如果 guest 试图在其中一个页面中执行，该页面将被分解为 4K 页面，然后将其标记为 **可执行**
* 巨页在非虚拟化场景下一般都用作存储，所以这个问题比较少。但在虚拟化场景下，整个 VM 都在巨页上，包括可执行代码段，这就很可能受该漏洞的困扰
* 可以用 per-VM 的 `ioctl(..., KVM_CAP_VM_DISABLE_NX_HUGE_PAGES)` 来禁用以上缓解措施以获得性能上的提升，前提是该 VM 的 workload 是受信任的
```c
commit 084cc29f8bbb034cf30a7ee07a816c115e0c28df
Author: Ben Gardon <bgardon@google.com>
Date:   Mon Jun 13 21:25:21 2022 +0000

    KVM: x86/MMU: Allow NX huge pages to be disabled on a per-vm basis
```

### KVM MMU Memory Cache
#### `struct kvm_mmu_memory_cache` MMU 内存 cache 结构
```cpp
#ifdef KVM_ARCH_NR_OBJS_PER_MEMORY_CACHE
/*
 * Memory caches are used to preallocate memory ahead of various MMU flows,
 * e.g. page fault handlers.  Gracefully handling allocation failures deep in
 * MMU flows is problematic, as is triggering reclaim, I/O, etc... while
 * holding MMU locks.  Note, these caches act more like prefetch buffers than
 * classical caches, i.e. objects are not returned to the cache on being freed.
 *
 * The @capacity field and @objects array are lazily initialized when the cache
 * is topped up (__kvm_mmu_topup_memory_cache()).
 */
struct kvm_mmu_memory_cache {
    int nobjs;
    gfp_t gfp_zero;
    gfp_t gfp_custom;
    struct kmem_cache *kmem_cache;
    int capacity;
    void **objects;
};
#endif
```
* `objects`：指向用 `kvmalloc_array()` 分配的数组，元素是 `capacity` 个 `void *` 类型的指针
* `capacity`：cache 的容量
* `nobjs`：cache 中的空闲的 object 数目
* `kmem_cache`：见 `mmu_memory_cache_alloc_obj()`
  * 不为空时，数组元素从 `kmem_cache` 指向的 kmem cache 里分配的 object
  * 否则用 `__get_free_page()` 从 buddy system 分配，此时每个指针用来指向一个 page 大小的页面
#### 目前有以下 KVM MMU Memory Cache
```cpp
 struct kvm_mmu_memory_cache mmu_pte_list_desc_cache;
 struct kvm_mmu_memory_cache mmu_shadow_page_cache;
 struct kvm_mmu_memory_cache mmu_shadowed_info_cache;
 struct kvm_mmu_memory_cache mmu_page_header_cache;
```
* `mmu_page_header_cache`：页表页管理结构 `struct kvm_mmu_page` 的 cache
* `mmu_shadow_page_cache`：页表页页面 `struct kvm_mmu_page.spt` 的 cache
* `mmu_pte_list_desc_cache`：rmap 的 `struct pte_list_desc` 的 cache
* `mmu_shadowed_info_cache`：非直接映射时 shadow page `struct kvm_mmu_page.shadowed_translation` 的 cache

#### KVM MMU Memory Cache 相关接口
* `kvm_mmu_topup_memory_cache()`：填充 cache 中的 object 到设定的容量
* `kvm_mmu_memory_cache_alloc()`：从 cache 中返回一个 object，`nobjs` 减 `1`
* `kvm_mmu_free_memory_cache()`：释放 cache 里的所有 objects，然后释放 cache
* `kvm_mmu_memory_cache_nr_free_objects()`：返回 cache 中的空闲 object 数目 `objs`
# References
- [The x86 kvm shadow mmu — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/virt/kvm/x86/mmu.html)
- [KVM的ept机制](https://www.cnblogs.com/scu-cjx/p/6878568.html)
- [KVM MMU EPT内存管理](https://blog.csdn.net/xelatex_kvm/article/details/17685123)
- [[PATCH v2 00/20] Introduce the TDP MMU](https://lore.kernel.org/all/20201014182700.2888246-1-bgardon@google.com/)