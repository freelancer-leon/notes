# Kprobes
* Kprobes debugfs 接口
  * `/sys/kernel/debug/kprobes`
* kprobes sysctl 接口
  * `/proc/sys/debug/kprobes-optimization`：打开或关闭 kprobes 优化

## 一些需要注意的地方
* 如果在内联函数中安装探针，则 Kprobes 不会尝试追查该函数的所有内联实例并在那里安装探针。而 gcc 可能会内联一个函数而不会被询问，因此如果您没有看到预期的探测结果，请记住这一点。
* Kprobes 不会尝试阻止探针处理程序彼此踩踏-例如，先探测`printk()`然后从探针处理程序中调用`printk()`。如果探针处理程序击中了探针，则该第二个探针的处理程序将不会在该实例中运行，并且第二个探针的`kprobe.nmissed`成员将递增。
* 探针处理程序在 **禁用抢占** 或 **禁用中断** 的情况下运行，这取决于体系结构和优化状态。（例如，kretprobe 处理程序和优化的 kprobe 处理程序在 x86/x86-64 上运行时没有禁用中断）。在任何情况下，您的处理程序都不应放弃 CPU（例如，尝试获取信号量或等待I /O）。

# kprobes

## `struct kprobe`结构
```c
struct kprobe {
    struct hlist_node hlist;

    /* list of kprobes for multi-handler support */
    struct list_head list;

    /*count the number of times this probe was temporarily disarmed */
    unsigned long nmissed;

    /* location of the probe point */
    kprobe_opcode_t *addr;

    /* Allow user to indicate symbol name of the probe point */
    const char *symbol_name;

    /* Offset into the symbol */
    unsigned int offset;

    /* Called before addr is executed. */
    kprobe_pre_handler_t pre_handler;

    /* Called after addr is executed, unless... */
    kprobe_post_handler_t post_handler;

    /*
     * ... called if executing addr causes a fault (eg. page fault).
     * Return 1 if it handled fault, otherwise kernel will see it.
     */
    kprobe_fault_handler_t fault_handler;

    /*
     * ... called if breakpoint trap occurs in probe handler.
     * Return 1 if it handled break, otherwise kernel will see it.
     */
    kprobe_break_handler_t break_handler;

    /* Saved opcode (which has been replaced with breakpoint) */
    kprobe_opcode_t opcode;

    /* copy of the original instruction */
    struct arch_specific_insn ainsn;

    /*
     * Indicates various status flags.
     * Protected by kprobe_mutex after this kprobe is registered.
     */
    u32 flags;
};
...*```
```

## 注册`kprobes`探针
```c
kernel/kprobes.c
register_kprobe()
-> prepare_kprobe()
   arch/x86/kernel/kprobes/core.c
   -> arch_prepare_kprobe()
      -> arch_copy_kprobe()
         -> text_poke()
            -> __text_poke()
```

## 激活`kprobe`探针
```c
arch/x86/entry/entry_64.S
idtentry int3           do_int3         has_error_code=0    paranoid=1 shift_ist=DEBUG_STACK
  arch/x86/kernel/traps.c
  -> do_int3()
     -> kprobe_int3_handler()
        +-> if (!p->pre_handler || !p->pre_handler(p, regs))
        |      setup_singlestep()
        +-> if (p->break_handler && p->break_handler(p, regs))
               setup_singlestep()
```

# kretprobes

## `struct kretprobe`结构
```c
/*
 * Function-return probe -
 * Note:
 * User needs to provide a handler function, and initialize maxactive.
 * maxactive - The maximum number of instances of the probed function that
 * can be active concurrently.
 * nmissed - tracks the number of times the probed function's return was
 * ignored, due to maxactive being too low.
 *
 */
struct kretprobe {
    struct kprobe kp;
    kretprobe_handler_t handler;
    kretprobe_handler_t entry_handler;
    int maxactive;
    int nmissed;
    size_t data_size;
    struct hlist_head free_instances;
    raw_spinlock_t lock;
};
```

## 注册`kretprobes`探针
* kernel/kprobes.c
```c
register_kretprobes()
-> register_kretprobe(struct kretprobe *rp)
   -> rp->kp.pre_handler = pre_handler_kretprobe;
   -> register_kprobe(&rp->kp)
