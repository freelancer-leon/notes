# Stop Machine

## 内核线程的创建
* `start_kernel()` 的最后会调用到 `rest_init()`，它会将第一个内核线程 `kthreadd` 线程创建出来，等待调度
  * `kthreadd` 的 PID 通常会是 `2`
  * `kthreadd` 负责创建其他内核线程
```cpp
//init/main.c
start_kernel()
-> arch_call_rest_init()
   -> rest_init()
         //kernel/fork.c
      -> pid = kernel_thread(kthreadd, NULL, NULL, CLONE_FS | CLONE_FILES) //创建内核线程 kthreadd
         -> kernel_clone() //clone 新线程（包括用户态进程）的入口函数，就是以前的 do_fork()
            -> copy_process(NULL, trace, NUMA_NO_NODE, args) //负责拷贝父进程的信息到子进程
                  if (args->kthread)
                     p->flags |= PF_KTHREAD; //进程的创建的 PF_KTHREAD 标志在这加上的，影响 copy_thread() 的走向
               -> copy_thread(p, args) //拷贝线程相关的信息，由各 arch 实现
      -> kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns)
```
* `kernel_thread()` 是创建内核线程的接口函数，在这个路径上，`.fn` 域被赋值为 `kthreadd()`
```cpp
/*
 * Create a kernel thread.
 */
pid_t kernel_thread(int (*fn)(void *), void *arg, const char *name,
            unsigned long flags)
{
    struct kernel_clone_args args = {
        .flags      = ((lower_32_bits(flags) | CLONE_VM |
                    CLONE_UNTRACED) & ~CSIGNAL),
        .exit_signal    = (lower_32_bits(flags) & CSIGNAL),
        .fn     = fn,      //在这个路径上为 kthreadd()
        .fn_arg     = arg,
        .name       = name,
        .kthread    = 1,   //这个标志会影响到进程的创建 flags 的 PF_KTHREAD 标志
    };

    return kernel_clone(&args); //创建新线程（包括用户态进程）的入口函数，就是以前的 do_fork()
}
```
### x86 的 `copy_thread()`
* 看 `copy_thread()` 之前要搞清楚涉及的两个结构体 `struct fork_frame` 和 `struct inactive_task_frame`
  * `struct fork_frame` 包含 `struct inactive_task_frame`
* `struct inactive_task_frame` 结构构成非活跃任务的的栈帧，进程切换时，非活跃任务的 `task_struct.thread.sp` 指针（`TASK_threadsp`）指向它
  * 因此，它的域的顺序必须与上下文切换时 `__switch_to_asm()` 的代码一致
```cpp
/*
 * This is the structure pointed to by thread.sp for an inactive task.  The
 * order of the fields must match the code in __switch_to_asm().
 */
struct inactive_task_frame {
#ifdef CONFIG_X86_64
    unsigned long r15;
    unsigned long r14;
    unsigned long r13;
    unsigned long r12;
#else
    unsigned long flags;
    unsigned long si;
    unsigned long di;
#endif
    unsigned long bx;

    /*
     * These two fields must be together.  They form a stack frame header,
     * needed by get_frame_pointer().
     */
    unsigned long bp;
    unsigned long ret_addr;
};
```
* 注意看 `__switch_to_asm()` 压栈和出栈的顺序，与 `struct inactive_task_frame` 的域的顺序保持一致
* arch/x86/entry/entry_64.S
```cpp
/*
 * %rdi: prev task
 * %rsi: next task
 */
.pushsection .text, "ax"
SYM_FUNC_START(__switch_to_asm)
    /*
     * Save callee-saved registers
     * This must match the order in inactive_task_frame
     */
    pushq   %rbp
    pushq   %rbx
    pushq   %r12
    pushq   %r13
    pushq   %r14
    pushq   %r15

    /* switch stack */
    movq    %rsp, TASK_threadsp(%rdi) //保存换出进程的栈指针到它自己的栈上，TASK_threadsp 即 task_struct.thread.sp 指向栈顶
    movq    TASK_threadsp(%rsi), %rsp //载入换入进程的栈指针，由此完成真正意义上的进程切换
    //切换栈防护的 canary
#ifdef CONFIG_STACKPROTECTOR
    movq    TASK_stack_canary(%rsi), %rbx
    movq    %rbx, PER_CPU_VAR(fixed_percpu_data) + FIXED_stack_canary
#endif

    /*
     * When switching from a shallower to a deeper call stack
     * the RSB may either underflow or use entries populated
     * with userspace addresses. On CPUs where those concerns
     * exist, overwrite the RSB with entries which capture
     * speculative execution to prevent attack.
     */
    FILL_RETURN_BUFFER %r12, RSB_CLEAR_LOOPS, X86_FEATURE_RSB_CTXSW

    /* restore callee-saved registers */
    popq    %r15  //进程切换完成后将 inactive_task_frame 的域的值弹出到寄存器，恢复现场
    popq    %r14
    popq    %r13
    popq    %r12
    popq    %rbx
    popq    %rbp

    jmp __switch_to
SYM_FUNC_END(__switch_to_asm)
.popsection
```
* 最精妙的地方在最后，`inactive_task_frame.ret_addr` 返回地址的值没有用汇编指令显式弹出的用意。
  * 由于 `__switch_to()` 是用 `jmp` 指令跳转过去的，因此 `__switch_to()` 返回时的 `ret` 等同于 `__switch_to_asm` 例程返回
  * 通常进程切换是 `__schedule() -> context_switch() -> switch_to() -> __switch_to_asm` 这样一个路径，返回也是通过这样一个路径返回调度点
  * 但我们这里是新创建的内核线程，因此，只要将 `inactive_task_frame.ret_addr` 统一设置为 `ret_from_fork_asm`，当新进程被调度的时候，内核线程从这里开始执行，后面我们会接着分析这一过程
  * 这里只要知道，`__switch_to()` 返回时，实际上是将 `inactive_task_frame.ret_addr` 的值弹出到 `$RIP` 完成到 `ret_from_fork_asm` 的跳转，然后再通过一系列调用跳转到内核线程函数的入口地址
