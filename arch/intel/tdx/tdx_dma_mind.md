# DMA

## Coherent DMA

### DMA buffer 用 DMA 接口函数分配

```cpp
dma_alloc_coherent(dev, size, dma_handle, gfp)
-> dma_alloc_attrs()
   -> dma_direct_alloc()
      -> dma_set_decrypted()
         -> set_memory_decrypted()
            -> __set_memory_enc_dec(addr, numpages, false) //第三个参数 false 表示解密
               -> x86_platform.guest.enc_status_change_finish(addr, numpages, enc)
               => tdx_enc_status_changed()
                  -> _tdx_hypercall(TDVMCALL_MAP_GPA, start, end - start, 0, 0)
```

### 分配时解决 cache 一致性问题

### DMA buffer 的生命周期通常与 module 一致

## Stream DMA

### DMA buffer 由其他模块提供

#### `dma_map_single(dev, cpu_addr, size, dir)`

### 建立映射时需指明数据在 DMA 通道中的流向

### 每次 DMA 传输重建一个 buffer

#### SWIOTLB bounce buffer

```cpp
start_kernel()
-> mm_init()
-> mem_encrypt_init()
      if (!cc_platform_has(CC_ATTR_MEM_ENCRYPT)) //TD VM 是有这个属性的，见 intel_cc_platform_has()
         return; //所以不会返回，而是往下走，转换 SWIOTLB buffer 为共享内存
   -> swiotlb_update_mem_attributes()
         struct io_tlb_mem *mem = &io_tlb_default_mem;
         vaddr = phys_to_virt(mem->start);
         bytes = PAGE_ALIGN(mem->nslabs << IO_TLB_SHIFT);
      -> set_memory_decrypted((unsigned long)vaddr, bytes >> PAGE_SHIFT);
         -> __set_memory_enc_dec(addr, numpages, false) //第三个参数 false 表示解密
            -> x86_platform.guest.enc_status_change_finish(addr, numpages, enc)
            => tdx_enc_status_changed()
               -> _tdx_hypercall(TDVMCALL_MAP_GPA, start, end - start, 0, 0)
         mem->vaddr = swiotlb_mem_remap(mem, bytes);
```
