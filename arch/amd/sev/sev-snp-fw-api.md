# SEV-SNP 固件 API

# 4 Guest 管理

## 4.5 启动一个 Guest

`SNP_LAUNCH_UPDATE` 可以向 guest 内存中插入两个特殊页面：秘密页面（secrets page）和 CPUID 页面。
* **秘密页面（secrets page）** 包含 guest 与固件交互时使用的加密密钥。
  * 由于 secrets page 是用 guest 的内存加密密钥加密的，因此 hypervisor 无法读取这些密钥。
* **CPUID 页面** 包含由 hypervisor 提供的、将传递给 guest 的 CPUID 功能值。
  * 固件会对这些值进行验证，以确保 hypervisor 提供的值在有效范围内。

# 7 Guest Messages

* **Guest messages** 为 guest 提供了一种与 PSP（平台安全处理器）通信的机制，可防范恶意 hypervisor 对发送的消息进行读取、篡改、丢弃或重放等风险。
* Guest 可通过 `SNP_GUEST_REQUEST` 命令向固件发出请求。
  * 该命令会在 guest 与 PSP 固件之间建立一条可信通道。
  * Hypervisor 无法在不被察觉的情况下篡改消息，也无法读取消息的明文内容。
* 固件使用 **虚拟机平台通信密钥（VMPCK，Virtual Machine Platform Communication key）** 构建该通道。
  * 每个 guest 拥有 4 个 VMPCK，这些密钥由固件生成，并在 guest 启动过程中作为特殊 *秘密页面* 的一部分提供给 guest（详见 8.17 节中的 SNP_LAUNCH_UPDATE）。
  * 只有 guest 和固件持有这些 VMPCK。
* 每条消息都包含一个 **基于 VMPCK 的序列号**。
  * 每发送一条消息，序列号就会递增。
  * Guest 发送给固件的消息以及固件发送给 guest 的消息必须按顺序传递。
  * 如果顺序错乱，当固件检测到序列号不同步时，会拒绝 guest 后续发送的消息。
* 每条消息都通过 **带关联数据的认证加密算法（AEAD，Authenticated Encryption with Associated Data）** 进行保护，具体采用的是 AES-256 GCM 算法。
* 关于如何通过 `SNP_GUEST_REQUEST` 命令发送消息的详细说明，可参见 8.26 节。
* **译注**：VMPCK 被用于：
  * 证明报告（Attestation reports）
  * 密钥派生（Key Derivation）
  * Export/Import – 对于 live migration