# TDX Guest
* Guest 通过 hypercall 与 VMM 进行通信。从历史上看，这些是使用已知会导致 VM-exit 的指令（如 `VMCALL`、`VMLAUNCH` 等）实现的。但是，对于 TDX，VM-exit 不再向 host 公开 guest 的状态。这会阻止旧的 hypercall 机制工作。因此，为了与 VMM 进行通信，TDX 规范定义了一个名为 `TDCALL` 的新指令
* 在基于 TDX 的 VM 中，由于 VMM 是不受信任的实体，中间层 —— TDX module —— 促进了 host 和 guest 之间的安全通信。TDX module 像固件一样加载到称为 SEAM 的特殊 CPU 模式中。TDX guest 使用 `TDCALL` 指令与 TDX modue 通信
* Guest 使用 `TDCALL` 与 TDX module 和 VMM 进行通信。执行 `TDCALL` 指令时 `RAX` 寄存器的值用于判断 `TDCALL` 类型。
  * 用于与 VMM 通信的 `TDCALL` 的叶子函数称为 **TDVMCALL**
  * `TDVMCALL` 在 `TDCALL` 中的 leaf ID 是 `0`

* KVM hypercall 使用 `VMCALL` 或 `VMMCALL` 指令。尽管 ABI 相似，但这些指令不再对 TDX guest 起作用
* 制作 vendor-specific 的 `TDVMCALL` 而不是 `VMCALL`，这使带 KVM 的 TDX guest 可以使用充当 hypervisor
* 除其他事项外，KVM hypercall 用于发送 IPI

## TDX Guest 启动
* 简单的启动过程如下
* 先是 bzImage 的 `_start` 例程开始
* arch/x86/boot/header.S
```s
    .globl  _start
_start:
        # Explicitly enter this as bytes, or the assembler
        # tries to generate a 3-byte jump here, which causes
        # everything else to push off to the wrong offset.
        .byte   0xeb        # short (2-byte) jump
        .byte   start_of_setup-1f
...
start_of_setup:
...
# Jump to C code (should not return)
    calll   main
...
```
* arch/x86/boot/main.c
```c
main()
-> go_to_protected_mode()
   -> protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4))
```
* 转到自解压程序
* arch/x86/boot/compressed/head_64.S
```c
...
    .code64
    .org 0x200
SYM_CODE_START(startup_64)
    /*
     * 64bit entry is 0x200 and it is ABI so immutable!
     * We come here either from startup_32 or directly from a
     * 64bit bootloader.
     * If we come here from a bootloader, kernel(text+data+bss+brk),
     * ramdisk, zero_page, command line could be above 4G.
     * We depend on an identity mapped page table being provided
     * that maps our entire kernel(text+data+bss+brk), zero page
     * and command line.
     */

    cld
    cli
...
/*
 * Jump to the relocated address.
 */
    leaq    rva(.Lrelocated)(%rbx), %rax
    jmp *%rax
SYM_CODE_END(startup_64)
...
    .text
SYM_FUNC_START_LOCAL_NOALIGN(.Lrelocated)
...
/*
 * Do the extraction, and jump to the new kernel..
 */
    pushq   %rsi            /* Save the real mode argument */
    movq    %rsi, %rdi      /* real mode address */
    leaq    boot_heap(%rip), %rsi   /* malloc area for uncompression */
    leaq    input_data(%rip), %rdx  /* input_data */
    movl    input_len(%rip), %ecx   /* input_len */
    movq    %rbp, %r8       /* output target address */
    movl    output_len(%rip), %r9d  /* decompressed length, end of relocs */
    call    extract_kernel      /* returns kernel entry point in %rax */
    popq    %rsi

/*
 * Jump to the decompressed kernel.
 */
    jmp *%rax
SYM_FUNC_END(.Lrelocated)
```
* 跳转到解压完的 kernel
* arch/x86/kernel/head_64.S
```c
SYM_CODE_START(secondary_startup_64)
    UNWIND_HINT_EMPTY
    ANNOTATE_NOENDBR
...
    pushq   $.Lafter_lret   # put return address on stack for unwinder
    xorl    %ebp, %ebp  # clear frame pointer
    movq    initial_code(%rip), %rax
    pushq   $__KERNEL_CS    # set correct cs
    pushq   %rax        # target address in negative space
    lretq
.Lafter_lret:
    ANNOTATE_NOENDBR
SYM_CODE_END(secondary_startup_64)
...
    /* Both SMP bootup and ACPI suspend change these variables */
    __REFDATA
    .balign 8
SYM_DATA(initial_code,  .quad x86_64_start_kernel)
SYM_DATA(initial_gs,    .quad INIT_PER_CPU_VAR(fixed_percpu_data))
...
```
* 我们看到的 TDX guest 的第一个修改是在 `x86_64_start_kernel()` 开始的，调用 `tdx_early_init()` 函数
* 由于 `copy_bootdata()` 是第一次使用 `cc_platform_has()` API 的地方，因此在它之前检测 TDX guest 的状态
* 在 TD guest 中执行 `CPUID 0x21` 能返回 `"IntelTDX    "` 才对
```sh
$ cpuid -1 -l 0x21
CPU:
   0x00000021 0x00: eax=0x00000000 ebx=0x65746e49 ecx=0x20202020 edx=0x5844546c
```
* 随后对该返回值的判断成为 TD 内核区分自己是 host 还是 guest 的分水岭
  * 如果是 `"IntelTDX    "` 会设置一系列标识，影响到将来 host 还是 guest 的判断
  * 特别是不会设置 `vendor` 为 `cc_set_vendor(CC_VENDOR_INTEL)`。因此，`cc_platform_has()` 总是返回 `false`
## SEAMCALL
* Host kernel 通过新的 `SEAMCALL` 指令转换到 P-SEAMLDR 或 TDX module，这本质上是从 VMX root operation VMExit 到 SEAM VMX root operation
  * `SEAMCALLs` 是由 P-SEAMLDR 和 TDX module 围绕新的 `SEAMCALL` 指令定义的叶函数
### TDX 错误码
* TDX module 绝对不会使用 bits `47:40` 作为错误码，因此软件可以用作保留的状态码分类
  * bit `63` 是 TDX 软件错误位，与 bits `47:40` 组合成 TDX 软件错误码
* arch/x86/include/asm/tdx.h
```c
/*
 * SW-defined error codes.
 *
 * Bits 47:40 == 0xFF indicate Reserved status code class that never used by
 * TDX module.
 */
#define TDX_ERROR_BIT           63
#define TDX_ERROR           _BITUL(TDX_ERROR_BIT)
#define TDX_SW_ERROR            (TDX_ERROR | GENMASK_ULL(47, 40))
#define TDX_SEAMCALL_VMFAILINVALID  (TDX_SW_ERROR | _UL(0xFFFF0000))

#define TDX_SEAMCALL_GP         (TDX_SW_ERROR | X86_TRAP_GP)
#define TDX_SEAMCALL_UD         (TDX_SW_ERROR | X86_TRAP_UD)
```
* `VMfailInvalid (CF=1)` 表示 SEAMCALL 本身出了问题，通常是由于目标 SEAM 固件未加载或禁用，或者 P-SEAMLDR 正忙于另一个 SEAMCALL。在这种情况下，`%rax` 没有改变，所以将它设为`TDX_SEAMCALL_VMFAILINVALID`
* SEAMCALL 导致 `#GP` 或 `#UD` 时，`%eax` 包含异常号，将它和 `TDX_SW_ERROR` 进行 `OR` 操作得到软件错误码
### TDX_MODULE_CALL 汇编宏
* `TDX_MODULE_CALL` 是用于 TDCALL 和 SEAMCALL 的通用汇编宏，返回值放在 `%rax` 里
* arch/x86/virt/vmx/tdx/tdxcall.S
```c
/*
 * TDCALL and SEAMCALL are supported in Binutils >= 2.36.
 */
#define tdcall      .byte 0x66,0x0f,0x01,0xcc //tdcall 的字节码
#define seamcall    .byte 0x66,0x0f,0x01,0xcf //seamcall 的字节码

/*
 * TDX_MODULE_CALL - common helper macro for both
 *                 TDCALL and SEAMCALL instructions.
 *
 * TDCALL   - used by TDX guests to make requests to the
 *            TDX module and hypercalls to the VMM.
 * SEAMCALL - used by TDX hosts to make requests to the
 *            TDX module.
 */
.macro TDX_MODULE_CALL host:req //该汇编宏需要入参，赋给伪变量 host
    /*
     * R12 will be used as temporary storage for struct tdx_module_output
     * pointer. Since R12-R15 registers are not used by TDCALL/SEAMCALL
     * services supported by this function, it can be reused.
     */

    /* Callee saved, so preserve it */ //callee save 说明被调用者不能随便用，使用前要先保存
    push %r12 //我们要用 %12 来临时存储 struct tdx_module_output 的指针，故先保存 %r12 的值到栈上

    /*
     * Push output pointer to stack.
     * After the operation, it will be fetched into R12 register.
     */
    push %r9 //输入参数 5 为输出结构 struct tdx_module_output 的指针，压入栈，后面把它弹出到 %r12
    //修整函数调用的 ABI 到 TDCALL/SEAMCALL 的 ABI，因为它们不是对应的
    /* Mangle function call ABI into TDCALL/SEAMCALL ABI: */
    /* Move Leaf ID to RAX */ //SEAMCALL Leaf number
    mov %rdi, %rax
    /* Move input 4 to R9 */  //输入参数 4
    mov %r8,  %r9
    /* Move input 3 to R8 */  //输入参数 3
    mov %rcx, %r8
    /* Move input 1 to RCX */ //输入参数 1
    mov %rsi, %rcx
    /* Leave input param 2 in RDX */ //只有第二个参数是对应的

    .if \host //伪变量 host 为 1，说明是 host 在执行该宏，所以是 seamcall
1:
    seamcall
    /*
     * SEAMCALL instruction is essentially a VMExit from VMX root
     * mode to SEAM VMX root mode.  VMfailInvalid (CF=1) indicates
     * that the targeted SEAM firmware is not loaded or disabled,
     * or P-SEAMLDR is busy with another SEAMCALL.  %rax is not
     * changed in this case.
     *
     * Set %rax to TDX_SEAMCALL_VMFAILINVALID for VMfailInvalid.
     * This value will never be used as actual SEAMCALL error code as
     * it is from the Reserved status code class.
     */
    jnc .Lseamcall_out //jnc 当 CF 不为 1 时跳转，即 seamcall 没出错
    mov $TDX_SEAMCALL_VMFAILINVALID, %rax // CF=1，VMfailInvalid，设置 %rax 为特定值
    jmp .Lseamcall_out //继续跳转
2:
    /*
     * SEAMCALL caused #GP or #UD.  By reaching here %eax contains
     * the trap number.  Convert the trap number to the TDX error
     * code by setting TDX_SW_ERROR to the high 32-bits of %rax.
     *
     * Note cannot OR TDX_SW_ERROR directly to %rax as OR instruction
     * only accepts 32-bit immediate at most.
     */
    //SEAMCALL 导致 #GP 或 #UD。到达此处，%eax 包含异常号
    mov $TDX_SW_ERROR, %r12 //通过将 TDX_SW_ERROR 设置为 %rax 的高 32 位，将异常号转换为 TDX 错误代码
    orq %r12, %rax //注意不能将 TDX_SW_ERROR（64 位）直接与 %rax 进行 OR 运算，因为 OR 指令最多只接受 32 位立即数

    _ASM_EXTABLE_FAULT(1b, 2b) //标号 2 的代码修正标号 1 产生的异常
.Lseamcall_out:
    .else //伪变量 host 不为 1，说明是 guest 在执行该宏，所以是 tdcall
    tdcall
    .endif

    /*
     * Fetch output pointer from stack to R12 (It is used
     * as temporary storage)
     */
    pop %r12 //从栈上弹出 struct tdx_module_output 的指针到 %r12

    /*
     * Since this macro can be invoked with NULL as an output pointer,
     * check if caller provided an output struct before storing output
     * registers.
     *
     * Update output registers, even if the call failed (RAX != 0).
     * Other registers may contain details of the failure.
     */
    test %r12, %r12 //由于可以使用 NULL 作为输出指针调用此宏，因此在存储输出寄存器之前检查调用者是否提供了 struct tdx_module_output
    jz .Lno_output_struct //如果没提供输出结构，跳过下面保存输出寄存器的过程
    //将输出到寄存器的值写入 struct tdx_module_output 对应的域
    /* Copy result registers to output struct: */
    movq %rcx, TDX_MODULE_rcx(%r12)
    movq %rdx, TDX_MODULE_rdx(%r12)
    movq %r8,  TDX_MODULE_r8(%r12)
    movq %r9,  TDX_MODULE_r9(%r12)
    movq %r10, TDX_MODULE_r10(%r12)
    movq %r11, TDX_MODULE_r11(%r12)

.Lno_output_struct:
    /* Restore the state of R12 register */
    pop %r12 //恢复之前保存在栈上的 caller 的 %r12
.endm
```
### `__seamcall()`
* `__seamcall()` 函数调用 ABI 和 SEAMCALL ABI 的对比见注释
* 比较简单，不做过多注释，注意 `host=1`
* arch/x86/virt/vmx/tdx/seamcall.S
```c
/*
 * __seamcall() - Host-side interface functions to SEAM software module
 *        (the P-SEAMLDR or the TDX module).
 *
 * Transform function call register arguments into the SEAMCALL register
 * ABI.  Return TDX_SEAMCALL_VMFAILINVALID if the SEAMCALL itself fails,
 * or the completion status of the SEAMCALL leaf function.  Additional
 * output operands are saved in @out (if it is provided by the caller).
 *
 *-------------------------------------------------------------------------
 * SEAMCALL ABI:
 *-------------------------------------------------------------------------
 * Input Registers:
 *
 * RAX                 - SEAMCALL Leaf number.
 * RCX,RDX,R8-R9       - SEAMCALL Leaf specific input registers.
 *
 * Output Registers:
 *
 * RAX                 - SEAMCALL completion status code.
 * RCX,RDX,R8-R11      - SEAMCALL Leaf specific output registers.
 *
 *-------------------------------------------------------------------------
 *
 * __seamcall() function ABI:
 *
 * @fn  (RDI)          - SEAMCALL Leaf number, moved to RAX
 * @rcx (RSI)          - Input parameter 1, moved to RCX
 * @rdx (RDX)          - Input parameter 2, moved to RDX
 * @r8  (RCX)          - Input parameter 3, moved to R8
 * @r9  (R8)           - Input parameter 4, moved to R9
 *
* @out (R9)           - struct tdx_module_output pointer
 *           stored temporarily in R12 (not
 *           used by the P-SEAMLDR or the TDX
 *           module). It can be NULL.
 *
 * Return (via RAX) the completion status of the SEAMCALL, or
 * TDX_SEAMCALL_VMFAILINVALID.
 */
SYM_FUNC_START(__seamcall)
    FRAME_BEGIN
    TDX_MODULE_CALL host=1
    FRAME_END
    RET      //见 commit c796f02162e4 ("x86/tdx: Fix RETs in TDX asm")
SYM_FUNC_END(__seamcall)
EXPORT_SYMBOL_GPL(__seamcall)
```

### TD VM 的通信接口
* `__tdx_module_call()`：用于与 TDX module 通信（通过 `TDCALL` 指令）
* `__tdx_hypercall()`：guest 用于从 VMM 请求服务（通过 `TDCALL` 的 `TDVMCALL` 叶子函数）
* `_tdx_hypercall()`：对 `__tdx_hypercall()` 的封装 ，添加了对 `TDCALL` 失败的错误处理支持

#### `__tdx_module_call()`
* `__tdx_module_call()` 函数调用 ABI 和 TDCALL ABI 的对比见注释
* 比较简单，不做过多注释，注意 `host=0`
* arch/x86/coco/tdx/tdcall.S
```c
/*
 * __tdx_module_call()  - Used by TDX guests to request services from
 * the TDX module (does not include VMM services) using TDCALL instruction.
 *
 * Transforms function call register arguments into the TDCALL register ABI.
 * After TDCALL operation, TDX module output is saved in @out (if it is
 * provided by the user).
 *
 *-------------------------------------------------------------------------
 * TDCALL ABI:
 *-------------------------------------------------------------------------
 * Input Registers:
 *
 * RAX                 - TDCALL Leaf number.
 * RCX,RDX,R8-R9       - TDCALL Leaf specific input registers.
 *
 * Output Registers:
 *
 * RAX                 - TDCALL instruction error code.
 * RCX,RDX,R8-R11      - TDCALL Leaf specific output registers.
 *
 *-------------------------------------------------------------------------
 *
 * __tdx_module_call() function ABI:
 *
 * @fn  (RDI)          - TDCALL Leaf ID,    moved to RAX
 * @rcx (RSI)          - Input parameter 1, moved to RCX
 * @rdx (RDX)          - Input parameter 2, moved to RDX
 * @r8  (RCX)          - Input parameter 3, moved to R8
 * @r9  (R8)           - Input parameter 4, moved to R9
 *
 * @out (R9)           - struct tdx_module_output pointer
 *                       stored temporarily in R12 (not
 *                       shared with the TDX module). It
 *                       can be NULL.
 *
 * Return status of TDCALL via RAX.
 */
SYM_FUNC_START(__tdx_module_call)
    FRAME_BEGIN
    TDX_MODULE_CALL host=0
    FRAME_END
    RET
SYM_FUNC_END(__tdx_module_call)
```
#### `__tdx_hypercall()`
##### TD VMCALL ABI
* 输入寄存器

输入寄存器 | 功能
----------|----------------------------------------------------------
RAX       | `TDCALL` 指令 leaf number（需设为 `0` 表示 `TDG.VP.VMCALL`）
RCX       | 控制哪些 TD Guest GPR 按原样传递给 VMM 并返回的位图
R10       | 设置 `0` 表示 `TDCALL` 遵循标准的 TDX ABI 规范。非零值表示供应商特定的 ABI。
R11       | `VMCALL` 子 function number
RBX, RBP, RDI, RSI | 用于传递 `VMCALL` 子函数特定参数
R8-R9, R12-R15     | 同上

* 输出参数

输出寄存器 | 功能
----------|------------------------------------------
RAX       | `TDCALL` 指令执行的状态（而不是 hypercall 的输出）
R10       | Hypercall 输出错误码
R11-R15   | Hypercall 子函数特定的输出值

##### `__tdx_hypercall()` 函数 ABI
* 第一个输入参数为指向 `struct tdx_hypercall_args` 的指针，x86 函数调用 ABI，放到 `%rdi`
* 第二个输入参数为 `TDX_HCALL_* flags`，放到 `%rsi`
```c
/*
 * Used in __tdx_hypercall() to pass down and get back registers' values of
 * the TDCALL instruction when requesting services from the VMM.
 *
 * This is a software only structure and not part of the TDX module/VMM ABI.
 */
struct tdx_hypercall_args {
    u64 r8;
    u64 r9;
    u64 r10;
    u64 r11;
    u64 r12;
    u64 r13;
    u64 r14;
    u64 r15;
    u64 rdi;
    u64 rsi;
    u64 rbx;
    u64 rdx;
};

/* Used to request services from the VMM */
u64 __tdx_hypercall(struct tdx_hypercall_args *args, unsigned long flags);
```

