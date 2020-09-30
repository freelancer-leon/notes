* RCU是基于其原理命名的，Read-Copy Update
  * **Read** 指的是对于被 RCU 保护的共享数据，reader 可以直接访问，不需要获得任何锁；
  * **Copy Update** 指的是 writer 修改数据前首先拷贝一个副本，然后在副本上进行修改，修改完毕后向reclaimer(垃圾回收器)注册一个回调函数(callback)，在适当的时机完成真正的修改操作 – 把原数据的指针重新指向新的被修改的数据，
  * 这里所说的适当的时机就是当既有的reader全都退出临界区的时候，而等待恰当时机的过程被称为 **grace period**。
* 在RCU机制中，writer 不需要和 reader 竞争任何锁，只在有多个 writer 的情况下它们之间需要某种锁进行同步作，如果写操作频繁的话RCU的性能会严重下降，所以 RCU 只适用于读多写少的情况。

* /sys/module/rcupdate/parameters/rcu_cpu_stall_suppress
* /sys/module/rcupdate/parameters/rcu_cpu_stall_timeout

# 导致 RCU Warning 的原因
* CPU 在 RCU read-side 临界区内忙循环
* CPU 在屏蔽中断的情况下忙循环
* CPU 在关闭抢占的情况下忙循环
  * 如果 *ksoftirqd* 在使用，会导致 RCU-bh stall
* CPU 在关闭下半部的情况下忙循环
  * 导致 RCU-sched 或 RCU-bh stall
* 对于 `!CONFIG_PREEMPT` 的内核，在内核中任何没有调用 `schedule()` 的忙循环
  * 如果该循环确实是期望和需要的行为，可能需要加一些 `cond_resched()`的调用
* Linux引导过程中输出信息很多，但是所用的 console 太慢，跟不上信息输出的速度
* 任何可能导致 RCU grace-period 内核线程无法运行的因素
  * 会导致 "All QSes seen" console-log 信息。该消息包含 kthread 最后一次运行的信息和预计应该多久运行一次的信息
  * 会导致 "rcu_.*kthread starved for" 的 *console-log* 消息和额外的调试信息。
* 对于 `CONFIG_PREEMPT` 的内核，CPU 绑定的优先级很高的实时任务占着 CPU 不放，抢占了恰好处于 RCU read-side 临界区的低优先级任务
  * 特别是当低优先级任务无法迁移至其他 CPU 时，下一个 RCU grace period 绝对不会完成
  * 最终导致系统内存耗尽和挂死
* 对于 `CONFIG_PREEMPT_RT` 的内核，运行着一个比 RCU softirq 线程优先级还高的实时进程
  * 这将阻止 RCU 回调被调用，并且在 `CONFIG_PREEMPT_RCU` 内核中将进一步阻止 *RCU宽限期（grace periods）* 完成
  * 无论哪种方式，系统最终会因内存耗尽而挂死
* 一个周期性中断的中断处理函数所用的时间比连续两个中断的时间间隔的时间还长
  * 这会阻止 RCU 的内核线程和 softirq 处理函数的运行
  * 注意一些高开销的 debug 选项，比如 function_graph tracer，开启后导致中断处理函数的运行时间比平时长
* 在快速系统上测试工作负载，将 stall-warning 超时调整为几乎不能避免 RCU CPU stall warning，然后在慢速系统上以相同的 stall-warning 超时运行相同的工作负载
* 硬件或软件问题会关闭未处于 dyntick-idle 模式的 CPU 上的 scheduler-clock 中断
  * 这个问题确实发生了，似乎最有可能导致 `CONFIG_NO_HZ_COMMON = n` 内核的 RCU CPU stall-warning。
* RCU代码bug
* 硬件故障

# References
- [Linux Performance - RCU CPU STALL DETECTOR](http://linuxperf.com/?p=125)
- [kernel doc - Using RCU's CPU Stall Detector](https://www.kernel.org/doc/Documentation/RCU/stallwarn.txt)
