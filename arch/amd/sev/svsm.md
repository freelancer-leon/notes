# SVSM

# 2 文档范围
* 本文档将 **安全虚拟机服务模块（Secure VM Service Module, SVSM）** 定义为一个可在 guest 内部托管特权模块的环境。
* 同时，本文档还定义了以下机制：
  * Guest OS 如何检测 SVSM 的存在
  * SVSM 所提供的服务集合
  * Guest 与这些服务进行通信的机制
* 文档不包含的内容
  * SVSM 内部实现细节：本文档鼓励开发多种 SVSM 实现，并定义了 SVSM 与 guest OS 通信的标准，以确保不同的 SVSM 实现能与多种 guest OS 兼容。
  * SVSM 加载机制：虚拟机管理器（VMM）加载 SVSM 的机制因 host 架构而异，本文档不做描述。
* **SVSM 和固件应作为初始 guest 镜像的一部分被加载和测量。** 这样，通过测量和远程证明机制，即可验证与特定 guest 相关联的 SVSM 和固件的身份。预期将对以下项目进行测量：
* 在 VMPL0 度量的项目
  * SVSM 二进制
  * SVSM BSP VM Save Area (VMSA) 作为一个 VMSA 页面
  * Firmware BSP VMSA 作为一个普通页面
  * Secrets 页面（secrets page）
  * CPUID 页面
* 在 VMPL1+ 度量的项目
  * 固件二进制
* Hypervisor 可以根据需要测量额外的页面。
* 本文档不描述 VMM 与 SVSM 通信或调用 SVSM 的机制。这些细节被认为是 host 环境特定的，尽管它们可以在不同的规范下被标准化。
* 本文档不描述 SVSM 的线程模型。不同的 SVSM 实现可能采用不同的模型：
  * 为每个 guest vCPU 分配一个独立的执行上下文（一个唯一的 VMSA）
  * 或使用单个执行上下文为所有 guest vCPU 提供服务
* SVSM 与 host 之间关于线程模型的协商，以及 host 指示哪个 vCPU 发起请求的机制，都被认为是 host 环境特定的，尽管它们也可以在不同的规范下被标准化。

# 3 环境

* SVSM 预期在 guest 的 **VMPL0** 级别执行。为确保安全敏感服务的特权分离，guest 的大部分组件应在非 0 的 VMPL 级别运行。
* SVSM 必须提供代理服务，处理那些原本需要在 VMPL0 级别执行但在其他级别下无法完成的操作，例如：
  - 使用 `PVALIDATE` 指令
  - 通过 `RMPADJUST` 创建 VMSA 页面
* 这些服务构成了“核心协议”的一部分。
* **地址空间**：SVSM 镜像及其初始数据应占据 guest 物理地址（gPA）空间的连续区域
* **权限设置**：该内存区域必须配置为仅允许 VMPL0 访问（数值越高的 VMPL 级别权限越低）
* **初始化要求**：初始内存配置必须足够支持 SVSM 完成初始化并提供所有核心协议服务
* **扩展机制**：文档定义了 SVSM 获取额外内存的机制，以支持 guest OS 可能请求的额外服务

# 4 发现
## 4.1 启动
* 当 guest 首次启动时，在任何较低特权级别的 VMPL 启动之前，它将首先在 SVSM 的上下文环境中执行。这使得 SVSM 能够完成自身的初始化，并使其能够被发现。
* SVSM 通过将信息写入 secrets page 来宣告其存在，如下表所述。
* 注意，secrets page 中位于此处描述的字节偏移处的部分，在 secrets page 的初始创建阶段会被清零，并保留供 SVSM 使用。
  * 除非由 SVSM 进行初始化，否则它们对于任何 guest 而言都将始终为 `0`。
* Table 1: Secrets Page Fields

字节偏移 | 大小   | 域名称             | 描述
--------|--------|-------------------|------------------------------------------
0x140   | 8 字节 | SVSM_BASE         | SVSM 区域的 gPA 基地址。必须是 `4KB` 的倍数
0x148   | 8 字节 | SVSM_SIZE         | SVSM 区域的字节数。必须是 `4KB` 的倍数
0x150   | 8 字节 | SVSM_CAA          | 用于 guest/SVSM 通信的 `4KB` 区域的 gPA
0x158   | 4 字节 | SVSM_MAX_VERSION  | SVSM 所支持的核心协议的最大版本
0x15C   | 1 字节 | SVSM_GUEST_VMPL   | 指示 guest 正在其下执行的 VMPL
0x15D   | 3 字节 | .                 | 保留

* 注意，SVSM 应将 secrets page 中包含与 VMPL0 相关联的 *虚拟机平台通信密钥（VMPCK）* 的部分清零，以防止 guest OS 干扰 SVSM 与 PSP 之间的任何通信。
* 当 guest OS 启动时，它必须首先读取 `SVSM_SIZE` 字段。
  * 如果该字段为 `0`，则表示不存在 SVSM。
  * 如果该字段非 `0`，表明存在 SVSM，那么 guest 必须记录 SVSM 所占据的内存范围，以避免尝试使用该范围内的任何内存。
    * （例如，它可以选择将该内存标识为 `EfiPlatformReserved`。）
    * 它必须使用指定的 `SVSM_CAA` 值来与 SVSM 进行通信。
