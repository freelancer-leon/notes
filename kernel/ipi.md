# IPI

## x86 APIC 中断

### x86 APIC 中断向量
* 比如说有以下一些 x86 APIC 中断向量
* arch/x86/include/asm/irq_vectors.h
```c
#define ERROR_APIC_VECTOR       0xfe
#define RESCHEDULE_VECTOR       0xfd
#define CALL_FUNCTION_VECTOR        0xfc
#define CALL_FUNCTION_SINGLE_VECTOR 0xfb
#define THERMAL_APIC_VECTOR     0xfa
#define THRESHOLD_APIC_VECTOR       0xf9
#define REBOOT_VECTOR           0xf8
```
### 声明 x86 APIC 中断向量
* 声明这些向量
  * arch/x86/include/asm/idtentry.h
```cpp
/* System vector entry points */
#ifdef CONFIG_X86_LOCAL_APIC
DECLARE_IDTENTRY_SYSVEC(ERROR_APIC_VECTOR,      sysvec_error_interrupt);
DECLARE_IDTENTRY_SYSVEC(SPURIOUS_APIC_VECTOR,       sysvec_spurious_apic_interrupt);
DECLARE_IDTENTRY_SYSVEC(LOCAL_TIMER_VECTOR,     sysvec_apic_timer_interrupt);
DECLARE_IDTENTRY_SYSVEC(X86_PLATFORM_IPI_VECTOR,    sysvec_x86_platform_ipi);
#endif

#ifdef CONFIG_SMP
DECLARE_IDTENTRY(RESCHEDULE_VECTOR,         sysvec_reschedule_ipi);
DECLARE_IDTENTRY_SYSVEC(IRQ_MOVE_CLEANUP_VECTOR,    sysvec_irq_move_cleanup);
DECLARE_IDTENTRY_SYSVEC(REBOOT_VECTOR,          sysvec_reboot);
DECLARE_IDTENTRY_SYSVEC(CALL_FUNCTION_SINGLE_VECTOR,    sysvec_call_function_single);
DECLARE_IDTENTRY_SYSVEC(CALL_FUNCTION_VECTOR,       sysvec_call_function);
#endif
#ifdef CONFIG_X86_LOCAL_APIC
# ifdef CONFIG_X86_MCE_THRESHOLD
DECLARE_IDTENTRY_SYSVEC(THRESHOLD_APIC_VECTOR,      sysvec_threshold);
# endif

# ifdef CONFIG_X86_MCE_AMD
DECLARE_IDTENTRY_SYSVEC(DEFERRED_ERROR_VECTOR,      sysvec_deferred_error);
# endif

# ifdef CONFIG_X86_THERMAL_VECTOR
DECLARE_IDTENTRY_SYSVEC(THERMAL_APIC_VECTOR,        sysvec_thermal);
# endif

# ifdef CONFIG_IRQ_WORK
DECLARE_IDTENTRY_SYSVEC(IRQ_WORK_VECTOR,        sysvec_irq_work);
# endif
#endif
```
* 以上声明在 C 代码和汇编代码中分别展开
* 在 C 代码中的展开
```cpp
/**
 * DECLARE_IDTENTRY - Declare functions for simple IDT entry points
 *            No error code pushed by hardware
 * @vector: Vector number (ignored for C)
 * @func:   Function name of the entry point
 *
 * Declares three functions:
 * - The ASM entry point: asm_##func
 * - The XEN PV trap entry point: xen_##func (maybe unused)
 * - The C handler called from the ASM entry point
 *
 * Note: This is the C variant of DECLARE_IDTENTRY(). As the name says it
 * declares the entry points for usage in C code. There is an ASM variant
 * as well which is used to emit the entry stubs in entry_32/64.S.
 */
#define DECLARE_IDTENTRY(vector, func)                  \
    asmlinkage void asm_##func(void);               \
    asmlinkage void xen_asm_##func(void);               \
    __visible void func(struct pt_regs *regs)

/**
 * DECLARE_IDTENTRY_SYSVEC - Declare functions for system vector entry points
 * @vector: Vector number (ignored for C)
 * @func:   Function name of the entry point
 *
 * Declares three functions:
 * - The ASM entry point: asm_##func
 * - The XEN PV trap entry point: xen_##func (maybe unused)
 * - The C handler called from the ASM entry point
 *
 * Maps to DECLARE_IDTENTRY().
 */
#define DECLARE_IDTENTRY_SYSVEC(vector, func)               \
    DECLARE_IDTENTRY(vector, func)
```
* 在汇编中展开
* arch/x86/include/asm/idtentry.h
```cpp
/* System vector entries */
#define DECLARE_IDTENTRY_SYSVEC(vector, func)               \
    idtentry_sysvec vector func
```
* arch/x86/entry/entry_64.S
```cpp
/**
 * idtentry_body - Macro to emit code calling the C function
 * @cfunc:      C function to be called
 * @has_error_code: Hardware pushed error code on stack
 */
.macro idtentry_body cfunc has_error_code:req

    /*
     * Call error_entry() and switch to the task stack if from userspace.
     *
     * When in XENPV, it is already in the task stack, and it can't fault
     * for native_iret() nor native_load_gs_index() since XENPV uses its
     * own pvops for IRET and load_gs_index().  And it doesn't need to
     * switch the CR3.  So it can skip invoking error_entry().
     */
    ALTERNATIVE "call error_entry; movq %rax, %rsp", \
            "call xen_error_entry", X86_FEATURE_XENPV

    ENCODE_FRAME_POINTER
    UNWIND_HINT_REGS

    movq    %rsp, %rdi          /* pt_regs pointer into 1st argument*/

    .if \has_error_code == 1
        movq    ORIG_RAX(%rsp), %rsi    /* get error code into 2nd argument*/
        movq    $-1, ORIG_RAX(%rsp) /* no syscall to restart */
    .endif

    call    \cfunc

    /* For some configurations \cfunc ends up being a noreturn. */
    REACHABLE

    jmp error_return
.endm

/**
 * idtentry - Macro to generate entry stubs for simple IDT entries
 * @vector:     Vector number
 * @asmsym:     ASM symbol for the entry point
 * @cfunc:      C function to be called
 * @has_error_code: Hardware pushed error code on stack
 *
 * The macro emits code to set up the kernel context for straight forward
 * and simple IDT entries. No IST stack, no paranoid entry checks.
 */
.macro idtentry vector asmsym cfunc has_error_code:req
SYM_CODE_START(\asmsym)

    .if \vector == X86_TRAP_BP
        /* #BP advances %rip to the next instruction */
        UNWIND_HINT_IRET_REGS offset=\has_error_code*8 signal=0
    .else
        UNWIND_HINT_IRET_REGS offset=\has_error_code*8
    .endif

    ENDBR
    ASM_CLAC
    cld

    .if \has_error_code == 0
        pushq   $-1         /* ORIG_RAX: no syscall to restart */
    .endif

    .if \vector == X86_TRAP_BP
        /*
         * If coming from kernel space, create a 6-word gap to allow the
         * int3 handler to emulate a call instruction.
         */
        testb   $3, CS-ORIG_RAX(%rsp)
        jnz .Lfrom_usermode_no_gap_\@
        .rept   6
        pushq   5*8(%rsp)
        .endr
        UNWIND_HINT_IRET_REGS offset=8
.Lfrom_usermode_no_gap_\@:
    .endif

    idtentry_body \cfunc \has_error_code

_ASM_NOKPROBE(\asmsym)
SYM_CODE_END(\asmsym)
.endm

/*
 * System vectors which invoke their handlers directly and are not
 * going through the regular common device interrupt handling code.
 */
.macro idtentry_sysvec vector cfunc
    idtentry \vector asm_\cfunc \cfunc has_error_code=0
.endm
```
* 比如说，`DECLARE_IDTENTRY(RESCHEDULE_VECTOR, sysvec_reschedule_ipi)` 展开以后会有：
  * 依照模板生成汇编例程 `asm_sysvec_reschedule_ipi`
  * 声明 C 函数 `sysvec_reschedule_ipi()`，该函数会被 `asm_sysvec_reschedule_ipi` 调用

