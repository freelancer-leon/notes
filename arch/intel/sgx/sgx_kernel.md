# SGX 内核部分
* Software Guard eXtensions（SGX）硬件使用户空间应用程序能够留出代码和数据的私有内存区域：
  * Privileged（ring-0）`ENCLS` functions 协调区域的构建
  * Unprivileged（ring-3）`ENCLU` functions 允许应用程序进入并在区域内执行
* 这些内存区域称为 **enclave**
  * Enclave 只能在一组固定的入口点进入
  * 每个入口点一次可以容纳一个硬件线程
* 虽然使用`ENCLS` functions 从常规二进制文件加载 enclave，但只有 enclave 内的线程可以访问其内存
* CPU 拒绝外部对该区域进行访问，并在离开 LLC 之前对其进行加密

## Enclave Page Cache
* SGX 利用 Enclave Page Cache（EPC）来存储与 enclave 关联的页面，它包含在 BIOS 保留的物理内存区域中
* 与用于常规内存的页面不同，页面只能在 enclave 构建期间使用特殊的受限 SGX 指令从 enclave 外部访问
* 只有在 enclave 内执行的 CPU 才能直接访问 enclave 内存
* 但是，在 enclave 内执行的 CPU 可以访问 enclave 外的正常内存
* 内核管理 enclave 内存的方式与其处理设备内存的方式类似

### Enclave 页面类型
#### SGX Enclave 控制结构 SGX Enclave Control Structure（SECS）
* Enclave 的地址范围、属性和其他全局数据由该结构定义
* SECS 是一个特殊的 enclave 页面，在地址空间中不可见
* 事实上，它是为任何 enclave 创建的第一个 EPC 页面
* 它通过`ENCLS[ECREATE]` function 从临时 buffer 复制到 EPC
  * `ENCLS[ECREATE]` 是 enclave 构建过程中执行的第一条指令
  * `ECREATE` 将 EPC 外部的 SECS 结构复制到 EPC 内部的 SECS 页中
  * 软件无法访问 SECS 的内部结构
  * 见 Intel SDM Vol. 3D 37-30
```cpp
/**
 * struct sgx_secs - SGX Enclave Control Structure (SECS)
 * @size:       size of the address space
 * @base:       base address of the  address space
 * @ssa_frame_size: size of an SSA frame
 * @miscselect:     additional information stored to an SSA frame
 * @attributes:     attributes for enclave
 * @xfrm:       XSave-Feature Request Mask (subset of XCR0)
 * @mrenclave:      SHA256-hash of the enclave contents
 * @mrsigner:       SHA256-hash of the public key used to sign the SIGSTRUCT
 * @config_id:      a user-defined value that is used in key derivation
 * @isv_prod_id:    a user-defined value that is used in key derivation
 * @isv_svn:        a user-defined value that is used in key derivation
 * @config_svn:     a user-defined value that is used in key derivation
 *
 * SGX Enclave Control Structure (SECS) is a special enclave page that is not
 * visible in the address space. In fact, this structure defines the address
 * range and other global attributes for the enclave and it is the first EPC
 * page created for any enclave. It is moved from a temporary buffer to an EPC
 * by the means of ENCLS[ECREATE] function.
 */
struct sgx_secs {
    u64 size;           //地址空间的大小
    u64 base;           //地址空间的基址
    u32 ssa_frame_size; //SSA frame 的大小
    u32 miscselect;     //存储在 SSA frame 的额外信息
    u8  reserved1[24];
    u64 attributes;     //enclave 的属性
    u64 xfrm;           //XSave-Feature 请求掩码
    u32 mrenclave[8];   //enclave 内容的 SHA256-hash
    u8  reserved2[32];
    u32 mrsigner[8];    //用于签名 SIGSTRUCT 的公钥的 SHA256-hash
    u8  reserved3[32];
    u32 config_id[16];
    u16 isv_prod_id;
    u16 isv_svn;
    u16 config_svn;
    u8  reserved4[3834];
} __packed;
```
#### 常规（REG）
* 常规 EPC 页面包含 enclave 的代码和数据
#### 线程控制结构 Thread Control Structure（TCS）
* 线程控制结构页面定义 enclave 的入口点并跟踪 enclave 线程的执行状态
* TCS 是在其地址空间中可见的 enclave 页，它定义了 enclave 内的入口点
* 线程通过将 TCS 的地址提供给`ENCLU(EENTER)`进入 enclave 内部
* 一次只能由一个线程进入 TCS
```cpp
/**
 * struct sgx_tcs - Thread Control Structure (TCS)
 * @state:      used to mark an entered TCS
 * @flags:      execution flags (cleared by EADD)
 * @ssa_offset:     SSA stack offset relative to the enclave base
 * @ssa_index:      the current SSA frame index (cleard by EADD)
 * @nr_ssa_frames:  the number of frame in the SSA stack
 * @entry_offset:   entry point offset relative to the enclave base
 * @exit_addr:      address outside the enclave to exit on an exception or
 *          interrupt
 * @fs_offset:      offset relative to the enclave base to become FS
 *          segment inside the enclave
 * @gs_offset:      offset relative to the enclave base to become GS
 *          segment inside the enclave
 * @fs_limit:       size to become a new FS-limit (only 32-bit enclaves)
 * @gs_limit:       size to become a new GS-limit (only 32-bit enclaves)
 *
 * Thread Control Structure (TCS) is an enclave page visible in its address
 * space that defines an entry point inside the enclave. A thread enters inside
 * an enclave by supplying address of TCS to ENCLU(EENTER). A TCS can be entered
 * by only one thread at a time.
 */
struct sgx_tcs {
    u64 state;
    u64 flags;         //执行 flags（由`EADD`清除）
    u64 ssa_offset;    //SSA 栈相对于 enclave 基址的偏移
    u32 ssa_index;     //当前 SSA frame 的索引（由`EADD`清除）
    u32 nr_ssa_frames; //在 SSA 栈中 frame 的数目
    u64 entry_offset;  //入口点相对于 enclave 基址的偏移
    u64 exit_addr;     //enclave 从异常或中断退出时在 enclave 外的地址
    u64 fs_offset;
    u64 gs_offset;
    u32 fs_limit;
    u32 gs_limit;
    u8  reserved[SGX_TCS_RESERVED_SIZE];
} __packed;
```
#### 版本数组 Version Array（VA）
* 版本数组页面包含`512`个 slots，每个 slots 可以包含一个从 EPC 中 evicted 出来的页面的版本号
* 一个 EPC 页面是 4 KiB，也就是说每个 slot 的长度是 8 字节

### Enclave Page Cache Map
* 处理器在称为 **Enclave Page Cache Map（EPCM）** 的硬件元数据结构中跟踪 EPC 页面
  * EPCM 包含每个 EPC page 的 entry，其中描述了页面是否已被使用、该页的拥有者、页面类型、地址映射和权限属性等
  * EPCM 中关于权限的管理与普通页表的权限是分开的。例如，这可以阻止内核允许把数据写入 enclave 希望保持只读的页面
  * EPCM 权限只能对普通 x86 页面权限施加额外的限制
* 出于所有意图和目的，SGX 架构允许处理器随意 *使所有 EPCM entries 无效*
  * 这要求软件随时准备处理 EPCM fault
  * 在实践中，这可能发生在电源转换等事件时，加密 enclave 内存的临时密钥丢失的时候

## 应用程序接口

### Enclave 构建函数
* 除了传统的编译器和链接器构建过程之外，SGX 还有一个单独的 enclave “构建”过程
  * 必须先构建 enclave，然后才能执行（进入）它们
* 构建 enclave 的第一步是打开 `/dev/sgx_enclave` 设备
* 由于 enclave 内存受到保护，无法直接访问，因此使用特殊特权指令将数据复制到 enclave 页面并建立 enclave 页面权限
#### `long sgx_ioc_enclave_create(struct sgx_encl *encl, void __user *arg)`
* 为 enclave 分配内核数据结构并调用`ECREATE`
  ```cpp
  static long sgx_ioc_enclave_create(struct sgx_encl *encl, void __user *arg)
  {
      struct sgx_enclave_create create_arg;
      void *secs;
      int ret;

      if (test_bit(SGX_ENCL_CREATED, &encl->flags))
          return -EINVAL;

      if (copy_from_user(&create_arg, arg, sizeof(create_arg)))
          return -EFAULT;

      secs = kmalloc(PAGE_SIZE, GFP_KERNEL);
      if (!secs)
          return -ENOMEM;
      //secs 指针需要提供类型为 struct sgx_secs 的 enclave 控制结构的数据
      if (copy_from_user(secs, (void __user *)create_arg.src, PAGE_SIZE))
          ret = -EFAULT;
      else
          ret = sgx_encl_create(encl, secs); //详解见下文

      kfree(secs);
      return ret;
  }
  ```

#### `long sgx_ioc_enclave_add_pages(struct sgx_encl *encl, void __user *arg)`
* 将一个或多个页面添加到未初始化的 enclave，并可选择使用页面内容扩展度量
  * SECINFO 和度量掩码应用于所有页面
* TCS 的 SECINFO 必须始终包含零权限，因为 CPU 会默默地将它们归零
  * 允许其他任何事情都会导致度量不匹配
* `mmap()`的保护位受页面权限的限制。对于每个页面地址，使用以下启发式计算最大保护位：
  1. 常规页面：`PROT_R`、`PROT_W` 和 `PROT_X` 匹配 SECINFO 权限
  2. TCS页面：`PROT_R | PROT_W`
* `mmap()`不允许超过给定地址范围内最大保护位的最小值
* 该函数取消初始化 enclave 的内核数据结构并在以下任何条件下返回`-EIO`：
  * Enclave Page Cache（EPC），即持有 enclave 的物理内存，已经失效。这将导致`EADD`和`EEXTEND`失败
  * 如果在执行`EADD`时源地址以某种方式损坏
#### `long sgx_ioc_enclave_init(struct sgx_encl *encl, void __user *arg)`
* 刷新任何未完成的排队`EADD`操作并执行`EINIT`
* Launch Enclave Public Key Hash MSR 会根据需要重写，以匹配 enclave 的`MRSIGNER`，`MRSIGNER`是从提供的 sigstruct 计算出来的
#### `long sgx_ioc_enclave_provision(struct sgx_encl *encl, void __user *arg)`
* 通过向`/dev/sgx_provision`提供文件句柄来允许 enclave 的`ATTRIBUTE.PROVISION_KEY`

### Enclave 运行时管理
* 支持 SGX2 的系统还支持更改已初始化的 enclave：
  * 修改 enclave 页面权限和类型，
  * 动态添加和删除 enclave 页面。
* 当 enclave 访问其地址范围内没有后备页面的地址时，新的常规页面将被动态添加到 enclave。但 enclave 仍然需要在新页面上运行`EACCEPT`才能使用。
#### `long sgx_ioc_enclave_restrict_permissions(struct sgx_encl *encl, void __user *arg)`
* 处理`SGX_IOC_ENCLAVE_RESTRICT_PERMISSIONS`
* SGX2 区分放开和限制由硬件维护的 enclave 页面权限（EPCM 权限），该页面属于已初始化（在`SGX_IOC_ENCLAVE_INIT`之后）的 enclave
* EPCM 权限不能在 enclave 内进行限制，enclave 要求内核运行特权级 0 指令`ENCLS[EMODPR]`和`ENCLS[ETRACK]`
  * 通过此调用放宽 EPCM 权限的尝试将被硬件忽略

#### `long sgx_ioc_enclave_modify_types(struct sgx_encl *encl, void __user *arg)`
* 处理`SGX_IOC_ENCLAVE_MODIFY_TYPES`
* 更改 enclave 页面类型的能力以支持以下用例：
  * 通过将常规页面（`SGX_PAGE_TYPE_REG`）的类型更改为 TCS（`SGX_PAGE_TYPE_TCS`）页面，可以将 TCS 页面添加到 enclave。有了这个支持，可以动态增加已初始化的 enclave 所支持的线程数。
  * 通过将页面类型更改为`SGX_PAGE_TYPE_TRIM`，可以从已初始化的 enclave 中动态删除常规或 TCS 页面。将页面类型更改为`SGX_PAGE_TYPE_TRIM`标记要删除的页面，实际上删除动作由`ioctl()`的`SGX_IOC_ENCLAVE_REMOVE_PAGES`的处理程序完成，在 enclave 中`ENCLU[EACCEPT]`运行，作用于`SGX_PAGE_TYPE_TRIM`页面，随后调用该处理程序。
#### `long sgx_ioc_enclave_remove_pages(struct sgx_encl *encl, void __user *arg)`
* 从初始化的 enclave 中删除页面的流程的最后一步。完整的流程是：
  1. 用户使用`SGX_IOC_ENCLAVE_MODIFY_TYPES`的`ioctl()`将要删除的页面类型更改为`SGX_PAGE_TYPE_TRIM`。
  2. 用户通过在 enclave 内运行`ENCLU[EACCEPT]`来批准页面删除。
  3. 用户使用`SGX_IOC_ENCLAVE_REMOVE_PAGES`的`ioctl()`启动实际的页面删除。
* 首先删除任何指向该页的页表条目，然后继续实际删除 enclave 页和支持它的数据。
* VA 页面不受此删除的影响。因此，enclave 结束的时候，可能会有以比支持其所有页面所需的更多的 VA 页面。

### Enclave vDSO
* 进入 enclave 只能通过 SGX 特定的`EENTER`和`ERESUME` function 完成，并且不是一个简单的过程。
  * 由于与 enclave 之间转换的复杂性，enclave 通常使用库来处理实际的转换。这大致类似于大多数应用程序如何使用 glibc 实现来包装系统调用。
* enclave 的另一个关键特征是它们可以生成异常作为其正常操作的一部分，这些异常需要在 enclave 中处理或者是 SGX 独有的。
* 与处理这些异常的传统信号机制不同，SGX 可以利用 vDSO 提供的特殊异常修复。内核提供的 vDSO function 封装了进出 enclave 的底层转换，例如`EENTER`和`ERESUME`。
  * vDSO 函数拦截异常，否则这些异常会生成信号并将故障信息直接返回给其调用者。这避免了摆弄处理信号处理程序。
#### vdso_sgx_enter_enclave_t
* Typedef: `__vdso_sgx_enter_enclave()`的原型，进入一个 SGX enclave 的 vDSO 函数。
* 注意：`__vdso_sgx_enter_enclave()`不能确保完全符合 x86-64 ABI，例如，不处理`XSAVE`状态。除了非易失性通用寄存器、`EFLAGS.DF`和`RSP`对齐之外，根据 x86-64 ABI 保持/设置状态是 enclave 及其运行时的责任，即`__vdso_sgx_enter_enclave()`不能从 C 代码中调用，除非 enclave 及其运行时都仔细考虑。
* 除了`RAX`、`RBX`和`RCX`之外的所有通用寄存器都按原样传递到 enclave。`RAX`、`RBX`和`RCX`由`EENTER`和`ERESUME`使用，并分别加载函数、异步退出指针和`run.tcs`。
* `RBP`和栈用于将`__vdso_sgx_enter_enclave()`锚定到 pre-enclave 状态，例如 在 enclave 退出后检索`run.exception`和`run.user_handler`。所有其他寄存器可供 enclave 及其运行时使用，例如，enclave 可以将附加数据压栈（并修改`RSP`）以将信息传递给可选的用户处理程序（见下文）。
* 在`ENCLU`上报告的大多数异常，包括在 enclave 内发生的异常，都已修复并同步报告，而不是通过标准信号传递。调试异常 (`#DB`) 和断点 (`#BP`) 永远不会修复，总是通过标准信号传递。在同步报告的异常上，返回`-EFAULT`并且有关异常的详细信息记录在`run.exception`和可选的`sgx_enclave_exception`结构中。

## 启动控制
* SGX 提供启动控制机制。在所有 enclave 页面都被复制后，内核执行`EINIT`函数，该函数初始化 enclave。只有在此之后，CPU 才能在 enclave 内执行。
* `EINIT`函数采用 enclave 度量的 RSA-3072 签名。该函数检查度量值是否正确，并使用散列到四个`IA32_SGXLEPUBKEYHASH{0, 1, 2, 3}` MSR 的 key 对 signature 进行签名，这些 MSR 表示公钥的 SHA256 哈希值。
* 这些 MSR 可由 BIOS 配置为可读或可写。Linux 仅支持可写配置，以便在启动控制策略上完全控制内核。在调用`EINIT`函数之前，驱动程序设置 MSR 以匹配 enclave 的签名 key。

## 加密引擎
* 从 Icelake 开始的 CPU 使用全内存加密 (TME) 代替 MEE。
  * 基于 TME 的 SGX 实现没有完整性 Merkle 树，这意味着完整性和重放攻击不会得到缓解。
  * 但它包括额外的更改以防止返回密文和创建的软件内存别名。
* MEE 和 TME 系统上的范围寄存器阻止 DMA 到 enclave 内存。
## 使用模式

### 共享库

### 应用程序容器

## 潜在的内核 SGX bug 的影响
### EPC 泄漏

## 数据结构

