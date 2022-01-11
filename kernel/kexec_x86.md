# Kexec x86-64
### kexec 加载内核

#### 用户态 kexec-tools

```c
main()
-> is_crashkernel_mem_reserved() //-p，在 /proc/iomen 中确定有没有预留 "Crash kernel" 这个范围
   -> kexec_iomem_for_each_line("Crash kernel\n", crashkernel_mem_callback, NULL)
      -> callback(data, nr, str, start, size)
      => crashkernel_mem_callback() //把传入的值更新到 crash_reserved_mem 数组的第 nr 个元素
-> arch_process_options()
if (do_kexec_file_syscall) //-s, --kexec-file-syscall
-> do_kexec_file_load()
   -> kexec_file_load() // 调用 kexec_file_load() 系统调用来加载文件
if (!do_kexec_file_syscall) //-c, --kexec-syscall
-> my_load()
   +-> kernel_buf = slurp_decompress_file() //尝试用 zlib 或 lzma 解压，否则只是将文件读入内存 
   |  -> slurp_file()
   |    -> slurp_file_generic()
   |       -> slurp_fd()
   |          -> buf = xmalloc(size) //kernel_buf 会指向这
   +-> get_memory_ranges() //得到物理内存的范围信息
   |   -> get_memory_ranges_sysfs() //优先尝试 /sys/firmware/memmap
   |      -> get_firmware_memmap_ranges()
   |   -> get_memory_ranges_proc_iomem() //上面遇到错误，回退到 /proc/iomem，返回 System RAM 等多条数据
   +-> file_type[i].probe(kernel_buf, kernel_size) // 探测传入的 kernel image 文件的类型
   |   => bzImage64_probe() // -t bzImage64
   +-> physical_arch() // 根据`uname()`的返回结果确定当前系统的 ARCH 的类型
   +-> file_type[i].load(argc, argv, kernel_buf, kernel_size, &info)
   |   => bzImage64_load()
   |      -> do_bzImage64_load()
   |         -> load_crashdump_segments() //-p 才会调用这个函数
   |               case KEXEC_ARCH_X86_64:
   |                    elf_info.machine = EM_X86_64;
   |            -> arch_options.core_header_type = get_core_type() //根据输入得到 elfcorehdr 的类型
   |               if (arch_options.core_header_type == CORE_TYPE_ELF64)
   |                  elf_info.class = ELFCLASS64; //这个类型会决定 ELF 的 class 类型
   |            -> get_crash_memory_ranges() //从 proc/iomen 中读取 "Crash kernel" 的范围
   |            -> get_backup_area() //找到第一个大于 640 KiB 的 System RAM region 作为备份区
   |            -> get_kernel_page_offset() //根据 elf_info->machine 得到内核直接映射物理内存的虚拟地址的基地址
   |            -> get_kernel_paddr() //根据 /proc/iomem 中的 "Kernel code" 得到运行时内核代码段的物理地址
   |               -> parse_iomem_single("Kernel code\n", &start, NULL);
   |                  elf_info->kern_paddr_start = start; //这个信息将来会更新到 elfcorehdr 里
   |            -> get_kernel_vaddr_and_size()
   |               -> buf = slurp_file_len(kcore, KCORE_ELF_HEADERS_SIZE, &size); //读 /proc/kcore 的 ELF header 部分（64 KB）到 buf
   |               -> build_elf_core_info() //根据 /proc/kcore 的 ELF header 填充 struct mem_ehdr ehdr
   |                  -> build_elf_info()
   |                     -> build_mem_ehdr()  //填充 ELF header
   |                     -> build_mem_phdrs() //构造并填充 Program herders
   |                     -> build_mem_shdrs() //构造并填充 Section headers，对于内核镜像应该是没有
   |                     -> build_mem_notes() //填充 Notes
   |                  for(phdr = ehdr.e_phdr; phdr != end_phdr; phdr++) // 处理 KASLR，查找真实的 PAGE_OFFSET 
   |                     lowest_vaddr = vaddr; // 从 PT_LOAD 中找到虚拟地址最小的 Program 段
   |                  elf_info->page_offset = lowest_vaddr; // 更新为真实的内核直接映射物理内存的虚拟地址的基地址
   |                  stext_sym = get_kernel_sym("_stext") // 从 /proc/kallsyms 中找到 "_stext" 符号的地址
   |                  // 根据"_stext"符号的地址落在那个 Program 段内，得到运行时内核代码段的起始虚拟地址
   |                  for(phdr = ehdr.e_phdr; stext_sym && phdr != end_phdr; phdr++) {
   |                     elf_info->kern_vaddr_start = saddr; // 得到运行时内核代码段的起始虚拟地址
   |                     elf_info->kern_size = size; // 得到运行时内核代码段所在 Program 段的大小
   |                     return 0;
   |                  }
   |                  // 上一个步骤找不到，根据硬编码的 X86_64__START_KERNEL_map（0xffffffff80000000ULL）再来查找
   |                  for(phdr = ehdr.e_phdr; phdr != end_phdr; phdr++) {
   |                     elf_info->kern_vaddr_start = saddr; //得到运行时内核代码段的起始虚拟地址
   |                     elf_info->kern_size = size; // 得到运行时内核代码段所在 Program 段的大小
   |                     return 0;
   |                  }
   |            -> info->backup_start = add_buffer(info, ...) //加入 backup region segment 用于存储备份数据
   |            -> crash_create_elf64_headers() //创建 elfcorehdr segment 用于存储 crash 内存镜像数据
   |                  elf = (EHDR *) bufp; //将 elf_info 里的信息填入 EHDR *elf 指向的 buffer
   |               -> get_kernel_vmcoreinfo() //读入 /sys/kernel/vmcoreinfo 的信息
   |                  -> get_vmcoreinfo("/sys/kernel/vmcoreinfo", addr, len);
   |                  for (i = 0; count_cpu > 0; i++) { //构造 percpu 的 Program header
   |                     get_note_info() //读入 /sys/devices/system/cpu/cpu%d/crash_notes 和 crash_notes_size 的信息
   |                     phdr->p_type   = PT_NOTE;
   |                     phdr->p_offset = phdr->p_paddr = notes_addr;
   |                  }
   |                  if (has_vmcoreinfo && !(info->kexec_flags & KEXEC_PRESERVE_CONTEXT)) {
   |                     phdr->p_type   = PT_NOTE;
   |                     phdr->p_offset = phdr->p_paddr = vmcoreinfo_addr; //构造 vmcoreinfo 所在的 Program header
   |                  }
   |                  if (elf_info->kern_size && !xen_present()) { //构造当前内核文本段的 Program header
   |                     phdr->p_type    = PT_LOAD;
   |                     phdr->p_offset  = phdr->p_paddr = elf_info->kern_paddr_start;
   |                     phdr->p_vaddr   = elf_info->kern_vaddr_start;
   |                  }
   |                  for (i = 0; i < ranges; i++, range++) {
   |                     mstart = range->start;
   |                     phdr->p_type    = PT_LOAD;
   |                     phdr->p_offset  = mstart; //为每个 System RAM 区域构造一个 Program header
   |                  }
   |            -> elfcorehdr = add_buffer(info, ...) //加入 elfcorehdr segment，返回其将会被加载到的物理地址
   |            -> cmdline_add_elfcorehdr(mod_cmdline, elfcorehdr) //添加 elfcorehdr=xxxK 参数
   |               if (arch_options.pass_memmap_cmdline) //是否配置了 --pass-memmap-cmdline 参数？
   |                  cmdline_add_memmap(mod_cmdline, memmap_p); //传递 RAM，ACPI，Pmem 类型的 memory regions
   |         -> elf_rel_build_load(info, &info->rhdr, purgatory, purgatory_size, ...)
   |            -> build_elf_rel_info(buf, len, ehdr, flags)) //根据 purgatory 数组（buf）构建 purgatory segment
   |               -> build_elf_info(buf, len, ehdr, flags) //根据传入的 buf 填充 ehdr，即 info->rhdr
   |                  -> build_mem_ehdr()  //填充 ELF header
   |                  -> build_mem_phdrs() //构造并填充 Program herders
   |                  -> build_mem_shdrs() //构造并填充 Section headers
   |                  -> build_mem_notes() //填充 Notes  
   |            -> elf_rel_load(ehdr, info, min, max, end) //加入 purgatory segment 并重定位 info->rhdr 中的 section headers
   |               -> buf_addr = add_buffer(info, buf, bufsz, ...)  //加入 purgatory segment
   |                  info->entry = (void *)entry; //最重要的是修正重定位后的 info->entry 和 info->rhdr.e_entry
   |         -> memcpy((unsigned char *)real_mode + 0x1f1, kernel + 0x1f1, setup_header_size); //从 bzImage 中拷贝 setup_header
   |         -> setup_base = add_buffer(info, real_mode, setup_size, setup_size, ...); //加入 real_mode_data segment
   |         -> addr = add_buffer(info, kernel + kern16_size, k_size, ...); //加入 crash kernel segment
   |         -> setup_linux_bootloader_parameters_high() //加入 initrd segment 并把 cmdline 追加到 real_mode buffer 里
   |         -> elf_rel_get_symbol(&info->rhdr, "entry64_regs", &regs64, sizeof(regs64));
   |            regs64.rsi = setup_base; /* Pointer to the parameters */
   |            regs64.rip = addr + 0x200; /* the entry point for startup_64 */ //设置 rip，entry64 例程跳转到这里
   |            regs64.rsp = elf_rel_get_addr(&info->rhdr, "stack_end"); /* Stack, unused */
   |         -> elf_rel_set_symbol(&info->rhdr, "entry64_regs", &regs64, sizeof(regs64));
   |         -> elf_rel_set_symbol(&info->rhdr, "cmdline_end", &cmdline_end, sizeof(unsigned long));
   |         -> setup_linux_system_parameters(info, real_mode) //填充正常情况下 BIOS 调用需要提供的信息
   |            -> setup_subarch()
   |               -> get_bootparam(&real_mode->hardware_subarch, ...); //从 /sys/kernel/boot_params/data 读启动参数
   |            -> setup_e820() //从 memmory_range 映射中读取 E820 的信息
   |            -> setup_edd_info() //从 /sys/firmware/edd 中读取 EDD 的信息
   |            -> real_mode->acpi_rsdp_addr = get_acpi_rsdp(); //获取 ACPI RSDP (Root System Description Pointer) 的地址
   +-> arch_compat_trampoline() //对于 x86_64 该函数实现为空
   +-> for (i = 0; i < info.nr_segments; i++) //验证是否所有的 segments 都加载到了内存中的有效位置
   |      valid_memory_segment(&info, info.segmen t +i);
   +-> update_purgatory(&info)
   |   -> arch_update_purgatory(info) //计算并设置 sha256 相关的信息，用于启动时 purgatory 做校验 
   +-> if (entry) info.entry = entry; //--entry 参数可以指定`entry`的位置
   +-> dbgprintf("kexec_load: entry = %p flags = 0x%lx\n", info.entry, info.kexec_flags); //打印重要的调试信息
   +-> kexec_load(info.entry, info.nr_segments, info.segment, info.kexec_flags);
-> my_exec() //如果有 -e 参数
   -> reboot(LINUX_REBOOT_CMD_KEXEC)
```

