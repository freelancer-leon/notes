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
* 对于 posted interrupt 的 notification vector（NV） - `POSTED_INTR_VECTOR`，通常会判断目标 vCPU 是不是处于 `GUEST_MODE`。
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
* 虚拟向量已在 `PIR` 中设置。发送 notification 事件以传递虚拟中断，除非 **vCPU 是当前正在运行的 vCPU**（`vcpu != kvm_get_running_vcpu()`，此时是处于 root mode），即该事件是从快速路径 VM-Exit handler 发送的，在这种情况下，`PIR` 将在重新进入 guest 虚拟机之前同步到 `vIRR`。
  * 所以可不发 IPI 就 `return`，这就是第一个 `return`
* 当目标不是正在运行的 vCPU 时，会出现以下可能性：
  * 情况 1：vCPU 处于 non-root 模式。发送 notification 事件将中断发布（post）到 vCPU。
  * 情况 2：vCPU 退出到 root 模式并且仍然可以运行。在重新进入 guest 之前，`PIR` 将同步到 `vIRR`。发送 notification 事件是可以的，因为 host IRQ 处理程序将忽略 spurious 事件。
  * 情况 3：vCPU 退出到 root 模式并被阻塞（调度出去）。`vcpu_block()` 已将 `PIR` 同步到 `vIRR`，并且如果 `vIRR` 不为空，则永远不会阻塞 vCPU。因此，此处阻塞的 vCPU 不会等待 `PIR` 中的任何请求中断，并且发送 notification 事件也会导致 benign，spurious 事件。
* vCPU 不在 guest mode；唤醒 vCPU，以防其阻塞，否则不执行任何操作，因为 KVM 将通过 `vcpu_enter_guest()` 中的 `->sync_pir_to_irr()` 获取最高优先级的挂起 IRQ。
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
* 虚拟设备通过 posted interrupt 方式向 vcpu 发送中断会遇到的情况：
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
* 以上是虚拟设备通过 posted interrupt 方式向 vcpu 发送中断的处理。对于物理外设产生的中断：
  * 如果该设备未分配给 guest，且 `external-interrupt exiting` 执行控制位为 `1` 时，会造成 VM-exit，由 host 侧来处理该中断
    * `process posted interrupts` 执行控制位为 `1` 时，`external-interrupt exiting` 会强制为 `1`
    * `external-interrupt exiting` 执行控制位为 `0` 时，该中断会直接投递给 guest，并跳到 guest IDT 给定的中断向量入口。但实践中几乎不会这么配
  * 如果该设备分配给 guest，如 VFIO 场景：
    * 如果有中断重映射的硬件支持，可以将分配给 guest 的设备产生的中断直接通过 posted interrupt 的方式透传给 guest（这是推荐的方式）
    * 如果没有中断重映射的硬件支持，该中断会先投递到 host，从 host IDT 的中断向量入口开始执行，host 侧的中断处理逻辑发现该中断的所属设备分配给了 guest VM，则需要给 guest 注入中断

* Posted Interrupt 唤醒向量也就是 SDM 中所说的 `WNV`，用于 IPI 虚拟化和 VT-d posted interrupt 的场景。
* 对于外设中断重映射的 posted interrupt（即 VT-d posted-interrupts），也存在几种场景：
1. **vCPU 在运行（Running）**。中断重映射硬件根据 poseted interrupt IRTE 的信息，找到 `PID`，设置 `PIR`，发送 `ANV`，完成整个中断注入过程
   * `ANV` 与 VMCS 中的 `NV` 是一致的，因此整个过程无需软件参与
2. **vCPU 被抢占（Preempted）**。vCPU 所在线程仍在运行队列上。
   * 由于不希望被干扰，因此被调度出去时设置了 `PID` 里的 `SN` bit，除非是紧急的中断（设置了 IRTE 中的 `URG` 域），这样中断重映射硬件仅设置 `PIR`，而无需发送 `ANV`
   * 等到 vCPU 线程下次被调度重新进入 guest 时，
     * 清除 `PID` 中的 `SN`
     * 对于 vCPU 被调度到其他 CPU 的情况，需要更新 `PID` 中的 `NDST` 域，设置 `NV` 为 `ANV`
     * 把 `PIR` 同步到 `vIRR`，完成中断的注入（这是已有的逻辑）
3. **vCPU 在睡眠（Blocked）**。
   * 睡眠之前 KVM 需要将 `PID` 中的 `NV` 的值换成 `WNV`（与 VMCS 中的 `NV` 仍然是 `ANV` 的值，保持不变），这样当中断重映射硬件找到 `PID`，设置 `PIR`，发送的通知向量则是 `WNV`
   * 根据 `vcpu->cpu`，vCPU 线程需要放到 per CPU 的唤醒队列上（`DEFINE_PER_CPU(struct list_head, wakeup_vcpus_on_cpu)`）
   * 当外设中断被重定向后，由于 `WNV` 与 VMCS 中的 `NV` 的值（`ANV`）不一致，硬件不会继续 posted interrupt processing，而是当作 host 侧的一个中断来处理，也就是跳到 Posted Interrupt 唤醒向量的处理函数 `wakeup_handler()`。
   * Posted Interrupt 唤醒向量的处理函数要做的事情非常简单，就是将唤醒队列上 vCPU 的 `PID` 的 `Outstanding（ON）` 域被设置的 vCPU 线程唤醒
   * 等到 vCPU 线程下次被调度重新进入 guest 时，
     * 根据 `vcpu->cpu` 将 vCPU 线程从 per CPU 的唤醒队列上移除
     * `PID` 中的 `NV` 的值恢复为 `ANV`
     * 更新 `PID` 中的 `NDST` 域为唤醒后运行的物理 CPU ID
     * 把 `PIR` 同步到 `vIRR`，完成中断的注入（这是已有的逻辑）

