# MCE

## 注册 MCE 中断处理函数
```c
start_kernel()
-> check_bugs()
   -> identify_boot_cpu()
      -> identify_cpu(&boot_cpu_data)
         -> mcheck_cpu_init(c)
            -> machine_check_vector = do_machine_check;
```
* Machine Check 的中断向量`X86_TRAP_MC`（中断号是`18`）
  * arch/x86/include/asm/trapnr.h
  ```c
  /* Interrupts/Exceptions */

  #define X86_TRAP_DE      0  /* Divide-by-zero */
  #define X86_TRAP_DB      1  /* Debug */
  #define X86_TRAP_NMI         2  /* Non-maskable Interrupt */
  #define X86_TRAP_BP      3  /* Breakpoint */
  #define X86_TRAP_OF      4  /* Overflow */
  #define X86_TRAP_BR      5  /* Bound Range Exceeded */
  #define X86_TRAP_UD      6  /* Invalid Opcode */
  #define X86_TRAP_NM      7  /* Device Not Available */
  #define X86_TRAP_DF      8  /* Double Fault */
  #define X86_TRAP_OLD_MF      9  /* Coprocessor Segment Overrun */
  #define X86_TRAP_TS     10  /* Invalid TSS */
  #define X86_TRAP_NP     11  /* Segment Not Present */
  #define X86_TRAP_SS     12  /* Stack Segment Fault */
  #define X86_TRAP_GP     13  /* General Protection Fault */
  #define X86_TRAP_PF     14  /* Page Fault */
  #define X86_TRAP_SPURIOUS   15  /* Spurious Interrupt */
  #define X86_TRAP_MF     16  /* x87 Floating-Point Exception */
  #define X86_TRAP_AC     17  /* Alignment Check */
  #define X86_TRAP_MC     18  /* Machine Check */
  #define X86_TRAP_XF     19  /* SIMD Floating-Point Exception */
  #define X86_TRAP_VE     20  /* Virtualization Exception */
  #define X86_TRAP_CP     21  /* Control Protection Exception */
  #define X86_TRAP_VC     29  /* VMM Communication Exception */
  #define X86_TRAP_IRET       32  /* IRET Exception */
  ```

* 定义 Machine Check 的中断向量`X86_TRAP_MC`的中断描述符表项
  * arch/x86/kernel/idt.c
  ```c
  ...
  #define G(_vector, _addr, _ist, _type, _dpl, _segment)  \
      {                       \
          .vector     = _vector,      \
          .bits.ist   = _ist,         \
          .bits.type  = _type,        \
          .bits.dpl   = _dpl,         \
          .bits.p     = 1,            \
          .addr       = _addr,        \
          .segment    = _segment,     \
      }
  ...
  #ifdef CONFIG_X86_64
  /*
  * Interrupt gate with interrupt stack. The _ist index is the index in
  * the tss.ist[] array, but for the descriptor it needs to start at 1.
  */
  #define ISTG(_vector, _addr, _ist)          \
      G(_vector, _addr, _ist + 1, GATE_INTERRUPT, DPL0, __KERNEL_CS)
  #else
  #define ISTG(_vector, _addr, _ist)  INTG(_vector, _addr)
  #endif
  ...
  /*
  * The default IDT entries which are set up in trap_init() before
  * cpu_init() is invoked. Interrupt stacks cannot be used at that point and
  * the traps which use them are reinitialized with IST after cpu_init() has
  * set up TSS.
  */
  static const __initconst struct idt_data def_idts[] = {
      ...
  #ifdef CONFIG_X86_MCE
      ISTG(X86_TRAP_MC,       asm_exc_machine_check, IST_INDEX_MCE),
  #endif
      ...
  };
  ```
  * 由此可见，其中断处理函数为`asm_exc_machine_check`

