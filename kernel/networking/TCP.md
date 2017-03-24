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

#### 当 *接受队列* 满了，还有连接收到对端的 `ACK` 而需从 *未完成队列* 移入时的行为
* 比如说，服务器端没有及时调用`accept()`消耗掉 *接收队列* 里的连接，又有新的连接连入并回复`ACK`时。
* `net/ipv4/tcp_ipv4.c:tcp_v4_syn_recv_sock()`会增加`/proc/net/netstat`中的`ListenOverflows`和`ListenDrops`计数；
* `net/ipv4/tcp_minisocks.c:tcp_check_req()`会看`/proc/sys/net/ipv4/tcp_abort_on_overflow`的设置：
	* 如果`sysctl_tcp_abort_on_overflow`已设置，发回`RST`包；
	* 否则什么都不做，该`ACK`包被忽略。后续的处理在另一个过程中：
		* `SYN RECEIVED`状态下有个关联的定时器，如果未收到`ACK`应答（或者被忽略），TCP 实现会重发`SYN/ACK`（抓包的时候注意这个特征，尽管服务器收到了`ACK`仍然不停地重发`SYN/ACK`）
		* 重发次数由`/proc/sys/net/ipv4/tcp_synack_retries`指定，采用[指数回退算法](http://en.wikipedia.org/wiki/Exponential_backoff)
		* 到达重发次数限制后，发送`RST`重置连接。
* 从客户端的角度来看，当收到服务器端的第一个`SYN/ACK`后，状态就变为`ESTABLISHED`，此时如果开始发送数据，会造成数据的重传。[TCP slow-start](http://en.wikipedia.org/wiki/Slow-start) 会限制此期间发送的 segments 的数目。

#### 当 *接受队列* 满了，还有连接收到对端的 `SYN` 时的行为
* 见`net/ipv4/tcp_ipv4.c:tcp_v4_conn_request()`的处理。
* 增加`/proc/net/netstat`中的`ListenOverflows`和`ListenDrops`计数。
* 如果收到太多的`SYN`包，会丢掉它们中的一些。

#### 如果在慢速网络连接中，客户端和服务器端的round-trip时间很长，但服务器端有足够的能力accept连接，此时可做什么样的优化调整？
* 增加应用程序的 backlog 是可以的，相当于给足够的时间让报文完成传输。
* 这种情况下更优的方案是调整`/proc/sys/net/ipv4/tcp_max_syn_backlog`，让系统保持更多的未完成连接。


## Reference
* [How TCP backlog works in Linux](http://veithen.github.io/2014/01/01/how-tcp-backlog-works-in-linux.html)
* [man listen](https://linux.die.net/man/2/listen)
* [man proc](https://linux.die.net/man/5/proc)
