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
   +-> file_type[i].probe(kernel_buf, kernel_size) //探测传入的 kernel image 文件的类型
   |   => bzImage64_probe() // -t bzImage64
   +-> physical_arch() // 根据`uname()`的返回结果确定当前系统的 ARCH 的类型
   +-> file_type[i].load(argc, argv, kernel_buf, kernel_size, &info)
   |   => bzImage64_load()
   |      -> do_bzImage64_load()
   |         -> load_crashdump_segments() //-p 才会调用这个函数
   |               switch(kexec_arch)
   |               case KEXEC_ARCH_X86_64:
   |                    elf_info.machine = EM_X86_64;
   |            -> get_crash_memory_ranges() //根据 /proc/iomem 的内容填充 crash_memory_range[] 数组
   |               // 1. 根据 /proc/iomem 填充 crash_memory_range[] 数组
   |               // 2. 从 crash_memory_range[] 数组中剔除 crash_reserved_mem[] 数组的范围
   |            -> get_backup_area() //找到第一个大于 640 KiB 的 System RAM region 作为备份区
   |               if (arch_options.core_header_type == CORE_TYPE_UNDEF) //没有通过参数指定准备的 elfcorehdr 的类型
   |                  arch_options.core_header_type = get_core_type() //根据 elf_info->machine 返回 elfcorehdr 的类型；如果不提供，则根据 range 来决定
   |               if (arch_options.core_header_type == CORE_TYPE_ELF64)
   |                  elf_info.class = ELFCLASS64; //这个类型会决定 ELF 的 class 类型
   |            -> get_kernel_page_offset() //根据 elf_info->machine 得到内核直接映射物理内存的虚拟地址的基地址(PAGE_OFFSET)
   |            -> get_kernel_paddr() //根据 /proc/iomem 中的 "Kernel code" 得到当前内核代码段的物理地址
   |                  if (parse_iomem_single("Kernel code\n", &start, NULL) == 0)
   |                     elf_info->kern_paddr_start = start; //当前内核代码段的物理地址的信息将来会更新到 elfcorehdr 里
   |            -> get_kernel_vaddr_and_size()
   |               -> buf = slurp_file_len(kcore, KCORE_ELF_HEADERS_SIZE, &size); //读 /proc/kcore 的 ELF header 部分（64 KB）到 buf
   |               -> build_elf_core_info() //根据 /proc/kcore 的 ELF header 填充 struct mem_ehdr ehdr
   |                  -> build_elf_info()
   |                     -> build_mem_ehdr()  //填充 ELF header
   |                     -> build_mem_phdrs() //构造并填充 Program herders
   |                     -> build_mem_shdrs() //构造并填充 Section headers，对于内核镜像应该是没有
   |                     -> build_mem_notes() //填充 Notes
   |                  for(phdr = ehdr.e_phdr; phdr != end_phdr; phdr++) // 处理 KASLR，查找真实的 PAGE_OFFSET 
   |                     if (lowest_vaddr == 0 || lowest_vaddr > vaddr)
   |                        lowest_vaddr = vaddr; // 从 PT_LOAD 的 PHDR 中找到虚拟地址最小的 Program 段
   |                  elf_info->page_offset = lowest_vaddr; //更新为真实的内核直接映射物理内存的虚拟地址的基地址
   |                  stext_sym = get_kernel_sym("_stext") //从 /proc/kallsyms 中找到 "_stext" 符号的地址
   |                  // 根据"_stext"符号的地址落在那个 Program 段内，得到运行时内核代码段的起始虚拟地址
   |                  for(phdr = ehdr.e_phdr; stext_sym && phdr != end_phdr; phdr++) {
   |                     elf_info->kern_vaddr_start = saddr; //得到运行时内核代码段的起始虚拟地址
   |                     elf_info->kern_size = size; //得到运行时内核代码段所在 Program 段的大小
   |                     return 0;
   |                  }
   |                  // 上一个步骤找不到，根据硬编码的 X86_64__START_KERNEL_map（0xffffffff80000000ULL）再来查找
   |                  for(phdr = ehdr.e_phdr; phdr != end_phdr; phdr++) {
   |                     elf_info->kern_vaddr_start = saddr; //得到运行时内核代码段的起始虚拟地址
   |                     elf_info->kern_size = size; // 得到运行时内核代码段所在 Program 段的大小
   |                     return 0;
   |                  }
   |            -> info->backup_start = add_buffer(info, ...) //加入 backup region segment 用于存储 backup data
   |            -> crash_create_elf64_headers(..., &elf_info, ..., &tmp, ...) //创建 elfcorehdr segment 用于存储 crash 内存镜像数据，即 FUNC()
   |               -> get_kernel_vmcoreinfo(&vmcoreinfo_addr, &vmcoreinfo_len) //读入 /sys/kernel/vmcoreinfo 的信息
   |                  -> get_vmcoreinfo("/sys/kernel/vmcoreinfo", addr, len); //结果存入 addr 和 len
   |                  elf = (EHDR *) bufp; //将 elf_info 里的信息填入 EHDR *elf 指向的 bufp，也就是 tmp
   |                  for (i = 0; count_cpu > 0; i++) { //构造 percpu 的 Program header
   |                     -> get_note_info() //读入 /sys/devices/system/cpu/cpu%d/crash_notes 和 crash_notes_size 的信息
   |                     => get_crash_notes_per_cpu()
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
   |            -> build_elf_rel_info(buf, len, ehdr, flags) //根据 purgatory 数组（buf）构建 purgatory segment
   |               -> build_elf_info(buf, len, ehdr, flags) //根据传入的 buf 填充 ehdr，即 info->rhdr
   |                  -> build_mem_ehdr()  //填充 ELF header
   |                  -> build_mem_phdrs() //构造并填充 Program herders
   |                  -> build_mem_shdrs() //构造并填充 Section headers
   |                  -> build_mem_notes() //填充 Notes  
   |            -> elf_rel_load(ehdr, info, min, max, end) //加入 purgatory segment 并重定位 info->rhdr 中的 section headers
   |               -> buf_addr = add_buffer(info, buf, bufsz, ...)  //加入 purgatory segment
   |                  info->entry = (void *)entry; //最重要的是修正重定位后的 info->entry 和 info->rhdr.e_entry
   |         -> memcpy((unsigned char *)real_mode + 0x1f1, kernel + 0x1f1, setup_header_size); //从 bzImage 中拷贝 setup_header
   |         -> setup_base = add_buffer(info, real_mode, setup_size, setup_size, ...); //把 real_mode_data 加入 segment[]
   |         -> addr = add_buffer(info, kernel + kern16_size, k_size, ...); //把 crash kernel 加入 segment[]
   |         -> setup_linux_bootloader_parameters_high() //加入 initrd segment 并把 cmdline 追加到 real_mode buffer 里
   |         -> elf_rel_get_symbol(&info->rhdr, "entry64_regs", &regs64, sizeof(regs64));
   |            regs64.rsi = setup_base; /* Pointer to the parameters */ // 指向 boot parameter，setup header 包含在里面，64 位启动协议的约定
   |            regs64.rip = addr + 0x200; /* the entry point for startup_64 */ //设置 rip，entry64 例程跳转到这里，64 位启动协议的约定
   |            regs64.rsp = elf_rel_get_addr(&info->rhdr, "stack_end"); /* Stack, unused */
   |         -> elf_rel_set_symbol(&info->rhdr, "entry64_regs", &regs64, sizeof(regs64));
   |         -> elf_rel_set_symbol(&info->rhdr, "cmdline_end", &cmdline_end, sizeof(unsigned long));
   |         -> setup_linux_system_parameters(info, real_mode) //拷贝正常情况下 BIOS 调用提供的信息
   |            -> setup_subarch()
   |               -> get_bootparam(&real_mode->hardware_subarch, ...); //从 /sys/kernel/boot_params/data 读启动参数
   |            -> setup_e820() //从 info->crash_range/memmory_range 中读取范围信息，填充到 real_mode->e820_map 条目
   |            -> setup_edd_info() //从 /sys/firmware/edd 中读取 EDD 的信息
   |            -> real_mode->acpi_rsdp_addr = get_acpi_rsdp(); //获取 ACPI RSDP (Root System Description Pointer) 的地址
   +-> arch_compat_trampoline() //对于 x86_64 该函数实现为空
   +-> for (i = 0; i < info.nr_segments; i++) //验证是否所有的 segments 都加载到了内存中的有效位置
   |      valid_memory_segment(&info, info.segmen t +i);
   +-> update_purgatory(&info)
   |   -> arch_update_purgatory(info) //设置 purgatory 期间用到的 arch 相关的参数
   |      //根据各 segments 做 sha256 计算并设置`sha256_regions`和`sha256_digest`全局变量
   |      //启动 purgatory 时`verify_sha256_digest()`会根据`sha256_regions`再次计算 sha256 哈希值，和`sha256_digest`比较
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
* `elf_rel_load()`由`elf_rel_build_load(info, &info->rhdr, purgatory, purgatory_size, ...)`调用，它负责把 purgatory segment 加入 segment[] 数组并重定位`info->rhdr`中的 section headers，最重要的是修正重定位后的`info->entry` 和`info->rhdr.e_entry`
  1. 根据`ehdr->e_entry`在 section header `ehdr->e_shdr`数组中找到 entry 所在的 section header，用`entry_shdr`记录，本地变量`entry`改为记录 entry 相对所在 section 起始地址的偏移
  2. 找到可重定位的对象，统计内存占用，记录在`bufsz`里；对于`.bss` section 的记录在`bsssz`里
  3. 为可重定位对象分配内存，并调用`buf_addr = add_buffer(info, buf, bufsz, bufsz + bss_pad + bsssz,...)`添加到 segment 数组里，并用`data_addr`和`bss_addr`记录重定位后的基地址
  4. 逐个把`ehdr->e_shdr`数组里的每个 section header 的原来的数据`shdr->sh_data`拷贝到新分配的`buf + off`当中，然后更新起始地址`shdr->sh_addr`为重定位后的地址以及指向数据的`shdr->sh_data`指针
  5. 修正`info->entry` 和`info->rhdr.e_entry`为重定位到 “Crash kernel” region 后的`purgatory_start`例程的物理地址
  6. 遍历`ehdr->e_shdr`，重定位 section 中的符号
* Backup data
> On x86 machines, the first 640 KB of physical memory is needed to boot, regardless of where the kernel loads. Therefore, kexec backs up this region just before rebooting into the dump-capture kernel. For simpler handling, the whole low 1M is reserved to avoid any later kernel or device driver writing data into this area. Like this, the low 1M can be reused as system RAM by kdump kernel without extra handling.
* `get_backup_area()`会找到第一个大于 640 KiB 的 System RAM region 作为备份区，比如`0000000000001000-000000000009fbff`，并存储在`info->backup_src_start`和`info->backup_src_size`里
* 把 backup data 加入到`segment[]`数组后会返回其在 Crash kernel region 的物理地址`info->backup_start = add_buffer(info, ...)`
* purgatory 阶段的`crashdump_backup_memory()`会把`backup_src_start`开始的长度为`backup_src_size`数据拷贝到`backup_start`处

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

##### setup.bin

###### Legacy 启动模式
> 在进行内核初始化时，需要一些信息，如显示信息、内存信息等。曾经，这些信息由工作在实模式下的 setup.bin 通过 BIOS 获取，保存在内核中的变量`boot_params`中，变量`boot_params`是结构体`struct boot_params`的一个实例。
>
> 在完成信息收集后，setup.bin 将 CPU 切换到保护模式，并跳转到内核的保护模式部分执行。

* 这些硬件相关的参数必须在实模式下借助 BIOS 中断获取
* BIOS 中断可以理解为由 BIOS 固件填充到内存中的中断处理例程
* 变量`struct boot_params boot_params`也即内核中所说的 **零页（zeropage）**
  * arch/x86/include/uapi/asm/bootparam.h
    ```c
    /* The so-called "zeropage" */
    struct boot_params {
        struct screen_info screen_info;         /* 0x000 */
        struct apm_bios_info apm_bios_info;     /* 0x040 */
        __u8  _pad2[4];                 /* 0x054 */
        __u64  tboot_addr;              /* 0x058 */
        struct ist_info ist_info;           /* 0x060 */
        __u64 acpi_rsdp_addr;               /* 0x070 */
        __u8  _pad3[8];                 /* 0x078 */
        __u8  hd0_info[16]; /* obsolete! */     /* 0x080 */
        __u8  hd1_info[16]; /* obsolete! */     /* 0x090 */
        struct sys_desc_table sys_desc_table; /* obsolete! */   /* 0x0a0 */
        struct olpc_ofw_header olpc_ofw_header;     /* 0x0b0 */
        __u32 ext_ramdisk_image;            /* 0x0c0 */
        __u32 ext_ramdisk_size;             /* 0x0c4 */
        __u32 ext_cmd_line_ptr;             /* 0x0c8 */
        __u8  _pad4[116];               /* 0x0cc */
        struct edid_info edid_info;         /* 0x140 */
        struct efi_info efi_info;           /* 0x1c0 */
        __u32 alt_mem_k;                /* 0x1e0 */
        __u32 scratch;      /* Scratch field! */    /* 0x1e4 */
        __u8  e820_entries;             /* 0x1e8 */
        __u8  eddbuf_entries;               /* 0x1e9 */
        __u8  edd_mbr_sig_buf_entries;          /* 0x1ea */
        __u8  kbd_status;               /* 0x1eb */
        __u8  secure_boot;              /* 0x1ec */
        __u8  _pad5[2];                 /* 0x1ed */
        /*
        * The sentinel is set to a nonzero value (0xff) in header.S.
        *
        * A bootloader is supposed to only take setup_header and put
        * it into a clean boot_params buffer. If it turns out that
        * it is clumsy or too generous with the buffer, it most
        * probably will pick up the sentinel variable too. The fact
        * that this variable then is still 0xff will let kernel
        * know that some variables in boot_params are invalid and
        * kernel should zero out certain portions of boot_params.
        */
        __u8  sentinel;                 /* 0x1ef */
        __u8  _pad6[1];                 /* 0x1f0 */
        struct setup_header hdr;    /* setup header */  /* 0x1f1 */
        __u8  _pad7[0x290-0x1f1-sizeof(struct setup_header)];
        __u32 edd_mbr_sig_buffer[EDD_MBR_SIG_MAX];  /* 0x290 */
        struct boot_e820_entry e820_table[E820_MAX_ENTRIES_ZEROPAGE]; /* 0x2d0 */
        __u8  _pad8[48];                /* 0xcd0 */
        struct edd_info eddbuf[EDDMAXNR];       /* 0xd00 */
        __u8  _pad9[276];               /* 0xeec */
    } __attribute__((packed));
    ```
###### EFI 启动模式
> 但是随着新的 BIOS 标准的出现，尤其是 EFI 的出现，为了支持这些新标准，开发者们制定了 32 位启动协议（32-bit boot protocol）。在 32 位启动协议下，由 Bootloader 实现收集这些信息的功能，内核启动时不再需要首先运行实模式部分（即 setup.bin），而是直接跳转到内核的保护模式部分。因此，在 32 位启动协议下，不再需要 setup.bin 收集内核初始化时需要的相关信息。但是这是否意味着可以彻底放弃 setup.bin 呢？
>
> 事实上，除了收集信息功能外，setup.bin 被忽略的另一个重要功能就是负责在内核和 Bootloader 之间传递信息。例如，在加载内核时，Bootloader 需要从 setup.bin 中获取内核是否是可重定位的、内核的对齐要求、内核建议的加载地址等。32 位启动协议约定在 setup.bin 中分配一块空间用来承载这些信息，在构建映像时，内核构建系统需要将这些信息写到 setup.bin 的这块空间中。所以，虽然 setup.bin 已经失去了其以往的作用，但还不能完全放弃，其还要作为内核与 Bootloader 之间传递数据的桥梁，而且还要照顾到某些不能使用 32 位启动协议的场合。

* 启动协议还规定，Bootloader 加载内核时`%rsi`必须存放`struct boot_params`的基地址
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
* 见 arch/x86/include/uapi/asm/bootparam.h 的`struct setup_header`结构体的原型。
* `arch/x86/boot/header.S`会被编译成`arch/x86/boot/header.o`，这个文件会和其他文件一起编译进`arch/x86/boot/setup.elf`，接着`setup.elf`会被`objcopy`去掉无用的信息成为`setup.bin`，这个文件会被和第二次编译出来的`vmlinux.bin`合并成`bzImage`。所以`header.o`和`setup.elf`去掉 ELF 头与`setup.bin`，`bzImage`的前面一段的内容是一样的，构成`x86_linux_header`（但`bzImage`的一部分内容会被`arch/x86/boot/tools/build`修改）。
* arch/x86/boot/header.S
```s
...
    # Kernel attributes; used by setup.  This is part 1 of the
    # header, from the old boot sector.

    .section ".header", "a"
    .globl  sentinel
sentinel:   .byte 0xff, 0xff        /* Used to detect broken loaders */

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

> 在 32 位引导协议下，除了传统的 16 位协议，Bootloader 取代内核中实模式部分负责收集硬件信息（即零页 zero-page 信息）的功能。而 Bootloader 会将 CPU 切换为保护模式，而是直接跳转到内核的保护模式部分。

* 引导协议规定，**协议数据** 从内核镜像的偏移`0x1f1`处开始，`setup.bin`的链接器脚本`arch/x86/boot/setup.ld`因此把它放在了偏移`495 + 2 = 497`处，前面填充了 2 字节的`0xff`作为哨兵

```lua
SECTIONS
{
...
    . = 495;
    .header     : { *(.header) }
}
```
* 所以 kexec/arch/x86_64/kexec-bzImage64.c`do_bzImage64_load()`拷贝 setup_header 就是`hdr/setup_sects`到`start_of_setup`这一段
```c
static int do_bzImage64_load(...)
{
    memcpy(&setup_header, kernel, sizeof(setup_header));
    setup_sects = setup_header.setup_sects;
    if (setup_sects == 0)
        setup_sects = 4;
    kern16_size = (setup_sects + 1) * 512;
    if (kernel_len < kern16_size) {
        fprintf(stderr, "BzImage truncated?\n");
        return -1;
    }
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
    regs64.rsi = setup_base;  /* Pointer to the parameters */ //boot parameters/real_mode_data 的地址，purgatory 会用到
    regs64.rip = addr + 0x200; /* the entry point for startup_64 */ //addr 是重定位后的 crash kernel 的地址，+ 0x200 的解释见下面
    regs64.rsp = elf_rel_get_addr(&info->rhdr, "stack_end"); /* Stack, unused */
    elf_rel_set_symbol(&info->rhdr, "entry64_regs", &regs64,
                 sizeof(regs64)); //本地变量 regs64 的内容设置回已加载的 segment 的变量 entry64_regs
...
    return 0;
}
```
* `regs64.rip = addr + 0x200;`里的`0x200`是 ABI 的规定。因为用的是 bzImage，所以我们看的是 arch/x86/boot/compressed/head_64.S 这个文件：
  * arch/x86/boot/compressed/head_64.S
  ```cpp
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
machine_kexec()
-> relocate_kernel
      identity_mapped // kernel
         purgatory_start //purgatory
         -> call purgatory
         -> jmp entry64
            -> jmpq *rip(%rip) // 即 jmpq *(%rip + addr + 0x200) => jmpq startup_64
               -> startup_64
```

* 所以说，`real_mode_data` segment 包含 bzImage 中拷贝出来的一小部分内容，然后被重定向以后交给内核，最后被用在了这里。此外，它还必须包含启动协议中 BIOS 需要向内核提供的数据，这些数据主要由`setup_linux_bootloader_parameters_high()`和`setup_linux_system_parameters()`这两个函数中填充。
  * 其中，`setup_linux_bootloader_parameters_high()`设置了`real_mode->cmd_line_ptr`指针，并拷贝了命令行参数到`((char *)real_mode) + cmdline_offset`处，也就时零页中的`boot_params.hdr.cmd_line_ptr`处。
  * 也就是说，命令行参数是通过`real_mode_data` segment 来传递的。
* 按照 64-bit 启动协议的规定，跳转到内核入口时，`%rsi` 必须持有 `struct boot_params` 的基地址，这是通过 kexec/arch/x86_64/kexec-bzImage64.c`do_bzImage64_load()` 中的 `regs64.rsi = setup_base` 预先设定的，因为这个值在 load image 到 crash kernel region 的时候就算出来了。

#### 为什么捕捉内核启动时只能看见和保留内存
1. 内核启动后能使用的内存信息是根据零页中的 `struct boot_e820_entry e820_table[E820_MAX_ENTRIES_ZEROPAGE]`得来的，例如，启动时可以看到打印如下：
  ```c
  BIOS-provided physical RAM map:
  BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
  BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
  BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
  BIOS-e820: [mem 0x0000000000100000-0x000000005ffd4fff] usable
  BIOS-e820: [mem 0x000000005ffd5000-0x000000005fffffff] reserved
  BIOS-e820: [mem 0x00000000b0000000-0x00000000bfffffff] reserved
  BIOS-e820: [mem 0x00000000fed1c000-0x00000000fed1ffff] reserved
  BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
  ```
2. 启动后会被 I/O 设备、内核逐渐使用掉，就是在`/proc/iomem`中看到的一些使用情况
3. `get_crash_memory_ranges()`会通过`/proc/iomem`的信息填充`crash_memory_range[]`
4. 从`crash_memory_range[]`数组中剔除（为 crash kernel 保留的内存 region）`crash_reserved_mem[]`数组的范围
5. 在`crash_memory_range[]`数组中找到第一个类型为`System RAM` 640 KiB region 作为 backup data
6. `load_crashdump_segments()`中，先分配`struct memory_range *memmap_p`
7. 把 backup data 范围通过`add_memmap()`函数加到`memmap_p`数组
8. 把`crash_reserved_mem[]`数组通过`add_memmap()`函数加到`memmap_p`数组
9. 创建 backup region segment，并把该范围通过`delete_memmap()`函数从`memmap_p`数组剔除
10. 创建`elfcorehdr` segment，把该范围通过`delete_memmap()`函数从`memmap_p`数组剔除
11. 遍历`mem_range`，即`get_crash_memory_ranges()`填充的`crash_memory_range[]`数组，把以下类型的 region 添加到`memmap_p`数组
  * `RANGE_ACPI`: ACPI Tables
  * `RANGE_ACPI_NVS`：ACPI Non-volatile Storage
  * `RANGE_RESERVED`：Reserved，reserved
  * `RANGE_PMEM`：Persistent Memory
  * `RANGE_PRAM`：Persistent Memory (legacy)
12. 在`setup_e820()`通过`add_e820_map_from_mr()`及`setup_e820_ext()`把`memmap_p`数组里的范围填充到`real_mode_data`的`e820_map[]`数组，其实也就是给 crash kernel 准备的零页的`e820_table[]`数组
    ```c
    BIOS-provided physical RAM map:
    BIOS-e820: [mem 0x0000000000000000-0x0000000000000fff] reserved
    BIOS-e820: [mem 0x0000000000001000-0x000000000009fbff] usable
    BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
    BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
    BIOS-e820: [mem 0x000000003f000000-0x000000005ef5cfff] usable
    BIOS-e820: [mem 0x000000005efffc00-0x000000005effffff] usable
    BIOS-e820: [mem 0x000000005ffd5000-0x000000005fffffff] reserved
    BIOS-e820: [mem 0x00000000b0000000-0x00000000bfffffff] reserved
    BIOS-e820: [mem 0x00000000fed1c000-0x00000000fed1ffff] reserved
    BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
    ```
* 对应的 segments 数组为
  ```c
  1) Crash kernel bzImage
  segment[0].buf   = 0x7f6449f95810
  segment[0].bufsz = 0x91ce60
  segment[0].mem   = 0x5d000000
  segment[0].memsz = 0x1f3f000
  
  2) real_mode_data
  segment[1].buf   = 0x55c74044aa10
  segment[1].bufsz = 0x3936
  segment[1].mem   = 0x5ef50000
  segment[1].memsz = 0x4000
  
  3) purgatory
  segment[2].buf   = 0x55c7404437b0
  segment[2].bufsz = 0x70e0
  segment[2].mem   = 0x5ef54000
  segment[2].memsz = 0x9000
  
  4) elfcoreheader
  segment[3].buf   = 0x55c740441910
  segment[3].bufsz = 0x400
  segment[3].mem   = 0x5ef5d000
  segment[3].memsz = 0x4000
  
  5) backup_data
  segment[4].buf   = 0x7f6449ef3010
  segment[4].bufsz = 0x9ec00
  segment[4].mem   = 0x5ef61000
  segment[4].memsz = 0x9f000
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
      -> copy_user_segment_list(image, nr_segments, segments) //把用户空间的 segment[] 数组拷贝到内核空间，注意只是拷贝这个数组
         if (kexec_on_panic) {
            /* Enable special crash kernel control page alloc policy. */
            image->control_page = crashk_res.start; //指向 "Crash kernel"，物理地址
            image->type = KEXEC_TYPE_CRASH;
         }
      -> sanity_check_segment_list(image) //对 segments 的健全性检查，例如范围不重叠、在预留范围内、不能太多
      -> image->control_code_page = kimage_alloc_control_pages(image, get_order(KEXEC_CONTROL_PAGE_SIZE))  //KEXEC_CONTROL_PAGE_SIZE 8192
            case KEXEC_TYPE_CRASH:
            pages = kimage_alloc_crash_control_pages(image, order); //用于从 Crash kernel region 分配一个 page，而不是 buddy system
               //以 "Crash kernel" 为起始地址，在该范围内找到一个不与任何 segments 重叠的一个页大小的空洞
               pages = pfn_to_page(hole_start >> PAGE_SHIFT); //要返回的指向 control page 的 page 结构的指针
               image->control_page = hole_end; //更新指针指向 control page 页帧的结尾，作为下一个分配 control page 的起点
   -> machine_kexec_prepare(image)
         start_pgtable = page_to_pfn(image->control_code_page) << PAGE_SHIFT; //根据 page 结构的指针找到 control_code_page 的物理地址
         init_pgtable(image, start_pgtable) //负责给 crash kernel 用到的内存建立恒等映射，见下面详解
         -> init_transition_pgtable(image, level4p)
   -> kimage_crash_copy_vmcoreinfo(image)
      -> vmcoreinfo_page = kimage_alloc_control_pages(image, 0); //对于 crash 需要拷贝一份 vmcoreinfo 到 "Crash kernel" region
      -> safecopy = vmap(&vmcoreinfo_page, 1, VM_MAP, PAGE_KERNEL) //给该页一个当前内核中的虚拟地址
         image->vmcoreinfo_data_copy = safecopy;
      -> crash_update_vmcoreinfo_safecopy(safecopy); //把 vmcoreinfo_data 中的内容拷贝至 "Crash kernel" region 里
         -> memcpy(ptr, vmcoreinfo_data, vmcoreinfo_size)
         -> vmcoreinfo_data_safecopy = ptr; //这样在 crash 的时候，crash_save_vmcoreinfo() 会更新 "Crash kernel" region 里 vmcoreinfo_data_safecopy
   -> for (i = 0; i < nr_segments; i++) //把用户空间传进来的 segments 数据拷贝到 "Crash kernel" region 里
        kimage_load_segment(image, &image->segment[i]);
        -> kimage_load_crash_segment(image, segment) //相对于 -l 的的情况，panic 要简单地多，不需要分配页帧了，直接拷贝就行
   -> kimage_terminate(image) // 结束 kimage->entry 的记录，对于 panic 的情况，此函数意义不大，除了设置 IND_DONE 标志位用以跳出 swap_pages 循环
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
* 在`kimage_alloc_crash_control_pages()`函数中，会在 "Crash kernel" region 中找一个大于 2<sup>order</sup> 的空洞，并返回对应的`struct page`的指针，作用和 buddy system 的`alloc_pages()`类似
* `image->control_code_page = kimage_alloc_control_pages(image, get_order(KEXEC_CONTROL_PAGE_SIZE))` 这里`KEXEC_CONTROL_PAGE_SIZE (4096UL + 4096UL)`会申请两个相连的页面，一个用作恒等映射 PGD，另一个用来放`relocate_kernel`例程的代码
* `init_pgtable()`会给 crash kernel 用到的内存建立恒等映射，里面有`level4p = (pgd_t *)__va(start_pgtable)`（`start_pgtable`是 control page 的物理地址）说明这个页面就是准备用来做 PGD 的！