* 本规范假设初始 guest 镜像仅包含一个用于启动 vCPU 的 VMSA，其内容已作为 guest 启动过程的一部分被测量，并在执行初始 guest 镜像之前由 SVSM 进行了验证。
  * 如果 guest 需要额外的 VMSA，它们应当被动态创建。（参见第 20 页的 “SVSM_CORE_CREATE_VCPU 调用” 一节。）
* 预期 SVSM 会检查为 guest 指定的 `SEV_FEATURES`，如果启用了不支持的特性，则会终止运行。
  * 例如，如果 guest OS 的 `SEV_FEATURES` 启用了 `VmsaRegProt` 特性，但 SVSM 不支持访问此类 VMSA，那么 SVSM 可以终止运行。

## 4.2 Post-Boot

* 在启动后，某些无法访问 secrets page 的 guest 组件（例如，在 guest OS 启动并接管 SVSM 接口后调用的 UEFI 运行时服务）可能仍需要调用 SVSM。对于这些组件，需要一种备用的发现机制。
* 为了支持独立组件，定义了一种需要对 `#VC` 异常进行特定处理的发现机制。
  * 该机制假设，如果在启动时已发现 SVSM，则实现 `#VC` 处理器的 guest 组件也知晓该 SVSM 及其 *调用区域（Calling Area）* 的位置。
1. 检测 SVSM 是否存在
   - 独立组件执行 `CPUID(EAX=8000_001F)` 指令
   - 这会触发 `#VC` 异常
   - `#VC` 处理函数应在响应中设置 `EAX[28]=1`
   - 组件看到此标志位后，即确认 SVSM 存在
* **注意**：此 bit 是专门为 SVSM 发现预留的，硬件或 PSP 生成的 CPUID 数据永远不会设置它。

2. 获取 SVSM 调用区域地址
   - 组件读取 `MSR C001_F000` 寄存器
   - 这会触发 `#VC` 异常
   - `#VC` 处理函数应返回 *SVSM 调用区域* 的 gPA 作为 MSR 值
* **注意**：此 MSR 不支持写入操作，且永远不会由硬件实现。
* 一旦定位到 SVSM 调用区域（Calling Area），该独立组件就可以正常向 SVSM 发起调用。
* 该独立组件不应以 guest 其他部分所不期望的方式来改变 SVSM 的状态（例如，任何组件都不应发出 `SVSM_CORE_REMAP_CA` 命令）。

# 5 调用约定
* 每次对 SVSM 的调用都会通过 **SVSM 调用区域**（其地址最初通过 secrets page 的 `SVSM_CAA` 字段进行配置）和寄存器的组合来传递数据。
  * 使用调用区域对于 SVSM 来说是必要的，这样它才能区分某次调用是由 guest 发起的，还是由行为不当（poorly behaved）的 host 进行的虚假调用（spurious invocation）。
  * 所有其他参数都通过寄存器传递。
* 最初配置的 SVSM 调用区域是一个位于初始 SVSM 内存范围之外的内存页，其 VMPL 权限未以任何方式受到限制。
  * 该地址保证按 `4 KB` 边界对齐，因此如果需要，该页的剩余部分可由 guest 用于基于内存的参数传递。
* 调用区域的内容如下表所述：
* Table 2: Calling Area

字节偏移 | 大小   | 域名称             | 描述
--------|--------|-------------------|------------------------------------------------------------
0x000   | 1 字节 | SVSM_CALL_PENDING | 指示 guest 是否已请求调用（`0` = 未请求调用，`1` = 已请求调用）
0x001   | 1 字节 | SVSM_MEM_AVAILABLE | 有可用的空闲内存可以收回
0x002   | 6 字节 | .                  | 保留。SVSM 不需要验证这些字节是否为 `0`

* 每个调用由一个 `32-bit` 的 **协议号** 和一个该协议特有的 `32-bit` **调用标识符** 来标识。已定义的协议如下：
* Table 3: Protocols

协议号    | 协议描述
---------|---------------
0        | Core
1        | Attestation
2        | vTPM
0x8000_0000 – 0x8000_FFFF | 为 AMD 参考实现特定协议保留

#### 调用发起（Guest 操作）
1. 准备调用参数
* 要调用 SVSM，guest OS 必须在 `RAX` 寄存器中加载调用的标识符，其中
  * `bits [63:32]` 存放 **协议号**
  * `bits [31:0]` 存放该协议内的 **调用标识符**
* 可能还需要用与所发出调用相关的特定值来配置其他寄存器和/或内存。
2. 发起调用请求
* 一旦所有内存和寄存器都准备就绪，guest OS 必须将调用区域（Calling Area）的 `SVSM_CALL_PENDING` 字段写入值 `1`，以表示其已准备好发出调用。
* 最后，guest OS 必须执行 `VMGEXIT` 指令，请求 host 代表调用方 vCPU 执行 SVSM（详见 Guest-Hypervisor Communication Block Standardization）。

