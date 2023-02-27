# KVM Page Fault

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
* `link`：`link`将该结构链接到`kvm->arch.active_mmu_pages`和`invalid_list`上，标注该页结构不同的状态
* `hash_link`：`hash_link`将该结构链接到`kvm->arch.mmu_page_hash`哈希表上，以便进行快速查找，hash key由接下来的`gfn`和`role`决定
* `gfn`：含义与 `.role.direct` 相关：
   * `.role.direct = 1`：在直接映射时，保存的是线性映射的基页帧；来自于 `tdp_mmu_init_sp(child_sp, iter->sptep, iter->gfn, role)` 中的 `iter->gfn`
   * `.role.direct = 0`：在非直接映射时，保存的是 guest page table，该 PT 包含了由该页映射的 translation
* `role`：该页的“角色”，详细参见上文对`union kvm_mmu_page_role`的说明
* `spt`：对应的 SPT/EPT 页帧的地址（HVA），SPT/EPT 页的`struct page`结构中`page->private`域会反向指向该`struct kvm_mmu_page`
  * 该域可以指向一个 lower-level shadow pages，也可以指向真正的数据 page
  * 见 `tdp_mmu_init_sp(struct kvm_mmu_page *sp, ...) -> set_page_private(virt_to_page(sp->spt), (unsigned long)sp)`
* `parent_ptes`：指向上一级`spt`，表示有哪些上一级页表页的页表项指向该页表页
* `unsync`：该域只对页结构的叶子节点有效，可以执行该页的翻译是否与 guest 的翻译一致
  * 如果为`false`，则可能出现修改了该页中的 pte 但没有更新 TLB，而 guest 读取了 TLB 中的数据，导致了不一致
* `root_count`：该页被多少个 vcpu 作为根页结构
* `unsync_children`：记录该页结构下面有多少个子节点是 unsync 状态的
* `mmu_valid_gen`：该页的 generation number
   * KVM维护了一个全局的的 gen number（`kvm->arch.mmu_valid_gen`），如果该域与全局的 gen number 不相等，则将该页标记为 invalid page
   * 该结构用来快速的碾压掉 KVM 的 MMU paging structure
     * 例如，如果想废弃掉当前所有的MMU页结构，需要处理掉所有的MMU页面和对应的映射；
     * 但是通过该结构，可以直接将`kvm->arch.mmu_valid_gen`加`1`，那么当前所有的 MMU 页结构都变成了 invalid，而处理掉页结构的过程可以留给后面的过程（如内存不够时）再处理，可以加快废弃所有 MMU 页结构的速度。
     * 当`mmu_valid_gen`值达到最大时，可以调用`kvm_mmu_invalidate_zap_all_pages`手动废弃掉所有的 MMU 页结构
* `unsync_child_bitmap`：记录了 unsync 的子结构的位图
* `clear_spte_count`：仅针对`32`位 host 有效，具体作用参考函数`__get_spte_lockless`的注释
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
* `tdp_iter_next()` 在分页结构中以先序遍历进入下一个 SPTE。为了到达下一个 SPTE，迭代器
  * 要么向下转向目标 GFN（如果当前是非最后一级 SPTE）
  * 要么转到映射更高 GFN 的 SPTE
* `struct tdp_iter` 是迭代器的核心数据结构
  * `tdp_ptep_t sptep`：指向当前遍历到的 SPTE 的指针，当然是个虚拟地址
  * `u64 old_spte`：`sptep` 指向的条目里的内容的快照，页表条目里的内容当然是物理地址了
  * `gfn_t gfn`：当前 SPTE 所在页表页的最低 GFN，会通过 `tdp_mmu_init_sp()` 赋值给 `struct kvm_mmu_page` 的 `gfn` 域
  * `tdp_ptep_t pt_path[PT64_ROOT_MAX_LEVEL]`：存储遍历到当前 SPTE 所经过的页表页的虚拟地址

#### GFN 对齐函数 `round_gfn_for_level()`
* `round_gfn_for_level()` 用于取对齐到不同层级的 PFN，分段为 9-9-9-9
```cpp
static gfn_t round_gfn_for_level(gfn_t gfn, int level)
{
    return gfn & -KVM_PAGES_PER_HPAGE(level);
}
```
* 对于不同的 `level`，`-KVM_PAGES_PER_HPAGE(level)` 的值分别是：
  * `4`：`0xfffffffff8000000`
  * `3`：`0xfffffffffffc0000`
  * `2`：`0xfffffffffffffe00`
  * `1`：`0xffffffffffffffff`
* 最后一级相当于不做对齐，这就是 `try_step_down()` 在最后一级时能让 `iter->gfn` 为 `fault->gfn` 的原因
  * 见 `iter->gfn = round_gfn_for_level(iter->next_last_level_gfn, iter->level)`
  * 故而随后的 `tdp_iter_refresh_sptep(iter)` 能让 `iter->sptep` 指向 `fault->gfn` 对应的 SPTE

