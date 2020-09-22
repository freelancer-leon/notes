# ARP

## ARP 相关的 sysctl 参数
### arp_ignore
* 在内核参数中除了每个网卡都有自己的`arp_ignore`配置外，还有两个（一个是默认 default，一个是全局 all）需要用到`arp_ignore`配置。
* 如果某个的网络接口（网卡）上没有配置`arp_ignore`参数的值则会把 default 上配置的`arp_ignore`应用到该网络接口上。
* 而在所有网络接口上实际生效的值是 *all* 和 *对应网络接口上配置的`arp_ignore`参数值* 中较大的那个值。
#### arp_ignore 参数的值及其含义
0 - (默认值): 回应任何网络接口（网卡）上对任何本机IP地址的 ARP 查询请求。
* 比如`eth0=192.168.0.1/24`,`eth1=10.1.1.1/24`,
  * 那么即使`eth0`收到来自`10.1.1.2`这样地址发起的对`10.1.1.1`的 ARP 查询也会给出正确的回应；
  * 而原本这个请求该是出现在`eth1`上，也该有`eth1`的回应。

1 - 只回答目标 IP 地址是本机上来访网络接口（网卡）IP 地址的 ARP 查询请求 。
* 比如`eth0=192.168.0.1/24`,`eth1=10.1.1.1/24`,
  * 那么即使`eth0`收到来自`10.1.1.2`这样地址发起的对`192.168.0.1`的查询会回应，
  * 而对`10.1.1.1`的 ARP 查询不会回应。

2 - 只回答目标 IP 地址是本机上来访网络接口（网卡）IP 地址的 ARP 查询请求，且来访 IP（源IP）必须与该网络接口（网卡）上的 IP（目标IP）在同一子网段内 。
* 比如`eth0=192.168.0.1/24`,`eth1=10.1.1.1/24`,
  * `eth1`收到来自`10.1.1.2`这样地址发起的对`192.168.0.1`的查询不会回应，
  * 而对`192.168.0.2`发起的对`192.168.0.1`的 ARP 查询是否会回应？

3 - do not reply for local addresses configured with scope host, only resolutions for global and link addresses are replied。

4-7 - 保留未使用

8 - 不回应所有（本机地址）的 ARP 查询

### arp_announce
* `arp_announce` 对网络接口（网卡）上发出的 **ARP 请求包中的源 IP 地址** 作出相应的限制；主机会根据这个参数值的不同选择使用 *IP 数据包的源 IP* 或 *当前网络接口卡的 IP 地址* 作为 ARP 请求包的 *源 IP地址*。
  * 对于 ARP 请求来说，*目的 IP 地址* 和 *源 MAC 地址* 为已知，*目的 MAC 地址* 为广播，那么剩下的 *源 IP 地址* 在有多个网卡的时候还是有选择的余地的，`arp_announce`就是控制这个的。
* 在内核参数中除了每个网卡都有自己的`arp_announce`配置外，还有两个（一个是默认 default，一个是全局 all）需要用到`arp_announce`配置。
* 如果某个的网络接口（网卡）上没有配置`arp_announce`参数的值则会把 default 上配置的`arp_announce`应用到该网络接口上。
* 而在所有网络接口上实际生效的值是 *all* 和 *对应网络接口上配置的`arp_announce`参数值* 中较大的那个值。
#### arp_announce 参数的值及其含义
0 - (默认) 在任意网络接口（`eth0`，`eth1`，`lo`）上使用任何本机地址进行 ARP 请求。
  * 也就是说如果 *IP 数据包中的源 IP* 与 *当前发送 ARP 请求的网络接口卡 IP 地址* **不同** 时（但这个 IP 依然是本主机上其他网络接口卡上的 IP 地址），ARP 请求包中的源 IP 地址将使用 **与 IP 数据包中的源 IP 相同的本主机上的 IP 地址**，而不是使用当前发送 ARP 请求的网络接口卡的 IP 地址。

1 - 尽量避免使用不在该网络接口（网卡）子网段内的 IP 地址做为 ARP 请求的源 IP 地址。
  * 当接收此 ARP 请求的主机要求 ARP 请求的源 IP 地址与接收方 IP 在同一子网段时，此模式非常有用。此时会检查 IP 数据包中的源 IP 是否为所有网络接口上子网段内的 IP 之一。
    * 如果找到了一个网络接口的 IP 正好与 IP 数据包中的源 IP 在同一子网段，则使用该网络接口卡进行 ARP 请求。
    * 如果 IP 数据包中的源 IP 不属于各个网络接口上子网段内的 IP，那么将采用级别 2 的方式来进行处理。

2 - 始终使用与目标 IP 地址对应的最佳本地 IP 地址作为 ARP 请求的源 IP 地址。
  * 在此模式下将忽略 IP 数据包的源 IP 地址并尝试选择能与目标 IP 地址通信的本机地址。
    * 首要是选择所有网络接口中子网包含该目标 IP 地址的本机 IP 地址。
    * 如果没有合适的地址，将选择当前的网络接口或其他的有可能接受到该 ARP 回应的网络接口来进行发送 ARP 请求，并把发送 ARP 请求的网络接口卡的 IP 地址设置为 ARP 请求的源 IP。

### arp_filter
* `arp_filter` 在至有一个 `conf/{all,interface}/arp_filter` 被设置为 true 时使能；否则它会被禁用。

#### arp_filter 参数的值及其含义
0 - （默认）内核可以用其他接口的地址回复 ARP 请求。
  * 这看起来好像有问题，但通常是合理的，因为这样增加了成功通信的机会。IP 地址是属于整个 Linux 主机的，而不是某个特定接口的。
  * 只有在更复杂的设置的时候，如负载均衡，这种行为才会引起问题。

1 - 允许多个网络接口在同一子网，并根据内核是否将来自 ARP 的 IP 的数据包路由到该接口，来回答每个接口的 ARP（因此，必须使用 source based routing 做这个工作）。
  * 换句话说，它可以用来控制哪块（通常是一块）网卡会用来回复 ARP 请求。



# References
* [/proc/sys/net/ipv4/* Variables](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
* [Linux 内核参数 arp_ignore & arp_announce 详解](https://www.jianshu.com/p/a682ecae9693)
* [Linux内核参数之arp_ignore和arp_announce](https://www.jianshu.com/p/734640384fda)
* [arp_ignore 和 arp_filter](http://huntxu.github.io/2015-12-24-arp-filter-vs-arp-ignore.html)
* [RFC 1122, 2.3.2.1 ARP Cache Validation](https://www.freesoft.org/CIE/RFC/1122/23.htm)