#### 调用处理（SVSM 操作）
1. 安全检查
* 当 SVSM 接收到调用时，它应将 guest OS 调用方 vCPU 的 `VMSA.EFER.SVME` 设为 `0`；
  * 这可确保在 SVSM 调用处理进行期间，host 无法尝试重新进入该调用方 vCPU。
  * （任何进入 guest 的尝试都会因 VMSA 内容无效而失败。）
* 随后，SVSM 应检查 `SVSM_CALL_PENDING` 字段，以确定 guest OS 是否确实请求了调用；
  * 如果是 host 非法进入 SVSM，该字段将为 `0`。在这种情况下，SVSM 除了为调用方 vCPU 设置 `VMSA.EFER.SVME=1` 并返回 guest 外，不会执行任何其他操作。
* 此外，SVSM 还应在设置 `VMSA.EFER.SVME=0` 之后检查 `VMSA.EXITCODE`，以确保在继续之前，guest 处于预期的 `VMGEXIT` 指令边界。
  * 如果退出码并非表示因 `VMGEXIT` 而退出，SVSM 应重置 `VMSA.EFER.SVME=1`，并在返回 guest 之前不采取任何进一步的操作。
2. 执行与返回
* 一旦 SVSM 确定调用请求是合法的，它将从请求 vCPU 的 VMSA 中读取 `RAX` 的值，并据此处理该调用。
* 调用完成后，SVSM 会将调用结果写入请求 vCPU 的 VMSA 中的 `RAX` 寄存器。然后，它会清空调用区域（Calling Area）中的 `SVSM_CALL_PENDING` 字段，以表示调用已完成。
* 最后，SVSM 会为调用方 vCPU 设置 `VMSA.EFER.SVME=1`（此操作仅在已设置好 `VMSA.RAX` 和 `SVSM_CALL_PENDING` 字段后进行），并返回 guest。

#### 结果验证（Guest 操作）
* 返回到 guest 后，guest 必须 **以原子方式** 清空调用区域（Calling Area）中的 `SVSM_CALL_PENDING` 字段，并检查其之前的值。
* 由于 guest **无法信任** host 已按预期执行了 SVSM 调用，也 **不能假设** host 不会在不恰当的时机尝试执行 SVSM 调用，因此 guest 必须在提取 `SVSM_CALL_PENDING` 字段的旧值进行检查的 **同时**，清除该挂起的请求。
  * 如果 `SVSM_CALL_PENDING` 的 **原值为 非 0**：这表明 SVSM 从未执行过该调用。Guest 必须选择重试该调用，或者接受 host 未执行其请求的事实。
  * 如果 `SVSM_CALL_PENDING` 的 **原值为 0**：这表明调用已经完成。此时，guest 可以检查 `RAX` 寄存器中的值，以确定该调用是否成功完成。

#### 结果码格式
* 在 `RAX` 中返回的结果值是 `32-bit` 值（`64-bit` 的符号扩展会被忽略），分为三类：
1.  **成功完成**：附带特定的完成信息
2.  **失败完成**：附带明确的失败原因
3.  **内存请求**：请求更多内存
* 大部分结果码是特定于各个协议的，但有一部分结果值空间被预留用于通用结果。
* Table 4: Result Codes

结果码       | 名称        | 含义
------------|-------------|-----------------
0x0000_0000 | SVSM_SUCCESS | The call completed successfully.
0x0000_0000 - 0x0000_0FFF | . | Reserved for future use.
0x0000_1000 - 0x3FFF_FFFF| . | Defined by the protocol that was invoked.
0x4000_0000 - 0x7FFF_FFFF | . | Additional memory is required to complete the requested operation. Bits 29:0 indicate the number of 4 KB pages that are required.
0x8000_0000 | SVSM_ERR_INCOMPLETE | The requested call was partially performed. The guest must request additional processing by setting  `SVSM_CALL_PENDING` and invoking the SVSM again.
0x8000_0001 | SVSM_ERR_UNSUPPORTED_PROTOCOL | The requested protocol is not supported.
0x8000_0002 | SVSM_ERR_UNSUPPORTED_CALL | The requested call ID is not supported by the requested protocol.
0x8000_0003 | SVSM_ERR_INVALID_ADDRESS | A gPA specified as part of a call is invalid.
0x8000_0004 | SVSM_ERR_INVALID_FORMAT | A reserved value was specified in `SVSM_CALL_PENDING`.
0x8000_0005 | SVSM_ERR_INVALID_PARAMETER | One or more invalid parameters were specified to a call.
0x8000_0006 | SVSM_ERR_INVALID_REQUEST | The request cannot be supported by the protocol handler that was invoked.
0x8000_0007 | SVSM_ERR_BUSY | The request cannot be handled at this time, the guest should issue the request again.
0x8000_0008 - 0x8000_0FFF | . | Reserved for future use.
0x8000_1000 - 0xFFFF_FFFF | . | Defined by the protocol that was invoked.