### 设置 Posted Interrupt 唤醒向量的处理函数
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
```c
//arch/x86/kvm/vmx/vmx.c
module_init(vmx_init)
vmx_init()
   //arch/x86/kvm/x86.c
-> kvm_x86_vendor_init(&vmx_init_ops)
   -> __kvm_x86_vendor_init()
      -> ops->hardware_setup()
         //arch/x86/kvm/vmx/vmx.c
      => hardware_setup()
```
* arch/x86/kvm/vmx/vmx.c
```cpp
static __init int hardware_setup(void)
{
...
    kvm_set_posted_intr_wakeup_handler(pi_wakeup_handler);
...
}
```
### Posted Interrupt 唤醒向量的处理
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

        if (pi_test_on(&vmx->pi_desc))   //如果 PID 的 Outstanding bit 设置上了，表明有 pending 的 posted interrupts
            kvm_vcpu_wake_up(&vmx->vcpu);//此时需要唤醒该 vCPU，也就是 wake_up_process() vCPU thread
    }
    raw_spin_unlock(spinlock);
}
```

### 虚拟机的调入与调出
* KVM module 初始化的时候就设定好了虚拟机被换入和换出的回调函数，虽然调度是以 vCPU 为单位的，但对于每个 vCPU 来说它们的调度回调操作都是一样的，所以用的是全局的操作结构 `struct preempt_ops kvm_preempt_ops`
```c
module_init(vmx_init)
vmx_init()
-> kvm_init()
```
* virt/kvm/kvm_main.c
```cpp
static __read_mostly struct preempt_ops kvm_preempt_ops;
...
int kvm_init(unsigned vcpu_size, unsigned vcpu_align, struct module *module)
{
...
    kvm_preempt_ops.sched_in = kvm_sched_in;
    kvm_preempt_ops.sched_out = kvm_sched_out;
...
}
```
* 当初始化每个 vCPU 时，会将 `kvm_preempt_ops` 与自身的 `vcpu.preempt_notifier` 联系起来
```c
//virt/kvm/kvm_main.c
kvm_vm_ioctl()
case KVM_CREATE_VCPU:
-> kvm_vm_ioctl_create_vcpu()
   -> kvm_vcpu_init()
      -> preempt_notifier_init(&vcpu->preempt_notifier, &kvm_preempt_ops);
```
* include/linux/preempt.h
```c
static inline void preempt_notifier_init(struct preempt_notifier *notifier,
                     struct preempt_ops *ops)
{
    INIT_HLIST_NODE(&notifier->link);
    notifier->ops = ops;
}
```
* 每个 vCPU 投入运行前，将 `vcpu.preempt_notifier` 注册成自身所在 task 的被抢占事件的监听者，同时把 vCPU ID 记录到了 per CPU 的 `kvm_running_vcpu` 变量上
```c
kvm_vcpu_ioctl()
case KVM_RUN:
-> kvm_arch_vcpu_ioctl_run(vcpu)
   -> vcpu_load(vcpu)
      -> __this_cpu_write(kvm_running_vcpu, vcpu);
      -> preempt_notifier_register(&vcpu->preempt_notifier);
      -> kvm_arch_vcpu_load(vcpu, cpu);
   -> vcpu_run(vcpu)
```
* kernel/sched/core.c
```c
/**
 * preempt_notifier_register - tell me when current is being preempted & rescheduled
 * @notifier: notifier struct to register
 */
void preempt_notifier_register(struct preempt_notifier *notifier)
{
    if (!static_branch_unlikely(&preempt_notifier_key))
        WARN(1, "registering preempt_notifier while notifiers disabled\n");

    hlist_add_head(&notifier->link, &current->preempt_notifiers);
}
```
* 当 vCPU 被调度前后都会通知被抢占事件的监听者，逐个调用监听者的换出回调 `.sched_out()` 和换入回调 `.sched_in()`，在 KVM 这个场景就是 `kvm_sched_out()` 和 `kvm_sched_in()` 了
  * 这两个函数最终调用到下面要讲的 `vmx_vcpu_pi_put()` 和 `vmx_vcpu_pi_loadd()`
```c
//kernel/sched/core.c
context_switch()
-> prepare_task_switch(rq, prev, next)
   -> fire_sched_out_preempt_notifiers(prev, next)
      -> __fire_sched_out_preempt_notifiers(curr, next)
         -> notifier->ops->sched_out(notifier, next)
            //virt/kvm/kvm_main.c
         => kvm_sched_out()
               //arch/x86/kvm/x86.c
            -> kvm_arch_vcpu_put(vcpu)
               -> static_call(kvm_x86_vcpu_put)(vcpu)
                  //arch/x86/kvm/vmx/vmx.c
               => vmx_vcpu_put()
                  //arch/x86/kvm/vmx/posted_intr.c
                  -> vmx_vcpu_pi_put()
