# 调试

# 常用调试选项
## lib/Kconfig.debug
```c
config DEBUG_INFO
    bool "Compile the kernel with debug info"
    depends on DEBUG_KERNEL && !COMPILE_TEST
    help
      If you say Y here the resulting kernel image will include
      debugging info resulting in a larger kernel image.
      This adds debug symbols to the kernel and modules (gcc -g), and
      is needed if you intend to use kernel crashdump or binary object
      tools like crash, kgdb, LKCD, gdb, etc on the kernel.
      Say Y here only if you plan to debug the kernel.

      If unsure, say N.
config DEBUG_KERNEL
    bool "Kernel debugging"
    help
      Say Y here if you are developing drivers or trying to debug and
      identify kernel problems.

```
* 不开启`COINFIG_DEBUG_INFO`无法在反汇编`vmlinux`时内联 C 源代码
## init/Kconfig
```c
config KALLSYMS
    bool "Load all symbols for debugging/ksymoops" if EXPERT
    default y
    help
      Say Y here to let the kernel print out symbolic crash information and
      symbolic stack backtraces. This increases the size of the kernel
      somewhat, as all symbols have to be loaded into the kernel image.

config KALLSYMS_ALL
    bool "Include all symbols in kallsyms"
    depends on DEBUG_KERNEL && KALLSYMS
    help
      Normally kallsyms only contains the symbols of functions for nicer
      OOPS messages and backtraces (i.e., symbols from the text and inittext
      sections). This is sufficient for most cases. And only in very rare
      cases (e.g., when a debugger is used) all symbols are required (e.g.,
      names of variables from the data sections, etc).

      This option makes sure that all symbols are loaded into the kernel
      image (i.e., symbols from all sections) in cost of increased kernel
      size (depending on the kernel configuration, it may be 300KiB or
      something like this).

      Say N unless you really need all symbols.
```

# 打印（print）

## printk
* `printk()`和`early_printk()`
* 内核参数
  ```
  earlyprintk=serial,ttyS0,115200,keep keep_bootcon
  ```
* 有时可能需要同时加`,keep`和`keep_bootcon`才能打印初始化早期的log。
  ```
  Append ``",keep"`` to not disable it when the real console takes over.
  ```

  ```c
  keep_bootcon    [KNL]
          Do not unregister boot console at start. This is only
          useful for debugging when something happens in the window
          between unregistering the boot console and initializing
          the real console.
  ```

### printk的频率限制
* 用法
  ```c
  if (error && printk_ratelimit())
    printk(KERN_DEBUG “error=%d\n”, error);
  ```
* 调整
  ```sh
  echo 5 > /proc/sys/kernel/printk_ratelimit        # Wait sec before re-open printk
  echo 10 > /proc/sys/kernel/printk_ratelimit_burst # Message number before limiting the rate
  ```

## log

### loglevel