##### `__tdx_hypercall()` 的实现
* arch/x86/coco/tdx/tdcall.S
```c
/*
 * __tdx_hypercall() - Make hypercalls to a TDX VMM using TDVMCALL leaf
 * of TDCALL instruction
 *
 * Transforms values in  function call argument struct tdx_hypercall_args @args
 * into the TDCALL register ABI. After TDCALL operation, VMM output is saved
 * back in @args.
 *
 *-------------------------------------------------------------------------
 * TD VMCALL ABI:
 *-------------------------------------------------------------------------
 *
 * Input Registers:
 *
 * RAX                 - TDCALL instruction leaf number (0 - TDG.VP.VMCALL)
 * RCX                 - BITMAP which controls which part of TD Guest GPR
 *                       is passed as-is to the VMM and back.
 * R10                 - Set 0 to indicate TDCALL follows standard TDX ABI
 *                       specification. Non zero value indicates vendor
 *                       specific ABI.
 * R11                 - VMCALL sub function number
 * RBX, RBP, RDI, RSI  - Used to pass VMCALL sub function specific arguments.
 * R8-R9, R12-R15      - Same as above.
 *
 * Output Registers:
 *
 * RAX                 - TDCALL instruction status (Not related to hypercall
 *                        output).
 * R10                 - Hypercall output error code.
 * R11-R15             - Hypercall sub function specific output values.
 *
 *-------------------------------------------------------------------------
  *
 * __tdx_hypercall() function ABI:
 *
 * @args  (RDI)        - struct tdx_hypercall_args for input and output
 * @flags (RSI)        - TDX_HCALL_* flags
 *
 * On successful completion, return the hypercall error code.
 */
SYM_FUNC_START(__tdx_hypercall)
    FRAME_BEGIN
    //callee-saved，被调用者使用前先保存的 GPRs
    /* Save callee-saved GPRs as mandated by the x86_64 ABI */
    push %r15
    push %r14
    push %r13
    push %r12
    push %rbx
    //__tdx_hypercall() 用了 %rdi 和 %rsi，我们现在也想用，所以把旧的值保存起来
    /* Free RDI and RSI to be used as TDVMCALL arguments */
    movq %rdi, %rax //指向 struct tdx_hypercall_args 的指针传输到 %rax
    push %rsi       //flags 保存在栈上
    //入参 struct tdx_hypercall_args 中指定的 GPRs 的值放到真正的寄存器
    /* Copy hypercall registers from arg struct: */
    movq TDX_HYPERCALL_r8(%rax),  %r8
    movq TDX_HYPERCALL_r9(%rax),  %r9
    movq TDX_HYPERCALL_r10(%rax), %r10
    movq TDX_HYPERCALL_r11(%rax), %r11
    movq TDX_HYPERCALL_r12(%rax), %r12
    movq TDX_HYPERCALL_r13(%rax), %r13
    movq TDX_HYPERCALL_r14(%rax), %r14
    movq TDX_HYPERCALL_r15(%rax), %r15
    movq TDX_HYPERCALL_rdi(%rax), %rdi
    movq TDX_HYPERCALL_rsi(%rax), %rsi
    movq TDX_HYPERCALL_rbx(%rax), %rbx
    movq TDX_HYPERCALL_rdx(%rax), %rdx
    //指向 struct tdx_hypercall_args 的指针保存在栈上
    push %rax
    //%rax 设为 0 表示要做 TDG.VP.VMCALL，TDVMCALL 在 TDCALL 中的 leaf ID 是 0
    /* Mangle function call ABI into TDCALL ABI: */
    /* Set TDCALL leaf ID (TDVMCALL (0)) in RAX */
    xor %eax, %eax //%rax 清零
    //需要暴露（传递）给 VMM 的 TD guest GPRs 的位图放入 %rcx
    movl $TDVMCALL_EXPOSE_REGS_MASK, %ecx
    //输入准备好了，发起 TDVMCALL
    tdcall
    //执行完后 %rax 不为 0 意味着 TDVMCALL 机制本身以及 TDX module 出现了严重错误
    /*
     * RAX!=0 indicates a failure of the TDVMCALL mechanism itself and that
     * something has gone horribly wrong with the TDX module.
     *
     * The return status of the hypercall operation is in a separate
     * register (in R10). Hypercall errors are a part of normal operation
     * and are handled by callers.
     */
    testq %rax, %rax //测试 tdcall 调用是否成功，test 相当于 and，只是不改变一些 rflags
    jne .Lpanic      //不为 0，TD guest 需要 panic
    //tdcall 执行没问题，将指向 struct tdx_hypercall_args 的指针从栈上弹出到 %rax
    pop %rax
    //此时栈顶是 TDX_HCALL_* flags，如果它的值是 1，说明期望拷贝 TDVMCALL 的输出
    /* Copy hypercall result registers to arg struct if needed */
    testq $TDX_HCALL_HAS_OUTPUT, (%rsp)
    jz .Lout //为 0 则跳转，跳过下面拷贝 GPRs 到输出参数的过程
    //VMM 的返回值放入 struct tdx_hypercall_args 实例对应的域
    movq %r8,  TDX_HYPERCALL_r8(%rax)
    movq %r9,  TDX_HYPERCALL_r9(%rax)
    movq %r10, TDX_HYPERCALL_r10(%rax)
    movq %r11, TDX_HYPERCALL_r11(%rax)
    movq %r12, TDX_HYPERCALL_r12(%rax)
    movq %r13, TDX_HYPERCALL_r13(%rax)
    movq %r14, TDX_HYPERCALL_r14(%rax)
    movq %r15, TDX_HYPERCALL_r15(%rax)
    movq %rdi, TDX_HYPERCALL_rdi(%rax)
    movq %rsi, TDX_HYPERCALL_rsi(%rax)
    movq %rbx, TDX_HYPERCALL_rbx(%rax)
    movq %rdx, TDX_HYPERCALL_rdx(%rax)
.Lout:
    /* TDVMCALL leaf return code is in R10 */
    movq %r10, %rax //TDVMCALL leaf 返回码是放在 %r10，把它转到 %rax 作为 __tdx_hypercall() 函数调用的返回值
    //将暴露给 VMM 的寄存器清零，以避免使用 VMM 控制的值进行推测执行，不清零的寄存器是因为它们的值会从栈上恢复
    /*
     * Zero out registers exposed to the VMM to avoid speculative execution
     * with VMM-controlled values. This needs to include all registers
     * present in TDVMCALL_EXPOSE_REGS_MASK, except RBX, and R12-R15 which
     * will be restored.
     */
    xor %r8d,  %r8d
    xor %r9d,  %r9d
    xor %r10d, %r10d
    xor %r11d, %r11d
    xor %rdi,  %rdi
    xor %rdx,  %rdx
    //此时栈顶是 TDX_HCALL_* flags，将它清理出栈，还放回 %rsi
    /* Remove TDX_HCALL_* flags from the stack */
    pop %rsi
    // 恢复之前被调用者 callee-saved 保存的 GPRs
    /* Restore callee-saved GPRs as mandated by the x86_64 ABI */
    pop %rbx
    pop %r12
    pop %r13
    pop %r14
    pop %r15

    FRAME_END
    //__tdx_hypercall() 函数调用返回。见 commit c796f02162e4 ("x86/tdx: Fix RETs in TDX asm")
    RET
.Lpanic:
    call __tdx_hypercall_failed //调用 panic("TDVMCALL failed. TDX module bug?")，因此该函数无法返回
    /* __tdx_hypercall_failed never returns */
    REACHABLE   //压制 objtool 的警告而加的标识
    jmp .Lpanic //如果意外能跳到这，回到上面的标号 .Lpanic
SYM_FUNC_END(__tdx_hypercall)
```
#### `_tdx_hypercall()`
* 再来看 `_tdx_hypercall()` 就非常简单了
```c
/*
 * Wrapper for standard use of __tdx_hypercall with no output aside from
 * return code.
 */
static inline u64 _tdx_hypercall(u64 fn, u64 r12, u64 r13, u64 r14, u64 r15)
{
    struct tdx_hypercall_args args = {
        .r10 = TDX_HYPERCALL_STANDARD, //表明是 TDVMCALL，%r10 为 0
        .r11 = fn, //TDVMCALL 的子功能
        .r12 = r12,
        .r13 = r13,
        .r14 = r14,
        .r15 = r15,
    };
    //flags 为 0 表示不期望得到 tdcall 返回的输出
    return __tdx_hypercall(&args, 0);
}
```
## Confidential Computing (CC) 的 API 一些适配
* AMD 和 Intel 都在页表中使用一个位来指示页的加密状态
  - 对于 AMD *set* 该位表示页面已加密
  - 对于 Intel *clear* 该位表示加密
  * 修改 `cc_mkenc()` 和 `cc_mkenc()`，当 `vendor` 是 `CC_VENDOR_INTEL` 清除或设置该位
* 加密位的信息来自 TDX 信息获取函数 `tdx_parse_tdinfo()`
  * arch/x86/coco/tdx/tdx.c
```c
static void tdx_parse_tdinfo(u64 *cc_mask)
{
    struct tdx_module_output out;
    unsigned int gpa_width;
    u64 td_attr;
    //其实用的是 TDCALL(TDG.VP.INFO) 来获取以下信息
    /*
     * TDINFO TDX module call is used to get the TD execution environment
     * information like GPA width, number of available vcpus, debug mode
     * information, etc. More details about the ABI can be found in TDX
     * Guest-Host-Communication Interface (GHCI), section 2.4.2 TDCALL
     * [TDG.VP.INFO].
     */
    tdx_module_call(TDX_GET_INFO, 0, 0, 0, 0, &out);
    //GPA 的最高位为 sharing 位，设置时为共享页面，清除时为私有页面
    /*
     * The highest bit of a guest physical address is the "sharing" bit.
     * Set it for shared pages and clear it for private pages.
     *
     * The GPA width that comes out of this call is critical. TDX guests
     * can not meaningfully run without it.
     */
    gpa_width = out.rcx & GENMASK(5, 0);//该 TD 的有效 GPA 宽度（用 bits 表示）
    *cc_mask = BIT_ULL(gpa_width - 1); //GPA 中的 SHARED bit 为 GPAW - 1，当前为 48 或 52

    /*
     * The kernel can not handle #VE's when accessing normal kernel
     * memory.  Ensure that no #VE will be delivered for accesses to
     * TD-private memory.  Only VMM-shared memory (MMIO) will #VE.
     */
    td_attr = out.rdx; //TD 的 ATTRIBUTES（作为 TDH.MNG.INIT 的输入提供）
    if (!(td_attr & ATTR_SEPT_VE_DISABLE)) {
        const char *msg = "TD misconfiguration: SEPT_VE_DISABLE attribute must be set.";

        /* Relax SEPT_VE_DISABLE check for debug TD. */
        if (td_attr & ATTR_DEBUG)
            pr_warn("%s\n", msg);
        else
            tdx_panic(msg);
    }
}
```
* 当 TD VM 访问普通的内核内存时，内核无法处理 `#VE`。为了确保不会为访问 TD 私有内存递交 `#VE` 而设置 TD Attributes 的 `ATTR_SEPT_VE_DISABLE`
  * 只有与 VMM 共享内存 (MMIO) 的场景才需要 `#VE`
* TD Attributes 的定义见 *TDX module spec, Chapter 20.2 TD Parameters Types, 20.2.1 ATTRIBUTES*
* 所有普通的内核内存都是“TDX 私有内存”，这包括从内核堆栈到内核文本的所有内容
  * 处理对内核内存的任意访问的异常基本上是不可能的，因为它们可能发生在内核进入/退出等非常讨厌的地方
  * 但是，TDX 硬件理论上可以在对私有内存的任何访问上 *传递* 虚拟化异常（`#VE`）
* 但是，它并不像听起来那么糟糕。TDX 可以配置为 *在具有称为 `ATTR_SEPT_VE_DISABLE` 的“TD Attributes”的私有内存上* 从不传递这些异常
  * Guest 无法 *设置* 此属性，但可以检查它
  * TD guest 需要确保在早期启动时设置了 `ATTR_SEPT_VE_DISABLE`，如果未设置则 `panic()`
  * 对于 Linux 来说，没有任何明智的方法可以在清除此属性的情况下运行，因此 `panic()` 是合适的
* 在检查内核具有早期 `#VE` 处理程序的位置之前，引导期间有一个小窗口。但是处理程序仅用于 port I/O，并且一旦它看到任何其他 `#VE`，例如由于私有内存访问生成的 `#VE`，也会 `panic()`

* Qemu 参数 `-object tdx-guest,id=tdx,sept-ve-disable=on`
* Qemu 对 `TDX_TD_ATTRIBUTES_SEPT_VE_DISABLE` 的定义和处理：
  * qemu/target/i386/kvm/tdx.c
```c
#define TDX_TD_ATTRIBUTES_DEBUG             BIT_ULL(0)
#define TDX_TD_ATTRIBUTES_SEPT_VE_DISABLE   BIT_ULL(28)
#define TDX_TD_ATTRIBUTES_PKS               BIT_ULL(30)
#define TDX_TD_ATTRIBUTES_PERFMON           BIT_ULL(63)
...
static void tdx_guest_init(Object *obj)
{
    TdxGuest *tdx = TDX_GUEST(obj);

    qemu_mutex_init(&tdx->lock);
    //初始值就是 TDX_TD_ATTRIBUTES_SEPT_VE_DISABLE
    tdx->attributes = TDX_TD_ATTRIBUTES_SEPT_VE_DISABLE;

    object_property_add_bool(obj, "sept-ve-disable",
                             tdx_guest_get_sept_ve_disable,
                             tdx_guest_set_sept_ve_disable);
...
}
```
* 在 `tdx_early_init()` 中 通过 `tdx_parse_tdinfo(&cc_mask)` 得到 GPAW 后通过 `cc_set_mask(cc_mask)` 设置 `cc_mask`
```c
void __init tdx_early_init(void)
{
    u64 cc_mask;
    u32 eax, sig[3];

    cpuid_count(TDX_CPUID_LEAF_ID, 0, &eax, &sig[0], &sig[2],  &sig[1]);
    //注意这里，TD 内核区分自己是 host 还是 guest 的分水岭
    if (memcmp(TDX_IDENT, sig, sizeof(sig)))
        return;
    //以下设置的标识会影响到 host 还是 guest 的判断
    setup_force_cpu_cap(X86_FEATURE_TDX_GUEST);

    cc_set_vendor(CC_VENDOR_INTEL); //设置全局的 enum cc_vendor vendor，TD host 不会设置 vendor
    tdx_parse_tdinfo(&cc_mask);
    cc_set_mask(cc_mask); //设置全局的 cc_mask
    //这个修改见下面《禁用 TD Guest 的 `NOTIFY_ENABLES` 功能》
    /* Kernel does not use NOTIFY_ENABLES and does not need random #VEs */
    tdx_module_call(TDX_WR, 0, TDCS_NOTIFY_ENABLES, 0, -1ULL, NULL);
    //GPAW 里偷了一位作为 shared bit，因此实际物理地址宽度为 47 或 51
    /*
     * All bits above GPA width are reserved and kernel treats shared bit
     * as flag, not as part of physical address.
     *
     * Adjust physical mask to only cover valid GPA bits.
     */
    physical_mask &= cc_mask - 1;

    x86_platform.guest.enc_cache_flush_required = tdx_cache_flush_required;
    x86_platform.guest.enc_tlb_flush_required   = tdx_tlb_flush_required;
    x86_platform.guest.enc_status_change_finish = tdx_enc_status_changed;

    pr_info("Guest detected\n");
}
```
* 在 x86 ARCH 代码中，`__PHYSICAL_MASK` 宏表示给定架构中物理地址的宽度。它用于为内核中的地址位创建 *物理 PAGE_MASK* 的时候
  * arch/x86/include/asm/page_types.h
```c
/* PAGE_SHIFT determines the page size */
#define PAGE_SHIFT      12
#define PAGE_SIZE       (_AC(1,UL) << PAGE_SHIFT)
#define PAGE_MASK       (~(PAGE_SIZE-1))
...
/* Cast P*D_MASK to a signed type so that it is sign-extended if
   virtual addresses are 32-bits but physical addresses are larger
   (ie, 32-bit PAE). */
#define PHYSICAL_PAGE_MASK  (((signed long)PAGE_MASK) & __PHYSICAL_MASK)
#define PHYSICAL_PMD_PAGE_MASK  (((signed long)PMD_MASK) & __PHYSICAL_MASK)
#define PHYSICAL_PUD_PAGE_MASK  (((signed long)PUD_MASK) & __PHYSICAL_MASK)
```
* 由于在 TDX guest 中，单个位用作元数据（shared bit），因此需要将其从有效物理地址位中排除，以避免在内核中使用不正确的地址位
* 使能 `CONFIG_INTEL_TDX_GUEST` 也使能 `CONFIG_DYNAMIC_PHYSICAL_MASK`，以使宏 `__PHYSICAL_MASK` 替换为全局变量 `physical_mask`
```c
#ifdef CONFIG_DYNAMIC_PHYSICAL_MASK
extern phys_addr_t physical_mask;
#define __PHYSICAL_MASK     physical_mask
#else
#define __PHYSICAL_MASK     ((phys_addr_t)((1ULL << __PHYSICAL_MASK_SHIFT) - 1))
#endif
```

## `#VE` 的支持
* 由于可能在用户空间或内核中发生的特定 guest 操作，**虚拟化异常（`#VE`）** 被传递给 TDX guest：
  * 特定的指令（以 `WBINVD` 为例）
  * 特定的 MSR 访问
  * 特定 `CPUID` leaf 访问
  * 访问特定的 guest 物理地址
* 由于以下原因，内核在系统调用间隙和 NMI 入口代码期间避免了 `#VE`
  * 系统调用入口代码有一个 critical 的窗口期，此时内核堆栈尚未设置。此窗口中的任何异常都会导致难以调试的问题，并可被利用来提升权限
  * NMI 入口代码中的异常也会导致问题。使用 `IRET` 从异常处理程序返回将重新启用 NMI，而嵌套的 NMI 将破坏 `NMI` 堆栈
* 入口代码路径不访问 TD 共享内存、MMIO 区域、使用 `#VE` 触发的 MSR、指令或 `CPUID` leaves 等可能生成 `#VE` 的操作
* VMM 可以随时从 TD 中删除内存，但访问未接受（或丢失）的私有内存会导致 VM 终止，而不是 `#VE`
* 与缺页和断点类似，一旦内核准备好处理嵌套的 NMI，`#VE` 就可以在 NMI 处理程序中使用
* 在 `#VE` 交付期间，所有中断（包括 NMI）都被阻止，直到调用 `TDG.VP.VEINFO.GET`。它会阻止 `#VE` 嵌套，直到内核读取 VE 信息
* `TDG.VP.VEINFO.GET` 从 TDX module 中检索 `#VE` 信息，这也会清除 **“#VE valid”** 标志。这必须在任何其他事情之前完成，因为在设置有效标志时发生的任何 `#VE` 都会被 TDX module 升级为 `#DF`。这将导致 OOPs
* 如果设置了 `#VE` 有效标志，虚拟 NMI 将被禁止。在调用 `TDG.VP.VEINFO.GET` 之前，不会递交 NMI
* 现在，通过直接调用 `ve_raise_fault()` 转换未处理的 `#VE`（所有内容，直到本系列的后面部分），使它们看起来就像 `#GP`
* `ve_raise_fault()` 函数类似于 `#GP` 处理程序，负责将 `SIGSEGV` 发送到用户空间和 CPU die，并通知调试器和其他 die chain 用户

### `#VE` 支持的基础框架
* 声明 `#VE` 处理函数的入口，`X86_TRAP_VE` 的异常向量号是 `20`
  * arch/x86/include/asm/idtentry.h
```c
#ifdef CONFIG_INTEL_TDX_GUEST
DECLARE_IDTENTRY(X86_TRAP_VE,       exc_virtualization_exception);
#endif
```
* 设置 `#VE` 处理函数的入口
  * arch/x86/kernel/idt.c
```c
/*
 * Early traps running on the DEFAULT_STACK because the other interrupt
 * stacks work only after cpu_init().
 */
static const __initconst struct idt_data early_idts[] = {
...
#ifdef CONFIG_INTEL_TDX_GUEST
    INTG(X86_TRAP_VE,       asm_exc_virtualization_exception),
#endif
};
```
* `#VE` 处理入口函数 `exc_virtualization_exception()` 的定义
  * arch/x86/kernel/traps.c