### struct sgx_numa_node
* 包含具有 EPC 页面的 NUMA 节点的跟踪数据
* 最重要的是，*节点本地的空闲页列表* 存储在这里
```cpp
/*
 * Contains the tracking data for NUMA nodes having EPC pages. Most importantly,
 * the free page list local to the node is stored here.
 */
struct sgx_numa_node {
    struct list_head free_page_list;       //节点本地的空闲页列表
    struct list_head sgx_poison_page_list; //节点本地的 poison 页列表
    unsigned long size;                    //节点的 EPC 页的大小的总和
    spinlock_t lock;
};
```
* `sgx_page_cache_init()`根据 NUMA 节点的个数分配空间

### struct sgx_epc_section
* Firmware 可以为物理内存的不同区域定义多个 EPC chunks，例如 用于每个节点的内存区域
* 此结构用于存储一个 EPC section 的 EPC 页和已映射页的（内核）虚拟内存区域
```cpp
/*
 * The firmware can define multiple chunks of EPC to the different areas of the
 * physical memory e.g. for memory areas of the each node. This structure is
 * used to store EPC pages for one EPC section and virtual memory area where
 * the pages have been mapped.
 */
struct sgx_epc_section {
    unsigned long phys_addr;    //该 section 的起始物理地址
    void *virt_addr;            //该 section 在内核 remapped 后的起始虚拟地址
    struct sgx_epc_page *pages; //指向 section 的 EPC 页面的管理结构数组
    struct sgx_numa_node *node; //该 section 所属的 struct sgx_numa_node
};
```
* `phys_addr`：`sgx_page_cache_init()`通过枚举`CPUID(SGX_CPUID, i + SGX_CPUID_EPC)`得到每个 section 的 起始物理地址和大小
* `virt_addr`：`sgx_setup_epc_section()`把每个 section 的 EPC pages 在内核中建立映射，用此域记录起始虚拟地址

### 初始化 SGX 相关的数据结构
```cpp
struct sgx_epc_section sgx_epc_sections[SGX_MAX_EPC_SECTIONS];
...
// Node 掩码，当该 node 有多于 1 个 EPC section 时置位
/* Nodes with one or more EPC sections. */
static nodemask_t sgx_numa_mask;

/*
 * Array with one list_head for each possible NUMA node.  Each
 * list contains all the sgx_epc_section's which are on that
 * node.
 */
static struct sgx_numa_node *sgx_numa_nodes;
...
/**
 * A section metric is concatenated in a way that @low bits 12-31 define the
 * bits 12-31 of the metric and @high bits 0-19 define the bits 32-51 of the
 * metric.
 */
static inline u64 __init sgx_calc_section_metric(u64 low, u64 high)
{
    return (low & GENMASK_ULL(31, 12)) +        //取 12-31 bits
           ((high & GENMASK_ULL(19, 0)) << 32); //取 32-51 bits
}
...
static bool __init sgx_setup_epc_section(u64 phys_addr, u64 size,
                     unsigned long index,
                     struct sgx_epc_section *section)
{
    unsigned long nr_pages = size >> PAGE_SHIFT; //EPC 页都是 4K，这里将大小转为 section 里 EPC 页的数目
    unsigned long i;
    //把每个 section 的 EPC pages 在内核中建立映射，记录起始虚拟地址
    section->virt_addr = memremap(phys_addr, size, MEMREMAP_WB);
    if (!section->virt_addr)
        return false;
    //分配 EPC 页面的管理结构数组
    section->pages = vmalloc(nr_pages * sizeof(struct sgx_epc_page));
    if (!section->pages) {
        memunmap(section->virt_addr);
        return false;
    }
    //记录每个 section 的 EPC pages 起始物理地址
    section->phys_addr = phys_addr;
    //以 section 的物理地址范围为 key，把 section 加入 xarray sgx_epc_address_space
    xa_store_range(&sgx_epc_address_space, section->phys_addr,
               phys_addr + size - 1, section, GFP_KERNEL);
    //初始化该 section 中的每个 EPC 页面
    for (i = 0; i < nr_pages; i++) {
        section->pages[i].section = index;
        section->pages[i].flags = 0;
        section->pages[i].owner = NULL; //暂时没有使用，owner 为空
        section->pages[i].poison = 0;
        list_add_tail(&section->pages[i].list, &sgx_dirty_page_list); //初始都在 sgx_dirty_page_list 上
    }

    return true;
}

static bool __init sgx_page_cache_init(void)
{
    u32 eax, ebx, ecx, edx, type;
    u64 pa, size;
    int nid;
    int i;
    //根据 NUMA 节点的个数分配 sgx_numa_nodes 的空间
    sgx_numa_nodes = kmalloc_array(num_possible_nodes(), sizeof(*sgx_numa_nodes), GFP_KERNEL);
    if (!sgx_numa_nodes)
        return false;
    //sgx_epc_sections[] 默认是 8 个元素
    for (i = 0; i < ARRAY_SIZE(sgx_epc_sections); i++) {
        cpuid_count(SGX_CPUID, i + SGX_CPUID_EPC, &eax, &ebx, &ecx, &edx);

        type = eax & SGX_CPUID_EPC_MASK;
        if (type == SGX_CPUID_EPC_INVALID) //是否是无效的 EPC section
            break;

        if (type != SGX_CPUID_EPC_SECTION) { //是否不是有效的 EPC section
            pr_err_once("Unknown EPC section type: %u\n", type);
            break;
        }
        //根据 CPUID(SGX_CPUID, i + SGX_CPUID_EPC) 得到每个 section 的 起始物理地址和大小
        pa   = sgx_calc_section_metric(eax, ebx); //该 section 的 EPC 的起始物理地址，0~51 bits
        size = sgx_calc_section_metric(ecx, edx); //该 section 的 EPC 的大小

        pr_info("EPC section 0x%llx-0x%llx\n", pa, pa + size - 1);
        //初始化每一个 section 结构的实例
        if (!sgx_setup_epc_section(pa, size, i, &sgx_epc_sections[i])) {
            pr_err("No free memory for an EPC section\n");
            break;
        }
        //根据物理地址找到其所属的 NUMA node，如果没有，则返回最近的 node
        nid = numa_map_to_online_node(phys_to_target_node(pa));
        if (nid == NUMA_NO_NODE) {
            /* The physical address is already printed above. */
            pr_warn(FW_BUG "Unable to map EPC section to online node. Fallback to the NUMA node 0.\n");
            nid = 0;
        }
        // 初始化 sgx_numa_nodes 数组元素，如果该 node 还未初始化过
        if (!node_isset(nid, sgx_numa_mask)) { //如果该 section 所属的 NUMA node 掩码未设置
            spin_lock_init(&sgx_numa_nodes[nid].lock);
            INIT_LIST_HEAD(&sgx_numa_nodes[nid].free_page_list);
            INIT_LIST_HEAD(&sgx_numa_nodes[nid].sgx_poison_page_list);
            node_set(nid, sgx_numa_mask); //设置该 section 所属的 NUMA node 掩码，下次不再进这个条件了
            sgx_numa_nodes[nid].size = 0;

            /* Make SGX-specific node sysfs files visible: */
            arch_update_sysfs_visibility(nid);
        }
        // 设置 section 结构的实例其他域的值
        sgx_epc_sections[i].node =  &sgx_numa_nodes[nid];
        sgx_numa_nodes[nid].size += size; //累加 section 的 EPC 页面的大小

        sgx_nr_epc_sections++; //累加实际的 EPC sections 的计数
    }

    if (!sgx_nr_epc_sections) {
        pr_err("There are zero EPC sections.\n");
        return false;
    }

    return true;
}
```
### struct sgx_va_page
* SDM Vol. 3D 34-19
* 为了安全地存储被驱逐的 EPC 页面的版本，SGX 定义了一种特殊的 EPC 页面类型，称为 **Version Array（VA）**
* 每个 VA 页面（`4KB`），包含`512`个 slots，每个 slot 为每个从 EPC 中逐出的页面包含一个`8 byte`的版本号
* 当 EPC 页面被驱逐时，软件会在 VA 页面中选择一个空 slot；此 slot 接收被驱逐页面的唯一版本号
* 重新加载 EPC 页面时，必须有一个 VA slot 来保存页面的版本
  * 如果页面重新加载成功，则清除 VA slot 中的版本
* VA 页面也可以被驱逐，就像任何其他 EPC 页面一样
  * 当驱逐一个 VA 页面时，必须使用某个其他 VA 页面中的版本 slot 来保存被驱逐的 VA 的版本
* Version Array 页面必须是`4K`字节对齐的
* `EPA` leaf function 在 EPC 页中创建一个空的 version array，其逻辑地址由`DS:RCX`给出，并为该页设置 EPCM 属性（SDM Vol. 3D 37-67）
```cpp
#define SGX_VA_SLOT_COUNT 512

struct sgx_va_page {
    struct sgx_epc_page *epc_page;
    DECLARE_BITMAP(slots, SGX_VA_SLOT_COUNT); //8 个 u64
    struct list_head list;
};
```
### struct sgx_encl
* arch/x86/kernel/cpu/sgx/encl.h
  ```cpp
  struct sgx_encl {
      unsigned long base;
      unsigned long size;
      unsigned long flags;
      unsigned int page_cnt;
      unsigned int secs_child_cnt;
      struct mutex lock;
      struct xarray page_array;
      struct sgx_encl_page secs;
      unsigned long attributes;
      unsigned long attributes_mask;

      cpumask_t cpumask;
      struct file *backing;
      struct kref refcount;
      struct list_head va_pages;
      unsigned long mm_list_version;
      struct list_head mm_list;
      spinlock_t mm_lock;
      struct srcu_struct srcu;
  }
  ```
* 空间在`sgx_open()`的时候由`kzalloc()`分配，即打开`/dev/sgx_enclave`文件的时候
  * `struct file *file->private_data = encl`
* `GENMASK_ULL(h, l)`创建一个连续的位掩码，从`l`bit 开始到 `h`bit 结束
  * 比如`GENMASK_ULL(39, 21)`创建了`64bit`的向量`0x000000ffffe00000`
* `base`：用户态创建 enclave 时传入的的虚拟地址基址
* `secs`：SGX Enclave 控制结构的专属 enclave page，指向 enclave 的第一个 EPC 页面，该页面在创建 enclave 时分配
  * `secs`不会加入`struct xarray page_array`
* `secs_child_cnt`：enclave 中的实际 EPC 页面数
  * *添加新 EPC 页面* 和 *换入 EPC 页面* 时增加
  * *释放 EPC 页面* 和 *换出 EPC 页面* 时减小
* arch/x86/kernel/cpu/sgx/driver.c
```cpp
static const struct file_operations sgx_encl_fops = {
    .owner          = THIS_MODULE,
    .open           = sgx_open,
    .release        = sgx_release,
    .unlocked_ioctl     = sgx_ioctl,
#ifdef CONFIG_COMPAT
    .compat_ioctl       = sgx_compat_ioctl,
#endif
    .mmap           = sgx_mmap,
    .get_unmapped_area  = sgx_get_unmapped_area,
};

static struct miscdevice sgx_dev_enclave = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "sgx_enclave",
    .nodename = "sgx_enclave",
    .fops = &sgx_encl_fops,
};

int __init sgx_drv_init(void)
{
    ...
    ret = misc_register(&sgx_dev_enclave);
    if (ret)
        return ret;

    return 0;
}
```
* `misc_register(&sgx_dev_enclave)`这就创建了`/dev/sgx_enclave`和`/sys/class/misc/sgx_enclave`目录
* 打开`/dev/sgx_enclave`的时候就调用`sgx_open()`给`struct sgx_encl encl`分配空间了
```cpp
static int sgx_open(struct inode *inode, struct file *file)
{
    struct sgx_encl *encl;
    int ret;
    //encl 分配空间
    encl = kzalloc(sizeof(*encl), GFP_KERNEL);
    if (!encl)
        return -ENOMEM;
    //初始化 encl 各个成员
    kref_init(&encl->refcount);
    xa_init(&encl->page_array);
    mutex_init(&encl->lock);
    INIT_LIST_HEAD(&encl->va_pages);
    INIT_LIST_HEAD(&encl->mm_list);
    spin_lock_init(&encl->mm_lock);

    ret = init_srcu_struct(&encl->srcu);
    if (ret) {
        kfree(encl);
        return ret;
    }
    //这个赋值很重要，之后需要它来根据文件找到 encl
    file->private_data = encl;

    return 0;
}
```

## Enclave 的创建
1. 先分配一个 `struct sgx_va_page` 管理结构，因为至少需要一个 VA 页面对应到 enclave 的 SECS 页面
2. 借用一个 EPC 页面管理结构，对应到刚分配的 VA 页面
3. 调用`EPA`将 VA 页面加入到 enclave
4. 将 VA 页面的管理结构挂到 enclave 的 `va_pages` 链表上，通过这个管理结构可以找到关联的 EPC 页面管理结构
5. 分配一个 per-enclave 的名为 `SGX backing` 共享内存作为换出 EPC 页面时的后备存储
6. 借用一个 EPC 页面管理结构，对应 enclave 的 SECS 页面，并用
7. 设置 enclave SECS 的管理结构`struct sgx_encl_page`的各个域，该管理结构嵌在 enclave 的`struct sgx_encl`的管理结构中，因此无需重新分配
```c
/* 'desc' bits holding the offset in the VA (version array) page. */
#define SGX_ENCL_PAGE_VA_OFFSET_MASK    GENMASK_ULL(11, 3)
```
* arch/x86/kernel/cpu/sgx/ioctl.c
```cpp
struct sgx_va_page *sgx_encl_grow(struct sgx_encl *encl, bool reclaim)
{
    struct sgx_va_page *va_page = NULL;
    void *err;

    BUILD_BUG_ON(SGX_VA_SLOT_COUNT !=
        (SGX_ENCL_PAGE_VA_OFFSET_MASK >> 3) + 1);
    //每当 enclave 的页计数满 512 就分配一个 va_page，计数为 0 的时候分配第一个
    if (!(encl->page_cnt % SGX_VA_SLOT_COUNT)) {
        va_page = kzalloc(sizeof(*va_page), GFP_KERNEL);
        if (!va_page)
            return ERR_PTR(-ENOMEM);

        va_page->epc_page = sgx_alloc_va_page(reclaim);
        if (IS_ERR(va_page->epc_page)) {
            err = ERR_CAST(va_page->epc_page);
            kfree(va_page);
            return err;
        }

        WARN_ON_ONCE(encl->page_cnt % SGX_VA_SLOT_COUNT);
    }
    encl->page_cnt++; //无论是否需要分配 va_page，都会增加 enclave 页计数
    return va_page;
}
// 该函数通常作为 sgx_encl_grow() 的对称函数出现，但也不是绝对
static void sgx_encl_shrink(struct sgx_encl *encl, struct sgx_va_page *va_page)
{
    encl->page_cnt--;

    if (va_page) {
        sgx_encl_free_epc_page(va_page->epc_page);
        list_del(&va_page->list);
        kfree(va_page);
    }
}

static int sgx_encl_create(struct sgx_encl *encl, struct sgx_secs *secs)
{
    struct sgx_epc_page *secs_epc;
    struct sgx_va_page *va_page;
    struct sgx_pageinfo pginfo;
    struct sgx_secinfo secinfo;
    unsigned long encl_size;
    struct file *backing;
    long ret;
    //先分配 VA 页面，允许触发 EPC 回收
    va_page = sgx_encl_grow(encl, true);
    if (IS_ERR(va_page)) //va_page 可以为 NULL，但不能出错
        return PTR_ERR(va_page); //出错则返回错误
    else if (va_page)    //va_page 不为空，每当 enclave 的页计数满 512 就分配一个 va_page
        list_add(&va_page->list, &encl->va_pages);
    /* else the tail page of the VA page list had free slots. */
    //每创建一个 enclave 时需要一个额外的 EPC 页面做 SGX enclave 控制结构（SECS）
    /* The extra page goes to SECS. */
    encl_size = secs->size + PAGE_SIZE;
    //每个页面（4kB）在后备存储中对应一个关联的 PCMD 页面（128 B），比例 32:1
    //故 enclave 的大小除以 2^5 = 32 即为 PCMD 页面大小的总和
    backing = shmem_file_setup("SGX backing", encl_size + (encl_size >> 5),
                   VM_NORESERVE);
    if (IS_ERR(backing)) {
        ret = PTR_ERR(backing);
        goto err_out_shrink;
    }

    encl->backing = backing;
    //SGX Enclave 控制结构，enclave 创建的第一个 EPC 页面，owner 是它自己（类型为 struct sgx_encl_page）
    secs_epc = sgx_alloc_epc_page(&encl->secs, true);
    if (IS_ERR(secs_epc)) {
        ret = PTR_ERR(secs_epc);
        goto err_out_backing;
    }

    encl->secs.epc_page = secs_epc;

    pginfo.addr = 0;
    pginfo.contents = (unsigned long)secs;
    pginfo.metadata = (unsigned long)&secinfo;
    pginfo.secs = 0;
    memset(&secinfo, 0, sizeof(secinfo));
    //调用 ENCLS[ECREATE] 指令将 EPC 外部的 SECS 结构复制到 EPC 内部的 SECS 页中
    ret = __ecreate((void *)&pginfo, sgx_get_epc_virt_addr(secs_epc));
    if (ret) {
        ret = -EIO;
        goto err_out;
    }
    if (secs->attributes & SGX_ATTR_DEBUG)
        set_bit(SGX_ENCL_DEBUG, &encl->flags);
    //encl 的很多信息来自传入的 struct sgx_secs 实参
    encl->secs.encl = encl;
    encl->secs.type = SGX_PAGE_TYPE_SECS;
    encl->base = secs->base; //用户态创建 enclave 时传入的的虚拟地址基址
    encl->size = secs->size;
    encl->attributes = secs->attributes;
    encl->attributes_mask = SGX_ATTR_DEBUG | SGX_ATTR_MODE64BIT | SGX_ATTR_KSS;

    /* Set only after completion, as encl->lock has not been taken. */
    set_bit(SGX_ENCL_CREATED, &encl->flags);

    return 0;

err_out:
    sgx_encl_free_epc_page(encl->secs.epc_page);
    encl->secs.epc_page = NULL;

err_out_backing:
    fput(encl->backing);
    encl->backing = NULL;

err_out_shrink:
    sgx_encl_shrink(encl, va_page); //如果创建不成功，递减页面计数，va_page 也要释放

    return ret;
}
```
* `sgx_alloc_va_page()`分配一个 EPC 页面，并将它转化位 Version Array（VA）page
```cpp
struct sgx_epc_page *sgx_alloc_va_page(bool reclaim)
{
    struct sgx_epc_page *epc_page;
    int ret;
    //分配一个 EPC 页面，owner 为 NULL，如有必要可触发页面回收
    epc_page = sgx_alloc_epc_page(NULL, reclaim);
    if (IS_ERR(epc_page))
        return ERR_CAST(epc_page);
    //根据 EPC 页面得到其内核虚拟地址，然后通过 EPA leaf function 在 EPC 页中创建一个空的 version array
    ret = __epa(sgx_get_epc_virt_addr(epc_page));
    if (ret) { //如果出错了
        WARN_ONCE(1, "EPA returned %d (0x%x)", ret, ret);
        sgx_encl_free_epc_page(epc_page); //释放刚才分配的 EPC 页面
        return ERR_PTR(-EFAULT);
    }

    return epc_page;
}
```
### struct sgx_encl_page
```cpp
struct sgx_encl_page {
    unsigned long desc;
    unsigned long vm_max_prot_bits:8;
    enum sgx_page_type type:16;
    struct sgx_epc_page *epc_page;
    struct sgx_encl *encl;
    struct sgx_va_page *va_page;
};
```
* `desc`：使用该页面的用户态虚拟地址
  * 在回收时，该地址的低位 bit 被复用：
    * `11~3` bits 存放的是该页面在 VA（version array）page 中的偏移
    * bit `3` 还被复用做页面是否正在被回收的标识：`SGX_ENCL_PAGE_BEING_RECLAIMED`
