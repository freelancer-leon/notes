# LAM_U57 Patchset

* LAM User 模式与分页模式的组合

No. | LAM User Mode          | Paging Mode                  | Valid
----|------------------------|------------------------------|-------
1   | LAM_U57 VA bit [62:57] | 5 level paging VA bit [56:0]	| V
2   | LAM_U48 VA bit [62:48] | 4 level paging VA bit [47:0]	| V
3   | LAM_U57 VA bit [62:57] | 4 level paging VA bit [47:0]	| V
4   | LAM_U48 VA bit [62:48] | 5 level paging VA bit [56:0]	| X

* 对于组合 4 可见 LAM_U48 屏蔽的地址位与用于 5 级分页寻址的位重叠，因此不是有效的组合

## [PATCH 01/19] x86/mm: Rework address range check in get_user() and put_user()
* 函数 `get_user()` 和 `put_user()` 检查目标地址范围是否位于虚拟地址空间的用户空间部分。为了执行此检查，函数将范围的末尾与 `TASK_SIZE_MAX` 进行比较。
* 对于使用 `CONFIG_X86_5LEVEL` 编译的内核，此过程需要使用 `ALTERNATIVE` 进行一些额外的技巧，因为 `TASK_SIZE_MAX` 取决于所使用的分页模式。
* Linus 建议可以针对 64 位内核简化此检查。检查地址的第 `63` 位就足以确保该范围属于用户空间。此外，如果设置了 bit `63`，则可以通过将目标地址设置为全 `1` 来避免使用分支。
* 无需检查访问范围的结尾，因为用户空间范围的结尾和内核范围的开头之间存在巨大差距。该差距由规范空洞和内核和用户空间端均未使用的范围组成。
* 如果设置了 bit `63` 的地址被向下传递，则会触发 `#GP` 异常。`_ASM_EXTABLE_UA()` 会抱怨这一点。将其替换为普通的 `_ASM_EXTABLE()`，因为它是现在的预期行为。
* 更新的 `get_user()` 和 `put_user()` 检查还与 Linear Address Masking 兼容，它允许用户空间在指针的高位中编码元数据，并且无需在处理地址之前取消标记地址。
---
* `do_get_user_call` 用于 `get_user()` 和 `__get_user()` 扩展为正确的特殊函数调用，该函数由于返回值和错误而具有奇怪的调用约定，并且这取决于传入的指针的大小。
* 注意：出于符号原因，我们必须将结果转换为指针的类型。
* 使用 `_ASM_DX` 作为 register specifier 有点简化，因为 gcc 只关心它作为起点而不关心大小：对于 64-bit 的值，它将在 32 位上使用 `%ecx:%edx`（`%ecx` 是 gcc 的 x86 寄存器序列中的下一个寄存器），而在 64 位上使用 `%rdx`。
* Clang/LLVM 关心寄存器的大小，但仍然希望基址寄存器最终成为一对。
* arch/x86/include/asm/uaccess.h
```cpp
/*
 * This is used for both get_user() and __get_user() to expand to
 * the proper special function call that has odd calling conventions
 * due to returning both a value and an error, and that depends on
 * the size of the pointer passed in.
 *
 * Careful: we have to cast the result to the type of the pointer
 * for sign reasons.
 *
 * The use of _ASM_DX as the register specifier is a bit of a
 * simplification, as gcc only cares about it as the starting point
 * and not size: for a 64-bit value it will use %ecx:%edx on 32 bits
 * (%ecx being the next register in gcc's x86 register sequence), and
 * %rdx on 64 bits.
 *
 * Clang/LLVM cares about the size of the register, but still wants
 * the base register for something that ends up being a pair.
 */
#define do_get_user_call(fn,x,ptr)                  \
({                                  \
    int __ret_gu; /*该变量作为 get_user 返回值*/ \
    register __inttype(*(ptr)) __val_gu asm("%"_ASM_DX); /*声明寄存器变量 __val_gu 并用 %rdx 存储*/ \
    __chk_user_ptr(ptr); /*触发编译器对 ptr 的类型检查？*/ \
    asm volatile("call __" #fn "_%P4"               \
             : "=a" (__ret_gu), "=r" (__val_gu),        \
            ASM_CALL_CONSTRAINT             \
             : "0" (ptr), "i" (sizeof(*(ptr))));        \
    instrument_get_user(__val_gu); /*在此插桩*/ \
    (x) = (__force __typeof__(*(ptr))) __val_gu; /*__val_gu 赋值给 x*/ \
    __builtin_expect(__ret_gu, 0);                  \
})

/**
 * get_user - Get a simple variable from user space.
 * @x:   Variable to store result.
 * @ptr: Source address, in user space.
 *
 * Context: User context only. This function may sleep if pagefaults are
 *          enabled.
 *
 * This macro copies a single simple variable from user space to kernel
 * space.  It supports simple types like char and int, but not larger
 * data types like structures or arrays.
 *
 * @ptr must have pointer-to-simple-variable type, and the result of
 * dereferencing @ptr must be assignable to @x without a cast.
 *
 * Return: zero on success, or -EFAULT on error.
 * On error, the variable @x is set to zero.
 */
#define get_user(x,ptr) ({ might_fault(); do_get_user_call(get_user,x,ptr); })
```
* 对于 `"call __" #fn "_%P4"`，`#fn` 被替换为 `get_user`，`%P4` 被替换为第 4 个参数的结果，即 `"i" (sizeof(*(ptr)))`，
  * 可能是 `1`（byte）、`2`（word）、`4`（DW）、`8`（QW），
  * 于是汇编结果根据 `ptr` 的类型可能为 `call __get_user_1`、`call __get_user_2`、`call __get_user_4`、`call __get_user_8`
* 输入部分 `"=a" (__ret_gu), "=r" (__val_gu), ASM_CALL_CONSTRAINT`
  * `"=a" (__ret_gu)`：`=` 表示操作数是 *只写* 的，`a` 使用寄存器 `%rax`
  * `"=r" (__val_gu)`：`r` 表示通用寄存器，上面说了，使用寄存器 `%rdx`
  * `ASM_CALL_CONSTRAINT` 宏展开如下，可知变量 `current_stack_pointer` 占用寄存器 `%rsp`
* 此输出约束应用于任何具有“`call`”指令的内联汇编。否则，可能会在包含函数设置 *帧指针* 之前插入 asm。如果您忘记执行此操作，`objtool` 可能会打印“call without frame pointer save/setup” 的警告。
* arch/x86/include/asm/asm.h
```cpp
/*
 * This output constraint should be used for any inline asm which has a "call"
 * instruction.  Otherwise the asm may be inserted before the frame pointer
 * gets set up by the containing function.  If you forget to do this, objtool
 * may print a "call without frame pointer save/setup" warning.
 */
register unsigned long current_stack_pointer asm(_ASM_SP);
#define ASM_CALL_CONSTRAINT "+r" (current_stack_pointer)
```
* 输出部分 `"0" (ptr), "i" (sizeof(*(ptr)))`
  * `"0" (ptr)`：`ptr` 与第 `0` 个操作数匹配，也就是 `__ret_gu`，用同一个寄存器 `%rax`