```c
DEFINE_IDTENTRY(exc_virtualization_exception)
{
    struct ve_info ve;
    //在执行 TDGETVEINFO TDCALL 之前，NMI/MC/IRQ 将处于禁用状态。这确保 VE 信息不会被嵌套的#VE 覆盖
    /*
     * NMIs/Machine-checks/Interrupts will be in a disabled state
     * till TDGETVEINFO TDCALL is executed. This ensures that VE
     * info cannot be overwritten by a nested #VE.
     */
    tdx_get_ve_info(&ve);

    cond_local_irq_enable(regs);
    //如果无法处理成功，则将其视为 #GP(0) 处理
    /*
     * If tdx_handle_virt_exception() could not process
     * it successfully, treat it as #GP(0) and handle it.
     */
    if (!tdx_handle_virt_exception(regs, &ve))
        ve_raise_fault(regs, 0); //error code 设为 0

    cond_local_irq_disable(regs);
}
```
* `#VE` 处理函数 `tdx_handle_virt_exception()`
```c
bool tdx_handle_virt_exception(struct pt_regs *regs, struct ve_info *ve)
{
    int insn_len;

    if (user_mode(regs)) //在用户态发生的 #VE
        insn_len = virt_exception_user(regs, ve);
    else                 //在内核态发生的 #VE
        insn_len = virt_exception_kernel(regs, ve);
    if (insn_len < 0) //无法处理该 #VE
        return false;
    //#VE 类似于 trap，处理完后需要让 %ip 前进到下一条指令
    /* After successful #VE handling, move the IP */
    regs->ip += insn_len;

    return true;
}
```
* 用户态的引发的 `#VE` 的处理函数 `virt_exception_user()`，目前仅有 `CPUID` 指令
```c
/*
 * Handle the user initiated #VE.
 *
 * On success, returns the number of bytes RIP should be incremented (>=0)
 * or -errno on error.
 */
static int virt_exception_user(struct pt_regs *regs, struct ve_info *ve)
{
    switch (ve->exit_reason) {
    case EXIT_REASON_CPUID:
        return handle_cpuid(regs, ve);
    default:
        pr_warn("Unexpected #VE: %lld\n", ve->exit_reason);
        return -EIO;
    }
}
```
* 内核态的引发的 `#VE` 的处理函数 `virt_exception_user()`
```c
/*
 * Handle the kernel #VE.
 *
 * On success, returns the number of bytes RIP should be incremented (>=0)
 * or -errno on error.
 */
static int virt_exception_kernel(struct pt_regs *regs, struct ve_info *ve)
{
    switch (ve->exit_reason) {
    case EXIT_REASON_HLT:           //halt 指令
        return handle_halt(ve);
    case EXIT_REASON_MSR_READ:      //rdmsr 指令
        return read_msr(regs, ve);
    case EXIT_REASON_MSR_WRITE:     //wrmsr 指令
        return write_msr(regs, ve);
    case EXIT_REASON_CPUID:         //cpuid 指令
        return handle_cpuid(regs, ve);
    case EXIT_REASON_EPT_VIOLATION: //MMIO 导致的 EPT violation
        if (is_private_gpa(ve->gpa))//导致 EPT 违规的 GPA 必定不能是私有内存
            panic("Unexpected EPT-violation on private memory.");
        return handle_mmio(regs, ve);
    case EXIT_REASON_IO_INSTRUCTION://in/out 等 port I/O 指令
        return handle_io(regs, ve);
    default:
        pr_warn("Unexpected #VE: %lld\n", ve->exit_reason);
        return -EIO;
    }
}
```
* 无法处理常规的 `#VE` 的处理
```c
#define VE_FAULT_STR "VE fault"

static void ve_raise_fault(struct pt_regs *regs, long error_code)
{   //用户态发生的 #VE，发信号 SIGSEGV 杀死进程
    if (user_mode(regs)) {
        gp_user_force_sig_segv(regs, X86_TRAP_VE, error_code, VE_FAULT_STR);
        return;
    }
    //内核态发生的 #VE，触发 CPU die，并通知调试器和其他 die chain 用户
    if (gp_try_fixup_and_notify(regs, X86_TRAP_VE, error_code, VE_FAULT_STR))
        return;
    //内核 oops
    die_addr(VE_FAULT_STR, regs, error_code, 0);
}
```
## 对 TDX Guest `hlt` 的支持
* `HLT` 指令是特权指令，执行它会停止指令执行并将处理器置于 **HALT** 状态
  * 它在内核中用于重启、空闲循环和异常修复处理程序等情况
  * 对于空闲的情况，将在 `HLT` 指令（也称为 `safe_halt()`）之前启用中断（使用 `STI` 指令）
* 要在 TDX guest 中支持 `HLT` 指令，需要使用 `TDVMCALL`（对 VMM 的 hypercall）对其仿真
  * 请参阅 *Intel Trust Domain Extensions (Intel TDX) Guest-Host-Communication Interface (GHCI) 规范* 的 *TDVMCALL [Instruction.HLT]* 部分
* 在 TDX guests 中，执行 `HLT` 指令会生成一个 `#VE`，用于模拟 `HLT` 指令
 * 但是基于 `#VE` 的仿真不适用于 `safe_halt()` 风格，因为它需要在 `TDCALL` 之前执行 `STI` 指令
 * 由于 idle loop 是 `safe_halt()` 变体的唯一用户，因此将其作为特殊情况处理
* 为避免在 idle 函数中调用 `safe_halt()`，定义 `tdx_guest_idle()` 并使用它覆盖有效 TDX guest 的 `x86_idle` 函数指针
* 已考虑添加 `safe_halt()` 支持的替代选择，如 PV ops。但被拒绝了，因为 HLT paravirt 调用仅存在于 PARAVIRT_XXL 下，并且在 TDX guest 中仅针对 `safe_halt()` 用例场景是不值得的
```c
void select_idle_routine(const struct cpuinfo_x86 *c)
{
...
    if (prefer_mwait_c1_over_halt()) {
        pr_info("using mwait in idle threads\n");
        static_call_update(x86_idle, mwait_idle);
    } else if (cpu_feature_enabled(X86_FEATURE_TDX_GUEST)) {
        pr_info("using TDX aware idle routine\n");
        static_call_update(x86_idle, tdx_halt);
    } else {
        static_call_update(x86_idle, default_idle);
    }
...
}
```
* `safe_halt` 语义指的是 halt 前需要开启中断，然后再 `hlt`，恢复后再关闭中断，例如：
  * `mwait_idle()` 会调 `__sti_mwait() -> "sti; mwait"`，待到恢复后再关闭中断
  * `default_idle()` 会调 `arch_safe_halt() -> native_safe_halt() -> "sti; hlt"`，待到后再关闭中断
* Commit e78a7614f3876ac649b3df608789cb6ef74d0480 将 `arch_cpu_idle()` 置于 IRQ disabled 的环境下：
```c
commit e78a7614f3876ac649b3df608789cb6ef74d0480
Author: Peter Zijlstra <peterz@infradead.org>
Date:   Wed Jun 5 07:46:43 2019 -0700

    idle: Prevent late-arriving interrupts from disrupting offline
```
* 于是 commit e80a48bade619ec5a92230b3d4ae84bfc2746822 将 `TDX_HCALL_ISSUE_STI` 相关的内容都移除了，因为 `__halt(irq_disabled)` 的参数 `irq_disabled` 才是告知 VMM 的中断开启状态的关键，而 TD guest 发出 tdcall 时中断的真实状态是关闭的
  * TD guest 中关闭中断的原因是 `safe_halt()` 函数是在 STI-shadow 状态下执行 `HLT` 指令，因此在执行 `TDCALL` 指令之前，必须保持中断请求（IRQ）禁用状态，以确保挂起的中断请求能被正确识别为唤醒事件。
  * VMM 对于 idle halt 会按 IRQ 开启的状态来处理，即虚拟中断来时会调度 vCPU
```c
commit e80a48bade619ec5a92230b3d4ae84bfc2746822
Author: Peter Zijlstra <peterz@infradead.org>
Date:   Thu Jan 12 20:43:36 2023 +0100

    x86/tdx: Remove TDX_HCALL_ISSUE_STI
```
* 于是 `__tdx_hypercall()` 就不再有关于 `tdcall` 前是否需要 `sti` 的判断了
```c
void __cpuidle tdx_safe_halt(void)
{   //idle 函数调用到此时，中断必然已被关闭
    const bool irq_disabled = false;
    //第一个参数是传递给 VMM 的，按照 safe_halt 语义要求 hlt 前中断是开启的，所以告诉 VMM 的中断处于开启状态
    /*
     * Use WARN_ONCE() to report the failure.
     */
    if (__halt(irq_disabled))
        WARN_ONCE(1, "HLT instruction emulation failed\n");
}
```
* 而他处执行 `hlt` 指令触发 `#VE` 弹射回来后通过 `handle_halt()` 请求模拟 TD guest halt
  * 发出警告的原因是，在 IRQ 启用的情况下 `HLT` 是不安全的。如果在处理 `HLT` 指令触发的 `#VE` 异常期间开启了中断请求（IRQ），恰好一个意图是唤醒事件的 IRQ 可能会在请求 `HLT` 模拟前被消费掉，让 vCPU 不确定地阻塞（被调度出去），导致 `HLT` 指令无限期执行
```c
static int handle_halt(struct ve_info *ve)
{   //不是 idle 函数调用过来的，需要获取一下中断的开关状态
    const bool irq_disabled = irqs_disabled();

    /*
     * HLT with IRQs enabled is unsafe, as an IRQ that is intended to be a
     * wake event may be consumed before requesting HLT emulation, leaving
     * the vCPU blocking indefinitely.
     */
    if (WARN_ONCE(!irq_disabled, "HLT emulation with IRQs enabled"))
        return -EIO;

    if (__halt(irq_disabled))
        return -EIO;

    return ve_instr_len(ve);
}
```
* `%r12` 用于传递调用 `TDG.VP.VMCALL<Instruction.HLT>` 时 TD guest 的中断状态
* 请求到达 VMM 后，它使用“IRQ disabled”参数来了解 TD guest 的 IRQ 启用状态（`RFLAGS.IF`），并确定有 IRQ 挂起（pending）时，它是否应该调度暂停的 vCPU
  * 例如，如果 IRQ 被禁用，VMM 可以将 vCPU 保持在虚拟 HLT 中，即使有 IRQ 挂起（pending），也不会挂起/打断 guest
  * 这么做是因为 KVM 无法通过读取加密的 TD VMCS，host 怎么知道 TD Exit 的 guest 是否有虚拟中断 pending，见后面的分析 
```c
static u64 __cpuidle __halt(const bool irq_disabled)
{
    struct tdx_hypercall_args args = {
        .r10 = TDX_HYPERCALL_STANDARD,
        .r11 = hcall_func(EXIT_REASON_HLT),
        .r12 = irq_disabled,
    };

    /*
     * Emulate HLT operation via hypercall. More info about ABI
     * can be found in TDX Guest-Host-Communication Interface
     * (GHCI), section 3.8 TDG.VP.VMCALL<Instruction.HLT>.
     *
     * The VMM uses the "IRQ disabled" param to understand IRQ
     * enabled status (RFLAGS.IF) of the TD guest and to determine
     * whether or not it should schedule the halted vCPU if an
     * IRQ becomes pending. E.g. if IRQs are disabled, the VMM
     * can keep the vCPU in virtual HLT, even if an IRQ is
     * pending, without hanging/breaking the guest.
     */
    return __tdx_hypercall(&args, 0);
}
```
* `select_idle_routine()` 设置的 `x86_idle()` 函数在如下路径被调用：
```c
//kernel/sched/idle.c
do_idle()
   //kernel/sched/idle.c
-> cpuidle_idle_call()
   if (cpuidle_not_available(drv, dev))
   -> default_idle_call()
      -> arch_cpu_idle()
         -> static_call(x86_idle)()
            //arch/x86/coco/tdx/tdx.c
         => tdx_safe_halt()
      //drivers/cpuidle/cpuidle.c
   -> call_cpuidle()
      -> cpuidle_enter()
         -> cpuidle_enter_state()
            -> default_idle_call()
```

### KVM 对 TDX 的 `hlt` 的模拟
* KVM 对 `hlt` 的模拟调用路径如下：
```c
//arch/x86/kvm/x86.c
vcpu_run()
-> vcpu_enter_guest(vcpu)
   -> kvm_x86_call(vcpu_run)(vcpu, run_flags)
      //arch/x86/kvm/vmx/tdx.c
   => tdx_vcpu_run()
   -> kvm_x86_call(handle_exit)(vcpu, exit_fastpath)
      //arch/x86/kvm/vmx/main.c
   => vt_handle_exit()
      -> tdx_handle_exit()
            case EXIT_REASON_HLT:
            //arch/x86/kvm/x86.c
         -> kvm_emulate_halt_noskip(vcpu)
            -> kvm_vcpu_has_events(vcpu)
               -> kvm_cpu_has_interrupt(vcpu)
                  -> kvm_x86_call(protected_apic_has_interrupt)(v)
                     //arch/x86/kvm/vmx/tdx.c
                  => tdx_protected_apic_has_interrupt()
            -> kvm_set_mp_state(vcpu, state) //把 vCPU 状态置为 KVM_MP_STATE_RUNNABLE，让外层循环去 halt vCPU
```
* `TDG.VP.VMCALL<Instruction.HLT>` 发出后，在 TDX module 运行期间中断是关闭的，此间进来的中断会造成 `RVI` 置位。只在 HALTED 且 IRQ 开启的场景检查 `RVI`
* 对于 non-HTL 的场景，KVM 则不在乎 `STI`/`SS` shadows。
* 如果中断在 TD exit 之前挂起的，那么 vCPU **必须** 被阻塞（调度走）
* 否则（中断在 TD exit 之后进来的），中断被当作在（模拟的 `hlt`）指令边界进来的来对待
```c
static bool tdx_protected_apic_has_interrupt(struct kvm_vcpu *vcpu)
{
    u64 vcpu_state_details;
    //如果有 pending 的 posted interrupt，返回有中断
    if (pi_has_pending_interrupt(vcpu))
        return true;

    /*
     * Only check RVI pending for HALTED case with IRQ enabled.
     * For non-HLT cases, KVM doesn't care about STI/SS shadows.  And if the
     * interrupt was pending before TD exit, then it _must_ be blocked,
     * otherwise the interrupt would have been serviced at the instruction
     * boundary.
     */
    if (vmx_get_exit_reason(vcpu).basic != EXIT_REASON_HLT ||
        to_tdx(vcpu)->vp_enter_args.r12)
        return false;

    vcpu_state_details =
        td_state_non_arch_read64(to_tdx(vcpu), TD_VCPU_STATE_DETAILS_NON_ARCH);

    return tdx_vcpu_state_details_intr_pending(vcpu_state_details);
}
```
* `td_state_non_arch_read64(to_tdx(vcpu), TD_VCPU_STATE_DETAILS_NON_ARCH)` 其实是用的 seamcall<`TDH.VP.RD`> 读取 TD 的 VCPU-scope metadata 中的 `TD_VCPU_STATE_DETAILS_NON_ARCH` 域
  * arch/x86/kvm/vmx/tdx.h
```c
#define TDX_BUILD_TDVPS_ACCESSORS(bits, uclass, lclass)             \
static __always_inline u##bits td_##lclass##_read##bits(struct vcpu_tdx *tdx,   \
                            u32 field)      \
{                                       \
    u64 err, data;                              \
                                        \
    tdvps_##lclass##_check(field, bits);                    \
    err = tdh_vp_rd(&tdx->vp, TDVPS_##uclass(field), &data);        \
    if (unlikely(err)) {                            \
        tdh_vp_rd_failed(tdx, #uclass, field, err);         \
        return 0;                           \
    }                                   \
    return (u##bits)data;                           \
}
...
TDX_BUILD_TDVPS_ACCESSORS(64, STATE_NON_ARCH, state_non_arch);
```
* `TD_VCPU_STATE_DETAILS_NON_ARCH` 域的定义见 vcpu_scope_metadata.json，其中 Bit 0 表示有一个虚拟中断 pending 等着递交
```json
{
  "Header": {
    "Copyright": "Copyright (c) 2022 - 2023 Intel Corporation. All rights reserved.",
    "Info": "TDX VCPU-Scope Metadata",
    "Version": "2.0"
  },
...
  "Fields": [
...
    {
      "Class": "Guest State",
      "Field Name": "VCPU_STATE_DETAILS",
      "Description": [
        "Bit 0:     INTR_PENDING:  Indicates",
        "           that a virtual interrupt",
        "           is pending delivery, i.e.",
        "           VMCS.RVI[7:4] >",
        "           TDVPS.VAPIC.VPPR[7:4]",
        "Bits 63:2: Reserved, set to 0"
      ],
      "Type": null,
      "Attributes": null,
      "VM Applicability": null,
      "Mutability": "Mutable",
      "Initial Value": [
        "N/A"
      ],
      "Field Size (Bytes)": "8",
      "Num Fields": "1",
      "Num Elements": "1",
      "Element Size (Bytes)": "8",
      "Overall Size (Bytes)": "8",
      "Base FIELD_ID (Hex)": "0x9120000300000100",
      "Host VMM Access for a Production TD": "ROS",
      "Host VMM Access for a Debug TD": "ROS",
      "Guest Access": "None",
      "Host VMM Rd Mask for a Production TD ": "-1",
      "Host VMM Wr Mask for a Production TD ": "0",
      "Host VMM Rd Mask for a Debug TD ": "-1",
      "Host VMM Wr Mask for a Debug TD ": "0",
      "Guest Rd Mask": "0",
      "Guest Wr Mask": "0"
    },
...
  ]
}
```
* 更详细的解释见 11.13.5. Pending Virtual Interrupt Delivery Indication, TDX Module Spec
* `TDVPS_STATE_NON_ARCH` 宏展开会用到如下定义：
  * arch/x86/kvm/vmx/tdx_arch.h
```c
/* TDX control structure (TDR/TDCS/TDVPS) field access codes */
#define TDX_NON_ARCH            BIT_ULL(63)
#define TDX_CLASS_SHIFT         56
#define TDX_FIELD_MASK          GENMASK_ULL(31, 0)

#define __BUILD_TDX_FIELD(non_arch, class, field)   \
    (((non_arch) ? TDX_NON_ARCH : 0) |      \
     ((u64)(class) << TDX_CLASS_SHIFT) |        \
     ((u64)(field) & TDX_FIELD_MASK))

#define BUILD_TDX_FIELD(class, field)           \
    __BUILD_TDX_FIELD(false, (class), (field))

#define BUILD_TDX_FIELD_NON_ARCH(class, field)      \
    __BUILD_TDX_FIELD(true, (class), (field))
...
#define TDVPS_CLASS_OTHER_GUEST     17ULL
...
#define TDVPS_STATE_NON_ARCH(field) BUILD_TDX_FIELD_NON_ARCH(TDVPS_CLASS_OTHER_GUEST, (field))
...
enum tdx_vcpu_guest_other_state {
    TD_VCPU_STATE_DETAILS_NON_ARCH = 0x100,
};
// Bit 0 表示是否有 pending 的虚拟中断
#define TDX_VCPU_STATE_DETAILS_INTR_PENDING BIT_ULL(0)

static inline bool tdx_vcpu_state_details_intr_pending(u64 vcpu_state_details)
{
    return !!(vcpu_state_details & TDX_VCPU_STATE_DETAILS_INTR_PENDING);
}
```
### TDX Module 怎知其执行期间有中断进来？
* 当用 seamcall<`TDH.VP.RD`> 读取 TD 的 VCPU-scope metadata 中的 `TD_VCPU_STATE_DETAILS_NON_ARCH` 域会调用如下路径：
```c
//src/vmm_dispatcher/tdx_vmm_dispatcher.c
tdx_vmm_dispatcher()
   case TDH_VP_RD_LEAF:
   //src/vmm_dispatcher/api_calls/tdh_vp_rd_wr.c
-> tdh_vp_rd(local_data->vmm_regs.rcx, field_code, leaf_opcode.version)
   -> tdh_vp_rd_wr(tdvpr_pa, field_code, local_data_ptr, false, 0, 0, version)
         //src/common/metadata_handlers/metadata_generic.c
      -> md_read_element(MD_CTX_VP, requested_field_id, access_type, access_qual, md_ctx, &rd_data)
            case MD_CTX_VP:
            //src/common/metadata_handlers/metadata_vp.c
         -> md_vp_read_element()
            -> md_vp_get_element()
               -> md_vp_get_element_special_rd_handle()
```
* 最后由 `md_vp_get_element_special_rd_handle()` 去查询 VMCS 中的 Guest 状态域中的中断状态
  * GIS 的 `RVI` 域是请求服务的最高优先级虚拟中断的向量，这个值是有虚拟中断评估过后写入的，可以认为是微码/硬件做的
  * `PPR` 是 `TPR` 和 `ISRV`（ISR 中设置的最高优先级位的向量号）中的较大者
  * `RVI > vCPU PPR` 认为 pending 的虚拟中断有效，写 `TDX_VCPU_STATE_DETAILS_INTR_PENDING` 位
  * src/common/metadata_handlers/metadata_vp.c
