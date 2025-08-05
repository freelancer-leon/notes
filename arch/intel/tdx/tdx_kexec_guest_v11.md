# [PATCHv11 00/19] x86/tdx: Add kexec support

* 该 patchset 添加了一些片段（bits and pieces），以使 kexec（和 crashkernel）在 TDX guest 上工作。
* 最后一个补丁根据批准的 ACPI 规范变更提议[1] 实现了 CPU offlining。
  * 它解锁了 kexec 在目标内核中使用可见的所有 CPU。
  * 它需要 BIOS 侧启用。如果缺少它，我们将回退到使用单 CPU 启动第二个内核。
* [1] https://lore.kernel.org/all/13356251.uLZWGnKmhe@kreacher
* 译注：关于 Multiprocessor Wakeup Structure 可以阅读 [TDX Guest 解析](tdx_guest.md) 中的 “多处理器启动” 章节。
  * 私有内存转换成共享内存时增加一个页面计数，当发生 kexec 时，需要把共享内存转回为私有的，避免在新系统中当作私有内存来访问
  * 确保存放未接受内存的 ACPI Nv 内存在 crash kernel 启动期间不被 zap 掉
  * CPU offline 的时候 AP 进入假死状态用的不是 `hlt` 指令而是构造切换环境
    * 给 E820 分配的内存、假死例程、`ResetVector` 在恒等映射页表中建立映射
    * 设置页表和中断 `RFLAGS` 等
    * 被 offline 的 CPU 跳转到 mailbox wakeup 结构里 **固件提供的 `ResetVector`** 进入 TDVF，等待唤醒
    * 发起 offline 的 CPU 发送 `TEST` mailbox 命令测试被 offline 的 CPU 是否 offlined，TDVF 将 mailbox 的 `Command` 域改为 `Noop(0)` 表示已接管

## [RFC] ACPI Code First ECR: Support for resetting the Multiprocessor Wakeup Mailbox

* MADT 中的 Multiprocessor Wakeup Structure 用于在没有适合此目的的硬件机制的系统上启动 application processors（AP）。
  * 它描述了一个 mailbox（Multiprocessor Wakeup Mailbox），允许 boot CPU 向 AP 发送命令，特别是使它们跳转到操作系统提供的唤醒向量。
* 根据当前定义，在使用 Multiprocessor Wakeup Mailbox 使给定 AP 跳转到唤醒向量后，它不能再与该 AP 一起使用，这在某些用例中会出现问题，例如 `kexec()` 或休眠（hibernation），因为它会使 *AP 的控制* 从一个操作系统实例传输到另一个操作系统实例很困难。
* 为此，更改了 Multiprocessor Wakeup Structure 的定义，以便允许重置 Multiprocessor Wakeup Mailbox，目的是为了再次与给定的 AP 一起使用。

### 题目: 增加对 Multiprocessor Wakeup Mailbox 的支持

### 变更摘要

* 将 `ResetVector` 字段添加到 Multiprocessor Wakeup Structure 中，以便为操作系统提供 Multiprocessor Wakeup Mailbox 入口点的物理地址，供 AP 在使用过一次 mailbox 以后，需要再次使用 mailbox。

### 变更的好处

* 它将允许启动新的操作系统实例，而无需执行完整的系统重置，例如在执行操作系统更新后。
* 在某些有效的用例中，以这种方式启动的新操作系统与目前为止运行的操作系统不同，并且在缺乏用于重置 AP 的硬件机制的情况下，将 AP 的控制权从一个操作系统实例传递到另一个操作系统实例的唯一方法是：通过平台固件。

### 变更的影响

* 操作系统和平台固件需要识别 Multiprocessor Wakeup Mailbox 的 version 1。

### 变更的详细说明

* 实施此提案所需的 ACPI 规范源代码更改如下：

> ~~对于每个 AP，mailbox 只能用于唤醒命令一次。AP 根据命令采取动作后，该 mailbox 将不再被该 AP 检查。其他处理器可以继续使用 mailbox 来处理下一个命令~~

* 对于每个 AP，mailbox 只能用于唤醒命令一次，除非 `MailBoxVersion` 字段值大于 `0` 并且 `ResetVector` 字段包含非零值。
* 当 AP 根据命令采取行动后，该 AP 将不再检查该 mailbox，直到如下所述为其重置 mailbox。其他处理器可以继续使用该 mailbox 来执行下一个命令。
* 如果需要再次使用 mailbox，例如为了启动新版本的操作系统而不执行完整的系统重置，则 `ResetVector` 字段值可用于使给定的 AP 进入一个检查 mailbox 的状态。为此，操作系统需要根据 `ResetVector` 字段描述为相关 AP 设置 mailbox 重置环境，并使该 AP 跳转到从 `ResetVector` 字段检索到的由固件提供的 mailbox 重置地址。
* 这需要对每个 AP 单独完成，并且对一个 AP 执行此操作不会影响其他 AP，因此它们可以继续不受干扰地运行。但是，如果 `ResetVector` 字段值为 `0`，则 mailbox 无法重置，因此只能使用一次。
* AP 跳转到复位地址后，操作系统需要通过向 mailbox 发送 `test` 命令来验证 mailbox 是否响应命令。当它通过将命令更改为 `noop` 进行响应时，操作系统就不再需要为给定的 AP 维护 mailbox 重置环境了。

##### Table 5.43: Multiprocessor Wakeup Structure

域             | 字节长度 | 字节偏移 | 描述
---------------|---------|---------|---------------------------------------
Type           | 1       | 0       | `0x10` Multiprocessor Wakeup structure
Length         | 1       | 1       | ~~16~~ 24
MailBoxVersion | 2       | 2       | mailbox 的版本。~~ACPI spec v6.5 版本为 0~~ ACPI spec v6.6 版本为 1
保留           | 4       | 4       | 必须为 0
MailBoxAddress | 8       | 8       | mailbox 的物理地址，必须在 ACPINvs，并且 4KB 对齐
ResetVector    | 8       | 16      | AP 的 mailbox 重置向量地址。对于 Intel 处理器，mailbox 重置环境为：</br> * 必须禁用中断。</br> * `RFLAGES.IF` 设置为 `0`。</br> * 长模式已启用。</br> * 启用分页模式，并且复位向量的物理内存是恒等映射的（虚拟地址等于物理地址）。</br> * 复位向量必须包含在一个物理页内。</br> * 段选择符设置为平坦，否则不使用。

##### Table 5.44: Multiprocessor Wakeup Mailbox Structure

域             | 字节长度 | 字节偏移 | 描述
---------------|---------|---------|---------------------------------------
Command        | 2       | 0       | `0: Noop` - 无操作</br> `1: Wakeup` – 跳转到唤醒向量</br> `2：Test` - 通过将命令更改为 `Noop` 来响应</br> `3-0xFFFF`: 保留
保留           | 2       | 2       | 必须为 0
ApicId         | 4       | 4       | 处理器的 local APIC ID。AP 应检查 *ApicId* 字段是否与自己的 APIC ID 匹配。在 APIC ID 不匹配的情况下，AP 应忽略该命令
WakeupVector   | 8       | 8       | AP 的唤醒地址。对于 Intel 处理器，执行环境是：</br> * 必须禁用中断</br> * `RFLAGES.IF` 设置为 `0` </br> * 启用长模式</br> * 启用分页模式，唤醒向量的物理内存是恒等映射的（虚拟地址等于物理地址）</br> * 唤醒向量必须包含在一个物理页面中</br> * 段选择符设置为平坦映射，否则不使用
ReservedForOs | 2032     | 16      | 保留给 OS 使用
ReservedForFirmware | 2048 | 2048  | 保留给固件使用

* 译注：
  * `WakeupVector` 是由 OS 提供给固件，固件令 AP 跳转到该地址完成唤醒
    * 对 OS 而言，该域是可写的，因此放在 Multiprocessor Wakeup Mailbox Structure
  * `ResetVector` 是由固件提供给 OS，OS 令 AP 跳转到该地址进入固件复位，等待被唤醒
    * 对 OS 而言，该域是只读的，因此放在 Multiprocessor Wakeup Structure

## [PATCHv11 01/19] x86/acpi: Extract ACPI MADT wakeup code into a separate file
* 为了准备扩展对 ACPI MADT 唤醒方法的支持，将相关代码移至单独的文件中。
* 引入一个新的配置选项，无需使用 `ifdef` 即可清楚地指示依赖关系。
* 没有任何功能变化。
---
* 新增 Kconfig `X86_ACPI_MADT_WAKEUP`，缺省为 `y`
```diff
diff --git a/arch/x86/Kconfig b/arch/x86/Kconfig
index e8837116704c..e30ea4129d2c 100644
--- a/arch/x86/Kconfig
+++ b/arch/x86/Kconfig
@@ -1118,6 +1118,13 @@ config X86_LOCAL_APIC
    depends on X86_64 || SMP || X86_32_NON_STANDARD || X86_UP_APIC || PCI_MSI
    select IRQ_DOMAIN_HIERARCHY

+config ACPI_MADT_WAKEUP
+   def_bool y
+   depends on X86_64
+   depends on ACPI
+   depends on SMP
+   depends on X86_LOCAL_APIC
+
 config X86_IO_APIC
    def_bool y
    depends on X86_LOCAL_APIC || X86_UP_IOAPIC
```
* 增加对 arch/x86/kernel/acpi/madt_wakeup.c 文件的编译的支持，需要开启 `CONFIG_ACPI_MADT_WAKEUP` 选项
```diff
diff --git a/arch/x86/kernel/acpi/Makefile b/arch/x86/kernel/acpi/Makefile
index fc17b3f136fe..2feba7257665 100644
--- a/arch/x86/kernel/acpi/Makefile
+++ b/arch/x86/kernel/acpi/Makefile
+obj-$(CONFIG_ACPI_MADT_WAKEUP) += madt_wakeup.o
```

* arch/x86/include/asm/acpi.h 增加对 `acpi_parse_mp_wake()` 的函数声明
```diff
diff --git a/arch/x86/include/asm/acpi.h b/arch/x86/include/asm/acpi.h
index 5af926c050f0..ceacac2b335d 100644
--- a/arch/x86/include/asm/acpi.h
+++ b/arch/x86/include/asm/acpi.h
@@ -78,6 +78,11 @@ static inline bool acpi_skip_set_wakeup_address(void)

 #define acpi_skip_set_wakeup_address acpi_skip_set_wakeup_address

+union acpi_subtable_headers;
+
+int __init acpi_parse_mp_wake(union acpi_subtable_headers *header,
+                 const unsigned long end);
+
 /*
  * Check if the CPU can handle C2 and deeper
  */
```
* 从 arch/x86/kernel/acpi/boot.c 中抽取以下内容到独立的 arch/x86/kernel/acpi/madt_wakeup.c 文件中
  * 变量 `static u64 acpi_mp_wake_mailbox_paddr`
  * 变量 `static struct acpi_madt_multiproc_wakeup_mailbox *acpi_mp_wake_mailbox`
  * 函数 `static int acpi_wakeup_cpu(u32 apicid, unsigned long start_ip)`
  * 函数 `int __init acpi_parse_mp_wake(union acpi_subtable_headers *header, const unsigned long end)`

## [PATCHv11 02/19] x86/apic: Mark acpi_mp_wake_\* variables as \_\_ro_after_init
* `acpi_mp_wake_mailbox_paddr` 和 `acpi_mp_wake_mailbox` 在 ACPI MADT 初始化期间初始化一次，并且从未更改。
---
* `acpi_mp_wake_mailbox_paddr` 来自函数 `acpi_parse_mp_wake()` 对 MADT 表中对 MP 唤醒条目的解析
* `acpi_mp_wake_mailbox` 来自函数 `acpi_wakeup_cpu()` 中对物理地址 `acpi_mp_wake_mailbox_paddr()` 重映射得到的虚拟地址
```diff
diff --git a/arch/x86/kernel/acpi/madt_wakeup.c b/arch/x86/kernel/acpi/madt_wakeup.c
index 7f164d38bd0b..cf79ea6f3007 100644
--- a/arch/x86/kernel/acpi/madt_wakeup.c
+++ b/arch/x86/kernel/acpi/madt_wakeup.c
@@ -6,10 +6,10 @@
 #include <asm/processor.h>

 /* Physical address of the Multiprocessor Wakeup Structure mailbox */
-static u64 acpi_mp_wake_mailbox_paddr;
+static u64 acpi_mp_wake_mailbox_paddr __ro_after_init;

 /* Virtual address of the Multiprocessor Wakeup Structure mailbox */
-static struct acpi_madt_multiproc_wakeup_mailbox *acpi_mp_wake_mailbox;
+static struct acpi_madt_multiproc_wakeup_mailbox *acpi_mp_wake_mailbox __ro_after_init;

 static int acpi_wakeup_cpu(u32 apicid, unsigned long start_ip)
```

## [PATCHv11 03/19] cpu/hotplug: Add support for declaring CPU offlining not supported
* ACPI MADT mailbox 唤醒方法不允许在唤醒后使 CPU offline。
* 目前，基于为 Intel TDX 设置的机密计算属性，以防止 offlining 热插拔。
  * 但 TDX 并不是唤醒方法的唯一可能用户。MADT 唤醒可以在机密计算环境之外实现。
  * Offline 支持是唤醒方法的属性，而不是 CoCo 实现的属性。
* 引入 `cpu_hotplug_disable_offlined()`，可以调用该函数来指示应禁用 CPU offlining。
* 此函数将取代 ACPI MADT 唤醒方法的 `CC_ATTR_HOTPLUG_DISABLED`。
---
* 增加全局变量 `cpu_hotplug_offline_disabled` 表明 CPU offline 热插拔已被禁用
* 新增函数 `cpu_hotplug_disable_offlining()` 声明不支持 CPU offline
  * kernel/cpu.c
```cpp
static bool cpu_hotplug_offline_disabled __ro_after_init;
...
/* Declare CPU offlining not supported */
void cpu_hotplug_disable_offlining(void)
{
    cpu_maps_update_begin();
    cpu_hotplug_offline_disabled = true;
    cpu_maps_update_done();
}
```
* CPU offline 热插拔已被禁用，阻止 CPU down 操作
```diff
diff --git a/kernel/cpu.c b/kernel/cpu.c
index 8f6affd051f7..08860baa6ce0 100644
--- a/kernel/cpu.c
+++ b/kernel/cpu.c
@@ -1518,7 +1528,8 @@ static int cpu_down_maps_locked(unsigned int cpu, enum cpuhp_state target)
     * If the platform does not support hotplug, report it explicitly to
     * differentiate it from a transient offlining failure.
     */
-   if (cc_platform_has(CC_ATTR_HOTPLUG_DISABLED))
+   if (cc_platform_has(CC_ATTR_HOTPLUG_DISABLED) ||
+       cpu_hotplug_offline_disabled)
        return -EOPNOTSUPP;
    if (cpu_hotplug_disabled)
        return -EBUSY;
```
* 从 `cpu_down_maps_locked()` 的调用者可见它用于 CPU offline 的场景
```cpp
1 kernel/cpu.c|1544| <<cpu_down>> err = cpu_down_maps_locked(cpu, target);
2 kernel/cpu.c|1595| <<smp_shutdown_nonboot_cpus>> error = cpu_down_maps_locked(cpu, CPUHP_OFFLINE);
3 kernel/cpu.c|2703| <<cpuhp_smt_disable>> ret = cpu_down_maps_locked(cpu, CPUHP_OFFLINE);
```

## [PATCHv11 04/19] cpu/hotplug, x86/acpi: Disable CPU offlining for ACPI MADT wakeup
* ACPI MADT 不允许 CPU 唤醒后 offline。
* 目前，基于 Intel TDX 设置的机密计算属性，以防止 CPU 热插拔。但 TDX 并不是唤醒方法的唯一可能用户。
* 在 ACPI MADT 唤醒枚举上禁用 CPU offlining。
* 这对用户没有明显的影响：目前，TDX 客户机是唯一使用 ACPI MADT 唤醒方法的平台。
---
* 从 `enum cc_attr` 中删除 `CC_ATTR_HOTPLUG_DISABLED` 属性，这是唤醒方法的属性，而不是 CoCo 实现的属性
```diff
diff --git a/include/linux/cc_platform.h b/include/linux/cc_platform.h
index 60693a145894..caa4b4430634 100644
--- a/include/linux/cc_platform.h
+++ b/include/linux/cc_platform.h
@@ -81,16 +81,6 @@ enum cc_attr {
 	 */
 	CC_ATTR_GUEST_SEV_SNP,
 
-	/**
-	 * @CC_ATTR_HOTPLUG_DISABLED: Hotplug is not supported or disabled.
-	 *
-	 * The platform/OS is running as a guest/virtual machine does not
-	 * support CPU hotplug feature.
-	 *
-	 * Examples include TDX Guest.
-	 */
-	CC_ATTR_HOTPLUG_DISABLED,
-
 	/**
 	 * @CC_ATTR_HOST_SEV_SNP: AMD SNP enabled on the host.
 	 *
```
* 删除相关的引用
```diff
diff --git a/kernel/cpu.c b/kernel/cpu.c
index 08860baa6ce0..a70767aee9d0 100644
--- a/kernel/cpu.c
+++ b/kernel/cpu.c
@@ -1528,8 +1528,7 @@ static int cpu_down_maps_locked(unsigned int cpu, enum cpuhp_state target)
 	 * If the platform does not support hotplug, report it explicitly to
 	 * differentiate it from a transient offlining failure.
 	 */
-	if (cc_platform_has(CC_ATTR_HOTPLUG_DISABLED) ||
-	    cpu_hotplug_offline_disabled)
+	if (cpu_hotplug_offline_disabled)
 		return -EOPNOTSUPP;
 	if (cpu_hotplug_disabled)
 		return -EBUSY;
```
* 目前，基于 Intel TDX 设置的机密计算属性，以防止 CPU 热插拔。但 TDX 并不是唤醒方法的唯一可能用户。
```diff
diff --git a/arch/x86/coco/core.c b/arch/x86/coco/core.c
index b31ef2424d19..0f81f70aca82 100644
--- a/arch/x86/coco/core.c
+++ b/arch/x86/coco/core.c
@@ -29,7 +29,6 @@ static bool noinstr intel_cc_platform_has(enum cc_attr attr)
 {
 	switch (attr) {
 	case CC_ATTR_GUEST_UNROLL_STRING_IO:
-	case CC_ATTR_HOTPLUG_DISABLED:
 	case CC_ATTR_GUEST_MEM_ENCRYPT:
 	case CC_ATTR_MEM_ENCRYPT:
 		return true;
```
* 在 ACPI MADT 唤醒枚举上禁用 CPU offlining。
```diff
diff --git a/arch/x86/kernel/acpi/madt_wakeup.c b/arch/x86/kernel/acpi/madt_wakeup.c
index cf79ea6f3007..d222be8d7a07 100644
--- a/arch/x86/kernel/acpi/madt_wakeup.c
+++ b/arch/x86/kernel/acpi/madt_wakeup.c
@@ -1,5 +1,6 @@
 // SPDX-License-Identifier: GPL-2.0-or-later
 #include <linux/acpi.h>
+#include <linux/cpu.h>
 #include <linux/io.h>
 #include <asm/apic.h>
 #include <asm/barrier.h>
@@ -76,6 +77,8 @@ int __init acpi_parse_mp_wake(union acpi_subtable_headers *header,
 
 	acpi_mp_wake_mailbox_paddr = mp_wake->base_address;
 
+	cpu_hotplug_disable_offlining();
+
 	apic_update_callback(wakeup_secondary_cpu_64, acpi_wakeup_cpu);
 
 	return 0;
```
## [PATCHv11 05/19] x86/relocate_kernel: Use named labels for less confusion
* `identity_mapped()` 函数非常喜欢用 label “`1`”，甚至让读者感到困惑。
* 为了清晰起见，在每个地方使用命名标签。
* 没有功能性的修改
---
```diff
diff --git a/arch/x86/kernel/relocate_kernel_64.S b/arch/x86/kernel/relocate_kernel_64.S
index 56cab1bb25f5..085eef5c3904 100644
--- a/arch/x86/kernel/relocate_kernel_64.S
+++ b/arch/x86/kernel/relocate_kernel_64.S
@@ -148,9 +148,10 @@ SYM_CODE_START_LOCAL_NOALIGN(identity_mapped)
     */
    movl    $X86_CR4_PAE, %eax
    testq   $X86_CR4_LA57, %r13
-   jz  1f
+   jz  .Lno_la57
    orl $X86_CR4_LA57, %eax
-1:
+.Lno_la57:
+
    movq    %rax, %cr4

    jmp 1f
@@ -165,9 +166,9 @@ SYM_CODE_START_LOCAL_NOALIGN(identity_mapped)
     * used by kexec. Flush the caches before copying the kernel.
     */
    testq   %r12, %r12
-   jz 1f
+   jz .Lsme_off
    wbinvd
-1:
+.Lsme_off:

    movq    %rcx, %r11
    call    swap_pages
@@ -187,7 +188,7 @@ SYM_CODE_START_LOCAL_NOALIGN(identity_mapped)
     */

    testq   %r11, %r11
-   jnz 1f
+   jnz .Lrelocate
    xorl    %eax, %eax
    xorl    %ebx, %ebx
    xorl    %ecx, %ecx
@@ -208,7 +209,7 @@ SYM_CODE_START_LOCAL_NOALIGN(identity_mapped)
    ret
    int3

-1:
+.Lrelocate:
    popq    %rdx
    leaq    PAGE_SIZE(%r10), %rsp
    ANNOTATE_RETPOLINE_SAFE
```