* 任何不符合对齐要求的输入参数，都将返回 `SVSM_ERR_INVALID_PARAMETER` 结果值。
* 任何在访问指定地址时导致错误（fault）的输入地址，都将返回 `SVSM_ERR_INVALID_ADDRESS` 结果值。
* `SVSM_ERR_BUSY` 结果码的返回取决于具体实现，在任何协议调用描述中都未做明确规定。
  * 任何返回此结果码的调用都不得使系统处于中间状态。
  * 任何返回此结果码的调用都应能描述已取得的进展，或者具备幂等性（idempotent）。
* 预期任何返回结果码在 `0x4000_0000` – `0x7FFF_FFFF` 范围内的调用，都不会使系统处于中间状态。
  * 任何返回这些结果码之一的调用都应能描述已取得的进展，或者具备幂等性（idempotent）。

# 6 核心协议
* 所有 SVSM 实现都必须支持核心协议（core protocol），其协议 ID 值为 `0`。
* 核心协议具有版本号，允许随着时间的推移对其进行扩展；该协议的初始版本为版本 1。
* 核心协议的版本化遵循严格的向后兼容原则（严格累加），即版本 1 中存在的所有调用都必须被未来的所有实现所支持。
* 下表列举了核心协议所支持的调用集合：
* Table 5: Core Protocol Services

调用 ID | 第一版本支持 | 名字            | 功能
--------|------------|-----------------|-----------------
0       | 1 | SVSM_CORE_REMAP_CA       | 重映射 SVSM Calling Area 到一个新的 gPA
1       | 1 | SVSM_CORE_PVALIDATE      | 执行 `PVALIDATE`
2       | 1 | SVSM_CORE_CREATE_VCPU    | 创建一个新 vCPU
3       | 1 | SVSM_CORE_DELETE_VCPU    | 删除一个 vCPU
4       | 1 | SVSM_CORE_DEPOSIT_MEM    | 存入额外的内存，以供 SVSM 使用
5       | 1 | SVSM_CORE_WITHDRAW_MEM   | 收回 SVSM 不再需要的未使用内存
6       | 1 | SVSM_CORE_QUERY_PROTOCOL | 查询某个协议的可用性
7       | 1 | SVSM_CORE_CONFIGURE_VTOM | 重新配置 vTOM 的使用

## 6.1 SVSM_CORE_REMAP_CA 调用
* 此调用用于请求将一个新的 gPA 用于此后与 SVSM 的所有通信。它仅影响调用它的 vCPU 的调用区域（Calling Area）。

寄存器 | 大小（字节）| 对齐（字节）| In/Out | 描述
------|------------|------------|--------|-----------------------
`RAX` | 4          | .          | Out    | 结果值
`RCX` | 8          | 4 KB       | In     | 新 Calling Area 的 gPA

* 调用完成后，先前配置的调用区域（Calling Area）的 `SVSM_CALL_PENDING` 字段会被清空，以表示该调用已经完成。
* 此外，如果调用成功，SVSM 还会将 **新** 调用区域的 `SVSM_CALL_PENDING` 字段设置为 `0`，以防止不合作的 host 通过虚假调用，诱使 SVSM 误以为 guest 又发起了新的调用请求。
* Guest 可以通过检查 `RAX` 寄存器的值来判断调用是否成功。如果调用成功，那么先前配置的旧调用区域将不再被 SVSM 访问，guest 可以将其回收用于任何其他目的。

## 6.2 SVSM_CORE_PVALIDATE 调用
* 此调用用于请求 SVSM 代表 guest 执行 `PVALIDATE` 操作。

寄存器 | 大小（字节）| 对齐（字节）| In/Out | 描述
------|------------|------------|--------|-----------------------
`RAX` | 4          | .          | Out    | 结果值
`RCX` | 8          | 8          | In     | 所请求操作列表的 gPA

* 所请求的操作列表是根据下表的格式来指定的。
* Table 6: PVALIDATE Operation

字节偏移 | 大小（字节）| 含义
--------|------------|------------------------------------------------------------------
0x000   | 2 | 列表中的条目数
0x002   | 2 | 待处理列表中下一条条目的索引
0x004   | 4 | 保留
0x008   | 8 | 列表中的第一条条目。每个条目对各 bits 的规定如下：
.       | . | `Bits 1:0`：`PVALIDATE` 操作 `RCX` 的值（`0 = 4KB page`，`1 = 2MB page`）
.       | . | `Bits 2`：`PVALIDATE` 操作 `RDX` 的值（`0 = make invalid`，`1 = make valid`）
.       | . | `Bits 3`：当置为 `1` 时忽略 `EFLAGS.CF` 的警告
.       | . | `Bits 11:4`：保留
.       | . | `Bits 63:12`：gPA 页号。注意：如果条目描述 `2 MB` 页时 bits `[20:12]` 必须是 `0`
0x010   | 8 | 列表中的第二条条目，如果有的话。其他的列表条目随其后

