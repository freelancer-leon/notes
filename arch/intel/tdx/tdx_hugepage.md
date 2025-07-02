# TDX Hugepage

## TDX TLB Shootdown 的优化
* 先看懂 TDX TLB shootdown 的一个优化

### KVM: x86/mmu: add SPTE_PRIVATE_ZAPPED

```cpp
commit 07d93390ca5bf56b7e4b768105fe75772758aedc
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Thu Jan 20 13:52:42 2022 -0800

    KVM: x86/mmu: add SPTE_PRIVATE_ZAPPED

    This is preparation to optimize TLB shootdown.  Steal one bit for batched
    tlb shootdown from MMIO counter bit.

    The existing code to zap the EPT entry always issues the TLB shootdown each
    the EPT entry, doesn't batch TLB shootdown for zapping multiple EPT
    entries.  The procedure is

    1) clear the EPT entry
    2) TDX SEAMCALL TDH.MEM.RANGE.BLOCK with GFN
       This corresponds to clearing the present bit.
    3) TDH.MEM.TRACK. corresponds to local tlb flush
    4) send IPI to remote vcpu. This corresponds to remote tlb flush.
    5) When destructing VM, TDH.MEM.REMOVE with PFN.
       There is no corresponding to the VMX EPT operation.

    At the last step, PFN is needed to unlink the private memory from the
    Secure EPT.  Because this procedure is doing synchronously, the PFN is
    saved on the stack.

    If we'd like to batched TLB shootdown, the PFNs needs to be saved somewhere
    because the stack can't be used as the array of PFNs can be large.

    1) multiple the EPT entries
    2) TDX SEAMCALL TDH.MEM.RANGE.BLOCK with GFNs
       This corresponds to clearing the present bit.
    The step 1) and 2) are repeated for multiple GFNs. and then

    3) TDH.MEM.TRACK. corresponds to local tlb flush
    4) send IPI to remote vcpu. This corresponds to remote tlb flush.
    3) and 4) is a batched TLB shootdown.

    5) When destructing VM, TDH.MEM.REMOVE with PFNs.
       There is no corresponding to the VMX EPT operation.

    For the step 5), PFNs needs to be remembered somewhere.  One option is to
    use the zapped EPT entry.  by setting the special flag.
    SPTE_PRIVATE_ZAPPED.  This will complicates the EPT violation path.

    The alternative for TDP MMU is to reuse rmap.  The KVM memory slot
    allocates array for rmap, 1 entry for 1 GFN.  So this array can be used.
    But this approach doesn't work for legacy MMU because rmap is used.
```
* 这是为优化 TLB shootdown 的准备。从 MMIO 计数器 bit 中偷取一个 bit 位为批量 TLB shootdown。
* 已存在的去除 EPT entry 的代码总是为每个 EPT entry 发出 TLB shootdown，而不是为去除多个 EPT entries 发出批量 TLB shootdown。
1. 清除 EPT entry
2. 以 `GFN` 发出 TDX SEAMCALL `TDH.MEM.RANGE.BLOCK`。对应地清除 `present` bit
3. `TDH.MEM.TRACK`，对应到本地 TLB flush
4. 发送 IPI 给远端的 vCPU，对应远端的 TLB flush
5. 当销毁 VM，以 `PFN` 发出 `TDH.MEM.REMOVE`。无对应的 VMX EPT 操作
* 最后一步，需要 `PFN` 来取消私有内存与 Secure EPT 的链接。由于该过程是同步执行的，因此 `PFN` 保存在栈中。

* 如果我们想批量 TLB shootdown，`PFN` 需要保持在某处。因为 PFN 的数组可以很巨大，所以不能再用栈来保存。
1. 多条 EPT entries
2. 以 `GFN` 发出 TDX SEAMCALL `TDH.MEM.RANGE.BLOCK`。对应地清除 `present` bit
* 对多条 GFNs 重复步骤 1 和 2。然后，

3. `TDH.MEM.TRACK`，对应到本地 TLB flush
4. 发送 IPI 给远端的 vCPU，对应远端的 TLB flush
* 步骤 3 和 4 是批量的 TLB shootdown

5. 当销毁 VM，以 `PFN` 发出 `TDH.MEM.REMOVE`。无对应的 VMX EPT 操作
* 对于步骤 5，PFNs 需要被记录在某处。一个选择是使用被去掉的 EPT entry，通过设置特别标志位 `SPTE_PRIVATE_ZAPPED`。这会使 EPT violation 的路径变得复杂
* 对于 TDP MMU 变通的方案是用 rmap。KVM memory slot 分配给 rmap 的数组，1 个 entry 对应 1 个 `GFN`
  * 所以可以用数组，但这个方法对于 legacy MMU 不工作，因为 rmap 被使用

```diff
diff --git a/arch/x86/kvm/mmu/spte.h b/arch/x86/kvm/mmu/spte.h
index 067ea1ae3a13..2b396a2fe7d4 100644
--- a/arch/x86/kvm/mmu/spte.h
+++ b/arch/x86/kvm/mmu/spte.h
@@ -14,6 +14,9 @@
  */
 #define SPTE_MMU_PRESENT_MASK          BIT_ULL(11)

+/* Masks that used to track metadata for not-present SPTEs. */
+#define SPTE_PRIVATE_ZAPPED            BIT_ULL(62)
+
 /*
  * TDP SPTES (more specifically, EPT SPTEs) may not have A/D bits, and may also
  * be restricted to using write-protection (for L2 when CPU dirty logging, i.e.
@@ -96,11 +99,11 @@ static_assert(!(EPT_SPTE_MMU_WRITABLE & SHADOW_ACC_TRACK_SAVED_MASK));
 #undef SHADOW_ACC_TRACK_SAVED_MASK

 /*
- * Due to limited space in PTEs, the MMIO generation is a 19 bit subset of
+ * Due to limited space in PTEs, the MMIO generation is a 18 bit subset of
  * the memslots generation and is derived as follows:
  *
  * Bits 0-7 of the MMIO generation are propagated to spte bits 3-10
- * Bits 8-18 of the MMIO generation are propagated to spte bits 52-62
+ * Bits 8-17 of the MMIO generation are propagated to spte bits 52-61
  *
  * The KVM_MEMSLOT_GEN_UPDATE_IN_PROGRESS flag is intentionally not included in
  * the MMIO generation number, as doing so would require stealing a bit from
@@ -114,7 +117,7 @@ static_assert(!(EPT_SPTE_MMU_WRITABLE & SHADOW_ACC_TRACK_SAVED_MASK));
 #define MMIO_SPTE_GEN_LOW_END          10

 #define MMIO_SPTE_GEN_HIGH_START       52
-#define MMIO_SPTE_GEN_HIGH_END         62
+#define MMIO_SPTE_GEN_HIGH_END         61

 #define MMIO_SPTE_GEN_LOW_MASK         GENMASK_ULL(MMIO_SPTE_GEN_LOW_END, \
                                                    MMIO_SPTE_GEN_LOW_START)
@@ -141,7 +144,7 @@ static_assert(!(SPTE_MMIO_ALLOWED_MASK &
 #define MMIO_SPTE_GEN_HIGH_BITS                (MMIO_SPTE_GEN_HIGH_END - MMIO_SPTE_GEN_HIGH_START + 1)

 /* remember to adjust the comment above as well if you change these */
-static_assert(MMIO_SPTE_GEN_LOW_BITS == 8 && MMIO_SPTE_GEN_HIGH_BITS == 11);
+static_assert(MMIO_SPTE_GEN_LOW_BITS == 8 && MMIO_SPTE_GEN_HIGH_BITS == 10);

 #define MMIO_SPTE_GEN_LOW_SHIFT                (MMIO_SPTE_GEN_LOW_START - 0)
 #define MMIO_SPTE_GEN_HIGH_SHIFT       (MMIO_SPTE_GEN_HIGH_START - MMIO_SPTE_GEN_LOW_BITS)
@@ -210,8 +213,12 @@ extern u64 __read_mostly shadow_nonpresent_or_rsvd_mask;
  */
 #define REMOVED_SPTE   (SHADOW_NONPRESENT_VALUE | 0x5a0ULL)

-/* Removed SPTEs must not be misconstrued as shadow present PTEs. */
+/*
+ * Removed SPTEs must not be misconstrued as shadow present PTEs, and
+ * temporarily blocked private PTEs.
+ */
 static_assert(!(REMOVED_SPTE & SPTE_MMU_PRESENT_MASK));
+static_assert(!(REMOVED_SPTE & SPTE_PRIVATE_ZAPPED));

 static inline bool is_removed_spte(u64 spte)
 {
@@ -318,6 +325,11 @@ static inline bool is_access_track_spte(u64 spte)
        return !spte_ad_enabled(spte) && (spte & shadow_acc_track_mask) == 0;
 }

+static inline bool is_private_zapped_spte(u64 spte)
+{
+       return !!(spte & SPTE_PRIVATE_ZAPPED);
+}
+
 static inline bool is_large_pte(u64 pte)
 {
        return pte & PT_PAGE_SIZE_MASK;
```
* patch 引入用于 SPTE 的 `SPTE_PRIVATE_ZAPPED`（`62`）位，用于标识临时被 blocked 的私有页面
  * 这个标志位和已映射标志位 `shadow present` 一样，必须不能与 `REMOVED_SPTE` 所用的标志位重合，为此增加了一个编译时断言
  * 该标志位偷取自 MMIO 的 generation 计数器，相应地做一些修改

### KVM: x86/tdp_mmu: optimize remote tlb flush
```cpp
commit bcb145afee35eecd071e1974a0b2725af3680b8a
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Wed Jan 19 04:32:38 2022 -0800

    KVM: x86/tdp_mmu: optimize remote tlb flush

    Implement batched TLB shootdown for optimization.  The current
    implementation to zap multiple the EPT entries, looks like as follows.

    Loop on GFNs
      1) Zap the EPT entry.  Zero the EPT entry.  PFN is saved on stack.
      2) TDX SEAMCALL TDH.MEM.RANGE.BLOCK with GFN. This corresponds to
         clearing the present bit.
      3) TDH.MEM.TRACK. This corresponds to local tlb flush.
      4) Send IPI to remote vcpus. This corresponds to remote tlb flush.
      5) When destructing VM, TDH.MEM.REMOVE with PFN. There is no
         corresponding to the VMX EPT operation.
      At the last of zapping
      6) kvm_flush_remote_tlbs_with_address(). This flushes shared EPT pointer.
         No operations on Secure EPT.

    The new implementation looks like as follows.  The number of TLB shootdown
    is reduced from the number of the EPT entries to zap to one.

    Loop on GFNs
      1) - Zap the EPT entry.
         - Clear present/dirty/access bits
         - Keep PFN
         - Set SPTE_PRIVATE_ZAPPED to indicate valid PFN to unlink from Secure
           EPT.
      2) TDX SEAMCALL TDH.MEM.RANGE.BLOCK with GFN.  This corresponds to
         clearing the present bit.
      KVM remote TLB flush callback does,
      3) TDH.MEM.TRACK. corresponds to local tlb flush
      4) Send IPI to remote vcpus. This corresponds to remote tlb flush.
      5) kvm_flush_remote_tlbs_with_address().  This flushes shared EPT
         pointer. No operations on Secure EPT.  When destructing VM, Check if
         SPTE_PRIVATE_ZAPPED and issue TDH.MEM.REMOVE with PFNs.

    For traversing private zapped EPT entries, introduce helper function in
    TDP MMU code.
```

#### 新增函数 `private_zapped_spte()`
* 按照设计，临时被 blocked 的私有页面的 SPTE 条目填入如下值：
  * 设置 `SHADOW_NONPRESENT_VALUE` 表示无映射
  * 设置专有的 `SPTE_PRIVATE_ZAPPED`（`62`bit）
  * `PFN` 部分与原来的一样
  * 如果是大页，需设置表示 SPTE 是映射大页的 `PT_PAGE_SIZE_MASK` 标志
