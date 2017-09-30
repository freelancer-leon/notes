# MACVLAN

## 创建端口
* drivers/net/macvlan.c
```c
static int macvlan_port_create(struct net_device *dev)
{
  struct macvlan_port *port;
  unsigned int i;
  int err;

  if (dev->type != ARPHRD_ETHER || dev->flags & IFF_LOOPBACK)
    return -EINVAL;

  if (netdev_is_rx_handler_busy(dev))
    return -EBUSY;

  port = kzalloc(sizeof(*port), GFP_KERNEL);
  if (port == NULL)
    return -ENOMEM;

  port->dev = dev;
  ether_addr_copy(port->perm_addr, dev->dev_addr);
  INIT_LIST_HEAD(&port->vlans);
  for (i = 0; i < MACVLAN_HASH_SIZE; i++)
    INIT_HLIST_HEAD(&port->vlan_hash[i]);
  for (i = 0; i < MACVLAN_HASH_SIZE; i++)
    INIT_HLIST_HEAD(&port->vlan_source_hash[i]);

  skb_queue_head_init(&port->bc_queue);
  INIT_WORK(&port->bc_work, macvlan_process_broadcast); /*初始化广播帧处理工作*/

  err = netdev_rx_handler_register(dev, macvlan_handle_frame, port);
  if (err)
    kfree(port);
  else
    dev->priv_flags |= IFF_MACVLAN_PORT;
  return err;
}
```

