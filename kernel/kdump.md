# kdump

## /proc/vmcore
* kexec 在加载捕获内核时，所有关于系统内核的转储镜像的必要信息会被编码成 ELF 格式，计算并生成一个 ELF core header，并且在 crash 前存储在一个内存的保留区域。
* ELF header 起始的物理地址通过`elfcorehdr=`内核命令行参数传递给捕捉内核，捕获内核根据从`elfcorehdr`接收到的系统内存信息创建 vmcore
  * 实现见 kernel/crash_dump.c 的源代码，`early_param("elfcorehdr", setup_elfcorehdr)`
* 对于 ARM64 来说作法有些特殊
  * kexec 在`load_crashdump_segments()`构造用于捕捉内核的`/proc/vmcore`的 ELF core header
  * 在 `setup_2nd_dtb()` 这个步骤通过修改 dtb 传递捕捉内核的命令行参数，此时会添加一个`linux,elfcorehdr`的 dtb 节点用于描述 ELF core header
  ```c
  main()
  -> my_load()
     -> file_type[i].load(argc, argv, kernel_buf, kernel_size, &info)
     => image_arm64_load() // -t Image
        -> if (info->kexec_flags & KEXEC_ON_CRASH)
              load_crashdump_segments() //创建和初始化 ELF core header segment
        -> arm64_load_other_segments(info, kernel_segment + arm64_mem.text_offset)
           -> setup_2nd_dtb(&dtb, command_line, info->kexec_flags & KEXEC_ON_CRASH)
  ```
* 在内核启动时，根据 dtb 里的信息留出用于存放 ELF core header 的 memblock
  ```c
  arch/arm64/kernel/setup.c
  setup_arch()
     arch/arm64/mm/init.c
     -> arm64_memblock_init()
        -> reserve_crashkernel()
        -> reserve_elfcorehdr()
           -> of_scan_flat_dt(early_init_dt_scan_elfcorehdr, NULL)
              -> early_init_dt_scan_elfcorehdr()
                 -> of_get_flat_dt_prop(node, "linux,elfcorehdr", &len)
                 -> elfcorehdr_addr = dt_mem_next_cell(dt_root_addr_cells, &reg)
                 -> elfcorehdr_size = dt_mem_next_cell(dt_root_size_cells, &reg)
           -> memblock_reserve(elfcorehdr_addr, elfcorehdr_size)
           -> pr_info("Reserving %lldKB of memory at 0x%llx for elfcorehdr\n", elfcorehdr_size >> 10, elfcorehdr_addr)
  ```
* 该过程中会更新全局变量`unsigned long long elfcorehdr_addr`的信息，该变量存储 crash 转储镜像的 ELF header 的物理地址
  * kernel/crash_dump.c
  ```c
  /*
   * stores the physical address of elf header of crash image
   *
   * Note: elfcorehdr_addr is not just limited to vmcore. It is also used by
   * is_kdump_kernel() to determine if we are booting after a panic. Hence put
   * it under CONFIG_CRASH_DUMP and not CONFIG_PROC_VMCORE.
   */
  unsigned long long elfcorehdr_addr = ELFCORE_ADDR_MAX;
  EXPORT_SYMBOL_GPL(elfcorehdr_addr);

  /*
   * stores the size of elf header of crash image
   */
  unsigned long long elfcorehdr_size;

  /*
   * elfcorehdr= specifies the location of elf core header stored by the crashed
   * kernel. This option will be passed by kexec loader to the capture kernel.
   *
   * Syntax: elfcorehdr=[size[KMG]@]offset[KMG]
   */
  static int __init setup_elfcorehdr(char *arg)
  {
      char *end;
      if (!arg)
          return -EINVAL;
      elfcorehdr_addr = memparse(arg, &end);
      if (*end == '@') {
          elfcorehdr_size = elfcorehdr_addr;
          elfcorehdr_addr = memparse(end + 1, &end);
      }   
      return end > arg ? 0 : -EINVAL;
  }
  early_param("elfcorehdr", setup_elfcorehdr);
  ...*```
  ```
* 在`fs_initcall(vmcore_init)`的时候会根据`elfcorehdr_addr`的信息来决定要不要创建`/proc/vmcore`
  * include/linux/crash_dump.h
  ```c
  #ifdef CONFIG_CRASH_DUMP
  #define ELFCORE_ADDR_MAX    (-1ULL)
  #define ELFCORE_ADDR_ERR    (-2ULL)
  /*
   * is_kdump_kernel() checks whether this kernel is booting after a panic of
   * previous kernel or not. This is determined by checking if previous kernel
   * has passed the elf core header address on command line.
   *
   * This is not just a test if CONFIG_CRASH_DUMP is enabled or not. It will
   * return true if CONFIG_CRASH_DUMP=y and if kernel is booting after a panic
   * of previous kernel.
   */

  static inline bool is_kdump_kernel(void)
  {
      return elfcorehdr_addr != ELFCORE_ADDR_MAX;
  }

  /* is_vmcore_usable() checks if the kernel is booting after a panic and
   * the vmcore region is usable.
   *
   * This makes use of the fact that due to alignment -2ULL is not
   * a valid pointer, much in the vain of IS_ERR(), except
   * dealing directly with an unsigned long long rather than a pointer.
   */

  static inline int is_vmcore_usable(void)
  {
      return is_kdump_kernel() && elfcorehdr_addr != ELFCORE_ADDR_ERR ? 1 : 0;
  }
  ...*```
  ```
  * fs/proc/vmcore.c
  ```c
  /* Init function for vmcore module. */
  static int __init vmcore_init(void)
  {
      int rc = 0;

      /* Allow architectures to allocate ELF header in 2nd kernel */
      rc = elfcorehdr_alloc(&elfcorehdr_addr, &elfcorehdr_size);
      if (rc)
          return rc;
      /*
       * If elfcorehdr= has been passed in cmdline or created in 2nd kernel,
       * then capture the dump.
       */
      if (!(is_vmcore_usable())) //是否创建 /proc/vmcore 需要检查 elfcorehdr_addr
          return rc;
      rc = parse_crash_elf_headers();
      if (rc) {
          pr_warn("Kdump: vmcore not initialized\n");
          return rc;
      }
      elfcorehdr_free(elfcorehdr_addr);
      elfcorehdr_addr = ELFCORE_ADDR_ERR;

      proc_vmcore = proc_create("vmcore", S_IRUSR, NULL, &proc_vmcore_operations);
      if (proc_vmcore)
          proc_vmcore->size = vmcore_size;
      return 0;
  }
  fs_initcall(vmcore_init);
  ...*```
  ```
  * 是否创建`/proc/vmcore`需要检查`elfcorehdr_addr`，间接地判断当前内核是系统内核还是捕捉内核
* 对于 crash 场景的 kexec 的过程
```c
kernel/panic.c
panic()
   kernel/kexec_core.c
