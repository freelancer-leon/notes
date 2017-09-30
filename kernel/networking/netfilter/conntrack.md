# Connection tracking

## Connection tracking系统与有状态的检查
* Connection tracking 在 Netfilter Framework 之上。
* Connection tracking system 把关于一个链接的状态信息存储在一个内存结构中，包含 IP 的源和目的地址、端口号对、协议类型、状态和超时。根据这些额外的信息，我们能定义更智能的过滤规则。
* 此外，有些应用层协议，例如 FTP、TFTP、IRC 和 PPTP，防火墙用传统的静态过滤方法有些方面难以跟踪。Connection tracking system 定义了一个机制来跟踪这些方面。
* 记住，**Connection tracking system 只跟踪，不过滤**。
* 当内核将 Connection tracking 支持编译进去时（`CONFIG_NF_CONNTRACK`被设置），即便 iptables 规则没有激活，Connection tracking hook 回调也会被调用。
  * 会有性能损失
  * 可以考虑编译成 module，当用到时才加载

## 连接的状态
* 对一个连接来说，有以下状态：
  * **NEW** 连接正在开始。如果包是有效的，即如果它属于一个有效的初始化序列（例如，在一个 TCP 连接中，收到一个`SYN`包），并且如果防火墙看到在某一方向上的通信（例如，防火墙还未看见任何回复包），那么就能到达这个状态。
  * **ESTABLISHED** 连接已经建立。换句话说，防火墙看见双向的通信了。
  * **RELATED** 这是一个期望的连接。
  * **INVALID** 这是一个用于包的连接不遵循期望行为的特定状态。根据情况，系统管理员可以有选择性地定义 iptables 的规则来记录或丢弃这些包。如前所述，connection tracking 不过滤包，但提供一个可以把它们过滤出来的方法。
* 按以上方法来描述，通常说的无状态的协议也是有状态的，例如，UDP。
* 这些状态和 TCP 的状态没有关系。


## References
- [The conntrack-tools user manual](http://conntrack-tools.netfilter.org/manual.html)
- [/proc/sys/net/netfilter/nf_conntrack_* Variables](https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt)
- [netfilter 链接跟踪机制与NAT原理](http://www.cnblogs.com/liushaodong/archive/2013/02/26/2933593.html)
- [Netfilter的expect连接的原理和利用](http://dev.dafan.info/detail/112677?p=29-50)
