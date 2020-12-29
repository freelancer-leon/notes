# Kexec

## 几种 Linux 内核镜像文件的区别：
1. `vmlinux`：编译出来的最原始的内核文件，未压缩。
2. `zImage` ：是`vmlinux`经过一系列处理（包括`gzip`压缩）后的文件。
3. `bzImage`：`bz`表示“big zImage”，不是用`bzip2`压缩的。两者的不同之处在于：
   * `zImage`解压缩内核到低端内存（第一个`640K`），
   * `bzImage`解压缩内核到高端内存（`1M`以上）。如果内核比较小，那么采用`zImage`或`bzImage`都行，如果比较大应该用`bzImage`。
4. `uImage` ：U-boot 专用的映像文件，它是在`zImage`之前加上一个长度为`0x40`的 tag（`64`个字节，说明这个映像文件的类型、加载位置、生成时间、大小等信息）。其实就是一个自动跟手动的区别，
   * 有了`uImage`头部的描述，U-boot 就知道对应 Image 的信息，如果没有头部则需要自己手动去搞那些参数。
   * 换句话说，如果直接从`uImage`的`0x40`位置开始执行，`zImage`和`uImage`没有任何区别。
5. `vmlinuz`：是`bzImage`/`zImage`文件的拷贝或指向`bzImage`/`zImage`的链接。
6. `initrd` ：是“initial ramdisk”的简写。一般被用来临时的引导硬件到实际内核`vmlinuz`能够接管并继续引导的状态。
7. `fitImage`：Flattened uImage Tree，根据 image source file(.its) 将 dtb、kernel image、initrd 打包而成的`.itb`镜像文件。
8. `Image|vmlinux.bin`
   * 未配置压缩内核镜像时用类似`objcopy -O binary -R .note -R .comment -R .note.gnu.build-id -S vmlinux`命令生成的镜像文件。
   * 配置压缩内核镜像时，先在`arch/[arch]/boot/compressed`目录下生成`Image|vmlinux.bin`，再压缩成`.gz|.bz2`文件，再加上一些解压信息，比如`piggy.S`，再链接，最终生成`arch/<arch>/boot/`下的`Image|vmlinux.bin`
9. `Image.[gz|bz2]/vmlinux.bin.[gz|bz2]`：可以理解为，仅是对瘦身后的内核镜像再压缩后得到的中间产物，通常还有随后的处理。
   * 配置了`CONFIG_KERNEL_GZIP`或`CONFIG_KERNEL_BZIP2`，会对`arch/<arch>/boot/compressed`目录的`Image/vmlinux.bin`文件进行压缩。

## 相关配置
* Kernel Configurations
  ```sh
  # CONFIG_KEXEC depends on: PM_SLEEP_SMP [=y]
  CONFIG_SUSPEND=y
  CONFIG_PM_SLEEP=y
  CONFIG_PM_SLEEP_SMP=y
  # Enable the kexec syscall
  CONFIG_KEXEC=y
  # Generate crash dump after being started by kexec.
  CONFIG_CRASH_DUMP=y
  # Exports the dump image of crashed kernel in ELF format.
  CONFIG_PROC_VMCORE=y
  CONFIG_DEBUG_INFO=y
  ```
* /etc/sysconfig/kdump
  ```
  KDUMP_COMMANDLINE_APPEND="maxcpus=1 "
  ```
## kexec
* kexec 相关的三个系统调用
  * `kexec_load(unsigned long entry, unsigned long nr_segments, struct kexec_segment *segments, unsigned long flags)`
  * `kexec_load_file()`
  * `reboot(LINUX_REBOOT_CMD_KEXEC)`
* 可以从`/proc/iomem`读到主内核启动时配置`crashkernel`而保留的内存地址范围

