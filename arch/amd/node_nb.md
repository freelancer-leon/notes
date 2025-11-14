# AMD Node 与 Northbridge
* AMD Node 概念涵盖多个系统系列，适用于众多子系统和驱动程序。
* 当前，AMD 北桥（Northbridge）代码中承载了过多的 AMD node 功能。但实际上，node 概念的范畴远不止北桥 —— 其应用场景更为广泛。
* AMD Node 是系统级芯片（SoC）内 I/O 设备的物理集合。每个封装（package）中可包含一个或多个 nodes。
* 这些 nodes 可通过 PCI 配置空间（PCI config space）被软件识别。
  * 所有 nodes 均在 `segment 0`，`bus 0` 上枚举。
  * Device（slot）编号范围为 `0x18` 至 `0x1F`（最多支持 `8` 个 nodes）
    * 其中 `0x18` 对应 `node 0`，`0x19` 对应 `node 1`，以此类推。
  * 每个 node 均可作为多功能设备（multi-function device）。
* 在传统系统（legacy systems）中，这些 node 设备代表集成的 Northbridge 功能。
* 在基于 Zen 架构的系统中，这些 node 设备代表 Data Fabric 功能。

### `amd_node_get_func()` helper
* 函数 `amd_node_get_func()` 根据输入的 `node` 和 `func`，通过比较 `pci_dev` 对象的 `domain`、`bus`、`device`、`function` 找到对象
```c
#define MAX_AMD_NUM_NODES   8
#define AMD_NODE0_PCI_SLOT  0x18

struct pci_dev *amd_node_get_func(u16 node, u8 func)
{   //最多支持 8 个 nodes
    if (node >= MAX_AMD_NUM_NODES)
        return NULL;
    //Domain = 0，Bus = 0，Device = 0x18 + node 和传入的 function ID
    return pci_get_domain_bus_and_slot(0, 0, PCI_DEVFN(AMD_NODE0_PCI_SLOT + node, func));
}
```
### AMD Node 数量
* 之前，AMD node 总数是通过使用 PCI ID 搜索并统计 CPU/node 设备来确定的。
  * 每个 node/northbridge 期望有一个 misc 设备，因此之前用遍历 `amd_nb_misc_ids[]` 得到 `misc_count` 作为 `amd_northbridges.num`
