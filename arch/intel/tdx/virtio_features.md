# Virtio Features

* 对 Qemu 配置如下 virtio 设备：
```sh
-m ${MEM} \
-object memory-backend-file,id=ramhuge,size=${MEM},mem-path=/dev/hugepages,share=on \
-object memory-backend-memfd-private,id=ram1,size=${MEM},shmemdev=ramhuge \
-object tdx-guest,id=tdx,debug=on,sept-ve-disable=on \
-machine q35,accel=kvm,confidential-guest-support=tdx,memory-backend=ram1 \
...
-device virtio-net-pci,netdev=mynet0,mac=00:16:3E:68:00:10,romfile= \
-netdev user,id=mynet0,hostfwd=tcp::${SSH_PORT}-:22,hostfwd=tcp::${PORT_2375}-:2375 \
-device vhost-vsock-pci,guest-cid=${VSOCK_GUEST_CID} \
-chardev stdio,id=mux,mux=on,logfile=./td-guest.log \
-device virtio-serial,romfile= \
-device virtconsole,chardev=mux -serial chardev:mux -monitor chardev:mux \
-drive file=${GUEST_IMG},if=virtio,format=qcow2,id=disk \
-chardev socket,id=spdk_vhost_scsi0,path=${SPDK_SOCK_PATH}/${VHOST_0} \
-device vhost-user-scsi-pci,id=scsi0,chardev=spdk_vhost_scsi0,num_queues=2 \
-chardev socket,id=spdk_vhost_blk0,path=${SPDK_SOCK_PATH}/${VHOST_1} \
-device vhost-user-blk-pci,chardev=spdk_vhost_blk0,num-queues=2 \
```

* 产生如下 log
```cpp
virtio_bus_device_plugged:55: name=virtio-net has_iommu=1 host_features=0000010239000000
virtio_bus_device_plugged:63: name=virtio-net has_iommu=1 host_features=0000010379000000
virtio_bus_device_plugged:80: name=virtio-net has_iommu=1 host_features=0000010379bf8064 &address_space_memory=0x55c2dfedc940
virtio_bus_device_plugged:91: name=virtio-net vdev_has_iommu=1 host_features=0000010379bf8064 dma_as=0x55c2e0de3c78
vhost_dev_init:1378: vhost_dev=0x55c2e0ef0458 features=000000013d000000
virtio_bus_device_plugged:55: name=vhost-vsock has_iommu=1 host_features=0000010239000000
virtio_bus_device_plugged:63: name=vhost-vsock has_iommu=1 host_features=0000010379000000
virtio_bus_device_plugged:80: name=vhost-vsock has_iommu=1 host_features=0000000379000000 &address_space_memory=0x55c2dfedc940
virtio_bus_device_plugged:91: name=vhost-vsock vdev_has_iommu=1 host_features=0000000379000000 dma_as=0x55c2e0ee7fe8
virtio_bus_device_plugged:55: name=virtio-serial has_iommu=1 host_features=0000010239000000
virtio_bus_device_plugged:63: name=virtio-serial has_iommu=1 host_features=0000010379000000
virtio_bus_device_plugged:80: name=virtio-serial has_iommu=1 host_features=0000010379000006 &address_space_memory=0x55c2dfedc940
virtio_bus_device_plugged:91: name=virtio-serial vdev_has_iommu=1 host_features=0000010379000006 dma_as=0x55c2e0fe8c38
VHOST_CONFIG: (/var/tmp/vhost.0) new vhost user connection is 536
VHOST_CONFIG: (/var/tmp/vhost.0) new device, handle is 0
VHOST_CONFIG: (/var/tmp/vhost.1) new vhost user connection is 541
VHOST_CONFIG: (/var/tmp/vhost.1) new device, handle is 1
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_GET_FEATURES
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_GET_PROTOCOL_FEATURES
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_PROTOCOL_FEATURES
VHOST_CONFIG: (/var/tmp/vhost.0) negotiated Vhost-user protocol features: 0x11cbf
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_GET_QUEUE_NUM
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_BACKEND_REQ_FD
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_OWNER
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_GET_FEATURES
vhost_dev_init:1378: vhost_dev=0x55c2e115b580 features=000000015c000007
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_VRING_CALL
VHOST_CONFIG: (/var/tmp/vhost.0) vring call idx:0 file:543
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_VRING_ERR
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_VRING_CALL
VHOST_CONFIG: (/var/tmp/vhost.0) vring call idx:1 file:544
virtio_bus_device_plugged:55: name=virtio-scsi has_iommu=1 host_features=0000010239000000
virtio_bus_device_plugged:63: name=virtio-scsi has_iommu=1 host_features=0000010379000000
vhost_scsi_common_get_features:126: vsc->host_features=0000000000000006 features=0000010379000000
vhost_scsi_common_get_features:129: vsc->dev=0x55c2e115b580 vsc->dev.features=000000015c000007 features=0000010379000006
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_VRING_CALL
VHOST_CONFIG: (/var/tmp/vhost.0) vring call idx:1 file:544
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_VRING_ERR
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_VRING_CALL
VHOST_CONFIG: (/var/tmp/vhost.0) vring call idx:2 file:545
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_VRING_ERR
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_VRING_CALL
VHOST_CONFIG: (/var/tmp/vhost.0) vring call idx:3 file:546
VHOST_CONFIG: (/var/tmp/vhost.0) read message VHOST_USER_SET_VRING_ERR
virtio_bus_device_plugged:80: name=virtio-scsi has_iommu=1 host_features=0000000358000006 &address_space_memory=0x55c2dfedc940
virtio_bus_device_plugged:91: name=virtio-scsi vdev_has_iommu=1 host_features=0000000358000006 dma_as=0x55c2e1153168
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_GET_FEATURES
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_GET_PROTOCOL_FEATURES
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_SET_PROTOCOL_FEATURES
VHOST_CONFIG: (/var/tmp/vhost.1) negotiated Vhost-user protocol features: 0x11ebf
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_GET_QUEUE_NUM
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_SET_BACKEND_REQ_FD
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_SET_OWNER
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_GET_FEATURES
vhost_dev_init:1378: vhost_dev=0x55c2e1262c98 features=000000015c007646
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_SET_VRING_CALL
VHOST_CONFIG: (/var/tmp/vhost.1) vring call idx:0 file:548
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_SET_VRING_ERR
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_SET_VRING_CALL
VHOST_CONFIG: (/var/tmp/vhost.1) vring call idx:1 file:549
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_SET_VRING_ERR
VHOST_CONFIG: (/var/tmp/vhost.1) read message VHOST_USER_GET_CONFIG
virtio_bus_device_plugged:55: name=virtio-blk has_iommu=1 host_features=0000010239006800
virtio_bus_device_plugged:63: name=virtio-blk has_iommu=1 host_features=0000010379006800
vhost_user_blk_get_features:272: sdev=0x55c2e1262c98 sdev.features=000000015c007646 features=0000010379007e76
virtio_bus_device_plugged:80: name=virtio-blk has_iommu=1 host_features=0000000158007646 &address_space_memory=0x55c2dfedc940
virtio_bus_device_plugged:91: name=virtio-blk vdev_has_iommu=0 host_features=0000000358007646 dma_as=0x55c2e125a898
qemu-system-x86_64: -device vhost-user-blk-pci,chardev=spdk_vhost_blk0,num-queues=2: iommu_platform=true is not supported by the device
VHOST_CONFIG: (/var/tmp/vhost.0) vhost peer closed
VHOST_CONFIG: (/var/tmp/vhost.1) vhost peer closed
```