-> switch_to(prev, next, prev);
-> return finish_task_switch(prev);
   -> fire_sched_in_preempt_notifiers(current)
      -> __fire_sched_in_preempt_notifiers(curr)
         -> notifier->ops->sched_in(notifier, raw_smp_processor_id())
            //virt/kvm/kvm_main.c
         => kvm_sched_in()
            //arch/x86/kvm/x86.c
            -> kvm_arch_sched_in()
               -> static_call(kvm_x86_sched_in)(vcpu, cpu)
                  //arch/x86/kvm/vmx/vmx.c
               => vmx_sched_in()
            //arch/x86/kvm/x86.c
            -> kvm_arch_vcpu_load()
               -> static_call(kvm_x86_vcpu_load)(vcpu, cpu)
                  //arch/x86/kvm/vmx/vmx.c
               => vmx_vcpu_load()
                  -> vmx_vcpu_load_vmcs(vcpu, cpu, NULL)
                  -> vmx_vcpu_pi_load(vcpu, cpu)
```
* kernel/sched/core.c
```cpp
static void __fire_sched_in_preempt_notifiers(struct task_struct *curr)
{
    struct preempt_notifier *notifier;

    hlist_for_each_entry(notifier, &curr->preempt_notifiers, link)
        notifier->ops->sched_in(notifier, raw_smp_processor_id());
}
...
static void
__fire_sched_out_preempt_notifiers(struct task_struct *curr,
                   struct task_struct *next)
{
    struct preempt_notifier *notifier;

    hlist_for_each_entry(notifier, &curr->preempt_notifiers, link)
        notifier->ops->sched_out(notifier, next);
}
```
* 注意 vCPU 换入和换出时对 `vcpu->preempted`、`vcpu->ready` 和 `kvm_running_vcpu` 几个变量的更新
  * 对 vCPU 线程还在运行队列的情况，即被抢占的情况，才将 `vcpu->preempted` 和 `vcpu->ready` 设为 `true`
  * vCPU 停止（halted），比如说 idle，而被调度出去的场景，vCPU 线程不在运行队列上，此时不算被抢占，而是睡眠，代码中通常用 `block` 一词
* virt/kvm/kvm_main.c
```cpp
static void kvm_sched_in(struct preempt_notifier *pn, int cpu)
{
    struct kvm_vcpu *vcpu = preempt_notifier_to_vcpu(pn);

    WRITE_ONCE(vcpu->preempted, false);
    WRITE_ONCE(vcpu->ready, false);

    __this_cpu_write(kvm_running_vcpu, vcpu);
    kvm_arch_sched_in(vcpu, cpu);
    kvm_arch_vcpu_load(vcpu, cpu);
}

