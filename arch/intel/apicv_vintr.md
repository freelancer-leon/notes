# 30 APIC 虚拟化和虚拟中断

* VMCS 包括启用中断虚拟化的控制和高级可编程中断控制器 (APIC)。
* 当使用这些控制时，处理器将模拟对 APIC 的许多访问，跟踪虚拟 APIC 的状态，并递交虚拟中断 —— 所有这些都在 VMX non-root operation 中进行，不会 VM exit。
* 处理器使用虚拟机监视器（VMM）识别的 *虚拟 APIC 页面* 跟踪虚拟 APIC 的状态。
* 以下是与 APIC 虚拟化和虚拟中断相关的 VM 执行控制域（有关这些控制域的位置的信息，请参阅第 25.6 节）：
* **Virtual-interrupt delivery**：该控制域可以评估和递交待处理的虚拟中断（第 30.2 节）。它还支持对控制中断优先级的 APIC 寄存器进行写入仿真（如已启用，内存映射或基于 MSR）。
* **Use TPR shadow**：该控制域允许通过 `CR8`（第 30.3 节）以及（如果启用）通过内存映射或基于 MSR 的接口模拟对 APIC 任务优先级寄存器（`TPR`）的访问。
* **Virtualize APIC accesses**：该控制域通过在访问 VMM 指定的 APIC access page 时导致 VM exit 来实现对 APIC 的内存映射访问的虚拟化（第 30.4 节）。某些其他控制域（如果设置）可能会导致其中一些访问被模拟，而不是导致 VM exit。
* **Virtualize x2APIC mode**：该控制支持对 APIC 的基于 MSR 的访问的虚拟化（第 30.5 节）。
* **APIC-register virtualization**：该控制允许从 *虚拟 APIC 页* 中以内存映射和基于 MSR 的方式 **读取**（如启用）大多数 APIC 寄存器。
  * 它将对 APIC access page 的内存映射 **写入** 定向到 *虚拟 APIC 页面*，随后 VM exit 用 VMM 模拟 。
* **Process posted interrupts**：该控制允许软件在数据结构中 post 虚拟中断并向另一个逻辑处理器发送通知；收到通知后，目标处理器将通过将 posted interrupts 复制到*虚拟 APIC 页面* 来处理它们（第 30.6 节）。
* **IPI virtualization**：该控制启用处理器间中断的虚拟化（第 30.1.6 节）。


## 30.1 虚拟 APIC 状态
* **虚拟 APIC 页（virtual-APIC page）** 是一个 `4 KB` 的内存区域，处理器使用它来虚拟化对 APIC 寄存器的某些访问并管理虚拟中断。
  * 虚拟 APIC 页的 *物理地址* 是 **虚拟 APIC 地址（virtual-APIC address）**，VMCS 中的一个 `64` bit 的 VM 执行控制字段（参见第 25.6.8 节）。
* 根据某些 VM 执行控制域的设置，处理器可能会虚拟化 *虚拟 APIC 页面* 上的某些字段，其功能类似于 local APIC 所执行的功能。
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

* 处理器执行 **`EOI` 虚拟化** 以响应以下操作：
1. 虚拟化在 APIC access page 上的偏移量 `0x0B0` 写入
2. 虚拟化 `ECX = 0x80B` 的 `WRMSR` 指令
* 只有当 “virtual-interrupt delivery” VM 执行控制域为 `1` 时，EOI 虚拟化才会发生。
* EOI 虚拟化使用并更新 guest interrupt status（具体来说，`SVI`；参见第 25.4.2 节）。
* 以下伪代码详细说明了 EOI 虚拟化的行为：
```vb
Vector := SVI;
VISR[Vector] := 0; (see Section 30.1.1 for definition of VISR)

IF any bits set in VISR
THEN SVI := highest index of bit set in VISR
ELSE SVI := 0;
FI;

perform PPR virtualiation (see Section 30.1.3);

IF EOI_exit_bitmap[Vector] = 1 (see Section 25.6.8 for definition of EOI_exit_bitmap)
THEN cause EOI-induced VM exit with Vector as exit qualification;
ELSE evaluate pending virtual interrupts; (see Section 30.2.1)
FI;
```

