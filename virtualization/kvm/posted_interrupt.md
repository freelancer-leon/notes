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