```cpp
struct fork_frame {
    struct inactive_task_frame frame;
    struct pt_regs regs;
};
```
* 看看 x86 的 `copy_thread()` 的具体实现
  * 参数 `const struct kernel_clone_args * args` 就指向上面定义的 `struct kernel_clone_args args`
  * arch/x86/kernel/process.c
```cpp
int copy_thread(struct task_struct *p, const struct kernel_clone_args *args)
{
    unsigned long clone_flags = args->flags;
    unsigned long sp = args->stack;
    unsigned long tls = args->tls;
    struct inactive_task_frame *frame;
    struct fork_frame *fork_frame;
    struct pt_regs *childregs;
    unsigned long new_ssp;
    int ret = 0;
    //task_pt_regs(p) 返回任务内核栈的栈顶（高地址）-（struct pt_regs）的位置
    childregs = task_pt_regs(p);
    fork_frame = container_of(childregs, struct fork_frame, regs); //fork_frame 指向 struct fork_frame 实例的起始地址
    frame = &fork_frame->frame; //因为 .frame 域是第一个成员，其实也指向 struct fork_frame 实例的起始地址
    //根据 $RSP 的值和配置设置帧指针
    frame->bp = encode_frame_pointer(childregs);
    frame->ret_addr = (unsigned long) ret_from_fork_asm; //这个赋值很关键！让子进程再也回溯不到它的父进程创建它的路径了
    p->thread.sp = (unsigned long) fork_frame; //刷新栈底指针的位置为准备好的子进程栈底
    p->thread.io_bitmap = NULL; //重置子进程用户态能用的 I/O 地址空间的位图
    p->thread.iopl_warn = 0;
    memset(p->thread.ptrace_bps, 0, sizeof(p->thread.ptrace_bps));

#ifdef CONFIG_X86_64
    current_save_fsgs(); //保存当前进程的 FS、GS 寄存器
    p->thread.fsindex = current->thread.fsindex;
    p->thread.fsbase = current->thread.fsbase;
    p->thread.gsindex = current->thread.gsindex;
    p->thread.gsbase = current->thread.gsbase;

    savesegment(es, p->thread.es); //刷新子进程的 ES 寄存器
    savesegment(ds, p->thread.ds); //刷新子进程的 DS 寄存器
    //不允许子进程修改 LAM 的设置
    if (p->mm && (clone_flags & (CLONE_VM | CLONE_VFORK)) == CLONE_VM)
        set_bit(MM_CONTEXT_LOCK_LAM, &p->mm->context.flags);
#else
...
#endif
    //给子进程分配新的 shadow stack（CET）
    /*
     * Allocate a new shadow stack for thread if needed. If shadow stack,
     * is disabled, new_ssp will remain 0, and fpu_clone() will know not to
     * update it.
     */
    new_ssp = shstk_alloc_thread_stack(p, clone_flags, args->stack_size);
    if (IS_ERR_VALUE(new_ssp))
        return PTR_ERR((void *)new_ssp);
    //克隆父进程的 FPU
    fpu_clone(p, clone_flags, args->fn, new_ssp);
    //源自 kernel_thread() 的 args.kthread = 1
    /* Kernel thread ? */
    if (unlikely(p->flags & PF_KTHREAD)) {
        p->thread.pkru = pkru_get_init_value(); //初始化 PKRU 的值
        memset(childregs, 0, sizeof(struct pt_regs));
        kthread_frame_init(frame, args->fn, args->fn_arg); //给 ret_from_fork_asm 传参在这设定，后面展开讲
        return 0; //子进程是内核线程时不进行如下处理
    }
    //能走到这的子进程必是用户空间进程。复制父进程的 PKRU 的值，tsk->thread.pkru 只有在被调度出去时是有效的
    /*
     * Clone current's PKRU value from hardware. tsk->thread.pkru
     * is only valid when scheduled out.
     */
    p->thread.pkru = read_pkru();
    //复制父进程通用寄存器的值给子进程，设置个别通用寄存器的值为 0
    frame->bx = 0;
    *childregs = *current_pt_regs();
    childregs->ax = 0;
    if (sp)
        childregs->sp = sp; //子进程被唤醒时的 $RSP 指向父进程的内核栈最底部 task_struct.stack
    //一个用户空间的线程，但没有返回 ret_after_fork()。为了给类似 GDB 这样的工具指示，我们重置栈和指令指针。
    //它执行与内核线程相同的内核栈帧设置以返回到内核函数。
    if (unlikely(args->fn)) {
        /*
         * A user space thread, but it doesn't return to
         * ret_after_fork().
         *
         * In order to indicate that to tools like gdb,
         * we reset the stack and instruction pointers.
         *
         * It does the same kernel frame setup to return to a kernel
         * function that a kernel thread does.
         */
        childregs->sp = 0;
        childregs->ip = 0;
        kthread_frame_init(frame, args->fn, args->fn_arg);
        return 0;
    }
    //如果 clone 标志带 CLONE_SETTLS，给子进程设置新的线程本地存储 TLS
    /* Set a new TLS for the child thread? */
    if (clone_flags & CLONE_SETTLS)
        ret = set_new_tls(p, tls);
    //如果父进程的 I/O port 位图是设置过的，子进程可以继承父进程的 I/O 位图
    if (!ret && unlikely(test_tsk_thread_flag(current, TIF_IO_BITMAP)))
        io_bitmap_share(p);

    return ret;
}
```