static void kvm_sched_out(struct preempt_notifier *pn,
              struct task_struct *next)
{
    struct kvm_vcpu *vcpu = preempt_notifier_to_vcpu(pn);
    //对 vCPU 线程还在运行队列的情况，即被抢占的情况，才将 vcpu->preempted 和 vcpu->ready 设为 true
    if (current->on_rq) {
        WRITE_ONCE(vcpu->preempted, true);
        WRITE_ONCE(vcpu->ready, true);
    }
    kvm_arch_vcpu_put(vcpu);
    __this_cpu_write(kvm_running_vcpu, NULL);
}
```
* 除了在调出场景会调用 `kvm_arch_vcpu_put()` 外，`vcpu_put()` 也会调用，并且此时会 **解除** 对 vCPU 线程被抢占事件的监听
  * virt/kvm/kvm_main.c
```cpp
void vcpu_put(struct kvm_vcpu *vcpu)
{
    preempt_disable();
    kvm_arch_vcpu_put(vcpu);
    preempt_notifier_unregister(&vcpu->preempt_notifier);
    __this_cpu_write(kvm_running_vcpu, NULL);
    preempt_enable();
}
```

### 启用 Posted Interrupt 唤醒向量的处理
* `vmx_can_use_ipiv()` 判断 VM 是否可以启用 IPI 虚拟化，满足以下条件返回 `true`
  * 如果 Local APIC 的模拟是放在 kernel 侧
  * 且 IPI 虚拟化的全局开关 `enable_ipiv` 打开
  * arch/x86/kvm/vmx/vmx.h
```cpp
static inline bool vmx_can_use_ipiv(struct kvm_vcpu *vcpu)
{
    return  lapic_in_kernel(vcpu) && enable_ipiv;
}
```
* `vmx_can_use_vtd_pi()` 判断 VM 是否可以启用 VT-d Posted Interrupt，满足以下条件返回 `true`
  * 如果 IRQ chip 放在 kernel 侧来模拟
  * APIC 虚拟化的全局开关 `enable_apicv` 打开
  * 该 VM 有被指派的设备
  * 且硬件具备中断重映射的 posted interrupt 能力
  * arch/x86/kvm/vmx/posted_intr.c
```cpp
static bool vmx_can_use_vtd_pi(struct kvm *kvm)
{
    return irqchip_in_kernel(kvm) && enable_apicv &&
        kvm_arch_has_assigned_device(kvm) &&
        irq_remapping_cap(IRQ_POSTING_CAP);
}
```
* `vmx_needs_pi_wakeup()` 判断 VM 是否需要 Posted Interrupt 唤醒，以上两个判断任何一个成立返回 `true`
  * 因为 *IPI 虚拟化* 和 *VT-d Posted Interrupt* 都是硬件自动完成 `PID` 的更新，硬件不知道 vCPU 是运行还是睡眠
  * 如果是睡眠，需要软件在调度出去前将 `NV` 从 `ANV` 修改为 `WNV` 并将目标 CPU 设置为 `NDST`，硬件只需要还按原来的逻辑照做就好了
  * 当然，vCPU 调度回来前还得恢复以上两个域
* 在 guest 模式之外调用时，默认 posted interrupt 向量（`ANV`）不执行任何操作。如果是使用 IPI 虚拟化或 VT-d PI 时的情况，返回阻塞的 vCPU 是否可以成为 posted interrupts 的目标，以便通知向量 `ANV` 切换到回调 `pi_wakeup_handler()` 函数的向量 `WNV`。
```cpp
static bool vmx_needs_pi_wakeup(struct kvm_vcpu *vcpu)
{
    /*
     * The default posted interrupt vector does nothing when
     * invoked outside guest mode.   Return whether a blocked vCPU
     * can be the target of posted interrupts, as is the case when
     * using either IPI virtualization or VT-d PI, so that the
     * notification vector is switched to the one that calls
     * back to the pi_wakeup_handler() function.
     */
    return vmx_can_use_ipiv(vcpu) || vmx_can_use_vtd_pi(vcpu->kvm);
}
```
* `vmx_interrupt_blocked()` 返回 vCPU 中断是否正处于被屏蔽状态。对于 vCPU 已退出到 host 模式的几个判断条件之一为 `true` 则认为 vCPU 屏蔽外部中断：
  * vCPU 的 `RFLAGS` 寄存器的 `X86_EFLAGS_IF` 已设置，表示开启外部中断。有个取反 `!` 操作，表示 vCPU 屏蔽外部中断
  * 或者，guest vCPU 正执行 `STI` 或 `MOV $SS` 指令
  * arch/x86/kvm/vmx/vmx.c
```cpp
bool vmx_interrupt_blocked(struct kvm_vcpu *vcpu)
{   //如果 vCPU 正处于 guest 模式且嵌套的 VMCS 的外部中断退出域已设置，则必然不屏蔽外部中断
    if (is_guest_mode(vcpu) && nested_exit_on_intr(vcpu))
        return false;
    //否则 vCPU 已退出到 host 模式，或嵌套 VMCS 未开启外部中断退出域
    return !(vmx_get_rflags(vcpu) & X86_EFLAGS_IF) ||
           (vmcs_read32(GUEST_INTERRUPTIBILITY_INFO) &
        (GUEST_INTR_STATE_STI | GUEST_INTR_STATE_MOV_SS));
}
```
* `kvm_vcpu_is_blocking()` 是 vCPU 停止执行（halted）的意思，不是进程阻塞、被调度出去的意思。否则就没必要判断 vCPU 是否已屏蔽外部中断了，进程阻塞必然要开中断，不然就是 bug
* 如果 vCPU 不是停止运行（halted），或 vCPU 已屏蔽外部中断，仍然被 VM-exit 了，那就要看是否 vCPU 被抢占了？如果是，这种情况是要设置 `PID` 的 `SN` 以抑制硬件自动发送通知向量的
```cpp
void vmx_vcpu_pi_put(struct kvm_vcpu *vcpu)
{
    struct pi_desc *pi_desc = vcpu_to_pi_desc(vcpu);
    //如果不需要 PI 唤醒通知，就此返回；否则继续前进
    if (!vmx_needs_pi_wakeup(vcpu))
        return;
    //如果 vCPU 睡眠（.wait 域不为空）且 vCPU 未屏蔽外部中断，则启用 Posted Interrupt 唤醒向量的处理
    if (kvm_vcpu_is_blocking(vcpu) && !vmx_interrupt_blocked(vcpu))
        pi_enable_wakeup_handler(vcpu);
    //当 vCPU 被抢占时设置 SN。请注意，vCPU 既可以被视为阻塞，也可以被视为抢占，例如，如果它在设置其等待状态和手动调度之间被抢占。
    /*
     * Set SN when the vCPU is preempted.  Note, the vCPU can both be seen
     * as blocking and preempted, e.g. if it's preempted between setting
     * its wait state and manually scheduling out.
     */
    if (vcpu->preempted)
        pi_set_sn(pi_desc);
}
```
* 前面已经说过了，在 vCPU 睡眠前，要启用 Posted Interrupt 唤醒向量 `WNV` 的处理
  * arch/x86/kvm/vmx/posted_intr.c
```cpp
/*
 * Put the vCPU on this pCPU's list of vCPUs that needs to be awakened and set
 * WAKEUP as the notification vector in the PI descriptor.
 */