### kexec 加载内核
#### 用户态 kexec-tools
```c
main()
-> arch_process_options()
if (do_kexec_file_syscall)
-> do_kexec_file_load()
if (!do_kexec_file_syscall)
-> my_load()
   +-> kernel_buf = slurp_decompress_file()
   |  -> slurp_file()
   |    -> slurp_file_generic()
   |       -> slurp_fd()
   |          -> buf = xmalloc(size) // kernel_buf 会指向这
   +-> get_memory_ranges()
   |   -> get_memory_ranges_iomem()
   |      -> kexec_iomem_for_each_line(NULL, get_memory_ranges_iomem_cb, array)
   |         -> proc_iomem()
   |            return proc_iomem_str; //"/proc/iomem"
   |         -> callback(data, nr, str, start, size)
   |         => get_memory_ranges_iomem_cb() //00000000-01ffffff : System RAM 等多条数据
   |            -> set_phys_offset(r->start)
   +-> file_type[i].probe(kernel_buf, kernel_size)
   |   => image_arm64_probe() // -t Image
   +-> physical_arch() //根据`uname()`的返回结果确定当前系统的 ARCH 的类型
   +-> file_type[i].load(argc, argv, kernel_buf, kernel_size, &info)
   |   => image_arm64_load() // -t Image
   |      -> arm64_process_image_header()
   |         -> arm64_header_check_magic() //检查`kernel_buf`的`magic[4]`是不是`"ARM\x64"`
   |         -> arm64_mem.text_offset = arm64_header_text_offset(h) //
   |         -> arm64_mem.image_size = arm64_header_image_size(h)
   |      -> arm64_locate_kernel_segment() //定位捕捉内核镜像可以放置的 segment 位置
   |         -> locate_hole(info, arm64_mem.text_offset + arm64_mem.image_size, MiB(2), 0, ULONG_MAX, 1)
   |      -> add_segment_phys_virt(info, kernel_buf, kernel_size, kernel_segment + arm64_mem.text_offset, arm64_mem.image_size, 0)
   |      -> arm64_load_other_segments(info, kernel_segment + arm64_mem.text_offset)
   |         -> dtb_base = add_buffer_phys_virt()
   |         -> dbgprintf("dtb: base %lx, size %lxh (%ld)\n", dtb_base, dtb.size, dtb.size);
   |         -> elf_rel_build_load(info, &info->rhdr, purgatory, purgatory_size, hole_min, hole_max, 1, 0);
   |            -> build_elf_rel_info() //根据`purgatory`数组的内容填充`info->rhdr`的 ELF header 信息
   |               -> build_elf_info()
   |                  -> build_mem_ehdr()
   |                  -> build_mem_phdrs()
   |                  -> build_mem_shdrs()
   |                  -> build_mem_notes()
   |               -> machine_verify_elf_rel()
   |            -> elf_rel_load()  //ELF 数据部分的空间分配和填充，产生大量的调试打印
   |         -> info->entry = (void * )elf_rel_get_addr(&info->rhdr, "purgatory_start"); //这个赋值很关键
   |         -> elf_rel_set_symbol(&info->rhdr, "arm64_kernel_entry", &image_base, sizeof(image_base));
   |         -> elf_rel_set_symbol(&info->rhdr, "arm64_dtb_addr", &dtb_base, sizeof(dtb_base));
   +-> arch_compat_trampoline()
   +-> for (i = 0; i < info.nr_segments; i++) //验证是否所有的 segments 都加载到了内存中的有效位置
   |      valid_memory_segment(&info, info.segmen t +i);
   +-> update_purgatory(&info)
   +-> if (entry) info.entry = entry; // --entry 参数可以指定`entry`的位置
   +-> dbgprintf("kexec_load: entry = %p flags = 0x%lx\n", info.entry, info.kexec_flags); //打印重要的调试信息
   +-> kexec_load(info.entry, info.nr_segments, info.segment, info.kexec_flags);
-> my_exec()
   -> reboot(LINUX_REBOOT_CMD_KEXEC)
```
* `slurp_fd()`将指定的文件（也可以是字符设备或块设备）读入到新分配的内存里
* 这个路径上的`get_memory_ranges()`从`/proc/iomem`读取`System RAM`的范围
* `arm64_process_image_header()`从提供的内核镜像文件的 header 信息中得到文本段的偏移和镜像的大小
* `add_segment_phys_virt()`分配存放 kernel image 的`info->segment[]`
* `arm64_load_other_segments()`分配存放 dtb、initrd、purgatory 的`info->segment[]`
  * 通过参数配置的 kernel 命令行参数借助 dtb 的来传递，kexec 带有解析 dtb 的库，可以用来找到 dtb 中命令行参数的位置
  * `add_buffer_phys_virt()`分配存放 dtb 的`info->segment[]`
  * `elf_rel_build_load()`根据`purgatory`数组的内容填充`info->rhdr`的 ELF header 信息，以及 ELF 数据部分的空间分配和填充
  * 将`info->entry`设置为`purgatory_start`的地址，这个值会作为参数传递给`kexec_load()`系统调用，内核态的`arm64_relocate_new_kernel`会先跳到这里，即`purgatory_start`
  * 设定符号`arm64_kernel_entry`的地址为`kernel_segment + arm64_mem.text_offset`
  * 设定符号`arm64_dtb_addr`的地址为上面`add_buffer_phys_virt()`返回的`dtb_base`
  * `purgatory_start`跳转到真正的内核入口地址`arm64_kernel_entry`，第一个参数为设备树的地址`arm64_dtb_addr`