* 由 EOI 虚拟化引起的任何 VM exit 都是类似陷阱的：导致 EOI 虚拟化的指令在 VM exit 发生之前完成（例如，保存在 VMCS 的 guest-state area 中的 `CS:RIP` 的值引用了下一条指令）。

### 30.1.5 Self-IPI 虚拟化
* 处理器执行 **self-IPI 虚拟化** 以响应以下操作：
1. 虚拟化写入 APIC access page 上的偏移量 `0x300`；
2. `ECX = 0x83F` 的 `WRMSR` 指令虚拟化。
* Self-IPI 虚拟化仅在 “virtual-interrupt delivery” VM 执行控制域为 `1` 时发生。
* 导致 self-IPI 虚拟化的每个操作都提供一个 8 位向量（参见第 30.4.3 节和第 30.5 节）。
* Self-IPI 虚拟化更新 guest interrupt status（具体来说，`RVI`；参见第 25.4.2 节）。
* 以下伪代码详细说明了 self-IPI 虚拟化的行为：
```vb
VIRR[Vector] := 1; (see Section 30.1.1 for definition of VIRR)
RVI := max{RVI,Vector};
evaluate pending virtual interrupts; (see Section 30.2.1)
```

### 30.1.6 IPI 虚拟化
* 处理器执行 **IPI 虚拟化** 以响应以下操作：
1. 虚拟化写入 APIC access page 上的偏移量 `0x300`（第 30.4.3 节）；
2. `ECX = 0x830` 的 `WRMSR` 指令的虚拟化（第 30.5 节）；
3. `SENDUIPI` 某些执行的虚拟化（第 30.7 节）。
* IPI 虚拟化仅 在“IPI virtualization” VM 执行控制域为 `1` 时发生。
* 导致 IPI 虚拟化的每个操作都提供一个 `8` 位虚拟向量 `V` 和一个 `32` 位虚拟 APIC ID `T`。
* IPI 虚拟化使用这些值来使用 PID-pointer table，初始所指示的虚拟 IPI。（回想一下，PID-pointer table 地址是 VMCS 中的一个字段，就像最后一个 PID-pointer index 一样。）
* 具体来说，CPU 使用虚拟 APIC ID 从 PID-pointer table 中选择一个条目。它使用该条目中的地址来定位 posted-interrupt descriptor（PID），然后在该 PID 中发布带有向量 `V` 的虚拟中断。
* 以下伪代码详细说明了自 IPI 虚拟化的行为：
```vb
IF V < 16
THEN APIC-write VM exit; // illegal vector
ELSE IF T ≤ last PID-pointer index
     THEN
     PID_ADDR := 8 bytes at (PID-pointer table address + (T << 3));

     IF PID_ADDR sets bits beyond the processor’s physical-address width OR
          PID_ADDR[5:0] ≠ 000001b // PID pointer not valid or reserved bits set
     THEN APIC-write VM exit; // See Section 30.4.3.3
     ELSE
          PIR := 32 bytes at PID_ADDR; // under lock
          PIR[V] := 1;
          store PIR at PID_ADDR; // release lock
          NotifyInfo := 8 bytes at PID_ADDR + 32; // under lock

          IF NotifyInfo.ON = 0 AND NotifyInfo.SN = 0
          THEN
               NotifyInfo.ON := 1;
               SendNotify := 1;
          ELSE SendNotify := 0;
          FI;

          store NotifyInfo at PID_ADDR + 32; // release lock

          IF SendNotify = 1
          THEN send an IPI specified by NotifyInfo.NDST and NotifyInfo.NV;
          FI;
     FI;
ELSE APIC-write VM exit; // virtual APIC ID beyond end of tables
FI;
```
* Notification IPI 的发送由所选 PID 中的字段指示：`NDST`（Notification destination `PID[319:288]`）和 `NV`（Notification vector `PID[279:272]`）：
  * 如果 local APIC 处于 xAPIC 模式，这是通过将 `NDST[15:8]`（`PID[303:296]`）写入 `ICR_HI[31:24]`（从 `IA32_APIC_BASE` 偏移 `0x310`）然后写入生成的 IPI NV 到 `ICR_LO`（从 `IA32_APIC_BASE` 偏移 `0x300`）。
  * 如果 local APIC 处于 x2APIC 模式，则这是通过 `ECX = 0x830`（`ICR`）、`EAX = NV` 和 `EDX = NDST` 执行 `WRMSR` 生成的 IPI。
