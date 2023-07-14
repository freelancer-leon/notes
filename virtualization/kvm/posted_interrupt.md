# Posted Interrupt
* 与 Posted interrupt 相关的三个中断向量
* 声明
  * arch/x86/include/asm/idtentry.h
```cpp
#ifdef CONFIG_HAVE_KVM
DECLARE_IDTENTRY_SYSVEC(POSTED_INTR_VECTOR,     sysvec_kvm_posted_intr_ipi);
DECLARE_IDTENTRY_SYSVEC(POSTED_INTR_WAKEUP_VECTOR,  sysvec_kvm_posted_intr_wakeup_ipi);
DECLARE_IDTENTRY_SYSVEC(POSTED_INTR_NESTED_VECTOR,  sysvec_kvm_posted_intr_nested_ipi);
#endif
```
* 注册
  * arch/x86/kernel/idt.c
```cpp
#ifdef CONFIG_X86_LOCAL_APIC
...
# ifdef CONFIG_HAVE_KVM
    INTG(POSTED_INTR_VECTOR,        asm_sysvec_kvm_posted_intr_ipi),
    INTG(POSTED_INTR_WAKEUP_VECTOR,     asm_sysvec_kvm_posted_intr_wakeup_ipi),
    INTG(POSTED_INTR_NESTED_VECTOR,     asm_sysvec_kvm_posted_intr_nested_ipi),
# endif
...
#endif
```
* 实现如下，可见它们都会应答中断 `ack_APIC_irq()`，并且增加相应的统计计数
  * arch/x86/kernel/irq.c
```cpp
/*
 * Handler for POSTED_INTERRUPT_VECTOR.
 */
DEFINE_IDTENTRY_SYSVEC_SIMPLE(sysvec_kvm_posted_intr_ipi)
{
    ack_APIC_irq();
    inc_irq_stat(kvm_posted_intr_ipis);
}

/*
 * Handler for POSTED_INTERRUPT_WAKEUP_VECTOR.
 */
DEFINE_IDTENTRY_SYSVEC(sysvec_kvm_posted_intr_wakeup_ipi)
{
    ack_APIC_irq();
    inc_irq_stat(kvm_posted_intr_wakeup_ipis);
    kvm_posted_intr_wakeup_handler();
}

/*
 * Handler for POSTED_INTERRUPT_NESTED_VECTOR.
 */
DEFINE_IDTENTRY_SYSVEC_SIMPLE(sysvec_kvm_posted_intr_nested_ipi)
{
    ack_APIC_irq();
    inc_irq_stat(kvm_posted_intr_nested_ipis);
}
```
## Posted Interrupt 通知向量
* 对于 posted interrupt 的 notification vector - `POSTED_INTR_VECTOR`，通常会判断目标 vCPU 是不是处于 `GUEST_MODE`。
  * 如果是，发送 NV IPI。但无法保证目标 vCPU 在 IPI 到达 CPU 的时候没有被调度出去，这时 VMM 会收到到这 NV，因此也需要在 host 侧注册 NV 的 dummy handler `sysvec_kvm_posted_intr_ipi()`
  * 如果不是，则走常规的中断注入路径，等目标 vCPU 再次运行（VM entry）的时候自然就会处理注入的中断
  ```c
  //arch/x86/kvm/lapic.c
  __apic_accept_irq()
  case APIC_DM_FIXED:
  -> static_call(kvm_x86_deliver_interrupt)()
  //arch/x86/kvm/vmx/vmx.c
  => vmx_deliver_interrupt()
  ```
* 虚拟向量已在 `PIR` 中设置。发送 notification 事件以传递虚拟中断，除非 **vCPU 是当前正在运行的 vCPU**（那此时是处于 root mode），即该事件是从快速路径 VM-Exit 处理程序发送的，在这种情况下，`PIR` 将在重新进入 guest 虚拟机之前同步到 `vIRR`。
  * 所以可不发 IPI 就 `return`，这就是第一个 `return`
