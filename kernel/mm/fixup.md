# Page Fault

## 异常修正表（Exception Fixup Table）
* 在访问内核地址空间时，缺页异常可能被各种条件出发，如下所述：
  * 内核本身的程序设计错误导致访问不正确的地址，这个在稳定版本中永远不会发生，在开发版本中偶尔会发生
  * 内核通过用户空间传递的参数访问了无效地址
  * 访问使用`vmalloc`分配的区域，触发缺页异常
* 前两种情况是真正的错误，内核必须使用最后的手段 - **异常修正（exception fixup）** 机制来进行处理
* `vmalloc`的情况是导致缺页异常的合理情况，必须加以校正。
  * 直至对应的缺页异常发生之前，`vmalloc`区域中的修改都不会传输到进程的页表中。
  * 因此在对`vmalloc`异常进行处理时，必须从主页表复制适当的访问权限信息到进程的页表中。
* 对于用户态发生的缺页异常，内核将使用按需调页机制，自动并透明地返回一个物理内存页
* 内核访问用户空间传入的地址时，过去需要通过`verify_area()`确保传入的地址在 VMA 范围内，然而大部分的检查是成功的，这导致大量不必要的性能开销。因此，后来改用缺页异常来处理
  * 如果用户空间传入的地址是合法的（在其 VMA 范围内），则建立映射，之后在内核态再访问同样的页就无需再次映射了
  * 如果用户空间传入的地址是非合法，内核也不能因此挂掉，因此需要修正内核的指令流，并最终给用户态程序返回一个错误
* 当访问发生在内核态，发生缺页时必须使用不同的手段进行校正。每次发生缺页异常时，将输出异常的原因和当前执行代码的地址。
* 这使得内核可以编译一个列表，列出所有可能执行未授权内存访问操作处理（异常处理）的代码块。这就是 **异常表（Exception Table）**
* 发生访问用户态内容发生内核缺页时`do_user_addr_fault()`
  * 如果在异常表里没有对应的 fixup 表项`if (!user_mode(regs) && !search_exception_tables(regs->ip))`
    * 进入错误处理`bad_area_nosemaphore(regs, error_code, address)`
  * 如果在异常表里有对应的 fixup 表项，且缺页地址在 VMA 内，则先在内核态建立好页的映射，修正完缺页之后，就可访问用户态的地址了
  * 如果缺页地址不在 VMA 内，则进入`bad_area()`处理，看`kernelmode_fixup_or_oops()->fixup_exception()`
    * 先查找异常表，如果异常表里有对应的 fixup 表项
      * 在`fixup_exception()`中根据不同的情况修正错误
      * 得到 fixup 地址
      * 缺页处理函数修改`regs->ip`为 fixup 地址，因此缺页处理函数返回的时候会跳到异常表给出的 fixup 地址
      * fixup 地址通常是之前因访问用户态内存导致缺页的下一条指令（回到同一条指令不是又继续缺页吗？所以必须是下一条）
    * 如果异常表里没有对应的 fixup 表项`fixup_exception()`返回错误`0`，或者是表项类型不对，这应该是内核的错误，会导致 oops
* **注意**：由于异常表的构建方式需要排序（方便运行时用`bsearch()`做 binary 搜索），因此仅对`.text`部分中的代码使用异常
* `CONFIG_BUILDTIME_TABLE_SORT`允许通过 host 工具`scripts/sorttable`对内核映像的链接后的`__ex_table` section 进行排序。它将符号`main_extable_sort_needed `设置为`0`，避免在引导时对`__ex_table` section 进行排序。

