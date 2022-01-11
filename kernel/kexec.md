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

## 内核配置
* 系统内核配置
  ```sh
  # Enable the kexec syscall
  CONFIG_KEXEC=y
  # The dump analysis tools require a vmlinux with debug symbols in order to
  # read and analyze a dump file.
  CONFIG_DEBUG_INFO=y

  # Provides a virtual ELF core file of the live kernel.
  CONFIG_PROC_KCORE=y
  # CONFIG_KEXEC depends on: PM_SLEEP_SMP [=y]
  CONFIG_SUSPEND=y
  CONFIG_PM_SLEEP=y
  CONFIG_PM_SLEEP_SMP=y
  ```
* 捕捉内核配置
  ```sh
  # Generate crash dump after being started by kexec.
  CONFIG_CRASH_DUMP=y
  # Exports the dump image of crashed kernel in ELF format.
  CONFIG_PROC_VMCORE=y
  # Device Hardware/Firmware Log Collection
  CONFIG_PROC_VMCORE_DEVICE_DUMP
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
   |         -> arm64_mem.text_offset = arm64_header_text_offset(h)
   |         -> arm64_mem.image_size = arm64_header_image_size(h)
   |      -> arm64_locate_kernel_segment() //定位捕捉内核镜像可以放置的 segment 位置
   |         -> locate_hole(info, arm64_mem.text_offset + arm64_mem.image_size, MiB(2), 0, ULONG_MAX, 1)
   |      -> add_segment_phys_virt(info, kernel_buf, kernel_size, kernel_segment + arm64_mem.text_offset, arm64_mem.image_size, 0)
   |      -> arm64_load_other_segments(info, kernel_segment + arm64_mem.text_offset)
   |         -> setup_2nd_dtb(&dtb, command_line, info->kexec_flags & KEXEC_ON_CRASH)
   |            -> set_bootargs(dtb, command_line) //通过修改 dtb 设置捕捉内核的命令行参数
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
  * 通过 kexec 参数配置的 kernel 命令行参数借助 dtb 的来传递，kexec 带有解析 dtb 的库，可以用来找到 dtb 中命令行参数的位置
  * `add_buffer_phys_virt()`分配存放 dtb 的`info->segment[]`
  * `elf_rel_build_load()`根据`purgatory`数组的内容填充`info->rhdr`的 ELF header 信息，以及 ELF 数据部分的空间分配和填充
  * 将`info->entry`设置为`purgatory_start`的地址，这个值会作为参数传递给`kexec_load()`系统调用，内核态的`arm64_relocate_new_kernel`会先跳到这里，即`purgatory_start`
  * 设定符号`arm64_kernel_entry`的内容为`kernel_segment + arm64_mem.text_offset`
  * 设定符号`arm64_dtb_addr`的内容为上面`add_buffer_phys_virt()`返回的`dtb_base`
  * `purgatory_start`跳转到真正的捕捉内核入口地址`[arm64_kernel_entry]`，第一个参数为设备树的地址`[arm64_dtb_addr]`
* 最终通过`kexec_load()`系统调用将之前所有收集好的用户态的信息和数据传递给内核

#### Purgatory
* 简单说，purgatory 就是一个 bootloader，一个为 kdump 定做的 boot loader
* 在特定体系架构上编译 kexec 时，purgatory 会从相应特定体系的源码生成。它是一个 ELF 格式的 relocatable 文件
* 为了使用上的方便，它被一个工具，`bin-to-hex`，翻译成一个数组并放在`kexec/purgatory.c`里。这样 kexec 的代码就可以直接使用它了
* 最终它要链接进`kexec`可执行程序中，并作为一个 segment 通过`kexec_load()`系统调用传递给内核，内核会把它放在计算好的位置
##### ARM64 Purgatory 的生成过程
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
* `kexec/purgatory.c`文件会被编译成目标文件并链接进`kexec`可执行程序`KEXEC_GENERATED_SRCS += $(PURGATORY_HEX_C)`
* `purgatory`数组和`purgatory_size`的变量在`arm64_load_other_segments()`中作为`elf_rel_build_load()`的参数，构建一个内存中的 ELF 文件
* `purgatory/arch/arm64/entry.S`会被编译成`purgatory/arch/arm64/entry.o`并最终链接到`purgatory/purgatory.ro.sym`
  * 见 purgatory/Makefile 里`PURGATORY_SRCS+=$($(ARCH)_PURGATORY_SRCS)`
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
      ldr x17, arm64_kernel_entry //这里是符号 arm64_kernel_entry 所在的地址
      ldr x0, arm64_dtb_addr //这里是符号 arm64_dtb_addr 所在的地址
      mov x1, xzr
      mov x2, xzr
      mov x3, xzr
      br  x17     ;跳转到地址 arm64_kernel_entry 的内容，第一个参数为地址 arm64_dtb_addr 的内容

  size purgatory_start
  ```
  * 内核态的`arm64_relocate_new_kernel`例程届时会先跳到`purgatory_start`
  * `purgatory_start`先调用`purgatory`
  * 再跳转到真正的捕捉内核入口地址`[arm64_kernel_entry]`，第一个参数为设备树的地址`[arm64_dtb_addr]`（之前通过`elf_rel_set_symbol()`设置）
    * `ldr`指令的格式为：`ldr{条件} 目的寄存器，<存储器地址>`，用于从存储器中将一个 64 位或 32 位（看使用`rn`还是`wn`）的字节数据传送到目的寄存器中
    * 符号`arm64_kernel_entry`的地址的内容已在 kexec-tools load 阶段设置为`kernel_segment + arm64_mem.text_offset`
    * 符号`arm64_dtb_addr`的地址的内容已在 kexec-tools load 阶段设置为存放设备树的地址
  * 剩下三个参数寄存器被填充为 0

#### 内核态 kexec
```c
kernel/kexec.c
SYSCALL_DEFINE4(kexec_load, unsigned long, entry, unsigned long, nr_segments, ...)
-> do_kexec_load(entry, nr_segments, segments, flags)
   -> kimage_alloc_init(&image, entry, nr_segments, segments, flags)
      -> image = do_kimage_alloc_init(); //这里分配和初始化 kimage 结构
         -> image = kzalloc(sizeof(*image), GFP_KERNEL);
            image->head = 0;
            image->entry = &image->head;
      -> image->start = entry; //这个赋值很关键
      -> copy_user_segment_list(image, nr_segments, segments)
      -> sanity_check_segment_list(image)
      -> image->control_code_page = kimage_alloc_control_pages(image, get_order(KEXEC_CONTROL_PAGE_SIZE))
         case KEXEC_TYPE_DEFAULT:
         pages = kimage_alloc_normal_control_pages(image, order); //KEXEC_CONTROL_PAGE_SIZE 4096
         -> kimage_alloc_pages(KEXEC_CONTROL_MEMORY_GFP, order)
            -> alloc_pages(gfp_mask & ~GFP_ZERO, order) // control page 来自这里
   -> machine_kexec_prepare(image)
   -> kimage_crash_copy_vmcoreinfo(image)
   -> for (i = 0; i < nr_segments; i++)
        kimage_load_segment(image, &image->segment[i]);
        -> kimage_load_normal_segment(image, segment)
           -> kimage_set_destination(image, maddr)
              -> kimage_add_entry()
                 -> kimage_alloc_page()
           -> kimage_alloc_page()
           -> kimage_add_page()
              -> kimage_add_entry()
   -> kimage_terminate(image)
   -> image = xchg(dest_image, image); //安装新捕捉内核镜像，卸载旧捕捉内核镜像
```
* `kimage_add_entry()`会设定`image->entry`的值