```
* 实际上这里还是利用了`kprobe`完成指令替换。
* 注意给`kprobe`探针的`pre_handler`成员的赋指，这样在探针插入的指令执行前，会先调用`pre_handler_kretprobe()`。

## 修改函数返回地址
* 调用`kprobe`探针的`pre_handler`回调的时机就是修改函数返回地址的时机，这之前的过程列过。
* 之前`pre_handler`回调赋值为`pre_handler_kretprobe()`
* 函数`pre_handler_kretprobe()`的主要任务是：
  * 调用`kretprobe`的`entry_handler`
  * 修改探针所在函数的返回地址为`kretprobe_trampoline()`
  * 保存探针所在函数的返回地址
* kernel/kprobes.c
```c
pre_handler_kretprobe()
-> rp->entry_handler()
-> arch/x86/kernel/kprobes/core.c
   arch_prepare_kretprobe()
```
* `arch_prepare_kretprobe()`由不同体系结构各自实现
* arch/x86/kernel/kprobes/core.c
```c
void arch_prepare_kretprobe(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    unsigned long *sara = stack_addr(regs);

    ri->ret_addr = (kprobe_opcode_t *) *sara;

    /* Replace the return addr with trampoline addr */
    /*替换函数返回地址为蹦床函数*/
    *sara = (unsigned long) &kretprobe_trampoline;
}
NOKPROBE_SYMBOL(arch_prepare_kretprobe);
```

* arch/x86/kernel/ptrace.c
```c
/*
 * X86_32 CPUs don't save ss and esp if the CPU is already in kernel mode
 * when it traps.  The previous stack will be directly underneath the saved
 * registers, and 'sp/ss' won't even have been saved. Thus the '&regs->sp'.
 *
 * Now, if the stack is empty, '&regs->sp' is out of range. In this
 * case we try to take the previous stack. To always return a non-null
 * stack pointer we fall back to regs as stack if no previous stack
 * exists.
 *
 * This is valid only for kernel mode traps.
 */
unsigned long kernel_stack_pointer(struct pt_regs *regs)
{
    unsigned long context = (unsigned long)regs & ~(THREAD_SIZE - 1);
    unsigned long sp = (unsigned long)&regs->sp;
    u32 *prev_esp;

    if (context == (sp & ~(THREAD_SIZE - 1)))
        return sp;

    prev_esp = (u32 *)(context);
    if (*prev_esp)
        return (unsigned long)*prev_esp;

    return (unsigned long)regs;
}
EXPORT_SYMBOL_GPL(kernel_stack_pointer);
```

## 激活`kretprobe`探针
* 之前在函数进入时，通过最终调用`arch_prepare_kretprobe()`修改了函数 **返回地址** 为蹦床函数`kretprobe_trampoline()`
* 函数返回时，通过`kretprobe_trampoline()`函数激活`kretprobe`探针，调用`kretprobe_handler_t handler`回调
* arch/x86/kernel/kprobes/core.c
```c
/*
 * When a retprobed function returns, this code saves registers and
 * calls trampoline_handler() runs, which calls the kretprobe's handler.
 */