* 最终通过`kexec_load()`系统调用将之前所有收集好的用户态的信息和数据传递给内核

### Purgatory
* 简单说，purgatory 就是一个 bootloader，一个为 kdump 定作的 boot loader
* 在特定体系架构上编译 kexec 时，purgatory 会从相应特定体系的源码生成。它是一个 ELF 格式的 relocatable 文件
* 为了使用上的方便，它被一个工具，`bin-to-hex`，翻译成一个数组并放在`kexec/purgatory.c`里。这样 kexec 的代码就可以直接使用它了
#### ARM64 Purgatory 的生成过程
* `purgatory/purgatory.c`实现了运行在内核态`purgatory()`函数
  * purgatory/purgatory.c
  ```c
  void purgatory(void)
  {
      printf("I'm in purgatory\n");
      setup_arch();
      if (verify_sha256_digest()) {
          for(;;) {
              /* loop forever */
          }
      }
      post_verification_setup_arch();
  }
  ...*```
  ```
* `purgatory/Makefile`将`purgatory/purgatory.c`编译最终生成`purgatory/purgatory.ro`
  * purgatory/Makefile
  ```makefile
  PURGATORY = purgatory/purgatory.ro
  ...
  $(PURGATORY): $(PURGATORY_OBJS)
      $(MKDIR) -p $(@D)
      $(CC) $(CFLAGS) $(LDFLAGS) -o $@.sym $^
  #   $(LD) $(LDFLAGS) $(EXTRA_LDFLAGS) --no-undefined -e purgatory_start -r -o $@ $(PURGATORY_OBJS) $(UTIL_LIB)
      $(STRIP) --strip-debug -o $@ $@.sym
  ```
* `util/Makefile`将`util/bin-to-hex.c`编译成`bin/bin-to-hex`
  * util/Makefile
  ```makefile
  BIN_TO_HEX:= bin/bin-to-hex

  $(BIN_TO_HEX): $(srcdir)/util/bin-to-hex.c
      @$(MKDIR) -p $(@D)
      $(LINK.o) $(CFLAGS) -o $@ $^
  ```
* `bin/bin-to-hex`将`purgatory/purgatory.ro`作为输入，输出到`kexec/purgatory.c`文件，内容是名为`purgatory`的字符数组（数组作为参数指定）和名为`purgatory_size`的全局变量
  * kexec/Makefile
  ```makefile
  PURGATORY_HEX_C = kexec/purgatory.c

  $(PURGATORY_HEX_C): $(PURGATORY) $(BIN_TO_HEX)
      $(MKDIR) -p $(@D)
      $(BIN_TO_HEX) purgatory < $(PURGATORY) > $@
  ...
  ```
  * kexec/purgatory.c
  ```c
  #include <stddef.h>
  const char purgatory[] = {
  0x7f, 0x45, 0x4c, 0x46, 0x02, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
  0x01, 0x00, 0xb7, 0x00, 0x01, 0x00, 0x00, 0x00, 0x80, 0x06, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
  ...
  };
  size_t purgatory_size = sizeof(purgatory);
  ```
* `kexec/purgatory.c`文件会被编译成目标文件并链接进`kexec`可执行程序
* `purgatory`数组和`purgatory_size`的变量在`arm64_load_other_segments()`中作为`elf_rel_build_load()`的参数，构建一个内存中的 ELF 文件
* `purgatory/arch/arm64/entry.S`会被编译成`purgatory/arch/arm64/entry.o`并最终链接到`purgatory/purgatory.ro.sym`
  * purgatory/arch/arm64/entry.S
  ```nasm
  /*
   * ARM64 purgatory.
   */

  .macro  size, sym:req    ;计算符号大小的汇编宏
      .size \sym, . - \sym ;“.”为当前位置，“\sym”替换为符号，即符号的起始位置，它们的差值即为符号大小
  .endm

  .text

  .globl purgatory_start
  purgatory_start:

      adr x19, .Lstack
      mov sp, x19

      bl  purgatory

      /* Start new image. */
      ldr x17, arm64_kernel_entry
      ldr x0, arm64_dtb_addr
      mov x1, xzr
      mov x2, xzr
      mov x3, xzr
      br  x17     ;跳转到 arm64_kernel_entry，第一个参数为 arm64_dtb_addr

  size purgatory_start
  ```
  * 内核态的`arm64_relocate_new_kernel`例程届时会先跳到`purgatory_start`
  * `purgatory_start`先调用`purgatory`
  * 再跳转到真正的内核入口地址`arm64_kernel_entry`，第一个参数为设备树的地址`arm64_dtb_addr`
  * 剩下三个参数寄存器被填充为 0

#### 内核态 kexec

### kexec 切换内核
```c
kernel/reboot.c
SYSCALL_DEFINE4(reboot,...)
  case LINUX_REBOOT_CMD_KEXEC:
  -> kernel_kexec()
     -> machine_kexec(kexec_image)