> Control pages are special, they are the intermediaries
> that are needed while we copy the rest of the pages
> to their final resting place.  As such they must
> not conflict with either the destination addresses
> or memory the kernel is already using.
>
> The only case where we really need more than one of
> these are for architectures where we cannot disable
> the MMU and must instead generate an identity mapped
> page table for all of the memory.
>
> At worst this runs in O(N) of the image size.
>
> -- kimage_alloc_normal_control_pages()

* Control pages 很特别，它是当我们拷贝其余的页面到它们最终要被安放的位置时的中间媒介。因此它们必须不和目的地址冲突，也不能与当前内核正在使用的内存冲突。
* 确实需要多于一页的 control page 的唯一情况是，对于那些无法禁用 MMU 体系结构，必须为所有内存生成一个恒等映射的页表。

### kexec 切换内核
```c
kernel/reboot.c
SYSCALL_DEFINE4(reboot,...)
  case LINUX_REBOOT_CMD_KEXEC:
  -> kernel_kexec()
     -> machine_kexec(kexec_image)
        -> reboot_code_buffer_phys = page_to_phys(kimage->control_code_page)
        -> reboot_code_buffer = phys_to_virt(reboot_code_buffer_phys)
        -> kexec_image_info(kimage)
        -> memcpy(reboot_code_buffer, arm64_relocate_new_kernel, arm64_relocate_new_kernel_size)
        -> pr_info("Bye!\n")
        -> local_daif_mask() //屏蔽所有的异常，包括中断
        -> cpu_soft_restart(kimage != kexec_crash_image, reboot_code_buffer_phys, kimage->head, kimage->start, 0)
           -> restart = (void *)__pa_symbol(__cpu_soft_restart)
           -> cpu_install_idmap() //包含 TTBR0_EL1 切换为 idmap_pg_dir 的关键步骤
           -> restart(el2_switch, entry, arg0, arg1, arg2)
           => __cpu_soft_restart //因为已经切换了恒等映射的页表，所以这里跳转到的是物理地址
              -> reboot_code_buffer_phys //拷贝自 arm64_relocate_new_kernel
                 -> purgatory_start //以下代码来自 kexec
                    -> purgatory
                       -> arm64_kernel_entry //跳转到捕捉内核入口点
```
* `arm64_relocate_new_kernel`例程正是拷贝到了 control page 的起始处
* `cpu_soft_restart()`的`entry`参数为 control page 的物理地址
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
  * **`MSR`指令** `MSR sysreg, Xn`即`sysreg = Xn`
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
  * `Empty_zero_page`是一个特殊的页面，用于初始化为零的数据和 COW
  * 该函数把进程部分的地址空间取消映射

* arch/arm64/include/asm/mmu_context.h
  ```c
  static inline void cpu_switch_mm(pgd_t *pgd, struct mm_struct *mm)
  {
      BUG_ON(pgd == swapper_pg_dir);
      cpu_set_reserved_ttbr0();
      cpu_do_switch_mm(virt_to_phys(pgd),mm);
  }
  ...
  /*
   * Set TCR.T0SZ to its default value (based on VA_BITS)
   */
  static inline void __cpu_set_tcr_t0sz(unsigned long t0sz)
  {
      unsigned long tcr;

      if (!__cpu_uses_extended_idmap())
          return;

      tcr = read_sysreg(tcr_el1); /*读取 TCR_EL1 寄存器的值到 tcr*/
      tcr &= ~TCR_T0SZ_MASK; /*清零 T0SZ 位(5-0 bit)的值，*/
      tcr |= t0sz << TCR_T0SZ_OFFSET;
      write_sysreg(tcr, tcr_el1); /*tcr写回到 TCR_EL1 寄存器*/
      isb();
  }

  #define cpu_set_default_tcr_t0sz()  __cpu_set_tcr_t0sz(TCR_T0SZ(VA_BITS))
  #define cpu_set_idmap_tcr_t0sz()    __cpu_set_tcr_t0sz(idmap_t0sz)
  ...
  static inline void cpu_install_idmap(void)
  {
      cpu_set_reserved_ttbr0(); /*把进程部分的地址空间取消映射*/
      local_flush_tlb_all();
      cpu_set_idmap_tcr_t0sz();

      cpu_switch_mm(lm_alias(idmap_pg_dir), &init_mm);
  }
  ```
