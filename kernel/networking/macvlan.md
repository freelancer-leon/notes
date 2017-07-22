# MACVLAN

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
		/*如果目的 MAC 地址的 hash 在过滤列表里，则将包排回广播队列*/
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