* 异常表条目的定义
  * arch/x86/include/asm/extable.h
  ```c
  /*
   * The exception table consists of two addresses relative to the
   * exception table entry itself and a type selector field.
   *
   * The first address is of an instruction that is allowed to fault, the
   * second is the target at which the program should continue.
   *
   * The type entry is used by fixup_exception() to select the handler to
   * deal with the fault caused by the instruction in the first field.
   *
   * All the routines below use bits of fixup code that are out of line
   * with the main instruction path.  This means when everything is well,
   * we don't even have to jump over them.  Further, they do not intrude
   * on our cache or tlb entries.
   */

  struct exception_table_entry {
      int insn, fixup, data;
  };
  ```
  * `insn`: 内核允许在虚拟地址空间发生异常的指令的地址（32 bit，64 bit 为到 exception table 起始地址的偏移）
  * `fixup`：指定了`fixup_exception()`修正完后要跳转到的地址
  * `data`：就是原来的`type`，原来`type`功能单一，但现在被分为多个段复用了
    * **type** `EX_DATA_TYPE_MASK ((int)0x000000FF)`: 异常表项的类型，和原来的`type`意思一样
    * **reg**  `EX_DATA_REG_MASK  ((int)0x00000F00)`：`fixup_exception()`修正时要操作的寄存器
    * **flag** `EX_DATA_FLAG_MASK ((int)0x0000F000)`：目前就两个 flags 可用，可以控制在修正时是否要清除`rax`或`rdx`寄存器
    * **imm**  `EX_DATA_IMM_MASK  ((int)0xFFFF0000)`：要给寄存器填入的立即数

* 插入条目，例如`_ASM_EXTABLE_CPY(1b, .L_fixup_4x8b_copy)`
  * arch/x86/include/asm/asm.h
  ```c
  /* Exception table entry */
  #ifdef __ASSEMBLY__

  # define _ASM_EXTABLE_TYPE(from, to, type)          \
      .pushsection "__ex_table","a" ;             \
      .balign 4 ;                     \
      .long (from) - . ;                  \
      .long (to) - . ;                    \
      .long type ;                        \
      .popsection
  ...
  # endif
  ```
  * 对于 x86-64 来说，`insn`和`fixup`存的是指令到其在 exception table 中对应条目的偏移
  * 通常 exception table 会放到代码段的后面，所以这里存的会是负数（整个代码段不超过`2GB`，`int`类型应该能覆盖），即使放在前面问题也不太大
* 条目转为指令地址
  * lib/extable.c
  ```c
  #ifndef ARCH_HAS_RELATIVE_EXTABLE
  #define ex_to_insn(x)   ((x)->insn)
  #else
  static inline unsigned long ex_to_insn(const struct exception_table_entry *x)
  {
      return (unsigned long)&x->insn + x->insn;
  }
  #endif
  ```
  * 在使用时，内联函数`ex_to_insn()`会根据条目的地址再加上（其实是减去）偏移从而得到指令的绝对地址

### 异常表的插入

### 异常表的使用
* 例如，以下宏作用如下：
  ```c
  1:  popl    %ds
  2:  popl    %es
  ...
  _ASM_EXTABLE_TYPE(1b, 2b, EX_TYPE_POP_ZERO|EX_REG_DS)
  ```
* 标号 1 处发生内核态访问用户态地址的缺页异常，`popl %ds`不会被执行，而是`fixup_exception()`对它进行修正
* `#define EX_TYPE_POP_ZERO (EX_TYPE_POP_REG | EX_DATA_IMM(0))`，由此可见，`EX_TYPE_POP_ZERO`是 *弹出栈顶元素到寄存器* 和 *寄存器的值为立即数 0* 的复合
* `EX_REG_DS`表明想弹出到寄存器`ds`
* 因为这个用户态地址已经被认定为非法引发异常了，我们当然不能在修正时再去访问它，`EX_TYPE_POP_REG`对应的修正操作就是`regs->sp += sizeof(long)`，收缩栈指针把它跳过
* 然后`fallthrough`到`ex_handler_imm_reg(e, regs, reg, imm)`
  ```c
  static bool ex_handler_imm_reg(const struct exception_table_entry *fixup,
                  struct pt_regs *regs, int reg, int imm)
  {
      *pt_regs_nr(regs, reg) = (long)imm;
      return ex_handler_default(fixup, regs);
  }
  ```
