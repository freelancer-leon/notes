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
```assembly
ENTRY(function_hook)
        retq
END(function_hook)
```

# function_graph tracer的实现
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
	* `ftrace_graph_entry`是 function_graph 跟踪器的第一个 hook 点，在被调用函数体执行前被调用
	* `ftrace_graph_return`是 function_graph 跟踪器的第二个 hook 点，在被调用函数返回前会被调用
* `ftrace_stub`之前列过了，通常为体系结构相关的汇编实现，而`ftrace_graph_entry_stub` 是通用的 C 实现
* kernel/trace/ftrace.c
	```c
	int ftrace_graph_entry_stub(struct ftrace_graph_ent *trace)
	{
	        return 0;
	}
	```

### ftrace_graph_caller
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
        leaq 8(%rdx), %rsi     ;保存被跟踪的函数的返回地址
        /* ftrace does sanity checks against frame pointers */
        movq (%rdx), %rdx
#endif
        call    prepare_ftrace_return

        restore_mcount_regs

        retq
END(ftrace_graph_caller)
```
* 与`ftrace_trace_function`指针稍有不同，传给`prepare_ftrace_return()`函数的`parent_ip`会是一个在栈上的指针，指向（`foo()`调用`bar()`）调用点的地址。
* 这么做的目的是让函数可以临时劫持返回地址，让它指向体系结构相关的`return_to_handler`函数。
	* 比如`foo()`调用`bar()`，那么劫持之后`bar()`函数运行完后不是返回到`foo()`，而是返回到`return_to_handler`。
	* 如果像`ftrace_trace_function`那样，将调用点（`foo()`代码段上）的地址传给`prepare_ftrace_return()`函数，是无法修改的。
	* 如果传的是栈上某个指针的地址，那么`prepare_ftrace_return()`就可以通过该地址修改指针的指向，让它指向`return_to_handler`，这样就 **劫持** 了被调用函数的返回。

### prepare_ftrace_return()
* `prepare_ftrace_return()`是 function_graph 跟踪器需要实现的一个和体系结构相关的函数，要做的事情主要有
	* 将栈上指针原来保存的被调用者的返回地址替换成`return_to_handler`的地址，即劫持
	* 调用 function_graph 的 hook 函数 `ftrace_graph_entry()`
	* 调用 function_graph 的体系结构无关的 `ftrace_push_return_trace()`函数
* arch/x86/kernel/ftrace.c
```c
void prepare_ftrace_return(unsigned long self_addr, unsigned long *parent,
                           unsigned long frame_pointer)
{
        unsigned long old;
        int faulted;
        struct ftrace_graph_ent trace;
        unsigned long return_hooker = (unsigned long)
                                &return_to_handler;
        ...
        /*parent是栈上指针的地址，而不是指针的内容（虽然内容也是地址）。
          该指针的指向是被调用者的返回地址，应为代码段的某个地址，即原返回地址。*/
        asm volatile(
                "1: " _ASM_MOV " (%[parent]), %[old]\n" /*将原返回地址保存到old*/
                "2: " _ASM_MOV " %[return_hooker], (%[parent])\n" /*指针指向return_to_handler*/
                "   movl $0, %[faulted]\n"
                "3:\n"

                ".section .fixup, \"ax\"\n"
                "4: movl $1, %[faulted]\n"
                "   jmp 3b\n"
                ".previous\n"

                _ASM_EXTABLE(1b, 4b)
                _ASM_EXTABLE(2b, 4b)

                : [old] "=&r" (old), [faulted] "=r" (faulted)
                : [parent] "r" (parent), [return_hooker] "r" (return_hooker)
                : "memory"
        );
        if (unlikely(faulted)) {
                ftrace_graph_stop(); /*如果以上过程出现问题，停止ftrace_graph跟踪*/
                WARN_ON(1);
                return;
        }
        /*保存被调用者地址和当前调用栈的深度到 ftrace_graph 跟踪条目 trace 变量*/
        trace.func = self_addr;
        trace.depth = current->curr_ret_stack + 1;

        /* Only trace if the calling function expects to */
        if (!ftrace_graph_entry(&trace)) { /*ftrace_graph hook 点 1*/
                *parent = old; /*如果出错了，还原指针还需指回原返回地址*/
                return;
        }
        /*由ftrace_graph实现的体系结构无关函数*/
        if (ftrace_push_return_trace(old, self_addr, &trace.depth,
                                     frame_pointer, parent) == -EBUSY) {
                *parent = old; /*如果出错了，还原指针还需指回原返回地址*/
                return;
        }
}
..._```
```