```c
loglevel=   All Kernel Messages with a loglevel smaller than the
        console loglevel will be printed to the console. It can
        also be changed with klogd or other programs. The
        loglevels are defined as follows:

        0 (KERN_EMERG)      system is unusable
        1 (KERN_ALERT)      action must be taken immediately
        2 (KERN_CRIT)       critical conditions
        3 (KERN_ERR)        error conditions
        4 (KERN_WARNING)    warning conditions
        5 (KERN_NOTICE)     normal but significant condition
        6 (KERN_INFO)       informational
        7 (KERN_DEBUG)      debug-level messages

debug       [KNL] Enable kernel debugging (events log level).
quiet       [KNL] Disable most log messages
```
* include/linux/kern_levels.h
```c
#define KERN_SOH        "\001"          /* ASCII Start Of Header */
#define KERN_SOH_ASCII  '\001'

#define KERN_EMERG      KERN_SOH "0"    /* system is unusable */
#define KERN_ALERT      KERN_SOH "1"    /* action must be taken immediately */
#define KERN_CRIT       KERN_SOH "2"    /* critical conditions */
#define KERN_ERR        KERN_SOH "3"    /* error conditions */
#define KERN_WARNING    KERN_SOH "4"    /* warning conditions */
#define KERN_NOTICE     KERN_SOH "5"    /* normal but significant condition */
#define KERN_INFO       KERN_SOH "6"    /* informational */
#define KERN_DEBUG      KERN_SOH "7"    /* debug-level messages */
#define KERN_DEFAULT    KERN_SOH "d"    /* the default kernel loglevel */
```
* include/linux/printk.h
```c
/* printk's without a loglevel use this.. */
#define MESSAGE_LOGLEVEL_DEFAULT CONFIG_MESSAGE_LOGLEVEL_DEFAULT

/* We show everything that is MORE important than this.. */
#define CONSOLE_LOGLEVEL_SILENT  0 /* Mum's the word */
#define CONSOLE_LOGLEVEL_MIN     1 /* Minimum loglevel we let people use */
#define CONSOLE_LOGLEVEL_QUIET   4 /* Shhh ..., when booted with "quiet" */
#define CONSOLE_LOGLEVEL_DEBUG  10 /* issue debug messages */
#define CONSOLE_LOGLEVEL_MOTORMOUTH 15  /* You can't shut this one up */

/*
 * Default used to be hard-coded at 7, we're now allowing it to be set from
 * kernel config.
 */
#define CONSOLE_LOGLEVEL_DEFAULT CONFIG_CONSOLE_LOGLEVEL_DEFAULT

extern int console_printk[];

#define console_loglevel (console_printk[0])
#define default_message_loglevel (console_printk[1])
#define minimum_console_loglevel (console_printk[2])
#define default_console_loglevel (console_printk[3])
```
* kernel/printk/printk.c
```c
int console_printk[4] = {
        CONSOLE_LOGLEVEL_DEFAULT,       /* console_loglevel */
        MESSAGE_LOGLEVEL_DEFAULT,       /* default_message_loglevel */
        CONSOLE_LOGLEVEL_MIN,           /* minimum_console_loglevel */
        CONSOLE_LOGLEVEL_DEFAULT,       /* default_console_loglevel */
};
```
* init/main.c
```c
static int __init debug_kernel(char *str)
{
        console_loglevel = CONSOLE_LOGLEVEL_DEBUG;
        return 0;
}

static int __init quiet_kernel(char *str)
{
        console_loglevel = CONSOLE_LOGLEVEL_QUIET;
        return 0;
}

early_param("debug", debug_kernel);
early_param("quiet", quiet_kernel);

static int __init loglevel(char *str)
{
        int newlevel;

        /*
         * Only update loglevel value when a correct setting was passed,
         * to prevent blind crashes (when loglevel being set to 0) that
         * are quite hard to debug
         */
        if (get_option(&str, &newlevel)) {
                console_loglevel = newlevel;
                return 0;
        }

        return -EINVAL;
}

early_param("loglevel", loglevel);
...*```
```
### 调整 log level

* 如果不指定 printk 打印的 log level，那么它缺省的级别是 `DEFAULT_MESSAGE_LOGLEVEL`（通常 "4"=KERN_WARNING）
* 该缺省值可以通过 `CONFIG_DEFAULT_MESSAGE_LOGLEVEL` kernel config 选项 (make menuconfig-> Kernel Hacking -> Default message log level) 调整。
* 内核打印消息的时候会比较该消息的 log level 和 `console_loglevel`（一个内核变量），当优先级高于（数值小于）`console_loglevel`，消息才会被打印
  ```
  $ cat /proc/sys/kernel/printk
  	7       4       1       7
  	current	default	minimum	boot-time-default
  ```
  * 第一个数是当前的`console_loglevel`
  * 第二个数是缺省的 log level
* 在 console 上打印所有的 kernel message
  ```
  # echo 8 > /proc/sys/kernel/printk
  ```
### logbuffer
```c
log_buf_len=n[KMG]  Sets the size of the printk ring buffer,
        in bytes.  n must be a power of two and greater
        than the minimal size. The minimal size is defined
        by LOG_BUF_SHIFT kernel config parameter. There is
        also CONFIG_LOG_CPU_MAX_BUF_SHIFT config parameter
        that allows to increase the default size depending on
        the number of CPUs. See init/Kconfig for more details.