```cpp
static u64 __private_zapped_spte(u64 old_spte)
{   //按照设计，临时被 blocked 的私有页面需设置以下 bits
    return SHADOW_NONPRESENT_VALUE | SPTE_PRIVATE_ZAPPED |
        (spte_to_pfn(old_spte) << PAGE_SHIFT) |            //PFN 部分与原来的一样
        (is_large_pte(old_spte) ? PT_PAGE_SIZE_MASK : 0);
}

static u64 private_zapped_spte(struct kvm *kvm, const struct tdp_iter *iter)
{
    if (!kvm_gfn_shared_mask(kvm))     //对于不支持 GFN shared bit 的 arch 而言，
        return SHADOW_NONPRESENT_VALUE;//无所谓临时被 blocked 的私有页面

    if (!is_private_sptep(iter->sptep))//对于共享的 SPTE 而言，
        return SHADOW_NONPRESENT_VALUE;//无所谓临时被 blocked 的私有页面
    //构造临时被 blocked 的私有页面的 PTE 的值
    return __private_zapped_spte(iter->old_spte);
}
```
##### 三处用到 `private_zapped_spte()` 的地方
* 这三处都在批量去除 SPTE 的路径上，也可看出临时被 blocked 的私有 SEPT 的用场
* 这就实现了批量 TLB shootdown 5 步走的第 1 步，设置私有 SPTE 为 *临时被 blocked 的私有页面的值*
1. `tdp_mmu_zap_spte_atomic()` 用在批量去除 SPTE 的路径上，这里用特殊的 *临时被 blocked 的私有页面的值* 取代原有的 `SHADOW_NONPRESENT_VALUE`
```diff
@@ -880,7 +974,7 @@ static inline int __must_check tdp_mmu_zap_spte_atomic(struct kvm *kvm,
         * SHADOW_NONPRESENT_VALUE (which sets "suppress #VE" bit) so it
         * can be set when EPT table entries are zapped.
         */
-       __kvm_tdp_mmu_write_spte(iter->sptep, SHADOW_NONPRESENT_VALUE);
+       __kvm_tdp_mmu_write_spte(iter->sptep, private_zapped_spte(kvm, iter));

        return 0;
 }
```
2. `tdp_mmu_zap_leafs()` 用在范围去除叶子 SPTE 的路径上，
   * 从私有的根页表中 *不去除私有的 SPTE* 不再是无事可做了，因为我们可以在这里将它们变为 *临时被 blocked 的私有页面*，一种半去除的状态
   * 这个函数的目的是去除叶子 SPTE，因此 **已经** 是 *未映射* 且 *非临时被 blocked 的私有页面* 就可以被跳过了，因为这些页面被认为是已去除了
     * `!is_shadow_present_pte(iter.old_spte) && !is_private_zapped_spte(iter.old_spte)`
     * 而 *未映射* 但却 *临时被 blocked 的私有页面* 也需要被去掉（回忆上面 `private_zapped_spte()` 构造的值）
   * 对于 *不去除非私有 SPTE* 时，如遍历到一个 **已经** 是 *临时被 blocked 的私有页面* 时，则跳过<sup>[1]</sup>
     * `!zap_private && is_private_zapped_spte(iter.old_spte)`
   * 将这条 SPTE 去除前还要判断一下：
     * 如果 *不去除非私有 SPTE*，要设置的 SPTE 的值是 *临时被 blocked 的私有页面的值* ，也就是上面 [1] 那个判断要跳过的情况
     * 如果 *要去除私有 SPTE*，则要设置的 SPTE 的值是 `SHADOW_NONPRESENT_VALUE`
```diff
@@ -1158,10 +1258,7 @@ static bool tdp_mmu_zap_leafs(struct kvm *kvm, struct kvm_mmu_page *root,
        end = min(end, tdp_mmu_max_gfn_exclusive());

        lockdep_assert_held_write(&kvm->mmu_lock);
-
        WARN_ON_ONCE(zap_private && !is_private);
-       if (!zap_private && is_private_sp(root))
-               return false;

        /*
         * start and end doesn't have GFN shared bit.  This function zaps
@@ -1180,8 +1277,15 @@ static bool tdp_mmu_zap_leafs(struct kvm *kvm, struct kvm_mmu_page *root,
                        continue;
                }

-               if (!is_shadow_present_pte(iter.old_spte) ||
-                   !is_last_spte(iter.old_spte, iter.level))
+               if (!is_last_spte(iter.old_spte, iter.level))
+                       continue;
+
+               /*
+                * Skip non-present SPTE, with exception of temporarily
+                * blocked private SPTE, which also needs to be zapped.
+                */
+               if (!is_shadow_present_pte(iter.old_spte) &&
+                   !is_private_zapped_spte(iter.old_spte))
                        continue;

                if (is_private && kvm_gfn_shared_mask(kvm) &&
@@ -1226,7 +1330,13 @@ static bool tdp_mmu_zap_leafs(struct kvm *kvm, struct kvm_mmu_page *root,
                        }
                }

-               tdp_mmu_set_spte(kvm, &iter, SHADOW_NONPRESENT_VALUE);
+               if (!zap_private && is_private_zapped_spte(iter.old_spte))
+                       continue;
+
+               tdp_mmu_set_spte(kvm, &iter,
+                                zap_private ?
+                                SHADOW_NONPRESENT_VALUE :
+                                private_zapped_spte(kvm, &iter));
                flush = true;
        }
```
* 但是私有根不去除私有页面的限制条件后来又被加上了，这导致 *不去除非私有 SPTE* 时设置的 SPTE 的值为 *临时被 blocked 的私有页面的值* 的这个条件就进不去了啊？！
```diff
commit 166e5b8c7b982f9405138d3e0cc852ae0cd9a73b
Author: Binbin Wu <binbin.wu@linux.intel.com>
Date:   Sun Jul 16 14:52:19 2023 +0800

    KVM: x86/tdp_mmu: Skip zapping leafs when not needed

    Skip zapping leafs for private root if the caller of tdp_mmu_zap_leafs()
    passes zap_private as false.

    Signed-off-by: Binbin Wu <binbin.wu@linux.intel.com>

diff --git a/arch/x86/kvm/mmu/tdp_mmu.c b/arch/x86/kvm/mmu/tdp_mmu.c
index e5f8fc816f4c..cbef1fc9dce7 100644
--- a/arch/x86/kvm/mmu/tdp_mmu.c
+++ b/arch/x86/kvm/mmu/tdp_mmu.c
@@ -1307,6 +1307,9 @@ static bool tdp_mmu_zap_leafs(struct kvm *kvm, struct kvm_mmu_page *root,
        lockdep_assert_held_write(&kvm->mmu_lock);
        WARN_ON_ONCE(zap_private && !is_private);

+       if (!zap_private && is_private)
+               return flush;
+
        /*
         * start and end doesn't have GFN shared bit.  This function zaps
         * a region including alias.  Adjust shared bit of [start, end) if the
```

3. `set_spte_gfn()` 则出现在 `kvm_mmu_notifier_change_pte() -> kvm_handle_hva_range()` 的回调上，这里也是个优化的点，SPTE 的值设为 *临时被 blocked 的私有页面*
```diff
@@ -1798,7 +1908,7 @@ static bool set_spte_gfn(struct kvm *kvm, struct tdp_iter *iter,
         * invariant that the PFN of a present * leaf SPTE can never change.
         * See __handle_changed_spte().
         */
-       tdp_mmu_set_spte(kvm, iter, SHADOW_NONPRESENT_VALUE);
+       tdp_mmu_set_spte(kvm, iter, private_zapped_spte(kvm, iter));

        if (!pte_write(range->pte)) {
                new_spte = kvm_mmu_changed_pte_notifier_make_spte(iter->old_spte,
```

#### 新增函数 `handle_private_zapped_spte()`
* 引入新函数 `handle_private_zapped_spte()` 处理临时被 blocked 的私有页面，处理的结果是页面从 TD 中被用 SEAMCALL `TDH.MEM.PAGE.REMOVE` 移除
* 有两种特殊的情况需要先处理：
1. 当 guest 访问私有页面时，需要恢复原来的映射（unblock）
2. 由于在缺页处理去除别名时，或者当 VM 被销毁时，需要真正地除去 SPTE
* 需要在处理新旧 SPTE 都是未映射（`!was_present && !is_present`）的情况之前处理以上情况，因为被 blocked 的私有页面也是未映射的 `non-present`
* 因为页面被 pin（参考 `kvm_faultin_pfn_private()`），为私有页面的页面迁移不应该被触发。KVM 私有 memory slot 的情况也应该阻止页面迁移
* **旧 SPTE 是未映射且临时被 blocked 的私有页面`NON-PRESENT + PRIVATE_ZAPPED`，新 SPTE 是未映射`NON-PRESENT`**，这种情况需要 SEAMCALL `TDH.MEM.PAGE.REMOVE` 从 TD 中彻底移除页面
  * 这就实现了批量 TLB shootdown 5 步走的第 5 步，删除临时被 blocked 的私有页面
```cpp
static int __must_check handle_private_zapped_spte(struct kvm *kvm, gfn_t gfn,
                           u64 old_spte, u64 new_spte,
                           int level)
{
    bool was_private_zapped = is_private_zapped_spte(old_spte); //旧 SPTE 是临时被 blocked 的私有页面
    bool is_private_zapped = is_private_zapped_spte(new_spte);  //新 SPTE 是临时被 blocked 的私有页面
    bool was_present = is_shadow_present_pte(old_spte);         //旧 SPTE 是已映射
    bool is_present = is_shadow_present_pte(new_spte);          //新 SPTE 是建立映射
    bool was_last = is_last_spte(old_spte, level);              //旧 SPTE 是叶子 PTE
    bool is_last = is_last_spte(new_spte, level);               //新 SPTE 是叶子 PTE
    kvm_pfn_t old_pfn = spte_to_pfn(old_spte);                  //旧 SPTE 中提取出旧 PFN
    kvm_pfn_t new_pfn = spte_to_pfn(new_spte);                  //新 SPTE 中提取出新 PFN
    int ret = 0;
    //临时被 blocked 的旧私有 SPTE 只能是叶子 PTE
    /* Temporarily blocked private SPTE can only be leaf. */
    KVM_BUG_ON(!is_last_spte(old_spte, level), kvm);
    KVM_BUG_ON(is_private_zapped, kvm);   //新 SPTE 不能也是临时被 blocked 的私有页面
    KVM_BUG_ON(was_present, kvm);         //旧 SPTE 不能还有映射
    KVM_BUG_ON(!was_private_zapped, kvm); //旧 SPTE 不能不是临时被 blocked 的私有页面，因为这正是我们要处理的

    /*
     * Handle special case of old_spte being temporarily blocked private
     * SPTE.  There are two cases: 1) Need to restore the original mapping
     * (unblock) when guest accesses the private page; 2) Need to truly
     * zap the SPTE because of zapping aliasing in fault handler, or when
     * VM is being destroyed.
     *
     * Do this before handling "!was_present && !is_present" case below,
     * because blocked private SPTE is also non-present.
     */
    if (is_present) {//新 SPTE 是建立映射
        /* map_gpa holds write lock. */
        lockdep_assert_held(&kvm->mmu_lock);

        if (old_pfn == new_pfn) { //新旧 SPTE 的 PFN 部分是一样的
            ret = static_call(kvm_x86_unzap_private_spte)(kvm, gfn, level); //unblock SPTE，恢复映射
        } else if (level > PG_LEVEL_4K && was_last && !is_last) { //之前是叶子 SPTE 且是大页，现在不是
            /*
             * Splitting private_zapped large page doesn't happen.
             * Unzap and then split.
             */
            pr_err("gfn 0x%llx old_spte 0x%llx new_spte 0x%llx level %d\n",
                   gfn, old_spte, new_spte, level);
            WARN_ON(1); //拆分临时被 blocked 的私有页面不期望走到这个分支，警告，期望走到前一个分支
        } else {
            /*
             * Because page is pined (refer to
             * kvm_faultin_pfn_private()), page migration shouldn't
             * be triggered for private page.  kvm private memory
             * slot case should also prevent page migration.
             */
            pr_err("gfn 0x%llx old_spte 0x%llx new_spte 0x%llx level %d\n",
                   gfn, old_spte, new_spte, level);
            WARN_ON(1);
        }
    } else {//新旧 SPTE 都是未映射，且根据上面 BUG_ON 条件知道旧 SPTE 是临时被 blocked 的私有页面
        lockdep_assert_held_write(&kvm->mmu_lock);
        ret = static_call(kvm_x86_remove_private_spte)(kvm, gfn, level, old_pfn);
        WARN_ON_ONCE(ret); //这种情况可以 TDH.MEM.PAGE.REMOVE 移除旧页面
    }

    return ret;
}
```
#### 修改 `__handle_changed_spte()`
* 该函数仅被 `__handle_changed_spte()` 调用
  * 不允许旧 SPTE 是临时被 blocked 的私有页面，而新 SPTE 不是私有的情况
    * 即 `KVM_BUG_ON(was_private_zapped && !is_private, kvm)`
    * 是因为旧的临时被 blocked 的私有页面需要被从 TD 删除，而不能直接转换为共享页面吗？
  * 当旧 SPTE 是临时被 blocked 的私有页面时，需先用 `handle_private_zapped_spte()` 处理。根据前面的分析知道这种 SPTE 必然是叶子 SPTE<sup>[2]</sup>
    * 这是批量 TLB shootdown 5 步走的第 5 步做准备
  * 当旧 SPTE 不是临时被 blocked 的私有页面，而新 SPTE 是时，用 `handle_changed_private_spte()` 处理（`was_private_zapped != is_private_zapped`）
    * 因为如果旧 SPTE 是临时被 blocked 的私有页面在之前 [2] 就被处理了，所以 `was_private_zapped && is_private_zapped` 不应该成立
    * 这是批量 TLB shootdown 5 步走的第 2 步做准备
  * 当持有写锁的时候，叶子 PTE 应该被去除或者禁止。而不是直接从 *已映射* 变为 *零 EPT 条目*
