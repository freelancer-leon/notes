# Multiple-Processors Management

### 9.4.1 BSP 和 AP 处理器
* MP 初始化协议定义了两类处理器：bootstrap processor (BSP) 和 application processors (AP)
  * 在 MP 系统上电或复位后，系统硬件动态选择系统总线上的处理器之一作为 BSP
  * 其余处理器被指定为 AP
* 作为 BSP 选择机制的一部分，在 BSP 的 `IA32_APIC_BASE` MSR 中设置 `BSP 标志`，表明它是 BSP。为所有其他处理器清除此标志
![IA32_APIC_BASE MSR](pic/ia32_apic_base_msr.PNG)
* BSP 执行 BIOS 的引导程序代码来配置 APIC 环境，设置系统范围的数据结构，并启动和初始化 AP
  * 当 BSP 和 AP 被初始化时，BSP 接着开始执行操作系统初始化代码
* 上电或复位后，AP 完成最小的自配置，然后等待来自 BSP 处理器的启动信号（SIPI 消息）
  * 接收到 SIPI 消息后，AP 将执行 BIOS AP 配置代码，该代码以 AP 置于 halt 状态结束
* 对于支持 Intel 超线程技术的 Intel 64 和 IA-32 处理器，MP 初始化协议将系统总线或 coherent link domain 上的每个逻辑处理器视为一个单独的处理器（具有唯一的 APIC ID）
  * 在启动期间，其中一个逻辑处理器被选为 BSP，其余逻辑处理器被指定为 AP

### 9.4.2 MP 初始化协议的要求和限制
* MP 初始化协议对系统提出了如下要求和限制：
* MP 协议仅在上电或复位后执行。如果 MP 协议已完成并选择了 BSP，则后续的 INIT（针对特定处理器或系统范围）不会导致重复 MP 协议。相反，每个逻辑处理器检查其 `BSP 标志`（在 `IA32_APIC_BASE` MSR 中）以确定
  * 它是否应该执行 BIOS 引导程序代码（如果它是 BSP）
  * 或进入 wait-for-SIPI 状态（如果它是 AP）
* 在 MP 初始化协议期间，必须禁止系统中所有能够向处理器提供中断的设备这样做。必须禁止中断的持续时间包括 *BSP 向 AP 发出 INIT-SIPI-SIPI 序列* 和 *AP 响应序列中的最后一个 SIPI* 之间的窗口

### 9.4.3 MP 系统的 MP 初始化协议算法
* MP 系统上电或复位之后，系统中的处理器执行 MP 初始化协议算法以初始化系统总线或 coherent link domain 上的每个逻辑处理器。在执行该算法的过程中，执行以下启动和初始化操作：
1. 根据系统拓扑，为每个逻辑处理器分配一个唯一的 APIC ID。如果处理器支持 `CPUID` leaf `0x0B`，则唯一 ID 为 `32` 位值，否则唯一 ID 为 `8` 位值。（参见第 9.4.5 节“识别 MP 系统中的逻辑处理器”）。
2. 每个逻辑处理器都根据其 APIC ID 分配一个唯一的仲裁优先级。
3. 每个逻辑处理器与系统中的其他逻辑处理器同时执行其内部 BIST。
4. 完成 BIST 后，逻辑处理器使用硬件定义的选择机制从系统总线上的可用逻辑处理器中选择 BSP 和 AP。BSP 选择机制根据处理器的系列（family）、型号（model）和 stepping ID 的不同而有所不同，如下所示：
   - 系列 `0x0F` 中的后续各代 IA 处理器（请参阅第 9.4 节）、具有系统总线的 IA 处理器（family=`0x06`、external_model=`0`、model>=`0x0E`）或所有其他现代 Intel 处理器（系列=`0x06`、extended_model>`0`）：
     * 逻辑处理器开始监视正在 toggling 的 `BNR#` 信号。当 `BNR#` 引脚停止 toggling 时，每个处理器都会尝试在系统总线上发出一个 `NOP` 的特殊 cycle。
     * 具有最高仲裁优先级的逻辑处理器成功发出一个 `NOP` 特殊 cycle 并被指定为 BSP。该处理器在其 `IA32_APIC_BASE` MSR 中设置 BSP 标志，然后获取并开始执行 BIOS 引导代码，从 reset vector（物理地址 `0xFFFF_FFF0`）开始。
     * 剩余的逻辑处理器（未能发出一个 `NOP` 特殊 cycle 的）被指定为 AP。它们将 BSP 标志保留在清除状态，并进入“wait-for-SIPI 状态”。
   - 系列 `0x0F` 中的早期 IA 处理器（family=`0x0F`，model=`0x0`，stepping<=`0x09`），支持 MP 操作的 P6 系列或更旧的处理器（family=`0x06`，extended_model=`0`，model<=`0x0D`；或 family<`0x06`） ：（略过）