#### 初始化内核栈帧 `kthread_frame_init()`
* 对于创建内核线程，`copy_thread()` 调用了一个很重要的函数就是 `kthread_frame_init()`
  * 它将准备好的栈帧的上的 `rbx` 和 `r12` 位置分别设置为内核线程的接口函数 `kernel_thread()` 的 `fn` 和 `args` 参数
  * 并且与 `ret_from_fork_asm` 约定，将 `rbx` 和 `r12` 的值作为参数传给 `ret_from_fork()` 的 `fn` 和 `fn_args` 参数
  * 从而实现了作为子进程的内核线程被调度后，会调用 `kernel_thread()` 创建时给的提供的内核线程运行函数，并用上相应的参数
```cpp
static inline void kthread_frame_init(struct inactive_task_frame *frame,
                      int (*fun)(void *), void *arg)
{
    frame->bx = (unsigned long)fun;
#ifdef CONFIG_X86_32
    frame->di = (unsigned long)arg;
#else
    frame->r12 = (unsigned long)arg;
#endif
}
```

### Fork 新进程的入口地址 `ret_from_fork_asm`
* `copy_thread()` 时将为 fork 出的新进程准备的帧上的返回地址预设为 `frame->ret_addr = (unsigned long) ret_from_fork_asm`
  * 由于 `__switch_to()` 是用 `jmp` 指令跳转过去的，因此 `__switch_to()` 返回时的 `ret` 等同于 `__switch_to_asm` 例程返回
  * 通常进程切换是 `__schedule() -> context_switch() -> switch_to() -> __switch_to_asm` 这样一个路径，返回也是通过这样一个路径返回调度点
  * 但新创建的内核线程时将 `inactive_task_frame.ret_addr` 统一设置为 `ret_from_fork_asm`，当新进程被调度的时候，内核线程就开始执行了
  * `__switch_to()` 返回时，将之前被压到栈上 `inactive_task_frame.ret_addr` 的值弹出到 `$RIP` 完成到 `ret_from_fork_asm` 的跳转，然后再通过一系列调用跳转到内核线程函数的入口地址
* 调用 `ret_from_fork_asm` 之前：
  * `$RAX` 是 `__switch_to()` 函数的返回值，即指向前一个进程的 `struct task_struct` 实例
  * `$RSP` 是为之前准备好的 `struct fork_frame` 实例的 `struct pt_regs regs` 域，`struct inactive_task_frame frame` 域部分已经被 `__switch_to_asm` 进程栈切换后的一些列操作弹出到对于的通用寄存器里的，而 `frame.ret_addr` 也随着 `__switch_to()` 的返回（也是 `__swith_to_asm` 的返回）弹出到 `$RIP` 了
  * `$RBX` 是之前 `copy_process() -> kthread_frame_init()` 预设好的内核线程函数
  * `$R12` 是之前预设好的内核线程函数的参数
* arch/x86/entry/entry_64.S
```cpp
/*
 * A newly forked process directly context switches into this address.
 *
 * rax: prev task we switched from
 * rbx: kernel thread func (NULL for user thread)
 * r12: kernel thread arg
 */
.pushsection .text, "ax"
SYM_CODE_START(ret_from_fork_asm)
    /*
     * This is the start of the kernel stack; even through there's a
     * register set at the top, the regset isn't necessarily coherent
     * (consider kthreads) and one cannot unwind further.
     *
     * This ensures stack unwinds of kernel threads terminate in a known
     * good state.
     */
    UNWIND_HINT_END_OF_STACK
    ANNOTATE_NOENDBR // copy_thread
    CALL_DEPTH_ACCOUNT

    movq    %rax, %rdi      /* prev */   //第一个参数为前一个进程
    movq    %rsp, %rsi      /* regs */   //第二个参数为通用寄存器结构
    movq    %rbx, %rdx      /* fn */     //第三个参数内核线程函数（用户空间线程为 NULL）
    movq    %r12, %rcx      /* fn_arg */ //第四个参数内核线程的参数
    call    ret_from_fork

    /*
     * Set the stack state to what is expected for the target function
     * -- at this point the register set should be a valid user set
     * and unwind should work normally.
     */
    UNWIND_HINT_REGS
    jmp swapgs_restore_regs_and_return_to_usermode
SYM_CODE_END(ret_from_fork_asm)
.popsection
```
* `ret_from_fork_asm` 调用 `ret_from_fork()`，再由后者调用内核线程函数，至此 **新** 内核线程开始执行
  * arch/x86/kernel/process.c
```cpp
__visible void ret_from_fork(struct task_struct *prev, struct pt_regs *regs,
                     int (*fn)(void *), void *fn_arg)
{
    schedule_tail(prev);

    /* Is this a kernel thread? */
    if (unlikely(fn)) {
        fn(fn_arg);
        /*
         * A kernel thread is allowed to return here after successfully
         * calling kernel_execve().  Exit to userspace to complete the
         * execve() syscall.
         */
        regs->ax = 0;
    }

    syscall_exit_to_user_mode(regs);
}
```

### 创建内核线程的 kthreadd 内核线程
* 负责创建内核线程的 kthreadd 内核线程被 `rest_init()` 创建出来后就等待被调度，接着再来看看它是怎么做的
  * 它保持在一个死循环 + 睡眠的状态中。如果被唤醒，就检查内核线程创建列表是否为空，
    * 如果不为空就取得内核线程创建信息 `struct kthread_create_info` 创建新内核线程
    * 如果为空，就继续睡眠
  * kernel/kthread.c
