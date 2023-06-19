# 26 VMX Non-Root Operation

## 26.4 在 VMX Non-Root Operation 下的其他变化
* 事件屏蔽、任务切换和 user interrupts 的处理在 VMX non-root operation 中有所不同，如以下部分所述。

### 26.4.1 事件屏蔽
* 在 VMX non-root operation 下事件屏蔽的修改如下：
* 如果 “external-interrupt exiting” VM-execution control 为 `1`，则 `RFLAGS.IF` 不控制外部中断的屏蔽
  * 在这种情况下，由于其他原因未被屏蔽的外部中断会导致 VM exit（即使 `RFLAGS.IF = 0`）
* 如果 “external-interrupt exiting” VM-execution control 为 `1`，则外部中断可能会或可能不会被 `STI` 或 `MOV SS` 屏蔽（行为特定于实现）
* 如果 “NMI exit” VM-execution control 为 `1`，则不可屏蔽中断 (NMI) 可能会或可能不会被 `STI` 或 `MOV SS` 阻止（行为特定于实现）

## 26.5 特定于 VMX Non-Root Operation 的特性

### 26.5.1 VMX-Preemption Timer
* 如果最后的 VM entry 是使用 “activate VMX-preemption timer” VM 执行控制位设置为 `1` 执行的，VMX-Preemption Timer 在 VMX non-root operation 倒计时，倒计时的值来自 VM entry 时的设置。
  * 当计时器倒计时到零时，它停止倒计时并发生 VM exit（请参阅第 26.2 节）
* VMX-Preemption Timer 以与时间戳计数器（TSC）的速率成正比的速率递减计数。
  * 具体来说，每当 TSC 中的位 X 由于 TSC 递增而发生变化时，定时器就会减 1。
  * X 的值在 0-31 范围内，可以通过咨询 VMX capability MSR `IA32_VMX_MISC`（参见附录 A.6）来确定。

* 这里的意思是说，host 的 CPU 频率和 guest vCPU 频率不一定一致，依赖于 CPU 频率计算的时间戳计数器（TSC）的定时器可能会因此产生不一致
  * 比如，host 侧的 CPU 频率为 2000 MHz，guest 侧的 vCPU 频率为 1000 MHz，那么 guest 侧设定的 1000 M 后触发的定时器可能会提前一半的时间触发
  * 为了解决这个问题需要设定一个用于换算的 X，使 host 侧的定时器计数与 guest 侧的能正确匹配
* 一些个人理解：
  * 这只影响定时器，不会影响 guest 的指令执行速度以及依据该值设定的一些参数，比如用于 `udelay()` 的 BogoMIPS
  * VMX-Preemption Timer 简化了 vLAPIC timer 的实现，定时器到期时不再是 host 发 IPI 让 guest VM exit 后再注入 timer 中断；而是 guest 直接 VM exit 到 host
* Guest Timer 则更进一步
  * Guest program TSC deadline 的时候不会发生 VM exit
  * Deadline 到期时，由硬件通过 posted interrupt 注入 timer 中断，也不需要 VMM 参与

* VMX-Preemption Timer 在 C 状态 C0、C1 和 C2 中运行；它还在关闭和等待 SIPI 状态下运行。
  * 如果计时器在除等待 SIPI 状态之外的任何状态下倒计时为零，则逻辑处理器转换为 C0 C 状态并导致 VM exit；
  * 如果计时器在等待 SIPI 状态下倒计时为零，则不会导致 VM exit。
  * 在比 C2 更深的 C 状态中，定时器不会递减。