### return_to_handler
* `return_to_handler`函数简单地调用通用的 C 函数`ftrace_return_to_handler()`返回最初的返回地址，这样我们就可以返回最初的调用地点了。
	* ftrace劫持完返回地址，完成在函数返回前所要做的事情后，最终还是要返回到`foo()`调用`bar()`的地方的，`return_to_handler`就是做这个事情的。
	* `ftrace_return_to_handler()`是 function_graph 实现的一个体系架构无关的通用函数，最主要的作用是要返回被调用函数（如`bar()`）的原返回地址。
* arch/x86/kernel/mcount_64.S
```nasm
GLOBAL(return_to_handler)
        subq  $24, %rsp

        /* Save the return values */
        movq %rax, (%rsp)
        movq %rdx, 8(%rsp)
        movq %rbp, %rdi

        call ftrace_return_to_handler

        movq %rax, %rdi     ;函数返回值保存在%rax，把它放到%rdi
        movq 8(%rsp), %rdx
        movq (%rsp), %rax
        addq $24, %rsp
        jmp *%rdi           ;跳到原返回地址
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
```dot
engine:dot
digraph G {
	rankdir="LR";
	node [shape="record"];
	preempt_disable[label="<f0> include/linux/preempt.h | <f1> preempt_disable"];
	preempt_count_add[label="preempt_count_add(1)"];
  preempt_latency_start[label="<f0> kernel/trace/trace_irqsoff.c | <f1> preempt_latency_start"];
	trace_preempt_off;
	start_critical_timing;
	__trace_function;

	preempt_disable:f1 -> preempt_count_inc -> preempt_count_add;
	preempt_count_inc -> preempt_latency_start:f1;

	subgraph cluster1 {
		node[shape="point", width=0, height=0];
		{
			edge[arrowhead="none", penwidth=0];
		  preempt_latency_start:f1 -> s01 -> s02 -> s03;
		}
		{
			edge[arrowhead="none", penwidth=0];
		  s10 -> trace_preempt_off -> s12 -> s13;
		}
		{
			edge[arrowhead="none", penwidth=0];
			s20 -> s21 -> start_critical_timing -> s23;
		}
		{
			edge[arrowhead="none", penwidth=0];
			s30 -> s31 -> s32 -> __trace_function;
		}
		preempt_latency_start:f1 -> trace_preempt_off -> start_critical_timing -> __trace_function;
	}
}
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

# 主框架
## 初始化
```c
init/main.c
start_kernel()
  |   kernel/trace/trace.c
  +-> trace_init()
  |     -> tracer_alloc_buffers()
  |     -> trace_event_init()
  |   kernel/trace/ftrace.c
  +-> ftrace_init()
  |     |
  |     +-> ftrace_process_locs()
  |     |    -> ftrace_allocate_pages()
  |     |    -> ftrace_update_code()
  |     +-> set_ftrace_early_filters()
  +-> rest_init()
```
* 关注两个全局变量`__stop_mcount_loc`和`__start_mcount_loc`，以下是我的猜测：
	* 由编译器脚本创建并放到特定的 section 里
	* 系统启动时创建对应的全局变量
	* 内容为由`-pg`选项加上`mcount`调用的所有函数地址
	* 随着模块的增加会动态调整
	* include/asm-generic/vmlinux.lds.h
	* arch/x86/kernel/vmlinux.lds.S
* 由于 ftrace plugins 的初始化函数多是用 initcalls 的宏进行声明，因此 plugins 的初始化在内核启动过程中是比较靠后的（见 initcalls 章节）
* 通过 initcalls 把 tracer 提供的初始化函数放到 initcalls 所属的 section，这个初始化函数最重要的事情是调用`register_tracer(tracer)`，注册`struct tracer`类型的 tracer plugin