```diff
--- a/arch/x86/kernel/amd_nb.c
+++ b/arch/x86/kernel/amd_nb.c
@@ -186,7 +186,6 @@ static int amd_cache_northbridges(void)
    const struct pci_device_id *misc_ids = amd_nb_misc_ids;
    struct pci_dev *misc;
    struct amd_northbridge *nb;
-   u16 misc_count = 0;
    u16 i;

    if (amd_northbridges.num)
@@ -196,25 +195,30 @@ static int amd_cache_northbridges(void)
        misc_ids = hygon_nb_misc_ids;
    }

-   misc = NULL;
-   while ((misc = next_northbridge(misc, misc_ids)))
-       misc_count++;
-
-   if (!misc_count)
-       return -ENODEV;
+   amd_northbridges.num = amd_num_nodes();

-   nb = kcalloc(misc_count, sizeof(struct amd_northbridge), GFP_KERNEL);
+   nb = kcalloc(amd_northbridges.num, sizeof(struct amd_northbridge), GFP_KERNEL);
    if (!nb)
        return -ENOMEM;

    amd_northbridges.nb = nb;
-   amd_northbridges.num = misc_count;
```
* 然而，AMD node 信息已可通过拓扑 CPUID/MSR 获取。因此对拓扑结构的重构优化，将原有的 node 统计代码替换为通过拓扑信息直接计算（node 总数）的简洁逻辑（具体为拓扑信息的乘积运算）。
* 每个 package 的 nodes 数可以通过 `cpuid leaf 0x8000001e` 的 `$ecx` 得到，Max packages 信息可以通过 ACPI 得到
```c
static inline u16 amd_num_nodes(void)
{
    return topology_amd_nodes_per_pkg() * topology_max_packages();
}
```
* 系统中的 AMD northbridge 用一个全局的 `amd_northbridges` 变量来管理
```cpp
struct amd_northbridge_info {
    u16 num;
    u64 flags;
    struct amd_northbridge *nb;
};
```
* 系统中 AMD northbridge 的数量就是上面 `amd_num_nodes()` 返回的数目
```cpp
static struct amd_northbridge_info amd_northbridges;

u16 amd_nb_num(void)
{
    return amd_northbridges.num;
}
EXPORT_SYMBOL_GPL(amd_nb_num);
```
### `misc` 和 `link` 设备
* 每个 AMD northbridge 有 `misc` 和 `link` 设备
```cpp
struct amd_northbridge {
    struct pci_dev *misc;
    struct pci_dev *link;
    struct amd_l3_cache l3_cache;
};
```
* 每个 AMD northbridge 的 `misc` 设备用 Node ID + Function ID `3` 得到，`link` 设备用 Node ID + Function ID `4` 得到
* 每个 AMD northbridge 必须有一个 `misc` 设备，如果没有，那么不要做任何初始化
```c
static int amd_cache_northbridges(void)
{
...
    amd_northbridges.num = amd_num_nodes();

    nb = kcalloc(amd_northbridges.num, sizeof(struct amd_northbridge), GFP_KERNEL);
    if (!nb)
        return -ENOMEM;

    amd_northbridges.nb = nb; 

    for (i = 0; i < amd_northbridges.num; i++) {
        node_to_amd_nb(i)->misc = amd_node_get_func(i, 3); 

        /*
         * Each Northbridge must have a 'misc' device.
         * If not, then uninitialize everything.
         */
        if (!node_to_amd_nb(i)->misc) {
            amd_northbridges.num = 0;
            kfree(nb);
            return -ENODEV;
        }

        node_to_amd_nb(i)->link = amd_node_get_func(i, 4); 
    }   
...
}
```
### `root` 设备
* “根设备”（root device）搜索功能的引入，旨在为 Zen 架构系统提供 SMN（系统管理网络，System Management Network）访问支持。
  * 该设备代表一个 PCIe root complex，与 slot `0x18-0x1F` 处的 “CPU/node” 设备并非同一类型。
* 一个 AMD node 内可能存在多个 PCIe root complex，服务器或高端桌面（HEDT，High-end Desktop）系统等场景即如此。因此，不能简单假设 “root <-> AMD node” 之间是 `1:1` 的对应关系。
* 之前，这一问题通过在搜索过程中跳过 “额外” 的 root complex 来处理。但实际上，硬件会提供 AMD node 的 root device 对应的 PCI 总线号。
* 利用硬件提供的信息获取 root device 的总线号获取 root device 的方法如下：
1. 用之前定义的 `amd_node_get_func()` helper，`Function ID = 0` 取得 Data Fabirc 的实例
2. 从 Data Fabirc 的配置地址空间的偏移 `DF_BLK_INST_CNT` 处读取 `Revision` 域
   * 如果 `Major revision` 是全零，那么总线号所在的偏移是 `DF_CFG_ADDR_CNTL_LEGACY`
   * 如果 `Major revision` 不是零，那么总线号所在的偏移是 `DF_CFG_ADDR_CNTL_DF4`