* 对于`idmap_t0sz`相关定义如下：
  ```c
  u64 idmap_t0sz = TCR_T0SZ(VA_BITS);`
  #define VA_BITS (CONFIG_ARM64_VA_BITS)
  CONFIG_ARM64_VA_BITS=39
  #define TCR_T0SZ(x) ((UL(64) - (x)) << TCR_T0SZ_OFFSET)
  ```
  *  所以`idmap_t0sz`为`25`，意思是说，当 TTBR 为`TTBR0_EL1`时，寻址范围为 2<sup>(64-25)</sup>=2<sup>39</sup>，即`0～0x0000_FFFF_FFFF_FFFF`，于是`__cpu_set_tcr_t0sz()`恢复`TCR.T0SZ`为缺省值

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

  1:  mov x18, x1             // entry ;reboot_code_buffer_phys 内容为 arm64_relocate_new_kernel
      mov x0, x2              // arg0  ;kimage->head
      mov x1, x3              // arg1  ;kimage->start
      mov x2, x4              // arg2  ;kimage->arch.dtb_mem
      br  x18
  ENDPROC(__cpu_soft_restart)

  .popsection
  ```
  * 该 routine 放入`.idmap.text`节，因为包含关闭 MMU 的操作，这意味对`__cpu_soft_restart`代码的映射需要存在两份：
    * 一份由`init_mm`的`pgd`页表映射到常规的内核代码段所在的物理地址，比如`VA: 0xffffff80088842f0 -> PA: 0x8842f0`（ARM64）
    * 另一份由`idmap_pg_dir`恒等映射到与虚拟地址一样的物理地址，比如`VA: 0x8842f0 -> PA: 0x8842f0`，但是用类似 crash 或者 gdb 查看时是无法直接查看`VA: 0x8842f0`的内容的。因为 MMU 正开，所以读到的始终是虚拟地址；但页表寄存器又不是恒等映射的页表，所以无法通过调试工具查看以上转换。但我们可以借助查看虚拟地址`0xffffffc0008842f0`来了解物理地址的内容，而`0xffffffc000000000`是一致映射的偏移（ARM64）
    * 在此之前调用的`cpu_install_idmap()->cpu_switch_mm(lm_alias(idmap_pg_dir), &init_mm)`切换了页表
  * **SCTLR_EL1** 为系统控制寄存器
  * 因为传递的`entry = reboot_code_buffer_phys`，这是物理地址，所以对新拷贝的`arm64_relocate_new_kernel`代码内容并不需要恒等映射
  * 关闭 MMU 后，第一段要执行的 routine 是`arm64_relocate_new_kernel`，但不是在原来的代码段上，而是数据段`reboot_code_buffer`的新`arm64_relocate_new_kernel`
  * `arm64_relocate_new_kernel`的主要工作是
    * 跳转到`kimage->start`，回忆前面在`kexec_alloc_init()`时`kexec_image->start`设为`entry`
    * 在 commit`4c9e7e649a3f291e1b939299458e6844c16afe70`**arm64: kexec_file: invoke the kernel without purgatory** 之后，ARM64 不再借助 purgatory 跳转到 capture kernel，而是直接跳转到新内核入口点，因此此处需设置 dtb 的地址为`x0`，即第一个参数
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
  * 注意，这里用了`__pa_symbol(__cpu_soft_restart)`，所以`restart`设置的是`__cpu_soft_restart()`的物理地址
  * `cpu_install_idmap()`将当前的页表寄存器切换成恒等映射的页全局目录`idmap_pg_dir`，恒等映射的细节见下面的解释
  * 所以调`restart`的时候，实际上是跳转到一个低内存地址段的一个恒等映射的虚拟地址（由于此时 MMU 还没关），比如，`0x8842f0`（`swapper_pg_dir`也将虚拟地址`0xffffff80088842f0`映射到这）
  * `__cpu_soft_restart`例程里会关闭 MMU，之后访问的就是物理地址了

#### 恒等映射（flat identity mapping）
* **恒等映射** 这里与常说的 *一致映射、线性映射、直接映射* 不是一个概念。
* 恒等映射的特点是虚拟地址和物理地址相同，是为了在开始处理器的 MMU 的一瞬间能够平滑过渡。
* 恒等映射是为恒等映射代码节（`.idmap.text`）创建的映射，`idmap_pg_dir`是恒等映射的页全局目录（即第一级页表，pgd）的起始地址（当然是物理地址）。
* 比如说，将恒等映射代码节`idmap.text`的所在的物理地址`0x800000`映射到虚拟地址`0xffffff8008800000`
  * 先安装恒等映射的页全局目录`idmap_pg_dir`到寄存器`TTBR0_EL1`，替换原有的用户态进程的页表映射
  * 关闭 MMU 前，假设跳转到虚拟地址`0x800004`，该条指令会关闭 MMU，`$pc`为`0x800004`
  * 接着执行下一条指令，位于地址`0x800008`，由于 MMU 已在上一条指令关闭，此时地址是物理地址，这样程序的执行就平滑过渡了

## arm64_image_header
* arch/arm64/Makefile
```makefile
...
# The byte offset of the kernel image in RAM from the start of RAM.
ifeq ($(CONFIG_ARM64_RANDOMIZE_TEXT_OFFSET), y)
TEXT_OFFSET := $(shell awk "BEGIN {srand(); printf \"0x%06x\n\", \
         int(2 * 1024 * 1024 / (2 ^ $(CONFIG_ARM64_PAGE_SHIFT)) * \
         rand()) * (2 ^ $(CONFIG_ARM64_PAGE_SHIFT))}")
else
TEXT_OFFSET := 0x00080000
endif
...
```
* `TEXT_OFFSET`来自这里，通常是`0x00080000`

* include/linux/init.h
```c
/* For assembly routines */
#define __HEAD      .section    ".head.text","ax"
#define __INIT      .section    ".init.text","ax"
```

