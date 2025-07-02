# 虚拟设备产生的虚拟外设中断
* 传统的 VM entry 方式注入中断
* Posted Interrupt 方式注入中断

# 真实物理设备产生的外设中断

## process posted interrupts == 0

### external-interrupt exiting == 0
* 中断直接找到 Guest IDT（非常规）

### external-interrupt exiting == 1
* VM exit，需要 IRQ Handler 分辨是否是指派给 Guest 设备产生的中断？
  * 是，则用传统 VM entry 方式注入中断
  * 否，Host 侧处理该中断

## process posted interrupts == 1

### external-interrupt exiting == 1
* 是否中断重映射到 PIR（即中断透传）？
  * 是，中断直接通过 PIR 注入 Guest（由硬件完成整个过程，软件不参与）
  * 否，VM exit，Host 侧处理该中断

# IPI
* physical vector == posted-interrupt notification vector？
  * 是，CPU 递交 PIR 中 pending 的中断给 guest
  * 否，VM exit，IPI 的处理路径