## Qemu Virtio Device Features
* 对于一个 virtio device，Qemu 给它的缺省 `host_features` 是 `0x0000010239000000`，对于 virtio-blk 是 `0x0000010239006800`
* 通常它由以下两个部分构成

### Qemu Virtio Common Features

* 预定义的 virtio 通用 features，它规定了第 `24`，`27`，`28`，`29`，`33`，`34`，`40` bit，
* `VIRTIO_F_IOMMU_PLATFORM`（bit 33） 和 `VIRTIO_F_RING_RESET`（bit 34），缺省值是 `false` 因此
  * 对于非 TD，`host_features` 是 `0x0100_3900_0000`
  * 对于 TD，`host_features` 是 `0x0102_3900_0000`，后面讲 bit 33 是在哪里设置上的
* qemu/include/hw/virtio/virtio.h
```cpp
#define VIRTIO_LEGACY_FEATURES ((0x1ULL << VIRTIO_F_BAD_FEATURE) | \
                                (0x1ULL << VIRTIO_F_NOTIFY_ON_EMPTY) | \
                                (0x1ULL << VIRTIO_F_ANY_LAYOUT))
...
#define DEFINE_VIRTIO_COMMON_FEATURES(_state, _field) \
    DEFINE_PROP_BIT64("indirect_desc", _state, _field,    \
                      VIRTIO_RING_F_INDIRECT_DESC, true), \
    DEFINE_PROP_BIT64("event_idx", _state, _field,        \
                      VIRTIO_RING_F_EVENT_IDX, true),     \
    DEFINE_PROP_BIT64("notify_on_empty", _state, _field,  \
                      VIRTIO_F_NOTIFY_ON_EMPTY, true), \
    DEFINE_PROP_BIT64("any_layout", _state, _field, \
                      VIRTIO_F_ANY_LAYOUT, true), \
    DEFINE_PROP_BIT64("iommu_platform", _state, _field, \
                      VIRTIO_F_IOMMU_PLATFORM, false), \
    DEFINE_PROP_BIT64("packed", _state, _field, \
                      VIRTIO_F_RING_PACKED, false), \
    DEFINE_PROP_BIT64("queue_reset", _state, _field, \
                      VIRTIO_F_RING_RESET, true)
```
* `virtio_device_class_init()` 初始化的时候设置 `virtio_properties` 的时候，`host_features` 会一并设置
  * qemu/hw/virtio/virtio.c
```cpp
static Property virtio_properties[] = {
    DEFINE_VIRTIO_COMMON_FEATURES(VirtIODevice, host_features),
    DEFINE_PROP_BOOL("use-started", VirtIODevice, use_started, true),
    DEFINE_PROP_BOOL("use-disabled-flag", VirtIODevice, use_disabled_flag, true),
    DEFINE_PROP_BOOL("x-disable-legacy-check", VirtIODevice,
                     disable_legacy_check, false),
    DEFINE_PROP_END_OF_LIST(),
};
...
static void virtio_device_class_init(ObjectClass *klass, void *data)
{
    /* Set the default value here. */
    VirtioDeviceClass *vdc = VIRTIO_DEVICE_CLASS(klass);
    DeviceClass *dc = DEVICE_CLASS(klass);

    dc->realize = virtio_device_realize;
    dc->unrealize = virtio_device_unrealize;
    dc->bus_type = TYPE_VIRTIO_BUS;
    device_class_set_props(dc, virtio_properties);
    vdc->start_ioeventfd = virtio_device_start_ioeventfd_impl;
    vdc->stop_ioeventfd = virtio_device_stop_ioeventfd_impl;
    //注意这里，第 24、27、30 bit 被设置为 legacy_features，这里后面会讲到
    vdc->legacy_features |= VIRTIO_LEGACY_FEATURES;

    QTAILQ_INIT(&virtio_list);
}
```
### Qemu Virtio User Block Features
* 对于 Virtio User Block 会多定义几个 features，分别是第 `11`、`13`、`14` bit，在 `vhost_user_blk_class_init()` 的时候加入
* qemu/hw/block/vhost-user-blk.c
```cpp
static Property vhost_user_blk_properties[] = {
    DEFINE_PROP_CHR("chardev", VHostUserBlk, chardev),
    DEFINE_PROP_UINT16("num-queues", VHostUserBlk, num_queues,
                       VHOST_USER_BLK_AUTO_NUM_QUEUES),
    DEFINE_PROP_UINT32("queue-size", VHostUserBlk, queue_size, 128),
    DEFINE_PROP_BIT64("config-wce", VHostUserBlk, parent_obj.host_features,
                      VIRTIO_BLK_F_CONFIG_WCE, true),
    DEFINE_PROP_BIT64("discard", VHostUserBlk, parent_obj.host_features,
                      VIRTIO_BLK_F_DISCARD, true),
    DEFINE_PROP_BIT64("write-zeroes", VHostUserBlk, parent_obj.host_features,
                      VIRTIO_BLK_F_WRITE_ZEROES, true),
    DEFINE_PROP_END_OF_LIST(),
};

static void vhost_user_blk_class_init(ObjectClass *klass, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);
    VirtioDeviceClass *vdc = VIRTIO_DEVICE_CLASS(klass);

    device_class_set_props(dc, vhost_user_blk_properties);
    dc->vmsd = &vmstate_vhost_user_blk;
    set_bit(DEVICE_CATEGORY_STORAGE, dc->categories);
    vdc->realize = vhost_user_blk_device_realize;
    vdc->unrealize = vhost_user_blk_device_unrealize;
    vdc->get_config = vhost_user_blk_update_config;
    vdc->set_config = vhost_user_blk_set_config;
    vdc->get_features = vhost_user_blk_get_features;
    vdc->set_status = vhost_user_blk_set_status;
    vdc->reset = vhost_user_blk_reset;
    vdc->get_vhost = vhost_user_blk_get_vhost;
}
```
* 因此，对于 virtio-blk 我们会看到
  * `virtio_bus_device_plugged:55: name=virtio-blk has_iommu=1 host_features=0000010239006800`

