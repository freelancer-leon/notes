# ftrace 的实现

![https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/image001.jpg](pic/ftrace_fw.jpg)

# Function tracer的实现
* Ftrace 采用 GCC 的 profile 特性在所有内核函数的开始部分加入一段 stub 代码，ftrace 重载这段代码来实现 trace 功能。
* gcc 的 `-pg` 选项将在每个函数入口处加入对 `mcount` 的调用代码。
* 增加 `-pg` 选项后，gcc 在函数的入口处加入了对 `mcount` 的调用：`call _mcount`。
* 原本 `mcount` 由 libc 实现，但您知道内核不会连接 libc 库，因此 ftrace 编写了自己的 `mcount` stub 函数，并借此实现 trace 功能。
* 在每个内核函数入口加入 trace 代码，必然会影响内核的性能，为了减小对内核性能的影响，ftrace 支持动态 trace 功能。
* 当未 `CONFIG_DYNAMIC_FTRACE` 选中时：
	* 每个函数调用都会根据不同的体系结构的实现调用`mcount`函数，
	* 如果 ftrace 使能了某些跟踪器，`ftrace_trace_function`指针不再指向`ftrace_stub`，而是指向具体的跟踪函数。
	* 否则就执行到体系结构相关的`ftrace_stub`从函数返回。
* 当 `CONFIG_DYNAMIC_FTRACE` 被选中后：
	* 内核编译时会调用一个 perl 脚本：`recordmcount.pl` 将每个函数的地址写入一个特殊的段：`__mcount_loc`
	* 在内核初始化的初期，ftrace 查询 `__mcount_loc` 段，得到每个函数的入口地址，并将 `mcount` 替换为 `nop` 指令。这样在默认情况下，ftrace 不会对内核性能产生影响。
	* 当用户打开 ftrace 功能时，ftrace 将这些 `nop` 指令动态替换为 `ftrace_caller`，该函数将调用用户注册的 trace 函数。

## ftrace 依赖的`-pg`编译器选项
* Makefile
	```makefile
	ifdef CONFIG_FUNCTION_TRACER
	ifndef CC_FLAGS_FTRACE
	CC_FLAGS_FTRACE := -pg
	endif
	export CC_FLAGS_FTRACE
	...
	KBUILD_CFLAGS	+= $(CC_FLAGS_FTRACE) ...
	```

* 对于tracer自身而言，是不需要`-pg`选项的，因此我们可以看到在`kernel/tracing/Makefile`中将`-pg`从该模块的`CFLAGS`中剔除了。
* kernel/tracing/Makefile
	```makefile
	# Do not instrument the tracer itself:

	ifdef CONFIG_FUNCTION_TRACER
	ORIG_CFLAGS := $(KBUILD_CFLAGS)
	KBUILD_CFLAGS = $(subst $(CC_FLAGS_FTRACE),,$(ORIG_CFLAGS))

	ifdef CONFIG_FTRACE_SELFTEST
	# selftest needs instrumentation 如自测功能开启单独给 trace_selftest_dynamic.c 文件插桩。
	CFLAGS_trace_selftest_dynamic.o = $(CC_FLAGS_FTRACE)
	obj-y += trace_selftest_dynamic.o
	endif
	endif

	```
* 对于不希望被 ftrace 跟踪的模块也可以如法炮制。

## ftrace跟踪函数

* `ftrace_trace_function`是个 *函数指针*，`call mcount`时会调用该指针指向的函数。
	* 函数原型
	* include/linux/ftrace.h
		```c
		typedef void (*ftrace_func_t)(unsigned long ip, unsigned long parent_ip,
		                              struct ftrace_ops *op, struct pt_regs *regs);
		```
	* 定义和初始值
	* kernel/trace/ftrace.c
		```c
		ftrace_func_t ftrace_trace_function __read_mostly = ftrace_stub;
		```