* 如果伪代码指定 APIC 写入 VM exit，则此 VM exit 的发生就像在 APIC access page 上对页面偏移量 `0x300` 进行了写访问一样（参见第 30.4.3.3 节）

## 30.2 虚拟中断的评估和递交
* 如果 “virtual-interrupt delivery” VM 执行控制域为 `1`，则 VMX non-root operation 中或 VM entry 期间的某些操作会导致处理器评估并递交虚拟中断。
* 虚拟中断的评估由某些更改虚拟 APIC 页面状态的操作触发，并在第 30.2.1 节中描述。该评估可能导致识别虚拟中断。
* 一旦识别出虚拟中断，处理器就可以在 VMX non-root operation 内传送它，而无需 VM exit。虚拟中断递交在第 30.2.2 节中描述。

### 30.2.1 评估挂起的虚拟中断
* 如果 “virtual-interrupt delivery” VM 执行控制域为 `1`，则某些操作会导致逻辑处理器评估挂起的虚拟中断。
* 以下操作会导致对挂起的虚拟中断进行评估：
  1. VM entry；
  2. TPR 虚拟化；
  3. EOI 虚拟化；
  4. self IPI 虚拟化；
  5. posted-interrupt processing。
* 有关何时执行挂起虚拟中断评估的详细信息，请参阅第 27.3.2.5 节、第 30.1.2 节、第 30.1.4 节、第 30.1.5 节和第 30.6 节。
* 没有其他操作会导致对挂起的虚拟中断进行评估，即使它们修改了 `RVI` 或 `VPPR`。
* 对挂起虚拟中断的评估使用 guest interrupt status（具体来说，`RVI`；参见第 25.4.2 节）。
* 以下伪代码详细说明了挂起虚拟中断的评估：
```vb
IF “interrupt-window exiting” is 0 AND
     RVI[7:4] > VPPR[7:4] (see Section 30.1.1 for definition of VPPR)
THEN recognize a pending virtual interrupt;
ELSE
     do not recognize a pending virtual interrupt;
FI;
```
* 一旦被识别，一个虚拟中断可以在 VMX non-root operation 下被传递； 参见第 30.2.2 节。
* 挂起虚拟中断的评估仅由 VM entry、TPR 虚拟化、EOI 虚拟化、self-IPI 虚拟化和 posted-interrupt processing 引起。没有其他操作这样做，即使它们修改了 `RVI` 或 `VPPR`。
* 在递交虚拟中断之后，逻辑处理器停止识别挂起的虚拟中断。

### 30.2.2 虚拟中断递交
* 如果已识别出虚拟中断（请参阅第 30.2.1 节），则在满足以下条件时，它会在指令边界处传送：
1. `RFLAGS.IF = 1`；
2. 没有被 `STI` 阻断；
3. 没有被 `MOV SS`或 `POP SS` 阻塞；
4. “interrupt-window exiting” VM 执行控制域为 `0`。
* 由于 “interrupt-window exiting” VM 执行控制域设置为 `1`，虚拟中断递交与 VM exit 具有相同的优先级。
  * 因此，不可屏蔽中断（NMI）和更高优先级的事件优先于递交虚拟中断；
  * 虚拟中断的传递优先于外部中断和较低优先级的事件。