-> __crash_kexec()
   -> machine_kexec(kexec_crash_image)
```
## makedumpfile
* `makedumpfile`需要系统内核的调试信息来分析系统内核如何使用内存的，从而区分非必须的页，这些信息来自`vmlinux`或者`vmcoreinfo`
* **vmcore** 也即 `/proc/vmcore`，系统内核的内存转储
* **vmcoreinfo** 包含系统内核的信息（结构大小、域偏移等），并且足够小到可以放入捕捉内核的 initrd
  * 如果捕捉内核运行在自己的 initrd 里，而不是挂载根文件系统，`makedumpfile`不能参考`vmlinux`，因为捕获内核的 initrd 是需要常驻内存的，因此不适合放入类似`vmlinux`这样的大文件
  * 为解决该问题，`makedumpfile`需要事先做好`vmcoreinfo`。当捕捉内核运行时，如果需要系统内核的信息就从`vmcoreinfo`得到就可以了
  * 从 linux-2.6.24 起，`vmcore`中就包含`vmcoreinfo`
* **-x VMLINUX**
  * 指定用系统内核带调试信息的`vmlinux`来分析系统内核的内存使用
  * 如果`vmcore`中不包含`vmcoreinfo`且`-i VMCOREINFO`也未指定，`dump_level`大于等于 2，该选项是必须的
  * 系统内核和捕获内核的页大小需匹配
  * `# makedumpfile -d 31 -x vmlinux /proc/vmcore dumpfile`
* **-i VMCOREINFO**
  * 指定用`vmcoreinfo`替代`vmlinux`来分析系统内核的内存使用
  * `vmcoreinfo`需要用`makedumpfile`的`-g`选项事先生成
  * 如果`vmcore`中不包含`vmcoreinfo`且`-x VMLINUX`也未指定，`dump_level`大于等于 2，该选项是必须的
  * `# makedumpfile -d 31 -i vmcoreinfo /proc/vmcore dumpfile`
* **-g VMCOREINFO**
  * 用系统内核带调试信息的`vmlinux`来生成`vmcoreinfo`
  * `vmcoreinfo`必须在运行系统内核的系统上生成
  * 用户可以用`-i`选项指定在其他系统上生成的`vmcoreinfo`作为有着相同内核的`makedumpfile`命令的输入
  * `# makedumpfile -g vmcoreinfo -x vmlinux`
### `elfcorehdr` vs `vmcoreinfo`
#### elfcorehdr
* *捕捉内核* 根据`elfcorehdr`提供的信息创建`/proc/vmcore`，提供给`crash`或`gdb`等调试工具使用
* 多个来源
#### vmcoreinfo
* `makedumpfile`根据`vmcoreinfo`的信息，在生成`vmcore`时根据使用情况而去除不必要的页
* 提取自系统内核的`vmlinux`

# References
- [linuxperf - KDUMP简介](http://linuxperf.com/?p=172)
