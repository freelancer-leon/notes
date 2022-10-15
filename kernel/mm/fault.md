# 缺页处理

## 缺页处理函数的声明
* arch/x86/include/asm/idtentry.h
```cpp
#define DECLARE_IDTENTRY_ERRORCODE(vector, func)            \
    asmlinkage void asm_##func(void);               \
    asmlinkage void xen_asm_##func(void);               \
    __visible void func(struct pt_regs *regs, unsigned long error_code)
...
/**
 * DECLARE_IDTENTRY_RAW_ERRORCODE - Declare functions for raw IDT entry points
 *                  Error code pushed by hardware
 * @vector: Vector number (ignored for C)
 * @func:   Function name of the entry point
 *
 * Maps to DECLARE_IDTENTRY_ERRORCODE()
 */
#define DECLARE_IDTENTRY_RAW_ERRORCODE(vector, func)            \
    DECLARE_IDTENTRY_ERRORCODE(vector, func)
...
DECLARE_IDTENTRY_RAW_ERRORCODE(X86_TRAP_PF, exc_page_fault);
```
## 缺页处理函数的注册

## 缺页处理函数的实现
* arch/x86/mm/fault.c
```cpp
static __always_inline void
handle_page_fault(struct pt_regs *regs, unsigned long error_code,
                  unsigned long address)
{
    trace_page_fault_entries(regs, error_code, address);

    if (unlikely(kmmio_fault(regs, address)))
        return;

    /* Was the fault on kernel-controlled part of the address space? */
    if (unlikely(fault_in_kernel_space(address))) {
        do_kern_addr_fault(regs, error_code, address);
    } else {
        do_user_addr_fault(regs, error_code, address);
        /*
         * User address page fault handling might have reenabled
         * interrupts. Fixing up all potential exit points of
         * do_user_addr_fault() and its leaf functions is just not
         * doable w/o creating an unholy mess or turning the code
         * upside down.
         */
        local_irq_disable();
    }
}

DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault)
{
    unsigned long address = read_cr2();
    irqentry_state_t state;

    prefetchw(&current->mm->mmap_lock);

    /*
     * KVM uses #PF vector to deliver 'page not present' events to guests
     * (asynchronous page fault mechanism). The event happens when a
     * userspace task is trying to access some valid (from guest's point of
     * view) memory which is not currently mapped by the host (e.g. the
     * memory is swapped out). Note, the corresponding "page ready" event
     * which is injected when the memory becomes available, is delivered via
     * an interrupt mechanism and not a #PF exception
     * (see arch/x86/kernel/kvm.c: sysvec_kvm_asyncpf_interrupt()).
     *
     * We are relying on the interrupted context being sane (valid RSP,
     * relevant locks not held, etc.), which is fine as long as the
     * interrupted context had IF=1.  We are also relying on the KVM
     * async pf type field and CR2 being read consistently instead of
     * getting values from real and async page faults mixed up.
     *
     * Fingers crossed.
     *
     * The async #PF handling code takes care of idtentry handling
     * itself.
     */
    if (kvm_handle_async_pf(regs, (u32)address))
        return;

    /*
     * Entry handling for valid #PF from kernel mode is slightly
     * different: RCU is already watching and ct_irq_enter() must not
     * be invoked because a kernel fault on a user space address might
     * sleep.
     *
     * In case the fault hit a RCU idle region the conditional entry
     * code reenabled RCU to avoid subsequent wreckage which helps
     * debuggability.
     */
    state = irqentry_enter(regs);

    instrumentation_begin();
    handle_page_fault(regs, error_code, address);
    instrumentation_end();

    irqentry_exit(regs, state);
}
```