# IOMMU

## Qemu
* hw/vfio/common.c
```cpp
static int vfio_dma_map(VFIOContainer *container, hwaddr iova,
                        ram_addr_t size, void *vaddr, bool readonly)
{
    struct vfio_iommu_type1_dma_map map = {
        .argsz = sizeof(map),
        .flags = VFIO_DMA_MAP_FLAG_READ,
        .vaddr = (__u64)(uintptr_t)vaddr,
        .iova = iova,
        .size = size,
    };

    if (!readonly) {
        map.flags |= VFIO_DMA_MAP_FLAG_WRITE;
    }

    /*
     * Try the mapping, if it fails with EBUSY, unmap the region and try
     * again.  This shouldn't be necessary, but we sometimes see it in
     * the VGA ROM space.
     */
    if (ioctl(container->fd, VFIO_IOMMU_MAP_DMA, &map) == 0 ||
        (errno == EBUSY && vfio_dma_unmap(container, iova, size, NULL) == 0 &&
         ioctl(container->fd, VFIO_IOMMU_MAP_DMA, &map) == 0)) {
        return 0;
    }

    error_report("VFIO_MAP_DMA failed: %s", strerror(errno));
    return -errno;
}
```

## Kernel
```cpp
//drivers/iommu/iommufd/vfio_compat.c
iommufd_vfio_ioctl()
case VFIO_IOMMU_MAP_DMA:
-> iommufd_vfio_map_dma(ictx, cmd, uarg)
      //drivers/iommu/iommufd/io_pagetable.c
   -> iopt_map_user_pages()
      -> iopt_map_pages()
            //drivers/iommu/iommufd/pages.c
         -> iopt_fill_domains_pages()
            -> batch_to_domain()
                  //drivers/iommu/iommu.c
               -> iommu_map()
                  -> __iommu_map()
                     -> __iommu_map_pages()
                        -> ops->map_pages()
                           //drivers/iommu/intel/iommu.c
                        => intel_iommu_map_pages()
                           -> intel_iommu_map(domain, iova, paddr, size, prot, gfp)
                              -> __domain_mapping(dmar_domain, iova >> VTD_PAGE_SHIFT, hpa >> VTD_PAGE_SHIFT, size, prot, gfp)
```

# References
- [Linux x86-64 IOMMU详解（一）——IOMMU简介](https://blog.csdn.net/qq_34719392/article/details/114834467)
- [Linux x86-64 IOMMU详解（二）——SWIOTLB（软件 IOMMU）](https://blog.csdn.net/qq_34719392/article/details/114873284)
- [Linux x86-64 IOMMU详解（三）——Intel IOMMU（硬件 IOMMU）的功能与基本原理](https://blog.csdn.net/qq_34719392/article/details/115374606)
- [Linux x86-64 IOMMU详解（四）——启用Intel IOMMU 的配置](https://blog.csdn.net/qq_34719392/article/details/116153505)
- [Linux x86-64 IOMMU详解（五）——Intel IOMMU 初始化流程](https://blog.csdn.net/qq_34719392/article/details/117563480)
- [Linux x86-64 IOMMU详解（六）——Intel IOMMU 参与下的 DMA Coherent Mapping 流程](https://blog.csdn.net/qq_34719392/article/details/117699839)