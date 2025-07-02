# Guest-first Memory
* 引入几个新的 KVM uAPI，最终在 KVM 中创建 guest 优先的内存子系统，即 guest_memfd。
  * guest 优先内存允许 KVM 提供在通用内存子系统中难以实现或完全不可能实现的功能、增强功能和优化。
* guest_memfd 的核心 KVM `ioctl()` 是 `KVM_CREATE_GUEST_MEMFD`，它与通用 `memfd_create()` 类似，创建一个匿名文件并返回引用它的文件描述符。
  * 与“常规”memfd 文件一样，guest_memfd 文件位于 RAM 中，具有易失性存储，并且在删除最后一个引用时自动释放。
  * memfd 文件（以及所有其他内存子系统）之间的主要区别在于 guest_memfd 文件绑定到其所属的虚拟机，无法由用户空间映射、读取或写入，并且无法调整大小。
  * 然而，guest_memfd 文件确实支持 `PUNCH_HOLE`，它可用于在共享状态和 guest 私有状态之间转换 guest 内存区域。
* 第二个 KVM `ioctl()` `KVM_SET_MEMORY_ATTRIBUTES` 允许用户空间为给定的 guest 内存页指定属性。
  * 从长远来看，它可能会扩展为允许用户空间指定 per-gfn `RWX` 保护，包括允许内存在 guest 中可写，但在 host 用户空间中也可写。
* guest_memfd 的直接驱动用例是机密（CoCo）VM，特别是 AMD 的 SEV-SNP、Intel 的 TDX 和 KVM 自己的 pKVM。
  * 对于此类用例，能够将内存映射到 KVM guest 而不需要将所述内存映射到 host 是一项硬性要求。
  * 虽然 SEV+ 和 TDX 通过加密 guest 内存来防止不受信任的软件读取 guest 私有数据，但 pKVM 无需依赖内存加密即可提供机密性和完整性。
  * 此外，对于 SEV-SNP，尤其是 TDX，访问 guest 私有内存对于 host 来说可能是致命的，即 KVM 必须阻止 host 用户空间访问 guest 内存，而不管硬件行为如何。
* 从长远来看，guest_memfd 对于 CoCo VM 之外的用例可能很有用，例如强化用户空间以防止对 guest 内存的无意访问。
  * 如前所述，KVM 的 ABI 使用用户空间 VMA 保护来定义允许 guest 保护（授予映射 guest 内存可执行文件的例外），并且类似地，KVM 目前要求 guest 映射大小是 host 用户空间映射大小的严格子集。
  * 解耦映射大小将允许用户空间仅精确映射所需的内容并具有所需的权限，而不会影响 guest 性能。
* Guest 优先的内存子系统还为诸如专用内存池（用于 slice-of-hardware VMs）和消除“`struct page`”（用于用户空间 **永远不需要** 从 guest 内存进行 DMA 或进入 guest 内存的 offload 设置）等提供了更清晰的视线 。
* Guest_memfd 是 3 年多开发和探索的结果；在 KVM 中承担内存管理职责并不是支持 CoCo VM 的第一、第二甚至第三选择。
  * 但是，在多次尝试避免 KVM 特定的后备内存失败后，看看结果，很明显，在所有尝试过的方法中，guest_memfd 是最简单、最健壮、最可扩展的，也是 KVM 和整个内核正确的做法。
* 这个版本的“开发周期”将会非常短；理想情况下，下周我将按原样将其合并到 kvm/next 中，在合并窗口结束后立即将其通过 6.8 的 KVM 树。
  * 该系列仍然基于 6.6（加上 6.7 的 KVM 更改），因此需要对 6.7 中提交 0ede61d8589c（“file: convert to SLAB_TYPESAFE_BY_RCU”）引入的 `get_file_rcu()` 进行小修改。
  * 修复将作为合并提交的一部分完成，上面的大部分文本将成为合并的提交消息。
* 因此，在 v13 中仅有的两个具有实质性注释的（substantial remarks）提交（取决于您对实质性 substantial 的定义）*不* 正式属于本系列，并且不会被合并：
  * KVM: Prepare for handling only shared mappings in mmu_notifier events
  * KVM: Add transparent hugepage support for dedicated guest memory
* 合并后待完成的工作包括：
  - 考虑使用 restrictedmem 框架来管理 guest 内存
  - 引入一种测试机制来有毒内存，可能使用此处介绍的相同内存属性
  - SNP 和 TDX 支持
* 非 KVM 人员，您可能想要显式 ACK 埋藏在本系列中间的两个补丁：
  - fs: Rename anon_inode_getfile_secure() and anon_inode_getfd_secure()
  - mm: Add AS_UNMOVABLE to mark mapping as completely unmovable
* 请注意，添加 `AS_UNMOVABLE` 并不是严格要求的，因为它“只是”一种优化，但我们希望立即将其安装到位。

## [PATCH 01/34] KVM: Tweak kvm_hva_range and hva_handler_t to allow reusing for gfn ranges
* 重新设计 “`struct kvm_hva_range`” 并将其重命名为 “`kvm_mmu_notifier_range`”，以便该结构可用于处理在 gfn 上下文上操作的通知，即不绑定到 host 虚拟地址的通知。
  * 也重命名处理程序 `typedef`（可以说它应该始终是 `gfn_handler_t`）。
* 实际上，对于 64 位内核这是个空操作，因为唯一有意义的更改是将 `start + end` 存储为 `u64`，而不是 *无符号长整型*。
---
* 主要做的事情是改名和修改数据类型
```diff
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index 486800a7024b..0524933856d4 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -541,18 +541,22 @@ static inline struct kvm *mmu_notifier_to_kvm(struct mmu_notifier *mn)
    return container_of(mn, struct kvm, mmu_notifier);
 }

-typedef bool (*hva_handler_t)(struct kvm *kvm, struct kvm_gfn_range *range);
+typedef bool (*gfn_handler_t)(struct kvm *kvm, struct kvm_gfn_range *range);

 typedef void (*on_lock_fn_t)(struct kvm *kvm, unsigned long start,
                 unsigned long end);

 typedef void (*on_unlock_fn_t)(struct kvm *kvm);

-struct kvm_hva_range {
-   unsigned long start;
-   unsigned long end;
+struct kvm_mmu_notifier_range {
+   /*
+    * 64-bit addresses, as KVM notifiers can operate on host virtual
+    * addresses (unsigned long) and guest physical addresses (64-bit).
+    */
+   u64 start;
+   u64 end;
    union kvm_mmu_notifier_arg arg;
-   hva_handler_t handler;
+   gfn_handler_t handler;
    on_lock_fn_t on_lock;
    on_unlock_fn_t on_unlock;
    bool flush_on_ret;
@@ -581,7 +585,7 @@ static const union kvm_mmu_notifier_arg KVM_MMU_NOTIFIER_NO_ARG;
 	     node = interval_tree_iter_next(node, start, last))	     \
 
 static __always_inline int __kvm_handle_hva_range(struct kvm *kvm,
-						  const struct kvm_hva_range *range)
+						  const struct kvm_mmu_notifier_range *range)
 {
 	bool ret = false, locked = false;
 	struct kvm_gfn_range gfn_range;
@@ -608,9 +612,9 @@ static __always_inline int __kvm_handle_hva_range(struct kvm *kvm,
 			unsigned long hva_start, hva_end;
 
 			slot = container_of(node, struct kvm_memory_slot, hva_node[slots->node_idx]);
-			hva_start = max(range->start, slot->userspace_addr);
-			hva_end = min(range->end, slot->userspace_addr +
-						  (slot->npages << PAGE_SHIFT));
+			hva_start = max_t(unsigned long, range->start, slot->userspace_addr);
+			hva_end = min_t(unsigned long, range->end,
+					slot->userspace_addr + (slot->npages << PAGE_SHIFT));
 
 			/*
 			 * To optimize for the likely case where the address
@@ -660,10 +664,10 @@ static __always_inline int kvm_handle_hva_range(struct mmu_notifier *mn,
 						unsigned long start,
 						unsigned long end,
 						union kvm_mmu_notifier_arg arg,
-						hva_handler_t handler)
+						gfn_handler_t handler)
 {
 	struct kvm *kvm = mmu_notifier_to_kvm(mn);
-	const struct kvm_hva_range range = {
+	const struct kvm_mmu_notifier_range range = {
 		.start		= start,
 		.end		= end,
 		.arg		= arg,
@@ -680,10 +684,10 @@ static __always_inline int kvm_handle_hva_range(struct mmu_notifier *mn,
 static __always_inline int kvm_handle_hva_range_no_flush(struct mmu_notifier *mn,
 							 unsigned long start,
 							 unsigned long end,
-							 hva_handler_t handler)
+							 gfn_handler_t handler)
 {
 	struct kvm *kvm = mmu_notifier_to_kvm(mn);
-	const struct kvm_hva_range range = {
+	const struct kvm_mmu_notifier_range range = {
 		.start		= start,
 		.end		= end,
 		.handler	= handler,
@@ -771,7 +775,7 @@ static int kvm_mmu_notifier_invalidate_range_start(struct mmu_notifier *mn,
 					const struct mmu_notifier_range *range)
 {
 	struct kvm *kvm = mmu_notifier_to_kvm(mn);
-	const struct kvm_hva_range hva_range = {
+	const struct kvm_mmu_notifier_range hva_range = {
 		.start		= range->start,
 		.end		= range->end,
 		.handler	= kvm_unmap_gfn_range,
@@ -835,7 +839,7 @@ static void kvm_mmu_notifier_invalidate_range_end(struct mmu_notifier *mn,
 					const struct mmu_notifier_range *range)
 {
 	struct kvm *kvm = mmu_notifier_to_kvm(mn);
-	const struct kvm_hva_range hva_range = {
+	const struct kvm_mmu_notifier_range hva_range = {
 		.start		= range->start,
 		.end		= range->end,
 		.handler	= (void *)kvm_null_fn,
```

## [PATCH 02/34] KVM: Assert that mmu_invalidate_in_progress *never* goes negative
* 将正在进行的失效计数（invdalidation count）的 assertion 从 primary MMU 的 notifier 路径移至 KVM 的公共 notification 路径，例如，即使失效来自 KVM 本身，断言计数也不会变为负数。
* 机会性地将断言转换为 `KVM_BUG_ON()`，即仅终止受影响的虚拟机，而不是整个内核。
  * 损坏的计数对于虚拟机来说是致命的，例如，非零（负）计数将导致 `mmu_invalidate_retry()` 阻止任何安装新映射的尝试。
  * 但这远不能保证没有 `start()` 的 `end()` 是致命的，甚至对目标 VM 以外的任何其他东西都是有问题的，例如，潜在的错误可能只是对 `end()` 的重复调用。
  * 而且更有可能的是，错过的 invalidation（即潜在的 use-after-free）将表现为没有任何通知，而不是没有 `start()` 的 `end()` 。
---
```diff
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index 0524933856d4..5a97e6c7d9c2 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -833,6 +833,7 @@ void kvm_mmu_invalidate_end(struct kvm *kvm, unsigned long start,
 	 * in conjunction with the smp_rmb in mmu_invalidate_retry().
 	 */
 	kvm->mmu_invalidate_in_progress--;
+	KVM_BUG_ON(kvm->mmu_invalidate_in_progress < 0, kvm);
 }
 
 static void kvm_mmu_notifier_invalidate_range_end(struct mmu_notifier *mn,
@@ -863,8 +864,6 @@ static void kvm_mmu_notifier_invalidate_range_end(struct mmu_notifier *mn,
 	 */
 	if (wake)
 		rcuwait_wake_up(&kvm->mn_memslots_update_rcuwait);
-
-	BUG_ON(kvm->mmu_invalidate_in_progress < 0);
 }
 
 static int kvm_mmu_notifier_clear_flush_young(struct mmu_notifier *mn,
```
* `kvm_mmu_notifier_invalidate_range_end()` 是 KVM 注册到 MMU 的 notifier 的回调，最终会调用到 `kvm_mmu_invalidate_end()`
```cpp
kvm_mmu_notifier_invalidate_range_end()
-> __kvm_handle_hva_range()
   -> range->on_lock()
   => kvm_mmu_invalidate_end()
```
* `kvm_zap_gfn_range()` 也会调用 `kvm_mmu_invalidate_end()` 以及它对应的 `kvm_mmu_invalidate_begin()`
  * `kvm->mmu_invalidate_in_progress` 仅在 `kvm_mmu_invalidate_begin()` 和 `kvm_mmu_invalidate_end()` 中修改
  * 如果在这个路径下计数出错将没有任何通知
* 显然，将失效计数（invdalidation count）的 assertion 放入 KVM 的公共 notification 路径更合理

## [PATCH 03/34] KVM: Use gfn instead of hva for mmu_notifier_retry
* 当前在 mmu_notifier invalidate 路径中，记录了 `HVA` 范围，然后通过缺页处理路径中的 `mmu_invalidate_retry_hva()` 进行检查。然而，对于即将引入的私有内存，缺页错误可能没有关联的 `HVA`，检查 `GFN`（`GPA`）更有意义。
* 对于现有的基于 `HVA` 的共享内存，`GFN` 预计也可以工作。唯一的缺点是，当将多个 `gfns` 别名到单个 `HVA` 时，检查多个范围的当前算法可能会导致更大的范围被拒绝。 这种混叠应该不常见，因此预计影响很小。
---
* `struct kvm` 的 `mmu_invalidate_range_start` 和 `mmu_invalidate_range_end` 域的类型修改为 `gfn_t`
```diff
diff --git a/include/linux/kvm_host.h b/include/linux/kvm_host.h
index fb6c6109fdca..11d091688346 100644
--- a/include/linux/kvm_host.h
+++ b/include/linux/kvm_host.h
@@ -787,8 +787,8 @@ struct kvm {
    struct mmu_notifier mmu_notifier;
    unsigned long mmu_invalidate_seq;
    long mmu_invalidate_in_progress;
-   unsigned long mmu_invalidate_range_start;
-   unsigned long mmu_invalidate_range_end;
+   gfn_t mmu_invalidate_range_start;
+   gfn_t mmu_invalidate_range_end;
 #endif
    struct list_head devices;
    u64 manual_dirty_log_protect;
```
* `kvm_mmu_invalidate_begin()` 函数拆分成 `kvm_mmu_invalidate_begin()` 和 `kvm_mmu_invalidate_range_add()` 两个函数
```diff
diff --git a/include/linux/kvm_host.h b/include/linux/kvm_host.h
index fb6c6109fdca..11d091688346 100644
--- a/include/linux/kvm_host.h
+++ b/include/linux/kvm_host.h
@@ -1392,10 +1392,9 @@ void kvm_mmu_free_memory_cache(struct kvm_mmu_memory_cache *mc);
 void *kvm_mmu_memory_cache_alloc(struct kvm_mmu_memory_cache *mc);
 #endif

-void kvm_mmu_invalidate_begin(struct kvm *kvm, unsigned long start,
-                 unsigned long end);
-void kvm_mmu_invalidate_end(struct kvm *kvm, unsigned long start,
-               unsigned long end);
+void kvm_mmu_invalidate_begin(struct kvm *kvm);
+void kvm_mmu_invalidate_range_add(struct kvm *kvm, gfn_t start, gfn_t end);
+void kvm_mmu_invalidate_end(struct kvm *kvm);

 long kvm_arch_dev_ioctl(struct file *filp,
            unsigned int ioctl, unsigned long arg);
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index 5a97e6c7d9c2..9cc57b23ec81 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -742,16 +741,29 @@ static void kvm_mmu_notifier_change_pte(struct mmu_notifier *mn,
    kvm_handle_hva_range(mn, address, address + 1, arg, kvm_change_spte_gfn);
 }

-void kvm_mmu_invalidate_begin(struct kvm *kvm, unsigned long start,
-                 unsigned long end)
+void kvm_mmu_invalidate_begin(struct kvm *kvm)
 {
+   lockdep_assert_held_write(&kvm->mmu_lock);
    /*
     * The count increase must become visible at unlock time as no
     * spte can be established without taking the mmu_lock and
     * count is also read inside the mmu_lock critical section.
     */
    kvm->mmu_invalidate_in_progress++;
+
    if (likely(kvm->mmu_invalidate_in_progress == 1)) {
+       kvm->mmu_invalidate_range_start = INVALID_GPA;
+       kvm->mmu_invalidate_range_end = INVALID_GPA;
+   }
+}
+
+void kvm_mmu_invalidate_range_add(struct kvm *kvm, gfn_t start, gfn_t end)
+{
+   lockdep_assert_held_write(&kvm->mmu_lock);
+
+   WARN_ON_ONCE(!kvm->mmu_invalidate_in_progress);
+
+   if (likely(kvm->mmu_invalidate_range_start == INVALID_GPA)) {
        kvm->mmu_invalidate_range_start = start;
        kvm->mmu_invalidate_range_end = end;
    } else {
```
* `kvm_mmu_invalidate_range_add()` 会被 `kvm_mmu_unmap_gfn_range()` 和 `kvm_pre_set_memory_attributes()` 单独使用
* `kvm_mmu_unmap_gfn_range()` 是新引入的函数，对 `kvm_unmap_gfn_range()` 封装，加上对 invalidate 范围重叠的处理
```cpp
bool kvm_mmu_unmap_gfn_range(struct kvm *kvm, struct kvm_gfn_range *range)
{
    kvm_mmu_invalidate_range_add(kvm, range->start, range->end);
    return kvm_unmap_gfn_range(kvm, range);
}
```
* 将原来 `kvm_mmu_notifier_invalidate_range_start()` 的 notifier `kvm_mmu_notifier_range` 的 `.handler` 由 `kvm_unmap_gfn_range()` 改为新增的 `kvm_mmu_unmap_gfn_range()`
```diff
@@ -778,7 +796,7 @@ static int kvm_mmu_notifier_invalidate_range_start(struct mmu_notifier *mn,
    const struct kvm_mmu_notifier_range hva_range = {
        .start      = range->start,
        .end        = range->end,
-       .handler    = kvm_unmap_gfn_range,
+       .handler    = kvm_mmu_unmap_gfn_range,
        .on_lock    = kvm_mmu_invalidate_begin,
        .on_unlock  = kvm_arch_guest_memory_reclaimed,
        .flush_on_ret   = true,
```
* `kvm_mmu_invalidate_end()` 新增持锁检查和地址范围检查的警告
```diff
@@ -817,9 +835,10 @@ static int kvm_mmu_notifier_invalidate_range_start(struct mmu_notifier *mn,
    return 0;
 }

-void kvm_mmu_invalidate_end(struct kvm *kvm, unsigned long start,
-               unsigned long end)
+void kvm_mmu_invalidate_end(struct kvm *kvm)
 {
+   lockdep_assert_held_write(&kvm->mmu_lock);
+
    /*
     * This sequence increase will notify the kvm page fault that
     * the page that is going to be mapped in the spte could have
@@ -834,6 +853,12 @@ void kvm_mmu_invalidate_end(struct kvm *kvm, unsigned long start,
     */
    kvm->mmu_invalidate_in_progress--;
    KVM_BUG_ON(kvm->mmu_invalidate_in_progress < 0, kvm);
+
+   /*
+    * Assert that at least one range was added between start() and end().
+    * Not adding a range isn't fatal, but it is a KVM bug.
+    */
+   WARN_ON_ONCE(kvm->mmu_invalidate_range_start == INVALID_GPA);
 }

 static void kvm_mmu_notifier_invalidate_range_end(struct mmu_notifier *mn,
```
* `mmu_invalidate_retry_hva()` 改名为 `mmu_invalidate_retry_gfn()`，引发以下函数的修改：
  * `host_pfn_mapping_level()` 的注释，该函数根据 `GFN` 和 `slot` 找到 `HVA`，然后返回该 `HVA` 在 host 侧映射的级别
  * `is_page_fault_stale()` 在 EPT violation 的路径，所以在 invalidate private memory range 时，如果有 private memory 引发 EPT violation 发生会被防住
  * `vmx_set_apic_access_page_addr()` 用于设置 VM 的 APIC Access Page 的 `HPA` 到 VMCS
* `mmu_invalidate_retry_gfn()` 的修改如下，其实就增加了一个告警
```diff
@@ -1970,9 +1969,9 @@ static inline int mmu_invalidate_retry(struct kvm *kvm, unsigned long mmu_seq)
    return 0;
 }

-static inline int mmu_invalidate_retry_hva(struct kvm *kvm,
+static inline int mmu_invalidate_retry_gfn(struct kvm *kvm,
                       unsigned long mmu_seq,
-                      unsigned long hva)
+                      gfn_t gfn)
 {
    lockdep_assert_held(&kvm->mmu_lock);
    /*
@@ -1981,10 +1980,20 @@ static inline int mmu_invalidate_retry_hva(struct kvm *kvm,
     * that might be being invalidated. Note that it may include some false
     * positives, due to shortcuts when handing concurrent invalidations.
     */
-   if (unlikely(kvm->mmu_invalidate_in_progress) &&
-       hva >= kvm->mmu_invalidate_range_start &&
-       hva < kvm->mmu_invalidate_range_end)
-       return 1;
+   if (unlikely(kvm->mmu_invalidate_in_progress)) {
+       /*
+        * Dropping mmu_lock after bumping mmu_invalidate_in_progress
+        * but before updating the range is a KVM bug.
+        */
+       if (WARN_ON_ONCE(kvm->mmu_invalidate_range_start == INVALID_GPA ||
+                kvm->mmu_invalidate_range_end == INVALID_GPA))
+           return 1;
+
+       if (gfn >= kvm->mmu_invalidate_range_start &&
+           gfn < kvm->mmu_invalidate_range_end)
+           return 1;
+   }
+
    if (kvm->mmu_invalidate_seq != mmu_seq)
        return 1;
    return 0;
```
* 还有就是函数指针 `on_lock_fn_t` 的参数修改引发的修改：
```diff
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index 5a97e6c7d9c2..9cc57b23ec81 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -543,9 +543,7 @@ static inline struct kvm *mmu_notifier_to_kvm(struct mmu_notifier *mn)

 typedef bool (*gfn_handler_t)(struct kvm *kvm, struct kvm_gfn_range *range);

-typedef void (*on_lock_fn_t)(struct kvm *kvm, unsigned long start,
-                unsigned long end);
-
+typedef void (*on_lock_fn_t)(struct kvm *kvm);
 typedef void (*on_unlock_fn_t)(struct kvm *kvm);

 struct kvm_mmu_notifier_range {
@@ -637,7 +635,8 @@ static __always_inline int __kvm_handle_hva_range(struct kvm *kvm,
                locked = true;
                KVM_MMU_LOCK(kvm);
                if (!IS_KVM_NULL_FN(range->on_lock))
-                   range->on_lock(kvm, range->start, range->end);
+                   range->on_lock(kvm);
+
                if (IS_KVM_NULL_FN(range->handler))
                    break;
            }
```
## [PATCH 04/34] KVM: WARN if there are dangling MMU invalidations at VM destruction
* 添加一个断言，表明在销毁 VM 时不存在正在进行的 MMU 失效，但 KVM 在 `.invalidate_range_start()` 调用和相应的 `.invalidate_range_end()` 调用之间 *取消注册其 MMU notifier* 的情况除外。
* 由于上述异常豁免，KVM 无法检测来自 mmu_notifier 的未配对调用，但断言可以检测 KVM 错误，例如，这样 *几乎* 逃脱了最初 guest_memfd 开发的错误。
---
* 如果计数没有不平衡，即 KVM 没有在 `start()` 和 `end()` 之间取消注册其 MMU notifier，那么不应该有任何正在进行的失效（意即不应该触发警告）。
```diff
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index 9cc57b23ec81..5422ce20dcba 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -1358,9 +1358,16 @@ static void kvm_destroy_vm(struct kvm *kvm)
     * No threads can be waiting in kvm_swap_active_memslots() as the
     * last reference on KVM has been dropped, but freeing
     * memslots would deadlock without this manual intervention.
+    *
+    * If the count isn't unbalanced, i.e. KVM did NOT unregister its MMU
+    * notifier between a start() and end(), then there shouldn't be any
+    * in-progress invalidations.
     */
    WARN_ON(rcuwait_active(&kvm->mn_memslots_update_rcuwait));
-   kvm->mn_active_invalidate_count = 0;
+   if (kvm->mn_active_invalidate_count)
+       kvm->mn_active_invalidate_count = 0;
+   else
+       WARN_ON(kvm->mmu_invalidate_in_progress);
 #else
    kvm_flush_shadow_all(kvm);
 #endif
```
* `kvm->mn_active_invalidate_count` 仅在 `kvm_mmu_notifier_invalidate_range_start()` 时增加，`kvm_mmu_notifier_invalidate_range_end()` 时减少，这两个函数是 primary MMU 的 notifier 路径，如果销毁 VM 时发现
  * 计数 `kvm->mn_active_invalidate_count` 不为零，说明 KVM 在 `start()` 和 `end()` 之间取消注册其 MMU notifier 的行为，这并不算错误，把它归零
  * 计数 `kvm->mn_active_invalidate_count` 为零，但 `kvm->mmu_invalidate_in_progress` 不为零，说明 KVM 存在 bug，发出警告

## [PATCH 05/34] KVM: PPC: Drop dead code related to KVM_ARCH_WANT_MMU_NOTIFIER
* 断言启用 KVM 时同时定义了 `KVM_ARCH_WANT_MMU_NOTIFIER` 和 `CONFIG_MMU_NOTIFIER`，并为 `CONFIG_KVM_BOOK3S_HV_POSSIBLE=n` 路径无条件返回“`1`”。
  * 所有类型的 PPC 对 KVM 的支持都选择 MMU_NOTIFIER，并且 `KVM_ARCH_WANT_MMU_NOTIFIER` 由 arch/powerpc/include/asm/kvm_host.h 无条件定义。
* 有效地放弃使用 `KVM_ARCH_WANT_MMU_NOTIFIER` 将简化未来的清理工作，将 `KVM_ARCH_WANT_MMU_NOTIFIER` 转换为 Kconfig，即允许将组合所有
```c
#if defined(CONFIG_MMU_NOTIFIER) && defined(KVM_ARCH_WANT_MMU_NOTIFIER)
```
转为单一的
```c
#ifdef CONFIG_KVM_GENERIC_MMU_NOTIFIER
```
* 不必担心 PPC 对 `KVM_ARCH_WANT_MMU_NOTIFIER` 的“bare”使用。
---
* 其实是在维持原意，因为 `defined(KVM_ARCH_WANT_MMU_NOTIFIER)` 被移除了
```diff
diff --git a/arch/powerpc/kvm/powerpc.c b/arch/powerpc/kvm/powerpc.c
index 7197c8256668..b0a512ede764 100644
--- a/arch/powerpc/kvm/powerpc.c
+++ b/arch/powerpc/kvm/powerpc.c
@@ -632,12 +632,13 @@ int kvm_vm_ioctl_check_extension(struct kvm *kvm, long ext)
        break;
 #endif
    case KVM_CAP_SYNC_MMU:
+#if !defined(CONFIG_MMU_NOTIFIER) || !defined(KVM_ARCH_WANT_MMU_NOTIFIER)
+       BUILD_BUG();
+#endif
 #ifdef CONFIG_KVM_BOOK3S_HV_POSSIBLE
        r = hv_enabled;
-#elif defined(KVM_ARCH_WANT_MMU_NOTIFIER)
-       r = 1;
 #else
-       r = 0;
+       r = 1;
 #endif
        break;
 #ifdef CONFIG_KVM_BOOK3S_HV_POSSIBLE
```
## [PATCH 06/34] KVM: PPC: Return '1' unconditionally for KVM_CAP_SYNC_MMU
* 通告 KVM 的 MMU 与所有类型的 PPC KVM 支持的 primary MMU 同步，即当 `CONFIG_KVM_BOOK3S_HV_POSSIBLE=y` 但 VM 未使用 hypervisor 模式（也称为 PR VM）时，通告 MMU 已同步。
* PR VM 通过 `kvm_unmap_gfn_range_pr()` 对 mmu_notifier 失效事件执行正确的操作，
  * 更明显的是，当 `CONFIG_KVM_BOOK3S_HV_POSSIBLE=n` 和 `CONFIG_KVM_BOOK3S_PR_POSSIBLE=y` 时，KVM 会为 `KVM_CAP_SYNC_MMU` 返回“`1`”，
    * 例如，KVM 已经为 PR VM 通告了同步 MMU，只是没有在 `CONFIG_KVM_BOOK3S_HV_POSSIBLE=y` 时。
---
```diff
diff --git a/arch/powerpc/kvm/powerpc.c b/arch/powerpc/kvm/powerpc.c
index b0a512ede764..8d3ec483bc2b 100644
--- a/arch/powerpc/kvm/powerpc.c
+++ b/arch/powerpc/kvm/powerpc.c
@@ -635,11 +635,7 @@ int kvm_vm_ioctl_check_extension(struct kvm *kvm, long ext)
 #if !defined(CONFIG_MMU_NOTIFIER) || !defined(KVM_ARCH_WANT_MMU_NOTIFIER)
        BUILD_BUG();
 #endif
-#ifdef CONFIG_KVM_BOOK3S_HV_POSSIBLE
-       r = hv_enabled;
-#else
        r = 1;
-#endif
        break;
 #ifdef CONFIG_KVM_BOOK3S_HV_POSSIBLE
    case KVM_CAP_PPC_HTAB_FD:
```

## [PATCH 07/34] KVM: Convert KVM_ARCH_WANT_MMU_NOTIFIER to CONFIG_KVM_GENERIC_MMU_NOTIFIER
* 将 `KVM_ARCH_WANT_MMU_NOTIFIER` 转换为 Kconfig 并在适当的情况下选择它以有效维护现有行为。
  * 使用正确的 Kconfig 将简化在 KVM 的 mmu_notifier 基础架构之上构建更多功能。
* 将 `kvm_gfn_range` 的前向声明添加到 kvm_types.h，以便包含带有 `CONFIG_KVM=n` 的 arch/powerpc/include/asm/kvm_ppc.h 不会因 `kvm_gfn_range` 未声明而生成警告。
  * PPC 定义了 PR 与 HV 的钩子，而不通过 `#ifdefery` 来保护它们，例如：
```cpp
  bool (*unmap_gfn_range)(struct kvm *kvm, struct kvm_gfn_range *range);
  bool (*age_gfn)(struct kvm *kvm, struct kvm_gfn_range *range);
  bool (*test_age_gfn)(struct kvm *kvm, struct kvm_gfn_range *range);
  bool (*set_spte_gfn)(struct kvm *kvm, struct kvm_gfn_range *range);
```
* 或者，PPC 可以前向声明 `kvm_gfn_range`，但没有充分的理由不在通用 KVM 中定义它。
---
```diff
diff --git a/arch/powerpc/kvm/powerpc.c b/arch/powerpc/kvm/powerpc.c
index 8d3ec483bc2b..aac75c98a956 100644
--- a/arch/powerpc/kvm/powerpc.c
+++ b/arch/powerpc/kvm/powerpc.c
@@ -632,9 +632,7 @@ int kvm_vm_ioctl_check_extension(struct kvm *kvm, long ext)
        break;
 #endif
    case KVM_CAP_SYNC_MMU:
-#if !defined(CONFIG_MMU_NOTIFIER) || !defined(KVM_ARCH_WANT_MMU_NOTIFIER)
-       BUILD_BUG();
-#endif
+       BUILD_BUG_ON(!IS_ENABLED(CONFIG_KVM_GENERIC_MMU_NOTIFIER));
        r = 1;
        break;
 #ifdef CONFIG_KVM_BOOK3S_HV_POSSIBLE
diff --git a/arch/x86/kvm/Kconfig b/arch/x86/kvm/Kconfig
index 950c12868d30..e61383674c75 100644
--- a/arch/x86/kvm/Kconfig
+++ b/arch/x86/kvm/Kconfig
@@ -24,7 +24,7 @@ config KVM
    depends on HIGH_RES_TIMERS
    depends on X86_LOCAL_APIC
    select PREEMPT_NOTIFIERS
-   select MMU_NOTIFIER
+   select KVM_GENERIC_MMU_NOTIFIER
    select HAVE_KVM_IRQCHIP
    select HAVE_KVM_PFNCACHE
    select HAVE_KVM_IRQFD
diff --git a/include/linux/kvm_host.h b/include/linux/kvm_host.h
index 11d091688346..5faba69403ac 100644
--- a/include/linux/kvm_host.h
+++ b/include/linux/kvm_host.h
@@ -253,7 +253,7 @@ bool kvm_setup_async_pf(struct kvm_vcpu *vcpu, gpa_t cr2_or_gpa,
 int kvm_async_pf_wakeup_all(struct kvm_vcpu *vcpu);
 #endif

-#ifdef KVM_ARCH_WANT_MMU_NOTIFIER
+#ifdef CONFIG_KVM_GENERIC_MMU_NOTIFIER
 union kvm_mmu_notifier_arg {
    pte_t pte;
 };
@@ -783,7 +783,7 @@ struct kvm {
    struct hlist_head irq_ack_notifier_list;
 #endif

-#if defined(CONFIG_MMU_NOTIFIER) && defined(KVM_ARCH_WANT_MMU_NOTIFIER)
+#ifdef CONFIG_KVM_GENERIC_MMU_NOTIFIER
    struct mmu_notifier mmu_notifier;
    unsigned long mmu_invalidate_seq;
    long mmu_invalidate_in_progress;
@@ -1946,7 +1946,7 @@ extern const struct _kvm_stats_desc kvm_vm_stats_desc[];
 extern const struct kvm_stats_header kvm_vcpu_stats_header;
 extern const struct _kvm_stats_desc kvm_vcpu_stats_desc[];

-#if defined(CONFIG_MMU_NOTIFIER) && defined(KVM_ARCH_WANT_MMU_NOTIFIER)
+#ifdef CONFIG_KVM_GENERIC_MMU_NOTIFIER
 static inline int mmu_invalidate_retry(struct kvm *kvm, unsigned long mmu_seq)
 {
    if (unlikely(kvm->mmu_invalidate_in_progress))
diff --git a/include/linux/kvm_types.h b/include/linux/kvm_types.h
index 6f4737d5046a..9d1f7835d8c1 100644
--- a/include/linux/kvm_types.h
+++ b/include/linux/kvm_types.h
@@ -6,6 +6,7 @@
 struct kvm;
 struct kvm_async_pf;
 struct kvm_device_ops;
+struct kvm_gfn_range;
 struct kvm_interrupt;
 struct kvm_irq_routing_table;
 struct kvm_memory_slot;
diff --git a/virt/kvm/Kconfig b/virt/kvm/Kconfig
index 484d0873061c..ecae2914c97e 100644
--- a/virt/kvm/Kconfig
+++ b/virt/kvm/Kconfig
@@ -92,3 +92,7 @@ config HAVE_KVM_PM_NOTIFIER

 config KVM_GENERIC_HARDWARE_ENABLING
        bool
+
+config KVM_GENERIC_MMU_NOTIFIER
+       select MMU_NOTIFIER
+       bool
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index 5422ce20dcba..dc81279ea385 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -535,7 +535,7 @@ void kvm_destroy_vcpus(struct kvm *kvm)
 }
 EXPORT_SYMBOL_GPL(kvm_destroy_vcpus);

-#if defined(CONFIG_MMU_NOTIFIER) && defined(KVM_ARCH_WANT_MMU_NOTIFIER)
+#ifdef CONFIG_KVM_GENERIC_MMU_NOTIFIER
 static inline struct kvm *mmu_notifier_to_kvm(struct mmu_notifier *mn)
 {
    return container_of(mn, struct kvm, mmu_notifier);
@@ -962,14 +962,14 @@ static int kvm_init_mmu_notifier(struct kvm *kvm)
    return mmu_notifier_register(&kvm->mmu_notifier, current->mm);
 }

-#else  /* !(CONFIG_MMU_NOTIFIER && KVM_ARCH_WANT_MMU_NOTIFIER) */
+#else  /* !CONFIG_KVM_GENERIC_MMU_NOTIFIER */

 static int kvm_init_mmu_notifier(struct kvm *kvm)
 {
    return 0;
 }

-#endif /* CONFIG_MMU_NOTIFIER && KVM_ARCH_WANT_MMU_NOTIFIER */
+#endif /* CONFIG_KVM_GENERIC_MMU_NOTIFIER */

 #ifdef CONFIG_HAVE_KVM_PM_NOTIFIER
 static int kvm_pm_notifier_call(struct notifier_block *bl,
@@ -1289,7 +1289,7 @@ static struct kvm *kvm_create_vm(unsigned long type, const char *fdname)
 out_err_no_debugfs:
    kvm_coalesced_mmio_free(kvm);
 out_no_coalesced_mmio:
-#if defined(CONFIG_MMU_NOTIFIER) && defined(KVM_ARCH_WANT_MMU_NOTIFIER)
+#ifdef CONFIG_KVM_GENERIC_MMU_NOTIFIER
    if (kvm->mmu_notifier.ops)
        mmu_notifier_unregister(&kvm->mmu_notifier, current->mm);
 #endif
@@ -1349,7 +1349,7 @@ static void kvm_destroy_vm(struct kvm *kvm)
        kvm->buses[i] = NULL;
    }
    kvm_coalesced_mmio_free(kvm);
-#if defined(CONFIG_MMU_NOTIFIER) && defined(KVM_ARCH_WANT_MMU_NOTIFIER)
+#ifdef CONFIG_KVM_GENERIC_MMU_NOTIFIER
    mmu_notifier_unregister(&kvm->mmu_notifier, kvm->mm);
    /*
     * At this point, pending calls to invalidate_range_start()
```