* `ftrace_stub`则是一个与体系结构相关 *函数*，通常是用汇编实现的一条简单的 **从函数返回** 的指令：
	* arch/powerpc/kernel/entry_64.S: `blr`
	* arch/powerpc/kernel/entry_32.S: `blr`
	* arch/arm/kernel/entry-ftrace.S: `ret     lr`
	* arch/arm64/kernel/entry-ftrace.S: `ret`
	* arch/x86/entry/entry_32.S: `ret`
	* arch/x86/kernel/mcount_64.S: `retq`

## ftrace函数跟踪器的 x86 64 实现
* 在该文件里，会用`function_hook`来替换`mcount`。
	* arch/x86/kernel/mcount_64.S
		```c
		#ifdef CC_USING_FENTRY
		# define function_hook  __fentry__
		EXPORT_SYMBOL(__fentry__)
		#else
		# define function_hook  mcount
		EXPORT_SYMBOL(mcount) /*记得为 modules 导出符号*/
		#endif
		```

### `CONFIG_DYNAMIC_FTRACE`未开启
* 对于`CONFIG_DYNAMIC_FTRACE`没有开启的情况，显然，在关闭 ftrace 的时候也会有性能损失。
* 具体表现在，无论是否开启 ftrace 调用函数时都会先调用 `mcount`，总是至少会多执行两条指令，即使`ftrace_trace_function`没有被指向某个跟踪函数。

* arch/x86/kernel/mcount_64.S
```nasm
ENTRY(function_hook)
        cmpq $ftrace_stub, ftrace_trace_function ; cmpq: compare quad words，比较 ftrace_stub 和指针 ftrace_trace_function
        jnz trace ; 如果上面比较结果不为 0，即不相等，跳转到标签 trace，开始跟踪；否则往下执行

fgraph_trace:
#ifdef CONFIG_FUNCTION_GRAPH_TRACER
        cmpq $ftrace_stub, ftrace_graph_return ; 接着比较，可见如果开启了CONFIG_FUNCTION_GRAPH_TRACER，即使关闭 ftrace，也会有此比较
        jnz ftrace_graph_caller ; 如果上面比较结果不为 0，即不相等，跳转到标签 ftrace_graph_caller，开始跟踪；否则往下执行

        cmpq $ftrace_graph_entry_stub, ftrace_graph_entry ; 含义和之前类似
        jnz ftrace_graph_caller
#endif

GLOBAL(ftrace_stub)
        retq ; 从 mcount 函数返回

trace:
        /* save_mcount_regs fills in first two parameters */
        save_mcount_regs ; 一个汇编宏，目的是填入前两个参数：ip 和 parent_ip

        /*
         * When DYNAMIC_FTRACE is not defined, ARCH_SUPPORTS_FTRACE_OPS is not
         * set (see include/asm/ftrace.h and include/linux/ftrace.h).  Only the
         * ip and parent ip are used and the list function is called when
         * function tracing is enabled.
         */
        call   *ftrace_trace_function ; 调用 ftrace_trace_function 指向的 C 实现的函数

        restore_mcount_regs ; 另一个汇编宏，恢复一些寄存器的值

        jmp fgraph_trace ; 跳转到标签 fgraph_trace 处，并执行至 retq 返回调用者
END(function_hook)
```

### 开启`CONFIG_DYNAMIC_FTRACE`
* 最初`mcount`的作用是直接从函数返回。
* arch/x86/kernel/mcount_64.S
```nasm
ENTRY(function_hook)
        retq
END(function_hook)
```