* arch/arm64/kernel/image.h
```c
#ifdef CONFIG_CPU_BIG_ENDIAN
#define __HEAD_FLAG_BE      1
#else
#define __HEAD_FLAG_BE      0
#endif

#define __HEAD_FLAG_PAGE_SIZE   ((PAGE_SHIFT - 10) / 2)

#define __HEAD_FLAG_PHYS_BASE   1

#define __HEAD_FLAGS        ((__HEAD_FLAG_BE << 0) |    \
                 (__HEAD_FLAG_PAGE_SIZE << 1) | \
                 (__HEAD_FLAG_PHYS_BASE << 3))

/*
 * These will output as part of the Image header, which should be little-endian
 * regardless of the endianness of the kernel. While constant values could be
 * endian swapped in head.S, all are done here for consistency.
 */
#define HEAD_SYMBOLS                        \
    DEFINE_IMAGE_LE64(_kernel_size_le, _end - _text);   \
    DEFINE_IMAGE_LE64(_kernel_offset_le, TEXT_OFFSET);  \
    DEFINE_IMAGE_LE64(_kernel_flags_le, __HEAD_FLAGS);
...__```
```
* `_kernel_size_le`是计算出来的
* `_kernel_offset_le`来自`TEXT_OFFSET`的定义
* `_kernel_flags_le`与 kernel config 关联

* arch/arm64/kernel/head.S
  ```nasm
  ...
  #include "efi-header.S"

  #define __PHYS_OFFSET   (KERNEL_START - TEXT_OFFSET)
  ...
  /*
   * Kernel startup entry point.
   * ---------------------------
   *
   * The requirements are:
   *   MMU = off, D-cache = off, I-cache = on or off,
   *   x0 = physical address to the FDT blob.
   *
   * This code is mostly position independent so you call this at
   * __pa(PAGE_OFFSET + TEXT_OFFSET).
   *
   * Note that the callee-saved registers are used for storing variables
   * that are useful before the MMU is enabled. The allocations are described
   * in the entry routines.
   */
      __HEAD
  _head:
      /*
       * DO NOT MODIFY. Image header expected by Linux boot-loaders.
       */
  #ifdef CONFIG_EFI
      /*  
       * This add instruction has no meaningful effect except that
       * its opcode forms the magic "MZ" signature required by UEFI.
       */
      add x13, x18, #0x16 //没有实际意义的一条指令，它的 opcode 恰好为 UEFI 的 magic “MZ” 签名
      b   stext           //跳转到内核文本段
  #else
      b   stext               // branch to kernel start, magic
      .long   0               // reserved
  #endif
      le64sym _kernel_offset_le       // Image load offset from start of RAM, little-endian
      le64sym _kernel_size_le         // Effective size of kernel image, little-endian
      le64sym _kernel_flags_le        // Informative flags, little-endian
      .quad   0               // reserved
      .quad   0               // reserved
      .quad   0               // reserved
      .ascii  "ARM\x64"           // Magic number
  #ifdef CONFIG_EFI
      .long   pe_header - _head       // Offset to the PE header.

  pe_header:
      __EFI_PE_HEADER //内容来自 arch/arm64/kernel/efi-header.S
  #else
      .long   0               // reserved
  #endif

      __INIT
      /*
       * The following callee saved general purpose registers are used on the
       * primary lowlevel boot path:
       *
       *  Register   Scope                      Purpose
       *  x21        stext() .. start_kernel()  FDT pointer passed at boot in x0
       *  x23        stext() .. start_kernel()  physical misalignment/KASLR offset
       *  x28        __create_page_tables()     callee preserved temp register
       *  x19/x20    __primary_switch()         callee preserved temp registers
       */
  ENTRY(stext)
      bl  preserve_boot_args
      bl  el2_setup           // Drop to EL1, w0=cpu_boot_mode
      adrp    x23, __PHYS_OFFSET
      and x23, x23, MIN_KIMG_ALIGN - 1    // KASLR offset, defaults to 0
      bl  set_cpu_boot_mode_flag
      bl  __create_page_tables
      /*
       * The following calls CPU setup code, see arch/arm64/mm/proc.S for
       * details.
       * On return, the CPU will be ready for the MMU to be turned on and
       * the TCR will have been set.
       */
      bl  __cpu_setup         // initialise processor
      b   __primary_switch
  ENDPROC(stext)
  ```
* `adrp`指令可以将符号地址变成运行时地址（通过 PC relative offset 形式），因此，当运行在 MMU OFF mode 下，通过`adrp`指令可以获取符号的物理地址。
* 不过`adrp`是 page 对齐的（`adrp`中的`p`就是 page 的意思），当符号会是 page size 对齐的时，不能直接使用`adrp`，而是使用`adr_l`这个宏进行处理。
* kexec/arch/arm64/image-header.h
  ```c
  /**
   * struct arm64_image_header - arm64 kernel image header.
   *
   * @pe_sig: Optional PE format 'MZ' signature.
   * @branch_code: Reserved for instructions to branch to stext.
   * @text_offset: The image load offset in LSB byte order.
   * @image_size: An estimated size of the memory image size in LSB byte order.
   * @flags: Bit flags in LSB byte order:
   *   Bit 0:   Image byte order: 1=MSB.
   *   Bit 1-2: Kernel page size: 1=4K, 2=16K, 3=64K.
   *   Bit 3:   Image placement: 0=low.
   * @reserved_1: Reserved.
   * @magic: Magic number, "ARM\x64".
   * @pe_header: Optional offset to a PE format header.
   **/

  struct arm64_image_header {
      uint8_t pe_sig[2];
      uint16_t branch_code[3];
      uint64_t text_offset;
      uint64_t image_size;
      uint64_t flags;
      uint64_t reserved_1[3];
      uint8_t magic[4];
      uint32_t pe_header;
  };

  static const uint8_t arm64_image_magic[4] = {'A', 'R', 'M', 0x64U};
  static const uint8_t arm64_image_pe_sig[2] = {'M', 'Z'};
  static const uint64_t arm64_image_flag_be = (1UL << 0);
  static const uint64_t arm64_image_flag_page_size = (3UL << 1);
  static const uint64_t arm64_image_flag_placement = (1UL << 3);
  ```
## 运行时的例子

### 例1：观察恒等映射
#### 查看虚拟地址的内容
* 观察处于恒等映射节的例程`__cpu_soft_restart`
```c
crash> p __cpu_soft_restart
__cpu_soft_restart = $1 =
 {<text variable, no debug info>} 0xffffff80088842f0 <__cpu_soft_restart>
```
* 此例中，例程`__cpu_soft_restart`有虚拟地址`0xffffff80088842f0`，熟悉 AArch64 的知道这是内核代码段的一个地址
```c
crash> vtop 0xffffff80088842f0
VIRTUAL           PHYSICAL
ffffff80088842f0  8842f0

PAGE DIRECTORY: ffffff8008d61000
   PGD: ffffff8008d61000 => 3ffffe803
   PMD: ffffffc3ffffe220 => 3ffffc803
   PTE: ffffffc3ffffc420 => d0000000884f93
  PAGE: 884000

     PTE        PHYSICAL  FLAGS
d0000000884f93   884000   (VALID|RDONLY|SHARED|AF|NG|UXN|DIRTY)

      PAGE               PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffffbf00022100           884000                0        0  1 800 reserved