```c
//根据 vcpu_scope_metadata.json 自动生成的头文件 include/auto_gen_1_5/tdvps_fields_lookup.h
//里面定义了 TDVPS 域的 Guest State 类的编码
#define MD_TDVPS_GUEST_STATE_CLASS_CODE 17ULL

//VMCS 中 Guest 状态域中的 GGuest Interrupt Status 域的编码
//src/common/x86_defs/vmcs_defs.h
#define VMX_GUEST_INTERRUPT_STATUS_ENCODE  0x0810

//VCPU state details 的域 .vmxip 就是 VCPU-scope metadata 中的
//TD_VCPU_STATE_DETAILS_NON_ARCH 域的 TDX_VCPU_STATE_DETAILS_INTR_PENDING 位
//src/common/data_structures/tdx_tdvps.h
/**
 * @struct vcpu_state__t
 *
 * @brief vcpu state details is a virtual TDVPS field. It is calculated on read
 */
typedef union vcpu_state_s
{
    struct
    {
        uint64_t vmxip    : 1;  
        uint64_t reserved : 63;
    };
    uint64_t raw;
}vcpu_state_t;

//VCPU-scope metadata 的 Guest State 类对应的结构如下
//src/common/data_structures/tdx_tdvps.h
typedef struct tdvps_guest_state_s
{
    gprs_state_t gpr_state;
    uint64_t dr0;
    uint64_t dr1;
    uint64_t dr2;
    uint64_t dr3;
    uint64_t dr6;
    uint64_t xcr0;
    uint64_t cr2;
    uint8_t  reserved[8]; /**< Reserved for aligning the next field */
    uint128_t  iwk_enckey[2]; /**< Last KeyLocker IWK loader.  Cache line aligned */
    uint128_t  iwk_intkey;
    loadiwkey_ctl_t iwk_flags;
    uint8_t  reserved_2[4]; /**< Reserved for aligning the next field */
    vcpu_state_t vcpu_state_details;
} tdvps_guest_state_t;

//GSI 的结构定义如下
//src/common/x86_defs/vmcs_defs.h
typedef union guest_interrupt_status_u
{
    struct
    {
        uint64_t rvi      : 8;
        uint64_t svi      : 8;
        uint64_t reserved : 48;
    };
    uint64_t raw;
}guest_interrupt_status_t;

static uint64_t md_vp_get_element_special_rd_handle(md_field_id_t field_id, md_access_t access_type,
                                                    md_context_ptrs_t md_ctx, uint16_t vm_id, uint64_t read_value)
{
...
    //读取的是 Guest State 类
    else if (field_id.class_code == MD_TDVPS_GUEST_STATE_CLASS_CODE)
    {
        switch (field_id.field_code)
        {   //读取的是 Guest State 类的 VCPU state details 域
            case MD_TDVPS_VCPU_STATE_DETAILS_FIELD_CODE:
            {
                // Calculate virtual interrupt pending status
                vcpu_state_t vcpu_state_details;
                guest_interrupt_status_t interrupt_status;
                uint64_t interrupt_status_raw;
                //用指令 vmptrld 加载 VM ID = 0 的 VMCS
                set_vm_vmcs_as_active(md_ctx.tdvps_ptr, 0);
                //读取 VMCS 中 Guest 状态域中的 Guest Interrupt Status 域的值
                ia32_vmread(VMX_GUEST_INTERRUPT_STATUS_ENCODE, &interrupt_status_raw);
                interrupt_status.raw = (uint16_t)interrupt_status_raw;
                vcpu_state_details.raw = 0ULL;
                if ((interrupt_status.rvi & 0xF0UL) > (md_ctx.tdvps_ptr->vapic.vapic[PPR_INDEX] & 0xF0UL))
                {   //如果请求服务的最高优先级虚拟中断的向量 > vCPU 的 vPPR，pending 的虚拟中断有效
                    vcpu_state_details.vmxip = 1ULL;
                }
                read_value = vcpu_state_details.raw;

                break;
            }
            default:
                break;
        }
    }
...
}
```

## `wrmsr` 场景
* 在 TD guest 中执行 MSR 有两种可行的方法，目前采用的是第一种，因为第二种需要长期维护一个列表：
1. 像大多数 VM 和 bare metal 一样执行 `RDMSR/WRMSR` 指令。有些会成功，有些会导致 `#VE`。所有导致 `#VE` 的事件都将通过 `TDCALL` 处理
2. 使用 paravirt 基础设施。paravirt hook 必须保留一份列表，列出哪些 MSR 会导致 `#VE` 并使用 `TDCALL`。所有其他 MSR 直接执行 `RDMSR/WRMSR` 指令
* 对于性能关键的 MSR 写入（如 `TSC_DEADLINE`），有相关的 patch 将用 *直接 `TDCALL`* 替换 `WRMSR/#VE` 序列
### TD Guest `wrmsr`
* TD guest 调用 `wrmsr()` 并最终执行一条 `wrmsr` 指令
```c
wrmsr()
-> native_write_msr()
   -> __wrmsr()
      -> wrmsr
```
### TDX Module 对 WRMSR 的处理
* TDX module 初始化了 TD exit 的入口点
```c
void init_td_vmcs(tdcs_t * tdcs_ptr, tdvps_t* tdvps_ptr, vmcs_host_values_t* host_fields_ptr)
{
...
    void (*td_entry_func_ptr)(void) = tdx_tdexit_entry_point;
...
    /**
     *  Initialize/Update fields which depend on TDCS/TDR values
     */
    // Host-State
    ia32_vmwrite(VMX_HOST_RIP_ENCODE, (uint64_t)td_entry_func_ptr);
...
}
```
* TD exit 的时候，TDX module 接盘
```c
tdx_tdexit_entry_point
-> callq tdx_td_dispatcher
      switch (vm_exit_reason.basic_reason)
      case VMEXIT_REASON_MSR_WRITE:
           td_wrmsr_exit()
              switch (msr_addr)
              default:
              -> rd_wr_msr_generic_case(VMEXIT_REASON_MSR_WRITE, msr_addr, true, tdvps_p, tdcs_p)
                    if (!is_msr_covered_by_bitmap(msr_addr))
                    -> tdx_inject_ve(vm_exit_reason, 0, tdvps_p, 0, 0)
```

### TD Guest 处理弹射回来的 `#VE`
* TD guest 处理 `#VE` 的入口为 `exc_virtualization_exception`
```c
#ifdef CONFIG_INTEL_TDX_GUEST
DECLARE_IDTENTRY(X86_TRAP_VE,  exc_virtualization_exception)
#endif
```
* 接着处理 WRMSR 弹射回来的 `#VE`，转成 tdvmcall 让 VMM 来处理
```c
exc_virtualization_exception()
-> tdx_get_ve_info(&ve)
-> tdx_handle_virt_exception()
   -> insn_len = virt_exception_kernel()
         switch (ve->exit_reason)
         case EXIT_REASON_MSR_WRITE:
              return write_msr(regs, ve)
                 .r11 = hcall_func(EXIT_REASON_MSR_WRITE) //退出原因是写 MSR
              -> __tdx_hypercall()
                 -> tdcall
   -> regs->ip += insn_len;
-> if failed -> ve_raise_fault()
```

### TD Host 能处理的 WRMSR
* TDX host 能处理的 VM exit 可以在 `tdx_handle_exit()` 里看到，可见并不包含 `WRMSR`
* TD guest 的 `WRMSR` 都是通过 `#VE` 弹射后用 tdvmcall 发出来的
```c
vcpu_enter_guest()
-> static_call(kvm_x86_handle_exit)(vcpu, exit_fastpath)
=> vt_handle_exit()
   -> tdx_handle_exit()
         switch (exit_reason.basic)
         case EXIT_REASON_TDCALL:
            return handle_tdvmcall(vcpu);
               switch (tdvmcall_leaf(vcpu))
               case EXIT_REASON_MSR_WRITE:
                    return tdx_emulate_wrmsr(vcpu);
                    -> kvm_set_msr()
                       -> kvm_set_msr_ignored_check()
                          -> __kvm_set_msr()
                             -> static_call(kvm_x86_set_msr)(vcpu, &msr)
                             => vt_set_msr()
                                -> kvm_set_msr_common()
```
* 如果是 VMM 模拟这次 `WRMSR`：
```cpp
static int tdx_emulate_wrmsr(struct kvm_vcpu *vcpu)
{
    u32 index = tdvmcall_a0_read(vcpu);
    u64 data = tdvmcall_a1_read(vcpu);
    //如果是允许的 MSR，则尝试设置。尝试失败或不允许写的 MSR，设置 TD VMCALL 返回码 INVALID_OPERAND
    if (!kvm_msr_allowed(vcpu, index, KVM_MSR_FILTER_WRITE) ||
        kvm_set_msr(vcpu, index, data)) {
        trace_kvm_msr_write_ex(index, data);
        tdvmcall_set_return_code(vcpu, TDG_VP_VMCALL_INVALID_OPERAND);
        return 1;
    }
    //尝试成功，设置 TD VMCALL 返回码 SUCCESS
    trace_kvm_msr_write(index, data);
    tdvmcall_set_return_code(vcpu, TDG_VP_VMCALL_SUCCESS);
    return 1;
}
```

## 通过 `#VE` 处理 `CPUID`
* 在 TDX guest 中，大多数 `CPUID` 叶/子叶组合由 TDX module 虚拟化，同时一些触发 `#VE`
* 通过 hypercall 传递 `EXIT_REASON_CPUID` 来实现 `#VE` 处理，这反过来让 TDX module 通过调用 host VMM 来处理它
* 请注意，处理 hypercall 的 VMM 不受信任。它可以返回可能将 guest 内核引导到错误方向的数据。因此只允许 VMM 控制为了与 hypervisor 通信而保留的范围
* 为 hypervisor 范围之外的任何 `CPUID` 返回全零。它匹配不受支持的叶子的 CPU 行为
* `handle_cpuid()` 是 TDX module 处理不了，`#VE` 弹射回来准备发给 VMM 处理的 `CPUID`：
```c
static int handle_cpuid(struct pt_regs *regs, struct ve_info *ve)
{
    struct tdx_hypercall_args args = {
        .r10 = TDX_HYPERCALL_STANDARD,
        .r11 = hcall_func(EXIT_REASON_CPUID),
        .r12 = regs->ax,
        .r13 = regs->cx,
    };

    /*
     * Only allow VMM to control range reserved for hypervisor
     * communication.
     *
     * Return all-zeros for any CPUID outside the range. It matches CPU
     * behaviour for non-supported leaf.
     */
    if (regs->ax < 0x40000000 || regs->ax > 0x4FFFFFFF) {
        regs->ax = regs->bx = regs->cx = regs->dx = 0;
        return ve_instr_len(ve);
    }

    /*
     * Emulate the CPUID instruction via a hypercall. More info about
     * ABI can be found in TDX Guest-Host-Communication Interface
     * (GHCI), section titled "VP.VMCALL<Instruction.CPUID>".
     */
    if (__tdx_hypercall(&args, TDX_HCALL_HAS_OUTPUT))
        return -EIO;

    /*
     * As per TDX GHCI CPUID ABI, r12-r15 registers contain contents of
     * EAX, EBX, ECX, EDX registers after the CPUID instruction execution.
     * So copy the register contents back to pt_regs.
     */
    regs->ax = args.r12;
    regs->bx = args.r13;
    regs->cx = args.r14;
    regs->dx = args.r15;

    return ve_instr_len(ve);
}
```
* `0x40000000 ~ 0x4FFFFFFF` 范围内的 CPUID 是无效的，`#VE` 弹射回来后返回 `0`，无需再通过 TDVMCALL 向 VMM 请求模拟
  * 见 SDM vol. 2A 3-233, CPUID - CPU identification

## 处理 in-kernel MMIO
* 在 legacy VM 中，MMIO 是通过为 guest 提供一个映射来实现的，该映射将在访问时导致 VM-exit（EPT misconfiguration），然后 VMM 模拟导致 VM-exit 的指令。然而这对于 TDX VM 是不可行的
* 要模拟一条指令，模拟器需要两件事：
  1. R/W 访问寄存器文件以读取/修改指令参数并查看引起缺页的指令 `%RIP`
  2. 读取指令所在的内存以查看要模拟的内容。在这种情况下，它是 guest 内核文本段
* 在 TDX 环境下，VMM 无法完成以上两件事情：
  1. 寄存器文件永远不会暴露给 VMM。当 TD 退出到 TDX module 时，它会将寄存器保存到为该 TD 分配的状态保存区域。然后，TDX module 在将执行控制返回给 VMM 之前清理这些寄存器，以帮助防止泄漏 TD 状态信息
  2. TDX 不允许 guest 从共享内存执行代码。所有可执行的指令都在 TD 私有内存中。由于是 TD 私有的，而 VMM 无法访问 TD 私有内存，也无法读取指令来对其进行解码和仿真
* 在 TDX 中，MMIO 区域由 VMM 配置为在 guest 中触发 `#VE` 异常
  * 这种方法对性能不利。但是，它（实际上）对内核映像的大小没有影响，并且适用于各种驱动程序。这允许 TDX 部署使用任意设备和设备驱动程序，包括 virtio。TDX guest 要求能够在其部署中使用 ramdom 设备
  * 换句话说，即使所有工作都以半虚拟化所有 x86 MMIO 用户和 virtio 完成，仍然需要这种方法。基本上没有办法摆脱这段代码
  * 这种方法适用于当前和未来的所有内核 MMIO 用户，并且代码量和内核映像膨胀最少
* MMIO 地址可以与的任何 CPU 指令可访问内存一起使用。仅通过 io.h 里的 helper 完成 MMIO 地址的访问，例如 `readl()` 或 `writeq()`
* 任何访问内存的 CPU 指令也可用于访问 MMIO。但是，按照惯例，MMIO 访问通常是通过 io.h 里的 helper（例如 `readl()` 或 `writeq()`）执行的
* io.h 里的 helper 在访问 MMIO 时有意使用一组有限的指令。这种已知的、有限的指令集使 MMIO 指令解码和仿真在当今的 KVM host 和 SEV guest 中变得可行
* 在没有 io.h helper 的情况下执行的 MMIO 访问受编译器的支配。编译器可以并将生成范围更广的指令集，这些指令实际上无法被解码和模拟。如果 TDX guest 遇到这些解码失败之一，他们将会 oops
* 这意味着 TDX guest **必须** 使用 io.h helper 来访问 MMIO
* 这个要求并不新鲜。KVM host 和 AMD SEV guest 都对 MMIO 访问有相同的限制

### 潜在的替代方案

#### 类虚拟化所有的 MMIO
* 让 MMIO 引发 `#VE` 异常的另一种方法是首先避免 `#VE`。与  port I/O 情况类似，理论上可以类虚拟化 MMIO 访问
* 与此处提供的基于异常的方法一样，完全半虚拟化的方法将仅限于利用通用基础设施（如 io.h 宏）的 MMIO 用户
* 然而，任何半虚拟方法都会修补大约 12 万个调用之处。任何半虚拟方法都需要用（至少）函数调用来替换原始的内存访问指令。保守估计每个调用点（`CALL` 指令）有 5 个字节的开销，这会导致代码膨胀 600k
* 许多驱动程序将永远不会在 TDX 环境中使用，并且膨胀是没有道理的

#### 给 TDX 驱动打 patch
* 与其触及整个内核，还可能只关注在 TDX guest 中使用 MMIO 的驱动程序 *并且* 对性能至关重要，以证明这种努力是合理的。现在，这仅限于 virtio
* 所有 virtio MMIO 似乎都是通过一个函数完成的，这使得 virtio 非常容易打补丁
* 这种方法将在未来被采用，去除大量的 MMIO `#VE`。基于 `#VE` 的 MMIO 将继续服务于非 virtio 用例

### MMIO `#VE` 的具体实现
* 参考 TDX Guest-Host-Communication Interface (GHCI)
  * Section 3.7, TDG.VP.VMCALL<#VE.RequestMMIO>
  * Table 2-4: TDG.VP.VMCALL-Instruction-execution sub-functions
* arch/x86/coco/tdx/tdx.c
```c
static bool mmio_read(int size, unsigned long addr, unsigned long *val)
{
    struct tdx_hypercall_args args = {
        .r10 = TDX_HYPERCALL_STANDARD,
        .r11 = hcall_func(EXIT_REASON_EPT_VIOLATION), //#VE.RequestMMIO 是 48
        .r12 = size,
        .r13 = EPT_READ, //方向是“读”
        .r14 = addr,     //读取的 MMIO 地址
        .r15 = *val,     //方向是读时 spec 未定义
    };

    if (__tdx_hypercall(&args, TDX_HCALL_HAS_OUTPUT))
        return false;
    *val = args.r11;     //方向是读时，读到的值
    return true;
}

static bool mmio_write(int size, unsigned long addr, unsigned long val)
{   //不在乎返回值的“写”
    return !_tdx_hypercall(hcall_func(EXIT_REASON_EPT_VIOLATION), size,
                   EPT_WRITE, addr, val);
}
```
* `#VE` 弹射回来的 MMIO 操作通过 `handle_mmio()` 处理，发送 TDVMCALL 并返回处理结果
```c
static int handle_mmio(struct pt_regs *regs, struct ve_info *ve)
{
    unsigned long *reg, val, vaddr;
    char buffer[MAX_INSN_SIZE];
    enum insn_mmio_type mmio;
    struct insn insn = {};
    int size, extend_size;
    u8 extend_val = 0;
    //当前仅支持内核态 MMIO
    /* Only in-kernel MMIO is supported */
    if (WARN_ON_ONCE(user_mode(regs)))
        return -EFAULT;
    //拷贝触发 MMIO 的指令开始的 15 个字节到 buffer[]
    if (copy_from_kernel_nofault(buffer, (void *)regs->ip, MAX_INSN_SIZE))
        return -EFAULT;
    //解码 buffer[] 中的指令，输出到 insn 结构
    if (insn_decode(&insn, buffer, MAX_INSN_SIZE, INSN_MODE_64))
        return -EINVAL;
    //解码 insn 结构中的 MMIO 指令
    mmio = insn_decode_mmio(&insn, &size);
    if (WARN_ON_ONCE(mmio == INSN_MMIO_DECODE_FAILED))
        return -EINVAL;
    //如果不是 写立即数 也不是 movs 指令
    if (mmio != INSN_MMIO_WRITE_IMM && mmio != INSN_MMIO_MOVS) {
        reg = insn_get_modrm_reg_ptr(&insn, regs); //基于 ModRM 字节得到操作的寄存器在 regs 中对应域的指针
        if (!reg)
            return -EINVAL;
    }
    //见下面的展开解释
    /*
     * Reject EPT violation #VEs that split pages.
     *
     * MMIO accesses are supposed to be naturally aligned and therefore
     * never cross page boundaries. Seeing split page accesses indicates
     * a bug or a load_unaligned_zeropad() that stepped into an MMIO page.
     *
     * load_unaligned_zeropad() will recover using exception fixups.
     */
    vaddr = (unsigned long)insn_get_addr_ref(&insn, regs); //获取 MMIO 的线性地址
    if (vaddr / PAGE_SIZE != (vaddr + size - 1) / PAGE_SIZE) //地址未对齐到页边界
        return -EFAULT; //拒绝此次 MMIO 模拟

    /* Handle writes first */
    switch (mmio) {
    case INSN_MMIO_WRITE:
        memcpy(&val, reg, size);
        if (!mmio_write(size, ve->gpa, val)) //mov reg mem
            return -EIO;
        return insn.length;
    case INSN_MMIO_WRITE_IMM:
        val = insn.immediate.value;
        if (!mmio_write(size, ve->gpa, val)) //mov imm mem
            return -EIO;
        return insn.length;
    case INSN_MMIO_READ:
    case INSN_MMIO_READ_ZERO_EXTEND:
    case INSN_MMIO_READ_SIGN_EXTEND:
        /* Reads are handled below */
        break;
    case INSN_MMIO_MOVS:
    case INSN_MMIO_DECODE_FAILED:
        /*
         * MMIO was accessed with an instruction that could not be
         * decoded or handled properly. It was likely not using io.h
         * helpers or accessed MMIO accidentally.
         */
        return -EINVAL;
    default:
        WARN_ONCE(1, "Unknown insn_decode_mmio() decode value?");
        return -EINVAL;
    }

    /* Handle reads */
    if (!mmio_read(size, ve->gpa, &val)) //mov mem val
        return -EIO;

    switch (mmio) {
    case INSN_MMIO_READ:
        /* Zero-extend for 32-bit operation */ //对 32 位操作的 0 扩展读
        extend_size = size == 4 ? sizeof(*reg) : 0;
        break;
    case INSN_MMIO_READ_ZERO_EXTEND:
        /* Zero extend based on operand size */
        extend_size = insn.opnd_bytes; //基于操作数大小的 0 扩展
        break;
    case INSN_MMIO_READ_SIGN_EXTEND:
        /* Sign extend based on operand size */ //基于符号的扩展
        extend_size = insn.opnd_bytes;
        if (size == 1 && val & BIT(7))
            extend_val = 0xFF;
        else if (size > 1 && val & BIT(15))
            extend_val = 0xFF;
        break;
    default:
        /* All other cases has to be covered with the first switch() */
        WARN_ON_ONCE(1);
        return -EINVAL;
    }

    if (extend_size)
        memset(reg, extend_val, extend_size); //先填充扩展内容
    memcpy(reg, &val, size); //（再）填写读到的值
    return insn.length;
}
```