## [PATCH 08/34] KVM: Introduce KVM_SET_USER_MEMORY_REGION2
* 引入 `KVM_SET_USER_MEMORY_REGION` 的“version 2”，以便可以提供附加信息，而不会导致用户空间失败。
* 新的 `kvm_userspace_memory_region2` 结构中的填充将用于传递除 `userspace_addr` 之外的 *文件描述符*，即允许用户空间指向文件描述符并将内存映射到 guest 中，却 **不** 映射到 host 用户空间中。
* 或者，KVM 可以简单地添加“`struct kvm_userspace_memory_region2`”而无需新的 `ioctl()`，但正如 Paolo 指出的那样，添加新的 `ioctl()` 可以使对坏标志的检测更加健壮。
  * 例如，如果新的 `fd` 域仅由标志而不是新的 `ioctl()` 保护，则用户空间错误（设置“坏”标志）将生成 *越界访问* 而不是 `-EINVAL` 错误。
---
* 引入 `struct kvm_userspace_memory_region2` 结构的定义，但还未加入 `guest_memfd_offset` 和 `guest_memfd` 域
```diff
diff --git a/include/uapi/linux/kvm.h b/include/uapi/linux/kvm.h
index 211b86de35ac..308cc70bd6ab 100644
--- a/include/uapi/linux/kvm.h
+++ b/include/uapi/linux/kvm.h
@@ -95,6 +95,16 @@ struct kvm_userspace_memory_region {
    __u64 userspace_addr; /* start of the userspace allocated memory */
 };

+/* for KVM_SET_USER_MEMORY_REGION2 */
+struct kvm_userspace_memory_region2 {
+   __u32 slot;
+   __u32 flags;
+   __u64 guest_phys_addr;
+   __u64 memory_size;
+   __u64 userspace_addr;
+   __u64 pad[16];
+};
+
```
* 引入 KVM 能力 `KVM_CAP_USER_MEMORY2`
```diff
@@ -1201,6 +1211,7 @@ struct kvm_ppc_resize_hpt {
 #define KVM_CAP_ARM_EAGER_SPLIT_CHUNK_SIZE 228
 #define KVM_CAP_ARM_SUPPORTED_BLOCK_SIZES 229
 #define KVM_CAP_ARM_SUPPORTED_REG_MASK_RANGES 230
+#define KVM_CAP_USER_MEMORY2 231

 #ifdef KVM_CAP_IRQ_ROUTING
```
* 引入 `ioctl()` 命令 `KVM_SET_USER_MEMORY_REGION2`，以下是该值生成的过程：
```cpp
//include/uapi/asm-generic/ioctl.h
#define _IOC_NRBITS 8
#define _IOC_TYPEBITS   8
...
#define _IOC_NRSHIFT    0
#define _IOC_TYPESHIFT  (_IOC_NRSHIFT+_IOC_NRBITS)
#define _IOC_SIZESHIFT  (_IOC_TYPESHIFT+_IOC_TYPEBITS)
#define _IOC_DIRSHIFT   (_IOC_SIZESHIFT+_IOC_SIZEBITS)
...
#define _IOC(dir,type,nr,size) \
    (((dir)  << _IOC_DIRSHIFT) | \
     ((type) << _IOC_TYPESHIFT) | \
     ((nr)   << _IOC_NRSHIFT) | \
     ((size) << _IOC_SIZESHIFT))

#ifndef __KERNEL__
#define _IOC_TYPECHECK(t) (sizeof(t))
#endif
...
#define _IOW(type,nr,size)  _IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
...
//include/uapi/linux/kvm.h
#define KVM_SET_USER_MEMORY_REGION2 _IOW(KVMIO, 0x49, \
                     struct kvm_userspace_memory_region2)
```
* 新增健全性检查宏 `SANITY_CHECK_MEM_REGION_FIELD(field)`，要求：
  * `struct kvm_userspace_memory_region` 中的域的偏移与 `struct kvm_userspace_memory_region2` 中的域的偏移必须相同
  * `struct kvm_userspace_memory_region` 中的域的大小与 `struct kvm_userspace_memory_region2` 中的域的大小必须相同
```cpp
#define SANITY_CHECK_MEM_REGION_FIELD(field)                    \
do {                                        \
    BUILD_BUG_ON(offsetof(struct kvm_userspace_memory_region, field) !=     \
             offsetof(struct kvm_userspace_memory_region2, field)); \
    BUILD_BUG_ON(sizeof_field(struct kvm_userspace_memory_region, field) !=     \
             sizeof_field(struct kvm_userspace_memory_region2, field)); \
} while (0)
```
* 引入新的标志 `KVM_SET_USER_MEMORY_REGION_V1_FLAGS`
  * 不访问 `struct kvm_userspace_memory_region2` 任何额外空间的标志。`KVM_SET_USER_MEMORY_REGION_V1_FLAGS` 仅允许这些。
```cpp
/*
 * Flags that do not access any of the extra space of struct
 * kvm_userspace_memory_region2.  KVM_SET_USER_MEMORY_REGION_V1_FLAGS
 * only allows these.
 */
#define KVM_SET_USER_MEMORY_REGION_V1_FLAGS \
    (KVM_MEM_LOG_DIRTY_PAGES | KVM_MEM_READONLY)
```
* 在 KVM 的 `ioctl()` 函数 `kvm_vm_ioctl()` 增加对 `KVM_SET_USER_MEMORY_REGION2` 命令的支持
  * 针对不同命令对大小的计算不一样
  * 用 `SANITY_CHECK_MEM_REGION_FIELD()` 确保两种结构的通用部分是一致的
  * 设置 `KVM_SET_USER_MEMORY_REGION` 又不设置 `KVM_SET_USER_MEMORY_REGION_V1_FLAGS` 标志是无效的设置
```diff
@@ -4845,15 +4862,39 @@ static long kvm_vm_ioctl(struct file *filp,
        r = kvm_vm_ioctl_enable_cap_generic(kvm, &cap);
        break;
    }
+   case KVM_SET_USER_MEMORY_REGION2:
    case KVM_SET_USER_MEMORY_REGION: {
-       struct kvm_userspace_memory_region kvm_userspace_mem;
+       struct kvm_userspace_memory_region2 mem;
+       unsigned long size;
+
+       if (ioctl == KVM_SET_USER_MEMORY_REGION) {
+           /*
+            * Fields beyond struct kvm_userspace_memory_region shouldn't be
+            * accessed, but avoid leaking kernel memory in case of a bug.
+            */
+           memset(&mem, 0, sizeof(mem));
+           size = sizeof(struct kvm_userspace_memory_region);
+       } else {
+           size = sizeof(struct kvm_userspace_memory_region2);
+       }
+
+       /* Ensure the common parts of the two structs are identical. */
+       SANITY_CHECK_MEM_REGION_FIELD(slot);
+       SANITY_CHECK_MEM_REGION_FIELD(flags);
+       SANITY_CHECK_MEM_REGION_FIELD(guest_phys_addr);
+       SANITY_CHECK_MEM_REGION_FIELD(memory_size);
+       SANITY_CHECK_MEM_REGION_FIELD(userspace_addr);

        r = -EFAULT;
-       if (copy_from_user(&kvm_userspace_mem, argp,
-                       sizeof(kvm_userspace_mem)))
+       if (copy_from_user(&mem, argp, size))
+           goto out;
+
+       r = -EINVAL;
+       if (ioctl == KVM_SET_USER_MEMORY_REGION &&
+           (mem.flags & ~KVM_SET_USER_MEMORY_REGION_V1_FLAGS))
            goto out;

-       r = kvm_vm_ioctl_set_memory_region(kvm, &kvm_userspace_mem);
+       r = kvm_vm_ioctl_set_memory_region(kvm, &mem);
        break;
    }
    case KVM_GET_DIRTY_LOG: {
```
* `ioctl()` 的 `KVM_CHECK_EXTENSION` 命令加入对 `KVM_CAP_USER_MEMORY2` 能力的支持

```diff
@@ -4568,6 +4576,7 @@ static int kvm_vm_ioctl_check_extension_generic(struct kvm *kvm, long arg)
 {
    switch (arg) {
    case KVM_CAP_USER_MEMORY:
+   case KVM_CAP_USER_MEMORY2:
    case KVM_CAP_DESTROY_MEMORY_REGION_WORKS:
    case KVM_CAP_JOIN_MEMORY_REGIONS_WORKS:
    case KVM_CAP_INTERNAL_ERROR_DATA:
```
* 以下函数受影响，改用 `struct kvm_userspace_memory_region2`
  * `check_memory_region_flags()`
  * `__kvm_set_memory_region()`
  * `kvm_set_memory_region()`
  * `__x86_set_memory_region()`

## [PATCH 09/34] KVM: Add KVM_EXIT_MEMORY_FAULT exit to report faults to userspace
* 添加新的 KVM exit 类型，以允许用户空间处理 KVM 无法解决的内存 fault，但用户空间 *可能* 能够处理（无需终止 guest 系统）。
* KVM 最初将使用 `KVM_EXIT_MEMORY_FAULT` 来报告私有内存和共享内存之间的隐式转换。
* 对于 guest 私有内存，将有两种内存转换：
  - 显式转换：当 guest 显式调用 KVM 来映射一个范围（私有或共享）时发生
  - 隐式转换：当 guest 尝试访问配置为“错误”状态（私有与共享）的 GFN 时发生
* 在 x86（第一个支持 guest 私有内存的体系结构）上，将通过 `KVM_EXIT_HYPERCALL + KVM_HC_MAP_GPA_RANGE` 报告显式转换，
  * 但报告 `KVM_EXIT_HYPERCALL` 进行隐式转换是不可取的，因为（显然）没有 hypercall，并且不能保证 guest 确实打算这在私有和共享之间进行转换，例如，KVM 认为的隐式转换“请求”实际上可能是 guest 代码错误的结果。
* `KVM_EXIT_MEMORY_FAULT` 将用于报告似乎是隐式转换的内存 faults。
* **注意**：为了允许将来 KVM 报告 `KVM_EXIT_MEMORY_FAULT` 并在任何未解决的 fault 上填充 `run->memory_fault`，KVM 返回“`-EFAULT`”（从用户空间的角度来看 `-1`，`errno == EFAULT`），而不是“`0`”！
  * 由于 KVM 中的历史包袱，从深度调用栈中以“`0`”退出到用户空间，例如，在模拟路径中，这是不可行的，因为这样做需要对 KVM 进行近乎彻底的检修，而 KVM 已经将 `-errno` 返回代码传播到用户空间，即使 `-errno` 源自低级 helper 也是如此。
* 报告 `gpa + size` 而不是单个 `gfn`，即使初始使用情况期望始终报告单个页面。
  * KVM 有朝一日完全有可能支持子页面粒度错误，甚至可能如此。
  * Intel 的子页面保护功能可实现 `128` 字节粒度的额外保护。
---
### `KVM_CAP_MEMORY_FAULT_INFO`
* 添加一个 KVM 能力 `KVM_CAP_MEMORY_FAULT_INFO`
* 此功能的存在表明，如果 KVM 无法解决 guest page fault 而 VM-Exit，例如，`KVM_RUN` 将填充 `kvm_run.memory_fault`。
  * 如果存在有效的 memslot，但没有对应 host 虚拟地址的后备 VMA。
  * 当且仅当 `KVM_RUN` 返回错误且 `errno=EFAULT` 或 `errno=EHWPOISON` *且* `kvm_run.exit_reason` 设置为 `KVM_EXIT_MEMORY_FAULT` 时，`kvm_run.memory_fault` 中的信息才有效。
* 注意：尝试解决内存 fault 以便他们可以重试 `KVM_RUN` 的用户空间（程序），鼓励它防止重复地接收相同的错误/annotated fault。
* include/uapi/linux/kvm.h
```cpp
#define KVM_CAP_MEMORY_FAULT_INFO 232
```
* 在 x86 的 check extension `ioctl()` 中加上对该能力的支持
```diff
diff --git a/arch/x86/kvm/x86.c b/arch/x86/kvm/x86.c
index 7b389f27dffc..8f9d8939b63b 100644
--- a/arch/x86/kvm/x86.c
+++ b/arch/x86/kvm/x86.c
@@ -4625,6 +4625,7 @@ int kvm_vm_ioctl_check_extension(struct kvm *kvm, long ext)
    case KVM_CAP_ENABLE_CAP:
    case KVM_CAP_VM_DISABLE_NX_HUGE_PAGES:
    case KVM_CAP_IRQFD_RESAMPLE:
+   case KVM_CAP_MEMORY_FAULT_INFO:
        r = 1;
        break;
    case KVM_CAP_EXIT_HYPERCALL:
```

### `KVM_EXIT_MEMORY_FAULT`
* 增加 KVM 退出原因 `KVM_EXIT_MEMORY_FAULT`
* include/uapi/linux/kvm.h
```cpp
#define KVM_EXIT_MEMORY_FAULT     39
```
* 在 `struct kvm_run` 结构体中增加 `memory_fault` 联合体成员以描述 `KVM_EXIT_MEMORY_FAULT` 的退出信息
```diff
diff --git a/include/uapi/linux/kvm.h b/include/uapi/linux/kvm.h
index 308cc70bd6ab..59010a685007 100644
--- a/include/uapi/linux/kvm.h
+++ b/include/uapi/linux/kvm.h
@@ -528,6 +529,12 @@ struct kvm_run {
 #define KVM_NOTIFY_CONTEXT_INVALID (1 << 0)
            __u32 flags;
        } notify;
+       /* KVM_EXIT_MEMORY_FAULT */
+       struct {
+           __u64 flags;
+           __u64 gpa;
+           __u64 size;
+       } memory_fault;
        /* Fix the size of the union. */
        char padding[256];
    };
```
* 新增函数 `kvm_prepare_memory_fault_exit()` 准备 KVM 的 `KVM_EXIT_MEMORY_FAULT` 的退出信息
```cpp
static inline void kvm_prepare_memory_fault_exit(struct kvm_vcpu *vcpu,
                         gpa_t gpa, gpa_t size,
                         bool is_write, bool is_exec,
                         bool is_private)
{
    vcpu->run->exit_reason = KVM_EXIT_MEMORY_FAULT;
    vcpu->run->memory_fault.gpa = gpa;
    vcpu->run->memory_fault.size = size;

    /* RWX flags are not (yet) defined or communicated to userspace. */
    vcpu->run->memory_fault.flags = 0;
    if (is_private)
        vcpu->run->memory_fault.flags |= KVM_MEMORY_EXIT_FLAG_PRIVATE;
}
```
* 最后的判断不是这个 patch 加上的。如果 `GPA` 是私有地址，`flags` 域加上 `KVM_MEMORY_EXIT_FLAG_PRIVATE` 标志以进一步标识退出原因

## [PATCH 10/34] KVM: Add a dedicated mmu_notifier flag for reclaiming freed memory
* 通过让 `__kvm_handle_hva_range()` 返回是否找到重叠 memslot（即是否获取了 `mmu_lock`）来处理 AMD SEV 的 `kvm_arch_guest_memory_reclaimed()` hook。
  * 使用 `.on_unlock()` hook 可以，但是 `kvm_arch_guest_memory_reclaimed()` 需要在释放 `mmu_lock` 之后运行，这使得 `.on_lock()` 和 `.on_unlock()` 不对称。
* 使用一个小结构返回 notifier-specific 返回的元组（tuple），以及是否发现重叠。
  * 因为迭代 helpers 是 `__always_inlined`，实际上来说，该结构永远不会从函数调用中实际返回（更不用说实际上该结构的大小将是两个字节）。
---
* 这个就是提到的小结构，它将 *arch- 和特定动作的返回值* 和 *一个用于指示是否找到至少一个 memslot 的标志（例如，handler 是否找到 guest handler）* 打包成一个元组（tuple）作为 helper 的返回
* 注意，大多数 notifiers 都反对布尔值，因此即使 KVM 将 arch 代码的返回值跟踪为 `bool`，外部帮助程序也会将其转换为 `int`。
```cpp
/*
 * The inner-most helper returns a tuple containing the return value from the
 * arch- and action-specific handler, plus a flag indicating whether or not at
 * least one memslot was found, i.e. if the handler found guest memory.
 *
 * Note, most notifiers are averse to booleans, so even though KVM tracks the
 * return from arch code as a bool, outer helpers will cast it to an int. :-(
 */
typedef struct kvm_mmu_notifier_return {
    bool ret;
    bool found_memslot;
} kvm_mn_ret_t;
```
* 修改 `__kvm_handle_hva_range()` 返回类型为 `struct kvm_mmu_notifier_return` 的返回值
  * 其实是把 `ret` 和 `locked` 变量合二为一了
```diff
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index 756b94ecd511..e18a7f152c0b 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -582,22 +595,25 @@ static const union kvm_mmu_notifier_arg KVM_MMU_NOTIFIER_NO_ARG;
         node;                               \
         node = interval_tree_iter_next(node, start, last))      \

-static __always_inline int __kvm_handle_hva_range(struct kvm *kvm,
-                         const struct kvm_mmu_notifier_range *range)
+static __always_inline kvm_mn_ret_t __kvm_handle_hva_range(struct kvm *kvm,
+                              const struct kvm_mmu_notifier_range *range)
 {
-   bool ret = false, locked = false;
+   struct kvm_mmu_notifier_return r = {
+       .ret = false,
+       .found_memslot = false,
+   };
    struct kvm_gfn_range gfn_range;
    struct kvm_memory_slot *slot;
    struct kvm_memslots *slots;
    int i, idx;

    if (WARN_ON_ONCE(range->end <= range->start))
-       return 0;
+       return r;

    /* A null handler is allowed if and only if on_lock() is provided. */
    if (WARN_ON_ONCE(IS_KVM_NULL_FN(range->on_lock) &&
             IS_KVM_NULL_FN(range->handler)))
-       return 0;
+       return r;

    idx = srcu_read_lock(&kvm->srcu);

@@ -631,8 +647,8 @@ static __always_inline int __kvm_handle_hva_range(struct kvm *kvm,
            gfn_range.end = hva_to_gfn_memslot(hva_end + PAGE_SIZE - 1, slot);
            gfn_range.slot = slot;

-           if (!locked) {
-               locked = true;
+           if (!r.found_memslot) {
+               r.found_memslot = true;
                KVM_MMU_LOCK(kvm);
                if (!IS_KVM_NULL_FN(range->on_lock))
                    range->on_lock(kvm);
@@ -640,14 +656,14 @@ static __always_inline int __kvm_handle_hva_range(struct kvm *kvm,
                if (IS_KVM_NULL_FN(range->handler))
                    break;
            }
-           ret |= range->handler(kvm, &gfn_range);
+           r.ret |= range->handler(kvm, &gfn_range);
        }
    }

-   if (range->flush_on_ret && ret)
+   if (range->flush_on_ret && r.ret)
        kvm_flush_remote_tlbs(kvm);

-   if (locked) {
+   if (r.found_memslot) {
        KVM_MMU_UNLOCK(kvm);
        if (!IS_KVM_NULL_FN(range->on_unlock))
            range->on_unlock(kvm);
@@ -655,8 +671,7 @@ static __always_inline int __kvm_handle_hva_range(struct kvm *kvm,

    srcu_read_unlock(&kvm->srcu, idx);

-   /* The notifiers are averse to booleans. :-( */
-   return (int)ret;
+   return r;
 }
```
* 修改调用 `__kvm_handle_hva_range()` 的函数，保持原来的语义
```diff
@@ -677,7 +692,7 @@ static __always_inline int kvm_handle_hva_range(struct mmu_notifier *mn,
        .may_block  = false,
    };

-   return __kvm_handle_hva_range(kvm, &range);
+   return __kvm_handle_hva_range(kvm, &range).ret;
 }

 static __always_inline int kvm_handle_hva_range_no_flush(struct mmu_notifier *mn,
@@ -696,7 +711,7 @@ static __always_inline int kvm_handle_hva_range_no_flush(struct mmu_notifier *mn
        .may_block  = false,
    };

-   return __kvm_handle_hva_range(kvm, &range);
+   return __kvm_handle_hva_range(kvm, &range).ret;
 }
```
* commit message 的第一个段落讲的就是这个事情
* 如果发现一个或多个 memslots 并因此被清除，通知 arch 代码 guest 内存已被回收。这需要在释放 `mmu_lock` *之后* 完成，因为 x86 的回收路径很慢。
```diff
@@ -798,7 +813,7 @@ static int kvm_mmu_notifier_invalidate_range_start(struct mmu_notifier *mn,
        .end        = range->end,
        .handler    = kvm_mmu_unmap_gfn_range,
        .on_lock    = kvm_mmu_invalidate_begin,
-       .on_unlock  = kvm_arch_guest_memory_reclaimed,
+       .on_unlock  = (void *)kvm_null_fn,
        .flush_on_ret   = true,
        .may_block  = mmu_notifier_range_blockable(range),
    };
@@ -830,7 +845,13 @@ static int kvm_mmu_notifier_invalidate_range_start(struct mmu_notifier *mn,
    gfn_to_pfn_cache_invalidate_start(kvm, range->start, range->end,
                      hva_range.may_block);

-   __kvm_handle_hva_range(kvm, &hva_range);
+   /*
+    * If one or more memslots were found and thus zapped, notify arch code
+    * that guest memory has been reclaimed.  This needs to be done *after*
+    * dropping mmu_lock, as x86's reclaim path is slooooow.
+    */
+   if (__kvm_handle_hva_range(kvm, &hva_range).found_memslot)
+       kvm_arch_guest_memory_reclaimed(kvm);

    return 0;
 }
```

## [PATCH 11/34] KVM: Drop .on_unlock() mmu_notifier hook
* 现在删除 mmu_notifer 的 `.on_unlock()` hook，因为它不再用于 notifying 的内存回收 arch 代码。
  * 添加 `.on_unlock()` 并在删除 `mmu_lock` *之后* 调用它是一个糟糕的主意，因为这样做会导致 `.on_lock()` 和 `.on_unlock()` 具有不同和不对称的行为，并使未来的开发人员面临失败，例如，KVM 依赖使用 `.on_unlock()` 尝试在持有 `mmu_lock` 的同时运行回调会导致错误。
* 机会性地在 `kvm_mmu_invalidate_end()` 中添加一个 lockdep 断言，以防止未来出现此类性质的错误。
---
* `on_unlock_fn_t` 函数指针的类型被删除
```diff
@@ -544,7 +544,6 @@ static inline struct kvm *mmu_notifier_to_kvm(struct mmu_notifier *mn)
 typedef bool (*gfn_handler_t)(struct kvm *kvm, struct kvm_gfn_range *range);

 typedef void (*on_lock_fn_t)(struct kvm *kvm);
-typedef void (*on_unlock_fn_t)(struct kvm *kvm);

 struct kvm_mmu_notifier_range {
    /*
```
* `struct kvm_mmu_notifier_range` 的 `on_unlock` 域被删除
```diff
@@ -556,7 +555,6 @@ struct kvm_mmu_notifier_range {
    union kvm_mmu_notifier_arg arg;
    gfn_handler_t handler;
    on_lock_fn_t on_lock;
-   on_unlock_fn_t on_unlock;
    bool flush_on_ret;
    bool may_block;
 };
```
* 于是，用到该域的地方都要删除
```diff
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index e18a7f152c0b..7f3291dec7a6 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -663,11 +661,8 @@ static __always_inline kvm_mn_ret_t __kvm_handle_hva_range(struct kvm *kvm,
    if (range->flush_on_ret && r.ret)
        kvm_flush_remote_tlbs(kvm);

-   if (r.found_memslot) {
+   if (r.found_memslot)
        KVM_MMU_UNLOCK(kvm);
-       if (!IS_KVM_NULL_FN(range->on_unlock))
-           range->on_unlock(kvm);
-   }

    srcu_read_unlock(&kvm->srcu, idx);

@@ -687,7 +682,6 @@ static __always_inline int kvm_handle_hva_range(struct mmu_notifier *mn,
        .arg        = arg,
        .handler    = handler,
        .on_lock    = (void *)kvm_null_fn,
-       .on_unlock  = (void *)kvm_null_fn,
        .flush_on_ret   = true,
        .may_block  = false,
    };
@@ -706,7 +700,6 @@ static __always_inline int kvm_handle_hva_range_no_flush(struct mmu_notifier *mn
        .end        = end,
        .handler    = handler,
        .on_lock    = (void *)kvm_null_fn,
-       .on_unlock  = (void *)kvm_null_fn,
        .flush_on_ret   = false,
        .may_block  = false,
    };
@@ -813,7 +806,6 @@ static int kvm_mmu_notifier_invalidate_range_start(struct mmu_notifier *mn,
        .end        = range->end,
        .handler    = kvm_mmu_unmap_gfn_range,
        .on_lock    = kvm_mmu_invalidate_begin,
-       .on_unlock  = (void *)kvm_null_fn,
        .flush_on_ret   = true,
        .may_block  = mmu_notifier_range_blockable(range),
    };
@@ -891,7 +883,6 @@ static void kvm_mmu_notifier_invalidate_range_end(struct mmu_notifier *mn,
        .end        = range->end,
        .handler    = (void *)kvm_null_fn,
        .on_lock    = kvm_mmu_invalidate_end,
-       .on_unlock  = (void *)kvm_null_fn,
        .flush_on_ret   = false,
        .may_block  = mmu_notifier_range_blockable(range),
    };
```

## [PATCH 12/34] KVM: Introduce per-page memory attributes
* 在机密计算用途中，page 是私有还是共享是 KVM 执行 page fault handling、page zapping 等操作的必要信息。
  * Per-page 内存属性还有其他潜在的用例，例如，使内存只读（或 no-exec，或 exec-only 等），而无需修改 memslots。
* 引入由 `KVM_CAP_MEMORY_ATTRIBUTES` 通告的，名为 `KVM_SET_MEMORY_ATTRIBUTES` 的 `ioctl`，以允许用户空间将 per-page 内存属性设置给一个 guest 内存范围。
* 使用 xarray 在内部存储 per-page 属性，并采用幼稚的、未完全优化的实现，即在初始实现中优先考虑正确性而不是性能。
* 使用 bit `3` 作为 `PRIVATE` 属性，以便 KVM 将来可以使用 bit `0-2` 作为 `RWX` 属性/保护，例如，为用户空间提供对 guest 内存的读、写和执行保护的细粒度控制。
* 提供 arch 的 hook 来处理 *设置新属性之前和之后的属性更改*，供公共代码使用，例如，x86 将使用“`pre`” hook 来取消所有相关映射，并使用“`post`” hook 来跟踪是否可以使用大页来映射范围。
* 为了简化实现，请使用 `kvm_mmu_invalidate_{begin,end}()` 包装整个序列，即使该操作不能严格保证是无效的。
  * 对于初始用例，x86 *将*始终使内存无效，并且防止 arch 代码在属性不断变化时创建新映射，使得更容易推断使用属性的正确性。
* 将来的使用可能不需要失效，例如，如果 KVM 最终支持 `RWX` 保护并且用户空间给予 *更多* 保护，但再次选择简单性并在需要时进行优化。
---
### `KVM_CAP_MEMORY_ATTRIBUTES`
* 新增 KVM 能力 `KVM_CAP_MEMORY_ATTRIBUTES`
* `KVM_SET_MEMORY_ATTRIBUTES` 允许用户空间为一系列 guest 物理内存设置内存属性。
```cpp
#define KVM_CAP_MEMORY_ATTRIBUTES 233

/* Available with KVM_CAP_MEMORY_ATTRIBUTES */
#define KVM_SET_MEMORY_ATTRIBUTES              _IOW(KVMIO,  0xd2, struct kvm_memory_attributes)

struct kvm_memory_attributes {
  __u64 address;
  __u64 size;
  __u64 attributes;
  __u64 flags;
};

#define KVM_MEMORY_ATTRIBUTE_PRIVATE           (1ULL << 3)
```
* `address` 和 `size` 必须页对齐。
* 可以通过 `KVM_CAP_MEMORY_ATTRIBUTES` 上的 `ioctl(KVM_CHECK_EXTENSION)` 检索支持的属性。
```diff
@@ -4641,6 +4841,10 @@ static int kvm_vm_ioctl_check_extension_generic(struct kvm *kvm, long arg)
    case KVM_CAP_BINARY_STATS_FD:
    case KVM_CAP_SYSTEM_EVENT_DATA:
        return 1;
+#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
+   case KVM_CAP_MEMORY_ATTRIBUTES:
+       return kvm_supported_mem_attributes(kvm);
+#endif
    default:
        break;
    }
```
* 如果在 VM 上执行，`KVM_CAP_MEMORY_ATTRIBUTES` 会精确返回 **该 VM 支持的属性**。
* 如果在系统范围内执行，`KVM_CAP_MEMORY_ATTRIBUTES` 将返回 **KVM 支持的所有属性**。
* 此时定义的唯一属性是 `KVM_MEMORY_ATTRIBUTE_PRIVATE`，它将关联的 `gfn` 标记为 guest 私有内存。
* 请注意，没有“get”API。用户空间负责根据需要显式跟踪 `gfn/page` 的状态。
* “`flags`”域保留用于将来的扩展，并且必须为“`0`”。
* 联合体 `union kvm_mmu_notifier_arg` 增加新域 `attributes`
```diff
 union kvm_mmu_notifier_arg {
    pte_t pte;
+   unsigned long attributes;
 };
```
* `struct kvm` 结构体增加新域 `struct xarray mem_attr_array`，该域
  * 受 `slots_locks` 保护（对于 *写*）
  * 受 RCU 保护（对于 *读*）
  * 该 xarray **键** 为 GFN，**值** 为内存属性
```diff
@@ -806,6 +807,10 @@ struct kvm {

 #ifdef CONFIG_HAVE_KVM_PM_NOTIFIER
    struct notifier_block pm_notifier;
+#endif
+#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
+   /* Protected by slots_locks (for writes) and RCU (for reads) */
+   struct xarray mem_attr_array;
 #endif
    char stats_id[KVM_STATS_NAME_SIZE];
 };
```
* 增加一系列 helpers 的声明和 `kvm_get_memory_attributes()` 的定义
```cpp
#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
static inline unsigned long kvm_get_memory_attributes(struct kvm *kvm, gfn_t gfn)
{   //根据 GFN 返回相应的 page 的内存属性
    return xa_to_value(xa_load(&kvm->mem_attr_array, gfn));
}

bool kvm_range_has_memory_attributes(struct kvm *kvm, gfn_t start, gfn_t end,
                     unsigned long attrs);
bool kvm_arch_pre_set_memory_attributes(struct kvm *kvm,
                    struct kvm_gfn_range *range);
bool kvm_arch_post_set_memory_attributes(struct kvm *kvm,
                     struct kvm_gfn_range *range);
#endif /* CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES */
```
* 新增 `KVM_GENERIC_MEMORY_ATTRIBUTES` 的 Kconfig
```diff
diff --git a/virt/kvm/Kconfig b/virt/kvm/Kconfig
index ecae2914c97e..5bd7fcaf9089 100644
--- a/virt/kvm/Kconfig
+++ b/virt/kvm/Kconfig
@@ -96,3 +96,7 @@ config KVM_GENERIC_HARDWARE_ENABLING
 config KVM_GENERIC_MMU_NOTIFIER
        select MMU_NOTIFIER
        bool
+
+config KVM_GENERIC_MEMORY_ATTRIBUTES
+       select KVM_GENERIC_MMU_NOTIFIER
+       bool
```
* 创建 VM 时初始化 per-VM 的 `mem_attr_array` xarray，销毁 VM 时顺带销毁它
```diff
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index 7f3291dec7a6..f1a575d39b3b 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -1211,6 +1211,9 @@ static struct kvm *kvm_create_vm(unsigned long type, const char *fdname)
    spin_lock_init(&kvm->mn_invalidate_lock);
    rcuwait_init(&kvm->mn_memslots_update_rcuwait);
    xa_init(&kvm->vcpu_array);
+#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
+   xa_init(&kvm->mem_attr_array);
+#endif

    INIT_LIST_HEAD(&kvm->gpc_list);
    spin_lock_init(&kvm->gpc_lock);
@@ -1391,6 +1394,9 @@ static void kvm_destroy_vm(struct kvm *kvm)
    }
    cleanup_srcu_struct(&kvm->irq_srcu);
    cleanup_srcu_struct(&kvm->srcu);
+#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
+   xa_destroy(&kvm->mem_attr_array);
+#endif
    kvm_arch_free_vm(kvm);
    preempt_notifier_dec();
    hardware_disable_all();
```
* 新增 `ioctl(KVM_SET_MEMORY_ATTRIBUTES)` 的支持，调用的是 `kvm_vm_ioctl_set_mem_attributes()` 入口函数
```diff
@@ -5034,6 +5238,18 @@ static long kvm_vm_ioctl(struct file *filp,
        break;
    }
 #endif /* CONFIG_HAVE_KVM_IRQ_ROUTING */
+#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
+   case KVM_SET_MEMORY_ATTRIBUTES: {
+       struct kvm_memory_attributes attrs;
+
+       r = -EFAULT;
+       if (copy_from_user(&attrs, argp, sizeof(attrs)))
+           goto out;
+
+       r = kvm_vm_ioctl_set_mem_attributes(kvm, &attrs);
+       break;
+   }
+#endif /* CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES */
    case KVM_CREATE_DEVICE: {
        struct kvm_create_device cd;
```
### 新增内存属性 helper 函数
* `kvm_range_has_memory_attributes()` 判断给定的 GFN 的范围的内存属性是否都是指定的属性
  * 如果都是，返回 `true`
```cpp
#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
/*
 * Returns true if _all_ gfns in the range [@start, @end) have attributes
 * matching @attrs.
 */
bool kvm_range_has_memory_attributes(struct kvm *kvm, gfn_t start, gfn_t end,
                     unsigned long attrs)
{
    XA_STATE(xas, &kvm->mem_attr_array, start); //将访问 xarray 的游标设置为 start
    unsigned long index;
    bool has_attrs;
    void *entry;

    rcu_read_lock();
    //当询问的属性的值为 NULL 时
    if (!attrs) {
        has_attrs = !xas_find(&xas, end - 1); //在 xarray 中查找下一个存在的条目直到索引为 end - 1
        goto out; //如果找不到，刚好说明都有 NULL 属性，于是返回 true；否则返回 false
    }
    //当属性不为 NULL 时，遍历 xarray 中指定的范围的条目
    has_attrs = true;
    for (index = start; index < end; index++) {
        do {
            entry = xas_next(&xas);
        } while (xas_retry(&xas, entry));
        //条目的索引和遍历记录的索引对不上，比如有空洞；或者条目的属性值与期望的不符
        if (xas.xa_index != index || xa_to_value(entry) != attrs) {
            has_attrs = false; //终止遍历返回 false
            break;
        }
    }
    //能走到这里说明可以返回 true
out:
    rcu_read_unlock();
    return has_attrs;
}
```
* `kvm_supported_mem_attributes()` 返回 KVM 或该 VM 支持的内存属性，当前仅支持 `KVM_MEMORY_ATTRIBUTE_PRIVATE` 一个属性
  * 如果在 VM 上执行，`KVM_CAP_MEMORY_ATTRIBUTES` 会精确返回 **该 VM 支持的属性**
    * 如果该 arch 有私有内存 `kvm_arch_has_private_mem(kvm)`
  * 如果在系统范围内执行，`KVM_CAP_MEMORY_ATTRIBUTES` 将返回 **KVM 支持的所有属性** 