* `sgx_encl_page`页表的类型有以下几种：
```cpp
/**
 * enum sgx_page_type - bits in the SECINFO flags defining the page type
 * %SGX_PAGE_TYPE_SECS: a SECS page
 * %SGX_PAGE_TYPE_TCS:  a TCS page
 * %SGX_PAGE_TYPE_REG:  a regular page
 * %SGX_PAGE_TYPE_VA:   a VA page
 * %SGX_PAGE_TYPE_TRIM: a page in trimmed state
 *
 * Make sure when making changes to this enum that its values can still fit
 * in the bitfield within &struct sgx_encl_page
 */
enum sgx_page_type {
    SGX_PAGE_TYPE_SECS,
    SGX_PAGE_TYPE_TCS,
    SGX_PAGE_TYPE_REG,
    SGX_PAGE_TYPE_VA,
    SGX_PAGE_TYPE_TRIM,
};
```

## EPC 页面的分配和添加

### EPC 页面的分配
* arch/x86/kernel/cpu/sgx/main.c
* `__sgx_alloc_epc_page_from_node()`页面分配最底层的函数，从指定的 NUMA node 的 free list 上找到并返回空闲 EPC 页面
* `struct sgx_epc_page page`的空间在初始化的时候就已经用`vmalloc()`固定分配好了，所以这里所谓的分配，不过是把 EPC page 的管理结构从 free list 上摘下来，并没有新的内存分配的行为产生
```cpp
static struct sgx_epc_page *__sgx_alloc_epc_page_from_node(int nid)
{
    struct sgx_numa_node *node = &sgx_numa_nodes[nid];
    struct sgx_epc_page *page = NULL;

    spin_lock(&node->lock);
    //如果 node 的 free list 已经空了，返回 NULL
    if (list_empty(&node->free_page_list)) {
        spin_unlock(&node->lock);
        return NULL;
    }
    //返回 free list 上第一个 EPC page 管理结构
    page = list_first_entry(&node->free_page_list, struct sgx_epc_page, list);
    list_del_init(&page->list);
    page->flags = 0;

    spin_unlock(&node->lock);
    atomic_long_dec(&sgx_nr_free_pages); //全局空闲 EPC pages 计数递减

    return page;
}
```
* `__sgx_alloc_epc_page(void)`迭代各 NUMA node，找到并返回一个空闲 EPC 页面
```cpp
/**
 * __sgx_alloc_epc_page() - Allocate an EPC page
 *
 * Iterate through NUMA nodes and reserve ia free EPC page to the caller. Start
 * from the NUMA node, where the caller is executing.
 *
 * Return:
 * - an EPC page:   A borrowed EPC pages were available.
 * - NULL:      Out of EPC pages.
 */
struct sgx_epc_page *__sgx_alloc_epc_page(void)
{
    struct sgx_epc_page *page;
    int nid_of_current = numa_node_id(); //得到当前 node 号
    int nid = nid_of_current;
    //检查 sgx_numa_mask 可知当前 NUMA node 是否有 sgx_epc_section
    if (node_isset(nid_of_current, sgx_numa_mask)) {
        page = __sgx_alloc_epc_page_from_node(nid_of_current);
        if (page)
            return page;
    }
    //当前 NUMA node 没有 sgx_epc_section，回退到其他 NUMA node 去分配
    /* Fall back to the non-local NUMA nodes: */
    while (true) {
        nid = next_node_in(nid, sgx_numa_mask);
        if (nid == nid_of_current) //找了一圈，没有
            break;                 //退出吧，别找了

        page = __sgx_alloc_epc_page_from_node(nid);
        if (page)
            return page;
    }

    return ERR_PTR(-ENOMEM);
}
```
* `sgx_alloc_epc_page()`分配一个 EPC 页面，返回指向其管理结构`struct sgx_epc_page`的指针
  * 遍历 EPC sections 并借给调用者一个空闲的 EPC 页面
  * 当不再需要某个页面时，必须使用`sgx_free_epc_page()`将其释放
  * 如果参数`reclaim`设置为`true`，当没有页面时直接回收页面
    * 此时，不能锁定任何 mm
  * 最后，当页面数低于低水标`SGX_NR_LOW_PAGES`时，在返回调用者之前，唤醒`ksgxd`
```cpp
/**
 * sgx_alloc_epc_page() - Allocate an EPC page
 * @owner:  the owner of the EPC page
 * @reclaim:    reclaim pages if necessary
 *
 * Iterate through EPC sections and borrow a free EPC page to the caller. When a
 * page is no longer needed it must be released with sgx_free_epc_page(). If
 * @reclaim is set to true, directly reclaim pages when we are out of pages. No
 * mm's can be locked when @reclaim is set to true.
 *
 * Finally, wake up ksgxd when the number of pages goes below the watermark
 * before returning back to the caller.
 *
 * Return:
 *   an EPC page,
 *   -errno on error
 */
struct sgx_epc_page *sgx_alloc_epc_page(void *owner, bool reclaim)
{
    struct sgx_epc_page *page;

    for ( ; ; ) {
        page = __sgx_alloc_epc_page(); //尝试分配一个新的 EPC 页面
        if (!IS_ERR(page)) {
            page->owner = owner; //设置 EPC 页面的 owner
            break;
        }
        //刚才没有分配到，尝试回收。先检查 active page list 上有没有页面？
        if (list_empty(&sgx_active_page_list))
            return ERR_PTR(-ENOMEM);
        //如果 active page pool 上没有新的 EPC 页面，又不允许回收，分配失败
        if (!reclaim) {
            page = ERR_PTR(-EBUSY);
            break;
        }
        //检查当前是否有信号未处理
        if (signal_pending(current)) {
            page = ERR_PTR(-ERESTARTSYS);
            break;
        }

        sgx_reclaim_pages();
        cond_resched(); //显然，如果不允许回收，该函数不适合放在原子上下文
    }
    //全局的空闲页低于低水标时，唤醒 ksgxd 进程进行 EPC 页回收
    if (sgx_should_reclaim(SGX_NR_LOW_PAGES))
        wake_up(&ksgxd_waitq);

    return page;
}
```
### EPC 页面的添加
```cpp
sgx_ioc_enclave_add_pages()
-> sgx_validate_secinfo(&secinfo)
   for (c = 0 ; c < add_arg.length; c += PAGE_SIZE)
       -> sgx_encl_add_page(encl, add_arg.src + c, add_arg.offset + c, &secinfo, add_arg.flags)
          -> encl_page = sgx_encl_page_alloc(encl, offset, secinfo->flags)
                         -> encl_page = kzalloc(sizeof(*encl_page), GFP_KERNEL)
                            encl_page->desc = encl->base + offset;
                            encl_page->encl = encl;
                         -> encl_page->vm_max_prot_bits = calc_vm_prot_bits(prot, 0)
          -> epc_page = sgx_alloc_epc_page(encl_page, true)
                        -> page = __sgx_alloc_epc_page()
                           page->owner = owner;
          -> va_page = sgx_encl_grow(encl)
                       -> va_page->epc_page = sgx_alloc_va_page()
                          -> epc_page = sgx_alloc_epc_page(NULL, true)
                          -> __epa(sgx_get_epc_virt_addr(epc_page)
             if (va_page)
                list_add(&va_page->list, &encl->va_pages)
          -> xa_insert(&encl->page_array, PFN_DOWN(encl_page->desc), encl_page, GFP_KERNEL)
          -> __sgx_encl_add_page(encl, encl_page, epc_page, secinfo, src)
             -> __eadd(&pginfo, sgx_get_epc_virt_addr(epc_page))
             encl_page->encl = encl;
             encl_page->epc_page = epc_page;
             encl->secs_child_cnt++;
             if (flags & SGX_PAGE_MEASURE)
                __sgx_encl_extend(encl, epc_page);
          -> sgx_mark_page_reclaimable(encl_page->epc_page)
             -> page->flags |= SGX_EPC_PAGE_RECLAIMER_TRACKED;
             -> list_add_tail(&page->list, &sgx_active_page_list)
```
* arch/x86/kernel/cpu/sgx/ioctl.c
```cpp
static struct sgx_encl_page *sgx_encl_page_alloc(struct sgx_encl *encl,
                         unsigned long offset,
                         u64 secinfo_flags)
{
    struct sgx_encl_page *encl_page;
    unsigned long prot;

    encl_page = kzalloc(sizeof(*encl_page), GFP_KERNEL);
    if (!encl_page)
        return ERR_PTR(-ENOMEM);
     //enclave page 的用户态虚拟地址，注意是基于 encl->base 的 offset 得到的，而不是 add_arg.src
    encl_page->desc = encl->base + offset;
    encl_page->encl = encl; //encl 域指向其所属的 enclave（struct sgx_encl）实例

    prot = _calc_vm_trans(secinfo_flags, SGX_SECINFO_R, PROT_READ)  |
           _calc_vm_trans(secinfo_flags, SGX_SECINFO_W, PROT_WRITE) |
           _calc_vm_trans(secinfo_flags, SGX_SECINFO_X, PROT_EXEC);

    /*
     * TCS pages must always RW set for CPU access while the SECINFO
     * permissions are *always* zero - the CPU ignores the user provided
     * values and silently overwrites them with zero permissions.
     */
    if ((secinfo_flags & SGX_SECINFO_PAGE_TYPE_MASK) == SGX_SECINFO_TCS)
        prot |= PROT_READ | PROT_WRITE;

    /* Calculate maximum of the VM flags for the page. */
    encl_page->vm_max_prot_bits = calc_vm_prot_bits(prot, 0);

    return encl_page;
}
...
static int __sgx_encl_add_page(struct sgx_encl *encl,
                   struct sgx_encl_page *encl_page,
                   struct sgx_epc_page *epc_page,
                   struct sgx_secinfo *secinfo, unsigned long src)
{
    struct sgx_pageinfo pginfo;
    struct vm_area_struct *vma;
    struct page *src_page;
    int ret;
    //根据用户态传入的 page data 的虚拟地址 src，查找其所在的 VMA
    /* Deny noexec. */
    vma = find_vma(current->mm, src);
    if (!vma)
        return -EFAULT;

    if (!(vma->vm_flags & VM_MAYEXEC))
        return -EACCES;
    //根据 src 得到未拷贝到 enclave 的 page data 的页描述符
    ret = get_user_pages(src, 1, 0, &src_page, NULL);
    if (ret < 1)
        return -EFAULT;
    //构造 PGINFO
    pginfo.secs = (unsigned long)sgx_get_epc_virt_addr(encl->secs.epc_page);
    pginfo.addr = encl_page->desc & PAGE_MASK; //页的当前用户态虚拟地址
    pginfo.metadata = (unsigned long)secinfo;
    pginfo.contents = (unsigned long)kmap_atomic(src_page);
    //用户态页数据拷贝进未完成的 enclave，第二个参数为 EPC 页内核态虚拟地址
    ret = __eadd(&pginfo, sgx_get_epc_virt_addr(epc_page));

    kunmap_atomic((void *)pginfo.contents);
    put_page(src_page);

    return ret ? -EIO : 0;
}
...
static int sgx_encl_add_page(struct sgx_encl *encl, unsigned long src,
                 unsigned long offset, struct sgx_secinfo *secinfo,
                 unsigned long flags)
{
    struct sgx_encl_page *encl_page;
    struct sgx_epc_page *epc_page;
    struct sgx_va_page *va_page;
    int ret;
    // 1) 先分配 struct sgx_encl_page 管理结构
    encl_page = sgx_encl_page_alloc(encl, offset, secinfo->flags);
    if (IS_ERR(encl_page))
        return PTR_ERR(encl_page);
    // 2) 再借用 EPC page 管理结构
    epc_page = sgx_alloc_epc_page(encl_page, true);
    if (IS_ERR(epc_page)) {
        kfree(encl_page);
        return PTR_ERR(epc_page);
    }
    // 3) enclave EPC 页面计数增长；如果需要，借用新的 EPC page 作为 VA page，并用 EPA 指令放入 EPC
    // 4) 如果刚才有新的 VA page 加入 EPC，分配新的 VA page 管理结构 struct sgx_va_page 来管理它
    va_page = sgx_encl_grow(encl);
    if (IS_ERR(va_page)) {
        ret = PTR_ERR(va_page);
        goto err_out_free;
    }

    mmap_read_lock(current->mm);
    mutex_lock(&encl->lock);
    // 5) 把 VA page 管理结构加到 enclave 的 va_page 链表上
    /*
     * Adding to encl->va_pages must be done under encl->lock.  Ditto for
     * deleting (via sgx_encl_shrink()) in the error path.
     */
    if (va_page)
        list_add(&va_page->list, &encl->va_pages);
    // 6) 把 encl_page 管理结构插入到 enclave 的 page_array xarray 上
    /*
     * Insert prior to EADD in case of OOM.  EADD modifies MRENCLAVE, i.e.
     * can't be gracefully unwound, while failure on EADD/EXTEND is limited
     * to userspace errors (or kernel/hardware bugs).
     */
    ret = xa_insert(&encl->page_array, PFN_DOWN(encl_page->desc),
            encl_page, GFP_KERNEL);
    if (ret)
        goto err_out_unlock;
    // 7) 把刚借到的 EPC page 加入到 EPC
    ret = __sgx_encl_add_page(encl, encl_page, epc_page, secinfo,
                  src);
    if (ret)
        goto err_out;

    /*
     * Complete the "add" before doing the "extend" so that the "add"
     * isn't in a half-baked state in the extremely unlikely scenario
     * the enclave will be destroyed in response to EEXTEND failure.
     */
    encl_page->encl = encl;
    encl_page->epc_page = epc_page;
    encl_page->type = (secinfo->flags & SGX_SECINFO_PAGE_TYPE_MASK) >> 8;
    encl->secs_child_cnt++; //enclave 实际页面计数 +1
    // 8) 如果需要，更新度量寄存器 MRENCLAVE，SDM 3D 35-6
    if (flags & SGX_PAGE_MEASURE) {
        ret = __sgx_encl_extend(encl, epc_page);
        if (ret)
            goto err_out;
    }
    // 9) 设置 EPC 页面的可回收标志位，并把 EPC page 结构的实例挂到 sgx_active_page_list 上
    sgx_mark_page_reclaimable(encl_page->epc_page);
    mutex_unlock(&encl->lock);
    mmap_read_unlock(current->mm);
    return ret;

err_out:
    xa_erase(&encl->page_array, PFN_DOWN(encl_page->desc));

err_out_unlock:
    sgx_encl_shrink(encl, va_page);
    mutex_unlock(&encl->lock);
    mmap_read_unlock(current->mm);

err_out_free:
    sgx_encl_free_epc_page(epc_page);
    kfree(encl_page);

    return ret;
}
...
/**
 * sgx_ioc_enclave_add_pages() - The handler for %SGX_IOC_ENCLAVE_ADD_PAGES
 * @encl:   an enclave pointer
 * @arg:    a user pointer to a struct sgx_enclave_add_pages instance
 *
 * Add one or more pages to an uninitialized enclave, and optionally extend the
 * measurement with the contents of the page. The SECINFO and measurement mask
 * are applied to all pages.
 *
 * A SECINFO for a TCS is required to always contain zero permissions because
 * CPU silently zeros them. Allowing anything else would cause a mismatch in
 * the measurement.
 *
 * mmap()'s protection bits are capped by the page permissions. For each page
 * address, the maximum protection bits are computed with the following
 * heuristics:
 *
 * 1. A regular page: PROT_R, PROT_W and PROT_X match the SECINFO permissions.
 * 2. A TCS page: PROT_R | PROT_W.
 *
 * mmap() is not allowed to surpass the minimum of the maximum protection bits
 * within the given address range.
 *
 * The function deinitializes kernel data structures for enclave and returns
 * -EIO in any of the following conditions:
 *
 * - Enclave Page Cache (EPC), the physical memory holding enclaves, has
 *   been invalidated. This will cause EADD and EEXTEND to fail.
 * - If the source address is corrupted somehow when executing EADD.
 *
 * Return:
 * - 0:     Success.
 * - -EACCES:   The source page is located in a noexec partition.
 * - -ENOMEM:   Out of EPC pages.
 * - -EINTR:    The call was interrupted before data was processed.
 * - -EIO:  Either EADD or EEXTEND failed because invalid source address
 *      or power cycle.
 * - -errno:    POSIX error.
 */
static long sgx_ioc_enclave_add_pages(struct sgx_encl *encl, void __user *arg)
{
    struct sgx_enclave_add_pages add_arg;
    struct sgx_secinfo secinfo;
    unsigned long c;
    int ret;

    if (!test_bit(SGX_ENCL_CREATED, &encl->flags) ||
        test_bit(SGX_ENCL_INITIALIZED, &encl->flags))
        return -EINVAL;

    if (copy_from_user(&add_arg, arg, sizeof(add_arg)))
        return -EFAULT;

    if (!IS_ALIGNED(add_arg.src, PAGE_SIZE))
        return -EINVAL;
    //对参数 offset 和 length 的有效性检查，比如页对齐和不要超出范围
    if (sgx_validate_offset_length(encl, add_arg.offset, add_arg.length))
        return -EINVAL;

    if (copy_from_user(&secinfo, (void __user *)add_arg.secinfo,
               sizeof(secinfo)))
        return -EFAULT;

    if (sgx_validate_secinfo(&secinfo))
        return -EINVAL;

    for (c = 0 ; c < add_arg.length; c += PAGE_SIZE) {
        if (signal_pending(current)) {
            if (!c)
                ret = -ERESTARTSYS;

            break;
        }

        if (need_resched())
            cond_resched();

        ret = sgx_encl_add_page(encl, add_arg.src + c, add_arg.offset + c,
                    &secinfo, add_arg.flags);
        if (ret)
            break;
    }

    add_arg.count = c;

    if (copy_to_user(arg, &add_arg, sizeof(add_arg)))
        return -EFAULT;

    return ret;
}
```
* arch/x86/include/uapi/asm/sgx.h
```cpp
/**
 * struct sgx_enclave_add_pages - parameter structure for the
 *                                %SGX_IOC_ENCLAVE_ADD_PAGE ioctl
 * @src:    start address for the page data
 * @offset: starting page offset
 * @length: length of the data (multiple of the page size)
 * @secinfo:    address for the SECINFO data
 * @flags:  page control flags
 * @count:  number of bytes added (multiple of the page size)
 */
struct sgx_enclave_add_pages {
    __u64 src;      //page 数据的起始地址
    __u64 offset;
    __u64 length;
    __u64 secinfo;
    __u64 flags;
    __u64 count;
};
```
* 如果`sgx_encl_add_page()`的调用者需要度量页面作为内容的证据，需要用`EEXTEND`指令为页面的`256 bytes`添加度量，并重复此操作直到整个页面被度量
```cpp
/*
 * If the caller requires measurement of the page as a proof for the content,
 * use EEXTEND to add a measurement for 256 bytes of the page. Repeat this
 * operation until the entire page is measured."
 */
static int __sgx_encl_extend(struct sgx_encl *encl,
                 struct sgx_epc_page *epc_page)
{
    unsigned long offset;
    int ret;

    for (offset = 0; offset < PAGE_SIZE; offset += SGX_EEXTEND_BLOCK_SIZE) {
        ret = __eextend(sgx_get_epc_virt_addr(encl->secs.epc_page),
                sgx_get_epc_virt_addr(epc_page) + offset);
        if (ret) {
            if (encls_failed(ret))
                ENCLS_WARN(ret, "EEXTEND");

            return -EIO;
        }
    }

    return 0;
}
```
## Init enclave