## 收包处理
### 收包回调
* 网卡驱动将 polling 得到的数据通过`netif_receive_skb()`提交给协议栈，该函数会一直调用到`__netif_receive_skb_core()`这里将处理 skb 相关的设备注册的回调
* drivers/net/macvlan.c
```c
/* called under rcu_read_lock() from netif_receive_skb */
static rx_handler_result_t macvlan_handle_frame(struct sk_buff **pskb)
{
	struct macvlan_port *port;
	struct sk_buff *skb = *pskb;
	const struct ethhdr *eth = eth_hdr(skb);
	const struct macvlan_dev *vlan;
	const struct macvlan_dev *src;
	struct net_device *dev;
	unsigned int len = 0;
	int ret;
	rx_handler_result_t handle_res;

	port = macvlan_port_get_rcu(skb->dev);
	/*如果目的 MAC 地址是多播*/
	if (is_multicast_ether_addr(eth->h_dest)) {
		unsigned int hash;

		skb = ip_check_defrag(dev_net(skb->dev), skb, IP_DEFRAG_MACVLAN);
		if (!skb)
			return RX_HANDLER_CONSUMED;
		*pskb = skb;
		eth = eth_hdr(skb);
		/*转发给源 MAC 地址与处于 source 模式的 MAC 地址相同的 up 起来的 macvlan 接口*/
		macvlan_forward_source(skb, port, eth->h_source);
		/*源 MAC 地址是否属于同一父接口下，且发出该多播的 macvlan 接口既不是 bridge 模式也不是
		  VEPA 模式，即 private 模式或 passthru 模式*/
		src = macvlan_hash_lookup(port, eth->h_source);
		if (src && src->mode != MACVLAN_MODE_VEPA &&
		    src->mode != MACVLAN_MODE_BRIDGE) {
			/* forward to original port. */
			vlan = src;
			/*最后一个参数 local = 0，不会广播该包，并且函数会返回 0，导致数据放回接收队列；
			  将 skb->dev (macvlan 接口) 换为 src->dev (父接口) 然后发回给源接口，
			  这样源接口会再次收到自己发的多播包，但接收设备成了父接口*/
			ret = macvlan_broadcast_one(skb, vlan, eth, 0) ?:
			      netif_rx(skb);
			handle_res = RX_HANDLER_CONSUMED;  /*该包处理完毕，消耗掉了*/
			goto out;
		}
		/*如果源 MAC 地址不是同一父接口下，或者是同一父接口但是 bridge 或 VEPA 模式*/
		hash = mc_hash(NULL, eth->h_dest);
		/*如果目的 MAC 地址的 hash 在过滤列表里，则将包排回 macvlan 广播帧处理工作队列;
		  通过 schedule_work(&port->bc_work) 唤醒工作队列，
			work 处理回调为 macvlan_process_broadcast()*/
		if (test_bit(hash, port->mc_filter))
			macvlan_broadcast_enqueue(port, src, skb);
		/*返回，表示 macvlan 并没有消耗调该包，让后续过程继续处理该包*/
		return RX_HANDLER_PASS;
	}
	/*如果目的 MAC 地址是单播*/
	/*转发给源 MAC 地址与处于 source 模式的 MAC 地址相同的 up 起来的 macvlan 接口*/
	macvlan_forward_source(skb, port, eth->h_source);
	if (port->passthru)
		vlan = list_first_or_null_rcu(&port->vlans,
					      struct macvlan_dev, list);
	else
		vlan = macvlan_hash_lookup(port, eth->h_dest);
	if (vlan == NULL)
		return RX_HANDLER_PASS;

	dev = vlan->dev;
	if (unlikely(!(dev->flags & IFF_UP))) {
		kfree_skb(skb);
		return RX_HANDLER_CONSUMED;
	}
	len = skb->len + ETH_HLEN;
	skb = skb_share_check(skb, GFP_ATOMIC);
	if (!skb) {
		ret = NET_RX_DROP;
		handle_res = RX_HANDLER_CONSUMED;
		goto out;
	}

	*pskb = skb;
	skb->dev = dev;
	skb->pkt_type = PACKET_HOST;

	ret = NET_RX_SUCCESS;
	handle_res = RX_HANDLER_ANOTHER;
out:
	macvlan_count_rx(vlan, len, ret == NET_RX_SUCCESS, false);
	return handle_res;
}
```
### 广播帧工作队列处理回调
* drivers/net/macvlan.c
```c
static void macvlan_process_broadcast(struct work_struct *w)
{
  struct macvlan_port *port = container_of(w, struct macvlan_port,
                                           bc_work);
  struct sk_buff *skb;
  struct sk_buff_head list;

  __skb_queue_head_init(&list);

  spin_lock_bh(&port->bc_queue.lock);
  skb_queue_splice_tail_init(&port->bc_queue, &list);
  spin_unlock_bh(&port->bc_queue.lock);

  while ((skb = __skb_dequeue(&list))) {
    /*macvlan_broadcast_enqueue() 时传入的 macvlan 广播源设备*/
    const struct macvlan_dev *src = MACVLAN_SKB_CB(skb)->src;

    rcu_read_lock();

    if (!src)
      /* frame comes from an external address */
      /*如果源设备为空，说明是来自外部的广播，将该广播发给该口下所有模式的 macvlan 接口*/
      macvlan_broadcast(skb, port, NULL,
                        MACVLAN_MODE_PRIVATE |
                        MACVLAN_MODE_VEPA    |    
                        MACVLAN_MODE_PASSTHRU|
                        MACVLAN_MODE_BRIDGE);
    /*如果源设备不为空，说明是 macvlan 内部的广播，且源设备为 source，passthru，private
      在 macvlan_handle_frame() 时处理了，进不到这个函数*/
    else if (src->mode == MACVLAN_MODE_VEPA)
      /* flood to everyone except source */
      /*如果源设备是 VEPA 模式，则广播给除自己之外的所有 VEPA 和 bridge 的接口*/
      macvlan_broadcast(skb, port, src->dev,
                        MACVLAN_MODE_VEPA |
                        MACVLAN_MODE_BRIDGE);
    else
      /*
       * flood only to VEPA ports, bridge ports
       * already saw the frame on the way out.
       */
      /*如果源设备是 bridge 模式，则广播给除自己之外的所有 VEPA 模式接口。
        birdge 模式接口在发包 macvlan_queue_xmit() 时会被广播到，所有这里无需重复处理。
        然而问题来了，bridge 模式接口发包广播时把帧排回接收队列，顺着接收流程会走到这，而这里
        又不处理，这是不是就是同一父接口下的 bridge mode 的 macvlan 之间 ARP request
        可以收到但不回复的原因？*/
      macvlan_broadcast(skb, port, src->dev,
                        MACVLAN_MODE_VEPA);

    rcu_read_unlock();

    if (src)
      dev_put(src->dev);
    kfree_skb(skb);
  }
}
```