```
* arch/arm64/include/asm/sysreg.h
  ```c
  /*
   * The "Z" constraint normally means a zero immediate, but when combined with
   * the "%x0" template means XZR.
   */
  #define write_sysreg(v, r) do {                 \
      u64 __val = (u64)(v);                   \
      asm volatile("msr " __stringify(r) ", %x0"      \
               : : "rZ" (__val));             \
  } while (0)
  ...__
  ```

* arch/arm64/include/asm/mmu_context.h
  ```c
  /*
   * Set TTBR0 to empty_zero_page. No translations will be possible via TTBR0.
   */
  static inline void cpu_set_reserved_ttbr0(void)
  {
      unsigned long ttbr = phys_to_ttbr(__pa_symbol(empty_zero_page));

      write_sysreg(ttbr, ttbr0_el1);
      isb(); /*asm volatile("isb" : : : "memory")*/
  }
  ```
  * `isb` 是指令同步屏障（Instruction Synchronize Barrier），该指令冲刷处理器的流水线，重新读取屏障指令后面的所有指令。
  * **TTBR** 是 “Translation Table Base Register” 的缩写，表示转换表基准寄存器。
  * **EL1** 是 ”Exception Level 1” 的缩写，表示异常级别 1。
  * 寄存器`TTBR1_EL1`存放 *内核的页全局目录* 的物理地址。
  * 寄存器`TTBR0_EL1`存放 *进程的页全局目录* 的物理地址。

* arch/arm64/include/asm/mmu_context.h
  ```c
  static inline void cpu_switch_mm(pgd_t *pgd, struct mm_struct *mm)
  {
      BUG_ON(pgd == swapper_pg_dir);
      cpu_set_reserved_ttbr0();
      cpu_do_switch_mm(virt_to_phys(pgd),mm);
  }

  static inline void cpu_install_idmap(void)
  {
      cpu_set_reserved_ttbr0();
      local_flush_tlb_all();
      cpu_set_idmap_tcr_t0sz();

      cpu_switch_mm(lm_alias(idmap_pg_dir), &init_mm);
  }
  ```

* arch/arm64/include/asm/sysreg.h
  ```c
  /* Common SCTLR_ELx flags. */
  #define SCTLR_ELx_EE    (1 << 25)
  #define SCTLR_ELx_IESB  (1 << 21)
  #define SCTLR_ELx_WXN   (1 << 19)
  #define SCTLR_ELx_I (1 << 12)
  #define SCTLR_ELx_SA    (1 << 3)
  #define SCTLR_ELx_C (1 << 2)
  #define SCTLR_ELx_A (1 << 1)
  #define SCTLR_ELx_M 1

  #define SCTLR_ELx_FLAGS (SCTLR_ELx_M  | SCTLR_ELx_A | SCTLR_ELx_C | \
               SCTLR_ELx_SA | SCTLR_ELx_I | SCTLR_ELx_IESB)
  ```
* arch/arm64/kernel/cpu-reset.S
  ```nasm
  .text
  .pushsection    .idmap.text, "awx"

  /*
   * __cpu_soft_restart(el2_switch, entry, arg0, arg1, arg2) - Helper for
   * cpu_soft_restart.
   *
   * @el2_switch: Flag to indicate a swich to EL2 is needed.
   * @entry: Location to jump to for soft reset.
   * arg0: First argument passed to @entry.
   * arg1: Second argument passed to @entry.
   * arg2: Third argument passed to @entry.
   *
   * Put the CPU into the same state as it would be if it had been reset, and
   * branch to what would be the reset vector. It must be executed with the
   * flat identity mapping.
   */
  ENTRY(__cpu_soft_restart)
      /* Clear sctlr_el1 flags. */
      mrs x12, sctlr_el1         ;x12 = sctlr_el1
      ldr x13, =SCTLR_ELx_FLAGS  ;x13 设为系统控制寄存器标志位
      bic x12, x12, x13          ;x12 = x12 & ~x13，清空标志位
      pre_disable_mmu_workaround
      msr sctlr_el1, x12         ;关闭 MMU
      isb

      cbz x0, 1f              // el2_switch?
      mov x0, #HVC_SOFT_RESTART
      hvc #0              // no return

  1:  mov x18, x1             // entry ;reboot_code_buffer_phys 即 arm64_relocate_new_kernel
      mov x0, x2              // arg0  ;kimage->head
      mov x1, x3              // arg1  ;kimage->start
      mov x2, x4              // arg2  ;kimage->arch.dtb_mem
      br  x18
  ENDPROC(__cpu_soft_restart)

  .popsection
  ```
  * 该 routine 放入`.idmap.text`节，因为包含关闭 MMU 的操作
  * **SCTLR_EL1** 为系统控制寄存器
  * 所以关闭 MMU 后，第一段要执行的 routine 是`arm64_relocate_new_kernel`
  * `arm64_relocate_new_kernel`的主要工作是
    * 设置 dtb 的地址为`x0`，即第一个参数
    * 跳转到`kimage->start`，记得在`kexec_alloc_init()`时`kexec_image->start`设为`entry`
  * 最后跳转到`entry`，一直往前追溯，`entry`为`kexec_load()`用户态传入的地址，在 kexec-tools 中被设置为`purgatory_start`的地址

* arch/arm64/kernel/cpu-reset.h
  ```c
  static inline void __noreturn cpu_soft_restart(unsigned long el2_switch,
      unsigned long entry, unsigned long arg0, unsigned long arg1,
      unsigned long arg2)
  {
      typeof(__cpu_soft_restart) *restart;

      el2_switch = el2_switch && !is_kernel_in_hyp_mode() &&
          is_hyp_mode_available();
      restart = (void *)__pa_symbol(__cpu_soft_restart);

      cpu_install_idmap();
      restart(el2_switch, entry, arg0, arg1, arg2);
      unreachable();
  }
  ...__```
  ```