```cpp
static u64 kvm_supported_mem_attributes(struct kvm *kvm)
{
    if (!kvm || kvm_arch_has_private_mem(kvm))
        return KVM_MEMORY_ATTRIBUTE_PRIVATE;

    return 0;
}
```
* `kvm_handle_gfn_range()` 根据给定的 `struct kvm_mmu_notifier_range *range` 参数调用 `handler`
  * `range` 中有起始和结束 GFN
  * `range` 可能跨越多个 memslots，也有可能包含在一个 memslot 里
  * `range->on_lock()` 回调有可能会被调用到
  * 返回值 `ret` 是依次调用 `range->handler(kvm, &gfn_range)` 的返回值按位与 `|` 起来的结果
  * 可能会 flush TLB
```cpp
static __always_inline void kvm_handle_gfn_range(struct kvm *kvm,
                         struct kvm_mmu_notifier_range *range)
{
    struct kvm_gfn_range gfn_range;
    struct kvm_memory_slot *slot;
    struct kvm_memslots *slots;
    struct kvm_memslot_iter iter;
    bool found_memslot = false;
    bool ret = false;
    int i;

    gfn_range.arg = range->arg;
    gfn_range.may_block = range->may_block;

    for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
        slots = __kvm_memslots(kvm, i);
        kvm_for_each_memslot_in_gfn_range(&iter, slots, range->start, range->end) {
            slot = iter.slot;
            gfn_range.slot = slot;
            //range 可能跨越多个 memslots，也有可能包含在一个 memslot 里
            gfn_range.start = max(range->start, slot->base_gfn);
            gfn_range.end = min(range->end, slot->base_gfn + slot->npages);
            if (gfn_range.start >= gfn_range.end)
                continue;

            if (!found_memslot) {
                found_memslot = true;
                KVM_MMU_LOCK(kvm);
                if (!IS_KVM_NULL_FN(range->on_lock))
                    range->on_lock(kvm);
            }
            //返回值是依次调用 range->handler(kvm, &gfn_range) 的返回值按位与起来的结果
            ret |= range->handler(kvm, &gfn_range);
        }
    }

    if (range->flush_on_ret && ret)
        kvm_flush_remote_tlbs(kvm);

    if (found_memslot)
        KVM_MMU_UNLOCK(kvm);
}
```
* “`pre`” hook `kvm_pre_set_memory_attributes()` 取消所有相关映射，但此 patch 还无实际动作
* 无条件地将范围添加到失效集合中，无论 arch 回调是否实际上需要 zap SPTE。
  * 例如，如果 KVM 将来支持 `RWX` 属性，并且属性来自 `R=>RW`，则此时 zapping 并不是绝对必要的。
* 无条件添加范围允许 KVM 要求 MMU invalidations 在 `begin()` 和 `end()` 之间添加至少一个范围，
  * 例如，允许 KVM 检测缺少 `add()` 的错误。
* 放宽规则 *可能* 是安全的，但 *在属性不断变化时允许新映射* 是否值得或值得复杂性并不明显。
```cpp
static bool kvm_pre_set_memory_attributes(struct kvm *kvm,
                      struct kvm_gfn_range *range)
{
    /*
     * Unconditionally add the range to the invalidation set, regardless of
     * whether or not the arch callback actually needs to zap SPTEs.  E.g.
     * if KVM supports RWX attributes in the future and the attributes are
     * going from R=>RW, zapping isn't strictly necessary.  Unconditionally
     * adding the range allows KVM to require that MMU invalidations add at
     * least one range between begin() and end(), e.g. allows KVM to detect
     * bugs where the add() is missed.  Relaxing the rule *might* be safe,
     * but it's not obvious that allowing new mappings while the attributes
     * are in flux is desirable or worth the complexity.
     */
    kvm_mmu_invalidate_range_add(kvm, range->start, range->end);

    return kvm_arch_pre_set_memory_attributes(kvm, range);
}
```
* `kvm_vm_set_mem_attributes()` 设置指定起始和结束页帧的内存属性
```cpp
/* Set @attributes for the gfn range [@start, @end). */
static int kvm_vm_set_mem_attributes(struct kvm *kvm, gfn_t start, gfn_t end,
                     unsigned long attributes)
{
    struct kvm_mmu_notifier_range pre_set_range = {
        .start = start,
        .end = end,
        .handler = kvm_pre_set_memory_attributes,
        .on_lock = kvm_mmu_invalidate_begin,
        .flush_on_ret = true,
        .may_block = true,
    };
    struct kvm_mmu_notifier_range post_set_range = {
        .start = start,
        .end = end,
        .arg.attributes = attributes,
        .handler = kvm_arch_post_set_memory_attributes,
        .on_lock = kvm_mmu_invalidate_end,
        .may_block = true,
    };
    unsigned long i;
    void *entry;
    int r = 0;
    //构造 xarray 条目，当 attributes 不为空时设置其值为 attributes
    entry = attributes ? xa_mk_value(attributes) : NULL;

    mutex_lock(&kvm->slots_lock);
    //如果整个范围都已有需要的属性了就什么都不做
    /* Nothing to do if the entire range as the desired attributes. */
    if (kvm_range_has_memory_attributes(kvm, start, end, attributes))
        goto out_unlock;
    //提前预留内存，以避免在设置新属性时必须处理中途故障。
    /*
     * Reserve memory ahead of time to avoid having to deal with failures
     * partway through setting the new attributes.
     */
    for (i = start; i < end; i++) {
        r = xa_reserve(&kvm->mem_attr_array, i, GFP_KERNEL_ACCOUNT);
        if (r)
            goto out_unlock;
    }
    //设置新属性之前调用 pre 回调
    kvm_handle_gfn_range(kvm, &pre_set_range);
    //将 kvm->mem_attr_array 这个 xarray 从 start 到 end 的元素的值存储为 entry
    for (i = start; i < end; i++) {
        r = xa_err(xa_store(&kvm->mem_attr_array, i, entry,
                    GFP_KERNEL_ACCOUNT));
        KVM_BUG_ON(r, kvm);
    }
    //设置新属性之后调用 post 回调
    kvm_handle_gfn_range(kvm, &post_set_range);

out_unlock:
    mutex_unlock(&kvm->slots_lock);

    return r;
}
```
* 接口函数 `kvm_vm_ioctl_set_mem_attributes()` 设置内存属性
```cpp
static int kvm_vm_ioctl_set_mem_attributes(struct kvm *kvm,
                       struct kvm_memory_attributes *attrs)
{
    gfn_t start, end;
    //flags 域当前还未使用，必须为 0
    /* flags is currently not used. */
    if (attrs->flags)
        return -EINVAL;
    if (attrs->attributes & ~kvm_supported_mem_attributes(kvm))
        return -EINVAL;  //如果设置的内存属性不在 VM 支持内存属性之列，返回错误
    if (attrs->size == 0 || attrs->address + attrs->size < attrs->address)
        return -EINVAL;  //设置内存属性大小为 0，或者导致发生环绕，返回错误
    if (!PAGE_ALIGNED(attrs->address) || !PAGE_ALIGNED(attrs->size))
        return -EINVAL;  //如果地址或者大小未按页对齐
    //计算起始和结束页帧
    start = attrs->address >> PAGE_SHIFT;
    end = (attrs->address + attrs->size) >> PAGE_SHIFT;
    //当前 xarray 用 "unsigned long" 来跟踪数据，这里做个简单的检查
    /*
     * xarray tracks data using "unsigned long", and as a result so does
     * KVM.  For simplicity, supports generic attributes only on 64-bit
     * architectures.
     */
    BUILD_BUG_ON(sizeof(attrs->attributes) != sizeof(unsigned long));
    //设置指定起始和结束页帧的内存属性
    return kvm_vm_set_mem_attributes(kvm, start, end, attrs->attributes);
}
#endif /* CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES */
``` 

## [PATCH 13/34] mm: Add AS_UNMOVABLE to mark mapping as completely unmovable
* 为在任何情况下都无法迁移的映射添加“unmovable”标志。
  * KVM 将使用该标志来支持即将推出的 `GUEST_MEMFD`，`GUEST_MEMFD` 不支持压缩/迁移，至少在可预见的将来不支持。
* 在 folio lock 下测试 `AS_UNMOVABLE`，就像已经针对异步压缩/dirty folio 情况所做的那样，因为在压缩运行时可以通过截断（truncation）来删除映射。
  * 为了避免必须用映射来 lock 每个 folio，假设/要求不可移动的映射也是不可删除的，并且将 `mapping_set_unmovable()` 也设置为 `AS_UNEVICTABLE`。
---
* 增加 `mapping_flags` 枚举值 `AS_UNMOVABLE`
```diff
diff --git a/include/linux/pagemap.h b/include/linux/pagemap.h
index 351c3b7f93a1..82c9bf506b79 100644
--- a/include/linux/pagemap.h
+++ b/include/linux/pagemap.h
@@ -203,7 +203,8 @@ enum mapping_flags {
    /* writeback related tags are not used */
    AS_NO_WRITEBACK_TAGS = 5,
    AS_LARGE_FOLIO_SUPPORT = 6,
-   AS_RELEASE_ALWAYS,  /* Call ->release_folio(), even if no private data */
+   AS_RELEASE_ALWAYS = 7,  /* Call ->release_folio(), even if no private data */
+   AS_UNMOVABLE    = 8,    /* The mapping cannot be moved, ever */
 };
```
* 增加两个 helper 函数来 *设置* 和 *测试* 该 flag
* 为了避免必须用映射来 lock 每个 folio，假设/要求 *不可移动的* 映射也是 *不可删除的*，同时设置 `AS_UNEVICTABLE`。
* 压缩迁移扫描器 `isolate_migratepages_block()` 据此减少页面的锁定
```cpp
static inline void mapping_set_unmovable(struct address_space *mapping)
{
    /*
     * It's expected unmovable mappings are also unevictable. Compaction
     * migrate scanner (isolate_migratepages_block()) relies on this to
     * reduce page locking.
     */
    set_bit(AS_UNEVICTABLE, &mapping->flags);
    set_bit(AS_UNMOVABLE, &mapping->flags);
}

static inline bool mapping_unmovable(struct address_space *mapping)
{
    return test_bit(AS_UNMOVABLE, &mapping->flags);
}
```
* 在将一个 folio 移动到一个新分配的 folio 时，如果该页面 `src` 的映射是不可移动的，则返回 `-EOPNOTSUPP` 错误
```diff
diff --git a/mm/migrate.c b/mm/migrate.c
index 06086dc9da28..60f2ff6b36aa 100644
--- a/mm/migrate.c
+++ b/mm/migrate.c
@@ -956,6 +956,8 @@ static int move_to_new_folio(struct folio *dst, struct folio *src,

        if (!mapping)
            rc = migrate_folio(mapping, dst, src, mode);
+       else if (mapping_unmovable(mapping))
+           rc = -EOPNOTSUPP;
        else if (mapping->a_ops->migrate_folio)
            /*
             * Most folios have a mapping and most filesystems
```
* `folio_test_unevictable(folio)` 和 `folio_test_dirty(folio)` 从表达式变为本地变量 `is_unevictable` 和 `is_dirty`
```diff
diff --git a/mm/compaction.c b/mm/compaction.c
index 38c8d216c6a3..12b828aed7c8 100644
--- a/mm/compaction.c
+++ b/mm/compaction.c
@@ -883,6 +883,7 @@ isolate_migratepages_block(struct compact_control *cc, unsigned long low_pfn,

    /* Time to isolate some pages for migration */
    for (; low_pfn < end_pfn; low_pfn++) {
+       bool is_dirty, is_unevictable;

        if (skip_on_failure && low_pfn >= next_skip_pfn) {
            /*
@@ -1080,8 +1081,10 @@ isolate_migratepages_block(struct compact_control *cc, unsigned long low_pfn,
        if (!folio_test_lru(folio))
            goto isolate_fail_put;

+       is_unevictable = folio_test_unevictable(folio);
+
        /* Compaction might skip unevictable pages but CMA takes them */
-       if (!(mode & ISOLATE_UNEVICTABLE) && folio_test_unevictable(folio))
+       if (!(mode & ISOLATE_UNEVICTABLE) && is_unevictable)
            goto isolate_fail_put;

        /*
@@ -1093,26 +1096,42 @@ isolate_migratepages_block(struct compact_control *cc, unsigned long low_pfn,
        if ((mode & ISOLATE_ASYNC_MIGRATE) && folio_test_writeback(folio))
            goto isolate_fail_put;

-       if ((mode & ISOLATE_ASYNC_MIGRATE) && folio_test_dirty(folio)) {
-           bool migrate_dirty;
+       is_dirty = folio_test_dirty(folio);
+
+       if (((mode & ISOLATE_ASYNC_MIGRATE) && is_dirty) ||
+           (mapping && is_unevictable)) {
+           bool migrate_dirty = true;
+           bool is_unmovable;
```
* 只有没有映射或具有 `->migrate_folio` 回调的 folio 才可以无阻塞地迁移。
  * 批注：`migrate_dirty` 为 `true`？
* 来自不可移动映射的 folio 不可迁移。
* 然而，我们可以与 truncation 竞争，这可以释放我们需要检查的映射。
  * Truncation 会持有 folio lock，直到 folio 从页面中删除之后，因此我们自己持有它就足够了。
  * 批注：因此需要在 folio lock 下测试 `AS_UNMOVABLE`
* 为了避免仅仅为了检查页面是否不可移动（unmovable）而锁定 folio，假设每个不可移动 folio 也是不可清除的（unevictable），这是一种更便宜的测试。
  * 如果我们的假设出错，这不是正确性错误，只是可能浪费周期。
```diff
            /*
             * Only folios without mappings or that have
-            * a ->migrate_folio callback are possible to
-            * migrate without blocking.  However, we may
-            * be racing with truncation, which can free
-            * the mapping.  Truncation holds the folio lock
-            * until after the folio is removed from the page
-            * cache so holding it ourselves is sufficient.
+            * a ->migrate_folio callback are possible to migrate
+            * without blocking.
+            *
+            * Folios from unmovable mappings are not migratable.
+            *
+            * However, we can be racing with truncation, which can
+            * free the mapping that we need to check. Truncation
+            * holds the folio lock until after the folio is removed
+            * from the page so holding it ourselves is sufficient.
+            *
+            * To avoid locking the folio just to check unmovable,
+            * assume every unmovable folio is also unevictable,
+            * which is a cheaper test.  If our assumption goes
+            * wrong, it's not a correctness bug, just potentially
+            * wasted cycles.
             */
            if (!folio_trylock(folio))
                goto isolate_fail_put;
```
* 增加了设置 `migrate_dirty` 的判断条件，只有在异步迁移模式且 folio 是脏页的情况下才继续 `migrate_dirty` 的设置（批注：why?）
* 增加了 `is_unmovable` 不可移动标志的设置
* 如果页面 *不能无阻塞地迁移* 或者 *是不可移动的页面*，隔离失败
```diff
            mapping = folio_mapping(folio);
-           migrate_dirty = !mapping ||
-                   mapping->a_ops->migrate_folio;
+           if ((mode & ISOLATE_ASYNC_MIGRATE) && is_dirty) {
+               migrate_dirty = !mapping ||
+                       mapping->a_ops->migrate_folio;
+           }
+           is_unmovable = mapping && mapping_unmovable(mapping);
            folio_unlock(folio);
-           if (!migrate_dirty)
+           if (!migrate_dirty || is_unmovable)
                goto isolate_fail_put;
        }
```

## [PATCH 14/34] fs: Rename anon_inode_getfile_secure() and anon_inode_getfd_secure()
* 对 `inode_init_security_anon()` LSM hook 的调用并不是使用 `anon_inode_getfile_secure()` 或 `anon_inode_getfd_secure()` 的唯一原因。
  * 例如，这些函数还允许创建一个非零大小的文件，而不需要成熟的（full-blown）文件系统。
  * 在这种情况下，您不需要“安全”版本，只需要唯一的索引节点；
  * 当前函数的名称令人困惑，并且不能很好地解释与更“标准”的 `anon_inode_getfile()` 和 `anon_inode_getfd()` 的区别。
* 当然，事情还有另一面。严格来说，io_uring 和 userfaultfd 都不需要不同的 inode，并且 `anon_inode_create_get{file,fd}()` 是否允许 LSM 拦截和阻止 inode 的创建也不再那么清楚。
* 如果有人愿意，可以保留 `anon_inode_getfile_secure()` 和 `anon_inode_getfd_secure()`，使用共享 inode 或新的 inode，具体取决于 `CONFIG_SECURITY`。
* 然而，这可能有点矫枉过正，并且可能会导致不同配置中潜在的错误。因此，只是向 io_uring 和 userfaultfd 添加注释来解释函数的选择。
* 在此期间，删除现在的 `anon_inode_create_getfd()` 的导出。
  * 没有 in-tree 模块使用它，而且旧名称无论如何都消失了。
  * 如果有人确实需要该符号，他们可以询问，或者他们可以使用 `anon_inode_create_getfile()`，它将很快导出以在 KVM 中使用。
---
* 这里侧重点在于，要不要创建 inode，而不是是否“安全”
  * `anon_inode_make_secure_inode()` 除了分配新的 inode 实例之外，还会调用 `inode_init_security_anon()` LSM hook
  * 因此这里把入参 `secure` 改为 `make_inode` 来突显意图
```diff
diff --git a/fs/anon_inodes.c b/fs/anon_inodes.c
index 24192a7667ed..42b02dc36474 100644
--- a/fs/anon_inodes.c
+++ b/fs/anon_inodes.c
@@ -79,7 +79,7 @@ static struct file *__anon_inode_getfile(const char *name,
                     const struct file_operations *fops,
                     void *priv, int flags,
                     const struct inode *context_inode,
-                    bool secure)
+                    bool make_inode)
 {
    struct inode *inode;
    struct file *file;
@@ -87,7 +87,7 @@ static struct file *__anon_inode_getfile(const char *name,
    if (fops->owner && !try_module_get(fops->owner))
        return ERR_PTR(-ENOENT);

-   if (secure) {
+   if (make_inode) {
        inode = anon_inode_make_secure_inode(name, context_inode);
        if (IS_ERR(inode)) {
            file = ERR_CAST(inode);
```
* 基于同样的理由，会调用 `__anon_inode_getfile(..., true)` 的 `anon_inode_getfile_secure()` 改名为 `anon_inode_create_getfile()`
```diff
@@ -149,13 +149,10 @@ struct file *anon_inode_getfile(const char *name,
 EXPORT_SYMBOL_GPL(anon_inode_getfile);

 /**
- * anon_inode_getfile_secure - Like anon_inode_getfile(), but creates a new
+ * anon_inode_create_getfile - Like anon_inode_getfile(), but creates a new
  *                             !S_PRIVATE anon inode rather than reuse the
  *                             singleton anon inode and calls the
- *                             inode_init_security_anon() LSM hook.  This
- *                             allows for both the inode to have its own
- *                             security context and for the LSM to enforce
- *                             policy on the inode's creation.
+ *                             inode_init_security_anon() LSM hook.
  *
  * @name:    [in]    name of the "class" of the new file
  * @fops:    [in]    file operations for the new file
@@ -164,11 +161,21 @@ EXPORT_SYMBOL_GPL(anon_inode_getfile);
  * @context_inode:
  *           [in]    the logical relationship with the new inode (optional)
  *
+ * Create a new anonymous inode and file pair.  This can be done for two
+ * reasons:
+ *
+ * - for the inode to have its own security context, so that LSMs can enforce
+ *   policy on the inode's creation;
+ *
+ * - if the caller needs a unique inode, for example in order to customize
+ *   the size returned by fstat()
+ *
  * The LSM may use @context_inode in inode_init_security_anon(), but a
- * reference to it is not held.  Returns the newly created file* or an error
- * pointer.  See the anon_inode_getfile() documentation for more information.
+ * reference to it is not held.
+ *
+ * Returns the newly created file* or an error pointer.
  */
-struct file *anon_inode_getfile_secure(const char *name,
+struct file *anon_inode_create_getfile(const char *name,
                       const struct file_operations *fops,
                       void *priv, int flags,
                       const struct inode *context_inode)
```
* `__anon_inode_getfd()` 是对 `__anon_inode_getfile()` 的封装，将其返回的文件结构 `struct file *` 转为文件描述符 `int fd`
  * 此处的修改为对其做入参重命名
```diff
@@ -181,7 +188,7 @@ static int __anon_inode_getfd(const char *name,
                  const struct file_operations *fops,
                  void *priv, int flags,
                  const struct inode *context_inode,
-                 bool secure)
+                 bool make_inode)
 {
    int error, fd;
    struct file *file;
@@ -192,7 +199,7 @@ static int __anon_inode_getfd(const char *name,
    fd = error;

    file = __anon_inode_getfile(name, fops, priv, flags, context_inode,
-                   secure);
+                   make_inode);
    if (IS_ERR(file)) {
        error = PTR_ERR(file);
        goto err_put_unused_fd;
```
* `anon_inode_getfd_secure()` 作为对 `__anon_inode_getfd(..., true)` 的封装，进行函数重命名，并取消了函数导出
```diff
@@ -231,10 +238,9 @@ int anon_inode_getfd(const char *name, const struct file_operations *fops,
 EXPORT_SYMBOL_GPL(anon_inode_getfd);

 /**
- * anon_inode_getfd_secure - Like anon_inode_getfd(), but creates a new
+ * anon_inode_create_getfd - Like anon_inode_getfd(), but creates a new
  * !S_PRIVATE anon inode rather than reuse the singleton anon inode, and calls
- * the inode_init_security_anon() LSM hook. This allows the inode to have its
- * own security context and for a LSM to reject creation of the inode.
+ * the inode_init_security_anon() LSM hook.
  *
  * @name:    [in]    name of the "class" of the new file
  * @fops:    [in]    file operations for the new file
@@ -243,16 +249,26 @@ EXPORT_SYMBOL_GPL(anon_inode_getfd);
  * @context_inode:
  *           [in]    the logical relationship with the new inode (optional)
  *
+ * Create a new anonymous inode and file pair.  This can be done for two
+ * reasons:
+ *
+ * - for the inode to have its own security context, so that LSMs can enforce
+ *   policy on the inode's creation;
+ *
+ * - if the caller needs a unique inode, for example in order to customize
+ *   the size returned by fstat()
+ *
  * The LSM may use @context_inode in inode_init_security_anon(), but a
  * reference to it is not held.
+ *
+ * Returns a newly created file descriptor or an error code.
  */
-int anon_inode_getfd_secure(const char *name, const struct file_operations *fops,
+int anon_inode_create_getfd(const char *name, const struct file_operations *fops,
                void *priv, int flags,
                const struct inode *context_inode)
 {
    return __anon_inode_getfd(name, fops, priv, flags, context_inode, true);
 }
-EXPORT_SYMBOL_GPL(anon_inode_getfd_secure);
```
* 对引用 `anon_inode_getfd_secure()` 的 userfaultfd 作出修改
```diff
diff --git a/fs/userfaultfd.c b/fs/userfaultfd.c
index 56eaae9dac1a..7a1cf8bab5eb 100644
--- a/fs/userfaultfd.c
+++ b/fs/userfaultfd.c
@@ -1033,7 +1033,7 @@ static int resolve_userfault_fork(struct userfaultfd_ctx *new,
 {
    int fd;

-   fd = anon_inode_getfd_secure("[userfaultfd]", &userfaultfd_fops, new,
+   fd = anon_inode_create_getfd("[userfaultfd]", &userfaultfd_fops, new,
            O_RDONLY | (new->flags & UFFD_SHARED_FCNTL_FLAGS), inode);
    if (fd < 0)
        return fd;
@@ -2205,7 +2205,8 @@ static int new_userfaultfd(int flags)
    /* prevent the mm struct to be freed */
    mmgrab(ctx->mm);

-   fd = anon_inode_getfd_secure("[userfaultfd]", &userfaultfd_fops, ctx,
+   /* Create a new inode so that the LSM can block the creation.  */
+   fd = anon_inode_create_getfd("[userfaultfd]", &userfaultfd_fops, ctx,
            O_RDONLY | (flags & UFFD_SHARED_FCNTL_FLAGS), NULL);
    if (fd < 0) {
        mmdrop(ctx->mm);
```
* 对引用 `anon_inode_getfile_secure()` 的 io_uring 作出修改
```diff
diff --git a/io_uring/io_uring.c b/io_uring/io_uring.c
index 8d1bc6cdfe71..22b98f47bb28 100644
--- a/io_uring/io_uring.c
+++ b/io_uring/io_uring.c
@@ -3835,7 +3835,8 @@ static struct file *io_uring_get_file(struct io_ring_ctx *ctx)
        return ERR_PTR(ret);
 #endif

-   file = anon_inode_getfile_secure("[io_uring]", &io_uring_fops, ctx,
+   /* Create a new inode so that the LSM can block the creation.  */
+   file = anon_inode_create_getfile("[io_uring]", &io_uring_fops, ctx,
                     O_RDWR | O_CLOEXEC, NULL);
 #if defined(CONFIG_UNIX)
    if (IS_ERR(file)) {
```
* 对头文件 include/linux/anon_inodes.h 中的修改不贴了

## [PATCH 15/34] KVM: Add KVM_CREATE_GUEST_MEMFD ioctl() for guest-specific backing memory
* 引入 `ioctl(KVM_CREATE_GUEST_MEMFD)`，以允许创建与特定 KVM 虚拟机绑定的基于文件的内存，其主要目的是为 guest 内存提供服务。
* 一个 guest-first 内存子系统允许进行优化和增强，而这些优化和增强在通用内存子系统中实现/支持是笨拙（kludgy）或完全不可行的（outright infeasible）。
* 使用 guest_memfd，*guest 保护* 和 *映射大小* 与 host 侧的用户空间映射完全解耦。
  * 例如，KVM 目前不支持将内存映射为在 guest 中可写，而不在 host 用户空间中也可写，因为 KVM 的 ABI 使用 VMA 保护来定义所允许的 guest 保护。
    * 用户空间可以通过建立两个映射来弥补（fudge）这一点，一个是 guest 可写的映射，一个是自身可读的映射，但这在多个方面（multiple fronts）都不是最佳的（suboptimal）。
  * 类似地，KVM 目前要求 guest 映射大小是 host 用户空间映射大小的严格子集，例如 KVM 不支持创建 `1GiB` guest 映射，除非用户空间也有 `1GiB` guest 映射。
    * 解耦映射大小将允许用户空间仅精确映射所需的内容，而不会影响 guest 性能，例如，加强对 guest 内存的无意访问。
* 解耦 guest 和用户空间映射还可以为 HugeTLB 的高粒度映射提供更清晰的替代方案，而 HugeTLB 已经陷入了僵局（reached a bit of an impasse），并且不太可能被合并。
* 一个 guest-first 内存子系统还为诸如专用内存池（用于 slice-of-hardware 虚拟机）和消除“`struct page`”（用于用户空间 *永远不* 需要 `mmap()` guest 内存的 offload setups）等提供了更清晰的视线。
* 更直接地说，能够将内存映射到 KVM guest 而不将所述内存映射到 host 对于机密 VM（CoCo VM）（guest_memfd 的初始用例）至关重要。
  * 虽然 AMD 的 SEV 和 Intel 的 TDX 通过使用不受信任的 host 无法使用的密钥加密 guest 内存来防止不受信任的软件读取 guest 私有数据，但 Protected KVM（pKVM）等项目 *无需* 依赖内存加密即可提供机密性和完整性。
  * 对于 SEV-SNP 和 TDX，访问 guest 私有内存可能对 host 来说是致命的，即 KVM 必须阻止 host 用户空间访问 guest 内存，无论硬件行为如何。
* 支持 CoCo VM 的 *尝试 #1* 是添加一个 VMA 标志来将内存标记为只能由 KVM（或类似的启发式（enlightened）内核子系统）映射。
  * 这种方法被放弃主要是因为它需要玩 `PROT_NONE` 游戏以防止用户空间访问 guest 内存。
* *尝试 #2* 是篡夺（usurp）`PG_hwpoison` 以阻止 host 将 guest 私有内存映射到用户空间，但该方法无法满足基于软件的 CoCo VM 的几个要求，
  * 例如 pKVM，因为内核无法轻松强制执行 1:1 的 page:guest 关联，更不用说 1:1 的 pfn:gfn 映射了。
  * 并且使用 `PG_hwpoison` 不适用于不受“`struct page`”支持的内存，例如，如果设备获得向 guest 暴露的加密内存区域的支持。
* *尝试 #3* 是扩展 `memfd()` 系统调用并包装 shmem 以提供基于文件的专用 guest 内存。
  * 在 Hugh Dickins 和 Christian Brauner（以及其他人）的反馈导致其消亡（demise）之前，这种方法已经发展到了 v10。
  * Hugh 的反对意见是，搭载（piggybacking ）shmem 对于 KVM 的用例来说没有任何意义，因为 KVM 实际上并不 *想要* shmem 提供的功能。
    * 例如，KVM 使用 `memfd()` 和 `shmem` 来避免直接管理内存，并不是因为 `memfd()` 和 `shmem` 是最佳解决方案，例如 shmem 中的 `read/write/mmap` 之类的东西很重（dead weight）。
  * Christian 指出了实现部分重叠（仅 wrapping shmem 中的 *一些*）的缺陷（flaws），
    * 例如，poking at `inode_operations` 或 `super_operations` 会显示 shmem 内容，但 `address_space_operations` 和 `file_operations` 会显示 KVM 的 overlay。
    * Christian 重重地解释了一下（Paraphrashing heavily），建议 KVM 不要再偷懒了，创建一个合适的 API。
---
### Documentation/virt/kvm/api.rst 新增说明
* `KVM_SET_USER_MEMORY_REGION2` 是 `KVM_SET_USER_MEMORY_REGION` 的扩展，允许将 guest_memfd 内存映射到 guest 虚拟机中。
  * 与 `KVM_SET_USER_MEMORY_REGION` 共享的所有字段完全相同。
  * 用户空间可以在标志中设置 `KVM_MEM_GUEST_MEMFD`，以使 KVM 将内存区域（memory region）绑定到给定的 guest_memfd 范围 `[guest_memfd_offset, guest_memfd_offset + memory_size]`。
  * 目标 guest_memfd 必须指向当前 VM 上通过 `KVM_CREATE_GUEST_MEMFD` 创建的一个文件，并且目标范围不得绑定到任何其他内存区域。
  * 所有标准边界检查均适用（使用常识）。
* 一个 `KVM_MEM_GUEST_MEMFD` 区域必须具有有效的 guest_memfd（私有内存）和 userspace_addr（共享内存）。
  * 然而，userspace_addr 的“有效”仅仅意味着该地址本身必须是合法的用户空间地址。
  * userspace_addr 的后备映射（backing mapping）不需要在 `KVM_SET_USER_MEMORY_REGION2` 时有效/填充，例如 共享内存可以延迟映射/按需分配。
* 将 `gfn` 映射到 guest 虚拟机时，KVM 根据 `gfn` 的 `KVM_MEMORY_ATTRIBUTE_PRIVATE` 状态选择共享与私有，即使用 userspace_addr 与 guest_memfd。
  * 在 VM 创建时，所有内存都是共享的，即所有 `gfns` 的 `PRIVATE` 属性均为“`0`”。
  * 用户空间可以根据需要通过 `KVM_SET_MEMORY_ATTRIBUTES` 切换 `KVM_MEMORY_ATTRIBUTE_PRIVATE` 属性来控制内存是共享还是私有。
* `KVM_CAP_GUEST_MEMFD`
```sh
Capability: KVM_CAP_GUEST_MEMFD
Architectures: none
Type: vm ioctl
Parameters: struct kvm_create_guest_memfd(in)
Returns: 0 on success, <0 on error
```
* `KVM_CREATE_GUEST_MEMFD` 创建一个匿名文件并返回引用它的文件描述符。
  * guest_memfd 文件大致类似（roughly analogous）于通过 `memfd_create()` 创建的文件，例如 guest_memfd 文件位于 RAM 中，具有易失性存储，并且在删除最后一个引用时自动释放。
  * 与“常规”`memfd_create()` 文件不同，guest_memfd 文件绑定到其所属的虚拟机（见下文），无法由用户空间映射、读取或写入，并且无法调整大小（但 guest_memfd 文件支持 `PUNCH_HOLE`）。
```cpp
struct kvm_create_guest_memfd {
   __u64 size;
   __u64 flags;
   __u64 reserved[6];
};
```
* 从概念上讲，支持 guest_memfd 文件的 `inode` 代表物理内存，例如，作为某个事物耦合到虚拟机，而不是“`struct kvm`”。
  * 文件本身绑定到“`struct kvm`”，是该实例的底层内存视图，例如，有效地提供 guest 地址到 host 内存的转换。
  * 这允许使用多个 KVM 结构来管理单个虚拟机的用例，例如，执行虚拟机的 host 内迁移（intrahost migration）时。
* KVM 目前仅支持通过 `KVM_SET_USER_MEMORY_REGION2` 映射 guest_memfd，
  * 更具体地说，通过“`struct kvm_userspace_memory_region2`”中的 `guest_memfd` 和 `guest_memfd_offset` 字段映射 guest_memfd，其中 `guest_memfd_offset` 是（批注：该 memory region）在 `guest_memfd` 实例中的偏移量。
  * 对于给定的 guest_memfd 文件，每页最多可以有一个映射，即不允许将多个内存区域（memory regions）绑定到 **单个 guest_memfd 范围**（可以将任意数量的内存区域绑定到单个 guest_memfd 文件，但绑定范围 **不得重叠**）。

### 代码修改
* 导出 `anon_inode_create_getfile()` 函数，创建一个匿名 inode（而不是共享 inode），返回指向它对应的 `struct file` 的指针
```diff
diff --git a/fs/anon_inodes.c b/fs/anon_inodes.c
index 42b02dc36474..8dd436ee985b 100644
--- a/fs/anon_inodes.c
+++ b/fs/anon_inodes.c
@@ -183,6 +183,7 @@ struct file *anon_inode_create_getfile(const char *name,
    return __anon_inode_getfile(name, fops, priv, flags,
                    context_inode, true);
 }
+EXPORT_SYMBOL_GPL(anon_inode_create_getfile);

 static int __anon_inode_getfd(const char *name,
                  const struct file_operations *fops,
```
* KVM 初始化 `kvm_init()` 的时候调用 `kvm_gmem_init()` 开启 gmem 的初始化
```diff
@@ -6409,6 +6456,8 @@ int kvm_init(unsigned vcpu_size, unsigned vcpu_align, struct module *module)
    if (WARN_ON_ONCE(r))
        goto err_vfio;

+   kvm_gmem_init(module);
+
    /*
     * Registration _must_ be the very last thing done, as this exposes
     * /dev/kvm to userspace, i.e. all infrastructure must be setup!
```
#### 在 `struct kvm_memory_slot` 引入与 gmem 相关的支持
* `struct kvm_memory_slot` 中添加 `gmem` 域，当配置 `CONFIG_KVM_PRIVATE_MEM` 时有效
```diff
diff --git a/include/linux/kvm_host.h b/include/linux/kvm_host.h
index 68a144cb7dbc..a6de526c0426 100644
--- a/include/linux/kvm_host.h
+++ b/include/linux/kvm_host.h
@@ -589,8 +589,20 @@ struct kvm_memory_slot {
    u32 flags;
    short id;
    u16 as_id;
+
+#ifdef CONFIG_KVM_PRIVATE_MEM
+   struct {
+       struct file __rcu *file;
+       pgoff_t pgoff;
+   } gmem;
+#endif
 };
```
#### 在 `struct kvm_userspace_memory_region2` 引入与 gmem 相关的支持
* 在 `struct kvm_userspace_memory_region2` 中引入用于 guest memory 文件描述符的 `guest_memfd` 和偏移 `guest_memfd_offset` 域