```diff
@@ -652,11 +719,14 @@ static int __must_check __handle_changed_spte(struct kvm *kvm, int as_id, gfn_t
        kvm_pfn_t old_pfn = spte_to_pfn(old_spte);
        kvm_pfn_t new_pfn = spte_to_pfn(new_spte);
        bool pfn_changed = old_pfn != new_pfn;
+       bool was_private_zapped = is_private_zapped_spte(old_spte);
+       bool is_private_zapped = is_private_zapped_spte(new_spte);

        WARN_ON(level > PT64_ROOT_MAX_LEVEL);
        WARN_ON(level < PG_LEVEL_4K);
        WARN_ON(gfn & (KVM_PAGES_PER_HPAGE(level) - 1));
        KVM_BUG_ON(kvm_is_private_gpa(kvm, gfn_to_gpa(gfn)) != is_private, kvm);
+       KVM_BUG_ON(was_private_zapped && !is_private, kvm);

        /*
         * If this warning were to trigger it would indicate that there was a
@@ -689,6 +759,9 @@ static int __must_check __handle_changed_spte(struct kvm *kvm, int as_id, gfn_t
        if (is_leaf)
                check_spte_writable_invariants(new_spte);

+       if (was_private_zapped)
+               return handle_private_zapped_spte(kvm, gfn, old_spte, new_spte, level);
+
        /*
         * The only times a SPTE should be changed from a non-present to
         * non-present state is when an MMIO entry is installed/modified/
@@ -745,8 +818,16 @@ static int __must_check __handle_changed_spte(struct kvm *kvm, int as_id, gfn_t
         */
        if (is_private &&
            /* Ignore change of software only bits. e.g. host_writable */
-           (was_leaf != is_leaf || was_present != is_present || pfn_changed))
+           (was_leaf != is_leaf || was_present != is_present || pfn_changed ||
+            was_private_zapped != is_private_zapped)) {
+               KVM_BUG_ON(was_private_zapped && is_private_zapped, kvm);
+               /*
+                * When write lock is held, leaf pte should be zapping or
+                * prohibiting.  Not directly was_present=1 -> zero EPT entry.
+                */
+               KVM_BUG_ON(!shared && is_leaf && !is_private_zapped, kvm);
                return handle_changed_private_spte(kvm, gfn, old_spte, new_spte, role.level);
+       }
        return 0;
 }
```

#### 对 `handle_removed_pt()` 的修改
* `handle_removed_pt()` 只被 `__handle_changed_spte()` 调用，用于删除子树，递归处理子页表，将其从分页结构中移除
* 当销毁 VM 时会去除所有页面，这时会走到这。在此时 TLB shootdown 优化并不合理，
* 子树迭代会跳过 *未映射* 且 *非临时被 blocked* 的 SPTE，这种页面已经是被去除的无效的页面了（即仅仅是 `non-present`）
* 子树迭代到 *已映射* 的 SPTE，还是维持原有逻辑，将 SPTE 设为 freeze 状态（`REMOVED_SPTE`），递归删除子树
* *未映射* 且 *临时被 blocked* 的 SPTE 走不到这个函数，在其父 SPTE 的子树迭代调用链 `handle_removed_pt() -> handle_changed_spte() -> __handle_changed_spte() -> handle_private_zapped_spte()` 中：
  * *未映射* 且 *临时被 blocked 的 SPTE* 指向的 *页面* 会在递归的末端的 `__handle_changed_spte() -> handle_private_zapped_spte()` 中被删除
  * 紧接着，*未映射* 且 *临时被 blocked* SPTE 会在递归末端`__handle_changed_spte()` 返回到 `handle_changed_spte()` 再返回到的 `handle_removed_pt()` 中被删除
    * 因为 SPTE 指向的数据页面已经被删除，所以删除其 SPTE 页表页也是合理的
  * 况且不满足调用 `handle_removed_pt()` 的 `!was_last` 这个条件
```diff
diff --git a/arch/x86/kvm/mmu/tdp_mmu.c b/arch/x86/kvm/mmu/tdp_mmu.c
index 13ed0263857c..8393b1c75f7c 100644
--- a/arch/x86/kvm/mmu/tdp_mmu.c
+++ b/arch/x86/kvm/mmu/tdp_mmu.c
@@ -497,7 +497,13 @@ static void handle_removed_pt(struct kvm *kvm, tdp_ptep_t pt, bool shared)
                         * unreachable.
                         */
                        old_spte = kvm_tdp_mmu_read_spte(sptep);
-                       if (!is_shadow_present_pte(old_spte))
+                       /*
+                        * It comes here when zapping all pages when destroying
+                        * vm.  It means TLB shootdown optimization doesn't make
+                        * sense.  Zap private_zapped entry.
+                        */
+                       if (!is_shadow_present_pte(old_spte) &&
+                           !is_private_zapped_spte(old_spte))
                                continue;

                        /*
```

#### 修改 `handle_changed_private_spte()`
* 接着 `handle_changed_private_spte()` 被 `__handle_changed_spte()` 调用，用于处理新 SPTE 发生变化的情况
* 改动在 *新 SPTE 不存在映射* 且 *旧 SPTE 又是最后一级条目* 的分支，即删除的情况
  * 如果新 SPTE 是临时被 blocked 的私有 SPTE，那就是要将 **已映射 SPTE 转为临时被 blocked 的私有 SPTE** 的情况，
    * 此时新旧 `PFN` 必须一致，按照批量 TLB shootdown 的设计，将要删除的 `PFN` 保存在这，因此此处必须一致
    * 但此时无需移除私有页面，等以后再统一删除，因此仅仅是用 `kvm_x86_zap_private_spte`（SEAMCALL `TDH.MEM.RANGE.BLOCK`）
    * 这就实现了批量 TLB shootdown 5 步走的第 2 步，只 block，不删除
  * 如果新 SPTE 既不存在映射也不是临时 blocked 的私有 SPTE，即`PRESENT => NON-PRESENT`，没有临时被 blocked 私有 SPTE 的中间状态，基本上按着旧有的逻辑进行，且新 SPTE 的 `PFN` 部分必须为全零

```diff
@@ -608,17 +668,24 @@ static int __must_check handle_changed_private_spte(struct kvm *kvm, gfn_t gfn,
                        ret = static_call(kvm_x86_link_private_spt)(kvm, gfn, level, private_spt);
                }
        } else if (was_leaf) {
-               /* non-present -> non-present doesn't make sense. */
-               KVM_BUG_ON(!was_present, kvm);
                /*
                 * Zap private leaf SPTE.  Zapping private table is done
                 * below in handle_removed_tdp_mmu_page().
                 */
                lockdep_assert_held_write(&kvm->mmu_lock);
                ret = static_call(kvm_x86_zap_private_spte)(kvm, gfn, level);
-               if (!ret) {
-                       ret = static_call(kvm_x86_remove_private_spte)(kvm, gfn, level, old_pfn);
-                       WARN_ON_ONCE(ret);
+               if (is_private_zapped) {
+                       KVM_BUG_ON(new_pfn != old_pfn, kvm);
+               } else {
+                       /* non-present -> non-present doesn't make sense. */
+                       KVM_BUG_ON(!was_present, kvm);
+                       KVM_BUG_ON(new_pfn, kvm);
+
+                       if (!ret) {
+                               ret = static_call(kvm_x86_remove_private_spte)(kvm, gfn,
+                                                                              level, old_pfn);
+                               WARN_ON_ONCE(ret);
+                       }
                }
        }
        return ret;
```

#### 对 TDP 迭代器的修改
* 注意，虽然对临时 blocked 的私有 SPTE `!is_shadow_present_pte()` 返回 `true`，但是它仍然被认为是有效的叶子节点，因为（映射到的）目标页面仍然在那儿
  * 所以 *未映射* 且 *非临时 blocked 的私有 SPTE* 才需要被跳过，仅仅是 *未映射* 且 *临时 blocked 的私有 SPTE* 还是会被迭代到
```diff
@@ -979,12 +1073,18 @@ static inline void tdp_mmu_set_spte_no_dirty_log(struct kvm *kvm,
 #define tdp_root_for_each_pte(_iter, _root, _start, _end) \
        for_each_tdp_pte(_iter, _root, _start, _end)

-#define tdp_root_for_each_leaf_pte(_iter, _root, _start, _end) \
+/*
+ * Note temporarily blocked private SPTE is considered as valid leaf, although
+ * !is_shadow_present_pte() returns true for it, since the target page (which
+ * the mapping maps to ) is still there.
+ */
+#define tdp_root_for_each_leaf_pte(_iter, _root, _start, _end)         \
        tdp_root_for_each_pte(_iter, _root, _start, _end)               \
-               if (!is_shadow_present_pte(_iter.old_spte) ||           \
-                   !is_last_spte(_iter.old_spte, _iter.level))         \
+               if ((!is_shadow_present_pte(_iter.old_spte) &&          \
+                    !is_private_zapped_spte(_iter.old_spte)) ||        \
+                    !is_last_spte(_iter.old_spte, _iter.level)) {      \
                        continue;                                       \
-               else
+               } else

 #define tdp_mmu_for_each_pte(_iter, _mmu, _private, _start, _end)      \
        for_each_tdp_pte(_iter,                                         \
```

#### `struct kvm_tdx` 引入 `has_range_blocked` 域
* `struct kvm_tdx` 引入 `has_range_blocked` 域，用于表明该 TDX VM 有被按范围 blocked 的 SPTE
```diff
diff --git a/arch/x86/kvm/vmx/tdx.h b/arch/x86/kvm/vmx/tdx.h
index 9057aa282259..cdc5aac791e4 100644
--- a/arch/x86/kvm/vmx/tdx.h
+++ b/arch/x86/kvm/vmx/tdx.h
@@ -24,6 +24,9 @@ struct kvm_tdx {
        atomic_t tdh_mem_track;

        u64 tsc_offset;
+
+       /* TDP MMU */
+       bool has_range_blocked;
 };

 union tdx_exit_reason {
```
* TDX VM 初始化时将该域设为 `false`
```diff
diff --git a/arch/x86/kvm/vmx/tdx.c b/arch/x86/kvm/vmx/tdx.c
index e39cf883405d..b848c28ef518 100644
--- a/arch/x86/kvm/vmx/tdx.c
+++ b/arch/x86/kvm/vmx/tdx.c
@@ -475,6 +475,8 @@ static int tdx_do_tdh_mng_key_config(void *param)

 int tdx_vm_init(struct kvm *kvm)
 {
+       struct kvm_tdx *kvm_tdx = to_kvm_tdx(kvm);
+
        /*
         * Because guest TD is protected, VMM can't parse the instruction in TD.
         * Instead, guest uses MMIO hypercall.  For unmodified device driver,
@@ -491,6 +493,8 @@ int tdx_vm_init(struct kvm *kvm)
        /* TDH.MEM.PAGE.AUG supports up to 2MB page. */
        kvm->arch.tdp_max_page_level = PG_LEVEL_2M;

+       kvm_tdx->has_range_blocked = false;
+
        /*
         * This function initializes only KVM software construct.  It doesn't
         * initialize TDX stuff, e.g. TDCS, TDR, TDCX, HKID etc.
```
* `tdx_sept_zap_private_spte()`，即 `kvm_x86_zap_private_spte` 的回调，最终会调用 SEAMCALL `TDH.MEM.RANGE.BLOCK`，所以在此处将 `has_range_blocked` 域置为 `true`
```diff
@@ -1675,6 +1679,8 @@ static int tdx_sept_zap_private_spte(struct kvm *kvm, gfn_t gfn,
        err = tdh_mem_range_block(kvm_tdx->tdr_pa, gpa, tdx_level, &out);
        if (err == TDX_ERROR_SEPT_BUSY)
                return -EAGAIN;
+
+       WRITE_ONCE(kvm_tdx->has_range_blocked, true);
        if (KVM_BUG_ON(err, kvm)) {
                pr_tdx_error(TDH_MEM_RANGE_BLOCK, err, &out);
                return -EIO;
```
* 在 TD Live migration 的函数 `tdx_write_block_private_pages()` 也会修改该域

#### 修改 `free_private_spt` 的回调 `tdx_sept_free_private_spt()`
* `free_private_spt()`（显然）是在 shadow page 被清除时调用的。当 TD 处于活动状态时，KVM 不会（还）清除私有 shadow page。
  * 注意：该函数适用于 private shadow page。不适用于 private guest page。在 TD 处于活动状态期间可以清除 private guest page。
  * 比如，共享 <-> 私有转换和 memory slot 移动/删除时。
* 对于 TD 处于活动状态时，如果调用该函数会做以下动作：
  1. SEAMCALL `TDH.MEM.RANGE.BLOCK` 父一级的 SPTE
  2. SEAMCALL `TDH.MEM.TRACK` TD
  3. SEAMCALL `TDH.MEM.SEPT.REMOVE` 父一级的 SPTE
  4. SEAMCALL `TDH.PHYMEM.PAGE.WBINVD` 私有 secure page `set_hkid_to_hpa(__pa(private_spt), kvm_tdx->hkid)`
  5. `tdx_set_page_present(__pa(private_spt))` 调的是 `set_direct_map_default_noflush(pfn_to_page(addr >> PAGE_SHIFT))`，将 page 对应的内核直接映射的页表中的 PTE 的值设为缺省值
