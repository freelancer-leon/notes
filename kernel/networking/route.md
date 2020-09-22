# 路由基本概念
## 搜索顺序
* 搜索主机地址
* 搜索网络地址
* 搜索默认路由

## 主机路由条目
* IP 目的地址需要与路由条目的 IP 地址完全匹配的
* `route -n`看到的 **H** flag 的条目
* 对于非主机路由条目，匹配的是网络地址部分

## 直接路由与间接路由
* **直接路由**：顾名思义，与主机直接相连的路由条目
  * 表明 IP 目的地址与 MAC 地址是同一主机
  * `route -n`的`Gateway`地址为外出 IP 地址，通常是外出 interface 的 IP 地址
  * `route -n`不带 **G** flag 的条目
* **间接路由**：与主机间接相连的路由条目，需要路由器转发到其他子网
  * IP 地址仍然是目的地址，但 MAC 地址得是路由器的 MAC 地址
  * `route -n`的`Gateway`地址为路由器的地址
  * `route -n`带 **G** flag 的条目
* *直接路由* 与 *间接路由* 和 *主机路由条目* 没直接联系，H、G、GH，都可能出现

# 路由表
### table id
* 添加路由表项的时候可以指定路由表 id。
* 路由表 id 可以是来自`/etc/iproute2/rt_tables`的数字或者字符串。
  ```
  $ cat /etc/iproute2/rt_tables
  #
  # reserved values
  #
  255     local
  254     main
  253     default
  0       unspec
  #
  # local
  #
  #1      inr.ruhep
  ```
* 添加路由表项时，如果忽略此参数，默认加到 main 表
* local,broadcast,NAT 路由例外，它们会被加到 local 表

### scope
* 目的的 scope 由路由前缀覆盖
* scope 可以是来自`/etc/iproute2/rt_scopes`的数字或者字符串。
  ```
  $ cat /etc/iproute2/rt_scopes
  #
  # reserved values
  #
  0       global
  255     nowhere
  254     host
  253     link
  #
  # pseudo-reserved
  #
  200     site
  ```
* 添加路由表项时，如果忽略此参数，所有的 gatewayed unicast 路由默认加到 **global** scope
* 直接单播和广播路由加到 **link** scope
* 本地路由加到 **host** scope

# 路由表缓存
```
/proc/net/stat/rt_cache
entries  in_hit in_slow_tot in_slow_mc in_no_route in_brd in_martian_dst in_martian_src  out_hit out_slow_tot out_slow_mc  gc_total gc_ignored gc_goal_miss gc_dst_overflow in_hlist_search out_hlist_search
00000032  00000000 00000198 0000000f 0000000a 00000192 00000000 00000000  00000000 00008901 00000189 00000000 00000000 00000000 00000000 00000000 00000000
00000032  00000000 0000001a 00000001 00000001 00000019 00000000 00000000  00000000 000088d5 0000012f 00000000 00000000 00000000 00000000 00000000 00000000
```

# References
- [Guide to IP Layer Network Administration with Linux - 4.8. Routing Tables](http://linux-ip.net/html/routing-tables.html)
- [ip-route (8) - Linux Man Pages](https://www.systutorials.com/docs/linux/man/8-ip-route/)
- [浅析Linux Kernel 哈希路由表实现(一)](http://basiccoder.com/intro-linux-kernel-hash-rt-1.html)
- [浅析Linux Kernel 哈希路由表实现(二)](http://basiccoder.com/intro-linux-kernel-hash-rt-2.html)
- [Linux kernel路由机制分析](http://lib.csdn.net/article/linux/37220)
- [路由表 rtable](http://abcdxyzk.github.io/blog/2015/08/25/kernel-net-rtable/)
- [路由的基本概念介绍](http://www.pagefault.info/?p=240)
- [The Network Administrators' Guide - Displaying the Routing Table](http://www.tldp.org/LDP/nag/node75.html#SECTION007910000)