## mmap 映射 enclave
* 将 enclave 使用的地址空间进行`mmap()`其实是重要的一环，创建 enclave 和添加 EPC 页面的过程中虽然会涉及 ELRANGE 以及一些 base address 的计算，但并不会创建页表项
* 创建 enclave 在应用程序地址空间中的虚拟地址到物理地址的映射的工作其实是由缺页处理程序完成的，即`sgx_vma_fault()`
  * 在内核中虽然通过 `memremap()` 创建了映射，但该虚拟地址仅限于管理 EPC 页面，不是用来执行 enclave 代码的。记住，enclave 程序是运行在 ring 3
* 那么创建应用程序地址空间的地址映射就涉及到 VMA 的问题，enclave 程序虽然是 ELF 格式，但是却不能用动态链接器或者 ELF loader 加载，也就是说，不能由内核根据 ELF 格式文件里的 segment 来创建 VMA
* 现在的做法就是，uRTS 在将 enclave 代码和数据通过 `ioctl(SGX_IOC_ENCLAVE_ADD_PAGES)` 后，还需要将 ELRANGE `mmap()` 到应用程序地址空间，创建出相应的 VMA，形如：
  ```cpp
  mmap(0x7fa950226000, 1118208, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_FIXED, 4</dev/sgx_enclave>, 0) = 0x7fa950226000
  ```
  * `fd`参数传的是`/dev/sgx_enclave`，所以缺页处理函数为`sgx_vma_fault()`
  * 这样应用程序执行或访问 enclave 代码或数据的时候，才能通过页表找到正确的映射
* 是的，OS 不属于 SGX 的 TCB，但 CPU 的 PMH 在做地址翻译的时候能通过以下检查（不是全部）确保即使是 kernel 填的页表，也无法从 enclave 外部访问 EPC page 里的内容：
  * 首先你的程序得运行在 SGX enclave mode，这就杜绝了非 enclave 程序通过修改页表访问的企图
  * 其次当前运行的 enclave 得拥有该 EPC 页面（通过 EPCM 中的`ENCLAVESECS`属性），阻止其他 enclave 的访问
  * 再次当前访问的地址得和 add EPC page 时填入的虚拟地址一致（通过 EPCM 中的`ENCLAVEADDRESS`属性，SDM vol3. 34.20 ENCLAVE PAGE CACHE MAP (EPCM)）
### 建立映射
* `sgx_encl_may_map()`检查是否可以映射指定范围的 enclave 页面到 VMA 区域
  * enclave 页面的权限不能低于要映射 VMA 区域的权限
```cpp
/**
 * sgx_encl_may_map() - Check if a requested VMA mapping is allowed
 * @encl:       an enclave pointer
 * @start:      lower bound of the address range, inclusive
 * @end:        upper bound of the address range, exclusive
 * @vm_flags:       VMA flags
 *
 * Iterate through the enclave pages contained within [@start, @end) to verify
 * that the permissions requested by a subset of {VM_READ, VM_WRITE, VM_EXEC}
 * do not contain any permissions that are not contained in the build time
 * permissions of any of the enclave pages within the given address range.
 *
 * An enclave creator must declare the strongest permissions that will be
 * needed for each enclave page. This ensures that mappings have the identical
 * or weaker permissions than the earlier declared permissions.
 *
 * Return: 0 on success, -EACCES otherwise
 */
int sgx_encl_may_map(struct sgx_encl *encl, unsigned long start,
             unsigned long end, unsigned long vm_flags)
{
    unsigned long vm_prot_bits = vm_flags & (VM_READ | VM_WRITE | VM_EXEC);
    struct sgx_encl_page *page;
    unsigned long count = 0;
    int ret = 0;
    //准备遍历 enclave pages 的 xarray，遍历的起始点是准备映射的 VMA 的起始地址
    XA_STATE(xas, &encl->page_array, PFN_DOWN(start));
    //如果该 enclave 已经初始化完结，不允许映射的地址范围超出 enclave 的范围
    /* Disallow mapping outside enclave's address range. */
    if (test_bit(SGX_ENCL_INITIALIZED, &encl->flags) &&
        (start < encl->base || end > encl->base + encl->size))
        return -EACCES;

    /*
     * Disallow READ_IMPLIES_EXEC tasks as their VMA permissions might
     * conflict with the enclave page permissions.
     */
    if (current->personality & READ_IMPLIES_EXEC)
        return -EACCES;
    //遍历的结束点是准备映射的 VMA 的结束地址（但不包含它）
    mutex_lock(&encl->lock);
    xas_lock(&xas);
    xas_for_each(&xas, page, PFN_DOWN(end - 1)) {
        if (~page->vm_max_prot_bits & vm_prot_bits) {
            ret = -EACCES;
            break; //enclave 页面的权限不能低于要映射 VMA 区域的权限，否则中止遍历，返回错误
        }
        //每遍历 XA_CHECK_SCHED（4096）个页面提供一次调度的机会，因为 xas_lock 是自旋锁
        /* Reschedule on every XA_CHECK_SCHED iteration. */
        if (!(++count % XA_CHECK_SCHED)) {
            xas_pause(&xas);
            xas_unlock(&xas);
            mutex_unlock(&encl->lock);

            cond_resched();

            mutex_lock(&encl->lock);
            xas_lock(&xas);
        }
    }
    xas_unlock(&xas);
    mutex_unlock(&encl->lock);

    return ret;
}
```
* `sgx_mmap()`用于将`/dev/sgx_enclave`做文件映射的时候
```cpp
static int sgx_mmap(struct file *file, struct vm_area_struct *vma)
{
    struct sgx_encl *encl = file->private_data; //sgx_open() 的时候赋的值
    int ret;
    //检查是否可以将当前 enclave 的页面映射到 VMA 区域
    ret = sgx_encl_may_map(encl, vma->vm_start, vma->vm_end, vma->vm_flags);
    if (ret)
        return ret;
    //检查通过，给 encl_mm 分配空间并挂上 encl 的 mm_list 链表
    ret = sgx_encl_mm_add(encl, vma->vm_mm); //vma->vm_mm 为 VMA 所属 mm_struct
    if (ret)
        return ret;

    vma->vm_ops = &sgx_vm_ops; //设置操作该 VMA 的 operations 结构，sgx_encl_find() 也会用到
    vma->vm_flags |= VM_PFNMAP | VM_DONTEXPAND | VM_DONTDUMP | VM_IO; //注意一下 VM_PFNMAP 这个 flag
    vma->vm_private_data = encl; //回头很多信息需要通过 encl 找到

    return 0;
}
```
### 添加 struct sgx_encl_mm 到 encl 的 mm_list
* 一个 enclave 可能被映射到多个`struct mm_struct mm`中，但一个`mm`只会在 enclave 的`mm_list`链表上出现一次
* `sgx_encl_mm`的主要目的是订阅 MMU 事件，通过`sgx_encl_mm_add()`注册 mmu_notifier 以便将来接收事件通知
* arch/x86/kernel/cpu/sgx/encl.c
```cpp
static struct sgx_encl_mm *sgx_encl_find_mm(struct sgx_encl *encl,
                        struct mm_struct *mm)
{
    struct sgx_encl_mm *encl_mm = NULL;
    struct sgx_encl_mm *tmp;
    int idx;

    idx = srcu_read_lock(&encl->srcu);
    //在 encl 的 mm_list 链表中找指定的 mm_struct
    list_for_each_entry_rcu(tmp, &encl->mm_list, list) {
        if (tmp->mm == mm) { //找到了
            encl_mm = tmp;   //记录下 encl_mm
            break;           //结束查找
        }
    }

    srcu_read_unlock(&encl->srcu, idx);

    return encl_mm;
}

int sgx_encl_mm_add(struct sgx_encl *encl, struct mm_struct *mm)
{
    struct sgx_encl_mm *encl_mm;
    int ret;

    /*
     * Even though a single enclave may be mapped into an mm more than once,
     * each 'mm' only appears once on encl->mm_list. This is guaranteed by
     * holding the mm's mmap lock for write before an mm can be added or
     * remove to an encl->mm_list.
     */
    mmap_assert_write_locked(mm);
    //struct mm_struct mm 已经在 enclave 的 mm_list 链表上了，不要再分配新的 encl_mm 了
    /*
     * It's possible that an entry already exists in the mm_list, because it
     * is removed only on VFS release or process exit.
     */
    if (sgx_encl_find_mm(encl, mm))
        return 0;
    //分配新的 encl_mm
    encl_mm = kzalloc(sizeof(*encl_mm), GFP_KERNEL);
    if (!encl_mm)
        return -ENOMEM;

    /* Grab a refcount for the encl_mm->encl reference: */
    kref_get(&encl->refcount); //enclave 又多了一个引用
    encl_mm->encl = encl;
    encl_mm->mm = mm;
    encl_mm->mmu_notifier.ops = &sgx_mmu_notifier_ops;
    //encl_mm 的主要目的，订阅 MMU 事件。这里注册 mmu_notifier 以便将来接收事件通知
    ret = __mmu_notifier_register(&encl_mm->mmu_notifier, mm);
    if (ret) {
        kfree(encl_mm);
        return ret;
    }

    spin_lock(&encl->mm_lock);
    list_add_rcu(&encl_mm->list, &encl->mm_list);
    /* Pairs with smp_rmb() in sgx_reclaimer_block(). */
    smp_wmb();
    encl->mm_list_version++; //回收的时候会用到这个域
    spin_unlock(&encl->mm_lock);

    return 0;
}
```
## 页面回收
* EPC 是全局的，然而 enclave 是独立的个体，当 enclave A 的 EPC 页面不活跃而 enclave B 的 EPC 页面不够用时，就会触发页面回收
* 从全局的 active page pool 的 head 获取固定数量的页面，并将它们回收到 enclave 的私有 shmem 文件中
* 跳过自上次扫描以来访问过的页面。将这些页面移动到 active page pool 的尾部，以类似 LRU 的方式扫描这些页面
* 批处理一大块页面（目前为`16`个），以降低潜在的需要 IPI 和`ETRACK`的数量
* `sgx_encl_ewb()`确实会在有三级`EWB` pipeline（`EWB`、`ETRACK + EWB`和`IPI + EWB`）的硬件线程中降级了一点，但还不够
* 一次回收一页也会有问题，因为它会过多地增加锁争用，这会阻止进程的前进

### 页面老化函数`sgx_reclaimer_age()`
* `sgx_encl_find()`根据提供的`mm_struct`和虚拟地址，查找包含该 enclave page 用户态虚拟地址的 VMA
  * 返回`0`，找到 VMA；否则返回错误码
  * arch/x86/kernel/cpu/sgx/encl.h
    ```cpp
    static inline int sgx_encl_find(struct mm_struct *mm, unsigned long addr,
                    struct vm_area_struct **vma)
    {
        struct vm_area_struct *result;
        //查找包含该虚拟地址的 VMA
        result = vma_lookup(mm, addr);
        if (!result || result->vm_ops != &sgx_vm_ops) //判断 VMA 是不是映射 enclave
            return -EINVAL; //没有包含该地址的 VMA 或 VMA 映射的不是 enclave

        *vma = result;

        return 0;
    }
    ```
* `sgx_encl_test_and_clear_young()`测试并重置`accessed`位
  * 返回值：`1`该页面最近被访问过；否则返回`0`，最近未被访问
  * 它会调用`apply_to_page_range()`对`addr`到`addr + PAGE_SIZE`这段虚拟内存区域做扫描，并传入`sgx_encl_test_and_clear_young_cb()`回调去做具体的事情
