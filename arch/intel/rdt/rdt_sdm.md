
## 4.6 Intel RDT 特性的非对称枚举

* RDT 的枚举功能已扩展到枚举非对称资源的平台。例如，混合平台（支持多种处理器类型）可能包含多个具有不同功能的最后一级缓存（LLC），或者某些处理器可能不包含同一枚举级别的 LLC
* 软件应考虑对 RDT 枚举进行以下更改，以便在所有平台上启用完整的 RDT 功能
* 非对称 RDT CPUID leaves `0x27` 和 `0x28` 可以枚举平台上不同逻辑处理器之间的不同功能，而 CPUID RDT leaves `0x0F` 和 `0x10` 可以枚举通用功能集，以确保软件兼容性
* 应按逻辑处理器枚举非对称 RDT CPUID leaves，以识别功能差异
* RDT 技术的枚举首先包括表 4-2 中的以下四个 CPUID leaves `0x07` 位和表 4-3 中的相应 CPUID leaves
* 非对称 RDT-M leaves `0x27` 及其 sub-leaves 的定义与 RDT-M leaves `0x0F` 相同
* 非对称 RDT-A leaves `0x28` 及其 sub-leaves 的定义与 RDT-A leaves `0x10` 相同
* 对称平台可能会枚举非对称 leaves 的存在，因此不能以此判断系统是否非对称
* 非对称 leaves `0x27` 和 `0x28` 中提供的数据是 leaves `0x0F` 和 `0x10` 的超集
* 通过这些 leaves 返回的枚举值和功能可以是以下任意一种：
1. 枚举 leaves `0x27` 或 `0x28` 并不表示支持 leaves `0x0F` 或 `0x10`
2. Leaves `0x0F` 和/或 `0x10` 返回的子集可能小于平台上存在 leaves `0x27` 或 `0x28` 的资源特征的交集
3. Leaves `0x0F` 和/或 `0x10` 返回平台上存在 leaves `0x27` 或 `0x28` 的资源特征的交集
4. 存在 leaves `0x27` 或 `0x28` 的平台上，Leaves `0x0F` 和/或 `0x10` 可以被枚举但包含 *无信息（no information）*
* Leaves `0x0F` 和 `0x10` 中提供的枚举不需要在 *存在 leaves `0x27` 或 `0x28` 的平台上* 提供资源特征信息
* 软件应使用以下算法来检测对 Intel® RDT 监控的支持：
```vb
IF CPUID.(EAX=07H,ECX=01H):ECX.Asymmetrical RDT-M[Bit 0] = 1 THEN
   // Perform asymmetrical enumeration using leaf 27H
ELSE IF CPUID.(EAX=07H,ECX=01H):EBX.RDT-M[Bit 12] = 1 THEN
        // Perform symmetrical enumeration using leaf 0FH
   ELSE
        // Intel® Resource Directory Technology Monitoring is not supported.
   ENDIF
ENDIF
```
* 软件应使用以下算法来检测对 Intel® RDT 分配的支持：
```vb
IF CPUID.(EAX=07H,ECX=01H):ECX.Asymmetrical RDT-A[Bit 1] = 1 THEN
   // Perform asymmetrical enumeration using leaf 28H
ELSE IF CPUID.(EAX=07H,ECX=01H):EBX.RDT-A[Bit 15] = 1 THEN
       // Perform symmetrical enumeration using leaf 10H
   ELSE
       // Intel® Resource Directory Technology Allocation is not supported.
   ENDIF
ENDIF
```
* 需要在每个逻辑处理器上读取非对称 leaves。软件可以确定所有逻辑处理器的功能交集，或者将具有匹配功能集的逻辑处理器分组
* 共享同一资源的逻辑处理器将报告对该资源的相同功能支持
  * 例如，枚举共享 L3 cache 的每组逻辑处理器，读取共享每个 L3 cache 的逻辑处理器上 L3 CAT 的 CPUID leaf `0x28`，并将这些设置应用于共享同一 L3 cache 的所有逻辑处理器