### 处理`load_unaligned_zeropad()`导致的加载跨越到一个共享页面
* `commit 1e7769653b06 ("x86/tdx: Handle load_unaligned_zeropad() page-cross to a shared page")`
* `load_unaligned_zeropad()` 会导致不期望的跨页面边界的加载
  * 这种不期望的加载通常是无害的。但是，它们可能会变成完全不相关甚至未映射的内存
  * `load_unaligned_zeropad()` 依赖于异常修复（`#PF`、`#GP` 和现在的 `#VE`）从这些不期望的加载中恢复
* 在 TDX guest 中，第二个页面可以是共享页面，VMM 可以将其配置为触发 `#VE`
  * 内核假定共享页面上的 `#VE` 是 MMIO 访问并尝试解码指令来处理它
  * 但在 `load_unaligned_zeropad()` 的情况下，它可能会导致混淆，因为它不是 MMIO 访问
* 在 `handle_mmio()` 通过检测分离页面 MMIO 访问并使它们失败来修复它
* `load_unaligned_zeropad()` 将使用异常修复进行恢复
* 该问题是通过分析发现并人工复现的，在测试期间未被触发

## TD Guest 的 Port I/O 处理
### TD VM 内核早期解压缩时的 TDX 的探测和 Port I/O
* 早期的解压代码为其控制台输出做 port I/O
  * 但是，处理解压时 port I/O 需要一种不同于正常运行时的方法，因为支持基于 `#VE` 的 port I/O 模拟所需的 IDT 尚未设置
  * 在解压缩步骤期间半虚拟化 I/O 调用是可以接受的，因为解压缩代码没有很多 IO 指令的调用处
* 为了在解压缩代码中支持 port I/O，必须在解压缩代码可能执行 port I/O 之前检测到 TDX。检测内核是否在 TDX guest 中运行
* 添加 `early_is_tdx_guest()` 接口，用于在解压代码中查询缓存的 TDX guest 的状态，且必须在 `console_init()` 前被调用
* 使用 `CPUID` 检测 TDX。使 `cpuid_count()` 可在 boot/cpuflags.c 之外访问
* arch/x86/boot/compressed/tdx.c
```c
void early_tdx_detect(void)
{
    u32 eax, sig[3];
    //获取 TDX 的 CPUID leaf 0x21
    cpuid_count(TDX_CPUID_LEAF_ID, 0, &eax, &sig[0], &sig[2],  &sig[1]);
    //如果不是 "IntelTDX    "，则我们不是 TD VM，返回
    if (memcmp(TDX_IDENT, sig, sizeof(sig)))
        return;
    //如果时 TD VM，使用 TDVMCALL 替换 I/O (in/out) 指令
    /* Use hypercalls instead of I/O instructions */
    pio_ops.f_inb  = tdx_inb;
    pio_ops.f_outb = tdx_outb;
    pio_ops.f_outw = tdx_outw;
}
```
* 添加一种方法，以使用新的 `pio_ops` 结构，以 hoot 的方式在 boot stub 中的备用 port I/O helper
  * arch/x86/boot/io.h
```c
#include <asm/shared/io.h>

#undef inb
#undef inw
#undef inl
#undef outb
#undef outw
#undef outl

struct port_io_ops {
    u8  (*f_inb)(u16 port);
    void    (*f_outb)(u8 v, u16 port);
    void    (*f_outw)(u16 v, u16 port);
};

extern struct port_io_ops pio_ops;

/*
 * Use the normal I/O instructions by default.
 * TDX guests override these to use hypercalls.
 */
static inline void init_default_io_ops(void)
{
    pio_ops.f_inb  = __inb;
    pio_ops.f_outb = __outb;
    pio_ops.f_outw = __outw;
}

/*
 * Redirect port I/O operations via pio_ops callbacks.
 * TDX guests override these callbacks with TDX-specific helpers.
 */
#define inb  pio_ops.f_inb
#define outb pio_ops.f_outb
#define outw pio_ops.f_outw
```
* 会被编译并链接到 arch/x86/boot/compressed/vmlinux，随后 `objcopy -O binary -S` 到 arch/x86/boot/vmlinux.bin
  * arch/x86/boot/compressed/misc.c
```c
asmlinkage __visible void *extract_kernel(void *rmode, memptr heap,
                  unsigned char *input_data,
                  unsigned long input_len,
                  unsigned char *output,
                  unsigned long output_len)
{
...
    init_default_io_ops(); //一开始 pio_ops 用原始的 in/out 指令
    //如果探测到 TD VM，替换 pio_ops 的 hook 为采用 TDVMCALL 模拟的 port I/O
    /*
     * Detect TDX guest environment.
     *
     * It has to be done before console_init() in order to use
     * paravirtualized port I/O operations if needed.
     */
    early_tdx_detect();

    console_init();
...
}
```
* 在更早的 arch/x86/boot/main.c:main()（setup.bin）中也需如此
```c
_start
   calll main
-> main()
   -> init_default_io_ops()
   -> copy_boot_params()
   -> console_init()
```

### TD VM 内核正常运行时的 Port I/O
* TDX hypervisor 无法直接模拟 port I/O 指令。这包括通常在 hypervisor 中模拟的 port I/O。TDX 中的所有 port I/O 指令都会触发 guest 中的 `#VE` 异常，并且通常会在那里进行模拟（转换）
* TDX 不支持字符串 I/O 操作。通过声明 `CC_ATTR_GUEST_UNROLL_STRING_IO` 机密计算属性来 unroll 它们
* 对用户空间的影响
* `ioperm()` 工具允许用户空间访问 I/O 指令，如 `inb/outb`。除此之外，这允许编写用户空间设备驱动程序
* 当前对 `ioperm()` 没有特殊处理。用户将能够成功请求 I/O 权限，但会在他们的第一个 I/O 指令上引发 `#VE` 并导致 `SIGSEGV`。如果这是不受欢迎的，用户可以使用`lockdown=integrity` 内核命令行选项启用内核锁定功能。它使 `ioperm()` 失败
  * arch/x86/coco/tdx/tdx.c
```c
/* Port I/O direction */
#define PORT_READ   0
#define PORT_WRITE  1

/* See Exit Qualification for I/O Instructions in VMX documentation */
#define VE_IS_IO_IN(e)      ((e) & BIT(3))
#define VE_GET_IO_SIZE(e)   (((e) & GENMASK(2, 0)) + 1)
#define VE_GET_PORT_NUM(e)  ((e) >> 16)
#define VE_IS_IO_STRING(e)  ((e) & BIT(4))
...
static bool handle_in(struct pt_regs *regs, int size, int port)
{
    struct tdx_hypercall_args args = {
        .r10 = TDX_HYPERCALL_STANDARD,
        .r11 = hcall_func(EXIT_REASON_IO_INSTRUCTION),
        .r12 = size,
        .r13 = PORT_READ,
        .r14 = port,
    };
    u64 mask = GENMASK(BITS_PER_BYTE * size, 0);
    bool success;

    /*
     * Emulate the I/O read via hypercall. More info about ABI can be found
     * in TDX Guest-Host-Communication Interface (GHCI) section titled
     * "TDG.VP.VMCALL<Instruction.IO>".
     */
    success = !__tdx_hypercall(&args, TDX_HCALL_HAS_OUTPUT);

    /* Update part of the register affected by the emulated instruction */
    regs->ax &= ~mask;
    if (success)
        regs->ax |= args.r11 & mask;

    return success;
}

static bool handle_out(struct pt_regs *regs, int size, int port)
{
    u64 mask = GENMASK(BITS_PER_BYTE * size, 0);

    /*
     * Emulate the I/O write via hypercall. More info about ABI can be found
     * in TDX Guest-Host-Communication Interface (GHCI) section titled
     * "TDG.VP.VMCALL<Instruction.IO>".
     */
    return !_tdx_hypercall(hcall_func(EXIT_REASON_IO_INSTRUCTION), size,
                   PORT_WRITE, port, regs->ax & mask);
}

/*
 * Emulate I/O using hypercall.
 *
 * Assumes the IO instruction was using ax, which is enforced
 * by the standard io.h macros.
 *
 * Return True on success or False on failure.
 */
static int handle_io(struct pt_regs *regs, struct ve_info *ve)
{
    u32 exit_qual = ve->exit_qual;
    int size, port;
    bool in, ret;

    if (VE_IS_IO_STRING(exit_qual))
        return -EIO;

    in   = VE_IS_IO_IN(exit_qual);
    size = VE_GET_IO_SIZE(exit_qual);
    port = VE_GET_PORT_NUM(exit_qual);


    if (in)
        ret = handle_in(regs, size, port);
    else
        ret = handle_out(regs, size, port);
    if (!ret)
        return -EIO;

    return ve_instr_len(ve);
}
```

### 内核解压后早期的 Port I/O 处理
* 在设置 IDT 处理程序之前，Port I/O 无法使用普通内核 `#VE` 处理程序进行模拟。要在此引导窗口期间支持基于 `#VE` 的仿真，需要在早期异常处理程序中添加最小的早期`#VE` 处理程序支持。这类似于 AMD SEV 所做的。这主要是为了支持 `earlyprintk` 的串行驱动程序，以及潜在的 VGA 驱动程序
* 早期的处理程序仅支持与 I/O 相关的 `#VE` 异常。未处理或失败的异常将通过 `early_fixup_exceptions()` 处理（就像正常的异常失败一样）
* 在运行时，与 I/O 相关的 `#VE` 异常（以及其他类型）由 `virt_exception_kernel()` 处理
  * arch/x86/coco/tdx/tdx.c
```c
/*
 * Early #VE exception handler. Only handles a subset of port I/O.
 * Intended only for earlyprintk. If failed, return false.
 */
__init bool tdx_early_handle_ve(struct pt_regs *regs)
{
    struct ve_info ve;
    int insn_len;

    tdx_get_ve_info(&ve);

    if (ve.exit_reason != EXIT_REASON_IO_INSTRUCTION)
        return false;
    //早期的处理程序仅支持与 I/O 相关的 #VE 异常
    insn_len = handle_io(regs, &ve);
    if (insn_len < 0)
        return false;

    regs->ip += insn_len;
    return true;
}
```
* 早期 IDT 通用处理例程 `early_idt_handler_common` 会调用 `call do_early_exception`
* 该文件 arch/x86/kernel/head64.c 会编进 vmlinux
```c
void __init do_early_exception(struct pt_regs *regs, int trapnr)
{
    if (trapnr == X86_TRAP_PF &&
        early_make_pgtable(native_read_cr2()))
        return;

    if (IS_ENABLED(CONFIG_AMD_MEM_ENCRYPT) &&
        trapnr == X86_TRAP_VC && handle_vc_boot_ghcb(regs))
        return;

    if (trapnr == X86_TRAP_VE && tdx_early_handle_ve(regs))
        return;

    early_fixup_exception(regs, trapnr);
}
```
## 造成 `#VE` 指令长度的计算
* `#VE` 处理成功后，`tdx_handle_virt_exception()` 必须将 `%RIP` 移动到下一条指令，因此，处理程序需要知道指令的长度
* 如果 `#VE` 由于指令执行而发生，`GET_VEINFO` TDX mdoule call 会在 `%R10` 中提供有关指令的信息，包括其长度
  * `hlt`
  * `rdmsr`
  * `wrmsr`
  * `cpuid`
  * `in/out`
* 对于由于 EPT violation 而导致的 `#VE`，`%R10` 中的信息未填充，内核必须手动解码指令以找出其长度
  * arch/x86/coco/tdx/tdx.c
```c
/*
 * The TDX module spec states that #VE may be injected for a limited set of
 * reasons:
 *
 *  - Emulation of the architectural #VE injection on EPT violation;
 *
 *  - As a result of guest TD execution of a disallowed instruction,
 *    a disallowed MSR access, or CPUID virtualization;
 *
 *  - A notification to the guest TD about anomalous behavior;
 *
 * The last one is opt-in and is not used by the kernel.
 *
 * The Intel Software Developer's Manual describes cases when instruction
 * length field can be used in section "Information for VM Exits Due to
 * Instruction Execution".
 *
 * For TDX, it ultimately means GET_VEINFO provides reliable instruction length
 * information if #VE occurred due to instruction execution, but not for EPT
 * violations.
 */
static int ve_instr_len(struct ve_info *ve)
{
    switch (ve->exit_reason) {
    case EXIT_REASON_HLT:
    case EXIT_REASON_MSR_READ:
    case EXIT_REASON_MSR_WRITE:
    case EXIT_REASON_CPUID:
    case EXIT_REASON_IO_INSTRUCTION:
        /* It is safe to use ve->instr_len for #VE due instructions */
        return ve->instr_len;
    case EXIT_REASON_EPT_VIOLATION:
        /*
         * For EPT violations, ve->insn_len is not defined. For those,
         * the kernel must decode instructions manually and should not
         * be using this function.
         */
        WARN_ONCE(1, "ve->instr_len is not defined for EPT violations");
        return 0;
    default:
        WARN_ONCE(1, "Unexpected #VE-type: %lld\n", ve->exit_reason);
        return ve->instr_len;
    }
}
```

## 多处理器启动

* 从历史上看，x86 平台使用 `INIT` 后跟 *start up IPI (SIPI)* 消息来启动辅助处理器 (AP)。在常规 VM 中，VMM 仿真支持此引导顺序。但是这样的唤醒模式对于像 TDX 这样的安全 VM 来说是致命的，其中 VMM 是一个不受信任的实体
  * 之所以说 INIT-SIPI-SIPI 的启动序列是不安全的，仅针对 TD 虚拟化场景。IPI 的虚拟化是通过 VMM（主要是 Qemu）以函数调用的方式模拟的（写 ICR 导致 VM Exit），SIPI 会包含 AP 的启动地址，这就意味着这些信息因此暴露在 TD 之外
* 为了解决这个问题，ACPI v6.4 中添加了一个新的唤醒模式，其中固件（如 TDX 虚拟 BIOS）将帮助启动 AP
  * 有关此唤醒模式的更多详细信息，请参阅 ACPI 规范 v6.4，标题为“多处理器唤醒结构”的部分
  * AP 未被唤醒前，执行 TDVF 的一段循环等待/读取 mailbox 的例程，而 TDVF 是作为 restricted_memfd 加入 TD 的，mailbox 的分配也是在 TD 内进行的，整个过程都在信任域中完成

### 5.2.12.19 Multiprocessor Wakeup Structure
* 平台固件发布了一个 **多处理器唤醒结构（Multiprocessor Wakeup Structure）**，让 bootstrap processor 用 mailbox 唤醒 application processors
  * Mailbox 是固件保留的内存，以便每个处理器都可以让操作系统向它们发送消息
  * 在系统引导期间，固件将 AP 置于检查 mailbox 的状态
  * 当固件将控制权转移到 OS 加载程序时，不允许固件修改 mailbox 位置

#### Mailbox 空间
* 共享 mailbox 是由固件在 ACPINvs 内存中分配的一个 `4K` 对齐的 `4K` 大小的内存块
  * Mailbox 分为两个 `2KB` 部分：
    * 操作系统部分
    * 固件部分
* **OS 部分** 只能 *由 OS 写入* 并 *由固件读取*，命令字段除外
  * AP 需要将命令清除到 `Noop(0)` 作为收到命令的确认（`Noop(0)` 是 No Operation 的缩写，命令编码为 `0`）
  * 固件必须缓存 mailbox 中的内容，以便稍后使用，在清除例如 `WakeupVector` 等命令之前
  * 只有将命令更改为 `Noop(0)` 后，OS 才能发送下一条命令
* **固件部分** 必须被视为 *对 OS 只读*，并且只能 *由固件写入*
* OS 和固件之间的所有数据通信都必须采用小端格式

#### 用 Mailbox 进行通信
* OS 部分包含：
  1. 命令
  2. 标志
  3. APIC ID
  4. 唤醒地址
* OS 从 MADT 表中检测到处理器编号后，OS 可以准备唤醒例程，在 mailbox 中填充 *唤醒地址* 字段，在 *APID ID* 字段中指示需要唤醒哪个处理器，并发送 *唤醒命令*
* 一旦 AP 检测到 *唤醒命令* 和 *它自己的 APIC ID*，AP 将跳转到 OS 提供的 *唤醒地址*。如果 APIC ID 与其自身不匹配，AP 将忽略该命令
* 对于每个 AP，mailbox 只能用于唤醒命令一次。AP 根据命令采取动作后，该 mailbox 将不再被该 AP 检查。其他处理器可以继续使用 mailbox 来处理下一个命令

##### Table 5.43: Multiprocessor Wakeup Structure

域             | 字节长度 | 字节偏移 | 描述
---------------|---------|---------|---------------------------------------
Type           | 1       | 0       | `0x10` Multiprocessor Wakeup structure
Length         | 1       | 1       | 16
MailBoxVersion | 2       | 2       | mailbox 的版本。ACPI spec v6.5 版本为 0
保留           | 4       | 4       | 必须为 0
MailBoxAddress | 8       | 8       | mailbox 的物理地址，必须在 ACPINvs，并且 4KB 对齐

##### Table 5.44: Multiprocessor Wakeup Mailbox Structure

域             | 字节长度 | 字节偏移 | 描述
---------------|---------|---------|---------------------------------------
Command        | 2       | 0       | `0: Noop` - 无操作 </br> `1: Wakeup` – 跳转到 wakeup vector </br> `2-0xFFFF`: 保留
保留           | 2       | 2       | 必须为 0
ApicId         | 4       | 4       | 处理器的 local APIC ID。AP 应检查 *ApicId* 字段是否与自己的 APIC ID 匹配。在 APIC ID 不匹配的情况下，AP 应忽略该命令
WakeupVector   | 8       | 8       | AP 的唤醒地址。对于 Intel 处理器，执行环境是：</br> * 必须禁用中断</br> * `RFLAGES.IF` 设置为 `0` </br> * 启用长模式</br> * 启用分页模式，唤醒向量的物理内存是恒等映射的（虚拟地址等于物理地址）</br> * 唤醒向量必须包含在一个物理页面中</br> * 段选择符设置为平坦映射，否则不使用
ReservedForOs | 2032     | 16      | 保留给 OS 使用
ReservedForFirmware | 2048 | 2048  | 保留给固件使用

### 用于通过固件转交启动 AP 的 Trampoline
* 由于现有的 trampoline 代码要求处理器以 16 位寻址以实模式启动，因此它不适用于此唤醒模型（因为它以 64 位模式启动 AP）
  * 要处理它，需要扩展 trampoline 代码以支持 64 位模式固件切换
  * 此外，还扩展 IDT 和 GDT 指针以支持 64 位模式切换
* 没有针对这种新引导方法的 TDX 特定检测。每当新的 ACPI 结构出现时，内核将依赖它作为唯一的引导方法
* MADT 多处理器唤醒结构的 ACPI 表解析器和使用该结构的唤醒方法将由本系列的以下补丁添加
* 给用于 trampoline 的 IDTR 的值在 `.rodata` section 中预留空间
  * 结构见 SDM vol.3, Figure 6-1. Relationship of the IDTR and IDT
  * arch/x86/realmode/rm/trampoline_common.S