```cpp
static int tdx_sept_free_private_spt(struct kvm *kvm, gfn_t gfn,
                     enum pg_level level, void *private_spt)
{
    /* +1 to remove this SEPT page from the parent's entry. */
    gpa_t parent_gpa = gfn_to_gpa(gfn) & KVM_HPAGE_MASK(level + 1); //得到父 GPA
    int parent_tdx_level = pg_level_to_tdx_sept_level(level + 1);   //得到父页表级数，转化为 TDX SPTE 级数
    struct kvm_tdx *kvm_tdx = to_kvm_tdx(kvm);
    struct tdx_module_output out;
    u64 err;
    //分配给该 TD 的 HKID 已被释放并且 cache 已被刷新。我们不用再 flush 了。
    /*
     * The HKID assigned to this TD was already freed and cache was
     * already flushed. We don't have to flush again.
     */
    if (!is_hkid_assigned(kvm_tdx))
        return tdx_reclaim_page(__pa(private_spt), PG_LEVEL_4K, false, 0);
    //效率低下。但这只是为了删除 memslot 而调用，这不是性能关键路径。
    /*
     * Inefficient. But this is only called for deleting memslot
     * which isn't performance critical path.
     *
     * free_private_spt() is (obviously) called when a shadow page is being
     * zapped.  KVM doesn't (yet) zap private SPs while the TD is active.
     * Note: This function is for private shadow page.  Not for private
     * guest page.   private guest page can be zapped during TD is active.
     * shared <-> private conversion and slot move/deletion.
     */
    if (kvm_tdx->td_initialized) {
        err = tdh_mem_range_block(kvm_tdx->tdr_pa, parent_gpa,
                      parent_tdx_level, &out);
        if (KVM_BUG_ON(err, kvm)) {
            pr_tdx_error(TDH_MEM_RANGE_BLOCK, err, &out);
            return -EIO;
        }
    }

    tdx_track(kvm_tdx);

    err = tdh_mem_sept_remove(kvm_tdx->tdr_pa, parent_gpa,
                  parent_tdx_level, &out);
    if (KVM_BUG_ON(err, kvm)) {
        pr_tdx_error(TDH_MEM_SEPT_REMOVE, err, &out);
        return -EIO;
    }

    err = tdh_phymem_page_wbinvd(set_hkid_to_hpa(__pa(private_spt),
                             kvm_tdx->hkid));
    if (WARN_ON_ONCE(err)) {
        pr_tdx_error(TDH_PHYMEM_PAGE_WBINVD, err, NULL);
        return -EIO;
    }

    tdx_set_page_present(__pa(private_spt));
    return 0;
}
```

#### 新增 `tdx_sept_tlb_remote_flush_with_range()` 函数
* `tdx_sept_tlb_remote_flush_with_range()` 取代了 `tdx_sept_tlb_remote_flush()`，而 `tdx_sept_tlb_remote_flush()` 改为对 `tdx_sept_tlb_remote_flush_with_range()` 的封装
  * `tdx_sept_tlb_remote_flush()` 是全范围的，是 `.tlb_remote_flush` 的对应回调
  * `tdx_sept_tlb_remote_flush_with_range()` 是指定范围的，是 `.tlb_remote_flush_with_range` 的对应回调
```diff
-int tdx_sept_tlb_remote_flush(struct kvm *kvm)
+int tdx_sept_tlb_remote_flush_with_range(struct kvm *kvm,
+                                        struct kvm_tlb_range *range)
 {
        struct kvm_tdx *kvm_tdx;

@@ -1796,6 +1834,16 @@ int tdx_sept_tlb_remote_flush(struct kvm *kvm)
        return 0;
 }

+int tdx_sept_tlb_remote_flush(struct kvm *kvm)
+{
+       struct kvm_tlb_range range = {
+               .start_gfn = 0,
+               .pages = -1ULL,
+       };
+
+       return tdx_sept_tlb_remote_flush_with_range(kvm, &range);
+}
+
 static int tdx_sept_remove_private_spte(struct kvm *kvm, gfn_t gfn,
                                         enum pg_level level, kvm_pfn_t pfn)
 {
```
* 于是，TDX 的 `.tlb_remote_flush_with_range` 回调可以支持了，即 `tdx_sept_tlb_remote_flush_with_range()`
```diff
diff --git a/arch/x86/kvm/vmx/main.c b/arch/x86/kvm/vmx/main.c
index cb7f4799eb70..ec9cc1181605 100644
--- a/arch/x86/kvm/vmx/main.c
+++ b/arch/x86/kvm/vmx/main.c
@@ -570,7 +570,7 @@ static int vt_tlb_remote_flush_with_range(struct kvm *kvm,
                                          struct kvm_tlb_range *range)
 {
        if (is_td(kvm))
-               return -EOPNOTSUPP; /* fall back to tlb_remote_flush */
+               return tdx_sept_tlb_remote_flush_with_range(kvm, range);

        return vmx_tlb_remote_flush_with_range(kvm, range);
 }
```
* 调用的源头是 `kvm_flush_remote_tlbs_with_address()`，调用者有多处
```cpp
//arch/x86/kvm/mmu/mmu.c
kvm_flush_remote_tlbs_with_address()
-> kvm_flush_remote_tlbs_with_range()
   -> static_call(kvm_x86_tlb_remote_flush_with_range)(kvm, range)
      //arch/x86/kvm/vmx/main.c
   => vt_tlb_remote_flush_with_range()
      -> tdx_sept_tlb_remote_flush_with_range()
         //arch/x86/kvm/vmx/tdx.c
         -> tdx_track()
```
* 这就实现了批量 TLB shootdown 5 步走的第 3、4 步，track 和 send IPI

### KVM: x86/tdp_mmu: Unzap and split private large SPTE on KVM page fault
```c
commit 1359e9b1ebb242ebafa7d13214907ae7fb3adb85
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Fri Jun 3 12:04:17 2022 -0700

    KVM: x86/tdp_mmu: Unzap and split private large SPTE on KVM page fault

    When faulting on large SPTE for lower level, the current logic is zap the
    large SPTE if necessary and go on to allocate lower SPTE page.  It doesn't
    work for private SPTE because guest private page contents will be lost.

    When faulting on large SPTE for lower level, unzap it if necessary and
    split it instead of zapping.
```
* 当缺页发生在大页的 SPTE 的下面的层级时，当前的逻辑是：如果必要的话，去除大页 SPTE，然后继续分配低层级的 SPTE 页表页
```c
kvm_tdp_mmu_map()
-> tdp_mmu_split_huge_page()
   -> tdp_mmu_link_sp()
      -> tdp_mmu_set_spte_atomic()
         -> __handle_changed_spte()
            -> handle_changed_private_spte()
               -> static_call(kvm_x86_zap_private_spte)()
               -> static_call(kvm_x86_split_private_spt)()
```
* 对于私有 SPTE 的情况这个做法不工作，因为这样会把 guest 私有页面的内容丢掉
* 新的做法是，缺页发生在大页的 SPTE 的下属层级时，如果必要的话，不去除大页并拆分它，取代去除它的做法
* 引入新函数 `tdp_mmu_unzap_large_spte()` 直接处理遍历到的临时被 blocked 的 SPTE 是大页的情况
  * `tdp_mmu_set_spte_atomic() -> __handle_changed_spte() -> handle_private_zapped_spte()` 会将临时被 blocked 的 SPTE 指向的大页 unblock
  * SPTE 的内容会被新构造的值取代
```cpp
/*
 * unzap large page spte: shortened version of tdp_mmu_map_handle_target_level()
 */
static int tdp_mmu_unzap_large_spte(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault,
                    struct tdp_iter *iter)
{
    struct kvm_mmu_page *sp = sptep_to_sp(rcu_dereference(iter->sptep));
    kvm_pfn_t mask = KVM_HPAGE_GFN_MASK(iter->level);
    u64 new_spte;
    //该函数用于临时被 blocked 的 SPTE，根据设计 SPTE 的值此时必然是 PFN，否则就是 bug
    KVM_BUG_ON((fault->pfn & mask) != spte_to_pfn(iter->old_spte), vcpu->kvm);
    make_spte(vcpu, sp, fault->slot, ACC_ALL, gpa_to_gfn(fault->addr),
          fault->pfn & mask, iter->old_spte, false, true,
          fault->map_writable, &new_spte);
    //重新构造的 SPTE 的值，和原来的值通常不会相等
    if (new_spte == iter->old_spte)
        return RET_PF_SPURIOUS;
    //将 SPTE 设置为新构造的值
    if (tdp_mmu_set_spte_atomic(vcpu->kvm, iter, new_spte))
        return RET_PF_RETRY;
    trace_kvm_mmu_set_spte(iter->level, iter->gfn,
                   rcu_dereference(iter->sptep));
    iter->old_spte = new_spte; //记录新的值到迭代器用于记录 SPTE 值的域
    return RET_PF_CONTINUE;
}
```
* 看调用它的条件：*临时被 blocked 的 SPTE* 且是 *大页*
* `RET_PF_CONTINUE` 表示 unblock 成功，可以继续拆分大页
```diff
@@ -1737,6 +1763,13 @@ int kvm_tdp_mmu_map(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault)
                    !is_large_pte(iter.old_spte))
                        continue;

+               if (is_private_zapped_spte(iter.old_spte) &&
+                   is_large_pte(iter.old_spte)) {
+                       if (tdp_mmu_unzap_large_spte(vcpu, fault, &iter) !=
+                           RET_PF_CONTINUE)
+                               break;
+               }
+
                /*
                 * The SPTE is either non-present or points to a huge page that
                 * needs to be split.
```

### KVM: x86/tdp_mmu: don't follow down of large spte
```diff
commit 37cfaa52503be0a8acffb74508fc3fcbe2126d50
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Fri Jun 3 16:59:15 2022 -0700

    KVM: x86/tdp_mmu: don't follow down of large spte

    When zapping, __handle_changed_spte() follows down if !was_leaf.  But
    was_leaf is defined as is_present && is_last.  So the check isn't correct
    for private_zapped large spte.  it wrongly follows down.  Use was_last
    check instead of was_leaf.

diff --git a/arch/x86/kvm/mmu/tdp_mmu.c b/arch/x86/kvm/mmu/tdp_mmu.c
index c1dfdfc02bf1..41ee1eb58f2f 100644
--- a/arch/x86/kvm/mmu/tdp_mmu.c
+++ b/arch/x86/kvm/mmu/tdp_mmu.c
@@ -714,7 +714,8 @@ static int __must_check __handle_changed_spte(struct kvm *kvm, int as_id, gfn_t
        int level = role.level;
        bool was_present = is_shadow_present_pte(old_spte);
        bool is_present = is_shadow_present_pte(new_spte);
-       bool was_leaf = was_present && is_last_spte(old_spte, level);
+       bool was_last = is_last_spte(old_spte, level);
+       bool was_leaf = was_present && was_last;
        bool is_leaf = is_present && is_last_spte(new_spte, level);
        kvm_pfn_t old_pfn = spte_to_pfn(old_spte);
        kvm_pfn_t new_pfn = spte_to_pfn(new_spte);
@@ -800,7 +801,7 @@ static int __must_check __handle_changed_spte(struct kvm *kvm, int as_id, gfn_t
         * SPTE being converted to a hugepage (leaf) or being zapped.  Shadow
         * pages are kernel allocations and should never be migrated.
         */
-       if (was_present && !was_leaf &&
+       if (was_present && !was_last &&
            (is_leaf || !is_present || WARN_ON_ONCE(pfn_changed))) {
                KVM_BUG_ON(is_private != is_private_sptep(spte_to_child_pt(old_spte, level)),
                           kvm);
```
* 修改 `__handle_changed_spte()` 调用 `handle_removed_pt()` 的判断条件，
  * 其中 `was_leaf` 的含义是旧 SPTE *存在映射* 且是 *最后一级* SPTE；那么 `!was_leaf` 成立的条件是旧 SPTE 不存在映射 **或** 不是最后一级 SPTE
  * `was_present && !was_leaf` 成立的条件是旧 SPTE *已映射* 且 *不是最后一级 SPTE*
    * 作者似乎认为，当处理一个临时被 blocked 的 SPTE 是大页时会因为 `!was_leaf` 这个条件成立而错误地往下走
    * 然而从上面的分析可以看到不存在这种情况：因为一个临时被 blocked 的 SPTE 是大页必然是最后一级，那么 `!was_leaf` 成立的条件就只剩 *旧 SPTE 不存在映射* 了，但这与前面那个 `was_present` 旧 SPTE 不存在映射的条件对立