# function_graph tracer
```c
foo()
  |
  +-> bar()
        |
        +-> mcount()
        |     |
        |     +-> ftrace_trace_function() /*function hook*/
        |     |
        |     +-> if (ftrace_graph_return != ftrace_stub ||
        |             ftrace_graph_entry != ftrace_graph_entry_stub)
        |             |
        |             +-> ftrace_graph_caller()
        |                   |
        |                   +-> [ Save the return address of traced function ]
        |                   +-> prepare_ftrace_return()
        |                         |
        |                         +-> [ hijack the return address ]
        |                         +-> ftrace_graph_entry() /*function_graph hook 1*/
        |                         +-> ftrace_push_return_trace()
        |
        +-> [ bar() function body ]
        |   /*bar() will return to the hijack address, but not foo()*/
        +-> return_to_handler()
              |
              +-> ftrace_return_to_handler() /*return the original address*/
              |     +-> ftrace_pop_return_trace()
              |     +-> ftrace_graph_return() /*function_graph hook 2*/
              |   /*jump to the original call site*/
              +-> jmp original_return_point
                         |
  +<---------------------+
  | /*back to original flow*/
  v
;
```
* function_graph 跟踪器要复杂一些，在 function 跟踪器调用完 `ftrace_trace_function` 指向的函数后执行，即`jmp fgraph_trace`
* 这里会比较`ftrace_graph_return`和`ftrace_stub`，`ftrace_graph_entry`和`ftrace_graph_entry_stub`
* 上面两个比较一旦有不相等，即`ftrace_graph_return`或`ftrace_graph_entry`被设置，都会跳到`ftrace_graph_caller`
* `ftrace_stub`之前列过了，通常为体系结构相关的汇编实现，而`ftrace_graph_entry_stub` 是通用的 C 实现。
* kernel/trace/ftrace.c
	```c
	int ftrace_graph_entry_stub(struct ftrace_graph_ent *trace)
	{
	        return 0;
	}
	```
* 接下来看 *`ftrace_graph_caller`*
* arch/x86/kernel/mcount_64.S
```nasm
ENTRY(ftrace_graph_caller)
        /* Saves rbp into %rdx and fills first parameter  */
        save_mcount_regs

#ifdef CC_USING_FENTRY
        leaq MCOUNT_REG_SIZE+8(%rsp), %rsi
        movq $0, %rdx   /* No framepointers needed */
#else
        /* Save address of the return address of traced function */
        leaq 8(%rdx), %rsi     /*保存被跟踪的函数的返回地址到栈上*/
        /* ftrace does sanity checks against frame pointers */
        movq (%rdx), %rdx
#endif
        call    prepare_ftrace_return

        restore_mcount_regs

        retq
END(ftrace_graph_caller)
```
* 与`ftrace_trace_function`指针稍有不同，传给`prepare_ftrace_return()`函数的`parent_ip`会是一个在栈上的指针。
* 这么做的目的是让函数可以临时劫持返回地址，让它指向体系结构相关的`return_to_handler`函数。
* `return_to_handler`函数简单地调用通用的 C 函数`ftrace_return_to_handler()`返回最初的返回地址，这样我们就可以返回最初的调用地点了。
* arch/x86/kernel/mcount_64.S
```nasm
GLOBAL(return_to_handler)
        subq  $24, %rsp

        /* Save the return values */
        movq %rax, (%rsp)
        movq %rdx, 8(%rsp)
        movq %rbp, %rdi

        call ftrace_return_to_handler

        movq %rax, %rdi
        movq 8(%rsp), %rdx
        movq (%rsp), %rax
        addq $24, %rsp
        jmp *%rdi
```

# irqsoff/preemptoff/preemptirqsoff tracer

* **Irqsoff tracer** 当中断被禁止时，系统无法相应外部事件，比如键盘和鼠标，时钟也无法产生 tick 中断。这意味着系统响应延迟，`irqsoff` 这个 tracer 能够跟踪并记录内核中哪些函数禁止了中断，对于其中中断禁止时间最长的，`irqsoff` 将在 log 文件的第一行标示出来，从而使开发人员可以迅速定位造成响应延迟的罪魁祸首。
* **Preemptoff tracer** 和前一个 tracer 类似，`preemptoff` tracer 跟踪并记录 **禁止内核抢且关闭中断占期间** 的函数，并清晰地显示出禁止抢占时间最长的内核函数。
* **Preemptirqsoff tracer** 同上，跟踪和记录禁止中断或者禁止抢占的内核函数，以及禁止时间最长的函数。