#### 建立恒等映射（identity mapping）
* 建立恒等映射的内存分配函数 `alloc_pgt_page()` 用的 `kimage_alloc_control_pages()` 从 control pages 里分配内存
  * arch/x86/kernel/machine_kexec_64.c
```c
static void *alloc_pgt_page(void *data)
{
    struct kimage *image = (struct kimage *)data;
    struct page *page;
    void *p = NULL;
    //对于 crash 场景会调用 kimage_alloc_crash_control_pages() 在 "Crash kernel" 区域中找一个空 page
    page = kimage_alloc_control_pages(image, 0);
    if (page) {
        p = page_address(page); // struct page 指针转成指向该页帧的虚拟地址
        clear_page(p);
    }
    //返回的是一个虚拟地址
    return p;
}
```
* 对于 x86，arch/x86/kernel/machine_kexec_64.c 中的`init_pgtable()`负责给 crash kernel 用到的内存建立恒等映射
```c
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
    //之前分配到的第一个 control page 作为 PGD
    level4p = (pgd_t *)__va(start_pgtable); //这里得到的是用当前 PGD 转换得到的虚拟地址
    clear_page(level4p);
...
    //给 E820 分配的内存建立恒等映射
    for (i = 0; i < nr_pfn_mapped; i++) {
        mstart = pfn_mapped[i].start << PAGE_SHIFT;
        mend   = pfn_mapped[i].end << PAGE_SHIFT;

        result = kernel_ident_mapping_init(&info,
                         level4p, mstart, mend);
        if (result)
            return result;
    }
    //给 segments 数组里的各 segment 建立恒等映射
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
    //给 EFI systab 和 ACPI tables 占用的内存建立恒等映射
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
    //设置例程 relocate_kernel 的恒等映射页表项，这是恒等映射的关键
    return init_transition_pgtable(image, level4p);
}
```
* `kernel_ident_mapping_init()`是负责建立一个范围的恒等映射的接口，大体思路是这样：
  1. 它先检查某一页表条目是否被映射
  2. 如果已经映射了，则调用它下一级页表的恒等映射初始化函数，如`ident_p4d_init(info, p4d, addr, next)`去初始化下一级的页表
  3. 如果没有映射，则调用`info->alloc_pgt_page(info->context)`，最终会调用`kimage_alloc_crash_control_pages()`在 "Crash kernel" 区域中找一个新 control page（会更新`image->control_page`）
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
* 恒等映射的关键是，变换代码在原来的页表和恒等映射的页表都有映射
  * 这里变换代码是 `relocate_kernel`，它在 `machine_kexec()` 的时候会被拷贝到 `image->control_code_page` 两个相邻页面的第二个页面的开始处
  * 这段代码在原来的页表中就有映射，现在在恒等映射页表中把它映射到 control page 中放置代码的起始处
  * 切换页表后，变换代码没走到跳转为恒等映射的低地址前，`$RIP` 还是原来的高地址，只是恒等映射页表已将它映射到 control page 区域中的物理地址了，CPU 实际上是从这些物理地址去取指令了
  * 这一段内存的映射是恒等映射中唯一不恒等的映射
