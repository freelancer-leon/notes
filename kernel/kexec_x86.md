# Kexec x86-64

## kexec 加载 crash 内核镜像

```c
start_kernel()
-> setup_arch()
   -> reserve_crashkernel()
      -> parse_crashkernel() //解析 crashkernel=XM 参数
      -> memblock_phys_alloc_range() //根据参数分配 memblock
      -> insert_resource(&iomem_resource, &crashk_res); //这样在 /proc/iomem 就能看到分配的 "Crash kernel" range 了
```

### 用户态 kexec-tools

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
```makefile
PURGATORY = purgatory/purgatory.ro
PURGATORY_SRCS =
PURGATORY_SRCS += purgatory/purgatory.c
PURGATORY_SRCS += purgatory/printf.c
PURGATORY_SRCS += purgatory/string.c
PURGATORY_MAP = purgatory/purgatory.map
...
PURGATORY_SRCS+=$($(ARCH)_PURGATORY_SRCS)
```

#### 拷贝 setup_header

##### x86_linux_header

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
* `setup_sects`存的是 setup section 的数目，每个 section 的大小是`512`（见 x86 启动协议），所以在 kexec/arch/x86_64/kexec-bzImage64.c:`do_bzImage64_load()`计算实模式部分的长度时是`kern16_size = (setup_sects + 1) * 512`

##### 内核中的 setup_header

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
    regs64.rsi = setup_base;  /* Pointer to the parameters */ // real_mode_data 的地址，purgatory 会用到
    regs64.rip = addr + 0x200; /* the entry point for startup_64 */ // addr 是重定位后的 crash kernel 的地址，+ 0x200 的解释见下面
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

* `regs64.rip = addr + 0x200;`里的`0x200`是 ABI 的规定，因为用的是 bzImage，所以我们看的是  arch/x86/boot/compressed/head_64.S 这个文件：
  * arch/x86/boot/compressed/head_64.S
  ```c
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
  ```

* `.org`伪指令作用如下：

> Advance the location counter of the current section to *new-lc*. *new-lc* is either an absolute expression or an expression with the same section as the current subsection. That is, you can’t use `.org` to cross sections: if *new-lc* has the wrong section, the `.org` directive is ignored. To be compatible with former assemblers, if the section of *new-lc* is absolute, as issues a warning, then pretends the section of *new-lc* is the same as the current subsection.
> ...
> Beware that the origin is relative to the start of the section, not to the start of the subsection. This is compatible with other people’s assemblers.

* 反汇编`head_64.o`可以看到`startup_64`在 section `.head.text`中的偏移`0x200`处：
  ```s
  ...
  Disassembly of section .head.text:

  0000000000000000 <startup_32>:
  startup_32():
    0:   fc                      cld
  ...
  0000000000000200 <startup_64>:
  startup_64():
  200:   fc                      cld
  201:   fa                      cli
  202:   31 c0                   xor    %eax,%eax
  204:   8e d8                   mov    %eax,%ds
  206:   8e c0                   mov    %eax,%es
  ...
  ```
* 跳转如下：
```c
purgatory_start
-> call purgatory
-> jmp entry64
   -> jmpq *rip(%rip) // 即 jmpq *(%rip + addr + 0x200) => jmpq startup_64
```