## [PATCHv11 06/19] x86/kexec: Keep CR4.MCE set during kexec for TDX guest
* TDX guest 从一开始就启用 MCA（`CR4.MCE=1b`）运行。如果在启动或 kexec 流程期间 `CR4` 寄存器重新编程期间清除该位，则会引发 `#VE` 异常，而 guest 内核无法处理该异常。
* 因此，请确保 `CR4.MCE` 设置也通过 kexec 保留，并避免引发任何 `#VE`。
* 此更改不会影响非 TDX guest 环境。
---
```diff
diff --git a/arch/x86/kernel/relocate_kernel_64.S b/arch/x86/kernel/relocate_kernel_64.S
index 085eef5c3904..b668a6be4f6f 100644
--- a/arch/x86/kernel/relocate_kernel_64.S
+++ b/arch/x86/kernel/relocate_kernel_64.S
@@ -5,6 +5,8 @@
  */

 #include <linux/linkage.h>
+#include <linux/stringify.h>
+#include <asm/alternative.h>
 #include <asm/page_types.h>
 #include <asm/kexec.h>
 #include <asm/processor-flags.h>
@@ -143,15 +145,17 @@ SYM_CODE_START_LOCAL_NOALIGN(identity_mapped)

    /*
     * Set cr4 to a known state:
-    *  - physical address extension enabled
     *  - 5-level paging, if it was enabled before
+    *  - Machine check exception on TDX guest, if it was enabled before.
+    *    Clearing MCE might not be allowed in TDX guests, depending on setup.
+    *  - physical address extension enabled
     */
-   movl    $X86_CR4_PAE, %eax
-   testq   $X86_CR4_LA57, %r13
-   jz  .Lno_la57
-   orl $X86_CR4_LA57, %eax
-.Lno_la57:
+   movl    $X86_CR4_LA57, %eax
+   ALTERNATIVE "", __stringify(orl $X86_CR4_MCE, %eax), X86_FEATURE_TDX_GUEST

+   /* R13 contains the original CR4 value, read in relocate_kernel() */
+   andl    %r13d, %eax
+   orl $X86_CR4_PAE, %eax
    movq    %rax, %cr4

    jmp 1f
```
* 这段汇编在构建 `identity_mapped` 期间，将 `CR4` 设置为已知的状态
* `%r13` 存的是旧 `%cr4` 的值，`%rax` 用于构建将要写入 `%cr4` 的值
* 如果之前五级分页 `$X86_CR4_LA57` 是开启的，现在也要开启
  * 因此将 `$X86_CR4_LA57` 传入 `%eax` 作为掩码位
* 如果之前在 TDX guest 中 *Machine check 异常* 是开启的，现在也要开启
  * 在 TDX guest 中清除 MCE 可能是不允许的，取决于设置
  * 因此用 `ALTERNATIVE` 有条件地加入到 `%eax` 作为掩码位
    * 内核启动时，如果检测到 `X86_FEATURE_TDX_GUEST` 特性开启，将这里的空 `""`（对于非 TDX guest，包括 host）替换为 `orl $X86_CR4_MCE, %eax` 指令
* `andl	%r13d, %eax` 将之前保存的旧 `%cr4` 与设置好的掩码 `%eax` 进行 *与操作*，结果保留在 `%eax`
* 开启物理地址扩展 `$X86_CR4_PAE`
  * 因此用 `orl	$X86_CR4_PAE, %eax` 的方式在 `%rax` 中 *无条件* 打开
* 最后用 `movq	%rax, %cr4` 设置已知状态的 `CR4` 寄存器
* 新逻辑利用 *掩码 `%eax`* + `andl %r13d, %eax` 操作，取代了原来的测试 `test` + `orl $impl, %eax` 操作，节省了几条指令的开销，来自 Sean Christopherson 的建议：
> The `TEST+Jcc+OR` sequences are rather odd, and require way more instructions and thus way more copy+paste than is necessary.
>
```cpp
 movl $X86_CR4_LA57, %eax
 ALTERNATIVE "", __stringify(orl $X86_CR4_MCE, %eax), X86_FEATURE_TDX_GUEST
 andl	%r13d, %eax
 orl	$X86_CR4_PAE, %eax
 movq	%rax, %cr4
```
>
> Then preserving new bits unconditionally only requires adding the flag to the initial move, and feature-dependent bits only need a single ALTERNATIVE line.
> And there's no branches, blazing fast kexec! ;-)

## [PATCHv11 07/19] x86/mm: Make x86_platform.guest.enc_status_change_*() return errno
* TDX 将有多个原因导致 `enc_status_change_prepare()` 失败。
* 将回调更改为返回 `errno`，而不是假设 `-EIO`；
* `enc_status_change_finish()` 也发生了变化以保持接口对称。
---
* 修改 `enc_status_change_prepare()` 和 `enc_status_change_finish()`
  * 原来返回 `true` 表示转换成功，`false` 表示转换失败
  * 现在返回 `0` 表示转换成功，返回 `errno`（`<0`）表示转换失败
```diff
diff --git a/arch/x86/coco/tdx/tdx.c b/arch/x86/coco/tdx/tdx.c
index c1cb90369915..26fa47db5782 100644
--- a/arch/x86/coco/tdx/tdx.c
+++ b/arch/x86/coco/tdx/tdx.c
@@ -798,28 +798,30 @@ static bool tdx_enc_status_changed(unsigned long vaddr, int numpages, bool enc)
    return true;
 }

-static bool tdx_enc_status_change_prepare(unsigned long vaddr, int numpages,
-                     bool enc)
+static int tdx_enc_status_change_prepare(unsigned long vaddr, int numpages,
+                    bool enc)
 {
    /*
     * Only handle shared->private conversion here.
     * See the comment in tdx_early_init().
     */
-   if (enc)
-       return tdx_enc_status_changed(vaddr, numpages, enc);
-   return true;
+   if (enc && !tdx_enc_status_changed(vaddr, numpages, enc))
+       return -EIO;
+
+   return 0;
 }

-static bool tdx_enc_status_change_finish(unsigned long vaddr, int numpages,
+static int tdx_enc_status_change_finish(unsigned long vaddr, int numpages,
                     bool enc)
 {
    /*
     * Only handle private->shared conversion here.
     * See the comment in tdx_early_init().
     */
-   if (!enc)
-       return tdx_enc_status_changed(vaddr, numpages, enc);
-   return true;
+   if (!enc && !tdx_enc_status_changed(vaddr, numpages, enc))
+       return -EIO;
+
+   return 0;
 }
```
* 相关的声明需要被修改
```diff
diff --git a/arch/x86/include/asm/x86_init.h b/arch/x86/include/asm/x86_init.h
index 6149eabe200f..28ac3cb9b987 100644
--- a/arch/x86/include/asm/x86_init.h
+++ b/arch/x86/include/asm/x86_init.h
@@ -151,8 +151,8 @@ struct x86_init_acpi {
  * @enc_cache_flush_required   Returns true if a cache flush is needed before changing page encryption status
  */
 struct x86_guest {
-   bool (*enc_status_change_prepare)(unsigned long vaddr, int npages, bool enc);
-   bool (*enc_status_change_finish)(unsigned long vaddr, int npages, bool enc);
+   int (*enc_status_change_prepare)(unsigned long vaddr, int npages, bool enc);
+   int (*enc_status_change_finish)(unsigned long vaddr, int npages, bool enc);
    bool (*enc_tlb_flush_required)(bool enc);
    bool (*enc_cache_flush_required)(void);
 };
diff --git a/arch/x86/kernel/x86_init.c b/arch/x86/kernel/x86_init.c
index d5dc5a92635a..a7143bb7dd93 100644
--- a/arch/x86/kernel/x86_init.c
+++ b/arch/x86/kernel/x86_init.c
@@ -134,8 +134,8 @@ struct x86_cpuinit_ops x86_cpuinit = {

 static void default_nmi_init(void) { };

-static bool enc_status_change_prepare_noop(unsigned long vaddr, int npages, bool enc) { return true; }
-static bool enc_status_change_finish_noop(unsigned long vaddr, int npages, bool enc) { return true; }
+static int enc_status_change_prepare_noop(unsigned long vaddr, int npages, bool enc) { return 0; }
+static int enc_status_change_finish_noop(unsigned long vaddr, int npages, bool enc) { return 0; }
 static bool enc_tlb_flush_required_noop(bool enc) { return false; }
 static bool enc_cache_flush_required_noop(void) { return false; }
 static bool is_private_mmio_noop(u64 addr) {return false; }
```
* 相关的引用也需要被修改
```diff
diff --git a/arch/x86/mm/pat/set_memory.c b/arch/x86/mm/pat/set_memory.c
index 19fdfbb171ed..498812f067cd 100644
--- a/arch/x86/mm/pat/set_memory.c
+++ b/arch/x86/mm/pat/set_memory.c
@@ -2196,7 +2196,8 @@ static int __set_memory_enc_pgtable(unsigned long addr, int numpages, bool enc)
        cpa_flush(&cpa, x86_platform.guest.enc_cache_flush_required());

    /* Notify hypervisor that we are about to set/clr encryption attribute. */
-   if (!x86_platform.guest.enc_status_change_prepare(addr, numpages, enc))
+   ret = x86_platform.guest.enc_status_change_prepare(addr, numpages, enc);
+   if (ret)
        goto vmm_fail;

    ret = __change_page_attr_set_clr(&cpa, 1);
@@ -2214,16 +2215,17 @@ static int __set_memory_enc_pgtable(unsigned long addr, int numpages, bool enc)
        return ret;

    /* Notify hypervisor that we have successfully set/clr encryption attribute. */
-   if (!x86_platform.guest.enc_status_change_finish(addr, numpages, enc))
+   ret = x86_platform.guest.enc_status_change_finish(addr, numpages, enc);
+   if (ret)
        goto vmm_fail;

    return 0;

 vmm_fail:
-   WARN_ONCE(1, "CPA VMM failure to convert memory (addr=%p, numpages=%d) to %s.\n",
-         (void *)addr, numpages, enc ? "private" : "shared");
+   WARN_ONCE(1, "CPA VMM failure to convert memory (addr=%p, numpages=%d) to %s: %d\n",
+         (void *)addr, numpages, enc ? "private" : "shared", ret);

-   return -EIO;
+   return ret;
 }
```
* AMD 的代码也保持一致的风格
```diff
diff --git a/arch/x86/mm/mem_encrypt_amd.c b/arch/x86/mm/mem_encrypt_amd.c
index 422602f6039b..e7b67519ddb5 100644
--- a/arch/x86/mm/mem_encrypt_amd.c
+++ b/arch/x86/mm/mem_encrypt_amd.c
@@ -283,7 +283,7 @@ static void enc_dec_hypercall(unsigned long vaddr, unsigned long size, bool enc)
 #endif
 }

-static bool amd_enc_status_change_prepare(unsigned long vaddr, int npages, bool enc)
+static int amd_enc_status_change_prepare(unsigned long vaddr, int npages, bool enc)
 {
    /*
     * To maintain the security guarantees of SEV-SNP guests, make sure
@@ -292,11 +292,11 @@ static bool amd_enc_status_change_prepare(unsigned long vaddr, int npages, bool
    if (cc_platform_has(CC_ATTR_GUEST_SEV_SNP) && !enc)
        snp_set_memory_shared(vaddr, npages);

-   return true;
+   return 0;
 }

 /* Return true unconditionally: return value doesn't matter for the SEV side */
-static bool amd_enc_status_change_finish(unsigned long vaddr, int npages, bool enc)
+static int amd_enc_status_change_finish(unsigned long vaddr, int npages, bool enc)
 {
    /*
     * After memory is mapped encrypted in the page table, validate it
@@ -308,7 +308,7 @@ static bool amd_enc_status_change_finish(unsigned long vaddr, int npages, bool e
    if (!cc_platform_has(CC_ATTR_HOST_MEM_ENCRYPT))
        enc_dec_hypercall(vaddr, npages << PAGE_SHIFT, enc);

-   return true;
+   return 0;
 }
```
* HyperV 的代码也保持一致的风格
```diff
diff --git a/arch/x86/hyperv/ivm.c b/arch/x86/hyperv/ivm.c
index 768d73de0d09..b4a851d27c7c 100644
--- a/arch/x86/hyperv/ivm.c
+++ b/arch/x86/hyperv/ivm.c
@@ -523,9 +523,9 @@ static int hv_mark_gpa_visibility(u16 count, const u64 pfn[],
  * transition is complete, hv_vtom_set_host_visibility() marks the pages
  * as "present" again.
  */
-static bool hv_vtom_clear_present(unsigned long kbuffer, int pagecount, bool enc)
+static int hv_vtom_clear_present(unsigned long kbuffer, int pagecount, bool enc)
 {
-   return !set_memory_np(kbuffer, pagecount);
+   return set_memory_np(kbuffer, pagecount);
 }

 /*
@@ -536,20 +536,19 @@ static bool hv_vtom_clear_present(unsigned long kbuffer, int pagecount, bool enc
  * with host. This function works as wrap of hv_mark_gpa_visibility()
  * with memory base and size.
  */
-static bool hv_vtom_set_host_visibility(unsigned long kbuffer, int pagecount, bool enc)
+static int hv_vtom_set_host_visibility(unsigned long kbuffer, int pagecount, bool enc)
 {
    enum hv_mem_host_visibility visibility = enc ?
            VMBUS_PAGE_NOT_VISIBLE : VMBUS_PAGE_VISIBLE_READ_WRITE;
    u64 *pfn_array;
    phys_addr_t paddr;
+   int i, pfn, err;
    void *vaddr;
    int ret = 0;
-   bool result = true;
-   int i, pfn;

    pfn_array = kmalloc(HV_HYP_PAGE_SIZE, GFP_KERNEL);
    if (!pfn_array) {
-       result = false;
+       ret = -ENOMEM;
        goto err_set_memory_p;
    }

@@ -568,10 +567,8 @@ static bool hv_vtom_set_host_visibility(unsigned long kbuffer, int pagecount, bo
        if (pfn == HV_MAX_MODIFY_GPA_REP_COUNT || i == pagecount - 1) {
            ret = hv_mark_gpa_visibility(pfn, pfn_array,
                             visibility);
-           if (ret) {
-               result = false;
+           if (ret)
                goto err_free_pfn_array;
-           }
            pfn = 0;
        }
    }
@@ -586,10 +583,11 @@ static bool hv_vtom_set_host_visibility(unsigned long kbuffer, int pagecount, bo
     * order to avoid leaving the memory range in a "broken" state. Setting
     * the PRESENT bits shouldn't fail, but return an error if it does.
     */
-   if (set_memory_p(kbuffer, pagecount))
-       result = false;
+   err = set_memory_p(kbuffer, pagecount);
+   if (err && !ret)
+       ret = err;

-   return result;
+   return ret;
 }
```

 ## [PATCHv11 08/19] x86/mm: Return correct level from lookup_address() if pte is none
* 目前，`lookup_address()` 返回两样东西：
1. 一条 “`pte_t`”（可能是 `p[g4um]d_t`）
2. 找到“`pte_t`”的页表的“`level`”（通过指针返回）
* 如果没有找到 `pte_t`，“`level`”本质上就是垃圾。
* 始终填写 `level`。对于 `NULL` “`pte_t`”，填写找到的 `p*d_none()` 条目的 `level`，镜像“找到了”的行为。
* 始终填写 `level` 允许使用 `lookup_address()` 在遍历内核页表时精确地跳过空洞。
* 在 `enum pg_level` 中再添加一项，以指示 5 级分页模式下一个 `PGD` 项所覆盖的 VA 的大小。
* 更新 `lookup_address()` 和 `lookup_address_in_pgd()` 的注释以反映接口中的更改。
---
* 5 级分页模式下一个 `PGD` 项所覆盖的 VA 的大小为 `256 TB`，`512` 个这样的 `PGD` 项能覆盖 `128 PB`
```diff
diff --git a/arch/x86/include/asm/pgtable_types.h b/arch/x86/include/asm/pgtable_types.h
index b78644962626..2f321137736c 100644
--- a/arch/x86/include/asm/pgtable_types.h
+++ b/arch/x86/include/asm/pgtable_types.h
@@ -549,6 +549,7 @@ enum pg_level {
    PG_LEVEL_2M,
    PG_LEVEL_1G,
    PG_LEVEL_512G,
+   PG_LEVEL_256T,
    PG_LEVEL_NUM
 };
```
* 修改过后即便返回 `NULL`，没有填充条目，也能通过 `level` 反映是在哪一级没填充的
```diff
diff --git a/arch/x86/mm/pat/set_memory.c b/arch/x86/mm/pat/set_memory.c
index 498812f067cd..a7a7a6c6a3fb 100644
--- a/arch/x86/mm/pat/set_memory.c
+++ b/arch/x86/mm/pat/set_memory.c
@@ -662,8 +662,9 @@ static inline pgprot_t verify_rwx(pgprot_t old, pgprot_t new, unsigned long star

 /*
  * Lookup the page table entry for a virtual address in a specific pgd.
- * Return a pointer to the entry, the level of the mapping, and the effective
- * NX and RW bits of all page table levels.
+ * Return a pointer to the entry (or NULL if the entry does not exist),
+ * the level of the entry, and the effective NX and RW bits of all
+ * page table levels.
  */
 pte_t *lookup_address_in_pgd_attr(pgd_t *pgd, unsigned long address,
                  unsigned int *level, bool *nx, bool *rw)
@@ -672,13 +673,14 @@ pte_t *lookup_address_in_pgd_attr(pgd_t *pgd, unsigned long address,
    pud_t *pud;
    pmd_t *pmd;

-   *level = PG_LEVEL_NONE;
+   *level = PG_LEVEL_256T;
    *nx = false;
    *rw = true;

    if (pgd_none(*pgd))
        return NULL;

+   *level = PG_LEVEL_512G;
    *nx |= pgd_flags(*pgd) & _PAGE_NX;
    *rw &= pgd_flags(*pgd) & _PAGE_RW;

@@ -686,10 +688,10 @@ pte_t *lookup_address_in_pgd_attr(pgd_t *pgd, unsigned long address,
    if (p4d_none(*p4d))
        return NULL;

-   *level = PG_LEVEL_512G;
    if (p4d_leaf(*p4d) || !p4d_present(*p4d))
        return (pte_t *)p4d;

+   *level = PG_LEVEL_1G;
    *nx |= p4d_flags(*p4d) & _PAGE_NX;
    *rw &= p4d_flags(*p4d) & _PAGE_RW;

@@ -697,10 +699,10 @@ pte_t *lookup_address_in_pgd_attr(pgd_t *pgd, unsigned long address,
    if (pud_none(*pud))
        return NULL;

-   *level = PG_LEVEL_1G;
    if (pud_leaf(*pud) || !pud_present(*pud))
        return (pte_t *)pud;

+   *level = PG_LEVEL_2M;
    *nx |= pud_flags(*pud) & _PAGE_NX;
    *rw &= pud_flags(*pud) & _PAGE_RW;

@@ -708,15 +710,13 @@ pte_t *lookup_address_in_pgd_attr(pgd_t *pgd, unsigned long address,
    if (pmd_none(*pmd))
        return NULL;

-   *level = PG_LEVEL_2M;
    if (pmd_leaf(*pmd) || !pmd_present(*pmd))
        return (pte_t *)pmd;

+   *level = PG_LEVEL_4K;
    *nx |= pmd_flags(*pmd) & _PAGE_NX;
    *rw &= pmd_flags(*pmd) & _PAGE_RW;

-   *level = PG_LEVEL_4K;
-
    return pte_offset_kernel(pmd, address);
 }

@@ -736,9 +736,8 @@ pte_t *lookup_address_in_pgd(pgd_t *pgd, unsigned long address,
  * Lookup the page table entry for a virtual address. Return a pointer
  * to the entry and the level of the mapping.
  *
- * Note: We return pud and pmd either when the entry is marked large
- * or when the present bit is not set. Otherwise we would return a
- * pointer to a nonexisting mapping.
+ * Note: the function returns p4d, pud or pmd either when the entry is marked
+ * large or when the present bit is not set. Otherwise it returns NULL.
  */
 pte_t *lookup_address(unsigned long address, unsigned int *level)
 {
```
* `p*d_leaf()` 返回 `true` 表示 `p*d` 是个大页或巨页，所以不往下走了
* 如果 `p*d_leaf()` 返回 `false` 表示 `p*d` **不是** 大页或巨页，进一步判断 `_PAGE_PRESENT` bit 是否置位
  * 如果置位，下一级的页表也填充了，继续往下走
  * 如果没置位，还未填充下一级页表，返回当前级别的页表条目的指针