```cpp
static int init_transition_pgtable(struct kimage *image, pgd_t *pgd)
{
    pgprot_t prot = PAGE_KERNEL_EXEC_NOENC;
    unsigned long vaddr, paddr;
    int result = -ENOMEM;
    p4d_t *p4d;
    pud_t *pud;
    pmd_t *pmd;
    pte_t *pte;
    //走表时用的索引还是根据虚拟地址来的
    vaddr = (unsigned long)relocate_kernel;
    paddr = __pa(page_address(image->control_code_page)+PAGE_SIZE);
    pgd += pgd_index(vaddr);
    if (!pgd_present(*pgd)) {
        p4d = (p4d_t *)get_zeroed_page(GFP_KERNEL); //页表页还是从 buddy 分的
        if (!p4d)
            goto err;
        image->arch.p4d = p4d;
        set_pgd(pgd, __pgd(__pa(p4d) | _KERNPG_TABLE));
    }
    p4d = p4d_offset(pgd, vaddr);
    if (!p4d_present(*p4d)) {
        pud = (pud_t *)get_zeroed_page(GFP_KERNEL);
        if (!pud)
            goto err;
        image->arch.pud = pud;
        set_p4d(p4d, __p4d(__pa(pud) | _KERNPG_TABLE));
    }
    pud = pud_offset(p4d, vaddr);
    if (!pud_present(*pud)) {
        pmd = (pmd_t *)get_zeroed_page(GFP_KERNEL);
        if (!pmd)
            goto err;
        image->arch.pmd = pmd;
        set_pud(pud, __pud(__pa(pmd) | _KERNPG_TABLE));
    }
    pmd = pmd_offset(pud, vaddr);
    if (!pmd_present(*pmd)) {
        pte = (pte_t *)get_zeroed_page(GFP_KERNEL);
        if (!pte)
            goto err;
        image->arch.pte = pte;
        set_pmd(pmd, __pmd(__pa(pte) | _KERNPG_TABLE));
    }
    pte = pte_offset_kernel(pmd, vaddr);

    if (cc_platform_has(CC_ATTR_GUEST_MEM_ENCRYPT))
        prot = PAGE_KERNEL_EXEC;
    //映射的最终目标是填入 control page 中的物理地址
    set_pte(pte, pfn_pte(paddr >> PAGE_SHIFT, prot));
    return 0;
err:
    return result;
}
```