3. 利用上一步得到的偏移从 DF 的配置地址空间中（`D18F0xXXX [Config Address Control] (DF::CfgAddressCntl)`）获取到的 `Bits [7:0] (SecBusNum)` 为 root device 的总线号
4. 在已知 domain、bus、device、function 的条件下，调用 `pci_get_domain_bus_and_slot(0, bus, 0)` 得到指向 root device 实例的指针
```c
#define DF_BLK_INST_CNT     0x040
#define DF_CFG_ADDR_CNTL_LEGACY 0x084
#define DF_CFG_ADDR_CNTL_DF4    0xC04

#define DF_MAJOR_REVISION   GENMASK(27, 24)

static u16 get_cfg_addr_cntl_offset(struct pci_dev *df_f0)
{
    u32 reg;

    /*
     * Revision fields added for DF4 and later.
     *
     * Major revision of '0' is found pre-DF4. Field is Read-as-Zero.
     */
    if (pci_read_config_dword(df_f0, DF_BLK_INST_CNT, &reg))
        return 0;

    if (reg & DF_MAJOR_REVISION)
        return DF_CFG_ADDR_CNTL_DF4;

    return DF_CFG_ADDR_CNTL_LEGACY;
}

struct pci_dev *amd_node_get_root(u16 node)
{
    struct pci_dev *root;
    u16 cntl_off;
    u8 bus;

    if (!cpu_feature_enabled(X86_FEATURE_ZEN))
        return NULL;

    /*
     * D18F0xXXX [Config Address Control] (DF::CfgAddressCntl)
     * Bits [7:0] (SecBusNum) holds the bus number of the root device for
     * this Data Fabric instance. The segment, device, and function will be 0.
     */
    struct pci_dev *df_f0 __free(pci_dev_put) = amd_node_get_func(node, 0);
    if (!df_f0)
        return NULL;

    cntl_off = get_cfg_addr_cntl_offset(df_f0);
    if (!cntl_off)
        return NULL;

    if (pci_read_config_byte(df_f0, cntl_off, &bus))
        return NULL;

    /* Grab the pointer for the actual root device instance. */
    root = pci_get_domain_bus_and_slot(0, bus, 0);

    pci_dbg(root, "is root for AMD node %u\n", node);
    return root;
}
```
### AMD Node 设备的 Node ID
* 旧的 `amd_pci_dev_to_node_id()` 尝试通过搜索和计数的方式，找到给定 AMD Node 设备的 node ID
```c
static inline u16 amd_pci_dev_to_node_id(struct pci_dev *pdev)
{   
    struct pci_dev *misc;
    int i;

    for (i = 0; i != amd_nb_num(); i++) {
        misc = node_to_amd_nb(i)->misc;
        //比较 Bus ID 和 Device ID，如果都相等，则找到了 node ID
        if (pci_domain_nr(misc->bus) == pci_domain_nr(pdev->bus) &&
            PCI_SLOT(misc->devfn) == PCI_SLOT(pdev->devfn))
            return i;
    }

    WARN(1, "Unable to find Hygon Northbridge id for %s\n", pci_name(pdev));
    return 0; 
}   
```
* 根据最开始的信息，我们看到新的 `amd_pci_dev_to_node_id()` 则通过将 AMD node 设备的 slot 号减去 AMD node 0 的 slot 号得到其所属的 node ID
```c
//include/uapi/linux/pci.h
/*
 * The PCI interface treats multi-function devices as independent
 * devices.  The slot/function address of each device is encoded
 * in a single byte as follows:
 *
 *  7:3 = slot
 *  2:0 = function
 */ 
#define PCI_DEVFN(slot, func)   ((((slot) & 0x1f) << 3) | ((func) & 0x07))
#define PCI_SLOT(devfn)     (((devfn) >> 3) & 0x1f)
#define PCI_FUNC(devfn)     ((devfn) & 0x07)
//drivers/hwmon/k10temp.c
static u16 amd_pci_dev_to_node_id(struct pci_dev *pdev)
{
    return PCI_SLOT(pdev->devfn) - AMD_NODE0_PCI_SLOT;
}
```

### 将 Root 设备 cache 下来
* SMN 初始化时将 root 设备信息 cache 以备将来使用
* `amd_cache_roots()` 根据 AMD node 的数量分配 `amd_roots[]` 指针数组，然后给每个数组元素指向该 node 对应的 root 设备
  * root 设备对象用前面提到的 `amd_node_get_root(node)` 函数得到