### 内核加载 crash 内核镜像
```c
kernel/kexec.c
SYSCALL_DEFINE4(kexec_load, unsigned long, entry, unsigned long, nr_segments, ...)
-> do_kexec_load(entry, nr_segments, segments, flags)
   -> kimage_free(xchg(dest_image, NULL)) //如果有需要，释放旧的 kernel image
   -> kimage_alloc_init(&image, entry, nr_segments, segments, flags)
      -> image = do_kimage_alloc_init(); //这里分配和初始化 kimage 控制结构
         -> image = kzalloc(sizeof(*image), GFP_KERNEL);
            image->head = 0;
            image->entry = &image->head;
      -> image->start = entry; //这个赋值很关键，entry 就是系统调用传进来的入口点，但 kimage 用域 "start" 来记录，因为域 "entry" 是别的意思
      -> copy_user_segment_list(image, nr_segments, segments) // 把用户空间的 segments 数组拷贝到内核空间
         if (kexec_on_panic) {
            /* Enable special crash kernel control page alloc policy. */
            image->control_page = crashk_res.start; //指向 "Crash Kernel"，物理地址
            image->type = KEXEC_TYPE_CRASH;
         }
      -> sanity_check_segment_list(image) //对 segments 的健全性检查，例如范围不重叠、在预留范围内、不能太多
      -> image->control_code_page = kimage_alloc_control_pages(image, get_order(KEXEC_CONTROL_PAGE_SIZE))  //KEXEC_CONTROL_PAGE_SIZE 4096
            case KEXEC_TYPE_CRASH:
            pages = kimage_alloc_crash_control_pages(image, order); //用于从 crash kernel region 分配一个 page，而不是 buddy system
               //以 "Crash Kernel" 为起始地址，在该范围内找到一个不与任何 segments 重叠的一个页大小的空洞
               pages = pfn_to_page(hole_start >> PAGE_SHIFT); //要返回的指向 control page 的 page 结构的指针
               image->control_page = hole_end; //这个指针现在指向 control page 页面的结尾了
   -> machine_kexec_prepare(image)
         start_pgtable = page_to_pfn(image->control_code_page) << PAGE_SHIFT; //根据 page 结构的指针找到 control page 的物理地址
         init_pgtable(image, start_pgtable) //见下面详解
   -> kimage_crash_copy_vmcoreinfo(image)
      -> vmcoreinfo_page = kimage_alloc_control_pages(image, 0); //对于 crash 需要拷贝一份 vmcoreinfo 到 "Crash Kernel" region
      -> safecopy = vmap(&vmcoreinfo_page, 1, VM_MAP, PAGE_KERNEL) //给该页一个当前内核中的虚拟地址
         image->vmcoreinfo_data_copy = safecopy;
      -> crash_update_vmcoreinfo_safecopy(safecopy); //把 vmcoreinfo_data 中的内容拷贝至 "Crash Kernel" region 里
         -> memcpy(ptr, vmcoreinfo_data, vmcoreinfo_size)
         -> vmcoreinfo_data_safecopy = ptr; //这样在 crash 的时候，crash_save_vmcoreinfo() 会更新 "Crash Kernel" region 里 vmcoreinfo note
   -> for (i = 0; i < nr_segments; i++) //把用户空间传进来的 segments 数据拷贝到 "Crash Kernel" region 里
        kimage_load_segment(image, &image->segment[i]);
        -> kimage_load_crash_segment(image, segment) //相对于 -e 的的情况，panic 要简单地多，不需要分配页帧了，直接拷贝就行
   -> kimage_terminate(image) // 结束 kimage->entry 的记录，对于 panic 的情况，没用到这个域，此函数意义不大
   -> image = xchg(dest_image, image); //安装新捕捉内核镜像，卸载旧捕捉内核镜像
   -> kimage_free(image) //释放 kimage 控制结构
```
#### Control pages
> Control pages are special, they are the intermediaries that are needed while we copy the rest of the pages to their final resting place.  As such they must not conflict with either the destination addresses or memory the kernel is already using.
>
> Control pages are also the only pags we must allocate when loading a crash kernel.  All of the other pages are specified by the segments and we just memcpy into them directly.
>
> The only case where we really need more than one of these are for architectures where we cannot disable the MMU and must instead generate an identity mapped page table for all of the memory.
>
> Given the low demand this implements a very simple allocator that finds the first hole of the appropriate size in the reserved memory region, and allocates all of the memory up to and including the hole.
* Control pages 很特别，它是当我们拷贝其余的页面到它们最终要被安放的位置时的中间媒介。因此它们必须不和目的地址冲突，也不能与当前内核正在使用的内存冲突。
* 当我们加载一个 crash kernel 时，control pages 是我们必须分配的唯一页面。所有其他的页面由 segments 指定，我们只是用`memcpy`把他们直接拷贝进来。
* 确实需要多于一页的 control page 的唯一情况是，对于那些无法禁用 MMU 体系结构，必须为所有内存生成一个恒等映射的页表。
* 由于需求不高，所以分配器的实现也非常简单，即在保留内存区域（reserved memory region）找到第一个大小合适的空洞，然后分配所有的内存，直到包含空洞。
* 在`kimage_alloc_crash_control_pages()`函数中，会在 "Crash kernel" 的 region 中找一个`4096 Byte`的空洞，并返回对应的`struct page`的指针，作用和 buddy system 的`alloc_pages()`类似
* `init_pgtable()`会给 crash kernel 用到的内存建立恒等映射，里面有`level4p = (pgd_t *)__va(start_pgtable)`（`start_pgtable`是 control page 的物理地址）说明这个 page 准备用来做 PGD 的！

#### 建立恒等映射（identity mapping）
* 对于 x86，arch/x86/kernel/machine_kexec_64.c 中的 `init_pgtable()` 负责给 crash kernel 用到的内存建立恒等映射
  * arch/x86/kernel/machine_kexec_64.c