* 当目标不是正在运行的 vCPU 时，会出现以下可能性：
  * 情况 1：vCPU 处于 non-root 模式。发送 notification 事件将中断发布（post）到 vCPU。
  * 情况 2：vCPU 退出到 root 模式并且仍然可以运行。在重新进入 guest 之前，`PIR` 将同步到 `vIRR`。发送 notification 事件是可以的，因为 host IRQ 处理程序将忽略spurious 事件。
  * 情况 3：vCPU 退出到 root 模式并被阻塞（调度出去）。`vcpu_block()` 已将 `PIR` 同步到 `vIRR`，并且如果 `vIRR` 不为空，则永远不会阻塞 vCPU。因此，此处阻塞的 vCPU 不会等待 `PIR` 中的任何请求中断，并且发送 notification 事件也会导致 benign，spurious 事件。
* vCPU 不在 guest；唤醒 vCPU，以防其阻塞，否则不执行任何操作，因为 KVM 将通过 `vcpu_enter_guest()` 中的 `->sync_pir_to_irr()` 获取最高优先级的挂起 IRQ。
* arch/x86/kvm/vmx/vmx.c
```cpp
static inline void kvm_vcpu_trigger_posted_interrupt(struct kvm_vcpu *vcpu,
                             int pi_vec)
{
#ifdef CONFIG_SMP
    if (vcpu->mode == IN_GUEST_MODE) {
        /*
         * The vector of the virtual has already been set in the PIR.
         * Send a notification event to deliver the virtual interrupt
         * unless the vCPU is the currently running vCPU, i.e. the
         * event is being sent from a fastpath VM-Exit handler, in
         * which case the PIR will be synced to the vIRR before
         * re-entering the guest.
         *
         * When the target is not the running vCPU, the following
         * possibilities emerge:
         *
         * Case 1: vCPU stays in non-root mode. Sending a notification
         * event posts the interrupt to the vCPU.
         *
         * Case 2: vCPU exits to root mode and is still runnable. The
         * PIR will be synced to the vIRR before re-entering the guest.
         * Sending a notification event is ok as the host IRQ handler
         * will ignore the spurious event.
         *
         * Case 3: vCPU exits to root mode and is blocked. vcpu_block()
         * has already synced PIR to vIRR and never blocks the vCPU if
         * the vIRR is not empty. Therefore, a blocked vCPU here does
         * not wait for any requested interrupts in PIR, and sending a
         * notification event also results in a benign, spurious event.
         */

        if (vcpu != kvm_get_running_vcpu())
            apic->send_IPI_mask(get_cpu_mask(vcpu->cpu), pi_vec);
        return;
    }
#endif
    /*
     * The vCPU isn't in the guest; wake the vCPU in case it is blocking,
     * otherwise do nothing as KVM will grab the highest priority pending
     * IRQ via ->sync_pir_to_irr() in vcpu_enter_guest().
     */
    kvm_vcpu_wake_up(vcpu);
}
```
* 通过 posted interrupt 方式向 vcpu 发送中断会遇到的情况：
  1. 如果目标 vcpu 正在运行（non-root mode），则向 vcpu 发送 posted interrupt notification，硬件将自动将 `PIR` 同步到 `vIRR`。
  2. 如果目标 vcpu 未运行（root mode），则启动它以在下一个 VM entry 中从 `PIR` 获取中断。