## kexec 内核切换

### 调用 machine_kexec() 的几个场景
* `machine_kexec()`是 kexec 统一的入口函数
* 对于`kexec -e`命令有：
  ```c
  SYSCALL_DEFINE4(reboot, ...)
    case LINUX_REBOOT_CMD_KEXEC:
    -> kernel_kexec()
       -> migrate_to_reboot_cpu() //将任务都迁移到 kexec 的 CPU
       -> kernel_restart_prepare("kexec reboot")
       -> cpu_hotplug_enable()
       -> machine_shutdown() //让其他 CPU 都停下来，见下面详解
       -> machine_kexec(kexec_image)
  ```
* 对于 kernel panic 的场景有，则通过`__crash_kexec()`调用`machine_kexec()`
  ```c
  panic()
  -> __crash_kexec(NULL)
     -> machine_kexec(kexec_crash_image)
  ```
  或者
  ```c
  oops_end()
    if (regs && kexec_should_crash(current))
       crash_kexec(regs);
       -> __crash_kexec(regs)
          -> crash_setup_regs(&fixed_regs, regs) //把 panic CPU 寄存器信息在 fixed_reg 另存了一份
          -> crash_save_vmcoreinfo() //更新 vmcoreinfo note
             -> vmcoreinfo_append_str("CRASHTIME=%lld\n", ktime_get_real_seconds()) //如果已加载了 crash kernel，更新的是 "Crash kernel" region 里的
             -> update_vmcoreinfo_note()
                -> append_elf_note(vmcoreinfo_note, VMCOREINFO_NOTE_NAME, 0, vmcoreinfo_data, ...)//拷贝 vmcoreinfo_data 到 vmcoreinfo_note
                -> final_note(vmcoreinfo_note)
          -> machine_crash_shutdown(&fixed_regs) //注意与快速切换场景的 machine_shutdown() 区分
             -> machine_ops.crash_shutdown(regs) //arch 相关的 crash_shutdown 回调，x86 的 arch/x86/kernel/crash.c
             => native_machine_crash_shutdown(regs) //快速切换场景则是 native_machine_shutdown()
                -> crash_smp_send_stop()
                   if (smp_ops.crash_stop_other_cpus)
                   -> smp_ops.crash_stop_other_cpus()
                   => kdump_nmi_shootdown_cpus()
                      -> nmi_shootdown_cpus(kdump_nmi_callback)
                            crashing_cpu = safe_smp_processor_id();
                            shootdown_callback = callback; // callback 是入参 kdump_nmi_callback
                            //crash_nmi_callback() 注册为 NMI 回调函数，会调 shootdown_callback() 即 kdump_nmi_callback()，
                            //kdump_nmi_callback() 调到 crash_save_cpu(regs, cpu)，从而让其他 CPU 的寄存器信息保存到 crash note
                            if (register_nmi_handler(NMI_LOCAL, crash_nmi_callback, NMI_FLAG_FIRST, "crash"))
                               return;
                            apic_send_IPI_allbutself(NMI_VECTOR);//触发 IPI 中断，让其他 CPU 调 crash_nmi_callback()
                -> crash_save_cpu(regs, safe_smp_processor_id()) //把 panic CPU 的寄存器信息更新到 crash note
          -> machine_kexec(kexec_crash_image)
             -> local_irq_disable() //关中断了
             -> hw_breakpoint_disable() //硬件断点也关了
             -> control_page = page_address(image->control_code_page) + PAGE_SIZE; //恒等映射的页表在当前内核的虚拟地址 + 4096 = 下一页的起始地址
             -> memcpy(control_page, relocate_kernel, KEXEC_CONTROL_CODE_MAX_SIZE); //把 relocate_kernel 拷贝到上面得到的虚拟地址
             -> load_segments() //段寄存器 ds，es，ss，fs，gs 清零
             -> native_idt_invalidate() //通过指令 lidt 将中断描述符表清零
             -> native_gdt_invalidate() //通过指令 lgdt 将全局段寄存器清零
             -> image->start = relocate_kernel((unsigned long)image->head, (unsigned long)page_list, image->start, ...)
  ```