## [PATCHv11 09/19] x86/tdx: Account shared memory
* 内核将在 kexec 期间将所有共享内存转换回私有内存。直接映射页表将提供有关共享哪些内存的信息。
* 转换所有共享内存极其重要。如果某个页面丢失，将会导致第二个内核在访问该页面时崩溃。
* 跟踪共享页面的数量。如果 shared bit 丢失，这将允许在直接映射和报告中对共享信息进行交叉检查。
---
* 增加统计共享页面的全局变量 `nr_shared`
* 在转换成功后，即 `tdx_enc_status_change_finish()` 函数返回成功前，增加/减少共享页面的计数
```diff
diff --git a/arch/x86/coco/tdx/tdx.c b/arch/x86/coco/tdx/tdx.c
index 26fa47db5782..979891e97d83 100644
--- a/arch/x86/coco/tdx/tdx.c
+++ b/arch/x86/coco/tdx/tdx.c
@@ -38,6 +38,8 @@

 #define TDREPORT_SUBTYPE_0 0

+static atomic_long_t nr_shared;
+
 /* Called from __tdx_hypercall() for unrecoverable failure */
 noinstr void __noreturn __tdx_hypercall_failed(void)
 {
@@ -821,6 +823,11 @@ static int tdx_enc_status_change_finish(unsigned long vaddr, int numpages,
    if (!enc && !tdx_enc_status_changed(vaddr, numpages, enc))
        return -EIO;

+   if (enc)
+       atomic_long_sub(numpages, &nr_shared);
+   else
+       atomic_long_add(numpages, &nr_shared);
+
    return 0;
 }
```
* Dave Hansen 建议追加说明：

> 也许还值得一提的是，转换速度很慢并且相对罕见，尽管用一个全局的 `atomic` 实际上并不可扩展，但也不值得做任何更花哨的事情。

## [PATCHv11 10/19] x86/mm: Add callbacks to prepare encrypted memory for kexec
* AMD SEV 和 Intel TDX guest 分配共享缓冲区来执行 I/O。这是通过从伙伴分配器正常分配页面，然后使用 `set_memory_decrypted()` 将它们转换为共享来完成的。
* 在 kexec 的场景中，第二个内核不知道哪个内存已以这种方式转换。它只看到 `E820_TYPE_RAM`。以私有方式访问共享内存是致命的。
* **批注**：个人推断，以不带 shared bit 的 *私有方式* 来访问 *共享页面*，理论上这样的 GPA 在 TDX module 维护的 SEPT 中没有映射，这会导致一次 VM Exit 到 host kernel。
  * KVM 如果用的是 guest memory 的实现，会发现缺页请求的是一个私有页面，与其所维护的内存属性还是“共享的”不一致，导致 EPT violation 并退出到 Qemu 去处理：
  1. 如果页面对应的 memory slot 不允许私有，比如对应的是 PCI 的 BAR（通常由 Qemu 创建，默认是共享的），结果是 VM Exit，这很致命；可问题是，MMIO 的地址范围也不在旧内核的 E820 表里，也不会被 dump 啊
  2. 如果页面对应的 memory slot 允许私有，这会将该内存页转为私有 `kvm_vm_ioctl(..., KVM_SET_MEMORY_ATTRIBUTES, &attr)`，好像还好。
     * 可问题是，通常这些内存是打算用作 direct/stream DMA 的（通过 `MapGPA -> PUNCH_HOLE` 转为共享的），比如 SWIOTLB buffer。
     * 如果是快速切换场景，重启前以上共享页面会被转换回私有；
     * 如果是 crash 场景，可能就来不及去做共享->私有转换，那么不排除设备会再次写这些它们看来是“共享”的页面，这会引起 `#MC`，这也很致命。
* 因此，在使用 kexec 启动新内核之前，必须将内存状态重置为原始状态。
* 将共享内存转换回私有内存的过程分两步进行：
  - `enc_kexec_begin()` 停止新的转换。
  - `enc_kexec_finish()` 取消所有现有共享内存的共享，将其恢复为私有内存。
---
* 给 `struct x86_guest` 增加两个回调函数 `enc_kexec_stop_conversion()` 和 `enc_kexec_unshare_mem()`
```diff
diff --git a/arch/x86/include/asm/x86_init.h b/arch/x86/include/asm/x86_init.h
index 28ac3cb9b987..6cade48811cc 100644
--- a/arch/x86/include/asm/x86_init.h
+++ b/arch/x86/include/asm/x86_init.h
@@ -149,12 +149,21 @@ struct x86_init_acpi {
  * @enc_status_change_finish   Notify HV after the encryption status of a range is changed
  * @enc_tlb_flush_required Returns true if a TLB flush is needed before changing page encryption status
  * @enc_cache_flush_required   Returns true if a cache flush is needed before changing page encryption status
+ * @enc_kexec_begin        Begin the two-step process of conversion shared memory back
+ *             to private. It stops the new conversions from being started
+ *             and waits in-flight conversions to finish, if possible.
+ * @enc_kexec_finish       Finish the two-step process of conversion shared memory to
+ *             private. All memory is private after the call.
+ *             It called with all CPUs but one shutdown and interrupts
+ *             disabled.
  */
 struct x86_guest {
    int (*enc_status_change_prepare)(unsigned long vaddr, int npages, bool enc);
    int (*enc_status_change_finish)(unsigned long vaddr, int npages, bool enc);
    bool (*enc_tlb_flush_required)(bool enc);
    bool (*enc_cache_flush_required)(void);
+   void (*enc_kexec_begin)(bool crash);
+   void (*enc_kexec_finish)(void);
 };
```
* 这两个回调暂时用的是空实现
* `struct x86_platform_ops x86_platform.guest` 全局变量的回调暂时用的空实现
```diff
diff --git a/arch/x86/kernel/x86_init.c b/arch/x86/kernel/x86_init.c
index a7143bb7dd93..8a79fb505303 100644
--- a/arch/x86/kernel/x86_init.c
+++ b/arch/x86/kernel/x86_init.c
@@ -138,6 +138,8 @@ static int enc_status_change_prepare_noop(unsigned long vaddr, int npages, bool
 static int enc_status_change_finish_noop(unsigned long vaddr, int npages, bool enc) { return 0; }
 static bool enc_tlb_flush_required_noop(bool enc) { return false; }
 static bool enc_cache_flush_required_noop(void) { return false; }
+static void enc_kexec_begin_noop(bool crash) {}
+static void enc_kexec_finish_noop(void) {}
 static bool is_private_mmio_noop(u64 addr) {return false; }

 struct x86_platform_ops x86_platform __ro_after_init = {
@@ -161,6 +163,8 @@ struct x86_platform_ops x86_platform __ro_after_init = {
        .enc_status_change_finish  = enc_status_change_finish_noop,
        .enc_tlb_flush_required    = enc_tlb_flush_required_noop,
        .enc_cache_flush_required  = enc_cache_flush_required_noop,
+       .enc_kexec_begin       = enc_kexec_begin_noop,
+       .enc_kexec_finish      = enc_kexec_finish_noop,
    },
 };
```
* 对于 kexec crash 场景，在 `native_machine_crash_shutdown()` 回调完成这两步动作
* Non-crash kexec 在调度仍处于活动状态时调用 `enc_kexec_begin()`。
  * 这允许回调等待所有正在进行的共享<->私有转换完成。
* 在 crash 的场景下，在除一个 CPU 外 *所有 CPU 都已关闭* 且（本 CPU）中断已禁用后，`enc_kexec_begin()` 才会被调用。
  * 这仅允许回调检测转换中的竞争并报告它。
```diff
diff --git a/arch/x86/kernel/crash.c b/arch/x86/kernel/crash.c
index f06501445cd9..74f6305eb9ec 100644
--- a/arch/x86/kernel/crash.c
+++ b/arch/x86/kernel/crash.c
@@ -128,6 +128,18 @@ void native_machine_crash_shutdown(struct pt_regs *regs)
 #ifdef CONFIG_HPET_TIMER
    hpet_disable();
 #endif
+
+   /*
+    * Non-crash kexec calls enc_kexec_begin() while scheduling is still
+    * active. This allows the callback to wait until all in-flight
+    * shared<->private conversions are complete. In a crash scenario,
+    * enc_kexec_begin() get call after all but one CPU has been shut down
+    * and interrupts have been disabled. This only allows the callback to
+    * detect a race with the conversion and report it.
+    */
+   x86_platform.guest.enc_kexec_begin(true);
+   x86_platform.guest.enc_kexec_finish();
+
    crash_save_cpu(regs, safe_smp_processor_id());
 }
```
* 对于 kexec 快速切换场景，在 `native_machine_shutdown()` 回调完成这两步动作
* 当所有 CPU 仍处于活动状态且中断已启用时，调用 `enc_kexec_begin()`。
  * 这将允许所有正在进行的内存转换顺利完成。
```diff
diff --git a/arch/x86/kernel/reboot.c b/arch/x86/kernel/reboot.c
index f3130f762784..097313147ad3 100644
--- a/arch/x86/kernel/reboot.c
+++ b/arch/x86/kernel/reboot.c
@@ -12,6 +12,7 @@
 #include <linux/delay.h>
 #include <linux/objtool.h>
 #include <linux/pgtable.h>
+#include <linux/kexec.h>
 #include <acpi/reboot.h>
 #include <asm/io.h>
 #include <asm/apic.h>
@@ -716,6 +717,14 @@ static void native_machine_emergency_restart(void)

 void native_machine_shutdown(void)
 {
+   /*
+    * Call enc_kexec_begin() while all CPUs are still active and
+    * interrupts are enabled. This will allow all in-flight memory
+    * conversions to finish cleanly.
+    */
+   if (kexec_in_progress)
+       x86_platform.guest.enc_kexec_begin(false);
+
    /* Stop the cpus and apics */
 #ifdef CONFIG_X86_IO_APIC
    /*
@@ -752,6 +761,9 @@ void native_machine_shutdown(void)
 #ifdef CONFIG_X86_64
    x86_platform.iommu_shutdown();
 #endif
+
+   if (kexec_in_progress)
+       x86_platform.guest.enc_kexec_finish();
 }
```

## [PATCHv11 11/19] x86/tdx: Convert shared memory back to private on kexec
* TDX guest 分配共享缓冲区来执行 I/O。它是通过通常从伙伴分配器分配页面并将它们转换为使用 `set_memory_decrypted()` 共享来完成的。
* 第二个内核不知道什么内存被以这种方式转换了。它只看到 `E820_TYPE_RAM`。
* 通过私有映射访问共享内存是致命的。它会导致不可恢复的 TD 退出。
* 在 kexec 上进行直接映射并将所有共享内存转换回私有内存。它使所有 RAM 再次变为私有，第二个内核可以正常使用它。
* 转换分两个步骤进行：
  * 停止新的转换和取消共享所有内存。**在正常 kexec 的情况下**，转换停止发生，而调度仍在运行。这允许等待任何正在进行的转换完成。
  * 当除一个 CPU 之外的所有 CPU 均处于非活动状态且禁用中断时，执行第二步。这可以防止与可能访问共享内存的代码发生任何冲突。
---
* 新增读写信号量 `mem_enc_lock` 和函数 `set_memory_enc_stop_conversion()` 停止新的私有内存到共享内存的转换
* 获取独占 `mem_enc_lock` 会等待正在进行的转换完成。
  * 锁不会被释放以防止开始新的转换。
* 如果不允许睡眠（如在 crash 情况下），会尝试获取锁。
  * 失败表明存在与转换的竞争。
```cpp
/*
 * The lock serializes conversions between private and shared memory.
 *
 * It is taken for read on conversion. A write lock guarantees that no
 * concurrent conversions are in progress.
 */
static DECLARE_RWSEM(mem_enc_lock);

/*
 * Stop new private<->shared conversions.
 *
 * Taking the exclusive mem_enc_lock waits for in-flight conversions to complete.
 * The lock is not released to prevent new conversions from being started.
 *
 * If sleep is not allowed, as in a crash scenario, try to take the lock.
 * Failure indicates that there is a race with the conversion.
 */
bool set_memory_enc_stop_conversion(bool wait)
{
    if (!wait)
        return down_write_trylock(&mem_enc_lock);

    down_write(&mem_enc_lock);

    return true;
}
```
* 新增函数 `tdx_kexec_begin()` 停止新的私有内存到共享内存的转换
  * Crash kernel 到达这时中断是关闭的：因此 *不能等待转换完成*，`!crash` 的值为 `false`
  * 如果发生竞争，仅报告和处理
```cpp
/* Stop new private<->shared conversions */
static void tdx_kexec_begin(bool crash)
{
    /*
     * Crash kernel reaches here with interrupts disabled: can't wait for
     * conversions to finish.
     *
     * If race happened, just report and proceed.
     */
    if (!set_memory_enc_stop_conversion(!crash))
        pr_warn("Failed to stop shared<->private conversions\n");
}
```
* `__set_memory_enc_dec()` 是私有/共享内存转换共有的一个函数，这是一个很好的控制点
```diff
diff --git a/arch/x86/mm/pat/set_memory.c b/arch/x86/mm/pat/set_memory.c
index 6c49f69c0368..21835339c0e6 100644
--- a/arch/x86/mm/pat/set_memory.c
+++ b/arch/x86/mm/pat/set_memory.c
 static int __set_memory_enc_dec(unsigned long addr, int numpages, bool enc)
 {
-   if (cc_platform_has(CC_ATTR_MEM_ENCRYPT))
-       return __set_memory_enc_pgtable(addr, numpages, enc);
+   int ret = 0;

-   return 0;
+   if (cc_platform_has(CC_ATTR_MEM_ENCRYPT)) {
+       if (!down_read_trylock(&mem_enc_lock))
+           return -EBUSY;
+
+       ret = __set_memory_enc_pgtable(addr, numpages, enc);
+
+       up_read(&mem_enc_lock);
+   }
+
+   return ret;
 }
```
* `__set_memory_enc_dec()` 和 kexec 竞争内存转换的锁 `mem_enc_lock`，它拿的读锁会阻止 kexec 拿写锁的请求；同时允许多个读者
* 对于快速切换场景 kexec，如果其他 CPU 上在转换内存，我们会在 `x86_platform.guest.enc_kexec_begin(false)` 处等其他 CPU 转完，然后我们会持有写锁 `mem_enc_lock`
  * 随后 `stop_other_cpus()` 会停止其他 CPU
  * 接着调用 `x86_platform.guest.enc_kexec_finish()` 进行共享 -> 私有的转换，我们是最后一个 CPU，不会有问题。
* 对于 crash kexec 场景，crash 核会发出的 `NMI_LOCAL` IPI 让其他 CPU 进入 `crash_nmi_callback()` 的等待重启死循环
  * 其他 CPU 如正在进行的转换，会被 Delivery Mode 是 NMI 类型的 IPI 打断进入等待重启死循环
  * 尽管其他 CPU 还持有读锁 `mem_enc_lock`，但此场景下到达 `tdx_kexec_begin()` 时是中断上下文不能睡眠，因此用的是 `trylock`，拿不到锁也只是报个错继续处理
  * 接着调用 `x86_platform.guest.enc_kexec_finish()` 进行共享 -> 私有的转换，这样看起来没有问题
  * 问题出在，crash 核对其他 CPU 是否已经进入死等状态的等待不是强制的，最多只会等待 1 秒，见 `nmi_shootdown_cpus()`
    * 如果其他正在进行私有 -> 共享转换的 CPU 没有在这 1 秒内收到 NMI 被打断，而是还在转换
    * 而随后 crash 核继续调 `x86_platform.guest.enc_kexec_finish() -> tdx_enc_status_changed()` 去转换共享 -> 私有内存，由于这里没有互斥机制，可能会与其他CPU 上正在进行的转换冲突，这是有问题的，尽管出问题的可能性（其他核 1 秒内还收不到 NMI）不大！
```cpp
native_machine_crash_shutdown()
-> crash_smp_send_stop()
   -> smp_ops.crash_stop_other_cpus()
   => kdump_nmi_shootdown_cpus()
      -> nmi_shootdown_cpus(kdump_nmi_callback)
         -> register_nmi_handler(NMI_LOCAL, crash_nmi_callback, NMI_FLAG_FIRST, "crash")
         -> apic_send_IPI_allbutself(NMI_VECTOR) //发 NMI 类型的 IPI 让其他核停止工作，进入等待重启死循环
            msecs = 1000; //crash CPU 等待 1 秒
         -> while ((atomic_read(&waiting_for_crash_ipi) > 0) && msecs) {
               mdelay(1);
               msecs--;
            }
-> x86_platform.guest.enc_kexec_begin(true)
=> tdx_kexec_begin(true) //无阻塞式争锁 trylock，拿不到也无所谓
-> x86_platform.guest.enc_kexec_finish()
=> tdx_kexec_finish() //共享 -> 私有的转换
```
* 如果出现漏网之鱼，比如 crash 核先将某页共享转私有，其他核又将该页私有转共享，那么捕捉 kernel 是否将来读取这些未转换的共享内存是否会有问题？
  * 我觉得是有的，因为捕捉内核会以私有的方式映射所有 old memory
```cpp
read_vmcore()
-> __read_vmcore()
   -> read_from_oldmem(iter, tsz, &start, cc_platform_has(CC_ATTR_MEM_ENCRYPT)) //对于机密容器最后一个参数 encrypted = true
         if (encrypted)
         -> copy_oldmem_page_encrypted()
            -> __copy_oldmem_page(iter, pfn, csize, offset, true)
               if (encrypted)
               -> vaddr = (__force void *)ioremap_encrypted(pfn << PAGE_SHIFT, PAGE_SIZE);
                  -> __ioremap_caller(phys_addr, size, _PAGE_CACHE_MODE_WB, __builtin_return_address(0), true)
                        if ((io_desc.flags & IORES_MAP_ENCRYPTED) || encrypted) //encrypted = true，走这个分支
                            prot = pgprot_encrypted(prot); //以私有的方式映射所有 old memory
                        else
                            prot = pgprot_decrypted(prot);
               -> csize = copy_to_iter(vaddr + offset, csize, iter);
               -> iounmap((void __iomem *)vaddr)
```
* 新增函数 `pte_decrypted()` 判断给定的 PTE 条目是否已解密
```cpp
static inline bool pte_decrypted(pte_t pte)
{
   return cc_mkdec(pte_val(pte)) == pte_val(pte);
}
```

* 新增函数 `tdx_kexec_finish()` 将所有共享内存转回为私有
* `PAGE_OFFSET` 是内核直接映射的起始地址，对于 x86 四级页表是 `0xffff888000000000`，五级页表是 `0xff11000000000000`
* `get_max_mapped()` 返回最大物理页帧 `max_pfn_mapped` 转成物理地址
* 由于物理页帧通常从 `0` 开始算起，`end  = PAGE_OFFSET + get_max_mapped()` 得到的就是直接映射的最大虚拟地址
```cpp
/* Walk direct mapping and convert all shared memory back to private */
static void tdx_kexec_finish(void)
{
    unsigned long addr, end;
    long found = 0, shared;

    lockdep_assert_irqs_disabled();
    //得到内核直接映射的地址范围
    addr = PAGE_OFFSET;
    end  = PAGE_OFFSET + get_max_mapped();
    //遍历内核直接映射的地址
    while (addr < end) {
        unsigned long size;
        unsigned int level;
        pte_t *pte;
        //得到映射该地址的 PTE 的指针和映射的级别
        pte = lookup_address(addr, &level);
        size = page_level_size(level); //该级别的页大小是多少字节
        //如果 PTE 条目有效且是共享映射
        if (pte && pte_decrypted(*pte)) {
            int pages = size / PAGE_SIZE; //得到页的数目
            //接触设置共享位的内存会触发隐式转换为共享。设置条目为 0 确保从现在开始没有人触及共享范围。
            /*
             * Touching memory with shared bit set triggers implicit
             * conversion to shared.
             *
             * Make sure nobody touches the shared range from
             * now on.
             */
            set_pte(pte, __pte(0));
            //将从 addr 开始的 pages 个共享页面转为私有
            if (!tdx_enc_status_changed(addr, pages, true)) {
                pr_err("Failed to unshare range %#lx-%#lx\n",
                       addr, addr + size);
            }
            //累加页面计数
            found += pages;
        }
        //往下一个条目的地址步进
        addr += size;
    }
    //刷新所有 TLB
    __flush_tlb_all();
    //读取之前的共享页的计数，与现在转换的共享页计数比较，如果不一致，报错
    shared = atomic_long_read(&nr_shared);
    if (shared != found) {
        pr_err("shared page accounting is off\n");
        pr_err("nr_shared = %ld, nr_found = %ld\n", shared, found);
    }
}
```
* 是时候将 `enc_kexec_begin` 和 `enc_kexec_finish` 替换成有具体实现的 `tdx_kexec_begin()` 和 `tdx_kexec_finish()`
```diff
diff --git a/arch/x86/coco/tdx/tdx.c b/arch/x86/coco/tdx/tdx.c
index 979891e97d83..59776ce1c1d7 100644
--- a/arch/x86/coco/tdx/tdx.c
+++ b/arch/x86/coco/tdx/tdx.c
@@ -890,6 +956,9 @@ void __init tdx_early_init(void)
    x86_platform.guest.enc_cache_flush_required  = tdx_cache_flush_required;
    x86_platform.guest.enc_tlb_flush_required    = tdx_tlb_flush_required;

+   x86_platform.guest.enc_kexec_begin       = tdx_kexec_begin;
+   x86_platform.guest.enc_kexec_finish      = tdx_kexec_finish;
+
    /*
     * TDX intercepts the RDMSR to read the X2APIC ID in the parallel
     * bringup low level code. That raises #VE which cannot be handled
```

