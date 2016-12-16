# 调试

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

  ```
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
  ```
  echo 5 > /proc/sys/kernel/printk_ratelimit        # Wait sec before re-open printk
  echo 10 > /proc/sys/kernel/printk_ratelimit_burst # Message number before limiting the rate
  ```

## log

### loglevel

```
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
```

### logbuffer
```
log_buf_len=n[KMG]  Sets the size of the printk ring buffer,
        in bytes.  n must be a power of two and greater
        than the minimal size. The minimal size is defined
        by LOG_BUF_SHIFT kernel config parameter. There is
        also CONFIG_LOG_CPU_MAX_BUF_SHIFT config parameter
        that allows to increase the default size depending on
        the number of CPUs. See init/Kconfig for more details.
```

## sysrq

* 开启sysrq特性
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

## git二分法查找
```
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