```c
static void *alloc_pgt_page(void *data)
{
    struct kimage *image = (struct kimage *)data;
    struct page *page;
    void *p = NULL;
    // 对于 crash case，会调用 kimage_alloc_crash_control_pages() 在 "Crash kernel" 区域中找一个空 page
    page = kimage_alloc_control_pages(image, 0);
    if (page) {
        p = page_address(page); // struct page 指针转成指向该页帧的虚拟地址
        clear_page(p);
    }
    // 返回的是一个虚拟地址
    return p;
}

static int init_pgtable(struct kimage *image, unsigned long start_pgtable)
{
    struct x86_mapping_info info = {
        .alloc_pgt_page = alloc_pgt_page,
        .context    = image,
        .page_flag  = __PAGE_KERNEL_LARGE_EXEC,
        .kernpg_flag    = _KERNPG_TABLE_NOENC,
    };
    unsigned long mstart, mend;
    pgd_t *level4p;
    int result;
    int i;
    // 之前分配到的第一个 control page 作为 PGD
    level4p = (pgd_t *)__va(start_pgtable);
    clear_page(level4p);
...
    // 给 E820 分配的内存建立恒等映射
    for (i = 0; i < nr_pfn_mapped; i++) {
        mstart = pfn_mapped[i].start << PAGE_SHIFT;
        mend   = pfn_mapped[i].end << PAGE_SHIFT;

        result = kernel_ident_mapping_init(&info,
                         level4p, mstart, mend);
        if (result)
            return result;
    }
    // 给 segments 数组里的各 segment 建立恒等映射
    /*
     * segments's mem ranges could be outside 0 ~ max_pfn,
     * for example when jump back to original kernel from kexeced kernel.
     * or first kernel is booted with user mem map, and second kernel
     * could be loaded out of that range.
     */
    for (i = 0; i < image->nr_segments; i++) {
        mstart = image->segment[i].mem;
        mend   = mstart + image->segment[i].memsz;

        result = kernel_ident_mapping_init(&info,
                         level4p, mstart, mend);

        if (result)
            return result;
    }
    // 给 EFI systab 和 ACPI tables 占用的内存建立恒等映射
    /*
     * Prepare EFI systab and ACPI tables for kexec kernel since they are
     * not covered by pfn_mapped.
     */
    result = map_efi_systab(&info, level4p);
    if (result)
        return result;

    result = map_acpi_tables(&info, level4p);
    if (result)
        return result;
    // 设置例程 relocate_kernel 在恒等映射页表项
    return init_transition_pgtable(image, level4p);
}
```
* `kernel_ident_mapping_init()`是负责建立一个范围的恒等映射的接口，大体思路是这样：
  1. 它先检查某一页表条目是否被映射
  2. 如果已经映射了，则调用它下一级页表的恒等映射初始化函数，如`ident_p4d_init(info, p4d, addr, next)`去初始化下一级的页表
  3. 如果没有映射，则调用`info->alloc_pgt_page(info->context)`，最终会调用`kimage_alloc_crash_control_pages()`在 "Crash kernel" 区域中找一个新 control page
  4. 然后调用它下一级页表的恒等映射初始化函数，如`ident_p4d_init(info, p4d, addr, next)`去初始化下一级的页表
  5. 设置页表条目的值`set_pgd(pgd, __pgd(__pa(pud) | info->kernpg_flag))`
  6. 其中，下一级页表的恒等映射初始化函数会继续调用下下一级的恒等映射初始化函数，重复 1 ~ 5 步骤类似的动作，直到最后`PMD`这一级（映射`2M`内存）不太一样，它填入的地址就是传入的`addr - info->offset`，这就意味着 MMU 到时用这个页表做地址转换的时候，（当`info->offset`为`0`）最后转换得到的物理地址和访问的虚拟地址会是一样的
* arch/x86/mm/ident_map.c
```c
static void ident_pmd_init(struct x86_mapping_info *info, pmd_t *pmd_page,
               unsigned long addr, unsigned long end)
{
    addr &= PMD_MASK;
    for (; addr < end; addr += PMD_SIZE) {
        pmd_t *pmd = pmd_page + pmd_index(addr);

        if (pmd_present(*pmd))
            continue;

        set_pmd(pmd, __pmd((addr - info->offset) | info->page_flag));
    }
}
```
## Panic 时的内核切换

## References

* [Org (Using as)](https://sourceware.org/binutils/docs/as/Org.html)