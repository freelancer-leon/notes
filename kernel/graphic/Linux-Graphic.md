# 概念

![render->framebuffer->display](pic/graph-r-2-d.png)

## 渲染

* **渲染**：将使用数学描述的图形转化为*像素阵列*，或者叫*像素数组*。
* 像素数组中每个元素是一个*颜色值*或*颜色索引*，对应图像的一个*像素*。
* 渲染分为两种：
  * 软件渲染：渲染过程由CPU完成
  * 硬件渲染：渲染过程由GPU完成，也即*硬件加速*
* 帧缓冲（Framebuffer）
  * 狭义：存储一帧屏幕图像像素数据的存储区。
  * 广义：多个缓冲区的统称。

## 显示

* 图像中一个像素对应屏幕上的一个像素。
* 将图像显示到屏幕的过程就是逐个读取Framebuffer中存储的图像的像素，根据其所代表的颜色值，控制显示器上对应的点显示相应颜色的过程。

### 显示过程经过的几个组件：

* **显示控制器（CRTC）**
  * 读取Framebuffer中的数据
  * 如果存储的是颜色的索引值，还需从颜色映射（Colormap）中查询具体的颜色值
  * 产生同步信号，如水平同步信号（HSYNC）和垂直同步信号（VSYNC）
    - 水平同步信号：通知显示设备开始显示新的一*行*
    - 垂直同步信号：通知显示设备开始显示新的一*帧*
* **编码器（Encoder）**
  * 将从CRTC中读取的数据encoding为适合具体物理接口的编码格式。
* **发射器（Transmitter）**
  * 将编码后的数据转变为物理信号
* **连接器（Connector）**
  * 也被称为Port，如VGA，LVDS等。
  * 直接连接着显示设备，将发射器发出的信号传递给显示设备。

## 显存

* **动态显存技术（Dynamic Video Memory Technology）** 不再是在内存中为GPU开辟专用显存，而是显存和系统按需动态共享整个主存。
* **GART（Graphics Address Remapping Table）**，亦即 **GTT（Graphics Translation Table）**，是动态显存技术的关键，也是GPU直接访问系统内存的关键。
* 整个GTT所能寻址的范围就代表了GPU逻辑寻址空间。
* GTT中的所有表项不必全部映射到物理内存，完全可以按需映射，GPU不再使用的表项指向的内存可以回收给系统使用。
* Kernel的DRM模块设计了特殊的互斥机制，保证CPU和GPU独立寻址物理内存时不会发生冲突。
* **Graphics Stolen Memory**，通过BIOS划分出一块对操作系统不可见，专用于显存的存储区域。
* BIOS负责在*Graphics Stolen Memory*中
  * 建立*GTT表*
  * 初始化*GTT表项*
  * 将*GTT相关信息*，如GTT的基址，写入到GPU的PCI的配置寄存器（PCI Configuration Registers），这样GPU就可以直接找到GTT了。

![GTT](pic/graph-gtt.png)

* 操作系统启动后，显存的分配和回收由操作系统负责，因此操作系统需要访问*GTT*。
* 操作系统从GPU的PCI配置寄存器中获取如GTT的基址等信息。

* drivers/char/agp/intel-gtt.c
```c
...
static const struct intel_gtt_driver i915_gtt_driver = {
    .gen = 3,
    .has_pgtbl_enable = 1,
    .setup = i9xx_setup,
    .cleanup = i9xx_cleanup,
    /* i945 is the last gpu to need phys mem (for overlay and cursors). */
    .write_entry = i830_write_entry,
    .dma_mask_size = 32,
    .check_flags = i830_check_flags,
    .chipset_flush = i9xx_chipset_flush,
};
...*
static const struct intel_gtt_driver ironlake_gtt_driver = {
    .gen = 5,
    .is_ironlake = 1,
    .setup = i9xx_setup,
    .cleanup = i9xx_cleanup,
    .write_entry = i965_write_entry,
    .dma_mask_size = 36,
    .check_flags = i830_check_flags,
    .chipset_flush = i9xx_chipset_flush,
};
...

static int i9xx_setup(void)
{
    phys_addr_t reg_addr;
    int size = KB(512);

    reg_addr = pci_resource_start(intel_private.pcidev, I915_MMADR_BAR);

    intel_private.registers = ioremap(reg_addr, size);
    if (!intel_private.registers)
        return -ENOMEM;

    switch (INTEL_GTT_GEN) {
    case 3:
        intel_private.gtt_phys_addr =
            pci_resource_start(intel_private.pcidev, I915_PTE_BAR);
        break;
    case 5:
        intel_private.gtt_phys_addr = reg_addr + MB(2);
        break;
    default:
        intel_private.gtt_phys_addr = reg_addr + KB(512);
        break;
    }

    intel_i9xx_setup_flush();

    return 0;
}
...
```

## Buffer Object
