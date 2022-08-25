# 虚拟化异常
* 异常向量为`20`
* 异常缩写为`#VE`
* 仅发生在 VMX non-root operation
* 当处理器遇到虚拟化异常时，将异常信息保存到虚拟化异常信息区（virtualization-exception information area）
* 保存虚拟化异常信息后，处理器会像处理任何其他异常一样提供虚拟化异常
* 虚拟化异常的传递会将值`0xFFFFFFFF`写入虚拟化异常信息区域中的偏移量`4`的位置
  * 因此，一旦发生虚拟化异常，只有在软件清除该字段时才会发生另一个异常

## 虚拟化异常信息

* 虚拟化异常信息区的格式

字节偏移 | 内容
---------|------
0  | 如果发生 VM exit 而不是虚拟化异常，则将`32`位值作为退出原因保存到 VMCS 中 。对于 EPT violations，此值为`48 (00000030H)`
4  | `0xFFFFFFFF`
8  | 如果发生 VM exit 而不是虚拟化异常，则将`64`位值作为退出条件（qualification）保存到 VMCS 中
16 | 如果发生 VM exit 而不是虚拟化异常，则将 guest 线性地址的`64`位值保存到 VMCS 中
24 | 如果发生 VM exit 而不是虚拟化异常，则将 guest 物理地址的`64`位值保存到 VMCS 中
32 | 当前由 EPTP 索引 的 VM 执行控制的`16`位值

* VMM 可以允许 guest 软件访问虚拟化异常信息区域
  * 如果是这样，guest 软件可能会修改该内存（例如，清除偏移量`4`处的`32`位值）

## 虚拟化异常的递交
* 保存虚拟化异常信息后，处理器会像处理其他异常一样处理虚拟化异常：
  * 如果 VMCS 的异常位图中的第`20 bit` (`#VE`) 为`1`，则虚拟化异常会导致 VM exit（见下文）。如果该位为`0`，则使用 IDT 中的门描述符`20`传递虚拟化异常
  * 虚拟化异常不会产生 error code。对于虚拟化异常的交付，CPU 不会将 error code 推送到堆栈上
  * 对于 double fault，虚拟化异常与 page fault 具有相同的 serverity
    * 如果虚拟化异常的传递遇到嵌套 fault（contributory faults 或 page fault），则会生成 double fault (`#DF`)
* 在传递另一个异常时不可能遇到虚拟化异常
* 如果虚拟化异常直接导致 VM exit（因为异常位图中的`bit 20`为`1`），异常信息正常保存在 VMCS 的 VM-exit *interruption information* 字段中
  * 具体来说，该事件被报告为异常向量`20`且没有 error code 的硬件异常。正常来说，该字段的第`12`位（由于`IRET`导致 NMI 解锁）会被设置
* 如果虚拟化异常间接导致 VM exit（因为异常位图中的`bit 20`为`0`，并且异常的传递会生成导致 VM exit 的事件），则有关异常的信息通常保存在 VMCS
  * 具体来说，该事件被报告为异常向量`20`且没有 error code 的硬件异常