static void pi_enable_wakeup_handler(struct kvm_vcpu *vcpu)
{
    struct pi_desc *pi_desc = vcpu_to_pi_desc(vcpu);
    struct vcpu_vmx *vmx = to_vmx(vcpu);
    struct pi_desc old, new;
    unsigned long flags;
    //保持中断状态后关中断
    local_irq_save(flags);
    //将 vCPU 加到 per CPU 的 PI 唤醒通知队列上，节点为 .pi_wakeup_list，回头可以通过它找到它的容器 vcpu_vmx
    raw_spin_lock(&per_cpu(wakeup_vcpus_on_cpu_lock, vcpu->cpu));
    list_add_tail(&vmx->pi_wakeup_list,
              &per_cpu(wakeup_vcpus_on_cpu, vcpu->cpu));
    raw_spin_unlock(&per_cpu(wakeup_vcpus_on_cpu_lock, vcpu->cpu));
    // PID 的 SN 如果被设置是不合理的，因为这是 vCPU 睡眠的场景，而不是被抢占
    WARN(pi_desc->sn, "PI descriptor SN field set before blocking");

    old.control = READ_ONCE(pi_desc->control);
    do {
        /* set 'NV' to 'wakeup vector' */
        new.control = old.control; //PID 的其他控制域保持不变
        new.nv = POSTED_INTR_WAKEUP_VECTOR; //PID 的 NV 域设为 “WNV”
    } while (pi_try_set_control(pi_desc, &old.control, new.control));
    //如果在更新通知向量为 WNV 之前可能已发出中断（PID 的 ON 被设置），
    //则向本 CPU 发送唤醒 self-IPI WNV，在这种情况下，IRQ 将随非唤醒向量 WNV
    //到达。这里需要 IPI 的方式，是因为不允许从 ->sched_out() 调用 try_to_wake_up()
    //直到可以安全地对正在被调度出去的任务调用 try_to_wake_up() 时才会启用 IRQ）
    /*
     * Send a wakeup IPI to this CPU if an interrupt may have been posted
     * before the notification vector was updated, in which case the IRQ
     * will arrive on the non-wakeup vector.  An IPI is needed as calling
     * try_to_wake_up() from ->sched_out() isn't allowed (IRQs are not
     * enabled until it is safe to call try_to_wake_up() on the task being
     * scheduled out).
     */
    if (pi_test_on(&new))
        apic->send_IPI_self(POSTED_INTR_WAKEUP_VECTOR);
    //还原原先的中断状态
    local_irq_restore(flags);
}
```

### 恢复 Posted Interrupt 通知向量
* 从上面的分析可以知道 `vmx_vcpu_pi_load()` 是在 vCPU 线程被换入的路径上的，在此之前它是由 Posted Interrupt 唤醒向量的处理函数唤醒的
  * arch/x86/kvm/vmx/posted_intr.c
```cpp
void vmx_vcpu_pi_load(struct kvm_vcpu *vcpu, int cpu)
{
    struct pi_desc *pi_desc = vcpu_to_pi_desc(vcpu);
    struct vcpu_vmx *vmx = to_vmx(vcpu);
    struct pi_desc old, new;
    unsigned long flags;
    unsigned int dest;
    //如果未启用 APIC 虚拟化，local APIC 的模拟也不在 kernel 侧，意味则不支持 PI，直接返回
    //为了简化热插拔和 APICv 的动态切换，即使没有分配的设备或者由于动态禁止位例如，Hyper-V 的 SyncIC，而未激活 APICv，也要保持 PI.NDST 和 PI.SN 的更新。
    /*
     * To simplify hot-plug and dynamic toggling of APICv, keep PI.NDST and
     * PI.SN up-to-date even if there is no assigned device or if APICv is
     * deactivated due to a dynamic inhibit bit, e.g. for Hyper-V's SyncIC.
     */
    if (!enable_apicv || !lapic_in_kernel(vcpu))
        return;
    //如果 vCPU 不在唤醒列表中并且未迁移，则可以跳过完整更新，因为向量和目标都不需要更改。
    /*
     * If the vCPU wasn't on the wakeup list and wasn't migrated, then the
     * full update can be skipped as neither the vector nor the destination
     * needs to be changed.
     */
    if (pi_desc->nv != POSTED_INTR_WAKEUP_VECTOR && vcpu->cpu == cpu) {
        /*
         * Clear SN if it was set due to being preempted.  Again, do
         * this even if there is no assigned device for simplicity.
         */
        //如果由于被抢占而设置了 SN，则清除 SN。同样，为简单起见，即使没有分配的设备也执行此操作。
        if (pi_test_and_clear_sn(pi_desc))
            goto after_clear_sn;
        return;
    }
    local_irq_save(flags);
    //如果 vCPU 正在等待唤醒，则将该 vCPU 从之前的 pCPU 的唤醒列表中删除，如果任务曾经被迁移过，则此前的 pCPU 将与当前 pCPU 不同。
    /*
     * If the vCPU was waiting for wakeup, remove the vCPU from the wakeup
     * list of the _previous_ pCPU, which will not be the same as the
     * current pCPU if the task was migrated.
     */
    if (pi_desc->nv == POSTED_INTR_WAKEUP_VECTOR) {
        raw_spin_lock(&per_cpu(wakeup_vcpus_on_cpu_lock, vcpu->cpu));
        list_del(&vmx->pi_wakeup_list);
        raw_spin_unlock(&per_cpu(wakeup_vcpus_on_cpu_lock, vcpu->cpu));
    }

    dest = cpu_physical_id(cpu);
    if (!x2apic_mode)
        dest = (dest << 8) & 0xFF00;

    old.control = READ_ONCE(pi_desc->control);
    do {
        new.control = old.control;
        //清除 SN（如上）并刷新目标 APIC ID 以处理任务迁移（@cpu != vcpu->cpu）。
        /*
         * Clear SN (as above) and refresh the destination APIC ID to
         * handle task migration (@cpu != vcpu->cpu).
         */
        new.ndst = dest;
        new.sn = 0;
        //恢复通知向量 ANV；在阻塞（vCPU halted）情况下，PID 在“put”路径上被修改以使用唤醒向量 WNV
        /*
         * Restore the notification vector; in the blocking case, the
         * descriptor was modified on "put" to use the wakeup vector.
         */
        new.nv = POSTED_INTR_VECTOR;
    } while (pi_try_set_control(pi_desc, &old.control, new.control));

    local_irq_restore(flags);

after_clear_sn:
    //读取 PIR 的 PIR 位图前先清除 SN。VT-d 固件以原子方式写入位图并读取 SN（规范中的 5.2.3），因此它实际上没有与之配对的内存屏障，但我们不能这样做，我们需要一个。
    /*
     * Clear SN before reading the bitmap.  The VT-d firmware
     * writes the bitmap and reads SN atomically (5.2.3 in the
     * spec), so it doesn't really have a memory barrier that
     * pairs with this, but we cannot do that and we need one.
     */
    smp_mb__after_atomic();
    //如果 PID 中的 PIR 不为空，则设置 PID 的 ON 位
    if (!pi_is_pir_empty(pi_desc))
        pi_set_on(pi_desc);
}
```
##### 为什么 vCPU 因为停止（halted、睡眠、阻塞）只更新了 `PID.NV` 为 `WNV` 而不更新 `PID.NDST` 了？难道不怕 vCPU 被迁移到其他 pCPU 上无法在旧 pCPU 上被唤醒吗？
* 其实原因就在 `vmx_vcpu_pi_load()` 这个函数，对于被 vCPU 被迁移这种情况它要做以下几个事情：
  1. vCPU 线程被从就 pCPU 的唤醒队列 `wakeup_vcpus_on_cpu` 上拿下来了
  2. `PID.NV` 被恢复为 `ANV`
  3. `PID.NDST` 被更新为现在 vCPU 所在的 pCPU
* 也就是说，即便它在其他 CPU 上被 load 就没打算再睡了，在哪个 pCPU 上被唤醒就在哪个 pCPU 上接着处理中断（如果有的话），而不是回到原来的 pCPU 上去处理。
* 是以下这个 commit 改变了这一切：
```c
commit d76fb40637fc0e84b27bf431cd72cf8fe3f813ef
Author: Sean Christopherson <seanjc@google.com>
Date:   Wed Dec 8 01:52:14 2021 +0000

    KVM: VMX: Handle PI descriptor updates during vcpu_put/load