## tracefs的初始化
* `tracefs`文件系统是借助`fs_initcall()`的方式来初始化的
* 为保证向后兼容那些挂载`debugfs`来获得跟踪功能的工具，`tracefs`自动挂载到`debugfs/tracing`目录下，即`/sys/kernel/debug/tracing`目录
```c
fs_initcall(tracer_init_tracefs)
  +-----------+
  +-> trace_access_lock_init()
  +-> tracing_init_dentry()
  |     -> debugfs_create_automount("tracing", NULL, trace_automount, NULL)
  +-> init_tracer_tracefs()
  |     +-> trace_create_file("available_tracers", ...)
  |     +-> trace_create_file("current_tracer", ...)
  |     +-> trace_create_file("trace_options", ...)
  |     +-> trace_create_file("trace", ...)
  |     +-> trace_create_file("trace_pipe", ...)
  |     +-> trace_create_file("tracing_on", ...)
  |     +-> trace_create_file ...
  |     +-> create_trace_options_dir()
  |     |    -> trace_options_init_dentry()
  |     |          -> tracefs_create_dir("options", ...)
  |     |    -> for (i = 0; trace_options[i]; i++) {
  |     |         ...
  |     |         create_trace_option_core_file(tr, trace_options[i], i)
  |     |       }
  |     |   kernel/trace/trace_functions.c
  |     +-> ftrace_create_function_files()
  |     |     -> allocate_ftrace_ops()
  |     |        kernel/trace/ftrace.c
  |     |     -> ftrace_create_filter_files()
  |     |          -> trace_create_file("set_ftrace_filter", ...)
  |     |          -> trace_create_file("set_ftrace_notrace", ...)
  |     +-> for_each_tracing_cpu(cpu)
  |     |      tracing_init_tracefs_percpu(tr, cpu);
  |     +-> ftrace_init_tracefs()
  |           -> trace_create_file("set_ftrace_pid", ...)
  +-> ftrace_init_tracefs_toplevel()
  +-> trace_create_file("tracing_thresh", ...)
  +-> trace_create_file("README", ...)
  +-> trace_create_file ...
  +-> trace_enum_init()
  +-> trace_create_enum_file()
  +-> create_trace_instances()
  +-> update_tracer_options(&global_trace)
```
* 通过`debugfs`提供的`fs/debugfs/inode.c::debugfs_create_automount()`函数，在`debugfs`文件系统创建自动挂载点
	* 传入回调函数`trace_automount()`会在路径名解析的时候被调用，该函数将完成挂载`tracefs`这个动作
* 通过`create_trace_options_dir()`会在初始化期间把`tracing/options`目录创建好，并根据`trace_options[]`数组在`tracing/options`目录下创建预定的 options 文件

## TRACE_FLAGS
* 通过不同的宏定义`C(a, b)`将同一个`TRACE_FLAGS`宏定义分别转成：
	* `enum trace_iterator_bits` 枚举 **位名称** 列表
	* `enum trace_iterator_flags` 枚举 **位掩码** 列表
	* `const char *trace_options[]` 给每个位提供对应的字符串定义，并且顺序与枚举位列表中的保持一致（因为是同一个`TRACE_FLAGS`转出来的）
* kernel/trace/trace.h
	```c
	/*
	 * trace_iterator_flags is an enumeration that defines bit
	 * positions into trace_flags that controls the output.
	 *
	 * NOTE: These bits must match the trace_options array in
	 *       trace.c (this macro guarantees it).
	 */
	#define TRACE_FLAGS                                             \
	                C(PRINT_PARENT,         "print-parent"),        \
	                C(SYM_OFFSET,           "sym-offset"),          \
	                C(SYM_ADDR,             "sym-addr"),            \
	                C(VERBOSE,              "verbose"),             \
	                C(RAW,                  "raw"),                 \
	                C(HEX,                  "hex"),                 \
	                C(BIN,                  "bin"),                 \
	                C(BLOCK,                "block"),               \
	                C(PRINTK,               "trace_printk"),        \
	                C(ANNOTATE,             "annotate"),            \
	                C(USERSTACKTRACE,       "userstacktrace"),      \
	                C(SYM_USEROBJ,          "sym-userobj"),         \
	                C(PRINTK_MSGONLY,       "printk-msg-only"),     \
	                C(CONTEXT_INFO,         "context-info"),   /* Print pid/cpu/time */ \
	                C(LATENCY_FMT,          "latency-format"),      \
	                C(RECORD_CMD,           "record-cmd"),          \
	                C(OVERWRITE,            "overwrite"),           \
	                C(STOP_ON_FREE,         "disable_on_free"),     \
	                C(IRQ_INFO,             "irq-info"),            \
	                C(MARKERS,              "markers"),             \
	                C(EVENT_FORK,           "event-fork"),          \
	                FUNCTION_FLAGS                                  \
	                FGRAPH_FLAGS                                    \
	                STACK_FLAGS                                     \
	                BRANCH_FLAGS

	/*
	 * By defining C, we can make TRACE_FLAGS a list of bit names
	 * that will define the bits for the flag masks.
	 */
	#undef C
	#define C(a, b) TRACE_ITER_##a##_BIT

	enum trace_iterator_bits {
	        TRACE_FLAGS
	        /* Make sure we don't go more than we have bits for */
	        TRACE_ITER_LAST_BIT
	};

	/*
	 * By redefining C, we can make TRACE_FLAGS a list of masks that
	 * use the bits as defined above.
	 */
	#undef C
	#define C(a, b) TRACE_ITER_##a = (1 << TRACE_ITER_##a##_BIT)

	enum trace_iterator_flags { TRACE_FLAGS };
	```