```
* 根据寄存器`TTBR1_EL1`的指引，查看其物理地址为`0x8842f0`
* 看看其内容与上面的源码是否一致
```nasm
crash> dis 0xffffff80088842f0
0xffffff80088842f0 <__cpu_soft_restart>:        mrs     x12, sctlr_el1
0xffffff80088842f4 <__cpu_soft_restart+4>:      ldr     x13, 0xffffff8008884328
0xffffff80088842f8 <__cpu_soft_restart+8>:      bic     x12, x12, x13
0xffffff80088842fc <__cpu_soft_restart+12>:     isb
0xffffff8008884300 <__cpu_soft_restart+16>:     msr     sctlr_el1, x12
0xffffff8008884304 <__cpu_soft_restart+20>:     isb
0xffffff8008884308 <__cpu_soft_restart+24>:     cbz     x0, 0xffffff8008884314 <__cpu_soft_restart+36>
0xffffff800888430c <__cpu_soft_restart+28>:     mov     x0, #0x1                        // #1
0xffffff8008884310 <__cpu_soft_restart+32>:     hvc     #0x0
0xffffff8008884314 <__cpu_soft_restart+36>:     mov     x18, x1
0xffffff8008884318 <__cpu_soft_restart+40>:     mov     x0, x2
0xffffff800888431c <__cpu_soft_restart+44>:     mov     x1, x3
0xffffff8008884320 <__cpu_soft_restart+48>:     mov     x2, x4
0xffffff8008884324 <__cpu_soft_restart+52>:     br      x18
```
* dump 该内存区的内容用于后面的比较
```
crash> rd -64 0xffffff80088842f0 32
ffffff80088842f0:  580001add538100c d5033fdf8a2d018c   ..8....X..-..?..
ffffff8008884300:  d5033fdfd518100c d2800020b4000060   .....?..`... ...
ffffff8008884310:  aa0103f2d4000002 aa0303e1aa0203e0   ................
ffffff8008884320:  d61f0240aa0403e2 000000000020100f   ....@..... .....
ffffff8008884330:  a9411404a9400c02 a9432809a9422006   ..@...A.. B..(C.
ffffff8008884340:  a945380da944300b d51bd063d51bd042   .0D..8E.B...c...
ffffff8008884350:  d5181046d518d024 b34014e8d5382047   $...F...G 8...@.
ffffff8008884360:  d518c009d5182048 d510024ad5034fdf   H .......O..J...
ffffff8008884370:  d518d08dd518100c d5101385d518410e   .........A......
ffffff8008884380:  d510108bd341056b 93482c00d5380500   k.A.......8..,H.
ffffff8008884390:  5400004bf100041f d503201fd51b9e1f   ....K..T..... ..
ffffff80088843a0:  d65f03c0d5033fdf d5034fdfd53b4222   .?...._."B;..O..
ffffff80088843b0:  aa0103e3f0002421 d5033fdfd5182023   !$......# ...?..
ffffff80088843c0:  d503379fd508871f aa0003e3d5033fdf   .....7...?......
ffffff80088843d0:  d5033fdfd5182023 d65f03c0d51b4222   # ...?.."B...._.
ffffff80088843e0:  d538202300000000 350008c010ffffc4   ....# 8........5
```
#### dump 内核页表
* 先看看页表映射
```sh
$ cat sys/kernel/debug/kernel_page_tables
...
---[ Linear Mapping ]---
0xffffffc000000000-0xffffffc000080000         512K PTE       RW NX SHD AF NG CON     UXN MEM/NORMAL
0xffffffc000080000-0xffffffc000200000        1536K PTE       ro NX SHD AF NG         UXN MEM/NORMAL
0xffffffc000200000-0xffffffc000a00000           8M PMD       ro NX SHD AF NG     BLK UXN MEM/NORMAL
0xffffffc000a00000-0xffffffc000ad0000         832K PTE       ro NX SHD AF NG         UXN MEM/NORMAL
0xffffffc000ad0000-0xffffffc000c00000        1216K PTE       RW NX SHD AF NG CON     UXN MEM/NORMAL
0xffffffc000c00000-0xffffffc002000000          20M PMD       RW NX SHD AF NG     BLK UXN MEM/NORMAL
0xffffffc003000000-0xffffffc004000000          16M PMD       RW NX SHD AF NG     BLK UXN MEM/NORMAL
0xffffffc004000000-0xffffffc030000000         704M PMD       RW NX SHD AF NG CON BLK UXN MEM/NORMAL
0xffffffc030020000-0xffffffc030200000        1920K PTE       RW NX SHD AF NG CON     UXN MEM/NORMAL
0xffffffc030200000-0xffffffc032000000          30M PMD       RW NX SHD AF NG     BLK UXN MEM/NORMAL
0xffffffc032000000-0xffffffc040000000         224M PMD       RW NX SHD AF NG CON BLK UXN MEM/NORMAL
0xffffffc040000000-0xffffffc400000000          15G PGD       RW NX SHD AF NG     BLK UXN MEM/NORMAL
---[ Linear Mapping ]---
0x0000000000800000-0x0000000000a00000           2M PMD       RW x  SHD AF        BLK MEM/NORMAL
```
* 最下面两行是我经过修改定制的，打印`idmap_pg_dir`的页表映射
  * 可见它只映射了从虚拟地址`0x800000`开始的 2M 的内容
  * 我们要查看的物理地址`0x8842f0`也在映射范围内

#### 观察恒等映射的地址转换
* 对于虚拟地址`0x8842f0`

PGD | PMD | PTE | Page offset
----|-----|-----|-----
9 | 9 | 9 | 12
000 0000 00 | 00 0000 100 | 0 1000 0010 | 0010 1111 0000

* 我们无法让`idmap_pg_dir`加载到`TTBR0_EL1`，但我们可以通过查看其内容追溯这一过程
* 先看看`idmap_pg_dir`的页表项
```c
crash> p -x idmap_pg_dir
idmap_pg_dir = $1 =
 {{
    pgd = 0xd5e003
  }, {
    pgd = 0x0
  }, {
    pgd = 0x0
  }, {
    ...
 }}
```
* 虚拟地址`0x8842f0` PGD 索引为`000 0000 00`，对应的 PGD 级的第一个页表项指向物理地址`0xd5e003`
* 查看 PMD 级的页表项内容，同样先得到它的虚拟地址
```c
crash> ptov 0xd5e003
VIRTUAL           PHYSICAL
ffffffc000d5e003  d5e003
crash>
```
* 再通过 dump 虚拟地址`0xffffffc000d5e003`的内容来查看，由于低 12 位是标志位，要去掉，所以查看的是`0xffffffc000d5e000`
```
crash> rd -64 0xffffffc000d5e000 512
ffffffc000d5e000:  0000000000000000 0000000000000000   ................
ffffffc000d5e010:  0000000000000000 0000000000000000   ................
ffffffc000d5e020:  0000000000800711 0000000000000000   ................
ffffffc000d5e030:  0000000000000000 0000000000000000   ................
ffffffc000d5e040:  0000000000000000 0000000000000000   ................
ffffffc000d5e050:  0000000000000000 0000000000000000   ................
...
crash> ptov 0000000000800000
VIRTUAL           PHYSICAL
ffffffc000800000  800000
```
* 虚拟地址`0x8842f0` PMD 索引为`00 0000 100`，故 PMD 级第四项是我们要找的，`0x711`是标志位，所以二级页表的物理地址是`0x800000`
* 后面的 21 个 bit 构成偏移，基址是物理地址`0x800000`
* 由此可见，`idmap_pg_dir`加载到`TTBR0_EL1`后，对虚拟地址`0x800000`会转换为对物理地址`0x800000`的访问
##### 为什么这里只到映射到 PMD 这一级？
* 对于采用 39 位有效虚拟地址的内核页表有三级`pgd(pud)-->pmd-->pte`
* 对于 kernel image 这样的 big block memory region，使用 4K 的 page 来 mapping 有点得不偿失，在这种情况下，可以考虑让 PMD 的 Translation table entry 指向一个 2M 的 memory region，而不是下一级的 Translation table
* 所谓的 section map 就是指使用 2M 的为单位进行映射。此时，PMD 的内容不是下一级的 table descriptor，而是基于 2M block 的 mapping（或者说 PMD 中 的描述符是 block descriptor），PTE 这一级映射就不需要了

#### 查看物理地址的内容
* 由于 MMU 已开启，我们无法绕过 MMU 直接查看物理地址的内容。好在`0x8842f0`这一物理地址处于线性映射的范围
* 我们可以找到`0x8842f0`线性映射的虚拟地址`0xffffffc0008842f0`
  * 在这个例子里`memstart_addr`恰好是 0，所以`PHYS_OFFSET`也为 0