* 虚拟中断递交将逻辑处理器从与外部中断相同的非活动活动状态中唤醒。具体来说，它将逻辑处理器从使用 `HLT` 和 `MWAIT` 指令进入的状态中唤醒。
  * 它不会唤醒处于关闭状态或等待 SIPI 状态的逻辑处理器。
* 虚拟中断传递更新 guest interrupt status（`RVI` 和 `SVI`；参见第 25.4.2 节）并传递 VMX non-root operation 内的事件，而无需 VM exit。
* 以下伪代码详细说明了虚拟中断递交的行为（有关 `VISR`、`VIRR` 和 `VPPR` 的定义，请参见第 30.1.1 节）：
```vb
Vector := RVI;
VISR[Vector] := 1;
SVI := Vector;
VPPR := Vector & F0H;
VIRR[Vector] := 0;

IF any bits set in VIRR
THEN RVI := highest index of bit set in VIRR
ELSE RVI := 0;
FI;

cease recognition of any pending virtual interrupt;

IF transactional execution is in effect
THEN abort transactional execution and transition to a non-transactional execution;
FI;

IF logical processor is in enclave mode
THEN cause an Asynchronous Enclave Exit (AEX) (see Chapter 37, “Enclave Exiting Events”)
FI;

IF CR4.UINTR = 1 AND IA32_EFER.LMA = 1 AND Vector = UINV
THEN virtualize user-interrupt notification identification and processing (see Section 30.2.3)
ELSE deliver interrupt with Vector through IDT;
FI;
```

## 30.4 虚拟化内存映射的 APIC 访问

* **APIC-access address (64 bits)**：该字段包含 `4 KB` APIC access page 的物理地址
  * 如果 “virtualize APIC accesses” VM 执行控制域为 `1`，则访问此页面可能会导致 VM exit 或被处理器虚拟化
  * 参见第 30.4 节。 APIC-access address 仅存在于支持 “virtualize APIC accesses” VM 执行控制域置 `1` 的处理器上
* **Virtual-APIC address (64 bits)**：该字段包含 `4 KB` 虚拟 APIC 页的物理地址
  * 处理器使用虚拟 APIC 页来虚拟化对 APIC 寄存器的某些访问并管理虚拟中断；参见第 30 章
  * 根据前面指出的控制域的设置，可以通过以下操作访问虚拟 APIC 页面：
    * `MOV CR8` 指令（参见第 30.3 节）
    * 此外，如果 “virtualize APIC accesses” VM 执行控制域为 `1`，则访问 APIC access page（参见第 30.4 节）
    * 此外，如果 `ECX` 的值在 `0x800–0x8FF` 范围内（指示 APIC MSR）并且 “virtualize x2APIC mode” VM 执行控制域为 `1`，则 `RDMSR` 和 `WRMSR` 指令（参见第 30.5 节）
* 如果 “use TPR shadow” VM 执行控制域为 `1`，VM entry 将确保虚拟 APIC 地址是 `4 KB` 对齐的
  * 虚拟 APIC 地址仅存在于支持 “use TPR shadow” VM 执行控制域置为 `1` 的处理器上
---
* 当 local APIC 处于 xAPIC 模式时，软件使用内存映射接口访问 local APIC 的控制寄存器。
  * 具体来说，软件使用线性地址转换为 `IA32_APIC_BASE` MSR 中基地址指示的页帧上的 **物理地址**（请参见第 11.4.4 节 “Local APIC Status and Location”）。本节介绍如何虚拟化这些访问。
* 虚拟机监视器（VMM）可以虚拟化这些内存映射的 APIC 访问，通过确保对任何线性地址的访问将访问 local APIC，而不是导致 VM exit。
  * 这可以使用分页或扩展页表机制（EPT）来完成。
  * 另一种方法是使用 “virtualize APIC accesses” VM 执行控制域置 `1` 。