```cpp
static int sgx_encl_test_and_clear_young_cb(pte_t *ptep, unsigned long addr,
                        void *data)
{
    pte_t pte;
    int ret;
    //测试该 PTE 最近是否被访问过（如果是，CPU 当时会将该 PTE 的 _PAGE_ACCESSED 置位，将来由软件清除）
    ret = pte_young(*ptep);
    if (ret) {
        pte = pte_mkold(*ptep);//根据 *ptep 构造 pte，清除 pte 的 _PAGE_ACCESSED 位
        set_pte_at((struct mm_struct *)data, addr, ptep, pte);//pte 的值写回 *ptep，完成成清除操作
    }

    return ret;
}

/**
 * sgx_encl_test_and_clear_young() - Test and reset the accessed bit
 * @mm:     mm_struct that is checked
 * @page:   enclave page to be tested for recent access
 *
 * Checks the Access (A) bit from the PTE corresponding to the enclave page and
 * clears it.
 *
 * Return: 1 if the page has been recently accessed and 0 if not.
 */
int sgx_encl_test_and_clear_young(struct mm_struct *mm,
                  struct sgx_encl_page *page)
{
    unsigned long addr = page->desc & PAGE_MASK; //enclave page 用户态虚拟地址
    struct sgx_encl *encl = page->encl;
    struct vm_area_struct *vma;
    int ret;
    //查找包含该 enclave page 用户态虚拟地址的 VMA
    ret = sgx_encl_find(mm, addr, &vma);
    if (ret)      //如果找不到
        return 0; //最近未访问过该页
    //如果该 VMA 映射的不是当前 enclave
    if (encl != vma->vm_private_data) //映射该 VMA 时的 encl 是否是当前 ENC page 所属的 encl
        return 0; //最近未访问过该页
    //扫描 addr 到 addr + PAGE_SIZE 这段虚拟内存区域，如果需要的话填充页表项，
    //并在每一级子页表上调用 sgx_encl_test_and_clear_young_cb(ptep, addr, vma->vm_mm)
    ret = apply_to_page_range(vma->vm_mm, addr, PAGE_SIZE,
                  sgx_encl_test_and_clear_young_cb, vma->vm_mm);
    if (ret < 0)
        return 0; //最近未访问过该页

    return ret;
}
```
* `sgx_reclaimer_age()`检查 EPC 页面是否老化
  * 返回`true`，最近未访问过该页，说明已经老化了
  * 返回`false`，该页被访问过，说明还年轻
```cpp
static bool sgx_reclaimer_age(struct sgx_epc_page *epc_page)
{
    struct sgx_encl_page *page = epc_page->owner;
    struct sgx_encl *encl = page->encl;
    struct sgx_encl_mm *encl_mm;
    bool ret = true;
    int idx;

    idx = srcu_read_lock(&encl->srcu);

    list_for_each_entry_rcu(encl_mm, &encl->mm_list, list) {
        //如果 encl_mm 的 mm_struct 的 mm_users 不为 0 则递增
        //该计数用于 pin 住该 mm，但不能保护该地址空间被修改或释放
        if (!mmget_not_zero(encl_mm->mm))
            continue; //如果为 0，则跳过该 encl_mm

        mmap_read_lock(encl_mm->mm);
        ret = !sgx_encl_test_and_clear_young(encl_mm->mm, page); //注意前面的反操作
        mmap_read_unlock(encl_mm->mm);
        //unpin 该 mm
        mmput_async(encl_mm->mm);
        //ret: 1，未被访问过；0，访问过
        if (!ret)
            break; //该页面最近被访问过
    }

    srcu_read_unlock(&encl->srcu, idx);

    if (!ret)
        return false; //该页面最近被访问过

    return true; //最近未访问过该页
}
```
### 页面回收函数`sgx_reclaim_pages()`
```cpp
/*
 * Take a fixed number of pages from the head of the active page pool and
 * reclaim them to the enclave's private shmem files. Skip the pages, which have
 * been accessed since the last scan. Move those pages to the tail of active
 * page pool so that the pages get scanned in LRU like fashion.
 *
 * Batch process a chunk of pages (at the moment 16) in order to degrade amount
 * of IPI's and ETRACK's potentially required. sgx_encl_ewb() does degrade a bit
 * among the HW threads with three stage EWB pipeline (EWB, ETRACK + EWB and IPI
 * + EWB) but not sufficiently. Reclaiming one page at a time would also be
 * problematic as it would increase the lock contention too much, which would
 * halt forward progress.
 */
static void sgx_reclaim_pages(void)
{
    struct sgx_epc_page *chunk[SGX_NR_TO_SCAN];
    struct sgx_backing backing[SGX_NR_TO_SCAN];
    struct sgx_encl_page *encl_page;
    struct sgx_epc_page *epc_page;
    pgoff_t page_index;
    int cnt = 0;
    int ret;
    int i;
    //1. 从 sgx_active_page_list 上收集 16 个 EPC pages 放到 chunk[] 数组
    spin_lock(&sgx_reclaimer_lock);
    for (i = 0; i < SGX_NR_TO_SCAN; i++) {
        if (list_empty(&sgx_active_page_list)) //已经没有在使用中的 page 了？
            break;

        epc_page = list_first_entry(&sgx_active_page_list,
                        struct sgx_epc_page, list);
        list_del_init(&epc_page->list); //将 EPC 页面管理结构从 sgx_active_page_list 拿下
        encl_page = epc_page->owner;
        //如果 enclave 的引用计数不为 0，则增加引用计数
        if (kref_get_unless_zero(&encl_page->encl->refcount) != 0)
            chunk[cnt++] = epc_page; //并将该 EPC 页面记录到 chunk[] 数组
        else //引用计数为 0 说明 enclave 正在释放该页，因此无需将该页放回可回收页列表
            /* The owner is freeing the page. No need to add the
             * page back to the list of reclaimable pages.
             */
            epc_page->flags &= ~SGX_EPC_PAGE_RECLAIMER_TRACKED;
    }
    spin_unlock(&sgx_reclaimer_lock);
    //2. 检查 chunk[] 数组，剔除最近访问过的页，并把它们放回 sgx_active_page_list 的末尾 
    for (i = 0; i < cnt; i++) {
        epc_page = chunk[i];
        encl_page = epc_page->owner;
        //检查该页面最近是否被访问过
        if (!sgx_reclaimer_age(epc_page))
            goto skip; //返回 false，该页最近被访问过，跳过
        //返回 true，该页面最近未被访问过。页面用户态虚拟地址 - 用户态 enclave 基址 = 用户态页索引
        page_index = PFN_DOWN(encl_page->desc - encl_page->encl->base);
        //根据 page_index 分配要回收 EPC 页面对应到后备存储中的页面，填充 backing 的各个域
        mutex_lock(&encl_page->encl->lock);
        ret = sgx_encl_alloc_backing(encl_page->encl, page_index, &backing[i]);
        if (ret) {
            mutex_unlock(&encl_page->encl->lock);
            goto skip;
        }
        //给该 encl 页面加上“正在被回收”的标志位
        encl_page->desc |= SGX_ENCL_PAGE_BEING_RECLAIMED;
        mutex_unlock(&encl_page->encl->lock);
        continue; //检查下一页

skip:
        spin_lock(&sgx_reclaimer_lock);
        list_add_tail(&epc_page->list, &sgx_active_page_list); //放回 sgx_active_page_list 尾部
        spin_unlock(&sgx_reclaimer_lock);

        kref_put(&encl_page->encl->refcount, sgx_encl_release);

        chunk[i] = NULL;
    }
    //3. 到此，chunk[] 数组里不包含最近被访问过的页，接着要把这些页面标为 BLOCKED，防止它们被分出去
    for (i = 0; i < cnt; i++) {
        epc_page = chunk[i];
        if (epc_page)
            sgx_reclaimer_block(epc_page);//把这些页面标为 BLOCKED
    }
    //4. 下一步该逐个将 EPC 页面从 EPC 写回到后备存储了
    for (i = 0; i < cnt; i++) {
        epc_page = chunk[i];
        if (!epc_page)
            continue;

        encl_page = epc_page->owner;
        sgx_reclaimer_write(epc_page, &backing[i]);//EPC 页面写回到后备存储，SECS 页面可能会率先回写
        //递减 encl 的引用计数，如果减为零了就释放该 encl
        kref_put(&encl_page->encl->refcount, sgx_encl_release);
        epc_page->flags &= ~SGX_EPC_PAGE_RECLAIMER_TRACKED; //清除 EPC 页面管理结构的回收标志位
        //将 EPC 页面还回 sgx_numa_node.free_page_list，可以其他地方借用了
        sgx_free_epc_page(epc_page);
    }
}
```
### 页面加密元数据（PCWD）
* Paging Crypto MetaData（PCMD）页面加密元数据
  * 将每个要交换出去的页面的 PCMD 存储到常规内存中
  * 当`ELDU`重新加载页面时，它会使用存储在 VA 页面中的`isvsvn`编码（page version，nonce）重新计算 MAC
  * 这两种结构（EPC 中的 VA page 和 PCMD）结合带来了完整性和回滚保护
    ```cpp
    /**
    * struct sgx_secinfo - describes attributes of an EPC page
    * @flags:  permissions and type
    *
    * Used together with ENCLS leaves that add or modify an EPC page to an
    * enclave to define page permissions and type.
    */
    struct sgx_secinfo {
        u64 flags;
        u8  reserved[56];
    } __packed __aligned(64);

    #define SGX_PCMD_RESERVED_SIZE 40

    /**
    * struct sgx_pcmd - Paging Crypto Metadata (PCMD)
    * @enclave_id: enclave identifier
    * @mac:    MAC over PCMD, page contents and isvsvn
    *
    * PCMD is stored for every swapped page to the regular memory. When ELDU loads
    * the page back it recalculates the MAC by using a isvsvn number stored in a
    * VA page. Together these two structures bring integrity and rollback
    * protection.
    */
    struct sgx_pcmd {
        struct sgx_secinfo secinfo;
        u64 enclave_id;
        u8  reserved[SGX_PCMD_RESERVED_SIZE];
        u8  mac[16];
    } __packed __aligned(128);
    ```
### SGX Backing 后备存储
* 当系统用完 enclave 内存时，SGX 可以通过交换到普通 RAM 来回收 EPC 页面。
  * 这些支持页面是通过每个 enclave 共享内存区域分配的。
  * 由于 SGX 允许在 EPC 内存上无限制地过度提交，因此回收线程可以分配大量后备 RAM 页面以响应 EPC 内存压力。
  * 当在回收线程上下文期间发生共享内存后备 RAM 分配时，共享内存被计入 root memory control group，并且未正确考虑 enclave 的 shmem 使用情况，使得 cgroups 在限制 enclave 可以消费的 RAM 数量无效。
* commit [0c9782e204d3 x86/sgx: Set active memcg prior to shmem allocation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0c9782e204d3cc5625b9e8bf4e8625d38dfe0139) 改变了这一状况。
* SGX 存储与 enclave 关联的`mm_structs`列表，因此，在回收期间选择其中一个，并使用 shmem 分配对该`mm`的`memcg`进行计量。
  * 被选中的那个`mm`是任意的，但这个`mm_list`几乎总是只有`mm`。
  * 具有超过一个`mm`的不同`memcg`的情况不值得考虑。
* 该 commit 还创建一个新函数 - `sgx_encl_alloc_backing()`。
  * 每当需要分配新的后备存储页面时，都会使用此函数。以前，相同的函数用于页面分配以及检索先前分配的页面。
  * 在后备存储页面分配之前，如果存在与请求分配的 enclave 关联的`mm_struct`，则将其设置为 active memory control group。
```cpp
/*
 * When called from ksgxd, returns the mem_cgroup of a struct mm stored
 * in the enclave's mm_list. When not called from ksgxd, just returns
 * the mem_cgroup of the current task.
 */
static struct mem_cgroup *sgx_encl_get_mem_cgroup(struct sgx_encl *encl)
{
    struct mem_cgroup *memcg = NULL;
    struct sgx_encl_mm *encl_mm;
    int idx;
    //如果调用者是普通任务上下文，返回当前任务 mm 的 mem_cgroup
    /*
     * If called from normal task context, return the mem_cgroup
     * of the current task's mm. The remainder of the handling is for
     * ksgxd.
     */
    if (!current_is_ksgxd())
        return get_mem_cgroup_from_mm(current->mm);
    //如果调用者是 ksgxd，传入的 encl 是要回收的 EPC 页面的所属的 enclave
    /*
     * Search the enclave's mm_list to find an mm associated with
     * this enclave to charge the allocation to.
     */
    idx = srcu_read_lock(&encl->srcu);
    //我们在其 mm_list 中选一个 mm 的 memory control group 返回
    list_for_each_entry_rcu(encl_mm, &encl->mm_list, list) {
        if (!mmget_not_zero(encl_mm->mm))
            continue;

        memcg = get_mem_cgroup_from_mm(encl_mm->mm);

        mmput_async(encl_mm->mm);

        break;
    }

    srcu_read_unlock(&encl->srcu, idx);
    //对于没有与 enclave 关联的 mm，我们选 root mem_cgroup 返回
    /*
     * In the rare case that there isn't an mm associated with
     * the enclave, set memcg to the current active mem_cgroup.
     * This will be the root mem_cgroup if there is no active
     * mem_cgroup.
     */
    if (!memcg)
        return get_mem_cgroup_from_mm(NULL);

    return memcg;
}

/**
 * sgx_encl_alloc_backing() - create a new backing storage page
 * @encl:   an enclave pointer
 * @page_index: enclave page index
 * @backing:    data for accessing backing storage for the page
 *
 * When called from ksgxd, sets the active memcg from one of the
 * mms in the enclave's mm_list prior to any backing page allocation,
 * in order to ensure that shmem page allocations are charged to the
 * enclave.  Create a backing page for loading data back into an EPC page with
 * ELDU.  This function takes a reference on a new backing page which
 * must be dropped with a corresponding call to sgx_encl_put_backing().
 *
 * Return:
 *   0 on success,
 *   -errno otherwise.
 */
int sgx_encl_alloc_backing(struct sgx_encl *encl, unsigned long page_index,
               struct sgx_backing *backing)
{
    struct mem_cgroup *encl_memcg = sgx_encl_get_mem_cgroup(encl); //选择一个合适的 memcg
    struct mem_cgroup *memcg = set_active_memcg(encl_memcg); //设置 active memcg，保存旧 memcg
    int ret;
    //根据 page_index 找到要回收 EPC 页面对应到后备存储中的页面，填充 backing 的各个域
    ret = __sgx_encl_get_backing(encl, page_index, backing);
    //还原 memcg
    set_active_memcg(memcg);
    mem_cgroup_put(encl_memcg); //对应之前调的 get

    return ret;
}
```
* `sgx_encl_lookup_backing()`获得一个已存在的后备存储页
  * 该函数通常用于`ELDU`获取后备页面以将数据加载回 EPC 页面。
  * 调用者有责任确保使用`sgx_encl_lookup_backing()`是合适的，而不是`sgx_encl_alloc_backing()`。
  * 如果没有正确使用该查找函数，会导致未计量的页分配。
  * 此函数获取现有后备页面上的引用，必须通过相应调用`sgx_encl_put_backing()`来删除该后备页面。
```cpp
/**
 * sgx_encl_lookup_backing() - retrieve an existing backing storage page
 * @encl:   an enclave pointer
 * @page_index: enclave page index
 * @backing:    data for accessing backing storage for the page
 *
 * Retrieve a backing page for loading data back into an EPC page with ELDU.
 * It is the caller's responsibility to ensure that it is appropriate to use
 * sgx_encl_lookup_backing() rather than sgx_encl_alloc_backing(). If lookup is
 * not used correctly, this will cause an allocation which is not accounted for.
 * This function takes a reference on an existing backing page which must be
 * dropped with a corresponding call to sgx_encl_put_backing().
 *
 * Return:
 *   0 on success,
 *   -errno otherwise.
 */
static int sgx_encl_lookup_backing(struct sgx_encl *encl, unsigned long page_index,
               struct sgx_backing *backing)
{   //因为已存在后备页，所以无需 memcg 相关的操作了
    return __sgx_encl_get_backing(encl, page_index, backing);
}
```

* `sgx_encl_get_backing_page_pcmd_offset()`得到和一个 enclave 页面关联的 PCMD struct 在后备存储中的偏移字节数
  * 在后备存储中，PCMD 正好在 EPC 数据的后面
  * 除了可见的 enclave 页面，还有一个额外的页面给 SECS，在 PCMD 结构之前
  * 因此最后返回的是从 *enclave 基址（`encl->base`）对应到后备存储的基址* 算起的到`pcmd`数据的偏移
```cpp
/*
 * Calculate byte offset of a PCMD struct associated with an enclave page. PCMD's
 * follow right after the EPC data in the backing storage. In addition to the
 * visible enclave pages, there's one extra page slot for SECS, before PCMD
 * structs.
 */
static inline pgoff_t sgx_encl_get_backing_page_pcmd_offset(struct sgx_encl *encl,
                                unsigned long page_index)
{   //enclave pages 之后是 SECS
    pgoff_t epc_end_off = encl->size + sizeof(struct sgx_secs);
    //page_index 是以页帧为单位的，每页帧配上一个 PCMD 结构；加起来最后得到 PCMD 的偏移
    return epc_end_off + page_index * sizeof(struct sgx_pcmd);
}
```
* `sgx_encl_get_backing_page()`根据页索引读到 EPC page 在后备存储中的 page 管理结构
```cpp
static struct page *sgx_encl_get_backing_page(struct sgx_encl *encl,
                          pgoff_t index)
{
    struct inode *inode = encl->backing->f_path.dentry->d_inode; //后备文件的 inode
    struct address_space *mapping = inode->i_mapping; //page cache 指针
    gfp_t gfpmask = mapping_gfp_mask(mapping);
    //根据索引在 page cache 中读到并返回 page 结构的指针
    return shmem_read_mapping_page_gfp(mapping, index, gfpmask);
}
```
* `__sgx_encl_get_backing()`函数根据用户态页索引填充`backing[]`数组元素的各个域
  * 读取 *EPC page 在后备存储中的 page 管理结构*，记录在`struct sgx_backing`的`contents`域
  * 读取 *与 EPC page 关联的 PCMD struct 在后备存储中的 page 管理结构*，记录在`struct sgx_backing`的`pcmd`域
  * `struct sgx_backing`的`pcmd_offset`域是根据`page_pcmd_off`转换得到`struct sgx_pcmd`实例在的其所处的页内的偏移