* `__builtin_expect(__ret_gu, 0)` 给编译器提供分支预测的信息，表示我们期望 `__ret_gu` 的值是 `0`<sup>[[1]](#references)</sup>
* 接下来可以看看修改过后 `__get_user_1` 的实现：
* arch/x86/lib/getuser.S
```cpp
.macro check_range size:req
.if IS_ENABLED(CONFIG_X86_64)
    mov %rax, %rdx  //__val_gu = ptr
    sar $63, %rdx   //__val_gu >> 63 -> %rdx，即虚拟地址的值算术右移 63 位后存入 %rdx
    or %rdx, %rax   //改造 ptr 中虚拟地址的值，与刚才移位后的结果进行“或”运算
.else
    cmp $TASK_SIZE_MAX-\size+1, %eax
    jae .Lbad_get_user
    sbb %edx, %edx      /* array_index_mask_nospec() */ //自己减自己，清零 %edx
    and %edx, %eax      //清零 %eax
.endif
.endm

    .text
SYM_FUNC_START(__get_user_1)
    check_range size=1
    ASM_STAC  // stac 设置 EFLAGS 的 AC bit
1:  movzbl (%_ASM_AX),%edx //__val_gu = *ptr
    xor %eax,%eax          //ptr = __ret_gu = 0
    ASM_CLAC  // clac 清除 EFLAGS 的 AC bit
    RET
SYM_FUNC_END(__get_user_1)
```
* 反汇编可以看到实际汇编成的代码如下：
```c
(gdb) disassemble __get_user_1
Dump of assembler code for function __get_user_1:
   0xffffffff82277360 <+0>:     mov    %rax,%rdx
   0xffffffff82277363 <+3>:     sar    $0x3f,%rdx
   0xffffffff82277367 <+7>:     or     %rdx,%rax
   0xffffffff8227736a <+10>:    nop
   0xffffffff8227736b <+11>:    nop
   0xffffffff8227736c <+12>:    nop
   0xffffffff8227736d <+13>:    movzbl (%rax),%edx
   0xffffffff82277370 <+16>:    xor    %eax,%eax
   0xffffffff82277372 <+18>:    nop
   0xffffffff82277373 <+19>:    nop
   0xffffffff82277374 <+20>:    nop
   0xffffffff82277375 <+21>:    jmpq   0xffffffff82295740 <__x86_return_thunk>
End of assembler dump.
```
* `sar $63, %rdx; or %rdx, %rax` 首先将 `ptr` 中虚拟地址的值的 copy（`%rdx`）算术右移 `63` 位<sup>[[2]](#references)</sup>。
  * 当 `ptr` 指向一个用户态的正常地址的时候，第 `63` 位为 `0`，算术右移 `63` 位的结果为全零，`or %rdx, %rax` 的结果是 `ptr` 所表示的虚拟地址值保持不变
  * 当 `ptr` 指向一个内核态的地址的时候，第 `63` 位为 `1`，算术右移 `63` 位的结果为全一，`or %rdx, %rax` 的结果是 `ptr` 所表示的虚拟地址值变为全一
    * 随后，`movzbl (%_ASM_AX),%edx` 会因内存地址是未映射的全一而引发 `#PF`
  * 当 `ptr` 指向一个用户态 LAM 地址的时候，第 `63` 位为 `0`，算术右移 `63` 位的结果为全零，`or %rdx, %rax` 的结果是 `ptr` 所表示的虚拟地址值保持不变
    * 如果开启了 LAM，`movzbl (%_ASM_AX),%edx` CPU 会在解引用 `ptr` 的地址前忽略 SW metadata 部分的 bits
    * 如果未开启 LAM，`movzbl (%_ASM_AX),%edx` 会因内存地址为非规范形式而引发 `#GP`，异常处理的修正结果是返回 `-EFAULT`
* 修改前是用 `ptr` 和 `TASK_SIZE_MAX` 比较，因此当 `ptr` 指向一个用户态 LAM 地址的时候，**无论是否开启了 LAM** 都会因大于 `TASK_SIZE_MAX` 而直接跳转到 `bad_get_user` 而返回 `-EFAULT`
* 对于 32 位，还用原来的范围比较的方式判断有没有访问超出用户态地址范围 `cmp $TASK_SIZE_MAX-\size+1, %eax`

### 对齐检查 `EFLAGS` 中的 `AC` bit
> * **Alignment check or access control (bit `18`)**
> * 如果在 `CR0` 寄存器中设置了 `AM` bit（Alignment Mask，bit `18`），则当且仅当该标志为 `1` 时，才会启用用户模式数据访问的对齐检查。
>   * 当引用未对齐的操作数时，会生成对齐检查异常（alignment-check exception，`#AC`）。例如，位于奇数字节地址处的 *字（word）* 或位于不是四的整数倍的地址处的 *双字（doubleword）*。
>   * **对齐检查异常仅在用户模式（权限级别 3）下生成**。默认为特权级别 0 的内存引用（例如段描述符加载）即使是由在用户模式下执行的指令引起的，也不会生成此异常。
> * 对齐检查异常可用于检查数据的对齐情况。当与需要对齐所有数据的处理器交换数据时，这非常有用。
> * 解释器还可以使用对齐检查异常，通过未对齐指针来将某些指针标记为特殊指针。这消除了检查每个指针的开销，并且仅在使用时处理特殊指针。
> * 如果 `CR4` 寄存器中的 `SMAP` 位（Supervisor Mode Access Prevention）被置位，当且仅当该位为 `1` 时，才允许对用户模式页面进行显式 supervisor-mode 数据访问。请参见第 4.6 节“Access Rights”。

* 因为这段代码运行在特权级别 0，不会引发 `#AC`，这里不是借助的这个功能。

### 对异常的修正
* arch/x86/lib/getuser.S
```cpp
SYM_CODE_START_LOCAL(__get_user_handle_exception)
    ASM_CLAC
.Lbad_get_user:
    xor %edx,%edx           //%edx 清零
    mov $(-EFAULT),%_ASM_AX //-EFAULT -> %rax，%rax 会作为返回值
    RET
SYM_CODE_END(__get_user_handle_exception)

/* get_user */
    _ASM_EXTABLE(1b, __get_user_handle_exception)
```
* 标号 `1` 处如发生 `#GP` 或 `#PF` 异常，`#GP` 和 `#PF` 异常修正函数都会通过 `fixup_exception()` 函数调用 `__get_user_handle_exception` 例程进行修正
* `%rax` 会作为 `__get_user_1` 的返回值，如果 `ptr` 传入的是内核态地址会导致异常，异常的修正结果是 `get_user()` 返回 `-EFAULT`，而不是 Oops
* `_ASM_EXTABLE_UA()` 被改为了 `_ASM_EXTABLE`，异常类型由 `EX_TYPE_UACCESS` 变为 `EX_TYPE_DEFAULT`，是为了避免因传入非规范地址走到 `ex_handler_uaccess()` 而警告打印

### 这么写的原由
* 看看 Linus 怎么说的：
* [Re: [PATCHv13 05/16] x86/uaccess: Provide untagged_addr() and remove tags before address check](https://lore.kernel.org/all/CAHk-=wgKTcOx1hhWAGJ-g9_9o7xiGJ9v9n2RskBSCkaUMBxDkw@mail.gmail.com/)
> 这是恶心和错误的。
>
> 我们将 `do_get_user_call()` 作为函数调用的全部原因是因为我们“不想”在调用处执行此类操作。我们曾经将其全部内联，但通过所有 `clac/stac` 和 `access_ok` 检查，一切最终都变得如此膨胀，以至于最好将其设为具有特定调用约定的特殊函数调用。
>
> `untagged_ptr()` 应该在该 asm 函数中完成，而不是在每个调用处中完成。
>
> 现在，可悲的是，不久前，当 Christoph 清理了旧的遗留 `set_fs()` 混乱时，我们 **摆脱** 了所有这些垃圾，并且我们能够使任务限制成为一个常量（好吧，是 *两个* 常量，取决于 LA57）。因此，我们必须重新引入令人讨厌的“动态查找任务大小”。有关删除必须恢复原状，参阅 commit 47058bb54b57（“x86：remove address space overrides using set_fs()”）。
>
> 但是请参阅上面关于“maybe it should be a per-cpu variable”的内容 - 并使 `ALTERNATIVE` 的事情变得更加糟糕。
>
> 另一种替代方案可能是 **仅** 测试 `get_user/put_user` 函数中的符号位，然后只接受错误。现在我们警告非规范地址，因为这意味着有人可能错过了 `access_ok()`，但我们只是将这些 `get_user/put_user` 访问标记为特殊。
>
> 这将使这一切完全脱离关键路径。大多数其他地址屏蔽都是针对相对罕见的事情（即 `mmap/munmap`），但用户访问很 hot。
>
> Hmm?

* [Re: [PATCHv13 05/16] x86/uaccess: Provide untagged_addr() and remove tags before address check](https://lore.kernel.org/all/CAHk-=wgmGqwDD0kvjZxekU6uYR2x6+QgRHeMKy3snL2XYEzwEw@mail.gmail.com/)

> 好吧，一旦人们真正开始使用它，开销就会变得更糟。因此，静态分支是否真的是正确的做法并不明显。
>
> 也就是说，虽然我认为 `UNTAG_ADDR` 现在相当合理，但我对 `getuser.S` 和 `putuser.S` 的研究越多，我就越认为完全摆脱 `TASK_SIZE` 比较是在 x86-64 上做的正确的事情 。
>
> 它确实相当令人讨厌，不仅仅是整个 LA57 替代品，而且它还做了一个大的 64 位常量。
>
> 现在，在 32 位上，我们确实必须显式地与 `TASK_SIZE` 进行比较，但在 32 位上，我们可以只使用 `cmp` 指令的立即数，因此即使在那里，整个“load constant”也不是真正最佳的。
>
> 而在 64 位上，我们实际上只需要检查高位。
>
> 事实上，我们甚至不想 **检查** 它，因为这样我们需要做那个令人厌恶的 `array_index_mask_nospec` 事情来屏蔽它的位，所以最好使用纯算术而不在任何地方使用条件。
>
> 这正是我们在 x86-64 上可以做的事情：
>

```c
movq %rdx,%rax
shrq $63,%rax
orq %rax,%rdx
```
> 实际上会比我们现在对 `TASK_SIZE` 检查和数组索引掩码所做的要好得多（对于 putuser.S，我们在该序列中使用 `%rbx` 而不是 `%rax`）。
>
> 上述三个简单的指令将完全取代我们现在玩的所有游戏

```c
LOAD_TASK_SIZE_MINUS_N(0)
cmp %_ASM_DX,%_ASM_AX
jae bad_get_user
sbb %_ASM_DX, %_ASM_DX          /* array_index_mask_nospec() */
and %_ASM_DX, %_ASM_AX
```
> 它只会将所有内核地址全部转换为全部为一，然后肯定会出错。因此，不需要任何在现实生活中永远不会触发的条件。
>
> 在 32 位上，我们仍然需要执行旧的序列，但我们将替换

```c
LOAD_TASK_SIZE_MINUS_N(0)
cmp %_ASM_DX,%_ASM_AX
```

> 为更简单的

```c
cmp $TASK_SIZE_MAX-(n),%_ASM_AX
```

> 因为我们执行加载立即数的唯一原因是没有 64 位的立即数比较指令。
>
> 一旦我们不对 `TASK_SIZE` 进行测试，对 `UNTAG_ADDR` 的需求就消失了，所以现在 LAM 也更好了。
>
> 换句话说，我们实际上可以改进当前的代码 *并* 简化 LAM 情况。双赢。
>
> 不管怎样，我并不讨厌你发布的补丁版本，但我确实认为，首先让 LAM 没 *有* 这个问题的双赢可能是更好的选择。
>
> “移位 `63` 并按位或”技巧确实需要的一件事是 `getuser/putuser` 的 `_ASM_EXTABLE_UA()` 必须有一个额外的注释来关闭在 `ex_handler_uaccess()` 中的
>

```c
WARN_ONCE(trapnr == X86_TRAP_GP, "General protection fault in user access. Non-canonical address?");
```

> 用户现在可以通过提供高位清零的非规范地址来引发 `GP` 异常。因此，对于这些情况，我们可能只需要一个新的 `EX_TYPE_*`，但这看起来仍然相当简单。
>
> Hmm?

* 在 v14 版的时候，Peter Z 对此提出[质疑](https://lore.kernel.org/all/CAHk-=wjtUnjYSttxsYtXu2VtsTeBjoaL8Je8cjWqFMJzw1-MhQ@mail.gmail.com/)：

> 新逻辑基本上将每个内核地址压缩为 `-1L` —— 一个已知的未映射地址，但是在 `{get,put}_user()` 中获取该地址仍然失败，对吧？
>
> 我们曾经对 `TASK_SIZE_MAX` 之外（的地址）手动分支到 `bad_get_user`，现在我们依赖 `#GP`。
>
> 那么为什么（出现异常时）要沉默呢？

* Linus 给他的回复和 Peter Z 随后的回复如下：

>> 我们不会将其静默 - 对于变成全 `1` 地址的内核地址，`_ASM_EXTABLE()` 仍会因缺页而导致 `-EFAULT`。
>>
>> 但这里的问题并不是高 bit 设置的情况。
>>
>
> 是的，显式的 `bad_get_user` 跳转不会打印该消息，现在删除 `_UA` 后也不会打印该消息（我似乎刚刚脑回路接错了）。
>
>> 问题是一个非规范的“正”地址。
>>
>> 针对 `TASK_SIZE_MAX` 进行测试将在访问之前捕获非规范地址，并且我们将返回 `-EFAULT`。
>>
>> 但现在我们不再针对 `TASK_SIZE_MAX` 进行测试，非规范访问将导致 `GP` 错误，而 **那个** 消息正是我们想要静默的。
>>
>
> 是的，但我认为我们明确允许这些地址是因为启用了 LAM，我们实际上会接受这些地址。
>
>> 我们仍然会返回 `-EFAULT`，当然，我们只是解决了

```c
WARN_ONCE(trapnr == X86_TRAP_GP, "General protection fault in user access. Non-canonical address?");
```

>> 的问题，由于来自于地址限制不再那么精确。

> 啊，确实如此，（当访问非规范的“正”地址时，如果不删除 `_UA`）对于 `!LAM`，我们现在将打印之前不会打印的消息（整个 `TASK_SIZE_MAX+` 范围）。
>
> 好的，同意。

* 最后 Linus 回复如下：

> 是的。
>
> 我们可以完全删除该警告，但是当人们做了坏事时（例如，用不检查的“`__copy_from_user()`”和友函数），它对于未被“`access_ok()`”未捕获的，却被 syzbot 捕获的随机用户地址很有用 。
>
> 我不确定这个警告还有多少价值——对于 `get_user()` 和 `put_user()` 本身来说，它对我们没有任何好处，因为根据定义 *它们* 会进行范围检查。Christoph 搞定了 `set_fs()` 模型简化了我们的许多用户地址检查。
>
> 但我认为更容易保留现有的关于“你如何在此处获得非规范地址”的警告以供其他用户访问，而只是让 `get/put_user()` 使用不执行此操作的 `_ASM_EXTABLE()` 版本。

* 至此，这一块的改动就大体定下来了，但后面 Linus 在 merge 这组 patch set 的时候针对上面的警告信息的出现条件又做了些修订，后面再讲。

## [PATCH 02/19] x86: Allow atomic MM_CONTEXT flags setting
* 到目前为止，不需要在 `mm_context_t::flags` 中对 MM 上下文标志进行原子设置。这些标志在 `exec` 早期设置，之后就不再改变。
* LAM 启用需要原子标志设置。即将到来的标志 `MM_CONTEXT_FORCE_TAGGED_SVA` 可以在存在多个线程的进程生命周期的后期进行设置。
* 将域转换为 `unsigned long` 并使用 `__set_bit()` 和 `test_bit()` 进行 `MM_CONTEXT_*` 访问。
* 没有功能变化。

## [PATCH 03/19] x86: CPUID and CR3/CR4 flags for Linear Address Masking
* 枚举 Linear Address Masking 并提供 `CR3` 和 `CR4` 标志的定义。
* 新的 `CONFIG_ADDRESS_MASKING` 选项启用内核中的功能支持。
* 处理器支持的 LAM 由 CPUID feature flag `CPUID.(EAX=0x07，ECX=0x01):EAX.LAM[bit 26]` 枚举。
```diff
diff --git a/arch/x86/include/asm/cpufeatures.h b/arch/x86/include/asm/cpufeatures.h
index 73c9672c123b..353b054812de 100644
--- a/arch/x86/include/asm/cpufeatures.h
+++ b/arch/x86/include/asm/cpufeatures.h
@@ -321,6 +321,7 @@
 #define X86_FEATURE_LKGS       (12*32+18) /* "" Load "kernel" (userspace) GS */
 #define X86_FEATURE_AMX_FP16       (12*32+21) /* "" AMX fp16 Support */
 #define X86_FEATURE_AVX_IFMA            (12*32+23) /* "" Support for VPMADD52[H,L]UQ */
+#define X86_FEATURE_LAM            (12*32+26) /* Linear Address Masking */
```
* LAM 的启用和配置由控制寄存器中的以下新位控制：
  * `CR3[62]`（**LAM_U48**）
  * `CR3[61]`（**LAM_U57**）
  * `CR4[28]`（**LAM_SUP**）
```diff
diff --git a/arch/x86/include/uapi/asm/processor-flags.h b/arch/x86/include/uapi/asm/processor-flags.h
index c47cc7f2feeb..d898432947ff 100644
--- a/arch/x86/include/uapi/asm/processor-flags.h
+++ b/arch/x86/include/uapi/asm/processor-flags.h
@@ -82,6 +82,10 @@
 #define X86_CR3_PCID_BITS  12
 #define X86_CR3_PCID_MASK  (_AC((1UL << X86_CR3_PCID_BITS) - 1, UL))

+#define X86_CR3_LAM_U57_BIT    61 /* Activate LAM for userspace, 62:57 bits masked */
+#define X86_CR3_LAM_U57        _BITULL(X86_CR3_LAM_U57_BIT)
+#define X86_CR3_LAM_U48_BIT    62 /* Activate LAM for userspace, 62:48 bits masked */
+#define X86_CR3_LAM_U48        _BITULL(X86_CR3_LAM_U48_BIT)
 #define X86_CR3_PCID_NOFLUSH_BIT 63 /* Preserve old PCID */
 #define X86_CR3_PCID_NOFLUSH    _BITULL(X86_CR3_PCID_NOFLUSH_BIT)

@@ -132,6 +136,8 @@
 #define X86_CR4_PKE        _BITUL(X86_CR4_PKE_BIT)
 #define X86_CR4_CET_BIT        23 /* enable Control-flow Enforcement Technology */
 #define X86_CR4_CET        _BITUL(X86_CR4_CET_BIT)
+#define X86_CR4_LAM_SUP_BIT    28 /* LAM for supervisor pointers */
+#define X86_CR4_LAM_SUP        _BITUL(X86_CR4_LAM_SUP_BIT)
```

## [PATCH 04/19] x86/mm: Handle LAM on context switch
* 用于用户空间指针的 Linear Address Masking 模式编码到 `CR3` 的 bits 中。
* 该模式是按 per-process 选择的，并存储在 `mm_context_t` 中。
* `switch_mm_irqs_off()` 现在依据所选的 LAM 模式并相应地构造 `CR3`。
* 激活的 LAM 模式记录在 `tlb_state` 中。
---
### 结构体中新增域以及新增 helpers
* `struct mm_context_t` 中新增域 `lam_cr3_mask`
```diff
diff --git a/arch/x86/include/asm/mmu.h b/arch/x86/include/asm/mmu.h
index efa3eaee522c..22fc9fbf1d0a 100644
--- a/arch/x86/include/asm/mmu.h
+++ b/arch/x86/include/asm/mmu.h
@@ -42,6 +42,11 @@ typedef struct {
    unsigned long flags;
 #endif

+#ifdef CONFIG_ADDRESS_MASKING
+   /* Active LAM mode:  X86_CR3_LAM_U48 or X86_CR3_LAM_U57 or 0 (disabled) */
+   unsigned long lam_cr3_mask;
+#endif
+
    struct mutex lock;
    void __user *vdso;          /* vdso base address */
    const struct vdso_image *vdso_image;    /* vdso image in use */
```
* `mm_context_t` 类型是 `struct mm_struct` 的域 `context` 的类型，用以描述体系结构特定的，不在 `mm_struct` 中的状态
* 新增两个操作它的 helpers：
```cpp
#ifdef CONFIG_ADDRESS_MASKING
static inline unsigned long mm_lam_cr3_mask(struct mm_struct *mm)
{
    return mm->context.lam_cr3_mask;
}

static inline void dup_lam(struct mm_struct *oldmm, struct mm_struct *mm)
{
    mm->context.lam_cr3_mask = oldmm->context.lam_cr3_mask;
    mm->context.untag_mask = oldmm->context.untag_mask;
}
#endif
```
* 父进程 fork 子进程时，子进程继承父进程 LAM 相关的属性：
```diff
@@ -169,6 +192,7 @@ static inline int arch_dup_mmap(struct mm_struct *oldmm, struct mm_struct *mm)
 {
    arch_dup_pkeys(oldmm, mm);
    paravirt_arch_dup_mmap(oldmm, mm);
+   dup_lam(oldmm, mm);
    return ldt_dup_context(oldmm, mm);
 }
```
* `struct tlb_state` 用于定义 per-CPU 变量 `cpu_tlbstate`
  * arch/x86/mm/init.c
```cpp
__visible DEFINE_PER_CPU_ALIGNED(struct tlb_state, cpu_tlbstate) = {
    .loaded_mm = &init_mm,
    .next_asid = 1,
    .cr4 = ~0UL,    /* fail hard if we screw up cr4 shadow initialization */
};
```
* `struct tlb_state` 新增域 `u8 lam` 用于记录 LAM 的激活状态，怎么用看对应的 helpers。
```diff
diff --git a/arch/x86/include/asm/tlbflush.h b/arch/x86/include/asm/tlbflush.h
index cda3118f3b27..e8b47f57bd4a 100644
--- a/arch/x86/include/asm/tlbflush.h
+++ b/arch/x86/include/asm/tlbflush.h
@@ -101,6 +102,16 @@ struct tlb_state {
     */
    bool invalidate_other;

+#ifdef CONFIG_ADDRESS_MASKING
+   /*
+    * Active LAM mode.
+    *
+    * X86_CR3_LAM_U57/U48 shifted right by X86_CR3_LAM_U57_BIT or 0 if LAM
+    * disabled.
+    */
+   u8 lam;
+#endif
+
    /*
     * Mask that contains TLB_NR_DYN_ASIDS+1 bits to indicate
     * the corresponding user PCID needs a flush next time we
```
* 相应地，增加两个 helpers：
  * set 的时候将 `mm->context.lam_cr3_mask` 中的 `X86_CR3_LAM_U57/U48` bit 右移 `X86_CR3_LAM_U57_BIT`（`61`）记录在 `cpu_tlbstate.lam`
```cpp
#ifdef CONFIG_ADDRESS_MASKING
static inline  u64 tlbstate_lam_cr3_mask(void)
{
    u64 lam = this_cpu_read(cpu_tlbstate.lam);

    return lam << X86_CR3_LAM_U57_BIT;
}

static inline void set_tlbstate_lam_mode(struct mm_struct *mm)
{
    this_cpu_write(cpu_tlbstate.lam,
               mm->context.lam_cr3_mask >> X86_CR3_LAM_U57_BIT);
    this_cpu_write(tlbstate_untag_mask, mm->context.untag_mask); //后面再说
}
#endif
```
### 构建 `CR3` 函数的修改
* 用于构建进程 `CR3` 的函数 `build_cr3()` 增加入参 `lam`，并让 `build_cr3_noflush()` 改为调用它
```diff
diff --git a/arch/x86/mm/tlb.c b/arch/x86/mm/tlb.c
index 92d73ccede70..724f98d0e10f 100644
--- a/arch/x86/mm/tlb.c
+++ b/arch/x86/mm/tlb.c
@@ -154,26 +154,30 @@ static inline u16 user_pcid(u16 asid)
-static inline unsigned long build_cr3(pgd_t *pgd, u16 asid)
+static inline unsigned long build_cr3(pgd_t *pgd, u16 asid, unsigned long lam)
 {
+   unsigned long cr3 = __sme_pa(pgd) | lam;
+
    if (static_cpu_has(X86_FEATURE_PCID)) {
-       return __sme_pa(pgd) | kern_pcid(asid);
+       VM_WARN_ON_ONCE(asid > MAX_ASID_AVAILABLE);
+       cr3 |= kern_pcid(asid);
    } else {
        VM_WARN_ON_ONCE(asid != 0);
-       return __sme_pa(pgd);
    }
+
+   return cr3;
 }

-static inline unsigned long build_cr3_noflush(pgd_t *pgd, u16 asid)
+static inline unsigned long build_cr3_noflush(pgd_t *pgd, u16 asid,
+                         unsigned long lam)
 {
-   VM_WARN_ON_ONCE(asid > MAX_ASID_AVAILABLE);
    /*
     * Use boot_cpu_has() instead of this_cpu_has() as this function
     * might be called during early boot. This should work even after
     * boot because all CPU's the have same capabilities:
     */
    VM_WARN_ON_ONCE(!boot_cpu_has(X86_FEATURE_PCID));
-   return __sme_pa(pgd) | kern_pcid(asid) | CR3_NOFLUSH;
+   return build_cr3(pgd, asid, lam) | CR3_NOFLUSH;
 }
```
* 毫无疑问，调用它们的函数都会收到影响。首先是 `load_new_mm_cr3()`，它仅被 `switch_mm_irqs_off()` 调用，根据入参构造好欲写入 `CR3` 的值，然后写入 `CR3`
```cpp
static void load_new_mm_cr3(pgd_t *pgdir, u16 new_asid, unsigned long lam,
                bool need_flush)
{
    unsigned long new_mm_cr3;

    if (need_flush) {
        invalidate_user_asid(new_asid);
        new_mm_cr3 = build_cr3(pgdir, new_asid, lam);
    } else {
        new_mm_cr3 = build_cr3_noflush(pgdir, new_asid, lam);
    }

    /*
     * Caution: many callers of this function expect
     * that load_cr3() is serializing and orders TLB
     * fills with respect to the mm_cpumask writes.
     */
    write_cr3(new_mm_cr3);
}
```
### 进程切换函数 `switch_mm_irqs_off()` 的修改
* `switch_mm_irqs_off()` 大多用于进程上下文切换时，其他还有一些场景
```diff
diff --git a/arch/x86/mm/tlb.c b/arch/x86/mm/tlb.c
index 92d73ccede70..724f98d0e10f 100644
--- a/arch/x86/mm/tlb.c
+++ b/arch/x86/mm/tlb.c
@@ -491,6 +496,7 @@ void switch_mm_irqs_off(struct mm_struct *prev, struct mm_struct *next,
 {
    struct mm_struct *real_prev = this_cpu_read(cpu_tlbstate.loaded_mm);
    u16 prev_asid = this_cpu_read(cpu_tlbstate.loaded_mm_asid);
+   unsigned long new_lam = mm_lam_cr3_mask(next); //得到新进程的 CR3 LAM 掩码
    bool was_lazy = this_cpu_read(cpu_tlbstate_shared.is_lazy);
    unsigned cpu = smp_processor_id();
    u64 next_tlb_gen;
@@ -520,7 +526,8 @@ void switch_mm_irqs_off(struct mm_struct *prev, struct mm_struct *next,
     * isn't free.
     */
 #ifdef CONFIG_DEBUG_VM
-   if (WARN_ON_ONCE(__read_cr3() != build_cr3(real_prev->pgd, prev_asid))) {
+   if (WARN_ON_ONCE(__read_cr3() != build_cr3(real_prev->pgd, prev_asid, //这个判断是要看 CR3 里的值和当前进程期望的 CR3 值是否相符，
+                          tlbstate_lam_cr3_mask()))) {                   //因此 LAM 的值来自 cpu_tlbstate.lam
        /*
         * If we were to BUG here, we'd be very likely to kill
         * the system so hard that we don't see the call trace.
@@ -552,9 +559,15 @@ void switch_mm_irqs_off(struct mm_struct *prev, struct mm_struct *next,
     * instruction.
     */
    if (real_prev == next) {
+       /* Not actually switching mm's */
        VM_WARN_ON(this_cpu_read(cpu_tlbstate.ctxs[prev_asid].ctx_id) !=
               next->context.ctx_id);

+       /*
+        * If this races with another thread that enables lam, 'new_lam'
+        * might not match tlbstate_lam_cr3_mask().
+        */
+
        /*
         * Even in lazy TLB mode, the CPU should stay set in the
         * mm_cpumask. The TLB shootdown code can figure out from
@@ -622,15 +635,16 @@ void switch_mm_irqs_off(struct mm_struct *prev, struct mm_struct *next,
        barrier();
    }

+   set_tlbstate_lam_mode(next); //确实需要切换进程了，把新进程的 mm->context.lam_cr3_mask 转换后更新到 cpu_tlbstate.lam
    if (need_flush) {
        this_cpu_write(cpu_tlbstate.ctxs[new_asid].ctx_id, next->context.ctx_id);
        this_cpu_write(cpu_tlbstate.ctxs[new_asid].tlb_gen, next_tlb_gen);
-       load_new_mm_cr3(next->pgd, new_asid, true);
+       load_new_mm_cr3(next->pgd, new_asid, new_lam, true); //更新 CR3，带上新进程的 CR3 LAM 掩码，不带 CR3_NOFLUSH 标志

        trace_tlb_flush(TLB_FLUSH_ON_TASK_SWITCH, TLB_FLUSH_ALL);
    } else {
        /* The new ASID is already up to date. */
-       load_new_mm_cr3(next->pgd, new_asid, false);
+       load_new_mm_cr3(next->pgd, new_asid, new_lam, false); //更新 CR3，带上新进程的 CR3 LAM 掩码，带 CR3_NOFLUSH 标志

        trace_tlb_flush(TLB_FLUSH_ON_TASK_SWITCH, 0);
    }
```
* 以上改动其实很好理解，`CR3` 在硬件上是 per-CPU 的，然而每个进程都有自己的 `CR3` 的值，当进程调度到 CPU 上运行时载入。LAM 的开关在 `CR3` 里，因此进程切换的时候需要将 per-task 的 `mm->context.lam_cr3_mask` 同步到 per-CPU 的 `cpu_tlbstate.lam`，然后在更新 `CR3` 时也把相应的 bits 设置上

### CPU 初始化函数 `initialize_tlbstate_and_flush()` 的修改
* `initialize_tlbstate_and_flush()` 被 `cpu_init()` 和 `fix_processor_context()` 调用，用于 CPU 热插拔、系统休眠唤醒、kexec 等需要回复处理器上下文的场景，它自己的注释如下：
  * 重新初始化 CPU 时调用此函数。它修复了以下潜在问题：
    - ASID 与 `cpu_tlbstate` 认为的不同（很可能是因为 CPU 被取下并恢复时 `CR3` 的 `PCID` 位被清除。CPU 热插拔可以做到这一点）。
    - TLB 在与非活动 ASID 对应的插槽中包含垃圾。
    - CPU 去很远的地方吃午餐，以至于它可能错过了 TLB 刷新。
* 那么对于它的改动就是初始化的时候，期望 LAM 是关闭的，且会关闭它
```diff
@@ -691,6 +705,10 @@ void initialize_tlbstate_and_flush(void)
    /* Assert that CR3 already references the right mm. */
    WARN_ON((cr3 & CR3_ADDR_MASK) != __pa(mm->pgd));

+   /* LAM expected to be disabled */
+   WARN_ON(cr3 & (X86_CR3_LAM_U48 | X86_CR3_LAM_U57));
+   WARN_ON(mm_lam_cr3_mask(mm));
+
    /*
     * Assert that CR4.PCIDE is set if needed.  (CR4.PCIDE initialization
     * doesn't work like other CR4 bits because it can only be set from
@@ -699,8 +717,8 @@ void initialize_tlbstate_and_flush(void)
    WARN_ON(boot_cpu_has(X86_FEATURE_PCID) &&
        !(cr4_read_shadow() & X86_CR4_PCIDE));

-   /* Force ASID 0 and force a TLB flush. */
-   write_cr3(build_cr3(mm->pgd, 0));
+   /* Disable LAM, force ASID 0 and force a TLB flush. */
+   write_cr3(build_cr3(mm->pgd, 0, 0));

    /* Reinitialize tlbstate. */
    this_cpu_write(cpu_tlbstate.last_user_mm_spec, LAST_USER_MM_INIT);
@@ -708,6 +726,7 @@ void initialize_tlbstate_and_flush(void)
    this_cpu_write(cpu_tlbstate.next_asid, 1);
    this_cpu_write(cpu_tlbstate.ctxs[0].ctx_id, mm->context.ctx_id);
    this_cpu_write(cpu_tlbstate.ctxs[0].tlb_gen, tlb_gen);
+   set_tlbstate_lam_mode(mm);

    for (i = 1; i < TLB_NR_DYN_ASIDS; i++)
        this_cpu_write(cpu_tlbstate.ctxs[i].ctx_id, 0);
```
### 快速获取 CR3 函数的修改
* 可以从进程上下文中使用该函数来得到 `CR3` 的值，而不需要执行 `__read_cr3()`（慢的方式）。
* 意图是用于像 KVM 这样偷偷更改 `CR3` 并需要恢复它的代码。它需要非常小心地使用。
  * arch/x86/mm/tlb.c
```cpp
/*
 * This can be used from process context to figure out what the value of
 * CR3 is without needing to do a (slow) __read_cr3().
 *
 * It's intended to be used for code like KVM that sneakily changes CR3
 * and needs to restore it.  It needs to be used very carefully.
 */
unsigned long __get_current_cr3_fast(void)
{
    unsigned long cr3 =
        build_cr3(this_cpu_read(cpu_tlbstate.loaded_mm)->pgd,
              this_cpu_read(cpu_tlbstate.loaded_mm_asid),
              tlbstate_lam_cr3_mask()); //从 cpu_tlbstate.lam 获取信息，转换得到 LAM 在 CR3 中的 bit

    /* For now, be very restrictive about when this can be called. */
    VM_WARN_ON(in_nmi() || preemptible());

    VM_BUG_ON(cr3 != __read_cr3());
    return cr3;
}
```

## [PATCH 05/19] mm: Introduce untagged_addr_remote()
* `untagged_addr()` 从地址中删除 tags/metadata 并将其转换为规范形式。该 helper 在 arm64 和 sparc 上实现。他们都根据全局规则进行 untagging。
* 然而，x86 上的 Linear Address Masking（LAM）引入了用于 untagging 的 per-process 设置。因此，`untagged_addr()` 现在仅适用于 untagging 当前进程的地址。
* 当地址指向远程进程时，必须使用新的 helper `untagged_addr_remote()`。它需要获取目标 mm 的 `mmap` 锁。
---
* 这个 patch 看起来改动很多，但做的事情基本上以上 commit message 能总结
* 所谓“远程进程”指的就是非当前进程，该 patch 把现有的远程进程中原来使用 `untagged_addr()` 的地方都换成 `untagged_addr_remote()`，用于本地的 `untagged_addr()` 在随后的 patch 中加入
* 通用的 `untagged_addr_remote()` 和 `untagged_addr()` 是这样的
  * include/linux/uaccess.h 
```cpp
/*
 * Architectures that support memory tagging (assigning tags to memory regions,
 * embedding these tags into addresses that point to these memory regions, and
 * checking that the memory and the pointer tags match on memory accesses)
 * redefine this macro to strip tags from pointers.
 *
 * Passing down mm_struct allows to define untagging rules on per-process
 * basis.
 *
 * It's defined as noop for architectures that don't support memory tagging.
 */
#ifndef untagged_addr
#define untagged_addr(addr) (addr)
#endif

#ifndef untagged_addr_remote
#define untagged_addr_remote(mm, addr)  ({      \
    mmap_assert_locked(mm);             \
    untagged_addr(addr);                \
})
#endif
```

## [PATCH 06/19] x86/uaccess: Provide untagged_addr() and remove tags before address check
* `untagged_addr()` 是 core-mm 用来剥离 tag 位并根据当前线程的规则获取规范形状的地址的 helper。它只处理用户空间地址。
* Untagging 掩码存储在 per-CPU 变量中，并在 task 的上下文切换时设置。
* 不得将 tags 包含在检查是否可以访问用户空间地址的过程中。在 `access_ok()` 中去除 tags。
---
### Per-process 和 per-CPU 的 untagging 掩码
* 和 `lam_cr3_mask` 一样，在 `struct mm_context_t` 中新增域 `untag_mask`
```diff
diff --git a/arch/x86/include/asm/mmu.h b/arch/x86/include/asm/mmu.h
index 22fc9fbf1d0a..9cac8c45a647 100644
--- a/arch/x86/include/asm/mmu.h
+++ b/arch/x86/include/asm/mmu.h
@@ -45,6 +45,9 @@ typedef struct {
 #ifdef CONFIG_ADDRESS_MASKING
    /* Active LAM mode:  X86_CR3_LAM_U48 or X86_CR3_LAM_U57 or 0 (disabled) */
    unsigned long lam_cr3_mask;
+
+   /* Significant bits of the virtual address. Excludes tag bits. */
+   u64 untag_mask;
 #endif

    struct mutex lock;
```
* 在 `dup_lam()` 的时候也复制父进程的 `untag_mask` 到子进程，代码上面已经贴过了
* 新增 helper `mm_reset_untag_mask()` 用于重置 `untag_mask`
```cpp
static inline void mm_reset_untag_mask(struct mm_struct *mm)
{
    mm->context.untag_mask = -1UL;
}
```
* 它的两个调用者：
  * `init_new_context()` 用于创建新进程的场景，对于 x86 主要由 `mm_init()` 调用
  * `arch_setup_new_exec()` 用于加载可执行文件的场景
```cpp
arch/x86/include/asm/mmu_context.h|157| <<init_new_context>> mm_reset_untag_mask(mm);
arch/x86/kernel/process.c|397| <<arch_setup_new_exec>> mm_reset_untag_mask(current->mm);
```
* 可以看看创建新进程场景中重置和复制 `untag_mask` 的路径
```c
copy_process()
-> copy_mm()
   -> dup_mm(tsk, current->mm)
      -> mm = allocate_mm();
      -> memcpy(mm, oldmm, sizeof(*mm))
      -> mm_init(mm, tsk, mm->user_ns)
         -> init_new_context(p, mm)
            -> mm_reset_untag_mask(mm)
                  mm->context.untag_mask = -1UL;
      -> dup_mmap(mm, oldmm);
         -> arch_dup_mmap(oldmm, mm)
            -> dup_lam(oldmm, mm)
                  mm->context.lam_cr3_mask = oldmm->context.lam_cr3_mask;
                  mm->context.untag_mask = oldmm->context.untag_mask;
```
* 声明和定义 per-CPU 变量 `tlbstate_untag_mask`
  * arch/x86/mm/init.c
```cpp
#ifdef CONFIG_ADDRESS_MASKING
DEFINE_PER_CPU(u64, tlbstate_untag_mask);
EXPORT_PER_CPU_SYMBOL(tlbstate_untag_mask);
#endif
```
### x86 的 `untagged_addr()` 相关的实现
* 接着，就引入了 x86 的 `untagged_addr()` 相关的实现，我们直接看最终版：
  * arch/x86/include/asm/uaccess_64.h
```cpp
#ifdef CONFIG_ADDRESS_MASKING
/*
 * Mask out tag bits from the address.
 */
static inline unsigned long __untagged_addr(unsigned long addr)
{
    /*
     * Refer tlbstate_untag_mask directly to avoid RIP-relative relocation
     * in alternative instructions. The relocation gets wrong when gets
     * copied to the target place.
     */
    asm (ALTERNATIVE("",
             "and %%gs:tlbstate_untag_mask, %[addr]\n\t", X86_FEATURE_LAM)
         : [addr] "+r" (addr) : "m" (tlbstate_untag_mask));

    return addr;
}

#define untagged_addr(addr) ({                  \
    unsigned long __addr = (__force unsigned long)(addr);       \
    (__force __typeof__(addr))__untagged_addr(__addr);      \
})
```
* `[addr] "+r" (addr)` 输出限定符
  * `[addr]`：在汇编代码中的名字为 `addr`
  * `"+r"`：`+` 表示参数可读写，`r` 前面说过是使用通用寄存器
  * `(addr)`：使用在 C 代码中的变量 `addr` 所在的寄存器
* `"m" (tlbstate_untag_mask)` 输入限定符引用内存地址 `tlbstate_untag_mask`
* `ALTERNATIVE("", "and %%gs:tlbstate_untag_mask, %[addr]\n\t", X86_FEATURE_LAM)` 表示当具备 `X86_FEATURE_LAM` flag 时，此处的空指令 `""` 被替换为 `"and %%gs:tlbstate_untag_mask, %[addr]\n\t"`
  * `"and %%gs:tlbstate_untag_mask, %[addr]\n\t"` 表示变量 `addr` 所用寄存器的值与 per-CPU 的 `tlbstate_untag_mask` 进行 `&` 操作，结果存入 `addr` 所用寄存器
* Remote 的 untagging 比较简单，核心操作就是 `addr & (mm)->context.untag_mask` 就可以了，但要求必须持有 mmap 锁
```cpp
static inline unsigned long __untagged_addr_remote(struct mm_struct *mm,
                           unsigned long addr)
{
    mmap_assert_locked(mm);
    return addr & (mm)->context.untag_mask;
}
#define untagged_addr_remote(mm, addr)  ({              \
    unsigned long __addr = (__force unsigned long)(addr);       \
    (__force __typeof__(addr))__untagged_addr_remote(mm, __addr);   \
})

#endif
```

### `access_ok()` 时不包含 tag
* 该 patch 还对 `access_ok()` 进行如下修改，意图不将 tags 包含在检查是否可以访问用户空间地址的过程中
```diff
diff --git a/arch/x86/include/asm/uaccess.h b/arch/x86/include/asm/uaccess.h
index 1cc756eafa44..c79ebdbd6356 100644
--- a/arch/x86/include/asm/uaccess.h
+++ b/arch/x86/include/asm/uaccess.h
@@ -38,10 +73,10 @@ static inline bool pagefault_disabled(void);
  * Return: true (nonzero) if the memory block may be valid, false (zero)
  * if it is definitely invalid.
  */
-#define access_ok(addr, size)                  \
+#define access_ok(addr, size)                      \
 ({                                 \
    WARN_ON_IN_IRQ();                       \
-   likely(__access_ok(addr, size));                \
+   likely(__access_ok(untagged_addr(addr), size));         \
 })

 #include <asm-generic/access_ok.h>
```
* 该修改后来被 Linus 在如下 commit 中移除
```c
commit 6014bc27561f2cc63e0acc18adbc4ed810834e32
Author: Linus Torvalds <torvalds@linux-foundation.org>
Date:   Fri Apr 28 12:55:10 2023 -0700

    x86-64: make access_ok() independent of LAM
```

## [PATCH 07/19] x86/mm: Reduce untagged_addr() overhead for systems without LAM
* 使用 alternatives 来减少 `untagged_addr()` 开销。
* 这个 patch 的修改后面都被 Linus 的修改覆盖了，略过

## [PATCH 08/19] x86/mm: Provide arch_prctl() interface for LAM
* 添加一些 `arch_prctl()` handlers：
  - `ARCH_ENABLE_TAGGED_ADDR` 启用 LAM。参数是所需的 tag bits。它被四舍五入到可以提供它的最接近的 LAM 模式。目前仅支持 `LAM_U57`，具有 6 个 tag bits。
  - `ARCH_GET_UNTAG_MASK` 返回 untag 掩码。它可以指示 tag bits 位于地址中的位置。
  - `ARCH_GET_MAX_TAG_BITS` 返回用户可以请求的最大 tag bits。如果不支持 LAM，则为零。
---
* `task_struct->mm->context.flags` 含有 `MM_CONTEXT_LOCK_LAM` 标志时，不允许修改进程的 LAM 模式
* arch/x86/include/asm/mmu.h
```cpp
/* Do not allow changing LAM mode */
#define MM_CONTEXT_LOCK_LAM     2
```
* arch/x86/include/uapi/asm/prctl.h
```cpp
#define ARCH_GET_UNTAG_MASK     0x4001
#define ARCH_ENABLE_TAGGED_ADDR     0x4002
#define ARCH_GET_MAX_TAG_BITS       0x4003
```
* Fork 的时候 `copy_process()` 会调用 `copy_thread()`，该函数主要拷贝父进程的 `struct task_struct` 的 `struct thread_struct thread;` 成员
  * 进程切换时，该域的内容主要在 `__switch_to()` 函数切换，而不是 `__switch_to_asm` 汇编例程，该汇编例程主要切换一些通用寄存器和栈寄存器
* 子进程如果是复制父进程的虚拟内存空间的话，不允许子进程修改 LAM 模式
  * arch/x86/kernel/process.c
```diff
diff --git a/arch/x86/kernel/process.c b/arch/x86/kernel/process.c
index bbc8c4c6e360..50d950771371 100644
--- a/arch/x86/kernel/process.c
+++ b/arch/x86/kernel/process.c
@@ -163,6 +163,9 @@ int copy_thread(struct task_struct *p, const struct kernel_clone_args *args)

    savesegment(es, p->thread.es);
    savesegment(ds, p->thread.ds);
+
+   if (p->mm && (clone_flags & (CLONE_VM | CLONE_VFORK)) == CLONE_VM)
+       set_bit(MM_CONTEXT_LOCK_LAM, &p->mm->context.flags);
 #else
    p->thread.sp0 = (unsigned long) (childregs + 1);
    savesegment(gs, p->thread.gs);
```
* 对于 kernel 而言，只关心启用的是哪个 LAM 模式，对应屏蔽的位是哪些，屏蔽位里的 metadata 的内容不关心
  * 所以有 `mm->context.untag_mask = ~GENMASK(62, 57)`，仅存 per-task 的屏蔽位的掩码
* arch/x86/kernel/process_64.c
```cpp
#ifdef CONFIG_ADDRESS_MASKING

#define LAM_U57_BITS 6
//参数是所需的 tag bits。它被四舍五入到可以提供它的最接近的 LAM 模式。目前仅支持 LAM_U57，具有 6 个 tag bits
static int prctl_enable_tagged_addr(struct mm_struct *mm, unsigned long nr_bits)
{
    if (!cpu_feature_enabled(X86_FEATURE_LAM))
        return -ENODEV;
    //mm 不匹配，不能修改其他进程的 LAM 设置
    /* PTRACE_ARCH_PRCTL */
    if (current->mm != mm)
        return -EINVAL;
    //PASID 有效意味着不能使用 LAM，除非 tagged SVA 被强制打开
    if (mm_valid_pasid(mm) &&
        !test_bit(MM_CONTEXT_FORCE_TAGGED_SVA, &mm->context.flags))
        return -EINVAL;
    //拿 mmap 写锁，可被信号杀死
    if (mmap_write_lock_killable(mm))
        return -EINTR;
    //如果 LAM 模式已经锁定，返回正在忙的错误类型
    if (test_bit(MM_CONTEXT_LOCK_LAM, &mm->context.flags)) {
        mmap_write_unlock(mm);
        return -EBUSY;
    }
    //tag bits 为零，无效参数
    if (!nr_bits) {
        mmap_write_unlock(mm);
        return -EINVAL;
    } else if (nr_bits <= LAM_U57_BITS) { //LAM_U57 有 6 个 tag bits
        mm->context.lam_cr3_mask = X86_CR3_LAM_U57; //设置 LAM_U57 在 CR3 中的开关位
        mm->context.untag_mask =  ~GENMASK(62, 57); //因为是 untag_mask，所以掩码部分为零
    } else {
        mmap_write_unlock(mm);
        return -EINVAL; // > 6 个 tag bits，无效参数
    }
    //当即更新 CR3，开启 LAM
    write_cr3(__read_cr3() | mm->context.lam_cr3_mask);
    set_tlbstate_lam_mode(mm); //更新 per-CPU 的 cpu_tlbstate.lam 和 tlbstate_untag_mask
    set_bit(MM_CONTEXT_LOCK_LAM, &mm->context.flags); //一旦开启，即锁定 LAM 模式
    //释放 mmap 写锁
    mmap_write_unlock(mm);

    return 0;
}
#endif

long do_arch_prctl_64(struct task_struct *task, int option, unsigned long arg2)
{
    int ret = 0;

    switch (option) {
...
#ifdef CONFIG_ADDRESS_MASKING
    case ARCH_GET_UNTAG_MASK: //返回 untag 掩码，就存在 per-task 的 untag_mask 里
        return put_user(task->mm->context.untag_mask,
                (unsigned long __user *)arg2);
    case ARCH_ENABLE_TAGGED_ADDR: //启用 LAM
        return prctl_enable_tagged_addr(task->mm, arg2);
    case ARCH_FORCE_TAGGED_SVA: //强制启用 tagged SVA
        if (current != task)
            return -EINVAL;
        set_bit(MM_CONTEXT_FORCE_TAGGED_SVA, &task->mm->context.flags); //设置 mm->context.flags 中的标志位
        return 0;
    case ARCH_GET_MAX_TAG_BITS: //返回用户可以请求的最大 tag bits
        if (!cpu_feature_enabled(X86_FEATURE_LAM)) //如果不支持 LAM
            return put_user(0, (unsigned long __user *)arg2); //返回零
        else
            return put_user(LAM_U57_BITS, (unsigned long __user *)arg2); //目前仅支持 LAM_U57
#endif
    default:
        ret = -EINVAL;
        break;
    }

    return ret;
}
```

## [PATCH 09/19] mm: Expose untagging mask in /proc/$PID/status
* 在 `/proc/$PID/status` 中添加一行以报告 `untag_mask`。它可以用来从外部找出进程的 LAM 状态。它对于调试器很有用。
* 其实就是返回 per-task 的 `mm->context.untag_mask` 就可以了
```diff
diff --git a/arch/x86/include/asm/mmu_context.h b/arch/x86/include/asm/mmu_context.h
index eb1387ac40fa..06eaaf75d572 100644
--- a/arch/x86/include/asm/mmu_context.h
+++ b/arch/x86/include/asm/mmu_context.h
@@ -104,6 +104,12 @@ static inline void dup_lam(struct mm_struct *oldmm, struct mm_struct *mm)
    mm->context.untag_mask = oldmm->context.untag_mask;
 }

+#define mm_untag_mask mm_untag_mask
+static inline unsigned long mm_untag_mask(struct mm_struct *mm)
+{
+   return mm->context.untag_mask;
+}
+
 static inline void mm_reset_untag_mask(struct mm_struct *mm)
 {
    mm->context.untag_mask = -1UL;
```
* 剩下的都是打印的事，`/proc/$PID/status` 对应的打印函数为 `proc_pid_status()`
```diff
diff --git a/fs/proc/array.c b/fs/proc/array.c
index 9b0315d34c58..6daea628bc76 100644
--- a/fs/proc/array.c
+++ b/fs/proc/array.c
@@ -423,6 +424,11 @@ static inline void task_thp_status(struct seq_file *m, struct mm_struct *mm)
    seq_printf(m, "THP_enabled:\t%d\n", thp_enabled);
 }

+static inline void task_untag_mask(struct seq_file *m, struct mm_struct *mm)
+{
+   seq_printf(m, "untag_mask:\t%#lx\n", mm_untag_mask(mm));
+}
+
 int proc_pid_status(struct seq_file *m, struct pid_namespace *ns,
            struct pid *pid, struct task_struct *task)
 {
@@ -438,6 +444,7 @@ int proc_pid_status(struct seq_file *m, struct pid_namespace *ns,
        task_mem(m, mm);
        task_core_dumping(m, task);
        task_thp_status(m, mm);
+       task_untag_mask(m, mm);
        mmput(mm);
    }
    task_sig(m, task);
```

## [PATCH 10/19] iommu/sva: Replace pasid_valid() helper with mm_valid_pasid()
* 内核很少有 `pasid_valid()` 的用户，除了一个之外，所有用户都会检查进程是否分配了 PASID。Helper 将 `ioasid_t` 作为输入。
* 将 helper 替换为以 `mm_struct` 作为参数的 `mm_valid_pasid()`。唯一检查未绑定到 `mm_struct` 的 PASID 的调用现在是开放编码的。
* 这是做准备 patch。它有助于避免 ifdefery：无需在通用代码中解引用 `mm->pasid` 来检查进程是否具有 PASID。
* Commit message 解释的其实已经比较清楚了，PASID 是 x86 特有的，在通用代码中解引用 `mm->pasid` 还不如只传 `mm_struct` 给 `mm_valid_pasid()` 判断。这个 patch 没有功能上的改变

## [PATCH 11/19] x86/mm/iommu/sva: Make LAM and SVA mutually exclusive
* 支持 IOMMU 和 SVA 的设备对 LAM 一无所知，只期望规范地址。尝试传递被 tagged 的指针将导致地址转换失败。
* 默认情况下，不允许在同一进程中同时启用 LAM 和使用 SVA。
* 新的 `ARCH_FORCE_TAGGED_SVA` `arch_prctl()` 覆盖了该限制。
* 通过使用 `arch_prctl()`，用户空间有责任不将 tagged 的地址传递给设备。
---
* 新增 `task_struct->mm->context.flags` 标志 `MM_CONTEXT_FORCE_TAGGED_SVA` 
  * arch/x86/include/asm/mmu.h
```cpp
/* Allow LAM and SVA coexisting */
#define MM_CONTEXT_FORCE_TAGGED_SVA 3
```
* 新增函数 `arch_pgtable_dma_compat()` 判断页表是否 DMA 兼容
  * 看起来 LAM 和 SVA 不兼容，因此首要条件是 `!mm_lam_cr3_mask(mm)` 即进程的 `CR3` 里的 LAM 开关 `mm->context.lam_cr3_mask` 没打开
  * 如果首要条件不满足，另一个条件是 tagged SVA 被强制打开，即 `MM_CONTEXT_FORCE_TAGGED_SVA` 标志位被设置
  * 如果两个条件都不满足，则页表与 DMA 不兼容
  * arch/x86/include/asm/mmu_context.h
```cpp
#define arch_pgtable_dma_compat arch_pgtable_dma_compat
static inline bool arch_pgtable_dma_compat(struct mm_struct *mm)
{
    return !mm_lam_cr3_mask(mm) ||
        test_bit(MM_CONTEXT_FORCE_TAGGED_SVA, &mm->context.flags);
}
```
* 页表与 DMA 不兼容时无法将进程地址空间绑定到设备
```diff
diff --git a/drivers/iommu/iommu-sva.c b/drivers/iommu/iommu-sva.c
index 4ee2929f0d7a..dd76a1a09cf7 100644
--- a/drivers/iommu/iommu-sva.c
+++ b/drivers/iommu/iommu-sva.c
@@ -32,6 +33,9 @@ int iommu_sva_alloc_pasid(struct mm_struct *mm, ioasid_t min, ioasid_t max)
        min == 0 || max < min)
        return -EINVAL;

+   if (!arch_pgtable_dma_compat(mm))
+       return -EBUSY;
+
    mutex_lock(&iommu_sva_lock);
    /* Is a PASID already associated with this mm? */
    if (mm_valid_pasid(mm)) {

```
* 新的 `ARCH_FORCE_TAGGED_SVA` `arch_prctl()` 参数
  * arch/x86/include/uapi/asm/prctl.h
```cpp
#define ARCH_FORCE_TAGGED_SVA      0x4004
```
* 上面 `do_arch_prctl_64()` 已经列出对应 `ARCH_FORCE_TAGGED_SVA` 的修改了
* 另一处是 `prctl_enable_tagged_addr()`，通过 `arch_prctl()` 启用 LAM，增加了检查：PASID 有效意味着不能使用 LAM，除非 tagged SVA 被强制打开
```diff
diff --git a/arch/x86/kernel/process_64.c b/arch/x86/kernel/process_64.c
index 186f34add865..b46924c9e46d 100644
--- a/arch/x86/kernel/process_64.c
+++ b/arch/x86/kernel/process_64.c
@@ -756,6 +756,10 @@ static int prctl_enable_tagged_addr(struct mm_struct *mm, unsigned long nr_bits)
    if (current->mm != mm)
        return -EINVAL;

+   if (mm_valid_pasid(mm) &&
+       !test_bit(MM_CONTEXT_FORCE_TAGGED_SVA, &mm->context.flags))
+       return -EINTR;
+
    if (mmap_write_lock_killable(mm))
        return -EINTR;

@@ -878,6 +882,9 @@ long do_arch_prctl_64(struct task_struct *task, int option, unsigned long arg2)
                (unsigned long __user *)arg2);
    case ARCH_ENABLE_TAGGED_ADDR:
        return prctl_enable_tagged_addr(task->mm, arg2);
+   case ARCH_FORCE_TAGGED_SVA:
+       set_bit(MM_CONTEXT_FORCE_TAGGED_SVA, &task->mm->context.flags);
+       return 0;
    case ARCH_GET_MAX_TAG_BITS:
        if (!cpu_feature_enabled(X86_FEATURE_LAM))
            return put_user(0, (unsigned long __user *)arg2);
```

## [PATCH 12/19] selftests/x86/lam: Add malloc and tag-bits test cases for linear-address masking
* LAM 仅在 64 位模式下受支持，并且仅适用于用于数据访问的地址。在 64 位模式下，线性地址有 64 位。LAM 应用于 64 位线性地址，并允许软件使用高位元数据。
* LAM 支持不同的配置，即哪些指针位被屏蔽并且可用于元数据。
* LAM包括以下模式：
  - `LAM_U57`，位置 `62:57` 中的指针位被屏蔽（LAM 宽度 `6`），允许用户指针的 bit `62:57` 用作元数据。
* 有一些 `arch_prctls`：
  * `ARCH_ENABLE_TAGGED_ADDR`：启用 LAM 模式，屏蔽用户指针的高位。
  * `ARCH_GET_UNTAG_MASK`：获取当前 untagged 的掩码。
  * `ARCH_GET_MAX_TAG_BITS`：用户可以请求的最大 tag bits。如果不支持 LAM，则为零。
* LAM 模式用于预处理，一个进程只有一次机会设置 LAM 模式。
* 但没有 API 可以禁用 LAM 模式。因此所有测试用例都在子进程下运行。
* 本次测试的功能：
  * MALLOC
    - `LAM_U57` 屏蔽用户指针的 bits `57:62`。用户空间上的进程可以解引用此类指针。
    - 禁用 LAM，解引用元数据高于 `48` bit 或 `57` bit 的指针会导致触发 `SIGSEGV`。
  * TAG_BITS
    - `LAM_U57` 的最大 tag bit 为 `6`。
---
* 目前测试用例（完整版）如下：
```sh
$ ./lam_64 -h
usage: lam [-h] [-t test list]
        -t test list: run tests specified in the test list, default:0x7f
                0x1:malloc; 0x2:max_bits; 0x4:mmap; 0x8:syscall; 0x10:io_uring; 0x20:inherit;
        -h: help
```
* 测试的核心框架如下：
  * `main()` 根据输入确定要测哪些测试集，并调用 `run_test()` 针对每个测试集运行测试
  * `run_test()` 跑每个测试集中的测试用例，以 `fork_test()` 的方式运行
  * `fork_test()` 在子进程中调测试用例的回调 `test->test_func(test)` 运行测试用例，父进程收集测试返回值
  * tools/testing/selftests/x86/lam.c
```cpp
struct testcases {
    unsigned int later;
    int expected; /* 2: SIGSEGV Error; 1: other errors */
    unsigned long lam;
    uint64_t addr;
    uint64_t cmd;
    int (*test_func)(struct testcases *test);
    const char *msg;
};

static int fork_test(struct testcases *test)
{
    int ret, child_ret;
    pid_t pid;

    pid = fork();
    if (pid < 0) {
        perror("Fork failed.");
        ret = 1;
    } else if (pid == 0) {
        ret = test->test_func(test);
        exit(ret);
    } else {
        wait(&child_ret);
        ret = WEXITSTATUS(child_ret);
    }

    return ret;
}

static void run_test(struct testcases *test, int count)
{
    int i, ret = 0;

    for (i = 0; i < count; i++) {
        struct testcases *t = test + i;

        /* fork a process to run test case */
        tests_cnt++;
        ret = fork_test(t);

        /* return 3 is not support LA57, the case should be skipped */
        if (ret == 3) {
            ksft_test_result_skip(t->msg);
            continue;
        }

        if (ret != 0)
            ret = (t->expected == ret);
        else
            ret = !(t->expected);

        ksft_test_result(ret, t->msg);
    }
}

int main(int argc, char **argv)
{
    int c = 0;
    unsigned int tests = TEST_MASK;

    tests_cnt = 0;
    if (!cpu_has_lam()) {
        ksft_print_msg("Unsupported LAM feature!\n");
        return -1;
    }

    while ((c = getopt(argc, argv, "ht:")) != -1) {
        switch (c) {
        case 't':
            tests = strtoul(optarg, NULL, 16);
            if (tests && !(tests & TEST_MASK)) {
                ksft_print_msg("Invalid argument!\n");
                return -1;
            }
            break;
        case 'h':
            cmd_help();
            return 0;
        default:
            ksft_print_msg("Invalid argument\n");
            return -1;
        }
    }

    /*
     * When tests is 0, it is not a real test case;
     * the option used by test case(execve) to check the lam mode in
     * process generated by execve, the process read back lam mode and
     * check with lam mode in parent process.
     */
    if (!tests)
        return (get_lam());

    /* Run test cases */
    if (tests & FUNC_MALLOC)
        run_test(malloc_cases, ARRAY_SIZE(malloc_cases));

    if (tests & FUNC_BITS)
        run_test(bits_cases, ARRAY_SIZE(bits_cases));

    if (tests & FUNC_MMAP)
        run_test(mmap_cases, ARRAY_SIZE(mmap_cases));

    if (tests & FUNC_SYSCALL)
        run_test(syscall_cases, ARRAY_SIZE(syscall_cases));

    if (tests & FUNC_URING)
        run_test(uring_cases, ARRAY_SIZE(uring_cases));

    if (tests & FUNC_INHERITE)
        run_test(inheritance_cases, ARRAY_SIZE(inheritance_cases));

    if (tests & FUNC_PASID)
        run_test(pasid_cases, ARRAY_SIZE(pasid_cases));

    ksft_set_plan(tests_cnt);

    return ksft_exit_pass();
}
```
* `set_lam()` 是给当前进程启用 LAM 的函数，返回 `0` 表示 LAM 模式启用成功
```cpp
/*
 * Set tagged address and read back untag mask.
 * check if the untagged mask is expected.
 *
 * @return:
 * 0: Set LAM mode successfully
 * others: failed to set LAM
 */
static int set_lam(unsigned long lam)
{
    int ret = 0;
    uint64_t ptr = 0;

    if (lam != LAM_U57_BITS && lam != LAM_NONE)
        return -1;

    /* Skip check return */
    syscall(SYS_arch_prctl, ARCH_ENABLE_TAGGED_ADDR, lam);

    /* Get untagged mask */
    syscall(SYS_arch_prctl, ARCH_GET_UNTAG_MASK, &ptr);

    /* Check mask returned is expected */
    if (lam == LAM_U57_BITS)
        ret = (ptr != ~(LAM_U57_MASK)); //这里有点绕，如果设置 LAM_U57 成功，ptr 等于 ~(LAM_U57_MASK)，!= 表达式为 false，故 ret = 0
    else if (lam == LAM_NONE)
        ret = (ptr != -1ULL);

    return ret;
}
```
* `set_metadata()` 是将传入的指针加工成带 metadata 的指针的函数
```cpp
/* According to LAM mode, set metadata in high bits */
static uint64_t set_metadata(uint64_t src, unsigned long lam)
{
    uint64_t metadata;

    srand(time(NULL));

    switch (lam) {
    case LAM_U57_BITS: /* Set metadata in bits 62:57 */
        /* Get a random non-zero value as metadata */
        metadata = (rand() % ((1UL << LAM_U57_BITS) - 1) + 1) << 57;
        metadata |= (src & ~(LAM_U57_MASK));
        break;
    default:
        metadata = src;
        break;
    }

    return metadata;
}
```
* MALLOC 测试用例所做的核心测试如下
  * tools/testing/selftests/x86/lam.c
```cpp
/*
 * Set metadata in user pointer, compare new pointer with original pointer.
 * both pointers should point to the same address.
 *
 * @return:
 * 0: value on the pointer with metadate and value on original are same
 * 1: not same.
 */
static int handle_lam_test(void *src, unsigned int lam)
{
    char *ptr;

    strcpy((char *)src, "USER POINTER");
    //指针 src 的 57:63 bit 被存入随机的 metadata
    ptr = (char *)set_metadata((uint64_t)src, lam);
    if (src == ptr) //应该是不相等，条件不满足
        return 0;
    //如果 LAM 起作用了会忽略 ptr 指针的 metadata bit，往与 src 相同的地址拷贝；如果没起作用，会解引用带 metadata 的指针导致 SIGSEGV
    /* Copy a string into the pointer with metadata */
    strcpy((char *)ptr, "METADATA POINTER");
    //如果 LAM 起作用了比较结果应该相等，返回 0
    return (!!strcmp((char *)src, (char *)ptr));
}
```
## [PATCH 13/19] selftests/x86/lam: Add mmap and SYSCALL test cases for linear-address masking
* 添加 `mmap` 和 `SYSCALL` 测试用例。
* `SYSCALL` 测试用例：
  - LAM 支持在用户指针的高位 `62:57`（`LAM_U57`）设置元数据，将指针传递给 `SYSCALL`，`SYSCALL` 可以解引用指针并返回正确的结果。
  - 禁用 LAM，将带有高位元数据的指针传递给 `SYSCALL`，`SYSCALL` 返回 `-1`（默认）。
* MMAP 测试用例：
  - 启用 `LAM_U57`，具有低地址（bit `47` 以下）的 MMAP，在地址的高位中设置元数据，应允许解引用地址。
  - 启用 `LAM_U57`，具有高地址（bit `47` 以上）的 MMAP，在地址的高位中设置元数据，应允许解引用地址。

## [PATCH 14/19] selftests/x86/lam: Add io_uring test cases for linear-address masking
* 内核线程应该支持 LAM，使用 `io_uring` 来验证 LAM 功能。
* 测试用例通过 `io_uring` 实现读取文件，测试用例选择一个 `iovec` 数组作为接收缓冲区，用于接收数据，根据 LAM 模式，在这些缓冲区的高位设置元数据。
* `io_uring` 可以处理这些指向具有高位元数据的指针的缓冲区。

## [PATCH 15/19] selftests/x86/lam: Add inherit test cases for linear-address masking
* LAM 按线程启用并在 `fork(2)/clone(2)` 上继承。`exec()` 将 LAM 状态恢复为默认禁用状态。有两种测试场景：
- Fork 测试用例：
  * 这些案例用于测试每个线程的 LAM 继承性，`fork()` 生成的子进程应该继承父进程的 LAM 功能，子进程可以获得与父进程相同的 LAM 模式。
- Execve 测试用例：
  * `execve()` 生成的进程与 `fork()` 生成的进程不同，这些进程将 LAM 状态恢复为禁用状态。

## [PATCH 16/19] selftests/x86/lam: Add ARCH_FORCE_TAGGED_SVA test cases for linear-address masking
* 默认情况下，不允许在同一进程中同时启用 LAM 和使用 SVA。
* 但新的 `ARCH_FORCE_TAGGED_SVA` `arch_prctl()` 覆盖了该限制。
* 为新的 `arch_prctl` 添加新的测试用例：
* 在使用 `ARCH_FORCE_TAGGED_SVA` 之前，不应允许启用 LAM/SVA 共存。测试用例应该是反向的（Negative）。
* 该测试依赖于 idxd 驱动程序和 iommu。测试之前，需要在内核命令行中添加“`intel_iommu=on,sm_on`”和 `insmod idxd` 驱动程序。

## [PATCH 17/19] selftests/x86/lam: Add test cases for LAM vs thread creation
* 仅当进程具有单线程时才允许启用 LAM。
  * 见 [[PATCH 08/19] x86/mm: Provide arch_prctl() interface for LAM](#patch-0819-x86mm-provide-arch_prctl-interface-for-lam)
  * 进程一旦 clone 出子线程后会把 LAM 置于锁定状态 `set_bit(MM_CONTEXT_LOCK_LAM, &p->mm->context.flags)`
* 主线程尝试在生成线程后启用 LAM 必须失败。
  * 原因和第一条一样
  * 同样，子进程也无法启用 LAM
* LAM 模式会被继承到子线程中。
  * 子线程与父线程共享 `mm` 嘛。也就是说，在 clone 之前启用 LAM 是可以的

## [PATCH 18/19] x86/mm/iommu/sva: Fix error code for LAM enabling failure due to SVA
* 通常，LAM 和 SVA 是互斥的。如果 SVA 已在使用中，LAM 启用将会失败。
* 更正失败的错误代码。`EINTR` 在那里是无意义的。
* 上面看到的 `prctl_enable_tagged_addr()` 已是最终版了，包含本 patch 的修改

## [PATCH 19/19] x86/mm/iommu/sva: Do not allow to set FORCE_TAGGED_SVA bit from outside
* `arch_prctl(ARCH_FORCE_TAGGED_SVA)` 覆盖默认值并允许 LAM 和 SVA 在进程中共存。当进程知道自己在做什么时，期望它被调用。
* `arch_prctl()` 对当前进程进行操作，但可以从 `ptrace` 访问相同的代码，可以在任意任务上调用它。
* 增加严格的限定，使其仅允许为当前进程设置 `MM_CONTEXT_FORCE_TAGGED_SVA`。
* 上面看到的 `prctl_enable_tagged_addr()` 已是最终版了，包含本 patch 的修改

# Linus Cleanup

## [PATCH 1/5] x86-64: make access_ok() independent of LAM
* Linear address masking（LAM）代码使 `access_ok()` 变得更加复杂，因为它现在需要去除地址标记（untag the address）才能验证访问范围。请参阅 commit 74c228d20a51 ("x86/uaccess：Provide untagged_addr() and remove tags before address check")。
* 我们能够通过简单地使用符号位进行地址检查来避免 `get_user/put_user` 代码路径中的开销，并根据 GP fault 判断地址是否不规范，这使得它完全独立于 LAM。
* 我们可以对 `access_ok()` 做同样的事情：简单地检查用户指针范围是否已清除高位。无需费心任何地址位掩码。
* 事实上，我们可以更进一步，只检查已知小访问范围的起始地址：任何溢出的访问仍将位于非规范区域中，并且仍将出现 GP fault。
* 为了仍然使 syzkaller 捕获任何可能未经检查的用户地址，我们将继续警告由非规范范围内的访问引起的 GP fault。但我们将其限制为纯粹的“高位设置并超过一页的‘slop’区域”。
* 我们可能可以对任意范围大小“仅检查起始地址”：实际上，所有内核对用户空间的访问都将从低地址开始完成。但我们还是把这种优化留到以后再说吧。事实上，这已经允许我们生成更简单的代码，而不必担心地址中的任何标记位。
* 需要注意的一件事是 GUP 地址检查：GUP 将手动查找页表，而不是实际复制虚拟地址范围中的数据（因此 GP fault 会捕获错误地址）。因此，需要检查页表限制，而这之前是由 `access_ok()` 隐式完成的。
* 通过宽松的 `access_ok()` 检查，我们只需在 GUP 代码中对 `TASK_SIZE_MAX` 进行显式检查即可。无论如何，GUP 代码已经需要取消 tag 位屏蔽，因此这一切都非常简单，并且不存在 LAM 问题。
---
* 在 arch/x86/include/asm/uaccess.h 中新增 `__access_ok()` 的定义
* 在 x86-64 上，用户指针中可能有 tag 位。不要屏蔽它们，只需更改 `__access_ok()` 的规则即可。
  * 规则是“`ptr+size`”不能溢出，并且不能设置高位。
  * 编译器通常了解无符号溢出和 `CF` 位，并为此生成合理的代码。
  * 尽管看起来这个组合至少会混淆 clang（而不是仅仅执行“`add`”然后测试 `SF` 和 `CF`，你会看到不必要的比较）。
* 对于可以在编译时检查的小尺寸的常见情况，甚至不必费心添加，只需检查基指针是否正常即可。
* arch/x86/include/asm/uaccess.h
```cpp
#ifdef CONFIG_X86_64
/*
 * On x86-64, we may have tag bits in the user pointer. Rather than
 * mask them off, just change the rules for __access_ok().
 *
 * Make the rule be that 'ptr+size' must not overflow, and must not
 * have the high bit set. Compilers generally understand about
 * unsigned overflow and the CF bit and generate reasonable code for
 * this. Although it looks like the combination confuses at least
 * clang (and instead of just doing an "add" followed by a test of
 * SF and CF, you'll see that unnecessary comparison).
 *
 * For the common case of small sizes that can be checked at compile
 * time, don't even bother with the addition, and just check that the
 * base pointer is ok.
 */
static inline bool __access_ok(const void __user *ptr, unsigned long size)
{
    if (__builtin_constant_p(size <= PAGE_SIZE) && size <= PAGE_SIZE) {
        return (long)ptr >= 0;
    } else {
        unsigned long sum = size + (unsigned long)ptr;
        return (long) sum >= 0 && sum >= (unsigned long)ptr;
    }
}
#define __access_ok __access_ok
#endif
```
* `__builtin_constant_p` 是编译器 GCC 内置函数，用于判断一个值是否为编译时常量，如果是常量，函数返回 `1`，否则返回 `0`。此内置函数的典型用法是在宏中用于手动编译时优化。
* `if (__builtin_constant_p(size <= PAGE_SIZE) && size <= PAGE_SIZE)` 即为判断小尺寸的情况，无论是否开启 LAM `ptr < 0` 都是访问内核地址
* `sum < 0` 意味着访问的范围达到内核地址的范围，无论是否开启 LAM 该检查都有效
* `sum < ptr` 意味着访问范围的结束甚至发生了回绕，无论是否开启 LAM 该检查都有效
```diff
diff --git a/arch/x86/include/asm/uaccess.h b/arch/x86/include/asm/uaccess.h
index 457e814712af..123135d60f72 100644
--- a/arch/x86/include/asm/uaccess.h
+++ b/arch/x86/include/asm/uaccess.h
@@ -91,11 +119,14 @@ static inline unsigned long __untagged_addr_remote(struct mm_struct *mm,
  *
  * Return: true (nonzero) if the memory block may be valid, false (zero)
  * if it is definitely invalid.
+ *
+ * This should not be x86-specific. The only odd things out here is
+ * the WARN_ON_IN_IRQ(), which doesn't exist in the generic version.
  */
-#define access_ok(addr, size)                      \
-({                                 \
-   WARN_ON_IN_IRQ();                       \
-   likely(__access_ok(untagged_addr(addr), size));         \
+#define access_ok(addr, size)          \
+({                     \
+   WARN_ON_IN_IRQ();           \
+   likely(__access_ok(addr, size));    \
 })

 #include <asm-generic/access_ok.h>
```
* `access_ok()` 不应该是 x86 特定的版本，唯一导致例外的是它有 `WARN_ON_IN_IRQ()`，这是通用版本所没有的
---
* 新增函数 `gp_fault_address_ok()` 判断 `#GP` 异常的地址访问是否合法
* 在 x86-64 上，我们最终对“`access_ok()`”不精确，并允许 *非规范用户地址* 的范围比较更简单，并且不必担心启用 LAM。
* 事实上，我们在符号边界处允许最多一页“倾斜”，这意味着我们可以通过检查指针的符号来执行 `access_ok()`，以应对具有较小访问大小的常见情况。
* arch/x86/mm/extable.c
```cpp
/*
 * On x86-64, we end up being imprecise with 'access_ok()', and allow
 * non-canonical user addresses to make the range comparisons simpler,
 * and to not have to worry about LAM being enabled.
 *
 * In fact, we allow up to one page of "slop" at the sign boundary,
 * which means that we can do access_ok() by just checking the sign
 * of the pointer for the common case of having a small access size.
 */
static bool gp_fault_address_ok(unsigned long fault_address)
{
#ifdef CONFIG_X86_64
    /* Is it in the "user space" part of the non-canonical space? */
    if ((long) fault_address >= 0)
        return true;

    /* .. or just above it? */
    fault_address -= PAGE_SIZE;
    if ((long) fault_address >= 0)
        return true;
#endif
    return false;
}
```
* `(long) fault_address` 将地址转为有符号的类型，对判断条件 `if ((long) fault_address >= 0)` 结果如下：
  * 如果是内核地址则为负数，条件不成立，减去 `PAGE_SIZE` 也不成立，因此地址不合法
    * 仅当该 *内核地址 - 4KB* （“高位设置并超过一页的‘slop’区域”）得到的非规范用户地址不触发警告
  * 如果是正常用户态地址，无论是否开启 LAM 都为正数，条件成立，因此地址合法
  * 如果是带 tag 的用户态地址，也不会返回 `false` 引发警告
    * 如果开启 LAM，tag 部分被 CPU 忽略
    * 如果关闭 LAM，视为非规范地址，会引起 `#GP`
* 修改 `ex_handler_uaccess()` 函数，新增了 `fault_address` 参数，传递给 `gp_fault_address_ok()` 判断是否需要打印 “`#GP` 发生在用户访问，非规范地址？”的警告
```diff
diff --git a/arch/x86/mm/extable.c b/arch/x86/mm/extable.c
index 60814e110a54..8d38dedadbb1 100644
--- a/arch/x86/mm/extable.c
+++ b/arch/x86/mm/extable.c
 static bool ex_handler_uaccess(const struct exception_table_entry *fixup,
-                  struct pt_regs *regs, int trapnr)
+                  struct pt_regs *regs, int trapnr,
+                  unsigned long fault_address)
 {
-   WARN_ONCE(trapnr == X86_TRAP_GP, "General protection fault in user access. Non-canonical address?");
+   WARN_ONCE(trapnr == X86_TRAP_GP && !gp_fault_address_ok(fault_address),
+       "General protection fault in user access. Non-canonical address?");
    return ex_handler_default(fixup, regs);
 }
```
* 受此影响，调用 `ex_handler_uaccess()` 都要修改，增加传递 `fault_address` 参数
---
* `internal_get_user_pages_fast()` 检查结束地址，如果超出用户态地址范围，返回 `-EFAULT`
```diff
diff --git a/mm/gup.c b/mm/gup.c
index ff689c88a357..bbe416236593 100644
--- a/mm/gup.c
+++ b/mm/gup.c
@@ -2970,6 +2970,8 @@ static int internal_get_user_pages_fast(unsigned long start,
    len = nr_pages << PAGE_SHIFT;
    if (check_add_overflow(start, len, &end))
        return 0;
+   if (end > TASK_SIZE_MAX)
+       return -EFAULT;
    if (unlikely(!access_ok((void __user *)start, len)))
        return -EFAULT;
```

## [PATCH 2/5] x86: mm: remove architecture-specific 'access_ok()' define
* asm-generic/access_ok.h 头文件中已经有“`access_ok()`”的通用定义，它与 x86 特定的唯一区别是（x86）添加了对 `WARN_ON_IN_IRQ()` 的检查。
* 事实证明，该检查的原因早已不复存在：它曾经使用依赖于当前线程的“`user_addr_max()`”内联函数，并在非线程上下文中引起问题。
* 有关详细信息，请参阅 commit 7c4788950ba5（“x86/uaccess，sched/preempt：Verify access_ok() context”），特别是 commit ae31fe51a3cc（“perf/x86: Restore TASK_SIZE check on frame pointer”）了解这是如何以及为何发生的 。
* 但是这个“current task”问题在 Christoph Hellwig 在 commit 47058bb54b57 中的 `set_fs()` 大4删除中被删除（“x86：remove address space overrides using set_fs()”）。
* 因此，测试的原因和特定于体系结构的 `access_ok()` 定义不再存在，而且现在实际上是有害的。例如，它导致了各种“`copy_from_user_nmi()`”游戏（例如使用 `__range_not_ok()` 代替，然后在可以时转换为 `__access_ok()`）。
* 这反过来意味着 LAM 在本系列之前的帧中被破坏，因为 `__access_ok()` 过去不进行地址 untagging。
* 在许多情况下访问用户状态仍然需要小心，但 `access_ok()` 不是进行此测试的地方。
---
* 该 patch 删除了 arch/x86/include/asm/uaccess.h 中的 x86 特定的 `access_ok()` 定义，这样 `access_ok()` 会使用 asm-generic/access_ok.h 中的通用定义

## [PATCH 3/5] x86: uaccess: move 32-bit and 64-bit parts into proper <asm/uaccess_N.h> header
* x86 <asm/uaccess.h> 文件增加了特定于 x86-64 的功能，例如 LAM 支持和相关的 `access_ok()` 更改。
* 它们确实应该位于 <asm/uaccess_64.h> 文件中，并且不会污染通用 x86 头文件。

## [PATCH 4/5] x86: mm: remove 'sign' games from LAM untagged_addr*() macros
* 符号游戏的目的是在 untagging 时不修改内核地址。然而，这有两个问题：
  (a) 它实际上并没有按预期工作，因为掩码在 _unsigned_ 地址上计算为“`addr >> 63`”。因此，你得到的不是内核地址的全掩码，而是“`1`”。
  (b) 无论如何，untagging 内核地址实际上并不是一个有效的操作。
* 现在，(a) 最初对于“`untagged_addr()`”及其远程版本都是正确的，但通过 commit e0bddc19ba95（“x86/mm: Reduce untagged_addr() overhead for systems without LAM”），意外地修复了 `untagged_addr()` 的常规版本。
  * 该指令将 *移位* 重写为 alternative asm 代码的一部分，并在此过程中将无符号移位更改为带符号的“`sar`”指令。
* 虽然我们确实不想通过屏蔽高位来将看起来像内核地址的内容转换为用户地址，但这不需要这些符号屏蔽游戏 - 它所需要的只是 mm 上下文“`untag_mask`” 的值设置了高位。
* 它总是这样做。
* 因此，只需删除地址掩码中多余的（在 `untagged_addr_remote()` 的情况下，仍然有问题）符号位游戏即可简化代码。
```diff
diff --git a/arch/x86/include/asm/uaccess_64.h b/arch/x86/include/asm/uaccess_64.h
index 20411e69e67f..e5b23e917f41 100644
--- a/arch/x86/include/asm/uaccess_64.h
+++ b/arch/x86/include/asm/uaccess_64.h
@@ -15,25 +15,17 @@
 #ifdef CONFIG_ADDRESS_MASKING
 /*
  * Mask out tag bits from the address.
- *
- * Magic with the 'sign' allows to untag userspace pointer without any branches
- * while leaving kernel addresses intact.
  */
 static inline unsigned long __untagged_addr(unsigned long addr)
 {
-   long sign;
-
    /*
     * Refer tlbstate_untag_mask directly to avoid RIP-relative relocation
     * in alternative instructions. The relocation gets wrong when gets
     * copied to the target place.
     */
    asm (ALTERNATIVE("",
-            "sar $63, %[sign]\n\t" /* user_ptr ? 0 : -1UL */
-            "or %%gs:tlbstate_untag_mask, %[sign]\n\t"
-            "and %[sign], %[addr]\n\t", X86_FEATURE_LAM)
-        : [addr] "+r" (addr), [sign] "=r" (sign)
-        : "m" (tlbstate_untag_mask), "[sign]" (addr));
+            "and %%gs:tlbstate_untag_mask, %[addr]\n\t", X86_FEATURE_LAM)
+        : [addr] "+r" (addr) : "m" (tlbstate_untag_mask));

    return addr;
 }
@@ -46,12 +38,8 @@ static inline unsigned long __untagged_addr(unsigned long addr)
 static inline unsigned long __untagged_addr_remote(struct mm_struct *mm,
                           unsigned long addr)
 {
-   long sign = addr >> 63;
-
    mmap_assert_locked(mm);
-   addr &= (mm)->context.untag_mask | sign;
-
-   return addr;
+   return addr & (mm)->context.untag_mask;
 }

 #define untagged_addr_remote(mm, addr) ({              \
```

## [PATCH 5/5] x86-64: mm: clarify the 'positive addresses' user address rules
* Dave Hansen 发现 x86-64 `access_ok` 检查中的“`(long) addr >= 0`”代码有些令人困惑，并建议使用 helper 来阐明代码的用途。
* 因此，这正是这样做的：通过添加一个帮助宏来明确符号位检查的含义，以明确其正在测试的内容。
* 这还添加了一些明确的注释，讨论即使启用了 LAM，任何带有符号位的地址仍然会在符号位上方的非规范区域中出现 GP 错误。
* 这就是允许我们仅使用符号位进行用户地址检查的全部原因，此外，对于甚至超过该点也可能使用额外偏移量完成的访问有点随意。
* （是的，这谈论的是“正数”，尽管零也是一个有效的用户地址，所以从技术上讲，我们应该称它们为“非负数”。但我不认为使用“非负数”最终会更容易理解）。
---
* 虚拟地址空间在逻辑上分为内核一半和用户一半。当转换为有符号类型时，用户指针为正，内核指针为负。
* 用户指针在 x86-64 上可以有 tag bits。该方案容忍这些位中的任意值，而不是将它们屏蔽掉。
* 执行两条规则：
1. `ptr` 必须位于地址空间的用户一半
2. `ptr+size` 不得溢出到内核地址
* 请注意，符号发生改变周围的地址不是有效地址，并且如果设置了符号位，即使启用了 LAM，也会出现 `GP` 错误（请参阅“`CR3.LAM_SUP`”，如果我们启用它，它可以缩小规范性检查范围，但不能完全删除它）。
* 因此，“溢出到内核地址”并不意味着符号位处突然出现精确边界，并且我们可以在大小检查上允许大量的倾斜。
* 事实上，我们很可能可以完全删除大小检查，因为任何内核访问都将从“`ptr`”开始按递增的地址顺序，即使末尾可能在内核空间中，在我们到达那里之前访问，我们也会遇到非规范的 `GP` 错误。
* 这是一个单独的优化，现在只处理小常量的情况。
* arch/x86/include/asm/uaccess_64.h
```cpp
/*
 * The virtual address space space is logically divided into a kernel
 * half and a user half.  When cast to a signed type, user pointers
 * are positive and kernel pointers are negative.
 */
#define valid_user_address(x) ((long)(x) >= 0)

/*
 * User pointers can have tag bits on x86-64.  This scheme tolerates
 * arbitrary values in those bits rather then masking them off.
 *
 * Enforce two rules:
 * 1. 'ptr' must be in the user half of the address space
 * 2. 'ptr+size' must not overflow into kernel addresses
 *
 * Note that addresses around the sign change are not valid addresses,
 * and will GP-fault even with LAM enabled if the sign bit is set (see
 * "CR3.LAM_SUP" that can narrow the canonicality check if we ever
 * enable it, but not remove it entirely).
 *
 * So the "overflow into kernel addresses" does not imply some sudden
 * exact boundary at the sign bit, and we can allow a lot of slop on the
 * size check.
 *
 * In fact, we could probably remove the size check entirely, since
 * any kernel accesses will be in increasing address order starting
 * at 'ptr', and even if the end might be in kernel space, we'll
 * hit the GP faults for non-canonical accesses before we ever get
 * there.
 *
 * That's a separate optimization, for now just handle the small
 * constant case.
 */
static inline bool __access_ok(const void __user *ptr, unsigned long size)
{
    if (__builtin_constant_p(size <= PAGE_SIZE) && size <= PAGE_SIZE) {
        return valid_user_address(ptr);
    } else {
        unsigned long sum = size + (unsigned long)ptr;
        return valid_user_address(sum) && sum >= (unsigned long)ptr;
    }
}
```
# References
1. [Other Builtins (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html)
2. [汇编语言SAL（算术左移）和SAR（算术右移）指令](http://c.biancheng.net/view/3592.html)
- [[PATCHv13 00/16] Linear Address Masking enabling](https://lore.kernel.org/all/20221227030829.12508-1-kirill.shutemov@linux.intel.com/)
- [[PATCHv14 00/17] Linear Address Masking enabling](https://lore.kernel.org/all/CAHk-=wj4PDt_73n5rG9obkXrRQFcxN8vUhG6T9DipxozybH9_w@mail.gmail.com/)
- [[PATCHv16 00/17] Linear Address Masking enabling](https://lore.kernel.org/lkml/20230312112612.31869-1-kirill.shutemov@linux.intel.com/)
- [[GIT PULL] x86/mm for 6.4](https://lore.kernel.org/lkml/20230427225647.1101172-1-dave.hansen@linux.intel.com/#r)
- [Re [GIT PULL] x86_mm for 6.4 - Linus Torvalds](https://lore.kernel.org/lkml/CAHk-=wgrdOm8RwO+u8bydrbgs0wXJV_9mBYAtzX9d9hEY7a25A@mail.gmail.com/)
