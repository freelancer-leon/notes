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
    /*源 MAC 地址是否属于同一父接口下*/
		src = macvlan_hash_lookup(port, eth->h_source);
		if (src && src->mode != MACVLAN_MODE_VEPA &&
		    src->mode != MACVLAN_MODE_BRIDGE) {
			/* forward to original port. */
      /*发出该多播的 macvlan 接口既不是 bridge 模式也不是 VEPA 模式，
        即 private 模式或 passthru 模式*/
			vlan = src;
      /*将 skb->dev (macvlan 接口) 换为 src->dev (父接口) 然后发回给原端口，
        这样原端口会再次收到自己发的多播包，但接收设备成了父接口*/
			ret = macvlan_broadcast_one(skb, vlan, eth, 0) ?:
			      netif_rx(skb);
			handle_res = RX_HANDLER_CONSUMED;  /*该包处理完毕，消耗掉了*/
			goto out;
		}
    /*如果源 MAC 地址不是同一父接口下，或者是同一父接口且是 bridge 或 VEPA 模式*/
		hash = mc_hash(NULL, eth->h_dest);
    /*如果目的 MAC 地址的 hash 在过滤列表里*/
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