```cpp
/**
 * __sgx_encl_get_backing() - Pin the backing storage
 * @encl:   an enclave pointer
 * @page_index: enclave page index
 * @backing:    data for accessing backing storage for the page
 *
 * Pin the backing storage pages for storing the encrypted contents and Paging
 * Crypto MetaData (PCMD) of an enclave page.
 *
 * Return:
 *   0 on success,
 *   -errno otherwise.
 */
int __sgx_encl_get_backing(struct sgx_encl *encl, unsigned long page_index,
             struct sgx_backing *backing)
{
    pgoff_t page_pcmd_off = sgx_encl_get_backing_page_pcmd_offset(encl, page_index);
    struct page *contents;
    struct page *pcmd;
    //根据用户态页索引读到 EPC page 在后备存储中的 page 管理结构
    contents = sgx_encl_get_backing_page(encl, page_index);
    if (IS_ERR(contents))
        return PTR_ERR(contents);
    //根据 EPC page 关联的 PCMD struct 的偏移，转换成页索引，读到其在后备存储中的 page 管理结构
    pcmd = sgx_encl_get_backing_page(encl, PFN_DOWN(page_pcmd_off));
    if (IS_ERR(pcmd)) {
        put_page(contents);
        return PTR_ERR(pcmd);
    }
    //根据以上信息构填充一个 backing 元素的域
    backing->contents = contents;
    backing->pcmd = pcmd;
    backing->pcmd_offset = page_pcmd_off & (PAGE_SIZE - 1); //变成页内偏移

    return 0;
}
```
### 移除 PTE 映射函数 `sgx_zap_enclave_ptes()`
* 一个 enclave page 可能被映射到多个 VMA 中，该函数将映射从每个 VMA 中移除
```cpp
/**
 * sgx_zap_enclave_ptes() - remove PTEs mapping the address from enclave
 * @encl: the enclave
 * @addr: page aligned pointer to single page for which PTEs will be removed
 *
 * Multiple VMAs may have an enclave page mapped. Remove the PTE mapping
 * @addr from each VMA. Ensure that page fault handler is ready to handle
 * new mappings of @addr before calling this function.
 */
void sgx_zap_enclave_ptes(struct sgx_encl *encl, unsigned long addr)
{
    unsigned long mm_list_version;
    struct sgx_encl_mm *encl_mm;
    struct vm_area_struct *vma;
    int idx, ret;

    do {//encl 被 mmap() 了，mm_list 就会多一个节点并增加 mm_list_version，这里通过版本号同步
        mm_list_version = encl->mm_list_version;
        //与 sgx_encl_mm_add() 那边的写屏障配对，确保先看到新版本号再去检查 mm_list 链表
        /* Pairs with smp_wmb() in sgx_encl_mm_add(). */
        smp_rmb();

        idx = srcu_read_lock(&encl->srcu);

        list_for_each_entry_rcu(encl_mm, &encl->mm_list, list) {
            if (!mmget_not_zero(encl_mm->mm)) //如果 encl_mm 的 mm_struct 的 mm_users 不为 0 则递增
                continue; //如果为 0，则跳过该 encl_mm

            mmap_read_lock(encl_mm->mm);
            //查找包含该 enclave page 用户态虚拟地址的 VMA
            ret = sgx_encl_find(encl_mm->mm, addr, &vma);
            if (!ret && encl == vma->vm_private_data)//查找到 VMA，且映射该 VMA 时的 encl 是当前 ENC page 所属的 encl
                zap_vma_ptes(vma, addr, PAGE_SIZE);//移除 VMA 中对该虚拟地址的 pte 映射，需要 VM_PFNMAP flag

            mmap_read_unlock(encl_mm->mm);

            mmput_async(encl_mm->mm);
        }

        srcu_read_unlock(&encl->srcu, idx); //这里隐含一个内存屏障，能确保下面得到最新的版本号
    } while (unlikely(encl->mm_list_version != mm_list_version));
}
```
### 回收页面置`BLOCKED`函数 sgx_reclaimer_block()
* `sgx_reclaimer_block()`主要负责移除 VMA 中对该虚拟地址的 pte 映射，并用`EBLOCK`指令把该 EPC 页面置为`BLOCKED`状态
```cpp
static void sgx_reclaimer_block(struct sgx_epc_page *epc_page)
{
    struct sgx_encl_page *page = epc_page->owner;
    unsigned long addr = page->desc & PAGE_MASK; //得到 EPC page 的用户态虚拟地址
    struct sgx_encl *encl = page->encl;
    int ret;
    //先移除 EPC 页面在各 VMA 中的映射
    sgx_zap_enclave_ptes(encl, addr);

    mutex_lock(&encl->lock);
    //用 EBLOCK 指令把该 EPC 页面置为 BLOCKED 状态，该状态下的 EPC 页面不会被创建新的 cached 虚拟地址到物理地址的映射
    ret = __eblock(sgx_get_epc_virt_addr(epc_page));//这里用的是内核态虚拟地址
    if (encls_failed(ret))
        ENCLS_WARN(ret, "EBLOCK");

    mutex_unlock(&encl->lock);
}
```

### EPC 页面回写
* `struct sgx_pageinfo` 是用于 EPC 管理指令的数据结构，要求`32 Byte`对齐
```cpp
/**
 * struct sgx_pageinfo - an enclave page descriptor
 * @addr:   address of the enclave page
 * @contents:   pointer to the page contents
 * @metadata:   pointer either to a SECINFO or PCMD instance
 * @secs:   address of the SECS page
 */
struct sgx_pageinfo {
    u64 addr;     //enclave 页面的线性地址
    u64 contents; //指向内容的页面的有效地址
    u64 metadata; //指向 SECINFO 或 PCMD 类似的元数据的有效地址
    u64 secs;     //SECS 页面的地址
} __packed __aligned(32);
```
* `sgx_alloc_va_slot()`从 VA 页面管理结构`struct sgx_va_page`的一个实例中分配一个 slot
  * 返回 slot 在 VA 页面中的偏移，而不是第几个 slot
```cpp
/**
 * sgx_alloc_va_slot - allocate a VA slot
 * @va_page:    a &struct sgx_va_page instance
 *
 * Allocates a slot from a &struct sgx_va_page instance.
 *
 * Return: offset of the slot inside the VA page
 */
unsigned int sgx_alloc_va_slot(struct sgx_va_page *va_page)
{   //在位图 va_page->slots 中找到第一个为 0 的位
    int slot = find_first_zero_bit(va_page->slots, SGX_VA_SLOT_COUNT);

    if (slot < SGX_VA_SLOT_COUNT)
        set_bit(slot, va_page->slots);//将位图 va_page->slots 中的第 slot 个位置 1
    //因为在 VA 页面中每个单元的长度是 8 字节，所以返回值需将索引（slot）乘以 8
    return slot << 3;
}
```
* `__sgx_encl_ewb()`尝试用`EWB`指令将 EPC 页面从 EPC 中换出到 system RAM
```cpp
static int __sgx_encl_ewb(struct sgx_epc_page *epc_page, void *va_slot,
              struct sgx_backing *backing)
{
    struct sgx_pageinfo pginfo;
    int ret;

    pginfo.addr = 0;
    pginfo.secs = 0;
    //得到后备存储页帧的虚拟地址和对应的元数据结构 PCMD 的虚拟地址
    pginfo.contents = (unsigned long)kmap_atomic(backing->contents);
    pginfo.metadata = (unsigned long)kmap_atomic(backing->pcmd) +
              backing->pcmd_offset;//pcmd 域是 struct page*，kmap_atomic() 返回的是页帧对应的的起始虚拟地址
    //传入的 va_slot 为指向 VA 页面内核虚拟地址中的偏移；此外传入 EPC 页面的内核虚拟地址
    ret = __ewb(&pginfo, sgx_get_epc_virt_addr(epc_page), va_slot); //调用 EWB 指令换出页面
    set_page_dirty(backing->pcmd); //在 encl->lock 锁内将这两页设为 dirty，避免与同时发生的缺页换入发生竞态
    set_page_dirty(backing->contents);

    kunmap_atomic((void *)(unsigned long)(pginfo.metadata -
                          backing->pcmd_offset));
    kunmap_atomic((void *)(unsigned long)pginfo.contents);

    return ret;
}
```
#### `sgx_encl_ewb()`函数
* 准确地说，是将 EPC 页面从 EPC 交换到 system RAM 中，当然，内容是加密过的
* 第一次尝试，假设 enclave 中的其他线程已经用`ETRACK`重置计数，且前面的线程计数已经清零，尽可能地去写页面
* 第二次尝试，在`EWB`前调用`ETRACK`。如果失败，踢掉所有的硬件线程然后再`EWB`，这应该能确保成功
  * 所谓的“踢掉”其实就是通过`on_each_cpu_mask(sgx_encl_cpumask(encl), sgx_ipi_cb, NULL, 1)`给 mask 里的 CPU 发 IPI。但这里你会发现 `sgx_ipi_cb()` 回调是个空函数！这是因为接收到 IPI 的 CPU 会导致 enclave 的 AEX，而 AEX 会导致硬件（微码）清除 TLB。因此，回调函数什么也不用做，SGX 实现就已经帮我们 flush TLB 了。
* 因为在 SGX 设计上 VA 页面与 enclave 无关，所以需要由软件来记录被换出页面与 VA 页面的关系。因此在回收成功后，内核需负责将：
  * *被换出的 EPC 页面的 nonce 存放在 VA 页面中 slot 的偏移* 记录在`struct encl_page`实例的`desc`域的低 `11~3` bits 当中
  * *指向被换出的 EPC 页面存放 nonce 的 VA 页面的管理结构 `struct sgx_va_page`的指针* 记录在`struct encl_page`实例的`va_page`域
  * 将来换入时`__sgx_encl_eldu()`调用`ELDU`指令时，`%rdx`需传入将要被换入 EPC 页面在 VA slot 的虚拟地址，我们根据以上记录就可以得到该地址
* 你可以认为 `ETRACK` 就是让 SGX 实现去跟踪每个逻辑处理器是否刷新了 TLB，在 `EWB` 的时候会检查这个状态，以确保所有相关的 TLB 都刷新过了，真正刷新 TLB 的操作期望由软件发送 IPI 导致的 AEX 来触发
```cpp
/*
 * Swap page to the regular memory transformed to the blocked state by using
 * EBLOCK, which means that it can no longer be referenced (no new TLB entries).
 *
 * The first trial just tries to write the page assuming that some other thread
 * has reset the count for threads inside the enclave by using ETRACK, and
 * previous thread count has been zeroed out. The second trial calls ETRACK
 * before EWB. If that fails we kick all the HW threads out, and then do EWB,
 * which should be guaranteed the succeed.
 */
 static void sgx_encl_ewb(struct sgx_epc_page *epc_page,
             struct sgx_backing *backing)
{
    struct sgx_encl_page *encl_page = epc_page->owner;
    struct sgx_encl *encl = encl_page->encl;
    struct sgx_va_page *va_page;
    unsigned int va_offset;
    void *va_slot;
    int ret;
    //清除 encl_page 的用户态虚拟地址上的“正在被回收”标志位
    encl_page->desc &= ~SGX_ENCL_PAGE_BEING_RECLAIMED;
    //从 encl 的 VA 页面管理结构链表上拿到第一个实例
    va_page = list_first_entry(&encl->va_pages, struct sgx_va_page,
                   list);
    va_offset = sgx_alloc_va_slot(va_page);//得到欲使用的 slot 在 VA 页面中的偏移
    va_slot = sgx_get_epc_virt_addr(va_page->epc_page) + va_offset;//va_slot 指向 VA 页面中的内核虚拟地址
    if (sgx_va_page_full(va_page)) //如果该 VA 页面的 slot 已满
        list_move_tail(&va_page->list, &encl->va_pages);//就把该实例移到 encl 的 VA 页面链表的末尾
    //反正肯定会有一个 slot，因为在添加 encl page 的时候会用 sgx_encl_grow() 保证每 512 个 EPC 页面对应一个 VA 页面
    ret = __sgx_encl_ewb(epc_page, va_slot, backing);//第一次尝试换出页面
    if (ret == SGX_NOT_TRACKED) {//第一次尝试失败
        ret = __etrack(sgx_get_epc_virt_addr(encl->secs.epc_page)); //用 ETRACK 指令跟踪该 EPC 页面
        if (ret) {
            if (encls_failed(ret))
                ENCLS_WARN(ret, "ETRACK");
        }

        ret = __sgx_encl_ewb(epc_page, va_slot, backing);//第二次尝试换出页面
        if (ret == SGX_NOT_TRACKED) {//第二次尝试失败
            /*
             * Slow path, send IPIs to kick cpus out of the
             * enclave.  Note, it's imperative that the cpu
             * mask is generated *after* ETRACK, else we'll
             * miss cpus that entered the enclave between
             * generating the mask and incrementing epoch.
             */
            //慢速路径，发送 IPI 把 CPUs 踢出 enclave
            //注意极其重要的一点，CPU 掩码在 ETRACK 后生成；
            //否则我们会错过在生成掩码和增加时刻之间进入 enclave 的 CPU。
            //遍历映射 encl 的`struct mm_struct`，收集使用该 encl 的 CPUs
            on_each_cpu_mask(sgx_encl_cpumask(encl),
                     sgx_ipi_cb, NULL, 1);
            ret = __sgx_encl_ewb(epc_page, va_slot, backing);//第三次尝试换出页面
        }
    }

    if (ret) {
        if (encls_failed(ret)) //如果以上的尝试都失败了
            ENCLS_WARN(ret, "EWB");
        //将 struct sgx_va_page 管理结构的实例的 slots 位图中对应的位清除
        sgx_free_va_slot(va_page, va_offset);
    } else {//如果回写成功
        //被换出的 encl 页面的用户态虚拟地址的低 11~3 bits 记录着它在 VA 页面中的偏移
        encl_page->desc |= va_offset;//恰好最大是 (2^9)<<3 = 2^12 = 4096
        encl_page->va_page = va_page;//在被换出的情况下，encl page 的 va_page 指向 VA 页管理结构的实例
    }
}

static void sgx_reclaimer_write(struct sgx_epc_page *epc_page,
                struct sgx_backing *backing)
{
    struct sgx_encl_page *encl_page = epc_page->owner;
    struct sgx_encl *encl = encl_page->encl;
    struct sgx_backing secs_backing; //这个结构只是 EWB 时临时用一下，用它的几个指针
    int ret;

    mutex_lock(&encl->lock);
    //将 encl 的 EPC 页面交换到后备存储
    sgx_encl_ewb(epc_page, backing);
    encl_page->epc_page = NULL; //不再指向 EPC 页面的管理结构
    encl->secs_child_cnt--; //enclave 实际页面计数减 1
    sgx_encl_put_backing(backing); //对应到前面进行 EWB 时的 get
    //如果 encl 的实际 EPC 页面为 0 且已经初始化过了，则考虑把 SECS 页面也换到后备存储上
    if (!encl->secs_child_cnt && test_bit(SGX_ENCL_INITIALIZED, &encl->flags)) {
        ret = sgx_encl_alloc_backing(encl, PFN_DOWN(encl->size), //SECS 页面在 EPC 后备页之后
                       &secs_backing);
        if (ret)
            goto out;
        //将 encl 的 SECS 页面交换到后备存储
        sgx_encl_ewb(encl->secs.epc_page, &secs_backing);
        //从 EPC 中移除 encl 的 SECS 页面，交还其所占的 epc_page 结构
        sgx_encl_free_epc_page(encl->secs.epc_page);
        encl->secs.epc_page = NULL; //页面释放了，epc_page 指针指向 NULL
        //unpin SECS 的后备存储页面
        sgx_encl_put_backing(&secs_backing);
    }

out:
    mutex_unlock(&encl->lock);
}
```
### 释放 EPC 页面
* `sgx_encl_free_epc_page()`释放之前指派给一个 enclave 的 EPC 页面
  * 调用`EREMOVE`指令释放 EPC 页面
  * 调用`sgx_free_epc_page()`将`sgx_epc_page`管理结构放回`node->free_page_list`空闲页链表