* `struct crash_elf_info elf_info`在`load_crashdump_segments()`的栈上分配
* `struct kexec_info info.kexec_flags`的`KEXEC_ARCH_MASK`缺省情况下是`KEXEC_ARCH_DEFAULT`，在`load_crashdump_segments()`会把它设置为`KEXEC_ARCH_NATIVE`，而这个值是 kexec/kexec-syscall.h 文件中根据编译器预定义的宏决定的：
  ```c
  #ifdef __x86_64__
  #define KEXEC_ARCH_NATIVE   KEXEC_ARCH_X86_64
  #endif
  ```

  * 这个值又会影响到`struct crash_elf_info elf_info`的`machine`的值，对于`KEXEC_ARCH_X86_64`为`EM_X86_64`（include/elf.h）
* `kexec`的 x86 arch 特定参数`--elf32-core-headers`和`--elf64-core-headers`可以用来指定 elfcorehdr 的类型。如果没有指定，`get_core_type()`会根据`elf_info.machine`的值选定一个，对于`EM_X86_64`会是`CORE_TYPE_ELF64`，这个值会最终影响到 ELF 的 class 是什么。
* `get_kernel_page_offset()`会根据`elf_info->machine`设置`elf_info->page_offset`，对于`EM_X86_64`为`X86_64_PAGE_OFFSET (0xffff888000000000ULL)`（kexec/arch/i386/crashdump-x86.h）
  * 这个值是内核直接映射的物理内存的基地址（“direct mapping of all physical memory (page_offset_base)” ，见 Documentation/x86/x86_64/mm.rst）
  * 对于 KASLR 的情况，这个值可能会变，这个会在随后的逻辑中进行处理