```diff
diff --git a/include/uapi/linux/kvm.h b/include/uapi/linux/kvm.h
index e8d167e54980..2802d10aa88c 100644
--- a/include/uapi/linux/kvm.h
+++ b/include/uapi/linux/kvm.h
@@ -102,7 +102,10 @@ struct kvm_userspace_memory_region2 {
    __u64 guest_phys_addr;
    __u64 memory_size;
    __u64 userspace_addr;
-   __u64 pad[16];
+   __u64 guest_memfd_offset;
+   __u32 guest_memfd;
+   __u32 pad1;
+   __u64 pad2[14];
 };
```
* 引入用于 `kvm_userspace_memory_region::flags` 的 flag bit `KVM_MEM_GUEST_MEMFD`
```diff
 #define KVM_MEM_LOG_DIRTY_PAGES    (1UL << 0)
 #define KVM_MEM_READONLY   (1UL << 1)
+#define KVM_MEM_GUEST_MEMFD    (1UL << 2)
```
* 引入对 memory slot 是否可以被私有的 helper 函数 `kvm_slot_can_be_private()`
```cpp
static inline bool kvm_slot_can_be_private(const struct kvm_memory_slot *slot)
{
    return slot && (slot->flags & KVM_MEM_GUEST_MEMFD);
}
```
#### 引入 KVM 是否支持 gmem 的 capability `KVM_CAP_GUEST_MEMFD`
```diff
@@ -1221,6 +1225,7 @@ struct kvm_ppc_resize_hpt {
 #define KVM_CAP_USER_MEMORY2 231
 #define KVM_CAP_MEMORY_FAULT_INFO 232
 #define KVM_CAP_MEMORY_ATTRIBUTES 233
+#define KVM_CAP_GUEST_MEMFD 234

 #ifdef KVM_CAP_IRQ_ROUTING
```
* `KVM_CAP_GUEST_MEMFD` 可用于 `kvm_vm_ioctl_check_extension_generic()`
  * 如果在 VM 上执行，`KVM_CAP_GUEST_MEMFD` 会精确返回 **该 VM 支持的 guest memfd 的能力**
    * 如果该 arch 有私有内存 `kvm_arch_has_private_mem(kvm)`
  * 如果在系统范围内执行，`KVM_CAP_GUEST_MEMFD` 将返回 **KVM 支持的 guest memfd 的能力** 
```diff
@@ -4844,6 +4875,10 @@ static int kvm_vm_ioctl_check_extension_generic(struct kvm *kvm, long arg)
 #ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
    case KVM_CAP_MEMORY_ATTRIBUTES:
        return kvm_supported_mem_attributes(kvm);
+#endif
+#ifdef CONFIG_KVM_PRIVATE_MEM
+   case KVM_CAP_GUEST_MEMFD:
+       return !kvm || kvm_arch_has_private_mem(kvm);
 #endif
    default:
        break;
```
#### 引入 `ioctl(KVM_CREATE_GUEST_MEMFD)`
* 引入 `KVM_CREATE_GUEST_MEMFD` 的 `ioctl()` command 号和使用它时传递的 `struct kvm_create_guest_memfd` 结构
```cpp
#define KVM_CREATE_GUEST_MEMFD  _IOWR(KVMIO,  0xd4, struct kvm_create_guest_memfd)

struct kvm_create_guest_memfd {
    __u64 size;
    __u64 flags;
    __u64 reserved[6];
};
```
* 在 `kvm_vm_ioctl()` 中引入对收到此命令时的支持，调用创建 guest_memfd 的入口函数 `kvm_gmem_create()`
```diff
@@ -5277,6 +5312,18 @@ static long kvm_vm_ioctl(struct file *filp,
    case KVM_GET_STATS_FD:
        r = kvm_vm_ioctl_get_stats_fd(kvm);
        break;
+#ifdef CONFIG_KVM_PRIVATE_MEM
+   case KVM_CREATE_GUEST_MEMFD: {
+       struct kvm_create_guest_memfd guest_memfd;
+
+       r = -EFAULT;
+       if (copy_from_user(&guest_memfd, argp, sizeof(guest_memfd)))
+           goto out;
+
+       r = kvm_gmem_create(kvm, &guest_memfd);
+       break;
+   }
+#endif
    default:
        r = kvm_arch_vm_ioctl(filp, ioctl, arg);
    }
```
#### 引入 `CONFIG_KVM_PRIVATE_MEM` 内核配置选项
```diff
diff --git a/virt/kvm/Kconfig b/virt/kvm/Kconfig
index 5bd7fcaf9089..08afef022db9 100644
--- a/virt/kvm/Kconfig
+++ b/virt/kvm/Kconfig
@@ -100,3 +100,7 @@ config KVM_GENERIC_MMU_NOTIFIER
 config KVM_GENERIC_MEMORY_ATTRIBUTES
        select KVM_GENERIC_MMU_NOTIFIER
        bool
+
+config KVM_PRIVATE_MEM
+       select XARRAY_MULTI
+       bool
```
* 在 Makefile 中引入对新文件 guest_memfd.c 的编译
```diff
diff --git a/virt/kvm/Makefile.kvm b/virt/kvm/Makefile.kvm
index 2c27d5d0c367..724c89af78af 100644
--- a/virt/kvm/Makefile.kvm
+++ b/virt/kvm/Makefile.kvm
@@ -12,3 +12,4 @@ kvm-$(CONFIG_KVM_ASYNC_PF) += $(KVM)/async_pf.o
 kvm-$(CONFIG_HAVE_KVM_IRQ_ROUTING) += $(KVM)/irqchip.o
 kvm-$(CONFIG_HAVE_KVM_DIRTY_RING) += $(KVM)/dirty_ring.o
 kvm-$(CONFIG_HAVE_KVM_PFNCACHE) += $(KVM)/pfncache.o
+kvm-$(CONFIG_KVM_PRIVATE_MEM) += $(KVM)/guest_memfd.o
```
#### 引入通用的 `kvm_arch_has_private_mem()`
* 引入通用的 `kvm_arch_has_private_mem()`，如果对私有内存的支持被启用，arch 的代码必须定义 `kvm_arch_has_private_mem()`
* include/linux/kvm_host.h
```cpp
/*
 * Arch code must define kvm_arch_has_private_mem if support for private memory
 * is enabled.
 */
#if !defined(kvm_arch_has_private_mem) && !IS_ENABLED(CONFIG_KVM_PRIVATE_MEM)
static inline bool kvm_arch_has_private_mem(struct kvm *kvm)
{
    return false;
}
#endif
```
#### 引入一些其他的 Helpers
* 引入 helper 函数 `kvm_mem_is_private()` 判断传入的 `gfn` 是否是私有的。判断的依据是
  * `CONFIG_KVM_PRIVATE_MEM` 配置是否打开
  * `gfn` 对应的内存属性 xarray 的对应条目是否设置了 `KVM_MEMORY_ATTRIBUTE_PRIVATE` 属性
```cpp
#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
static inline bool kvm_mem_is_private(struct kvm *kvm, gfn_t gfn)
{
    return IS_ENABLED(CONFIG_KVM_PRIVATE_MEM) &&
           kvm_get_memory_attributes(kvm, gfn) & KVM_MEMORY_ATTRIBUTE_PRIVATE;
}
...
#else
static inline bool kvm_mem_is_private(struct kvm *kvm, gfn_t gfn)
{
    return false;
}
#endif /* CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES */
```
* 引入 `kvm_gmem_get_pfn()` 的通用空实现
  * include/linux/kvm_host.h
```cpp
#ifdef CONFIG_KVM_PRIVATE_MEM
int kvm_gmem_get_pfn(struct kvm *kvm, struct kvm_memory_slot *slot,
             gfn_t gfn, kvm_pfn_t *pfn, int *max_order);
#else
static inline int kvm_gmem_get_pfn(struct kvm *kvm,
                   struct kvm_memory_slot *slot, gfn_t gfn,
                   kvm_pfn_t *pfn, int *max_order)
{
    KVM_BUG_ON(1, kvm);
    return -EIO;
}
#endif /* CONFIG_KVM_PRIVATE_MEM */
```
* 在 virt/kvm/kvm_mm.h 引入一些函数声明和空函数实现
```cpp
#ifdef CONFIG_KVM_PRIVATE_MEM
void kvm_gmem_init(struct module *module);
int kvm_gmem_create(struct kvm *kvm, struct kvm_create_guest_memfd *args);
int kvm_gmem_bind(struct kvm *kvm, struct kvm_memory_slot *slot,
          unsigned int fd, loff_t offset);
void kvm_gmem_unbind(struct kvm_memory_slot *slot);
#else
static inline void kvm_gmem_init(struct module *module)
{

}

static inline int kvm_gmem_bind(struct kvm *kvm,
                     struct kvm_memory_slot *slot,
                     unsigned int fd, loff_t offset)
{
    WARN_ON_ONCE(1);
    return -EIO;
}

static inline void kvm_gmem_unbind(struct kvm_memory_slot *slot)
{
    WARN_ON_ONCE(1);
}
#endif /* CONFIG_KVM_PRIVATE_MEM */
```

#### virt/kvm/guest_memfd.c 文件
* 引入 virt/kvm/guest_memfd.c 文件
* 新增本地结构体 `struct kvm_gmem`
```cpp
struct kvm_gmem {
    struct kvm *kvm;
    struct xarray bindings;
    struct list_head entry;
};
```
* 函数 `kvm_gmem_get_folio()` 根据提供的文件偏移 `index` 返回文件对应的 page cache 的 folio
  * **注意**：在把内存交给 guest 前用 `up-to-date` 标志来跟踪是否内存已经被清零
  * 由于没有后备存储，所以 folio 会保持 `up-to-date` 直到被移除
  * TODO：跳过清除页面，当把内存指派给 guest 时由受信任的 firmware 去清除
```cpp
static struct folio *kvm_gmem_get_folio(struct inode *inode, pgoff_t index)
{
    struct folio *folio;
    //查找偏移对应的 folio，如果找不到会创建一个 folio。如果都失败了则返回错误码
    /* TODO: Support huge pages. */
    folio = filemap_grab_folio(inode->i_mapping, index);
    if (IS_ERR_OR_NULL(folio))
        return NULL;

    /*
     * Use the up-to-date flag to track whether or not the memory has been
     * zeroed before being handed off to the guest.  There is no backing
     * storage for the memory, so the folio will remain up-to-date until
     * it's removed.
     *
     * TODO: Skip clearing pages when trusted firmware will do it when
     * assigning memory to the guest.
     */
    if (!folio_test_uptodate(folio)) {
        unsigned long nr_pages = folio_nr_pages(folio);
        unsigned long i;
        //folio 里的 pages 逐页清零
        for (i = 0; i < nr_pages; i++)
            clear_highpage(folio_page(folio, i));
        //设置 folio 首页的 up-to-date 标志位
        folio_mark_uptodate(folio);
    }
    //忽略已访问、已引用和脏标志。因为没有存储可以写回，内存是 unevictable 的
    /*
     * Ignore accessed, referenced, and dirty flags.  The memory is
     * unevictable and there is no storage to write back to.
     */
    return folio;
}
```
* 它的两个调用者可以知道它用于 guest_memfd *分配（内存）空间* 和 *缺页处理*，由此可知 guest_memfd 的 pages 都是哪来的
```cpp
1 virt/kvm/guest_memfd.c|148| <<kvm_gmem_allocate>> folio = kvm_gmem_get_folio(inode, index);
2 virt/kvm/guest_memfd.c|506| <<kvm_gmem_get_pfn>> folio = kvm_gmem_get_folio(file_inode(file), index);
```
* 新增 `kvm_gmem_invalidate_begin()` 用于 invalidate 给定范围的 guest memory，取消映射
  * `slot->gmem.pgoff` 是 slot 在 guest_memfd 文件中的偏移，来自于 `struct kvm_userspace_memory_region2.guest_memfd_offset`
  * 如果有实质的动作发生，会刷新 TLB
```cpp
static void kvm_gmem_invalidate_begin(struct kvm_gmem *gmem, pgoff_t start,
                      pgoff_t end)
{
    bool flush = false, found_memslot = false;
    struct kvm_memory_slot *slot;
    struct kvm *kvm = gmem->kvm;
    unsigned long index;
    //迭代 gmem->bindings xarray 的指定范围的条目，即 slot。范围可能跨越多个 slot，所以要分段 invalidate
    xa_for_each_range(&gmem->bindings, index, slot, start, end - 1) {
        pgoff_t pgoff = slot->gmem.pgoff;
        //每段 slot->base_gfn 并不一定是从文件的起始位置开始算起，pgoff 则是 slot 在 guest_memfd 文件中的偏移
        struct kvm_gfn_range gfn_range = {
            .start = slot->base_gfn + max(pgoff, start) - pgoff,
            .end = slot->base_gfn + min(pgoff + slot->npages, end) - pgoff,
            .slot = slot,
            .may_block = true,
        };

        if (!found_memslot) {
            found_memslot = true;

            KVM_MMU_LOCK(kvm);
            kvm_mmu_invalidate_begin(kvm);
        }
        //取消映射
        flush |= kvm_mmu_unmap_gfn_range(kvm, &gfn_range);
    }
    //如发生过实质动作，刷 TLB
    if (flush)
        kvm_flush_remote_tlbs(kvm);

    if (found_memslot)
        KVM_MMU_UNLOCK(kvm);
}
```
* `kvm_gmem_invalidate_begin()` 在打洞、释放和处理错误 folio 的时候被调用
```c
1 virt/kvm/guest_memfd.c|112| <<kvm_gmem_punch_hole>> kvm_gmem_invalidate_begin(gmem, start, end);
2 virt/kvm/guest_memfd.c|223| <<kvm_gmem_release>> kvm_gmem_invalidate_begin(gmem, 0, -1ul);
3 virt/kvm/guest_memfd.c|282| <<kvm_gmem_error_folio>> kvm_gmem_invalidate_begin(gmem, start, end);
```
* 新增 `kvm_gmem_invalidate_end()` 函数也会在以上场景被调用到，调用 `kvm_mmu_invalidate_end()` 对应 `kvm_gmem_invalidate_begin()` 中的 `kvm_mmu_invalidate_begin()`
```cpp
static void kvm_gmem_invalidate_end(struct kvm_gmem *gmem, pgoff_t start,
                    pgoff_t end)
{
    struct kvm *kvm = gmem->kvm;

    if (xa_find(&gmem->bindings, &start, end - 1, XA_PRESENT)) {
        KVM_MMU_LOCK(kvm);
        kvm_mmu_invalidate_end(kvm);
        KVM_MMU_UNLOCK(kvm);
    }
}
```
##### 文件操作 `struct file_operations kvm_gmem_fops`
* 引入 guest_memfd 的文件操作 `struct file_operations kvm_gmem_fops`
  * `.open` 是通用的文件打开操作
  * `.release` 最后一个文件引用不存在时调用
  * `.fallocate` 允许给该文件分配空间和打洞等操作
```cpp
static struct file_operations kvm_gmem_fops = {
    .open       = generic_file_open,
    .release    = kvm_gmem_release,
    .fallocate  = kvm_gmem_fallocate,
};
```
* `kvm_gmem_init()` 就是简单地设定 guest_memfd 的 `kvm_gmem_fops.owner` 为 `module`，比如 VMX module
```cpp
void kvm_gmem_init(struct module *module)
{
    kvm_gmem_fops.owner = module;
}
```
* 新增 `kvm_gmem_punch_hole()` 函数在 gmem 上打洞，提供 guest_memfd 的 `inode`，在指定偏移 `offset` 处开始打一个长度为 `len` 的洞
```cpp
static long kvm_gmem_punch_hole(struct inode *inode, loff_t offset, loff_t len)
{
    struct list_head *gmem_list = &inode->i_mapping->i_private_list;
    pgoff_t start = offset >> PAGE_SHIFT;       //起始页帧
    pgoff_t end = (offset + len) >> PAGE_SHIFT; //结束页帧
    struct kvm_gmem *gmem;

    /*
     * Bindings must be stable across invalidation to ensure the start+end
     * are balanced.
     */
    filemap_invalidate_lock(inode->i_mapping);
    //遍历 guest_memfd 上的 struct kvm_gmem 实例，取消映射
    list_for_each_entry(gmem, gmem_list, entry)
        kvm_gmem_invalidate_begin(gmem, start, end);
    //核心函数，从 page cache 中把指定范围的 pages 移除
    truncate_inode_pages_range(inode->i_mapping, offset, offset + len - 1);
    //对应地要调用 invalidate 的 end 函数
    list_for_each_entry(gmem, gmem_list, entry)
        kvm_gmem_invalidate_end(gmem, start, end);

    filemap_invalidate_unlock(inode->i_mapping);

    return 0;
}
```
* `kvm_gmem_allocate()` 在 `inode` 相应的 guest_memfd 上分配 page cache，*起止范围* 根据 `offset` 和 `len` 得到
```cpp
static long kvm_gmem_allocate(struct inode *inode, loff_t offset, loff_t len)
{
    struct address_space *mapping = inode->i_mapping;
    pgoff_t start, index, end;
    int r;
    //不能超过文件大小
    /* Dedicated guest is immutable by default. */
    if (offset + len > i_size_read(inode))
        return -EINVAL;

    filemap_invalidate_lock_shared(mapping);
    //偏移和长度转为起始和结束页帧号
    start = offset >> PAGE_SHIFT;
    end = (offset + len) >> PAGE_SHIFT;
    //遍历范围内页帧号
    r = 0;
    for (index = start; index < end; ) {
        struct folio *folio;
        //如果收到信号，会导致分配过程被中断，用户态程序需要处理好这次中断
        if (signal_pending(current)) {
            r = -EINTR;
            break;
        }
        //根据 index 得到 folio，这个函数是带分配的，底层动作会分配 page cache
        folio = kvm_gmem_get_folio(inode, index);
        if (!folio) {
            r = -ENOMEM;
            break;
        }
        //得到下一个 folio 的索引
        index = folio_next_index(folio);

        folio_unlock(folio);
        folio_put(folio);

        /* 64-bit only, wrapping the index should be impossible. */
        if (WARN_ON_ONCE(!index))
            break;
        //加入一个条件调度点，不一定要等内核抢占的 4 个情况
        cond_resched();
    }

    filemap_invalidate_unlock_shared(mapping);

    return r;
}
```
* `kvm_gmem_fallocate()` 针对的是操作文件空间的 `fallocate()` 系统调用，缺省操作是在文件范围内的指定偏移处 `offset` 分配指定长度 `len` 的磁盘空间
  * 可以支持打洞模式 `FALLOC_FL_PUNCH_HOLE` mode
```cpp
static long kvm_gmem_fallocate(struct file *file, int mode, loff_t offset,
                   loff_t len)
{
    int ret;
    //不允许通过该操作减小文件大小
    if (!(mode & FALLOC_FL_KEEP_SIZE))
        return -EOPNOTSUPP;
    //仅支持固定大小的分配和打洞模式
    if (mode & ~(FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE))
        return -EOPNOTSUPP;
    //偏移和长度必须对齐
    if (!PAGE_ALIGNED(offset) || !PAGE_ALIGNED(len))
        return -EINVAL;
    //根据模式选择不同的分支进行处理
    if (mode & FALLOC_FL_PUNCH_HOLE)
        ret = kvm_gmem_punch_hole(file_inode(file), offset, len);
    else
        ret = kvm_gmem_allocate(file_inode(file), offset, len);
    //修改文件时处理强制的 VFS 更改
    if (!ret)
        file_modified(file);
    return ret;
}
```
* `kvm_gmem_release()` 释放 guest_memfd 文件
* `mutex_lock(&kvm->slots_lock)` 防止同时尝试“解除绑定”一个 memslot。
  * 这是对文件的最后一个引用，因此无法创建新的绑定，但是为已有绑定解除引用 slot 需要被保护，免受 memslot 更新，
  * 特别是这样解除绑定不会竞争和释放 memslot（`kvm_gmem_get_file()` 将返回 `NULL`）。
* 所有正在进行的操作都结束了，可以创建新的绑定。
  * 去除该文件指向的所有 SPTE。
  * 不要释放后备内存，因为它的生命周期与 inode 相关，而不是与文件相关。
```cpp
static int kvm_gmem_release(struct inode *inode, struct file *file)
{
    struct kvm_gmem *gmem = file->private_data;
    struct kvm_memory_slot *slot;
    struct kvm *kvm = gmem->kvm;
    unsigned long index;

    /*
     * Prevent concurrent attempts to *unbind* a memslot.  This is the last
     * reference to the file and thus no new bindings can be created, but
     * dereferencing the slot for existing bindings needs to be protected
     * against memslot updates, specifically so that unbind doesn't race
     * and free the memslot (kvm_gmem_get_file() will return NULL).
     */
    mutex_lock(&kvm->slots_lock);

    filemap_invalidate_lock(inode->i_mapping);
    //把 bindings xarray 的条目全部设为 NULL
    xa_for_each(&gmem->bindings, index, slot)
        rcu_assign_pointer(slot->gmem.file, NULL);

    synchronize_rcu();

    /*
     * All in-flight operations are gone and new bindings can be created.
     * Zap all SPTEs pointed at by this file.  Do not free the backing
     * memory, as its lifetime is associated with the inode, not the file.
     */
    kvm_gmem_invalidate_begin(gmem, 0, -1ul);
    kvm_gmem_invalidate_end(gmem, 0, -1ul);
    //将 gmem 从其 address_space 的 i_private_list 链表上删除
    list_del(&gmem->entry);

    filemap_invalidate_unlock(inode->i_mapping);

    mutex_unlock(&kvm->slots_lock);
    //销毁 bindings xarray
    xa_destroy(&gmem->bindings);
    kfree(gmem);

    kvm_put_kvm(kvm);

    return 0;
}
```
##### address_space 操作 `struct address_space_operations kvm_gmem_aops`
* 引入 guest_memfd 的 address_space 操作 `struct address_space_operations kvm_gmem_aops`
```cpp
static const struct address_space_operations kvm_gmem_aops = {
    .dirty_folio = noop_dirty_folio,
    .migrate_folio  = kvm_gmem_migrate_folio,
    .error_remove_folio = kvm_gmem_error_folio,
};
```
* 目前还不支持迁移 guest_memfd，所以 `.migrate_folio` 的回调 `kvm_gmem_migrate_folio()` 是出错函数
```cpp
static int kvm_gmem_migrate_folio(struct address_space *mapping,
                  struct folio *dst, struct folio *src,
                  enum migrate_mode mode)
{
    WARN_ON_ONCE(1);
    return -EINVAL;
}
```
##### guest_memfd 页面出现 MCE 时的处理
* 绑定完之后，如果 guest_memfd 范围内的页发生变化时，比如 MCE，会通过 `struct address_space` page cache 的文件映射操作符的`mapping->a_ops->error_remove_page(mapping, p)` 回调解除映射
```cpp
memory_failure()
-> hwpoison_user_mappings()
-> identify_page_state(pfn, p, page_flags) // 在 struct page_state error_states[] 中找到 ps
   -> page_action(ps, p, pfn)
      -> ps->action(ps, p) // ps->action 回调的定义在 struct page_state error_states[] 中
      => me_pagecache_dirty()
         -> me_pagecache_clean(ps, p)
            -> truncate_error_page(p, page_to_pfn(p), mapping)
               -> mapping->a_ops->error_remove_page(mapping, p)
               => kvm_gmem_error_folio()
                  -> kvm_gmem_invalidate_begin()
                     -> kvm_mmu_invalidate_begin()
                     -> kvm_mmu_unmap_gfn_range()
                     -> kvm_flush_remote_tlbs()
                  -> kvm_gmem_invalidate_end()
                     -> kvm_mmu_invalidate_end()
```
* 在创建 guest_mem 时，其核心函数 `__kvm_gmem_create()`
  * 将 `inode->i_mapping->a_ops = &kvm_gmem_aops`
  * 将 `gmem->entry` 挂在 `inode->i_mapping->i_private_list` 上：`list_add(&gmem->entry, &inode->i_mapping->i_private_list)`
* `kvm_gmem_error_folio()` 遍历 guest_memfd 的 `gmem_list`，借助 `kvm_gmem_invalidate_begin()` 取消范围映射（unmap）
* 不要截断范围（truncate the range），对错误采取什么操作是用户空间的决定（假设架构支持优雅地处理内存错误）。
  * 如果/当 guest 尝试访问中毒页面时，`kvm_gmem_get_pfn()` 将返回 `-EHWPOISON`，此时 KVM 可以终止 VM 或将错误传播到用户空间。
  * 译注：这里主要对比 `kvm_gmem_punch_hole()` 而言，少了 `truncate_inode_pages_range()` 调用，不把 poisoned page 从 page cache 中删掉，因为前面已经设置了该 page 设置 `PageHWPoison` 标志了
* 译注：在 `memory_failure()` 路径上设置页面 `PageHWPoison` 标志并在该函数取消映射，因此再（异步）访问该页面时会发生缺页。
  * KVM 的缺页处理函数调用 `kvm_faultin_pfn_private() -> kvm_gmem_get_pfn()` 返回 `-EHWPOISON`
  * 接着 `kvm_faultin_pfn_private() -> kvm_prepare_memory_fault_exit()` 会设置好给用户空间的 KVM 退出原因等信息，然后返回 `-EHWPOISON`
  * 这样就将错误传播到用户空间
* 从 `memory_failure() -> hwpoison_user_mappings() ---> mmu_notifier_invalidate_range_start() ---> kvm_mmu_notifier_invalidate_range_start()` 路径也能走到 `kvm_mmu_unmap_gfn_range()`，这与上面的那个路径岂不是殊途同归？！
```cpp
static int kvm_gmem_error_folio(struct address_space *mapping, struct folio *folio)
{
    struct list_head *gmem_list = &mapping->i_private_list;
    struct kvm_gmem *gmem;
    pgoff_t start, end;

    filemap_invalidate_lock_shared(mapping);

    start = folio->index;
    end = start + folio_nr_pages(folio);

    list_for_each_entry(gmem, gmem_list, entry)
        kvm_gmem_invalidate_begin(gmem, start, end);

    /*
     * Do not truncate the range, what action is taken in response to the
     * error is userspace's decision (assuming the architecture supports
     * gracefully handling memory errors).  If/when the guest attempts to
     * access a poisoned page, kvm_gmem_get_pfn() will return -EHWPOISON,
     * at which point KVM can either terminate the VM or propagate the
     * error to userspace.
     */

    list_for_each_entry(gmem, gmem_list, entry)
        kvm_gmem_invalidate_end(gmem, start, end);

    filemap_invalidate_unlock_shared(mapping);

    return MF_DELAYED;
}
```

##### inode 操作 `struct inode_operations kvm_gmem_iops`
* 引入 guest_memfd 的 inode 操作 `struct inode_operations kvm_gmem_iops`
  * 仅支持 `.getattr` 和 `.setattr` 两个操作
  * 而且 `.setattr` 的实现返回错误，等于不允许该操作
  * `.getattr` 的实现也仅仅是调用 `generic_fillattr()` 填充最通用的 `struct kstat stat`
```cpp
static int kvm_gmem_getattr(struct mnt_idmap *idmap, const struct path *path,
                struct kstat *stat, u32 request_mask,
                unsigned int query_flags)
{
    struct inode *inode = path->dentry->d_inode;

    generic_fillattr(idmap, request_mask, inode, stat);
    return 0;
}

static int kvm_gmem_setattr(struct mnt_idmap *idmap, struct dentry *dentry,
                struct iattr *attr)
{
    return -EINVAL;
}
static const struct inode_operations kvm_gmem_iops = {
    .getattr    = kvm_gmem_getattr,
    .setattr    = kvm_gmem_setattr,
};
```

##### 创建 guest_mem 的核心函数
* `__kvm_gmem_create()` 是创建 guest_mem 的核心函数
```cpp
static int __kvm_gmem_create(struct kvm *kvm, loff_t size, u64 flags)
{
    const char *anon_name = "[kvm-gmem]";
    struct kvm_gmem *gmem;
    struct inode *inode;
    struct file *file;
    int fd, err;
    //得到一个新的文件描述符号
    fd = get_unused_fd_flags(0);
    if (fd < 0)
        return fd;
    //分配类型为 struct kvm_gmem 的一个实例 gmem
    gmem = kzalloc(sizeof(*gmem), GFP_KERNEL);
    if (!gmem) {
        err = -ENOMEM;
        goto err_fd;
    }
    //创建一个匿名 inode，文件操作为 kvm_gmem_fops，file->private_data 为 gmem，可读写，返回文件结构
    file = anon_inode_create_getfile(anon_name, &kvm_gmem_fops, gmem,
                     O_RDWR, NULL);
    if (IS_ERR(file)) {
        err = PTR_ERR(file);
        goto err_gmem;
    }

    file->f_flags |= O_LARGEFILE;

    inode = file->f_inode;
    WARN_ON(file->f_mapping != inode->i_mapping); //在 __anon_inode_getfile() 赋值

    inode->i_private = (void *)(unsigned long)flags;
    inode->i_op = &kvm_gmem_iops;                    //设置 inode 的操作
    inode->i_mapping->a_ops = &kvm_gmem_aops;        //设置 address_space 的操作
    inode->i_mode |= S_IFREG;                        //文件类型为 regular 文件
    inode->i_size = size;                            //设置文件大小
    mapping_set_gfp_mask(inode->i_mapping, GFP_HIGHUSER); //gfp_mask 为 GFP_USER | __GFP_HIGHMEM
    mapping_set_unmovable(inode->i_mapping); //设置 address_space 的 flags 为不可移动和 AS_UNEVICTABLE
    /* Unmovable mappings are supposed to be marked unevictable as well. */
    WARN_ON_ONCE(!mapping_unevictable(inode->i_mapping)); //刚设完又检查？有异步事件？
    //增加 KVM 的引用计数
    kvm_get_kvm(kvm);
    gmem->kvm = kvm;  //gmem 的 kvm 域回指 kvm 实例便于反向查找
    xa_init(&gmem->bindings); //初始化绑定 xarray，其“键”为 guest_memfd 范围内的 GFN，“值”为 GFN 所在的 slot
    list_add(&gmem->entry, &inode->i_mapping->i_private_list); //gmem 挂在 inode 的 address_space 的 i_private_list 链表上
    //将 fd 装入进程的打开文件数组
    fd_install(fd, file);
    return fd;

err_gmem:
    kfree(gmem);
err_fd:
    put_unused_fd(fd);
    return err;
}
```
* 从这个函数往下深挖可以看到，gmem 的内存来源是 `anon_inodefs` 类型的文件系统，而不是 restrictedmem 时期对 `shmem` 的封装了
```cpp
static struct file_system_type anon_inode_fs_type = {
    .name       = "anon_inodefs",
    .init_fs_context = anon_inodefs_init_fs_context,
    .kill_sb    = kill_anon_super,
};
```

* `kvm_gmem_create()` 是对 `__kvm_gmem_create()` 的封装，对入参 `struct kvm_create_guest_memfd *args` 做健全性检查
```cpp
int kvm_gmem_create(struct kvm *kvm, struct kvm_create_guest_memfd *args)
{
    loff_t size = args->size;
    u64 flags = args->flags;
    u64 valid_flags = 0;
    //目前 flags 必须是全 0，否则返回错误
    if (flags & ~valid_flags)
        return -EINVAL;
    //size 必须页对齐
    if (size <= 0 || !PAGE_ALIGNED(size))
        return -EINVAL;

    return __kvm_gmem_create(kvm, size, flags);
}
```

##### guest_memfd 文件的绑定
* 如果 `slot->gmem.file` 已经关闭，则不返回；最后一次 `fput()` 和 `kvm_gmem_release()` 清除 `slot->gmem.file` 之间可能有一段时间，并且您不想在此期间打转。
```cpp
static inline struct file *kvm_gmem_get_file(struct kvm_memory_slot *slot)
{
    /*
     * Do not return slot->gmem.file if it has already been closed;
     * there might be some time between the last fput() and when
     * kvm_gmem_release() clears slot->gmem.file, and you do not
     * want to spin in the meanwhile.
     */
    return get_file_active(&slot->gmem.file);
}
```
* `kvm_gmem_bind()` 由 `__kvm_set_memory_region(kvm, new, mem->guest_memfd, mem->guest_memfd_offset)` 调用
  * 该函数在 memory slot 和 guest_memfd 范围内的 GFN 之间建立绑定关系
  * `bindings`：绑定 xarray，其 **键** 为 guest_memfd 范围内的 GFN，**值** 为 GFN 所在的 slot
  * 一个 guest_memfd 可能覆盖多个 memory slots，`__kvm_set_memory_region()` 在新建 memory slot 的路径上，每新建一个用于私有内存的 memory slot 随即建立与 guest_memfd 的绑定关系
```cpp
int kvm_gmem_bind(struct kvm *kvm, struct kvm_memory_slot *slot,
          unsigned int fd, loff_t offset)
{
    loff_t size = slot->npages << PAGE_SHIFT; //slot 的页帧总数
    unsigned long start, end;
    struct kvm_gmem *gmem;
    struct inode *inode;
    struct file *file;
    int r = -EINVAL;
    //slot->gmem.pgoff 类型应为 gfn_t，否则可能引起编译错误
    BUILD_BUG_ON(sizeof(gfn_t) != sizeof(slot->gmem.pgoff));
    //文件描述符转文件
    file = fget(fd);
    if (!file)
        return -EBADF;
    //guest_memfd 文件的操作必须为 kvm_gmem_fops
    if (file->f_op != &kvm_gmem_fops)
        goto err;
    //file->private_data 见 __kvm_gmem_create() 调用 anon_inode_create_getfile()
    gmem = file->private_data;
    if (gmem->kvm != kvm) //gmem 的 kvm 必须是创建它的 kvm
        goto err;
    //file 转 inode
    inode = file_inode(file);
    //offset 的健全性检查：不能是负数，必须对齐，offset 和页帧数的和不能超出 inode 大小的范围
    if (offset < 0 || !PAGE_ALIGNED(offset) ||
        offset + size > i_size_read(inode))
        goto err;

    filemap_invalidate_lock(inode->i_mapping);
    //根据内存区域在 guest_memfd 中的偏移和大小得到它在 guest_memfd 中的起始页帧和结束页帧
    start = offset >> PAGE_SHIFT;
    end = start + slot->npages;
    //如果 bindings xarray 不为空且在范围内存在条目，即新内存范围与 guest_memfd 已有内存范围重叠，则绑定出错
    if (!xa_empty(&gmem->bindings) &&
        xa_find(&gmem->bindings, &start, end - 1, XA_PRESENT)) {
        filemap_invalidate_unlock(inode->i_mapping);
        goto err;
    }
    //如果 bindings xarray 为空或者范围内不存在条目，记录下 slot 对应的 gmem file 指针和 guest_memfd 中的起始页帧号
    /*
     * No synchronize_rcu() needed, any in-flight readers are guaranteed to
     * be see either a NULL file or this new file, no need for them to go
     * away.
     */
    rcu_assign_pointer(slot->gmem.file, file);
    slot->gmem.pgoff = start;
    //设定 bindings xarray 起止范围内的条目的值为 slot，键为其在 guest_memfd 中的起始到终止 GFN
    xa_store_range(&gmem->bindings, start, end - 1, slot, GFP_KERNEL);
    filemap_invalidate_unlock(inode->i_mapping);
    //即使成功，也要删除对该文件的引用。该文件 pins KVM，而不是相反。如果在 memslots 被销毁之前关闭文件，则 active 绑定将失效。
    /*
     * Drop the reference to the file, even on success.  The file pins KVM,
     * not the other way 'round.  Active bindings are invalidated if the
     * file is closed before memslots are destroyed.
     */
    r = 0;
err:
    fput(file);
    return r;
}
```
* `kvm_gmem_unbind()` 解除 guest_memfd 中 GFN 与 memory slot 的绑定关系
```cpp
void kvm_gmem_unbind(struct kvm_memory_slot *slot)
{
    unsigned long start = slot->gmem.pgoff;
    unsigned long end = start + slot->npages;
    struct kvm_gmem *gmem;
    struct file *file;
    //如果底层文件已经关闭（或正在关闭），则无需执行任何操作， kvm_gmem_release() 会使所有绑定无效。
    /*
     * Nothing to do if the underlying file was already closed (or is being
     * closed right now), kvm_gmem_release() invalidates all bindings.
     */
    file = kvm_gmem_get_file(slot);
    if (!file)
        return;
    //file->private_data 见 __kvm_gmem_create() 调用 anon_inode_create_getfile()
    gmem = file->private_data;
    //设置 guest_memfd bindings xarray 起止 GFN 范围内的条目的值为 NULL
    filemap_invalidate_lock(file->f_mapping);
    xa_store_range(&gmem->bindings, start, end - 1, NULL, GFP_KERNEL);
    rcu_assign_pointer(slot->gmem.file, NULL); //清空记录的文件结构
    synchronize_rcu();                         //同步 RCU 更新完成
    filemap_invalidate_unlock(file->f_mapping);

    fput(file);
}
```
* 回过头来看 `kvm_gmem_invalidate_begin()` 和 `kvm_gmem_invalidate_end()` 会在打洞、释放和处理错误 folio 的时候被调用，
  * 然后他们又会根据 bindings xarray 中的属于 invalidate 范围的条目，找回其所属 slot
  * 之后根据 `slot->gmem.file` 和 `slot->base_gfn` 构造出传给取消映射函数 `kvm_mmu_unmap_gfn_range()` 的范围 `struct kvm_gfn_range gfn_range` 的起始和结束地址