* kernel/trace/trace.c
	```c
	/*
	 * TRACE_FLAGS is defined as a tuple matching bit masks with strings.
	 * It uses C(a, b) where 'a' is the enum name and 'b' is the string that
	 * matches it. By defining "C(a, b) b", TRACE_FLAGS becomes a list
	 * of strings in the order that the enums were defined.
	 */
	#undef C
	#define C(a, b) b

	/* These must match the bit postions in trace_iterator_flags */
	static const char *trace_options[] = {
	        TRACE_FLAGS
	        NULL
	};
	..._
	```
* 例如，对于`TRACE_FLAGS C(VERBOSE, "verbose")`会有`TRACE_ITER_VERBOSE_BIT`，`TRACE_ITER_VERBOSE`两个枚举成员和一个`/sys/kernel/debug/tracing/option/verbose`文件被创建出来

## `struct tracer`结构
* **`struct tracer`结构** 用于定义一个跟踪器以及它与 tracefs 交互的回调函数
* kernel/trace/trace.h
	```c
	/**
	 * struct tracer - a specific tracer and its callbacks to interact with tracefs
	 * @name: the name chosen to select it on the available_tracers file
	 * @init: called when one switches to this tracer (echo name > current_tracer)
	 * @reset: called when one switches to another tracer
	 * @start: called when tracing is unpaused (echo 1 > tracing_on)
	 * @stop: called when tracing is paused (echo 0 > tracing_on)
	 * @update_thresh: called when tracing_thresh is updated
	 * @open: called when the trace file is opened
	 * @pipe_open: called when the trace_pipe file is opened
	 * @close: called when the trace file is released
	 * @pipe_close: called when the trace_pipe file is released
	 * @read: override the default read callback on trace_pipe
	 * @splice_read: override the default splice_read callback on trace_pipe
	 * @selftest: selftest to run on boot (see trace_selftest.c)
	 * @print_headers: override the first lines that describe your columns
	 * @print_line: callback that prints a trace
	 * @set_flag: signals one of your private flags changed (trace_options file)
	 * @flags: your private flags
	 */
	struct tracer {
	        const char              *name;
	        int                     (*init)(struct trace_array *tr);
	        void                    (*reset)(struct trace_array *tr);
	        void                    (*start)(struct trace_array *tr);
	        void                    (*stop)(struct trace_array *tr);
	        int                     (*update_thresh)(struct trace_array *tr);
	        void                    (*open)(struct trace_iterator *iter);
	        void                    (*pipe_open)(struct trace_iterator *iter);
	        void                    (*close)(struct trace_iterator *iter);
	        void                    (*pipe_close)(struct trace_iterator *iter);
	        ssize_t                 (*read)(struct trace_iterator *iter,
	                                        struct file *filp, char __user *ubuf,
	                                        size_t cnt, loff_t *ppos);
	        ssize_t                 (*splice_read)(struct trace_iterator *iter,
	                                               struct file *filp,
	                                               loff_t *ppos,
	                                               struct pipe_inode_info *pipe,
	                                               size_t len,
	                                               unsigned int flags);
	#ifdef CONFIG_FTRACE_STARTUP_TEST
	        int                     (*selftest)(struct tracer *trace,
	                                            struct trace_array *tr);
	#endif
	        void                    (*print_header)(struct seq_file *m);
	        enum print_line_t       (*print_line)(struct trace_iterator *iter);
	        /* If you handled the flag setting, return 0 */
	        int                     (*set_flag)(struct trace_array *tr,
	                                            u32 old_flags, u32 bit, int set);
	        /* Return 0 if OK with change, else return non-zero */
	        int                     (*flag_changed)(struct trace_array *tr,
	                                                u32 mask, int set);
	        struct tracer           *next;
	        struct tracer_flags     *flags;
	        int                     enabled;
	        int                     ref;
	        bool                    print_max;
	        bool                    allow_instances;
	#ifdef CONFIG_TRACER_MAX_TRACE
	        bool                    use_max_tr;
	#endif
	};
	...*```
	```
* `name` 用于在`available_tracers`中显示的跟踪器名字，选择跟踪器时用的也是这个名字
* `flags` 跟踪器的私有 flags，用于设定特定于某个跟踪器的选项的集合，定义如下
* kernel/trace/trace.h
	```c
	/*
	 * An option specific to a tracer. This is a boolean value.
	 * The bit is the bit index that sets its value on the
	 * flags value in struct tracer_flags.
	 */
	struct tracer_opt {
	        const char      *name; /* Will appear on the trace_options file */
	        u32             bit; /* Mask assigned in val field in tracer_flags */
	};

	/*
	 * The set of specific options for a tracer. Your tracer
	 * have to set the initial value of the flags val.
	 */
	struct tracer_flags {
	        u32                     val;
	        struct tracer_opt       *opts;
	        struct tracer           *trace;
	};

	/* Makes more easy to define a tracer opt */
	#define TRACER_OPT(s, b)        .name = #s, .bit = b
	...*```
	```