* 列表中的条目数量不得过多，以至于参数列表跨越 `4KB` 边界。
  * 条目数量至少为 `1` 个。
  * 若条目数量不在有效范围内，调用将返回 `SVSM_ERR_INVALID_PARAMETER`（参数无效错误）。
#### 待处理的下一条目索引
* *待处理的下一条目索引* 必须严格小于列表中的条目总数；否则，调用将返回 `SVSM_ERR_INVALID_PARAMETER`（参数无效错误）。
* 调用完成后，*待处理的下一条目索引* 将指示列表中已成功处理的条目数量：
  * 若调用返回 `SVSM_ERR_INCOMPLETE`（处理未完成错误），说明 SVSM 无法通过单次操作处理整个列表，此时 guest 需在 `RAX` 寄存器中重新加载正确的调用代码（调用期间 `RCX` 寄存器保持不变），并重新发起调用；SVSM 将根据 *待处理的下一条目索引* 继续处理剩余条目。
  * 若调用返回其他任何错误，*待处理的下一条目索引* 将指示处理失败的条目所在位置。
  * 若调用成功，*待处理的下一条目索引* 将等于列表中的条目总数。
#### 执行 `PVALIDATE` 指令
* SVSM 需检查 guest 是否试图对 **已分配给 SVSM 自身的 gPA** 执行 `PVALIDATE` 操作。
  * 若 SVSM 检测到 guest 存在此类行为，调用将返回 `SVSM_ERR_INVALID_ADDRESS`
* 若某一条目设置 `bit 2 = 0`（表示请求将页面失效），SVSM 将先执行 `RMPADJUST` 指令，撤销除 VMPL0 外所有 VMPL 的权限。
  * 此操作是必要的 —— 无论 host 在任何时间点是否执行 `RMPUPDATE` 指令，都能确保后续的页验证操作可观测到一致的 VMPL 权限状态。
* 若执行 `PVALIDATE` 指令后，`EFLAGS` 寄存器的 *进位标志位* `EFLAGS.CF = 1`，且触发该 `CF=1` 警告的条目未设置对应的标志位，调用将失败，并返回错误码 `0x8000_1010`。
* 若执行 `PVALIDATE` 指令（或 `RMPADJUST` 指令）后，寄存器的值 `EAX != 0`，调用将失败，并返回 `0x8000_1000` 至 `0x8000_100F` 范围内的错误码，具体值等于 `(0x8000_1000 + EAX)`。
  * 从架构定义上，`PVALIDATE` 指令的 `EAX` 错误码范围为 `0x0000-0x000F`；
  * 若 `PVALIDATE` 意外返回超出该范围的值（例如，未来架构修订中扩展了错误码空间），调用将失败，并返回错误码 `0x8000_1011`。
* 若 `PVALIDATE` 指令执行成功，且某一条目设置 `bit 2 = 1`（表示请求页验证），SVSM 还需执行以下操作：
  * 将该页的内存清零（此举可防止 hypervisor 恶意操作时，VMPL0 的内存泄露给权限更低的 VMPL）；
  * 执行 `RMPADJUST` 指令，为发起请求的 vCPU 所属 VMPL，以及所有权限更高的 VMPL（数值小于或等于请求方 VMPL）授予完全权限。

## 6.3 SVSM_CORE_CREATE_VCPU 调用
* 此调用用于请求创建一个新的 vCPU 上下文。

寄存器 | 大小（字节）| 对齐（字节）| In/Out | 描述
------|------------|------------|--------|-----------------------
`RAX` | 4          | .          | Out    | 结果值
`RCX` | 8          | 4 KB       | In     | 为该 vCPU 创建的 VMSA 的 gPA
`RDX` | 8          | 4 KB       | In     | 与 vCPU 关联的调用区域 的 gPA
`R8`  | 4          | 8          | In     | vCPU 的 APIC ID

* SVSM 会检查所指定的两个 gPA 值，确保其不与任何现有使用场景（包括 SVSM 私有页、任何活跃的 VMSA 地址或任何活跃的调用区域地址）产生冲突。
  * 若检测到冲突，该调用将返回 `SVSM_ERR_INVALID_ADDRESS`。
* SVSM 会撤销除自身外所有 VMPL 对该地址的访问权限，以确保在 VMSA 验证和分配过程中，VMSA 页不会被篡改。
* SVSM 会对新 VMSA 进行如下验证：
  * 若 `VMSA.VMPL == 0`，调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
  * 若 VMSA 的 VMPL 字段（`VMSA.VMPL`）小于发起调用的 vCPU 的 VMPL 字段（`vCPU.VMPL`），调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
  * 若 `VMSA.EFER.SVME != 1`，调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
  * 若 VMSA 的 SEV 功能字段（`VMSA.SEV_FEATURES`）与启动 vCPU 的 VMSA 的 SEV 功能字段（startup vCPU `VMSA.SEV_FEATURES`）不一致，调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