* 现在新加一个 `was_last` 单纯表明旧 SPTE 是最后一级 SPTE，与是否存在映射无关，
  * 结果 `__handle_changed_spte()` 调用 `handle_removed_pt()` 的判断条件的前面部分依然是，旧 SPTE *已映射* 且 *不是最后一级 SPTE*，语义没改变啊？！

### KVM: x86/tdp_mmu: Allow nx huge page recovery for TDX
```diff
commit 4d2be474b73da3bff98a78e35f3c91d598f3f66e
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Thu Dec 8 09:58:17 2022 -0800

    KVM: x86/tdp_mmu: Allow nx huge page recovery for TDX

    Now x86/tdp_mmu private mapping supports large page merging, allow nx huge
    page recovery.
diff --git a/arch/x86/kvm/mmu/tdp_mmu.c b/arch/x86/kvm/mmu/tdp_mmu.c
index 41ee1eb58f2f..7a89598a21f3 100644
--- a/arch/x86/kvm/mmu/tdp_mmu.c
+++ b/arch/x86/kvm/mmu/tdp_mmu.c
@@ -1214,6 +1214,7 @@ static void tdp_mmu_zap_root(struct kvm *kvm, struct kvm_mmu_page *root,
 bool kvm_tdp_mmu_zap_sp(struct kvm *kvm, struct kvm_mmu_page *sp)
 {
        u64 old_spte;
+       u64 new_spte;

        /*
         * This helper intentionally doesn't allow zapping a root shadow page,
@@ -1226,8 +1227,13 @@ bool kvm_tdp_mmu_zap_sp(struct kvm *kvm, struct kvm_mmu_page *sp)
        if (WARN_ON_ONCE(!is_shadow_present_pte(old_spte)))
                return false;

+       if (kvm_gfn_shared_mask(kvm) && is_private_sp(sp))
+               new_spte = __private_zapped_spte(old_spte);
+       else
+               new_spte = SHADOW_NONPRESENT_VALUE;
+
        __tdp_mmu_set_spte(kvm, kvm_mmu_page_as_id(sp), sp->ptep, old_spte,
-                          SHADOW_NONPRESENT_VALUE, sp->gfn, sp->role.level + 1,
+                          new_spte, sp->gfn, sp->role.level + 1,
                           true, true);

        return true;
@@ -2443,13 +2449,6 @@ void kvm_tdp_mmu_zap_collapsible_sptes(struct kvm *kvm,

        lockdep_assert_held_read(&kvm->mmu_lock);

-       /*
-        * This should only be reachable when diryt-log is supported. It's a
-        * bug to reach here.
-        */
-       if (WARN_ON_ONCE(!kvm_arch_dirty_log_supported(kvm)))
-               return;
-
        for_each_valid_tdp_mmu_root_yield_safe(kvm, root, slot->as_id, true)
                zap_collapsible_spte_range(kvm, root, slot);
 }
```
* 现在 TDP MMU 私有映射支持大页合并，因此允许 NX 大页的恢复？

### KVM: TDX: Allow splitting private zapped large page
```c
commit 61b92fc86705eb7ec381f86eb0cf59c91f945385
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Thu Jul 7 12:39:15 2022 -0700

    KVM: TDX: Allow splitting private zapped large page

    It can happen to split private zapped large page.  Implement the path.
    Demote large page and block the range to keep private zapped bit.
```
* 允许拆分临时被 blocked 的大页，前面分析 `KVM: x86/tdp_mmu: Unzap and split private large SPTE on KVM page fault` 的时候已经看到会出现这种情况
* 并且这种情况会走到上面的分支 `if (old_pfn == new_pfn)`，走下面的分支是不被期望的，警告
```diff
diff --git a/arch/x86/kvm/mmu/spte.c b/arch/x86/kvm/mmu/spte.c
index 9c874bca69f6..10425024ae32 100644
--- a/arch/x86/kvm/mmu/spte.c
+++ b/arch/x86/kvm/mmu/spte.c
@@ -280,7 +280,8 @@ u64 make_huge_page_split_spte(struct kvm *kvm, u64 huge_spte, union kvm_mmu_page
 {
        u64 child_spte;

-       if (WARN_ON_ONCE(!is_shadow_present_pte(huge_spte)))
+       if (WARN_ON_ONCE(!is_shadow_present_pte(huge_spte) &&
+                        !is_private_zapped_spte(huge_spte)))
                return 0;

        if (WARN_ON_ONCE(!is_large_pte(huge_spte)))
diff --git a/arch/x86/kvm/mmu/tdp_mmu.c b/arch/x86/kvm/mmu/tdp_mmu.c
index 7a89598a21f3..3dec4928878b 100644
--- a/arch/x86/kvm/mmu/tdp_mmu.c
+++ b/arch/x86/kvm/mmu/tdp_mmu.c
@@ -586,6 +586,8 @@ static int __must_check handle_private_zapped_spte(struct kvm *kvm, gfn_t gfn,
        bool is_private_zapped = is_private_zapped_spte(new_spte);
        bool was_present = is_shadow_present_pte(old_spte);
        bool is_present = is_shadow_present_pte(new_spte);
+       bool was_last = is_last_spte(old_spte, level);
+       bool is_last = is_last_spte(new_spte, level);
        kvm_pfn_t old_pfn = spte_to_pfn(old_spte);
        kvm_pfn_t new_pfn = spte_to_pfn(new_spte);
        int ret = 0;
@@ -607,10 +609,19 @@ static int __must_check handle_private_zapped_spte(struct kvm *kvm, gfn_t gfn,
         * because blocked private SPTE is also non-present.
         */
        if (is_present) {
-               lockdep_assert_held_read(&kvm->mmu_lock);
+               /* map_gpa holds write lock. */
+               lockdep_assert_held(&kvm->mmu_lock);

                if (old_pfn == new_pfn) {
                        ret = static_call(kvm_x86_unzap_private_spte)(kvm, gfn, level);
+               } else if (level > PG_LEVEL_4K && was_last && !is_last) {
+                       /*
+                        * Splitting private_zapped large page doesn't happen.
+                        * Unzap and then split.
+                        */
+                       pr_err("gfn 0x%llx old_spte 0x%llx new_spte 0x%llx level %d\n",
+                              gfn, old_spte, new_spte, level);
+                       WARN_ON(1);
                } else {
                        /*
                         * Because page is pined (refer to
```

### KVM: x86/tdp_mmu: Try to merge pages into a large page
```c
commit a6279d32800b9cb3e2334ea6f6639b71407ecb0c
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Wed Oct 12 17:06:16 2022 -0700

    KVM: x86/tdp_mmu: Try to merge pages into a large page
```
* 该 commit 的 subject 和 message 和 `cf73bfed3d1df69aefa8f4500e92e2ca9812199a` 一样啊？！

## TDX 大页支持

### KVM: TDP_MMU: Go to next level if smaller private mapping exists
```diff
commit f0e497f5aaa55d0a6018fd8ef4bd465915a71f6a
Author: Xiaoyao Li <xiaoyao.li@intel.com>
Date:   Wed May 25 14:41:54 2022 +0800

    KVM: TDP_MMU: Go to next level if smaller private mapping exists

    Cannot map a private page as large page if any smaller mapping exists.

    It has to wait for all the not-mapped smaller page to be mapped and
    promote it to larger mapping.

    Signed-off-by: Xiaoyao Li <xiaoyao.li@intel.com>

diff --git a/arch/x86/kvm/mmu/tdp_mmu.c b/arch/x86/kvm/mmu/tdp_mmu.c
index c65b48b2ba50..1a71aad62bd3 100644
--- a/arch/x86/kvm/mmu/tdp_mmu.c
+++ b/arch/x86/kvm/mmu/tdp_mmu.c
@@ -1394,7 +1394,8 @@ int kvm_tdp_mmu_map(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault)
        tdp_mmu_for_each_pte(iter, mmu, is_private, raw_gfn, raw_gfn + 1) {
                int r;

-               if (fault->nx_huge_page_workaround_enabled)
+               if (fault->nx_huge_page_workaround_enabled ||
+                   kvm_gfn_shared_mask(vcpu->kvm))
                        disallowed_hugepage_adjust(fault, iter.old_spte, iter.level);

                /*
```
* 只要有任何更小的映射存在，一个私有页面就无法被映射成为大页，必须等到所有未映射的更小的页面都被映射了，然后 promote 到一个更大的映射
* 这与调用 `disallowed_hugepage_adjust()` 的语义是一样的，所以把 VM arch 是否支持 GFN 的 shared bit 作为是否调用 `disallowed_hugepage_adjust()` 的条件
* 但 `disallowed_hugepage_adjust()` 的 `spte_to_child_sp(spte)->nx_huge_page_disallowed)` 条件是否能够满足我很怀疑
  * 如果当前 VM arch 没启用不允许可执行大页的缓解措施，出现这种情况也没有对此进行调整？！
  * 没看到哪里对 TDX 必须启用该缓解措施修改啊

### KVM: x86/tdp_mmu: Split the large page when zap leaf
```c
commit c35eb764d73fcf9f016158f3d2a2f78850fd1952
Author: Xiaoyao Li <xiaoyao.li@intel.com>
Date:   Wed May 25 14:41:57 2022 +0800

    KVM: x86/tdp_mmu: Split the large page when zap leaf

    When TDX enabled, a large page cannot be zapped if it contains mixed
    pages. In this case, it has to split the large page.
```
* 当启用 TDX 的时候，去除叶子 SPTE 时如果遇到一个大页包含共享和私有混合的页面，那么它不能被直接去除，必须先将该大页拆分成非混合页面的小页分别去除
* 用 `struct kvm_lpage_info` 的 `.disallow_lpage` 域的 `31` bit 表明在这个级别中是否有私有和共享页面混合
```cpp
/*
 * Use a bit in disallow_lpage to indicate private/shared pages mixed at the
 * level. The remaining bits are used as a reference count.
 */
#define KVM_LPAGE_PRIVATE_SHARED_MIXED      (1U << 31)
```
* 新增加的 `kvm_mem_attr_is_mixed()` 根据给的 `gfn` 得到其所在 `slot` 的某个 `level` 上是否有私有和共享页面混合的情况
```cpp
static bool linfo_is_mixed(struct kvm_lpage_info *linfo)
{
    return linfo->disallow_lpage & KVM_LPAGE_PRIVATE_SHARED_MIXED;
}

bool kvm_mem_attr_is_mixed(struct kvm_memory_slot *slot, gfn_t gfn, int level)
{
    struct kvm_lpage_info *linfo = lpage_info_slot(gfn, slot, level);

    WARN_ON_ONCE(level == PG_LEVEL_4K);
    return linfo_is_mixed(linfo);
}
```
* `tdp_mmu_zap_leafs()` 从根页表开始清除指定范围的叶子 SPTE
* 如果可让步 `can_yield = true`，以下情况会释放 MMU lock
  * 调度器需要 CPU
  * 有 MMU lock 争用