* 如果 “virtualize APIC accesses” VM 执行控制域为 `1`，使用线性地址会转换为 `4 KB` **APIC access page** 中的物理地址，则逻辑处理器会特殊处理这些线性地址的内存访问。（APIC access page 由 **APIC-access address** 指出，它是 VMCS 中的一个字段；参见第 25.6.8 节。）
  1. 即使使用 EPT 转换地址（请参阅第 29.3 节），APIC 访问是否发生 VM exit 的决定也取决于访问的物理地址（HPA），而不是其 guest 物理地址（GPA）。即使 `CR0.PG = 0`，软件访问普通内存也使用线性地址；`CR0.PG = 0` 的事实仅意味着恒等转换用于将线性地址转换为物理（或 guest 物理）地址。
  2. 如果启用了 EPT，并且对 guest 物理地址进行写入，该地址转换为 APIC access page 上符合子页面写入权限的地址（请参阅第 29.3.4.1 节），则处理器可能会将 **写入** 就好像 “virtualize APIC accesses” VM 执行控制域为 `0`（并且不应用本节中指定的处理）时对待。因此，建议软件不要配置任何转换为 APIC access page 上的地址的 guest 物理地址，以获得子页面写入权限。
* 通常，对 APIC access page 的访问会导致 **APIC access VM exit**。APIC access VM exit 向 VMM 提供有关导致 VM 退出的访问的信息。第 30.4.1 节讨论了 APIC access VM exit 的优先级。
* 某些 VM 执行控制使处理器能够虚拟化对 APIC access page 的某些访问，而无需 VM exit。一般来说，这种虚拟化导致对 *虚拟APIC页面* 进行这些访问，而不是 *APIC access page* 。
* **注意**：
  * 除非另有说明，本节仅描述对 APIC access page 的线性访问；对 APIC access page 的访问是线性访问，如果
    1. 它是由使用线性地址的内存访问产生的；
    2. 访问的物理地址是该线性地址的转换。30.4.6 节讨论了对 APIC access page 的非线性访问。
  * APIC access page 和 *虚拟 APIC 页面* 之间的 **区别** 在于：
    * 允许 VMM 在一个虚拟机的虚拟处理器之间共享分页结构或 EPT 分页结构（引用相同 *APIC access address* 的 **共享** 分页结构，出现在所有虚拟处理器的 VMCS），
    * 而为每个虚拟处理器提供自己的虚拟 APIC（每个虚拟处理器的 VMCS 将具有 **唯一** 的 *虚拟 APIC 地址*）。
* 第 30.4.2 节讨论处理器何时以及如何虚拟化来自 APIC access page 的读取访问。
* 第 30.4.3 节对写访问执行相同的操作。当虚拟化对 APIC access page 的写入时，除了将写入传递到 *虚拟 APIC 页面* 之外，处理器通常还采取其他操作。
* 这部分的讨论中在可能发生这些内存访问中使用了 **操作** 的概念。对于这些讨论，“操作”可以是带有 `REP` 前缀的字符串指令的迭代、任何其他指令的执行或通过 IDT 传递事件。
* “virtualize APIC accesses” VM执行控制域置 `1` 也可能影响对不直接由线性地址产生的 APIC access page 的访问。这将在第 30.4.6 节中讨论。
* 特殊处理可能适用于 SGX 指令或逻辑处理器处于 enclave 模式的情况。详细信息请参见第 39.5.3 节

