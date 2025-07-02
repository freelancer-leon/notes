# [PATCH v3 0/5] TDX host: kexec() support
- [[PATCH v3 0_5] TDX host kexec() support - Kai Huang](https://lore.kernel.org/linux-kernel/cover.1712493366.git.kai.huang@intel.com/)

## [PATCH 1/5] x86/kexec: do unconditional WBINVD for bare-metal in stop_this_cpu()
### TL;DR:
* 更改为在 bare metal 的 `stop_this_cpu()` 中执行无条件 `WBINVD`，以涵盖对 AMD SME 和 Intel TDX 的 `kexec` 支持，尽管曾经存在一些问题阻止这样做，但现在已得到修复。

### 长版本：
* 由于内存加密，SME 和 TDX 都可能使缓存处于不一致状态。在 kexec 期间，必须在跳转到第二个内核之前刷新缓存，以避免第二个内核的内存静默地损坏。
* 目前，对于 SME，当内核确定硬件支持 SME 时，内核仅在 `stop_this_cpu()` 中执行 `WBINVD`。为了支持 TDX，一种选择是将特定检查扩展到涵盖 SME 和 TDX。
* 然而，最好只执行无条件 `WBINVD`，而不是散布 vendor-specific 的检查。`Kexec()` 是一条缓慢的路径，为了使代码简单且易于维护，可以接受额外的 `WBINVD`。
* 但仅对 bare-metal 执行 `WBINVD`，因为 TDX guest 和 SEV-ES/SEV-SNP guest 将获得意外（但不必要）的 `#VE`，并且可能无法处理（例如，TDX guest 在由于 `WBINVD` 原因获得 `#VE` 时会出现 panics）。

### 注意：
* 从历史上看，存在一个阻止执行无条件 `WBINVD` 的问题，但该问题已得到解决。
* 当 SME `kexec()` 支持最初在 commit bba4ed011a52: ("x86/mm, kexec: Allow kexec to be used with SME") 中添加时，`WBINVD` 是无条件完成的。
  * 然而，从那时起，有报道称不同的 Intel 系统会因该 commit 而 hang 或 reset。
* 为了尝试修复它，稍后的 commit f23d74f6c66c: ("x86/mm: Rework wbinvd, hlt operation in stop_this_cpu()") 随后更改为仅在硬件支持 SME 时执行 `WBINVD`。
* 虽然此 commit 使报告的问题消失，但它没有查明根本原因。此外，它没有正确处理极端情况[*]，这导致了根本原因的揭示和最终修复的 commit 1f5e7eb7868e: ("x86/smp: Make stop_other_cpus() more robust")
  * 请参阅 [1][2] 了解更多信息。
* 根据上述修复对有问题的机器（最初报告的问题）进行无条件 `WBINVD` 的进一步测试确认了无法重现这些问题。
  * 请参阅 [3][4] 了解更多信息。
* 因此，现在进行无条件 `WBINVD` 是安全的。

- [*] 该 commit 未检查 `CPUID` leaf 是否可用。在 Intel 上用不支持的 `CPUID` leaf 会返回垃圾，导致意外的 `WBINVD`，从而导致一些问题（随后进行分析并揭示最终的根本原因）。极端情况由 commit 9b040453d444: ("x86/smp: Dont access non-existing CPUID leaf") 独立修复。
- [1] https://lore.kernel.org/lkml/CALu+AoQKmeixJdkO07t7BtttN7v3RM4_aBKi642bQ3fTBbSAVg@mail.gmail.com/T/#m300f3f9790850b5daa20a71abcc200ae8d94a12a
- [2] https://lore.kernel.org/lkml/CALu+AoQKmeixJdkO07t7BtttN7v3RM4_aBKi642bQ3fTBbSAVg@mail.gmail.com/T/#ma7263a7765483db0dabdeef62a1110940e634846
- [3] https://lore.kernel.org/lkml/CALu+AoQKmeixJdkO07t7BtttN7v3RM4_aBKi642bQ3fTBbSAVg@mail.gmail.com/T/#mc043191f2ff860d649c8466775dc61ac1e0ae320
- [4] https://lore.kernel.org/lkml/CALu+AoQKmeixJdkO07t7BtttN7v3RM4_aBKi642bQ3fTBbSAVg@mail.gmail.com/T/#md23f1a8f6afcc59fa2b0ac1967f18e418e24347c
---
* 注释来自“长版本”
```diff
 arch/x86/kernel/process.c | 18 ++++++++----------
 1 file changed, 8 insertions(+), 10 deletions(-)

diff --git a/arch/x86/kernel/process.c b/arch/x86/kernel/process.c
index b8441147eb5e..5ba8a9c1e47a 100644
--- a/arch/x86/kernel/process.c
+++ b/arch/x86/kernel/process.c
@@ -813,18 +813,16 @@ void __noreturn stop_this_cpu(void *dummy)
 	mcheck_cpu_clear(c);
 
 	/*
-	 * Use wbinvd on processors that support SME. This provides support
-	 * for performing a successful kexec when going from SME inactive
-	 * to SME active (or vice-versa). The cache must be cleared so that
-	 * if there are entries with the same physical address, both with and
-	 * without the encryption bit, they don't race each other when flushed
-	 * and potentially end up with the wrong entry being committed to
-	 * memory.
+	 * The kernel could leave caches in incoherent state on SME/TDX
+	 * capable platforms.  Flush cache to avoid silent memory
+	 * corruption for these platforms.
 	 *
-	 * Test the CPUID bit directly because the machine might've cleared
-	 * X86_FEATURE_SME due to cmdline options.
+	 * stop_this_cpu() is not a fast path, just do unconditional
+	 * WBINVD for simplicity.  But only do WBINVD for bare-metal
+	 * as TDX guests and SEV-ES/SEV-SNP guests will get unexpected
+	 * (and unnecessary) #VE and may unable to handle.
 	 */
-	if (c->extended_cpuid_level >= 0x8000001f && (cpuid_eax(0x8000001f) & BIT(0)))
+	if (!boot_cpu_has(X86_FEATURE_HYPERVISOR))
 		native_wbinvd();
```

## [PATCH v3 2/5] x86/kexec: do unconditional WBINVD for bare-metal in relocate_kernel()
* 由于内存加密，SME 和 TDX 都可能使缓存处于不一致状态。在 `kexec` 期间，必须在跳转到第二个内核之前刷新缓存，以避免第二个内核的静默地内存损坏。
* 在 kexec 期间，`stop_this_cpu()` 中的 `WBINVD` 会在所有远程 cpu 停止时刷新它们的缓存。对于 SME，`relocate_kernel()` 中的 `WBINVD` 会刷新最后运行的 cpu（正在执行 kexec）的缓存。
* 同样，为了支持 TDX host 的 kexec，在停止所有远程 cpu 并刷新缓存后，内核需要刷新最后运行的 cpu 的缓存。
* 也使用 `relocate_kernel()` 中现有的 `WBINVD` 来覆盖 TDX host。
* 然而，不要散布 vendor-specific 的检查，只需执行无条件 `WBINVD` 来覆盖 SME 和 TDX。Kexec 不是一种快速路径，因此为没有 SME/TDX 的平台添加一个额外的 `WBINVD` 是可以接受的。
* 但仅对 bare-metal 执行 `WBINVD`，因为 TDX guest 和 SEV-ES/SEV-SNP guest 将获得意外（但不必要）的 `#VE`，内核在现阶段无法处理。
---
```diff
diff --git a/arch/x86/include/asm/kexec.h b/arch/x86/include/asm/kexec.h
index 91ca9a9ee3a2..455f8a6c66a9 100644
--- a/arch/x86/include/asm/kexec.h
+++ b/arch/x86/include/asm/kexec.h
@@ -128,7 +128,7 @@ relocate_kernel(unsigned long indirection_page,
 		unsigned long page_list,
 		unsigned long start_address,
 		unsigned int preserve_context,
-		unsigned int host_mem_enc_active);
+		unsigned int bare_metal);
 #endif
 
 #define ARCH_HAS_KIMAGE_ARCH
```
* 根据 x86 的 ABI 约定，第六个参数 `bare_metal` 放在 `%r8`，随后将该值放入 `%r12`
* `relocate_kernel` 调用（实则是“返回”）`identity_mapped`，在这里判断是不是 `bare_metal`？是则 `wbinvd`
```diff
diff --git a/arch/x86/kernel/relocate_kernel_64.S b/arch/x86/kernel/relocate_kernel_64.S
index 56cab1bb25f5..3e04c5e5687f 100644
--- a/arch/x86/kernel/relocate_kernel_64.S
+++ b/arch/x86/kernel/relocate_kernel_64.S
@@ -50,7 +50,7 @@ SYM_CODE_START_NOALIGN(relocate_kernel)
 	 * %rsi page_list
 	 * %rdx start address
 	 * %rcx preserve_context
-	 * %r8  host_mem_enc_active
+	 * %r8  bare_metal
 	 */
 
 	/* Save the CPU context, used for jumping back */
@@ -78,7 +78,7 @@ SYM_CODE_START_NOALIGN(relocate_kernel)
 	pushq $0
 	popfq
 
-	/* Save SME active flag */
+	/* Save the bare_metal */
 	movq	%r8, %r12
 
 	/*
@@ -160,9 +160,13 @@ SYM_CODE_START_LOCAL_NOALIGN(identity_mapped)
 	movq	%r9, %cr3
 
 	/*
-	 * If SME is active, there could be old encrypted cache line
-	 * entries that will conflict with the now unencrypted memory
-	 * used by kexec. Flush the caches before copying the kernel.
+	 * The kernel could leave caches in incoherent state on SME/TDX
+	 * capable platforms.  Just do unconditional WBINVD to avoid
+	 * silent memory corruption to the new kernel for these platforms.
+	 *
+	 * But only do WBINVD for bare-metal because TDX guests and
+	 * SEV-ES/SEV-SNP guests will get #VE which the kernel is unable
+	 * to handle at this stage.
 	 */
 	testq	%r12, %r12
 	jz 1f
    wbinvd
1:
```
* 不要散布 vendor-specific 的检查，只需执行无条件 `WBINVD` 来覆盖 SME 和 TDX。Kexec 不是一种快速路径，因此为没有 SME/TDX 的平台添加一个额外的 `WBINVD` 是可以接受的。
```diff
diff --git a/arch/x86/kernel/machine_kexec_64.c b/arch/x86/kernel/machine_kexec_64.c
index b180d8e497c3..a454477b7b4c 100644
--- a/arch/x86/kernel/machine_kexec_64.c
+++ b/arch/x86/kernel/machine_kexec_64.c
@@ -358,7 +358,7 @@ void machine_kexec(struct kimage *image)
                       (unsigned long)page_list,
                       image->start,
                       image->preserve_context,
-                      cc_platform_has(CC_ATTR_HOST_MEM_ENCRYPT));
+                      !boot_cpu_has(X86_FEATURE_HYPERVISOR));

 #ifdef CONFIG_KEXEC_JUMP
    if (image->preserve_context)
```

## [PATCH v3 3/5] x86/kexec: Reset TDX private memory on platforms with TDX erratum
### TL;DR:
* 在具有 TDX“部分写入机器检查” erratum 的平台上，在 kexec 期间，在跳转到第二个内核之前将 TDX 私有内存转换回正常状态，以避免第二个内核看到潜在的意外 machine check。

### 长版本
* 前几代 TDX 硬件有一个 erratum。对 TDX 私有内存 cachline 的部分写入会默默地“poison”该行。后续读取将消耗 poison 并生成 machine check。根据 TDX 硬件规范，这两件事都不应该发生。

#### 背景
* 事实上，所有内核内存访问操作都发生在完整的 cacheline 中。实际上，写入一个“字节”内存通常会读取内存的 `64` 字节 cacheline，对其进行修改，然后将整行写回。这些操作不会触发此问题。
* 此问题是由“partial”写入触发的，其中少于 cacheline 的写入事务到达内存控制器。
  * CPU 通过非临时（non-temporal）写指令（如 `MOVNTI`）或通过 `UC/WC` 内存映射来完成这些操作。
  * 通过 DMA 进行部分写入的设备也可能从 CPU 触发该问题。

#### 问题
* 一次 fast warm reset 不会重置 TDX 私有内存。`Kexec()` 还可以直接引导到新内核。因此，如果旧内核在平台上留下了带有此 erratum 的任何 TDX 私有页面，则新内核可能会受到意外的 machine check。
* 请注意，如果没有此 erratum，任何内核在 TDX 私有内存上读/写都不会导致 machine check，因此旧内核可以按原样保留 TDX 私有页面。

#### 解决方案
* 简而言之，通过这个 erratum，内核需要显式地将所有 TDX 私有页面转换回正常状态，以便在 `kexec()` 之后为新内核提供干净的状态。
* BIOS 还应禁用 fast warm reset 作为此 erratum 的解决方法，因此此实现不会尝试在内核中重新启动情况下重置 TDX 私有内存，而是依赖 BIOS 来启用该解决方法。
* 在所有远程 cpu 停止并且在所有 cpu 上完成 cache 刷新之后，当不再发生更多 TDX 活动时，将 TDX 专用页面转换回正常状态（使用 `MOVDIR64B` 清除这些页面）。
* 在 `machine_kexec()` 中执行此操作以覆盖正常 kexec 和 crash kexec。
* 目前 TDX 私有内存只能是 PAMT 页。在这里涵盖所有类型的 TDX 私有内存是理想的选择，但这样做存在一些实际问题：
  1. 没有现有的基础设施来跟踪 TDX 私有页面；
  2. 无法查询 TDX module 的页面类型，因为 SEAMCALL 所需的 VMX 已被禁用；
  3. 即使查询 TDX module 可行，结果也可能不准确。例如，远程 CPU 可以在 `MOVDIR64B` 之前停止。
* 一种临时解决方案是盲目地转换所有内存页面，但这样做也有问题，因为在直接映射中并非所有页面都映射为可写。
  * 可以通过切换到为 `kexec()` 创建的相同映射或新的页表来完成，但复杂性看起来有些过大。
* 因此，与其做一些戏剧性的事情，不如在这里重置 PAMT 页面。
* 当将来其他 TDX 私有页面可能存在时，把他们重置作为将来的工作。
---
* 新增 `kexec_save_processor_start()` 函数
```cpp
static void kexec_save_processor_start(struct kimage *image)
{
#ifdef CONFIG_KEXEC_JUMP
    if (image->preserve_context)
        save_processor_state();
#endif
}
```
* 将 TDX 私有内存转换回正常状态（需要时），以避免第二个内核可能看到意外的 machine check。
  * 然而，当 `preserve_context` 打开时跳过这个。通过到达这里，TDX（如果内核启用了的话）在 `preserve_context` 打开时从 suspend 中幸存下来，并且它可以在从第二个内核跳回后继续工作。
```diff
@@ -298,10 +307,20 @@ void machine_kexec(struct kimage *image)
 	void *control_page;
 	int save_ftrace_enabled;
 
-#ifdef CONFIG_KEXEC_JUMP
-	if (image->preserve_context)
-		save_processor_state();
-#endif
+	kexec_save_processor_start(image);
+
+	/*
+	 * Convert TDX private memory back to normal (when needed) to
+	 * avoid the second kernel potentially seeing unexpected machine
+	 * check.
+	 *
+	 * However skip this when preserve_context is on.  By reaching
+	 * here, TDX (if ever got enabled by the kernel) has survived
+	 * from the suspend when preserve_context is on, and it can
+	 * continue to work after jumping back from the second kernel.
+	 */
+	if (!image->preserve_context)
+		tdx_reset_memory();
 
 	save_ftrace_enabled = __ftrace_enabled_save();
```
* 新增 `tdx_reset_memory()` 重置 TDX 私有内存，目前只能是 PAMT 页
```cpp
void tdx_reset_memory(void)
{
    if (!boot_cpu_has(X86_FEATURE_TDX_HOST_PLATFORM))
        return;

    /*
     * Converting TDX private pages back to normal must be done
     * when there's no TDX activity anymore on all remote cpus.
     * Verify this is only called when all remote cpus have
     * been stopped.
     */
    WARN_ON_ONCE(num_online_cpus() != 1);

    /*
     * Kernel read/write to TDX private memory doesn't cause
     * machine check on hardware w/o this erratum.
     */
    if (!boot_cpu_has_bug(X86_BUG_TDX_PW_MCE))
        return;

    /*
     * Nothing to convert if it's not possible to have any TDX
     * private pages.
     */
    if (!tdx_may_have_private_memory)
        return;

    /*
     * Ensure the 'tdx_tdmr_list' is stable for reading PAMTs
     * when tdx_may_have_private_memory reads true, paired with
     * the smp_wmb() in mark_may_have_private_memory().
     */
   smp_rmb();

    /*
     * All remote cpus have been stopped, and their caches have
     * been flushed in stop_this_cpu().  Now flush cache for the
     * last running cpu _before_ converting TDX private pages.
     */
    native_wbinvd();

    /*
     * Tell all in-kernel TDX users to reset TDX private pages
     * that they manage.
     */
    if (notify_reset_memory())
        pr_err("Failed to reset all TDX private pages.\n");

    /*
     * The only remaining TDX private pages are PAMT pages.
     * Reset them.
     */
    tdmrs_reset_pamt_all(&tdx_tdmr_list);
}
```

* 新增全局变量 `tdx_may_have_private_memory` 表面当前 host 可能有 TD 私有内存
```diff
diff --git a/arch/x86/virt/vmx/tdx/tdx.c b/arch/x86/virt/vmx/tdx/tdx.c
index 49a1c6890b55..7f5d388c5461 100644
--- a/arch/x86/virt/vmx/tdx/tdx.c
+++ b/arch/x86/virt/vmx/tdx/tdx.c
@@ -52,6 +52,8 @@ static DEFINE_MUTEX(tdx_module_lock);
 /* All TDX-usable memory regions.  Protected by mem_hotplug_lock. */
 static LIST_HEAD(tdx_memlist);
 
+static bool tdx_may_have_private_memory __read_mostly;
+
 typedef void (*sc_err_func_t)(u64 fn, u64 err, struct tdx_module_args *args);
 
 static inline void seamcall_err(u64 fn, u64 err, struct tdx_module_args *args)
```
* 新增函数 `mark_may_have_private_memory()` 将全局变量 `tdx_may_have_private_memory` 设为指定的值
* 加入写屏障
  * 确保所有 cpu 都可以看到 `tdx_may_have_private_memory` 的更新。
  * 这确保了当任何远程 cpu 将其读取为 `true` 时，“`tdx_tdmr_list`”对于读取 PAMT 必须稳定。
```cpp
static void mark_may_have_private_memory(bool may)
{
    tdx_may_have_private_memory = may;

    /*
     * Ensure update to tdx_may_have_private_memory is visible to all
     * cpus.  This ensures when any remote cpu reads it as true, the
     * 'tdx_tdmr_list' must be stable for reading PAMTs.
     */
    smp_wmb();
}
```
* 在初始化 TDX module 的时候，初始化或释放 `tdx_tdmr_list` 之前设置全局变量 `tdx_may_have_private_memory`
```diff
@@ -1141,6 +1155,12 @@ static int init_tdx_module(void)
 	if (ret)
 		goto err_reset_pamts;
 
+	/*
+	 * Starting from this point the system is possible to have
+	 * TDX private memory.
+	 */
+	mark_may_have_private_memory(true);
+
 	/* Initialize TDMRs to complete the TDX module initialization */
 	ret = init_tdmrs(&tdx_tdmr_list);
 	if (ret)
@@ -1172,6 +1192,7 @@ static int init_tdx_module(void)
 	 * as suggested by the TDX spec.
 	 */
 	tdmrs_reset_pamt_all(&tdx_tdmr_list);
+	mark_may_have_private_memory(false);
 err_free_pamts:
 	tdmrs_free_pamt_all(&tdx_tdmr_list);
 err_free_tdmrs:
@@ -1489,3 +1510,61 @@ void __init tdx_init(void)
 
 	check_tdx_erratum();
 }
```

## [PATCH v3 4/5] x86/virt/tdx: Remove the !KEXEC_CORE dependency
* 现在 TDX host 可以使用 `kexec()`。删除 `!KEXEC_CORE` 依赖项。
---
```diff
diff --git a/arch/x86/Kconfig b/arch/x86/Kconfig
index c62db6b853d7..bfafc8a16a07 100644
--- a/arch/x86/Kconfig
+++ b/arch/x86/Kconfig
@@ -1967,7 +1967,6 @@ config INTEL_TDX_HOST
 	depends on X86_X2APIC
 	select ARCH_KEEP_MEMBLOCK
 	depends on CONTIG_ALLOC
-	depends on !KEXEC_CORE
 	depends on X86_MCE
 	help
 	  Intel Trust Domain Extensions (TDX) protects guest VMs from malicious
```

## [PATCH v3 5/5] x86/virt/tdx: Add TDX memory reset notifier to reset other private pages
### TL;DR:
* 为了涵盖正常 kexec 和 crash kexec，添加 TDX 特定内存重置 notifier，以让“in-kernel TDX users”使用自己的方式在 `tdx_reset_memory()` 中转换 TDX 私有页面（他们分别管理）。

### 长版本：
* 在具有 TDX“partial write machine check” erratum 的平台上，在 kexec 期间，内核需要在跳转到第二个内核之前将 TDX 私有内存转换回正常状态，以避免第二个内核看到潜在的 machine check。
* 目前 `tdx_reset_memory()` 仅重置 PAMT 页。KVM 将是第一个支持运行 TDX guest 的内核 TDX 用户，届时其他 TDX 私有页面将开始存在。他们也需要被覆盖。
* 目前内核没有统一的方法来判断给定页面是否是 TDX 私有页面。一种选择是添加这种统一的方式，有几种选择可以做到这一点：
  1. 使用位图或 Xarray 等来跟踪所有 `PFN` 的 TDX 私有页面；
  2. 在直接映射 PTE 中使用“software-only” bit 来标记给定页面是 TDX 私有页面；
  3. 在'`struct page`'中使用新标志来标记 TDX 私有页面；
  4. ...潜在的其他方式。
* 以上都不是理想的。
  * 选项 1) 消耗额外的内存。例如，如果使用位图，则开销为 `“总 RAM 页数/8”字节`。
  * 当一页被分配为 TDX 私有页面时，选项 2) 会导致在直接映射中将大页面映射拆分为 4K 映射，并导致额外的 TLB 刷新等。对于此类用例来说，这并不理想。
  * 选项 3) 显然与减少“`struct page`”标志的使用的努力相矛盾。