* 取消 `kvm_mmu_unmap_gfn_range()` 函数的本地限制
```diff
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index f1a575d39b3b..8f46d757a2c5 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -791,7 +791,7 @@ void kvm_mmu_invalidate_range_add(struct kvm *kvm, gfn_t start, gfn_t end)
    }
 }

-static bool kvm_mmu_unmap_gfn_range(struct kvm *kvm, struct kvm_gfn_range *range)
+bool kvm_mmu_unmap_gfn_range(struct kvm *kvm, struct kvm_gfn_range *range)
 {
    kvm_mmu_invalidate_range_add(kvm, range->start, range->end);
    return kvm_unmap_gfn_range(kvm, range);
```
* 释放 memslot 的时候 `kvm_free_memslot()`，如果 slot 带有 `KVM_MEM_GUEST_MEMFD` 标志，解除 guest_memfd 中 GFN 与 slot 的绑定
  * 因为 guest_memfd 可能对应多个 memory slots，因此关闭 guest_memfd 才需要最终释放 binding。而只是释放一个 memslot 只需要解绑，不一样的方向
```diff
@@ -1027,6 +1027,9 @@ static void kvm_destroy_dirty_bitmap(struct kvm_memory_slot *memslot)
 /* This does not remove the slot from struct kvm_memslots data structures */
 static void kvm_free_memslot(struct kvm *kvm, struct kvm_memory_slot *slot)
 {
+   if (slot->flags & KVM_MEM_GUEST_MEMFD)
+       kvm_gmem_unbind(slot);
+
    kvm_destroy_dirty_bitmap(slot);

    kvm_arch_free_memslot(kvm, slot);
```
* `check_memory_region_flags()` 会被 `kvm_set_memory_region() -> __kvm_set_memory_region()` 调用，检查内存区域的 flags 是否都有效
  * 如果 arch 支持私有内存，*有效标志`valid_flags`* 会包含 `KVM_MEM_GUEST_MEMFD`
  * 因为当前还不支持私有页面的 dirty logging，所以当要检查内存区域带有 `KVM_MEM_GUEST_MEMFD` 时，将 `KVM_MEM_LOG_DIRTY_PAGES` 从有效标志中剔除
```diff
-static int check_memory_region_flags(const struct kvm_userspace_memory_region2 *mem)
+static int check_memory_region_flags(struct kvm *kvm,
+                    const struct kvm_userspace_memory_region2 *mem)
 {
    u32 valid_flags = KVM_MEM_LOG_DIRTY_PAGES;

+   if (kvm_arch_has_private_mem(kvm))
+       valid_flags |= KVM_MEM_GUEST_MEMFD;
+
+   /* Dirty logging private memory is not currently supported. */
+   if (mem->flags & KVM_MEM_GUEST_MEMFD)
+       valid_flags &= ~KVM_MEM_LOG_DIRTY_PAGES;
+
 #ifdef __KVM_HAVE_READONLY_MEM
    valid_flags |= KVM_MEM_READONLY;
 #endif
```
* 修改 `__kvm_set_memory_region()`
  * `check_memory_region_flags()` 增加入参 `kvm`，看上面会用到
  * 如果内存区域带有 `KVM_MEM_GUEST_MEMFD` 标志，那么其在 guest_memfd 中的偏移需要页对齐或者内存区域的结束不能发生回绕
  * 私有的 memslots 是不可变的（immutable），如果修改一个 **已有的** 内存区域带有 `KVM_MEM_GUEST_MEMFD` 标志，则返回错误
  * 如果是一个 **新的** 内存区域带有 `KVM_MEM_GUEST_MEMFD` 标志，这里是一个合适的时机将 memslot 与 guest_memfd 进行绑定
  * 加入出错解绑的逻辑
```diff
@@ -2018,7 +2029,7 @@ int __kvm_set_memory_region(struct kvm *kvm,
    int as_id, id;
    int r;

-   r = check_memory_region_flags(mem);
+   r = check_memory_region_flags(kvm, mem);
    if (r)
        return r;

@@ -2037,6 +2048,10 @@ int __kvm_set_memory_region(struct kvm *kvm,
         !access_ok((void __user *)(unsigned long)mem->userspace_addr,
            mem->memory_size))
        return -EINVAL;
+   if (mem->flags & KVM_MEM_GUEST_MEMFD &&
+       (mem->guest_memfd_offset & (PAGE_SIZE - 1) ||
+        mem->guest_memfd_offset + mem->memory_size < mem->guest_memfd_offset))
+       return -EINVAL;
    if (as_id >= KVM_ADDRESS_SPACE_NUM || id >= KVM_MEM_SLOTS_NUM)
        return -EINVAL;
    if (mem->guest_phys_addr + mem->memory_size < mem->guest_phys_addr)
@@ -2075,6 +2090,9 @@ int __kvm_set_memory_region(struct kvm *kvm,
        if ((kvm->nr_memslot_pages + npages) < kvm->nr_memslot_pages)
            return -EINVAL;
    } else { /* Modify an existing slot. */
+       /* Private memslots are immutable, they can only be deleted. */
+       if (mem->flags & KVM_MEM_GUEST_MEMFD)
+           return -EINVAL;
        if ((mem->userspace_addr != old->userspace_addr) ||
            (npages != old->npages) ||
            ((mem->flags ^ old->flags) & KVM_MEM_READONLY))
@@ -2103,10 +2121,23 @@ int __kvm_set_memory_region(struct kvm *kvm,
    new->npages = npages;
    new->flags = mem->flags;
    new->userspace_addr = mem->userspace_addr;
+   if (mem->flags & KVM_MEM_GUEST_MEMFD) {
+       r = kvm_gmem_bind(kvm, new, mem->guest_memfd, mem->guest_memfd_offset);
+       if (r)
+           goto out;
+   }

    r = kvm_set_memslot(kvm, old, new, change);
    if (r)
-       kfree(new);
+       goto out_unbind;
+
+   return 0;
+
+out_unbind:
+   if (mem->flags & KVM_MEM_GUEST_MEMFD)
+       kvm_gmem_unbind(new);
+out:
+   kfree(new);
    return r;
 }
 EXPORT_SYMBOL_GPL(__kvm_set_memory_region);
```

##### gmem GFN 转 PFN 函数 `kvm_gmem_get_pfn()`
* 引入函数 `kvm_gmem_get_pfn()` 将传入的 gmem GFN 转为其映射的 PFN
  * `index = gfn - slot->base_gfn + slot->gmem.pgoff` 将 `gfn` 转为其在 guest_memfd 文件中的索引 `index`
    * `gfn - slot->base_gfn` 计算 GFN 在 slot 中的 delta（单位为 page）
    * `slot->gmem.pgoff` 是 slot 在 guest_memfd 中的偏移（单位为 page），加上 delta 即输入的 `gfn` 在 guest_memfd 中的索引
  * 根据索引得到 guest_memfd 的 page cache 中 `folio`
  * `folio` 再转 `page`
  * `page` 再转 `pfn`
```cpp
int kvm_gmem_get_pfn(struct kvm *kvm, struct kvm_memory_slot *slot,
             gfn_t gfn, kvm_pfn_t *pfn, int *max_order)
{
    pgoff_t index = gfn - slot->base_gfn + slot->gmem.pgoff;
    struct kvm_gmem *gmem;
    struct folio *folio;
    struct page *page;
    struct file *file;
    int r;
    //从 slot 得到 gmem 文件结构
    file = kvm_gmem_get_file(slot);
    if (!file)
        return -EFAULT;
    //gmem 文件结构的私有数据是 gmem 结构
    gmem = file->private_data;
    //binding xarray 的值是 slot，这里比较传入的 slot 与 xarray 记录的 gfn 对应的 slot 是不是一个
    if (WARN_ON_ONCE(xa_load(&gmem->bindings, index) != slot)) {
        r = -EIO;
        goto out_fput;
    }
    //文件结构转 inode，再根据 index 得到 folio
    folio = kvm_gmem_get_folio(file_inode(file), index);
    if (!folio) {
        r = -ENOMEM;
        goto out_fput;
    }
    //测试 folio 是否有错误
    if (folio_test_hwpoison(folio)) {
        r = -EHWPOISON;
        goto out_unlock;
    }
    //folio 再转 page
    page = folio_file_page(folio, index);
    //page 转 PFN 作为输出
    *pfn = page_to_pfn(page);
    if (max_order)
        *max_order = 0;

    r = 0;

out_unlock:
    folio_unlock(folio);
out_fput:
    fput(file);

    return r;
}
EXPORT_SYMBOL_GPL(kvm_gmem_get_pfn);
```

## [PATCH 16/34] KVM: x86: "Reset" vcpu->run->exit_reason early in KVM_RUN
* 在 `KVM_RUN` 早期将 `run->exit_reason` 初始化为 `KVM_EXIT_UNKNOWN`，以减少退出到用户空间时 *看起来（appears）* 是有效的但实则陈旧的（stale） `run->exit_reason` 的可能性。
* 为了支持 fd-based 的 guest memory（没有相应用户空间虚拟地址的 guest 内存），KVM 将因各种各样的内存相关错误而退出到用户空间，用户空间 *可能* 能够解决这些错误，而不是使用例如 `BUS_MCEERR_AR`。
  * 在更遥远的未来，当用户空间映射丢失时，即当 fast `gup()` 失败时，KVM 也可能利用相同的功能让用户空间“拦截”并处理内存 fault。
* 由于许多 KVM 与 guest 内存相关的内部 API 使用“`0`”表示 *“成功，继续”* 而不是 *“退出到用户空间”*，
  * 因此向用户空间报告内存 fault/错误将设置 `run->exit_reason` 以及 *`run` 结构中的相应字段* 与 *非零*、*负返回码* 结合使用，例如 `-EFAULT` 或 `-EHWPOISON`。
  * 而且由于 KVM 已经在许多路径中返回 `-EFAULT`，因此 KVM 在不设置 `run->exit_reason` 的情况下返回 `-EFAULT` 的可能性相对较高，在这种情况下，报告 `KVM_EXIT_UNKNOWN` 比报告 `run` 结构中发生的任何退出原因要好得多。
* 请注意，KVM 必须等到 `run->immediate_exit` 服务完成后才能清理 `run->exit_reason`，因为 KVM 的 ABI 是当 `run->immediate_exit` 为 `true` 时，`run->exit_reason` 会在 `KVM_RUN` 中保留。
---
```diff
 arch/x86/kvm/x86.c | 1 +
 1 file changed, 1 insertion(+)

diff --git a/arch/x86/kvm/x86.c b/arch/x86/kvm/x86.c
index 8f9d8939b63b..f661acb01c58 100644
--- a/arch/x86/kvm/x86.c
+++ b/arch/x86/kvm/x86.c
@@ -11082,6 +11082,7 @@ static int vcpu_run(struct kvm_vcpu *vcpu)
 {
    int r;

+   vcpu->run->exit_reason = KVM_EXIT_UNKNOWN;
    vcpu->arch.l1tf_flush_l1d = true;

    for (;;) {
```

## [PATCH 17/34] KVM: x86: Disallow hugepages when memory attributes are mixed
* 禁止创建具有混合内存属性的巨页，例如，共享 versus 私有，因为在这种情况下映射巨页将允许 guest 访问具有错误属性的内存，例如，用共享巨页覆盖私有内存。
* 通过现有的 `disallow_lpage` 域跟踪属性是否混合，但使用“`disallow_lpage`”中的 **最高有效位** 来指示 **巨页具有混合属性**，而不是使用正常的引用计数。
  * 属性是否混合是二元的；他们要么是，要么不是。
  * 尝试将该信息压缩到引用计数中不必要地复杂，因为在更新属性时需要了解混合计数的先前状态。
  * 使用标志意味着 KVM 只需要确保当前状态反映在 memslots 中。
---
* 有关 `lpage_info[]` 的设计见 [TDX 共享和私有内存转换](tdx_share_to_private.md)
* 引入宏 `KVM_LPAGE_MIXED_FLAG` 复用了 `struct kvm_lpage_info.disallow_lpage` 的 *最高有效位*
* `disallow_lpage` 中的 *最高有效位* 跟踪内存属性是否混合，即当前级别的所有 gfns 是否不相同。
* 低阶位用于引用计数，不允许使用巨页的其他情况，例如，如果 KVM 在 gfn 上有一个页表的影子。
* 如果 `disallow_lpage` 的引用计数的变更有 `KVM_LPAGE_MIXED_FLAG` 位，则报警一次
```diff
diff --git a/arch/x86/kvm/mmu/mmu.c b/arch/x86/kvm/mmu/mmu.c
index b2d916f786ca..f5c6b0643645 100644
--- a/arch/x86/kvm/mmu/mmu.c
+++ b/arch/x86/kvm/mmu/mmu.c
@@ -795,16 +795,26 @@ static struct kvm_lpage_info *lpage_info_slot(gfn_t gfn,
    return &slot->arch.lpage_info[level - 2][idx];
 }

+/*
+ * The most significant bit in disallow_lpage tracks whether or not memory
+ * attributes are mixed, i.e. not identical for all gfns at the current level.
+ * The lower order bits are used to refcount other cases where a hugepage is
+ * disallowed, e.g. if KVM has shadow a page table at the gfn.
+ */
+#define KVM_LPAGE_MIXED_FLAG   BIT(31)
+
 static void update_gfn_disallow_lpage_count(const struct kvm_memory_slot *slot,
                        gfn_t gfn, int count)
 {
    struct kvm_lpage_info *linfo;
-   int i;
+   int old, i;

    for (i = PG_LEVEL_2M; i <= KVM_MAX_HUGEPAGE_LEVEL; ++i) {
        linfo = lpage_info_slot(gfn, slot, i);
+
+       old = linfo->disallow_lpage;
        linfo->disallow_lpage += count;
-       WARN_ON_ONCE(linfo->disallow_lpage < 0);
+       WARN_ON_ONCE((old ^ linfo->disallow_lpage) & KVM_LPAGE_MIXED_FLAG);
    }
 }
```
* `hugepage_test_mixed()` 测试是否因 *有私有页面* 导致该页无法在该级上成为巨页
```cpp
#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
static bool hugepage_test_mixed(struct kvm_memory_slot *slot, gfn_t gfn,
                int level)
{
    return lpage_info_slot(gfn, slot, level)->disallow_lpage & KVM_LPAGE_MIXED_FLAG;
}
```
* 清除和设置该页在 `lpage_info` 所在级别的 `struct kvm_lpage_info.disallow_lpage` 域的混合私有页面标志
```cpp
static void hugepage_clear_mixed(struct kvm_memory_slot *slot, gfn_t gfn,
                 int level)
{
    lpage_info_slot(gfn, slot, level)->disallow_lpage &= ~KVM_LPAGE_MIXED_FLAG;
}

static void hugepage_set_mixed(struct kvm_memory_slot *slot, gfn_t gfn,
                   int level)
{
    lpage_info_slot(gfn, slot, level)->disallow_lpage |= KVM_LPAGE_MIXED_FLAG;
}
```
* `hugepage_has_attrs()` 判断给定的 `gfn` 在指定的 `level` 上是否有属性 `attrs`
  * 当级别是 2M 大页时，无法优化，在 `kvm->mem_attr_array` xarray 中逐个对比，比较低效
  * 当级别高于 2M 大页时，优先借助 `lpage_info[level-1]` 中的信息进行判断
    * 如果下一级是私有和共享混合而无法成为大页，那么再判断属性也没意义
    * 如果下一级不是混合页，而是完整的大页，则比较是否具备指定的属性
```cpp
static bool hugepage_has_attrs(struct kvm *kvm, struct kvm_memory_slot *slot,
                   gfn_t gfn, int level, unsigned long attrs)
{   //确定起始和结束 GFN
    const unsigned long start = gfn;
    const unsigned long end = start + KVM_PAGES_PER_HPAGE(level);

    if (level == PG_LEVEL_2M)
        return kvm_range_has_memory_attributes(kvm, start, end, attrs);

    for (gfn = start; gfn < end; gfn += KVM_PAGES_PER_HPAGE(level - 1)) {
        if (hugepage_test_mixed(slot, gfn, level - 1) ||
            attrs != kvm_get_memory_attributes(kvm, gfn))
            return false;
    }
    return true;
}
```
* 回忆一下在之前的 patch 构造的以下调用路径可以知道 `kvm_arch_post_set_memory_attributes()` 的使用场景
```cpp
kvm_vm_ioctl()
   case KVM_SET_MEMORY_ATTRIBUTES:
   -> kvm_vm_ioctl_set_mem_attributes()
      -> kvm_vm_set_mem_attributes()
            struct kvm_mmu_notifier_range post_set_range = {
               ...
               .handler = kvm_arch_post_set_memory_attributes,
               ...
            }
         -> kvm_handle_gfn_range(kvm, &post_set_range)
            -> range->handler(kvm, &gfn_range)
            => kvm_arch_post_set_memory_attributes()
```
* 计算哪些范围可以使用大页映射，即使 slot 无法映射内存为 PRIVATE。KVM 无法在具有 PRIVATE GFN 的一个范围上创建共享大页，相反，将范围转换为共享现在可能允许大页。
```cpp
bool kvm_arch_post_set_memory_attributes(struct kvm *kvm,
                     struct kvm_gfn_range *range)
{
    unsigned long attrs = range->arg.attributes;
    struct kvm_memory_slot *slot = range->slot;
    int level;
    //断言锁的持有情况
    lockdep_assert_held_write(&kvm->mmu_lock);
    lockdep_assert_held(&kvm->slots_lock);
    //因为当前只支持 PRIVATE 一个内存属性，因此当 VM 不支持私有内存时什么都不做
    /*
     * Calculate which ranges can be mapped with hugepages even if the slot
     * can't map memory PRIVATE.  KVM mustn't create a SHARED hugepage over
     * a range that has PRIVATE GFNs, and conversely converting a range to
     * SHARED may now allow hugepages.
     */
    if (WARN_ON_ONCE(!kvm_arch_has_private_mem(kvm)))
        return false;
    //这里的顺序很重要：上层消耗下层扫描的结果。
    /*
     * The sequence matters here: upper levels consume the result of lower
     * level's scanning.
     */
    for (level = PG_LEVEL_2M; level <= KVM_MAX_HUGEPAGE_LEVEL; level++) {
        gfn_t nr_pages = KVM_PAGES_PER_HPAGE(level);
        gfn_t gfn = gfn_round_for_level(range->start, level);
        //如果头部的页跨越了范围，则对其进行处理。包括 range->start 对齐到边界但按大页来算却超过 range->end 的情况
        /* Process the head page if it straddles the range. */
        if (gfn != range->start || gfn + nr_pages > range->end) {
            //如果对齐的 gfn 没有被 memslot 覆盖，则跳过混合跟踪，无论内存属性如何，由于地址未对齐，KVM 都无法使用大页。
            /*
             * Skip mixed tracking if the aligned gfn isn't covered
             * by the memslot, KVM can't use a hugepage due to the
             * misaligned address regardless of memory attributes.
             */
            if (gfn >= slot->base_gfn) { //range->start 不一定是 slot->base_gfn，但也在 slot 范围内，
                if (hugepage_has_attrs(kvm, slot, gfn, level, attrs)) //因此也要检查是否要设置属性
                    hugepage_clear_mixed(slot, gfn, level);
                else
                    hugepage_set_mixed(slot, gfn, level);
            }
            gfn += nr_pages;
        }
        //该范围完全覆盖的页面保证仅具有刚刚设置的属性。
        /*
         * Pages entirely covered by the range are guaranteed to have
         * only the attributes which were just set.
         */
        for ( ; gfn + nr_pages <= range->end; gfn += nr_pages)
            hugepage_clear_mixed(slot, gfn, level);
        //如果最后一个尾页跨越 range->end 并且包含在 memslot 中，则处理它。与头页一样，如果 slot 大小未对齐，KVM 无法创建大页。
        /*
         * Process the last tail page if it straddles the range and is
         * contained by the memslot.  Like the head page, KVM can't
         * create a hugepage if the slot size is misaligned.
         */
        if (gfn < range->end &&
            (gfn + nr_pages) <= (slot->base_gfn + slot->npages)) {
            if (hugepage_has_attrs(kvm, slot, gfn, level, attrs))
                hugepage_clear_mixed(slot, gfn, level);
            else
                hugepage_set_mixed(slot, gfn, level);
        }
    }
    return false; //返回 false 仅仅是为了 kvm_handle_gfn_range() 不用刷 TLB
}
```
* `kvm_mmu_init_memslot_memory_attributes()` 初始化 memslot 的内存属性，确切地说是 memslot 的 `lpage_info[]` 的 `disallow_lpage` 的混合属性
* 不要费心跟踪由于不可能对齐而不能成为巨页的混合属性，也就是说，仅处理 memslot 完全包含的页面。
  * 比如说，如果 `slot->base_gfn` 是 2M 对齐的，当处理 1G level 的时候，`slot->base_gfn` 向下对齐到 1G 会导致 `start < slot->base_gfn`，这个时候前面这些未对齐到 1G 的 2M 大页就不需要管了，因为他们无法成为 1G 巨页的一部分
* 与设置属性不同，每个潜在的巨页都需要手动检查，因为属性可能已经混合。
```cpp
void kvm_mmu_init_memslot_memory_attributes(struct kvm *kvm,
                        struct kvm_memory_slot *slot)
{
    int level;
    //因为当前只支持 PRIVATE 一个内存属性，因此当 VM 不支持私有内存时什么都不做
    if (!kvm_arch_has_private_mem(kvm))
        return;

    for (level = PG_LEVEL_2M; level <= KVM_MAX_HUGEPAGE_LEVEL; level++) {
        /*
         * Don't bother tracking mixed attributes for pages that can't
         * be huge due to alignment, i.e. process only pages that are
         * entirely contained by the memslot.
         */
        gfn_t end = gfn_round_for_level(slot->base_gfn + slot->npages, level);
        gfn_t start = gfn_round_for_level(slot->base_gfn, level);
        gfn_t nr_pages = KVM_PAGES_PER_HPAGE(level);
        gfn_t gfn;
        //不可能成为这个级别的巨页无需处理，详见代码解析
        if (start < slot->base_gfn)
            start += nr_pages;
        //处理下面的子页面有私有页面导致我这一级无法成为巨页的情况
        /*
         * Unlike setting attributes, every potential hugepage needs to
         * be manually checked as the attributes may already be mixed.
         */
        for (gfn = start; gfn < end; gfn += nr_pages) {
            unsigned long attrs = kvm_get_memory_attributes(kvm, gfn);

            if (hugepage_has_attrs(kvm, slot, gfn, level, attrs))
                hugepage_clear_mixed(slot, gfn, level);
            else
                hugepage_set_mixed(slot, gfn, level);
        }
    }
}
```
* 在 KVM 分配 memslot 元数据的时候初始化 memslot 的内存属性，`lpage_info[]` 就属于 memslot 的元数据
```diff
diff --git a/arch/x86/kvm/x86.c b/arch/x86/kvm/x86.c
index f661acb01c58..e1aad0c81f6f 100644
--- a/arch/x86/kvm/x86.c
+++ b/arch/x86/kvm/x86.c
@@ -12728,6 +12728,10 @@ static int kvm_alloc_memslot_metadata(struct kvm *kvm,
        }
    }

+#ifdef CONFIG_KVM_GENERIC_MEMORY_ATTRIBUTES
+   kvm_mmu_init_memslot_memory_attributes(kvm, slot);
+#endif
+
    if (kvm_page_track_create_memslot(kvm, slot, npages))
        goto out_free;
```

## [PATCH 18/34] KVM: x86/mmu: Handle page fault for private memory

* 添加对解决虚拟机的 guest 私有内存缺页的支持，以区分“shared”和“private”内存。
  * 对于此类 VM，`KVM_MEM_GUEST_MEMFD` memslots 可以包括 fd-based 的私有内存和 hva-based 的共享内存，
  * 并且 KVM 需要映射到“正确”的变体，例如，KVM 需要根据 gfn 的 `KVM_MEMORY_ATTRIBUTE_PRIVATE` 标志的当前状态，适当地映射 gfn 为 shared/private。
* 对于 AMD 的 SEV-SNP 和 Intel 的 TDX，guest 通过 guest 页表中的一个位有效地请求共享与私有，即 guest 想要的可能与当前内存属性冲突。
  * 为了支持此类“隐式”转换请求，用 `KVM_EXIT_MEMORY_FAULT` 退出到用户，以将请求转发到用户空间。
  * 添加一个新的 memory faults 标志 `KVM_MEMORY_EXIT_FLAG_PRIVATE`，以传达 guest 是否想要将内存映射为共享内存还是私有内存。
* 与 `KVM_MEMORY_ATTRIBUTE_PRIVATE` 类似，使用 bit `3` 来标记私有内存，以便当用户空间需要此类信息时，KVM 可以使用 bit `0-2` 来捕获 `RWX` 行为，
  * 例如，`KVM_EXIT_MEMORY_FAULT` 的一个可能用户是在处理 guest 缺页 VM-Exit 时因缺少映射而退出。在这种情况下，用户空间将需要知道 `RWX` 信息，以便正确/精确地解决故障。
* **注意**，私有内存 *必须* 由 guest_memfd 作为后备存储，例如，共享映射始终来自 host 用户空间页表，而私有映射始终来自 guest_memfd 实例。
---
* `KVM_MEMORY_EXIT_FLAG_PRIVATE` 设置后，表示私有内存访问时发生 memory fault。清除时，表示共享访问发生 memory fault。
```diff
diff --git a/Documentation/virt/kvm/api.rst b/Documentation/virt/kvm/api.rst
index 1e61faf02b2a..726c87c35d57 100644
--- a/Documentation/virt/kvm/api.rst
+++ b/Documentation/virt/kvm/api.rst
@@ -6952,6 +6952,7 @@ spec refer, https://github.com/riscv/riscv-sbi-doc.

        /* KVM_EXIT_MEMORY_FAULT */
        struct {
+  #define KVM_MEMORY_EXIT_FLAG_PRIVATE (1ULL << 3)
            __u64 flags;
            __u64 gpa;
            __u64 size;
@@ -6960,8 +6961,11 @@ spec refer, https://github.com/riscv/riscv-sbi-doc.
 KVM_EXIT_MEMORY_FAULT indicates the vCPU has encountered a memory fault that
 could not be resolved by KVM.  The 'gpa' and 'size' (in bytes) describe the
 guest physical address range [gpa, gpa + size) of the fault.  The 'flags' field
-describes properties of the faulting access that are likely pertinent.
-Currently, no flags are defined.
+describes properties of the faulting access that are likely pertinent:
+
+ - KVM_MEMORY_EXIT_FLAG_PRIVATE - When set, indicates the memory fault occurred
+   on a private memory access.  When clear, indicates the fault occurred on a
+   shared access.

 Note!  KVM_EXIT_MEMORY_FAULT is unique among all KVM exit reasons in that it
 accompanies a return code of '-1', not '0'!  errno will always be set to EFAULT
```
* 原 `kvm_mmu_max_mapping_level()` 改名为将要被封装的 `__kvm_mmu_max_mapping_level()`
  * 增加 `bool is_private` 入参
```diff
diff --git a/arch/x86/kvm/mmu/mmu.c b/arch/x86/kvm/mmu/mmu.c
index f5c6b0643645..754a5aaebee5 100644
--- a/arch/x86/kvm/mmu/mmu.c
+++ b/arch/x86/kvm/mmu/mmu.c
@@ -3147,9 +3147,9 @@ static int host_pfn_mapping_level(struct kvm *kvm, gfn_t gfn,
    return level;
 }

-int kvm_mmu_max_mapping_level(struct kvm *kvm,
-                 const struct kvm_memory_slot *slot, gfn_t gfn,
-                 int max_level)
+static int __kvm_mmu_max_mapping_level(struct kvm *kvm,
+                      const struct kvm_memory_slot *slot,
+                      gfn_t gfn, int max_level, bool is_private)
 {
    struct kvm_lpage_info *linfo;
    int host_level;
```
* 新 `kvm_mmu_max_mapping_level()` 作为外层函数，保持参数不变
  * 对于 slot 可以是私有的且 `gfn` 对应的 `kvm->mem_attr_array` xarray 带有私有属性 `KVM_MEMORY_ATTRIBUTE_PRIVATE`，新入参 `is_private = true`
  * 新加 `if (is_private)` 的判断：如果缺页发生在私有地址，`lpage_info[]` 找到的 `gfn` 的页面级别就是它可以被映射的最大级别
```cpp
static int __kvm_mmu_max_mapping_level(struct kvm *kvm,
                       const struct kvm_memory_slot *slot,
                       gfn_t gfn, int max_level, bool is_private)
{
    struct kvm_lpage_info *linfo;
    int host_level;
    //从当前传入的最大页面级别和 EPT 所能支持的最大巨页级别中选出最小值
    max_level = min(max_level, max_huge_page_level);
    for ( ; max_level > PG_LEVEL_4K; max_level--) { //比如说级别从 512G -> 1G -> 2M 级别依次遍历
        linfo = lpage_info_slot(gfn, slot, max_level); //根据 GFN 找到在 slot 上某个大页级别上对应的是否在该级别上允许大页的信息
        if (!linfo->disallow_lpage) //比如在 512G 级别上该 GFN 无法支持大页，则尝试 1G 大页，如果支持 1G 则结束遍历
            break;
    }
    //如果缺页发生在私有地址，lpage_info[] 找到的 gfn 的页面级别就是它可以被映射的最大级别
    if (is_private)
        return max_level;
    //此时的大页级别是在 slot 上能支持的最大级别，比如能支持 1G 自然也就能支持 2M。但 4K 就没必要再往下调查了
    if (max_level == PG_LEVEL_4K)
        return PG_LEVEL_4K;
    //如果缺页不是发生在私有地址，根据 GFN 和 slot 找到 HVA，然后返回该 HVA 在 host 侧映射的级别
    host_level = host_pfn_mapping_level(kvm, gfn, slot);
    return min(host_level, max_level); //由此可见，此处 guest 所能支持的页面级别同时受限于其 host 侧的映射的页面级别
}

int kvm_mmu_max_mapping_level(struct kvm *kvm,
                  const struct kvm_memory_slot *slot, gfn_t gfn,
                  int max_level)
{
    bool is_private = kvm_slot_can_be_private(slot) &&
              kvm_mem_is_private(kvm, gfn);

    return __kvm_mmu_max_mapping_level(kvm, slot, gfn, max_level, is_private);
}
```
* 原来调外层函数的 `kvm_mmu_hugepage_adjust()` 改调内层函数
```diff
@@ -3188,8 +3201,9 @@ void kvm_mmu_hugepage_adjust(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault
     * Enforce the iTLB multihit workaround after capturing the requested
     * level, which will be used to do precise, accurate accounting.
     */
-   fault->req_level = kvm_mmu_max_mapping_level(vcpu->kvm, slot,
-                            fault->gfn, fault->max_level);
+   fault->req_level = __kvm_mmu_max_mapping_level(vcpu->kvm, slot,
+                              fault->gfn, fault->max_level,
+                              fault->is_private);
    if (fault->req_level == PG_LEVEL_4K || fault->huge_page_disallowed)
        return;
```
* 新增 `kvm_max_level_for_order()` 函数根据传入的阶 `order` 返回所能支持的最大页面级别（page level）
```cpp
static inline u8 kvm_max_level_for_order(int order)
{   //当前 KVM 最大的巨页级别如果大于 1G 编译报错
    BUILD_BUG_ON(KVM_MAX_HUGEPAGE_LEVEL > PG_LEVEL_1G);
    //传入的阶与 1G、2M、4K 的阶都不匹配，警告
    KVM_MMU_WARN_ON(order != KVM_HPAGE_GFN_SHIFT(PG_LEVEL_1G) &&
            order != KVM_HPAGE_GFN_SHIFT(PG_LEVEL_2M) &&
            order != KVM_HPAGE_GFN_SHIFT(PG_LEVEL_4K));

    if (order >= KVM_HPAGE_GFN_SHIFT(PG_LEVEL_1G))
        return PG_LEVEL_1G;

    if (order >= KVM_HPAGE_GFN_SHIFT(PG_LEVEL_2M))
        return PG_LEVEL_2M;

    return PG_LEVEL_4K;
}
```
* `kvm_mmu_prepare_memory_fault_exit()` 是对 `kvm_prepare_memory_fault_exit()` 的简单封装，将 `struct kvm_page_fault *fault` 展开了
```cpp
static void kvm_mmu_prepare_memory_fault_exit(struct kvm_vcpu *vcpu,
                          struct kvm_page_fault *fault)
{
    kvm_prepare_memory_fault_exit(vcpu, fault->gfn << PAGE_SHIFT,
                      PAGE_SIZE, fault->write, fault->exec,
                      fault->is_private);
}
```
* 新增 `kvm_faultin_pfn_private()` 是处理 KVM 私有页面缺页的入口函数
  * 调用它的 `__kvm_faultin_pfn()` 的 `if (fault->is_private)` 来自 `gfn` 在 `kvm->mem_attr_array` xarray 条目是否有 `KVM_MEMORY_ATTRIBUTE_PRIVATE` 内存属性
  * `kvm_slot_can_be_private()` 来自 memslot 是否有 `KVM_MEM_GUEST_MEMFD` 的 flag
  * 也就是说，光通过 `ioctl(KVM_SET_MEMORY_ATTRIBUTES)` 设置一段内存为私有是不够的，memslot 必须带有 `KVM_MEM_GUEST_MEMFD` flag，否则返回 `-EFAULT`