```c
    .section ".rodata","a" //追加进 .rodata section
    .balign 16             //对齐到 16 字节

/*
 * When a bootloader hands off to the kernel in 32-bit mode an
 * IDT with a 2-byte limit and 4-byte base is needed. When a boot
 * loader hands off to a kernel 64-bit mode the base address
 * extends to 8-bytes. Reserve enough space for either scenario.
 */
SYM_DATA_START_LOCAL(tr_idt)
    .short  0  //IDTR.limit 占 0 ~ 15 (16 bit)
    .quad   0  //IDTR.base_address 占 16 ~ 47 (32 bit)，这里四字肯定是够 8 字节了
SYM_DATA_END(tr_idt)
```
* arch/x86/realmode/rm/stack.S
```c
    .data //在 .data section 预留
SYM_DATA(HEAP,      .long rm_heap) //变量 HEAP 内容为 realmode 堆的地址
SYM_DATA(heap_end,  .long rm_stack)//变量 heap_end 内容为 realmode 栈的地址

    .bss //在 .bss section 预留
    .balign 16
SYM_DATA(rm_heap,   .space 2048) //预留 realmode 堆大小为 2048 字节
//预留 realmode 栈大小为 2048 字节，紧邻着 realmode 堆
SYM_DATA_START(rm_stack)
    .space  2048
SYM_DATA_END_LABEL(rm_stack, SYM_L_GLOBAL, rm_stack_end)
```
* arch/x86/realmode/rm/trampoline_64.S
```c
SYM_CODE_START(pa_trampoline_compat)
    /*
     * In compatibility mode.  Prep ESP and DX for startup_32, then disable
     * paging and complete the switch to legacy 32-bit mode.
     */
    movl    $rm_stack_end, %esp //把 realmode 栈顶地址传输到 %esp
    movw    $__KERNEL_DS, %dx   //内核数据段选择符到 %dx

    movl    $(CR0_STATE & ~X86_CR0_PG), %eax //准备关闭分页
    movl    %eax, %cr0 //放到 %cr0 关闭分页
    ljmpl   $__KERNEL32_CS, $pa_startup_32 //长跳转至 32 位启动地址 startup_32 开始执行
SYM_CODE_END(pa_trampoline_compat)
```
* 采用新的唤醒模式，64 位 BIOS（TDX 虚拟 BIOS，TDVF）带着恒等映射页表直接转移到 AP，并从这里开始
* 加载内核的 GDT 以降低到 32 位模式（处理 4 级与 5 级分页），并（重新）加载段寄存器
* 加载零 IDT，这样任何异常都会触发关闭而不是跳回 BIOS
```c
SYM_CODE_START(trampoline_start64)
    /*
     * APs start here on a direct transfer from 64-bit BIOS with identity
     * mapped page tables.  Load the kernel's GDT in order to gear down to
     * 32-bit mode (to handle 4-level vs. 5-level paging), and to (re)load
     * segment registers.  Load the zero IDT so any fault triggers a
     * shutdown instead of jumping back into BIOS.
     */
    lidt    tr_idt(%rip)   //把一个全零的值 load 到 IDTR
    lgdt    tr_gdt64(%rip) //load 符号 tr_gdt64 的内容到 GDTR

    ljmpl   *tr_compat(%rip) //长跳转至符号 tr_compat 的内容处，即跳转到 pa_trampoline_compat
SYM_CODE_END(trampoline_start64)

    .section ".rodata","a"
    # Duplicate the global descriptor table
    # so the kernel can live anywhere
    .balign 16
SYM_DATA_START(tr_gdt) //trampoline GDT 内容如下
    .short  tr_gdt_end - tr_gdt - 1 # gdt limit //第一个段描述符
    .long   pa_tr_gdt //段基址为 tr_gdt 的物理地址
    .short  0
    .quad   0x00cf9b000000ffff  # __KERNEL32_CS //第二个段描述符为 32 位内核代码段
    .quad   0x00af9b000000ffff  # __KERNEL_CS   //第三个段描述符为内核代码段
    .quad   0x00cf93000000ffff  # __KERNEL_DS   //第四个段描述符为内核数据段
SYM_DATA_END_LABEL(tr_gdt, SYM_L_LOCAL, tr_gdt_end)

SYM_DATA_START(tr_gdt64) //符号 tr_gdt64 的内容如下
    .short  tr_gdt_end - tr_gdt - 1 # gdt limit
    .long   pa_tr_gdt //GDTR.base_address 为符号 tr_gdt 的物理地址
    .long   0
SYM_DATA_END(tr_gdt64)

SYM_DATA_START(tr_compat) //符号 tr_compat 的内容为 pa_trampoline_compat 的地址
    .long   pa_trampoline_compat
    .short  __KERNEL32_CS
SYM_DATA_END(tr_compat)
```
* arch/x86/realmode/rm/pasyms.h 的生成，里面定义了类似 `pa_trampoline_compat`、`pa_tr_gdt`、`pa_trampoline_start64` 的值
```sh
cmd_arch/x86/realmode/rm/pasyms.h := nm arch/x86/realmode/rm/header.o arch/x86/realmode/rm/trampoline_64.o arch/x86/realmode/rm/stack.o arch/x86/realmode/rm/reboot.o arch/x86/realmode/rm/wakeup_asm.o arch/x86/realmode/rm/wakemain.o arch/x86/realmode/rm/video-mode.o arch/x86/realmode/rm/copy.o arch/x86/realmode/rm/bioscall.o arch/x86/realmode/rm/regs.o arch/x86/realmode/rm/video-vga.o arch/x86/realmode/rm/video-vesa.o arch/x86/realmode/rm/video-bios.o | sed -n -r -e 's/^([0-9a-fA-F]+) [ABCDGRSTVW] (.+)$$/pa_\2 = \2;/p' | sort | uniq > arch/x86/realmode/rm/pasyms.h
```
* 这些值都会放到 arch/x86/realmode/rm/header.S 定义的 `real_mode_header`，并放到 `realmode.elf` 文件的开头
```sh
cmd_arch/x86/realmode/rm/realmode.elf := ld  -m elf_x86_64   -m elf_i386 --emit-relocs -T arch/x86/realmode/rm/realmode.lds arch/x86/realmode/rm/header.o arch/x86/realmode/rm/trampoline_64.o arch/x86/realmode/rm/stack.o arch/x86/realmode/rm/reboot.o arch/x86/realmode/rm/wakeup_asm.o arch/x86/realmode/rm/wakemain.o arch/x86/realmode/rm/video-mode.o arch/x86/realmode/rm/copy.o arch/x86/realmode/rm/bioscall.o arch/x86/realmode/rm/regs.o arch/x86/realmode/rm/video-vga.o arch/x86/realmode/rm/video-vesa.o arch/x86/realmode/rm/video-bios.o -o arch/x86/realmode/rm/realmode.elf
```
* `realmode.elf` 文件的二进制部分又会被做成 `realmode.bin`
```sh
cmd_arch/x86/realmode/rm/realmode.bin := objcopy  -O binary arch/x86/realmode/rm/realmode.elf arch/x86/realmode/rm/realmode.bin
```
* `realmode.bin` 会被 `arch/x86/realmode/rmpiggy.S` 包含作为数据，编译成 arch/x86/realmode/rmpiggy.o，然后打包进 arch/x86/realmode/built-in.a
```sh
cmd_arch/x86/realmode/built-in.a := rm -f arch/x86/realmode/built-in.a; ar cDPrST arch/x86/realmode/built-in.a arch/x86/realmode/init.o arch/x86/realmode/rmpiggy.o
```
* 随后 arch/x86/realmode/built-in.a 会被 `ar` 打包进 arch/x86/built-in.a

### 传递 AP 的起始地址 `trampoline_start64`
* 辅助 CPU 启动当前使用称为“INIT/SIPI 协议”来执行。此协议需要 VMM 的协助来引导 guest。由于 TDX VMM 不受 guest 信任，因此现在应该是一个熟悉的故事，无法向 TDX guest 提供支持
* 为了补救这种情况，新的 “Multiprocessor Wakeup Structure” 已添加到现有的 ACPI 表 (MADT)。这个结构提供了一个“mailbox”的物理地址。然后，对 mailbox 的写入会将辅助 CPU 引导至启动的代码
* 添加 ACPI MADT 唤醒结构解析支持和唤醒支持。使用此支持在存在时唤醒 CPU，而不是 INIT/SIPI
* 虽然理论上可以在 32 位内核上使用此结构，但没有 32 位 TDX guest 内核。但尚未经过测试，实际上无法 *在 32 位上进行* 测试。故仅将其设置为 64 位
* BSP 启动的时候会解析 MADT 表，包括类型为 `ACPI_MADT_TYPE_MULTIPROC_WAKEUP` 的子表
```c
start_kernel()
-> setup_arch()
   -> acpi_boot_init()
      -> acpi_process_madt() //Process the Multiple APIC Description Table (MADT), if present
         ---> acpi_table_parse_madt(ACPI_MADT_TYPE_MULTIPROC_WAKEUP, acpi_parse_mp_wake, 1) //Parse MADT MP Wake entry

(gdb) bt
#0  acpi_parse_mp_wake (header=0xffffffffff2400fe, end=18446744073695133966) at linux/arch/x86/kernel/acpi/boot.c:1152
#1  0xffffffff8195a08c in call_handler (end=18446744073695133966, hdr=0xffffffffff2400fe, proc=0xffffffff83803dc8)
    at linux/drivers/acpi/tables.c:300
#2  acpi_parse_entries_array (max_entries=1, proc_num=1, proc=0xffffffff83803dc8, table_header=<optimized out>, table_size=<optimized out>,
    id=0xffffffff82e93ccb "APIC") at linux/drivers/acpi/tables.c:358
#3  acpi_table_parse_entries_array (id=<optimized out>, table_size=<optimized out>, proc=0xffffffff83803dc8, proc_num=1, max_entries=1)
    at linux/drivers/acpi/tables.c:417
#4  0xffffffff842a58a6 in __acpi_table_parse_entries (max_entries=16, arg=0x0 <fixed_percpu_data>, handler_arg=0x0 <fixed_percpu_data>,
    handler=0x2c <fixed_percpu_data+44>, entry_id=-2098643765, table_size=44, id=0xffffffff82e93ccb "APIC")
    at linux/drivers/acpi/tables.c:436
#5  acpi_table_parse_entries (id=id@entry=0xffffffff82e93ccb "APIC", table_size=table_size@entry=44, entry_id=entry_id@entry=16,
    handler=handler@entry=0xffffffff8426cc16 <acpi_parse_mp_wake>, max_entries=max_entries@entry=1)
    at linux/drivers/acpi/tables.c:455
#6  0xffffffff842a58f2 in acpi_table_parse_madt (id=id@entry=ACPI_MADT_TYPE_MULTIPROC_WAKEUP,
    handler=handler@entry=0xffffffff8426cc16 <acpi_parse_mp_wake>, max_entries=max_entries@entry=1)
    at linux/drivers/acpi/tables.c:462
#7  0xffffffff8426d9ac in acpi_process_madt () at linux/arch/x86/kernel/acpi/boot.c:1371
#8  acpi_boot_init () at linux/arch/x86/kernel/acpi/boot.c:1723
#9  0xffffffff84260a0b in setup_arch (cmdline_p=cmdline_p@entry=0xffffffff83803f08) at linux/arch/x86/kernel/setup.c:1273
#10 0xffffffff8425528c in start_kernel () at linux/init/main.c:969
#11 0xffffffff81000145 in secondary_startup_64 () at linux/arch/x86/kernel/head_64.S:358
#12 0x0000000000000000 in ?? ()
(gdb)
```
* 函数 `acpi_parse_mp_wake()` 解析 MADT 表中的 MP 唤醒条目
* arch/x86/kernel/acpi/boot.c
```c
static int __init acpi_parse_mp_wake(union acpi_subtable_headers *header,
                     const unsigned long end)
{
    struct acpi_madt_multiproc_wakeup *mp_wake;

    if (!IS_ENABLED(CONFIG_SMP))
        return -ENODEV;

    mp_wake = (struct acpi_madt_multiproc_wakeup *)header;
    if (BAD_MADT_ENTRY(mp_wake, end))
        return -EINVAL;

    acpi_table_print_madt_entry(&header->common);
    //读取 mailbox 的物理地址
    acpi_mp_wake_mailbox_paddr = mp_wake->base_address;
    //设置 ACPI 驱动的 wakeup AP 的回调为 acpi_wakeup_cpu()
    acpi_wake_cpu_handler_update(acpi_wakeup_cpu);

    return 0;
}
```
* arch/x86/kernel/apic/apic.c
```cpp
#ifdef CONFIG_X86_64
void __init acpi_wake_cpu_handler_update(wakeup_cpu_handler handler)
{
    struct apic **drv;

    for (drv = __apicdrivers; drv < __apicdrivers_end; drv++)
        (*drv)->wakeup_secondary_cpu_64 = handler;
}
#endif
```
* 猜测先通过以下路径得到 `real_mode_header` 的内容
```cpp
early_initcall(do_init_real_mode)
---> x86_platform.realmode_init()
  => init_real_mode()
     -> setup_real_mode()
           base = (unsigned char *)real_mode_header;
        -> memcpy(base, real_mode_blob, size);
```
### 唤醒 AP
* 于是，在 BSP 唤醒 AP 时有如下路径，**有几个 AP 该路径就会被 BSP 调用几次**：
```c
bringup_cpu()
-> __cpu_up()
   -> smp_ops.cpu_up()
   => native_cpu_up()
      -> do_boot_cpu()
         -> apic->wakeup_secondary_cpu_64(apicid, start_ip)
         => acpi_wakeup_cpu()

(gdb) bt
#0  acpi_wakeup_cpu (apicid=1, start_ip=631040) at linux/arch/x86/kernel/acpi/boot.c:351
#1  0xffffffff81077b98 in do_boot_cpu (apicid=apicid@entry=1, cpu=cpu@entry=1,
    cpu0_nmi_registered=cpu0_nmi_registered@entry=0xffa000000001bd0c, idle=<optimized out>, idle=<optimized out>)
    at linux/arch/x86/kernel/smpboot.c:1144
#2  0xffffffff81078292 in native_cpu_up (cpu=1, tidle=0xff110001005cc000) at linux/arch/x86/kernel/smpboot.c:1237
#3  0xffffffff81140a68 in __cpu_up (tidle=0xff110001005cc000, cpu=1) at linux/arch/x86/include/asm/smp.h:83
#4  bringup_cpu (cpu=1) at linux/kernel/cpu.c:608
#5  0xffffffff8113f879 in cpuhp_invoke_callback (cpu=cpu@entry=1, state=CPUHP_BRINGUP_CPU, bringup=bringup@entry=true,
    node=node@entry=0x0 <fixed_percpu_data>, lastp=lastp@entry=0x0 <fixed_percpu_data>) at linux/kernel/cpu.c:192
#6  0xffffffff8113fce9 in __cpuhp_invoke_callback_range (bringup=bringup@entry=true, cpu=cpu@entry=1, st=st@entry=0xff1100017bb1b960,
    target=target@entry=CPUHP_BRINGUP_CPU, nofail=nofail@entry=false) at linux/kernel/cpu.c:678
#7  0xffffffff81140e6b in cpuhp_invoke_callback_range (target=CPUHP_BRINGUP_CPU, st=0xff1100017bb1b960, cpu=1, bringup=true)
    at linux/kernel/cpu.c:1411
#8  cpuhp_up_callbacks (target=CPUHP_BRINGUP_CPU, st=0xff1100017bb1b960, cpu=1) at linux/kernel/cpu.c:733
#9  _cpu_up (cpu=cpu@entry=1, tasks_frozen=tasks_frozen@entry=0, target=CPUHP_BRINGUP_CPU, target@entry=CPUHP_ONLINE)
    at linux/kernel/cpu.c:1411
#10 0xffffffff8114105f in cpu_up (target=CPUHP_ONLINE, cpu=1) at linux/kernel/cpu.c:1447
#11 cpu_up (cpu=<optimized out>, target=CPUHP_ONLINE) at linux/kernel/cpu.c:1419
#12 0xffffffff811415c6 in bringup_nonboot_cpus (setup_max_cpus=8192) at linux/kernel/cpu.c:1513
#13 0xffffffff84285d7f in smp_init () at linux/kernel/smp.c:1112
#14 0xffffffff84255d20 in kernel_init_freeable () at linux/init/main.c:1637
#15 0xffffffff8241d946 in kernel_init (unused=<optimized out>) at linux/init/main.c:1534
#16 0xffffffff81006eb9 in ret_from_fork () at linux/arch/x86/entry/entry_64.S:311
#17 0x0000000000000000 in ?? ()
```
* 在 `do_boot_cpu()` 优先尝试用新的方式启用 AP，其中 `real_mode_header->trampoline_start` 就是 AP 的起始地址 `trampoline_start64`
```cpp
static int do_boot_cpu(int apicid, int cpu, struct task_struct *idle,
               int *cpu0_nmi_registered)
{
    /* start_ip had better be page-aligned! */
    unsigned long start_ip = real_mode_header->trampoline_start;
...
#ifdef CONFIG_X86_64
    /* If 64-bit wakeup method exists, use the 64-bit mode trampoline IP */
    if (apic->wakeup_secondary_cpu_64)
        start_ip = real_mode_header->trampoline_start64;
#endif
...
    /*
     * Wake up a CPU in difference cases:
     * - Use a method from the APIC driver if one defined, with wakeup
     *   straight to 64-bit mode preferred over wakeup to RM.
     * Otherwise,
     * - Use an INIT boot APIC message for APs or NMI for BSP.
     */
    if (apic->wakeup_secondary_cpu_64)
        boot_error = apic->wakeup_secondary_cpu_64(apicid, start_ip);
    else if (apic->wakeup_secondary_cpu)
        boot_error = apic->wakeup_secondary_cpu(apicid, start_ip);
    else
        boot_error = wakeup_cpu_via_init_nmi(cpu, start_ip, apicid,
                             cpu0_nmi_registered);
...
}
```
* `apic->wakeup_secondary_cpu_64(apicid, start_ip)` 就是前面通过 `acpi_wake_cpu_handler_update(acpi_wakeup_cpu)` 设置的 `acpi_wakeup_cpu()`
* `acpi_mp_wake_mailbox_paddr` 是前面解析 MADT 表的 *多处理器唤醒结构（Multiprocessor Wakeup Structure）* 找到的 MP wakeup mailbox struct 的物理地址
* 仅在第一次调用 `acpi_wakeup_cpu()` 时重新映射 mailbox 结构到当前内存地址空间
* 辅助 CPU 的唤醒在核心代码中完全序列化，因此无需保护 `acpi_mp_wake_mailbox` 免受并发访问
* Mailbox 内存在固件和操作系统之间共享。固件将监听 mailbox 命令地址，一旦收到 *唤醒命令*，与给定的 apicid 关联的辅助 CPU 将被启动
* 在 *唤醒命令* 可见之前，`apic_id` 和 `wakeup_vector` 的值必须对固件可见。`smp_store_release()` 确保排序和可见性
* 之后忙等待辅助 CPU 被唤醒
* 被唤醒的辅助 CPU 本质上是在自旋循环中等待被唤醒。它应该不会花很长时间唤醒并通过清零 ->command 进行确认
* ACPI 规范没有提供任何关于内核必须等待唤醒确认多长时间的指导。如果唤醒请求花费的时间太长，它也没有提供取消唤醒请求的方法
* 在 TDX 环境中，VMM 可以控制唤醒辅助 CPU 所需的时间
  * 它可以无限期推迟调度辅助 vCPU
  * 放弃唤醒请求并报告错误为 VMM 开启了可能的攻击途径：它可以在内核不期望的情况下唤醒辅助 CPU
  * 所以等待唤醒请求直到得到肯定的结果
* arch/x86/kernel/acpi/boot.c
```c
static int acpi_wakeup_cpu(int apicid, unsigned long start_ip)
{
    /*
     * Remap mailbox memory only for the first call to acpi_wakeup_cpu().
     *
     * Wakeup of secondary CPUs is fully serialized in the core code.
     * No need to protect acpi_mp_wake_mailbox from concurrent accesses.
     */
    if (!acpi_mp_wake_mailbox) {
        acpi_mp_wake_mailbox = memremap(acpi_mp_wake_mailbox_paddr,
                        sizeof(*acpi_mp_wake_mailbox),
                        MEMREMAP_WB);
    }

    /*
     * Mailbox memory is shared between the firmware and OS. Firmware will
     * listen on mailbox command address, and once it receives the wakeup
     * command, the CPU associated with the given apicid will be booted.
     *
     * The value of 'apic_id' and 'wakeup_vector' must be visible to the
     * firmware before the wakeup command is visible.  smp_store_release()
     * ensures ordering and visibility.
     */
    acpi_mp_wake_mailbox->apic_id       = apicid;
    acpi_mp_wake_mailbox->wakeup_vector = start_ip;
    smp_store_release(&acpi_mp_wake_mailbox->command,
              ACPI_MP_WAKE_COMMAND_WAKEUP);

    /*
     * Wait for the CPU to wake up.
     *
     * The CPU being woken up is essentially in a spin loop waiting to be
     * woken up. It should not take long for it wake up and acknowledge by
     * zeroing out ->command.
     *
     * ACPI specification doesn't provide any guidance on how long kernel
     * has to wait for a wake up acknowledgement. It also doesn't provide
     * a way to cancel a wake up request if it takes too long.
     *
     * In TDX environment, the VMM has control over how long it takes to
     * wake up secondary. It can postpone scheduling secondary vCPU
     * indefinitely. Giving up on wake up request and reporting error opens
     * possible attack vector for VMM: it can wake up a secondary CPU when
     * kernel doesn't expect it. Wait until positive result of the wake up
     * request.
     */
    while (READ_ONCE(acpi_mp_wake_mailbox->command))
        cpu_relax();

    return 0;
}
```
### TDVF 对 Mailbox 的操作
* TDVF 的代码见 EDK2 的仓库：https://github.com/tianocore/edk2.git
* 由以下 commit 引入 Mailbox 的支持
```c
commit 6a608255bb431a1b71e49899763ca72108f8ed3f
Author: Min Xu <min.m.xu@intel.com>
Date:   Sun Jul 18 10:45:30 2021 +0800

    OvmfPkg: Add TdxMailboxLib

    RFC: https://bugzilla.tianocore.org/show_bug.cgi?id=3429

    In Tdx BSP may issues commands to APs for some task, for example, to
    accept pages paralelly. BSP also need to wait until all the APs have
    done the task. TdxMailboxLib wraps these common funtions for BSP.
```