## MCE 中断处理函数的声明
* `idtentry_mce_db`宏的定义
  * arch/x86/entry/entry_64.S
  ```s
  /**
   * idtentry_mce_db - Macro to generate entry stubs for #MC and #DB
   * @vector:     Vector number
   * @asmsym:     ASM symbol for the entry point
   * @cfunc:      C function to be called
   *
   * The macro emits code to set up the kernel context for #MC and #DB
   *
   * If the entry comes from user space it uses the normal entry path
   * including the return to user space work and preemption checks on
   * exit.
   *
   * If hits in kernel mode then it needs to go through the paranoid
   * entry as the exception can hit any random state. No preemption
   * check on exit to keep the paranoid path simple.
   */
  .macro idtentry_mce_db vector asmsym cfunc
  SYM_CODE_START(\asmsym)
      UNWIND_HINT_IRET_REGS
      ASM_CLAC

      pushq   $-1         /* ORIG_RAX: no syscall to restart */

      /*
       * If the entry is from userspace, switch stacks and treat it as
       * a normal entry.
       */
      testb   $3, CS-ORIG_RAX(%rsp)
      jnz .Lfrom_usermode_switch_stack_\@

      /* paranoid_entry returns GS information for paranoid_exit in EBX. */
      call    paranoid_entry

      UNWIND_HINT_REGS

      movq    %rsp, %rdi      /* pt_regs pointer */

      call    \cfunc

      jmp paranoid_exit

      /* Switch to the regular task stack and use the noist entry point */
  .Lfrom_usermode_switch_stack_\@:
      idtentry_body noist_\cfunc, has_error_code=0

  _ASM_NOKPROBE(\asmsym)
  SYM_CODE_END(\asmsym)
  .endm
  ```
  * arch/x86/include/asm/idtentry.h
  ```c
  #ifdef CONFIG_X86_64
  # define DECLARE_IDTENTRY_MCE(vector, func)             \
    idtentry_mce_db vector asm_##func func
    ...
  #endif
  ...
  #ifdef CONFIG_X86_MCE
  #ifdef CONFIG_X86_64
  DECLARE_IDTENTRY_MCE(X86_TRAP_MC,   exc_machine_check);
  ...
  #endif
  ```
## MCE 中断处理函数的定义
* arch/x86/include/asm/idtentry.h
  ```c
  #ifndef __ASSEMBLY__
  ...
  /**
   * DEFINE_IDTENTRY_RAW - Emit code for raw IDT entry points
   * @func:   Function name of the entry point
   *
   * @func is called from ASM entry code with interrupts disabled.
   *
   * The macro is written so it acts as function definition. Append the
   * body with a pair of curly brackets.
   *
   * Contrary to DEFINE_IDTENTRY() this does not invoke the
   * idtentry_enter/exit() helpers before and after the body invocation. This
   * needs to be done in the body itself if applicable. Use if extra work
   * is required before the enter/exit() helpers are invoked.
   */
  #define DEFINE_IDTENTRY_RAW(func)                   \
  __visible noinstr void func(struct pt_regs *regs)
  ...
  /**
   * DEFINE_IDTENTRY_IST - Emit code for IST entry points
   * @func:   Function name of the entry point
   *
   * Maps to DEFINE_IDTENTRY_RAW
   */
  #define DEFINE_IDTENTRY_IST(func)                   \
      DEFINE_IDTENTRY_RAW(func)
  ...
  #ifdef CONFIG_X86_64
  ...
  #define DEFINE_IDTENTRY_MCE     DEFINE_IDTENTRY_IST
  ...
  #endif
  ...
  #endif
  ```
* 所以最后看到的中断处理函数`exc_machine_check()`的定义是这样的：
  * arch/x86/kernel/cpu/mce/core.c
  ```c
  ...
  #ifdef CONFIG_X86_64
  /* MCE hit kernel mode */
  DEFINE_IDTENTRY_MCE(exc_machine_check)
  {
      unsigned long dr7;

      dr7 = local_db_save();
      exc_machine_check_kernel(regs);
      local_db_restore(dr7);
  }
  ...
  #endif
  ```