#### 快速切换场景时停止其他 CPU
* 对于 panic 的场景，需要调用 `crash_save_cpu()` 让各 CPU 将其上下文保存到 vmcore 中的 `NOTE` segment 的名为 `CORE`，类型为 `NT_PRSTATUS` 的字段中去
* 对于 `kexec -e` 的重启场景，则不需要 `crash_save_cpu()`，只需让其他 CPU 停止运行，然后等待 reboot CPU 切换到新内核
```c
kernel_kexec()
-> machine_shutdown()
   -> machine_ops.shutdown()
   => native_machine_shutdown()
   -> stop_other_cpus() //让其他 CPU 都停下来
      -> smp_ops.stop_other_cpus(1)
      => native_stop_other_cpus()
         -> apic_send_IPI_allbutself(REBOOT_VECTOR) //第一次尝试发 IPI(REBOOT_VECTOR) 的方式，如果尝试失败或超时
         -> register_stop_handler() //第二次尝试用 NMI DM 的 IPI 尝试让让其他 CPU 停下，回调为 smp_stop_nmi_callback()
             -> register_nmi_handler(NMI_LOCAL, smp_stop_nmi_callback, NMI_FLAG_FIRST, "smp_stop")
            for_each_cpu(cpu, &cpus_stop_mask)
            -> __apic_send_IPI(cpu, NMI_VECTOR)
-> machine_kexec(kexec_image)
```
* 上面已经看到了 `machine_shutdown()` 会做两次尝试，最终都是为了其他 CPU 能最终调用到 `stop_this_cpu()`，最终停止在 `hlt`
```cpp
void __noreturn stop_this_cpu(void *dummy)
{
...
    /*
     * This brings a cache line back and dirties it, but
     * native_stop_other_cpus() will overwrite cpus_stop_mask after it
     * observed that all CPUs reported stop. This write will invalidate
     * the related cache line on this CPU.
     */
    cpumask_clear_cpu(cpu, &cpus_stop_mask);

    for (;;) {
        /*
         * Use native_halt() so that memory contents don't change
         * (stack usage and variables) after possibly issuing the
         * native_wbinvd() above.
         */
        native_halt();
    }
}
```
##### 第一种尝试：`IPI(REBOOT_VECTOR)` 的方式
* 这种方式比较简单，就是通过发送 `REBOOT_VECTOR` 的 IPI，对应的中断向量如下：
* arch/x86/kernel/smp.c
```cpp
#define REBOOT_VECTOR           0xf8
/*
 * this function calls the 'stop' function on all other CPUs in the system.
 */
DEFINE_IDTENTRY_SYSVEC(sysvec_reboot)
{
    apic_eoi();
    cpu_emergency_disable_virtualization();
    stop_this_cpu(NULL);
}
```
* arch/x86/kernel/idt.c
```cpp
/*
 * The APIC and SMP idt entries
 */
static const __initconst struct idt_data apic_idts[] = {
...
#ifdef CONFIG_SMP
    INTG(RESCHEDULE_VECTOR,         asm_sysvec_reschedule_ipi),
    INTG(CALL_FUNCTION_VECTOR,      asm_sysvec_call_function),
    INTG(CALL_FUNCTION_SINGLE_VECTOR,   asm_sysvec_call_function_single),
    INTG(REBOOT_VECTOR,         asm_sysvec_reboot),
#endif
...
}
```
##### 第二种尝试：NMI 的方式
* `register_stop_handler()` 注册了类型为 `NMI_LOCAL` 的 NMI 回调函数 `smp_stop_nmi_callback()`，`NMI_FLAG_FIRST` 让它最先被 NMI 处理函数执行
```cpp
static int smp_stop_nmi_callback(unsigned int val, struct pt_regs *regs)
{
    /* We are registered on stopping cpu too, avoid spurious NMI */
    if (raw_smp_processor_id() == atomic_read(&stopping_cpu))
        return NMI_HANDLED;

    cpu_emergency_disable_virtualization();
    stop_this_cpu(NULL);

    return NMI_HANDLED;
}
```

### 切换内核
* `machine_kexec()`调用`relocate_kernel`例程开启了切换内核之旅，传入的参数如下：
```c
void machine_kexec(struct kimage *image)
{
    unsigned long page_list[PAGES_NR];
    void *control_page;
    ...
     // 回忆之前 kimage_alloc_init() 调用 kimage_alloc_control_pages() 分了两个连续的页面
    control_page = page_address(image->control_code_page) + PAGE_SIZE; //指向第二个页面
    memcpy(control_page, relocate_kernel, KEXEC_CONTROL_CODE_MAX_SIZE);//往第二个页面拷贝 relocate_kernel 的代码

    page_list[PA_CONTROL_PAGE] = virt_to_phys(control_page);  //control page 的物理地址
    page_list[VA_CONTROL_PAGE] = (unsigned long)control_page; //control page 的虚拟地址
    page_list[PA_TABLE_PAGE] =
      (unsigned long)__pa(page_address(image->control_code_page)); //恒等映射页表的物理地址

    if (image->type == KEXEC_TYPE_DEFAULT)
        page_list[PA_SWAP_PAGE] = (page_to_pfn(image->swap_page)
                        << PAGE_SHIFT);
    ...
    /* now call it */ //注意，这里执行的是当前内核文本段的 relocate_kernel 例程
    image->start = relocate_kernel((unsigned long)image->head,
                       (unsigned long)page_list,
                       image->start,
                       image->preserve_context,
                       sme_active());
}
```

