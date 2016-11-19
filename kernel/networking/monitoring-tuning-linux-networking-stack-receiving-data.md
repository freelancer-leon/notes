# 概览

* 高层次上看一个包从到达至socket receive buffer的路径如下：
  1. 驱动被加载和初始化。
  2. 包从网络到达网卡（NIC）。
  3. 包（经由DMA）拷贝至内核内存中的一个ring buffer。
  4. 产生硬件中断，让系统知道有一个包在内存中。
  5. 驱动调用进入[NAPI](http://www.linuxfoundation.org/collaborate/workgroups/networking/napi)，如果还没有轮询循环的话，则开始一个新的。
  6. `ksoftirqd`进程运行在系统的每个CPU上。它们在启动时注册。`ksoftirqd`进程通过调用设备驱动在初始化期间注册的NAPI `poll` 函数把包从ring buffer里拉过来。
  7. 之前网络数据写入的ring buffer里的内存regions被unmap。
  8. 被DMA到内存里的数据作为一个“skb”向上传递给网络层做进一步的处理。
  9. 如果packet steering开启，或者，NIC有多个接收队列，进来的网络数据帧会被分发给多个CPU。
  10. 网络数据帧从队列交给协议层。
  11. 协议层处理数据。
  12. 数据被添加到被协议层附着在socket上的receive buffer中。

# 详细查看

## 网络设备驱动

### 初始化

#### PCI 初始化

#### PCI probe

* 绝大部分驱动有许多让设备能够投入使用的代码。确切要做的事情则因设备而异。
* 一些典型的操作包括：
  1. 使能PCI设备。
  2. 请求内存范围和[IO端口](http://wiki.osdev.org/I/O_Ports)。
  3. 设置 [DMA](https://en.wikipedia.org/wiki/Direct_memory_access) mask。
  4. 注册驱动支持的 ethtool（下面详述）函数。
  5. 任何需要的 watchdog 任务（例如，e1000e有一个 watchdog 任务去检查硬件是不是被挂住了）。
  6. 其他设备相关的事情，像 workaround 或 处理设备相关的 quirks 或 类似的事情。
  7. 创建、初始化、注册一个`struct net_device_ops`结构。这个结构包含指向各种需要的操作的函数指针，如打开设备、发送数据到网络、设置MAC地址等等。
  8. 创建、初始化、注册一个高层次的`struct net_device`，表示一个网络设备。

#### NAPI

* NAPI 和以往的收取数据的方法有几个重要的不同。NAPI 允许一个设备驱动注册一个`poll`函数，NAPI子系统会调用该函数收取数据帧。
* 在网络设备驱动中使用NAPI的目的：
* NAPI
> NAPI (“New API”) is an extension to the device driver packet processing framework, which is designed to improve the performance of high-speed networking. NAPI works through:
>
> **Interrupt mitigation**
>
> High-speed networking can create thousands of interrupts per second, all of which tell the system something it already knew: it has lots of packets to process. NAPI allows drivers to run with (some) interrupts disabled during times of high traffic, with a corresponding decrease in system load.
>
> **Packet throttling**
>
> When the system is overwhelmed and must drop packets, it's better if those packets are disposed of before much effort goes into processing them. NAPI-compliant drivers can often cause packets to be dropped in the network adaptor itself, before the kernel sees them at all.
>
> New drivers should use NAPI if the hardware can support it. However, NAPI additions to the kernel do not break backward compatibility and drivers may still process completions directly in interrupt context if necessary.

#### igb驱动中NAPI的初始化

### 唤起一个网络设备

* 当一个网络设备被唤起时（例如，用`ifconfig eth0 up`），附着在结构`net_device_ops`的`ndo_open`域的函数被调用。
* 一个典型的`ndo_open`函数会做以下一些事情：
  1. 分配RX和TX队列的内存
  2. 使能NAPI
  3. 注册一个中断处理函数
  4. 使能硬件中断
  5. 其他一些事情

##### 准备从网络接受数据
* RX队列的大小的数值可以通过`ethtool`工具调整，调整这些值会显著地影响处理帧数和丢帧数。
* NIC使用一个哈希函数利用包头域（如源、目的、端口）计算的结果来决定数据应该被导向哪个队列。
* 一些NIC可以让你调整RX队列的权重，从而让你可以发送更多的通信量到特定的队列。
* 极少数的NIC可以让你调整哈希函数。

##### 使能NAPI

##### 注册一个中断处理函数
* MSI-X中断是优先考虑的方法，特别是那些支持多RX队列的NIC。
  * 这是因为每个RX队列可以有属于自己的硬件中断指派。
  * 这样中断可以（通过`irqbalance`或者修改`/proc/irq/IRQ_NUMBER/smp_affinity`）被特定的CPU处理。
  * 如此，处理中断的CPU将会是处理包的CPU，这样到达的包从硬件中断直到网络层都可以被CPU分别处理。
* 关于MSI-X和MSI，更多的信息看[这里](https://en.wikipedia.org/wiki/Message_Signaled_Interrupts)。
* drivers/net/ethernet/intel/igb/igb_main.c
```c
static int igb_request_irq(struct igb_adapter *adapter)
{
  struct net_device *netdev = adapter->netdev;
  struct pci_dev *pdev = adapter->pdev;
  int err = 0;

  if (adapter->msix_entries) {
    err = igb_request_msix(adapter);
    if (!err)
      goto request_done;
    /*fall back to MSI*/

    /*...*/
  }

  /*...*/

  if (adapter->flags & IGB_FLAG_HAS_MSI) {
    err = request_irq(pdev->irq, igb_intr_msi, 0,
          netdev->name, adapter);
    if (!err)
      goto request_done;

    /*fall back to legacy interrupts*/

    /*...*/
  }

  err = request_irq(pdev->irq, igb_intr, IRQF_SHARED,
        netdev->name, adapter);

  if (err)
    dev_err(&pdev->dev, "Error %d getting interrupt\n", err);

request_done:
  return err;
}
```

##### 使能中断

### 监视网络设备
* `ethtool -S eth0`
* `cat /sys/class/net/eth0/statistics/xxx`
* `cat /proc/net/dev`

### 调节网络设备

## Linux网络设备子系统

### 网络设备子系统的初始化

* 网络设备（netdev）子系统在`net_dev_init`函数中初始化。

#### 初始化`struct softnet_data`结构体

* `net_dev_init`会为系统里的每个CPU创建一组`struct softnet_data`结构。
* 这些结构里有处理网络数据的几件重要的事情的指针：
  * 被注册到这个CPU上的NAPI结构的列表
  * 用于数据处理backlog
  * 处理`weight`
  * [receive offload](https://en.wikipedia.org/wiki/Large_receive_offload)结构列表
  * [Receive packet steering](https://lwn.net/Articles/362339/)设置
  * 其他

#### 初始化softirq处理函数

* net/core/dev.c
```c
static int __init net_dev_init(void)
{
  /*...*/

  open_softirq(NET_TX_SOFTIRQ, net_tx_action);
  open_softirq(NET_RX_SOFTIRQ, net_rx_action);

 /*...*/
}

subsys_initcall(net_dev_init);
```

### 数据到达

#### 中断处理函数

* drivers/net/ethernet/intel/igb/igb_main.c
```c
static irqreturn_t igb_msix_ring(int irq, void *data)
{
  struct igb_q_vector *q_vector = data;

  /*Write the ITR value calculated from the previous interrupt.*/
  igb_write_itr(q_vector);

  napi_schedule(&q_vector->napi);

  return IRQ_HANDLED;
}
```
* 如果NAPI处理循环没有激活，`napi_schedule`负责唤醒它。
* NAPI处理循环在softirq中执行，而不是由中断处理函数执行。

#### NAPI和`napi_schedule`
* NAPI特地为了收取网络数据，但无需由NIC产生的中断来通知数据已经准备好了，而存在。
* NAPI是关闭的，直到第一个包到达，NIC发起一个中断的那一个点，然后NAPI被使能并且开始。
* NAPI可以被关闭，在它再次启动之前需要一个硬件中断唤醒。
* net/core/dev.c
```c
/* Called with irq disabled */
static inline void ____napi_schedule(struct softnet_data *sd,
                     struct napi_struct *napi)
{   
    list_add_tail(&napi->poll_list, &sd->poll_list);
    __raise_softirq_irqoff(NET_RX_SOFTIRQ);
}

/*...*/

/**
 * __napi_schedule - schedule for receive
 * @n: entry to schedule
 *
 * The entry's receive function will be scheduled to run.
 * Consider using __napi_schedule_irqoff() if hard irqs are masked.
 */
void __napi_schedule(struct napi_struct *n)
{
    unsigned long flags;

    local_irq_save(flags);
    ____napi_schedule(this_cpu_ptr(&softnet_data), n);
    local_irq_restore(flags);
}
EXPORT_SYMBOL(__napi_schedule);
/*...____```*/
```

* `____napi_schedule()`完成两件重要的事情：
  1. 从设备驱动的中断处理函数代码传上来的`struct napi_struct`被添加到附在与当前CPU关联的`softnet_data`结构的`poll_list`。
  2. `__raise_softirq_irqoff`被用来“唤醒”（或触发）一个`NET_RX_SOFTIRQ` softirq。如果网络设备子系统初始化期间注册的`net_rx_action`没有被执行的话，这会导致它被执行。

#### 关于CPU和网络数据处理要注意的一点
* 驱动的中断处理函数本身作的事情非常少，软中断的处理函数会与驱动的中断处理函数同一CPU上执行。
* 这就是为什么设置一个特定的IRQ由那个CPU处理这么重要：该CPU不但执行驱动的中断处理函数，而且相同的CPU会被用于通过NAPI在一个softirq中收取包。
* [Receive Packet Steering](https://lwn.net/Articles/362339/)可以用来分发一些工作到其他CPU，之后上升到网络协议栈。

#### 监视网络数据的到达

#### 调整网络数据的到达
##### 中断合并（Interrupt coalescing）
* [Interrupt coalescing](https://en.wikipedia.org/wiki/Interrupt_coalescing)是一种阻止从设备发起的中断到CPU的方法，直至一个特定的工作量或者挂起的事件数到达后。
* 可以防止中断风暴，能帮助增加吞吐量或延迟。

> **Note**: while interrupt coalescing seems to be a very useful optimization at first glance, the rest of the networking stack internals also come into the fold when attempting to optimize. Interrupt coalescing can be useful in some cases, but you should ensure that the rest of your networking stack is also tuned properly. Simply modifying your coalescing settings alone will likely provide minimal benefit in and of itself.


##### 调整IRQ亲和性
* 调整IRQ亲和性首先要检查`irqbalance`守护进程是否在运行。
  * 该守护进程会尝试将IRQ均衡到每个CPU，有可能会覆盖你的设置。
  * 要么关闭`irqbalance`，要么用`--banirq`和`IRQBALANCE_BANNED_CPUS`来让`irqbalance`知道不要去碰那些你指派给自己用的IRQ集合和CPU。
* 其次检查`/proc/interrupts`里，你的NIC给每个网络RX队列的IRQ号的列表。
* 最后，为那些IRQ通过`/proc/irq/IRQ_NUMBER/smp_affinity`逐个调整，哪个CPU处理这些中断。
  * 例子：设置中断亲和性，IRQ 8 由 CPU 0处理
  ```
  $ sudo bash -c 'echo 1 > /proc/irq/8/smp_affinity'
  ```



# 参考资料
* [Monitoring and Tuning the Linux Networking Stack: Receiving Data](http://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/)
* [Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data](http://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/)