## MCE Severity
* MCE 的严重程度分为以下级别：
* arch/x86/kernel/cpu/mce/internal.h
```c
enum severity_level {
    MCE_NO_SEVERITY,
    MCE_DEFERRED_SEVERITY,
    MCE_UCNA_SEVERITY = MCE_DEFERRED_SEVERITY, // Uncorrected no action required
    MCE_KEEP_SEVERITY,
    MCE_SOME_SEVERITY,
    MCE_AO_SEVERITY,    // Action optional
    MCE_UC_SEVERITY,    // Uncorrected error
    MCE_AR_SEVERITY,    // Action required
    MCE_PANIC_SEVERITY,
};
```
* 对应的，有个`struct severity`类型的数组`severities[]`
* arch/x86/kernel/cpu/mce/severity.c
```c
static struct severity {
    u64 mask;
    u64 result;
    unsigned char sev;
    unsigned char mcgmask;
    unsigned char mcgres;
    unsigned char ser;
    unsigned char context;
    unsigned char excp;
    unsigned char covered;
    unsigned char cpu_model;
    unsigned char cpu_minstepping;
    unsigned char bank_lo, bank_hi;
    char *msg;
} severities[] = {
#define MCESEV(s, m, c...) { .sev = MCE_ ## s ## _SEVERITY, .msg = m, ## c }
#define BANK_RANGE(l, h) .bank_lo = l, .bank_hi = h
#define MODEL_STEPPING(m, s) .cpu_model = m, .cpu_minstepping = s
#define  KERNEL     .context = IN_KERNEL
#define  USER       .context = IN_USER
#define  KERNEL_RECOV   .context = IN_KERNEL_RECOV
#define  SER        .ser = SER_REQUIRED
#define  NOSER      .ser = NO_SER
#define  EXCP       .excp = EXCP_CONTEXT
#define  NOEXCP     .excp = NO_EXCP
#define  BITCLR(x)  .mask = x, .result = 0
#define  BITSET(x)  .mask = x, .result = x
#define  MCGMASK(x, y)  .mcgmask = x, .mcgres = y
#define  MASK(x, y) .mask = x, .result = y
#define MCI_UC_S (MCI_STATUS_UC|MCI_STATUS_S)
#define MCI_UC_AR (MCI_STATUS_UC|MCI_STATUS_AR)
#define MCI_UC_SAR (MCI_STATUS_UC|MCI_STATUS_S|MCI_STATUS_AR)
#define MCI_ADDR (MCI_STATUS_ADDRV|MCI_STATUS_MISCV)

    MCESEV(
        NO, "Invalid",
        BITCLR(MCI_STATUS_VAL)
        ),
    MCESEV(
        NO, "Not enabled",
        EXCP, BITCLR(MCI_STATUS_EN)
        ),
    MCESEV(
        PANIC, "Processor context corrupt",
        BITSET(MCI_STATUS_PCC)
        ),
    /* When MCIP is not set something is very confused */
    MCESEV(
        PANIC, "MCIP not set in MCA handler",
        EXCP, MCGMASK(MCG_STATUS_MCIP, 0)
        ),
    /* Neither return not error IP -- no chance to recover -> PANIC */
    MCESEV(
        PANIC, "Neither restart nor error IP",
        EXCP, MCGMASK(MCG_STATUS_RIPV|MCG_STATUS_EIPV, 0)
        ),
    MCESEV(
        PANIC, "In kernel and no restart IP",
        EXCP, KERNEL, MCGMASK(MCG_STATUS_RIPV, 0)
        ),
    MCESEV(
        PANIC, "In kernel and no restart IP",
        EXCP, KERNEL_RECOV, MCGMASK(MCG_STATUS_RIPV, 0)
        ),
    /*
     * known AO MCACODs reported via MCE or CMC:
     *
     * SRAO could be signaled either via a machine check exception or
     * CMCI with the corresponding bit S 1 or 0. So we don't need to
     * check bit S for SRAO.
     */
    MCESEV(
        AO, "Action optional: memory scrubbing error",
        SER, MASK(MCI_UC_AR|MCACOD_SCRUBMSK, MCI_STATUS_UC|MCACOD_SCRUB)
        ),
    MCESEV(
        AO, "Action optional: last level cache writeback error",
        SER, MASK(MCI_UC_AR|MCACOD, MCI_STATUS_UC|MCACOD_L3WB)
        ),
    /*
     * Quirk for Skylake/Cascade Lake. Patrol scrubber may be configured
     * to report uncorrected errors using CMCI with a special signature.
     * UC=0, MSCOD=0x0010, MCACOD=binary(000X 0000 1100 XXXX) reported
     * in one of the memory controller banks.
     * Set severity to "AO" for same action as normal patrol scrub error.
     */
    MCESEV(
        AO, "Uncorrected Patrol Scrub Error",
        SER, MASK(MCI_STATUS_UC|MCI_ADDR|0xffffeff0, MCI_ADDR|0x001000c0),
        MODEL_STEPPING(INTEL_FAM6_SKYLAKE_X, 4), BANK_RANGE(13, 18)
    ),

    /* ignore OVER for UCNA */
    MCESEV(
        UCNA, "Uncorrected no action required",
        SER, MASK(MCI_UC_SAR, MCI_STATUS_UC)
        ),
    MCESEV(
        PANIC, "Illegal combination (UCNA with AR=1)",
        SER,
        MASK(MCI_STATUS_OVER|MCI_UC_SAR, MCI_STATUS_UC|MCI_STATUS_AR)
        ),
    MCESEV(
        KEEP, "Non signaled machine check",
        SER, BITCLR(MCI_STATUS_S)
        ),

    MCESEV(
        PANIC, "Action required with lost events",
        SER, BITSET(MCI_STATUS_OVER|MCI_UC_SAR)
        ),

    /* known AR MCACODs: */
#ifdef  CONFIG_MEMORY_FAILURE
    MCESEV(
        KEEP, "Action required but unaffected thread is continuable",
        SER, MASK(MCI_STATUS_OVER|MCI_UC_SAR|MCI_ADDR, MCI_UC_SAR|MCI_ADDR),
        MCGMASK(MCG_STATUS_RIPV|MCG_STATUS_EIPV, MCG_STATUS_RIPV)
        ),
    MCESEV(
        AR, "Action required: data load in error recoverable area of kernel",
        SER, MASK(MCI_STATUS_OVER|MCI_UC_SAR|MCI_ADDR|MCACOD, MCI_UC_SAR|MCI_ADDR|MCACOD_DATA),
        KERNEL_RECOV
        ),
    MCESEV(
        AR, "Action required: data load error in a user process",
        SER, MASK(MCI_STATUS_OVER|MCI_UC_SAR|MCI_ADDR|MCACOD, MCI_UC_SAR|MCI_ADDR|MCACOD_DATA),
        USER
        ),
    MCESEV(
        AR, "Action required: instruction fetch error in a user process",
        SER, MASK(MCI_STATUS_OVER|MCI_UC_SAR|MCI_ADDR|MCACOD, MCI_UC_SAR|MCI_ADDR|MCACOD_INSTR),
        USER
        ),
    MCESEV(
        PANIC, "Data load in unrecoverable area of kernel",
        SER, MASK(MCI_STATUS_OVER|MCI_UC_SAR|MCI_ADDR|MCACOD, MCI_UC_SAR|MCI_ADDR|MCACOD_DATA),
        KERNEL
        ),
    MCESEV(
        PANIC, "Instruction fetch error in kernel",
        SER, MASK(MCI_STATUS_OVER|MCI_UC_SAR|MCI_ADDR|MCACOD, MCI_UC_SAR|MCI_ADDR|MCACOD_INSTR),
        KERNEL
        ),
#endif
    MCESEV(
        PANIC, "Action required: unknown MCACOD",
        SER, MASK(MCI_STATUS_OVER|MCI_UC_SAR, MCI_UC_SAR)
        ),

    MCESEV(
        SOME, "Action optional: unknown MCACOD",
        SER, MASK(MCI_STATUS_OVER|MCI_UC_SAR, MCI_UC_S)
        ),
    MCESEV(
        SOME, "Action optional with lost events",
        SER, MASK(MCI_STATUS_OVER|MCI_UC_SAR, MCI_STATUS_OVER|MCI_UC_S)
        ),

    MCESEV(
        PANIC, "Overflowed uncorrected",
        BITSET(MCI_STATUS_OVER|MCI_STATUS_UC)
        ),
    MCESEV(
        UC, "Uncorrected",
        BITSET(MCI_STATUS_UC)
        ),
    MCESEV(
        SOME, "No match",
        BITSET(0)
        )   /* always matches. keep at end */
};
```

