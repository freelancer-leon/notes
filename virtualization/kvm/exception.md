# 异常处理
* 异常（故障、陷阱和中止）导致 VM 根据 *异常位图（exception bitmap）* VM exit。如果发生异常，它的向量（在 0-31 范围内）用于在异常位图中选择一个位
  * 如果该位为 `1`，则发生 VM exit；
  * 如果该位为 `0`，则通过 guest IDT 正常传递异常
* 异常位图的这种使用也适用于由指令 `INT1`、`INT3`、`INTO`、`BOUND`、`UD0`、`UD1` 和 `UD2`
  * `INT1` 和 `INT3` 分别指操作码为 `F1` 和 `CC` 的指令，而不是指 `n` 值为 `1` 或 `3` 的 `INT n`

## 缺页异常
* Guest 发生缺页异常（向量 `14` 的异常）被特殊处理。当缺页异常发生时，处理器会咨询：
  1. VMCS 异常位图的第`14`位
  2. 缺页异常产生的错误码`[PFEC]`
  3. VMCS 缺页异常错误码的掩码字段`[PFEC_MASK]`（VMCS 32-Bit Control Fields，Page-fault error-code mask）
  4. VMCS 缺页异常错误码的匹配字段`[PFEC_MATCH]`（VMCS 32-Bit Control Fields，Page-fault error-code match）
* 处理器检查 `if PFEC & PFEC_MASK == PFEC_MATCH`
  * 如果相等，则遵循异常位图中第 `14` 位的规范（例如，如果设置了该位，则会发生 VM exit）
  * 如果不相等，则该位的含义相反（例如，如果该位被清除，则会发生 VM exit）
* 因此，如果软件希望在所有缺页异常时 VM exit，它可以将异常位图中的第`14`位设置为`1`，并将缺页异常错误码的 *mask* 和 *match* 字段分别设置为 `0x00000000`
  * 译注：此时 `if PFEC & PFEC_MASK == PFEC_MATCH` 必然相等
* 如果软件希望在缺页异常时不会 VM exit（VM exits on no page faults），它可以将异常位图中的第`14`位设置为`1`，将缺页异常错误码的 *mask* 字段设置为 `0x00000000`，并将缺页异常错误码的 *match* 字段设置为 `0xFFFFFFFF`
  * 译注：此时 `if PFEC & PFEC_MASK == PFEC_MATCH` 必然不等

## 虚拟化异常
* 异常向量为`20`
* 异常缩写为`#VE`
* 仅发生在 VMX non-root operation
* 当处理器遇到虚拟化异常时，将异常信息保存到 **虚拟化异常信息区（virtualization-exception information area）**
* 保存虚拟化异常信息后，处理器会像处理任何其他异常一样提供虚拟化异常
* 虚拟化异常的传递会将值`0xFFFFFFFF`写入虚拟化异常信息区域中的偏移量`4`的位置
  * 因此，一旦发生虚拟化异常，只有在软件清除该字段时才会发生另一个异常

### 虚拟化异常信息

* 虚拟化异常信息区的格式

字节偏移 | 内容
---------|------
0  | 如果发生 VM exit 而不是虚拟化异常，则将`32`位值作为退出原因保存到 VMCS 中 。对于 EPT violations，此值为`48 (00000030H)`
4  | `0xFFFFFFFF`
8  | 如果发生 VM exit 而不是虚拟化异常，则将`64`位值作为退出条件（qualification）保存到 VMCS 中
16 | 如果发生 VM exit 而不是虚拟化异常，则将 guest 线性地址的`64`位值保存到 VMCS 中
24 | 如果发生 VM exit 而不是虚拟化异常，则将 guest 物理地址的`64`位值保存到 VMCS 中
32 | 当前由 EPTP 索引 的 VM 执行控制的`16`位值

* VMM 可以允许 guest 软件访问虚拟化异常信息区域
  * 如果是这样，guest 软件可能会修改该内存（例如，清除偏移量`4`处的`32`位值）