* 如果不让步 `can_yield = false`，则这个过程中不会释放 MMU lock 或者重新调度，那么调用者必须确保不要提供太巨大的 GFN 范围，否则该操作会引起 soft lockup
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
    bool is_private = is_private_sp(root); //根页表是私有的
    struct kvm_mmu_page *split_sp = NULL;
    struct tdp_iter iter;
    //将 TDP MMU walk 限定在 host.MAXPHYADDR 范围内
    end = min(end, tdp_mmu_max_gfn_exclusive());

    lockdep_assert_held_write(&kvm->mmu_lock);
    WARN_ON_ONCE(zap_private && !is_private); //根页表是共享的，请求是清除私有页表页，发出警告

    if (!zap_private && is_private) //根页表是私有的，请求是不清除私有页表页
        return flush;               //本函数没事可做，返回传入的 flush

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
            flush = false; //如果可以调度且发生了调度，回来这里
            continue;
        }
        //跳过非叶子 SPTE
        if (!is_last_spte(iter.old_spte, iter.level))
            continue;
        //跳过未映射且非临时被 blocked 的私有页面，"已映射"或"未映射和临时被 blocked 的私有页面"也需被去除
        /*
         * Skip non-present SPTE, with exception of temporarily
         * blocked private SPTE, which also needs to be zapped.
         */
        if (!is_shadow_present_pte(iter.old_spte) &&
            !is_private_zapped_spte(iter.old_spte))
            continue;
        //拆分私有大页
        if (is_private && kvm_gfn_shared_mask(kvm) && //根页表是私有，VM 的 GFN 支持 shared bit
            is_large_pte(iter.old_spte)) {            //SPTE 指向一个大页
            gfn_t gfn = iter.gfn & ~kvm_gfn_shared_mask(kvm); //去掉 GFN 中的 shared bit
            gfn_t mask = KVM_PAGES_PER_HPAGE(iter.level) - 1; //根据页面级别得到 GFN 的掩码
            struct kvm_memory_slot *slot;
            struct kvm_mmu_page *sp;

            slot = gfn_to_memslot(kvm, gfn); //该 GFN 所属的 slot
            if (kvm_mem_attr_is_mixed(slot, gfn, iter.level) || //如果该 GFN 在该层级上有私有和共享页面混合
                (gfn & mask) < start ||                         //或者 GFN 在层级上所属的页的起始地址小于 start
                end < (gfn & mask) + KVM_PAGES_PER_HPAGE(iter.level)) { //或者 GFN 在层级上所属的页的结束地址超过 end
                WARN_ON_ONCE(!can_yield); //如果输入是“不能让步”，发出警告
                if (split_sp) {      //如果有暂存的 SP
                    sp = split_sp;   //继续之前的工作
                    split_sp = NULL;
                    sp->role = tdp_iter_child_role(&iter); //新分配的 SP 的 role 来自当前迭代到的 SPTE，当然 role.level 已减一
                } else {//总是先有 iter.yielded = true 再有 split_sp = sp，然后 continue 再通过 tdp_iter_restart() 将 iter.yielded = false
                    WARN_ON(iter.yielded); //所以 split_sp 为空时 iter.yielded = true 就发出警告
                    if (flush && can_yield) {
                        kvm_flush_remote_tlbs(kvm);
                        flush = false;
                    }
                    sp = tdp_mmu_alloc_sp_for_split(kvm, &iter, false, //互斥的方式分配一个 shadow page，分配过程可以挂起
                                    can_yield);
                    if (iter.yielded) {
                        split_sp = sp;  //暂存一下 SP
                        continue;
                    }
                }
                KVM_BUG_ON(!sp, kvm);
                //初始化分配好的 SP 的内容
                tdp_mmu_init_sp(sp, iter.sptep, iter.gfn);
                if (tdp_mmu_split_huge_page(kvm, &iter, sp, false)) { //拆分 SP 对应的大页
                    kvm_flush_remote_tlbs(kvm); //出错了，flush TLB
                    flush = false;
                    /* force retry on this gfn. */
                    iter.yielded = true;        //出错了，强制重试该 GFN
                } else
                    flush = true; //拆分成功
                continue; //别往下走去去除的步骤，只要符合条件就继续迭代和拆分
            }
        }

        if (!zap_private && is_private_zapped_spte(iter.old_spte))
            continue;
        //只将最后一级 SPTE 清除映射，改动会被传播到 TDX modules
        tdp_mmu_set_spte(kvm, &iter,
                 zap_private ?
                 SHADOW_NONPRESENT_VALUE :
                 private_zapped_spte(kvm, &iter));
        flush = true;
    }

    rcu_read_unlock();
    //如果有因失败而暂存的 SP，释放它
    if (split_sp) {
        WARN_ON(!can_yield);
        if (flush) {
            kvm_flush_remote_tlbs(kvm);
            flush = false;
        }

        write_unlock(&kvm->mmu_lock);
        tdp_mmu_free_sp(split_sp);
        write_lock(&kvm->mmu_lock);
    }

    /*
     * Because this flow zaps _only_ leaf SPTEs, the caller doesn't need
     * to provide RCU protection as no 'struct kvm_mmu_page' will be freed.
     */
    return flush;
}
```
* 问题：回过头来想想，这种混合私有/共享页面的大页怎么搞出来的？
  * 见 [TDX 共享和私有内存转换](tdx_share_to_private.md)


### KVM: x86/tdp_mmu, TDX: Split a large page when 4KB page within it converted to shared
* 当大页中的 4KB 页被转为 shared 时拆分大页
```c
commit fa18214d70710daa6351985c9ffdcde58bad0554
Author: Xiaoyao Li <xiaoyao.li@intel.com>
Date:   Tue Aug 31 15:34:48 2021 +0800

    KVM: x86/tdp_mmu, TDX: Split a large page when 4KB page within it converted to shared

    When mapping the shared page for TDX, it needs to zap private alias.

    In the case that private page is mapped as large page (2MB), it can be
    removed directly only when the whole 2MB is converted to shared.
    Otherwise, it has to split 2MB page into 512 4KB page, and only remove
    the pages that converted to shared.

    When a present large leaf spte switches to present non-leaf spte, TDX needs
    to split the corresponding SEPT page to reflect it.
```
* 当为 TDX 映射一个 shared 页面时，需要去除它的私有别名
* 在私有页面被映射为大页（2MB）的情况，只有在整个 2MB 范围都被转为 shared 时才能直接移除
  * 否则，不得不将 2MB 大页拆分为 512 个 4KB 的页，并且只移除转换为 shared 的页面
* 当一个已映射的大页的叶子 SPTE 切换为已映射的非叶子 SPTE 时，TDX 需要拆分相应的 SPTE 页来反映这一点

#### 增加 `split_private_spt` 的 `x86_ops` 方法
```diff
diff --git a/arch/x86/include/asm/kvm-x86-ops.h b/arch/x86/include/asm/kvm-x86-ops.h
index 0cf928d12067..1e86542141f7 100644
--- a/arch/x86/include/asm/kvm-x86-ops.h
+++ b/arch/x86/include/asm/kvm-x86-ops.h
@@ -97,6 +97,7 @@ KVM_X86_OP_OPTIONAL_RET0(get_mt_mask)
 KVM_X86_OP(load_mmu_pgd)
 KVM_X86_OP_OPTIONAL(link_private_spt)
 KVM_X86_OP_OPTIONAL(free_private_spt)
+KVM_X86_OP_OPTIONAL(split_private_spt)
 KVM_X86_OP_OPTIONAL(set_private_spte)
 KVM_X86_OP_OPTIONAL(remove_private_spte)
 KVM_X86_OP_OPTIONAL(zap_private_spte)
diff --git a/arch/x86/include/asm/kvm_host.h b/arch/x86/include/asm/kvm_host.h
index 9687d8c8031c..7c6f8380b7e8 100644
--- a/arch/x86/include/asm/kvm_host.h
+++ b/arch/x86/include/asm/kvm_host.h
@@ -1695,6 +1695,8 @@ struct kvm_x86_ops {
                                void *private_spt);
        int (*free_private_spt)(struct kvm *kvm, gfn_t gfn, enum pg_level level,
                                void *private_spt);