#### SPTE 转页表函数 `spte_to_child_pt()`
* 该函数入参为 SPTE 的内容（PA）和该 SPTE 的层级，返回该条目所指向的子页表页的虚拟地址
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

	if (WARN_ON_ONCE(sp->role.level != fault->goal_level))
		return RET_PF_RETRY;
	//new_spte 是新页表项的内容，期望是一个物理地址
	if (unlikely(!fault->slot))
		new_spte = make_mmio_spte(vcpu, iter->gfn, ACC_ALL);
	else
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
{   //子页表页的虚拟地址转换为物理地址作为 SPTE 的内容
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
    struct tdp_iter iter;
    struct kvm_mmu_page *sp;
    int ret;

    kvm_mmu_hugepage_adjust(vcpu, fault);

    trace_kvm_mmu_spte_requested(fault);

    rcu_read_lock();
    //以先序遍历的方式逐级建立映射
    tdp_mmu_for_each_pte(iter, mmu, fault->gfn, fault->gfn + 1) {
        if (fault->nx_huge_page_workaround_enabled)
            disallowed_hugepage_adjust(fault, iter.old_spte, iter.level);
        //到了目标层级，跳出循环；否则建立目标层级以上逐级的页表映射
        if (iter.level == fault->goal_level)
            break;
        //如果有一个 SPTE 在比目标更高的层级上映射大页，则必须清除该 SPTE 并用非叶子 SPTE 替换
        /*
         * If there is an SPTE mapping a large page at a higher level
         * than the target, that SPTE must be cleared and replaced
         * with a non-leaf SPTE.
         */
        if (is_shadow_present_pte(iter.old_spte) &&
            is_large_pte(iter.old_spte)) {
            if (tdp_mmu_zap_spte_atomic(vcpu->kvm, &iter))
                break;
            //映射已经清除，更新 SPTE 内容快照，重要的是不再 present
            /*
             * The iter must explicitly re-read the spte here
             * because the new value informs the !present
             * path below.
             */
            iter.old_spte = kvm_tdp_mmu_read_spte(iter.sptep);
        }
        //SPTE 映射不存在，包括上面清除的情况；不包括 SPTE 已经映射了页表页的情况
        if (!is_shadow_present_pte(iter.old_spte)) {
            bool account_nx = fault->huge_page_disallowed &&
                      fault->req_level >= iter.level;
            //如果 SPTE 已经被另一个线程冻结，直接放弃重试，避免不必要的页表分配和释放
            /*
             * If SPTE has been frozen by another thread, just
             * give up and retry, avoiding unnecessary page table
             * allocation and free.
             */
            if (is_removed_spte(iter.old_spte))
                break;
            //分配准备用于子页表页的 kvm_mmu_page 实例及其要管理的页表页，用 spt 域（HVA）指向它
            sp = tdp_mmu_alloc_sp(vcpu);
            tdp_mmu_init_child_sp(sp, &iter); //初始化子页表页 kvm_mmu_page 其他域
            //将子页表页链接到父页表页，即父页表页（从 iter->sptep 得到）的 SPTE 内容设为子页表页
            //（由 sp->spt 转换得到）的物理地址，并将子页表页的 kvm_mmu_page 实例加入 kvm->arch.tdp_mmu_pages 链表
            if (tdp_mmu_link_sp(vcpu->kvm, &iter, sp, account_nx, true)) {
                tdp_mmu_free_sp(sp);
                break;
            }
        }
    }
    //如果上一层 SPTE 未映射，或者如果目标叶子 SPTE 被另一个 CPU 冻结，则强制 guest 重试访问
    /*
     * Force the guest to retry the access if the upper level SPTEs aren't
     * in place, or if the target leaf SPTE is frozen by another CPU.
     */
    if (iter.level != fault->goal_level || is_removed_spte(iter.old_spte)) {
        rcu_read_unlock();
        return RET_PF_RETRY;
    }
    //前面的准备工作已经完成，数据页 fault->pfn 在前面 kvm_faultin_pfn() 已经分配了，此处需将它填充最后一级页表页的 SPTE 完成映射
    ret = tdp_mmu_map_handle_target_level(vcpu, fault, &iter);
    rcu_read_unlock();

    return ret;
}
```
* `tdp_mmu_set_spte_atomic()` 调用的 `__handle_changed_spte()` 如果修改过程中需要从分页结构中删除了子树，则递归处理子页表
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

### KVM MMU Memory Cache
```cpp
 struct kvm_mmu_memory_cache mmu_pte_list_desc_cache;
 struct kvm_mmu_memory_cache mmu_shadow_page_cache;
 struct kvm_mmu_memory_cache mmu_shadowed_info_cache;
 struct kvm_mmu_memory_cache mmu_page_header_cache;
```
* `mmu_page_header_cache`：页表页管理结构 `struct kvm_mmu_page` 的 cache
* `mmu_shadow_page_cache`：页表页页面 `struct kvm_mmu_page.spt` 的 cache
* `mmu_pte_list_desc_cache`：rmap 的 `struct pte_list_desc` 的 cache
* `mmu_shadowed_info_cache`：非直接映射时 shadow page `struct kvm_mmu_page.shadowed_translation` 的 cache，

# References
- [The x86 kvm shadow mmu — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/virt/kvm/x86/mmu.html)
- [KVM的ept机制](https://www.cnblogs.com/scu-cjx/p/6878568.html)
- [KVM MMU EPT内存管理](https://blog.csdn.net/xelatex_kvm/article/details/17685123)
- [[PATCH v2 00/20] Introduce the TDP MMU](https://lore.kernel.org/all/20201014182700.2888246-1-bgardon@google.com/)