5. 作为引导代码的一部分，BSP 创建 ACPI 表和/或 MP 表，并根据需要将其初始 APIC ID 添加到这些表中。
6. 在引导程序结束时，BSP 将处理器计数器设置为 `1`，然后向系统中的所有 AP 广播 `SIPI` 消息。这里，`SIPI` 消息包含 BIOS AP 初始化代码的向量（位于 `0x000VV000`，其中 `VV` 是 SIPI 消息中包含的向量）。
7. AP 初始化代码的第一个操作是（在 AP 之间）建立 BIOS 初始化 semaphore 的竞争。Semaphore 的第一个 AP 开始执行初始化代码。（有关 semaphore 实现的详细信息，请参见第 9.4.4 节“MP 初始化示例”。）作为 AP 初始化过程的一部分，AP 将其 APIC ID 号添加到适当的 ACPI 和/或 MP 表中，并将处理器计数器增加 `1`。初始化过程完成后，AP 执行 `CLI` 指令并自行停止（halts）。
8. 当每个 AP 获得对 semaphore 的访问并执行 AP 初始化代码时，BSP 建立连接到系统总线的处理器数量的计数，完成执行 BIOS 引导代码，然后开始执行操作系统引导程序（boot-strap）和启动代码。
9. 当 BSP 执行操作系统引导和启动代码时，AP 保持暂停（halted）状态。在此状态下，它们将仅响应 INIT、NMI 和 SMI。它们还将响应 snoops 和 `STPCLK#` 引脚的 assertions 。
* 以下部分给出了在 MP 配置中运行的多个处理器的 MP 初始化协议的示例（有代码）。

### 9.4.4 MP 初始化的例子
* 随附的代码示例中使用了以下常量和数据定义

#### 9.4.4.1 典型的 BSP 初始化序列
* 在选择了 BSP 和 AP 之后（通过硬件协议，参见第 9.4.3 节，“MP 系统的 MP 初始化协议算法”），BSP 开始执行位于常规的 IA-32 架构起始地址（`0xFFFF_FFF0`）的 BIOS boot-strap 代码（POST）。引导程序代码通常执行以下操作：
1. 初始化内存
2. 将微码更新加载到处理器中
3. 初始化 MTRR
4. 启用 caches
5. 执行 `EAX` 寄存器中值为 `0x0` 的 `CPUID` 指令，然后读取 `EBX`、`ECX` 和 `EDX` 寄存器以确定 BSP 是否为 “GenuineIntel”
6. 执行 `EAX` 寄存器中值为 `0x1` 的 `CPUID` 指令，然后将 `EAX`、`ECX` 和 `EDX` 寄存器中的值保存在 RAM 中的系统配置空间中，以备后用
7. 将 AP 的启动代码加载到低 1 MByte 内存中的 4 KByte 页中执行
8. 切换到保护模式并确保 APIC 地址空间映射到 strong uncacheable (UC) 内存类型
9. 从 local APIC ID 寄存器（默认为 `0`）确定 BSP 的 APIC ID，下面的代码片段是一个适用于系统中逻辑处理器的示例，其 local APIC 单元在 xAPIC 模式下运行，APIC 寄存器使用内存映射访问接口：
    ```asm
    MOV ESI, APIC_ID ;Address of local APIC ID register
    MOV EAX, [ESI]
    AND EAX, 0FF000000H ;Zero out all other bits except APIC ID
    MOV BOOT_ID, EAX ;Save in memory
    ```
    * 将 APIC ID 保存在 ACPI 和/或 MP 表中，并可选择保存在 RAM 中的系统配置空间中
10. 将 *AP 启动代码的 4-KByte 页的基地址* 转换为 8-bit 向量。8 位向量定义了实地址模式地址空间（1 MB 空间）中 4 字节页面的地址。例如，向量 `0x0BD` 指定了启动内存地址为 `0x000BD000`
11. 通过设置 APIC spurious vector register（SVR）的第 `8` 位启用 local APIC
    ```asm
    MOV ESI, SVR  ;Address of SVR
    MOV EAX, [ESI]
    OR EAX, APIC_ENABLED ;Set bit 8 to enable (0 on reset)
    MOV [ESI], EAX
    ```