#### 恒等映射（flat identity mapping）
* **恒等映射** 这里与常说的 *一致映射、线性映射、直接映射* 不是一个概念。
* 恒等映射的特点是虚拟地址和物理地址相同，是为了在开始处理器的 MMU 的一瞬间能够平滑过渡。
* 恒等映射是为恒等映射代码节（`.idmap.text`）创建的映射，`idmap_pg_dir`是恒等映射的页全局目录（即第一级页表，pgd）的起始地址（当然是物理地址）。

# References
- [深入探索 Kdump，第 1 部分：带你走进 Kdump 的世界](https://www.ibm.com/developerworks/cn/linux/l-cn-kdump1/index.html)
- [深入探索 Kdump，第 2 部分：Kdump 图形化配置工具](https://www.ibm.com/developerworks/cn/linux/l-cn-kdump2/index.html)
- [深入探索 Kdump，第 3 部分：Kdump 原理探秘](https://www.ibm.com/developerworks/cn/linux/l-cn-kdump3/index.html)
- [深入探索 Kdump，第 4 部分：kdump 的亲密战友 crash](https://www.ibm.com/developerworks/cn/linux/l-cn-kdump4/index.html)
- [kexec - A travel to the purgatory](https://eastrivervillage.com/kexec-tools-with-the-hidden-purgatory/)
- [u-boot FIT image介绍](http://www.wowotech.net/u-boot/fit_image_overview.html)
