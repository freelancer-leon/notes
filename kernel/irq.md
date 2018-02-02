# 中断

# x86的do_IRQ()
* `struct pt_regs`结构体的定义见`arch/x86/include/uapi/asm/ptrace.h`
* Per-CPU的`struct pt_regs`类型的`irq_regs`变量用于保存被中断时的寄存器的值。
  * 这些值是在调用`do_IRQ()`前在汇编入口例程中保存的。
* arch/x86/include/asm/irq_regs.h
```c
DECLARE_PER_CPU(struct pt_regs *, irq_regs);

static inline struct pt_regs *get_irq_regs(void)
{
        return this_cpu_read(irq_regs);
}

static inline struct pt_regs *set_irq_regs(struct pt_regs *new_regs)
{
        struct pt_regs *old_regs;
        /*被中断进程的寄存器的值的地址存入old_regs，irq_regs写入新的值*/
        old_regs = get_irq_regs();
        this_cpu_write(irq_regs, new_regs);
        /*该函数返回存旧寄存器值的地址*/
        return old_regs;
}
```
* arch/x86/kernel/irq.c
```c
/*
 * do_IRQ handles all normal device IRQ's (the special
 * SMP cross-CPU interrupts have their own specific
 * handlers).
 */
__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
        struct pt_regs *old_regs = set_irq_regs(regs);
        struct irq_desc * desc;
        /* high bit used in ret_from_ code  */
        unsigned vector = ~regs->orig_ax;

        /*
         * NB: Unlike exception entries, IRQ entries do not reliably
         * handle context tracking in the low-level entry code.  This is
         * because syscall entries execute briefly with IRQs on before
         * updating context tracking state, so we can take an IRQ from
         * kernel mode with CONTEXT_USER.  The low-level entry code only
         * updates the context if we came from user mode, so we won't
         * switch to CONTEXT_KERNEL.  We'll fix that once the syscall
         * code is cleaned up enough that we can cleanly defer enabling
         * IRQs.
         */

        entering_irq();

        /* entering_irq() tells RCU that we're not quiescent.  Check it. */
        RCU_LOCKDEP_WARN(!rcu_is_watching(), "IRQ failed to wake up RCU");

        desc = __this_cpu_read(vector_irq[vector]);

        if (!handle_irq(desc, regs)) {
                ack_APIC_irq();

                if (desc != VECTOR_RETRIGGERED) {
                        pr_emerg_ratelimited("%s: %d.%d No irq handler for vector\n",
                                             __func__, smp_processor_id(),
                                             vector);
                } else {
                        __this_cpu_write(vector_irq[vector], VECTOR_UNUSED);
                }   
        }   

        exiting_irq();

        set_irq_regs(old_regs);
        return 1;
}
```
* arch/x86/entry/entry_64.S
```c
/*
 * Build the entry stubs with some assembler magic.
 * We pack 1 stub into every 8-byte block.
 */
        .align 8
ENTRY(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
        pushq   $(~vector+0x80)                 /* Note: always in signed byte range */
    vector=vector+1
        jmp     common_interrupt  /*跳转至x86通用的汇编中断处理*/
        .align  8
    .endr
END(irq_entries_start)

/*
 * Interrupt entry/exit.
 *
 * Interrupt entry points save only callee clobbered registers in fast path.
 *
 * Entry runs with interrupts off.
 */

/* 0(%rsp): ~(interrupt number) */
        .macro interrupt func
        cld
        ALLOC_PT_GPREGS_ON_STACK
        SAVE_C_REGS
        SAVE_EXTRA_REGS
        ENCODE_FRAME_POINTER

        testb   $3, CS(%rsp)
        jz      1f

        /*
         * IRQ from user mode.  Switch to kernel gsbase and inform context
         * tracking that we're in kernel mode.
         */
        SWAPGS

        /*
         * We need to tell lockdep that IRQs are off.  We can't do this until
         * we fix gsbase, and we should do it before enter_from_user_mode
         * (which can take locks).  Since TRACE_IRQS_OFF idempotent,
         * the simplest way to handle it is to just call it twice if
         * we enter from user mode.  There's no reason to optimize this since
         * TRACE_IRQS_OFF is a no-op if lockdep is off.
         */
        TRACE_IRQS_OFF

        CALL_enter_from_user_mode

1:
        /*
         * Save previous stack pointer, optionally switch to interrupt stack.
         * irq_count is used to check if a CPU is already on an interrupt stack
         * or not. While this is essentially redundant with preempt_count it is
         * a little cheaper to use a separate counter in the PDA (short of
         * moving irq_enter into assembly, which would be too much work)
         */
        movq    %rsp, %rdi
        incl    PER_CPU_VAR(irq_count)
        cmovzq  PER_CPU_VAR(irq_stack_ptr), %rsp
        pushq   %rdi
        /* We entered an interrupt context - irqs are off: */
        TRACE_IRQS_OFF

        call    \func   /* rdi points to pt_regs */
        .endm

        /*
         * The interrupt stubs push (~vector+0x80) onto the stack and
         * then jump to common_interrupt.
         */
        .p2align CONFIG_X86_L1_CACHE_SHIFT
common_interrupt:
        ASM_CLAC
        addq    $-0x80, (%rsp)                  /* Adjust vector to [-256, -1] range */
        interrupt do_IRQ       /*跳转至x86通用的 C 中断处理，在上面列出了*/
        /* 0(%rsp): old RSP */
ret_from_intr:                 /*注意，这里是连着的，do_IRQ 返回后会接着执行后面的指令*/
        DISABLE_INTERRUPTS(CLBR_ANY)
        TRACE_IRQS_OFF
        decl    PER_CPU_VAR(irq_count)

        /* Restore saved previous stack */
        popq    %rsp

        testb   $3, CS(%rsp)  /*读寄存器，判断中断是返回到 user space 还是 kernel space*/
        jz      retint_kernel

        /* Interrupt came from user space */
GLOBAL(retint_user)
        mov     %rsp,%rdi
        call    prepare_exit_to_usermode
        TRACE_IRQS_IRETQ
        SWAPGS
        jmp     restore_regs_and_iret

/* Returning to kernel space */
retint_kernel:
#ifdef CONFIG_PREEMPT
        /* Interrupts are off */
        /* Check if we need preemption */
        bt      $9, EFLAGS(%rsp)                /* were interrupts off? */
        jnc     1f
0:      cmpl    $0, PER_CPU_VAR(__preempt_count) /*读取抢占计数，看能否进行内核抢占*/
        jnz     1f                    /*如果抢占计数不为 0，通过跳转到 lable 1 返回原执行点*/
        call    preempt_schedule_irq  /*如果抢占计数为 0，触发内核抢占，这里是内核抢占的一个点*/
        jmp     0b   /* preempt_schedule_irq 返回后再次跳回 label 0 检查抢占计数 */
1:
#endif
        /*
         * The iretq could re-enable interrupts:
         */
        TRACE_IRQS_IRETQ
```