```

## sysrq

* 开启 sysrq 特性
  ```
  echo 1 > /proc/sys/kernel/sysrq
  ```

* 查看sysrq输出
  ```
  dmesg -w
  ```
  或
  ```
  tail -f /var/log/syslog
  ```

* `SysRq`+`Alt`+对应字母键或输出对应字符到`/proc/sysrq-trigger`
  ```
  # echo h > /proc/sysrq-trigger &
  # dmesg
  ...
  Jun  5 16:23:33 hostdomain kernel: [26953.774424] SysRq : HELP : loglevel(0-9) reboot(b) crash(c) terminate-all-tasks(e) memory-full-oom-kill(f) kill-all-tasks(i) thaw-filesystems(j) sak(k) show-backtrace-all-active-cpus(l) show-memory-usage(m) nice-all-RT-tasks(n) poweroff(o) show-registers(p) show-all-timers(q) unraw(r) sync(s) show-task-states(t) unmount(u) force-fb(V) show-blocked-tasks(w) dump-ftrace-buffer(z)
  ```
* sysrq 的键映射表见 drivers/tty/sysrq.c 的`struct sysrq_key_op *sysrq_key_table[]`数组

## git二分法查找
```sh
git bisect start
git bisect bad [revision bug on]
git bisect good [revision no bug]
...
# compile and test kernel
...
git bisect good
git bisect bad
...
# indicate dir
git bisect start - arch/x86
```

# Dynamic Debug
* `CONFIG_DYNAMIC_DEBUG` (lib/Kconfig.debug)
```c
config DYNAMIC_DEBUG
    bool "Enable dynamic printk() support"
    default n
    depends on PRINTK
    depends on (DEBUG_FS || PROC_FS)
    select DYNAMIC_DEBUG_CORE
    help

      Compiles debug level messages into the kernel, which would not
      otherwise be available at runtime. These messages can then be
      enabled/disabled based on various levels of scope - per source file,
      function, module, format string, and line number. This mechanism
      implicitly compiles in all pr_debug() and dev_dbg() calls, which
      enlarges the kernel text size by about 2%.

      If a source file is compiled with DEBUG flag set, any
      pr_debug() calls in it are enabled by default, but can be
      disabled at runtime as below.  Note that DEBUG flag is
      turned on by many CONFIG_*DEBUG* options.

      Usage:

      Dynamic debugging is controlled via the 'dynamic_debug/control' file,
      which is contained in the 'debugfs' filesystem or procfs.
      Thus, the debugfs or procfs filesystem must first be mounted before
      making use of this feature.
      We refer the control file as: <debugfs>/dynamic_debug/control. This
      file contains a list of the debug statements that can be enabled. The
      format for each line of the file is:

        filename:lineno [module]function flags format

      filename : source file of the debug statement
      lineno : line number of the debug statement
      module : module that contains the debug statement
      function : function that contains the debug statement
      flags : '=p' means the line is turned 'on' for printing
      format : the format used for the debug statement

      From a live system:

        nullarbor:~ # cat <debugfs>/dynamic_debug/control
        # filename:lineno [module]function flags format
        fs/aio.c:222 [aio]__put_ioctx =_ "__put_ioctx:\040freeing\040%p\012"
        fs/aio.c:248 [aio]ioctx_alloc =_ "ENOMEM:\040nr_events\040too\040high\012"
        fs/aio.c:1770 [aio]sys_io_cancel =_ "calling\040cancel\012"

        Example usage:

          // enable the message at line 1603 of file svcsock.c
          nullarbor:~ # echo -n 'file svcsock.c line 1603 +p' >
                          <debugfs>/dynamic_debug/control

          // enable all the messages in file svcsock.c
          nullarbor:~ # echo -n 'file svcsock.c +p' >
                          <debugfs>/dynamic_debug/control

          // enable all the messages in the NFS server module
          nullarbor:~ # echo -n 'module nfsd +p' >
                          <debugfs>/dynamic_debug/control

          // enable all 12 messages in the function svc_process()
          nullarbor:~ # echo -n 'func svc_process +p' >
                          <debugfs>/dynamic_debug/control

          // disable all 12 messages in the function svc_process()
          nullarbor:~ # echo -n 'func svc_process -p' >
                          <debugfs>/dynamic_debug/control

        See Documentation/admin-guide/dynamic-debug-howto.rst for additional
        information.
```

# References
* [Debugging by printing](https://elinux.org/Debugging_by_printing)
