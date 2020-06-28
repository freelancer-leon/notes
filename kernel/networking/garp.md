
# Gratuitous ARP

> A Gratuitous ARP is an ARP packet sent by a node in order to
  spontaneously cause other nodes to update an entry in their ARP
  cache.  A gratuitous ARP MAY use either an ARP Request or an ARP
  Reply packet.  In either case, the ARP Sender Protocol Address and
  ARP Target Protocol Address are both set to the IP address of the
  cache entry to be updated, and the ARP Sender Hardware Address is
  set to the link-layer address to which this cache entry should be
  updated.  When using an ARP Reply packet, the Target Hardware
  Address is also set to the link-layer address to which this cache
  entry should be updated (this field is not used in an ARP Request
  packet).
>
> In either case, for a gratuitous ARP, the ARP packet MUST be
  transmitted as a local broadcast packet on the local link.  As
  specified in [16], any node receiving any ARP packet (Request or
  Reply) MUST update its local ARP cache with the Sender Protocol
  and Hardware Addresses in the ARP packet, if the receiving node
  has an entry for that IP address already in its ARP cache.  This
  requirement in the ARP protocol applies even for ARP Request
  packets, and for ARP Reply packets that do not match any ARP
  Request transmitted by the receiving node.

* [RFC5944](https://tools.ietf.org/html/rfc5944#section-4.6)
* [RFC2002](https://tools.ietf.org/html/rfc2002#section-4.6)
## 特征
* GARP 可以是 *ARP 请求* 也可以是 *ARP 回复*
  * 如果 GARP 是以 *ARP 回复* 出现，**ARP 目的硬件地址** 也应设为要更新的 ARP cache 条目的链路层地址，此时，**ARP 发送者硬件地址** 与 **ARP 目的硬件地址** 相同
  * 如果 GARP 是以 *ARP 请求* 出现，**ARP 目的硬件地址** 不使用
* 源 IP 地址和目的 IP 地址相同，为要更新的 ARP cache 条目的 IP 地址
* **ARP 发送者硬件地址** 需设置为要更新的 ARP cache 条目的链路层地址


## sysctl 参数
> **arp_accept** - BOOLEAN
>
> Define behavior for gratuitous ARP frames who's IP is not already present in the ARP table:
>	0 - don't create new entries in the ARP table
>	1 - create new entries in the ARP table
>
>	Both replies and requests type gratuitous arp will trigger the ARP table to be updated, if this setting is on.
>
>	If the ARP table already contains the IP address of the	gratuitous arp frame, the arp table will be updated regardless if this setting is on or off.

> **arp_notify** - BOOLEAN
	Define mode for notification of address and device changes.
	0 - (default): do nothing
	1 - Generate gratuitous arp requests when device is brought up or hardware address changes.

## 相关处理
* net/ipv4/arp.c
```c
static bool arp_is_garp(struct net *net, struct net_device *dev,
                        int *addr_type, __be16 ar_op,
                        __be32 sip, __be32 tip,
                        unsigned char *sha, unsigned char *tha)
{
        bool is_garp = tip == sip;

        /* Gratuitous ARP _replies_ also require target hwaddr to be
         * the same as source.
         */
        if (is_garp && ar_op == htons(ARPOP_REPLY))
                is_garp =
                        /* IPv4 over IEEE 1394 doesn't provide target
                         * hardware address field in its ARP payload.
                         */
                        tha &&
                        !memcmp(tha, sha, dev->addr_len);

        if (is_garp) {
                *addr_type = inet_addr_type_dev_table(net, dev, sip);
                if (*addr_type != RTN_UNICAST)
                        is_garp = false;
        }
        return is_garp;
}
...__```
```
* ARP packet 中 *源IP* 与 *目的IP* 不相同，肯定不是 GARP，这是个比较 general 的前提
* *源IP* 与 *目的IP* 相同，且 ARP Operation 字段是 `ARPOP_REPLY`
  * 根据协议规定，ARP reply packet 里的 *目的 MAC 地址* 需设置为 *要更新 ARP cache 条目的 MAC 地址*，即 *源 MAC 地址*
  * `memcmp(tha, sha, dev->addr_len)` 要处理的就是这个情况，不符合的不认为是 GARP
* *源IP* 与 *目的IP* 相同，但是是 ARP request
  * 查询 *源IP* 的地址类型，如果不是单播，即`RTN_UNICAST`，则也不认为是 GARP

# References
- [RFC5944](https://tools.ietf.org/html/rfc5944)
- [RFC2002](https://tools.ietf.org/html/rfc2002)
- [/proc/sys/net/ipv4/* Variables](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
- [Gratuitous_ARP - The Wireshark Wiki](https://wiki.wireshark.org/Gratuitous_ARP)
- [图解ARP协议（五）免费ARP：地址冲突了肿么办？](https://zhuanlan.zhihu.com/p/29011567)