```

## vCPU Blocking

### 如何判断 vCPU Blocking
* 之前在 `vmx_vcpu_pi_put()` 判断要不要启用 Posted interrupt 唤醒向量功能前，用了一个判断条件 `kvm_vcpu_is_blocking()`，这里的 `blocking` 是 vCPU 停止执行的意思，不是常规意义上的进程阻塞、被调度出去的意思
* 判断条件其实很简单，就是 `vcpu->wait->task` 变量是不是空，不是空证明 vCPU 被阻塞
* include/linux/rcuwait.h
```cpp
/*
 * Note: this provides no serialization and, just as with waitqueues,
 * requires care to estimate as to whether or not the wait is active.
 */
static inline int rcuwait_active(struct rcuwait *w)
{
    return !!rcu_access_pointer(w->task);
}
...
/*
 * The caller is responsible for locking around rcuwait_wait_event(),
 * and [prepare_to/finish]_rcuwait() such that writes to @task are
 * properly serialized.
 */

static inline void prepare_to_rcuwait(struct rcuwait *w)
{
    rcu_assign_pointer(w->task, current);
}
```
* include/linux/kvm_host.h
```cpp
static inline struct rcuwait *kvm_arch_vcpu_get_wait(struct kvm_vcpu *vcpu)
{
#ifdef __KVM_HAVE_ARCH_WQP
    return vcpu->arch.waitp;
#else
    return &vcpu->wait;
#endif
}
...
static inline bool kvm_vcpu_is_blocking(struct kvm_vcpu *vcpu)
{
    return rcuwait_active(kvm_arch_vcpu_get_wait(vcpu));
}
```

### vCPU Blocking 主要做些什么
* `kvm_vcpu_block()` 阻塞 vCPU，直到 vCPU 可运行、事件到达或信号待处理。这主要在停止 vCPU 时使用，但也可以直接用于其他 vCPU 不可运行状态，例如 x86 的等待 SIPI
  * `vcpu->wait->task` 的值也是在这个时候被赋上的，见 `prepare_to_rcuwait(wait)`
  * virt/kvm/kvm_main.c
```cpp
static int kvm_vcpu_check_block(struct kvm_vcpu *vcpu)
{
    int ret = -EINTR;
    int idx = srcu_read_lock(&vcpu->kvm->srcu);
    //arch 相关的 vCPU 可运行条件，比如 x86 arch 有 mp_state、INIT、SIPI、NMI、异常、timer
    if (kvm_arch_vcpu_runnable(vcpu))
        goto out;
    if (kvm_cpu_has_pending_timer(vcpu)) //有 pending 的 timer event
        goto out;
    if (signal_pending(current)) //vCPU 有 pending 的信号
        goto out;
    if (kvm_check_request(KVM_REQ_UNBLOCK, vcpu)) //vCPU 有 KVM_REQ_UNBLOCK 请求
        goto out;

    ret = 0;
out:
    srcu_read_unlock(&vcpu->kvm->srcu, idx);
    return ret;
}