* `val` tracer 私有 flags 的初始值
* `opts` 指向 tracer 私有的 options
* `trace` 回指向所属的 tracer plugin 实例

### tracer 的方法

方法名称 | 作用
---|---
init | 切换到这个 tracer 时调用 (`echo name > current_tracer`)
reset | 换到其他 tracer 时调用
start | 跟踪开启时调用 (`echo 1 > tracing_on`)
stop | 跟踪暂停时调用 (`echo 0 > tracing_on`)
update_thresh | `tracing_thresh` 文件更新时调用，当延迟大于该文件指定的 *微秒*，延迟跟踪器记录一条trace
open | `trace` 文件打开时调用
pipe_open | `trace_pipe` 文件被打开时调用
close | `trace` 文件释放时调用
pipe_close | `trace_pipe` 被释放时调用
read | 让 tracer 可以覆写 `trace_pipe` 缺省的 `read` 回调
splice_read | 让 tracer 可以覆写 `trace_pipe` 缺省的 `splice_read` 回调
selftest | 在启动时运行 selftest (见 trace_selftest.c)
print_headers | 覆写首行的列描述
print_line | 打印一条 trace 的回调
set_flag | 当 tracer 的私有 flags 发生改变时调用 (`trace_options` 文件)
flag_changed | 当通用的 flag 发生改变时调用，让 tracer 有机会决定是否接受某个 option 的改变

