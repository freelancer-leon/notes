# Linux Deadline调度

- [概览](#概览)
	- [简介](#简介)
	- [概念](#概念)
	- [内核实现的CBS算法简述](#内核实现的cbs算法简述)
- [参考资料](#参考资料)

# 概览

## 简介

* 自 version 3.14 引入
* `SCHED_DEADLINE`调度策略是 **Earliest Deadline First (EDF)** 调度算法的一个基本实现
* 增加了 **Constant Bandwidth Server, CBS** 机制，让隔离任务之间彼此的行为成为可能
* `SCHED_DEADLINE`调度使用的三个参数 **runtime**、**period**、**deadline** 来调度任务
* 每`period`微秒（us），一个`SCHED_DEADLINE`可以收到`runtime`微秒的执行时间
* 这`runtime`微秒在从周期开始的`deadline`微秒之内有效
* 每次任务唤醒的时候，调度器（用 CBS 算法）计算出一个“scheduling deadline”
* 然后，在这些 scheduling deadlines 上用 EDF 算法调度任务，有最早 scheduling deadline 的任务被选中执行
> Summing up, the **CBS**[2,3] algorithm **assigns scheduling deadlines** to tasks so that each task runs for at most its *runtime* every *period*, avoiding any interference between different tasks (bandwidth isolation), while the **EDF**[1] algorithm **selects the task** with the earliest scheduling deadline as the one to be executed next. Thanks to this feature, tasks that do not strictly comply with the "traditional" real-time task model (see Section 3) can effectively use the new policy.
>
>Documentation/scheduler/sched-deadline.txt

## 概念

* **sporadic task** 有一系列的工作的任务，并且在每个周期每个工作最多激活一次
* **relative deadline** 在这个期限前该工作需要完成执行
* **computation time** 执行任务所必须的 CPU 时间
* **arrival time** 一个任务由于一项新的工作不得不被执行而被唤醒的时刻（也被叫做 *request time* 或 *release time*）
* **start time** 任务开始执行的时间
* **absolute deadline** 由 *arrival time* 加上 *relative time* 得到
  ```
  arrival/wakeup                    absolute deadline
        |    start time                    |
        |        |                         |
        v        v                         v
   -----x--------xooooooooooooooooo--------x--------x---
                 |<- comp. time ->|
        |<------- relative deadline ------>|
        |<-------------- period ------------------->|
  ```
* 用[`sched_setattr(2)`](http://man7.org/linux/man-pages/man2/sched_setattr.2.html)设置时可指定的 *Runtime*，*Deadline* 和 *Period*
  * **Runtime** 通常在实践上把 *Runtime* 设置为大于 *computation time* 的平均值（或者对于硬实时任务来说为 *最坏情况执行时间*）
  * **Deadline** 为 *relative deadline*
  * **Period** 为任务的周期
  ```
  arrival/wakeup                    absolute deadline
        |    start time                    |
        |        |                         |
        v        v                         v
   -----x--------xooooooooooooooooo--------x--------x---
                 |<-- Runtime ------->|
        |<----------- Deadline ----------->|
        |<-------------- Period ------------------->|
  ```
* *Runtime*，*Deadline* 和 *Period* 对应到`sched_attr`结构体的`sched_runtime`，`sched_deadline`和`sched_period`域
  * 内核要求它们之间有以下关系
  ```
  sched_runtime <= sched_deadline <= sched_period
  ```
* CBS 通过限流尝试跑超它们给定的 *Runtime* 的线程来确保任务间互不干扰
* 内核必须防止在给定的限制条件下，`SCHED_DEADLINE`线程不可调度的情况的出现，因此当修改或设置`SCHED_DEADLINE`策略的属性的时候，会有准入测试
  * 该准入测试失败 `sched_setattr()` 会把 [`errno`](http://man7.org/linux/man-pages/man3/errno.3.html) 设置为 **EBUSY**
* 为满足当一个线程进入`SCHED_DEADLINE`策略时所作出的保证，`SCHED_DEADLINE`线程是系统中（用户可控制的）最高优先级的进程
  * 如果任何`SCHED_DEADLINE`进程是可运行的，它会抢占任何调度策略优先级在它之下的线程
* `SCHED_DEADLINE`线程调用[`fork(2)`](http://man7.org/linux/man-pages/man2/fork.2.html)会失败，错误码为 **EAGAIN**
  * 除非设置 **reset-on-fork** flag，使其在 fork 时不去继承一些调度策略相关的属性
* 当一个`SCHED_DEADLINE`线程调用[`sched_yield(2)`](http://man7.org/linux/man-pages/man2/sched_yield.2.html)会让出当前的工作，等待下一个新的周期才开始

## 内核实现的CBS算法简述
* 每个`SCHED_DEADLINE`任务都以“runtime”，”deadline“和”period“参数为特征
* 任务的状态由一个 **调度期限（scheduling deadline）** 和 **剩余时间（remaining runtime）** 来描述。这两个参数都初始化为 **0**
* 当一个`SCHED_DEADLINE`任务醒来（变为准备运行）时，调度器做如下检查
  ```
           remaining runtime                  runtime
  ----------------------------------    >    ---------
  scheduling deadline - current time           period
  ```
  * 如果 *调度期限* 小于 *当前时间*，或者
  * 以上不等式成立
  * 调度期限和剩余时间需要被重新初始化为
  ```
  scheduling deadline = current time + deadline
  remaining runtime = runtime
  ```
  * 否则，调度期限和剩余时间保持不变
  * 不等式的语义：
    * `runtime/period` 表示服务带宽
    * `scheduling deadline - current time` 表示当前还留给任务的最多可运行时间
    * `remaining runtime` 表示当前任务运行完还需要运行的时间
    * 大于不等式成立表明，根据当前带宽，在截止期限到来之前，肯定无法把剩余的运行时间跑完（尽管有剩余运行时间可能比还留给任务运行的时间还要短的情况，但按分配比例来说，不太可能跑完）
* 当一个`SCHED_DEADLINE`任务执行一定的时间`t`，它的剩余运行时间减为：
  ```
  remaining runtime = remaining runtime - t
  ```
  （技术上来说，运行时间在每个 tick 会递减，或者当任务被 *解除调度/抢占* 的时候也会被减）
* 当剩余的运行时间小于或等于 **0** 时，任务被称为“限流（throttled）”（在有关实时的文献中也被称为”耗尽（depleted）“），并且在它的调度期限没有到来之前，它不再能被调度。
  * 补充该任务的运行时间（replenishment time，见下一项）被设为当前调度期限的值
* 当前时间等于一个限流任务的补充时间的时候，该任务的调度期限和剩余时间更新为：
  ```
  scheduling deadline = scheduling deadline + period
  remaining runtime = remaining runtime + runtime
  ```
* 更学术更完整的 CBS 算法及其证明见以下论文
  * Integrating Multimedia Applications in Hard Real-Time Systems - Luca Abeni and Giorgio Buttazzo, Scuola Superiore S. Anna, Pisa
  * Server Mechanisms for Multimedia Applications - Luca Abeni, Scuola Superiore S. Anna, Pisa
  * Constant Bandwidth Server Revisited - Luca Abeni, Giuseppe Lipari and Juri Lelli

# 参考资料
- [sched(7) - Linux manual page](http://man7.org/linux/man-pages/man7/sched.7.html)
- [SCHED_DEADLINE - Wikipedia](https://en.wikipedia.org/wiki/SCHED_DEADLINE)
- [The IRMOS realtime scheduler [LWN.net]](https://lwn.net/Articles/398470/)