### 虚拟化异常的递交
* 保存虚拟化异常信息后，处理器会像处理其他异常一样处理虚拟化异常：
  * 如果 VMCS 的异常位图中的第`20 bit` (`#VE`) 为`1`，则虚拟化异常会导致 VM exit（见下文）。如果该位为`0`，则使用 IDT 中的门描述符`20`传递虚拟化异常
  * 虚拟化异常不会产生 error code。对于虚拟化异常的递交，CPU 不会将 error code 推送到堆栈上
  * 对于 double fault，虚拟化异常与 page fault 具有相同的 serverity
    * 如果虚拟化异常的递交遇到嵌套 fault（contributory faults 或 page fault），则会生成 double fault (`#DF`)
* 在递交另一个异常时不可能遇到虚拟化异常
* 如果虚拟化异常直接导致 VM exit（因为异常位图中的`bit 20`为`1`），异常信息正常保存在 VMCS 的 VM-exit *interruption information* 字段中
  * 具体来说，该事件被报告为异常向量`20`且没有 error code 的硬件异常。正常来说，该字段的第`12`位（由于`IRET`导致 NMI 解锁）会被设置
* 如果虚拟化异常间接导致 VM exit（因为异常位图中的`bit 20`为`0`，并且异常的递交会生成导致 VM exit 的事件），则有关异常的信息通常保存在 VMCS
  * 具体来说，该事件被报告为异常向量`20`且没有 error code 的硬件异常

## 关于 Acknowledge Interrupt on VM-Exit
* VMX 有一个 acknowledge interrupt on exit 特性，《系统虚拟化原理》5.3.2 曾提到该特性有助于更快地响应外部中断，具体怎么快没说
* 该特性通过 VMCS 中的 VM-Exit 控制域的第 `15` 位 `Acknowledge interrupt on exit` 来启用，看看 SDM 怎么说：

> This control affects VM exits due to external interrupts:
> * If such a VM exit occurs and this control is `1`, the logical processor acknowledges the interrupt controller, acquiring the interrupt’s vector. The vector is stored in the VM-exit interruption-information field, which is marked valid.
> * If such a VM exit occurs and this control is `0`, the interrupt is not acknowledged and the VM-exit interruption-information field is marked invalid.