* 因此，不是提供统一的方法来判断给定页面是否是 TDX 私有页面，取而代之的是将“重置 TDX 私有页面”留给 TDX 的“in-kernel user”。
* 这是因为 KVM 已经在维护一个 Xarray 来跟踪每个 guest 的每个 GFN 的“内存属性 memory attributes（例如，私有或共享）”。因此 KVM 可以使用自己的方式找到它管理的所有 TDX 私有页面并将它们转换回正常状态。
* 对于普通的 kexec，可以使用 reboot notifier，但它不包括 crash kexec。
* 添加 TDX 特定内存 reset notifier 以实现此目的。
  * In-kernel TDX users 将需要注册自己的 notifier 来重置 TDX 私有页面。
  * 在重置 PAMT 页之前，在 `tdx_reset_memory()` 中调用这些 notifier。
* KVM 将是该 notifier 的第一个用户。导出“register”和“unregister” API 供 KVM 使用。
---
* 定义全局的 TDX 内存重置 notifier 通知链的头指针 `tdx_memory_reset_chain` 以及“register”和“unregister” API
```cpp
static BLOCKING_NOTIFIER_HEAD(tdx_memory_reset_chain);

int tdx_register_memory_reset_notifier(struct notifier_block *nb)
{
    return blocking_notifier_chain_register(&tdx_memory_reset_chain, nb);
}
EXPORT_SYMBOL_GPL(tdx_register_memory_reset_notifier);

void tdx_unregister_memory_reset_notifier(struct notifier_block *nb)
{
    blocking_notifier_chain_unregister(&tdx_memory_reset_chain, nb);
}
EXPORT_SYMBOL_GPL(tdx_unregister_memory_reset_notifier);
```
* 定义入口函数 `notify_reset_memory` 给监听在 `tdx_memory_reset_chain` 链上的监听者发出通知，调用注册时用到回调函数
```cpp
static int notify_reset_memory(void)
{
    int ret;

    ret = blocking_notifier_call_chain(&tdx_memory_reset_chain, 0, NULL);

    return notifier_to_errno(ret);
}
```
* 在重置 TDX 内存的时候 `tdx_reset_memory()`，调用 `notify_reset_memory()` 发出通知，告诉所有的 in-kernel TDX users 去重置他们所管理的 TDX 私有页面
```diff
@@ -1553,18 +1577,15 @@ void tdx_reset_memory(void)
 	native_wbinvd();
 
 	/*
-	 * It's ideal to cover all types of TDX private pages here, but
-	 * currently there's no unified way to tell whether a given page
-	 * is TDX private page or not.
-	 *
-	 * Just convert PAMT pages now, as currently TDX private pages
-	 * can only be PAMT pages.
-	 *
-	 * TODO:
-	 *
-	 * This leaves all other types of TDX private pages undealt
-	 * with.  They must be handled in _some_ way when they become
-	 * possible to exist.
+	 * Tell all in-kernel TDX users to reset TDX private pages
+	 * that they manage.
+	 */
+	if (notify_reset_memory())
+		pr_err("Failed to reset all TDX private pages.\n");
+
+	/*
+	 * The only remaining TDX private pages are PAMT pages.
+	 * Reset them.
 	 */
 	tdmrs_reset_pamt_all(&tdx_tdmr_list);
 }
```