12. 通过为 APIC 错误处理程序建立一个 `8` 位向量来设置 LVT 错误处理入口
    ```asm
    MOV ESI, LVT3
    MOV EAX, [ESI]
    AND EAX, FFFFFF00H ;Clear out previous vector.
    OR EAX, 000000xxH  ;xx is the 8-bit vector the APIC error handler.
    MOV [ESI], EAX;
    ```
13. 将 Lock Semaphore 变量 `VACANT` 初始化为 `0x00`。AP 使用此信号量来确定它们执行 BIOS AP 初始化代码的顺序
14. 执行以下操作以设置 BSP 以检测系统中 AP 的存在和处理器的数量（在有限的持续时间内，最少 100 毫秒）：
    - 将 `COUNT` 变量的值设置为 `1`
    - 在 AP BIOS 初始化代码中，AP 将递增 `COUNT` 变量以指示其存在。等待 `COUNT` 被更新的有限持续时间可以用定时器来完成。当计时器到期时，BSP 检查 `COUNT` 变量的值。如果计时器到期并且 `COUNT` 变量没有增加，则没有 AP 存在或发生了一些错误
15. 向 AP 广播 INIT-SIPI-SIPI IPI 序列以唤醒它们并初始化它们。或者，在加电或重置之后，由于所有 AP 都已处于“wait-for-SIPI 状态”，因此 BSP 可以仅向 AP 广播单个 SIPI IPI 以唤醒它们并初始化它们。如果软件知道它希望唤醒多少个逻辑处理器，它可能会选择轮询 `COUNT` 变量。如果预期的处理器在 100 毫秒计时器到期之前出现，则可以取消计时器并跳到第 16 步
16. 读取并评估 `COUNT` 变量并建立处理器计数
17. 如有必要，重新配置 APIC 并根据需要继续进行剩余的系统诊断

#### 9.4.4.2 典型的 AP 初始化序列
* 当 AP 收到 SIPI 时，它开始在 SIPI 中编码的向量处执行 BIOS AP 初始化代码。AP 初始化代码通常执行以下操作：
1. 等待 BIOS 初始化 Lock Semaphore。当获得对信号量的控制时，初始化继续
2. 将微码更新加载到处理器中
3. 初始化 MTRR（使用与 BSP 相同的映射）
4. 启用 caches
5. 执行 `EAX` 寄存器中值为 `0x0` 的 `CPUID` 指令，然后读取 `EBX`、`ECX` 和 `EDX` 寄存器以确定 AP 是否为“GenuineIntel”
6. 执行 `EAX` 寄存器中值为 `0x1` 的 `CPUID` 指令，然后将 `EAX`、`ECX` 和 `EDX` 寄存器中的值保存在 RAM 中的系统配置空间中，以备后用
7. 切换到保护模式并确保 APIC 地址空间映射到 strong uncacheable (UC) 内存类型
8. 从 local APIC ID 寄存器中确定 AP 的 APIC ID，并将其添加到 MP 和 ACPI 表中，并可选择添加到 RAM 中的系统配置空间中
9. 通过设置 `SVR` 寄存器中的第 8 位并设置用于错误处理的 `LVT3`（错误 LVT）来初始化和配置本地 APIC（如第 9.4.4.1 节“典型 BSP 初始化序列”中的步骤 9 和 10 所述）。
10. 配置 APs 的 SMI 执行环境（每个 AP 和 BSP 必须有不同的 SMBASE 地址）
11. 将 `COUNT` 变量加 `1`
12. 释放信号量
13. 执行以下操作之一：
    * `CLI` 和 `HLT` 指令（如果不支持 `MONITOR/MWAIT`），或
    * 执行 `CLI`、`MONITOR` 和 `MWAIT` 序列，进入深度 C-state
14. 等待 INIT IPI