### `preemptoff`与`irqsoff`跟踪器
* `preemptoff`与`irqsoff`跟踪器用的跟踪函数是相同的，都是`irqsoff_tracer_call()`。
* `preemptoff`与`irqsoff`跟踪器的不同之处
	* `irqsoff`跟踪器的 start 点在开启或关闭中断的地方，如 `local_irq_disable()`；
	* `preemptoff`跟踪器的 start 点在开启或关闭抢占的地方，如 `preempt_disable()`。
	* 老版本的`preemptoff`跟踪器不会记录 **中断开启期间的函数，尽管抢占计数不为 0**；目前版本（4.9）的`preemptoff`跟踪器不会记录 **中断开启且抢占计数为 0 时的函数**（好像是废话）。

#### `preemptoff`跟踪器的开启和关闭
* 跟踪器触发时函数调用简化流程
```
include/linux/preempt.h
preempt_disable()
  -> preempt_count_inc() -> preempt_count_add(1)
		-> preempt_latency_start()
			kernel/trace/trace_irqsoff.c
			-> trace_preempt_off()
				-> start_critical_timing()
					-> __trace_function()
```
* kernel/sched/core.c
```c
/*
 * If the value passed in is equal to the current preempt count
 * then we just disabled preemption. Start timing the latency.
 */
static inline void preempt_latency_start(int val)
{
        /*仅在抢占计数与要改变的递增值相等时才调用函数去动态改变插桩，不相等时是不需要去改变
          插桩的。我们需要记录的是抢占关闭到抢占开启期间的 trace，在此期间增加禁止抢占的深度
          并不改变不能被抢占的状况。*/
        if (preempt_count() == val) {
                unsigned long ip = get_lock_parent_ip();
...
                trace_preempt_off(CALLER_ADDR0, ip);
        }    
}

void preempt_count_add(int val)
{
...
        __preempt_count_add(val);
...
        preempt_latency_start(val);
}
EXPORT_SYMBOL(preempt_count_add);
NOKPROBE_SYMBOL(preempt_count_add);

/*
 * If the value passed in equals to the current preempt count
 * then we just enabled preemption. Stop timing the latency.
 */
static inline void preempt_latency_stop(int val)
{
        /*仅在抢占计数与要改变的递减值相等时才调用函数去动态改变插桩，不相等时是不需要去改变
          插桩的。我们需要记录的是抢占关闭到抢占开启期间的 trace，在此期间减少禁止抢占的深度
          并不改变不能被抢占的状况。*/
        if (preempt_count() == val)
                trace_preempt_on(CALLER_ADDR0, get_lock_parent_ip());
}

void preempt_count_sub(int val)
{
...
        preempt_latency_stop(val);
        __preempt_count_sub(val);
}
EXPORT_SYMBOL(preempt_count_sub);
NOKPROBE_SYMBOL(preempt_count_sub);
...
```
* kernel/trace/trace_irqsoff.c
```c
static DEFINE_PER_CPU(int, tracing_cpu);
...
static inline int
preempt_trace(void)
{       /*preemptoff跟踪器开启且抢占计数不为 0*/
        return ((trace_type & TRACER_PREEMPT_OFF) && preempt_count());
}
...
static inline int
irq_trace(void)
{       /*irqsoff跟踪器开启且在中断上下文*/
        return ((trace_type & TRACER_IRQS_OFF) &&
                irqs_disabled());
}
...
static inline void
start_critical_timing(unsigned long ip, unsigned long parent_ip)
{
        int cpu;
        struct trace_array *tr = irqsoff_trace;
        struct trace_array_cpu *data;
        unsigned long flags;
        /*如果当前跟踪器未使能或跟踪功能没开启，直接返回*/
        if (!tracer_enabled || !tracing_is_enabled())
                return;

        cpu = raw_smp_processor_id();
        /*tracing_cpu是个 Per-CPU 变量，它仅在该函数下面置 1，也就是说该函数下面的部分
          只在开启跟踪的时候运行一次。*/
        if (per_cpu(tracing_cpu, cpu))
                return;

        data = per_cpu_ptr(tr->trace_buffer.data, cpu);
        /*data->disabled 控制是否同时 trace_function*/
        if (unlikely(!data) || atomic_read(&data->disabled))
                return;
        /*不允许伪并发的 trace_function，注意，这是 Per-CPU 的*/
        atomic_inc(&data->disabled);

        data->critical_sequence = max_sequence;
        data->preempt_timestamp = ftrace_now(cpu);
        data->critical_start = parent_ip ? : ip;
        /*将 flags 寄存器的信息取出来*/
        local_save_flags(flags);
        /*写当前跟踪信息*/
        __trace_function(tr, ip, parent_ip, flags, preempt_count());
        /*此处将 tracing_cpu 置 1，不会因为伪并发而被运行两次start*/
        per_cpu(tracing_cpu, cpu) = 1;
        /*写完信息了，放开保护允许继续 trace*/
        atomic_dec(&data->disabled);
}
...
#ifdef CONFIG_PREEMPT_TRACER
void trace_preempt_on(unsigned long a0, unsigned long a1)
{
        if (preempt_trace() && !irq_trace())
                stop_critical_timing(a0, a1);
}

void trace_preempt_off(unsigned long a0, unsigned long a1)
{
        if (preempt_trace() && !irq_trace())
                start_critical_timing(a0, a1);
}
#endif /* CONFIG_PREEMPT_TRACER */
```
* include/linux/ftrace.h
```c
/* Archs may use other ways for ADDR1 and beyond */
#ifndef ftrace_return_address
# ifdef CONFIG_FRAME_POINTER
#  define ftrace_return_address(n) __builtin_return_address(n)
# else
#  define ftrace_return_address(n) 0UL
# endif
#endif

#define CALLER_ADDR0 ((unsigned long)ftrace_return_address0)
#define CALLER_ADDR1 ((unsigned long)ftrace_return_address(1))
#define CALLER_ADDR2 ((unsigned long)ftrace_return_address(2))
#define CALLER_ADDR3 ((unsigned long)ftrace_return_address(3))
#define CALLER_ADDR4 ((unsigned long)ftrace_return_address(4))
#define CALLER_ADDR5 ((unsigned long)ftrace_return_address(5))
#define CALLER_ADDR6 ((unsigned long)ftrace_return_address(6))

static inline unsigned long get_lock_parent_ip(void)
{
        unsigned long addr = CALLER_ADDR0;

        if (!in_lock_functions(addr)) /*如果地址不在锁函数的.text段范围内*/
                return addr;          /*返回该地址*/
        addr = CALLER_ADDR1;          /*否则向上走一层*/
        if (!in_lock_functions(addr))
                return addr;
        return CALLER_ADDR2;
}
...__```
```

#### `irqsoff`跟踪器的开启和关闭
* 还是之前提到的，与`preemptoff`最大的不同之处就是起始点：
* include/linux/irqflags.h
```c
#define local_irq_enable() \
        do { trace_hardirqs_on(); raw_local_irq_enable(); } while (0)
#define local_irq_disable() \
        do { raw_local_irq_disable(); trace_hardirqs_off(); } while (0)
```

* kernel/trace/trace_irqsoff.c
```c
/*
 * We are only interested in hardirq on/off events:
 */
void trace_hardirqs_on(void)
{
        if (!preempt_trace() && irq_trace())
                stop_critical_timing(CALLER_ADDR0, CALLER_ADDR1);
}
EXPORT_SYMBOL(trace_hardirqs_on);

void trace_hardirqs_off(void)
{
        if (!preempt_trace() && irq_trace())
                start_critical_timing(CALLER_ADDR0, CALLER_ADDR1);
}
EXPORT_SYMBOL(trace_hardirqs_off);
```

# Reference

* [ftrace 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/)
* [Ftrace 实现原理与开发实践](http://tinylab.org/ftrace-principle-and-practice/)