* 若 VMSA 验证通过，SVSM 将执行 `RMPADJUST` 指令，将该页标记为 **VMSA 页**，使其可立即投入使用。
* 同时，SVSM 会缓存 **该 VMSA 的 gPA** 以及 **与目标 vCPU 关联的调用区域的 gPA**，以便在后续对 SVSM 的调用（如寄存器输入、协议请求等）中使用。
* 一旦某页被确立为 VMSA 页，在检测内存使用冲突时，该页将被视为由 SVSM 私有拥有。
  * 任何将 VMSA 页的 gPA 指定为输入 gPA 的调用，都会失败并返回 `SVSM_ERR_INVALID_ADDRESS`。
  * 启动 vCPU 的 VMSA 也遵循此规则（该 VMSA 无需位于分配给 SVSM 的初始连续页范围内，因为 guest 应知晓其自身 VMSA 的位置）。

## 6.4 SVSM_CORE_DELETE_VCPU 调用
* 此调用用于删除一个之前配置的 vCPU

寄存器 | 大小（字节）| 对齐（字节）| In/Out | 描述
------|------------|------------|--------|---------------------
`RAX` | 4          | .          | Out    | 结果值
`RCX` | 8          | 4 KB       | In     | 被删除的 VMSA 的 gPA

* SVSM 会验证指定的 gPA 是否属于已知的 VMSA 地址。
  * 若该 gPA 并非已知的 VMSA 地址，调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
* 启动 vCPU（startup vCPU）**不允许被删除**。
  * 若待删除的 VMSA 与启动 vCPU 相关联，调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
* 若待删除 VMSA 的 VMPL 小于请求方（发起删除调用的 vCPU）的 VMPL，调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
* SVSM 会将 VMSA 的 `EFER` 寄存器的 `SVME` 位（`VMSA.EFER.SVME`）设置为 `0`。
  * 若 SVSM 无法将该位设为 `0`（例如 VMSA 当前正处于执行状态），调用将返回结果值 `0x80001003`（等同于 `FAIL_INUSE` 错误码）。
* SVSM 会执行 `RMPADJUST` 指令，将该 VMSA 页转换为普通页，并确保发起请求的 vCPU 所属 VMPL 及所有权限更高的 VMPL（数值小于或等于请求方 VMPL）均拥有对该页的完全访问权限。
* vCPU 删除操作完成后，此前配置的 VMSA 和调用区域（Calling Area）将不再被 SVSM 访问，guest 可将其回收并用于任何其他用途。
* 若待删除的 VMSA 正是发起请求的 vCPU 自身的 VMSA，则 SVSM 不会向调用方返回结果（因该 vCPU 已被删除，无返回对象）。

## 6.5 SVSM_CORE_DEPOSIT_MEM 调用
* 当 SVSM 执行其工作需要额外内存时，可通过此调用分配额外内存，供 SVSM 独占使用。
* Guest 可通过以下方式知晓是否需要分配额外内存：SVSM 会返回一个 `0x4nnn_nnnn` 范围内的状态码，该状态码会指示所需额外 `4KB` 页的数量（注：`nnn_nnnn` 部分用于承载具体页数量信息）。
* 调用方（发起此调用的主体）负责协调所有 vCPU 的内存存入（deposit）或内存收回（withdraw）调用。

寄存器 | 大小（字节）| 对齐（字节）| In/Out | 描述
------|------------|------------|--------|---------------------
`RAX` | 4          | .          | Out    | 结果值
`RCX` | 8          | 8          | In     | 内存页的列表的 gPA

* 内存页列表需根据下表所示格式进行指定。
* Table 7: Deposit Memory Operation

字节偏移 | 大小（字节）| 含义
--------|------------|------------------------------------------------------------------
0x000   | 2 | 列表中的条目数
0x002   | 2 | 待处理列表中下一条条目的索引
0x004   | 4 | 保留
0x008   | 8 | 列表中的第一条条目。每个条目对各 bits 的规定如下：
.       | . | `Bits 1:0`：该条目描述的内存范围的大小（`0 = 4KB page`，`1 = 2MB page`）
.       | . | `Bits 11:2`：保留
.       | . | `Bits 63:12`：gPA 页号。注意：如果条目描述 `2 MB` 页时 bits `[20:12]` 必须是 `0`
0x010   | 8 | 列表中的第二条条目，如果有的话。其他的列表条目随其后

* 列表中的条目数量不得过多，以至于参数列表跨越 `4KB` 边界。
  * 条目数量至少为 `1` 个。
  * 若条目数量不在有效范围内，调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
* *待处理的下一条目索引* 必须严格小于列表中的条目总数；否则，调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
* 调用完成后，“待处理的下一条目索引”将指示列表中已成功处理的条目数量：
  * 若调用返回 `SVSM_ERR_INCOMPLETE`，说明 SVSM 无法通过单次操作处理整个列表，此时 guest 需在 `RAX` 寄存器中重新加载正确的调用代码（调用期间 `RCX` 寄存器保持不变），并重新发起调用；SVSM 将根据“待处理的下一条目索引”继续处理剩余条目。
  * 若调用返回其他任何错误，“待处理的下一条目索引”将指示处理失败的条目所在位置。
  * 若调用成功，“待处理的下一条目索引”将等于列表中的条目总数。