* `KEXEC_CONTROL_CODE_MAX_SIZE`为`2048`，意思是把`4096 Byte`的 control_page 分为两部分，前`2048 B`放`relocate_kernel`例程的代码，后面`2048 B`用做数据存储、栈和 jump back
* arch/x86/kernel/relocate_kernel_64.S 这个文件很重要，需要重点了解一下了
```s
/*
 * Must be relocatable PIC code callable as a C function
 */

#define PTR(x) (x << 3)
#define PAGE_ATTR (_PAGE_PRESENT | _PAGE_RW | _PAGE_ACCESSED | _PAGE_DIRTY)

/*
 * control_page + KEXEC_CONTROL_CODE_MAX_SIZE
 * ~ control_page + PAGE_SIZE are used as data storage and stack for
 * jumping back
 */
#define DATA(offset)		(KEXEC_CONTROL_CODE_MAX_SIZE+(offset))

/* Minimal CPU state */
#define RSP			DATA(0x0)  // 0x800
#define CR0			DATA(0x8)  // 0x808
#define CR3			DATA(0x10) // 0x810
#define CR4			DATA(0x18) // 0x818

/* other data */
#define CP_PA_TABLE_PAGE	DATA(0x20)     // 0x820
#define CP_PA_SWAP_PAGE		DATA(0x28)     // 0x828
#define CP_PA_BACKUP_PAGES_MAP	DATA(0x30) // 0x830

	.text
	.align PAGE_SIZE
	.code64
SYM_CODE_START_NOALIGN(relocate_kernel)
	UNWIND_HINT_EMPTY
	/*
	 * %rdi indirection_page
	 * %rsi page_list
	 * %rdx start address
	 * %rcx preserve_context
	 * %r8  sme_active
	 */

	/* Save the CPU context, used for jumping back */
	pushq %rbx
	pushq %rbp
	pushq %r12
	pushq %r13
	pushq %r14
	pushq %r15
	pushf       // 将标志寄存器的值压栈

	movq	PTR(VA_CONTROL_PAGE)(%rsi), %r11 //control page 的虚拟地址放到 %r11
	movq	%rsp, RSP(%r11) //当前 %rsp 存入 control page 的数据区
	movq	%cr0, %rax      //%cr0 的值无法直接传输到内存，需借助寄存器中转
	movq	%rax, CR0(%r11) //当前 %cr0 存入 control page 的数据区
	movq	%cr3, %rax
	movq	%rax, CR3(%r11) //当前 %cr3 存入 control page 的数据区
	movq	%cr4, %rax
	movq	%rax, CR4(%r11) //当前 %cr4 存入 control page 的数据区

	/* Save CR4. Required to enable the right paging mode later. */
	movq	%rax, %r13      //当前 %cr4 存一份到 %r13

	/* zero out flags, and disable interrupts */
	pushq $0  //将立即数 0 压栈
	popfq     //从栈中弹出数据，加载到标志寄存器；
	//刚才压入了 0，所以这里相当于把标志寄存器清零了，包括 IF (Interrupt enable flag) 标志位，所以效果就是关中断
	/* Save SME active flag */
	movq	%r8, %r12  // 第五个参数 sme_active() 传入 %r12
	//因为要切换页表了，所以先把需要的关键信息从内存中存到寄存器里
	/*
	 * get physical address of control page now
	 * this is impossible after page table switch
	 */
	movq	PTR(PA_CONTROL_PAGE)(%rsi), %r8 //control page 的物理地址放到 %r8

	/* get physical address of page table now too */
	movq	PTR(PA_TABLE_PAGE)(%rsi), %r9 //恒等映射页表的物理地址放到 %r9

	/* get physical address of swap page now */
	movq	PTR(PA_SWAP_PAGE)(%rsi), %r10 //swap page 的物理地址放到 %r10

	/* save some information for jumping back */
	movq	%r9, CP_PA_TABLE_PAGE(%r11) //copy 恒等映射页表的物理地址到 control page 的数据区
	movq	%r10, CP_PA_SWAP_PAGE(%r11) //copy swap page 的物理地址到 control page 的数据区
	movq	%rdi, CP_PA_BACKUP_PAGES_MAP(%r11) //copy 页映射备份的起始物理地址到 control page 的数据区

	/* Switch to the identity mapped page tables */
	movq	%r9, %cr3 //切换页表到恒等映射的页表，和当前运行内核的页表说 byebye 了
	//虽然 ret 指令前 %rip 还是高地址段，但 CPU 实际上是从映射到了 control page 中的物理地址去取指令了
	/* setup a new stack at the end of the physical control page */
	lea	PAGE_SIZE(%r8), %rsp //设置 control page 的页结束的物理地址为新的栈底

	/* jump to identity mapped page */
	addq	$(identity_mapped - relocate_kernel), %r8 //control page 的起始 PA + relocate_kernel 的长度 = identity_mapped 例程的物理地址
	pushq	%r8 //identity_mapped 例程的物理地址入新的栈
	ret         //相当于把 identity_mapped 例程的物理地址弹出到 %rip，调用 identity_mapped 例程，这一跳转，%rip 里的值就变成恒等映射的虚拟地址了
SYM_CODE_END(relocate_kernel)

SYM_CODE_START_LOCAL_NOALIGN(identity_mapped)
	UNWIND_HINT_EMPTY
	/* set return address to 0 if not preserving context */
	pushq	$0
	/* store the start address on the stack */
	pushq   %rdx // kimage->start 为重定位到 “Crash kernel” region 后的 purgatory_start 例程的物理地址

	/*
	 * Set cr0 to a known state:   设置 cr0 控制寄存器为一个已知状态
	 *  - Paging enabled           置 PG 位，开启分页机制
	 *  - Alignment check disabled 清 AM 位，关闭对齐检查
	 *  - Write protect disabled   清 WP 位，关闭写保护，允许超级用户程序（例如特权级 0 的程序）向用户级只读页面执行写操作
	 *  - No task switch           清 TS 位，见下面的详解
	 *  - Don't do FP software emulation. 清 EM 位，表示系统有协处理器，不要软件来模拟浮点指令
	 *  - Protected mode enabled   置 PE（Protedted Enable）位，启动保护模式（PE = 0，则在实模式下运行）
	 */
	movq	%cr0, %rax
	andq	$~(X86_CR0_AM | X86_CR0_WP | X86_CR0_TS | X86_CR0_EM), %rax
	orl	$(X86_CR0_PG | X86_CR0_PE), %eax //开启分页机制，PE 和 PG 标志都要置位
	movq	%rax, %cr0                   //载入定制好的 cr0 状态

	/*
	 * Set cr4 to a known state:
	 *  - physical address extension enabled
	 *  - 5-level paging, if it was enabled before
	 */
	movl	$X86_CR4_PAE, %eax
	testq	$X86_CR4_LA57, %r13 //旧 %cr4 曾存了一份到 %r13，测试原来 5 级页表是否开启？
	jz	1f                      //未曾开启 5 级页表，跳转到标号 1
	orl	$X86_CR4_LA57, %eax     //曾开启了 5 级页表，保持开启
1:
	movq	%rax, %cr4          //载入定制好的 cr4 状态控制寄存器

	jmp 1f                      //这个跳转大有玄机，见下面详解
1:

	/* Flush the TLB (needed?) */
	movq	%r9, %cr3 //通过再次加载恒等映射页表到 %cr3 的方式刷新 TLB，不想用处理器相关指令

	/*
	 * If SME is active, there could be old encrypted cache line
	 * entries that will conflict with the now unencrypted memory
	 * used by kexec. Flush the caches before copying the kernel.
	 */
	testq	%r12, %r12 //测试第五个参数 sme_active()
	jz 1f              //未激活，跳转到标号 1
	wbinvd
1:

	movq	%rcx, %r11 //image->preserve_context 传到 %r11
	call	swap_pages //调用 swap_pages 例程

	/*
	 * To be certain of avoiding problems with self-modifying code
	 * I need to execute a serializing instruction here.
	 * So I flush the TLB by reloading %cr3 here, it's handy,
	 * and not processor dependent.
	 */
	movq	%cr3, %rax
	movq	%rax, %cr3 //刷 TLB，不想用处理器相关的指令

	/*
	 * set all of the registers to known values
	 * leave %rsp alone
	 */

	testq	%r11, %r11 //%r11 存的是 image->preserve_context 的值
	jnz 1f             //想保留上下文，向前跳到标号 1，跳过以下寄存器清零操作
	xorl	%eax, %eax
	xorl	%ebx, %ebx
	xorl    %ecx, %ecx
	xorl    %edx, %edx
	xorl    %esi, %esi
	xorl    %edi, %edi
	xorl    %ebp, %ebp
	xorl	%r8d, %r8d
	xorl	%r9d, %r9d
	xorl	%r10d, %r10d
	xorl	%r11d, %r11d
	xorl	%r12d, %r12d
	xorl	%r13d, %r13d
	xorl	%r14d, %r14d
	xorl	%r15d, %r15d
	//还记得上面曾把重定位到 “Crash kernel” region 后的 purgatory_start 例程的物理地址放到栈上了吗？
	ret //通过这条指令我们跳到了 “Crash kernel” region 里的当时用户态构造好的 purgatory_start

1:
	popq	%rdx
	leaq	PAGE_SIZE(%r10), %rsp
	ANNOTATE_RETPOLINE_SAFE
	call	*%rdx

	/* get the re-entry point of the peer system */
	movq	0(%rsp), %rbp
	leaq	relocate_kernel(%rip), %r8
	movq	CP_PA_SWAP_PAGE(%r8), %r10
	movq	CP_PA_BACKUP_PAGES_MAP(%r8), %rdi
	movq	CP_PA_TABLE_PAGE(%r8), %rax
	movq	%rax, %cr3
	lea	PAGE_SIZE(%r8), %rsp
	call	swap_pages
	movq	$virtual_mapped, %rax
	pushq	%rax
	ret
SYM_CODE_END(identity_mapped)

SYM_CODE_START_LOCAL_NOALIGN(virtual_mapped)
	UNWIND_HINT_EMPTY
	movq	RSP(%r8), %rsp
	movq	CR4(%r8), %rax
	movq	%rax, %cr4
	movq	CR3(%r8), %rax
	movq	CR0(%r8), %r8
	movq	%rax, %cr3
	movq	%r8, %cr0
	movq	%rbp, %rax

	popf
	popq	%r15
	popq	%r14
	popq	%r13
	popq	%r12
	popq	%rbp
	popq	%rbx
	ret
SYM_CODE_END(virtual_mapped)

	/* Do the copies */
SYM_CODE_START_LOCAL_NOALIGN(swap_pages)
	UNWIND_HINT_EMPTY
	movq	%rdi, %rcx 	/* Put the page_list in %rcx */ //image->head 传到 %rcx
	xorl	%edi, %edi  //清零 %edi
	xorl	%esi, %esi  //清零 %esi，即已失效的 page_list 虚拟地址
	jmp	1f              //向前跳到标号 1

0:	/* top, read another word for the indirection page */

	movq	(%rbx), %rcx
	addq	$8,	%rbx
1:
	testb	$0x1,	%cl   /* is it a destination page? */ //panic case 该测试失败
	jz	2f                //向前跳到标号 2
	movq	%rcx,	%rdi
	andq	$0xfffffffffffff000, %rdi //低 12 位清零
	jmp	0b
2:
	testb	$0x2,	%cl   /* is it an indirection page? */ //panic case 该测试失败
	jz	2f                //向前跳到下一个标号 2
	movq	%rcx,   %rbx
	andq	$0xfffffffffffff000, %rbx
	jmp	0b
2:
	testb	$0x4,	%cl   /* is it the done indicator? */ //panic case 该测试成功，回忆 kimage_terminate()
	jz	2f                //测试失败会向前跳到下一个标号 2
	jmp	3f                //测试成功向前跳到标号 3
2:
	testb	$0x8,	%cl   /* is it the source indicator? */
	jz	0b	      /* Ignore it otherwise */
	movq	%rcx,   %rsi  /* For ever source page do a copy */
	andq	$0xfffffffffffff000, %rsi

	movq	%rdi, %rdx
	movq	%rsi, %rax

	movq	%r10, %rdi
	movl	$512, %ecx
	rep ; movsq

	movq	%rax, %rdi
	movq	%rdx, %rsi
	movl	$512, %ecx
	rep ; movsq

	movq	%rdx, %rdi
	movq	%r10, %rsi
	movl	$512, %ecx
	rep ; movsq

	lea	PAGE_SIZE(%rax), %rsi
	jmp	0b
3:
	ret     //返回 identity_mapped 例程
SYM_CODE_END(swap_pages)

	.globl kexec_control_code_size
.set kexec_control_code_size, . - relocate_kernel
```