### trace_options
* 例如一个 function_graph 跟踪器的例子
* kernel/trace/trace_functions_graph.c
	```c
	static struct tracer_opt trace_opts[] = {
	        /* Display overruns? (for self-debug purpose) */
	        { TRACER_OPT(funcgraph-overrun, TRACE_GRAPH_PRINT_OVERRUN) },
	        /* Display CPU ? */
	        { TRACER_OPT(funcgraph-cpu, TRACE_GRAPH_PRINT_CPU) },
	        /* Display Overhead ? */
	        { TRACER_OPT(funcgraph-overhead, TRACE_GRAPH_PRINT_OVERHEAD) },
	        /* Display proc name/pid */
	        { TRACER_OPT(funcgraph-proc, TRACE_GRAPH_PRINT_PROC) },
	        /* Display duration of execution */
	        { TRACER_OPT(funcgraph-duration, TRACE_GRAPH_PRINT_DURATION) },
	        /* Display absolute time of an entry */
	        { TRACER_OPT(funcgraph-abstime, TRACE_GRAPH_PRINT_ABS_TIME) },
	        /* Display interrupts */
	        { TRACER_OPT(funcgraph-irqs, TRACE_GRAPH_PRINT_IRQS) },
	        /* Display function name after trailing } */
	        { TRACER_OPT(funcgraph-tail, TRACE_GRAPH_PRINT_TAIL) },
	        /* Include sleep time (scheduled out) between entry and return */
	        { TRACER_OPT(sleep-time, TRACE_GRAPH_SLEEP_TIME) },
	        /* Include time within nested functions */
	        { TRACER_OPT(graph-time, TRACE_GRAPH_GRAPH_TIME) },
	        { } /* Empty entry */
	};

	static struct tracer_flags tracer_flags = {
	        /* Don't display overruns, proc, or tail by default */
	        .val = TRACE_GRAPH_PRINT_CPU | TRACE_GRAPH_PRINT_OVERHEAD |
	               TRACE_GRAPH_PRINT_DURATION | TRACE_GRAPH_PRINT_IRQS |
	               TRACE_GRAPH_SLEEP_TIME | TRACE_GRAPH_GRAPH_TIME,
	        .opts = trace_opts
	};
	```
* `struct tracer_opt trace_opts[]`提供了一个跟踪器选项集合
* `struct tracer_flags tracer_flags`的 **`val`域** 则是 function_graph 的初始选项，通过 **`opts`域** 把变量`tracer_flags`能支持的选项与`trace_opts[]`集合联系起来
* kernel/trace/trace.c
	```c
	static const struct file_operations tracing_iter_fops = {
	        .open           = tracing_trace_options_open,
	        .read           = seq_read,
	        .llseek         = seq_lseek,
	        .release        = tracing_single_release_tr,
	        .write          = tracing_trace_options_write,
	};
	...
	static void
	init_tracer_tracefs(struct trace_array *tr, struct dentry *d_tracer)
	{
	    ...
	    /*/sys/kernel/debug/tracing/trace_options*/
	    trace_create_file("trace_options", 0644, d_tracer,
	        tr, &tracing_iter_fops);
	    ...
	};
	```
* 打开`/sys/kernel/debug/tracing/trace_options`文件时会通过`tracing_trace_options_open()`设置`seq_read`时的回调为`tracing_trace_options_show()`
* 写`/sys/kernel/debug/tracing/trace_options`文件时最终会调用 tracer 的`set_flag`
  ```c
  tracing_trace_options_write()
    -> trace_set_options()
          |
          +-> set_tracer_flag()
          |     -> tr->current_trace->flag_changed(tr, mask, !!enabled)
          +-> set_tracer_option()
                -> set_tracer_option()
                      -> __set_tracer_option()
                          -> trace->set_flag(tr, tracer_flags->val, opts->bit, !neg);
  ```
* kernel/trace/trace.c
	```c
	/* These must match the bit postions in trace_iterator_flags */
	static const char *trace_options[] = {
	        TRACE_FLAGS /*定义在 kernel/trace/trace.h 中的一列由 option 名字组成的字符串*/
	        NULL
	};

	static int trace_set_options(struct trace_array *tr, char *option)
	{
	        char *cmp;
	        int neg = 0;
	        int ret = -ENODEV;
	        int i;
	        size_t orig_len = strlen(option);
	        /*去除多余的空格*/
	        cmp = strstrip(option);
	        /*关闭某个 option 只需在其之前加上“no”*/
	        if (strncmp(cmp, "no", 2) == 0) {
	                neg = 1;
	                cmp += 2;
	        }

	        mutex_lock(&trace_types_lock);
	        /*设置 tracer 通用的 option，设置完一个 option 后结束循环*/
	        for (i = 0; trace_options[i]; i++) {
	                if (strcmp(cmp, trace_options[i]) == 0) {
	                        ret = set_tracer_flag(tr, 1 << i, !neg);
	                        break;
	                }
	        }
	        /*如果设置的不是通用 option，测试设置的是不是 tracer 私有的 options*/
	        /* If no option could be set, test the specific tracer options */
	        if (!trace_options[i])
	                ret = set_tracer_option(tr, cmp, neg);

	        mutex_unlock(&trace_types_lock);

	        /*
	         * If the first trailing whitespace is replaced with '\0' by strstrip,
	         * turn it back into a space.
	         */
	        if (orig_len > strlen(option))
	                option[strlen(option)] = ' ';

	        return ret;
	}
	```