```cpp
static DEFINE_SPINLOCK(kthread_create_lock);
static LIST_HEAD(kthread_create_list);
struct task_struct *kthreadd_task;
...
int kthreadd(void *unused)
{
    struct task_struct *tsk = current;

    /* Setup a clean context for our children to inherit. */
    set_task_comm(tsk, "kthreadd"); //修改 task->comm 为 kthreadd
    ignore_signals(tsk);            //忽略所有 signals
    set_cpus_allowed_ptr(tsk, housekeeping_cpumask(HK_TYPE_KTHREAD)); //只允许在 housekeeping CPU 上被运行
    set_mems_allowed(node_states[N_MEMORY]); //只允许在有常规内存（包括 high 和 movetable）的 node 上运行

    current->flags |= PF_NOFREEZE;
    cgroup_init_kthreadd(); //kthreadd 被所有内核线程继承，将其保留在 root group 中，以便保证新的内核线程保留在 root group 中直到初始化完成
    //kthreadd 死循环，不退出的
    for (;;) {
        set_current_state(TASK_INTERRUPTIBLE); //进程状态为可中断的睡眠状态
        if (list_empty(&kthread_create_list))  //如果内核线程创建列表为空
            schedule();                        //kthreadd 调度出去，保持可中断睡眠状态
        __set_current_state(TASK_RUNNING);     //否则变成运行状态

        spin_lock(&kthread_create_lock);       //持有内核线程创建锁
        while (!list_empty(&kthread_create_list)) { //持锁后再次判断内核线程创建列表是否为空
            struct kthread_create_info *create;
            //取下内核线程创建列表上的内核线程创建信息的实例
            create = list_entry(kthread_create_list.next,
                        struct kthread_create_info, list);
            list_del_init(&create->list);      //将实例从内核线程创建列表上移除
            spin_unlock(&kthread_create_lock); //释放内核线程创建锁
            //调用内核线程创建接口函数 kernel_thread() 的特定封装函数
            create_kthread(create);

            spin_lock(&kthread_create_lock);  //再次持有内核线程创建锁
        }
        spin_unlock(&kthread_create_lock);    //释放内核线程创建锁
    }

    return 0;
}
```
* `kthreadd()` 调用内核线程创建接口函数 `kernel_thread()` 的特定封装函数 `create_kthread()` 创建新内核线程
  * kthreadd 创建时，传给 `kernel_thread()` 的入口函数是 `kthreadd()`
  * 而其他内核线程创建时，传给 `kernel_thread()` 的入口函数是 `kthread()`
    * 因此，其他内核线程的公用入口函数是 `kthread()`
```cpp
static void create_kthread(struct kthread_create_info *create)
{
    int pid;

#ifdef CONFIG_NUMA
    current->pref_node_fork = create->node;
#endif
    /* We want our own signal handler (we take no signals by default). */
    pid = kernel_thread(kthread, create, create->full_name,
                CLONE_FS | CLONE_FILES | SIGCHLD);
    if (pid < 0) { //如果创建失败，回收资源并唤醒等待内核线程创建的完成量
        /* Release the structure when caller killed by a fatal signal. */
        struct completion *done = xchg(&create->done, NULL);

        kfree(create->full_name);
        if (!done) {
            kfree(create);
            return;
        }
        create->result = ERR_PTR(pid);
        complete(done);
    }
}
```

### 内核线程的公用入口函数 `kthread()`
* `kthread()` 是内核线程的公用入口函数
  * kernel/kthread.c
```cpp
static int kthread(void *_create)
{
    static const struct sched_param param = { .sched_priority = 0 };
    /* Copy data: it's on kthread's stack */
    struct kthread_create_info *create = _create;
    int (*threadfn)(void *data) = create->threadfn; //最重要的，内核线程的入口函数
    void *data = create->data;
    struct completion *done;
    struct kthread *self;
    int ret;

    self = to_kthread(current);
    //如果内核线程已被致命 signal 杀死，释放内核线程创建信息结构的实例
    /* Release the structure when caller killed by a fatal signal. */
    done = xchg(&create->done, NULL);
    if (!done) {
        kfree(create->full_name);
        kfree(create);
        kthread_exit(-EINTR);
    }

    self->full_name = create->full_name;
    self->threadfn = threadfn;
    self->data = data;
    //新线程继承了 kthreadd 的优先级和 CPU 掩码。如果它们已更改，重置回默认值。
    /*
     * The new thread inherited kthreadd's priority and CPU mask. Reset
     * back to default in case they have been changed.
     */
    sched_setscheduler_nocheck(current, SCHED_NORMAL, &param);
    set_cpus_allowed_ptr(current, housekeeping_cpumask(HK_TYPE_KTHREAD));
    //将 result 域设置为 current，告诉用户我们已经生成了，正等待被停止或唤醒
    /* OK, tell user we're spawned, wait for stop or wakeup */
    __set_current_state(TASK_UNINTERRUPTIBLE);
    create->result = current;
    /*
     * Thread is going to call schedule(), do not preempt it,
     * or the creator may spend more time in wait_task_inactive().
     */
    preempt_disable();
    complete(done);   //唤醒等待内核线程创建的完成量
    schedule_preempt_disabled();
    preempt_enable();
    //如果内核线程没设置停止标志 KTHREAD_SHOULD_STOP，这就开始运行了
    ret = -EINTR;
    if (!test_bit(KTHREAD_SHOULD_STOP, &self->flags)) {
        cgroup_kthread_ready(); //该 kthread 完成了初始化。如果该 kthread 应该保留在 root group 中，那么创建者应该设置 PF_NO_SETAFFINITY
        __kthread_parkme(self); //设置了 KTHREAD_SHOULD_PARK 也会导致内核线程被调度出去
        ret = threadfn(data);   //调用内核线程的特定入口函数
    }
    kthread_exit(ret);
}
```

