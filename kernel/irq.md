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