## 注册跟踪器
* 全局变量`trace_types`指向所有可用 tracer 组成的链表
* `trace_types_lock`是保护`trace_types`链表的锁
* kernel/trace/trace.c
```c
/* trace_types holds a link list of available tracers. */
static struct tracer            *trace_types __read_mostly;

/*
 * trace_types_lock is used to protect the trace_types list.
 */
DEFINE_MUTEX(trace_types_lock);
...
#define MAX_TRACER_SIZE         100
static char bootup_tracer_buf[MAX_TRACER_SIZE] __initdata;
static char *default_bootup_tracer;
...
static int __init set_cmdline_ftrace(char *str)
{
        strlcpy(bootup_tracer_buf, str, MAX_TRACER_SIZE);
        default_bootup_tracer = bootup_tracer_buf;
        /* We are using ftrace early, expand it */
        ring_buffer_expanded = true;
        return 1;
}
/*可以通过内核命令行定制缺省启动 tracer*/
__setup("ftrace=", set_cmdline_ftrace);
...
/*
 * The global_trace is the descriptor that holds the top-level tracing
 * buffers for the live tracing.
 */
static struct trace_array global_trace = {
        .trace_flags = TRACE_DEFAULT_FLAGS,
};
...
static char trace_boot_options_buf[MAX_TRACER_SIZE] __initdata;

static int __init set_trace_boot_options(char *str)
{               
        strlcpy(trace_boot_options_buf, str, MAX_TRACER_SIZE);
        return 0;
}
/*可以通过内核命令行定制缺省启动 tracer 的option*/
__setup("trace_options=", set_trace_boot_options);
...
static void
create_trace_option_files(struct trace_array *tr, struct tracer *tracer)
{
        struct trace_option_dentry *topts;
        struct trace_options *tr_topts;
        struct tracer_flags *flags;
        struct tracer_opt *opts;
        int cnt;
        int i;

        if (!tracer)
                return;
        /*跟踪器的私有 flags*/
        flags = tracer->flags;

        if (!flags || !flags->opts)
                return;

        /*
         * If this is an instance, only create flags for tracers
         * the instance may have.
         */
        if (!trace_ok_for_array(tracer, tr))
                return;
        /*新的 flags 与原有的 flags 不能重复，比较 struct trace_flags类型的实例地址*/
        for (i = 0; i < tr->nr_topts; i++) {
                /* Make sure there's no duplicate flags. */
                if (WARN_ON_ONCE(tr->topts[i].tracer->flags == tracer->flags))
                        return;
        }
        /*该 struct trace_flags 类型实例包含的 options*/
        opts = flags->opts;
        /*计算新 options 的数目*/
        for (cnt = 0; opts[cnt].name; cnt++)
                ;
        /*给新 options 分配空间*/
        topts = kcalloc(cnt + 1, sizeof(*topts), GFP_KERNEL);
        if (!topts)
                return;
        /*原 options 数组扩容*/
        tr_topts = krealloc(tr->topts, sizeof(*tr->topts) * (tr->nr_topts + 1),
                            GFP_KERNEL);
        if (!tr_topts) {
                kfree(topts);
                return;
        }
        /*指向扩容后的 options 数组地址*/
        tr->topts = tr_topts;
        tr->topts[tr->nr_topts].tracer = tracer; /*放入新 options 元素*/
        tr->topts[tr->nr_topts].topts = topts;
        tr->nr_topts++; /*options 数组计数增加*/
        /*在 /sys/kernel/debug/tracing/options 下创建新的 option 文件*/
        for (cnt = 0; opts[cnt].name; cnt++) {
                create_trace_option_file(tr, &topts[cnt], flags,
                                         &opts[cnt]);
                WARN_ONCE(topts[cnt].entry == NULL,
                          "Failed to create trace option: %s",
                          opts[cnt].name);
        }
}
...
static void add_tracer_options(struct trace_array *tr, struct tracer *t)
{
        /* Only enable if the directory has been created already. */
        if (!tr->dir)
                return;

        create_trace_option_files(tr, t);
}
...
/**
 * register_tracer - register a tracer with the ftrace system.
 * @type - the plugin for the tracer
 *
 * Register a new plugin tracer.
 */
int __init register_tracer(struct tracer *type)
{
        struct tracer *t;
        int ret = 0;
        /*tracer 必须得有名字*/
        if (!type->name) {
                pr_info("Tracer must have a name\n");
                return -1;
        }
        /*tracer 名字最大长度是 100*/
        if (strlen(type->name) >= MAX_TRACER_SIZE) {
                pr_info("Tracer has a name longer than %d\n", MAX_TRACER_SIZE);
                return -1;
        }

        mutex_lock(&trace_types_lock);

        tracing_selftest_running = true;
        /*新 tracer 名字不能和已有的重名*/
        for (t = trace_types; t; t = t->next) {
                if (strcmp(type->name, t->name) == 0) {
                        /* already found */
                        pr_info("Tracer %s already registered\n",
                                type->name);
                        ret = -1;
                        goto out;
                }
        }
        /*新 tracer 没有定义自己的 set_flag 回调则给它一个空的 dummy_set_flag() 函数*/
        if (!type->set_flag)
                type->set_flag = &dummy_set_flag;
        /*新 tracer 没有定义自己的私有 flag 则给它一个空的 tracer_flags 实例*/
        if (!type->flags) {
                /*allocate a dummy tracer_flags*/
                type->flags = kmalloc(sizeof(*type->flags), GFP_KERNEL);
                if (!type->flags) {
                        ret = -ENOMEM;
                        goto out;
                }
                type->flags->val = 0;
                type->flags->opts = dummy_tracer_opt;
        } else
                if (!type->flags->opts)
                        type->flags->opts = dummy_tracer_opt;
        /*设置私有 option 时通过这个回指的指针找到提供该私有 flag 的 tracer，并调用它提供
          的 set_flag 回调函数*/
        /* store the tracer for __set_tracer_option */
        type->flags->trace = type;
        /*如果开启了 tracer 自测，会在这里先测试一下*/
        ret = run_tracer_selftest(type);
        if (ret < 0)
                goto out;
        /*前插的方式插入 trace_types 链表*/
        type->next = trace_types;
        trace_types = type;
        add_tracer_options(&global_trace, type);/*新跟踪器的 flags 合并到全局 options*/

 out:
        tracing_selftest_running = false;
        mutex_unlock(&trace_types_lock);
        /*如果之前都没出错，且没通过内核命令行参数设置缺省启动 tracer，则注册过程结束*/
        if (ret || !default_bootup_tracer)
                goto out_unlock;
        /*通过内核命令行参数设置缺省启动 tracer，则看看是不是缺省启动 tracer*/
        if (strncmp(default_bootup_tracer, type->name, MAX_TRACER_SIZE))
                goto out_unlock;

        printk(KERN_INFO "Starting tracer '%s'\n", type->name);
        /* Do we want this tracer to start on bootup? */
        tracing_set_tracer(&global_trace, type->name); /*启动 tracer*/
        default_bootup_tracer = NULL; /*tracer 已启动，该值可以清空*/
        /*应用内核命令行"trace_options="设置的 options*/
        apply_trace_boot_options();

        /* disable other selftests, since this will break it. */
        tracing_selftest_disabled = true;
#ifdef CONFIG_FTRACE_STARTUP_TEST
        printk(KERN_INFO "Disabling FTRACE selftests due to running tracer '%s'\n",
               type->name);
#endif

 out_unlock:
        return ret;
}
...__```
```

## function tracer的初始化
* function tracer 用`core_initcall()`进行注册
* kernel/trace/trace_functions.c
```c
static struct tracer function_trace __tracer_data =
{
        .name           = "function",
        .init           = function_trace_init,
        .reset          = function_trace_reset,
        .start          = function_trace_start,
        .flags          = &func_flags,
        .set_flag       = func_set_flag,
        .allow_instances = true,
#ifdef CONFIG_FTRACE_SELFTEST
        .selftest       = trace_selftest_startup_function,
#endif
};
...
static __init int init_function_trace(void)
{
        init_func_cmd_traceon();
        return register_tracer(&function_trace);
}
core_initcall(init_function_trace);
```

# Reference

* [ftrace 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/)
* [Ftrace 实现原理与开发实践](http://tinylab.org/ftrace-principle-and-practice/)
