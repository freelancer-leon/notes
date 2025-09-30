# APM-SEV

## 15.34 Secure Encrypted Virtualization

* 当 CPU 利用 AMD-V 虚拟化特性在 guest mode 下运行时，可启用 **安全加密虚拟化（Secure Encrypted Virtualization，简称 SEV）** 功能。
  * SEV 能够支持加密虚拟机（VM）的运行，在该模式下，虚拟机的代码与数据会受到安全保护，仅虚拟机自身内部可获取其解密版本。
  * 每个虚拟机可关联一个唯一的加密密钥：若其他实体使用不同密钥访问数据，SEV 加密虚拟机的数据会因解密密钥错误而无法正确解密，最终得到无法识别的乱码数据。
* 需重点注意的是，SEV 模式与 *标准 x86 虚拟化安全模型存在显著差异*
  * 在 SEV 模式下，hypervisor 不再能检查或修改 guest 的全部代码与数据。
  * 由 guest 管理的 guest 页表（guest page tables）可将数据内存页标记为 “私有”（private）或 “共享”（shared），允许被选中的页面在 guest 外部进行共享访问。
    * “私有页” 则通过 guest 专属密钥进行加密，
    * “共享页” 可被 hypervisor 访问。

### 15.34.1 确定对 SEV 的支持

* 内存加密功能的支持情况会在 CPUID `8000_001F [EAX]` 中报告，具体描述参见 7.10.1 节 “Determining Support for Secure Memory Encryption” 第 238 页。其中`Bit 1` 用于指示是否支持安全加密虚拟化。
* 当内存加密功能存在时，CPUID `8000_001F [EBX]` 和 `CPUID 8000_001F [ECX]` 会提供与内存加密使用相关的额外信息，例如同时支持的密钥数量、用于将页面标记为加密状态的页表位等。
* 此外，在部分实现中，启用内存加密功能后，处理器的物理地址大小可能会缩减（例如从 `48` bits 缩减至 `43` bits）。
  * 以该示例而言，物理地址的第 `47` 至 `43` 位（`bits 47:43`）将被视为保留位，除非有其他特别说明。
  * 若某一实现支持内存加密，CPUID `8000_001F [EBX]` 会报告是否存在物理地址大小缩减的情况。
* **此模式下的保留位与其他页表保留位的处理方式一致：若在地址转换过程中发现这些保留位的值非零，将触发缺页异常（page fault）。**
* **译注**：这就杜绝 hypervisor 试图修改 PT/NPT 在 SPA 中嵌入信息的可能性。
  * 对于 TDX 也是类似的，当不在 SEAM mode 时，为编码 TDX private KeyID 而保留的物理地址位被视为保留位（详见 TDX CPU Arch Spec）：
    * 试图修改 PT 在 PA 中嵌入 keyID 会导致翻译线性地址（VA）时产生一个原因是 *保留位被修改* 的 `#PF`；
    * 试图修改 EPT 在 HPA 中嵌入 keyID 会导致翻译 GPA 时产生一次原因是 *EPT miscofig* 的 VM-Exit
* 有关内存加密功能的完整 CPUID 详情，可参见《卷 3》（Volume 3）的 E.4.17 节。

### 15.34.2 Key 管理
* 在本文档定义的内存加密扩展功能下，每个启用 SEV 的 geuest 虚拟机都关联有一个内存加密密钥，而 SME 模式（若使用，参见第 238 页的 7.10 节）则关联另一个独立密钥。
* SEV 功能的密钥管理不由 CPU 负责，而是由 AMD SOC（系统级芯片）上搭载的独立处理器 —— **AMD 安全处理器（AMD Secure Processor，简称 AMD-SP）** 处理。关于 AMD-SP 运行机制的详细讨论，超出了本手册的涵盖范围。
* CPU 软件无法感知这些密钥的具体值，但 hypervisor 应通过 *AMD-SP 驱动* 协调虚拟机密钥的加载。
  * 该协调过程还会确定 hypervisor 应为特定 guest 分配哪个 ASID（地址空间标识符）。
* 在 SEV 模式下，ASID 用作密钥索引，用于标识应使用哪一个加密密钥对与该启用 SEV 的 guest 相关的内存流量进行加密/解密。
* ==加密密钥本身始终对 CPU 软件不可见，且绝不会以明文形式存储在芯片外部。==

### 15.34.3 启用 SEV
* 在启动加密虚拟机之前，软件必须按照第 238 页 7.10.2 节 “Enabling Memory Encryption Extensions” 中的描述，将 `SYSCFG` MSR 中的 `MemEncryptionModEn` 设为 `1`。
* 之后，若 hypervisor 将 VMCB 偏移量 `0x090` 处的 *SEV 使能位*（`bit 1`）设为 `1`，则可在执行 `VMRUN` 指令期间，为特定虚拟机启用 SEV。
* 当 VMCB 中启用 SEV 后，`VMRUN` 指令执行过程中会额外进行以下一致性检查：
  * 必须启用嵌套分页（Nested paging）
  * 必须设置 `HWCR` MSR 中的 `SmmLock`（系统管理模式锁定）bit
  * ASID 不得大于 CPUID `Fn8000_001F_ECX [NumEncryptedGuests]` 所定义的最大值
* 若上述任一一致性检查失败，VMRUN 指令将终止，并返回 `VMEXIT_INVALID` 错误码。
* 若 `MemEncryptionModEn` 为 `0`，则无法启用 SEV，且 VMCB 中用于 SEV 的控制位将被忽略。
* 需注意，在 CPUID `Fn8000_001F_EAX[64BitHost]` 设为 `1` 的系统上，hypervisor 必须处于 64-bit 模式，才能执行 `VMRUN` 指令以启动 *SEV-enabled* 的 guest。
  * 若不满足此条件，`VMRUN` 将失败，并返回 `VMEXIT_INVALID` 错误码。

### 15.34.4 支持的操作模式
* 安全加密虚拟化（Secure Encrypted Virtualization，SEV）可在运行于任意操作模式的 guest 上启用。但 guest 仅能在长模式（long mode）或传统物理地址扩展模式（legacy PAE mode） 下，对内存加密进行控制。
* 在所有其他操作模式下，guest 的所有内存访问都会被无条件判定为 “私有访问”，并使用该 guest 专属的密钥进行加密。

### 15.34.5 SEV 加密行为
* 当 guest 在启用 SEV 的情况下运行时，guest 页表用于确定内存页的 `C-bit`，进而确定该内存页的加密状态。
  * 这使得 guest 能够决定哪些页是私有页，哪些是共享页，==但这种控制仅适用于数据页==。
* 代表指令提取和客户机页表遍历进行的内存访问始终被视为私有访问，无论 C 位的软件值如何。此行为可确保非客户机实体（如管理程序）无法向启用 SEV 的客户机注入自己的代码或数据。如果客户机确实希望让指令页或页表中的数据可被客户机外部的代码访问，则必须将这些数据显式复制到共享数据页中。
* 需要注意的是，虽然客户机可以选择在指令页和页表地址上显式设置 C 位，但在这种情况下，该位的值无关紧要，因为硬件始终会将这些访问作为私有访问来处理。