asm(
    ".global kretprobe_trampoline\n"  /*这里通过内联汇编的方式定义了一个 C 函数*/
    ".type kretprobe_trampoline, @function\n"
    "kretprobe_trampoline:\n"
#ifdef CONFIG_X86_64
    /* We don't bother saving the ss register */
    "   pushq %rsp\n"
    "   pushfq\n"
    SAVE_REGS_STRING               /*保存寄存器到栈上*/
    "   movq %rsp, %rdi\n"
    "   call trampoline_handler\n" /*最重要的，调用trampoline_handler()*/
    /* Replace saved sp with true return address. */
    "   movq %rax, 152(%rsp)\n" /*x86将函数返回值放在rax寄存器中，trampoline_handler()的返回值是真实的返回地址*/
    RESTORE_REGS_STRING            /*恢复寄存器*/
    "   popfq\n"
#else
    "   pushf\n"
    SAVE_REGS_STRING
    "   movl %esp, %eax\n"
    "   call trampoline_handler\n"
    /* Move flags to cs */
    "   movl 56(%esp), %edx\n"
    "   movl %edx, 52(%esp)\n"
    /* Replace saved flags with true return address. */
    "   movl %eax, 56(%esp)\n"
    RESTORE_REGS_STRING
    "   popf\n"
#endif
    "   ret\n"
    ".size kretprobe_trampoline, .-kretprobe_trampoline\n"
);
NOKPROBE_SYMBOL(kretprobe_trampoline);
STACK_FRAME_NON_STANDARD(kretprobe_trampoline);

/*
 * Called from kretprobe_trampoline
 */
__visible __used void *trampoline_handler(struct pt_regs *regs)
{
	struct kretprobe_instance *ri = NULL;
	struct hlist_head *head, empty_rp;
	struct hlist_node *tmp;
	unsigned long flags, orig_ret_address = 0;
	unsigned long trampoline_address = (unsigned long)&kretprobe_trampoline;
	kprobe_opcode_t *correct_ret_addr = NULL;

	INIT_HLIST_HEAD(&empty_rp);
	kretprobe_hash_lock(current, &head, &flags);
	/* fixup registers */
#ifdef CONFIG_X86_64
	regs->cs = __KERNEL_CS;
#else
	regs->cs = __KERNEL_CS | get_kernel_rpl();
	regs->gs = 0;
#endif
	regs->ip = trampoline_address;
	regs->orig_ax = ~0UL;

	/*
	 * It is possible to have multiple instances associated with a given
	 * task either because multiple functions in the call path have
	 * return probes installed on them, and/or more than one
	 * return probe was registered for a target function.
	 *
	 * We can handle this because:
	 *     - instances are always pushed into the head of the list
	 *     - when multiple return probes are registered for the same
	 *	 function, the (chronologically) first instance's ret_addr
	 *	 will be the real return address, and all the rest will
	 *	 point to kretprobe_trampoline.
	 */
	hlist_for_each_entry_safe(ri, tmp, head, hlist) {
		if (ri->task != current)
			/* another task is sharing our hash bucket */
			continue;

		orig_ret_address = (unsigned long)ri->ret_addr;

		if (orig_ret_address != trampoline_address)
			/*
			 * This is the real return address. Any other
			 * instances associated with this task are for
			 * other calls deeper on the call stack
			 */
			break;
	}

	kretprobe_assert(ri, orig_ret_address, trampoline_address);

	correct_ret_addr = ri->ret_addr;
	hlist_for_each_entry_safe(ri, tmp, head, hlist) {
		if (ri->task != current)
			/* another task is sharing our hash bucket */
			continue;

		orig_ret_address = (unsigned long)ri->ret_addr; /* 之前在 arch_prepare_kretprobe() 处保存的真实返回地址 */
		if (ri->rp && ri->rp->handler) {
			__this_cpu_write(current_kprobe, &ri->rp->kp);
			get_kprobe_ctlblk()->kprobe_status = KPROBE_HIT_ACTIVE;
			ri->ret_addr = correct_ret_addr;
			ri->rp->handler(ri, regs);  /* kretprobe 的 handler 回调最终在此处被调用 */
			__this_cpu_write(current_kprobe, NULL);
		}

		recycle_rp_inst(ri, &empty_rp);

		if (orig_ret_address != trampoline_address)
			/*
			 * This is the real return address. Any other
			 * instances associated with this task are for
			 * other calls deeper on the call stack
			 */
			break;
	}

	kretprobe_hash_unlock(current, &flags);

	hlist_for_each_entry_safe(ri, tmp, &empty_rp, hlist) {
		hlist_del(&ri->hlist);
		kfree(ri);
	}
	return (void *)orig_ret_address;  /*返回真实的函数原返回地址*/
}
NOKPROBE_SYMBOL(trampoline_handler);

```
