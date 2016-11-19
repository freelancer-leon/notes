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