### Virtio Device Realize 时重整 `host_features`
* `virtio_device_realize()` 阶段会将 SPDK 定义的 features、Qemu 定义的 virtio features、`user_feature_bits` 进行整合，得到最终的 `host_features`
1. 首先会给 SPDK vhost 发消息，获取 SPDK 定义的 `features`
2. `klass->pre_plugged()` 回调，更新 virtio device 的 `host_features`
3. `vdc->get_features()` 回调将以上三种定义进行整合
```cpp
//hw/virtio/virtio.c
virtio_device_realize()
-> vdc->realize(dev, &err)
   //hw/block/vhost-user-blk.c
=> vhost_user_blk_device_realize() //由 hw/block/vhost-user-blk.c::vhost_user_blk_class_init() 设置
   -> vhost_user_init(&s->vhost_user, &s->chardev, errp)
      //hw/virtio/virtio.c
   -> virtio_init(vdev, VIRTIO_ID_BLOCK, config_size)
      -> vdev->name = virtio_id_to_name(device_id);
   -> vhost_user_blk_realize_connect(s, errp)
      -> vhost_user_blk_connect(dev, errp)
         //hw/virtio/vhost.c
      -> vhost_dev_init(&s->dev, &s->vhost_user, VHOST_BACKEND_TYPE_USER, 0, errp)
         -> vhost_set_backend_type(hdev, backend_type)
               case VHOST_BACKEND_TYPE_KERNEL:
                    dev->vhost_ops = &kernel_ops;
               case VHOST_BACKEND_TYPE_USER:
                    dev->vhost_ops = &user_ops; //注意：backend_type 即 VHOST_BACKEND_TYPE_USER
               case VHOST_BACKEND_TYPE_VDPA:
                    dev->vhost_ops = &vdpa_ops;
         -> hdev->vhost_ops->vhost_get_features(hdev, &features);
            //hw/virtio/vhost-user.c
         => vhost_user_get_features() //由 hw/virtio/vhost-user.c::const VhostOps user_ops 定义
            -> vhost_user_get_u64(dev, VHOST_USER_GET_FEATURES, features) //给 SPDK vhost 发消息，获取 SPDK 定义的 features
            hdev->features = features;
            //vhost_dev_init:1378: vhost_dev=0x55f992c62c98 features=000000015c007646
   -> qemu_chr_fe_set_handlers(&s->chardev, vhost_user_blk_event, ...);
-> virtio_bus_device_plugged(vdev, &err);
   //virtio_bus_device_plugged:55: name=virtio-blk has_iommu=1 host_features=0000010239006800
   -> klass->pre_plugged(qbus->parent, &local_err);
      //hw/virtio/virtio-pci.c
   => virtio_pci_pre_plugged() //由 hw/virtio/virtio-pci.c::virtio_pci_bus_class_init() 设置
      -> virtio_add_feature(&vdev->host_features, VIRTIO_F_VERSION_1)   //设置 bit 32
      -> virtio_add_feature(&vdev->host_features, VIRTIO_F_BAD_FEATURE) //设置 bit 30
   //virtio_bus_device_plugged:63: name=virtio-blk has_iommu=1 host_features=0000010379006800
   -> vdev->host_features = vdc->get_features(vdev, vdev->host_features, &local_err);
   => vhost_user_blk_get_features() //由 hw/block/vhost-user-blk.c::vhost_user_blk_class_init() 设置
      -> virtio_add_feature(&features, VIRTIO_BLK_F_SIZE_MAX);
      -> virtio_add_feature(&features, VIRTIO_BLK_F_SEG_MAX);
      -> virtio_add_feature(&features, VIRTIO_BLK_F_GEOMETRY);
      -> virtio_add_feature(&features, VIRTIO_BLK_F_TOPOLOGY);
      -> virtio_add_feature(&features, VIRTIO_BLK_F_BLK_SIZE);
      -> virtio_add_feature(&features, VIRTIO_BLK_F_FLUSH);
      -> virtio_add_feature(&features, VIRTIO_BLK_F_RO);
      -> virtio_add_feature(&features, VIRTIO_BLK_F_MQ);
      //vhost_user_blk_get_features:272: sdev=0x55f992c62c98 sdev.features=000000015c007646 features=0000010379007e76
      -> vhost_get_features(&s->dev, user_feature_bits, features);
   //virtio_bus_device_plugged:80: name=virtio-blk has_iommu=1 host_features=0000000158007646 &address_space_memory=0x55f9918dc940
   -> vdev_has_iommu = virtio_host_has_feature(vdev, VIRTIO_F_IOMMU_PLATFORM);
   -> virtio_add_feature(&vdev->host_features, VIRTIO_F_IOMMU_PLATFORM);
   //virtio_bus_device_plugged:91: name=virtio-blk vdev_has_iommu=0 host_features=0000000358007646 dma_as=0x55f992c5a898
      if (!vdev_has_iommu && vdev->dma_as != &address_space_memory)
      -> error_setg(errp, "iommu_platform=true is not supported by the device");
      //qemu-system-x86_64: -device vhost-user-blk-pci,chardev=spdk_vhost_blk0,num-queues=2: iommu_platform=true is not supported by the device
```
#### Qemu 定义的 `user_feature_bits`
* 对于 user virtio-blk，`user_feature_bits` 是
  * qemu/hw/block/vhost-user-blk.c
```cpp
static const int user_feature_bits[] = {
    VIRTIO_BLK_F_SIZE_MAX,
    VIRTIO_BLK_F_SEG_MAX,
    VIRTIO_BLK_F_GEOMETRY,
    VIRTIO_BLK_F_BLK_SIZE,
    VIRTIO_BLK_F_TOPOLOGY,
    VIRTIO_BLK_F_MQ,
    VIRTIO_BLK_F_RO,
    VIRTIO_BLK_F_FLUSH,
    VIRTIO_BLK_F_CONFIG_WCE,
    VIRTIO_BLK_F_DISCARD,
    VIRTIO_BLK_F_WRITE_ZEROES,
    VIRTIO_F_VERSION_1,
    VIRTIO_RING_F_INDIRECT_DESC,
    VIRTIO_RING_F_EVENT_IDX,
    VIRTIO_F_NOTIFY_ON_EMPTY,
    VIRTIO_F_RING_PACKED,
    VIRTIO_F_IOMMU_PLATFORM,
    VIRTIO_F_RING_RESET,
    VHOST_INVALID_FEATURE_BIT
};
...
static uint64_t vhost_user_blk_get_features(VirtIODevice *vdev,
                                            uint64_t features,
                                            Error **errp)
{
    VHostUserBlk *s = VHOST_USER_BLK(vdev);

    /* Turn on pre-defined features */
    virtio_add_feature(&features, VIRTIO_BLK_F_SIZE_MAX);
    virtio_add_feature(&features, VIRTIO_BLK_F_SEG_MAX);
    virtio_add_feature(&features, VIRTIO_BLK_F_GEOMETRY);
    virtio_add_feature(&features, VIRTIO_BLK_F_TOPOLOGY);
    virtio_add_feature(&features, VIRTIO_BLK_F_BLK_SIZE);
    virtio_add_feature(&features, VIRTIO_BLK_F_FLUSH);
    virtio_add_feature(&features, VIRTIO_BLK_F_RO);

    if (s->num_queues > 1) {
        virtio_add_feature(&features, VIRTIO_BLK_F_MQ);
    }

    return vhost_get_features(&s->dev, user_feature_bits, features);
}
```

* 对于 user virtio-scsi，`user_feature_bits` 是
  * qemu/hw/scsi/vhost-user-scsi.c