* `data`的`imm`位域得到`imm`的值是立即数`0`，`EX_DATA_IMM(0)`
* `data`的`reg`位域得到要操作的寄存器是`ds`，`EX_REG_DS`，故`reg`是`ds`在`struct pt_regs *regs`中的偏移
* 所以赋值语句其实是`regs->ds = 0`，因为栈上弹出的是非法的用户态地址，我们不能再用它，所以存个立即数`0`到要弹出的寄存器中吧
* 然后做通用的异常修正处理`ex_handler_default(fixup, regs)`
  ```c
  static inline unsigned long
  ex_fixup_addr(const struct exception_table_entry *x)
  {
      return (unsigned long)&x->fixup + x->fixup;
  }

  static bool ex_handler_default(const struct exception_table_entry *e,
                      struct pt_regs *regs)
  {
      if (e->data & EX_FLAG_CLEAR_AX)
          regs->ax = 0;
      if (e->data & EX_FLAG_CLEAR_DX)
          regs->dx = 0;

      regs->ip = ex_fixup_addr(e);
      return true;
  }
  ```
* `(unsigned long)&x->fixup + x->fixup` 之前已经说过，根据修正条目的地址`&x->fixup`再加上偏移`x->fixup`从而得到修正指令的绝对地址，这里应该是标号 2 的地址
* `regs->ip = ex_fixup_addr(e)`会让异常处理返回的时候返回到标号 2
* `EX_FLAG_CLEAR_AX`和`EX_FLAG_CLEAR_DX` flags 会导致清除`ax`和`dx`寄存器

#### 宏`_ASM_EXTABLE_TYPE_REG`
* 宏`_ASM_EXTABLE_TYPE_REG`要根据汇编阶段的寄存器类型来生成`data`域的`reg`位域的值
* arch/x86/include/asm/asm.h
```c
# define DEFINE_EXTABLE_TYPE_REG \
    ".macro extable_type_reg type:req reg:req\n" \ //定义汇编宏 extable_type_reg，该宏需要入参 type 和 reg
    ".set found, 0\n"                            \ //设置 as 变量 found = 0
    ".set regnr, 0\n"                            \ //设置 as 变量 regnr = 0
    ".irp rs,rax,rcx,rdx,rbx,rsp,rbp,rsi,rdi,r8,r9,r10,r11,r12,r13,r14,r15\n" \ //重复后面的是参数列表，rs 为要替换的占位符，注意，这里参数数据对应到数组 pt_regoff[] 里的偏移，不能乱改
    ".ifc \\reg, %%\\rs\n"                       \ //如果两个字符串相同，则汇编以下代码部分（字符串比较区分大小写）
    ".set found, found+1\n"                      \ //found = fount + 1
    ".long \\type + (regnr << 8)\n"              \ //在此处放置数据`data`，`type`、`flag`、`imm`都是实参给的，8~11 位`reg`是这里需要确定的
    ".endif\n"                                   \ //结束字符串比较
    ".set regnr, regnr+1\n"                      \ //regnr = regnr + 1，因为会重复，`reg`会依次对应到参数列表中的寄存器
    ".endr\n"                                    \ //.irq 重复结束
    ".set regnr, 0\n"                            \
    ".irp rs,eax,ecx,edx,ebx,esp,ebp,esi,edi,r8d,r9d,r10d,r11d,r12d,r13d,r14d,r15d\n" \ //再来一组 32 位的通用寄存器
    ".ifc \\reg, %%\\rs\n"                       \
    ".set found, found+1\n"                      \
    ".long \\type + (regnr << 8)\n"              \
    ".endif\n"                                   \
    ".set regnr, regnr+1\n"                      \
    ".endr\n"                                    \
    ".if (found != 1)\n"                         \ //两次`.irq`遍历了两组寄存器，理论上`.ifc`能确定一个传入的寄存器，`found`不是 1 肯定是出错了
    ".error \"extable_type_reg: bad register argument\"\n" \
    ".endif\n"                                   \
    ".endm\n"

# define UNDEFINE_EXTABLE_TYPE_REG \
    ".purgem extable_type_reg\n"    //删除汇编宏 extable_type_reg 的定义
...
# define _ASM_EXTABLE_TYPE_REG(from, to, type, reg)             \
    " .pushsection \"__ex_table\",\"a\"\n"                  \
    " .balign 4\n"                              \
    " .long (" #from ") - .\n"                      \
    " .long (" #to ") - .\n"                        \
    DEFINE_EXTABLE_TYPE_REG                         \
    "extable_type_reg reg=" __stringify(reg) ", type=" __stringify(type) " \n"\
    UNDEFINE_EXTABLE_TYPE_REG                       \
    " .popsection\n"
```