* 就是说，当 VM exit 发生的时候，如果该控制位设为 `1`，那么逻辑处理器会先应答中断控制器，获取中断向量并把它放在 VM-exit 的中断信息域；如果该控制位设为 `0`，那么 CPU 就不会应答该中断。这怎么就能加速外部中断的响应了呢？
* Linux kernel 在以下 [commit](https://lore.kernel.org/all/1365679516-13125-2-git-send-email-yang.z.zhang@intel.com/) 中默认启用了该特性
  ```git
  commit a547c6db4d2f16ba5ce8e7054bffad6acc248d40
  Author: Yang Zhang <yang.z.zhang@Intel.com>
  Date:   Thu Apr 11 19:25:10 2013 +0800

      KVM: VMX: Enable acknowledge interupt on vmexit

      The "acknowledge interrupt on exit" feature controls processor behavior
      for external interrupt acknowledgement. When this control is set, the
      processor acknowledges the interrupt controller to acquire the
      interrupt vector on VM exit.

      After enabling this feature, an interrupt which arrived when target cpu is
      running in vmx non-root mode will be handled by vmx handler instead of handler
      in idt. Currently, vmx handler only fakes an interrupt stack and jump to idt
      table to let real handler to handle it. Further, we will recognize the interrupt
      and only delivery the interrupt which not belong to current vcpu through idt table.
      The interrupt which belonged to current vcpu will be handled inside vmx handler.
      This will reduce the interrupt handle cost of KVM.

      Also, interrupt enable logic is changed if this feature is turnning on:
      Before this patch, hypervior call local_irq_enable() to enable it directly.
      Now IF bit is set on interrupt stack frame, and will be enabled on a return from
      interrupt handler if exterrupt interrupt exists. If no external interrupt, still
      call local_irq_enable() to enable it.

      Refer to Intel SDM volum 3, chapter 33.2.
  ```
* 该 commit 主要做了以下事情支持该特性：
  * 在代码中设置相应的 bit，默认启用了该特性
  * 新增的 `vmx_handle_external_intr()` 作为 `kvm_x86_ops->handle_external_intr` 的方法
  * 在 `vcpu_enter_guest()` 调用新增的 `vmx_handle_external_intr()`
    * 在 VMX handler 中先根据 VMCS 中的中断退出信息找到外部中断向量
    * 根据 vector 找到该中断的 host 侧的 handler
    * 伪造一个中断发生时 CPU 的压栈现场
    * 通过修改压到栈上的 `RFLAGS.IF` 位为 `1`，让将来在 `iret` 的时候顺带开启中断
    * `call` 该中断的 host 侧的 handler
* 再看看 SDM 对 VM-execution 域的 `external-interrupt exiting` 执行控制位的解释：

> If this control is `1`, external interrupts cause VM exits. Otherwise, they are delivered normally through the guest interrupt-descriptor table (IDT). If this control is 1, the value of `RFLAGS.IF` does not affect interrupt blocking.
>
> -- Table 24-5. Definitions of Pin-Based VM-Execution Controls

  * 当设为 `1` 时，不透传给 Guest 的中断会导致 VM-exit
  * 否则中断直接透传给 Guest，根据 VMCS 中的 Guest IDTR 找到 Guest IDT，然后从 Guest 的中断入口开始处理中断
  * 这里说的，该位为 `1` 时，`RFLAGS.IF` 不影响中断的屏蔽，应该指的是 Guest 的 `RFLAGS.IF` 不影响 Host 侧的中断屏蔽与否
* 仍然没有回答一个问题，或者说，不使用该特性到底慢在哪？问题的症结在于，`external-interrupt exiting` 执行控制位为 `1` 时，CPU 的行为和中断控制器的状态是怎么样的？
* 在旧版本 SDM 还有这样一段话，新版本已经删掉了，给了我们一些启示：

> **33.2 INTERRUPT HANDLING IN VMX OPERATION**
>
> **Acknowledge interrupt on exit**. The “acknowledge interrupt on exit” VM-exit control in the controlling VMCS controls processor behavior for external interrupt acknowledgement. If the control is 1, the processor acknowledges the interrupt controller to acquire the interrupt vector upon VM exit, and stores the vector in the VM-exit interruption-information field. If the control is 0, the external interrupt is not acknowledged during VM exit. Since RFLAGS.IF is automatically cleared on VM exits due to external interrupts, VMM re-enabling of interrupts (setting RFLAGS.IF = 1) initiates the external interrupt acknowledgement and vectoring of the external interrupt through the monitor/host IDT

* 看到最后一句话了吗？这意味着，
  * 外部中断导致 VM-Exit 的时候，`RFLAGS.IF`会被自动清除，此时 host 侧中断必然是关着的！
  * 需要 VMM 重新开启中断，从而完成应答外部中断和递交外部中断到 host 侧的 IDT 入口的过程
* 也就是说（以 8259A 作为中断控制器举例），
1. `Acknowledge Interrupt on Exit = 0` 会导致 VM-exit 后该中断在 8259A 的 `IRR` 中 pending
2. 直到 VMM 使能中断，这个时候 CPU 通过管脚 `INTR` 知道 8259A 有中断等待处理，通过管脚 `INTA` 第一次中断应答
3. 8259A 收到 CPU 发来的 `INTA` 信号后，置位最高优先级的中断在 `ISR(In-Service Register)` 中对应的位，并清空 `IRR` 中对应的位
4. 通常，x86 CPU 会发送第二次 `INTA`，在收到第二次 `INTA` 后，8259A 会将中断向量号(vector)送上数据总线 `D0~D7`
5. 如果 8259A 设置为 `AEOI(Automatic End Of Interrupt)` 模式，那么 8259A 复位 `ISR` 中对应的 bit，否则 `ISR` 中对应的 bit 一直保持到收到系统的中断服务程序发来的 `EOI` 命令
6. 再往后就是 host 侧常规的中断处理过程，先根据 IDTR 找到 IDT table ......
7. 在系统的中断服务程序处理完中断的最后，发来 `EOI` 命令，8259A 复位 `ISR` 中对应的 bit
* 对比 `Acknowledge Interrupt on Exit = 1`，VM-exit 时就已经完成了第 1 ~ 5 步，并且 host 中断处于关闭状态，VM-exit 后处于 VMX handler 中，于是那个 commit 所要做的工作就是衔接上第 6 步就可以了。
* 我的理解，虽然说 `vcpu_enter_guest()` 在没那个 commit 前在那个位置调用的也是 `local_irq_enable();` 来开始第 2 步，但理论上它也可以往后放。比起现在的响应方式终究还是慢一点吧。

## VMX Posted Interrupt Processing

* arch/x86/include/asm/vmxfeatures.h
```cpp
#define VMX_FEATURE_POSTED_INTR     ( 0*32+  7) /* Posted Interrupts */
```
* arch/x86/include/asm/vmx.h
```cpp
#define VMCS_CONTROL_BIT(x) BIT(VMX_FEATURE_##x & 0x1f)
...
#define PIN_BASED_POSTED_INTR                   VMCS_CONTROL_BIT(POSTED_INTR)
```
* arch/x86/kvm/vmx/vmx.c
```cpp
static u32 vmx_pin_based_exec_ctrl(struct vcpu_vmx *vmx)
{
    u32 pin_based_exec_ctrl = vmcs_config.pin_based_exec_ctrl;

    if (!kvm_vcpu_apicv_active(&vmx->vcpu))
        pin_based_exec_ctrl &= ~PIN_BASED_POSTED_INTR;

    if (!enable_vnmi)
        pin_based_exec_ctrl &= ~PIN_BASED_VIRTUAL_NMIS;

    if (!enable_preemption_timer)
        pin_based_exec_ctrl &= ~PIN_BASED_VMX_PREEMPTION_TIMER;

    return pin_based_exec_ctrl;
}
...
static void init_vmcs(struct vcpu_vmx *vmx)
{
...
    /* Control */
    pin_controls_set(vmx, vmx_pin_based_exec_ctrl(vmx));

    exec_controls_set(vmx, vmx_exec_control(vmx));
...
    if (enable_apicv && lapic_in_kernel(&vmx->vcpu)) {
        vmcs_write64(EOI_EXIT_BITMAP0, 0);
        vmcs_write64(EOI_EXIT_BITMAP1, 0);
        vmcs_write64(EOI_EXIT_BITMAP2, 0);
        vmcs_write64(EOI_EXIT_BITMAP3, 0);

        vmcs_write16(GUEST_INTR_STATUS, 0);

        vmcs_write16(POSTED_INTR_NV, POSTED_INTR_VECTOR);  //设置 posted interrupt notification vector
        vmcs_write64(POSTED_INTR_DESC_ADDR, __pa((&vmx->pi_desc))); //设置 posted interrupt descriptor 的物理地址
    }
...
}
```

# References
* Intel SDM Vol3: Chapter 27.2: Other Causes of VM Exits
* [Linux中断虚拟化之二](https://www.51cto.com/article/693199.html)
* [Intel SDM Chapter 29: APIC Virtualizaton & Virtual Interrupts](https://tcbbd.moe/ref-and-spec/intel-sdm/sdm-vmx-ch29/)
* [StackOverflow - "Acknowledge interrupt on exit" control in VT-x causes CPU lockup](https://stackoverflow.com/questions/48030293/acknowledge-interrupt-on-exit-control-in-vt-x-causes-cpu-lockup)