## Per CPU 热插拔线程
* 可以看看目前有哪些热插拔线程：
```cpp
1 drivers/powercap/idle_inject.c|420| <<idle_inject_init>> return smpboot_register_percpu_thread(&idle_inject_threads);
2 kernel/cpu.c|1208| <<cpuhp_threads_init>> BUG_ON(smpboot_register_percpu_thread(&cpuhp_threads));
3 kernel/irq_work.c|324| <<irq_work_init_threads>> BUG_ON(smpboot_register_percpu_thread(&irqwork_threads));
4 kernel/rcu/tree.c|2557| <<rcu_spawn_core_kthreads>> WARN_ONCE(smpboot_register_percpu_thread(&rcu_cpu_thread_spec),
5 kernel/softirq.c|970| <<spawn_ksoftirqd>> BUG_ON(smpboot_register_percpu_thread(&softirq_threads));
6 kernel/stop_machine.c|579| <<cpu_stop_init>> BUG_ON(smpboot_register_percpu_thread(&cpu_stop_threads));
```
* 对应名字是：
```cpp
1 idle_inject_threads.thread_comm = "idle_inject/%u"
2 cpuhp_threads.thread_comm       = "cpuhp/%u"
3 irqwork_threads.thread_comm     = "irq_work/%u"
4 rcu_cpu_thread_spec.thread_comm = "rcuc/%u"
5 softirq_threads.thread_comm     = "ksoftirqd/%u"
6 cpu_stop_threads.thread_comm    = "migration/%u"
```
* `smpboot_register_percpu_thread()` 接口函数用于注册 per-CPU 的热插拔线程，但并不调用这些线程的入口函数
* 其中的关键点是，`smpboot_thread_fn()` 是 per-CPU 内核线程的公用入口函数
```cpp
smpboot_register_percpu_thread()
   for_each_online_cpu(cpu)
   -> __smpboot_create_thread()
      -> tsk = kthread_create_on_cpu(smpboot_thread_fn, td, cpu, ht->thread_comm) //smpboot_thread_fn 是 per-CPU 内核线程的公用入口函数
         -> kthread_create_on_node(threadfn, data, cpu_to_node(cpu), namefmt, cpu)
            -> __kthread_create_on_node(threadfn, data, node, namefmt, args) //这个要展开讲
         -> kthread_bind(p, cpu) //因为是 Per-CPU 线程绑定 CPU
         *per_cpu_ptr(ht->store, cpu) = tsk; //这也是个很隐蔽的要点，ht->store 指向 &cpu_stopper.thread
      -> ht->create(cpu) //调用 struct smp_hotplug_thread 的 create 函数指针
```
* **注意**：`ht->store` 指向 `&cpu_stopper.thread`，所以 `*per_cpu_ptr(ht->store, cpu) = tsk` 让 `cpu_stopper.thread` 指向刚创建好的内核线程
  * 将来 `__cpu_stop_queue_work()` 唤醒 stopper 线程的时候，用的正是这个指针
* kernel/kthread.c
```cpp
struct task_struct *__kthread_create_on_node(int (*threadfn)(void *data),
                            void *data, int node,
                            const char namefmt[],
                            va_list args)
{
    DECLARE_COMPLETION_ONSTACK(done);
    struct task_struct *task;
    struct kthread_create_info *create = kmalloc(sizeof(*create),
                             GFP_KERNEL); //分配内核线程创建信息的实例

    if (!create)
        return ERR_PTR(-ENOMEM);
    create->threadfn = threadfn; //设置内核线程的特定入口函数
    create->data = data;         //设置内核线程的特定入口函数参数
    create->node = node;
    create->done = &done;
    create->full_name = kvasprintf(GFP_KERNEL, namefmt, args);
    if (!create->full_name) {
        task = ERR_PTR(-ENOMEM);
        goto free_create;
    }

    spin_lock(&kthread_create_lock);
    list_add_tail(&create->list, &kthread_create_list); //将信息添加到内核线程创建列表
    spin_unlock(&kthread_create_lock);

    wake_up_process(kthreadd_task); //唤醒 kthreadd 线程去创建内核线程
    /*
     * Wait for completion in killable state, for I might be chosen by
     * the OOM killer while kthreadd is trying to allocate memory for
     * new kernel thread.
     */
    if (unlikely(wait_for_completion_killable(&done))) {
        /*
         * If I was killed by a fatal signal before kthreadd (or new
         * kernel thread) calls complete(), leave the cleanup of this
         * structure to that thread.
         */
        if (xchg(&create->done, NULL))
            return ERR_PTR(-EINTR);
        /*
         * kthreadd (or new kernel thread) will call complete()
         * shortly.
         */
        wait_for_completion(&done); //等待 kthread() 结束完成量
    }
    task = create->result;
free_create:
    kfree(create);
    return task;  //返回创建好的内核线程
}
```
### per-CPU 内核线程的公用入口函数 `smpboot_thread_fn()`
* 在 Per CPU 热插拔线程这个场景中，内核线程的公用入口函数 `kthread()` 会调用 per-CPU 内核线程的公用入口函数 `smpboot_thread_fn()`
* 检查线程停止（stop）和停放（park）状况。为已注册的线程调用必要的设置、清理、停放和取消停放函数。
* 最重要的是，它会调用特定的内核线程入口函数 `ht->thread_fn(td->cpu)`
* kernel/smpboot.c
```cpp
/**
 * smpboot_thread_fn - percpu hotplug thread loop function
 * @data:   thread data pointer
 *
 * Checks for thread stop and park conditions. Calls the necessary
 * setup, cleanup, park and unpark functions for the registered
 * thread.
 *
 * Returns 1 when the thread should exit, 0 otherwise.
 */
static int smpboot_thread_fn(void *data)
{
    struct smpboot_thread_data *td = data;
    struct smp_hotplug_thread *ht = td->ht;
    //Per CPU 热插拔线程一般不退出
    while (1) {
        set_current_state(TASK_INTERRUPTIBLE);
        preempt_disable();
 ...
        if (!ht->thread_should_run(td->cpu)) {
            preempt_enable_no_resched();
            schedule();
        } else {
            __set_current_state(TASK_RUNNING);
            preempt_enable();
            ht->thread_fn(td->cpu);
        }
    }
}
```