```cpp
/**
 * sgx_encl_free_epc_page - free an EPC page assigned to an enclave
 * @page:   EPC page to be freed
 *
 * Free an EPC page assigned to an enclave. It does EREMOVE for the page, and
 * only upon success, it puts the page back to free page list.  Otherwise, it
 * gives a WARNING to indicate page is leaked.
 */
void sgx_encl_free_epc_page(struct sgx_epc_page *page)
{
    int ret;
    //这里不应该还有这个标志，在之前就已经清除了
    WARN_ON_ONCE(page->flags & SGX_EPC_PAGE_RECLAIMER_TRACKED);
    //调用`EREMOVE`指令释放 EPC 页面，如果释放失败管理就不要放回空闲链表了
    ret = __eremove(sgx_get_epc_virt_addr(page));
    if (WARN_ONCE(ret, EREMOVE_ERROR_MESSAGE, ret, ret))
        return;
    //将`sgx_epc_page`管理结构放回`node->free_page_list`空闲页链表
    sgx_free_epc_page(page);
}
```
* `sgx_free_epc_page()`将`sgx_epc_page`管理结构放回`node->free_page_list`空闲页链表
```cpp
/**
 * sgx_free_epc_page() - Free an EPC page
 * @page:   an EPC page
 *
 * Put the EPC page back to the list of free pages. It's the caller's
 * responsibility to make sure that the page is in uninitialized state. In other
 * words, do EREMOVE, EWB or whatever operation is necessary before calling
 * this function.
 */
void sgx_free_epc_page(struct sgx_epc_page *page)
{
    struct sgx_epc_section *section = &sgx_epc_sections[page->section];
    struct sgx_numa_node *node = section->node;

    spin_lock(&node->lock);

    page->owner = NULL; //空闲页面没有 owner
    if (page->poison) //内存错误的 EPC 页面其管理结构要放到中毒列表
        list_add(&page->list, &node->sgx_poison_page_list);
    else //否则将管理结构放回空闲页链表
        list_add_tail(&page->list, &node->free_page_list);
    page->flags = SGX_EPC_PAGE_IS_FREE;//空闲页面是有空闲标志位的

    spin_unlock(&node->lock);
    atomic_long_inc(&sgx_nr_free_pages); //全局空闲 EPC pages 计数递增
}
```
## 缺页的处理

```cpp
sgx_vma_fault()
   encl = vma->vm_private_data; //建立映射 sgx_mmap() 是赋值为 enclave 结构的实例
-> sgx_encl_load_page_in_vma()
       entry = xa_load(&encl->page_array, PFN_DOWN(addr));
   -> __sgx_encl_load_page(encl, entry)
          if (!(encl->secs.epc_page)) //如果连 SECS 的 EPC 页面都不在，先把 SECS 页换入
             sgx_encl_eldu(&encl->secs, NULL); //下面再展开
      -> sgx_encl_eldu(entry, encl->secs.epc_page)
            epc_page = sgx_alloc_epc_page(encl_page, false); //不可回收式分配
         -> __sgx_encl_eldu(encl_page, epc_page, secs_page)
               page_pcmd_off = sgx_encl_get_backing_page_pcmd_offset(encl, page_index);
            -> sgx_encl_lookup_backing(encl, page_index, &b)
               -> __sgx_encl_get_backing(encl, page_index, backing)
                      pgoff_t page_pcmd_off = sgx_encl_get_backing_page_pcmd_offset(encl, page_index); //EPC 页在后备存储中对应的 PCMD 结构的偏移
                      contents = sgx_encl_get_backing_page(encl, page_index); //EPC 页在后备存储中对应的 page
                      pcmd = sgx_encl_get_backing_page(encl, PFN_DOWN(page_pcmd_off)) //EPC 页在后备存储中对应的 PCMD 结构所在的 page
                      backing->contents = contents;
                      backing->pcmd = pcmd;
                      backing->pcmd_offset = page_pcmd_off & (PAGE_SIZE - 1); //EPC 页在后备存储中对应的 PCMD 结构所在的 page 中的偏移
               pginfo.addr = encl_page->desc & PAGE_MASK; //找回在 encl 中的虚拟地址
               pginfo.contents = (unsigned long)kmap_atomic(b.contents); //在内核中映射 EPC 页在后备存储中对应的 page
               pcmd_page = kmap_atomic(b.pcmd); //在内核中映射EPC 页在后备存储中对应的 PCMD 结构所在的 page
               pginfo.metadata = (unsigned long)pcmd_page + b.pcmd_offset;//EPC 页在后备存储中对应的 PCMD 结构的虚拟地址
            -> __eldu(&pginfo, sgx_get_epc_virt_addr(epc_page), sgx_get_epc_virt_addr(encl_page->va_page->epc_page) + va_offset)
            -> memset(pcmd_page + b.pcmd_offset, 0, sizeof(struct sgx_pcmd))
            -> set_page_dirty(b.pcmd)
               pcmd_page_empty = !memchr_inv(pcmd_page, 0, PAGE_SIZE); //后备存储中关联的 PCMD 页是否为空
            -> sgx_encl_put_backing(&b)
            -> sgx_encl_truncate_backing_page(encl, page_index) //从后备存储中删除 EPC 页
                   struct inode *inode = file_inode(encl->backing);
               -> shmem_truncate_range(inode, PFN_PHYS(page_index), PFN_PHYS(page_index) + PAGE_SIZE - 1)
               if (pcmd_page_empty && !reclaimer_writing_to_pcmd(encl, pcmd_first_page)) //检查共享同一 PCMD 页的其他 encl 页
                  sgx_encl_truncate_backing_page(encl, PFN_DOWN(page_pcmd_off)) //其他 encl 页都没在被回收的过程中，才能删 PCMD 页
         -> sgx_free_va_slot(encl_page->va_page, va_offset);
         -> list_move(&encl_page->va_page->list, &encl->va_pages);
            encl_page->desc &= ~SGX_ENCL_PAGE_VA_OFFSET_MASK;
            encl_page->epc_page = epc_page;
      -> sgx_mark_page_reclaimable(entry->epc_page)
             page->flags |= SGX_EPC_PAGE_RECLAIMER_TRACKED;
         -> list_add_tail(&page->list, &sgx_active_page_list) //EPC 页管理结构放到 active 链表上
   phys_addr = sgx_get_epc_phys_addr(entry->epc_page);
-> vmf_insert_pfn(vma, addr, PFN_DOWN(phys_addr))
-> sgx_encl_test_and_clear_young(vma->vm_mm, entry)
   -> sgx_encl_find(mm, addr, &vma)
      -> vma_lookup(mm, addr)
   -> apply_to_page_range(vma->vm_mm, addr, PAGE_SIZE, sgx_encl_test_and_clear_young_cb, vma->vm_mm)
```

* `sgx_mmap()`时将`vma->vm_ops`的域设置为`sgx_vm_ops`
```cpp
const struct vm_operations_struct sgx_vm_ops = {
    .fault = sgx_vma_fault,
    .mprotect = sgx_vma_mprotect,
    .open = sgx_vma_open,
    .access = sgx_vma_access,
};
```
* 这样，当 VMA 区域发生缺页时就会用`sgx_vma_fault()`进行处理
```cpp
static vm_fault_t sgx_vma_fault(struct vm_fault *vmf)
{
    unsigned long addr = (unsigned long)vmf->address;
    struct vm_area_struct *vma = vmf->vma;
    struct sgx_encl_page *entry;
    unsigned long phys_addr;
    struct sgx_encl *encl;
    vm_fault_t ret;

    encl = vma->vm_private_data; //sgx_mmap()时赋的值
    //在 sgx_vma_open() 时可能会因分不出内存给 mm_list 条目而失败导致 encl 为 NULL
    /*
     * It's very unlikely but possible that allocating memory for the
     * mm_list entry of a forked process failed in sgx_vma_open(). When
     * this happens, vm_private_data is set to NULL.
     */
    if (unlikely(!encl))
        return VM_FAULT_SIGBUS;
    /*
     * The page_array keeps track of all enclave pages, whether they
     * are swapped out or not. If there is no entry for this page and
     * the system supports SGX2 then it is possible to dynamically add
     * a new enclave page. This is only possible for an initialized
     * enclave that will be checked for right away.
     */
    if (cpu_feature_enabled(X86_FEATURE_SGX2) &&
        (!xa_load(&encl->page_array, PFN_DOWN(addr))))
        return sgx_encl_eaug_page(vma, encl, addr); //SGX2 支持动态添加一个 enclave 页面

    mutex_lock(&encl->lock);
    //提供缺页的用户态虚拟地址，载入/换入 EPC 页面到 EPC
    entry = sgx_encl_load_page_in_vma(encl, addr, vma->vm_flags);
    if (IS_ERR(entry)) {
        mutex_unlock(&encl->lock);

        if (PTR_ERR(entry) == -EBUSY)
            return VM_FAULT_NOPAGE;

        return VM_FAULT_SIGBUS;
    }
    //根据 EPC 页面的管理结构得到其物理地址
    phys_addr = sgx_get_epc_phys_addr(entry->epc_page);
    //将物理地址转为 PFN 后插入到 VMA
    ret = vmf_insert_pfn(vma, addr, PFN_DOWN(phys_addr));
    if (ret != VM_FAULT_NOPAGE) {
        mutex_unlock(&encl->lock);

        return VM_FAULT_SIGBUS;
    }
    //测试并清除 accessed 位
    sgx_encl_test_and_clear_young(vma->vm_mm, entry);
    mutex_unlock(&encl->lock);

    return VM_FAULT_NOPAGE;
}
```
* `sgx_encl_truncate_backing_page()`根据用户态虚拟地址转换成的页索引从后备存储中释放页面
```cpp
/*
 * Free a page from the backing storage in the given page index.
 */
static inline void sgx_encl_truncate_backing_page(struct sgx_encl *encl, unsigned long page_index)
{
    struct inode *inode = file_inode(encl->backing);
    //从 shmem 中释放指定范围的页面
    shmem_truncate_range(inode, PFN_PHYS(page_index), PFN_PHYS(page_index) + PAGE_SIZE - 1);
}

// 32 个 PCMD 条目共享一个 PCMD 页面。
// PCMD_FIRST_MASK 用于确定与 PCMD 页内的第一个 PCMD 条目相关联的页索引。
/*
 * 32 PCMD entries share a PCMD page. PCMD_FIRST_MASK is used to
 * determine the page index associated with the first PCMD entry
 * within a PCMD page.
 */
#define PCMD_FIRST_MASK GENMASK(4, 0)
...
/*
 * ELDU: Load an EPC page as unblocked. For more info, see "OS Management of EPC
 * Pages" in the SDM.
 */
static int __sgx_encl_eldu(struct sgx_encl_page *encl_page,
               struct sgx_epc_page *epc_page,
               struct sgx_epc_page *secs_page)
{   //被换出页面的用户态虚拟地址低 11~3 位会被来记录其在 VA 页面中的偏移
    unsigned long va_offset = encl_page->desc & SGX_ENCL_PAGE_VA_OFFSET_MASK;
    struct sgx_encl *encl = encl_page->encl;
    pgoff_t page_index, page_pcmd_off;
    unsigned long pcmd_first_page;
    struct sgx_pageinfo pginfo;
    struct sgx_backing b;
    bool pcmd_page_empty;
    u8 *pcmd_page;
    int ret;
    //载入常规页时，secs_page 为指向 SECS 页面的 EPC 页面管理结构
    if (secs_page)  //根据偏移算出 PFN 索引
        page_index = PFN_DOWN(encl_page->desc - encl_page->encl->base);
    else //SECS 页面在常规页面的最后，所以这里直接用 encl->size 就能得到它的索引了
        page_index = PFN_DOWN(encl->size);
    //使用 PCMD 页内的第一个条目的 enclave 页的地址
    /*
     * Address of enclave page using the first entry within the PCMD page.
     */
    pcmd_first_page = PFN_PHYS(page_index & ~PCMD_FIRST_MASK) + encl->base;
    //根据 PFN 索引得到和一个 enclave 页面关联的 PCMD struct 的偏移字节数
    page_pcmd_off = sgx_encl_get_backing_page_pcmd_offset(encl, page_index);
    //根据用户态页索引填充 b 的各个域，是 lookup 不是 alloc，涉及到的页是已分配的
    ret = sgx_encl_lookup_backing(encl, page_index, &b);
    if (ret)
        return ret;
    //填充用于 ELDU 指令的 PGINFO 的各个域，contents 和 pcmd 都是 struct *page，需要 kmap_atomic() 转为有效的线性地址
    pginfo.addr = encl_page->desc & PAGE_MASK;
    pginfo.contents = (unsigned long)kmap_atomic(b.contents);
    pcmd_page = kmap_atomic(b.pcmd);
    pginfo.metadata = (unsigned long)pcmd_page + b.pcmd_offset;

    if (secs_page)
        pginfo.secs = (u64)sgx_get_epc_virt_addr(secs_page);
    else
        pginfo.secs = 0;
    //调用 ELDU 指令将后备存储中的页面换入 EPC
    ret = __eldu(&pginfo, sgx_get_epc_virt_addr(epc_page),
             sgx_get_epc_virt_addr(encl_page->va_page->epc_page) + va_offset);
    if (ret) {
        if (encls_failed(ret))
            ENCLS_WARN(ret, "ELDU");

        ret = -EFAULT;
    }
    //pcmd_page 指向 pcmd 所在的页临时映射的内核虚拟地址，b.pcmd_offset 为 pcmd 在页中的偏移
    memset(pcmd_page + b.pcmd_offset, 0, sizeof(struct sgx_pcmd));//换入成功，将对应的 pcmd 清零
    set_page_dirty(b.pcmd); //load 了一个页面，清楚了对应的 PCMD 条目，所以单独设置了 PCMD 页面 dirty 以写回
    //memchr_inv() 在 pcmd_page 中找到一个不匹配 0 的字符，找不到意味着整个页都为 0，即所有的 pcmd 都清零了
    /*
     * The area for the PCMD in the page was zeroed above.  Check if the
     * whole page is now empty meaning that all PCMD's have been zeroed:
     */
    pcmd_page_empty = !memchr_inv(pcmd_page, 0, PAGE_SIZE);
    //解除上面的临时映射
    kunmap_atomic(pcmd_page);
    kunmap_atomic((void *)(unsigned long)pginfo.contents);

    get_page(b.pcmd);
    sgx_encl_put_backing(&b); //对应到上面的 lookup_backing
    //EPC 页面已换入 EPC 了，释放后备存储中的备份页
    sgx_encl_truncate_backing_page(encl, page_index);
    //光检查 PCMD 页面里的内容还不够，可能有共享该 PCMD 页面的 enclave 页面正在被回收，此时看到 PCMD 页面里内容也为空
    //因此还要额外检查一下其他共享该 PCMD 页面的 enclave 页面是否正在被回收
    if (pcmd_page_empty && !reclaimer_writing_to_pcmd(encl, pcmd_first_page)) {
        sgx_encl_truncate_backing_page(encl, PFN_DOWN(page_pcmd_off)); //释放后备存储中的 PCMD 页
        pcmd_page = kmap_atomic(b.pcmd);
        if (memchr_inv(pcmd_page, 0, PAGE_SIZE))
            pr_warn("PCMD page not empty after truncate.\n");//呃，好吧，还是有页面在截断后插进来了
        kunmap_atomic(pcmd_page);
    }

    put_page(b.pcmd);

    return ret;
}
```
* 注意，换入会导致`va_page`被移动到`encl->va_pages`链表的前面
  * 回忆之前换出时是从`encl->va_pages`链表的第一个节点取`va_page`，当 VA 相关操作完成会检查 VA 页面的 slots 是否已满，如果满了就把`va_page`节点移到链表末尾
  * 如果换入的页面不在一个`va_page`的 slots 管理范围之内，这里会不会导致频繁的链表操作呢？因为 slots 管理的页面是位置无关的吧？
    * 比如，连续页面范围内，当时有的页面活跃，有的不活跃。不活跃的被率先换出，占据了 VA 页面靠前的 slot，这就导致 slot 的顺序不一定对应的页面的位置
    * 而换入的时候是根据缺页时的用户态虚拟地址来找到 EPC 页面的，这就不好说该 EPC 页面被换出时 VA 页面 slot 是否是在`encl->va_pages`链表的前面了
    * 这并不 LRU，有改进的空间？
  * 但这么做至少能保证`encl->va_pages`链表的第一个`va_page`肯定有空的 slot
```cpp
static struct sgx_epc_page *sgx_encl_eldu(struct sgx_encl_page *encl_page,
                      struct sgx_epc_page *secs_page)
{
    //还记得吗？被换出页面的用户态虚拟地址低 11~3 bits 会被来记录其在 VA 页面中的偏移
    unsigned long va_offset = encl_page->desc & SGX_ENCL_PAGE_VA_OFFSET_MASK;
    struct sgx_encl *encl = encl_page->encl;
    struct sgx_epc_page *epc_page;
    int ret;
    //分配（借出）一个空闲 EPC 页面
    epc_page = sgx_alloc_epc_page(encl_page, false);//此时不要再触发回收
    if (IS_ERR(epc_page))
        return epc_page;
    //将页面从后备存储换入到 EPC
    ret = __sgx_encl_eldu(encl_page, epc_page, secs_page);
    if (ret) {
        sgx_encl_free_epc_page(epc_page);
        return ERR_PTR(ret);
    }
    //根据换出时在 VA 页面中的偏移，清除其 va_page 的 slots 位图中的位
    sgx_free_va_slot(encl_page->va_page, va_offset);
    list_move(&encl_page->va_page->list, &encl->va_pages);//把 va_page 移动到 encl va_pages 链表的前面
    encl_page->desc &= ~SGX_ENCL_PAGE_VA_OFFSET_MASK;//已经被换入了，不需要再记录在 VA 页面中的偏移
    encl_page->epc_page = epc_page;//页面在 EPC 中了 ，更新指向 EPC 页面管理结构的指针
    //返回指向 EPC 页面管理结构的指针
    return epc_page;
}
```
* `__sgx_encl_load_page()`负责加载给定用户态虚拟地址`addr`对应的 EPC 页面
  * 它有两个封装函数`sgx_encl_load_page_in_vma()`和`sgx_encl_load_page()`，前者有对 VMA 权限的检查，要载入的页构建时赋予的权限需高于 VMA 的权限