```cpp
/* Features supported by the host application */
static const int user_feature_bits[] = {
    VIRTIO_F_NOTIFY_ON_EMPTY,
    VIRTIO_RING_F_INDIRECT_DESC,
    VIRTIO_RING_F_EVENT_IDX,
    VIRTIO_SCSI_F_HOTPLUG,
    VIRTIO_F_RING_RESET,
    VHOST_INVALID_FEATURE_BIT
};
...
static void vhost_user_scsi_class_init(ObjectClass *klass, void *data)
{
...
    vdc->get_features = vhost_scsi_common_get_features;
...
}
```
* qemu/hw/scsi/vhost-scsi-common.c
```cpp
uint64_t vhost_scsi_common_get_features(VirtIODevice *vdev, uint64_t features,
                                        Error **errp)
{
    VHostSCSICommon *vsc = VHOST_SCSI_COMMON(vdev);

    /* Turn on predefined features supported by this device */
    features |= vsc->host_features;

    return vhost_get_features(&vsc->dev, vsc->feature_bits, features);
}
```
#### SPDK 定义的 Features
* spdk/lib/vhost/vhost_internal.h
```cpp
#define SPDK_VHOST_FEATURES ((1ULL << VHOST_F_LOG_ALL) | \
    (1ULL << VHOST_USER_F_PROTOCOL_FEATURES) | \
    (1ULL << VIRTIO_F_VERSION_1) | \
    (1ULL << VIRTIO_F_NOTIFY_ON_EMPTY) | \
    (1ULL << VIRTIO_RING_F_EVENT_IDX) | \
    (1ULL << VIRTIO_RING_F_INDIRECT_DESC) | \
    (1ULL << VIRTIO_F_ANY_LAYOUT))
```
* spdk/lib/vhost/vhost_blk.c
```cpp
/* Minimal set of features supported by every SPDK VHOST-BLK device */
#define SPDK_VHOST_BLK_FEATURES_BASE (SPDK_VHOST_FEATURES | \
        (1ULL << VIRTIO_BLK_F_SIZE_MAX) | (1ULL << VIRTIO_BLK_F_SEG_MAX) | \
        (1ULL << VIRTIO_BLK_F_GEOMETRY) | (1ULL << VIRTIO_BLK_F_BLK_SIZE) | \
        (1ULL << VIRTIO_BLK_F_TOPOLOGY) | (1ULL << VIRTIO_BLK_F_BARRIER)  | \
        (1ULL << VIRTIO_BLK_F_SCSI)     | (1ULL << VIRTIO_BLK_F_CONFIG_WCE) | \
        (1ULL << VIRTIO_BLK_F_MQ))
...
int
spdk_vhost_blk_construct(const char *name, const char *cpumask, const char *dev_name,
             const char *transport, const struct spdk_json_val *params)
{
...
    vdev->virtio_features = SPDK_VHOST_BLK_FEATURES_BASE;
...
}
```
* spdk/lib/vhost/vhost_scsi.c
```cpp
/* Features supported by SPDK VHOST lib. */
#define SPDK_VHOST_SCSI_FEATURES    (SPDK_VHOST_FEATURES | \
                    (1ULL << VIRTIO_SCSI_F_INOUT) | \
                    (1ULL << VIRTIO_SCSI_F_HOTPLUG) | \
                    (1ULL << VIRTIO_SCSI_F_CHANGE ) | \
                    (1ULL << VIRTIO_SCSI_F_T10_PI ))
```

#### 最后的整合函数 `vhost_get_features()`
* 函数 `vhost_get_features()` 将 SPDK 定义的 features、Qemu 定义的 virtio device features、`user_feature_bits` 进行整合
  * `hdev->features`：来自 *SPDK 定义的 features*
  * `feature_bits`：来自 `user_feature_bits`
  * `features`：来自 *Qemu 定义的 virtio device features*
* 遍历 `user_feature_bits`，如果 *SPDK 定义的 features* 也没有，则从 *Qemu 定义的 virtio device features* 剔除该 feature
* **注意**：遍历的主体是 `user_feature_bits`，如果 `user_feature_bits` 未设置的 bit 不受影响，这也是导致 user virt-blk 设置在 TD 场景下报错的地方
  * 对于 user virt-blk 和 user virt-scsi，*SPDK 定义的 features* **都没有设置** `VIRTIO_F_IOMMU_PLATFORM`
  * 对于 user virt-scsi，`user_feature_bits` 没设置 `VIRTIO_F_IOMMU_PLATFORM`，尽管 *SPDK 定义的 features* 未包含 `VIRTIO_F_IOMMU_PLATFORM`，该函数也不会清除该 bit
  * 对于 user virt-blk，`user_feature_bits` 设置了 `VIRTIO_F_IOMMU_PLATFORM`，而 *SPDK 定义的 features* 未包含 `VIRTIO_F_IOMMU_PLATFORM`，因此该函数会 **清除** 该 bit
    * 尽管后来 `virtio_add_feature(&vdev->host_features, VIRTIO_F_IOMMU_PLATFORM)` 也于事无补，因为 Qemu 已经认定该 user virt-blk 不支持 IOMMU platform
* qemu/hw/virtio/vhost.c
```cpp
uint64_t vhost_get_features(struct vhost_dev *hdev, const int *feature_bits,
                            uint64_t features)
{
    const int *bit = feature_bits;
    while (*bit != VHOST_INVALID_FEATURE_BIT) {
        uint64_t bit_mask = (1ULL << *bit);
        if (!(hdev->features & bit_mask)) {
            features &= ~bit_mask;
        }
        bit++;
    }
    return features;
}
```
* qemu/hw/virtio/virtio-bus.c
```cpp
/* A VirtIODevice is being plugged */
void virtio_bus_device_plugged(VirtIODevice *vdev, Error **errp)
{
    DeviceState *qdev = DEVICE(vdev);
    BusState *qbus = BUS(qdev_get_parent_bus(qdev));
    VirtioBusState *bus = VIRTIO_BUS(qbus);
    VirtioBusClass *klass = VIRTIO_BUS_GET_CLASS(bus);
    VirtioDeviceClass *vdc = VIRTIO_DEVICE_GET_CLASS(vdev);
    bool has_iommu = virtio_host_has_feature(vdev, VIRTIO_F_IOMMU_PLATFORM);
    bool vdev_has_iommu;
    Error *local_err = NULL;

    DPRINTF("%s: plug device.\n", qbus->name);
    //由 hw/virtio/virtio-pci.c::virtio_pci_bus_class_init() 设置 bit 30 和 32
    if (klass->pre_plugged != NULL) {
        klass->pre_plugged(qbus->parent, &local_err);
        if (local_err) {
            error_propagate(errp, local_err);
            return;
        }
    }

    /* Get the features of the plugged device. */
    assert(vdc->get_features != NULL);
    vdev->host_features = vdc->get_features(vdev, vdev->host_features,
                                            &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        return;
    }

    if (klass->device_plugged != NULL) {
        klass->device_plugged(qbus->parent, &local_err);
    }
    if (local_err) {
        error_propagate(errp, local_err);
        return;
    }

    vdev->dma_as = &address_space_memory;
    if (has_iommu) {
        vdev_has_iommu = virtio_host_has_feature(vdev, VIRTIO_F_IOMMU_PLATFORM);
        /*
         * Present IOMMU_PLATFORM to the driver iff iommu_plattform=on and
         * device operational. If the driver does not accept IOMMU_PLATFORM
         * we fail the device.
         */
        virtio_add_feature(&vdev->host_features, VIRTIO_F_IOMMU_PLATFORM);
        if (klass->get_dma_as) {
            vdev->dma_as = klass->get_dma_as(qbus->parent);
            if (!vdev_has_iommu && vdev->dma_as != &address_space_memory) {
                error_setg(errp,
                       "iommu_platform=true is not supported by the device");
                return;
            }
        }
    }
}
```