/*
 * Block the vCPU until the vCPU is runnable, an event arrives, or a signal is
 * pending.  This is mostly used when halting a vCPU, but may also be used
 * directly for other vCPU non-runnable states, e.g. x86's Wait-For-SIPI.
 */
bool kvm_vcpu_block(struct kvm_vcpu *vcpu)
{
    struct rcuwait *wait = kvm_arch_vcpu_get_wait(vcpu);
    bool waited = false;

    vcpu->stat.generic.blocking = 1;

    preempt_disable();
    kvm_arch_vcpu_blocking(vcpu); //对于 VMX 没有实现该回调
    prepare_to_rcuwait(wait); //给 vcpu->wait->task 赋值 current，vCPU 阻塞的标志
    preempt_enable();

    for (;;) {
        set_current_state(TASK_INTERRUPTIBLE); //vCPU 线程置为可中断睡眠状态
        //检查阻塞条件是否满足
        if (kvm_vcpu_check_block(vcpu) < 0)
            break; //返回值 < 0，阻塞被打断，vCPU 随后需要被投入运行
        //返回值 >=0，不满足，继续阻塞（调度）
        waited = true;
        schedule();
    }

    preempt_disable();
    finish_rcuwait(wait); //vCPU 不再阻塞了，vcpu->wait->task 置为 NULL，且 vCPU 线程状态变为运行状态
    kvm_arch_vcpu_unblocking(vcpu); //对于 VMX 没有实现该回调
    preempt_enable();

    vcpu->stat.generic.blocking = 0;

    return waited;
}
```
* vCPU 不再阻塞了，`vcpu->wait->task` 作为阻塞的标志需要置为 `NULL`，并且 vCPU 线程状态变为运行状态 `TASK_RUNNING`
  * kernel/rcu/update.c
```cpp
void finish_rcuwait(struct rcuwait *w)
{
    rcu_assign_pointer(w->task, NULL);
    __set_current_state(TASK_RUNNING);
}
```

### 如何进入 vCPU Blocking
* vCPU 进入这种状态的三条路径如下：
```c
//virt/kvm/kvm_main.c
kvm_vcpu_ioctl()
case KVM_RUN:
   //arch/x86/kvm/x86.c
-> kvm_arch_vcpu_ioctl_run(vcpu)
      //virt/kvm/kvm_main.c
   -> vcpu_load(vcpu)
         //arch/x86/kvm/x86.c
      -> kvm_arch_vcpu_load()
         -> static_call(kvm_x86_vcpu_load)(vcpu, cpu)
            //arch/x86/kvm/vmx/vmx.c
         => vmx_vcpu_load()
            -> vmx_vcpu_load_vmcs(vcpu, cpu, NULL)
            -> vmx_vcpu_pi_load(vcpu, cpu)
      if (unlikely(vcpu->arch.mp_state == KVM_MP_STATE_UNINITIALIZED))
         //virt/kvm/kvm_main.c
   ->    kvm_vcpu_block(vcpu) //路径 1
      //arch/x86/kvm/x86.c
   -> vcpu_run(vcpu)
         for (;;) {
            if (kvm_vcpu_running(vcpu)) {
                r = vcpu_enter_guest(vcpu);
                       for (;;) {
                       -> exit_fastpath = static_call(kvm_x86_vcpu_run)(vcpu);
                          //arch/x86/kvm/vmx/vmx.c
                       => vmx_vcpu_run(vcpu)
                          -> vmx_vcpu_enter_exit(vcpu, __vmx_vcpu_run_flags(vmx))
                                //arch/x86/kvm/vmx/vmenter.S
                             -> __vmx_vcpu_run(vmx, (unsigned long *)&vcpu->arch.regs, flags);s
                       }
            } else {
                r = vcpu_block(vcpu);
                       if (!kvm_arch_vcpu_runnable(vcpu))
                           if (vcpu->arch.mp_state == KVM_MP_STATE_HALTED)
                              //virt/kvm/kvm_main.c
                           -> kvm_vcpu_halt(vcpu);
                              -> waited = kvm_vcpu_block(vcpu); //路径 3
                           else
                              //virt/kvm/kvm_main.c
                           -> kvm_vcpu_block(vcpu); //路径 2

            }
         }
```

* 路径 1：把 vCPU run 起来的是否发现 vCPU 状态还未初始化 `if (unlikely(vcpu->arch.mp_state == KVM_MP_STATE_UNINITIALIZED))`
* 路径 2：`vcpu_run()` 时发现 vCPU 状态还不是可运行状态，如果状态也不是 `KVM_MP_STATE_HALTED`，直接进入 `kvm_vcpu_block()`
* 路径 3：应该就是 halt polling 状态吧，看看注释的解释
  * 模拟 vCPU halt 条件，例如 x86 上的 `HLT`、arm 上的 WFI（wait-for-init？）等...
  * 如果启用了 halt polling，会在阻塞之前忙等待一小段时间，假如在 vCPU halted 后不久出现唤醒事件时，就可以避免昂贵的阻塞 + 解除阻塞序列。
```cpp
/*
 * Emulate a vCPU halt condition, e.g. HLT on x86, WFI on arm, etc...  If halt
 * polling is enabled, busy wait for a short time before blocking to avoid the
 * expensive block+unblock sequence if a wake event arrives soon after the vCPU
 * is halted.
 */