#### AP 循环等待接收 Mailbox 的命令
* edk2/OvmfPkg/TdxDxe/X64/ApRunLoop.nasm 是 guest TD 中 AP 运行循环的汇编代码
```asm
%include "TdxCommondefs.inc"

DEFAULT REL

SECTION .text

BITS 64

%define TDVMCALL_EXPOSE_REGS_MASK       0xffec
%define TDVMCALL                        0x0
%define EXIT_REASON_CPUID               0xa

%macro  tdcall  0
  db  0x66, 0x0f, 0x01, 0xcc
%endmacro
; 重定位 AP mailbox 循环
;
; Relocated Ap Mailbox loop
;
; @param[in]  RBX:  Relocated mailbox address
; @param[in]  RBP:  vCpuId
;
; @return     None  This routine does not return
;
global ASM_PFX(AsmRelocateApMailBoxLoop)
ASM_PFX(AsmRelocateApMailBoxLoop):
AsmRelocateApMailBoxLoopStart:

    mov         rax, TDVMCALL ;准备发的 tdcall 是 TDG.VP.VMCALL，即 TDVMCALL
    mov         rcx, TDVMCALL_EXPOSE_REGS_MASK ;需要暴露（传递）给 VMM 的 TD guest GPRs 的位图
    xor         r10, r10      ;%r10 为 0 表示 TDVMCALL
    mov         r11, EXIT_REASON_CPUID ;TDG.VP.VMCALL<Instruction.CPUID>
    mov         r12, 0xb      ;%r12 对应到 cpuid 的 %eax 寄存器，这里填入 0xb，获取 x2APIC 信息
    tdcall                    ;发起 tdcall，TDG.VP.VMCALL<Instruction.CPUID>
    test        r10, r10      ;TDVMCALL leaf 返回码是放在 %r10
    jnz         Panic         ;TDVMCALL leaf 返回码不为 0 均作为 panic 处理
    mov         r8, r15       ;返回后 %r15 对应到 cpuid 的 %edx 寄存器，x2APIC ID，将该值传入 %r8

MailBoxLoop:
    ; Spin until command set
    cmp        dword [rbx + CommandOffset], MpProtectedModeWakeupCommandNoop ;当前 command 域是 Noop(0) 吗？
    je         MailBoxLoop    ;是，回到上面 MailBoxLoop 的 label；不是，往下走
    ; Determine if this is a broadcast or directly for my apic-id, if not, ignore
    cmp        dword [rbx + ApicidOffset], MailboxApicidBroadcast ;APIC ID 域的值是广播吗？
    je         MailBoxProcessCommand ;是，直接跳到处理 mailbox 的命令；不是，往下走
    cmp        dword [rbx + ApicidOffset], r8d ;BSP 要唤醒的是 我吗？比较 APIC ID 域和我的 x2APIC ID
    jne        MailBoxLoop    ;不是，回到上面 MailBoxLoop 的 label；是，往下走
MailBoxProcessCommand:
    cmp        dword [rbx + CommandOffset], MpProtectedModeWakeupCommandWakeup ;当前 command 域是 Wakeup(1) 吗？
    je         MailBoxWakeUp  ;是，跳到唤醒 AP 的例程；不是，判断是不是下一个命令
    cmp        dword [rbx + CommandOffset], MpProtectedModeWakeupCommandSleep ;当前 command 域是 Sleep 吗？
    je         MailBoxSleep   ;是，跳到 AP 睡眠的例程；不是，往下走
    ; Don't support this command, so ignore
    jmp        MailBoxLoop    ;对于不支持的命令，忽略，回到循环的起点
MailBoxWakeUp:
    mov        rax, [rbx + WakeupVectorOffset] ;WakeupVector 域是 AP 唤醒后第一条指令的地址，传入 %rax
    ; OS sends a wakeup command for a given APIC ID, firmware is supposed to reset
    ; the command field back to zero as acknowledgement.
    mov        qword [rbx + CommandOffset], 0  ;固件重置 command 域为零，表示固件 ACK 这个命令
    jmp        rax            ;跳转到 AP 的启动地址，即 BSP 的内核提供的 trampoline_start64
MailBoxSleep:
    jmp       $               ;$ 属于“隐式地”藏在本行前的标号，也就是当前安排的地址，每一行都有，jmp $ 即在此条指令循环
Panic:
    ud2                       ;用 ud 指令引发 panic
BITS 64
AsmRelocateApMailBoxLoopEnd:

;-------------------------------------------------------------------------------------
;  AsmGetRelocationMap (&RelocationMap);
;-------------------------------------------------------------------------------------
global ASM_PFX(AsmGetRelocationMap)
ASM_PFX(AsmGetRelocationMap):
    lea        rax, [AsmRelocateApMailBoxLoopStart]
    mov        qword [rcx], rax
    mov        qword [rcx +  8h], AsmRelocateApMailBoxLoopEnd - AsmRelocateApMailBoxLoopStart
    ret
```

### 禁用 TDX Guest 对 CPU online/offline 的控制
* 与常规 VM 不同，TDX guest 在引导过程中使用固件切换唤醒方法来唤醒 AP。此唤醒模型使用 mailbox 与固件通信以启动 AP
* 根据设计，此 mailbox 只能用于给定 AP 一次，这意味着在 AP 启动后，不能使用同一 mailbox 使给定 AP offline/online
  * 有关此要求的更多详细信息，请参阅 Intel TDX Virtual Firmware Design Guide，标题为“AP initialization in OS”部分和标题为“Hotplug Device”的部分
* 由于该体系结构不支持任何方式使 CPU offline，因此在内核中禁用 CPU 热插拔支持
* 由于此热插拔禁用功能可被其他 VM guest 重新使用，因此添加新的 CC 属性 `CC_ATTR_HOTPLUG_DISABLED` 并使用它来禁用热插拔支持
* 尝试使 CPU offline 将失败并显示 `-EOPNOTSUPP`
```c
drivers/base/cpu.c
register_cpu()
   cpu->dev.bus = &cpu_subsys;
-> device_register()

cpu_subsys_offline()
-> cpu_device_down()
   -> cpu_down_maps_locked()
         if (cc_platform_has(CC_ATTR_HOTPLUG_DISABLED))
            return -EOPNOTSUPP;
```

## `CR0.NE` 的处理

### `CR0.NE` 位
* `CR0.NE` 即 **数字错误 Numeric Error**（`CR0`寄存器的第 `5` 位）
  * 在设置时，启用原生（内部）机制报告 x87 FPU 错误
  * 清零时启用 PC 风格的 x87 FPU 错误报告机制
* 当 `NE` 标志清零且 `IGNNE#`（ignore numeric error 引脚）输入有效时，x87 FPU 错误将被忽略
* 当 `NE` 标志清零且 `IGNNE#` 输入无效时，未屏蔽的 x87 FPU 错误会导致处理器使 `FERR#`（浮点数错误）引脚有效以生成外部中断，并在执行下一个等待的浮点指令或 `WAIT/FWAIT` 指令之前立即停止指令执行
* `FERR#` 引脚用于驱动外部中断控制器的输入（`FERR#` 引脚模拟 Intel 287 和 Intel 387 DX 数学协处理器的 `ERROR#` 引脚）
* `NE` 标志、`IGNNE#` 引脚和 `FERR#` 引脚与外部逻辑一起使用以实现 PC 式错误报告
* 现代操作系统不推荐使用 `FERR#` 和 `IGNNE#` 来处理浮点异常
* 这种非原生的方法还限制较新的处理器在一个逻辑处理器处于活动状态时运行

### TDX 对 `CR0.NE` 的处理
* TDX guest 需要设置 `CR0.NE`。清除该位会触发 `#GP(0)`
* 如果 `CR0.NE` 为 `0`，则选择处理浮点异常的 MS-DOS 兼容模式。在此模式下，使用处理器的 `FERR#`、`INTR` 和 `IGNNE#` 引脚从外部调用浮点异常的软件异常处理程序
* 不推荐使用 `FERR#` 和 `IGNNE#` 处理浮点异常。`CR0.NE=0` 还限制较新的处理器在一个逻辑处理器处于活动状态时运行
* 内核使用 `CR0_STATE` 常量来初始化 `CR0`。它设置了 `NE` 位
* 但是在早期引导期间，内核有更多的临时方法来设置寄存器中的位。在某些临时操作期间，`CR0.NE` 被清除。这会导致 TDX guest 中出现 `#GP`，并使其在早期启动时死亡
* 使 `CR0` 初始化一致，从 `CR0_STATE` 导出 `CR0` 的初始值。由于 `CR0_STATE` 始终具有 `CR0.NE=1`，这确保 `CR0.NE` 永远不会为 `0` 并避免 `#GP`
* arch/x86/include/uapi/asm/processor-flags.h
```c
#define CR0_STATE   (X86_CR0_PE | X86_CR0_MP | X86_CR0_ET | \
             X86_CR0_NE | X86_CR0_WP | X86_CR0_AM | \
             X86_CR0_PG)
```

### TDX 平台启动时避免 `#VE`
* 内核通常需要在引导期间修改一些 MSR 和控制寄存器位。但是，TDX 不允许修改这些寄存器以帮助提供一致的安全保证。幸运的是，TDX 确保这些在内核加载之前都处于正确的状态，这意味着内核不需要修改它们
* TD VM 启动要避免的条件是：
  * 对 `EFER` MSR 的任何写入
  * 清除 `CR4.MCE`
* 这在理论上使 guest 引导更加脆弱了
  * 例如，如果 `EFER` 设置不正确并且执行了 `WRMSR`，它将触发（guest）早期异常 panic，或者，如果它是在早期异常设置之前，则会引发 triple fault。然而，这很可能在控制权到达内核之前很久就绊倒了 guest BIOS
  * 无论如何，这类问题不太可能出现在生产环境中，开发人员有很好的调试工具可以快速修复它们
* `EFER` MSR 相关 bit 见 SDM vol3. 2.2.1 Extended Feature Enable Register
* arch/x86/boot/compressed/head_64.S
```diff
--- a/arch/x86/boot/compressed/head_64.S
+++ b/arch/x86/boot/compressed/head_64.S
@@ -642,12 +642,28 @@ SYM_CODE_START(trampoline_32bit_src)
    movl    $MSR_EFER, %ecx
    rdmsr
    btsl    $_EFER_LME, %eax
+   /* Avoid writing EFER if no change was made (for TDX guest) */
+   jc  1f
    wrmsr
-   popl    %edx
+1: popl    %edx
    popl    %ecx

+#ifdef CONFIG_X86_MCE
+   /*
+    * Preserve CR4.MCE if the kernel will enable #MC support.
+    * Clearing MCE may fault in some environments (that also force #MC
+    * support). Any machine check that occurs before #MC support is fully
+    * configured will crash the system regardless of the CR4.MCE value set
+    * here.
+    */
+   movl    %cr4, %eax
+   andl    $X86_CR4_MCE, %eax
+#else
+   movl    $0, %eax
+#endif
+
    /* Enable PAE and LA57 (if required) paging modes */
-   movl    $X86_CR4_PAE, %eax
+   orl $X86_CR4_PAE, %eax
    testl   %edx, %edx
    jz  1f
    orl $X86_CR4_LA57, %eax
```
* 第一段修改的逻辑是：
  * `EFER` MSR 在内核加载之前都处于正确的状态，这意味着内核不需要修改它们
  * 如果 `EFER` MSR 确实没有被修改，那么 `jc 1f` 跳过 `wrmsr` 写 `EFER` MSR 的动作，避免 TD VM 产生 `#VE`
* 第二段修改的逻辑是：
  * 修改前 `movl $X86_CR4_PAE, %eax` 会无差别地清除 `CR4.MCE`，这会导致 TD VM 产生 `#VE`
  * 修改后不管 `CR4.MCE` 有没有设置，先保存它的值到 `%eax`，这样就避免因为清除 `CR4.MCE` 而产生 `#VE`
* arch/x86/kernel/head_64.S 的修改思路和上面一段类似
* arch/x86/realmode/rm/trampoline_64.S
```diff
--- a/arch/x86/realmode/rm/trampoline_64.S
+++ b/arch/x86/realmode/rm/trampoline_64.S
@@ -143,11 +143,22 @@ SYM_CODE_START(startup_32)
    movl    %eax, %cr3

    # Set up EFER
+   movl    $MSR_EFER, %ecx
+   rdmsr
+   /*
+    * Skip writing to EFER if the register already has desired
+    * value (to avoid #VE for the TDX guest).
+    */
+   cmp pa_tr_efer, %eax
+   jne .Lwrite_efer
+   cmp pa_tr_efer + 4, %edx
+   je  .Ldone_efer
+.Lwrite_efer:
    movl    pa_tr_efer, %eax
    movl    pa_tr_efer + 4, %edx
-   movl    $MSR_EFER, %ecx
    wrmsr

+.Ldone_efer:
    # Enable paging and in turn activate Long Mode.
    movl    $CR0_STATE, %eax
    movl    %eax, %cr0
```
* 将无条件写 `EFER` MSR `movl $MSR_EFER, %ecx` 改为有条件写
* 只有当 `EFER` MSR 和 `trampoline_header` 中 `tr_efer` 域的值不一样时才写
* 因为 `EFER` MSR 是 8 Byte，而这一段例程是运行在 32 位，因此比较是分两段比较，写也是分两段写（连寄存器都是 32 位的 `%eax`）

## 将 `ioremap()` 时的 pages 设为共享
* 在 TDX guest 中，guest 内存受到 host 访问保护。如果 guest 执行 I/O，它需要明确地与 host 共享 I/O 内存
* 将所有不受正常内存（`IORES_DESC_NONE` 或 `IORES_DESC_RESERVED`）支持的 `ioremap()`ed 页面映射为共享
* `PAGE_KERNEL_IO` 中的权限已经适用于 AMD SEV/SME 系统上的“解密”内存。这意味着他们不需要进行 `pgprot_decrypted()` 调用
  * AMD SEV/SME 的共享页面的第 63 位为 `0`，因此不需要在这里改什么
* 另一方面，TDX guest *需要* 更改为 `PAGE_KERNEL_IO` 以进行“解密”映射。为 TDX 添加 `pgprot_decrypted()`
```diff
diff --git a/arch/x86/mm/ioremap.c b/arch/x86/mm/ioremap.c
index 17a492c27306..1ad0228f8ceb 100644
--- a/arch/x86/mm/ioremap.c
+++ b/arch/x86/mm/ioremap.c
@@ -242,10 +242,15 @@ __ioremap_caller(resource_size_t phys_addr, unsigned long size,
     * If the page being mapped is in memory and SEV is active then
     * make sure the memory encryption attribute is enabled in the
     * resulting mapping.
+    * In TDX guests, memory is marked private by default. If encryption
+    * is not requested (using encrypted), explicitly set decrypt
+    * attribute in all IOREMAPPED memory.
     */
    prot = PAGE_KERNEL_IO;
    if ((io_desc.flags & IORES_MAP_ENCRYPTED) || encrypted)
        prot = pgprot_encrypted(prot);
+   else
+       prot = pgprot_decrypted(prot);

    switch (pcm) {
    case _PAGE_CACHE_MODE_UC:
```
* 例如，TD VM panic 的时候，crash kernel 会走到 fs/proc/vmcore.c:`__ioremap_caller()`
```c
elfcorehdr_read_notes() -> read_from_oldmem(&iter, count, ppos, cc_platform_has(CC_ATTR_MEM_ENCRYPT))
__read_vmcore() -> read_from_oldmem(iter, tsz, &start, cc_platform_has(CC_ATTR_MEM_ENCRYPT))
read_from_oldmem()
   if (encrypted)
   -> copy_oldmem_page_encrypted()
      -> __copy_oldmem_page(iter, pfn, csize, offset, true)
         if (encrypted)
         -> ioremap_encrypted(pfn << PAGE_SHIFT, PAGE_SIZE)
            -> __ioremap_caller(phys_addr, size, _PAGE_CACHE_MODE_WB, __builtin_return_address(0), true)
               -> ioremap_page_range(vaddr, vaddr + size, phys_addr, prot)
```

## 添加对 TDX 共享内存的支持
* TDX 保护 guest 内存免受 VMM 访问。因此，guest 与 VMM 通信所需的任何内存必须被显式转为共享。这是一个两步过程：
1. Guest 设置页表条目中的 shared bit 并通知 VMM 有关更改
2. 使用 `MapGPA` hypercall 发生通知
* 转换回私有内存需要清除 shared bit，使用 `MapGPA` hypercall 通知 VMM，然后使用 `AcceptPage` hypercall 接受内存
* 提供 `x86_platform.guest.*` 回调的 TDX 版本。它使 `__set_memory_enc_pgtable()` 在 TDX guest 中正常工作

### 是否需要刷新 TLB 回调 `tdx_tlb_flush_required()`
* TDX guest 负责在 *私有->共享* 转换时刷新 TLB，因为这个转换必然是由内而外发起的
* VMM 负责刷新 *共享->私有* 转换时刷新 TLB，因为页表是由 VMM （经过 TDX module）提供给 TD 的
* VMM *无法* 刷新私有地址，因为它无法使用 guest 的 HKID 生成 PA
* 共享内存不受完整性检查的约束，即 VMM 不需要刷新以保护自己
* 从 *共享转换为私有* 时无需（guest）刷新，因为在这种情况下刷新是 VMM 的责任，例如，VMM 必须刷新以避免在遇到错误或恶意 guest 时出现完整性故障
```c
static bool tdx_tlb_flush_required(bool private)
{
    /*
     * TDX guest is responsible for flushing TLB on private->shared
     * transition. VMM is responsible for flushing on shared->private.
     *
     * The VMM _can't_ flush private addresses as it can't generate PAs
     * with the guest's HKID.  Shared memory isn't subject to integrity
     * checking, i.e. the VMM doesn't need to flush for its own protection.
     *
     * There's no need to flush when converting from shared to private,
     * as flushing is the VMM's responsibility in this case, e.g. it must
     * flush to avoid integrity failures in the face of a buggy or
     * malicious guest.
     */
    return !private;
}
```
### 是否需要刷新 Cache 回调 `tdx_cache_flush_required()`
* 如果 HW 强制缓存一致性，AMD SME/SEV 可以避免 cache 刷新
* TDX 没有这样的能力，因此无条件刷新缓存
```c
static bool tdx_cache_flush_required(void)
{
    /*
     * AMD SME/SEV can avoid cache flushing if HW enforces cache coherence.
     * TDX doesn't have such capability.
     *
     * Flush cache unconditionally.
     */
    return true;
}
```
### 加密状态改变回调 `tdx_enc_status_changed()`
* `tdx_enc_status_changed()` 将 guest 对此物理页面的意图告知 VMM：
  * 与 VMM 共享该物理页面
  * 该页面仅对 guest 私有
* 通过 `tdvmcall(TDG.VP.VMCALL<MapGPA>)` 将转换申请转给 VMM 处理，详情请见 [TDX 共享和私有内存转换](tdx_share_to_private.md) 一文
  * 作为响应，VMM 应更改其页面映射
  * 对于 *私有->共享*，guest 的操作可以到此结束
  * 对于 *共享->私有*，guest 还需要 accept VMM 通过 augment 操作追加进来的处于 pending 状态的页面