## Stop Task 的创建
* `stop_cpu[s]()` 是简单化的 Per CPU 最高优先级 CPU 独占机制。
  * 调用者可以指定要在单个或多个 CPU 上执行的非睡眠函数，抢占所有其他进程并独占这些 CPU，直到其完成。
  * 此机制的资源是在 CPU 启动时预先分配的，只要目标 CPU online，就保证请求得到服务。
* 一个典型的 Per CPU 热插拔线程就是 CPU stop task 线程，也就是 `ps aux` 命令看到的 per CPU 的 `migration/*` 内核线程
* 首先是 `struct smp_hotplug_thread cpu_stop_threads` 全局变量，重点注意 `.thread_fn` 和 `.create` 两个域：
* kernel/stop_machine.c
```cpp
static struct smp_hotplug_thread cpu_stop_threads = {
    .store          = &cpu_stopper.thread,
    .thread_should_run  = cpu_stop_should_run,
    .thread_fn      = cpu_stopper_thread,
    .thread_comm        = "migration/%u",
    .create         = cpu_stop_create,
    .park           = cpu_stop_park,
    .selfparking        = true,
};
```
* 系统启动时通过 `early_initcall(cpu_stop_init)` 去注册 stop task 的初始化函数 `cpu_stop_init()`
* 可以看到它调用 `smpboot_register_percpu_thread()` 接口函数将 `cpu_stop_threads` 注册为 per-CPU 的热插拔线程
  * 注册的过程即向内核线程创建列表中添加内核线程信息的过程
  * 注册完成后唤醒 kthreadd 去创建 stop task 进程
  * kthreadd 创建好 stop task 进程后让它等待调度，而不是直接调用它的入口函数
* `cpu_stop_init()` 会给每个可能的 CPU 创建一个 stopper，而每个 stopper 有一个 works 链表，表示还有多少工作要做
```cpp
static DEFINE_PER_CPU(struct cpu_stopper, cpu_stopper);
static bool stop_machine_initialized = false;
...
static int __init cpu_stop_init(void)
{
    unsigned int cpu;
    //给每个可能的 CPU 创建一个 stopper
    for_each_possible_cpu(cpu) {
        struct cpu_stopper *stopper = &per_cpu(cpu_stopper, cpu);

        raw_spin_lock_init(&stopper->lock);
        INIT_LIST_HEAD(&stopper->works); //每个 stopper 有一个 works 链表
    }
    //注册为 per-CPU 的热插拔线程
    BUG_ON(smpboot_register_percpu_thread(&cpu_stop_threads));
    stop_machine_unpark(raw_smp_processor_id());
    stop_machine_initialized = true; //设置 stop task 初始化完成全局标志，stop_machine_cpuslocked() 会检查这一标志
    return 0;
}
early_initcall(cpu_stop_init);
```
* `struct smp_hotplug_thread cpu_stop_threads` 的 `.create` 函数指针指向 `cpu_stop_create()`
  * 它会调用 `sched_set_stop_task()` 将 stop task 的调度器类设置为优先级最高的 `stop_sched_class`
  * 并不是所有的 CPU 热插拔任务都是 stop class，而是 CPU stop task 任务这么设置的
```cpp
static void cpu_stop_create(unsigned int cpu)
{
    sched_set_stop_task(cpu, per_cpu(cpu_stopper.thread, cpu));
}
```
* `struct smp_hotplug_thread cpu_stop_threads` 的 `.thread_fn` 函数指针指向 `cpu_stopper_thread()`
* 通过 `kthread() -> smpboot_thread_fn() -> ht->thread_fn(td->cpu)` 调用 `cpu_stopper_thread()` 回调
  * kernel/stop_machine.c
```cpp
static void cpu_stopper_thread(unsigned int cpu)
{
    struct cpu_stopper *stopper = &per_cpu(cpu_stopper, cpu);
    struct cpu_stop_work *work;

repeat:
    work = NULL;
    raw_spin_lock_irq(&stopper->lock);
    if (!list_empty(&stopper->works)) {          //如果还有工作要做
        work = list_first_entry(&stopper->works,
                    struct cpu_stop_work, list); //取出第一项工作
        list_del_init(&work->list);              //将工作从链表上删除
    }
    raw_spin_unlock_irq(&stopper->lock);
    //如果有工作要做，在本函数处理；如果没工作要做，函数返回，调用者 smpboot_thread_fn() 还有一个循环，stopper 线程不会退出
    if (work) {
        cpu_stop_fn_t fn = work->fn; //CPU stop 的工作回调
        void *arg = work->arg;
        struct cpu_stop_done *done = work->done;
        int ret;
        //CPU stop 回调一定不能睡眠，让 in_atomic() == true
        /* cpu stop callbacks must not sleep, make in_atomic() == T */
        stopper->caller = work->caller;
        stopper->fn = fn;
        preempt_count_inc(); //增加抢占计数
        ret = fn(arg);       //调用 CPU stop 的工作回调
        if (done) {
            if (ret)
                done->ret = ret;
            cpu_stop_signal_done(done); //如果要做的工作都完成了，结束 stop 的完成量
        }
        preempt_count_dec(); //减小抢占计数
        stopper->fn = NULL;
        stopper->caller = 0;
        WARN_ONCE(preempt_count(),
              "cpu_stop: %ps(%p) leaked preempt count\n", fn, arg); //是否有抢占计数泄露？
        goto repeat; //重复这一过程，处理下一项工作
    }
}
```
* **注意**：CPU stop 回调一定不能睡眠，它处于原子上下文