### 定义 x86 APIC 中断向量
* 以 `Rescheduling interrupts` 和 `Function call interrupts` 为例，定义如下
* arch/x86/kernel/smp.c
```cpp
/*
 * Reschedule call back. KVM uses this interrupt to force a cpu out of
 * guest mode.
 */
DEFINE_IDTENTRY_SYSVEC_SIMPLE(sysvec_reschedule_ipi)
{
    ack_APIC_irq();
    trace_reschedule_entry(RESCHEDULE_VECTOR);
    inc_irq_stat(irq_resched_count);
    scheduler_ipi();
    trace_reschedule_exit(RESCHEDULE_VECTOR);
}

DEFINE_IDTENTRY_SYSVEC(sysvec_call_function)
{
    ack_APIC_irq();
    trace_call_function_entry(CALL_FUNCTION_VECTOR);
    inc_irq_stat(irq_call_count);
    generic_smp_call_function_interrupt();
    trace_call_function_exit(CALL_FUNCTION_VECTOR);
}
```
* 定义的宏如下：
```cpp
/**
 * DEFINE_IDTENTRY_SYSVEC - Emit code for system vector IDT entry points
 * @func:   Function name of the entry point
 *
 * irqentry_enter/exit() and irq_enter/exit_rcu() are invoked before the
 * function body. KVM L1D flush request is set.
 *
 * Runs the function on the interrupt stack if the entry hit kernel mode
 */
#define DEFINE_IDTENTRY_SYSVEC(func)                    \
static void __##func(struct pt_regs *regs);             \
                                    \
__visible noinstr void func(struct pt_regs *regs)           \
{                                   \
    irqentry_state_t state = irqentry_enter(regs);          \
                                    \
    instrumentation_begin();                    \
    kvm_set_cpu_l1tf_flush_l1d();                   \
    run_sysvec_on_irqstack_cond(__##func, regs);            \
    instrumentation_end();                      \
    irqentry_exit(regs, state);                 \
}                                   \
                                    \
static noinline void __##func(struct pt_regs *regs)

/**
 * DEFINE_IDTENTRY_SYSVEC_SIMPLE - Emit code for simple system vector IDT
 *                 entry points
 * @func:   Function name of the entry point
 *
 * Runs the function on the interrupted stack. No switch to IRQ stack and
 * only the minimal __irq_enter/exit() handling.
 *
 * Only use for 'empty' vectors like reschedule IPI and KVM posted
 * interrupt vectors.
 */
#define DEFINE_IDTENTRY_SYSVEC_SIMPLE(func)             \
static __always_inline void __##func(struct pt_regs *regs);     \
                                    \
__visible noinstr void func(struct pt_regs *regs)           \
{                                   \
    irqentry_state_t state = irqentry_enter(regs);          \
                                    \
    instrumentation_begin();                    \
    __irq_enter_raw();                      \
    kvm_set_cpu_l1tf_flush_l1d();                   \
    __##func (regs);                        \
    __irq_exit_raw();                       \
    instrumentation_end();                      \
    irqentry_exit(regs, state);                 \
}                                   \
                                    \
static __always_inline void __##func(struct pt_regs *regs)
```
* 比如说 `DEFINE_IDTENTRY_SYSVEC_SIMPLE(sysvec_reschedule_ipi)` 会展开：
  * 按照宏的模板展开的 C 函数 `sysvec_reschedule_ipi()`，由 `asm_sysvec_reschedule_ipi` 例程调用
  * 特定的 `__sysvec_reschedule_ipi()`，由 `sysvec_reschedule_ipi()` 函数调用