* 对于列表中的每个条目，SVSM 会验证所描述的内存是否尚未成为 SVSM 的私有内存，且是否与任何已配置为调用区域（Calling Area）的页存在重叠。
  * 若检测到任何重叠，调用将返回 `SVSM_ERR_INVALID_ADDRESS`。
* 对于列表中的每个有效条目，SVSM 会执行 `RMPADJUST` 指令限制 VMPL 权限，确保这些内存页仅能被 VMPL0 访问，从而使这些页成为 SVSM 的私有内存。
* 调用可能返回失败，且“下一条目索引”的值不为 `0`。这表明部分内存已成功存入 SVSM，而另一部分内存的存入操作失败。

## 6.6 SVSM_CORE_WITHDRAW_MEM 调用
* 此调用允许 guest 回收那些已设为 SVSM 私有内存、但 SVSM 不再需要的内存。
* 任何能产生可回收空闲内存的 SVSM 操作，都会在调用区域（Calling Area）中设置 `SVSM_MEM_AVAILABLE` 标志（用于提示 guest 存在可回收内存）。
* 调用方（发起此调用的主体）负责协调所有 vCPU 的内存存入（deposit）或内存收回（withdraw）调用。

寄存器 | 大小（字节）| 对齐（字节）| In/Out | 描述
------|------------|------------|--------|---------------------
`RAX` | 4          | .          | Out    | 结果值
`RCX` | 8          | 8          | In     | 用于存放内存页列表的区域的 gPA

* 内存页列表由 SVSM 填充，并根据下表所示格式进行指定。
* Table 8: Withdraw Memory Operation

字节偏移 | 大小（字节）| 含义
--------|------------|----------------------------------------------------------
0x000   | 2         | 列表中的条目数
0x002   | 6         | 未使用
0x008   | 8 x 条目数 | 不再使用的 `4 KB` 页面的 gPA 值列表

* 返回的条目最大数量受到限制，以确保返回的列表不会跨越 `4KB` 边界。
  * 如果参数页的对齐方式导致没有空间容纳任何条目（即参数页的 gPA 对齐到 `4KB` 边界加 `0xFF8` 字节的位置），调用将返回 `SVSM_ERR_INVALID_PARAMETER`。
  * 如果没有可收回的内存，条目数量将为 `0`。列表末尾之外的未使用条目不会被清零。
* SVSM 必须对所有待收回的内存执行 `RMPADJUST` 指令，以授予发起请求的 vCPU 所属 VMPL 以及所有权限更高的 VMPL（数值小于或等于请求方 VMPL）完全访问权限。
* 调用完成后，SVSM 将不再访问这些页面，guest 可将其重新用于任何用途。
* 启动 vCPU 的调用区域（Calling Area）中的 `SVSM_MEM_AVAILABLE` 标志可能会被更新，以指示是否还有额外内存可收回。
* 此调用绝不会返回 `SVSM_ERR_INCOMPLETE`（处理未完成错误）。
  * 如果 SVSM 无法收回所有可用内存，调用必须以 `SVSM_SUCCESS`（成功）状态完成，且启动 vCPU 的调用区域中的 `SVSM_MEM_AVAILABLE` 标志将指示仍有额外内存可收回。

## 6.7 SVSM_CORE_QUERY_PROTOCOL 调用
* 此调用用于确定特定协议的可用性。
  * 寄存器 `RCX` 的 bits `[63:32]` 包含请求的协议编号。
  * 寄存器 `RCX` 的 bits `[31:0]` 包含请求协议的期望版本。
* 调用完成后，寄存器 `RCX` 会被设置以指示所请求协议的可用性：
  * 如果所请求版本的协议不可用，寄存器 `RCX` 将包含值 `0`。
  * 如果所请求版本的协议可用，寄存器 `RCX` 的 bits `[63:32]` 将表示支持的最高协议版本号，bits `[31:0]` 将表示支持的最低协议版本号。
* 此调用始终返回 `SVSM_SUCCESS`，因为协议的可用性通过 `RCX` 寄存器来告知。
* 查询协议是否存在时，不允许请求额外的 SVSM 内存（该协议的调用可能会请求存入内存）。

## 6.8 SVSM_CORE_CONFIGURE_VTOM 调用
* 此调用用于查询或重新配置调用处理器上 vTOM 的使用。
  * 为了支持 *基于 vTOM 的机密性* 与 *完全依赖页表项 `C-bit` 的机密性判定* 之间的转换，此调用还会更改 guest 的 `CR3`、`RIP` 和 `RSP` 的值，以实现从一种环境到另一种环境的平滑过渡。
* `SVSM_CORE_CONFIGURE_VTOM` 调用有两种形式：查询和配置，由 `RCX` 的第 `0` 位指示（`RCX[0]=1` 表示查询，`RCX[0]=0` 表示配置）。
* 调用的寄存器约定如下表所述。
* Table 9: vTOM Configuration Operation

