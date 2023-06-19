# 25 Virtual Machine Control Structure

## 25.4 Guest-State Area
* 本节描述 VMCS 的 Guest 状态域中包含的字段。VM entry 从这些域加载处理器状态，VM exit 将处理器状态存储到这些域中

### 25.4.1 Guest Register State
### 25.4.2 Guest Non-Register State

#### Interruptibility state（32 bits）
* IA-32 体系结构包括允许某些事件在一段时间内被屏蔽的功能。该域包含有关此类屏蔽的信息
* Table 25-3. Interruptibility State 的格式

Bit 位置 | bit 名称             | 描述
---------|---------------------|-----------------------------------------
0       | 被 `STI` 屏蔽         | 在 `RFLAGS.IF = 0` 的情况下执行 `STI` 会在其执行后屏蔽指令边界上的可屏蔽中断。`1` 设置此位表示此屏蔽有效
1       | 被 `MOV SS` 屏蔽      | 执行 `MOV` 到 `SS` 或 `POP` 到 `SS` 会屏蔽或抑制某些调试异常以及执行后指令边界上的中断（可屏蔽和不可屏蔽）。设置此位表示此屏蔽生效。本文档使用术语 *被 `MOV SS` 屏蔽*，但它同样适用于 `POP SS`
2       | 被 SMI 屏蔽           | 设置此位表示 SMI 的屏蔽生效
3       | 被 NMI 屏蔽           | 传递不可屏蔽中断 (NMI) 或系统管理中断 (SMI) 会阻止后续 NMI，直到下一次执行 `IRET`。请参阅第 26.3 节了解 `IRET` 的这种行为在 VMX non-root operation 中如何改变。设置该位表示 NMI 屏蔽生效。清除该位并不意味着 NMI 不会（暂时）因其他原因而被屏蔽。如果 “virtual NMIs” VM 执行控制域（参见第 25.6.1 节）为 `1`，则该位不控制 NMI 的屏蔽。相反，它指的是 “virtual-NMI blocking”（guest 软件还没有为 NMI 做好准备）
4       | Enclave interruption | 如果在逻辑处理器处于 enclave 模式时发生 VM exit，则设置为 `1`。此类 VM exit 包括由中断、不可屏蔽中断、系统管理中断、INIT 信号和 enclave 模式中发生的异常以及在将此类事件事件传递到 enclave 模式期间遇到的异常引起的退出。与 VM entry 注入的事件交付相关的 VM exit 不修改该位
5       | 保留                 | 如果这些位不为 `0`，VM entry 将会失败

## 25.6 VM-Execution Control Field

* VM-Execution Control 域管理 VMX non-root operation

### 25.6.1 Pin-Based VM-Execution Controls
* 基于引脚的 VM 执行控制域构成一个 `32` 位向量，用于管理异步事件（例如：中断）的处理
  * 无论基于引脚的 VM 执行控件域的设置如何，一些异步事件都会导致 VM exit
* Table 25-5. Pin-Based VM-Execution Controls 的定义

Bit 位置 | 名称                        | 描述
---------|----------------------------|-----------------------------------------
0        | External-interrupt exiting | 如果此控制位为 `1`，则外部中断导致 VM exit。否则，它们将通过 guest 的中断描述符表（IDT）正常传送。如果此控制位为 `1`，则 `RFLAGS.IF` 的值不影响中断屏蔽
3        | NMI exit                   | 如果此控制位为 `1`，则不可屏蔽中断 (NMI) 会导致 VM exit。否则，它们通常使用 IDT 的描述符 `2` 传送。此控制位还确定 `IRET` 与 NMI 阻塞之间的交互（参见第 26.3 节）
5        | Virtual NMIs               | 如果此控制位为 `1`，则 NMI 永远不会被屏蔽，并且中断状态域中的 “blocking by NMI” 位（第 3 位）指示 “virtual-NMI blocking”（参见表 25-3）。此控制位还与 “NMI-window exiting” VM 执行控制域交互（请参阅第 25.6.2 节）
6        | Activate VMXpreemption timer | 如果此控制位为 `1`，则 VMX non-root operation 中 VMX-preemption timer 倒计时；参见第 26.5.1 节。当计时器倒计时到零时，VM exit；参见第 26.2 节
7       | Process posted interrupts | 如果此控制位为 `1`，则处理器将使用 posted-interrupt notification 向量（参见第 25.6.8 节）特殊处理中断，并使用posted-interrupt requests 更新 *虚拟 APIC 页面*（参见第 30.6 节）