### 注册向量到 IDT
* 准备 IDT entries
  * arch/x86/kernel/idt.c
```cpp
/*
 * The APIC and SMP idt entries
 */
static const __initconst struct idt_data apic_idts[] = {
#ifdef CONFIG_SMP
    INTG(RESCHEDULE_VECTOR,         asm_sysvec_reschedule_ipi),
    INTG(CALL_FUNCTION_VECTOR,      asm_sysvec_call_function),
    INTG(CALL_FUNCTION_SINGLE_VECTOR,   asm_sysvec_call_function_single),
    INTG(IRQ_MOVE_CLEANUP_VECTOR,       asm_sysvec_irq_move_cleanup),
    INTG(REBOOT_VECTOR,         asm_sysvec_reboot),
#endif

#ifdef CONFIG_X86_THERMAL_VECTOR
    INTG(THERMAL_APIC_VECTOR,       asm_sysvec_thermal),
#endif

#ifdef CONFIG_X86_MCE_THRESHOLD
    INTG(THRESHOLD_APIC_VECTOR,     asm_sysvec_threshold),
#endif

#ifdef CONFIG_X86_MCE_AMD
    INTG(DEFERRED_ERROR_VECTOR,     asm_sysvec_deferred_error),
#endif

#ifdef CONFIG_X86_LOCAL_APIC
    INTG(LOCAL_TIMER_VECTOR,        asm_sysvec_apic_timer_interrupt),
    INTG(X86_PLATFORM_IPI_VECTOR,       asm_sysvec_x86_platform_ipi),
# ifdef CONFIG_HAVE_KVM
    INTG(POSTED_INTR_VECTOR,        asm_sysvec_kvm_posted_intr_ipi),
    INTG(POSTED_INTR_WAKEUP_VECTOR,     asm_sysvec_kvm_posted_intr_wakeup_ipi),
    INTG(POSTED_INTR_NESTED_VECTOR,     asm_sysvec_kvm_posted_intr_nested_ipi),
# endif
# ifdef CONFIG_IRQ_WORK
    INTG(IRQ_WORK_VECTOR,           asm_sysvec_irq_work),
# endif
    INTG(SPURIOUS_APIC_VECTOR,      asm_sysvec_spurious_apic_interrupt),
    INTG(ERROR_APIC_VECTOR,         asm_sysvec_error_interrupt),
#endif
};
```
* 设置 IDT entries
```cpp
start_kernel()
-> init_IRQ()
   -> x86_init.irqs.intr_init()
   => native_init_IRQ()
      -> idt_setup_apic_and_irq_gates()
         -> idt_setup_from_table(idt_table, apic_idts, ARRAY_SIZE(apic_idts), true)

static __init void
idt_setup_from_table(gate_desc *idt, const struct idt_data *t, int size, bool sys)
{
    gate_desc desc;

    for (; size > 0; t++, size--) {
        idt_init_desc(&desc, t);
        write_idt_entry(idt, t->vector, &desc);
        if (sys)
            set_bit(t->vector, system_vectors);
    }
}
```

