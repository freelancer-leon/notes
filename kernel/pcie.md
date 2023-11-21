# PCI 和 PCIe

```cpp
//drivers/pci/msi/api.c
pci_alloc_irq_vectors()
-> pci_alloc_irq_vectors_affinity()
      //drivers/pci/msi/msi.c
   -> __pci_enable_msi_range()
      -> msi_capability_init()
            /* Configure MSI capability structure */
            //drivers/pci/msi/irqdomain.c
         -> pci_msi_setup_msi_irqs(dev, nvec, PCI_CAP_ID_MSI)
```
* 配置 PCIe MSI Capabilities 结构的路径如下：
```cpp
//kernel/irq/chip.c
irq_activate_and_startup()
-> irq_activate()
      //kernel/irq/irqdomain.c
   -> irq_domain_activate_irq()
      -> __irq_domain_activate_irq()
         -> domain->ops->activate(domain, irqd, reserve)
            //kernel/irq/msi.c
         => msi_domain_activate()
               //kernel/irq/chip.c
            -> irq_chip_compose_msi_msg(irq_data, msg)
               -> pos->chip->irq_compose_msi_msg(pos, msg)
               //arch/x86/kernel/apic/msi.c
               => dmar_msi_compose_msg()
                     //arch/x86/kernel/apic/apic.c
                  -> __irq_msi_compose_msg(irqd_cfg(data), msg, true) //构造 struct msi_msg
               //kernel/irq/msi.c
            -> irq_chip_write_msi_msg(irq_data, msg)
               -> data->chip->irq_write_msi_msg(data, msg)
                  //drivers/pci/msi/irqdomain.c
               => pci_msi_domain_write_msg()
                  //drivers/pci/msi/msi.c
                  -> __pci_write_msi_msg(desc, msg) //写入 PCI 配置地址空间
                        if (entry->pci.msi_attrib.is_msix)
                           -> pci_write_msg_msix(entry, msg)
                        else
                           -> pci_write_msg_msi(dev, entry, msg)
```

## References
* [PCIE MSI and MSI-X - 知乎](https://zhuanlan.zhihu.com/p/623790570)
* [PCI-E配置MSI中断流程解析](https://developer.aliyun.com/article/48070)
* [60 MSI-X 介绍-CSDN](https://blog.csdn.net/linjiasen/article/details/105858038)