* arch/x86/include/asm/mce.h
```c
/* MCG_STATUS register defines */
#define MCG_STATUS_RIPV     BIT_ULL(0)   /* restart ip valid */
#define MCG_STATUS_EIPV     BIT_ULL(1)   /* ip points to correct instruction */
#define MCG_STATUS_MCIP     BIT_ULL(2)   /* machine check in progress */
#define MCG_STATUS_LMCES    BIT_ULL(3)   /* LMCE signaled */

/* MCG_EXT_CTL register defines */
#define MCG_EXT_CTL_LMCE_EN BIT_ULL(0) /* Enable LMCE */

/* MCi_STATUS register defines */
#define MCI_STATUS_VAL      BIT_ULL(63)  /* valid error */
#define MCI_STATUS_OVER     BIT_ULL(62)  /* previous errors lost */
#define MCI_STATUS_UC       BIT_ULL(61)  /* uncorrected error */
#define MCI_STATUS_EN       BIT_ULL(60)  /* error enabled */
#define MCI_STATUS_MISCV    BIT_ULL(59)  /* misc error reg. valid */
#define MCI_STATUS_ADDRV    BIT_ULL(58)  /* addr reg. valid */
#define MCI_STATUS_PCC      BIT_ULL(57)  /* processor context corrupt */
#define MCI_STATUS_S        BIT_ULL(56)  /* Signaled machine check */
#define MCI_STATUS_AR       BIT_ULL(55)  /* Action required */
#define MCI_STATUS_CEC_SHIFT    38           /* Corrected Error Count */
#define MCI_STATUS_CEC_MASK GENMASK_ULL(52,38)
#define MCI_STATUS_CEC(c)   (((c) & MCI_STATUS_CEC_MASK) >> MCI_STATUS_CEC_SHIFT)

/* AMD-specific bits */
#define MCI_STATUS_TCC      BIT_ULL(55)  /* Task context corrupt */
#define MCI_STATUS_SYNDV    BIT_ULL(53)  /* synd reg. valid */
#define MCI_STATUS_DEFERRED BIT_ULL(44)  /* uncorrected error, deferred exception */
#define MCI_STATUS_POISON   BIT_ULL(43)  /* access poisonous data */
#define MCI_STATUS_SCRUB    BIT_ULL(40)  /* Error detected during scrub operation */
...
/*
 * Indicates an MCE which has happened in kernel space but from
 * which the kernel can recover simply by executing fixup_exception()
 * so that an error is returned to the caller of the function that
 * hit the machine check.
 */
#define MCE_IN_KERNEL_RECOV BIT_ULL(6)

/*
 * Indicates an MCE that happened in kernel space while copying data
 * from user. In this case fixup_exception() gets the kernel to the
 * error exit for the copy function. Machine check handler can then
 * treat it like a fault taken in user mode.
 */
#define MCE_IN_KERNEL_COPYIN    BIT_ULL(7)
...
```

## References
- [怎样诊断Machine-check Exception](http://linuxperf.com/?p=105)
- [x86架构—MCA](https://www.codenong.com/cs106476885/)
- [CPU RAS](https://zhuanlan.zhihu.com/p/345444988)
- [Machine-check架构](https://huataihuang.gitbooks.io/cloud-atlas/content/os/linux/kernel/cpu/analysis_cpu_mce.html)