* 关于`CR0`的`TS`位
> `TS`：`CR0`的 bit `3`是 **任务已切换（Task Switched）标志**。该标志用于推迟保存任务切换时的协处理器内容，直到新任务开始实际执行协处理器指令。处理器在每次任务切换时都会设置该标志，并且在执行协处理器指令时测试该标志。
> 如果设置了`TS`标志并且`CR0`的`EM`标志为`0`，那么在执行任何协处理器指令之前会产生一个 *设备不存在异常*。
> 如果设置了`TS`标志但没有设置`CR0`的`MP`和`EM`标志，那么在执行协处理器指令`WAIT/FWAIT`之前不会产生 *设备不存在异常*。
> 如果设置了`EM`标志，那么`TS`标志对协处理器指令的执行无影响。
>
> 在任务切换时，处理器并不自动保存协处理器的上下文，而是会设置`TS`标志。
> 这个标志会使得处理器在执行新任务指令流的任何时候遇到一条协处理器指令时产生 *设备不存在异常*。*设备不存在异常* 的处理程序可使用`CLTS`指令清除`TS`标志，并且保存协处理器的上下文。
> 如果任务从没有使用过协处理器，那么相应协处理器上下文就不用保存。

* 这里面多次先把要调用的函数`push`到栈上，再用`ret`指令跳转的方式完成函数调用，而不是用`call`指令。我能想到的原因是`call`指令需要把下一条指令压栈作为将来的返回地址，然而我们这用`ret`调用的方式大多是不需要返回的，所以用这样的方式能保持栈的干净。而为什么不是`jmp`指令呢？是因为要跳转的地址都是动态的吗？