### 30.4.1 APIC-Access VM Exits 的优先级
* 以下各项指定 APIC access VM exit 相对于其他事件的优先级。
* 由于内存访问而导致的 APIC access VM exit 的优先级低于该访问可能导致的任何缺页或 EPT 违规。也就是说，如果访问会导致缺页或 EPT 违规，则不会导致 APIC access VM exit。
* 在分页结构（包括 EPT 分页结构，如果启用）中设置 accessed flags 之前，内存访问不会导致 APIC access VM exit。
* 在适当的分页结构和 EPT 分页结构（如果启用）中设置 drity flags 之前，写访问不会导致 APIC access VM exit。
* 对于所有其他事件，由于内存访问而导致的任何 APIC access VM exit 与该访问可能导致的任何缺页或 EPT 违规具有相同的优先级。（此项适用于该访问可能产生的其他事件以及同一操作的其他访问可能产生的事件。）
* 这些原则意味着，除其他外，APIC access VM exit 可能会在执行重复字符串指令（包括 `INS` 和 `OUTS`）期间发生。
  * 例如，假设此类指令的前 n 次迭代（n 可以是 `0`）不访问 APIC access，而下一次迭代确实访问该页面。因此，前 n 次迭代可能会完成，然后 APIC access VM exit。
  * VMCS 中保存的指令指针（`RIP`）引用重复的字符串指令，通用寄存器的值反映 n 次迭代的完成情况。

### 30.4.2 虚拟化从 APIC-Access Page 读取
* 如果满足以下任一条件，则来自 APIC access page 的读取访问会导致 APIC access VM exit：
  * “use TPR shadow” VM 执行控制域为 `0`。
  * 该访问用于取指令。
  * 访问大小超过 `32` 位。
  * 该访问是处理器已虚拟化对 APIC access page 写入的操作的一部分。
  * 访问并不完全包含在自然对齐的 `16` 字节区域的低 `4` 字节内。即访问地址的第 `3:2` 位为 `0`，访问的最高字节地址也是如此。
* 如果上述情况均不成立，则读取访问是否虚拟化取决于 “APIC-register virtualization” 和 “virtual-interrupt delivery” VM 执行控制的设置：
  * 如果页偏移量为 `0x080`（任务优先级）的读访问，则无论 “APIC-register virtualization” 和 “virtual-interrupt delivery” VM 执行控制的设置如何，都会被虚拟化。
  * 如果 “virtual-interrupt delivery” VM 执行控制为 `1`，则如果其页偏移量为`0x0B0`（end of interrupt）或 `0x300`（interrupt command — low），则该读取访问将被虚拟化。
  * 如果 “APIC-register virtualization” 为 `1`，则如果读访问完全在以下偏移范围之一内，则该读访问将被虚拟化：
  * 在所有其他情况下，访问都会导致 APIC access VM exit。
* **来自虚拟化的 APIC access page 的读取访问从 *虚拟 APIC 页面* 上的相应页面偏移返回数据。**
  * 用于从 *虚拟 APIC 页* 读取的访问的内存类型在 `IA32_VMX_BASIC` MSR 的 bit `53:50` 中报告（请参阅附录 A.1）。

### 30.6 Posted-interrupt Processing
* Posted-interrupt processing 是处理器通过在虚拟 APIC 页面上将虚拟中断记录为 pending 来处理虚拟中断的功能。
* 通过设置“process posted interrupts” VM 执行控制来启用 posted-interrupt processing。
* 该处理是响应具有 **posted-interrupt notification vector** 的中断的到来而执行的。
* 为了响应这种中断，处理器需处理记录在称为 **posted-interrupt descriptor** 的数据结构中的虚拟中断。
* *Posted-interrupt notification vector* 和 *posted-interrupt descriptor 的地址* 是 VMCS 中的域。
* 如果“process posted interrupts” VM 执行控制为`1`，则逻辑处理器使用位于 *posted-interrupt descriptor 地址* 的 `64-byte` *posted-interrupt descriptor*。
* *Posted-interrupt descriptor* 具有以下格式：

Bit 位置 | 名称 | 描述
--------|----------------------|--------------------
255:0   | Posted-interrupt 请求 | 每个中断向量一位。如果相应的 bit 为 `1`，该向量有一个 posted-interrupt 请求
256     | Outstanding 通知      | 如果设置了该位，则在 `255:0` bits 中存在一个或多个 posted interrupts 的未完成通知
511:257 | 保留给软件或其他 agent | 这些位可由软件和系统中的其他代理（例如芯片组）使用。处理器不会修改这些位

