# 30 APIC 虚拟化和虚拟中断

* VMCS 包括启用中断虚拟化的控制和高级可编程中断控制器 (APIC)。
* 当使用这些控制时，处理器将模拟对 APIC 的许多访问，跟踪虚拟 APIC 的状态，并递交虚拟中断 —— 所有这些都在 VMX non-root operation 中进行，不会 VM exit。
* 处理器使用虚拟机监视器（VMM）识别的虚拟 APIC 页面跟踪虚拟 APIC 的状态。

## 30.1 虚拟 APIC 状态
* **虚拟 APIC 页（virtual-APIC page）** 是一个 `4 KB` 的内存区域，处理器使用它来虚拟化对 APIC 寄存器的某些访问并管理虚拟中断。
  * 虚拟 APIC 页的 *物理地址* 是 **虚拟 APIC 地址（virtual-APIC address）**，VMCS 中的一个 `64` bit 的 VM 执行控制字段（参见第 25.6.8 节）。
* 根据某些 VM 执行控制域的设置，处理器可能会虚拟化虚拟 APIC 页面上的某些字段，其功能类似于 local APIC 所执行的功能。
* 除了与虚拟化 APIC 寄存器（在第 30.1.1 节中定义）相对应的字段外，软件可以修改 VMX non-root operation 中逻辑处理器的当前 VMCS 引用的虚拟 APIC 页面。

### 30.1.1 虚拟 APIC 寄存器
* 根据某些 VM 执行控制域的设置，逻辑处理器可以使用虚拟 APIC 页面上的以下字段虚拟化对 APIC 寄存器的某些访问：
  * **Virtual task-priority register (VTPR)**: the 32-bit field located at offset `0x080` on the virtual-APIC page.
  * **Virtual processor-priority register (VPPR)**: the 32-bit field located at offset `0x0A0` on the virtual-APIC page.
  * **Virtual end-of-interrupt register (VEOI)**: the 32-bit field located at offset `0x0B0` on the virtual-APIC page.
  * **Virtual interrupt-service register (VISR)**: 256 bit 值包含八个不连续的 32 位字段，位于虚拟 APIC 页上的偏移量 `0x100`、`0x110`、`0x120`、`0x130`、`0x140`、`0x150`、`0x160` 和 `0x170`。
    * `VISR` 的 `bit x` 位于偏移量 `(0x100 | ((x & 0xE0) >> 1))` 的 bit 位置 `(x & 0x1F)`。
    * 处理器仅使用偏移量为 `0x100`、`0x110`、`0x120`、`0x130`、`0x140`、`0x150`、`0x160` 和 `0x170` 的每个 `16` 字节字段的低 `4` 字节
  * **Virtual interrupt-request register (VIRR)**: 256 bit 值包含八个不连续的 32 位字段，位于虚拟 APIC 页上的偏移量 `0x200`、`0x210`、`0x220`、`0x230`、`0x240`、`0x250`、`0x260` 和 `0x270`。
    * `VIRR` 的 `bit x` 位于偏移量 `(0x200 | ((x & 0xE0) >> 1))` 的 bit 位置 `(x & 0x1F)`。
    * 处理器仅使用偏移量为 `0x200`、`0x210`、`0x220`、`0x230`、`0x240`、`0x250`、`0x260` 和 `0x270` 的每个 `16` 字节字段的低 `4` 字节。
  * **Virtual interrupt-command register (VICR_LO)**: the 32-bit field located at offset `0x300` on the virtual-APIC page.
  * **Virtual interrupt-command register (VICR_HI)**: the 32-bit field located at offset `0x310` on the virtual-APIC page.
* 每当 “use TPR shadow” VM 执行控制域为 `1` 时，`VTPR` 字段虚拟化 `TPR`。
  * 只要 “virtual-interrupt delivery” VM 执行控制域为 `1`，上述其他字段虚拟化相应的 APIC 寄存器。
  * 当 “IPI virtualization” VM 执行控制域为 `1` 时，`VICR_LO` 和 `VICR_HI` 也虚拟化 `ICR`。

### 30.1.2 `TPR` 虚拟化
* 处理器响应以下操作执行 `TPR` 虚拟化：
  1. `MOV` 到 `CR8` 指令的虚拟化；
  2. 往 APIC access page 上 `0x080` 偏移量写入的虚拟化；
  3. `ECX = 0x808` 的 `WRMSR` 指令的虚拟化。
* 以下伪代码详细描述了 `TPR` 虚拟化的行为：
```vb
IF “virtual-interrupt delivery” is 0
THEN
    IF VTPR[7:4] < TPR threshold (see Section 25.6.8)
       THEN cause VM exit due to TPR below threshold;
    FI;
ELSE
    perform PPR virtualization (see Section 30.1.3);
    evaluate pending virtual interrupts (see Section 30.2.1);
FI;
```

* 由 `TPR` 虚拟化引起的任何 VM exit 都是类似陷阱的：导致 `TPR` 虚拟化的指令在 VM exit 发生之前完成（例如，保存在 VMCS 的 guest-state area 中的 `CS:RIP` 的值引用了 *下一条指令*）。

### 30.1.3 `PPR` 虚拟化
* 处理器执行 `PPR` 虚拟化以响应以下操作：
  1. VM entry；
  2. `TPR` 虚拟化；
  3. `EOI` 虚拟化。
* `PPR` 虚拟化使用 guest interrupt status（特别是 `SVI`；请参阅第 25.4.2 节）和 `VTPR`。
* **Guest interrupt status**（16 位）。此字段仅在 “virtual-interrupt delivery” VM 执行控制域设置为 `1` 的处理器上受支持。它表征了 guest 虚拟 APIC 状态的一部分，不对应于任何处理器或 APIC 寄存器。它包括两个 8 位子字段：
  * **Requesting virtual interrupt（RVI）**：这是 guest interrupt status 的低字节。处理器将此值视为请求服务的最高优先级虚拟中断的向量。（值为 `0` 表示没有此类中断。）
  * **Servicing virtual interrupt（SVI）**：这是 guest interrupt status 的高字节。处理器将此值视为正在服务的最高优先级虚拟中断的向量。（值为 `0` 表示没有此类中断。）
* 以下伪代码详细描述了 PPR 虚拟化的行为：
```vb
IF VTPR[7:4] ≥ SVI[7:4]
THEN VPPR := VTPR & 0xFF;
ELSE VPPR := SVI & 0xF0;
FI;
```
* `PPR` 虚拟化始终清除 `VPPR` 的字节 `3:1`
* `PPR` 虚拟化仅由 `TPR` 虚拟化、`EOI` 虚拟化和 VM entry 引起。
  * 虚拟中断的传递也会修改 `VPPR`，但方式不同（请参阅第 30.2.2 节）。
  * 没有其他操作修改 `VPPR`，即使它们修改了 `SVI`、`VISR` 或 `VTPR`。

### 30.1.4 `EOI` 虚拟化