## TD 的 `host_features` 中的 `VIRTIO_F_IOMMU_PLATFORM`

* 对于 TD 的场景，`VIRTIO_F_IOMMU_PLATFORM` 会被强制加到 `host_features` 上，因此，在 TD 场景里，`virtio_bus_device_plugged()` 看到的 `vdev->host_features=0000010239006800` 的第 `33` bit 被设置

```diff
commit 9f88a7a3df11a5aaa6212ea535d40d5f92561683
Author: David Gibson <david@gibson.dropbear.id.au>
Date:   Thu Jun 4 14:20:24 2020 +1000

    confidential guest support: Alter virtio default properties for protected guests

    The default behaviour for virtio devices is not to use the platforms normal
    DMA paths, but instead to use the fact that it's running in a hypervisor
    to directly access guest memory.  That doesn't work if the guest's memory
    is protected from hypervisor access, such as with AMD's SEV or POWER's PEF.

    So, if a confidential guest mechanism is enabled, then apply the
    iommu_platform=on option so it will go through normal DMA mechanisms.
    Those will presumably have some way of marking memory as shared with
    the hypervisor or hardware so that DMA will work.

    Signed-off-by: David Gibson <david@gibson.dropbear.id.au>
    Reviewed-by: Dr. David Alan Gilbert <dgilbert@redhat.com>
    Reviewed-by: Cornelia Huck <cohuck@redhat.com>
    Reviewed-by: Greg Kurz <groug@kaod.org>

diff --git a/hw/core/machine.c b/hw/core/machine.c
index f45a795478..970046f438 100644
--- a/hw/core/machine.c
+++ b/hw/core/machine.c
@@ -33,6 +33,8 @@
 #include "migration/global_state.h"
 #include "migration/vmstate.h"
 #include "exec/confidential-guest-support.h"
+#include "hw/virtio/virtio.h"
+#include "hw/virtio/virtio-pci.h"

 GlobalProperty hw_compat_5_2[] = {};
 const size_t hw_compat_5_2_len = G_N_ELEMENTS(hw_compat_5_2);
@@ -1196,6 +1198,17 @@ void machine_run_board_init(MachineState *machine)
          * areas.
          */
         machine_set_mem_merge(OBJECT(machine), false, &error_abort);
+
+        /*
+         * Virtio devices can't count on directly accessing guest
+         * memory, so they need iommu_platform=on to use normal DMA
+         * mechanisms.  That requires also disabling legacy virtio
+         * support for those virtio pci devices which allow it.
+         */
+        object_register_sugar_prop(TYPE_VIRTIO_PCI, "disable-legacy",
+                                   "on", true);
+        object_register_sugar_prop(TYPE_VIRTIO_DEVICE, "iommu_platform",
+                                   "on", false);
     }

     machine_class->init(machine);
```

## TD Guest 对 Virtio IOMMU 的启用
* TD Guest 在启动的时候通过 `tdx_early_init() -> virtio_set_mem_acc_cb(virtio_require_restricted_mem_acc)` 设置 Virtio 的内存访问回调函数
```diff
commit 83ec258066e469077078e557f446859971cc31c9
Author: Andi Kleen <ak@linux.intel.com>
Date:   Tue Jun 1 23:21:09 2021 -0700

    x86/tdx: Enable virtio restricted memory access for TDX

    In virtio the host decides whether the guest uses the DMA
    API or not using the strangely named VIRTIO_F_ACCESS_PLATFORM
    bit (which really indicates whether the DMA API is used or not)

    For hardened virtio on TDX, it is expected that swiotlb is
    always used, which requires using the DMA API.  While IO wouldn't
    really work without the swiotlb, it might be possible that an
    attacker forces swiotlbless IO to manipulate memory in the guest.

    So TDX has requirement to force the DMA API (which then forces
    swiotlb), but without relying on the host.

    Enable virtio restricted memory access for TDX.

    Signed-off-by: Andi Kleen <ak@linux.intel.com>
    Signed-off-by: Alexander Shishkin <alexander.shishkin@linux.intel.com>

diff --git a/arch/x86/coco/tdx/tdx.c b/arch/x86/coco/tdx/tdx.c
index 00fa8f2a81ae..31e1181d2874 100644
--- a/arch/x86/coco/tdx/tdx.c
+++ b/arch/x86/coco/tdx/tdx.c
@@ -8,6 +8,7 @@
 #include <linux/export.h>
 #include <linux/io.h>
 #include <linux/random.h>
+#include <linux/virtio_anchor.h>
 #include <asm/coco.h>
 #include <asm/tdx.h>
 #include <asm/vmx.h>
@@ -831,6 +832,9 @@ void __init tdx_early_init(void)
         */
        panic_on_oops = 1;

+       /* Set restricted memory access for virtio. */
+       virtio_set_mem_acc_cb(virtio_require_restricted_mem_acc);
+
        cc_set_vendor(CC_VENDOR_INTEL);
        tdx_parse_tdinfo(&cc_mask);
        cc_set_mask(cc_mask);
```
* `virtio_set_mem_acc_cb()` 是设置回调函数的接口
  * include/linux/virtio_anchor.h
```cpp
static inline void virtio_set_mem_acc_cb(bool (*func)(struct virtio_device *))
{
    virtio_check_mem_acc_cb = func;
}
```
* 有两个回调函数可供调用，分别用于 *限制* 和 *不限制* virtio 访问 guest 内存。显然，对于 TD 场景，需要限制对 guest 内存的访问
  * drivers/virtio/virtio_anchor.c
```cpp
bool virtio_require_restricted_mem_acc(struct virtio_device *dev)
{
    return true;
}
EXPORT_SYMBOL_GPL(virtio_require_restricted_mem_acc);

static bool virtio_no_restricted_mem_acc(struct virtio_device *dev)
{
    return false;
}
```
* 于是，在 virtio device probe 的时候检查 features，就可以对此做出判断
  * drivers/virtio/virtio.c
```cpp
/* Do some validation, then set FEATURES_OK */
static int virtio_features_ok(struct virtio_device *dev)
{
    unsigned int status;

    might_sleep();
    //对于 TD，必须检查 virtio device 是否具备以下 features
    if (virtio_check_mem_acc_cb(dev)) {
        if (!virtio_has_feature(dev, VIRTIO_F_VERSION_1)) {
            dev_warn(&dev->dev,
                 "device must provide VIRTIO_F_VERSION_1\n");
            return -ENODEV;
        }

        if (!virtio_has_feature(dev, VIRTIO_F_ACCESS_PLATFORM)) {
            dev_warn(&dev->dev,
                 "device must provide VIRTIO_F_ACCESS_PLATFORM\n");
            return -ENODEV;
        }
    }

    if (!virtio_has_feature(dev, VIRTIO_F_VERSION_1))
        return 0;
...
}
...
static int virtio_dev_probe(struct device *_d)
{
...
    err = virtio_features_ok(dev);
    if (err)
        goto err;
...
}
```