## [PATCHv11 12/19] x86/mm: Make e820__end_ram_pfn() cover E820_TYPE_ACPI ranges
* `e820__end_of_ram_pfn()` 用于计算 `max_pfn`，除此之外，它还指导直接映射结束的位置。任何高于 `max_pfn` 的内存都不会出现在直接映射中。
* `e820__end_of_ram_pfn()` 根据最高的 `E820_TYPE_RAM` 范围查找 ram 的末尾。但它不会把 `E820_TYPE_ACPI` 的范围到算在其中。
* 尽管名为 `E820_TYPE_ACPI`，但它不仅涵盖 ACPI 数据，还涵盖 EFI 表，并且内核可能需要它才能正常运行。
* 问题是隐藏的，因为通常在 `E820_TYPE_ACPI` 之上还有一些 `E820_TYPE_RAM` 内存。
  * 但 crashkernel 仅在启动时将预分配的 crash 内存呈现为 `E820_TYPE_RAM`。而如果预分配范围较小，它可以适合出现在最后一个 `E820_TYPE_ACPI` 之前的范围内。
* 修改 `e820__end_of_ram_pfn()` 和 `e820__end_of_low_ram_pfn()` 以覆盖 `E820_TYPE_ACPI` 内存。
* 该问题是在调试 TDX guest 的 kexec 过程中发现的。TDX guest 使用 `E820_TYPE_ACPI` 来存储待接受的内存位图，并在 kexec 上的内核之间传递它。
---
* `e820_end_pfn()` 改名为 `e820__end_ram_pfn()`，且去掉最后一个参数，因为它仅有的两个调用者传的都是 `E820_TYPE_RAM`
```diff
diff --git a/arch/x86/kernel/e820.c b/arch/x86/kernel/e820.c
index fb8cf953380d..a72857ed03ee 100644
--- a/arch/x86/kernel/e820.c
+++ b/arch/x86/kernel/e820.c
@@ -827,7 +827,7 @@ u64 __init e820__memblock_alloc_reserved(u64 size, u64 align)
 /*
  * Find the highest page frame number we have available
  */
-static unsigned long __init e820_end_pfn(unsigned long limit_pfn, enum e820_type type)
+static unsigned long __init e820__end_ram_pfn(unsigned long limit_pfn)
 {
    int i;
    unsigned long last_pfn = 0;
```
* 过 E820 条目的时候也考虑 `E820_TYPE_ACPI` 类型，尽管名为 `E820_TYPE_ACPI`，但它不仅涵盖 ACPI 数据，还涵盖 EFI 表，并且内核可能需要它才能正常运行。
  * 这里举的一个例子是 TDX guest 使用 `E820_TYPE_ACPI` 来存储待接受的内存位图，并在 kexec 之上的内核之间传递它。
* **批注**：尝试理解一下遇到的问题，在 crash 场景，crash kernel 会遍历 E820 表划定直接映射的范围，如果分给 crash system 的保留内存很小，`E820_TYPE_ACPI` 类型的内存有可能在保留内存之后，导致 `E820_TYPE_ACPI` 类型的内存不在 crash kernel 直接映射的范围内，从而导致 crash kernel 无法访问到待接受的内存位图。因此，划定直接映射的范围时需包含 `E820_TYPE_ACPI` 类型的内存。
```diff
@@ -838,7 +838,8 @@ static unsigned long __init e820_end_pfn(unsigned long limit_pfn, enum e820_type
        unsigned long start_pfn;
        unsigned long end_pfn;

-       if (entry->type != type)
+       if (entry->type != E820_TYPE_RAM &&
+           entry->type != E820_TYPE_ACPI)
            continue;

        start_pfn = entry->addr >> PAGE_SHIFT;
```
* 修改它的两个调用者，适配新的改动
```diff
@@ -864,12 +865,12 @@ static unsigned long __init e820_end_pfn(unsigned long limit_pfn, enum e820_type

 unsigned long __init e820__end_of_ram_pfn(void)
 {
-   return e820_end_pfn(MAX_ARCH_PFN, E820_TYPE_RAM);
+   return e820__end_ram_pfn(MAX_ARCH_PFN);
 }

 unsigned long __init e820__end_of_low_ram_pfn(void)
 {
-   return e820_end_pfn(1UL << (32 - PAGE_SHIFT), E820_TYPE_RAM);
+   return e820__end_ram_pfn(1UL << (32 - PAGE_SHIFT));
 }
```

## [PATCHv11 13/19] x86/mm: Do not zap page table entries mapping unaccepted memory table during kdump.
* 在 crashkernel 启动期间，仅预分配的 crash 内存被显示为 `E820_TYPE_RAM`。这可能会导致 *映射待接受的内存表的页表条目* 在 `phys_pte_init()`、`phys_pmd_init()`、`phys_pud_init()` 和 `phys_p4d_init()` 期间被 zapped，因为 SNP/TDX guest 使用 `E820_TYPE_ACPI` 来存储 *未接受的内存表* 并在 kexec/kdump 之间的内核传递它。
* `E820_TYPE_ACPI` 不仅涵盖 ACPI 数据，还涵盖 EFI 表，内核可能需要它才能正常运行。
* 该问题是在调试 SNP guest 的 kdump 时发现的。使用 `E820_TYPE_ACPI` 存储并在 kdump 之间的内核传递的 *未接受的内存表* 正在被 zapped，因为映射它的 `PMD` 条目高于保留的 crashkernel 内存的 `E820_TYPE_RAM` 范围。
---
* 看懂这个修改的前提是看懂内核建立直接映射的过程。修改的地方位于清除不属于指定的 e820 表映射范围的页表条目，把 *映射待接受的内存表* 的 `E820_TYPE_ACPI` 类型的映射囊括在指定范围内，可以避免 *映射待接受的内存表的页表条目* 在 kexec/kdump 之间的内核传递过程中被清除
```diff
diff --git a/arch/x86/mm/init_64.c b/arch/x86/mm/init_64.c
index 7e177856ee4f..28002cc7a37d 100644
--- a/arch/x86/mm/init_64.c
+++ b/arch/x86/mm/init_64.c
@@ -469,7 +469,9 @@ phys_pte_init(pte_t *pte_page, unsigned long paddr, unsigned long paddr_end,
                !e820__mapped_any(paddr & PAGE_MASK, paddr_next,
                         E820_TYPE_RAM) &&
                !e820__mapped_any(paddr & PAGE_MASK, paddr_next,
-                        E820_TYPE_RESERVED_KERN))
+                        E820_TYPE_RESERVED_KERN) &&
+               !e820__mapped_any(paddr & PAGE_MASK, paddr_next,
+                        E820_TYPE_ACPI))
                set_pte_init(pte, __pte(0), init);
            continue;
        }
@@ -524,7 +526,9 @@ phys_pmd_init(pmd_t *pmd_page, unsigned long paddr, unsigned long paddr_end,
                !e820__mapped_any(paddr & PMD_MASK, paddr_next,
                         E820_TYPE_RAM) &&
                !e820__mapped_any(paddr & PMD_MASK, paddr_next,
-                        E820_TYPE_RESERVED_KERN))
+                        E820_TYPE_RESERVED_KERN) &&
+               !e820__mapped_any(paddr & PMD_MASK, paddr_next,
+                        E820_TYPE_ACPI))
                set_pmd_init(pmd, __pmd(0), init);
            continue;
        }
@@ -611,7 +615,9 @@ phys_pud_init(pud_t *pud_page, unsigned long paddr, unsigned long paddr_end,
                !e820__mapped_any(paddr & PUD_MASK, paddr_next,
                         E820_TYPE_RAM) &&
                !e820__mapped_any(paddr & PUD_MASK, paddr_next,
-                        E820_TYPE_RESERVED_KERN))
+                        E820_TYPE_RESERVED_KERN) &&
+               !e820__mapped_any(paddr & PUD_MASK, paddr_next,
+                        E820_TYPE_ACPI))
                set_pud_init(pud, __pud(0), init);
            continue;
        }
@@ -698,7 +704,9 @@ phys_p4d_init(p4d_t *p4d_page, unsigned long paddr, unsigned long paddr_end,
                !e820__mapped_any(paddr & P4D_MASK, paddr_next,
                         E820_TYPE_RAM) &&
                !e820__mapped_any(paddr & P4D_MASK, paddr_next,
-                        E820_TYPE_RESERVED_KERN))
+                        E820_TYPE_RESERVED_KERN) &&
+               !e820__mapped_any(paddr & P4D_MASK, paddr_next,
+                        E820_TYPE_ACPI))
                set_p4d_init(p4d, __p4d(0), init);
            continue;
        }
```

## [PATCHv11 14/19] x86/acpi: Rename fields in acpi_madt_multiproc_wakeup structure
* 为了准备添加对 MADT 唤醒结构 version 1 的支持，有必要为结构中的字段提供更合适的名称。
* `mailbox_version` 域已重命名为 `version`。该域表示结构和相关协议的版本，而不是 mailbox 的版本。到目前为止，该域尚未在代码中使用。
* `base_address` 域已重命名为 `mailbox_address`，以阐明它所代表的地址类型。在 version 1 中，该结构包括复位向量地址。清晰明确的命名有助于防止任何混淆。
---
```diff
diff --git a/include/acpi/actbl2.h b/include/acpi/actbl2.h
index 9775384d61c6..e1a395af7591 100644
--- a/include/acpi/actbl2.h
+++ b/include/acpi/actbl2.h
@@ -1117,9 +1117,9 @@ struct acpi_madt_generic_translator {

 struct acpi_madt_multiproc_wakeup {
    struct acpi_subtable_header header;
-   u16 mailbox_version;
+   u16 version;
    u32 reserved;       /* reserved - must be zero */
-   u64 base_address;
+   u64 mailbox_address;
 };

 #define ACPI_MULTIPROC_WAKEUP_MB_OS_SIZE        2032
```
* 引用 `base_address` 域的地方做出相应的修改，变为 `mailbox_address`
```diff
diff --git a/arch/x86/kernel/acpi/madt_wakeup.c b/arch/x86/kernel/acpi/madt_wakeup.c
index d222be8d7a07..004801b9b151 100644
--- a/arch/x86/kernel/acpi/madt_wakeup.c
+++ b/arch/x86/kernel/acpi/madt_wakeup.c
@@ -75,7 +75,7 @@ int __init acpi_parse_mp_wake(union acpi_subtable_headers *header,

    acpi_table_print_madt_entry(&header->common);

-   acpi_mp_wake_mailbox_paddr = mp_wake->base_address;
+   acpi_mp_wake_mailbox_paddr = mp_wake->mailbox_address;

    cpu_hotplug_disable_offlining();
```