* 缩写 **PIR**（posted-interrupt requests）指的是 posted-interrupt descriptor 中的 `256` 个 posted-interrupt 位。
* Posted-interrupt descriptor 的使用与 VMCS 中指针引用的其他数据结构的使用不同。
  * 一般需要软件确保，只有在没有 *引用当前 VMCS 的逻辑处理器* 处于 VMX non-root operation 时，才修改每个这样的数据结构。
  * 该要求不适用于 posted-interrupt descriptor。但是，需要使用锁定的 read-modify-write 指令来完成此类修改。

* 仅当“external-interrupt exiting” VM 执行控制位也为 `1` 时，VM entry 确保“process posted interrupts” VM 执行控制位为 `1`
* 如果“external-interrupt exiting” VM 执行控制位为 `1`，则任何未屏蔽的外部中断都会导致 VM exit。
* 如果“process posted interrupts” VM 执行控制位也是 `1`，则此行为会改变，处理器会按如下方式处理外部中断：
1. 本地 APIC 被确认；这为处理器核提供了一个中断向量，这里称为 **物理向量**。
2. 如果 *物理向量* 等于 posted-interrupt notification 向量，则逻辑处理器继续下一步。否则，由于外部中断，通常会发生 VM exit；该向量保存在 VM-exit 中断信息字段中。
3. 处理器清除 posted-interrupt descriptor 中的 outstanding-notification 位。这是以原子方式完成的，以使描述符的其余部分保持不变（例如，使用锁定的`AND`操作）。
4. 处理器将零写入本地 APIC 中的 EOI 寄存器；这将解除来自本地 APIC 的 posted-interrupt notification 向量的中断。
5. 逻辑处理器将 PIR 与 VIRR 执行 *逻辑或*，并清除 PIR。在读取 PIR 位（以确定要用什么和 VIRR 进行或运算）和清除它之间，没有其他代理可以读取或写入 PIR 位（或一组位）。
6. 逻辑处理器将 RVI 设置为 RVI 的旧值和 PIR 中设置的所有位的最高索引中的最大值；如果在 PIR 中没有设置任何位，则 RVI 保持不变。
7. 逻辑处理器 evaluates pending 的虚拟中断，如第 29.2.1 节所述。
* 逻辑处理器以不可中断的方式执行上述步骤。如果第 7 步导致识别出虚拟中断，处理器可以立即传递该中断。
* 当中断控制器向 CPU 核提供未屏蔽的外部中断时，会发生上述 1 到 7 步。
* 以下 iterms 考虑了某些中断传递的情况：
  * 中断传递可以发生在以 `REP` 为前缀的指令的迭代之间（在至少一个迭代完成之后但在所有迭代完成之前）。如果发生这种情况，以下 items 会在 posted-interrupt 处理完成后和 guest 执行恢复之前表征处理器状态：
    * `RIP` 引用以 `REP` 为前缀的指令；
    * 更新 `RCX`、`RSI` 和 `RDI` 以反映已完成的迭代；和
    * `RFLAGS.RF = 1`。
  * 当逻辑处理器处于活动、`HLT` 或 `MWAIT` 状态时，可能会发生中断传递。
    * 如果逻辑处理器在中断到达之前处于活动或 `MWAIT` 状态，则在完成第 7 步后处于活动状态；
    * 如果它一直处于 `HLT` 状态，它会在第 7 步之后返回到 `HLT` 状态（如果识别到 pending 的虚拟中断，则逻辑处理器可以立即从 `HLT` 状态唤醒）。
  * 当逻辑处理器处于 enclave 模式时，可能会发生中断传递。
    * 如果逻辑处理器在中断到达之前处于 enclave 模式，则 Asynchronous Enclave Exit（AEX）可能会在第 1 到 7 步之前发生。
    * 如果在第 1 步之前没有发生 AEX，并且在第 2 步发生了 VM exit，则在传递 VM exit 之前会发生 AEX。