### Legacy VM 对 IOMMU 和 SWIOTLB 的使用
* Legacy VM 的 virtio 设备默认并不使用 IOMMU，如果想让它也使用 IOMMU 需要加上 `iommu_platform=on,disable-legacy=on` 参数，会让 `vq->use_dma_api` 为 `true`
```sh
-device vhost-user-scsi-pci,id=scsi0,chardev=spdk_vhost_scsi0,num_queues=2,iommu_platform=on,disable-legacy=on
```
* 再配合 kernel command line `swiotlb=force` 会让 legacy VM 的 stream DMA 也使用 SWIOTLB bounce buffer，这导致性能下降
* 而对于 TD VM，virtio 设备的 IOMMU 和 SWIOTLB 都是必须的，因此性能会比不用以上参数的 legacy VM 性能差一些（1.1 GB/s -> 945 MB/s）

## 检查 Virtio Device 的 Features
* 例如某 virtio-blk 设备：
```sh
$ cat /sys/class/block/vda/device/features
0110001001101110000000000000000011000000000000000000000000000000
```
* 对应的函数在 `features_show()`
```cpp
//include/linux/virtio_config.h
/**
 * __virtio_test_bit - helper to test feature bits. For use by transports.
 *                     Devices should normally use virtio_has_feature,
 *                     which includes more checks.
 * @vdev: the device
 * @fbit: the feature bit
 */
static inline bool __virtio_test_bit(const struct virtio_device *vdev,
                     unsigned int fbit)
{
    /* Did you forget to fix assumptions on max features? */
    if (__builtin_constant_p(fbit))
        BUILD_BUG_ON(fbit >= 64);
    else
        BUG_ON(fbit >= 64);

    return vdev->features & BIT_ULL(fbit);
}
...
//drivers/virtio/virtio.c
static ssize_t features_show(struct device *_d,
                 struct device_attribute *attr, char *buf)
{
    struct virtio_device *dev = dev_to_virtio(_d);
    unsigned int i;
    ssize_t len = 0;

    /* We actually represent this as a bitstring, as it could be
     * arbitrary length in future. */
    for (i = 0; i < sizeof(dev->features)*8; i++)
        len += sysfs_emit_at(buf, len, "%c",
                   __virtio_test_bit(dev, i) ? '1' : '0');
    len += sysfs_emit_at(buf, len, "\n");
    return len;
}
```
* 显然，对于小端字节序的机器，这里的输出是从第 `0` 位到第 `sizeof(dev->features)*8 - 1 = 63` 位的输出
* 对应到十六进制就是 `0x0003 1000 7646`，为什么对应的不是 Qemu 最后整合出来的 `0x0003 5800 7646` 呢？！第 `27` 和 `30` bit 到哪去了？

## Kernel 读取时再次过滤 Features

### Kernel 读取到的 Features
* 调查发现 `device_features` 是 `dev->features` 的主要来源，此时它就已经是 `0x0003 1000 7646` 了
* drivers/virtio/virtio.c
```cpp
static int virtio_dev_probe(struct device *_d)
{
    int err, i;
    struct virtio_device *dev = dev_to_virtio(_d);
    struct virtio_driver *drv = drv_to_virtio(dev->dev.driver);
    u64 device_features;
    u64 driver_features;
    u64 driver_features_legacy;
...
    /* Figure out what features the device supports. */
    device_features = dev->config->get_features(dev);
...
    if (device_features & (1ULL << VIRTIO_F_VERSION_1))
        dev->features = driver_features & device_features;
    else
        dev->features = driver_features_legacy & device_features;
...
}
```
* 如果它是 virtio_pci_modern 设备，那么 `dev->config->get_features()` 对应到 drivers/virtio/virtio_pci_modern.c:`vp_get_features()`
  * drivers/virtio/virtio_pci_modern.c
```cpp
static u64 vp_get_features(struct virtio_device *vdev)
{
    struct virtio_pci_device *vp_dev = to_vp_device(vdev);

    return vp_modern_get_features(&vp_dev->mdev);
}
...
static const struct virtio_config_ops virtio_pci_config_ops = {
    .get        = vp_get,
    .set        = vp_set,
    .generation = vp_generation,
    .get_status = vp_get_status,
    .set_status = vp_set_status,
    .reset      = vp_reset,
    .find_vqs   = vp_modern_find_vqs,
    .del_vqs    = vp_del_vqs,
    .synchronize_cbs = vp_synchronize_vectors,
    .get_features   = vp_get_features,
    .finalize_features = vp_finalize_features,
    .bus_name   = vp_bus_name,
    .set_vq_affinity = vp_set_vq_affinity,
    .get_vq_affinity = vp_get_vq_affinity,
    .get_shm_region  = vp_get_shm_region,
    .disable_vq_and_reset = vp_modern_disable_vq_and_reset,
    .enable_vq_after_reset = vp_modern_enable_vq_after_reset,
};
```
* `vp_modern_get_features()` 的实现如下：
  * drivers/virtio/virtio_pci_modern_dev.c
```cpp
/*
 * vp_modern_get_features - get features from device
 * @mdev: the modern virtio-pci device
 *
 * Returns the features read from the device
 */
u64 vp_modern_get_features(struct virtio_pci_modern_device *mdev)
{
    struct virtio_pci_common_cfg __iomem *cfg = mdev->common;

    u64 features;

    vp_iowrite32(0, &cfg->device_feature_select);
    features = vp_ioread32(&cfg->device_feature);
    vp_iowrite32(1, &cfg->device_feature_select);
    features |= ((u64)vp_ioread32(&cfg->device_feature) << 32);

    return features;
}
```
* 看来 `cfg->device_feature_select` 和 `cfg->device_feature` 的地址都被映射到了 IO Port 地址，他们都来自 `struct virtio_pci_common_cfg __iomem *cfg = mdev->common`
* 因为一次只能从 IO Port 地址读出 32 位，因此 64 位的 features 只能分两次读，并用 `device_feature_select` 来控制是读高 32 位还是低 32 位
* 最后合并成 64 bits 的 features 返回
* 比如下面一个例子，某个 virtio PCI 设备的 common config capability 在 guest 中的虚拟地址（GVA）为 `0xffa000000008d000`：
```sh
(gdb) p /x cfg
$8 = 0xffa000000008d000
(gdb) p cfg
$9 = (struct virtio_pci_common_cfg *) 0xffa000000008d000
(gdb) p /x *cfg
$10 = {device_feature_select = 0x0, device_feature = 0x30000006, guest_feature_select = 0x0, guest_feature = 0x0, msix_config = 0xffff,
  num_queues = 0x0, device_status = 0x3, config_generation = 0x0, queue_select = 0x0, queue_size = 0x80, queue_msix_vector = 0x0,
  queue_enable = 0x0, queue_notify_off = 0x0, queue_desc_lo = 0x0, queue_desc_hi = 0x0, queue_avail_lo = 0x0, queue_avail_hi = 0x0,
  queue_used_lo = 0x0, queue_used_hi = 0x0}
(gdb) p &cfg->device_feature_select
$11 = (__le32 *) 0xffa000000008d000
(gdb) p &cfg->device_feature
$12 = (__le32 *) 0xffa000000008d004
```
* 从 Qemu 控制台上可以看到它被映射到了 GPA `0x700000008000`
```sh
(qemu) gva2gpa 0xffa000000008d000
gpa: 0x700000008000
```