+       int (*split_private_spt)(struct kvm *kvm, gfn_t gfn, enum pg_level level,
+                                 void *private_spt);
        int (*set_private_spte)(struct kvm *kvm, gfn_t gfn, enum pg_level level,
                                 kvm_pfn_t pfn);
        int (*remove_private_spte)(struct kvm *kvm, gfn_t gfn, enum pg_level level,
diff --git a/arch/x86/kvm/vmx/tdx.c b/arch/x86/kvm/vmx/tdx.c
index bdfcbd0db531..3fb7eb0df3aa 100644
--- a/arch/x86/kvm/vmx/tdx.c
+++ b/arch/x86/kvm/vmx/tdx.c
@@ -2725,6 +2745,7 @@ int __init tdx_hardware_setup(struct kvm_x86_ops *x86_ops)

        x86_ops->link_private_spt = tdx_sept_link_private_spt;
        x86_ops->free_private_spt = tdx_sept_free_private_spt;
+       x86_ops->split_private_spt = tdx_sept_split_private_spt;
        x86_ops->set_private_spte = tdx_sept_set_private_spte;
        x86_ops->remove_private_spte = tdx_sept_remove_private_spte;
        x86_ops->zap_private_spte = tdx_sept_zap_private_spte;
```
#### 新增 `tdx_sept_split_private_spt()` 为 `split_private_spt` 的回调
```cpp
static int tdx_sept_split_private_spt(struct kvm *kvm, gfn_t gfn,
                      enum pg_level level, void *private_spt)
{
    int tdx_level = pg_level_to_tdx_sept_level(level); //页面级别转为 TDX SPTE 级别
    struct kvm_tdx *kvm_tdx = to_kvm_tdx(kvm);
    gpa_t gpa = gfn_to_gpa(gfn);   //要被拆分的大页的 GFN -> GPA
    hpa_t hpa = __pa(private_spt); //准备映射该 GFN 的新 secure EPT 页的 HVA -> HPA
    struct tdx_module_output out;
    u64 err;
    //调用 SEAMCALL<TDH.MEM.PAGE.DEMOTE>
    /* See comment in tdx_sept_set_private_spte() */
    err = tdh_mem_page_demote(kvm_tdx->tdr_pa, gpa, tdx_level, hpa, &out);
    if (KVM_BUG_ON(err, kvm)) {
        pr_tdx_error(TDH_MEM_PAGE_DEMOTE, err, &out);
        return -EIO;
    }

    return 0;
}
```
* 新增对 SEAMCALL `TDH.MEM.PAGE.DEMOTE` 的封装 `tdh_mem_page_demote()`

#### 新增对 `handle_changed_private_spte()` 的修改
* 修改在新 SPTE 存在映射的分支
* 由于存在旧 SPTE 被分割成新 SPTE 的情况，`present -> present` 并非不可能了
* 如果当前 SPTE 的级别大于 4KB，是叶子 SPTE，新 SPTE 不是叶子页
  * `level > PG_LEVEL_4K && was_leaf && !is_leaf`
  * 说明是在拆分大页的路径上过来的，例如：
```c
tdp_mmu_split_huge_page()
   -> tdp_mmu_link_sp()
      -> tdp_mmu_set_spte_atomic()
         -> __handle_changed_spte()
            -> handle_changed_private_spte()
```
* `private_spt` 为准备映射该 GFN 的新 secure EPT 页的 `HVA`
* `kvm_x86_zap_private_spte => tdx_sept_zap_private_spte()`，SEAMCALL `TDH.MEM.RANGE.BLOCK` 给定 `GFN` 和页面级别确定的范围
* `kvm_x86_split_private_spt => tdx_sept_split_private_spt()`，SEAMCALL `TDH.MEM.PAGE.DEMOTE` 将页面拆分请求传给 TDX module，修改 Secure EPT 页表
```diff
diff --git a/arch/x86/kvm/mmu/tdp_mmu.c b/arch/x86/kvm/mmu/tdp_mmu.c
index 2e55454c3e51..2fa6ec89a0fd 100644
--- a/arch/x86/kvm/mmu/tdp_mmu.c
+++ b/arch/x86/kvm/mmu/tdp_mmu.c
@@ -585,18 +585,24 @@ static int __must_check handle_changed_private_spte(struct kvm *kvm, gfn_t gfn,

    lockdep_assert_held(&kvm->mmu_lock);
    if (is_present) {
-       /* TDP MMU doesn't change present -> present */
-       KVM_BUG_ON(was_present, kvm);
+       void *private_spt;

-       /*
-        * Use different call to either set up middle level
-        * private page table, or leaf.
-        */
-       if (is_leaf)
+       if (level > PG_LEVEL_4K && was_leaf && !is_leaf) {
+           /*
+            * splitting large page into 4KB.
+            * tdp_mmu_split_huage_page() => tdp_mmu_link_sp()
+            */
+           private_spt = get_private_spt(gfn, new_spte, level);
+           KVM_BUG_ON(!private_spt, kvm);
+           ret = static_call(kvm_x86_zap_private_spte)(kvm, gfn, level);
+           kvm_flush_remote_tlbs(kvm);
+           if (!ret)
+               ret = static_call(kvm_x86_split_private_spt)(kvm, gfn,
+                                        level, private_spt);
+       } else if (is_leaf)
            ret = static_call(kvm_x86_set_private_spte)(kvm, gfn, level, new_pfn);
        else {
-           void *private_spt = get_private_spt(gfn, new_spte, level);
-
+           private_spt = get_private_spt(gfn, new_spte, level);
            KVM_BUG_ON(!private_spt, kvm);
            ret = static_call(kvm_x86_link_private_spt)(kvm, gfn, level, private_spt);
        }
```

### KVM: x86/tdp_mmu: Try to merge pages into a large page
* 尽力把页面合并成大页
```c
commit cf73bfed3d1df69aefa8f4500e92e2ca9812199a
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Wed Oct 12 17:06:16 2022 -0700

    KVM: x86/tdp_mmu: Try to merge pages into a large page

    When a large page is passed to the KVM page fault handler and some of sub
    pages are already populated, try to merge sub pages into a large page.
    This situation can happen when the guest converts small pages into shared
    and convert it back into private.

    When a large page is passed to KVM mmu page fault handler and the spte
    corresponding to the page is non-leaf (one or more of sub pages are already
    populated at lower page level), the current kvm mmu zaps non-leaf spte at a
    large page level, and populate a leaf spte at that level.  Thus small pages
    are converted into a large page.  However, it doesn't work for TDX because
    zapping and re-populating results in zeroing page content.  Instead,
    populate all small pages and merge them into a large page.

    Merging pages into a large page can fail when some sub pages are accepted
    and some are not.  In such case, with the assumption that guest tries to
    accept at large page size for performance when possible, don't try to be
    smart to identify which page is still pending, map all pages at lower page
    level, and let vcpu re-execute.
```
* 当一个大页传给 KVM 缺页 handler 并且一些子页面已经被填充了的时候，尽可能将子页面合并成一个大页。这种情况在 guest 将一些页面转为 shared 然后再转回到 private 的情况下可能会发生。
* 当一个大页传给 KVM MMU 缺页 handler 并且 SPTE 对应的页面是非叶子时（一个或多个子页面已经在低层的页面层级上被填充了），当前 KVM MMU 在大页的这一级上去除非叶子 SPTE，然后在该级上填充一个叶子 SPTE。因此，小的页面被转换成一个大的页面。
  * 然而，对于 TDX 这种方式是不工作的，因为去除和重填会导致零页面的内容。
  * 取而代之的是，填充所有小的页面，然后将它们合并成一个大的页面。
* 当一些子页面已被 accepted 而一些没有时，合并页面为一个大页可能会失败。在这种情况下，为了性能，假设 guest 在可能的情况下尽可能以大页的大小接受页面，
  * 而不是尽可能只能地识别哪些页面仍然 pending，在更低层的页面级别映射所有的页面，并且让 vCPU 重新执行。

#### 修改 `tdp_mmu_map_handle_target_level()`
* `__tdp_mmu_map_handle_target_level()` 取代了 `tdp_mmu_set_spte_atomic()` 的位置，必然含有相应的动作，即将新页表项的内容 `new_spte` 填充到 `iter->sptep` 指向的页表项，当然会引入一些其他的新的处理逻辑
```diff
diff --git a/arch/x86/kvm/mmu/tdp_mmu.c b/arch/x86/kvm/mmu/tdp_mmu.c
index 2fa6ec89a0fd..fc8f457292b9 100644
--- a/arch/x86/kvm/mmu/tdp_mmu.c
+++ b/arch/x86/kvm/mmu/tdp_mmu.c
@@ -1366,7 +1504,7 @@ static int tdp_mmu_map_handle_target_level(struct kvm_vcpu *vcpu,

    if (new_spte == iter->old_spte)
        ret = RET_PF_SPURIOUS;
-   else if (tdp_mmu_set_spte_atomic(vcpu->kvm, iter, new_spte))
+   else if (__tdp_mmu_map_handle_target_level(vcpu, fault, iter, new_spte))
        return RET_PF_RETRY;
    else if (is_shadow_present_pte(iter->old_spte) &&
         !is_last_spte(iter->old_spte, iter->level))
```

#### 新增函数 `__tdp_mmu_map_handle_target_level()`
* 私有页面有大小更小的页。例如，子页面从 shared 转为私有的页面，并且现在它可以被映射成为一个大页。尽可能地将小的页面合并成一个大的页面。
```cpp
static int __tdp_mmu_map_handle_target_level(struct kvm_vcpu *vcpu,
                         struct kvm_page_fault *fault,
                         struct tdp_iter *iter, u64 new_spte)
{
    /*
     * The private page has smaller-size pages.  For example, the child
     * pages was converted from shared to page, and now it can be mapped as
     * a large page.  Try to merge small pages into a large page.
     */
    if (fault->slot &&                           //缺页有对应的 slot
        kvm_gfn_shared_mask(vcpu->kvm) &&        //VM arch 支持 shared bit
        iter->level > PG_LEVEL_4K &&             //当前迭代的级别高于 4K
        kvm_is_private_gpa(vcpu->kvm, gfn_to_gpa(fault->gfn)) && //发生缺页的 GFN 是私有 GPA
        is_shadow_present_pte(iter->old_spte) && //当前 SPTE 映射已存在
        !is_large_pte(iter->old_spte))           //当前 SPTE 映射不是大页
        return tdp_mmu_merge_private_spt(vcpu, fault, iter, new_spte); //合并成大页，new_spte 是大页的 SPTE 的值
    //即将新页表项的内容 new_spte 填充到 iter->sptep 指向的页表项
    return tdp_mmu_set_spte_atomic(vcpu->kvm, iter, new_spte);
}
```
* 如果缺页级别被升级，例如经过 `kvm_mmu_hugepage_adjust()` 的调整，`fault->goal_level` 变大了

#### 修改 TDM MMU 迭代器
* 从 `try_step_down()` 中抽取代码成新函数 `step_down()`
```cpp
static void step_down(struct tdp_iter *iter, tdp_ptep_t child_pt)
{
    iter->level--;
    iter->pt_path[iter->level - 1] = child_pt;
    iter->gfn = round_gfn_for_level(iter->next_last_level_gfn, iter->level);
    tdp_iter_refresh_sptep(iter);
}
```
* 新增 `tdp_iter_step_down()` 函数用到了新函数 `step_down()`，用于向下步进 freezed 的 SPTE。不再重读 `sptep` 的原因是它已经被 freezed 了
```cpp
/* Steps down for freezed spte.  Don't re-read sptep because it was freezed. */
void tdp_iter_step_down(struct tdp_iter *iter, tdp_ptep_t child_pt)
{
    WARN_ON_ONCE(!child_pt);
    WARN_ON_ONCE(iter->yielded);
    WARN_ON_ONCE(iter->level == iter->min_level);

    step_down(iter, child_pt);
}
```
* 从 `try_step_side()` 中抽取代码成新函数 `try_step_side()`，用于侧向步进 SPTE，跳过前面是否迭代到当前页表页的边缘的检查
```cpp
void tdp_iter_step_side(struct tdp_iter * iter)
{
    iter->gfn += KVM_PAGES_PER_HPAGE(iter->level);
    iter->next_last_level_gfn = iter->gfn;
    iter->sptep++;
    iter->old_spte = kvm_tdp_mmu_read_spte(iter->sptep);
}
```

#### 新增函数 `tdp_mmu_merge_private_spt()`
* `tdp_mmu_merge_private_spt()` 
```cpp
static int tdp_mmu_merge_private_spt(struct kvm_vcpu *vcpu,
                     struct kvm_page_fault *fault,
                     struct tdp_iter *iter, u64 new_spte)
{
    u64 *sptep = rcu_dereference(iter->sptep);
    struct kvm_mmu_page *child_sp;
    struct kvm *kvm = vcpu->kvm;
    struct tdp_iter child_iter;
    bool ret_pf_retry = false;
    int level = iter->level;
    gfn_t gfn = iter->gfn;
    u64 old_spte = *sptep;
    tdp_ptep_t child_pt;
    u64 child_spte;
    int ret = 0;
    int i;
    //TDX KVM 只支持 2MB 大页。此时尚不支持合并 2MB 页到 1GB 大页
    /*
     * TDX KVM supports only 2MB large page.  It's not supported to merge
     * 2MB pages into 1GB page at the moment.
     */
    WARN_ON_ONCE(fault->goal_level != PG_LEVEL_2M);
    WARN_ON_ONCE(iter->level != PG_LEVEL_2M);
    WARN_ON_ONCE(!is_large_pte(new_spte)); //因为是合并，新 SPTE 必然是大页
    //冻结迭代器中的 old_spte 记录
    /* Freeze the spte to prevent other threads from working spte. */
    if (!try_cmpxchg64(sptep, &iter->old_spte, REMOVED_SPTE))
        return -EBUSY;
    //步入到子 SPTE。因为 tdp_iter_next() 假设父 SPTE 不是冻结的，手动去做
    /*
     * Step down to the child spte.  Because tdp_iter_next() assumes the
     * parent spte isn't freezed, do it manually.
     */
    child_pt = spte_to_child_pt(iter->old_spte, iter->level);
    child_sp = sptep_to_sp(child_pt);
    WARN_ON_ONCE(child_sp->role.level != PG_LEVEL_4K); //因为当前只支持 2MB 大页，所以子 SPTE 必然是 4KB
    WARN_ON_ONCE(!kvm_mmu_page_role_is_private(child_sp->role)); //合并只针对私有页面
    //不要修改迭代器，因为调用者后面还要使用这个迭代器。所以这里是值拷贝
    /* Don't modify iter as the caller will use iter after this function. */
    child_iter = *iter;
    /* Adjust the target gfn to the head gfn of the large page. */
    child_iter.next_last_level_gfn &= -KVM_PAGES_PER_HPAGE(level); //调整目的 GFN 到大页的开头的 GFN
    tdp_iter_step_down(&child_iter, child_pt); //迭代器步进到下一级
    //所有的子页面都需要被填充，以便将它们合并到一个大页。填充所有的子 SPTE
    /*
     * All child pages are required to be populated for merging them into a
     * large page.  Populate all child spte.
     */
    for (i = 0; i < SPTE_ENT_PER_PAGE; i++) {
        WARN_ON_ONCE(child_iter.level != PG_LEVEL_4K); //因为当前只支持 2MB 大页，所以子 SPTE 必然是 4KB
        if (is_shadow_present_pte(child_iter.old_spte)) { //对于已映射的子 SPTE
            /* TODO: relocate page for huge page. */
            WARN_ON_ONCE(spte_to_pfn(child_iter.old_spte) != spte_to_pfn(new_spte) + i);
            if (i < SPTE_ENT_PER_PAGE - 1)
                tdp_iter_step_side(&child_iter); //用提取出来的新函数侧移迭代器
            continue; //不停地侧移，直至移到一条未映射的子 SPTE
        }
        //设置这条尚未映射的子 SPTE
        WARN_ON_ONCE(is_private_zapped_spte(old_spte) && //SPTE 如果是临时被 blocked 的私有 SPTE 
                 spte_to_pfn(child_iter.old_spte) != spte_to_pfn(new_spte) + i); //按照设计新旧 SPTE 的 PFN 部分是一样的
        child_spte = make_huge_page_split_spte(kvm, new_spte, child_sp->role, i);//构造一个被拆分大页的子页 SPTE 的内容
        /*
         * Because other thread may have started to operate on this spte
         * before freezing the parent spte,  Use atomic version to
         * prevent race.
         */
        //因为其他线程可能开始在冻结其父 SPTE 前操作这条 SPTE，所以用原子的版本防止竞争
        ret = tdp_mmu_set_spte_atomic(vcpu->kvm, &child_iter, child_spte); //设置子页 SPTE 的内容为刚才构造的值
        if (ret == -EBUSY || ret == -EAGAIN)
            /*
             * There was a race condition.  Populate remaining 4K
             * spte to resolve fault->gfn to guarantee the forward
             * progress.
             */
            //有一个竞争条件。填充剩余的 4K SPTE 来解析 fault->gfn 来保证前面的过程
            ret_pf_retry = true;
        else if (ret)
            goto out;
        //迭代器侧移到下一条子页 SPTE
        if (i < SPTE_ENT_PER_PAGE - 1)
            tdp_iter_step_side(&child_iter);
    } //循环结束，处理完子页，该处理大页了
    if (ret_pf_retry) {
        ret = -EBUSY;
        goto out;
    }
    //调用 SEAMCALL<TDH.MEM.RANGE.BLOCK> 来阻止 Secure-EPT 条目被使用
    /* Prevent the Secure-EPT entry from being used. */
    ret = static_call(kvm_x86_zap_private_spte)(kvm, gfn, level);
    if (ret)
        goto out;
    kvm_flush_remote_tlbs_with_address(kvm, gfn, KVM_PAGES_PER_HPAGE(level)); //Flush TLB
    //调用 SEAMCALL<TDH.MEM.PAGE.PROMOTE> 将多个子页合并为一个大页
    /* Merge pages into a large page. */
    ret = static_call(kvm_x86_merge_private_spt)(kvm, gfn, level,
                             kvm_mmu_private_spt(child_sp));
    /*
     * Failed to merge pages because some pages are accepted and some are
     * pending.  Since the child page was mapped above, let vcpu run.
     */
    if (ret == -EAGAIN) //由于一些页面已被 accepted，一些还在 pending 导致失败
        ret = -EBUSY;   //因为子页面在上面已经映射了，让 vCPU 继续运行
    if (ret)
        goto unzap;
    //往 SPTE 写入新的大页的值，解除 SPTE 的映射
    /* Unfreeze spte. */
    __kvm_tdp_mmu_write_spte(sptep, new_spte);
    //释放无用的子 SP。在 TDX level 中的 Secure-EPT 页面已经由 kvm_x86_merge_private_spt 释放
    /*
     * Free unused child sp.  Secure-EPT page was already freed at TDX level
     * by kvm_x86_merge_private_spt().
     */
    tdp_unaccount_mmu_page(kvm, child_sp);
    tdp_mmu_free_sp(child_sp); //释放原来的子 SP 在内核中的相关数据
    return -EBUSY;             //这里有错吧，返回的难道不应该是 ret?!

unzap: //出错的情况，调用 SEAMCALL<TDH.MEM.RANGE.UNBLOCK> 来解除 Secure-EPT 条目的屏蔽
    if (static_call(kvm_x86_unzap_private_spte)(kvm, gfn, level))
        old_spte = __private_zapped_spte(old_spte); //构造临时被 blocked 的旧 SPTE 值
out:
    __kvm_tdp_mmu_write_spte(sptep, old_spte); //还原旧的 SPTE 值
    return ret;
}
```

### KVM: x86/tdp_mmu: TDX: Implement merge pages into a large page
```c
commit 6e43481574a00a5d6223e3955a65ca29a447d620
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Wed Oct 12 17:36:35 2022 -0700

    KVM: x86/tdp_mmu: TDX: Implement merge pages into a large page

    Implement merge_private_stp callback.
```
* 新增 SEAMCALL `TDH.MEM.PAGE.PROMOTE` 封装函数 `tdx_sept_merge_private_spt()`
* 新增 SEAMCALL `TDH.MEM.RANGE.UNBLOCK` 封装函数 `tdx_sept_unzap_private_spte()`
```cpp
static int tdx_sept_merge_private_spt(struct kvm *kvm, gfn_t gfn,
                      enum pg_level level, void *private_spt)
{
    int tdx_level = pg_level_to_tdx_sept_level(level);
    struct kvm_tdx *kvm_tdx = to_kvm_tdx(kvm);
    struct tdx_module_output out;
    gpa_t gpa = gfn_to_gpa(gfn);
    u64 err;

    /* See comment in tdx_sept_set_private_spte() */
    err = tdh_mem_page_promote(kvm_tdx->tdr_pa, gpa, tdx_level, &out);
    if (unlikely(err == TDX_ERROR_SEPT_BUSY))
        return -EAGAIN;
    if (seamcall_masked_status(err) == TDX_EPT_INVALID_PROMOTE_CONDITIONS)
        /*
         * Some pages are accepted, some pending.  Need to wait for TD
         * to accept all pages.  Tell it the caller.
         */
        return -EAGAIN;
    if (KVM_BUG_ON(err, kvm)) {
        pr_tdx_error(TDH_MEM_PAGE_PROMOTE, err, &out);
        return -EIO;
    }
    WARN_ON_ONCE(out.rcx != __pa(private_spt));
    //TDH.MEM.PAGE.PROMOTE 会释放低级别的 Secure-EPT 页面，flush cache 以重用
    /*
     * TDH.MEM.PAGE.PROMOTE frees the Secure-EPT page for the lower level.
     * Flush cache for reuse.
     */
    do {
        err = tdh_phymem_page_wbinvd(set_hkid_to_hpa(__pa(private_spt),
                                 to_kvm_tdx(kvm)->hkid));
   } while (err == (TDX_OPERAND_BUSY | TDX_OPERAND_ID_RCX));
    if (WARN_ON_ONCE(err)) {
        pr_tdx_error(TDH_PHYMEM_PAGE_WBINVD, err, NULL);
        return -EIO;
    }

    tdx_set_page_present(__pa(private_spt));
    return 0;
}
```
### KVM: x86/mmu: Make kvm fault handelr aware of large page of private memslot
* 让 KVM 缺页处理函数感知到私有 memslot 的大页
```c
commit fcde5c2c0f6908c00ea8326ead28e21ba1bf5f1d
Author: Isaku Yamahata <isaku.yamahata@intel.com>
Date:   Fri Jun 3 21:59:01 2022 -0700

    KVM: x86/mmu: Make kvm fault handelr aware of large page of private memslot

    struct kvm_page_fault.req_level is the page level which takes care of the
    faulted-in page size.  For now its calculation is only for the conventional
    kvm memslot by host_pfn_mapping_level() that traverses page table.

    However, host_pfn_mapping_level() cannot be used for private kvm memslot
    because pages of private kvm memlost aren't mapped into user virtual
    address space.  Instead page order is given when getting pfn.  Remember it
    in struct kvm_page_fault and use it.
```
* `struct kvm_page_fault.req_level` 是顾及 faulted-in 大小的页面级别，如今它的计算只是对传统的 KVM memslot，依靠 `host_pfn_mapping_level()`，它会遍历（host）页表
* 然而，`host_pfn_mapping_level()` 不能被用于私有 KVM memslot，因为私有 KVM memslot 的页面没有被映射到用户空间虚拟地址。与页面的阶来自 `PFN` 不同，把它记录在 `struct kvm_page_fault` 并使用它


#### 新增辅助函数 `kvm_is_faultin_private()`
* `kvm_is_faultin_private()` 用于判断 KVM 缺页是否是私有地址产生的缺页
```cpp
static inline bool kvm_is_faultin_private(const struct kvm_page_fault *fault)
{
    if (IS_ENABLED(CONFIG_HAVE_KVM_RESTRICTED_MEM))
        return fault->is_private && kvm_slot_can_be_private(fault->slot);
    return false;
}
```
* `kvm_faultin_pfn()` 的时候用新函数 `kvm_is_faultin_private()`
```diff
@@ -4338,7 +4348,7 @@ static int kvm_faultin_pfn(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault)
    if (fault->is_private != kvm_mem_is_private(vcpu->kvm, fault->gfn))
        return kvm_do_memory_fault_exit(vcpu, fault);

-   if (fault->is_private && kvm_slot_can_be_private(slot))
+   if (kvm_is_faultin_private(fault))
        return kvm_faultin_pfn_private(vcpu, fault);

    async = false;
```

#### `struct kvm_page_fault` 新增域 `host_level`
* `struct kvm_page_fault` 新增域 `host_level`，仅用于私有 memslot 和私有 `GFN`
```diff
diff --git a/arch/x86/kvm/mmu/mmu_internal.h b/arch/x86/kvm/mmu/mmu_internal.h
index b2774c164abb..1e73faad6268 100644
--- a/arch/x86/kvm/mmu/mmu_internal.h
+++ b/arch/x86/kvm/mmu/mmu_internal.h
@@ -337,6 +337,7 @@ struct kvm_page_fault {
        kvm_pfn_t pfn;
        hva_t hva;
        bool map_writable;
+       enum pg_level host_level; /* valid only for private memslot && private gfn */
 };

 int kvm_tdp_page_fault(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault);
```
* 处理私有地址缺页时调用私有页面专属的 faultin 函数 `kvm_faultin_pfn_private()`（见上一段），仅在该函数中给 `fault->host_level` 赋值
  * 原因 commit message 说了
* 对于私有页面，`order` 来自 `restrictedmem_get_page()`，是根据分到的页面的大小算出来的，因此不可能小于 `PG_LEVEL_NONE`
* 可见 `fault->host_level` 也会限制 `fault->max_level`
```diff
@@ -4294,7 +4303,8 @@ static int kvm_faultin_pfn_private(struct kvm_vcpu *vcpu,
    if (kvm_restricted_mem_get_pfn(slot, fault->gfn, &fault->pfn, &order))
        return RET_PF_RETRY;

-   fault->max_level = min(order_to_level(order), fault->max_level);
+   fault->host_level = order_to_level(order);
+   fault->max_level = min((u8)fault->host_level, fault->max_level);
    fault->map_writable = !(slot->flags & KVM_MEM_READONLY);
    return RET_PF_CONTINUE;
 }
```

#### 新增函数 `__kvm_mmu_max_mapping_level()`
* `kvm_mmu_max_mapping_level()` 改为对新函数 `__kvm_mmu_max_mapping_level()` 的封装，`__kvm_mmu_max_mapping_level()` 是在原来的 `kvm_mmu_max_mapping_level()` 的基础上做的修改
* 新增了 `host_level` 参数
```cpp
static int __kvm_mmu_max_mapping_level(struct kvm *kvm,
                       const struct kvm_memory_slot *slot,
                       gfn_t gfn, int max_level, int host_level,
                       bool faultin_private)
{
    struct kvm_lpage_info *linfo;
    //从当前传入的最大页面级别和 EPT 所能支持的最大巨页级别中选出最小值
    max_level = min(max_level, max_huge_page_level);
    for ( ; max_level > PG_LEVEL_4K; max_level--) { //比如说级别从 512G -> 1G -> 2M 级别依次遍历
        linfo = lpage_info_slot(gfn, slot, max_level); //根据 GFN 找到在 slot 上某个大页级别上对应的是否在该级别上允许大页的信息
        if (!linfo->disallow_lpage) //比如在 512G 级别上该 GFN 无法支持大页，则尝试 1G 大页，如果支持 1G 则结束遍历
            break;
    }
    //此时的大页级别是在 slot 上能支持的最大级别，比如能支持 1G 自然也就能支持 2M。但 4K 就没必要再往下调查了
    if (max_level == PG_LEVEL_4K)
        return PG_LEVEL_4K;
    //如果缺页不是发生在私有地址，根据 GFN 和 slot 找到 HVA，然后返回该 HVA 在 host 侧映射的级别
    if (!faultin_private) {
        WARN_ON_ONCE(host_level != PG_LEVEL_NONE);
        host_level = host_pfn_mapping_level(kvm, gfn, slot);
    }
    WARN_ON_ONCE(host_level == PG_LEVEL_NONE); //不管是否私有，此时 host_level 都不可能为 PG_LEVEL_NONE
    return min(host_level, max_level); //由此可见，此处 guest 所能支持的页面级别同时受限于其 host 侧的映射的页面级别
}

int kvm_mmu_max_mapping_level(struct kvm *kvm,
                  const struct kvm_memory_slot *slot, gfn_t gfn,
                  int max_level, bool faultin_private)
{
    return __kvm_mmu_max_mapping_level(kvm, slot, gfn, max_level, PG_LEVEL_NONE, faultin_private);
}
```

#### 修改 `kvm_mmu_hugepage_adjust()` 函数
* `kvm_mmu_hugepage_adjust()` 函数用于调整缺页的级别，调用考虑 `host_level` 的版本
```diff
diff --git a/arch/x86/kvm/mmu/mmu.c b/arch/x86/kvm/mmu/mmu.c
index 961e103e674a..dc767125922e 100644
--- a/arch/x86/kvm/mmu/mmu.c
+++ b/arch/x86/kvm/mmu/mmu.c
@@ -3162,9 +3170,10 @@ void kvm_mmu_hugepage_adjust(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault
     * Enforce the iTLB multihit workaround after capturing the requested
     * level, which will be used to do precise, accurate accounting.
     */
-   fault->req_level = kvm_mmu_max_mapping_level(vcpu->kvm, slot,
-                            fault->gfn, fault->max_level,
-                            fault->is_private);
+   fault->req_level = __kvm_mmu_max_mapping_level(vcpu->kvm, slot,
+                              fault->gfn, fault->max_level,
+                              fault->host_level,
+                              kvm_is_faultin_private(fault));
    if (fault->req_level == PG_LEVEL_4K || fault->huge_page_disallowed)
        return;
```

### KVM: TDX: Allow 2MB large page for TD GUEST
* 允许 TD Guest 用 2MB 大页
```c
commit db25cba9615f5dcf3651fa4b2e63efae17458bb4
Author: Xiaoyao Li <xiaoyao.li@intel.com>
Date:   Tue Aug 31 15:34:52 2021 +0800

    KVM: TDX: Allow 2MB large page for TD GUEST

    Now that everything is there to support 2MB page for TD guest.  Because TDX
    module TDH.MEM.PAGE.AUG supports 4KB page and 2MB page, set struct
    kvm_arch.tdp_max_page_level to 2MB page level.
```
* 万事俱备，只欠东风。因为 TDX module `TDH.MEM.PAGE.AUG` 支持 `4KB` 页和 `2MB` 页，设置 `struct kvm_arch.tdp_max_page_level` 为 `2MB` 的页面级别
```diff
diff --git a/arch/x86/kvm/vmx/tdx.c b/arch/x86/kvm/vmx/tdx.c
index 4ed76ef46b0d..3084fa846460 100644
--- a/arch/x86/kvm/vmx/tdx.c
+++ b/arch/x86/kvm/vmx/tdx.c
@@ -487,8 +487,8 @@ int tdx_vm_init(struct kvm *kvm)
         */
        kvm_mmu_set_mmio_spte_value(kvm, 0);

-       /* TODO: Enable 2mb and 1gb large page support. */
-       kvm->arch.tdp_max_page_level = PG_LEVEL_4K;
+       /* TDH.MEM.PAGE.AUG supports up to 2MB page. */
+       kvm->arch.tdp_max_page_level = PG_LEVEL_2M;

        /*
         * This function initializes only KVM software construct.  It doesn't
```
* 把 TDX KVM patchset 里加上的拆分私有大页的警告去掉
```diff
diff --git a/arch/x86/kvm/mmu/tdp_mmu.c b/arch/x86/kvm/mmu/tdp_mmu.c
index fc8f457292b9..7091c45697ef 100644
--- a/arch/x86/kvm/mmu/tdp_mmu.c
+++ b/arch/x86/kvm/mmu/tdp_mmu.c
@@ -1630,14 +1630,9 @@ int kvm_tdp_mmu_map(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault)

                sp->nx_huge_page_disallowed = fault->huge_page_disallowed;

-               if (is_shadow_present_pte(iter.old_spte)) {
-                       /*
-                        * TODO: large page support.
-                        * Doesn't support large page for TDX now
-                        */
-                       KVM_BUG_ON(is_private_sptep(iter.sptep), vcpu->kvm);
+               if (is_shadow_present_pte(iter.old_spte))
                        r = tdp_mmu_split_huge_page(kvm, &iter, sp, true);
-               } else
+               else
                        r = tdp_mmu_link_sp(kvm, &iter, sp, true);

                /*
```