* `add_memmap()`用于将一个 segment 添加到新内核可以用来启动的内存区（memory region），segment 的起始和结束地址需对齐到 1K 边界
* `crash_create_elf64_headers()`的定义见：
  * kexec/crashdump.c，`#define FUNC crash_create_elf64_headers`
  * kexec/crashdump-elf.c，`int FUNC(...)`
* `/sys/kernel/vmcoreinfo`里存的是内核的 crash notes buffer 的起始物理地址和长度，例如：`0x000000047c73c000 1024`
* **PerCPU Crash Notes**
  * percpu crash notes buffer 的物理地址的位置和大小可从`/sys/devices/system/cpu/cpu%d/crash_notes`和`crash_notes_size`读到
  * 一个 percpu 区域，当 system crash 时，这里存放 CPU 状态
  * 该区域以一个空 note 作为结束
  * Note Name: `Core`
  * Note Type: `NT_PRSTATUS(1)`
  * 有关于当前 pid 和 CPU 寄存器的信息
* `kexec`的 x86 arch 特定参数`--pass-memmap-cmdline`可用于把 E820 建立的内存映射传给 panic kernel（Pass memory map via command line in kexec on panic case）
  * 其实是追加了`memmap=exactmap`和一系列附属参数，引入该参数见 [Introducing the memmap= kernel command line option](https://lwn.net/Articles/132338/)
* `elf_rel_load()`由`elf_rel_build_load(info, &info->rhdr, purgatory, purgatory_size, ...)`调用，它负责加入 purgatory segment 并重定位`info->rhdr`中的 section headers，最重要的是修正重定位后的`info->entry` 和`info->rhdr.e_entry`
  1. 根据`ehdr->e_entry`在 section header `ehdr->e_shdr`数组中找到 entry 所在的 section header，用`entry_shdr`记录，本地变量`entry`改为记录 entry 相对所在 section 起始地址的偏移
  2. 找到可重定位的对象，统计内存占用，记录在`bufsz`里；对于`.bss` section 的记录在`bsssz`里
  3. 为可重定位对象分配内存，并调用`buf_addr = add_buffer(info, buf, bufsz, bufsz + bss_pad + bsssz,...)`添加到 segment 数组里，并用`data_addr`和`bss_addr`记录重定位后的基地址
  4. 更新`ehdr->e_shdr`数组里每个 section header 的地址`sh_addr`为重定位后的地址和`sh_data`指针，还把原`shdr->sh_data`中的数据拷贝到新分配的`buf + off`当中
  5. 修正重定位后的`info->entry` 和`info->rhdr.e_entry`
  6. 遍历`ehdr->e_shdr`，重定位 section 中的符号

#### Purgatory - x86

```c
purgatory()
-> printf("I'm in purgatory\n");
   -> vsprintf()
      -> putchar()
         -> putchar_vga()
         -> putchar_serial()
-> setup_arch();
      if (reset_vga)    x86_reset_vga();
      if (legacy_pic)   x86_setup_legacy_pic();
   if (!skip_checks && verify_sha256_digest()) {
      for(;;) { /* loop forever */ } //校验失败会在这死循环
   }
-> post_verification_setup_arch()
      if (panic_kernel)    crashdump_backup_memory();
      if (jump_back_entry) x86_setup_jump_back_entry();
```

* `purgatory/arch`目录下的很多文件都会被编进`purgatory.ro.sym`，其中很多重要的变量和例程都在`.S`文件里提供
  * purgatory/arch/x86_64/setup-x86_64.S 提供了例程`purgatory_start`，它会`call purgatory`并`jmp entry64`
  * purgatory/arch/x86_64/entry64.S 提供了例程`entry64`和变量`entry64_regs`，例程`entry64`的最后会`jmpq *rip(%rip)`，`rip`的值在`do_bzImage64_load()`会被改成`regs64.rip = addr + 0x200`
  * purgatory/arch/x86_64/entry64-32.S 提供了例程`entry32`和变量`entry32_regs`
  * purgatory/arch/x86_64/entry32-16.S 提供了例程`entry16`和变量`entry16_regs`
* purgatory/arch/x86_64/Makefile
```makefile
x86_64_PURGATORY_SRCS_native = purgatory/arch/x86_64/entry64-32.S
x86_64_PURGATORY_SRCS_native += purgatory/arch/x86_64/entry64.S
x86_64_PURGATORY_SRCS_native += purgatory/arch/x86_64/setup-x86_64.S
x86_64_PURGATORY_SRCS_native += purgatory/arch/x86_64/stack.S
x86_64_PURGATORY_SRCS_native += purgatory/arch/x86_64/purgatory-x86_64.c

x86_64_PURGATORY_SRCS += $(x86_64_PURGATORY_SRCS_native)
...
# Don't add sources in i386/ to dist, as i386/Makefile adds them
x86_64_PURGATORY_SRCS += purgatory/arch/i386/entry32-16.S
x86_64_PURGATORY_SRCS += purgatory/arch/i386/entry32-16-debug.S
x86_64_PURGATORY_SRCS += purgatory/arch/i386/crashdump_backup.c
x86_64_PURGATORY_SRCS += purgatory/arch/i386/console-x86.c
x86_64_PURGATORY_SRCS += purgatory/arch/i386/vga.c
x86_64_PURGATORY_SRCS += purgatory/arch/i386/pic.c
```
* purgatory/Makefile
```make
PURGATORY = purgatory/purgatory.ro
PURGATORY_SRCS =
PURGATORY_SRCS += purgatory/purgatory.c
PURGATORY_SRCS += purgatory/printf.c
PURGATORY_SRCS += purgatory/string.c
PURGATORY_MAP = purgatory/purgatory.map
...
PURGATORY_SRCS+=$($(ARCH)_PURGATORY_SRCS)
```

## 拷贝 setup_header

### x86_linux_header

* kexec/include/x86/x86-linux.h

```c
struct x86_linux_header {
    uint8_t  reserved1[0xc0];       /* 0x000 */
    uint32_t ext_ramdisk_image;     /* 0x0c0 */
    uint32_t ext_ramdisk_size;      /* 0x0c4 */
    uint32_t ext_cmd_line_ptr;      /* 0x0c8 */
    uint8_t  reserved1_1[0x1f1-0xcc];   /* 0x0cc */
    uint8_t  setup_sects;           /* 0x1f1 */
    uint16_t root_flags;            /* 0x1f2 */
    uint32_t syssize;           /* 0x1f4 */
    uint16_t ram_size;          /* 0x1f8 */
    uint16_t vid_mode;          /* 0x1fa */
    uint16_t root_dev;          /* 0x1fc */
    uint16_t boot_sector_magic;     /* 0x1fe */
    /* 2.00+ */
    uint16_t jump;              /* 0x200 */
    uint8_t  header_magic[4];       /* 0x202 */
    uint16_t protocol_version;      /* 0x206 */
    uint32_t realmode_swtch;        /* 0x208 */
    uint16_t start_sys;         /* 0x20c */
    uint16_t kver_addr;         /* 0x20e */
    uint8_t  type_of_loader;        /* 0x210 */
    uint8_t  loadflags;         /* 0x211 */
    uint16_t setup_move_size;       /* 0x212 */
    uint32_t code32_start;          /* 0x214 */
    uint32_t ramdisk_image;         /* 0x218 */
    uint32_t ramdisk_size;          /* 0x21c */
    uint32_t bootsect_kludge;       /* 0x220 */
    /* 2.01+ */
    uint16_t heap_end_ptr;          /* 0x224 */
    uint8_t  ext_loader_ver;        /* 0x226 */
    uint8_t  ext_loader_type;       /* 0x227 */
    /* 2.02+ */
    uint32_t cmd_line_ptr;          /* 0x228 */
    /* 2.03+ */
    uint32_t initrd_addr_max;       /* 0x22c */

    uint32_t kernel_alignment;      /* 0x230 */
    uint8_t  relocatable_kernel;        /* 0x234 */
    uint8_t  min_alignment;         /* 0x235 */
    uint16_t xloadflags;            /* 0x236 */
    uint32_t cmdline_size;          /* 0x238 */
    uint32_t hardware_subarch;      /* 0x23C */
    uint64_t hardware_subarch_data;     /* 0x240 */
    uint32_t payload_offset;        /* 0x248 */
    uint32_t payload_size;          /* 0x24C */
    uint64_t setup_data;            /* 0x250 */
    uint64_t pref_address;          /* 0x258 */
    uint32_t init_size;         /* 0x260 */
    uint32_t handover_offset;       /* 0x264 */
} __attribute__((packed));
```
* `setup_sects`存的是 setup section 的数目，每个 section 的大小是`512`，所以在 kexec/arch/x86_64/kexec-bzImage64.c:`do_bzImage64_load()`计算实模式部分的长度时是`kern16_size = (setup_sects + 1) * 512`
### 内核中的 setup_header

* `arch/x86/boot/header.S`会被编译成`arch/x86/boot/header.o`，这个文件会和其他文件一起编译进`arch/x86/boot/setup.elf`，接着`setup.elf`会被`objcopy`去掉无用的信息成为`setup.bin`，这个文件会被和第二次编译出来的`vmlinux.bin`合并成`bzImage`。所以`header.o`和`setup.elf`去掉 ELF 头与`setup.bin`，`bzImage`的前面一段的内容是一样的，构成`x86_linux_header`
* arch/x86/boot/header.S

```s
...
    .globl  hdr
hdr:
setup_sects:    .byte 0         /* Filled in by build.c */ # struct x86_linux_header 域 uint8_t setup_sects; /* 0x1f1 */ 就是这里
root_flags: .word ROOT_RDONLY
syssize:    .long 0         /* Filled in by build.c */
ram_size:   .word 0         /* Obsolete */
vid_mode:   .word SVGA_MODE
root_dev:   .word 0         /* Filled in by build.c */
boot_flag:  .word 0xAA55

    # offset 512, entry point

    .globl  _start
_start:
        # Explicitly enter this as bytes, or the assembler
        # tries to generate a 3-byte jump here, which causes
        # everything else to push off to the wrong offset.
        .byte   0xeb        # short (2-byte) jump //kexec 的 struct x86_linux_header 域 jump; /* 0x200 */ 就是这里
        .byte   start_of_setup-1f # 这两个标号相减的结果就是 setup header 的长度，存储在这个地方，kexec 的 kernel[0x201] 就在这里
1:
    # Part 2 of the header, from the old setup.S

        .ascii  "HdrS"      # header signature //struct x86_linux_header 域 header_magic[4]; /* 0x202 */ 是这里
        .word   0x020f      # header version number (>= 0x0105)
                    # or else old loadlin-1.5 will fail)
...
kernel_alignment:  .long CONFIG_PHYSICAL_ALIGN  #physical addr alignment
                        #required for protected mode
                        #kernel //struct x86_linux_header 域 kernel_alignment; /* 0x230 */ 在这里
...
init_size:      .long INIT_SIZE     # kernel initialization size //struct x86_linux_header 域 init_size; /* 0x260 */ 在这里
handover_offset:    .long 0         # Filled in by build.c
kernel_info_offset: .long 0         # Filled in by build.c

# End of setup header #####################################################

    .section ".entrytext", "ax"
start_of_setup:
# Force %es = %ds
    movw    %ds, %ax
    movw    %ax, %es
    cld
...
```
* 所以 kexec/arch/x86_64/kexec-bzImage64.c`do_bzImage64_load()`拷贝 setup_header 就是`hdr/setup_sects`到`start_of_setup`这一段
```c
static int do_bzImage64_load(...)
{
   ...
    /* only copy setup_header */
    setup_header_size = kernel[0x201] + 0x202 - 0x1f1; // (start_of_setup-1f) + &"HdrS" - setup_sects
    if (setup_header_size > 0x7f)
        setup_header_size = 0x7f;
    memcpy((unsigned char *)real_mode + 0x1f1, kernel + 0x1f1,
         setup_header_size); //拷贝了 bzImage 的 setup_header 部分作为 real_mode_data segment

    /* No real mode code will be executing. setup segment can be loaded
     * anywhere as we will be just reading command line.
     */
    setup_base = add_buffer(info, real_mode, setup_size, setup_size,
                16, 0x3000, -1, -1);

    dbgprintf("Loaded real_mode_data and command line at 0x%lx\n",
            setup_base);

    /* The main kernel segment */
    k_size = kernel_len - kern16_size;
    /* need to use run-time size for buffer searching */
    dbgprintf("kernel init_size 0x%x\n", real_mode->init_size); // 看上面的 header.S，来自 bzImage
    size = _ALIGN(real_mode->init_size, 4096);
    align = real_mode->kernel_alignment; // 看上面的 header.S，来自 bzImage
    addr = add_buffer(info, kernel + kern16_size, k_size, // kernel segment 跳过了 setup sections
              size, align, 0x100000, -1, -1);
    if (addr == ULONG_MAX)
        die("can not load bzImage64");
    dbgprintf("Loaded 64bit kernel at 0x%lx\n", addr);
    /* Tell the kernel what is going on */
    setup_linux_bootloader_parameters_high(info, real_mode, setup_base,
            kern16_size_needed, command_line, command_line_len,
            initrd, initrd_len, 1); /* put initrd high too */

    elf_rel_get_symbol(&info->rhdr, "entry64_regs", &regs64,
                 sizeof(regs64)); //从已加载的 segment 中找到变量 entry64_regs，把内容存到本地变量 regs64
    regs64.rbx = 0;           /* Bootstrap processor */
    regs64.rsi = setup_base;  /* Pointer to the parameters */
    regs64.rip = addr + 0x200; /* the entry point for startup_64 */
    regs64.rsp = elf_rel_get_addr(&info->rhdr, "stack_end"); /* Stack, unused */
    elf_rel_set_symbol(&info->rhdr, "entry64_regs", &regs64,
                 sizeof(regs64)); //本地变量 regs64 的内容设置回已加载的 segment 的变量 entry64_regs

    cmdline_end = setup_base + kern16_size_needed + command_line_len - 1;
    elf_rel_set_symbol(&info->rhdr, "cmdline_end", &cmdline_end,
               sizeof(unsigned long));

    /* Fill in the information BIOS calls would normally provide. */
    setup_linux_system_parameters(info, real_mode);

    return 0;
}
```