### Kernel 映射 `virtio_pci_common_cfg` 的空间
* 那么 `struct virtio_pci_common_cfg __iomem *cfg = mdev->common` 这个映射又是怎么来的呢？
* include/uapi/linux/virtio_pci.h
```cpp
/* Fields in VIRTIO_PCI_CAP_COMMON_CFG: */
struct virtio_pci_common_cfg {
    /* About the whole device. */
    __le32 device_feature_select;   /* read-write */
    __le32 device_feature;      /* read-only */
    __le32 guest_feature_select;    /* read-write */
    __le32 guest_feature;       /* read-write */
    __le16 msix_config;     /* read-write */
    __le16 num_queues;      /* read-only */
    __u8 device_status;     /* read-write */
    __u8 config_generation;     /* read-only */
...
};
```
* include/uapi/linux/virtio_pci.h
```cpp
#define VIRTIO_PCI_COMMON_DFSELECT  0
#define VIRTIO_PCI_COMMON_DF        4
```
* drivers/virtio/virtio_pci_modern_dev.c
```cpp
...
//检查结构体 struct virtio_pci_common_cfg 中各个域的偏移，
//比如 device_feature_select 必须是 0，device_feature 必须是 4
/* This is part of the ABI.  Don't screw with it. */
static inline void check_offsets(void)
{
...
    BUILD_BUG_ON(VIRTIO_PCI_COMMON_DFSELECT !=
             offsetof(struct virtio_pci_common_cfg,
                  device_feature_select));
    BUILD_BUG_ON(VIRTIO_PCI_COMMON_DF !=
             offsetof(struct virtio_pci_common_cfg, device_feature));
...
}
...
/*
 * vp_modern_probe: probe the modern virtio pci device, note that the
 * caller is required to enable PCI device before calling this function.
 * @mdev: the modern virtio-pci device
 *
 * Return 0 on succeed otherwise fail
 */
int vp_modern_probe(struct virtio_pci_modern_device *mdev)
{
...
    int err, common, isr, notify, device;
...
    //检查结构体 struct virtio_pci_common_cfg 中各个域的偏移
    check_offsets();
...
    //找到 common config，ISR，notify，device config 几个 capabilities 在 PCI 配置空间中的偏移
    /* check for a common config: if not, use legacy mode (bar 0). */
    common = virtio_pci_find_capability(pci_dev, VIRTIO_PCI_CAP_COMMON_CFG,
                        IORESOURCE_IO | IORESOURCE_MEM,
                        &mdev->modern_bars);
    if (!common) {
        dev_info(&pci_dev->dev,
             "virtio_pci: leaving for legacy driver\n");
        return -ENODEV;
    }

    /* If common is there, these should be too... */
    isr = virtio_pci_find_capability(pci_dev, VIRTIO_PCI_CAP_ISR_CFG,
                     IORESOURCE_IO | IORESOURCE_MEM,
                     &mdev->modern_bars);
    notify = virtio_pci_find_capability(pci_dev, VIRTIO_PCI_CAP_NOTIFY_CFG,
                        IORESOURCE_IO | IORESOURCE_MEM,
                        &mdev->modern_bars);
    if (!isr || !notify) {
        dev_err(&pci_dev->dev,
            "virtio_pci: missing capabilities %i/%i/%i\n",
            common, isr, notify);
        return -EINVAL;
    }
...
    /* Device capability is only mandatory for devices that have
     * device-specific configuration.
     */
    device = virtio_pci_find_capability(pci_dev, VIRTIO_PCI_CAP_DEVICE_CFG,
                        IORESOURCE_IO | IORESOURCE_MEM,
                        &mdev->modern_bars);
...
   //映射 common config 的 IO Port 地址到 mdev->common，ISA，notify，device config 也是类似的
   mdev->common = vp_modern_map_capability(mdev, common,
                      sizeof(struct virtio_pci_common_cfg), 4,
                      0, sizeof(struct virtio_pci_common_cfg),
                      NULL, NULL);
    if (!mdev->common)
        goto err_map_common;
    mdev->isr = vp_modern_map_capability(mdev, isr, sizeof(u8), 1,
                         0, 1,
                         NULL, NULL);
...
    /* We don't know how many VQs we'll map, ahead of the time.
     * If notify length is small, map it all now.
     * Otherwise, map each VQ individually later.
     */
    if ((u64)notify_length + (notify_offset % PAGE_SIZE) <= PAGE_SIZE) {
        mdev->notify_base = vp_modern_map_capability(mdev, notify,
                                 2, 2,
                                 0, notify_length,
                                 &mdev->notify_len,
                                 &mdev->notify_pa);
        if (!mdev->notify_base)
            goto err_map_notify;
    } else {
        mdev->notify_map_cap = notify;
    }

    /* Again, we don't know how much we should map, but PAGE_SIZE
     * is more than enough for all existing devices.
     */
    if (device) {
        mdev->device = vp_modern_map_capability(mdev, device, 0, 4,
                            0, PAGE_SIZE,
                            &mdev->device_len,
                            NULL);
        if (!mdev->device)
            goto err_map_device;
    }

    return 0;
...
}
```
* 再看映射的具体细节 `vp_modern_map_capability()`
  * drivers/virtio/virtio_pci_modern_dev.c
```cpp
/*
 * vp_modern_map_capability - map a part of virtio pci capability
 * @mdev: the modern virtio-pci device
 * @off: offset of the capability
 * @minlen: minimal length of the capability
 * @align: align requirement
 * @start: start from the capability
 * @size: map size
 * @len: the length that is actually mapped
 * @pa: physical address of the capability
 *
 * Returns the io address of for the part of the capability
 */
static void __iomem *
vp_modern_map_capability(struct virtio_pci_modern_device *mdev, int off,
             size_t minlen, u32 align, u32 start, u32 size,
             size_t *len, resource_size_t *pa)
{
    struct pci_dev *dev = mdev->pci_dev;
    u8 bar;
    u32 offset, length;
    void __iomem *p;
    //根据在 PCI 配置空间的偏移，从中读取 capability 在哪一个 BAR，在这个 BAR 中的 offset 和 size（length）的值
    pci_read_config_byte(dev, off + offsetof(struct virtio_pci_cap, bar),
                 &bar);
    pci_read_config_dword(dev, off + offsetof(struct virtio_pci_cap, offset),
                 &offset);
    pci_read_config_dword(dev, off + offsetof(struct virtio_pci_cap, length),
                  &length);
...
    //根据以上配置空间读到的信息，映射一段 IO 地址范围，并返回它的 IO 地址
    p = pci_iomap_range(dev, bar, offset, length);
    if (!p)
        dev_err(&dev->dev,
            "virtio_pci: unable to map virtio %u@%u on bar %i\n",
            length, offset, bar);
    else if (pa)
        *pa = pci_resource_start(dev, bar) + offset;

    return p;
}
```
* 如果对返回地址感到困惑可以读一读 `pci_iomap_range()` 的注释

### Qemu 为 virtio-pci 设备创建的 IO 地址空间
* 比如对于某 virtio-blk 设备，有两个 BAR
  * 其中第二个 BAR 为 `Region 4: Memory at 700000010000 (64-bit, prefetchable) [size=16K]`，地址范围从 `0x700000010000 ~ 0x700000013fff`
