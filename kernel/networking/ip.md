### drop_unicast_in_l2_multicast
> Drop any unicast IP packets that are received in link-layer multicast (or broadcast) frames.
> This behavior (for multicast) is actually a SHOULD in RFC 1122, but is disabled by default for compatibility reasons.Default: off (0)

* [RFC 1122, 4.1.3.6 Invalid Addresses](https://www.freesoft.org/CIE/RFC/1122/81.htm)
* [[v3,1/4] ipv4: add option to drop unicast encapsulated in L2 multicast](https://patchwork.kernel.org/patch/7559041/)

# References
* [/proc/sys/net/ipv4/* Variables](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
* [Introduction to Linux interfaces for virtual networking](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/)