void kvm_vcpu_halt(struct kvm_vcpu *vcpu)
{
...
    waited = kvm_vcpu_block(vcpu);
...
}
```

## 设置 Posted Interrupt IRTE

* 对于 VFIO 可以走到 `vmx_pi_update_irte()`
```c
//drivers/vfio/pci/vfio_pci_core.c
vfio_pci_core_ioctl()
case VFIO_DEVICE_SET_IRQS:
-> vfio_pci_ioctl_set_irqs()
      //drivers/vfio/pci/vfio_pci_intrs.c
   -> vfio_pci_set_irqs_ioctl()
      case VFIO_PCI_MSI_IRQ_INDEX:
      case VFIO_PCI_MSIX_IRQ_INDEX:
         switch (flags & VFIO_IRQ_SET_ACTION_TYPE_MASK)
            case VFIO_IRQ_SET_ACTION_TRIGGER:
               func = vfio_pci_set_msi_trigger;
      -> func(vdev, index, start, count, flags, data)
      => vfio_pci_set_msi_trigger()
         -> vfio_msi_set_block()
            -> vfio_msi_set_vector_signal()
                  //virt/lib/irqbypass.c
               -> irq_bypass_register_producer(&vdev->ctx[vector].producer)
                  -> __connect(producer, consumer)
                        //virt/kvm/eventfd.c::kvm_irqfd_assign()
                        //irqfd->consumer.add_producer = kvm_arch_irq_bypass_add_producer;
                     -> cons->add_producer(cons, prod); 
                        //arch/x86/kvm/x86.c
                     => kvm_arch_irq_bypass_add_producer()
                        -> static_call(kvm_x86_pi_update_irte)(irqfd->kvm, prod->irq, irqfd->gsi, 1)
                           //arch/x86/kvm/vmx/posted_intr.c
                        => vmx_pi_update_irte()
```

* 对于 `ioctl(KVM_SET_GSI_ROUTING)` 或 `ioctl(KVM_CREATE_IRQCHIP)`，也有可能走到 `vmx_pi_update_irte()`
```c
//virt/kvm/kvm_main.c
kvm_vm_ioctl()
default:
   //arch/x86/kvm/x86.c
-> kvm_arch_vm_ioctl()
   case KVM_SET_GSI_ROUTING:
      //virt/kvm/irqchip.c
   -> kvm_set_irq_routing(kvm, entries, routing.nr, routing.flags);
         //virt/kvm/eventfd.c
      -> kvm_irq_routing_update()
            if (irqfd->producer && kvm_arch_irqfd_route_changed(&old, &irqfd->irq_entry))
            //arch/x86/kvm/x86.c
            -> kvm_arch_update_irqfd_routing(irqfd->kvm, irqfd->producer->irq, irqfd->gsi, 1);
               -> static_call(kvm_x86_pi_update_irte)(kvm, host_irq, guest_irq, set);
                  //arch/x86/kvm/vmx/posted_intr.c
               => vmx_pi_update_irte()
   case KVM_CREATE_IRQCHIP:
   -> kvm_setup_default_irq_routing()
      -> kvm_set_irq_routing(kvm, default_routing, ARRAY_SIZE(default_routing), 0);
```

* 最后由 `vmx_pi_update_irte()` 更新中断重映射的 IRTE 为支持 Posted interrupt 的条目
```c
//arch/x86/kvm/vmx/posted_intr.c
vmx_pi_update_irte()
-> kvm_set_msi_irq(kvm, e, &irq)
   vcpu_info.pi_desc_addr = __pa(vcpu_to_pi_desc(vcpu));
   vcpu_info.vector = irq.vector;
-> trace_kvm_pi_irte_update()
   //kernel/irq/manage.c
-> irq_set_vcpu_affinity(host_irq, &vcpu_info)
   -> struct irq_desc *desc = irq_get_desc_lock(irq, &flags, 0);
   -> struct irq_data *data = irq_desc_get_irq_data(desc);
   -> struct irq_chip *chip = irq_data_get_irq_chip(data);
   -> chip->irq_set_vcpu_affinity(data, vcpu_info);
      //drivers/iommu/intel/irq_remapping.c
      -> intel_ir_set_vcpu_affinity()
         -> modify_irte(&ir_data->irq_2_iommu, &irte_pi)
```

## References
- [Intel VT-d（4）- Interrupt Posting - 知乎](https://zhuanlan.zhihu.com/p/51018597)
- [[PATCH v10 0_7] KVM VMX Add Posted Interrupt supporting - Yang Zhang](https://lore.kernel.org/all/1365679516-13125-1-git-send-email-yang.z.zhang@intel.com/)
- [[PATCH v9 00_18] Add VT-d Posted-Interrupts support - including prerequisite series - Feng Wu](https://lore.kernel.org/lkml/1442586596-5920-1-git-send-email-feng.wu@intel.com/)