## 发包回调
* drivers/net/macvlan.c
```c
static int macvlan_queue_xmit(struct sk_buff *skb, struct net_device *dev)
{
  const struct macvlan_dev *vlan = netdev_priv(dev);
  const struct macvlan_port *port = vlan->port;
  const struct macvlan_dev *dest;
  /*如果发送 macvlan 是 bridge 模式，单播或多播都会进入以下条件*/
  if (vlan->mode == MACVLAN_MODE_BRIDGE) {
      const struct ethhdr *eth = (void *)skb->data;
      /*如果目的 MAC 地址是多播地址，广播给同一父接口下的 bridge 模式的 macvlan，除了自己；
        此外，该 frame 还要被发往外部*/
      /* send to other bridge ports directly */
      if (is_multicast_ether_addr(eth->h_dest)) {
          macvlan_broadcast(skb, port, dev, MACVLAN_MODE_BRIDGE);
          goto xmit_world; /*发到外部*/
      }
      /*如果目的 MAC 是单播地址，先看看在不在同一父接口下的 macvlan 地址里，且目的 macvlan
        是 bridge 模式；否则该单播直接发往外部*/
      dest = macvlan_hash_lookup(port, eth->h_dest);
      if (dest && dest->mode == MACVLAN_MODE_BRIDGE) {
          /* send to lowerdev first for its network taps */
          /*该单播是发给 bridge 模式的本地 macvlan，先转给底层（父）接口*/
          dev_forward_skb(vlan->lowerdev, skb);
          /*该单播不会再发给外部，在内部消化掉了*/
          return NET_XMIT_SUCCESS;
      }
  }    
  /*内部转发或广播处理结束，发往外部*/
xmit_world:
  skb->dev = vlan->lowerdev;
  return dev_queue_xmit(skb);
}
```

##

## Ping 试验
```
ip link add link ens33 name macvlan0 type macvlan mode bridge
ip link add link ens33 name macvlan1 type macvlan mode bridge
ip link add link ens33 name macvlan2 type macvlan mode vepa
ip link add link ens33 name macvlan3 type macvlan mode private

dhclient -v macvlan0
dhclient -v macvlan1
dhclient -v macvlan2
dhclient -v macvlan3
```

* macvlan0 (bridge) ====ping====> macvlan1 (bridge)
  * macvlan0: see the ARP request (broadcast), forward to local bridge macvlan (macvlan1)
  * macvlan1: see the ARP request (broadcast), it will pass macvlan rx_handler, upper interface ignore this request, so no ARP response

* macvlan0 (bridge) ====ping====> macvlan2 (vepa)
  * macvlan0: see the ARP request (broadcast), go through upper interface, not forward to macvlan2
  * macvlan1: cannot see the ARP request (broadcast)

* macvlan0 (bridge) ====ping====> macvlan3 (private)
  * macvlan0: see ARP the request (broadcast), go through upper interface, not forward to macvlan3
  * macvlan1: cannot see the ARP request (broadcast)

* macvlan[2-3] (non-bridge) ====ping====> macvlan0 (bridge)
  * macvlan[2-3]: see ARP the request (broadcast), go through upper interface, not forward to macvlan0
  * macvlan0: cannot see the ARP request (broadcast)

# References
- [Linux Networking: MAC VLANs and Virtual Ethernets](http://www.pocketnix.org/posts/Linux%20Networking:%20MAC%20VLANs%20and%20Virtual%20Ethernets)
- [commit: 618e1b7482f7a8a4c6c6e8ccbe140e4c331df4e9 - macvlan: implement bridge, VEPA and private mode](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=618e1b7482f7a8a4c6c6e8ccbe140e4c331df4e9)
- [Linux 上虚拟网络与真实网络的映射](https://www.ibm.com/developerworks/cn/linux/1312_xiawc_linuxvirtnet/index.html)
- [图解几个与Linux网络虚拟化相关的虚拟网卡-VETH/MACVLAN/MACVTAP/IPVLAN](http://blog.csdn.net/dog250/article/details/45788279)
- [linux 网络虚拟化： macvlan](http://cizixs.com/2017/02/14/network-virtualization-macvlan)
- [Macvlan Bridge Mode Example Usage](https://docs.docker.com/engine/userguide/networking/get-started-macvlan/#macvlan-bridge-mode-example-usage)
- [About Veth and Macvlan](https://docs.oracle.com/cd/E37670_01/E37355/html/ol_mcvnbr_lxc.html)
- [Docker Networking: macvlan bridge](http://hicu.be/docker-networking-macvlan-bridge-mode-configuration)
- [Edge Virtual Bridging](http://wikibon.org/wiki/v/Edge_Virtual_Bridging)
