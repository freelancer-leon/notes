# Bus Lock

## Bus Lock 探测和处理
### Split Lock 带来的问题
* **Split lock** 是其操作数跨越两个高速缓存行的任何原子操作。由于操作数跨越两个缓存行并且操作必须是原子的，因此系统在 CPU 访问两个缓存行时锁定总线
* **Bus lock**：获取 bus lock 是通过对写回 (WB) 内存的 split locked 访问或对非 WB 存储器的任何锁定的访问来实现的
  * 这通常比高速缓存行中的原子操作慢数千个周期
  * 它还会破坏其他内核的性能并使整个系统性能极大地降低
### Split lock 的探测
* Intel 处理器可能支持以下一种或两种硬件机制来检测 split lock 和 bus lock
#### `#AC` 探测 Split Lock
* 从 Tremont Atom 开始，CPU split lock 操作可能会在尝试拆分锁定操作时引发对齐检查 (`#AC`) 异常
* Split lock 产生 `#AC` 异常由 `TEST_CTRL` MSR 的第 `29` 位，*split lock 探测位* 来启用
#### `#DB` 探测 Bus Lock
* 某些 CPU 具有在用户指令获取 bus lock 并 **执行后** 通过 `#DB` trap 通知内核的能力
* 这允许内核终止应用程序或强制执行以达到节流的目的
### 软件方面的处理
* 内核 `#AC` 和 `#DB` 处理程序根据内核参数 `split_lock_detect` 处理 bus lock。以下是不同选项的摘要：

`split_lock_detect=` | split lock 引起的 `#AC` | bus lock 导致的 `#DB`
---------------------|------------------------|-----------------------
`off`                | 什么都不做              | 什么都不做
`warn`（缺省）        | 当支持这两个功能时，每个任务警告一次内核 OOPs 并禁用未来的检查，在 `#AC` 中警告 | 每个任务警告一次并继续运行
`fatal`              | 当支持这两个功能时，内核 OOPs，并向用户态发送 `SIGBUS`，在 `#AC` 中产生 fatal 错误 | 向用户态发送 `SIGBUS`
`ratelimit:N (0 < N <= 1000)` | 什么都不做 | 将系统范围内的 bus lock 速率限制为每秒 N 次总线锁定，并在总线锁定时发出警告

## Bus Lock 调试异常
* `DB6` 寄存器 bit `11` 在 Bus Lock Trap 异常时由处理器清零
* 当使用 `DEBUGCTL`（`IA32_DEBUGCTL` MSR `0x1D9`）的第 `2` 位启用 Bus Lock Trap 时，任何导致总线锁定的指令（主要是使用 `LOCK` 前缀在不可缓存内存上执行内存原子操作的指令）将清除 `DR6` 的第 `11` 位并导致 trap 类型为 `#DB` 的异常
  * **注意**：处理器不会以其他方式设置或清除此位
  * 为避免在辩别调试异常时出现混淆，软件调试异常处理程序应在返回被中断任务之前将此位设置为 `1`
* 在不支持 Bus Lock Trap 异常的处理器上，`DR6` 的第 `11` 位是只读位，其作用与第 `10:4` 位相同，都是 `1`
* CPU 使用 `CPUID.(EAX=7, ECX=0).ECX[24]` 枚举对该特性的支持，设置为 `1` 表示支持
* 当 `CPL > 0` 时，硬件仅生成用于 bus lock 检测的`#DB`，以避免在处理第一个 `#DB` 时来自多个 bus lock 的嵌套 `#DB`
* *Breakpoint* 和 *bus lock* 都可以在同一指令中触发 `#DB` trap，处理它们的顺序由内核 `#DB` 处理程序选择
* 在 `/proc/cpuinfo` 中的 CPU feature 标志是 `bus_lock_detect`

* 启用 split lock 的调用路径
```c
early_identify_cpu()
-> sld_setup()
   -> split_lock_setup()
      -> __split_lock_setup() //设置 split lock 的 CPU feature
   -> sld_state_setup() //得到 split_lock_detect= 的设置
   -> sld_state_show() //在启动时打印 split lock 和 bus lock 的支持情况
```
* 启用 bus lock 的调用路径
```c
identify_cpu()
-> this_cpu->c_init(c)
=> init_intel()
   -> split_lock_init()
   -> bus_lock_init()
```

### Bus Lock 调试异常虚拟化的支持
* VM exit 时设置 VMCS 的 guest-state area 中的 pending debug exception 字段的第 `11` 位，以指示总线锁定调试异常 pending 但未交付
* 设置此位的 VM exit 也会设置该字段的第 `12` 位（VM exit 还设置第 `12` 位以指示至少遇到一个数据或 I/O 断点并在 `DR7` 中启用，或者发生与 RTM transactional regions 的高级调试相关的调试异常。）
* 启用后，如果处理器检测到一个或多个 bus lock 是在 VMX non-root operation 执行期间导致的，则处理器会生成一个 VM exit， exit reason 为 `74`
  * 这种 VM exit 类似于陷阱，在执行获取总线锁的指令后交付
  * 如果此 VM exit 的交付被更高优先级的 VM exit 抢占，则 VMCS 中 exit reason 字段的第 `26` 位设置为 `1`
* VMM 可以通过设置 secondary processor-based 执行控制的第 `30` 位，以在 VMX non-root operation 中获取的总线锁时发生 VM exit
* 处理器通过设置 `IA32_VMX_PROCBASED_CTLS2` MSR 的第 `62` 位来枚举对此控制的的支持，设置为 `1` 表示支持

## References
- [Bus lock detection and handling — The Linux Kernel documentation](https://docs.kernel.org/x86/buslock.html)
- [x86 debug register - Wikipedia](https://en.wikipedia.org/wiki/X86_debug_register)
- [Intel Instruction Set Extension Chapter 9](https://software.intel.com/content/dam/develop/public/us/en/documents/architecture-instruction-set-extensions-programming-reference.pdf)
- [[PATCH v6 0_3] x86_bus_lock Enable bus lock detection - Fenghua Yu](https://lore.kernel.org/all/20210322135325.682257-1-fenghua.yu@intel.com/#r)