### 给 Stop Task 添加工作
* 给 Stop Task 添加工作接口目前有以下几个：
  * `stop_machine()`，最常用的，我们就看这个
  * `stop_machine_cpuslocked()`，如果已持有 `cpus_read_lock()`
  * `stop_core_cpuslocked()`
  * `stop_machine_from_inactive_cpu()`
* `stop_machine()` 是对 `stop_machine_cpuslocked()` 的封装，它会持有和释放 `cpus_read_lock()`
```cpp
int stop_machine_cpuslocked(cpu_stop_fn_t fn, void *data,
                const struct cpumask *cpus)
{   //构造一个本地 multi_stop_data 数据结构的实例
    struct multi_stop_data msdata = {
        .fn = fn,
        .data = data,
        .num_threads = num_online_cpus(),
        .active_cpus = cpus,
    };

    lockdep_assert_cpus_held();
    //cpu_stop_init() 会将此全局标志置为 true，如果为 false 说明 stop task 还未初始化完成
    if (!stop_machine_initialized) {
        /*
         * Handle the case where stop_machine() is called
         * early in boot before stop_machine() has been
         * initialized.
         */
        unsigned long flags;
        int ret;
        //处理 stop_machine() 早于 CPU stop task 初始化完成的情况
        WARN_ON_ONCE(msdata.num_threads != 1);
        //其实就是直接调用提供给的回调函数了
        local_irq_save(flags);
        hard_irq_disable();
        ret = (*fn)(data);
        local_irq_restore(flags);
        //后面就直接返回了，不往 CPU stop task 里排工作了，因为没初始化好
        return ret;
    }
    //将状态置为初始状态然后在所有在线 CPU 上执行 stop task
    /* Set the initial state and stop all online cpus. */
    set_state(&msdata, MULTI_STOP_PREPARE);
    return stop_cpus(cpu_online_mask, multi_cpu_stop, &msdata);
}

int stop_machine(cpu_stop_fn_t fn, void *data, const struct cpumask *cpus)
{
    int ret;

    /* No CPUs can come up or down during this. */
    cpus_read_lock();
    ret = stop_machine_cpuslocked(fn, data, cpus);
    cpus_read_unlock();
    return ret;
}
```
* 注意 `multi_cpu_stop()` 这个函数，后面要展开讲的
* 从 `stop_machine()` 的调用路径上来看，它要做的事情并不复杂，就是
  1. 把要调用的函数 `fn` 及其参数 `data` 包装成工作 `work` 加入到 per CPU stop task 的工作列表上
  2. 唤醒 per CPU stop task 或者说 stopper 去处理工作
```cpp
stop_machine(cpu_stop_fn_t fn, void *data, const struct cpumask *cpus)
-> stop_machine_cpuslocked(fn, data, cpus)
      struct multi_stop_data msdata = {
          .fn = fn,
          .data = data,
          .num_threads = num_online_cpus(),
          .active_cpus = cpus,
      };
      if (!stop_machine_initialized) //如果 stop task 未初始化好
      -> (*fn)(data);                //直接调用回调后返回
   -> set_state(&msdata, MULTI_STOP_PREPARE)
   -> stop_cpus(cpu_online_mask, multi_cpu_stop, &msdata)
      -> __stop_cpus(cpumask, fn, arg)
         -> cpu_stop_init_done(&done, cpumask_weight(cpumask)) //初始化完成量
         -> queue_stop_cpus_work(cpumask, fn, arg, &done)
               for_each_cpu(cpu, cpumask) {
                  work = &per_cpu(cpu_stopper.stop_work, cpu);
                  work->fn = fn;
                  work->arg = arg;
                  work->done = done;
                  work->caller = _RET_IP_;
               -> cpu_stop_queue_work(cpu, work)
                     struct cpu_stopper *stopper = &per_cpu(cpu_stopper, cpu);
                     DEFINE_WAKE_Q(wakeq);                          //创建唤醒队列
                  -> __cpu_stop_queue_work(stopper, work, &wakeq)
                     -> list_add_tail(&work->list, &stopper->works) //把工作加入 stopper 工作列表
                     -> wake_q_add(wakeq, stopper->thread)          //把 CPU stopper 进程加入唤醒队列
                  -> wake_up_q(&wakeq);                             //唤醒队列上的进程
               }
         -> wait_for_completion(&done.completion) //等待 stop task 执行完成
```
* `msdata` 是 `stop_machine_cpuslocked()` 栈上的一个本地变量，因此它的生命周期随着它代表的 stop task 执行结束而消亡
  * 它更像是一个 upvalue
  * 这里设计了一个简单的状态迁移，每个 stop task 通过在各 CPU 之间通过状态来同步进度

#### 多 CPU stop 函数 `multi_cpu_stop()`
* 通过 `cpu_stop_threads.thread_comm = "migration/%u"` 这个内核线程调用 stop machine task
```cpp
ret_from_fork_asm -> ret_from_fork()
-> kthread()
   -> threadfn(data)
   => smpboot_thread_fn()
      while (1) {
      -> __set_current_state(TASK_RUNNING);
      -> preempt_enable()
      -> ht->thread_fn(td->cpu) //前面是 CPU 热插拔内核线程的公用路径
      => cpu_stopper_thread()   //遍历 works 链表，逐个处理 work
            cpu_stop_fn_t fn = work->fn;
         -> fn(arg);
         => multi_cpu_stop()
      }
```
* 函数 `multi_cpu_stop()` 是保证多 CPU 同步执行的关键，首先涉及一个状态机
  * 每个状态代表生命周期中所处的不同阶段，除了 `MULTI_STOP_PREPARE` 比较特殊，其实涵盖了在它之后的几个状态
