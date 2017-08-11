# TCP

![http://coolshell.cn//wp-content/uploads/2014/05/tcpfsm.png](pic/tcpfsm.png)

## Listen
* 当连接的一端收到`SYN`，回复`SYN/ACK`后，状态由`LISTEN`变为`SYN RECEIVED`
* 当收到连接的另一端回的`ACK`后，状态才变为`ESTABLISHED`，此时应用程序调用`accept()`才会返回这个连接。
* [`listen()`](https://linux.die.net/man/2/listen)系统调用的`backlog`参数所指的队列长度指的是什么？
	* 监听 socket 其实是有两个队列：
		* 完全建立连接，等待被`accept`的socket的队列，长度由`listen()`的`backlog`参数指定，应用程序来设定；
		* 未完成的连接请求队列，长度由`/proc/sys/net/ipv4/tcp_max_syn_backlog`指定，系统级的设定。
	* `backlog`能支持的最大队列长度受`/proc/sys/net/core/somaxconn`的限制，超过限制会被默然截断。

### 当 *接受队列* 满了，还有连接收到对端的 `ACK` 而需从 *未完成队列* 移入时的行为
* 比如说，服务器端没有及时调用`accept()`消耗掉 *接收队列* 里的连接，又有新的连接连入并回复`ACK`时。
* `net/ipv4/tcp_ipv4.c:tcp_v4_syn_recv_sock()`会增加`/proc/net/netstat`中的`ListenOverflows`和`ListenDrops`计数；
* `net/ipv4/tcp_minisocks.c:tcp_check_req()`会看`/proc/sys/net/ipv4/tcp_abort_on_overflow`的设置：
	* 如果`sysctl_tcp_abort_on_overflow`已设置，发回`RST`包；
	* 否则什么都不做，该`ACK`包被忽略。后续的处理在另一个过程中：
		* `SYN RECEIVED`状态下有个关联的定时器，如果未收到`ACK`应答（或者被忽略），TCP 实现会重发`SYN/ACK`（抓包的时候注意这个特征，尽管服务器收到了`ACK`仍然不停地重发`SYN/ACK`）
		* 重发次数由`/proc/sys/net/ipv4/tcp_synack_retries`指定，采用[指数回退算法](http://en.wikipedia.org/wiki/Exponential_backoff)
		* 到达重发次数限制后，发送`RST`重置连接。
* 从客户端的角度来看，当收到服务器端的第一个`SYN/ACK`后，状态就变为`ESTABLISHED`，此时如果开始发送数据，会造成数据的重传。[TCP slow-start](http://en.wikipedia.org/wiki/Slow-start) 会限制此期间发送的 segments 的数目。

### 当 *接受队列* 满了，还有连接收到对端的 `SYN` 时的行为
* 见`net/ipv4/tcp_ipv4.c:tcp_v4_conn_request()`的处理。
* 增加`/proc/net/netstat`中的`ListenOverflows`和`ListenDrops`计数。
* 如果收到太多的`SYN`包，会丢掉它们中的一些。

### 如果在慢速网络连接中，客户端和服务器端的round-trip时间很长，但服务器端有足够的能力accept连接，此时可做什么样的优化调整？
* 增加应用程序的 backlog 是可以的，相当于给足够的时间让报文完成传输。
* 这种情况下更优的方案是调整`/proc/sys/net/ipv4/tcp_max_syn_backlog`，让系统保持更多的未完成连接。

## /proc接口
### /proc/sys/net/ipv4
#### /proc/sys/net/ipv4/tcp_mem
* 三个整型值 `low`，`pressure`，`high`
* 这些限制被 TCP 用于跟踪它的内存使用
* 以系统 page size 为单位表示
* 缺省值在启动时根据可用内存进行计算
	* 在 32 位系统上 TCP 只能用较少的内存，因此被限制在大约 900 MB 左右
	* 在 64 位系统上则不受此限制
* **low**
	* 当 TCP 使用了低于该值的内存页面数时，TCP 不会考虑释放内存
* **pressure**
	* 当 TCP 使用了超过该值的内存页面数量时，TCP 试图稳定其内存使用，进入`pressure`模式，当内存消耗低于`low`值时则退出`pressure`状态
* **high**
	* TCP 将会分配的全局的最大的页面数
	* 该值覆盖任何其他的被内核强加的限制

#### /proc/sys/net/ipv4/tcp_rmem
* 三个整型值 `min`，`default`，`max`
* 这些参数被 TCP 用于管理接收缓冲区
* TCP 从这三个缺省值开始，动态调整接收缓冲的大小，范围在这些值之间，取决于可用内存的多少
* **min**
  * 每个 TCP socket 接收缓冲的最小 size
  * 缺省值为系统的 *page size*
  * 该值用于确保即使在`pressure`模式下，低于该值的分配依然会成功（真实作用）
  * 而不是用于约束用`SO_RCVBUF`可设的 socket 接收缓冲的大小（容易被误认的作用）
* **default**
	* TCP socket 接收缓冲的缺省 size
	* 该值覆盖通用目的，对所有协议的初始的缺省缓冲区大小`net.core.rmem_default`的定义
	* 缺省值是 87380 字节
	* 如果希望增加接收缓冲区的大小，应该增大该值（影响所有 sockets）
	* 使用大的 TCP 窗口时，`net.ipv4.tcp_window_scaling`必须开启（缺省开启）
* **max**
	* 用于每个 TCP socket 接收缓冲的最大 size
	* 该值不覆盖全局的 `net.core.rmem_max`
	* 不用于限制用`SO_RCVBUF`可设的 socket 接收缓冲的大小（容易被误认的作用）
	* 缺省值采用如下公式计算
		* `max(87380, min(4MB, tcp_mem[1]*PAGE_SIZE/128))`

#### /proc/sys/net/ipv4/tcp_wmem
* 三个整型值 `min`，`default`，`max`
* 这些参数被 TCP 用于管理发送缓冲区
* TCP 从这三个缺省值开始，动态调整发送缓冲的大小，范围在这些值之间，取决于可用内存的多少
* **min**
  * 每个 TCP socket 发送缓冲的最小 size
  * 缺省值为系统的 *page size*
  * 该值用于确保即使在`pressure`模式下，低于该值的分配依然会成功（真实作用）
  * 而不是用于约束用`SO_SNDBUF`可设的 socket 发送缓冲的大小（容易被误认的作用）
* **default**
	* TCP socket 发送缓冲的缺省 size
	* 该值覆盖通用目的，对所有协议的初始的缺省缓冲区大小`net.core.wmem_default`的定义
	* 缺省值是 16 KB 字节
	* 如果希望增加发送缓冲区的大小，应该增大该值（影响所有 sockets）
	* 使用大的 TCP 窗口时，`net.ipv4.tcp_window_scaling`必须设置位非零值（即开启，缺省开启）
* **max**
	* 用于每个 TCP socket 发送缓冲的最大 size
	* 该值不覆盖全局的 `net.core.wmem_max`
	* 不用于限制用`SO_SNDBUF`可设的 socket 发送缓冲的大小（容易被误认的作用）
	* 缺省值采用如下公式计算
		* `max(65536, min(4MB, tcp_mem[1]*PAGE_SIZE/128))`

## Reference
* [How TCP backlog works in Linux](http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html)
* [man listen](https://linux.die.net/man/2/listen)
* [man proc](https://linux.die.net/man/5/proc)
* [man tcp](http://man7.org/linux/man-pages/man7/tcp.7.html)
* [proc/sys/net/ipv4/下各项的意义](http://lijichao.blog.51cto.com/67487/308509)
* [Linux之TCPIP内核参数优化](http://www.cnblogs.com/fczjuever/archive/2013/04/17/3026694.html)
