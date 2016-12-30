# Ftrace

* 设定要跟踪的函数
```
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo function > /sys/kernel/debug/tracing/current_tracer
echo do_page_fault > /sys/kernel/debug/tracing/current_tracer
```

* 指定跟踪特定的CPU
```
echo "cpu-id" > /sys/kernel/debug/tracing/tracing_cpumask
```

* 开始跟踪
```
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe | tee /tmp/ftrace.log
```

* 输出示例
```
            TASK-PID CPU irqs-off need-resched hardirq/softirq preempt-depth delay TIMESTAMP FUNCTION
            lttng-879   [000] d..1 2437395.004331: do_page_fault
<-do_PrefetchAbort
            lttng-879   [000] d..1 2437395.004331: do_page_fault
<-do_PrefetchAbort
            lttng-879   [000] d..1 2437395.004346: do_page_fault
<-do_DataAbort
            lttng-879   [000] d..1 2437395.004362: do_page_fault
<-do_PrefetchAbort
            lttng-879   [000] d..1 2437395.004377: do_page_fault
<-do_PrefetchAbort
```

# Reference

* [ftrace 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/)
* [使用 ftrace 调试 Linux 内核，第 1 部分](http://www.ibm.com/developerworks/cn/linux/l-cn-ftrace1/)
* [使用 ftrace 调试 Linux 内核，第 2 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace2/)
* [使用 ftrace 调试 Linux 内核，第 3 部分](http://www.ibm.com/developerworks/cn/linux/l-cn-ftrace3/)