* `show_stack_regs(regs)`可以根据传入的`struct pt_regs *regs`来 dump stack
* `k0`寄存器
> AVX-512 还引入了 8 个 OPMask 寄存器，`k0`到`k7`。`k0`是一个特殊的情况，它的行为很像一些 RISC ISA 上的 “零 “寄存器：它不能被存储到，而且从它的加载总是产生一个全部为1的位掩码。

### 旧的实现
* 在旧实现里，异常表里的`fixup`域都指向放在`.fixup section`里的修正代码，对应现在的缺页异常处理是这样的
  * 先查找异常表，如果异常表里有对应的 fixup 表项
    * 得到 fixup 地址
    * 缺页处理函数修改`regs->ip`为 fixup 地址，因此缺页处理函数返回的时候会跳到异常表给出的 fixup 地址
    * 跳到`.fixup section`里的 fixup 例程后接着修复（大部分情况是把`RAX`设为`-EFAULT == -14`，让用户态程序得到一个错误）
    * fixup 例程的最后通常会跳转到之前因访问用户态内存导致缺页的下一条指令（回到同一条指令不是又继续缺页吗？所以必须是下一条）
  * 如果异常表里没有对应的 fixup 表项`fixup_exception()`返回错误`0`，或者是表项类型不对，这应该是内核的错误，会导致 oops
* Peter Zijlstra 在 https://lore.kernel.org/all/20211110100102.250793167@infradead.org/ `v5.17-rc1`里把旧的实现改成现在的样子。
  * 因为旧的实现运行在`.fixup section`里的代码回给栈回溯造成麻烦，比如说符号的查找和 unwind。
  * 而且，这么多分散的`.fixup`项做的事情差不多，然后又集中放到`.fixup section`里，既冗余的且没什么必要
* 修改的思路：
  * 把`struct exception_table_entry`的单一功能`type`域变为`type`、`reg`、`flags`、`imm`几个位域复合而成的`data`域
  * 在各个`pushsection __ex_table`时就制定好几个位域的值
  * 把`struct exception_table_entry`的`fixup`域通常指定为导致缺页的下一条指令，而不是 fixup 例程的地址
  * 在`fixup_exception()`根据制定好的位域的值，进行 case by case 的修正处理，替代原有的分散的各个 fixup 例程
  * 修正完之后修改`regs->ip`为 fixup 地址
  * 异常处理完后返回 fixup 地址

## References
* [Kernel level exception handling](https://docs.kernel.org/x86/exception-tables.html)
* [修改掉Linux内核缺页中断处理的exception fixup表](https://blog.csdn.net/dog250/article/details/106105523)
* [内核缺页异常处理](https://blog.csdn.net/CToday/article/details/25418991)
* [利用异常表处理 Linux 内核态缺页异常](http://tjpm.blog.chinaunix.net/uid-23860671-id-294456.html)
* [C11标准的泛型机制](https://www.cnblogs.com/zenny-chen/archive/2012/09/20/2695381.html)
* [X86_64 机器上一共有多少个寄存器](https://www.owalle.com/2021/12/26/all-registers-x86-64/)
* [C/C++ 中嵌入汇编总结](https://www.owalle.com/2019/05/22/inline-assembly/)