* 在 `vcpu_enter_guest()` 中设置 `vcpu->mode` 之后，`pi_test_and_set_on()` 中隐含的屏障与 `smp_mb_*()` 配对。因此，如果触发 posted interrupt “失败”，vCPU 保证看到 `PID.ON=1` 并将 `PIR` 同步到 `IRR`，因为 `vcpu->mode != IN_GUEST_MODE`。
```c
/*
 * Send interrupt to vcpu via posted interrupt way.
 * 1. If target vcpu is running(non-root mode), send posted interrupt
 * notification to vcpu and hardware will sync PIR to vIRR atomically.
 * 2. If target vcpu isn't running(root mode), kick it to pick up the
 * interrupt from PIR in next vmentry.
 */
static int vmx_deliver_posted_interrupt(struct kvm_vcpu *vcpu, int vector)
{
    struct vcpu_vmx *vmx = to_vmx(vcpu);
    int r;

    r = vmx_deliver_nested_posted_interrupt(vcpu, vector);
    if (!r)
        return 0;
    //如果 local APIC 虚拟化激活了才有可能往下走
    /* Note, this is called iff the local APIC is in-kernel. */
    if (!vcpu->arch.apic->apicv_active)
        return -1;
    //设置 PI 描述符中的 PIR 字段中对应 vector 的 bit
    if (pi_test_and_set_pir(vector, &vmx->pi_desc))
        return 0;
    //如果之前已经发过 IPI posted interrupt NV 了，就不要继续往下走了
    /* If a previous notification has sent the IPI, nothing to do.  */
    if (pi_test_and_set_on(&vmx->pi_desc))
        return 0;

    /*
     * The implied barrier in pi_test_and_set_on() pairs with the smp_mb_*()
     * after setting vcpu->mode in vcpu_enter_guest(), thus the vCPU is
     * guaranteed to see PID.ON=1 and sync the PIR to IRR if triggering a
     * posted interrupt "fails" because vcpu->mode != IN_GUEST_MODE.
     */
    kvm_vcpu_trigger_posted_interrupt(vcpu, POSTED_INTR_VECTOR);
    return 0;
}
```
* 如果 local APIC 虚拟化未激活，无法通过 posted interrupt 注入中断，这是就要回归传统方式注入中断了
  * 更新 KVM `IRR`
  * 创建 `KVM_REQ_EVENT` 请求，`vcpu_enter_guest()` 会检查该请求并作出响应，比如 `kvm_check_and_inject_events()`
  * `kvm_vcpu_kick()` 主要是针对目标 vCPU 处于 *阻塞* 或 *non-root mode* 的情况，效果是：
    * 如果目标 vCPU 处于阻塞状态，则唤醒它，尽快处理中断
    * 如果目标 vCPU 处于 non-root mode，则发送 reschedule IPI 让它尽快 VM exit，以便再次 VM entry 时注入中断
    * 如果目标 vCPU 处于 root mode 不需要做什么，因为上面已经更新了 `IRR`，目标 vCPU 再次 VM entry 时注入中断
* 否则此时需要做的仅仅是触发 trace event
```c
static void vmx_deliver_interrupt(struct kvm_lapic *apic, int delivery_mode,
                  int trig_mode, int vector)
{
    struct kvm_vcpu *vcpu = apic->vcpu;

    if (vmx_deliver_posted_interrupt(vcpu, vector)) {
        kvm_lapic_set_irr(vector, apic);
        kvm_make_request(KVM_REQ_EVENT, vcpu);
        kvm_vcpu_kick(vcpu);
    } else {
        trace_kvm_apicv_accept_irq(vcpu->vcpu_id, delivery_mode,
                       trig_mode, vector);
    }
}
```

## Posted Interrupt 唤醒向量

### Posted Interrupt 唤醒向量的处理
* `POSTED_INTR_WAKEUP_VECTOR` 的处理函数需要专门设置，否则就是简单的同步 RCU
  * arch/x86/kernel/irq.c
```cpp
void kvm_set_posted_intr_wakeup_handler(void (*handler)(void))
{
    if (handler)
        kvm_posted_intr_wakeup_handler = handler;
    else {
        kvm_posted_intr_wakeup_handler = dummy_handler;
        synchronize_rcu();
    }
}
```
* `POSTED_INTR_WAKEUP_VECTOR` 的处理函数是在 VM 初始化时设置的 `pi_wakeup_handler()`
  * arch/x86/kvm/vmx/vmx.c
```cpp
static __init int hardware_setup(void)
{
...
    kvm_set_posted_intr_wakeup_handler(pi_wakeup_handler);
...
}
```
* arch/x86/kvm/vmx/posted_intr.c
```cpp
/*
 * Handler for POSTED_INTERRUPT_WAKEUP_VECTOR.
 */
void pi_wakeup_handler(void)
{
    int cpu = smp_processor_id();
    struct list_head *wakeup_list = &per_cpu(wakeup_vcpus_on_cpu, cpu); //得到 per CPU 的唤醒链表
    raw_spinlock_t *spinlock = &per_cpu(wakeup_vcpus_on_cpu_lock, cpu);
    struct vcpu_vmx *vmx;

    raw_spin_lock(spinlock);
    list_for_each_entry(vmx, wakeup_list, pi_wakeup_list) { //遍历唤醒链表上的 vCPU

        if (pi_test_on(&vmx->pi_desc))   //如果 PID 的 PIR 上设置 pending 的中断
            kvm_vcpu_wake_up(&vmx->vcpu);//此时需要唤醒该 vCPU，也就是 wake_up_process() vCPU thread
    }
    raw_spin_unlock(spinlock);
}
```

## References
* [Intel VT-d（4）- Interrupt Posting - 知乎](https://zhuanlan.zhihu.com/p/51018597)