```cpp
static int kvm_faultin_pfn_private(struct kvm_vcpu *vcpu,
                   struct kvm_page_fault *fault)
{
    int max_order, r;

    if (!kvm_slot_can_be_private(fault->slot)) {
        kvm_mmu_prepare_memory_fault_exit(vcpu, fault);
        return -EFAULT;
    }
    //gmem 的 faultin 函数，确定 GFN 映射到的 PFN，目前将 max_order = 0
    r = kvm_gmem_get_pfn(vcpu->kvm, fault->slot, fault->gfn, &fault->pfn,
                 &max_order);
    if (r) { //如果 faultin 出错，准备给用户的出错信息
        kvm_mmu_prepare_memory_fault_exit(vcpu, fault);
        return r;
    }
    //因为目前 max_order = 0，kvm_max_level_for_order() 返回 4K
    fault->max_level = min(kvm_max_level_for_order(max_order),
                   fault->max_level); //在 kvm_mmu_do_page_fault() 初始化为 KVM_MAX_HUGEPAGE_LEVEL
    fault->map_writable = !(fault->slot->flags & KVM_MEM_READONLY); //如果不是只读 slot

    return RET_PF_CONTINUE;
}
```
* 第一个条件可能是针对
>  * 对于 AMD 的 SEV-SNP 和 Intel 的 TDX，guest 通过 guest 页表中的一个位有效地请求共享与私有，即 guest 想要的可能与当前内存属性冲突。
>    * 为了支持此类“隐式”转换请求，用 `KVM_EXIT_MEMORY_FAULT` 退出到用户，以将请求转发到用户空间。
>    * 添加一个新的 memory faults 标志 `KVM_MEMORY_EXIT_FLAG_PRIVATE`，以传达 guest 是否想要将内存映射为共享内存还是私有内存。
* 译注：在 `kvm_mmu_do_page_fault()` 里 `fault.is_private = kvm_mem_is_private(vcpu->kvm, cr2_or_gpa >> PAGE_SHIFT)`，与 `__kvm_faultin_pfn()` 用的是同一个函数和同一个 `gfn` 啊？！
  * `gfn` 来自 `kvm_mmu_do_page_fault()` 中的 `fault.addr = cr2_or_gpa; fault.gfn = fault.addr >> PAGE_SHIFT;`
  * 所以以下修改的第一个条件怎么会进得去呢？是为将来做准备吧？TDX 是 `fault.is_private = kvm_is_private_gpa(vcpu->kvm, cr2_or_gpa)`，`gfn` 与内存属性是有可能不一致的
```diff
@@ -4301,6 +4364,14 @@ static int __kvm_faultin_pfn(struct kvm_vcpu *vcpu, struct kvm_page_fault *fault
            return RET_PF_EMULATE;
    }

+   if (fault->is_private != kvm_mem_is_private(vcpu->kvm, fault->gfn)) {
+       kvm_mmu_prepare_memory_fault_exit(vcpu, fault);
+       return -EFAULT;
+   }
+
+   if (fault->is_private)
+       return kvm_faultin_pfn_private(vcpu, fault);
+
    async = false;
    fault->pfn = __gfn_to_pfn_memslot(slot, fault->gfn, false, false, &async,
                      fault->write, &fault->map_writable,
```
* `kvm_arch_pre_set_memory_attributes()` 和 `kvm_arch_post_set_memory_attributes()` 调用的路径类似，但它被 `kvm_pre_set_memory_attributes()` 调用
* 即使 slot 无法映射为 PRIVATE，也可 zap SPTE。
* KVM x86 仅支持 `KVM_MEMORY_ATTRIBUTE_PRIVATE`，因此 *看起来* KVM 可以简单地忽略此类 slots。（第一个条件）
* 但如果用户空间将内存设为私有，那么 KVM 必须阻止 guest **以共享的方式** 访问内存。
* 如果用户空间正在使内存 *共享* 并且达到这里，则该范围内的至少一个页面以前是 *私有的*，例如，slot 的可能的大页范围正在改变。
  * 在这种情况下 zapping SPTE 可确保 KVM 重新评估是否可以将大页用于受影响的范围。
```cpp
bool kvm_arch_pre_set_memory_attributes(struct kvm *kvm,
                    struct kvm_gfn_range *range)
{
    /*
     * Zap SPTEs even if the slot can't be mapped PRIVATE.  KVM x86 only
     * supports KVM_MEMORY_ATTRIBUTE_PRIVATE, and so it *seems* like KVM
     * can simply ignore such slots.  But if userspace is making memory
     * PRIVATE, then KVM must prevent the guest from accessing the memory
     * as shared.  And if userspace is making memory SHARED and this point
     * is reached, then at least one page within the range was previously
     * PRIVATE, i.e. the slot's possible hugepage ranges are changing.
     * Zapping SPTEs in this case ensures KVM will reassess whether or not
     * a hugepage can be used for affected ranges.
     */
    if (WARN_ON_ONCE(!kvm_arch_has_private_mem(kvm)))
        return false;

    return kvm_unmap_gfn_range(kvm, range);
}
```
* `struct kvm_page_fault` 增加 `is_private` 域
```diff
diff --git a/arch/x86/kvm/mmu/mmu_internal.h b/arch/x86/kvm/mmu/mmu_internal.h
index decc1f153669..86c7cb692786 100644
--- a/arch/x86/kvm/mmu/mmu_internal.h
+++ b/arch/x86/kvm/mmu/mmu_internal.h
@@ -201,6 +201,7 @@ struct kvm_page_fault {

    /* Derived from mmu and global state.  */
    const bool is_tdp;
+   const bool is_private;
    const bool nx_huge_page_workaround_enabled;

    /*
```
* 添加一个新的 memory faults 标志 `KVM_MEMORY_EXIT_FLAG_PRIVATE`，以传达 guest 是否想要将内存映射为共享内存还是私有内存。
```diff
diff --git a/include/uapi/linux/kvm.h b/include/uapi/linux/kvm.h
index 2802d10aa88c..8eb10f560c69 100644
--- a/include/uapi/linux/kvm.h
+++ b/include/uapi/linux/kvm.h
@@ -535,6 +535,7 @@ struct kvm_run {
        } notify;
        /* KVM_EXIT_MEMORY_FAULT */
        struct {
+#define KVM_MEMORY_EXIT_FLAG_PRIVATE   (1ULL << 3)
            __u64 flags;
            __u64 gpa;
            __u64 size;
```
* `kvm_prepare_memory_fault_exit()` 之前已经给出过完整版了，其实有一部分修改是在本 patch 引入的
```diff
diff --git a/include/linux/kvm_host.h b/include/linux/kvm_host.h
index a6de526c0426..67dfd4d79529 100644
--- a/include/linux/kvm_host.h
+++ b/include/linux/kvm_host.h
@@ -2357,14 +2357,18 @@ static inline void kvm_account_pgtable_pages(void *virt, int nr)
 #define  KVM_DIRTY_RING_MAX_ENTRIES  65536

 static inline void kvm_prepare_memory_fault_exit(struct kvm_vcpu *vcpu,
-                        gpa_t gpa, gpa_t size)
+                        gpa_t gpa, gpa_t size,
+                        bool is_write, bool is_exec,
+                        bool is_private)
 {
    vcpu->run->exit_reason = KVM_EXIT_MEMORY_FAULT;
    vcpu->run->memory_fault.gpa = gpa;
    vcpu->run->memory_fault.size = size;

-   /* Flags are not (yet) defined or communicated to userspace. */
+   /* RWX flags are not (yet) defined or communicated to userspace. */
    vcpu->run->memory_fault.flags = 0;
+   if (is_private)
+       vcpu->run->memory_fault.flags |= KVM_MEMORY_EXIT_FLAG_PRIVATE;
 }
```

## [PATCH 19/34] KVM: Drop superfluous __KVM_VCPU_MULTIPLE_ADDRESS_SPACE macro
* 删除 `__KVM_VCPU_MULTIPLE_ADDRESS_SPACE` 并改为检查 `KVM_ADDRESS_SPACE_NUM` 的值。
* 无意进行功能改变。
---
```diff
diff --git a/arch/x86/include/asm/kvm_host.h b/arch/x86/include/asm/kvm_host.h
index fa0d42202405..061eec231299 100644
--- a/arch/x86/include/asm/kvm_host.h
+++ b/arch/x86/include/asm/kvm_host.h
@@ -2136,7 +2136,6 @@ enum {
 #define HF_SMM_MASK        (1 << 1)
 #define HF_SMM_INSIDE_NMI_MASK (1 << 2)

-# define __KVM_VCPU_MULTIPLE_ADDRESS_SPACE
 # define KVM_ADDRESS_SPACE_NUM 2
 # define kvm_arch_vcpu_memslots_id(vcpu) ((vcpu)->arch.hflags & HF_SMM_MASK ? 1 : 0)
 # define kvm_memslots_for_spte_role(kvm, role) __kvm_memslots(kvm, (role).smm)
diff --git a/include/linux/kvm_host.h b/include/linux/kvm_host.h
index 67dfd4d79529..db423ea9e3a4 100644
--- a/include/linux/kvm_host.h
+++ b/include/linux/kvm_host.h
@@ -690,7 +690,7 @@ bool kvm_arch_irqchip_in_kernel(struct kvm *kvm);
 #define KVM_MEM_SLOTS_NUM SHRT_MAX
 #define KVM_USER_MEM_SLOTS (KVM_MEM_SLOTS_NUM - KVM_INTERNAL_MEM_SLOTS)

-#ifndef __KVM_VCPU_MULTIPLE_ADDRESS_SPACE
+#if KVM_ADDRESS_SPACE_NUM == 1
 static inline int kvm_arch_vcpu_memslots_id(struct kvm_vcpu *vcpu)
 {
    return 0;
```

## [PATCH 20/34] KVM: Allow arch code to track number of memslot address spaces per VM
* 让 x86 跟踪每个 VM 的地址空间数量，以便 KVM 可以禁止机密 VM 使用 SMM memslot。
  * 机密虚拟机从根本上与模拟 SMM 不兼容，顾名思义，模拟 SMM 需要能够读写 guest 内存和寄存器状态。
* 禁止 SMM 将简化对 guest 私有内存的支持，因为 KVM 无需担心跟踪多个地址空间的内存属性（SMM 是所有架构中唯一的“非默认”地址空间）。
---
### x86 部分
* `KVM_ADDRESS_SPACE_NUM` 宏改名为 `KVM_MAX_NR_ADDRESS_SPACES`
* 新增内联函数 `kvm_arch_nr_memslot_as_ids()` 返回 `KVM_MAX_NR_ADDRESS_SPACES`
```diff
diff --git a/arch/x86/include/asm/kvm_host.h b/arch/x86/include/asm/kvm_host.h
index 061eec231299..75ab0da06e64 100644
--- a/arch/x86/include/asm/kvm_host.h
+++ b/arch/x86/include/asm/kvm_host.h
@@ -2136,9 +2136,15 @@ enum {
 #define HF_SMM_MASK        (1 << 1)
 #define HF_SMM_INSIDE_NMI_MASK (1 << 2)

-# define KVM_ADDRESS_SPACE_NUM 2
+# define KVM_MAX_NR_ADDRESS_SPACES 2
 # define kvm_arch_vcpu_memslots_id(vcpu) ((vcpu)->arch.hflags & HF_SMM_MASK ? 1 : 0)
 # define kvm_memslots_for_spte_role(kvm, role) __kvm_memslots(kvm, (role).smm)
+
+static inline int kvm_arch_nr_memslot_as_ids(struct kvm *kvm)
+{
+   return KVM_MAX_NR_ADDRESS_SPACES;
+}
+
 #else
 # define kvm_memslots_for_spte_role(kvm, role) __kvm_memslots(kvm, 0)
 #endif
```
* 引用之处需要替换
```diff
diff --git a/arch/x86/kvm/debugfs.c b/arch/x86/kvm/debugfs.c
index ee8c4c3496ed..42026b3f3ff3 100644
--- a/arch/x86/kvm/debugfs.c
+++ b/arch/x86/kvm/debugfs.c
@@ -111,7 +111,7 @@ static int kvm_mmu_rmaps_stat_show(struct seq_file *m, void *v)
 	mutex_lock(&kvm->slots_lock);
 	write_lock(&kvm->mmu_lock);
 
-	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
+	for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
 		int bkt;
 
 		slots = __kvm_memslots(kvm, i);
diff --git a/arch/x86/kvm/mmu/mmu.c b/arch/x86/kvm/mmu/mmu.c
index 754a5aaebee5..4de7670d5976 100644
--- a/arch/x86/kvm/mmu/mmu.c
+++ b/arch/x86/kvm/mmu/mmu.c
@@ -3763,7 +3763,7 @@ static int mmu_first_shadow_root_alloc(struct kvm *kvm)
 	    kvm_page_track_write_tracking_enabled(kvm))
 		goto out_success;
 
-	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
+	for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
 		slots = __kvm_memslots(kvm, i);
 		kvm_for_each_memslot(slot, bkt, slots) {
 			/*
@@ -6309,7 +6309,7 @@ static bool kvm_rmap_zap_gfn_range(struct kvm *kvm, gfn_t gfn_start, gfn_t gfn_e
 	if (!kvm_memslots_have_rmaps(kvm))
 		return flush;
 
-	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
+	for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
 		slots = __kvm_memslots(kvm, i);
 
 		kvm_for_each_memslot_in_gfn_range(&iter, slots, gfn_start, gfn_end) {
@@ -6806,7 +6806,7 @@ void kvm_mmu_invalidate_mmio_sptes(struct kvm *kvm, u64 gen)
 	 * modifier prior to checking for a wrap of the MMIO generation so
 	 * that a wrap in any address space is detected.
 	 */
-	gen &= ~((u64)KVM_ADDRESS_SPACE_NUM - 1);
+	gen &= ~((u64)kvm_arch_nr_memslot_as_ids(kvm) - 1);
 
 	/*
 	 * The very rare case: if the MMIO generation number has wrapped,
diff --git a/arch/x86/kvm/x86.c b/arch/x86/kvm/x86.c
index e1aad0c81f6f..f521c97f5c64 100644
--- a/arch/x86/kvm/x86.c
+++ b/arch/x86/kvm/x86.c
@@ -12577,7 +12577,7 @@ void __user * __x86_set_memory_region(struct kvm *kvm, int id, gpa_t gpa,
 		hva = slot->userspace_addr;
 	}
 
-	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
+	for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
 		struct kvm_userspace_memory_region2 m;
 
 		m.slot = id | (i << 16);
```
### Common 部分
* `KVM_ADDRESS_SPACE_NUM` 宏改名为 `KVM_MAX_NR_ADDRESS_SPACES`
* 新增内联函数 `kvm_arch_nr_memslot_as_ids()` 返回 `KVM_MAX_NR_ADDRESS_SPACES`
```diff
diff --git a/include/linux/kvm_host.h b/include/linux/kvm_host.h
index db423ea9e3a4..3ebc6912c54a 100644
--- a/include/linux/kvm_host.h
+++ b/include/linux/kvm_host.h
@@ -80,8 +80,8 @@
 /* Two fragments for cross MMIO pages. */
 #define KVM_MAX_MMIO_FRAGMENTS	2
 
-#ifndef KVM_ADDRESS_SPACE_NUM
-#define KVM_ADDRESS_SPACE_NUM	1
+#ifndef KVM_MAX_NR_ADDRESS_SPACES
+#define KVM_MAX_NR_ADDRESS_SPACES	1
 #endif
 
 /*
@@ -690,7 +690,12 @@ bool kvm_arch_irqchip_in_kernel(struct kvm *kvm);
 #define KVM_MEM_SLOTS_NUM SHRT_MAX
 #define KVM_USER_MEM_SLOTS (KVM_MEM_SLOTS_NUM - KVM_INTERNAL_MEM_SLOTS)
 
-#if KVM_ADDRESS_SPACE_NUM == 1
+#if KVM_MAX_NR_ADDRESS_SPACES == 1
+static inline int kvm_arch_nr_memslot_as_ids(struct kvm *kvm)
+{
+	return KVM_MAX_NR_ADDRESS_SPACES;
+}
+
 static inline int kvm_arch_vcpu_memslots_id(struct kvm_vcpu *vcpu)
 {
 	return 0;
```

```diff
@@ -745,9 +750,9 @@ struct kvm {
 	struct mm_struct *mm; /* userspace tied to this vm */
 	unsigned long nr_memslot_pages;
 	/* The two memslot sets - active and inactive (per address space) */
-	struct kvm_memslots __memslots[KVM_ADDRESS_SPACE_NUM][2];
+	struct kvm_memslots __memslots[KVM_MAX_NR_ADDRESS_SPACES][2];
 	/* The current active memslot set for each address space */
-	struct kvm_memslots __rcu *memslots[KVM_ADDRESS_SPACE_NUM];
+	struct kvm_memslots __rcu *memslots[KVM_MAX_NR_ADDRESS_SPACES];
 	struct xarray vcpu_array;
 	/*
 	 * Protected by slots_lock, but can be read outside if an
@@ -1017,7 +1022,7 @@ void kvm_put_kvm_no_destroy(struct kvm *kvm);
 
 static inline struct kvm_memslots *__kvm_memslots(struct kvm *kvm, int as_id)
 {
-	as_id = array_index_nospec(as_id, KVM_ADDRESS_SPACE_NUM);
+	as_id = array_index_nospec(as_id, KVM_MAX_NR_ADDRESS_SPACES);
 	return srcu_dereference_check(kvm->memslots[as_id], &kvm->srcu,
 			lockdep_is_held(&kvm->slots_lock) ||
 			!refcount_read(&kvm->users_count));
diff --git a/virt/kvm/dirty_ring.c b/virt/kvm/dirty_ring.c
index c1cd7dfe4a90..86d267db87bb 100644
--- a/virt/kvm/dirty_ring.c
+++ b/virt/kvm/dirty_ring.c
@@ -58,7 +58,7 @@ static void kvm_reset_dirty_gfn(struct kvm *kvm, u32 slot, u64 offset, u64 mask)
 	as_id = slot >> 16;
 	id = (u16)slot;
 
-	if (as_id >= KVM_ADDRESS_SPACE_NUM || id >= KVM_USER_MEM_SLOTS)
+	if (as_id >= kvm_arch_nr_memslot_as_ids(kvm) || id >= KVM_USER_MEM_SLOTS)
 		return;
 
 	memslot = id_to_memslot(__kvm_memslots(kvm, as_id), id);
diff --git a/virt/kvm/kvm_main.c b/virt/kvm/kvm_main.c
index 8f46d757a2c5..8758cb799e18 100644
--- a/virt/kvm/kvm_main.c
+++ b/virt/kvm/kvm_main.c
@@ -615,7 +615,7 @@ static __always_inline kvm_mn_ret_t __kvm_handle_hva_range(struct kvm *kvm,
 
 	idx = srcu_read_lock(&kvm->srcu);
 
-	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
+	for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
 		struct interval_tree_node *node;
 
 		slots = __kvm_memslots(kvm, i);
@@ -1241,7 +1241,7 @@ static struct kvm *kvm_create_vm(unsigned long type, const char *fdname)
 		goto out_err_no_irq_srcu;
 
 	refcount_set(&kvm->users_count, 1);
-	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
+	for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
 		for (j = 0; j < 2; j++) {
 			slots = &kvm->__memslots[i][j];
 
@@ -1391,7 +1391,7 @@ static void kvm_destroy_vm(struct kvm *kvm)
 #endif
 	kvm_arch_destroy_vm(kvm);
 	kvm_destroy_devices(kvm);
-	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
+	for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
 		kvm_free_memslots(kvm, &kvm->__memslots[i][0]);
 		kvm_free_memslots(kvm, &kvm->__memslots[i][1]);
 	}
@@ -1682,7 +1682,7 @@ static void kvm_swap_active_memslots(struct kvm *kvm, int as_id)
 	 * space 0 will use generations 0, 2, 4, ... while address space 1 will
 	 * use generations 1, 3, 5, ...
 	 */
-	gen += KVM_ADDRESS_SPACE_NUM;
+	gen += kvm_arch_nr_memslot_as_ids(kvm);
 
 	kvm_arch_memslots_updated(kvm, gen);
 
@@ -2052,7 +2052,7 @@ int __kvm_set_memory_region(struct kvm *kvm,
 	    (mem->guest_memfd_offset & (PAGE_SIZE - 1) ||
 	     mem->guest_memfd_offset + mem->memory_size < mem->guest_memfd_offset))
 		return -EINVAL;
-	if (as_id >= KVM_ADDRESS_SPACE_NUM || id >= KVM_MEM_SLOTS_NUM)
+	if (as_id >= kvm_arch_nr_memslot_as_ids(kvm) || id >= KVM_MEM_SLOTS_NUM)
 		return -EINVAL;
 	if (mem->guest_phys_addr + mem->memory_size < mem->guest_phys_addr)
 		return -EINVAL;
@@ -2188,7 +2188,7 @@ int kvm_get_dirty_log(struct kvm *kvm, struct kvm_dirty_log *log,
 
 	as_id = log->slot >> 16;
 	id = (u16)log->slot;
-	if (as_id >= KVM_ADDRESS_SPACE_NUM || id >= KVM_USER_MEM_SLOTS)
+	if (as_id >= kvm_arch_nr_memslot_as_ids(kvm) || id >= KVM_USER_MEM_SLOTS)
 		return -EINVAL;
 
 	slots = __kvm_memslots(kvm, as_id);
@@ -2250,7 +2250,7 @@ static int kvm_get_dirty_log_protect(struct kvm *kvm, struct kvm_dirty_log *log)
 
 	as_id = log->slot >> 16;
 	id = (u16)log->slot;
-	if (as_id >= KVM_ADDRESS_SPACE_NUM || id >= KVM_USER_MEM_SLOTS)
+	if (as_id >= kvm_arch_nr_memslot_as_ids(kvm) || id >= KVM_USER_MEM_SLOTS)
 		return -EINVAL;
 
 	slots = __kvm_memslots(kvm, as_id);
@@ -2362,7 +2362,7 @@ static int kvm_clear_dirty_log_protect(struct kvm *kvm,
 
 	as_id = log->slot >> 16;
 	id = (u16)log->slot;
-	if (as_id >= KVM_ADDRESS_SPACE_NUM || id >= KVM_USER_MEM_SLOTS)
+	if (as_id >= kvm_arch_nr_memslot_as_ids(kvm) || id >= KVM_USER_MEM_SLOTS)
 		return -EINVAL;
 
 	if (log->first_page & 63)
@@ -2493,7 +2493,7 @@ static __always_inline void kvm_handle_gfn_range(struct kvm *kvm,
 	gfn_range.arg = range->arg;
 	gfn_range.may_block = range->may_block;
 
-	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
+	for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
 		slots = __kvm_memslots(kvm, i);
 
 		kvm_for_each_memslot_in_gfn_range(&iter, slots, range->start, range->end) {
@@ -4958,7 +4960,7 @@ bool kvm_are_all_memslots_empty(struct kvm *kvm)
 
 	lockdep_assert_held(&kvm->slots_lock);
 
-	for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
+	for (i = 0; i < kvm_arch_nr_memslot_as_ids(kvm); i++) {
 		if (!kvm_memslots_empty(__kvm_memslots(kvm, i)))
 			return false;
 	}
```
* 唯一不太一样的地方是这里，如果是检查 KVM 的地址空间数则返回宏 `KVM_MAX_NR_ADDRESS_SPACES`，如果是具体某个 VM 则是函数 `kvm_arch_nr_memslot_as_ids(kvm)` 的返回值
```diff
@@ -4848,9 +4848,11 @@ static int kvm_vm_ioctl_check_extension_generic(struct kvm *kvm, long arg)
 	case KVM_CAP_IRQ_ROUTING:
 		return KVM_MAX_IRQ_ROUTES;
 #endif
-#if KVM_ADDRESS_SPACE_NUM > 1
+#if KVM_MAX_NR_ADDRESS_SPACES > 1
 	case KVM_CAP_MULTI_ADDRESS_SPACE:
-		return KVM_ADDRESS_SPACE_NUM;
+		if (kvm)
+			return kvm_arch_nr_memslot_as_ids(kvm);
+		return KVM_MAX_NR_ADDRESS_SPACES;
 #endif
 	case KVM_CAP_NR_MEMSLOTS:
 		return KVM_USER_MEM_SLOTS;
```

## [PATCH 21/34] KVM: x86: Add support for "protected VMs" that can utilize private memory
* 添加新的 x86 VM 类型 `KVM_X86_SW_PROTECTED_VM`，作为机密（CoCo）VM 的开发和测试工具，甚至有可能在遥远的将来成为“真正的”产品，例如 一个 pKVM。
* KVM x86 中的私有内存支持针对 AMD 的 SEV-SNP 和 Intel 的 TDX，但这些技术极其复杂（轻描淡写）、难以调试、不支持作为嵌套 guests 运行，并且需要无法普遍访问的硬件 。
  * 例如，依靠 SEV-SNP 或 TDX 来维护 guest 私有内存并不是一个现实的选择。
* 至少，`KVM_X86_SW_PROTECTED_VM` 将为 guest_memfd 和私有内存支持启用各种自测试，而不需要独特的硬件。
---
* 引入 `KVM_CAP_VM_TYPES` VM 类型的能力
```rst
8.41 KVM_CAP_VM_TYPES
---------------------

:Capability: KVM_CAP_MEMORY_ATTRIBUTES
:Architectures: x86
:Type: system ioctl

This capability returns a bitmap of support VM types.  The 1-setting of bit @n
means the VM type with value @n is supported.  Possible values of @n are::

  #define KVM_X86_DEFAULT_VM    0
  #define KVM_X86_SW_PROTECTED_VM   1
```
* `struct kvm_arch` 引入 `vm_type` 域
```diff
diff --git a/arch/x86/include/asm/kvm_host.h b/arch/x86/include/asm/kvm_host.h
index 75ab0da06e64..a565a2e70f30 100644
--- a/arch/x86/include/asm/kvm_host.h
+++ b/arch/x86/include/asm/kvm_host.h
@@ -1255,6 +1255,7 @@ enum kvm_apicv_inhibit {
 };

 struct kvm_arch {
+   unsigned long vm_type;
    unsigned long n_used_mmu_pages;
    unsigned long n_requested_mmu_pages;
    unsigned long n_max_mmu_pages;
```
* 新增加宏 `kvm_arch_has_private_mem()` 根据 VM 的类型返回 VM 是否具有私有内存
```cpp
#ifdef CONFIG_KVM_PRIVATE_MEM
#define kvm_arch_has_private_mem(kvm) ((kvm)->arch.vm_type != KVM_X86_DEFAULT_VM)
#else
#define kvm_arch_has_private_mem(kvm) false
#endif
```
* 修改 x86 的 `kvm_arch_nr_memslot_as_ids()` 定义，VM 的类型决定返回支持的地址空间数目
  * TD VM 不支持 SMM，因此地址空间数是 `1`
```diff
diff --git a/arch/x86/include/asm/kvm_host.h b/arch/x86/include/asm/kvm_host.h
index 75ab0da06e64..a565a2e70f30 100644
--- a/arch/x86/include/asm/kvm_host.h
+++ b/arch/x86/include/asm/kvm_host.h
@@ -2137,14 +2144,10 @@ enum {
 #define HF_SMM_INSIDE_NMI_MASK (1 << 2)

 # define KVM_MAX_NR_ADDRESS_SPACES 2
+/* SMM is currently unsupported for guests with private memory. */
+# define kvm_arch_nr_memslot_as_ids(kvm) (kvm_arch_has_private_mem(kvm) ? 1 : 2)
 # define kvm_arch_vcpu_memslots_id(vcpu) ((vcpu)->arch.hflags & HF_SMM_MASK ? 1 : 0)
 # define kvm_memslots_for_spte_role(kvm, role) __kvm_memslots(kvm, (role).smm)
-
-static inline int kvm_arch_nr_memslot_as_ids(struct kvm *kvm)
-{
-   return KVM_MAX_NR_ADDRESS_SPACES;
-}
-
 #else
 # define kvm_memslots_for_spte_role(kvm, role) __kvm_memslots(kvm, 0)
 #endif
```
* 引入 VM 类型的两个宏
  * arch/x86/include/uapi/asm/kvm.h
```cpp
#define KVM_X86_DEFAULT_VM 0
#define KVM_X86_SW_PROTECTED_VM    1
```
* 引入 Kconfig `KVM_SW_PROTECTED_VM`
* arch/x86/kvm/Kconfig
```py
config KVM_SW_PROTECTED_VM
    bool "Enable support for KVM software-protected VMs"
    depends on EXPERT
    depends on KVM && X86_64
    select KVM_GENERIC_PRIVATE_MEM
    help
      Enable support for KVM software-protected VMs.  Currently "protected"
      means the VM can be backed with memory provided by
      KVM_CREATE_GUEST_MEMFD.

      If unsure, say "N".
```
* KVM 缺页的时候根据 guest 的 `CR2` 的 GPA `cr2_or_gpa` 确定是不是私有内存发生缺页
  * 目前用的 `kvm_mem_is_private()` 函数是查询 `kvm->mem_attr_array` xarray 中记录的内存属性来决定的
  * 在 TDX 的实现中是根据 `Shared` bit 是否置位来决定的
```diff
diff --git a/arch/x86/kvm/mmu/mmu_internal.h b/arch/x86/kvm/mmu/mmu_internal.h
index 86c7cb692786..b66a7d47e0e4 100644
--- a/arch/x86/kvm/mmu/mmu_internal.h
+++ b/arch/x86/kvm/mmu/mmu_internal.h
@@ -297,6 +297,7 @@ static inline int kvm_mmu_do_page_fault(struct kvm_vcpu *vcpu, gpa_t cr2_or_gpa,
        .max_level = KVM_MAX_HUGEPAGE_LEVEL,
        .req_level = PG_LEVEL_4K,
        .goal_level = PG_LEVEL_4K,
+       .is_private = kvm_mem_is_private(vcpu->kvm, cr2_or_gpa >> PAGE_SHIFT),
    };
    int r;
```
* 引入新函数 `kvm_is_vm_type_supported()` 判断 VM 的类型是否支持
  * 如果是缺省类型 `KVM_X86_DEFAULT_VM`，必然是支持的
  * 如果是 `KVM_X86_SW_PROTECTED_VM` 类型，则需开启 `CONFIG_KVM_SW_PROTECTED_VM` 和 TDP MMU
```cpp
static bool kvm_is_vm_type_supported(unsigned long type)
{
    return type == KVM_X86_DEFAULT_VM ||
           (type == KVM_X86_SW_PROTECTED_VM &&
        IS_ENABLED(CONFIG_KVM_SW_PROTECTED_VM) && tdp_enabled);
}
```
* 在 `kvm_vm_ioctl_check_extension()` 中引入对 `KVM_CAP_VM_TYPES` 能力的支持
  * 缺省类型 `KVM_X86_DEFAULT_VM` 是都支持的，将 bit `0` 置位
  * 如果开启了 `KVM_X86_SW_PROTECTED_VM`，将 bit `1` 置位
```diff
@@ -4739,6 +4746,11 @@ int kvm_vm_ioctl_check_extension(struct kvm *kvm, long ext)
    case KVM_CAP_X86_NOTIFY_VMEXIT:
        r = kvm_caps.has_notify_vmexit;
        break;
+   case KVM_CAP_VM_TYPES:
+       r = BIT(KVM_X86_DEFAULT_VM);
+       if (kvm_is_vm_type_supported(KVM_X86_SW_PROTECTED_VM))
+           r |= BIT(KVM_X86_SW_PROTECTED_VM);
+       break;
    default:
        break;
    }
```
* 在 `kvm_arch_init_vm()` 中检查将要创建的 VM 的类型是否支持，如果支持，记录在 `kvm->arch.vm_type` 中
```diff
@@ -12436,9 +12448,11 @@ int kvm_arch_init_vm(struct kvm *kvm, unsigned long type)
    int ret;
    unsigned long flags;

-   if (type)
+   if (!kvm_is_vm_type_supported(type))
        return -EINVAL;

+   kvm->arch.vm_type = type;
+
    ret = kvm_page_track_init(kvm);
    if (ret)
        goto out;
```
* 引入 `KVM_CAP_VM_TYPES` 的宏定义
```diff
diff --git a/include/uapi/linux/kvm.h b/include/uapi/linux/kvm.h
index 8eb10f560c69..e9cb2df67a1d 100644
--- a/include/uapi/linux/kvm.h
+++ b/include/uapi/linux/kvm.h
@@ -1227,6 +1227,7 @@ struct kvm_ppc_resize_hpt {
 #define KVM_CAP_MEMORY_FAULT_INFO 232
 #define KVM_CAP_MEMORY_ATTRIBUTES 233
 #define KVM_CAP_GUEST_MEMFD 234
+#define KVM_CAP_VM_TYPES 235

 #ifdef KVM_CAP_IRQ_ROUTING
```
* 引入 `KVM_GENERIC_PRIVATE_MEM` Kconfig
  * virt/kvm/Kconfig
```py
config KVM_GENERIC_PRIVATE_MEM
       select KVM_GENERIC_MEMORY_ATTRIBUTES
       select KVM_PRIVATE_MEM
       bool
```

## [PATCH] KVM: guest-memfd: fix unused-function warning
```cpp
With migration disabled, one function becomes unused:

virt/kvm/guest_memfd.c:262:12: error: 'kvm_gmem_migrate_folio' defined but not used [-Werror=unused-function]
  262 | static int kvm_gmem_migrate_folio(struct address_space *mapping,
      |            ^~~~~~~~~~~~~~~~~~~~~~

Remove the #ifdef around the reference so that fallback_migrate_folio()
is never used.  The gmem implementation of the hook is trivial; since
the gmem mapping is unmovable, the pages should not be migrated anyway.

Fixes: a7800aa80ea4 ("KVM: Add KVM_CREATE_GUEST_MEMFD ioctl() for guest-specific backing memory")
```
---
```diff
diff --git a/virt/kvm/guest_memfd.c b/virt/kvm/guest_memfd.c
index b99272396119..c2e2371720a9 100644
--- a/virt/kvm/guest_memfd.c
+++ b/virt/kvm/guest_memfd.c
@@ -300,9 +300,7 @@ static int kvm_gmem_error_page(struct address_space *mapping, struct page *page)

 static const struct address_space_operations kvm_gmem_aops = {
    .dirty_folio = noop_dirty_folio,
-#ifdef CONFIG_MIGRATION
    .migrate_folio  = kvm_gmem_migrate_folio,
-#endif
    .error_remove_page = kvm_gmem_error_page,
 };
```

## [PATCH 22/34] KVM: selftests: Drop unused kvm_userspace_memory_region_find() helper
* 删除 `kvm_userspace_memory_region_find()`，它未使用且是一个糟糕的 API（可能是它未使用的原因）。
* 如果 kvm_util.c 之外的任何内容需要访问 memslot，则可以暴露 `userspace_mem_region_find()`，以便其他人可以完全访问所有内存区域/slot 信息。

## [PATCH 23/34] KVM: selftests: Convert lib's mem regions to KVM_SET_USER_MEMORY_REGION2
* 在整个 KVM 的 selftest 库中使用 `KVM_SET_USER_MEMORY_REGION2`，以便可以添加对 guest 私有内存的支持，而无需完全独立的 helpers 集。
* 请注意，这显然会使 selftest 从现在起与旧的 KVM 版本向后不兼容。
---
* 诸如此类的改动，不全贴了
```diff
diff --git a/tools/testing/selftests/kvm/include/kvm_util_base.h b/tools/testing/selftests/kvm/include/kvm_util_base.h
index 967eaaeacd75..9f144841c2ee 100644
--- a/tools/testing/selftests/kvm/include/kvm_util_base.h
+++ b/tools/testing/selftests/kvm/include/kvm_util_base.h
@@ -44,7 +44,7 @@ typedef uint64_t vm_paddr_t; /* Virtual Machine (Guest) physical address */
 typedef uint64_t vm_vaddr_t; /* Virtual Machine (Guest) virtual address */

 struct userspace_mem_region {
-   struct kvm_userspace_memory_region region;
+   struct kvm_userspace_memory_region2 region;
    struct sparsebit *unused_phy_pages;
    int fd;
    off_t offset;
diff --git a/tools/testing/selftests/kvm/lib/kvm_util.c b/tools/testing/selftests/kvm/lib/kvm_util.c
index f09295d56c23..3676b37bea38 100644
--- a/tools/testing/selftests/kvm/lib/kvm_util.c
+++ b/tools/testing/selftests/kvm/lib/kvm_util.c
@@ -453,8 +453,9 @@ void kvm_vm_restart(struct kvm_vm *vmp)
 		vm_create_irqchip(vmp);
 
 	hash_for_each(vmp->regions.slot_hash, ctr, region, slot_node) {
-		int ret = ioctl(vmp->fd, KVM_SET_USER_MEMORY_REGION, &region->region);
-		TEST_ASSERT(ret == 0, "KVM_SET_USER_MEMORY_REGION IOCTL failed,\n"
+		int ret = ioctl(vmp->fd, KVM_SET_USER_MEMORY_REGION2, &region->region);
+
+		TEST_ASSERT(ret == 0, "KVM_SET_USER_MEMORY_REGION2 IOCTL failed,\n"
 			    "  rc: %i errno: %i\n"
 			    "  slot: %u flags: 0x%x\n"
 			    "  guest_phys_addr: 0x%llx size: 0x%llx",
```