寄存器         | 内容 | .
--------------|------|--------------------
条目上的 `RCX` | Query | Bit 0：必须是 `1`
.             | .     | Bit 63:1：必须是 `0`
.             | Configure | Bit 0：必须是 `0`
.             | .         | Bit 1：设为 `0` 禁用 vTOM，或设为 `1` 启用 vTOM
.             | .         | Bit 2：若设为 `1`，则在调用成功完成后，将把 `VMSA.CR3` 设置为 `RDX` 中的值
.             | .         | Bit 3：若设为 `1`，则在调用成功完成后，将把 `VMSA.RIP` 设置为 `R8` 中的值
.             | .         | Bit 4：若设为 `1`，则在调用成功完成后，将把 `VMSA.RSP` 设置为 `R9` 中的值
.             | .         | Bit 11:5：必须是 `0`
---           | ---       | Bit 63:12：所需 vTOM 值的 bits `63:12`。若要禁用 vTOM，此部分必须设为 `0`
`RCX` 的结果  | .   | Bit 0：会是 `0`
.            | .   | Bit 1：如果 vTOM 配置是支持的，会是 `1`；否则为 `0`
.            | .   | Bit 11:2：会是 `0`
.            | .   | Bit 19:12：vTOM 对齐要求，以 2 的幂表示（值为 `20` 表示 vTOM 必须对齐到 `1 MB` 边界）
---          | --- | Bit 63:20：`0`
`RCX` 的结果  | .   | 若支持 vTOM 配置，则为最小有效 vTOM 值；否则（若不支持 vTOM 配置），该值未定义
`R8` 的结果   | .   | 若支持 vTOM 配置，则为最大有效 vTOM 值；否则（若不支持 vTOM 配置），该值未定义

* 如果调用成功，vTOM 将按请求被报告或重新配置。
  * 如果正在重新配置 vTOM，则 `CR3` 和 `RSP` 寄存器将按请求更新，执行将在指定的 `RIP` 处继续，且 `RAX` 将包含 `SVSM_SUCCESS` 的值。
* 如果调用失败，VMSA 不会发生任何更改，执行将在 `VMGEXIT` 指令之后的下一条指令处继续，且 `RAX` 将包含相应的错误代码。
  * 如果 `RCX` 中的任何保留位被不当设置，调用可能会失败；这将导致调用返回 `SVSM_ERR_INVALID_PARAMETER`。
* 如果 SVSM 无法支持该请求，可能会拒绝此调用。
  * 例如，如果已配置多个 vCPU，或者请求的配置与其他 vCPU 上的配置不一致，SVSM 可能无法重新配置 vTOM。
    * 这种情况下，调用将返回 `SVSM_ERR_INVALID_REQUEST`。
* 如果 vTOM 的值是宿主环境无法支持的，则调用将返回 `SVSM_ERR_INVALID_ADDRESS`。

# 7 Attestation 协议
* 证明协议（attestation protocol）的协议 ID 值为 `1`。
* 该证明协议支持版本控制，以便后续对其进行扩展；证明协议的初始版本为版本 1。
* 证明协议的版本控制采用严格的 “增量式” 规则，即：版本 1 中包含的所有调用，在未来所有版本的实现中都必须得到支持。
* 下表列出了该证明协议所支持的调用集合：
* Table 10: Attestation Protocol Services

调用 ID | 第一版本支持 | 名字            | 功能
--------|------------|-----------------|-----------------
0       | 1 | SVSM_ATTEST_SERVICES       | 获取所有 SVSM 服务（例如 vTPM 等）的证明报告
1       | 1 | SVSM_ATTEST_SINGLE_SERVICE | 获取一个单个 SVSM 服务的证明报告

## 7.1 SVSM_ATTEST_SERVICES 调用
## 7.2 SVSM_ATTEST_SINGLE_SERVICE 调用

# 8 vTPM Protocol
* vTPM 协议的协议 ID 值为 `2`。该 vTPM 协议支持版本控制，以便后续对其进行扩展；vTPM 协议的初始版本为版本 1。
* vTPM 协议的版本控制采用严格的 “增量式” 规则，即：版本 1 中包含的所有调用，在未来所有版本的实现中都必须得到支持。
* vTPM 协议遵循 [Official TPM 2.0 Reference Implementation (by Microsoft)](https://github.com/microsoft/ms-tpm-20-ref) 类似的协议
* 下表列出了该 vTPM 协议所支持的调用集合：
* Table 14: Attestation Protocol Services

调用 ID | 第一版本支持 | 名字            | 功能
--------|------------|-----------------|-----------------
0       | 1 | SVSM_VTPM_QUERY | 查询 vTPM 的命令及功能支持情况
1       | 1 | SVSM_VTPM_CMD   | 执行一条 TPM 命令

## 8.1 SVSM_VTPM_QUERY 调用
## 8.2 SVSM_VTPM_CMD 调用
## 8.3 Service Attestation 数据