```sh
# lspci -vvv
00:05.0 SCSI storage controller: Red Hat, Inc. Virtio block device (rev 01)
        Subsystem: Red Hat, Inc. Device 1100
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupt: pin A routed to IRQ 2
        Region 1: Memory at c0002000 (32-bit, non-prefetchable) [size=4K]
        Region 4: Memory at 700000010000 (64-bit, prefetchable) [size=16K]
        Capabilities: [98] MSI-X: Enable+ Count=3 Masked-
                Vector table: BAR=1 offset=00000000
                PBA: BAR=1 offset=00000800
        Capabilities: [84] Vendor Specific Information: VirtIO: <unknown>
                BAR=0 offset=00000000 size=00000000
        Capabilities: [70] Vendor Specific Information: VirtIO: Notify
                BAR=4 offset=00003000 size=00001000 multiplier=00000004
        Capabilities: [60] Vendor Specific Information: VirtIO: DeviceCfg
                BAR=4 offset=00002000 size=00001000
        Capabilities: [50] Vendor Specific Information: VirtIO: ISR
                BAR=4 offset=00001000 size=00001000
        Capabilities: [40] Vendor Specific Information: VirtIO: CommonCfg
                BAR=4 offset=00000000 size=00001000
        Kernel driver in use: virtio-pci
```
* 对应到 Qemu 控制台中观察到的地址空间就是：
```c
(qemu) info mtree
...
memory-region: pci
  0000000000000000-ffffffffffffffff (prio -1, i/o): pci
...
    0000700000010000-0000700000013fff (prio 1, i/o): virtio-pci
      0000700000010000-0000700000010fff (prio 0, i/o): virtio-pci-common-virtio-blk
      0000700000011000-0000700000011fff (prio 0, i/o): virtio-pci-isr-virtio-blk
      0000700000012000-0000700000012fff (prio 0, i/o): virtio-pci-device-virtio-blk
      0000700000013000-0000700000013fff (prio 0, i/o): virtio-pci-notify-virtio-blk
...
```
* 到这里我们应该意识到 virtio PCI 设备的 device_features 是从其 common config capability 中读出来的
  * capability 中有它的 BAR 索引和在 BAR 中的偏移，就能定位它的位置
  * 再从它的位置读出它各个域的值，比如 device_features
* Guest 读写 IO 地址空间时会 VM exit 到 KVM，KVM 再将控制权转交给 Qemu 来模拟这次读写操作。接下来我们再看看 Qemu 是如何模拟的。
* 首先，Qemu 也定义了同样的 `struct virtio_pci_common_cfg` 结构
  * include/standard-headers/linux/virtio_pci.h
```cpp
/* Common configuration */
#define VIRTIO_PCI_CAP_COMMON_CFG   1
/* Notifications */
#define VIRTIO_PCI_CAP_NOTIFY_CFG   2
/* ISR access */
#define VIRTIO_PCI_CAP_ISR_CFG      3
...
/* Fields in VIRTIO_PCI_CAP_COMMON_CFG: */
struct virtio_pci_common_cfg {
    /* About the whole device. */
    uint32_t device_feature_select; /* read-write */
    uint32_t device_feature;        /* read-only */
    uint32_t guest_feature_select;  /* read-write */
    uint32_t guest_feature;     /* read-write */
    uint16_t msix_config;       /* read-write */
    uint16_t num_queues;        /* read-only */
    uint8_t device_status;      /* read-write */
    uint8_t config_generation;      /* read-only */
...
};
...
#define VIRTIO_PCI_COMMON_DFSELECT  0
#define VIRTIO_PCI_COMMON_DF        4
...
```
* 在将 virtio PCI 设备具现化的时候会定义好 common config 的在 BAR 中的偏移和大小
* hw/virtio/virtio-pci.c
```cpp
static void virtio_pci_realize(PCIDevice *pci_dev, Error **errp)
{
    VirtIOPCIProxy *proxy = VIRTIO_PCI(pci_dev);
...
    /*
     * virtio pci bar layout used by default.
     * subclasses can re-arrange things if needed.
     *
     *   region 0   --  virtio legacy io bar
     *   region 1   --  msi-x bar
     *   region 2   --  virtio modern io bar (off by default)
     *   region 4+5 --  virtio modern memory (64bit) bar
     *
     */
    proxy->legacy_io_bar_idx  = 0;
    proxy->msix_bar_idx       = 1;
    proxy->modern_io_bar_idx  = 2;
    proxy->modern_mem_bar_idx = 4;

    proxy->common.offset = 0x0;
    proxy->common.size = 0x1000;
    proxy->common.type = VIRTIO_PCI_CAP_COMMON_CFG;
...
}
```
* 插入 virtio PCI 设备的时候，通过以下路径将 `virtio-pci-common-virtio-blk` 加入到 `memory-region: pci` 中
```cpp
//qemu/hw/virtio/virtio-pci.c
virtio_pci_device_plugged()
-> virtio_pci_modern_regions_init(proxy, vdev->name)
      static const MemoryRegionOps common_ops = {
          .read = virtio_pci_common_read,
          .write = virtio_pci_common_write,
          .impl = {
              .min_access_size = 1,
              .max_access_size = 4,
          },
          .endianness = DEVICE_LITTLE_ENDIAN,
      };
   -> g_string_printf(name, "virtio-pci-common-%s", vdev_name);
   -> memory_region_init_io(&proxy->common.mr, OBJECT(proxy),
                            &common_ops,
                            proxy,
                            name->str,
                            proxy->common.size);
-> virtio_pci_modern_mem_region_map(proxy, &proxy->common, &cap);
   -> virtio_pci_modern_region_map(proxy, region, cap, &proxy->modern_bar, proxy->modern_mem_bar_idx)
      -> memory_region_add_subregion(mr, region->offset, &region->mr)
      -> virtio_pci_add_mem_cap(proxy, cap)
```
* 这里我们关注的是 `MemoryRegionOps common_ops`，它定义了读写操作的回调函数，比如读操作的回调函数就是 `virtio_pci_common_read()`
* hw/virtio/virtio-pci.c
```cpp
static uint64_t virtio_pci_common_read(void *opaque, hwaddr addr,
                                       unsigned size)
{
    VirtIOPCIProxy *proxy = opaque;
    VirtIODevice *vdev = virtio_bus_get_device(&proxy->bus);
    uint32_t val = 0;
...
    switch (addr) {
    case VIRTIO_PCI_COMMON_DFSELECT:
        val = proxy->dfselect;
        break;
    case VIRTIO_PCI_COMMON_DF:
        if (proxy->dfselect <= 1) {
            VirtioDeviceClass *vdc = VIRTIO_DEVICE_GET_CLASS(vdev);

            val = (vdev->host_features & ~vdc->legacy_features) >>
                (32 * proxy->dfselect);
        }
        break;
...
    }

    return val;
}
```
* 根据约定的偏移，实际的读取操作时 `VIRTIO_PCI_COMMON_DFSELECT`、`VIRTIO_PCI_COMMON_DF` 等等
* 回忆之前 `vdc->legacy_features` 的第 `24`、`27`、`30` bit 已经被设置，这里取反 `~` 和与 `&` 将它们从 `vdev->host_features` 中清除掉了
* 因此，最后 kernel 读到的 `device_features` 就是没有 第 `24`、`27`、`30` bit 的 `0x0003 1000 7646`