## [PATCH 24/34] KVM: selftests: Add support for creating private memslots
* 添加对通过 `KVM_CREATE_GUEST_MEMFD` 和 `KVM_SET_USER_MEMORY_REGION2` 创建“私有”memslot 的支持。
  * 使 `vm_userspace_mem_region_add()` 成为其有效替代品 `vm_mem_add()` 的 wrapper，以便私有 memslots 完全选择加入，即不需要更新所有添加内存区域的测试。
* 以 `KVM_MEM_PRIVATE` 标志为中心，而不是“gmem”文件描述符的有效性，这样简单的测试就可以让 `vm_mem_add()` 完成创建 guest memfd 的繁重工作，
  * 同时还允许调用者传入显式的 `fd + offset`，以便更高级的测试可以执行诸如使用单个文件支持多个 memslots 之类的操作。
  * 如果调用者传入一个 `fd`，则 `dup()` 该 `fd`，以便
    * (a) `__vm_mem_region_delete()` 可以关闭与内存区域关联的 `fd`，而无需另一个标志，并且
    * (b) 以便调用者可以安全地关闭其副本 `fd` 无需首先销毁 memslots。
---
* tools/testing/selftests/kvm/include/kvm_util_base.h
```cpp
static inline int __vm_create_guest_memfd(struct kvm_vm *vm, uint64_t size,
                    uint64_t flags)
{
    struct kvm_create_guest_memfd guest_memfd = {
        .size = size,
        .flags = flags,
    };

    return __vm_ioctl(vm, KVM_CREATE_GUEST_MEMFD, &guest_memfd);
}

static inline int vm_create_guest_memfd(struct kvm_vm *vm, uint64_t size,
                    uint64_t flags)
{
    int fd = __vm_create_guest_memfd(vm, size, flags);

    TEST_ASSERT(fd >= 0, KVM_IOCTL_ERROR(KVM_CREATE_GUEST_MEMFD, fd));
    return fd;
}
```

## [PATCH 25/34] KVM: selftests: Add helpers to convert guest memory b/w private and shared
* 添加 helpers 以通过 KVM 的内存属性在私有和共享之间转换内存，以及添加 helpers 以通过 `fallocate()` 释放/分配 guest_memfd 内存。
  * 用户空间（即测试）在转换内存时不需要执行 `fallocate()`，因为属性是唯一的事实来源。
* 提供 `allocate()` helpers，以便测试可以模拟在转换时释放私有内存的用户空间，例如 将内存使用优先于性能。
---
* 很好的例子，看看用户态怎么用
```cpp
static inline void vm_set_memory_attributes(struct kvm_vm *vm, uint64_t gpa,
                        uint64_t size, uint64_t attributes)
{
    struct kvm_memory_attributes attr = {
        .attributes = attributes,
        .address = gpa,
        .size = size,
        .flags = 0,
    };

    /*
     * KVM_SET_MEMORY_ATTRIBUTES overwrites _all_ attributes.  These flows
     * need significant enhancements to support multiple attributes.
     */
    TEST_ASSERT(!attributes || attributes == KVM_MEMORY_ATTRIBUTE_PRIVATE,
            "Update me to support multiple attributes!");

    vm_ioctl(vm, KVM_SET_MEMORY_ATTRIBUTES, &attr);
}

static inline void vm_mem_set_private(struct kvm_vm *vm, uint64_t gpa,
                      uint64_t size)
{
    vm_set_memory_attributes(vm, gpa, size, KVM_MEMORY_ATTRIBUTE_PRIVATE);
}

static inline void vm_mem_set_shared(struct kvm_vm *vm, uint64_t gpa,
                     uint64_t size)
{
    vm_set_memory_attributes(vm, gpa, size, 0);
}

void vm_guest_mem_fallocate(struct kvm_vm *vm, uint64_t gpa, uint64_t size,
                bool punch_hole);

static inline void vm_guest_mem_punch_hole(struct kvm_vm *vm, uint64_t gpa,
                       uint64_t size)
{
    vm_guest_mem_fallocate(vm, gpa, size, true);
}

static inline void vm_guest_mem_allocate(struct kvm_vm *vm, uint64_t gpa,
                     uint64_t size)
{
    vm_guest_mem_fallocate(vm, gpa, size, false);
}
```
* 最后一个参数 `punch_hole` 控制要不要打洞
```cpp
void vm_guest_mem_fallocate(struct kvm_vm *vm, uint64_t base, uint64_t size,
                bool punch_hole)
{
    const int mode = FALLOC_FL_KEEP_SIZE | (punch_hole ? FALLOC_FL_PUNCH_HOLE : 0);
    struct userspace_mem_region *region;
    uint64_t end = base + size;
    uint64_t gpa, len;
    off_t fd_offset;
    int ret;

    for (gpa = base; gpa < end; gpa += len) {
        uint64_t offset;

        region = userspace_mem_region_find(vm, gpa, gpa);
        TEST_ASSERT(region && region->region.flags & KVM_MEM_GUEST_MEMFD,
                "Private memory region not found for GPA 0x%lx", gpa);

        offset = gpa - region->region.guest_phys_addr;
        fd_offset = region->region.guest_memfd_offset + offset;
        len = min_t(uint64_t, end - gpa, region->region.memory_size - offset);

        ret = fallocate(region->region.guest_memfd, mode, fd_offset, len);
        TEST_ASSERT(!ret, "fallocate() failed to %s at %lx (len = %lu), fd = %d, mode = %x, offset = %lx",
                punch_hole ? "punch hole" : "allocate", gpa, len,
                region->region.guest_memfd, mode, fd_offset);
    }
}
```
## [PATCH 26/34] KVM: selftests: Add helpers to do KVM_HC_MAP_GPA_RANGE hypercalls (x86)
* 为 x86 guest 添加 helpers 以调用 `KVM_HC_MAP_GPA_RANGE` hypercall，KVM 会将其转发到用户空间，因此测试可以使用它来协调 host 用户空间代码和 guest 代码之间的私有<=>共享转换。
---
* 这些函数运行在 guest mode
```cpp
static inline uint64_t __kvm_hypercall_map_gpa_range(uint64_t gpa,
                             uint64_t size, uint64_t flags)
{
    return kvm_hypercall(KVM_HC_MAP_GPA_RANGE, gpa, size >> PAGE_SHIFT, flags, 0);
}

static inline void kvm_hypercall_map_gpa_range(uint64_t gpa, uint64_t size,
                           uint64_t flags)
{
    uint64_t ret = __kvm_hypercall_map_gpa_range(gpa, size, flags);

    GUEST_ASSERT(!ret);
}
```

## [PATCH 27/34] KVM: selftests: Introduce VM "shape" to allow tests to specify the VM type
* 添加“`vm_shape`”结构来封装 selftest 定义的“mode”，以及 KVM 定义的“type”，以便在创建新 VM 时使用。
  * “mode”跟踪物理和虚拟地址属性，以及首选的后备内存类型，而“type”对应于 VM 类型。
* 采用 VM 类型将允许添加对 `KVM_CREATE_GUEST_MEMFD`（又名 guest 私有内存）的测试，而不需要一组完全独立的 helper。
  Guest 私有内存只能由机密 VM 类型有效使用，预计 x86 将加倍，并且需要 TDX 和 SNP guest 的独特 VM 类型。
---
* tools/testing/selftests/kvm/include/kvm_util_base.h
```cpp
struct vm_shape {
    enum vm_guest_mode mode;
    unsigned int type;
};

#define VM_TYPE_DEFAULT         0

#define VM_SHAPE(__mode)            \
({                      \
    struct vm_shape shape = {       \
        .mode = (__mode),       \
        .type = VM_TYPE_DEFAULT     \
    };                  \
                        \
    shape;                  \
})
...
#define VM_SHAPE_DEFAULT   VM_SHAPE(VM_MODE_DEFAULT)
```
* 其他改动没贴

## [PATCH 28/34] KVM: selftests: Add GUEST_SYNC[1-6] macros for synchronizing more data
* 添加 `GUEST_SYNC[1-6]()` 以便测试可以通过 `ucall()` 传递支持的最大信息量，而无需诉诸共享内存。
---
```cpp
#define GUEST_SYNC1(arg0)   ucall(UCALL_SYNC, 1, arg0)
#define GUEST_SYNC2(arg0, arg1) ucall(UCALL_SYNC, 2, arg0, arg1)
#define GUEST_SYNC3(arg0, arg1, arg2) \
                ucall(UCALL_SYNC, 3, arg0, arg1, arg2)
#define GUEST_SYNC4(arg0, arg1, arg2, arg3) \
                ucall(UCALL_SYNC, 4, arg0, arg1, arg2, arg3)
#define GUEST_SYNC5(arg0, arg1, arg2, arg3, arg4) \
                ucall(UCALL_SYNC, 5, arg0, arg1, arg2, arg3, arg4)
#define GUEST_SYNC6(arg0, arg1, arg2, arg3, arg4, arg5) \
                ucall(UCALL_SYNC, 6, arg0, arg1, arg2, arg3, arg4, arg5)
```

## [PATCH 29/34] KVM: selftests: Add x86-only selftest for private memory conversions
* 添加 selftest 以在 KVM 中执行隐式/显式转换功能并验证：
  - 共享内存对 host 用户空间可见
  - 私有内存对 host 用户空间不可见
  - Host 用户空间和 guest 可以通过共享内存进行通信
  - 共享 backing 中的数据在转换中保留（测试的 host 用户空间不会释放数据）
  - 私有内存与 VM 的生命周期绑定
* 理想情况下，KVM 的 selftests 基础设施将被重新设计，以允许为 *所有* backing 类型和 shape 支持具有多个 memslot 的单个 guest 内存区域，
  * 例如，理想情况下，跨多个 memslots 使用单个 backing fd 的代码也适用于“常规”内存。
  * 但遗憾的是，对 `KVM_CREATE_GUEST_MEMFD` 的支持已经停滞（languished）太久了，彻底检修（overhauling）selftest 的 memslots 基础设施可能就像打开一盒蠕虫，即进一步延迟事情。
* 除了更明显的测试之外，还要验证 `PUNCH_HOLE` 是否确实释放了内存。
  * 如果可能的话，直接验证 KVM 释放内存是不切实际的，因此可以通过断言 guest 在 `PUNCH_HOLE` 之后读取零来间接验证内存是否已释放。
  * 例如，如果 KVM 去除 SPTE 但实际上并未在 inode 中打洞，则后续读取仍将看到先前的值。
    * 显然，打一个洞不应该引起爆炸。
* 让用户在私有内存转换测试中指定 memslots 的数量，即不要求 memslots 的数量为“`1`”或“`nr_vcpus`”。
  * 创建比 vCPU 更多的 memslots 特别有趣，例如，它可能导致单个 `KVM_SET_MEMORY_ATTRIBUTES` 跨越多个 memslots。
  * 为了保持数学合理，将每个 vCPU 的 chunk 调整为至少 `2MiB`（大小为 `2MiB+4KiB`），并要求总大小能够完全除以 memslot 的数量。
  * 目标是能够验证 KVM 是否能够与多个 memslots 配合良好，能够创建真正任意数量的 memslots 并不会增加有意义的价值，即不值得付出成本。
* 故意不对 `KVM_CAP_GUEST_MEMFD` 提出要求，`KVM_CAP_MEMORY_FAULT_INFO`、`KVM_MEMORY_ATTRIBUTE_PRIVATE` 等，因为在没有先决条件的情况下宣传 `KVM_X86_SW_PROTECTED_VM` 是一个 KVM bug。
---
* 引入内存转换测试文件 `private_mem_conversions_test.c`
```diff
diff --git a/tools/testing/selftests/kvm/Makefile b/tools/testing/selftests/kvm/Makefile
index a5963ab9215b..ecdea5e7afa8 100644
--- a/tools/testing/selftests/kvm/Makefile
+++ b/tools/testing/selftests/kvm/Makefile
@@ -91,6 +91,7 @@ TEST_GEN_PROGS_x86_64 += x86_64/monitor_mwait_test
 TEST_GEN_PROGS_x86_64 += x86_64/nested_exceptions_test
 TEST_GEN_PROGS_x86_64 += x86_64/platform_info_test
 TEST_GEN_PROGS_x86_64 += x86_64/pmu_event_filter_test
+TEST_GEN_PROGS_x86_64 += x86_64/private_mem_conversions_test
 TEST_GEN_PROGS_x86_64 += x86_64/set_boot_cpu_id
 TEST_GEN_PROGS_x86_64 += x86_64/set_sregs_test
 TEST_GEN_PROGS_x86_64 += x86_64/smaller_maxphyaddr_emulation_test
```
* 引入 guest 侧样式验证宏和 host 侧样式验证宏
  * tools/testing/selftests/kvm/x86_64/private_mem_conversions_test.c
```cpp
#define BASE_DATA_SLOT		10
#define BASE_DATA_GPA		((uint64_t)(1ull << 32))
#define PER_CPU_DATA_SIZE	((uint64_t)(SZ_2M + PAGE_SIZE))

/* Horrific macro so that the line info is captured accurately :-( */
#define memcmp_g(gpa, pattern,  size)								\
do {												\
	uint8_t *mem = (uint8_t *)gpa;								\
	size_t i;										\
												\
	for (i = 0; i < size; i++)								\
		__GUEST_ASSERT(mem[i] == pattern,						\
			       "Guest expected 0x%x at offset %lu (gpa 0x%lx), got 0x%x",	\
			       pattern, i, gpa + i, mem[i]);					\
} while (0)

static void memcmp_h(uint8_t *mem, uint64_t gpa, uint8_t pattern, size_t size)
{
	size_t i;

	for (i = 0; i < size; i++)
		TEST_ASSERT(mem[i] == pattern,
			    "Host expected 0x%x at gpa 0x%lx, got 0x%x",
			    pattern, gpa + i, mem[i]);
}
```
* 使用显式转换运行内存转换测试：
  * 执行 KVM hypercall 来映射/取消映射 gpa 范围，这将导致用户空间退出以恢复/取消恢复私有内存。guest 对 gpa 范围的后续访问不会导致退出用户空间。
* 通过以下步骤测试内存转换场景：
  1) 使用 *私有访问* 来访问私有内存，并验证内存内容对用户空间不可见。
  2) 使用显式转换将内存转换为共享内存，并确保用户空间能够访问共享区域。
  3) 使用显式转换将内存转换回私有内存，并确保用户空间再次无法访问转换后的私有区域。
* 测试的接口函数和宏定义
```cpp
/*
 * Run memory conversion tests with explicit conversion:
 * Execute KVM hypercall to map/unmap gpa range which will cause userspace exit
 * to back/unback private memory. Subsequent accesses by guest to the gpa range
 * will not cause exit to userspace.
 *
 * Test memory conversion scenarios with following steps:
 * 1) Access private memory using private access and verify that memory contents
 *   are not visible to userspace.
 * 2) Convert memory to shared using explicit conversions and ensure that
 *   userspace is able to access the shared regions.
 * 3) Convert memory back to private using explicit conversions and ensure that
 *   userspace is again not able to access converted private regions.
 */

#define GUEST_STAGE(o, s) { .offset = o, .size = s }

enum ucall_syncs {
	SYNC_SHARED,
	SYNC_PRIVATE,
};
```
* `guest_sync_shared()` 发 `ucall(UCALL_SYNC)` 让 host 的用户态测试 `[gpa, gpa + size]` 范围的内容是否为当前样式 `current_pattern`，如果没问题，将范围的内容设置为新样式 `new_pattern`
* `guest_sync_private()` 发 `ucall(UCALL_SYNC)` 让 host 的用户态测试 `[gpa, gpa + size]` 范围的内容是否为样式 `pattern`
```cpp
static void guest_sync_shared(uint64_t gpa, uint64_t size,
			      uint8_t current_pattern, uint8_t new_pattern)
{
	GUEST_SYNC5(SYNC_SHARED, gpa, size, current_pattern, new_pattern);
}

static void guest_sync_private(uint64_t gpa, uint64_t size, uint8_t pattern)
{
	GUEST_SYNC4(SYNC_PRIVATE, gpa, size, pattern);
}

/* Arbitrary values, KVM doesn't care about the attribute flags. */
#define MAP_GPA_SET_ATTRIBUTES	BIT(0)
#define MAP_GPA_SHARED		BIT(1)
#define MAP_GPA_DO_FALLOCATE	BIT(2)

static void guest_map_mem(uint64_t gpa, uint64_t size, bool map_shared,
			  bool do_fallocate)
{   //必然会设置内存属性
	uint64_t flags = MAP_GPA_SET_ATTRIBUTES;

	if (map_shared)
		flags |= MAP_GPA_SHARED;
	if (do_fallocate)
		flags |= MAP_GPA_DO_FALLOCATE;
	kvm_hypercall_map_gpa_range(gpa, size, flags);
}

static void guest_map_shared(uint64_t gpa, uint64_t size, bool do_fallocate)
{
	guest_map_mem(gpa, size, true, do_fallocate);
}

static void guest_map_private(uint64_t gpa, uint64_t size, bool do_fallocate)
{
	guest_map_mem(gpa, size, false, do_fallocate);
}
```
* 测试的 GPA 和大小的定义
```cpp
struct {
	uint64_t offset;
	uint64_t size;
} static const test_ranges[] = {
	GUEST_STAGE(0, PAGE_SIZE),
	GUEST_STAGE(0, SZ_2M),
	GUEST_STAGE(PAGE_SIZE, PAGE_SIZE),
	GUEST_STAGE(PAGE_SIZE, SZ_2M),
	GUEST_STAGE(SZ_2M, PAGE_SIZE),
};

static void guest_test_explicit_conversion(uint64_t base_gpa, bool do_fallocate)
{
	const uint8_t def_p = 0xaa;
	const uint8_t init_p = 0xcc;
	uint64_t j;
	int i;
	//GPA 基址开始大小为 PER_CPU_DATA_SIZE 的内容设为 0xaa
	/* Memory should be shared by default. */
	memset((void *)base_gpa, def_p, PER_CPU_DATA_SIZE);
	memcmp_g(base_gpa, def_p, PER_CPU_DATA_SIZE); //比较范围的内容和 0xaa
	guest_sync_shared(base_gpa, PER_CPU_DATA_SIZE, def_p, init_p); //ucall(SYNC_SHARED) 将信息传到用户空间
	//host 接收到 ucall(SYNC_SHARED) 后先比较指定 GPA 开始大小为 PER_CPU_DATA_SIZE 的内容为 0xaa，然后将内容设为 0xcc
	memcmp_g(base_gpa, init_p, PER_CPU_DATA_SIZE); //检查 host 对 call(SYNC_SHARED) 的处理，内容是否设为 0xcc

	for (i = 0; i < ARRAY_SIZE(test_ranges); i++) {
		uint64_t gpa = base_gpa + test_ranges[i].offset;
		uint64_t size = test_ranges[i].size;
		uint8_t p1 = 0x11;
		uint8_t p2 = 0x22;
		uint8_t p3 = 0x33;
		uint8_t p4 = 0x44;
		//将测试区域设置为样式 1，以将其与整个数据范围区分开来（包含初始样式）。
		/*
		 * Set the test region to pattern one to differentiate it from
		 * the data range as a whole (contains the initial pattern).
		 */
		memset((void *)gpa, p1, size);
		//转换为私有，设置并验证私有数据，然后验证（映射为共享的）其余数据仍然保留初始样式，并且 host 始终看到共享内存（初始样式）。
		//与共享内存不同，在私有内存中打洞具有破坏性，即不能保证保留先前的值。
		/*
		 * Convert to private, set and verify the private data, and
		 * then verify that the rest of the data (map shared) still
		 * holds the initial pattern, and that the host always sees the
		 * shared memory (initial pattern).  Unlike shared memory,
		 * punching a hole in private memory is destructive, i.e.
		 * previous values aren't guaranteed to be preserved.
		 */
		guest_map_private(gpa, size, do_fallocate); //转为私有，如果 do_fallocate 为 true 会产生打洞效果
		//如果大小大于 1 页则将该页设为样式 2，然后跳到最后；
		if (size > PAGE_SIZE) {
			memset((void *)gpa, p2, PAGE_SIZE);
			goto skip;
		}
		//否则，将测试范围都设为样式 2，并让 host 将区域范围内容与样式 1 比较
		memset((void *)gpa, p2, size);
		guest_sync_private(gpa, size, p1); //转为私有后 host 为何看到的是样式 1？！
		//验证私有内存是否设置为样式 2，并且共享内存是否仍保留初始样式。
		/*
		 * Verify that the private memory was set to pattern two, and
		 * that shared memory still holds the initial pattern.
		 */
		memcmp_g(gpa, p2, size);
		if (gpa > base_gpa)
			memcmp_g(base_gpa, init_p, gpa - base_gpa); //验证前面部分是否还是初始值
		if (gpa + size < base_gpa + PER_CPU_DATA_SIZE)
			memcmp_g(gpa + size, init_p,
				 (base_gpa + PER_CPU_DATA_SIZE) - (gpa + size)); //验证后面部分是否还是初始值
		//将奇数 page 转换回共享 page，以验证 KVM 是否也能正确处理私有范围中的洞。
		/*
		 * Convert odd-number page frames back to shared to verify KVM
		 * also correctly handles holes in private ranges.
		 */
		for (j = 0; j < size; j += PAGE_SIZE) {
			if ((j >> PAGE_SHIFT) & 1) { //奇数 page
				guest_map_shared(gpa + j, PAGE_SIZE, do_fallocate); //转换回共享 page
				guest_sync_shared(gpa + j, PAGE_SIZE, p1, p3); //host 来验证 guest 设置的样式 1，并设为样式 3
                //guest 来验证 host 设置的样式 3
				memcmp_g(gpa + j, p3, PAGE_SIZE);
			} else { //偶数 page
				guest_sync_private(gpa + j, PAGE_SIZE, p1); //host 验证私有内容是否为样式 1
			}
		}

skip:	//将整个区域转换回共享区域，在要求 host 验证（并写入样式 4）之前显式写入样式 3 以填充偶数帧。
		/*
		 * Convert the entire region back to shared, explicitly write
		 * pattern three to fill in the even-number frames before
		 * asking the host to verify (and write pattern four).
		 */
		guest_map_shared(gpa, size, do_fallocate); //转为共享
		memset((void *)gpa, p3, size);             //写入样式 3
		guest_sync_shared(gpa, size, p3, p4);      //要求 host 验证样式 3 并写入样式 4
		memcmp_g(gpa, p4, size);                   //guest 验证刚才 host 写入的样式 4
		//将共享内存重置回初始样式。
		/* Reset the shared memory back to the initial pattern. */
		memset((void *)gpa, init_p, size);
		//释放（通过 PUNCH_HOLE）*所有*私有内存，以便下一次迭代从干净的石板开始，例如，关于 guest_mem 中是否有 page/folios。
		/*
		 * Free (via PUNCH_HOLE) *all* private memory so that the next
		 * iteration starts from a clean slate, e.g. with respect to
		 * whether or not there are pages/folios in guest_mem.
		 */
		guest_map_shared(base_gpa, PER_CPU_DATA_SIZE, true);
	}
}
```
* 通过 `fallocate()` 将内存“映射”为共享是借助 `PUNCH_HOLE` 完成的。
```cpp
static void guest_punch_hole(uint64_t gpa, uint64_t size)
{
	/* "Mapping" memory shared via fallocate() is done via PUNCH_HOLE. */
	uint64_t flags = MAP_GPA_SHARED | MAP_GPA_DO_FALLOCATE;

	kvm_hypercall_map_gpa_range(gpa, size, flags);
}
```
* 测试 `PUNCH_HOLE` 实际上通过打洞来释放内存，而不进行适当的转换。
  * 释放（`PUNCH_HOLE`）应该消除 SPTE，并且重新分配（后续 fault）应该将内存归零。
```cpp
/*
 * Test that PUNCH_HOLE actually frees memory by punching holes without doing a
 * proper conversion.  Freeing (PUNCH_HOLE) should zap SPTEs, and reallocating
 * (subsequent fault) should zero memory.
 */
static void guest_test_punch_hole(uint64_t base_gpa, bool precise)
{
	const uint8_t init_p = 0xcc;
	int i;
	//将整个范围转换为私有，这个测试用例都是关于在 guest_memfd 中打洞，即不需要共享映射。
	/*
	 * Convert the entire range to private, this testcase is all about
	 * punching holes in guest_memfd, i.e. shared mappings aren't needed.
	 */
	guest_map_private(base_gpa, PER_CPU_DATA_SIZE, false);

	for (i = 0; i < ARRAY_SIZE(test_ranges); i++) {
		uint64_t gpa = base_gpa + test_ranges[i].offset;
		uint64_t size = test_ranges[i].size;
		//在每次迭代之前释放所有内存，即使对于内存将出现故障的不精确情况也是如此。释放和重新分配显然应该有效，并且释放所有内存可以最大限度地减少跨测试用例影响的可能性。
		/*
		 * Free all memory before each iteration, even for the !precise
		 * case where the memory will be faulted back in.  Freeing and
		 * reallocating should obviously work, and freeing all memory
		 * minimizes the probability of cross-testcase influence.
		 */
		guest_punch_hole(base_gpa, PER_CPU_DATA_SIZE);
		//Fault-in 和初始化内存，且校验样式
		/* Fault-in and initialize memory, and verify the pattern. */
		if (precise) { //仅目标范围 faulted in
			memset((void *)gpa, init_p, size);
			memcmp_g(gpa, init_p, size);
		} else { //整个 guest_memfd faulted in
			memset((void *)base_gpa, init_p, PER_CPU_DATA_SIZE);
			memcmp_g(base_gpa, init_p, PER_CPU_DATA_SIZE);
		}
		//在目标范围打一个洞，并验证来自 guest 的读取是否成功并返回零。
		/*
		 * Punch a hole at the target range and verify that reads from
		 * the guest succeed and return zeroes.
		 */
		guest_punch_hole(gpa, size);
		memcmp_g(gpa, 0, size);
	}
}
```
* Guest 要跑的代码的入口
* 运行转换测试两次，在共享和私有之间转换时，在 guest_memfd backing 上分别执行或不执行 `fallocate()`。
* 运行 `PUNCH_HOLE` 测试两次，一次是整个 guest_memfd faulted in，一次是仅目标范围 faulted in。
```cpp
static void guest_code(uint64_t base_gpa)
{
	/*
	 * Run the conversion test twice, with and without doing fallocate() on
	 * the guest_memfd backing when converting between shared and private.
	 */
	guest_test_explicit_conversion(base_gpa, false); //guest_map_private() 只设置内存属性为 KVM_MEMORY_ATTRIBUTE_PRIVATE，不打洞
	guest_test_explicit_conversion(base_gpa, true); //guest_map_private() 不但设置内存属性 KVM_MEMORY_ATTRIBUTE_PRIVATE，还会打洞

	/*
	 * Run the PUNCH_HOLE test twice too, once with the entire guest_memfd
	 * faulted in, once with only the target range faulted in.
	 */
	guest_test_punch_hole(base_gpa, false);
	guest_test_punch_hole(base_gpa, true);
	GUEST_DONE();
}
```
* 用户态的 hypercall 处理的入口，根据请求分配、打洞、设置内存的私有属性
```cpp
static void handle_exit_hypercall(struct kvm_vcpu *vcpu)
{
	struct kvm_run *run = vcpu->run;
	uint64_t gpa = run->hypercall.args[0];
	uint64_t size = run->hypercall.args[1] * PAGE_SIZE;
	bool set_attributes = run->hypercall.args[2] & MAP_GPA_SET_ATTRIBUTES;
	bool map_shared = run->hypercall.args[2] & MAP_GPA_SHARED;
	bool do_fallocate = run->hypercall.args[2] & MAP_GPA_DO_FALLOCATE;
	struct kvm_vm *vm = vcpu->vm;

	TEST_ASSERT(run->hypercall.nr == KVM_HC_MAP_GPA_RANGE,
		    "Wanted MAP_GPA_RANGE (%u), got '%llu'",
		    KVM_HC_MAP_GPA_RANGE, run->hypercall.nr);

	if (do_fallocate)
		vm_guest_mem_fallocate(vm, gpa, size, map_shared);

	if (set_attributes)
		vm_set_memory_attributes(vm, gpa, size,
					 map_shared ? 0 : KVM_MEMORY_ATTRIBUTE_PRIVATE);
	run->hypercall.ret = 0;
}
```
* 测试内存转换的主循环函数，在用户态把 VM run 起来，处理各种 VM exit 事件
  * Guest 通过 `ucall()`（实为 hypercall）到 host 后，host 通过 `get_ucall()` 获取 guest 传递过来的参数
  * Case `UCALL_SYNC` 是要处理的主要内容
```cpp
static bool run_vcpus;

static void *__test_mem_conversions(void *__vcpu)
{
	struct kvm_vcpu *vcpu = __vcpu;
	struct kvm_run *run = vcpu->run;
	struct kvm_vm *vm = vcpu->vm;
	struct ucall uc;

	while (!READ_ONCE(run_vcpus))
		;

	for ( ;; ) {
		vcpu_run(vcpu);

		if (run->exit_reason == KVM_EXIT_HYPERCALL) {
			handle_exit_hypercall(vcpu);
			continue;
		}

		TEST_ASSERT(run->exit_reason == KVM_EXIT_IO,
			    "Wanted KVM_EXIT_IO, got exit reason: %u (%s)",
			    run->exit_reason, exit_reason_str(run->exit_reason));

		switch (get_ucall(vcpu, &uc)) {
		case UCALL_ABORT:
			REPORT_GUEST_ASSERT(uc);
		case UCALL_SYNC: {
			uint64_t gpa  = uc.args[1];
			size_t size = uc.args[2];
			size_t i;

			TEST_ASSERT(uc.args[0] == SYNC_SHARED ||
				    uc.args[0] == SYNC_PRIVATE,
				    "Unknown sync command '%ld'", uc.args[0]);

			for (i = 0; i < size; i += vm->page_size) {
				size_t nr_bytes = min_t(size_t, vm->page_size, size - i);
				uint8_t *hva = addr_gpa2hva(vm, gpa + i);

				/* In all cases, the host should observe the shared data. */
				memcmp_h(hva, gpa + i, uc.args[3], nr_bytes);

				/* For shared, write the new pattern to guest memory. */
				if (uc.args[0] == SYNC_SHARED)
					memset(hva, uc.args[4], nr_bytes);
			}
			break;
		}
		case UCALL_DONE:
			return NULL;
		default:
			TEST_FAIL("Unknown ucall 0x%lx.", uc.cmd);
		}
	}
}
```
* `test_mem_conversions()` 是测试入口函数，负责以下主要工作：
  * 创建虚拟机
  * 创建 memfd
  * 分配和创建 memslots
  * 给每个 CPU 的一段地址建立 VM 虚拟地址到 VM 的物理地址的映射
  * 调用 guest 主循环 `__test_mem_conversions()`
```cpp
static void test_mem_conversions(enum vm_mem_backing_src_type src_type, uint32_t nr_vcpus,
				 uint32_t nr_memslots)
{
	/*
	 * Allocate enough memory so that each vCPU's chunk of memory can be
	 * naturally aligned with respect to the size of the backing store.
	 */
	const size_t alignment = max_t(size_t, SZ_2M, get_backing_src_pagesz(src_type));
	const size_t per_cpu_size = align_up(PER_CPU_DATA_SIZE, alignment);
	const size_t memfd_size = per_cpu_size * nr_vcpus;
	const size_t slot_size = memfd_size / nr_memslots;
	struct kvm_vcpu *vcpus[KVM_MAX_VCPUS];
	pthread_t threads[KVM_MAX_VCPUS];
	struct kvm_vm *vm;
	int memfd, i, r;

	const struct vm_shape shape = {
		.mode = VM_MODE_DEFAULT,
		.type = KVM_X86_SW_PROTECTED_VM,
	};

	TEST_ASSERT(slot_size * nr_memslots == memfd_size,
		    "The memfd size (0x%lx) needs to be cleanly divisible by the number of memslots (%u)",
		    memfd_size, nr_memslots);
	vm = __vm_create_with_vcpus(shape, nr_vcpus, 0, guest_code, vcpus);

	vm_enable_cap(vm, KVM_CAP_EXIT_HYPERCALL, (1 << KVM_HC_MAP_GPA_RANGE));

	memfd = vm_create_guest_memfd(vm, memfd_size, 0);

	for (i = 0; i < nr_memslots; i++)
		vm_mem_add(vm, src_type, BASE_DATA_GPA + slot_size * i,
			   BASE_DATA_SLOT + i, slot_size / vm->page_size,
			   KVM_MEM_GUEST_MEMFD, memfd, slot_size * i);

	for (i = 0; i < nr_vcpus; i++) {
		uint64_t gpa =  BASE_DATA_GPA + i * per_cpu_size;

		vcpu_args_set(vcpus[i], 1, gpa);

		/*
		 * Map only what is needed so that an out-of-bounds access
		 * results #PF => SHUTDOWN instead of data corruption.
		 */
		virt_map(vm, gpa, gpa, PER_CPU_DATA_SIZE / vm->page_size);

		pthread_create(&threads[i], NULL, __test_mem_conversions, vcpus[i]);
	}

	WRITE_ONCE(run_vcpus, true);

	for (i = 0; i < nr_vcpus; i++)
		pthread_join(threads[i], NULL);

	kvm_vm_free(vm);
	//关闭 VM fd 后，从 guest_memfd 分配和释放内存。guest_memfd 被赋予对其所属虚拟机的引用，例如，应防止虚拟机被完全销毁，直到对 guest_memfd 的最后一个引用也被放回。
	/*
	 * Allocate and free memory from the guest_memfd after closing the VM
	 * fd.  The guest_memfd is gifted a reference to its owning VM, i.e.
	 * should prevent the VM from being fully destroyed until the last
	 * reference to the guest_memfd is also put.
	 */
	r = fallocate(memfd, FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE, 0, memfd_size);
	TEST_ASSERT(!r, __KVM_SYSCALL_ERROR("fallocate()", r));

	r = fallocate(memfd, FALLOC_FL_KEEP_SIZE, 0, memfd_size);
	TEST_ASSERT(!r, __KVM_SYSCALL_ERROR("fallocate()", r));
}
```
* `main()` 函数和帮助信息函数 `usage()`，其中 `main()` 根据参数调用测试入口函数 `test_mem_conversions()`
```cpp
static void usage(const char *cmd)
{
	puts("");
	printf("usage: %s [-h] [-m nr_memslots] [-s mem_type] [-n nr_vcpus]\n", cmd);
	puts("");
	backing_src_help("-s");
	puts("");
	puts(" -n: specify the number of vcpus (default: 1)");
	puts("");
	puts(" -m: specify the number of memslots (default: 1)");
	puts("");
}

int main(int argc, char *argv[])
{
	enum vm_mem_backing_src_type src_type = DEFAULT_VM_MEM_SRC;
	uint32_t nr_memslots = 1;
	uint32_t nr_vcpus = 1;
	int opt;

	TEST_REQUIRE(kvm_check_cap(KVM_CAP_VM_TYPES) & BIT(KVM_X86_SW_PROTECTED_VM));

	while ((opt = getopt(argc, argv, "hm:s:n:")) != -1) {
		switch (opt) {
		case 's':
			src_type = parse_backing_src_type(optarg);
			break;
		case 'n':
			nr_vcpus = atoi_positive("nr_vcpus", optarg);
			break;
		case 'm':
			nr_memslots = atoi_positive("nr_memslots", optarg);
			break;
		case 'h':
		default:
			usage(argv[0]);
			exit(0);
		}
	}

	test_mem_conversions(src_type, nr_vcpus, nr_memslots);

	return 0;
}
```