## 发送 IPI 中断
* 通过写 APIC 的 Interrupt Command Register（`ICR`）寄存器来发送 IPI
* 对于 xAPIC 是 MMIO 映射的寄存器地址 `0xFEE0 0300`（0-31 bit）和 `0xFEE0 0310`（32-63 bit）
* 对于 x2APIC 是 MSR 地址 `0x830`

### IPI 是否可屏蔽？

* Linux 中大部分 IPI 都是以 **Fix** Delivery Mode 发送，行为和外设中断类似，接到 `INTR` pin，都属于可屏蔽的
* 以下一些场景发送的 IPI 都是 NMI 的方式，接入 `NMI` pin
  * crash 时 kdump 发出的 IPI
  * 让 CPU 产生 backtrace，sysrq 有类似的命令
  * 通过 `/dev/mcelog` 注入 MCE
  * Stop CPU
* 另外还有一些 IPI 是多处理器启动时 BSP 启动 AP 时用到的 `INIT` 和 `SIPI`，从 LAPIC 结构图上看走的也是 `NMI` pin，应该也属于不可屏蔽的

## Check if specific interrupt is enabled ( IPI )
* [Check if specific interrupt is enabled ( IPI )](https://stackoverflow.com/questions/51174097/check-if-specific-interrupt-is-enabled-ipi)
> The term **IPI** designates a *category* of interrupts, specifically: [NMI](https://wiki.osdev.org/Non_Maskable_Interrupt), [SMI](https://wiki.osdev.org/System_Management_Mode), SIPI, INIT and Fixed interrupts.
> Of these, the NMI, SMI, SIPI and INIT cannot be masked without disabling the LAPIC globally.
---
> The [NMI (Non-maskable Interrupt)](https://wiki.osdev.org/Non_Maskable_Interrupt) was designed to be non-maskable from the beginning, as the name implies.
> There is a hack to mask it: setting the `bit7` of port `70h`.
> However, this is a hardware trick, it ties the #NMI (today it is the LINT1 but this is configurable) high.
> The IPIs are sent through a software message so this trick won't work.
> 
> The [SMI (System Management Interrupt)](https://wiki.osdev.org/System_Management_Mode) is used to enter the SMM (see previous link), a mode designed to be as trasparent to the software as possible.
> It is not maskable, for what the software is concerned, it doesn't exist.
> 
> The [INIT and SIPI (Startup-IPI)](https://wiki.osdev.org/Symmetric_Multiprocessing#Initialisation_of_an_old_SMP_system) interrupts are used to reset and wake up a CPU.
> They cannot be masked by design (the BIOS usually put the APs, Application Processors, to sleep with a `cli / hlt` sequence).
> 
> The Fixed interrupts can be masked with the `IF` flag or when a higher priority fixed interrupt ISR is executing (just like > the legacy IRQs with the PIC).
---
> It's possible to mask only the Fixed interrupts, something that may actually corresponds to the common understanding of the term IPI, by *soft* disabling the APIC.
> This can be done by clearing `bit8` of the *Spurious Interrupt Vector Register* at offset `0f0h` from the LAPIC base (the LAPIC base is set in the `IA32_APIC_BASE` MSR, its address is `1bh`).
> Of course, clearing `IF` will also do.
>
> Alternatively it's possible to disable the LAPIC entirely (it won't respond to any IPI) by clearing bit11 of the `IA32_APIC_BASE` MSR.
---
> To check if the IPIs are enable you have to check the `IF` flag, the *Spurious Interrupt Vector Register* and the `IA32_APIC_BASE` MSR.

# References
- [Intel SDM Chapter 10 APIC tcbbd的博客](https://tcbbd.moe/ref-and-spec/intel-sdm/sdm-basic-ch10/)