## Linux 的 MP 唤醒实现
```cpp
#0  do_boot_cpu (apicid=apicid@entry=1, cpu=cpu@entry=1, idle=idle@entry=0xff1100000570a040,
    cpu0_nmi_registered=cpu0_nmi_registered@entry=0xffa0000000013d34) at linux/arch/x86/kernel/smpboot.c:1076
#1  0xffffffff81119bc0 in native_cpu_up (cpu=1, tidle=0xff1100000570a040) at linux/arch/x86/kernel/smpboot.c:1230
#2  0xffffffff811b1a18 in __cpu_up (tidle=0xff1100000570a040, cpu=1) at linux/arch/x86/include/asm/smp.h:83
#3  bringup_cpu (cpu=1) at linux/kernel/cpu.c:608
#4  0xffffffff811b1ea1 in cpuhp_invoke_callback (cpu=cpu@entry=1, state=CPUHP_BRINGUP_CPU, bringup=bringup@entry=true,
    node=node@entry=0x0 <fixed_percpu_data>, lastp=lastp@entry=0x0 <fixed_percpu_data>) at linux/kernel/cpu.c:192
#5  0xffffffff811b20c6 in __cpuhp_invoke_callback_range (bringup=bringup@entry=true, cpu=cpu@entry=1, st=st@entry=0xff1100007fb1b6e0,
    target=target@entry=CPUHP_BRINGUP_CPU, nofail=nofail@entry=false) at linux/kernel/cpu.c:678
#6  0xffffffff811b2edf in cpuhp_invoke_callback_range (target=CPUHP_BRINGUP_CPU, st=0xff1100007fb1b6e0, cpu=1, bringup=true)
    at linux/kernel/cpu.c:702
#7  cpuhp_up_callbacks (target=CPUHP_BRINGUP_CPU, st=0xff1100007fb1b6e0, cpu=1) at linux/kernel/cpu.c:733
#8  _cpu_up (cpu=cpu@entry=1, tasks_frozen=tasks_frozen@entry=0, target=CPUHP_BRINGUP_CPU, target@entry=CPUHP_ONLINE)
    at linux/kernel/cpu.c:1411
#9  0xffffffff811b30e2 in cpu_up (target=CPUHP_ONLINE, cpu=1) at linux/kernel/cpu.c:1447
#10 cpu_up (cpu=<optimized out>, target=CPUHP_ONLINE) at linux/kernel/cpu.c:1419
#11 0xffffffff811b36ff in bringup_nonboot_cpus (setup_max_cpus=8192) at linux/kernel/cpu.c:1513
#12 0xffffffff83eb2c16 in smp_init () at linux/kernel/smp.c:1112
#13 0xffffffff83e78bd2 in kernel_init_freeable () at linux/init/main.c:1629
#14 0xffffffff8209c2d6 in kernel_init (unused=<optimized out>) at linux/init/main.c:1526
#15 0xffffffff81002129 in ret_from_fork () at linux/arch/x86/entry/entry_64.S:308
#16 0x0000000000000000 in ?? ()
```
* arch/x86/kernel/smpboot.c
```cpp
/*
 * NOTE - on most systems this is a PHYSICAL apic ID, but on multiquad
 * (ie clustered apic addressing mode), this is a LOGICAL apic ID.
 * Returns zero if CPU booted OK, else error code from
 * ->wakeup_secondary_cpu.
 */
static int do_boot_cpu(int apicid, int cpu, struct task_struct *idle,
               int *cpu0_nmi_registered)
{
    /* start_ip had better be page-aligned! */
    unsigned long start_ip = real_mode_header->trampoline_start;

    unsigned long boot_error = 0;
    unsigned long timeout;

#ifdef CONFIG_X86_64
    /* If 64-bit wakeup method exists, use the 64-bit mode trampoline IP */
    if (apic->wakeup_secondary_cpu_64)
        start_ip = real_mode_header->trampoline_start64;
#endif
    idle->thread.sp = (unsigned long)task_pt_regs(idle);
    early_gdt_descr.address = (unsigned long)get_cpu_gdt_rw(cpu);
    initial_code = (unsigned long)start_secondary;
    initial_stack  = idle->thread.sp;
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
    return boot_error;
}
```
### INIT-SIPI-SIPI 序列
```cpp
static int
wakeup_secondary_cpu_via_init(int phys_apicid, unsigned long start_eip)
{
    unsigned long send_status = 0, accept_status = 0;
    int maxlvt, num_starts, j;

    maxlvt = lapic_get_maxlvt();

    /*
     * Be paranoid about clearing APIC errors.
     */
    if (APIC_INTEGRATED(boot_cpu_apic_version)) {
        if (maxlvt > 3)     /* Due to the Pentium erratum 3AP.  */
            apic_write(APIC_ESR, 0);
        apic_read(APIC_ESR);
    }

    pr_debug("Asserting INIT\n");

    /*
     * Turn INIT on target chip
     */
    /*
     * Send IPI
     */
    apic_icr_write(APIC_INT_LEVELTRIG | APIC_INT_ASSERT | APIC_DM_INIT,
               phys_apicid);

    pr_debug("Waiting for send to finish...\n");
    send_status = safe_apic_wait_icr_idle();

    udelay(init_udelay);

    pr_debug("Deasserting INIT\n");

    /* Target chip */
    /* Send IPI */
    apic_icr_write(APIC_INT_LEVELTRIG | APIC_DM_INIT, phys_apicid);

    pr_debug("Waiting for send to finish...\n");
    send_status = safe_apic_wait_icr_idle();

    mb();

    /*
     * Should we send STARTUP IPIs ?
     *
     * Determine this based on the APIC version.
     * If we don't have an integrated APIC, don't send the STARTUP IPIs.
     */
    if (APIC_INTEGRATED(boot_cpu_apic_version))
        num_starts = 2;
    else
        num_starts = 0;

    /*
     * Run STARTUP IPI loop.
     */
    pr_debug("#startup loops: %d\n", num_starts);

    for (j = 1; j <= num_starts; j++) {
        pr_debug("Sending STARTUP #%d\n", j);
        if (maxlvt > 3)     /* Due to the Pentium erratum 3AP.  */
            apic_write(APIC_ESR, 0);
        apic_read(APIC_ESR);
        pr_debug("After apic_write\n");

        /*
         * STARTUP IPI
         */

        /* Target chip */
        /* Boot on the stack */
        /* Kick the second */
        apic_icr_write(APIC_DM_STARTUP | (start_eip >> 12),
                   phys_apicid);

        /*
         * Give the other CPU some time to accept the IPI.
         */
        if (init_udelay == 0)
            udelay(10);
        else
            udelay(300);

        pr_debug("Startup point 1\n");

        pr_debug("Waiting for send to finish...\n");
        send_status = safe_apic_wait_icr_idle();

        /*
         * Give the other CPU some time to accept the IPI.
         */
        if (init_udelay == 0)
            udelay(10);
        else
            udelay(200);

        if (maxlvt > 3)     /* Due to the Pentium erratum 3AP.  */
            apic_write(APIC_ESR, 0);
        accept_status = (apic_read(APIC_ESR) & 0xEF);
        if (send_status || accept_status)
            break;
    }
    pr_debug("After Startup\n");

    if (send_status)
        pr_err("APIC never delivered???\n");
    if (accept_status)
        pr_err("APIC delivery error (%lx)\n", accept_status);

    return (send_status | accept_status);
}
...
/*
 * Wake up AP by INIT, INIT, STARTUP sequence.
 *
 * Instead of waiting for STARTUP after INITs, BSP will execute the BIOS
 * boot-strap code which is not a desired behavior for waking up BSP. To
 * void the boot-strap code, wake up CPU0 by NMI instead.
 *
 * This works to wake up soft offlined CPU0 only. If CPU0 is hard offlined
 * (i.e. physically hot removed and then hot added), NMI won't wake it up.
 * We'll change this code in the future to wake up hard offlined CPU0 if
 * real platform and request are available.
 */
static int
wakeup_cpu_via_init_nmi(int cpu, unsigned long start_ip, int apicid,
           int *cpu0_nmi_registered)
{
    int id;
    int boot_error;

    preempt_disable();

    /*
     * Wake up AP by INIT, INIT, STARTUP sequence.
     */
    if (cpu) {
        boot_error = wakeup_secondary_cpu_via_init(apicid, start_ip);
        goto out;
    }

    /*
     * Wake up BSP by nmi.
     *
     * Register a NMI handler to help wake up CPU0.
     */
    boot_error = register_nmi_handler(NMI_LOCAL,
                      wakeup_cpu0_nmi, 0, "wake_cpu0");

    if (!boot_error) {
        enable_start_cpu0 = 1;
        *cpu0_nmi_registered = 1;
        id = apic->dest_mode_logical ? cpu0_logical_apicid : apicid;
        boot_error = wakeup_secondary_cpu_via_nmi(id, start_ip);
    }

out:
    preempt_enable();

    return boot_error;
}
```