## [PATCH 30/34] KVM: selftests: Add KVM_SET_USER_MEMORY_REGION2 helper
* 添加 helpers 以直接调用 `KVM_SET_USER_MEMORY_REGION2`，以便测试可以验证“set user memory region”的“version 2”特有的功能，例如，对 gmem_fd 和 gmem_offset 进行负面测试。
* 提供原始版本以及断言成功版本，以减少基本使用所需的样板（boilerplate）代码量。
---
```cpp
int __vm_set_user_memory_region2(struct kvm_vm *vm, uint32_t slot, uint32_t flags,
                 uint64_t gpa, uint64_t size, void *hva,
                 uint32_t guest_memfd, uint64_t guest_memfd_offset)
{
    struct kvm_userspace_memory_region2 region = {
        .slot = slot,
        .flags = flags,
        .guest_phys_addr = gpa,
        .memory_size = size,
        .userspace_addr = (uintptr_t)hva,
        .guest_memfd = guest_memfd,
        .guest_memfd_offset = guest_memfd_offset,
    };

    return ioctl(vm->fd, KVM_SET_USER_MEMORY_REGION2, &region);
}

void vm_set_user_memory_region2(struct kvm_vm *vm, uint32_t slot, uint32_t flags,
                uint64_t gpa, uint64_t size, void *hva,
                uint32_t guest_memfd, uint64_t guest_memfd_offset)
{
    int ret = __vm_set_user_memory_region2(vm, slot, flags, gpa, size, hva,
                           guest_memfd, guest_memfd_offset);

    TEST_ASSERT(!ret, "KVM_SET_USER_MEMORY_REGION2 failed, errno = %d (%s)",
            errno, strerror(errno));
}
```

## [PATCH 31/34] KVM: selftests: Expand set_memory_region_test to validate guest_memfd()
* 展开 `set_memory_region_test` 来练习私有内存的各种正向和负向测试用例。
  - 私有内存的非 `guest_memfd()` 文件描述符
  - 来自不同虚拟机的 `guest_memfd()`
  - 重叠的绑定
  - 未对齐的绑定
---
* tools/testing/selftests/kvm/include/kvm_util_base.h
```cpp
#ifdef __x86_64__
static inline struct kvm_vm *vm_create_barebones_protected_vm(void)
{
    const struct vm_shape shape = {
        .mode = VM_MODE_DEFAULT,
        .type = KVM_X86_SW_PROTECTED_VM,
    };

    return ____vm_create(shape);
}
#endif
```
* tools/testing/selftests/kvm/set_memory_region_test.c
```cpp
#ifdef __x86_64__
static void test_invalid_guest_memfd(struct kvm_vm *vm, int memfd,
                     size_t offset, const char *msg)
{
    int r = __vm_set_user_memory_region2(vm, MEM_REGION_SLOT, KVM_MEM_GUEST_MEMFD,
                         MEM_REGION_GPA, MEM_REGION_SIZE,
                         0, memfd, offset);
    TEST_ASSERT(r == -1 && errno == EINVAL, "%s", msg);
}
//一组反向测试
static void test_add_private_memory_region(void)
{
    struct kvm_vm *vm, *vm2;
    int memfd, i;

    pr_info("Testing ADD of KVM_MEM_GUEST_MEMFD memory regions\n");

    vm = vm_create_barebones_protected_vm();
    //私有内存的非 guest_memfd() 文件描述符：KVM fd 和 VM fd
    test_invalid_guest_memfd(vm, vm->kvm_fd, 0, "KVM fd should fail");
    test_invalid_guest_memfd(vm, vm->fd, 0, "VM's fd should fail");
    //常规 memfd 而不是 guest_memfd
    memfd = kvm_memfd_alloc(MEM_REGION_SIZE, false);
    test_invalid_guest_memfd(vm, memfd, 0, "Regular memfd() should fail");
    close(memfd);
    //来自不同虚拟机的 guest_memfd()
    vm2 = vm_create_barebones_protected_vm(); //创建第二个虚拟机
    memfd = vm_create_guest_memfd(vm2, MEM_REGION_SIZE, 0); //创建第二个虚拟机的 guest_memfd
    test_invalid_guest_memfd(vm, memfd, 0, "Other VM's guest_memfd() should fail"); //vm1 用 vm2 的 guest_memfd

    vm_set_user_memory_region2(vm2, MEM_REGION_SLOT, KVM_MEM_GUEST_MEMFD,
                   MEM_REGION_GPA, MEM_REGION_SIZE, 0, memfd, 0);
    close(memfd);
    kvm_vm_free(vm2);
    //未对齐的绑定反向测试
    memfd = vm_create_guest_memfd(vm, MEM_REGION_SIZE, 0);
    for (i = 1; i < PAGE_SIZE; i++)
        test_invalid_guest_memfd(vm, memfd, i, "Unaligned offset should fail");

    vm_set_user_memory_region2(vm, MEM_REGION_SLOT, KVM_MEM_GUEST_MEMFD,
                   MEM_REGION_GPA, MEM_REGION_SIZE, 0, memfd, 0);
    close(memfd);

    kvm_vm_free(vm);
}
//测试重叠的绑定
static void test_add_overlapping_private_memory_regions(void)
{
    struct kvm_vm *vm;
    int memfd;
    int r;

    pr_info("Testing ADD of overlapping KVM_MEM_GUEST_MEMFD memory regions\n");

    vm = vm_create_barebones_protected_vm();

    memfd = vm_create_guest_memfd(vm, MEM_REGION_SIZE * 4, 0);

    vm_set_user_memory_region2(vm, MEM_REGION_SLOT, KVM_MEM_GUEST_MEMFD,
                   MEM_REGION_GPA, MEM_REGION_SIZE * 2, 0, memfd, 0);

    vm_set_user_memory_region2(vm, MEM_REGION_SLOT + 1, KVM_MEM_GUEST_MEMFD,
                  MEM_REGION_GPA * 2, MEM_REGION_SIZE * 2,
                   0, memfd, MEM_REGION_SIZE * 2);
    //删除第一个 memslot，然后尝试重新创建它，除了用一个“错误的”偏移，导致 guest_memfd() 中出现重叠。
    /*
     * Delete the first memslot, and then attempt to recreate it except
     * with a "bad" offset that results in overlap in the guest_memfd().
     */
    vm_set_user_memory_region2(vm, MEM_REGION_SLOT, KVM_MEM_GUEST_MEMFD,
                   MEM_REGION_GPA, 0, NULL, -1, 0);
    //与另一个 slot 的前半部分重叠。
    /* Overlap the front half of the other slot. */
    r = __vm_set_user_memory_region2(vm, MEM_REGION_SLOT, KVM_MEM_GUEST_MEMFD,
                     MEM_REGION_GPA * 2 - MEM_REGION_SIZE,
                     MEM_REGION_SIZE * 2,
                     0, memfd, 0);
    TEST_ASSERT(r == -1 && errno == EEXIST, "%s",
            "Overlapping guest_memfd() bindings should fail with EEXIST");
    //现在是另一个 slot 的后半部分。
    /* And now the back half of the other slot. */
    r = __vm_set_user_memory_region2(vm, MEM_REGION_SLOT, KVM_MEM_GUEST_MEMFD,
                     MEM_REGION_GPA * 2 + MEM_REGION_SIZE,
                     MEM_REGION_SIZE * 2,
                     0, memfd, 0);
    TEST_ASSERT(r == -1 && errno == EEXIST, "%s",
            "Overlapping guest_memfd() bindings should fail with EEXIST");

    close(memfd);
    kvm_vm_free(vm);
}
#endif
```
* 增加这两个测试用例
```diff
@@ -402,6 +494,14 @@ int main(int argc, char *argv[])
    test_add_max_memory_regions();

 #ifdef __x86_64__
+   if (kvm_has_cap(KVM_CAP_GUEST_MEMFD) &&
+       (kvm_check_cap(KVM_CAP_VM_TYPES) & BIT(KVM_X86_SW_PROTECTED_VM))) {
+       test_add_private_memory_region();
+       test_add_overlapping_private_memory_regions();
+   } else {
+       pr_info("Skipping tests for KVM_MEM_GUEST_MEMFD memory regions\n");
+   }
+
    if (argc > 1)
        loops = atoi_positive("Number of iterations", argv[1]);
    else
```

## [PATCH 32/34] KVM: selftests: Add basic selftest for guest_memfd()
* 添加 selftest 来验证 `guest_memfd()` 的基本功能：
  - 使用 `guest_memfd()` `ioctl` 创建的文件描述符 *不允许读/写/`mmap` 操作*
  - 从 `fstat` 返回的文件大小和 block 大小符合预期
  - 在 `fd` 上 `fallocate`，检查 `fallocate(FALLOC_FL_PUNCH_HOLE)` 上的偏移量/长度应页对齐
  - 拒绝无效输入（尺寸未对齐、无效标志）
  - 文件大小和 `inode` 是唯一的（听起来无害的 `anon_inode_getfile()` 支持具有单个 `inode` 的所有文件...）
---
* 增加 `guest_memfd_test` 测试文件
```diff
diff --git a/tools/testing/selftests/kvm/Makefile b/tools/testing/selftests/kvm/Makefile
index ecdea5e7afa8..fd3b30a4ca7b 100644
--- a/tools/testing/selftests/kvm/Makefile
+++ b/tools/testing/selftests/kvm/Makefile
@@ -134,6 +134,7 @@ TEST_GEN_PROGS_x86_64 += access_tracking_perf_test
 TEST_GEN_PROGS_x86_64 += demand_paging_test
 TEST_GEN_PROGS_x86_64 += dirty_log_test
 TEST_GEN_PROGS_x86_64 += dirty_log_perf_test
+TEST_GEN_PROGS_x86_64 += guest_memfd_test
 TEST_GEN_PROGS_x86_64 += guest_print_test
 TEST_GEN_PROGS_x86_64 += hardware_disable_test
 TEST_GEN_PROGS_x86_64 += kvm_create_max_vcpus
```
* 测试读写 guest_memfd 文件，应该会失败
```cpp
static void test_file_read_write(int fd)
{
	char buf[64];

	TEST_ASSERT(read(fd, buf, sizeof(buf)) < 0,
		    "read on a guest_mem fd should fail");
	TEST_ASSERT(write(fd, buf, sizeof(buf)) < 0,
		    "write on a guest_mem fd should fail");
	TEST_ASSERT(pread(fd, buf, sizeof(buf), 0) < 0,
		    "pread on a guest_mem fd should fail");
	TEST_ASSERT(pwrite(fd, buf, sizeof(buf), 0) < 0,
		    "pwrite on a guest_mem fd should fail");
}
```
* 测试 `mmap()` guest_memfd 文件，应该会失败
```cpp
static void test_mmap(int fd, size_t page_size)
{
	char *mem;

	mem = mmap(NULL, page_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
	TEST_ASSERT_EQ(mem, MAP_FAILED);
}
```
* 测试 `fstat()` guest_memfd 文件，应该会成功
```cpp
static void test_file_size(int fd, size_t page_size, size_t total_size)
{
	struct stat sb;
	int ret;

	ret = fstat(fd, &sb);
	TEST_ASSERT(!ret, "fstat should succeed");
	TEST_ASSERT_EQ(sb.st_size, total_size);
	TEST_ASSERT_EQ(sb.st_blksize, page_size);
}
```
* 测试 `fallocate()` guest_memfd 文件
```cpp
static void test_fallocate(int fd, size_t page_size, size_t total_size)
{
	int ret;
	//以对齐的偏移和大小 fallocate，应该成功
	ret = fallocate(fd, FALLOC_FL_KEEP_SIZE, 0, total_size);
	TEST_ASSERT(!ret, "fallocate with aligned offset and size should succeed");
	//以非对齐的偏移打洞，应该失败
	ret = fallocate(fd, FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE,
			page_size - 1, page_size);
	TEST_ASSERT(ret, "fallocate with unaligned offset should fail");
	//以保持大小的方式，偏移又在结尾处开始分配，应该失败
	ret = fallocate(fd, FALLOC_FL_KEEP_SIZE, total_size, page_size);
	TEST_ASSERT(ret, "fallocate beginning at total_size should fail");
	//以保持大小的方式，偏移又在结尾 + page_size 处开始分配，应该失败
	ret = fallocate(fd, FALLOC_FL_KEEP_SIZE, total_size + page_size, page_size);
	TEST_ASSERT(ret, "fallocate beginning after total_size should fail");
	//以保持大小的方式，偏移又在结尾处开始打洞，应该成功
	ret = fallocate(fd, FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE,
			total_size, page_size);
	TEST_ASSERT(!ret, "fallocate(PUNCH_HOLE) at total_size should succeed");
	//以保持大小的方式，偏移又在结尾 + page_size 处开始打洞，应该成功
	ret = fallocate(fd, FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE,
			total_size + page_size, page_size);
	TEST_ASSERT(!ret, "fallocate(PUNCH_HOLE) after total_size should succeed");
	//以非对齐的尺寸去打洞，应该失败
	ret = fallocate(fd, FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE,
			page_size, page_size - 1);
	TEST_ASSERT(ret, "fallocate with unaligned size should fail");
	//以对齐的偏移和尺寸去打洞，应该成功
	ret = fallocate(fd, FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE,
			page_size, page_size);
	TEST_ASSERT(!ret, "fallocate(PUNCH_HOLE) with aligned offset and size should succeed");
	//以对齐的偏移和尺寸去还原被打过的洞，应该成功
	ret = fallocate(fd, FALLOC_FL_KEEP_SIZE, page_size, page_size);
	TEST_ASSERT(!ret, "fallocate to restore punched hole should succeed");
}
```
* 一组关于 guest_memfd 打洞的反向测试，应该都会失败
  * 1-3 偏移从 0 开始，尺寸不对齐
  * 4-7 偏移不对齐，其中 6 尺寸对齐，其余的的尺寸也不对齐
  * 8-10 偏移对齐，尺寸不对齐
```cpp
static void test_invalid_punch_hole(int fd, size_t page_size, size_t total_size)
{
	struct {
		off_t offset;
		off_t len;
	} testcases[] = {
		{0, 1},
		{0, page_size - 1},
		{0, page_size + 1},

		{1, 1},
		{1, page_size - 1},
		{1, page_size},
		{1, page_size + 1},

		{page_size, 1},
		{page_size, page_size - 1},
		{page_size, page_size + 1},
	};
	int ret, i;

	for (i = 0; i < ARRAY_SIZE(testcases); i++) {
		ret = fallocate(fd, FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE,
				testcases[i].offset, testcases[i].len);
		TEST_ASSERT(ret == -1 && errno == EINVAL,
			    "PUNCH_HOLE with !PAGE_SIZE offset (%lx) and/or length (%lx) should fail",
			    testcases[i].offset, testcases[i].len);
	}
}
```
* 一组关于创建 guest_memfd 的反向测试
  * 大小从 `1~4095` 这样非对齐的方式
  * 目前约定的有效 `flag` 是 `0`，因此任何非零的 `flags` 都会导致返回失败 `EINVAL`
```cpp
static void test_create_guest_memfd_invalid(struct kvm_vm *vm)
{
	size_t page_size = getpagesize();
	uint64_t flag;
	size_t size;
	int fd;

	for (size = 1; size < page_size; size++) {
		fd = __vm_create_guest_memfd(vm, size, 0);
		TEST_ASSERT(fd == -1 && errno == EINVAL,
			    "guest_memfd() with non-page-aligned page size '0x%lx' should fail with EINVAL",
			    size);
	}

	for (flag = 0; flag; flag <<= 1) {
		fd = __vm_create_guest_memfd(vm, page_size, flag);
		TEST_ASSERT(fd == -1 && errno == EINVAL,
			    "guest_memfd() with flag '0x%lx' should fail with EINVAL",
			    flag);
	}
}
```
* 一组 guest_memfd 的正向测试
  * 第一次 guest_memfd 的大小不受第二次创建 guest_memfd 的影响
  * 两次 guest_memfd 的 inode 号必然不能相等
```cpp
static void test_create_guest_memfd_multiple(struct kvm_vm *vm)
{
	int fd1, fd2, ret;
	struct stat st1, st2;

	fd1 = __vm_create_guest_memfd(vm, 4096, 0);
	TEST_ASSERT(fd1 != -1, "memfd creation should succeed");

	ret = fstat(fd1, &st1);
	TEST_ASSERT(ret != -1, "memfd fstat should succeed");
	TEST_ASSERT(st1.st_size == 4096, "memfd st_size should match requested size");

	fd2 = __vm_create_guest_memfd(vm, 8192, 0);
	TEST_ASSERT(fd2 != -1, "memfd creation should succeed");

	ret = fstat(fd2, &st2);
	TEST_ASSERT(ret != -1, "memfd fstat should succeed");
	TEST_ASSERT(st2.st_size == 8192, "second memfd st_size should match requested size");

	ret = fstat(fd1, &st1);
	TEST_ASSERT(ret != -1, "memfd fstat should succeed");
	TEST_ASSERT(st1.st_size == 4096, "first memfd st_size should still match requested size");
	TEST_ASSERT(st1.st_ino != st2.st_ino, "different memfd should have different inode numbers");
}
```
* 调用以上 selftest cases
```cpp
int main(int argc, char *argv[])
{
	size_t page_size;
	size_t total_size;
	int fd;
	struct kvm_vm *vm;

	TEST_REQUIRE(kvm_has_cap(KVM_CAP_GUEST_MEMFD));

	page_size = getpagesize();
	total_size = page_size * 4;

	vm = vm_create_barebones();

	test_create_guest_memfd_invalid(vm);
	test_create_guest_memfd_multiple(vm);

	fd = vm_create_guest_memfd(vm, total_size, 0);

	test_file_read_write(fd);
	test_mmap(fd, page_size);
	test_file_size(fd, page_size, total_size);
	test_fallocate(fd, page_size, total_size);
	test_invalid_punch_hole(fd, page_size, total_size);

	close(fd);
}
```

## [PATCH 33/34] KVM: selftests: Test KVM exit behavior for private memory/access
* “删除 memslot 时测试私有访问”测试当 VM 正在使用私有 memslot 时删除私有 memslot 时 KVM 的行为。
  * 当 KVM 查找已删除（`slot = NULL`）memslot 时，KVM 应使用 `KVM_EXIT_MEMORY_FAULT` 退出到用户空间。
* 在第二个测试中，在对 *非私有* memslot 进行 *私有* 访问时，KVM 还应使用 `KVM_EXIT_MEMORY_FAULT` 退出到用户空间。
* 有意不要对 `KVM_CAP_GUEST_MEMFD`、`KVM_CAP_MEMORY_FAULT_INFO`、`KVM_MEMORY_ATTRIBUTE_PRIVATE` 等提出要求，因为在没有先决条件的情况下宣传 `KVM_X86_SW_PROTECTED_VM` 是一个 KVM 错误。
---
* 增加 `private_mem_kvm_exits_test` 测试用例
```diff
diff --git a/tools/testing/selftests/kvm/Makefile b/tools/testing/selftests/kvm/Makefile
index fd3b30a4ca7b..69ce8e06b3a3 100644
--- a/tools/testing/selftests/kvm/Makefile
+++ b/tools/testing/selftests/kvm/Makefile
@@ -92,6 +92,7 @@ TEST_GEN_PROGS_x86_64 += x86_64/nested_exceptions_test
 TEST_GEN_PROGS_x86_64 += x86_64/platform_info_test
 TEST_GEN_PROGS_x86_64 += x86_64/pmu_event_filter_test
 TEST_GEN_PROGS_x86_64 += x86_64/private_mem_conversions_test
+TEST_GEN_PROGS_x86_64 += x86_64/private_mem_kvm_exits_test
 TEST_GEN_PROGS_x86_64 += x86_64/set_boot_cpu_id
 TEST_GEN_PROGS_x86_64 += x86_64/set_sregs_test
 TEST_GEN_PROGS_x86_64 += x86_64/smaller_maxphyaddr_emulation_test
```
* tools/testing/selftests/kvm/x86_64/private_mem_kvm_exits_test.c
```cpp
/* Arbitrarily selected to avoid overlaps with anything else */
#define EXITS_TEST_GVA 0xc0000000
#define EXITS_TEST_GPA EXITS_TEST_GVA
#define EXITS_TEST_NPAGES 1
#define EXITS_TEST_SIZE (EXITS_TEST_NPAGES * PAGE_SIZE)
#define EXITS_TEST_SLOT 10
//由 Guest 调用，重复读取地址 `EXITS_TEST_GVA`
static uint64_t guest_repeatedly_read(void)
{
    volatile uint64_t value;

    while (true)
        value = *((uint64_t *) EXITS_TEST_GVA);

    return value;
}

static uint32_t run_vcpu_get_exit_reason(struct kvm_vcpu *vcpu)
{
    int r;

    r = _vcpu_run(vcpu);
    if (r) {
        TEST_ASSERT(errno == EFAULT, KVM_IOCTL_ERROR(KVM_RUN, r));
        TEST_ASSERT_EQ(vcpu->run->exit_reason, KVM_EXIT_MEMORY_FAULT);
    }
    return vcpu->run->exit_reason;
}

const struct vm_shape protected_vm_shape = {
    .mode = VM_MODE_DEFAULT,
    .type = KVM_X86_SW_PROTECTED_VM,
};

static void test_private_access_memslot_deleted(void)
{
    struct kvm_vm *vm;
    struct kvm_vcpu *vcpu;
    pthread_t vm_thread;
    void *thread_return;
    uint32_t exit_reason;
    //创建机密虚拟机，设置 guest 入口点为 guest_repeatedly_read
    vm = vm_create_shape_with_one_vcpu(protected_vm_shape, &vcpu,
                       guest_repeatedly_read);
    //添加一个私有的 memslot（flags = KVM_MEM_GUEST_MEMFD）
    vm_userspace_mem_region_add(vm, VM_MEM_SRC_ANONYMOUS,
                    EXITS_TEST_GPA, EXITS_TEST_SLOT,
                    EXITS_TEST_NPAGES,
                    KVM_MEM_GUEST_MEMFD);
    //帮 guest 建立好 GVA 到 GPA 的映射关系
    virt_map(vm, EXITS_TEST_GVA, EXITS_TEST_GPA, EXITS_TEST_NPAGES);
    //设置 memslot 的内存属性为 KVM_MEMORY_ATTRIBUTE_PRIVATE
    /* Request to access page privately */
    vm_mem_set_private(vm, EXITS_TEST_GPA, EXITS_TEST_SIZE);
    //让虚拟机跑起来
    pthread_create(&vm_thread, NULL,
               (void *(*)(void *))run_vcpu_get_exit_reason,
               (void *)vcpu);
    //这个时候，删除 memslot，这回导致 guest 因为内存出错而退出
    vm_mem_region_delete(vm, EXITS_TEST_SLOT);
    //获取线程函数 run_vcpu_get_exit_reason() 的返回值
    pthread_join(vm_thread, &thread_return);
    exit_reason = (uint32_t)(uint64_t)thread_return;
    //检查 vcpu->run->memory_fault 的各个域的值是否符合预期
    TEST_ASSERT_EQ(exit_reason, KVM_EXIT_MEMORY_FAULT);
    TEST_ASSERT_EQ(vcpu->run->memory_fault.flags, KVM_MEMORY_EXIT_FLAG_PRIVATE);
    TEST_ASSERT_EQ(vcpu->run->memory_fault.gpa, EXITS_TEST_GPA);
    TEST_ASSERT_EQ(vcpu->run->memory_fault.size, EXITS_TEST_SIZE);
    //释放 VM
    kvm_vm_free(vm);
}

static void test_private_access_memslot_not_private(void)
{
    struct kvm_vm *vm;
    struct kvm_vcpu *vcpu;
    uint32_t exit_reason;
    //创建机密虚拟机，设置 guest 入口点为 guest_repeatedly_read
    vm = vm_create_shape_with_one_vcpu(protected_vm_shape, &vcpu,
                       guest_repeatedly_read);
    //添加一个非私有的 memslot（flags = 0）
    /* Add a non-private memslot (flags = 0) */
    vm_userspace_mem_region_add(vm, VM_MEM_SRC_ANONYMOUS,
                    EXITS_TEST_GPA, EXITS_TEST_SLOT,
                    EXITS_TEST_NPAGES, 0);
    //帮 guest 建立好 GVA 到 GPA 的映射关系
    virt_map(vm, EXITS_TEST_GVA, EXITS_TEST_GPA, EXITS_TEST_NPAGES);
    //设置 memslot 的内存属性为 KVM_MEMORY_ATTRIBUTE_PRIVATE
    /* Request to access page privately */
    vm_mem_set_private(vm, EXITS_TEST_GPA, EXITS_TEST_SIZE);
    //在对 非私有 memslot 进行 私有 访问时，KVM 还应使用 KVM_EXIT_MEMORY_FAULT 退出到用户空间
    exit_reason = run_vcpu_get_exit_reason(vcpu);
    //检查 vcpu->run->memory_fault 的各个域的值是否符合预期
    TEST_ASSERT_EQ(exit_reason, KVM_EXIT_MEMORY_FAULT);
    TEST_ASSERT_EQ(vcpu->run->memory_fault.flags, KVM_MEMORY_EXIT_FLAG_PRIVATE);
    TEST_ASSERT_EQ(vcpu->run->memory_fault.gpa, EXITS_TEST_GPA);
    TEST_ASSERT_EQ(vcpu->run->memory_fault.size, EXITS_TEST_SIZE);
    //释放 VM
    kvm_vm_free(vm);
}
//把上面两个 cases 跑起来
int main(int argc, char *argv[])
{
    TEST_REQUIRE(kvm_check_cap(KVM_CAP_VM_TYPES) & BIT(KVM_X86_SW_PROTECTED_VM));

    test_private_access_memslot_deleted();
    test_private_access_memslot_not_private();
}
```

## [PATCH 34/34] KVM: selftests: Add a memory region subtest to validate invalid flags
* 将子测试添加到 `set_memory_region_test` 以验证 KVM 是否拒绝无效标志以及与 `-EINVAL` 的组合。
* 无论如何，KVM 可能会或可能不会因 `EINVAL` 而失败，但我们至少可以尝试。
---
* 上面给出的 `test_create_guest_memfd_invalid()` 已经是修改过后的版本。
删掉的部分意图是想把有效标志与无效
```diff
diff --git a/tools/testing/selftests/kvm/guest_memfd_test.c b/tools/testing/selftests/kvm/guest_memfd_test.c
index fd389663c49b..318ba6ba8bd3 100644
--- a/tools/testing/selftests/kvm/guest_memfd_test.c
+++ b/tools/testing/selftests/kvm/guest_memfd_test.c
@@ -136,20 +136,13 @@ static void test_create_guest_memfd_invalid(struct kvm_vm *vm)
                size);
    }

-   for (flag = 1; flag; flag <<= 1) {
+   for (flag = 0; flag; flag <<= 1) {
        uint64_t bit;

        fd = __vm_create_guest_memfd(vm, page_size, flag);
        TEST_ASSERT(fd == -1 && errno == EINVAL,
                "guest_memfd() with flag '0x%lx' should fail with EINVAL",
                flag);
-
-       for_each_set_bit(bit, &valid_flags, 64) {
-           fd = __vm_create_guest_memfd(vm, page_size, flag | BIT_ULL(bit));
-           TEST_ASSERT(fd == -1 && errno == EINVAL,
-                   "guest_memfd() with flags '0x%llx' should fail with EINVAL",
-                   flag | BIT_ULL(bit));
-       }
    }
 }
```
* 测试无效的内存区域 flags
  * tools/testing/selftests/kvm/set_memory_region_test.c
```cpp
static void test_invalid_memory_region_flags(void)
{
    uint32_t supported_flags = KVM_MEM_LOG_DIRTY_PAGES; //当前支持的 flags
    const uint32_t v2_only_flags = KVM_MEM_GUEST_MEMFD; //仅 v2 memory region 支持的 flags
    struct kvm_vm *vm;
    int r, i;
    //当前支持的 flags 加上 Arch64 和 x86-64 支持的 flags
#if defined __aarch64__ || defined __x86_64__
    supported_flags |= KVM_MEM_READONLY;
#endif
   //如果是 x86-64 且 VM 能力支持机密虚拟机，创建机密虚拟机
#ifdef __x86_64__
    if (kvm_check_cap(KVM_CAP_VM_TYPES) & BIT(KVM_X86_SW_PROTECTED_VM))
        vm = vm_create_barebones_protected_vm();
    else
#endif
        vm = vm_create_barebones();
    //如果内存属性支持私有内存，当前支持的 flags 加上 KVM_MEM_GUEST_MEMFD
    if (kvm_check_cap(KVM_CAP_MEMORY_ATTRIBUTES) & KVM_MEMORY_ATTRIBUTE_PRIVATE)
        supported_flags |= KVM_MEM_GUEST_MEMFD;
    //遍历 32 个 flag bits
    for (i = 0; i < 32; i++) {
        if ((supported_flags & BIT(i)) && !(v2_only_flags & BIT(i)))
            continue; //跳过支持，但又不在仅 v2 memory region 支持的 flags 范围内的 bit
        //不支持的 flags，或者是仅 v2 memory region 支持的 flags，用 v1 接口把 memory region 设为该 flag
        r = __vm_set_user_memory_region(vm, 0, BIT(i),
                        0, MEM_REGION_SIZE, NULL);
        //这会导致出错，因为不支持，或者因为是属于 v2 的 flag
        TEST_ASSERT(r && errno == EINVAL,
                "KVM_SET_USER_MEMORY_REGION should have failed on v2 only flag 0x%lx", BIT(i));
        //已支持的 flags 就不测了，跳过
        if (supported_flags & BIT(i))
            continue;
        //未支持的 flags 即便用 v2 接口去设置也会导致出错
        r = __vm_set_user_memory_region2(vm, 0, BIT(i),
                         0, MEM_REGION_SIZE, NULL, 0, 0);
        TEST_ASSERT(r && errno == EINVAL,
                "KVM_SET_USER_MEMORY_REGION2 should have failed on unsupported flag 0x%lx", BIT(i));
    }
    //如果支持 guest_memfd 标志，测试和 dirty log 标志组合设置，预期会出错
    if (supported_flags & KVM_MEM_GUEST_MEMFD) {
        r = __vm_set_user_memory_region2(vm, 0,
                         KVM_MEM_LOG_DIRTY_PAGES | KVM_MEM_GUEST_MEMFD,
                         0, MEM_REGION_SIZE, NULL, 0, 0);
        TEST_ASSERT(r && errno == EINVAL,
                "KVM_SET_USER_MEMORY_REGION2 should have failed, dirty logging private memory is unsupported");
    }
}
```

## Qemu 对 Guest Memory 的支持
* TDX 类型 VM 初始化的时候，把 `Machine` 的 `backend->guest_memfd` 设置为 `true`
```cpp
//target/i386/kvm/tdx.c
tdx_kvm_init()
   ms->require_guest_memfd = true;
//backends/hostmem.c
host_memory_backend_init()
   backend->guest_memfd = machine_require_guest_memfd(machine);
   -> return machine->require_guest_memfd;
```
* 创建 memory backend 的时候在 `ram_block_add()` 函数中调用 `kvm_create_guest_memfd() -> ioctl(KVM_CREATE_GUEST_MEMFD)` 把 `guest_memfd` 创建好
```cpp
//backends/hostmem-file.c
file_backend_memory_alloc()
   ram_flags |= backend->guest_memfd ? RAM_GUEST_MEMFD : 0;
   //system/memory.c
-> memory_region_init_ram_from_file(&backend->mr, OBJECT(backend), name, backend->size, fb->align, ram_flags, fb->mem_path, fb->offset, errp);
      //system/physmem.c
   -> qemu_ram_alloc_from_file(size, mr, ram_flags, path, offset, &err)
      -> qemu_ram_alloc_from_fd(size, mr, ram_flags, fd, offset, errp)

//backends/hostmem-ram.c
ram_backend_memory_alloc()
   ram_flags |= backend->guest_memfd ? RAM_GUEST_MEMFD : 0;
-> memory_region_init_ram_flags_nomigrate(&backend->mr, OBJECT(backend), name, backend->size, ram_flags, errp))

//backends/hostmem-memfd.c
memfd_backend_memory_alloc()
   ram_flags |= backend->guest_memfd ? RAM_GUEST_MEMFD : 0;
   //system/memory.c
-> memory_region_init_ram_from_fd(&backend->mr, OBJECT(backend), name, backend->size, ram_flags, fd, 0, errp);
   -> memory_region_init()
      //system/physmem.c
   -> mr->ram_block = qemu_ram_alloc_from_fd(size, mr, ram_flags, fd, offset, &err)
         new_block->guest_memfd = -1;
      -> new_block->host = file_ram_alloc(new_block, size, fd, !file_size, offset, errp);
            //util/mmap-alloc.c
         -> qemu_ram_mmap(fd, memory, block->mr->align, qemu_map_flags, offset);
            -> mmap_activate(guardptr + offset, size, fd, qemu_map_flags, map_offset)
               -> mmap()
         //system/physmem.c
      -> ram_block_add(new_block, &local_err)
            //accel/kvm/kvm-all.c
         -> new_block->guest_memfd = kvm_create_guest_memfd(new_block->max_length, 0, errp)
            -> fd = kvm_vm_ioctl(kvm_state, KVM_CREATE_GUEST_MEMFD, &guest_memfd)
```
* 在 commit memory region 的时候，把 `guest_memfd` 相关的信息放在 `ioctl(KVM_SET_USER_MEMORY_REGION2)` 的参数内传递给 kernel
* 设置 gmem xarray 的属性为 `KVM_MEMORY_ATTRIBUTE_PRIVATE`，
  * 调用 `kvm_vm_ioctl(kvm_state, KVM_SET_MEMORY_ATTRIBUTES, &attrs)` 会导致 KVM 最终调用 `tdp_mmu_zap_leafs()` 去把共享页面的页表拆掉（如果有）
  * 避免将来逐个访问 guest memory region 的私有内存产生 `KVM_EXIT_MEMORY_FAULT`，再 upcall 到 qemu 进行 `kvm_convert_memory()` shared -> private 转化的超长路径
```cpp
//accel/kvm/kvm-all.c
kvm_region_commit()
-> kvm_set_phys_mem()
      /* register the new slot */
      mem->guest_memfd = mr->ram_block->guest_memfd;
      mem->guest_memfd_offset = (uint8_t*)ram - mr->ram_block->host;
   -> kvm_set_user_memory_region(kml, mem, true)
         mem.guest_memfd = slot->guest_memfd;
         mem.guest_memfd_offset = slot->guest_memfd_offset;
      -> kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION2, &mem)//让 KVM 去建立新的 memory region
      if (memory_region_has_guest_memfd(mr))//设置 gmem xarray 的属性为 PRIVATE
      -> kvm_set_memory_attributes_private(start_addr, slot_size)
         -> kvm_set_memory_attributes(start, size, KVM_MEMORY_ATTRIBUTE_PRIVATE)
            -> kvm_vm_ioctl(kvm_state, KVM_SET_MEMORY_ATTRIBUTES, &attrs)
```

## References
- [LWN: Guest-first memory for KVM](https://lwn.net/Articles/949277/)
- [LWN：KVM 使用 Guest-first memory](https://mp.weixin.qq.com/s/XqYuS3Btcdf20ipgEtd7Ug)