#### 向前跳转 `jmp 1f` 到下一字节
* 在 `identity_mapped` 例程中有一个怪异的跳转，向前跳转 `jmp 1f` 到下一字节
```asm
SYM_CODE_START_LOCAL_NOALIGN(identity_mapped)
...
	jmp 1f
1:
```
* Andrew Cooper 在 [这里](https://lore.kernel.org/lkml/55bc0649-c017-49ab-905d-212f140a403f@citrix.com/) 对它进行了解释：

> * `jmp 1f` 可以追溯到古老的 8086，它开创了指令指针只是 ISA 想象出来的趋势[^1]
> * 硬件维护指向下一个要获取的字节的指针（预取队列最多 `6` 个字节），并且有一个微操作可以从累加器中减去预取队列的当前长度。
> * 在那些日子里，预取队列与主内存不一致，并且跳转（指令流中的不连续性）只是刷新了预取队列。
> * 在修改可执行代码后，这是必要的，因为否则你最终可能会执行预取队列中的陈旧的字节，然后执行非陈旧的字节。（也称为区分 8086 和 8088 的方法，因为后者只有 `4` 字节预取队列。）
> * 无论如何。这是在该术语进入体系结构之前你用来拼写“序列化操作”的方式。Linux 仍然支持 Pentium 之前的 CPU，因此仍然需要关注 486 中的预取队列。
> * 但是，此示例似乎是 64 位代码，并且跟在一个对 `CR4` 的写入之后，这将完全序列化，因此它可能是从 32 位代码复制粘贴的，原则上这是必要的。

* [^1][8086 专利](https://patents.google.com/patent/US4449184A) 描述了 8086 中的程序计数器如何不保存“真实”值：

> `PC` is not a real or true program counter in that it does not, nor does any other register within CPU, maintain the actual execution point at any time. `PC` actually points to the next byte to be input into queue. The real program counter is calculated by instruction whenever a relative `jump` or `call` is required by subtracting the number of accessed instructions still remaining unused in queue from `PC`.

## vmcoreinfo
### 分配 vmcoreinfo 空间
* `vmcoreinfo`通过`subsys_initcall(crash_save_vmcoreinfo_init)`调用时分配，分为两部分
  * `vmcoreinfo_data`：用来存储收集到的数据
  * `vmcoreinfo_note`：用来提供`/sys/kernel/vmcoreinfo`，kexec 根据里面提供的信息构造 elfcorehdr 中存放 vmcoreinfo 的类型为`PT_NOTE`的 program header
    * kernel/ksysfs.c:`vmcoreinfo_show()`
* kernel/crash_core.c
```c
static void update_vmcoreinfo_note(void)
{
    u32 *buf = vmcoreinfo_note;

    if (!vmcoreinfo_size)
        return;
    buf = append_elf_note(buf, VMCOREINFO_NOTE_NAME, 0, vmcoreinfo_data,
                  vmcoreinfo_size);
    final_note(buf);
}

static int __init crash_save_vmcoreinfo_init(void)
{
    vmcoreinfo_data = (unsigned char *)get_zeroed_page(GFP_KERNEL);
    if (!vmcoreinfo_data) {
        pr_warn("Memory allocation for vmcoreinfo_data failed\n");
        return -ENOMEM;
    }

    vmcoreinfo_note = alloc_pages_exact(VMCOREINFO_NOTE_SIZE,
                        GFP_KERNEL | __GFP_ZERO);
    if (!vmcoreinfo_note) {
        free_page((unsigned long)vmcoreinfo_data);
        vmcoreinfo_data = NULL;
        pr_warn("Memory allocation for vmcoreinfo_note failed\n");
        return -ENOMEM;
    }

    VMCOREINFO_OSRELEASE(init_uts_ns.name.release);
    VMCOREINFO_BUILD_ID();
    VMCOREINFO_PAGESIZE(PAGE_SIZE);

    VMCOREINFO_SYMBOL(init_uts_ns);
    VMCOREINFO_OFFSET(uts_namespace, name);
    VMCOREINFO_SYMBOL(node_online_map);
#ifdef CONFIG_MMU
    VMCOREINFO_SYMBOL_ARRAY(swapper_pg_dir);
#endif
    VMCOREINFO_SYMBOL(_stext);
    VMCOREINFO_SYMBOL(vmap_area_list);

#ifndef CONFIG_NUMA
    VMCOREINFO_SYMBOL(mem_map);
    VMCOREINFO_SYMBOL(contig_page_data);
#endif
#ifdef CONFIG_SPARSEMEM
    VMCOREINFO_SYMBOL_ARRAY(mem_section);
    VMCOREINFO_LENGTH(mem_section, NR_SECTION_ROOTS);
    VMCOREINFO_STRUCT_SIZE(mem_section);
    VMCOREINFO_OFFSET(mem_section, section_mem_map);
    VMCOREINFO_NUMBER(SECTION_SIZE_BITS);
    VMCOREINFO_NUMBER(MAX_PHYSMEM_BITS);
#endif
    VMCOREINFO_STRUCT_SIZE(page);
    VMCOREINFO_STRUCT_SIZE(pglist_data);
    VMCOREINFO_STRUCT_SIZE(zone);
    VMCOREINFO_STRUCT_SIZE(free_area);
    VMCOREINFO_STRUCT_SIZE(list_head);
    VMCOREINFO_SIZE(nodemask_t);
    VMCOREINFO_OFFSET(page, flags);
    VMCOREINFO_OFFSET(page, _refcount);
    VMCOREINFO_OFFSET(page, mapping);
    VMCOREINFO_OFFSET(page, lru);
    VMCOREINFO_OFFSET(page, _mapcount);
    VMCOREINFO_OFFSET(page, private);
    VMCOREINFO_OFFSET(page, compound_dtor);
    VMCOREINFO_OFFSET(page, compound_order);
    VMCOREINFO_OFFSET(page, compound_head);
    VMCOREINFO_OFFSET(pglist_data, node_zones);
    VMCOREINFO_OFFSET(pglist_data, nr_zones);
#ifdef CONFIG_FLATMEM
    VMCOREINFO_OFFSET(pglist_data, node_mem_map);
#endif
    VMCOREINFO_OFFSET(pglist_data, node_start_pfn);
    VMCOREINFO_OFFSET(pglist_data, node_spanned_pages);
    VMCOREINFO_OFFSET(pglist_data, node_id);
    VMCOREINFO_OFFSET(zone, free_area);
    VMCOREINFO_OFFSET(zone, vm_stat);
    VMCOREINFO_OFFSET(zone, spanned_pages);
    VMCOREINFO_OFFSET(free_area, free_list);
    VMCOREINFO_OFFSET(list_head, next);
    VMCOREINFO_OFFSET(list_head, prev);
    VMCOREINFO_OFFSET(vmap_area, va_start);
    VMCOREINFO_OFFSET(vmap_area, list);
    VMCOREINFO_LENGTH(zone.free_area, MAX_ORDER);
    log_buf_vmcoreinfo_setup();
    VMCOREINFO_LENGTH(free_area.free_list, MIGRATE_TYPES);
    VMCOREINFO_NUMBER(NR_FREE_PAGES);
    VMCOREINFO_NUMBER(PG_lru);
    VMCOREINFO_NUMBER(PG_private);
    VMCOREINFO_NUMBER(PG_swapcache);
    VMCOREINFO_NUMBER(PG_swapbacked);
    VMCOREINFO_NUMBER(PG_slab);
#ifdef CONFIG_MEMORY_FAILURE
    VMCOREINFO_NUMBER(PG_hwpoison);
#endif
    VMCOREINFO_NUMBER(PG_head_mask);
#define PAGE_BUDDY_MAPCOUNT_VALUE   (~PG_buddy)
    VMCOREINFO_NUMBER(PAGE_BUDDY_MAPCOUNT_VALUE);
#ifdef CONFIG_HUGETLB_PAGE
    VMCOREINFO_NUMBER(HUGETLB_PAGE_DTOR);
#define PAGE_OFFLINE_MAPCOUNT_VALUE (~PG_offline)
    VMCOREINFO_NUMBER(PAGE_OFFLINE_MAPCOUNT_VALUE);
#endif

    arch_crash_save_vmcoreinfo(); //arch 相关的信息
    update_vmcoreinfo_note();

    return 0;
}

subsys_initcall(crash_save_vmcoreinfo_init);
```

### x86 相关的 vmcoreinfo
* arch/x86/kernel/crash_core_64.c
```c
void arch_crash_save_vmcoreinfo(void)
{
    u64 sme_mask = sme_me_mask;

    VMCOREINFO_NUMBER(phys_base);
    VMCOREINFO_SYMBOL(init_top_pgt);
    vmcoreinfo_append_str("NUMBER(pgtable_l5_enabled)=%d\n",
                  pgtable_l5_enabled());

#ifdef CONFIG_NUMA
    VMCOREINFO_SYMBOL(node_data);
    VMCOREINFO_LENGTH(node_data, MAX_NUMNODES);
#endif
    vmcoreinfo_append_str("KERNELOFFSET=%lx\n", kaslr_offset());
    VMCOREINFO_NUMBER(KERNEL_IMAGE_SIZE);
    VMCOREINFO_NUMBER(sme_mask);
}
```

## Core Files
### /proc/kcore
* 提供一个 live kernel 的虚拟的 ELF core 文件，可以用 gdb，crash-utility，readelf 等 ELF 工具读取
* 内核选项`CONFIG_PROC_KCORE`
* 通过`fs_initcall(proc_kcore_init)`初始化
* fs/proc/kcore.c:`read_kcore()`提供内容

### /proc/vmcore
* 通过`fs_initcall(vmcore_init)`初始化
* fs/proc/vmcore.c:`__read_vmcore()`提供内容
* 读取旧系统的内存时用的是`read_from_oldmem()`，它会调用`copy_oldmem_page()`，里面会用`ioremap_cache(pfn << PAGE_SHIFT, PAGE_SIZE)`将指定的页面映射到捕捉内核，然后`copy_to_user()`就可以了

### Crash notes
* 当系统崩溃时存储 CPU states 的 Per CPU 内存
* 通过`subsys_initcall(crash_notes_memory_init)`给 Per CPU 数据结构`note_buf_t __percpu *crash_notes`分配空间
* 通过 drivers/base/cpu.c 导出文件
  * `static DEVICE_ATTR_ADMIN_RO(crash_notes)`导出文件`/sys/devices/system/cpu/cpu%d/crash_notes`，指示 Per CPU `crash_notes[i]`的地址
  * `static DEVICE_ATTR_ADMIN_RO(crash_notes_size)`导出文件`/sys/devices/system/cpu/cpu%d/crash_notes_size`，指示 Per CPU `crash_notes[i]`的大小
* kexec 构造 elfcorehdr 的时候会分别读取每个 CPU 以上两个文件的信息，放入`PT_NOTE`类型的多个 program headers
* Panic 时通过`crash_save_cpu()`填充数据 crash CPU 的 `prstatus`，`pid`，寄存器等
* 创建`/proc/vmcore`的时候会合并成一个`PT_NOTE`类型的 program header（fs/proc/vmcore.c:`parse_crash_elf64_headers()`）

## References
* [Org (Using as)](https://sourceware.org/binutils/docs/as/Org.html)