```c
static bool try_accept_one(phys_addr_t *start, unsigned long len,
              enum pg_level pg_level)
{   //根据页的级别得到该级别一个页所覆盖的地址范围
    unsigned long accept_size = page_level_size(pg_level);
    u64 tdcall_rcx;
    u8 page_size;
    //地址不对齐到该页的大小，非法输入
    if (!IS_ALIGNED(*start, accept_size))
        return false;
    //传入的长度不符合页的大小，非法输入
    if (len < accept_size)
        return false;
    //将页面物理地址传递给 TDX module 以接受 pending 的私有页面
    /*
     * Pass the page physical address to the TDX module to accept the
     * pending, private page.
     *
     * Bits 2:0 of RCX encode page size: 0 - 4K, 1 - 2M, 2 - 1G.
     */
    switch (pg_level) {
    case PG_LEVEL_4K:
        page_size = TDX_PS_4K;
        break;
    case PG_LEVEL_2M:
        page_size = TDX_PS_2M;
        break;
    case PG_LEVEL_1G:
        page_size = TDX_PS_1G;
        break;
    default:
        return false;
    }

    tdcall_rcx = *start | page_size;
    if (__tdx_module_call(TDX_ACCEPT_PAGE, tdcall_rcx, 0, 0, 0, NULL))
        return false;

    *start += accept_size;
    return true;
}

/*
 * Inform the VMM of the guest's intent for this physical page: shared with
 * the VMM or private to the guest.  The VMM is expected to change its mapping
 * of the page in response.
 */
static bool tdx_enc_status_changed(unsigned long vaddr, int numpages, bool enc)
{
    phys_addr_t start = __pa(vaddr);
    phys_addr_t end   = __pa(vaddr + numpages * PAGE_SIZE);
    //如果意图是共享该页面，设置起止地址的 shared bit 为 1
    if (!enc) {
        /* Set the shared (decrypted) bits: */
        start |= cc_mkdec(0); //直接写成 start = cc_mkdec(start) 不更易读吗？
        end   |= cc_mkdec(0);
    }
    //通过 tdvmcall(TDG.VP.VMCALL<MapGPA>) 将转换申请转给 VMM 处理
    /*
     * Notify the VMM about page mapping conversion. More info about ABI
     * can be found in TDX Guest-Host-Communication Interface (GHCI),
     * section "TDG.VP.VMCALL<MapGPA>"
     */
    if (_tdx_hypercall(TDVMCALL_MAP_GPA, start, end - start, 0, 0))
        return false;
    //对于私有转共享，guest 的处理到此结束，不需要后面的操作
    /* private->shared conversion  requires only MapGPA call */
    if (!enc)
        return true;
    //对于共享转私有，host 会将请求的页面通过 augment 追加给 guest，guest 需要 accept 这些页面，完成转换
    /*
     * For shared->private conversion, accept the page using
     * TDX_ACCEPT_PAGE TDX module call.
     */
    while (start < end) {
        unsigned long len = end - start;
        //首先尝试接受更大的页面。让 VMM 有机会在可能的情况下保留 1G/2M SEPT 条目，并通过减少 hypercalls 的数量（如果成功）来加快处理
        /*
         * Try larger accepts first. It gives chance to VMM to keep
         * 1G/2M SEPT entries where possible and speeds up process by
         * cutting number of hypercalls (if successful).
         */

        if (try_accept_one(&start, len, PG_LEVEL_1G))
            continue;

        if (try_accept_one(&start, len, PG_LEVEL_2M))
            continue;

        if (!try_accept_one(&start, len, PG_LEVEL_4K))
            return false;
    }

    return true;
}
```
## TD Guest 对于 IOAPIC 的修改
* 内核通过一个特殊的 MMIO 页面与每个 bare-metal IOAPIC 交互。在 KVM 下运行时，guest 的 IOAPIC 由 KVM 模拟
* 当作为一个 TDX guest 运行时，guest 需要将每个 IOAPIC 映射标记为与 host “共享”。这确保 TDX 私有保护不会应用于该页面，从而允许 TDX host 完成模拟
* 虽然 `ioremap()` 创建的映射（例如 virtio）将默认标记为共享，但是，IOAPIC 代码不使用 `ioremap()` 而是使用 fixmap 机制
* 为 IOAPIC 代码引入一个特殊的 fixmap helpers，确保将 IOAPIC 页面标记为“共享”
  * 用 `__set_fixmap()` 替换了 `set_fixmap_nocache()`，因为 `__set_fixmap()` 允许自定义 `prot` 的值
* AMD SEV 共享 IOAPIC 页面，因为 `FIXMAP_PAGE_NOCACHE` 已清除 `_ENC` 位。而 TDX 必须设置该位才能与 host 共享页面
```c
static void io_apic_set_fixmap(enum fixed_addresses idx, phys_addr_t phys)
{
    pgprot_t flags = FIXMAP_PAGE_NOCACHE;

    /*
     * Ensure fixmaps for IOAPIC MMIO respect memory encryption pgprot
     * bits, just like normal ioremap():
     */
    flags = pgprot_decrypted(flags);

    __set_fixmap(idx, phys, flags);
}
```
* `pgprot_decrypted(flags)` 把 shared bit 加入到 `flags` 里
* `pfn_pte(phys >> PAGE_SHIFT, flags)` 制作 PTE 的时候会把 shared bit 带上
  * arch/x86/mm/pgtable.c
```c
void native_set_fixmap(unsigned /* enum fixed_addresses */ idx,
               phys_addr_t phys, pgprot_t flags)
{
    /* Sanitize 'prot' against any unsupported bits: */
    pgprot_val(flags) &= __default_kernel_pte_mask;

    __native_set_fixmap(idx, pfn_pte(phys >> PAGE_SHIFT, flags));
}
```

## 避免在 VM 内部刷新 Cache
* 在虚拟机内部运行时，内核可以绕过缓存刷新。更改虚拟机中的睡眠状态（即 VM 睡眠）不会影响 host 系统的睡眠状态，也不会导致数据丢失
* 在进入睡眠状态之前，ACPI 代码使用 `WBINVD` 指令刷新缓存以防止数据丢失。这种机制在 bare metal 上是必需的
* 但是，在 guest 内部使用 `WBINVD` 是毫无价值的
  * 更改虚拟机中的睡眠状态不会影响 host 系统的睡眠状态，也不会导致数据丢失，因此大多数 hypervisors 会简单地忽略它
  * 尽管如此，ACPI 代码无论如何都会无条件地调用 `WBINVD`，尽管它没有用，但通常也是无害的
* 然而，在 TDX guest 中，`WBINVD` 不再是无害的；它会触发虚拟化异常 (`#VE`) 如果 ACPI cache-flushing `WBINVD` 的代码还保留原位，TDX guest 将需要处理以从异常中恢复
* 每当 kernel 在 hypervisor 下运行时（即 TD guest），需避免使用 `WBINVD`。这既删除了无用的 `WBINVD`，又避免了 TDX 实施 `WBINVD` 处理
```diff
diff --git a/arch/x86/include/asm/acenv.h b/arch/x86/include/asm/acenv.h
index 9aff97f0de7f..d937c55e717e 100644
--- a/arch/x86/include/asm/acenv.h
+++ b/arch/x86/include/asm/acenv.h
@@ -13,7 +13,19 @@

 /* Asm macros */

-#define ACPI_FLUSH_CPU_CACHE() wbinvd()
+/*
+ * ACPI_FLUSH_CPU_CACHE() flushes caches on entering sleep states.
+ * It is required to prevent data loss.
+ *
+ * While running inside virtual machine, the kernel can bypass cache flushing.
+ * Changing sleep state in a virtual machine doesn't affect the host system
+ * sleep state and cannot lead to data loss.
+ */
+#define ACPI_FLUSH_CPU_CACHE()                 \
+do {                               \
+   if (!cpu_feature_enabled(X86_FEATURE_HYPERVISOR))   \
+       wbinvd();                   \
+} while (0)

 int __acpi_acquire_global_lock(unsigned int *lock);
 int __acpi_release_global_lock(unsigned int *lock);
```
## 引入 `tdx_panic()` 来报告早期错误
* commit `71acdcd7cd0abee7698c2d5e5a8a66602a00b12c` x86/tdx: Use ReportFatalError to report missing SEPT_VE_DISABLE
* Linux TDX guest 需要设置 `SEPT_VE_DISABLE` TD Attribute
  * 如果未设置，理论上内核需要在访问内核内存的任何位置处理异常，包括 NMI 处理程序和系统调用入口间隙等位置
* 在此 commit 前的做法是，如果 `SEPT_VE_DISABLE` 未设置，内核甚至不会尝试处理这些异常，而是拒绝运行
* 然而，`SEPT_VE_DISABLE` 检测和拒绝代码在启动的早期就发生了，甚至在 `earlyprintk` 运行之前。调用 `panic()` 将有效地挂起系统
* 取而代之的是，调用特定于 TDX 的 `panic()` 函数。这使得一个非常简单的 TDVMCALL 可以在没有任何控制台基础设施的情况下将一个简短的错误字符串发送到 hypervisor
* 使用 `TDG.VP.VMCALL<ReportFatalError>` 报告错误。`hypercall` 可以在八个寄存器中对的消息进行编码，因此消息的长度最多有 64 字节
  * arch/x86/coco/tdx/tdx.c
```cpp
static void __noreturn tdx_panic(const char *msg)
{
    struct tdx_hypercall_args args = {
        .r10 = TDX_HYPERCALL_STANDARD, //表明是 TDVMCALL，%r10 为 0
        .r11 = TDVMCALL_REPORT_FATAL_ERROR, //leaf ID
        .r12 = 0, /* Error code: 0 is Panic */
    };
    union {
        /* Define register order according to the GHCI */
        struct { u64 r14, r15, rbx, rdi, rsi, r8, r9, rdx; };

        char str[64];
    } message;
    //将错误消息编码到 8 个寄存器中
    /* VMM assumes '\0' in byte 65, if the message took all 64 bytes */
    strncpy(message.str, msg, 64);

    args.r8  = message.r8;
    args.r9  = message.r9;
    args.r14 = message.r14;
    args.r15 = message.r15;
    args.rdi = message.rdi;
    args.rsi = message.rsi;
    args.rbx = message.rbx;
    args.rdx = message.rdx;

    /*
     * This hypercall should never return and it is not safe
     * to keep the guest running. Call it forever if it
     * happens to return.
     */
    while (1)
        __tdx_hypercall(&args, 0);
}
```
* 当 TDX guest 由于接触未正确映射到“secure EPT”的内存时，会出现“SEPT `#VE`”，这可能是 hypervisor 的攻击或 bug 导致的，*或* guest 的 bug
* 最值得注意的是，有 buggy guest 可能会因许多不同的内存安全漏洞（如缓冲区溢出）而接触未接受的内存
* TDX guest 不想在 hypervisor 的攻击或 bug 面前继续，他们希望尽可能快和安全地终止。`SEPT_VE_DISABLE` 确保 TDX guest 在遇到此类问题时 *不能* 继续
* 但是，这会导致问题：无法继续的 TDX guest 无法吐出 oopses 或其他调试信息。本质上 `SEPT_VE_DISABLE=1` 的 guest 是不可调试的，因此
  * 当调试 TD 时放宽 `SEPT_VE_DISABLE` 检查，仅仅发出警告
  * 在 `#VE` handler 中对私有内存上的 EPT-violation 引发 `panic()`，它将产生有用的回溯

## 禁用 TD Guest 的 `NOTIFY_ENABLES` 功能
* commit `8de62af018cc9262649d7688f7eb1409b2d8f594` x86/tdx: Disable NOTIFY_ENABLES
### 背景
* 有一类针对 SGX enclave 的 side-channel 攻击称为“SGX Step”[1]。这些攻击在 enclave 内部产生了许多异常。基本上，运行一条 in-enclave 指令，导致一次异常。一遍又一遍
* 有人担心 VMM 可能会通过引起大量 `#VE` 以相同的方式攻击 TDX guest。TDX 架构包含针对这些攻击的新对策：总的来说，它通过计算异常的数量，并且一旦 VMM 引发的 `#VE` 的数量达到临界阈值[2]，就可以发送另一个 *特殊* 异常
- [1] https://github.com/jovanbulck/sgx-step
- [2] https://intel.github.io/ccc-linux-guest-hardening-docs/security-spec.html#safety-against-ve-in-kernel-code

### 问题
* 但是，这些特殊异常与 guest 采取的任何操作无关
  * 它们可以发生在 guest 执行的任何地方，包括敏感区域，例如键入密码
  * （non-paranoid）`#VE` 处理程序无法处理这些区域中的异常

### 解决方案
* 幸运的是，guest 可以通过写入 TDCS 的 `NOTIFY_ENABLES` 字段来禁用该特殊异常
  * `NOTIFY_ENABLES` 默认情况下被禁用，但可能在当前内核运行之前由引导加载程序、固件或早期内核启用
* 显式且无条件地禁用 `NOTIFY_ENABLES` 功能
  * 在此之前发生的任何基于 `NOTIFY_ENABLES` 的 `#VE` 将在早期的 `#VE` 异常处理程序中结束并由于意外退出而死亡

## 不要禁用已锁定的 x2APIC
* commit `b8d1d163604bd1e600b062fb00de5dc42baa355f` x86/apic: Don't disable x2APIC if locked
* APIC 支持两种模式，传统 APIC（或 xAPIC）和扩展 APIC（或 x2APIC）
  * x2APIC 模式主要与 legacy APIC 兼容，但它禁用内存映射 APIC 接口以支持使用 MSR 的接口
  * APIC 模式由 APIC MSR 中的 `EXT` 位控制
* MMIO/xAPIC 接口有一些问题，最显着的是 APIC LEAK [1]。此错误允许攻击者使用 APIC MMIO 接口从 SGX enclave 提取数据
* 引入对一项新功能的支持，该功能将允许 BIOS 在 x2APIC 模式下锁定 APIC
  * 如果 APIC 在 x2APIC 模式下被锁定并且内核试图禁用 APIC 或恢复到 legacy APIC 模式，则会发生 `#GP` 异常
* 引入对新 MSR（`IA32_XAPIC_DISABLE_STATUS`）的支持，当设置 `LEGACY_XAPIC_DISABLED` bit 时，通过 *阻止内核尝试禁用 x2APIC* 来处理新的锁定模式
* 在具有 `IA32_XAPIC_DISABLE_STATUS` MSR 的平台上，如果启用了 SGX 或 TDX，则 BIOS 将设置 `LEGACY_XAPIC_DISABLED` 禁用 legacy APIC
  * 反之，如果需要 legacy APIC，则需要在 BIOS 中禁用 SGX 和 TDX
* [1]: https://aepicleak.com/aepicleak.pdf
```diff
diff --git a/arch/x86/Kconfig b/arch/x86/Kconfig
index f9920f1341c8..159c025ebb03 100644
--- a/arch/x86/Kconfig
+++ b/arch/x86/Kconfig
@@ -448,6 +448,11 @@ config X86_X2APIC
      This allows 32-bit apic IDs (so it can support very large systems),
      and accesses the local apic via MSRs not via mmio.

+     Some Intel systems circa 2022 and later are locked into x2APIC mode
+     and can not fall back to the legacy APIC modes if SGX or TDX are
+     enabled in the BIOS.  They will be unable to boot without enabling
+     this option.
+
      If you don't know what to do here, say N.

 config X86_MPPARSE
@@ -1919,7 +1924,7 @@ endchoice

 config X86_SGX
    bool "Software Guard eXtensions (SGX)"
-   depends on X86_64 && CPU_SUP_INTEL
+   depends on X86_64 && CPU_SUP_INTEL && X86_X2APIC
    depends on CRYPTO=y
    depends on CRYPTO_SHA256=y
    select SRCU
```
* `x2apic_hw_locked()` 判断 x2APIC 是否被硬件锁定
```cpp
enum {
    X2APIC_OFF,
    X2APIC_DISABLED,
    /* All states below here have X2APIC enabled */
    X2APIC_ON,
    X2APIC_ON_LOCKED
};
static int x2apic_state;

static bool x2apic_hw_locked(void)
{
    u64 ia32_cap;
    u64 msr;
    //读取硬件能力的 MSR
    ia32_cap = x86_read_arch_cap_msr();
    if (ia32_cap & ARCH_CAP_XAPIC_DISABLE) {        //如果 Arch 有禁用 xAPIC 的能力
        rdmsrl(MSR_IA32_XAPIC_DISABLE_STATUS, msr); //读取 IA32_XAPIC_DISABLE_STATUS MSR
        return (msr & LEGACY_XAPIC_DISABLED);       //LEGACY_XAPIC_DISABLED 位被设置意味着 x2APIC 被硬件锁定
    }
    return false;
}
```
* 如果通过 kernel command line 设置了 `nox2apic` 选项想关闭 x2APIC，由 `setup_nox2apic()` 处理
```cpp
static int __init setup_nox2apic(char *str)
{   //如果 x2APIC 已经启用
    if (x2apic_enabled()) {
        int apicid = native_a.pic_msr_read(APIC_ID); //读取 APIC ID
        //如果 APIC ID 多于 255，必须使用 x2APIC，因为 xAPIC 的 ID 域仅有 8 位，而 x2APIC 则支持多达 4G 个 ID
        if (apicid >= 255) {
            pr_warn("Apicid: %08x, cannot enforce nox2apic\n",
                apicid);
            return 0;
        }
        if (x2apic_hw_locked()) { //如果硬件已经锁定 x2APIC 则无法通过选项关闭
            pr_warn("APIC locked in x2apic mode, can't disable\n");
            return 0;
        }
        pr_warn("x2apic already enabled.\n");
        __x2apic_disable(); //如果没以上限制，则通过选项禁用 x2APIC
    }
    setup_clear_cpu_cap(X86_FEATURE_X2APIC);
    x2apic_state = X2APIC_DISABLED;
    x2apic_mode = 0;
    return 0;
}
early_param("nox2apic", setup_nox2apic);
```
* `x2apic_setup()` 的调用者为 `cpu_init()`，被用于初始化 AP
* 该函数的目的是使 AP 的 x2APIC 状态与 BSP 保持一致
  * 如果 BSP 的 x2apic_state 不是锁定 x2APIC 状态，但 AP 是锁定状态，发出警告
  * 如果 BSP 的 x2APIC 未开启或者未被锁定在开启状态，AP 的 x2APIC 也需被禁用
  * 否则，BSP 的 x2APIC 处于开启或被锁定在开启状态，AP 的 x2APIC 也需要被启用
```cpp
/* Called from cpu_init() to enable x2apic on (secondary) cpus */
void x2apic_setup(void)
{   //如果 BSP 的 x2apic_state 不是锁定 x2APIC 状态，但 AP 是锁定状态，发出警告
    /*
     * Try to make the AP's APIC state match that of the BSP,  but if the
     * BSP is unlocked and the AP is locked then there is a state mismatch.
     * Warn about the mismatch in case a GP fault occurs due to a locked AP
     * trying to be turned off.
     */
    if (x2apic_state != X2APIC_ON_LOCKED && x2apic_hw_locked())
        pr_warn("x2apic lock mismatch between BSP and AP.\n");
    /*
     * If x2apic is not in ON or LOCKED state, disable it if already enabled
     * from BIOS.
     */
    if (x2apic_state < X2APIC_ON) {
        __x2apic_disable();
        return;
    }
    __x2apic_enable();
}
```
* x2APIC 禁用函数 `x2apic_disable()` 不会让 x2APIC 被轻易地禁掉，会做一系列检查
```cpp
static __init void x2apic_disable(void)
{
    u32 x2apic_id, state = x2apic_state;

    x2apic_mode = 0;
    x2apic_state = X2APIC_DISABLED;

    if (state != X2APIC_ON)
        return;
    //读取 APIC ID，多于 255 个无法禁用 x2APIC，原因前面说过
    x2apic_id = read_apic_id();
    if (x2apic_id >= 255)
        panic("Cannot disable x2apic, id: %08x\n", x2apic_id);
    //硬件锁定开启状态也无法被禁用
    if (x2apic_hw_locked()) {
        pr_warn("Cannot disable locked x2apic, id: %08x\n", x2apic_id);
        return;
    }
    //如果能通过以上检查则可以真正实施禁用操作
    __x2apic_disable();
    register_lapic_address(mp_lapic_addr);
}
```

## Guest MCE
* 值得注意的，当报告内存错误时，某些系统报告 `IA32_MCi_ADDR` machine check MSR 可能会含有 keyID bit。将物理地址转为 PFN 传给 `memory_failure()` 前需要将 keyID bit 去掉
```c
commit 8a01ec97dc066009dd89e43bfcf55644f2dd6d19
Author: Tony Luck <tony.luck@intel.com>
Date:   Mon Jan 9 07:29:36 2023 -0800

    x86/mce: Mask out non-address bits from machine check bank
```

## References
- [Nasm $,$$和vstart,org - 字节流](http://www.bytekits.com/nasm/dollar-flag.html)