```c
static struct pci_dev **amd_roots;

static int amd_cache_roots(void)
{
    u16 node, num_nodes = amd_num_nodes();

    amd_roots = kcalloc(num_nodes, sizeof(*amd_roots), GFP_KERNEL);
    if (!amd_roots)
        return -ENOMEM;

    for (node = 0; node < num_nodes; node++)
        amd_roots[node] = amd_node_get_root(node);

    return 0;
}

static int __init amd_smn_init(void)
{
    int err;

    if (!cpu_feature_enabled(X86_FEATURE_ZEN))
        return 0;

    guard(mutex)(&smn_mutex);

    if (amd_roots)
        return 0;

    err = amd_cache_roots();
    if (err)
        return err;

    return 0;
}

fs_initcall(amd_smn_init);
```
* 将来使用的时候直接用 `node` ID 来索引 `amd_roots[]` 得到 root 设备 
```diff
@@ -135,10 +136,10 @@ static int __amd_smn_rw(u16 node, u32 address, u32 *value, bool write)
 	struct pci_dev *root;
 	int err = -ENODEV;
 
-	if (node >= amd_nb_num())
+	if (node >= amd_num_nodes())
 		return err;
 
-	root = node_to_amd_nb(node)->root;
+	root = amd_roots[node];
 	if (!root)
 		return err;
```
### SMN 读写辅助函数
* 软件可使用的 SMN（系统管理网络）索引/数据对不止一组。这些寄存器的偏移量不同，但遵循相同的通信协议。
* 为 SMN 偏移量定义宏定义（defines），并允许将索引/数据偏移量传递给读写辅助函数（read/write helper function）。
```c
#define SMN_INDEX_OFFSET	0x60
#define SMN_DATA_OFFSET		0x64
...
static int __amd_smn_rw(u8 i_off, u8 d_off, u16 node, u32 address, u32 *value, bool write)
{
    struct pci_dev *root;
    int err = -ENODEV;

    if (node >= amd_num_nodes())
        return err;

    root = amd_roots[node];
    if (!root)
        return err;

    if (!smn_exclusive)
        return err;

    guard(mutex)(&smn_mutex);

    err = pci_write_config_dword(root, i_off, address);
    if (err) {
        pr_warn("Error programming SMN address 0x%x.\n", address);
        return pcibios_err_to_errno(err);
    }

    err = (write ? pci_write_config_dword(root, d_off, *value)
             : pci_read_config_dword(root, d_off, value));

    return pcibios_err_to_errno(err);
}

int __must_check amd_smn_read(u16 node, u32 address, u32 *value)
{
    int err = __amd_smn_rw(SMN_INDEX_OFFSET, SMN_DATA_OFFSET, node, address, value, false);

    if (PCI_POSSIBLE_ERROR(*value)) {
        err = -ENODEV;
        *value = 0;
    }   

    return err;
}
EXPORT_SYMBOL_GPL(amd_smn_read);

int __must_check amd_smn_write(u16 node, u32 address, u32 value)
{
    return __amd_smn_rw(SMN_INDEX_OFFSET, SMN_DATA_OFFSET, node, address, &value, true);
}
EXPORT_SYMBOL_GPL(amd_smn_write);
```
* HSMP 接口只是一个有着不同偏移的 SMN 接口
```c
#define HSMP_INDEX_OFFSET	0xc4
#define HSMP_DATA_OFFSET	0xc8
...
int __must_check amd_smn_hsmp_rdwr(u16 node, u32 address, u32 *value, bool write)
{
    return __amd_smn_rw(HSMP_INDEX_OFFSET, HSMP_DATA_OFFSET, node, address, value, write);
}
EXPORT_SYMBOL_GPL(amd_smn_hsmp_rdwr);
```

### [PATCH v2 15/16] x86/amd_node: Add SMN offsets to exclusive region access

### 访问 SMN 的 debugfs 文件
* 在AMD Zen系统上，有某些寄存器只能通过 SMN 进行访问。
* 引入一个新接口，该接口提供用于访问 SMN 的 debugfs 文件。
  * * 在 `/sys/kernel/debug/amd_smn` 目录下增加 `node`、`address`、`value` 文件节点
  * 由于这会使用户空间能够以不可预测的方式操作硬件，因此在写入时会标记内核为受污染状态
* 引入一个用于启用 debugfs 接口的内核参数 `amd_smn_debugfs_enable`。特意不对此参数进行文档记录，以阻止该接口的使用。

## References
* [[PATCH v2 00/16] AMD NB and SMN rework](https://lore.kernel.org/all/20241206161210.163701-1-yazen.ghannam@amd.com/)