```c
crash> ptov 8842f0
VIRTUAL           PHYSICL
ffffffc0008842f0  8842f0
```
* 从而得到它物理地址的内容
```
crash> rd -64 ffffffc0008842f0 32
ffffffc0008842f0:  580001add538100c d5033fdf8a2d018c   ..8....X..-..?..
ffffffc000884300:  d5033fdfd518100c d2800020b4000060   .....?..`... ...
ffffffc000884310:  aa0103f2d4000002 aa0303e1aa0203e0   ................
ffffffc000884320:  d61f0240aa0403e2 000000000020100f   ....@..... .....
ffffffc000884330:  a9411404a9400c02 a9432809a9422006   ..@...A.. B..(C.
ffffffc000884340:  a945380da944300b d51bd063d51bd042   .0D..8E.B...c...
ffffffc000884350:  d5181046d518d024 b34014e8d5382047   $...F...G 8...@.
ffffffc000884360:  d518c009d5182048 d510024ad5034fdf   H .......O..J...
ffffffc000884370:  d518d08dd518100c d5101385d518410e   .........A......
ffffffc000884380:  d510108bd341056b 93482c00d5380500   k.A.......8..,H.
ffffffc000884390:  5400004bf100041f d503201fd51b9e1f   ....K..T..... ..
ffffffc0008843a0:  d65f03c0d5033fdf d5034fdfd53b4222   .?...._."B;..O..
ffffffc0008843b0:  aa0103e3f0002421 d5033fdfd5182023   !$......# ...?..
ffffffc0008843c0:  d503379fd508871f aa0003e3d5033fdf   .....7...?......
ffffffc0008843d0:  d5033fdfd5182023 d65f03c0d51b4222   # ...?.."B...._.
ffffffc0008843e0:  d538202300000000 350008c010ffffc4   ....# 8........5
```
* 显然，内容和上面 dump 的结果一致


### 例2：观察跳转到内核镜像 Image
#### kexec 加载内核
```sh
# kexec -d -c -t Image -l /root/Image --reuse-cmdline
arch_process_options:154: command_line: root=/dev/vda rw rootfstype=ext4 earlycon earlyprintk crashkernel=384M@0
arch_process_options:156: initrd: (null)
arch_process_options:157: dtb: (null)
arch_process_options:159: console: (null)
Try gzip decompression.
Try LZMA decompression.
lzma_decompress_file: read on /root/Image of 65536 bytes failed
kernel: 0x7faa968010 kernel_size: 0xc89200
get_memory_ranges_iomem_cb: 0000000040000000 - 000000023fffffff : System RAM
image_arm64_load: kernel_segment: 0000000040000000
image_arm64_load: text_offset:    0000000000080000
image_arm64_load: image_size:     0000000000ce4000
image_arm64_load: phys_offset:    0000000040000000
image_arm64_load: vp_offset:      ffffffffffffffff
image_arm64_load: PE format:      yes
arm64_load_other_segments:613: purgatory sink: 0x0
read_1st_dtb: found /sys/firmware/fdt
dtb:    base 40d64000, size 1dd6h (7638)
sym: sha256_starts info: 12 other: 00 shndx: 2 value: f60 size: 50
sym: sha256_starts value: 40d66f90 addr: 40d66044
machine_apply_elf_rel: CALL26 9000000094000000->90000000940003d3
sym:      .text info: 03 other: 00 shndx: 2 value: 0 size: 0
sym: .text value: 40d66128 addr: 40d66048
...
kexec_load: entry = 0x40d666b0 flags = 0xb70000 #计算出来的 purgatory_start 的物理地址
nr_segments = 3
segment[0].buf   = 0x7faa968010
segment[0].bufsz = 0xc89200
segment[0].mem   = 0x40080000  #存放内核镜像的段
segment[0].memsz = 0xce4000
segment[1].buf   = 0x55a0304de0
segment[1].bufsz = 0x1dd6
segment[1].mem   = 0x40d64000 #存放 dtb 的段，0x40d64000 = 0x40080000 + 0xce4000
segment[1].memsz = 0x2000
segment[2].buf   = 0x55a0307100
segment[2].bufsz = 0x3570
segment[2].mem   = 0x40d66000 #存放 purgatory 的段，0x40d66000 = 0x40d64000 + 0x2000
segment[2].memsz = 0x4000
# 以下为内核打印的信息
[   93.892994] do_kexec_load:173: kexec_image:              ffffffc1f4b6ec00
[   93.893647] do_kexec_load:175: control_code_page:        ffffffbf07ce4f40
[   93.893992] do_kexec_load:177: reboot_code_buffer_phys:  0x000000023393d000
[   93.894344] do_kexec_load:179: reboot_code_buffer:       ffffffc1f393d000
[   93.894668] do_kexec_load:181: relocate_new_kernel:      ffffff800809dc18
```
* `__cpu_soft_restart`会跳转到`0x000000023393d000`，`arm64_relocate_new_kernel`例程的拷贝

#### 跳转到内核镜像
* 看看运行时怎么跳到内核镜像`Image`的这一过程
* 加载`idmap_pg_dir`到`TTBR0_EL1`后就可以跳转到恒等映射的`__cpu_soft_restart`例程了，往后一直运行
* 现在已经运行到了`purgatory_start`的最后阶段，`purgatory`例程已经执行过了
  ```c
  (gdb) x /32i 0x0000000040d666b0
     0x40d666b0:  adr     x19, 0x40d66ee0
     0x40d666b4:  mov     sp, x19
     0x40d666b8:  bl      0x40d66158      //跳转到 purgatory 例程
     0x40d666bc:  ldr     x17, 0x40d69518 //这里是符号 arm64_kernel_entry 所在的地址
     0x40d666c0:  ldr     x0, 0x40d69520  //这里是符号 arm64_dtb_addr 所在的地址
     0x40d666c4:  mov     x1, xzr
     0x40d666c8:  mov     x2, xzr
     0x40d666cc:  mov     x3, xzr
  => 0x40d666d0:  br      x17
     0x40d666d4:  nop
     0x40d666d8:  nop
     0x40d666dc:  nop
  (gdb) i reg
     x0             0x40d64000          1087782912
     x1             0x0                 0
     x2             0x0                 0
     x3             0x0                 0
     x4             0x20                32
     x5             0x6                 6
     x6             0x5511e342          1427235650
     x7             0x773ae706          2000348934
     x8             0x96e74118          2531737880
     x9             0x15cfa4a8          365929640
     x10            0x69b1c47f          1773257855
     x11            0xafa3c749          2946746185
     x12            0x31434030          826490928
     x13            0xff6adceb          4285193451
     x14            0x1b6795a7          459773351
     x15            0xc5fe6594          3321783700
     x16            0x6d9258c2          1838307522
     x17            0x40080000          1074266112
     x18            0xf0821b25          4035058469
     x19            0x40d66ee0          1087794912
     x20            0x40d6a000          1087807488
     x21            0x22f789000         9386364928
     x22            0xffffff8008896b60  -549612590240
     x23            0xffffff8008a507a0  -549610780768
     x24            0xffffff8008ada000  -549610217472
     x25            0x23393d000         9455259648
     x26            0x234a8a002         9473400834
     x27            0x0                 0
     x28            0xffffff8008ada000  -549610217472
     x29            0xffffffc1f3abfce0  -266494804768
     x30            0x40d666bc          1087792828
     sp             0x40d66ee0          0x40d66ee0
     pc             0x40d666d0          0x40d666d0
     cpsr           0x800003c5          -2147482683
     fpsr           0x0                 0
     fpcr           0x0                 0
  (gdb) si
     0x0000000040080000 in ?? ()
  (gdb) x /32i 0x0000000040080000
     => 0x40080000:  add     x13, x18, #0x16 //UEFI 的 magic “MZ” 签名
        0x40080004:  b       0x40b40000      //跳转到内核文本段 stext
        0x40080008:  .inst   0x00080000 ; undefined
        0x4008000c:  .inst   0x00000000 ; undefined
        0x40080010:  .inst   0x00ce4000 ; undefined
        0x40080014:  .inst   0x00000000 ; undefined
        0x40080018:  .inst   0x0000000a ; undefined
        0x4008001c:  .inst   0x00000000 ; undefined
        0x40080020:  .inst   0x00000000 ; undefined
        0x40080024:  .inst   0x00000000 ; undefined
        0x40080028:  .inst   0x00000000 ; undefined
        0x4008002c:  .inst   0x00000000 ; undefined
        0x40080030:  .inst   0x00000000 ; undefined
        0x40080034:  .inst   0x00000000 ; undefined
        0x40080038:  fcmla   z1.h, p4/m, z18.h, z13.h, #180
        0x4008003c:  .inst   0x00000040 ; undefined
        0x40080040:  .inst   0x00004550 ; undefined
        0x40080044:  .inst   0x0002aa64 ; undefined
        0x40080048:  .inst   0x00000000 ; undefined
        0x4008004c:  .inst   0x00000000 ; undefined
        0x40080050:  .inst   0x00000000 ; undefined
        0x40080054:  .inst   0x020600a0 ; undefined
        0x40080058:  b       0x40100884
        0x4008005c:  .inst   0x00b4a000 ; undefined
        0x40080060:  .inst   0x00199000 ; undefined
  ```
* 检视运行时被加载到内存中的`Image`的内容
  ```lua
  (gdb) x /64b 0x0000000040080000
     0x40080000:     0x4d    0x5a    0x00    0x91    0xff    0xff    0x2a    0x14
     0x40080008:     0x00    0x00    0x08    0x00    0x00    0x00    0x00    0x00
     0x40080010:     0x00    0x40    0xce    0x00    0x00    0x00    0x00    0x00
     0x40080018:     0x0a    0x00    0x00    0x00    0x00    0x00    0x00    0x00
     0x40080020:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
     0x40080028:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
     0x40080030:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
     0x40080038:     0x41    0x52    0x4d    0x64    0x40    0x00    0x00    0x00
  ```
* 对比`Image`二进制文件的内容，可见它们是一样的
  ```lua
  $ hexdump Image -C -n 80
  00000000  4d 5a 00 91 ff ff 2a 14  00 00 08 00 00 00 00 00  |MZ....*.........|
  00000010  00 40 ce 00 00 00 00 00  0a 00 00 00 00 00 00 00  |.@..............|
  00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
  00000030  00 00 00 00 00 00 00 00  41 52 4d 64 40 00 00 00  |........ARMd@...|
  00000040  50 45 00 00 64 aa 02 00  00 00 00 00 00 00 00 00  |PE..d...........|
  00000050  00 00 00 00 a0 00 06 02  0b 02 02 14 00 a0 b4 00  |................|
  00000060  00 90 19 00 00 00 00 00  18 69 ac 00 00 10 00 00  |.........i......|
  ```
* `purgatory_start`跳转的第一条指令就是`Image`的第一条指令`add x13, x18, #0x16`，但它没做什么实际的事情
* 紧接着`b 0x40b40000`跳转到真正的内核文本段`stext`
* 检视运行时`stext`的内容，对比 arch/arm64/kernel/head.S 中的`ENTRY(stext)`
  ```c
  (gdb) si
  0x0000000040080004 in ?? ()
  (gdb)
  0x0000000040b40000 in ?? ()
  (gdb) x /32i 0x0000000040b40000
  => 0x40b40000:  bl      0x40b40020   //ENTRY(stext)
     0x40b40004:  bl      0x4088a008
     0x40b40008:  adrp    x23, 0x40000000
     0x40b4000c:  and     x23, x23, #0x1fffff
     0x40b40010:  bl      0x4088a184
     0x40b40014:  bl      0x40b40040
     0x40b40018:  bl      0x4088a54c
     0x40b4001c:  b       0x4088a2b0  //ENDPROC(stext)
     0x40b40020:  mov     x21, x0
     0x40b40024:  adrp    x0, 0x40c35000
     0x40b40028:  add     x0, x0, #0x0
     0x40b4002c:  stp     x21, x1, [x0]
     0x40b40030:  stp     x2, x3, [x0, #16]
     0x40b40034:  dmb     sy
     0x40b40038:  mov     x1, #0x20                       // #32
     0x40b4003c:  b       0x400a0d68
     0x40b40040:  mov     x28, x30
     0x40b40044:  adrp    x0, 0x40d5e000
     0x40b40048:  adrp    x1, 0x40d64000
     0x40b4004c:  sub     x1, x1, x0
     0x40b40050:  bl      0x400a0d68
     0x40b40054:  adrp    x0, 0x40d5e000
     0x40b40058:  adrp    x1, 0x40d64000
     0x40b4005c:  sub     x1, x1, x0
     0x40b40060:  stp     xzr, xzr, [x0], #16
     0x40b40064:  stp     xzr, xzr, [x0], #16
     0x40b40068:  stp     xzr, xzr, [x0], #16
     0x40b4006c:  stp     xzr, xzr, [x0], #16
     0x40b40070:  subs    x1, x1, #0x40
     0x40b40074:  b.ne    0x40b40060  // b.any
     0x40b40078:  mov     x7, #0x711                      // #1809
     0x40b4007c:  adrp    x0, 0x40d5e000
  (gdb)
  ```

# Troubleshooting
## 导出运行时设备树
* 有一些 u-boot 可能会在 dtb 中动态插入一些关键的节点，导致用编译出来的 dtb 作为`kexec --dtb=`参数时，捕获内核用该 dtb 无法正常启动
* 这种情况你需要的是真正的 dtb，一个简单的办法是导出运行时 dtb
  ```sh
  dtc -I fs -O dtb -o runtime.dtb /proc/device-tree
  ```
* 可以用内核编译目录的`dtc`来反编译 dtb 来检查一下：
  ```sh
  scripts/dtc/dtc -I dtb -O dts -o runtime.dts runtime.dtb
  ```
## 捕捉内核无法 reboot
* 有的 BSP 在切换到捕捉内核后执行`reboot`命令无法重启系统，跟踪到最后发现，执行`smc`指令时挂死
```c
SYSCALL_DEFINE4(reboot,...)
-> kernel_restart(NULL)
   -> migrate_to_reboot_cpu()
   -> machine_restart(cmd)
      -> if (arm_pm_restart)
         -> arm_pm_restart(reboot_mode, cmd)
         => axxia_pm_restart()
            -> initiate_retention_reset()
               -> saved_arm_pm_restart(0, NULL);
               => psci_sys_reset()
                  -> invoke_psci_fn(PSCI_0_2_FN_SYSTEM_RESET, 0, 0, 0);
                  => __invoke_psci_fn_smc() /*根据设备树中的 soc.psci.method 的信息在 get_set_conduit_method() -> set_conduit() 中设定*/
                     -> arm_smccc_smc(function_id, arg0, arg1, arg2, 0, 0, 0, 0, &res);
                     => __arm_smccc_smc(__VA_ARGS__, NULL) /*宏定义*/
                        arch/arm64/kernel/smccc-call.S
                        -> SMCCC   smc /*SMC Call Convention*/
                           -> .macro SMCCC instr
                              -> \instr  #0 /*即 smc #0*/
```
* 该挂死是由于一些特殊硬件平台的限制导致的，有的硬件平台仅允许 CPU `MPIDR`为 0 的 CPU 发送 psci 命令
* arch/arm64/include/asm/cputype.h
  ```c
  /*
   * The CPU ID never changes at run time, so we might as well tell the
   * compiler that it's constant.  Use this function to read the CPU ID
   * rather than directly reading processor_id or read_cpuid() directly.
   */
  static inline u32 __attribute_const__ read_cpuid_id(void)
  {
      return read_cpuid(MIDR_EL1);
  }

  static inline u64 __attribute_const__ read_cpuid_mpidr(void)
  {
      return read_cpuid(MPIDR_EL1);
  }
  ```
  * `MIDR`读出来的内容对于每个核都一样，是按一定规范组织的，例如厂商信息
  * `MPIDR`读出来的内容对于每个核则是不同的
* 通常情况下，CPU `MPIDR`与逻辑 ID 是一一对应的，所以重启时，`migrate_to_reboot_cpu()`会将执行`reboot`命令的任务迁移到`reboot_cpu`（可通过`reboot=`参数去设定）上去执行，而`reboot_cpu`缺省值通常为 0，所以没有问题
  * 可以将 CPU0 offline 来证实这一点，CPU0 offline 后 reboot 也会挂死
* 问题在于，当发生 panic 时，不一定就发生在 CPU0，而且 panic 时会有 CPU offline 操作，因此切换到捕捉内核后，逻辑 CPU0 是发生 panic 前的 CPU，之后各 CPU 重新 online，这样会导致 CPU `MPIDR`与逻辑 ID 不再是一一对应的。问题就来了，如果发生 panic 发生在 `MPIDR`不为 0 的 CPU，在捕捉内核的系统中 reboot 就会挂死
### 解决方案
1. 重启时将任务迁移到 `MPIDR`为 0 的 CPU 上去执行
```diff
diff --git a/kernel/reboot.c b/kernel/reboot.c
--- a/kernel/reboot.c
+++ b/kernel/reboot.c
@@ -218,9 +225,18 @@ void migrate_to_reboot_cpu(void)
        int cpu = reboot_cpu;

        cpu_hotplug_disable();
-
+#if defined(CONFIG_CRASH_DUMP)
+       /* For XXXX Aarch64 only the CPU which MPIDR is 0 can reboot system */
+       if (!arch_match_cpu_phys_id(cpu, CPU_MPIDR_0)) {
+               cpu = get_logical_index(CPU_MPIDR_0);
+               pr_warn("The MPIDR of reboot_cpu(%d) is not 0,"
+                       "migrates to CPU%d\n", reboot_cpu, cpu);
+       }
        /* Make certain the cpu I'm about to reboot on is online */
+       if (cpu == -EINVAL || !cpu_online(cpu))
+#else
        if (!cpu_online(cpu))
+#endif
                cpu = cpumask_first(cpu_online_mask);

        /* Prevent races with other tasks migrating this task */
```
2. 这有个前提，就是不能再给捕捉内核传递`maxcpus=`选项了，因为你无法保证`MPIDR`为 0 的 CPU 在 online CPU 中
3. 这又引入一个新问题，看 SMP CPU 启动的过程，
```c
smp_init()
-> for_each_present_cpu(cpu)
   -> cpu_up(cpu)
      arch/arm64/kernel/smp.c
      -> __cpu_up()
         -> boot_secondary(cpu, idle);
            -> cpu_ops[cpu]->cpu_boot(cpu);
            arch/arm64/kernel/psci.c
            => cpu_psci_cpu_boot()
               -> psci_ops.cpu_on(cpu_logical_map(cpu), __pa_symbol(secondary_entry))
               => psci_cpu_on(unsigned long cpuid, unsigned long entry_point)
                  -> invoke_psci_fn(fn, cpuid, entry_point, 0)
```
* 注意，最后发送`PSCI_FN_CPU_ON`PSCI 命令时传递的起始地址是`secondary_entry`。但偏偏有的 BSP 又限制了起始地址的范围必须在物理地址的 1GB 以内，ATF 中会有相应的检查。如果 capture kernel 放置的物理地址不在允许的范围内，其他核会因此无法 online，也还是会造成以上 patch 和配置失效。
  * 为了解决这个问题，我们可以借助`crashkernel=size@offset`参数设定 reserved memory 的物理地址偏移，从而保证 capture kernel 被放在允许的物理地址范围内。

# References
- [深入探索 Kdump，第 1 部分：带你走进 Kdump 的世界](https://www.ibm.com/developerworks/cn/linux/l-cn-kdump1/index.html)
- [深入探索 Kdump，第 2 部分：Kdump 图形化配置工具](https://www.ibm.com/developerworks/cn/linux/l-cn-kdump2/index.html)
- [深入探索 Kdump，第 3 部分：Kdump 原理探秘](https://www.ibm.com/developerworks/cn/linux/l-cn-kdump3/index.html)
- [深入探索 Kdump，第 4 部分：kdump 的亲密战友 crash](https://www.ibm.com/developerworks/cn/linux/l-cn-kdump4/index.html)
- [kexec - A travel to the purgatory](https://eastrivervillage.com/kexec-tools-with-the-hidden-purgatory/)
- [u-boot FIT image介绍](http://www.wowotech.net/u-boot/fit_image_overview.html)
- [Linux Memory Cheat Sheet](https://medium.com/@gabrio.tognozzi/linux-memory-cheat-sheet-2c7454aa1e29)
- [How does a higher half kernel work?](https://medium.com/@connorstack/how-does-a-higher-half-kernel-work-107194e46a64)
- [identity mapped paging](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2012-June/005552.html)
- [深入学习kdump原理](https://dumphex.github.io/2020/02/15/kdump/)
- [ARM指令浅析2（adrp、b）](https://blog.csdn.net/liao392781/article/details/79162919)
- [汇编七、ADRP指令](https://www.jianshu.com/p/e5452c97cfbd)