## [PATCHv11 15/19] x86/acpi: Do not attempt to bring up secondary CPUs in kexec case
* ACPI MADT 不允许在 CPU onlined 后将其 offline。这限制了 kexec：第二个内核将无法使用多个 CPU。
* 为了防止 kexec 内核使用已经 onlining 的辅助 CPU，使 ACPI MADT 唤醒结构中的 mailbox 地址无效，从而阻止 kexec 内核使用它。
* 这是安全的，因为引导内核已经缓存了 mailbox 地址，并且 `acpi_wakeup_cpu()` 使用缓存的值来启动辅助 CPU。
* 注意：这是 Linux 特定约定，ACPI 规范未涵盖。
---
* 目前的 commit message 来自 tglx：[Re: [PATCHv4 13/14] x86/acpi: Do not attempt to bring up secondary CPUs in kexec case](https://lore.kernel.org/lkml/87a5qbmdp5.ffs@tglx/)，由此可知这个 commit 原来是针对 kdump 的场景
* 新增函数 `acpi_mp_disable_offlining()` 禁止 CPU offline，并且使 ACPI MADT 唤醒结构中的 mailbox 地址无效，这样 capture kernel 看到的 mailbox 地址就是 `0`
* 这里有两个很关键的点，以下是我的理解：
1. 为什么 capture kernel 看到的 mailbox 地址就是 `0`？
   * Capture kernel 看到的 E820 表是 kexec tools 给它构造的，不是真的用 BIOS E820 中断枚举得到的
   * Kexec tools 根据在生产环境中收集到的内存信息构造捕捉环境的 E820 表，放在 crash kernel region 的 real_mode_data segment 中
   * 因为用的是 real_mode_data 中指引的 ACPI data 的范围，capture kernel 看到的 ACPI data 和 production kernel 的是一样的
   * 因此 production kernel 对 ACPI MADT 唤醒结构的修改在 crash kernel 起来后还会保留在内存中，并被 capture kernel 看到
2. Kexec 快速切换场景第二内核看到的 mailbox 地址是什么？
   * Kexec 快速切换的第二内核的 E820 表的构造过程与 kexec crash 的类似，引用的也是 ACPI data 中的数据
   * 因此直至此 patch，kexec 第二内核的效果也是只能 online 一个 CPU
* 后面的 patch 做出修改，在 `acpi_parse_mp_wake() -> acpi_mp_setup_reset(mp_wake->reset_vector)` 返回成功的情况下，是不会调用 `acpi_mp_disable_offlining(mp_wake)` 的，即 CPU offline 不会被禁用，也不会清除 mailbox 地址。这样就 kexec 第二内核或者 crash kernel 也可以多处理器唤醒了
```cpp
static void acpi_mp_disable_offlining(struct acpi_madt_multiproc_wakeup *mp_wake)
{
    cpu_hotplug_disable_offlining();

    /*
     * ACPI MADT doesn't allow to offline a CPU after it was onlined. This
     * limits kexec: the second kernel won't be able to use more than one CPU.
     *
     * To prevent a kexec kernel from onlining secondary CPUs invalidate the
     * mailbox address in the ACPI MADT wakeup structure which prevents a
     * kexec kernel to use it.
     *
     * This is safe as the booting kernel has the mailbox address cached
     * already and acpi_wakeup_cpu() uses the cached value to bring up the
     * secondary CPUs.
     *
     * Note: This is a Linux specific convention and not covered by the
     *       ACPI specification.
     */
    mp_wake->mailbox_address = 0;
}
```
* 在启动过程中解析 MADT 中的 Multiprocessor Wakeup Structure 的时候，调用新添加的 `acpi_mp_disable_offlining()`，目的是为了防止 capture kernel 使用 onlining 的辅助 CPU
* 前一行语句中我们看到 ACPI MADT 唤醒结构中的 mailbox 地址已经缓存到 `acpi_mp_wake_mailbox_paddr` 全局变量中了
* 下一行将 `acpi_wakeup_cpu()` 设置为 `apic->wakeup_secondary_cpu_64()` 的回调
```diff
@@ -77,7 +104,7 @@ int __init acpi_parse_mp_wake(union acpi_subtable_headers *header,

    acpi_mp_wake_mailbox_paddr = mp_wake->mailbox_address;

-   cpu_hotplug_disable_offlining();
+   acpi_mp_disable_offlining(mp_wake);

    apic_update_callback(wakeup_secondary_cpu_64, acpi_wakeup_cpu);
```
* BSP 唤醒 AP 时有如下路径，**有几个 AP 该路径就会被 BSP 调用几次**，最后调用之前设置好的 `acpi_wakeup_cpu()` 回调：
```cpp
bringup_cpu()
-> __cpu_up()
   -> smp_ops.cpu_up()
   => native_cpu_up()
      -> do_boot_cpu()
         -> apic->wakeup_secondary_cpu_64(apicid, start_ip)
         => acpi_wakeup_cpu()
```
* 引导内核用缓存的 mailbox 地址 `acpi_mp_wake_mailbox_paddr` 来启动辅助 CPU。
```diff
diff --git a/arch/x86/kernel/acpi/madt_wakeup.c b/arch/x86/kernel/acpi/madt_wakeup.c
index 004801b9b151..30820f9de5af 100644
--- a/arch/x86/kernel/acpi/madt_wakeup.c
+++ b/arch/x86/kernel/acpi/madt_wakeup.c
@@ -14,6 +14,11 @@ static struct acpi_madt_multiproc_wakeup_mailbox *acpi_mp_wake_mailbox __ro_afte

 static int acpi_wakeup_cpu(u32 apicid, unsigned long start_ip)
 {
+   if (!acpi_mp_wake_mailbox_paddr) {
+       pr_warn_once("No MADT mailbox: cannot bringup secondary CPUs. Booting with kexec?\n");
+       return -EOPNOTSUPP;
+   }
+
    /*
     * Remap mailbox memory only for the first call to acpi_wakeup_cpu().
     *
```
* 因为第一个内核将 ACPI MADT 唤醒结构中的 mailbox 地址无效了，因此 capture kernel 在 `acpi_parse_mp_wake()` 时看到的 mailbox 是 `0`，给 `acpi_mp_wake_mailbox_paddr` 赋值的是 `0`，当 crash kernel 的 BSP 走到这里时会直接返回 `-EOPNOTSUPP`，不去唤醒辅助 CPU

## [PATCHv11 16/19] x86/smp: Add smp_ops.stop_this_cpu() callback
* 如果定义了 helper，则在 `stop_this_cpu()` 结束时以及 crash CPU 关闭时调用它而不是 `halt()` 来停止 CPU。
* ACPI MADT 将使用它将 CPU 移交给 BIOS，以便能够在 kexec 之后再次唤醒它。
---
* 增加 `struct smp_ops` 函数指针 `stop_this_cpu`
```diff
diff --git a/arch/x86/include/asm/smp.h b/arch/x86/include/asm/smp.h
index a35936b512fe..ca073f40698f 100644
--- a/arch/x86/include/asm/smp.h
+++ b/arch/x86/include/asm/smp.h
@@ -35,6 +35,7 @@ struct smp_ops {
    int (*cpu_disable)(void);
    void (*cpu_die)(unsigned int cpu);
    void (*play_dead)(void);
+   void (*stop_this_cpu)(void);

    void (*send_call_func_ipi)(const struct cpumask *mask);
    void (*send_call_func_single_ipi)(int cpu);
```
* 在 kexec 快速切换场景，其他 CPU 最终会走到 `stop_this_cpu()` 进入 `halt()`，在此之前调用 `smp_ops.stop_this_cpu()`
```diff
diff --git a/arch/x86/kernel/process.c b/arch/x86/kernel/process.c
index 5ba8a9c1e47a..829ee0a03e98 100644
--- a/arch/x86/kernel/process.c
+++ b/arch/x86/kernel/process.c
@@ -833,6 +833,13 @@ void __noreturn stop_this_cpu(void *dummy)
     */
    cpumask_clear_cpu(cpu, &cpus_stop_mask);

+#ifdef CONFIG_SMP
+   if (smp_ops.stop_this_cpu) {
+       smp_ops.stop_this_cpu();
+       unreachable();
+   }
+#endif
+
    for (;;) {
        /*
         * Use native_halt() so that memory contents don't change
```
* 在 kexec crash 场景，其他 CPU 最终会走到 `crash_nmi_callback()` 进入 `halt()`，在此之前调用 `smp_ops.stop_this_cpu()`
```diff
diff --git a/arch/x86/kernel/reboot.c b/arch/x86/kernel/reboot.c
index 1ec478f40963..293ded05a4b0 100644
--- a/arch/x86/kernel/reboot.c
+++ b/arch/x86/kernel/reboot.c
@@ -880,6 +880,12 @@ static int crash_nmi_callback(unsigned int val, struct pt_regs *regs)
    cpu_emergency_disable_virtualization();

    atomic_dec(&waiting_for_crash_ipi);
+
+   if (smp_ops.stop_this_cpu) {
+       smp_ops.stop_this_cpu();
+       unreachable();
+   }
+
    /* Assume hlt works */
    halt();
    for (;;)
```

## [PATCHv11 17/19] x86/mm: Introduce kernel_ident_mapping_free()
* 该 helper 补充了 `kernel_ident_mapping_init()`：它释放先前分配的恒等映射。它将在错误路径中用于释放部分分配的映射或不再需要该映射。
* 调用者提供一个 `struct x86_mapping_info`，其中挂接了 `free_pgd_page()` 回调和要释放的 `pgd_t`。
---
* 在 ` struct x86_mapping_info` 中添加新函数指针 `free_pgt_page()`
* 添加新函数 `kernel_ident_mapping_free()` 的声明
```diff
diff --git a/arch/x86/include/asm/init.h b/arch/x86/include/asm/init.h
index cc9ccf61b6bd..14d72727d7ee 100644
--- a/arch/x86/include/asm/init.h
+++ b/arch/x86/include/asm/init.h
@@ -6,6 +6,7 @@

 struct x86_mapping_info {
    void *(*alloc_pgt_page)(void *); /* allocate buf for page table */
+   void (*free_pgt_page)(void *, void *); /* free buf for page table */
    void *context;           /* context for alloc_pgt_page */
    unsigned long page_flag;     /* page flag for PMD or PUD entry */
    unsigned long offset;        /* ident mapping offset */
@@ -16,4 +17,6 @@ struct x86_mapping_info {
 int kernel_ident_mapping_init(struct x86_mapping_info *info, pgd_t *pgd_page,
                unsigned long pstart, unsigned long pend);

+void kernel_ident_mapping_free(struct x86_mapping_info *info, pgd_t *pgd);
+
 #endif /* _ASM_X86_INIT_H */
```
* 新增释放恒等映射的接口函数 `kernel_ident_mapping_free()` 和支撑它的子函数
  * arch/x86/mm/ident_map.c
```cpp
static void free_pte(struct x86_mapping_info *info, pmd_t *pmd)
{   //pmd 条目指向 ptd 页表页的第一条记录，即 pte 页表页的起始地址
    pte_t *pte = pte_offset_kernel(pmd, 0);
    //释放这个 pte 页表页
    info->free_pgt_page(pte, info->context);
}

static void free_pmd(struct x86_mapping_info *info, pud_t *pud)
{   //pud 条目指向 pmd 页表页的第一条记录，即 pmd 页表页的起始地址
    pmd_t *pmd = pmd_offset(pud, 0);
    int i;
    //遍历 pmd 的 512 个条目
    for (i = 0; i < PTRS_PER_PMD; i++) {
        if (!pmd_present(pmd[i])) //如果该条目未映射，则跳过
            continue;
        //如果该条目已映射，且映射的是大页（即下一级是一个数据页），跳过
        if (pmd_leaf(pmd[i]))
            continue;
        //如果该条目已映射，则将释放操作传递到该条目下一级的 pte 页表页
        free_pte(info, &pmd[i]);
    }
    //释放这个 pmd 页表页
    info->free_pgt_page(pmd, info->context);
}

static void free_pud(struct x86_mapping_info *info, p4d_t *p4d)
{   //p4d 条目指向 pud 页表页的第一条记录，即 pud 页表页的起始地址
    pud_t *pud = pud_offset(p4d, 0);
    int i;
    //遍历 pud 的 512 个条目
    for (i = 0; i < PTRS_PER_PUD; i++) {
        if (!pud_present(pud[i])) //如果该条目未映射，则跳过
            continue;
        //如果该条目已映射，且映射的是大页（即下一级是一个数据页），跳过
        if (pud_leaf(pud[i]))
            continue;
        //如果该条目已映射，则将释放操作传递到该条目下一级的 pmd 页表页
        free_pmd(info, &pud[i]);
    }
    //释放这个 pud 页表页
    info->free_pgt_page(pud, info->context);
}

static void free_p4d(struct x86_mapping_info *info, pgd_t *pgd)
{   //pgd 条目指向 p4d 页表页的第一条记录，即 p4d 页表页的起始地址
    p4d_t *p4d = p4d_offset(pgd, 0);
    int i;
    //遍历 p4d 的 512 个条目
    for (i = 0; i < PTRS_PER_P4D; i++) {
        if (!p4d_present(p4d[i])) //如果该条目未映射，则跳过
            continue;
        //如果该条目已映射，则将释放操作传递到该条目下一级的 pud 页表页
        free_pud(info, &p4d[i]);
    }
    //如果启用 5 级页表，p4d 有实质内容，释放这个 p4d 页表页
    if (pgtable_l5_enabled())
        info->free_pgt_page(pgd, info->context);
}

void kernel_ident_mapping_free(struct x86_mapping_info *info, pgd_t *pgd)
{
    int i;
    //遍历 pgd 的 512 个条目
    for (i = 0; i < PTRS_PER_PGD; i++) {
        if (!pgd_present(pgd[i])) //如果该条目未映射，则跳过
            continue;
        //如果该条目已映射，则将释放操作传递到该条目下一级的 p4d 页表页
        free_p4d(info, &pgd[i]);
    }
    //释放这个 pgd 页表页
    info->free_pgt_page(pgd, info->context);
}
```

## [PATCHv11 18/19] x86/acpi: Add support for CPU offlining for ACPI MADT wakeup method
* MADT Multiprocessor Wakeup Structure version 1 带来了对 CPU offlining 的支持：
  * BIOS 提供了一个 *复位向量*，CPU 必须跳转到该向量才能自行 offline。
  * 新的 `TEST` mailbox 命令可用于测试 CPU 是否自行 offlined，这意味着 BIOS 可以控制 CPU 并可以通过 ACPI MADT 唤醒方法使其再次 online。
* 通过实现自定义的 `cpu_die()`、`play_dead()` 和 `stop_this_cpu()` 等 SMP 操作，添加对 ACPI MADT 唤醒方法的 CPU offling 支持。
* CPU offlining 使得可以将辅助 CPU 移交给 kexec，而不是将第二个内核限制为单个 CPU。
* 此更改符合已批准的 ACPI 规范更改提案。请参阅链接。
---
### 新增的宏和复位向量域
* 描述 MADT 中的 Multiprocessor Wakeup Structure 的 `struct acpi_madt_multiproc_wakeup` 增加 `reset_vector` 代表 **复位向量（reset vector）**
```diff
diff --git a/include/acpi/actbl2.h b/include/acpi/actbl2.h
index e1a395af7591..2aedda70ef88 100644
--- a/include/acpi/actbl2.h
+++ b/include/acpi/actbl2.h
@@ -1120,8 +1120,20 @@ struct acpi_madt_multiproc_wakeup {
    u16 version;
    u32 reserved;       /* reserved - must be zero */
    u64 mailbox_address;
+   u64 reset_vector;
 };
```
* 增加复位向量支持的 Multiprocessor Wakeup Structure 版本为 `1`，增加 `enum acpi_madt_multiproc_wakeup_version` 枚举支持的 Multiprocessor Wakeup Structure 的版本值
```cpp
/* Values for Version field above */

enum acpi_madt_multiproc_wakeup_version {
   ACPI_MADT_MP_WAKEUP_VERSION_NONE = 0,
   ACPI_MADT_MP_WAKEUP_VERSION_V1 = 1,
   ACPI_MADT_MP_WAKEUP_VERSION_RESERVED = 2, /* 2 and greater are reserved */
};
```
* 增加复位向量支持的 Multiprocessor Wakeup Structure 版本 `1` 增加了 `ResetVector` 域，因此该结构大小变大了。增加描述 Multiprocessor Wakeup Structure 大小的宏
```cpp
#define ACPI_MADT_MP_WAKEUP_SIZE_V0    16
#define ACPI_MADT_MP_WAKEUP_SIZE_V1    24
```
* 版本 1 的 Multiprocessor Wakeup Mailbox Structure 增加新的 `TEST` 命令，增加了代表该命令的宏
```diff
@@ -1134,7 +1146,8 @@ struct acpi_madt_multiproc_wakeup_mailbox {
    u8 reserved_firmware[ACPI_MULTIPROC_WAKEUP_MB_FIRMWARE_SIZE];   /* reserved for firmware use */
 };

-#define ACPI_MP_WAKE_COMMAND_WAKEUP    1
+#define ACPI_MP_WAKE_COMMAND_WAKEUP    1
+#define ACPI_MP_WAKE_COMMAND_TEST  2

 /* 17: CPU Core Interrupt Controller (ACPI 6.5) */
``` 
### 新增 madt_playdead.S 文件
* 新增 arch/x86/kernel/acpi/madt_playdead.S 文件，该文件目前仅定义了 `asm_acpi_mp_play_dead` 一个例程
* `asm_acpi_mp_play_dead` 用于把 CPU 的控制权交给 BIOS
  * `rdi`：第一个参数，ACPI MADT Multiprocessor Wakeup Structure 中的复位向量（reset vector）的物理地址（其实用的是恒等映射）
  * `rsi`：第二个参数，恒等映射的 `PGD`
```cpp
#include <linux/linkage.h>
#include <asm/nospec-branch.h>
#include <asm/page_types.h>
#include <asm/processor-flags.h>

    .text
    .align PAGE_SIZE

/*
 * asm_acpi_mp_play_dead() - Hand over control of the CPU to the BIOS
 *
 * rdi: Address of the ACPI MADT MPWK ResetVector
 * rsi: PGD of the identity mapping
 */
SYM_FUNC_START(asm_acpi_mp_play_dead)
    /* Turn off global entries. Following CR3 write will flush them. */
    movq    %cr4, %rdx            //先将 CR4 的值取出
    andq    $~(X86_CR4_PGE), %rdx //将全局页的位清零，因为标识为全局页的页表条目在切换 CR3 时也不会被从 TLB 中清除
    movq    %rdx, %cr4            //将值写回 CR4 关闭全局页
    //改用恒等映射的页表，所以上面要关闭全局页以防止 TLB 中残留旧页表中的全局页表条目
    /* Switch to identity mapping */
    movq    %rsi, %cr3
    //跳转到复位向量，系统/CPU 进入假死状态，控制权交给 BIOS
    /* Jump to reset vector */
    ANNOTATE_RETPOLINE_SAFE
    jmp *%rdi
SYM_FUNC_END(asm_acpi_mp_play_dead)
```
* 修改 arch/x86/kernel/acpi/Makefile，新增对 arch/x86/kernel/acpi/madt_playdead.S 文件的编译
```diff
diff --git a/arch/x86/kernel/acpi/Makefile b/arch/x86/kernel/acpi/Makefile
index 8c7329c88a75..37b1f28846de 100644
--- a/arch/x86/kernel/acpi/Makefile
+++ b/arch/x86/kernel/acpi/Makefile
@@ -4,7 +4,7 @@ obj-$(CONFIG_ACPI)          += boot.o
 obj-$(CONFIG_ACPI_SLEEP)       += sleep.o wakeup_$(BITS).o
 obj-$(CONFIG_ACPI_APEI)            += apei.o
 obj-$(CONFIG_ACPI_CPPC_LIB)        += cppc.o
-obj-$(CONFIG_X86_ACPI_MADT_WAKEUP) += madt_wakeup.o
+obj-$(CONFIG_X86_ACPI_MADT_WAKEUP) += madt_wakeup.o madt_playdead.o

 ifneq ($(CONFIG_ACPI_PROCESSOR),)
 obj-y                  += cstate.o
```

### 新增的辅助函数
* 新增全局变量 `acpi_mp_pgd` 记录用于设置复位向量创建的恒等映射的 `PGD`
* 新增全局变量 `acpi_mp_reset_vector_paddr` 记录从 mailbox 中读到复位向量的物理地址
```cpp
static u64 acpi_mp_pgd __ro_after_init;
static u64 acpi_mp_reset_vector_paddr __ro_after_init;
```
* 新增两个假死函数，通过汇编例程 `asm_acpi_mp_play_dead` 把 CPU 的控制权交给 BIOS
  * `acpi_mp_play_dead() -> play_dead_common() -> local_irq_disable()` 会关闭本地中断
```cpp
static void acpi_mp_stop_this_cpu(void)
{
    asm_acpi_mp_play_dead(acpi_mp_reset_vector_paddr, acpi_mp_pgd);
}

static void acpi_mp_play_dead(void)
{
    play_dead_common();
    asm_acpi_mp_play_dead(acpi_mp_reset_vector_paddr, acpi_mp_pgd);
}
```
* 新增函数 `acpi_mp_cpu_die()` 用于检测指定 CPU 是否已被 BIOS 接管，逻辑如下：
1. 得到 CPU 的物理 ID
2. 将物理 ID 写入 Multiprocessor Wakeup Mailbox Structure 的 `ApicId` 域，指定要检测的 CPU
3. 将 `ACPI_MP_WAKE_COMMAND_TEST` 写入 mailbox 的 `Command` 域，发送 `Test(1)` 命令让 BIOS 证明它已接管了 CPU
4. 循环读取 mailbox 的 `Command` 域 1 秒，等待 BIOS 将它写为 `Noop(0)` 表示 *已接管 CPU*
5. 如果超过 1 秒，打印错误信息，否则直接返回
```cpp
static void acpi_mp_cpu_die(unsigned int cpu)
{   //得到 CPU 的物理 ID，见 cpu_physical_id() 宏，注意和 cpu_acpi_id() 区分
    u32 apicid = per_cpu(x86_cpu_to_apicid, cpu);
    unsigned long timeout;
    //在宣称 CPU 死亡之前，用 TEST mailbox 命令让 BIOS 证明它已接管了 CPU。证明的方式是 BIOS 把命令清零，即 Noop
    /*
     * Use TEST mailbox command to prove that BIOS got control over
     * the CPU before declaring it dead.
     *
     * BIOS has to clear 'command' field of the mailbox.
     */
    acpi_mp_wake_mailbox->apic_id = apicid;
    smp_store_release(&acpi_mp_wake_mailbox->command,
              ACPI_MP_WAKE_COMMAND_TEST);
    //不要等待超过 1 秒
    /* Don't wait longer than a second. */
    timeout = USEC_PER_SEC;
    while (READ_ONCE(acpi_mp_wake_mailbox->command) && --timeout)
        udelay(1);

    if (!timeout)
        pr_err("Failed to hand over CPU %d to BIOS\n", cpu);
}
```
* 新增内存分配函数 `alloc_pgt_page()` 从 boot memory block 中分配一个 `4KB` 的页，对齐到 `4KB`，返回其虚拟地址
```cpp
/* The argument is required to match type of x86_mapping_info::alloc_pgt_page */
static void __init *alloc_pgt_page(void *dummy)
{
    return memblock_alloc(PAGE_SIZE, PAGE_SIZE);
}
```
* 新增内存释放函数 `free_pgt_page()`，将之前从 boot memory 中分配的内存释放，地址为 memory block 的起始地址
```cpp
static void __init free_pgt_page(void *pgt, void *dummy)
{
    return memblock_free(pgt, PAGE_SIZE);
}
```
* `init_transition_pgtable()` 的代码和 arch/x86/kernel/machine_kexec_64.c:`init_transition_pgtable(struct kimage *image, pgd_t *pgd)` 类似
  * 确保 `asm_acpi_mp_play_dead()` 存在于恒等映射中与内核页表中相同的位置。
  * `asm_acpi_mp_play_dead()` 切换到恒等映射，并且 **该函数在切换页表之前和之后出现在虚拟地址空间中的同一位置**。
* 这里的关键在于，`asm_acpi_mp_play_dead()` 作为从线性映射到恒等映射的过渡代码，需要在线性映射的页表和恒等映射的页表中 **都有映射**
  * 因为 `$RIP` 在执行完切换 `CR3` 的指令后并不会立即跳变成低地址，往恒等映射的低地址跳变是通过跳转指令（`jmp`，`ret`，`iret` 等）完成的，在此之前 `$RIP` 还是高地址。
    * 从 *切换完 `CR3`* 到 *跳转指令* 之间的一小段指令需要在 *恒等映射页表* 中做出 *不恒等映射*
  * 映射的目的物理地址无所谓，只要能让 CPU 能够平滑地取到指令就行
    * 在用 `asm_acpi_mp_play_dead()` 假死这个场景，它被恒等映射页表映射到 `__pa(asm_acpi_mp_play_dead)` 这个地址，因此切换前后的取指令的物理地址是一样的
    * 在用 `relocate_kernel` kexec 这个场景，切换前的代码被复制了一份到切换后的物理地址（control page 空间），并且恒等映射页表映射到了新物理地址，因此切换前后取指令的物理地址是 **不一样** 的
```cpp
/*
 * Make sure asm_acpi_mp_play_dead() is present in the identity mapping at
 * the same place as in the kernel page tables. asm_acpi_mp_play_dead() switches
 * to the identity mapping and the function has be present at the same spot in
 * the virtual address space before and after switching page tables.
 */
static int __init init_transition_pgtable(pgd_t *pgd)
{
    pgprot_t prot = PAGE_KERNEL_EXEC_NOENC;
    unsigned long vaddr, paddr;
    p4d_t *p4d;
    pud_t *pud;
    pmd_t *pmd;
    pte_t *pte;
    //得到假死例程的虚拟地址，作为走表时的索引
    vaddr = (unsigned long)asm_acpi_mp_play_dead;
    pgd += pgd_index(vaddr);  //得到恒等映射页表 PGD 一级的页表条目的指针
    if (!pgd_present(*pgd)) { //解引用 pgd，如果 pgd 一级的条目不存在
        p4d = (p4d_t *)alloc_pgt_page(NULL); //分配 p4d 的页表页
        if (!p4d)             //如果分配失败
            return -ENOMEM;   //返回无内存错误
        set_pgd(pgd, __pgd(__pa(p4d) | _KERNPG_TABLE)); //设置 pgd 条目的内容为 p4d 页表页的物理地址
    }
    p4d = p4d_offset(pgd, vaddr); //到这 pgd 一级的条目已存在，得到 p4d 一级条目的指针
    if (!p4d_present(*p4d)) { //解引用 p4d，如果 p4d 一级的条目不存在
        pud = (pud_t *)alloc_pgt_page(NULL); //分配 pud 的页表页
        if (!pud)
            return -ENOMEM;
        set_p4d(p4d, __p4d(__pa(pud) | _KERNPG_TABLE)); //设置 p4d 条目的内容为 pud 页表页的物理地址
    }
    pud = pud_offset(p4d, vaddr); //到这 p4d 一级的条目已存在，得到 pud 一级条目的指针
    if (!pud_present(*pud)) { //解引用 pud，如果 pud 一级的条目不存在
        pmd = (pmd_t *)alloc_pgt_page(NULL); //分配 pmd 的页表页
        if (!pmd)
            return -ENOMEM;
        set_pud(pud, __pud(__pa(pmd) | _KERNPG_TABLE)); //设置 pud 条目的内容为 pmd 页表页的物理地址
    }
    pmd = pmd_offset(pud, vaddr); //到这 pud 一级的条目已存在，得到 pmd 一级条目的指针
    if (!pmd_present(*pmd)) { //解引用 pud，如果 pud 一级的条目不存在
        pte = (pte_t *)alloc_pgt_page(NULL); //分配 pte 的页表页
        if (!pte)
            return -ENOMEM;
        set_pmd(pmd, __pmd(__pa(pte) | _KERNPG_TABLE)); //设置 pmd 条目的内容为 pte 页表页的物理地址
    }
    pte = pte_offset_kernel(pmd, vaddr); //到这 pmd 一级的条目已存在，得到 pte 一级条目的指针
    //假死例程的虚拟地址转为物理地址，切换前后的物理地址是一样的
    paddr = __pa(vaddr);
    set_pte(pte, pfn_pte(paddr >> PAGE_SHIFT, prot)); //设置 pte 条目的内容为假死例程的物理地址所在的物理页

    return 0;
}
```
* 指定分配恒等映射内存所用的分配函数，因此建立恒等映射的内存都是从 boot memory 中分配的，而不是从 buddy system 中分配的
```cpp
static int __init acpi_mp_setup_reset(u64 reset_vector)
{
    struct x86_mapping_info info = {
        .alloc_pgt_page = alloc_pgt_page, //分配恒等映射内存所用的分配函数
        .free_pgt_page  = free_pgt_page,  //释放恒等映射内存所用的释放函数
        .page_flag      = __PAGE_KERNEL_LARGE_EXEC,
        .kernpg_flag    = _KERNPG_TABLE_NOENC,
    };
    pgd_t *pgd;
    //分配恒等映射的顶级页表页，即 pgd 页表页
    pgd = alloc_pgt_page(NULL);
    if (!pgd)
        return -ENOMEM;
    //给 E820 分配的内存在恒等映射页表中建立映射
    for (int i = 0; i < nr_pfn_mapped; i++) {
        unsigned long mstart, mend;

        mstart = pfn_mapped[i].start << PAGE_SHIFT;
        mend   = pfn_mapped[i].end << PAGE_SHIFT;
        if (kernel_ident_mapping_init(&info, pgd, mstart, mend)) {
            kernel_ident_mapping_free(&info, pgd);
            return -ENOMEM;
        }
    }
    //给复位向量在恒等映射页表中建立映射
    if (kernel_ident_mapping_init(&info, pgd,
                      PAGE_ALIGN_DOWN(reset_vector),
                      PAGE_ALIGN(reset_vector + 1))) {
        kernel_ident_mapping_free(&info, pgd);
        return -ENOMEM;
    }
    //给假死例程在恒等映射页表中建立映射
    if (init_transition_pgtable(pgd)) {
        kernel_ident_mapping_free(&info, pgd);
        return -ENOMEM;
    }
    //设置 SMP 的几个 operations 
    smp_ops.play_dead = acpi_mp_play_dead;         //offlied 的 CPU 假死时用到
    smp_ops.stop_this_cpu = acpi_mp_stop_this_cpu; //快速切换和 crash 时用到
    smp_ops.cpu_die = acpi_mp_cpu_die;             //BSP 检测 AP 是否已被移交
    //记录复位向量和恒等映射页表的物理地址给它们准备好的全局变量
    acpi_mp_reset_vector_paddr = reset_vector;
    acpi_mp_pgd = __pa(pgd);

    return 0;
}
```
* 在 BSP 启动时解析 MP 唤醒结构时，如果用的唤醒结构是 V1，则建立将来 kexec 切换时用到的恒等映射，并且禁止 CPU offline
* 无法使用标准 `BAD_MADT_ENTRY()` 来健全性检查 `mp_wake` 条目。`sizeof (struct acpi_madt_multiproc_wakeup)` 可以大于 ACPI 表中 MP 唤醒条目的实际大小，因为 `reset_vector` 仅在 V1 MP 唤醒结构中可用。
```diff
@@ -97,14 +254,37 @@ int __init acpi_parse_mp_wake(union acpi_subtable_headers *header,
    struct acpi_madt_multiproc_wakeup *mp_wake;

    mp_wake = (struct acpi_madt_multiproc_wakeup *)header;
-   if (BAD_MADT_ENTRY(mp_wake, end))
+
+   /*
+    * Cannot use the standard BAD_MADT_ENTRY() to sanity check the @mp_wake
+    * entry.  'sizeof (struct acpi_madt_multiproc_wakeup)' can be larger
+    * than the actual size of the MP wakeup entry in ACPI table because the
+    * 'reset_vector' is only available in the V1 MP wakeup structure.
+    */
+   if (!mp_wake)
+       return -EINVAL;
+   if (end - (unsigned long)mp_wake < ACPI_MADT_MP_WAKEUP_SIZE_V0)
+       return -EINVAL;
+   if (mp_wake->header.length < ACPI_MADT_MP_WAKEUP_SIZE_V0)
        return -EINVAL;

    acpi_table_print_madt_entry(&header->common);

    acpi_mp_wake_mailbox_paddr = mp_wake->mailbox_address;

-   acpi_mp_disable_offlining(mp_wake);
+   if (mp_wake->version >= ACPI_MADT_MP_WAKEUP_VERSION_V1 &&
+       mp_wake->header.length >= ACPI_MADT_MP_WAKEUP_SIZE_V1) {
+       if (acpi_mp_setup_reset(mp_wake->reset_vector)) {
+           pr_warn("Failed to setup MADT reset vector\n");
+           acpi_mp_disable_offlining(mp_wake);
+       }
+   } else {
+       /*
+        * CPU offlining requires version 1 of the ACPI MADT wakeup
+        * structure.
+        */
+       acpi_mp_disable_offlining(mp_wake);
+   }

    apic_update_callback(wakeup_secondary_cpu_64, acpi_wakeup_cpu);
```

## [PATCHv11 19/19] ACPI: tables: Print MULTIPROC_WAKEUP when MADT is parsed
* 解析 MADT 时，打印 `MULTIPROC_WAKEUP` 信息：
  `ACPI: MP Wakeup (version[1], mailbox[0x7fffd000], reset[0x7fffe068])`
* 此调试信息在启动过程中将非常有帮助。
---
* 这里打印的是 ACPI subtable 中的数据
  * drivers/acpi/tables.c
```cpp
void acpi_table_print_madt_entry(struct acpi_subtable_header *header)
{
...
   case ACPI_MADT_TYPE_MULTIPROC_WAKEUP:
       {
           struct acpi_madt_multiproc_wakeup *p =
               (struct acpi_madt_multiproc_wakeup *)header;
           u64 reset_vector = 0;

           if (p->version >= ACPI_MADT_MP_WAKEUP_VERSION_V1)
               reset_vector = p->reset_vector;

           pr_debug("MP Wakeup (version[%d], mailbox[%#llx], reset[%#llx])\n",
                p->version, p->mailbox_address, reset_vector);
       }
       break;
...
}
```

## 继续探索

### 到 `acpi_mp_stop_this_cpu()` 的路径
* `kexec -e` 快速切换的时候，执行 CPU 会通过发送 `IPI(REBOOT_VECTOR)` 和 `NMI_LOCAL` NMI 两种方式让其他 CPU 停止运行
* 下面就是一个空闲 CPU 收到 kexec 执行 CPU 发过来的 `IPI(REBOOT_VECTOR)` 的 call trace
```c
RIP: 0010:acpi_mp_stop_this_cpu+0x5/0x20
Call Trace:
 <IRQ>
 stop_this_cpu+0x7a/0x80
 __sysvec_reboot+0x56/0x60
 sysvec_reboot+0x7b/0xb0
 </IRQ>
 <TASK>
 asm_sysvec_reboot+0x16/0x20
RIP: 0010:__tdx_hypercall+0x34/0x80
 __trace_tdx_hypercall+0x2e/0x1c0
 ? timer_clear_idle+0x12/0x30
 tdx_safe_halt+0x2c/0x60
 default_idle_call+0x48/0x110
 do_idle+0x1fc/0x2a0
 cpu_startup_entry+0x19/0x20
 start_secondary+0x12e/0x130
 secondary_startup_64_no_verify+0xe0/0xeb
 </TASK>
```
* 对于 crash 场景，则是 `crash_nmi_callback()` 调用的 `acpi_mp_stop_this_cpu()`
* 在这两个场景下，发起 CPU 是不需要为 AP 维护 mailbox 重置环境的，因此不需要调用 `acpi_mp_cpu_die()` 去给 AP 的固件发 `TEST` 命令。重启后给各 AP 发 `Wakeup` 命令唤醒即可。

### 到 `acpi_mp_cpu_die()` 的路径
* 这个路径出现在 CPU offline 的场景，比如 `echo 0 > /sys/devices/system/cpu/cpu1/online`

#### 发起下线操作的 CPU
* `_cpu_down()` 会调用 `cpuhp_kick_ap()` 函数将 per-CPU 的热插拔内核线程 `cpuhp/%u` 唤醒，去完成热插拔状态第一阶段的转换，其中就包含将 offline CPU 上所有进程迁移走的步骤，这样该 CPU 上就只剩下绑核的 stop machine 进程了
```cpp
(gdb) bt
#0  sched_cpu_deactivate (cpu=3) at kernel/sched/core.c:9560
#1  0xffffffff81140b39 in cpuhp_invoke_callback (cpu=cpu@entry=3, state=CPUHP_AP_ACTIVE, bringup=bringup@entry=false,
    node=0x0 <fixed_percpu_data>, lastp=0xff1100017bb9b988) at kernel/cpu.c:192
#2  0xffffffff81141b6f in cpuhp_thread_fun (cpu=3) at kernel/cpu.c:825
#3  0xffffffff81175aa3 in smpboot_thread_fn (data=0xff11000100073210) at kernel/smpboot.c:164
#4  0xffffffff8116b066 in kthread (_create=0xff110001005aab80) at kernel/kthread.c:376
#5  0xffffffff81006ff9 in ret_from_fork () at arch/x86/entry/entry_64.S:311
#6  0x0000000000000000 in ?? ()
```
* **注意**：`takedown_cpu()` 在调用 `__cpu_die (cpu=1) -> acpi_mp_cpu_die(cpu=1)` 去给 AP 固件发送 `TEST` 命令之前其实调用了 `stop_machine_cpuslocked(take_cpu_down, NULL, cpumask_of(cpu))`
  * 该函数给 offline CPU 安排了一个 stop machine task `migration/%u`，该 task 会运行函数 `take_cpu_down()`
```cpp
(gdb) bt
#0  take_cpu_down (_param=0x0 <fixed_percpu_data>) at kernel/cpu.c:1038
#1  0xffffffff8123761e in multi_cpu_stop (data=0xffa0000000b53c98) at kernel/stop_machine.c:220
#2  0xffffffff812370e2 in cpu_stopper_thread (cpu=<optimized out>) at kernel/stop_machine.c:545
#3  0xffffffff81175aa3 in smpboot_thread_fn (data=0xff110001000731b0) at kernel/smpboot.c:164
#4  0xffffffff8116b066 in kthread (_create=0xff110001005aa2c0) at kernel/kthread.c:376
#5  0xffffffff81006ff9 in ret_from_fork () at arch/x86/entry/entry_64.S:311
#6  0x0000000000000000 in ?? ()
```
* 该函数最后会调用 `stop_machine_park(cpu)` 让 `migration/%u` 下线，被下线 CPU 上就只剩下 idle task。
* 被下线 CPU 上的 idle task 会调用 `smp_ops.play_dead()` 在 `acpi_mp_setup_reset()` 的时候被配置为 `acpi_mp_play_dead()` 的回调函数进入假死状态
* 最后，发起 CPU offline 的 CPU 调用 `acpi_mp_cpu_die()` 收割状态，看到的 call trace 是这样的
```cpp
(gdb) bt
#0  acpi_mp_cpu_die (cpu=1) at arch/x86/kernel/acpi/madt_wakeup.c:39
#1  0xffffffff81140217 in __cpu_die (cpu=1) at arch/x86/include/asm/smp.h:94
#2  takedown_cpu (cpu=1) at kernel/cpu.c:1110
#3  0xffffffff81140b39 in cpuhp_invoke_callback (cpu=cpu@entry=1, state=CPUHP_TEARDOWN_CPU, bringup=bringup@entry=false,
    node=node@entry=0x0 <fixed_percpu_data>, lastp=lastp@entry=0x0 <fixed_percpu_data>) at kernel/cpu.c:192
#4  0xffffffff81140fa9 in __cpuhp_invoke_callback_range (bringup=bringup@entry=false, cpu=cpu@entry=1, st=st@entry=0xff1100017ba9b960,
    target=target@entry=CPUHP_OFFLINE, nofail=nofail@entry=false) at kernel/cpu.c:688
#5  0xffffffff824491a8 in cpuhp_invoke_callback_range (target=CPUHP_OFFLINE, st=0xff1100017ba9b960, cpu=1, bringup=false)
    at kernel/cpu.c:1206
#6  cpuhp_down_callbacks (target=CPUHP_OFFLINE, st=0xff1100017ba9b960, cpu=1) at kernel/cpu.c:1145
#7  _cpu_down (cpu=cpu@entry=1, tasks_frozen=tasks_frozen@entry=0, target=target@entry=CPUHP_OFFLINE) at kernel/cpu.c:1206
#8  0xffffffff81142065 in cpu_down_maps_locked (target=CPUHP_OFFLINE, cpu=1) at kernel/cpu.c:1238
#9  cpu_down (cpu=1, target=CPUHP_OFFLINE) at kernel/cpu.c:1246
    cpu_subsys_offline() at drivers/base/core.c:4039
#10 0xffffffff81c7e2c6 in device_offline (dev=0xff1100017ba9a928) at drivers/base/core.c:4039
#11 device_offline (dev=0xff1100017ba9a928) at drivers/base/core.c:4023
#12 0xffffffff81c7e3da in online_store (dev=0xff1100017ba9a928, attr=<optimized out>, buf=<optimized out>, count=2)
    at drivers/base/core.c:2547
#13 0xffffffff814d61ab in kernfs_fop_write_iter (iocb=0xffa000000066bea0, iter=<optimized out>) at fs/kernfs/file.c:334
#14 0xffffffff81420995 in call_write_iter (file=0xff110001082d0140, iter=0x0 <fixed_percpu_data>, kio=0x1 <fixed_percpu_data+1>)
    at include/linux/fs.h:2189
#15 new_sync_write (ppos=0xffa000000066bf08, len=2,
    buf=0x55b1e8001ce0 "0\n0;root@td-guest:/sys/devices/system/cpu/cpu1\avices/cpu\a \"$COMP_WORDBREAKS\" == *:* ]]; then\n        local colo
n_word=${1%\"${1##*:}\"};\n        local i=${#COMPREPLY[*]};\n        while [[ $((--i)) -ge 0"..., filp=0xff110001082d0140)
    at fs/read_write.c:491
#16 vfs_write (pos=0xffa000000066bf08, count=2,
    buf=0x55b1e8001ce0 "0\n0;root@td-guest:/sys/devices/system/cpu/cpu1\avices/cpu\a \"$COMP_WORDBREAKS\" == *:* ]]; then\n        local colo
n_word=${1%\"${1##*:}\"};\n        local i=${#COMPREPLY[*]};\n        while [[ $((--i)) -ge 0"..., file=0xff110001082d0140)
    at fs/read_write.c:584
#17 vfs_write (file=0xff110001082d0140,
    buf=0x55b1e8001ce0 "0\n0;root@td-guest:/sys/devices/system/cpu/cpu1\avices/cpu\a \"$COMP_WORDBREAKS\" == *:* ]]; then\n        local colo
n_word=${1%\"${1##*:}\"};\n        local i=${#COMPREPLY[*]};\n        while [[ $((--i)) -ge 0"..., count=<optimized out>,
    pos=0xffa000000066bf08) at fs/read_write.c:564
#18 0xffffffff81420c6c in ksys_write (fd=<optimized out>,
    buf=0x55b1e8001ce0 "0\n0;root@td-guest:/sys/devices/system/cpu/cpu1\avices/cpu\a \"$COMP_WORDBREAKS\" == *:* ]]; then\n        local colo
n_word=${1%\"${1##*:}\"};\n        local i=${#COMPREPLY[*]};\n        while [[ $((--i)) -ge 0"..., count=2)
    at fs/read_write.c:637
#19 0xffffffff8244339f in do_syscall_x64 (nr=<optimized out>, regs=0xffa000000066bf58) at arch/x86/entry/common.c:50
#20 do_syscall_64 (regs=0xffa000000066bf58, nr=<optimized out>) at arch/x86/entry/common.c:80
#21 0xffffffff826000aa in entry_SYSCALL_64 () at arch/x86/entry/entry_64.S:123

//以下是用户态的栈，gdb 没有符号解析不出来的
#22 0x0000000000000002 in fixed_percpu_data ()
#23 0x00007f7a9e7bb880 in ?? ()
#24 0x0000000000000002 in fixed_percpu_data ()
#25 0x00007f7a9e7c06e0 in ?? ()
#26 0x000055b1e8001ce0 in ?? ()
#27 0x0000000000000002 in fixed_percpu_data ()
#28 0x0000000000000246 in ?? ()
#29 0x000000000000000a in fixed_percpu_data ()
#30 0x00007f7a9e580820 in ?? ()
#31 0x000000000000000a in fixed_percpu_data ()
#32 0xffffffffffffffda in ?? ()
#33 0x00007f7a9e5205c8 in ?? ()
#34 0x0000000000000002 in fixed_percpu_data ()
#35 0x000055b1e8001ce0 in ?? ()
#36 0x0000000000000001 in fixed_percpu_data ()
#37 0x0000000000000001 in fixed_percpu_data ()
#38 0x00007f7a9e5205c8 in ?? ()
#39 0x0000000000000033 in ?? ()
#40 0x0000000000000246 in ?? ()
#41 0x00007ffc3abdb408 in ?? ()
#42 0x000000000000002b in fixed_percpu_data ()
Backtrace stopped: Cannot access memory at address 0xffa000000066c000
```

### 到 `acpi_mp_play_dead()` 的路径
#### 被下线的 CPU
* 上面说到被下线的 CPU 上只剩下 idle task，修改 `smp_ops.play_dead()` 让它进入假死状态
* `smp_ops.play_dead()` 在 `acpi_mp_setup_reset()` 的时候被配置为 `acpi_mp_play_dead()`
* AP 被 offline 后进入假死状态，call trace 如下：
```cpp
(gdb) bt
#0  acpi_mp_play_dead () at arch/x86/kernel/acpi/madt_wakeup.c:33
    smp_ops.play_dead() at arch/x86/include/asm/smp.h
    play_dead () at arch/x86/kernel/process.c
    arch_cpu_idle_dead () at kernel/sched/idle.c
#1  0xffffffff8119dd65 in do_idle () at kernel/sched/idle.c:287
#2  0xffffffff8119e039 in cpu_startup_entry (state=state@entry=CPUHP_AP_ONLINE_IDLE) at kernel/sched/idle.c:400
#3  0xffffffff8107864e in start_secondary (unused=<optimized out>) at arch/x86/kernel/smpboot.c:267
#4  0xffffffff81000145 in secondary_startup_64 () at arch/x86/kernel/head_64.S:358
#5  0x0000000000000000 in ?? ()
```

# TDVF 对应的修改
## Commit 列表
```lua
(HEAD -> TDVF, tag: tdvf-kexec, staging/TDVF)
* c229fca09ebc - UefiCpuPkg/MpInitLibUp: Update the ProcessorNumber (2024-01-02 03:31:20 -0500) <Ceping Sun>
* aaac8f2de726 - OvmfPkg: Add the ResetVector in TDX MailBox (2023-11-30 21:00:32 -0500) <Ceping Sun>
* 912c70be3281 - OvmfPkg: Add the Test command in TDX MailBox (2023-11-30 20:31:51 -0500) <Ceping Sun>
* 542faef78c50 - OvmfPkg/WorkArea.h: Add MAILBOX_GDT (2023-11-30 00:54:17 -0500) <Ceping Sun>
* cf6b53baf070 - MdePkg/Acpi64.h: Add ResetVector in Multiprocessor Wakeup Struct (2023-11-30 02:22:39 -0500) <Ceping Sun>
```
## 代码修改
* 给 Multiprocessor Wakeup Structure 的结构体 `EFI_ACPI_6_4_MULTIPROCESSOR_WAKEUP_STRUCTURE` 新增域 `ResetVector`
  * MdePkg/Include/IndustryStandard/Acpi64.h
```cpp
///
/// Multiprocessor Wakeup Structure
///
typedef struct {
  UINT8     Type;
  UINT8     Length;
  UINT16    MailBoxVersion;
  UINT32    Reserved;
  UINT64    MailBoxAddress;
  UINT64    ResetVector;
} EFI_ACPI_6_4_MULTIPROCESSOR_WAKEUP_STRUCTURE;
```
* 新增描述 `GDTR` 的结构体 `IA32_GDTR`
* 新增描述 GDT 条目的结构体 `IA32_GDT`
  * OvmfPkg/Include/WorkArea.h
```cpp
#pragma pack (1)
typedef struct {
  UINT16    Limit;
  UINTN     Base;
} IA32_GDTR;

typedef union {
  struct {
    UINT32    LimitLow    : 16;
    UINT32    BaseLow     : 16;
    UINT32    BaseMid     : 8;
    UINT32    Type        : 4;
    UINT32    System      : 1;
    UINT32    Dpl         : 2;
    UINT32    Present     : 1;
    UINT32    LimitHigh   : 4;
    UINT32    Software    : 1;
    UINT32    Reserved    : 1;
    UINT32    DefaultSize : 1;
    UINT32    Granularity : 1;
    UINT32    BaseHigh    : 8;
  } Bits;
  UINT64    Uint64;
} IA32_GDT;
#pragma pack()
```
* 新增 mailbox GDT 的描述结构 `MAILBOX_GDT`
* `MAILBOX_GDT_SIZE` 定义了 mailbox GDT 条目有 `5` 个
```cpp
#define MAILBOX_GDT_SIZE (sizeof(IA32_GDT) * 5)

typedef struct _MAILBOX_GDT {
  IA32_GDTR          Gdtr;
  UINT8              Data[MAILBOX_GDT_SIZE];
} MAILBOX_GDT;
```
* 在 `TDX_WORK_AREA` 结构中新增 `MAILBOX_GDT MailboxGdt` 域
```cpp
typedef struct _TDX_WORK_AREA {
  CONFIDENTIAL_COMPUTING_WORK_AREA_HEADER    Header;
  SEC_TDX_WORK_AREA                          SecTdxWorkArea;
  MAILBOX_GDT                                MailboxGdt;
} TDX_WORK_AREA;
```
* 新增数组 `mGdtEntries[5]` 用于存储 mailbox GDT 的 `5` 个条目
  * edk2/OvmfPkg/TdxDxe/TdxAcpiTable.c
```cpp
IA32_GDT  mGdtEntries[] = {
  {
    { 0,      0, 0, 0,   0, 0, 0, 0,   0, 0, 0, 0, 0 }
  },                                                            /* 0x0:  reserve */
  { //第 10 个元素 .Reserved 其实对应到 CS.L，0 表示该代码段中的指令以 compatibility mode 执行
    { 0xFFFF, 0, 0, 0xB, 1, 0, 1, 0xF, 0, 0, 1, 1, 0 }
  },                                                            /* 0x8:  compatibility mode */
  { //第 10 个元素 .Reserved 其实对应到 CS.L，1 表示该代码段中的指令以 64-bit mode 执行
    { 0xFFFF, 0, 0, 0xB, 1, 0, 1, 0xF, 0, 1, 0, 1, 0 }
  },                                                            /* 0x10: for long mode */
  {
    { 0xFFFF, 0, 0, 0x3, 1, 0, 1, 0xF, 0, 0, 1, 1, 0 }
  },                                                            /* 0x18: data */
  {
    { 0,      0, 0, 0,   0, 0, 0, 0,   0, 0, 0, 0, 0 }
  },                                                            /* 0x20: reserve */
};
```
* 该 GDT 表，以便操作系统跳转到 Mailbox 中的 `ResetVector` 时切换分页模式。
* 新增函数 `SetMailboxResetVectorGDT()` 设置 mailbox 的 `ResetVector` 的 GDT 条目
  * 首先得到全局的 `TDX_WORK_AREA PcdOvmfWorkAreaBase` 的基地址
  * 清空 `MAILBOX_GDT MailboxGdt` 域中的数据
  * 填充 `MAILBOX_GDT MailboxGdt` 域中 `Data[]` 数组域的为数组 `mGdtEntries[5]`
  * 填充 `MAILBOX_GDT MailboxGdt` 域中 `IA32_GDTR Gdtr` 域的 `.Base` 和 `.Limit` 域
  * edk2/OvmfPkg/TdxDxe/TdxAcpiTable.c
```cpp
/**
  At the beginning of ResetVector in OS, the GDT needs to be reloaded.
**/
VOID
SetMailboxResetVectorGDT (
 VOID
 )
{
  TDX_WORK_AREA  *TdxWorkArea;

  TdxWorkArea = (TDX_WORK_AREA *)(UINTN)FixedPcdGet32 (PcdOvmfWorkAreaBase);
  ASSERT (TdxWorkArea != NULL);
  ZeroMem ((VOID *)TdxWorkArea->MailboxGdt.Data, sizeof (TdxWorkArea->MailboxGdt.Data));

  CopyMem((VOID *)TdxWorkArea->MailboxGdt.Data, (VOID *)mGdtEntries, sizeof(mGdtEntries));
  TdxWorkArea->MailboxGdt.Gdtr.Base = (UINTN)TdxWorkArea->MailboxGdt.Data;
  TdxWorkArea->MailboxGdt.Gdtr.Limit  = sizeof (mGdtEntries) - 1;
}
```
* 回忆 ACPI Spec 6.6 的修改
> AP 跳转到复位地址后，操作系统需要通过向 mailbox 发送 `test` 命令来验证 mailbox 是否响应命令。当它通过将命令更改为 `noop` 进行响应时，操作系统就不再需要为给定的 AP 维护 mailbox 重置环境了

* 增加 TDVF 固件中对 `Test` 命令的支持
```diff
diff --git a/OvmfPkg/Include/IndustryStandard/IntelTdx.h b/OvmfPkg/Include/IndustryStandard/IntelTdx.h
index cc849be2fb59..fc383468489d 100644
--- a/OvmfPkg/Include/IndustryStandard/IntelTdx.h
+++ b/OvmfPkg/Include/IndustryStandard/IntelTdx.h
@@ -20,8 +20,9 @@
 typedef enum {
   MpProtectedModeWakeupCommandNoop        = 0,
   MpProtectedModeWakeupCommandWakeup      = 1,
-  MpProtectedModeWakeupCommandSleep       = 2,
-  MpProtectedModeWakeupCommandAcceptPages = 3,
+  MpProtectedModeWakeupCommandTest        = 2,
+  MpProtectedModeWakeupCommandSleep       = 3,
+  MpProtectedModeWakeupCommandAcceptPages = 4,
 } MP_CPU_PROTECTED_MODE_WAKEUP_CMD;
 
 #pragma pack(1)
diff --git a/OvmfPkg/Include/TdxCommondefs.inc b/OvmfPkg/Include/TdxCommondefs.inc
index a29d2fad4233..200fa405731e 100644
--- a/OvmfPkg/Include/TdxCommondefs.inc
+++ b/OvmfPkg/Include/TdxCommondefs.inc
@@ -41,8 +41,9 @@ ERROR_INVALID_FALLBACK_PAGE_LEVEL         equ       3
 
 MpProtectedModeWakeupCommandNoop          equ       0
 MpProtectedModeWakeupCommandWakeup        equ       1
-MpProtectedModeWakeupCommandSleep         equ       2
-MpProtectedModeWakeupCommandAcceptPages   equ       3
+MpProtectedModeWakeupCommandTest          equ       2
+MpProtectedModeWakeupCommandSleep         equ       3
+MpProtectedModeWakeupCommandAcceptPages   equ       4
 
 MailboxApicIdInvalid                      equ       0xffffffff
 MailboxApicidBroadcast                    equ       0xfffffffe
```
* 修改的 edk2/OvmfPkg/TdxDxe/X64/ApRunLoop.nasm 是 guest TD 中 AP 运行循环的汇编代码
```asm
%include "TdxCommondefs.inc"

DEFAULT REL

SECTION .text

BITS 64

%define TDVMCALL_EXPOSE_REGS_MASK       0xffec
%define TDVMCALL                        0x0
%define EXIT_REASON_CPUID               0xa

%macro  tdcall  0
  db  0x66, 0x0f, 0x01, 0xcc
%endmacro
; 重定位 AP mailbox 循环
;
; Relocated Ap Mailbox loop
;
; @param[in]  RBX:  Relocated mailbox address
; @param[in]  RBP:  vCpuId
;
; @return     None  This routine does not return
;
global ASM_PFX(AsmRelocateApMailBoxLoop)
ASM_PFX(AsmRelocateApMailBoxLoop):
AsmRelocateApMailBoxLoopStart:

    mov         rax, TDVMCALL ;准备发的 tdcall 是 TDG.VP.VMCALL，即 TDVMCALL
    mov         rcx, TDVMCALL_EXPOSE_REGS_MASK ;需要暴露（传递）给 VMM 的 TD guest GPRs 的位图
    xor         r10, r10      ;%r10 为 0 表示 TDVMCALL
    mov         r11, EXIT_REASON_CPUID ;TDG.VP.VMCALL<Instruction.CPUID>
    mov         r12, 0xb      ;%r12 对应到 cpuid 的 %eax 寄存器，这里填入 0xb，获取 x2APIC 信息
    tdcall                    ;发起 tdcall，TDG.VP.VMCALL<Instruction.CPUID>
    test        r10, r10      ;TDVMCALL leaf 返回码是放在 %r10
    jnz         Panic         ;TDVMCALL leaf 返回码不为 0 均作为 panic 处理
    mov         r8, r15       ;返回后 %r15 对应到 cpuid 的 %edx 寄存器，x2APIC ID，将该值传入 %r8
    mov         qword[rel mailbox_address], rbx ;把 %rbx 中 mailbox 地址写入标号为 mailbox_address 的内存

MailBoxLoop:
    ; Spin until command set
    cmp        dword [rbx + CommandOffset], MpProtectedModeWakeupCommandNoop   ;当前 command 域是 Noop(0) 吗？
    je         MailBoxLoop    ;是，回到上面 MailBoxLoop 的 label；不是，往下走
    ; Determine if this is a broadcast or directly for my apic-id, if not, ignore
    cmp        dword [rbx + ApicidOffset], MailboxApicidBroadcast ;APIC ID 域的值是广播吗？
    je         MailBoxProcessCommand ;是，直接跳到处理 mailbox 的命令；不是，往下走
    cmp        dword [rbx + ApicidOffset], r8d ;BSP 要唤醒的是 我吗？比较 APIC ID 域和我的 x2APIC ID
    jne        MailBoxLoop    ;不是，回到上面 MailBoxLoop 的 label；是，往下走
MailBoxProcessCommand:
    cmp        dword [rbx + CommandOffset], MpProtectedModeWakeupCommandWakeup ;当前 command 域是 Wakeup(1) 吗？
    je         MailBoxWakeUp  ;是，跳到唤醒 AP 的例程；不是，判断是不是下一个命令
    cmp        dword [rbx + CommandOffset], MpProtectedModeWakeupCommandTest   ;当前 command 域是 Test(2) 吗？
    je         MailBoxTest    ;是，跳到 Test 的例程；不是，判断是不是下一个命令
    cmp        dword [rbx + CommandOffset], MpProtectedModeWakeupCommandSleep  ;当前 command 域是 Sleep(3) 吗？
    je         MailBoxSleep   ;是，跳到 AP 睡眠的例程；不是，往下走
    ; Don't support this command, so ignore
    jmp        MailBoxLoop    ;对于不支持的命令，忽略，回到循环的起点
MailBoxWakeUp:
    mov        rax, [rbx + WakeupVectorOffset] ;WakeupVector 域是 AP 唤醒后第一条指令的地址，传入 %rax
    ; OS sends a wakeup command for a given APIC ID, firmware is supposed to reset
    ; the command field back to zero as acknowledgement.
    mov        qword [rbx + CommandOffset], 0  ;固件重置 command 域为零，表示固件 ACK 了 Wakeup(1) 命令
    jmp        rax            ;跳转到 AP 的启动地址，即 BSP 的内核提供的 trampoline_start64
MailBoxTest:
    mov        qword [rbx + CommandOffset], 0  ;固件重置 command 域为零，表示固件 ACK 了 Test(2) 命令
    jmp        MailBoxLoop    ;跳回到循环的起点，固件接管 AP 进入 mailbox 等待唤醒的循环
MailBoxSleep:
    jmp       $               ;$ 属于“隐式地”藏在本行前的标号，也就是当前安排的地址，每一行都有，jmp $ 即在此条指令循环
Panic:
    ud2                       ;用 ud 指令引发 panic
BITS 64
AsmRelocateApMailBoxLoopEnd:
```
* 在 `MP_RELOCATION_MAP` 结构中增加域 `UINT8 *RelocateApResetVector`，该结构用于在 C 和汇编之间共享 AP 的重定位代码信息
```cpp
//
// AP relocation code information including code address and size,
// this structure will be shared be C code and assembly code.
// It is natural aligned by design.
//
typedef struct {
  UINT8    *RelocateApLoopFuncAddress;
  UINTN    RelocateApLoopFuncSize;
  UINT8    *RelocateApResetVector;
} MP_RELOCATION_MAP;
```
* 汇编函数 `AsmGetRelocationMap(&RelocationMa)` 将重定位后的 AP 的 mailbox 循环例程的相关信息通过入参 `MP_RELOCATION_MAP &RelocationMap` 传递给 C 程序
  * 新增将 AP 重定位后的 `ResetVector` 的地址 `AsmRelocateApResetVector` 放入 `RelocateApResetVector` 域的操作
  * **注意**：这里用的 ABI 是 [x86-64 Microsoft x64 calling convention](https://en.wikipedia.org/wiki/X86_calling_conventions)，所以传递第一个参数 `&RelocationMap` 用的寄存器 `%rcx`
```asm
;-------------------------------------------------------------------------------------
;  AsmGetRelocationMap (&RelocationMap);
;-------------------------------------------------------------------------------------
global ASM_PFX(AsmGetRelocationMap)
ASM_PFX(AsmGetRelocationMap):
    ; mov        byte[TDX_WORK_AREA_MB_PGTBL_READY], 0
    lea        rax, [AsmRelocateApMailBoxLoopStart]
    mov        qword [rcx], rax ;AP mailbox 循环的起始地址放到 RelocateApLoopFuncAddress 域
    mov        qword [rcx +  8h], AsmRelocateApMailBoxLoopEnd - AsmRelocateApMailBoxLoopStart ;循环例程的大小放入 RelocateApLoopFuncSize 域
    lea        rax, [AsmRelocateApResetVector]
    mov        qword [rcx + 10h], rax ;AP ResetVector 的地址放入 RelocateApResetVector 域
    ret
```

* 对 `RelocateMailbox()` 进行修改，修改前的函数见 [TDVF](TDVF.md)
* 增加入参 `ResetVector` 指向复位向量
* 通过汇编例程 `AsmGetRelocationMap` 得到重定位后的 `MP_RELOCATION_MAP RelocationMap` 信息，从中提取出重定位后的复位向量例程地址 `RelocationMap.RelocateApResetVector`
* 将重定位后的复位向量地址转换为拷贝到 ACPI Nvs 内存后复位向量的地址，这部分内存 guest OS 可见，赋给传入的 `ResetVector`
```cpp
EFI_PHYSICAL_ADDRESS
EFIAPI
RelocateMailbox (
  EFI_PHYSICAL_ADDRESS *ResetVector
  )
{
...
  //通过汇编例程 AsmGetRelocationMap 得到重定位后的 MP_RELOCATION_MAP RelocationMap 信息
  AsmGetRelocationMap (&RelocationMap);
...
  //将重定位后的 AP mailbox 循环的字节码拷贝至新分配的 ACPI Nvs 内存的第二页中
  CopyMem (
    ApLoopFunc,
    RelocationMap.RelocateApLoopFuncAddress,
    RelocationMap.RelocateApLoopFuncSize
    );
...
  //设置 mailbox 的 ResetVector 的 GDT 条目
  SetMailboxResetVectorGDT();
...
  //BSP 通过 MpSendWakeupCommand() 唤醒 AP，结束在 SEC 阶段进入的第二次忙等
  MpSendWakeupCommand (
    MpProtectedModeWakeupCommandWakeup,
    (UINT64)ApLoopFunc,
    (UINT64)RelocatedMailBox,
    0,
    0,
    0
    );
  //给传入的复位向量赋值重定位后的复位向量例程拷贝到 ACPI Nvs 内存中的地址
  *ResetVector = (UINT64)ApLoopFunc + (RelocationMap.RelocateApResetVector -
                       RelocationMap.RelocateApLoopFuncAddress);
  DEBUG ((
    DEBUG_INFO,
    "Ap Relocation: reset_vector %llx\n",
    *ResetVector
    ));
  return Address;
}
```
* 对 `RelocateMailbox()` 的调用者 `AlterAcpiTable()` 进行修改
  * 增加本地变量 `RelocateResetVector`
  * 调用 `RelocateMailbox()` 的时候传入 `RelocateResetVector` 获得复位向量拷贝到 ACPI Nvs 内存后的地址
  * 填充新分配的，即将投入使用的 MADT Multiprocessor Wakeup Structure，其中
    * `MadtMpWk->MailBoxVersion = 1` ACPI spec v6.6 版本为 1
    * `MadtMpWk->ResetVector` 域设置为重定位后的复位向量地址 `RelocateResetVector`，供 OS 跳转到这里
```diff
@@ -142,6 +192,7 @@ AlterAcpiTable (
   UINT8                                                *NewMadtTable;
   UINTN                                                NewMadtTableLength;
   EFI_PHYSICAL_ADDRESS                                 RelocateMailboxAddress;
+  EFI_PHYSICAL_ADDRESS                                 RelocateResetVector;
   EFI_ACPI_6_4_MULTIPROCESSOR_WAKEUP_STRUCTURE         *MadtMpWk;
   EFI_ACPI_1_0_MULTIPLE_APIC_DESCRIPTION_TABLE_HEADER  *MadtHeader;

@@ -155,7 +206,7 @@ AlterAcpiTable (
     return;
   }

-  RelocateMailboxAddress = RelocateMailbox ();
+  RelocateMailboxAddress = RelocateMailbox (&RelocateResetVector);
   if (RelocateMailboxAddress == 0) {
     ASSERT (FALSE);
     DEBUG ((DEBUG_ERROR, "Failed to relocate Td mailbox\n"));
@@ -186,9 +237,10 @@ AlterAcpiTable (
       MadtMpWk                 = (EFI_ACPI_6_4_MULTIPROCESSOR_WAKEUP_STRUCTURE *)(NewMadtTable + Table->Length);
       MadtMpWk->Type           = EFI_ACPI_6_4_MULTIPROCESSOR_WAKEUP;
       MadtMpWk->Length         = sizeof (EFI_ACPI_6_4_MULTIPROCESSOR_WAKEUP_STRUCTURE);
-      MadtMpWk->MailBoxVersion = 0;
+      MadtMpWk->MailBoxVersion = 1;
       MadtMpWk->Reserved       = 0;
       MadtMpWk->MailBoxAddress = RelocateMailboxAddress;
+      MadtMpWk->ResetVector    = RelocateResetVector;

       Status = AcpiTableProtocol->InstallAcpiTable (AcpiTableProtocol, NewMadtTable, NewMadtTableLength, &NewTableKey);
       if (EFI_ERROR (Status)) {
```

### `AsmRelocateApResetVector` 例程
* 最后再来看看 `AsmRelocateApResetVector` 例程
* 进入：Guest vCPU AP offline 时需要最终跳到这里
* 退出：该例程需要跳到 `AsmRelocateApMailBoxLoopStart` 例程，让 AP 进入等待 mailbox 唤醒状态，并响应（ACK）BSP 的 `Test` 命令
* 终点：最终目的是 guest BSP 完成 kexec/kdump 切换后，还按照原有的多处理器 AP 唤醒流程把等待中的 AP 都唤醒
  * edk2/OvmfPkg/TdxDxe/X64/ApRunLoop.nasm
```asm
SECTION .bss
global STACK_BASE
STACK_BASE:
 resb 1024 ;保留 1024 字节作为栈
STACK_TOP:

SECTION .text
;MAILBOX_GDT MailboxGdt 域在全局的 TDX_WORK_AREA PcdOvmfWorkAreaBase 的基地址 + 128 偏移处
%define TDX_WORK_AREA_MAILBOX_GDTR   (FixedPcdGet32 (PcdOvmfWorkAreaBase) + 128)
;该宏得到 SEC 页表的基址 + 偏移
%define PT_ADDR(Offset)                 (FixedPcdGet32 (PcdOvmfSecPageTablesBase) + (Offset))
...
AsmRelocateApResetVector:
;准备栈
.prepareStack:
    ; The stack can then be used to switch from long mode to compatibility mode
    mov rsp, STACK_TOP         ;设置栈指针为保留的 STACK_TOP（高地址）
;加载为 Mailbox 准备的 GDT
.loadGDT:
    cli                        ;关闭中断
    mov      rax, TDX_WORK_AREA_MAILBOX_GDTR ;TDX_WORK_AREA.MailboxGdt 域放入 rax
    lgdt     [rax]             ;将 TDX_WORK_AREA.MailboxGdt.Gdtr 载入 GDTR
;加载 mode 切换的代码
.loadSwicthModeCode:
    ;0x10 的二进制表示为 0001_0000，用作段选择符的时候 bit 0-1 用作 RPL，bit 2 用作 TI，剩下的 bits 用作 GDT 中的索引
    ;回忆 long mode 的代码段在 GDT 表 mGdtEntries 的第二个条目，对应到 bit 3~4（二进制值 10）
    mov     rcx, dword 0x10    ; load long mode selector
    shl     rcx, 32            ;左移 32 位，用于长跳转时加载到 CS 段选择符部分，索引为 2
    lea     rdx, [LongMode]    ; assume address < 4G
    or      rcx, rdx           ;高 32 位为 CS 段选择符部分，低 32 位为 LongMode 例程在段中的偏移
    push    rcx                ;第一次压入 CS 段选择符 和 LongMode 例程段中偏移，为第二次调用 retf 准备
    ;0x08 的二进制表示为 0000_1000，用作段选择符的时候 bit 0-2 已说过，剩下的 bits 用作 GDT 中的索引
    ;回忆 Compatible mode 的代码段在 GDT 表 mGdtEntries 的第一个条目，对应到 bit 3~4（二进制值 01）
    mov     rcx, dword 0x08    ; load compatible mode selector
    shl     rcx, 32            ;左移 32 位，用于长跳转时加载到 CS 段选择符部分，索引为 1
    lea     rdx, [Compatible]  ; assume address < 4G
    or      rcx, rdx           ;高 32 位为 CS 段选择符部分，低 32 位为 Compatible 例程在段中的偏移
    push    rcx                ;第二次压入 CS 段选择符 和 Compatible 例程段中偏移，为第一次调用 retf 准备（就在下一条指令）
    retf    ;远返回指令，也叫段间返回。从栈顶弹出 Compatible 例程的偏移到 EIP 寄存器，再弹出段选择符到 CS 寄存器
;这里开始 32 位指令格式，CS.L = 0 该代码段中的指令以 compatibility mode 执行
BITS 32
Compatible:
    mov     eax, dword 0x18    ;0x18 的二进制表示为 0001_1000，bit 15~3 为 GDT 表的索引，故以下寄存器引用 mGdtEntries 的第三个条目
;     ; reload DS/ES/SS to make sure they are correct referred to current GDT
    mov     ds, ax
    mov     es, ax
    mov     ss, ax
    ; reload the fs and gs
    mov     fs, ax
    mov     gs, ax
    ;关闭分页前必须清除 CR4.PCIDE bit，即 CR4 的 bit 17
    ; Must clear the CR4.PCIDE before clearing paging
    mov     ecx, cr4
    btc     ecx, 17   ;Bit Test and Complement 第 17 位
    mov     cr4, ecx
    ;
    ; Disable paging
    ;关闭分页 CR0.PG（bit 31）设为 0
    mov     ecx, cr0
    btc     ecx, 31
    mov     cr0, ecx
    ;恢复 CR0，仅设置 Protection Enable(bit 0)、Numeric Error(bit 5)、Extension Type(bit 4) 几个位
RestoreCr0:
    ; Only enable  PE(bit 0), NE(bit 5), ET(bit 4) 0x31
    mov    eax, dword 0x31
    mov    cr0, eax

    ;恢复 CR4，仅启用 Machine-Check(bit 6)，TDX 强制 VMX-Enable(bit 13) = 1 且在 VMM 中 mask 它，所以不设置该位
    ; Only Enable MCE(bit 6), VMXE(bit 13) 0x2040
    ; TDX enforeced the VMXE = 1 and mask it in VMM, so not set it.
RestoreCr4:
    mov     eax, 0x40
    mov     cr4, eax
SetCr3:
    ;可以使用启动时的 SEC 页表，因为它保留下来了，把 0 偏移，即页表基址移入 CR3
    ; Can use the boot page tables since it's reserved

    mov     eax, PT_ADDR (0)
    mov     cr3, eax
;启用 PAE，Physical Address Extension 在 CR4 的第 5 位
EnablePAE:
    mov     eax, cr4
    bts     eax, 5
    mov     cr4, eax
;启用分页，在 CR0 的第 31 位
EnablePaging:
    mov     eax, cr0
    bts     eax, 31                     ; set PG
    mov     cr0, eax                    ; enable paging
    ; return to LongMode
    retf    ;之前第一个 push 压入了 CS 段选择符 和 LongMode 例程段中偏移，这里利用远返回更新 CS 寄存器和跳到 LongMode 例程
;这里开始 64 位指令格式，CS.L = 1 该代码段中的指令以 long mode 执行
BITS  64
LongMode:
    mov      rbx, qword[rel mailbox_address] ;把下面标号 mailbox_address 的内容放入 rbx 寄存器
    jmp      AsmRelocateApMailBoxLoopStart   ;跳转到 mailbox 循环的开始，完成 AP 在固件中的复位
align 16
mailbox_address:
    dq 0     ;Double-quadword，两个四字，128 bits，初始内容为 0。其实是给记录 mailbox 的地址预留一个空间
BITS 64
AsmRelocateApMailBoxLoopEnd:
```

#### 复位场景下如何再次得到 mailbox 地址？
* 在之前 kernel 代码分析的时候我们看到，AP 只能是跳转到复位向量，而没有传入 mailbox 地址。事实上，出于安全因素的考虑，这个地址也不能由 OS 提供。
  * 然而，AP 经过复位后会再次进入忙等例程 `AsmRelocateApMailBoxLoopStart`，而忙等例程会使用到 mailbox
  * 那么复位场景进入的忙等例程是如何再次得到 mailbox 地址的呢？
* 启动时，BSP 通过如下路径唤醒处于第二次忙等的 APs，并使其进入第三次忙等，等待 OS 最终的唤醒
  * AP 第三次忙等的入口地址 `RelocationMap.RelocateApLoopFuncAddress` 即 `AsmRelocateApMailBoxLoopStart`
  * 重定位到 ACPI NVs 后的 mailbox 地址 `RelocatedMailBox` 通过 `PcdOvmfSecGhcbBackupBase`（见 edk2/OvmfPkg/Library/TdxMailboxLib/TdxMailbox.c::`GetTdxMailBox()`）传递给 APs
```cpp
//edk2/OvmfPkg/TdxDxe/TdxAcpiTable.c
AlterAcpiTable()
-> RelocateMailbox(&RelocateResetVector)
   -> AsmGetRelocationMap (&RelocationMap)
   -> gBS->AllocatePages (AllocateAnyPages, EfiACPIMemoryNVS, RelocationPages, &Address)
   -> CopyMem (ApLoopFunc, RelocationMap.RelocateApLoopFuncAddress, RelocationMap.RelocateApLoopFuncSize)
      RelocatedMailBox = (MP_WAKEUP_MAILBOX *)Address;
   -> MpSendWakeupCommand (MpProtectedModeWakeupCommandWakeup, (UINT64)ApLoopFunc, (UINT64)RelocatedMailBox, ...)
```
* 在 edk2/OvmfPkg/IntelTdx/Sec/X64/IntelTdxAPs.nasm 的 `.do_wait_loop` 中第二次忙等的 APs 收到 `MpProtectedModeWakeupCommandWakeup` 命令后跳到 `.do_wakeup` 例程
  * edk2/OvmfPkg/IntelTdx/Sec/X64/SecEntry.nasm 中 `mov rsp, FixedPcdGet32 (PcdOvmfSecGhcbBackupBase)`，最后 `%include "IntelTdxAPs.nasm"`
  * edk2/OvmfPkg/IntelTdx/Sec/X64/IntelTdxAPs.nasm
```asm
;------------------------------------------------------------------------------
; @file
; Intel TDX APs
;
; Copyright (c) 2021 - 2022, Intel Corporation. All rights reserved.<BR>
; SPDX-License-Identifier: BSD-2-Clause-Patent
;
;------------------------------------------------------------------------------

%include "TdxCommondefs.inc"

    ;
    ; Note: BSP never gets here. APs will be unblocked by DXE
    ;
    ; R8  [31:0]  NUM_VCPUS
    ;     [63:32] MAX_VCPUS
    ; R9  [31:0]  VCPU_INDEX
    ;
ParkAp:

do_wait_loop:
    ;
    ; register itself in [rsp + CpuArrivalOffset]
    ;
    mov       rax, 1
    lock xadd dword [rsp + CpuArrivalOffset], eax
    inc       eax

.check_arrival_cnt:
    cmp       eax, r8d
    je        .check_command
    mov       eax, dword[rsp + CpuArrivalOffset]
    jmp       .check_arrival_cnt

.check_command: ;AP 的第二个忙等循环
    mov     eax, dword[rsp + CommandOffset]
    cmp     eax, MpProtectedModeWakeupCommandNoop
    je      .check_command

    cmp     eax, MpProtectedModeWakeupCommandWakeup
    je      .do_wakeup

    cmp     eax, MpProtectedModeWakeupCommandAcceptPages
    je      .do_accept_pages

    ; Don't support this command, so ignore
    jmp     .check_command
...
.do_wakeup:
    ;
    ; BSP sets these variables before unblocking APs
    ;   RAX:  WakeupVectorOffset
    ;   RBX:  Relocated mailbox address
    ;   RBP:  vCpuId
    ;
    mov     rax, 0
    mov     eax, dword[rsp + WakeupVectorOffset]    ;AsmRelocateApMailBoxLoopStart 的地址载入 %rax 
    mov     rbx, [rsp + WakeupArgsRelocatedMailBox] ;与 AsmRelocateApMailBoxLoopStart 约定好用 %rbx 来传递 mailbox 地址
    nop
    jmp     rax ;AP 跳出第二个忙等循环，进入第三个忙等循环
    jmp     $
```
* 从 `%rsp` 取出 `AsmRelocateApMailBoxLoopStart` 的地址
* 从 `%rsp` 取出 `RelocatedMailBox` 的地址
* 跳转到 `AsmRelocateApMailBoxLoopStart`
* 再捋一捋这个过程：
1. `AsmRelocateApResetVector` 例程的最后给记录 mailbox 的地址预留了空间 `mailbox_address:`
2. 第一次调用 `AsmRelocateApMailBoxLoopStart` 例程的时候就通过 `mov qword[rel mailbox_address], rbx` 把重定位后的 mailbox 地址记录在这个地方
3. 以后每次通过 `AsmRelocateApResetVector` 例程复位 AP，都会通过 `mov rbx, qword[rel mailbox_address]` 把 mailbox 地址从这取出，又传给 `AsmRelocateApMailBoxLoopStart` 例程
4. 这样，通过复位路径进入的 `AsmRelocateApMailBoxLoopStart` 例程依旧可以通过 `%rbx` 寄存器得到重定位后的 mailbox 地址

### [PATCH] UefiCpuPkg/MpInitLibUp: Update the ProcessorNumber
* 更新 `ProcessorNumber` 以避免在创建 `MpInformation2` HOB 时断言。
---
```diff
diff --git a/UefiCpuPkg/Library/MpInitLibUp/MpInitLibUp.c b/UefiCpuPkg/Library/MpInitLibUp/MpInitLibUp.c
index 86f9fbf90366..1b6e1962a214 100644
--- a/UefiCpuPkg/Library/MpInitLibUp/MpInitLibUp.c
+++ b/UefiCpuPkg/Library/MpInitLibUp/MpInitLibUp.c
@@ -104,7 +104,8 @@ MpInitLibGetProcessorInfo (
     return EFI_INVALID_PARAMETER;
   }

-  if (ProcessorNumber != 0) {
+  // Lower 24 bits contains the actual processor number.
+  if ((ProcessorNumber &= BIT24 - 1) != 0 ) {
     return EFI_NOT_FOUND;
   }
```
* `MpInitLibGetProcessorInfo()` 在进行此调用时获取有关所请求处理器的 MP 详细信息。此服务只能从 BSP 调用。
  * 函数的入参 `ProcessorNumber` 是处理器号，因为只能从 BSP 调用，因此必须是 `0`
  * 然而实际情况是，低 `24 bits` 包含实际的处理器号，因此只检查低 `24 bits` 即可

# 后续问题的修复

## 防止读取 `/proc/kcore` 和 `/proc/vmcore` 时访问未接受内存
* 对于尚未被接受的内存，当然也未被 guest TD 使用，因此通过 `/proc/kcore` 和 `/proc/vmcore` 内存镜像文件去访问这些页面的时候会导致 TD Exit，由于这不是期望的 EPT violation，会导致 VMM 杀死 TD。
* Qemu console
```c
# makedumpfile -l --message-level 31 -d 0 /proc/vmcore vmcore
KVM: unknown exit, hardware reason 8000401c
error: kvm run failed Input/output error
qemu: Failed to get registers: Input/output error
error: kvm run failed Input/output error
qemu: Failed to get registers: Input/output error
qemu: Failed to get registers: Input/output error
...
qemu: Failed to get registers: Input/output error
./start-qemu8.sh: line 113: 826713 Segmentation fault 
```
* 在旧一点的 TDX module 1.5 实现中，设置 TD 的 `ATTRIBUTES.SEPT_VE_DISABLE` 属性为 `1` 会导致访问未接受页面时 TDX module 不是注入 `#VE` 而是 TD Exit 到 VMM，并告知以 EPT violation 的信息
  * VMM 修复完 EPT violation 后回到 TD 的现场，但是由于 Secure EPT 条目的状态仍然是 `SEPT_PENDING` 状态，TD 会因为同样的原因 TD Exit
  * VMM 已经处理过了，所以又会先返回到 TDX module，TDX module 再回到 TD，形成死循环
* 新的 TDX module 1.5 遵循 TDX module spec，Table 9.2: EPT Violation TD Exit Cases and Possible Host VMM Actions，TD 会以 "**exit qualification type 0x6**" 的原因退出
* tdx-module/src/td_dispatcher/vm_exits/td_ept_violation.c
```c
void td_ept_violation_exit(vmx_exit_qualification_t exit_qualification, vm_vmexit_exit_reason_t vm_exit_reason)
{
...
    if (!shared_bit)
    {
        // Check if the EPT violation happened due to an access to a PENDING page.
        // If so, there are two options:
        //  - #VE injection to the guest TD.
        //  - TD exit with Extended Exit Qualification set to denote a PENDING page.
...
        if (sept_state_is_any_pending_and_guest_acceptable(sept_entry))
        {
            // This is a pending page waiting for acceptable by the TD
            if (tdcs_p->executions_ctl_fields.td_ctls.pending_ve_disable)
            {
                // The TD is configured to TD exit on access to a PENDING page
                ext_exit_qual.type = VMX_EEQ_PENDING_EPT_VIOLATION;
            }
            else
            {
                // The TD is configured to throw a #VE on access to a PENDING page
                uint64_t gla;
                ia32_vmread(VMX_VM_EXIT_GUEST_LINEAR_ADDRESS_ENCODE, &gla);
                tdx_inject_ve(vm_exit_reason.raw, exit_qualification.raw,
                                tdx_local_data_ptr->vp_ctx.tdvps, gpa.raw, gla);
                return;
            }
        }

        // At this point we're going to do a TD exit. If the GPA is private, log suspected 0-step
        // attacks that repeatedly cause EPT violations with the same RIP.
        td_exit_epf_stepping_log(gpa);
    }

    tdx_ept_violation_exit_to_vmm(gpa, vm_exit_reason, exit_qualification.raw, ext_exit_qual.raw);
}
```
* KVM 的也会处理发现这个错误把 TD kill 掉
* Host dmesg
```c
[86969.097013] kvm_intel: EPT violation at gpa 0x110a00000, with invalid ext exit qualification type 0x6
[86969.729426] CPU 3/KVM[826722]: segfault at 7fb218b37ff8 ip 00007fb22b28536c sp 00007fb218b38000 error 6 in libc.so.6[7fb22b200000+1ed000] likely on CPU 9 (core 9, socket 0)
[86969.729436] Code: 01 48 89 53 08 0f b6 00 48 83 c4 08 5b 5d c3 0f 1f 80 00 00 00 00 e8 13 c9 ff ff eb d1 90 f3 0f 1e fa 41 57 31 c0 41 56 41 55 <41> 54 55 53 48 83 ec 18 48 89 14 24 48 85 d2 0f 84 97 00 00 00 4c 
```
* 要修复这个问题，需完善设计上的漏洞，需要以下两个 commits

### efi/unaccepted: do not let /proc/vmcore try to access unaccepted memory
* commit `7cd34dd3c9bf` efi/unaccepted: do not let /proc/vmcore try to access unaccepted memory
* 补丁系列“Do not try to access unaccepted memory”，v2。
* 最近添加了对未接受的内存的支持，请参阅 commit `dcdfdd40fa82`（“mm：Add support for unaccepted memory”），虚拟机可能需要先接受内存才能使用。
* 填补了一些内存暴露的空白，而无需检查它是否是未接受的内存。
* 不要让 `/proc/vmcore` 尝试访问未接受的内存，因为这可能会导致 guest fail。
* 对于只读的 `/proc/vmcore`，这意味着未接受的内存的读取或 `mmap` 将返回零。
---
* 新增接口函数 `pfn_is_unaccepted_memory()`，输入 `pfn` 返回该 `pfn` 是否是已接受的内存
```c
static inline bool pfn_is_unaccepted_memory(unsigned long pfn)
{
       phys_addr_t paddr = pfn << PAGE_SHIFT;

       return range_contains_unaccepted_memory(paddr, paddr + PAGE_SIZE);
}
```
* 新增 `unaccepted_memory_init_kdump()` 函数，在内核初始化时，注册 vmcore 回调 `vmcore_cb` 结构实例
* 该实例的成员 `.pfn_is_ram = unaccepted_memory_vmcore_pfn_is_ram` 是新增的回调函数 `unaccepted_memory_vmcore_pfn_is_ram()`
  * 该回调函数通过 `pfn_is_unaccepted_memory()` 接口返回 `pfn` 是否是已接受的内存
```cpp
#ifdef CONFIG_PROC_VMCORE
static bool unaccepted_memory_vmcore_pfn_is_ram(struct vmcore_cb *cb,
                                               unsigned long pfn)
{
       return !pfn_is_unaccepted_memory(pfn);
}

static struct vmcore_cb vmcore_cb = {
       .pfn_is_ram = unaccepted_memory_vmcore_pfn_is_ram,
};

static int __init unaccepted_memory_init_kdump(void)
{
       register_vmcore_cb(&vmcore_cb);
       return 0;
}
core_initcall(unaccepted_memory_init_kdump);
#endif /* CONFIG_PROC_VMCORE */
```
* 该回调函数仅被 `pfn_is_ram()` 调用
  * fs/proc/vmcore.c
```cpp
static bool pfn_is_ram(unsigned long pfn)
{
    struct vmcore_cb *cb;
    bool ret = true;

    list_for_each_entry_srcu(cb, &vmcore_cb_list, next,
                 srcu_read_lock_held(&vmcore_cb_srcu)) {
        if (unlikely(!cb->pfn_is_ram))
            continue;
        ret = cb->pfn_is_ram(cb, pfn);
        if (!ret)
            break;
    }

    return ret;
}
```
* 而 `pfn_is_ram()` 在以下场景被调用：
```c
1 fs/proc/kcore.c|566| <<read_kcore_iter>> is_page_hwpoison(page) || !pfn_is_ram(pfn) ||
2 fs/proc/vmcore.c|153| <<read_from_oldmem>> if (!pfn_is_ram(pfn)) {
3 fs/proc/vmcore.c|517| <<remap_oldmem_pfn_checked>> if (!pfn_is_ram(pos)) {
```

### proc/kcore: do not try to access unaccepted memory
* commit `e538a5820978` proc/kcore: do not try to access unaccepted memory
* 最近添加了对未接受的内存的支持，请参阅 commit `dcdfdd40fa82`（“mm：Add support for unaccepted memory”），虚拟机可能需要先接受内存才能使用。
* 不要尝试访问未接受的内存，因为这可能会导致 guest fail。
* 对于只读且不支持 mmap 的 `/proc/kcore`，这意味着读取未接受的内存将返回零。
---
```diff
diff --git a/fs/proc/kcore.c b/fs/proc/kcore.c
index 23fc24d16b31..6422e569b080 100644
--- a/fs/proc/kcore.c
+++ b/fs/proc/kcore.c
@@ -546,7 +546,8 @@ static ssize_t read_kcore_iter(struct kiocb *iocb, struct iov_iter *iter)
                         * and explicitly excluded physical ranges.
                         */
                        if (!page || PageOffline(page) ||
-                           is_page_hwpoison(page) || !pfn_is_ram(pfn)) {
+                           is_page_hwpoison(page) || !pfn_is_ram(pfn) ||
+                           pfn_is_unaccepted_memory(pfn)) {
                                if (iov_iter_zero(tsz, iter) != tsz) {
                                        ret = -EFAULT;
                                        goto out;
```
* 修复的原理很简单，在迭代读取 kcore 的时候判断 `pfn` 是否是未接受内存，如果是则跳过
* 感觉这个 commit 可以不要，因为以下路径也可以走到 `pfn_is_unaccepted_memory(pfn)`（该函数前的 `!` 与 `pfn_is_ram()` 前的 `!` 抵消）
```c
pfn_is_ram(pfn)
-> cb->pfn_is_ram(cb, pfn)
=> unaccepted_memory_vmcore_pfn_is_ram()
   -> !pfn_is_unaccepted_memory(pfn)
```

## Guest kernel panic while AP online later with kernel parameter `maxcpus=1`
* [[PATCH RESEND v4] x86/acpi: fix panic while AP online later with kernel parameter maxcpus=1](https://lore.kernel.org/all/20240805103531.1230635-1-zhiquan1.li@intel.com/)
* commit `ab84ba647f2c` [x86/acpi: Remove __ro_after_init from acpi_mp_wake_mailbox](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ab84ba647f2c94ac4d0c3fc6951c49f08aa1fcf7)

## References
* [[PATCHv11 00/19] x86/tdx: Add kexec support](https://lore.kernel.org/lkml/20240528095522.509667-1-kirill.shutemov@linux.intel.com/)
* [[PATCHv12 00/19] x86/tdx: Add kexec support](https://lore.kernel.org/lkml/20240614095904.1345461-1-kirill.shutemov@linux.intel.com/)
* [[RFC] ACPI Code First ECR: Support for resetting the Multiprocessor Wakeup Mailbox](https://lore.kernel.org/all/13356251.uLZWGnKmhe@kreacher/)