```cpp
/* This controls the threads on each CPU. */
enum multi_stop_state {
    /* Dummy starting state for thread. */   //线程的初始状态
    MULTI_STOP_NONE,
    /* Awaiting everyone to be scheduled. */ //等待线程在各个 CPU 上都被调度过
    MULTI_STOP_PREPARE,
    /* Disable interrupts. */                //禁用中断
    MULTI_STOP_DISABLE_IRQ,
    /* Run the function */                   //运行回调函数
    MULTI_STOP_RUN,
    /* Exit */                               //stop 任务退出
    MULTI_STOP_EXIT,
};

struct multi_stop_data {
    cpu_stop_fn_t       fn;
    void            *data;
    /* Like num_online_cpus(), but hotplug cpu uses us, so we need this. */
    unsigned int        num_threads;
    const struct cpumask    *active_cpus;

    enum multi_stop_state   state;           //记录线程所处的阶段
    atomic_t        thread_ack;
};
```
* 详细看看 `multi_cpu_stop()` 前了解一下 `ack_state()`
  * 它会在每个 CPU 上被调用
  * 每次它会递减响应 stop task 的计数
  * 当计数减为 `0` 时，它会 **递增状态机的状态**，从而推动整个 stop 事件的前进
* `multi_cpu_stop()` 则把控整个状态和 stop work 的调用
  * kernel/stop_machine.c
```cpp
/* Last one to ack a state moves to the next state. */
static void ack_state(struct multi_stop_data *msdata)
{
    if (atomic_dec_and_test(&msdata->thread_ack))
        set_state(msdata, msdata->state + 1);
}

/* This is the cpu_stop function which stops the CPU. */
static int multi_cpu_stop(void *data)
{
    struct multi_stop_data *msdata = data; //stop_machine_cpuslocked() 栈上的一个本地变量
    enum multi_stop_state newstate, curstate = MULTI_STOP_NONE; //本地状态初始为 NONE
    int cpu = smp_processor_id(), err = 0;
    const struct cpumask *cpumask;
    unsigned long flags;
    bool is_active;
    //当从 stop_machine_from_inactive_cpu() 调用时，irq 可能已被禁用。 保存状态并在退出时恢复它
    /*
     * When called from stop_machine_from_inactive_cpu(), irq might
     * already be disabled.  Save the state and restore it on exit.
     */
    local_save_flags(flags);

    if (!msdata->active_cpus) {
        cpumask = cpu_online_mask;
        is_active = cpu == cpumask_first(cpumask);
    } else {
        cpumask = msdata->active_cpus;
        is_active = cpumask_test_cpu(cpu, cpumask);
    }

    /* Simple state machine */
    do {
        /* Chill out and ensure we re-read multi_stop_state. */
        stop_machine_yield(cpumask);
        newstate = READ_ONCE(msdata->state); //读取全局的状态
        if (newstate != curstate) {          //本地状态与全局状态不一致
            curstate = newstate;             //更新本地状态为全局状态
            switch (curstate) {
            case MULTI_STOP_DISABLE_IRQ:     //所有的 CPU 通过走到这里关闭硬中断
                local_irq_disable();
                hard_irq_disable();
                break;
            case MULTI_STOP_RUN:             //所有的 CPU 通过走到这里执行工作的回调函数
                if (is_active)
                    err = msdata->fn(msdata->data);
                break;
            default:
                break;
            }
            ack_state(msdata);               //如果所有 CPU 都到达同一个状态，则推进全局状态到下一个阶段
        } else if (curstate > MULTI_STOP_PREPARE) { //如果本地状态与全局状态一致且经过准备阶段
            /*
             * At this stage all other CPUs we depend on must spin
             * in the same loop. Any reason for hard-lockup should
             * be detected and reported on their side.
             */
            touch_nmi_watchdog(); //喂 NMI watchdog，防止出现 hard lockup
        }
        rcu_momentary_dyntick_idle();
    } while (curstate != MULTI_STOP_EXIT);   //所有的 CPU 都执行完工作回调，退出循环

    local_irq_restore(flags);
    return err;
}
```

## 设置 MTRR 的例子
* 通过以下路径把 cache 汇合处理程序 `cache_rendezvous_handler()` 给 stop task 安排上了
```cpp
smp_init()
-> smp_cpus_done(setup_max_cpus)
   -> smp_ops.smp_cpus_done(max_cpus)
   => native_smp_cpus_done()
      -> cache_aps_init()
         -> stop_machine(cache_rendezvous_handler, NULL, cpu_online_mask)
            -> stop_machine_cpuslocked()
                  if (!stop_machine_initialized) {
                  -> (*fn)(data);
                  => cache_rendezvous_handler()
                  }
               -> stop_cpus(cpu_online_mask, multi_cpu_stop, &msdata)
```
* Per CPU 的 `migration/%u` stop task 被唤醒后会调用 `cache_rendezvous_handler()`
```cpp
ret_from_fork_asm -> ret_from_fork()
-> kthread()
   -> threadfn(data)
   => smpboot_thread_fn()
      while (1) {
      -> ht->thread_fn(td->cpu) //前面是 CPU 热插拔内核线程的公用路径
      => cpu_stopper_thread()   //遍历 works 链表，逐个处理 work
            cpu_stop_fn_t fn = work->fn;
         -> fn(arg);
         => multi_cpu_stop()
            case MULTI_STOP_RUN:
            -> msdata->fn(msdata->data);
            => cache_rendezvous_handler()
               -> cache_cpu_init()
                  -> cache_disable()
                     if (memory_caching_control & CACHE_MTRR)
                     -> mtrr_generic_set_state();
                     if (memory_caching_control & CACHE_PAT)
                     -> pat_cpu_init();
                  -> cache_enable();
      }
```