```cpp
static struct sgx_encl_page *__sgx_encl_load_page(struct sgx_encl *encl,
                          struct sgx_encl_page *entry)
{
    struct sgx_epc_page *epc_page;
    //如果当初是通过 add_page 加过这一页的，接着处理对应的 EPC 页面各种情况
    /* Entry successfully located. */
    if (entry->epc_page) {
        if (entry->desc & SGX_ENCL_PAGE_BEING_RECLAIMED) //该页面正在被回收
            return ERR_PTR(-EBUSY); //缺页处理出错
        //成功定位用户态虚拟地址对应的 encl_page，该页面未被回收
        return entry;
    }
    //如果该 encl 的 SECS 页面已被换出 EPC，先要把它换回来
    if (!(encl->secs.epc_page)) {
        epc_page = sgx_encl_eldu(&encl->secs, NULL);//重新换入 SECS 页面到 EPC
        if (IS_ERR(epc_page))
            return ERR_CAST(epc_page);
    }
    //重新换入缺页的常规页面到 EPC
    epc_page = sgx_encl_eldu(entry, encl->secs.epc_page);
    if (IS_ERR(epc_page))
        return ERR_CAST(epc_page);

    encl->secs_child_cnt++;//encl 有效 EPC 页面数递增
    sgx_mark_page_reclaimable(entry->epc_page);//标识该页可回收，并放到 sgx_active_page_list 上
    //成功换入用户态虚拟地址对应的 encl_page
    return entry;
}

static struct sgx_encl_page *sgx_encl_load_page_in_vma(struct sgx_encl *encl,
                               unsigned long addr,
                               unsigned long vm_flags)
{
    unsigned long vm_prot_bits = vm_flags & (VM_READ | VM_WRITE | VM_EXEC);
    struct sgx_encl_page *entry;
    //根据用户态虚拟地址从 xarray 中找到 sgx_encl_page 实例
    entry = xa_load(&encl->page_array, PFN_DOWN(addr));
    if (!entry) //如果找不到，说明当初就没用 add_page 加过这一页
        return ERR_PTR(-EFAULT);
    //要载入的页构建时赋予的权限需高于 VMA 的权限
    /*
     * Verify that the page has equal or higher build time
     * permissions than the VMA permissions (i.e. the subset of {VM_READ,
     * VM_WRITE, VM_EXECUTE} in vma->vm_flags).
     */
    if ((entry->vm_max_prot_bits & vm_prot_bits) != vm_prot_bits)
        return ERR_PTR(-EFAULT);

    return __sgx_encl_load_page(encl, entry);
}

struct sgx_encl_page *sgx_encl_load_page(struct sgx_encl *encl,
                     unsigned long addr)
{
    struct sgx_encl_page *entry;
    //根据用户态虚拟地址从 xarray 中找到 sgx_encl_page 实例
    entry = xa_load(&encl->page_array, PFN_DOWN(addr));
    if (!entry) //如果找不到，说明当初就没用 add_page 加过这一页
        return ERR_PTR(-EFAULT);

    return __sgx_encl_load_page(encl, entry);
}
```
* `sgx_mark_page_reclaimable()`把一个 EPC 页面标识为可回收，需传入 EPC 页面管理结构的指针
  * 该函数会把`SGX_EPC_PAGE_RECLAIMER_TRACKED` flag 给设上，表示该页可回收
  * 重要的是它会把 EPC 页面管理结构加到全局`sgx_active_page_list`的末尾
  * 该函数会被`sgx_encl_add_page()`和`sgx_encl_load_page()`调用，即 *加入新 EPC 页面到 enclave* 和 *重新将页面换入 EPC* 时
```cpp
/**
 * sgx_mark_page_reclaimable() - Mark a page as reclaimable
 * @page:   EPC page
 *
 * Mark a page as reclaimable and add it to the active page list. Pages
 * are automatically removed from the active list when freed.
 */
void sgx_mark_page_reclaimable(struct sgx_epc_page *page)
{
    spin_lock(&sgx_reclaimer_lock);
    page->flags |= SGX_EPC_PAGE_RECLAIMER_TRACKED;
    list_add_tail(&page->list, &sgx_active_page_list);
    spin_unlock(&sgx_reclaimer_lock);
}
```
* `sgx_unmark_page_reclaimable()`移除页面的可回收标志，并把页面从全局`sgx_active_page_list`上移除
  * 该函数被`sgx_encl_release()`调用，即释放 enclave 的时候要先检查该页是否有可回收标志，如果有先从全局`sgx_active_page_list`上移除，然后再由`sgx_encl_release()`调用`sgx_encl_free_epc_page()`放回`node->free_page_list`
```cpp
/**
 * sgx_unmark_page_reclaimable() - Remove a page from the reclaim list
 * @page:   EPC page
 *
 * Clear the reclaimable flag and remove the page from the active page list.
 *
 * Return:
 *   0 on success,
 *   -EBUSY if the page is in the process of being reclaimed
 */
int sgx_unmark_page_reclaimable(struct sgx_epc_page *page)
{
    spin_lock(&sgx_reclaimer_lock);
    if (page->flags & SGX_EPC_PAGE_RECLAIMER_TRACKED) {
        /* The page is being reclaimed. */
        if (list_empty(&page->list)) {
            spin_unlock(&sgx_reclaimer_lock);
            return -EBUSY;
        }

        list_del(&page->list);
        page->flags &= ~SGX_EPC_PAGE_RECLAIMER_TRACKED;
    }
    spin_unlock(&sgx_reclaimer_lock);

    return 0;
}
```
* `reclaimer_writing_to_pcmd()` 查询是否有任何 enclave page 关联到处于回收过程中的 PCMD 页面
* 一个 PCMD 页面只有在以下情况时才能被截断：
  1. 为空，
  2. 不在获取数据的过程中（因此很快就非空）。
* 通过检查共享该 PCMD 页面的 enclave 页面是否在被回收的过程中进行测试。
* 当回收者打算回收该 enclave 页面时，它会设置`SGX_ENCL_PAGE_BEING_RECLAIMED`标志 - 这意味着与该 enclave 页面关联的 PCMD 页面即将得到一些数据，因此即使 PCMD 页面为空，也不应该被截断。
* 如果`SGX_ENCL_PAGE_BEING_RECLAIMED`标志不为空，则在获取数据（被回收）的过程中，返回`1`。
```cpp
/**
 * reclaimer_writing_to_pcmd() - Query if any enclave page associated with
 *                               a PCMD page is in process of being reclaimed.
 * @encl:        Enclave to which PCMD page belongs
 * @start_addr:  Address of enclave page using first entry within the PCMD page
 *
 * When an enclave page is reclaimed some Paging Crypto MetaData (PCMD) is
 * stored. The PCMD data of a reclaimed enclave page contains enough
 * information for the processor to verify the page at the time
 * it is loaded back into the Enclave Page Cache (EPC).
 *
 * The backing storage to which enclave pages are reclaimed is laid out as
 * follows:
 * Encrypted enclave pages:SECS page:PCMD pages
 *
 * Each PCMD page contains the PCMD metadata of
 * PAGE_SIZE/sizeof(struct sgx_pcmd) enclave pages.
 *
 * A PCMD page can only be truncated if it is (a) empty, and (b) not in the
 * process of getting data (and thus soon being non-empty). (b) is tested with
 * a check if an enclave page sharing the PCMD page is in the process of being
 * reclaimed.
 *
 * The reclaimer sets the SGX_ENCL_PAGE_BEING_RECLAIMED flag when it
 * intends to reclaim that enclave page - it means that the PCMD page
 * associated with that enclave page is about to get some data and thus
 * even if the PCMD page is empty, it should not be truncated.
 *
 * Context: Enclave mutex (&sgx_encl->lock) must be held.
 * Return: 1 if the reclaimer is about to write to the PCMD page
 *         0 if the reclaimer has no intention to write to the PCMD page
 */
static int reclaimer_writing_to_pcmd(struct sgx_encl *encl,
                     unsigned long start_addr)
{
    int reclaimed = 0;
    int i;

    /*
     * PCMD_FIRST_MASK is based on number of PCMD entries within
     * PCMD page being 32.
     */
    BUILD_BUG_ON(PCMDS_PER_PAGE != 32);
    //逐个检查与该 PCMD 页关联的 enclave 页，共 32 个
    for (i = 0; i < PCMDS_PER_PAGE; i++) {
        struct sgx_encl_page *entry;
        unsigned long addr;
        //start_addr 为使用 PCMD 页内的第一个条目的 enclave 页的地址
        addr = start_addr + i * PAGE_SIZE;

        /*
         * Stop when reaching the SECS page - it does not
         * have a page_array entry and its reclaim is
         * started and completed with enclave mutex held so
         * it does not use the SGX_ENCL_PAGE_BEING_RECLAIMED
         * flag.
         */
        if (addr == encl->base + encl->size)
            break;
        //从 enclave 页面 XA 中找到该地址对应的 encl_page
        entry = xa_load(&encl->page_array, PFN_DOWN(addr));
        if (!entry)
            continue;
        //VA slot ID 用 desc 的 11~3 bit，BEING_RECLAIMED flag 用第 3 bit，
        //但已在后备存储的页面 entry->epc_page 为 NULL，所以这里先判断一下：
        //1. 如果为 NULL，则该页已回收，不再考虑
        //2. 如果不为 NULL，则该页可能在被回收的过程中，也可能不在，进一步检查标志位
        /*
         * VA page slot ID uses same bit as the flag so it is important
         * to ensure that the page is not already in backing store.
         */
        if (entry->epc_page &&
            (entry->desc & SGX_ENCL_PAGE_BEING_RECLAIMED)) {
            reclaimed = 1;
            break;
        }
    }

    return reclaimed;
}
```
## EPC 页面全局链表
* EPC 页面管理结构会被挂在三个全局链表的其中一个上，分别是：
1. `sgx_dirty_page_list`：系统启动时创建好的 EPC 页面管理结构都挂在该链表上
2. `sgx_active_page_list`：正在使用的 EPC 页面其对应的管理结构挂在该链表上
   * `add_page`把页面加入 EPC 后
   * 从后备存储 **换入** EPC 后
3. `sgx_numa_node[].free_page_list`：未被使用的 EPC 页面其管理结构挂在该链表上
   * 页面从 EPC **回收（换出）** 到后备存储后
   * 释放 EPC 页面后，比如各种出错、释放 enclave 等
4. 另外还有一个`sgx_numa_node[].sgx_poison_page_list`链表用来挂出现 memory failure 的页面
5. 还值得一提的是用于 VA 的 EPC 页面管理结构不在以上任何一个链表上，而是通过 per enclave 的`struct sgx_encl`的`va_pages`管理的`struct sgx_va_page`来管理

## ksgxd
* SGX 负责页面回收的内核线程叫做`ksgxd`
### EPC sanitization
* `ksgxd`在 SGX 初始化时启动。当处理器通电或重置时，enclave 内存通常已准备好使用。但是，如果 SGX 自重置后一直在使用，enclave 页面可能处于不一致状态。例如，这可能发生在崩溃和`kexec()`周期之后。在启动时，`ksgxd`重新初始化所有 enclave 页面，以便它们可以被分配和重用。
* 清理是通过遍历 EPC 地址空间并将`EREMOVE` function 应用于每个物理页面来完成的
  * 一些像 SECS 页面这样的 enclave 页面对其他页面具有硬件依赖性，这会阻止`EREMOVE`运行
  * 执行两轮`EREMOVE`以删除依赖
  * 消杀成功的 EPC 页面会从`sgx_dirty_page_list`移到`node->free_page_list`，这样分配器才可以分配该页面
### ksgxd 内存回收
* 与核心`kswapd`类似，`ksgxd`负责管理 enclave 内存的过度使用。如果系统用完 enclave 内存，`ksgxd`会将 enclave 内存“交换”到正常内存。

### ksgxd 的实现
* 将 kexec 后的 EPC 页面重置为未初始化状态
* 从输入的链表中把这些页面删除，并供页面分配器使用
* 在输入的链表中添加其子项的 SECS 页面保持不变
```cpp
static LIST_HEAD(sgx_dirty_page_list);

/*
 * Reset post-kexec EPC pages to the uninitialized state. The pages are removed
 * from the input list, and made available for the page allocator. SECS pages
 * prepending their children in the input list are left intact.
 */
static void __sgx_sanitize_pages(struct list_head *dirty_page_list)
{
    struct sgx_epc_page *page;
    LIST_HEAD(dirty);
    int ret;
    //dirty_page_list 是本地线程的，所以不需要锁，我的理解这里指的是硬件
    /* dirty_page_list is thread-local, no need for a lock: */
    while (!list_empty(dirty_page_list)) {
        if (kthread_should_stop())
            return;

        page = list_first_entry(dirty_page_list, struct sgx_epc_page, list);
        //处理内存失败的页面，都挂在 node->sgx_poison_page_list 链表
        /*
         * Checking page->poison without holding the node->lock
         * is racy, but losing the race (i.e. poison is set just
         * after the check) just means __eremove() will be uselessly
         * called for a page that sgx_free_epc_page() will put onto
         * the node->sgx_poison_page_list later.
         */
        if (page->poison) {
            struct sgx_epc_section *section = &sgx_epc_sections[page->section];
            struct sgx_numa_node *node = section->node;

            spin_lock(&node->lock);
            list_move(&page->list, &node->sgx_poison_page_list);
            spin_unlock(&node->lock);

            continue;
        }
        //调用 EEREMOVE 指令从 EPC 中移除页面
        ret = __eremove(sgx_get_epc_virt_addr(page));
        if (!ret) { //如果消杀成功
            /*
             * page is now sanitized.  Make it available via the SGX
             * page allocator:
             */
            list_del(&page->list); //从原来的链表（sgx_dirty_page_list）上移除
            sgx_free_epc_page(page); //放到 node->free_page_list 上，并置上空闲标志，这样分配器就可以重新分配了
        } else { //如果消杀失败
            /* The page is not yet clean - move to the dirty list. */
            list_move_tail(&page->list, &dirty); //暂存在本地 dirty 链表上
        }

        cond_resched();
    }
    //暂存在本地 dirty 链表上的节点拼接回 dirty_page_list，也就是 sgx_dirty_page_list
    list_splice(&dirty, dirty_page_list);
}
```
* `ksgxd()`最开始启动时需要执行两轮`EREMOVE`以删除依赖，之后就不用了
```cpp
static DECLARE_WAIT_QUEUE_HEAD(ksgxd_waitq);
...
// 唤醒 ksgxd 的条件之一
static bool sgx_should_reclaim(unsigned long watermark)
{
    return atomic_long_read(&sgx_nr_free_pages) < watermark &&
           !list_empty(&sgx_active_page_list);
}

static int ksgxd(void *p)
{
    set_freezable();

    /*
     * Sanitize pages in order to recover from kexec(). The 2nd pass is
     * required for SECS pages, whose child pages blocked EREMOVE.
     */
    __sgx_sanitize_pages(&sgx_dirty_page_list);
    __sgx_sanitize_pages(&sgx_dirty_page_list);
    //刚启动时 sgx_dirty_page_list 链表不太可能为空
    /* sanity check: */
    WARN_ON(!list_empty(&sgx_dirty_page_list));
    //该 kthread 是否该停止？不该停止就继续循环
    while (!kthread_should_stop()) {
        if (try_to_freeze())
            continue;

        wait_event_freezable(ksgxd_waitq, //在等待队列上睡眠，后面的是唤醒条件
                     kthread_should_stop() || //kthread 该停止了
                     sgx_should_reclaim(SGX_NR_HIGH_PAGES)); //空闲页低于高水标
        //空闲页低于高水标，（继续）回收 EPC 页面
        if (sgx_should_reclaim(SGX_NR_HIGH_PAGES))
            sgx_reclaim_pages();

        cond_resched();
    }

    return 0;
}
```

## Sysfs Interfaces
* `/sys/devices/system/node/node0/x86/sgx_total_bytes`
## References
- [Software Guard eXtensions (SGX) — The Linux Kernel documentatio](https://www.kernel.org/doc/html/latest/x86/sgx.html)
- [PAT (Page Attribute Table) — The Linux Kernel documentatio](https://www.kernel.org/doc/html/latest/x86/pat.html)
- [MTRR and PAT - Gentoo Wiki](https://wiki.gentoo.org/wiki/MTRR_and_PAT)
- [Linux 共享内存](https://gohalo.me/post/linux-program-shared-memory.html)
- [宋宝华：世上最好的共享内存(Linux共享内存最透彻的一篇)](https://cloud.tencent.com/developer/article/1551288)
- [内存管理源码分析-mmap函数在内核的运行机制以及源码分析](https://blog.csdn.net/u011649400/article/details/104525439)
- [谁来拯救存量SGX1平台？又一个内核特性合并的血泪史｜推荐](https://mp.weixin.qq.com/s/hqPF3fuGdmQrRiXX0OS4kw)