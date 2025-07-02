# [GIT PULL] KVM: x86: LAM support for 6.8
* KVM 对 LAM 的支持主要是模拟 LAM 修改后的规范性检查。
  * KVM 采取的方法是使用翻译地址的最高位来“填充”元数据位，例如 对于 LAM-48，bit `47` 符号扩展为 bit `62:48`。
  * 最重要的 bit `63` 不会被修改，即来自原始、未标记虚拟地址的值将被保留用于规范性检查。
* 除了模拟 LAM 的规范检查行为之外，LAM 还具有用于可选功能的常用 KVM 接触点：枚举（`CPUID.7.1:EAX.LAM[bit 26]`、通过 `CR3` 和 `CR4` bit 来启用 LAM 等。

## [PATCH 01/12] KVM: x86: Consolidate flags for __linearize()
* 将 `__linearize()` 的 `@write` 和 `@fetch` 合并到一组 `flags` 中，以便可以添加额外的标志，而无需更多/新的布尔参数，以精确识别访问类型。
* 无意改变功能。
---
* 引入模拟操作是写入 `X86EMUL_F_WRITE` 还是取址 `X86EMUL_F_FETCH` 操作的两个用于设置 `flags` 的 bits
```diff
diff --git a/arch/x86/kvm/kvm_emulate.h b/arch/x86/kvm/kvm_emulate.h
index be7aeb9b8ea3..e24c8ac7b930 100644
--- a/arch/x86/kvm/kvm_emulate.h
+++ b/arch/x86/kvm/kvm_emulate.h
@@ -88,6 +88,10 @@ struct x86_instruction_info {
 #define X86EMUL_IO_NEEDED       5 /* IO is needed to complete emulation */
 #define X86EMUL_INTERCEPTED     6 /* Intercepted by nested VMCB/VMCS */

+/* x86-specific emulation flags */
+#define X86EMUL_F_WRITE            BIT(0)
+#define X86EMUL_F_FETCH            BIT(1)
+
```
* `__linearize()` 模拟了将用于 *段寻址* 的 *逻辑地址* 转换为 *线性地址* 的操作，称之为 *线性化*
* 修改了 `__linearize()` 函数的接口，将原来的参数 `@write` 和 `@fetch` 合并到一组 `flags` 中
  * 调用它的函数都进行相应的修改
```diff
diff --git a/arch/x86/kvm/emulate.c b/arch/x86/kvm/emulate.c
index 2673cd5c46cb..87ee1802166a 100644
--- a/arch/x86/kvm/emulate.c
+++ b/arch/x86/kvm/emulate.c
@@ -687,8 +687,8 @@ static unsigned insn_alignment(struct x86_emulate_ctxt *ctxt, unsigned size)
 static __always_inline int __linearize(struct x86_emulate_ctxt *ctxt,
                       struct segmented_address addr,
                       unsigned *max_size, unsigned size,
-                      bool write, bool fetch,
-                      enum x86emul_mode mode, ulong *linear)
+                      enum x86emul_mode mode, ulong *linear,
+                      unsigned int flags)
 {
    struct desc_struct desc;
    bool usable;
@@ -717,11 +717,11 @@ static __always_inline int __linearize(struct x86_emulate_ctxt *ctxt,
        if (!usable)
            goto bad;
        /* code segment in protected mode or read-only data segment */
-       if ((((ctxt->mode != X86EMUL_MODE_REAL) && (desc.type & 8))
-                   || !(desc.type & 2)) && write)
+       if ((((ctxt->mode != X86EMUL_MODE_REAL) && (desc.type & 8)) || !(desc.type & 2)) &&
+           (flags & X86EMUL_F_WRITE))
            goto bad;
        /* unreadable code segment */
-       if (!fetch && (desc.type & 8) && !(desc.type & 2))
+       if (!(flags & X86EMUL_F_FETCH) && (desc.type & 8) && !(desc.type & 2))
            goto bad;
        lim = desc_limit_scaled(&desc);
        if (!(desc.type & 8) && (desc.type & 4)) {
@@ -757,8 +757,8 @@ static int linearize(struct x86_emulate_ctxt *ctxt,
             ulong *linear)
 {
    unsigned max_size;
-   return __linearize(ctxt, addr, &max_size, size, write, false,
-              ctxt->mode, linear);
+   return __linearize(ctxt, addr, &max_size, size, ctxt->mode, linear,
+              write ? X86EMUL_F_WRITE : 0);
 }

 static inline int assign_eip(struct x86_emulate_ctxt *ctxt, ulong dst)
@@ -771,7 +771,8 @@ static inline int assign_eip(struct x86_emulate_ctxt *ctxt, ulong dst)

    if (ctxt->op_bytes != sizeof(unsigned long))
        addr.ea = dst & ((1UL << (ctxt->op_bytes << 3)) - 1);
-   rc = __linearize(ctxt, addr, &max_size, 1, false, true, ctxt->mode, &linear);
+   rc = __linearize(ctxt, addr, &max_size, 1, ctxt->mode, &linear,
+            X86EMUL_F_FETCH);
    if (rc == X86EMUL_CONTINUE)
        ctxt->_eip = addr.ea;
    return rc;
@@ -907,8 +908,8 @@ static int __do_insn_fetch_bytes(struct x86_emulate_ctxt *ctxt, int op_size)
     * boundary check itself.  Instead, we use max_size to check
     * against op_size.
     */
-   rc = __linearize(ctxt, addr, &max_size, 0, false, true, ctxt->mode,
-            &linear);
+   rc = __linearize(ctxt, addr, &max_size, 0, ctxt->mode, &linear,
+            X86EMUL_F_FETCH);
    if (unlikely(rc != X86EMUL_CONTINUE))
        return rc;
```

## [PATCH 02/12] KVM: x86: Add an emulation flag for implicit system access
* 添加模拟标志 `X86EMUL_F_IMPLICIT` 以识别指令模拟中的隐式系统访问。
* 此时不必费心连接任何用法，因为 Linear Address Space Separation（LASS）将是该标志的第一个“真正”使用者，并且 LASS 支持将需要专用 hooks，即没有任何现有调用传递的 `X86EMUL_F_IMPLICIT` 是有意义的。
* 即使没有立即使用，也添加 `IMPLICIT` 标志，以便 Linear Address Masking（LAM）支持可以引用该标志来记录隐式访问的地址是未标记（untagged）。
---
```diff
diff --git a/arch/x86/kvm/kvm_emulate.h b/arch/x86/kvm/kvm_emulate.h
index e24c8ac7b930..65fc7ef5ca3d 100644
--- a/arch/x86/kvm/kvm_emulate.h
+++ b/arch/x86/kvm/kvm_emulate.h
@@ -91,6 +91,7 @@ struct x86_instruction_info {
 /* x86-specific emulation flags */
 #define X86EMUL_F_WRITE            BIT(0)
 #define X86EMUL_F_FETCH            BIT(1)
+#define X86EMUL_F_IMPLICIT     BIT(2)
```

## [PATCH 03/12] KVM: x86: Add X86EMUL_F_INVLPG and pass it in em_invlpg()
* 添加模拟标志 `X86EMUL_F_INVLPG`，用于标识在没有真正内存访问的情况下进行 TLB invalidation 的指令。
* 只有模拟器中实现的 `invlpg` & `invlpga` 属于此类。
  * `invlpga` 不需要额外的信息来进行模拟。只需将标志传递给 `em_invlpg()` 即可。
* Linear Address Masking（LAM）和 Linear Address Space Separation（LASS）不适用于作为 TLB invalidation 输入的地址。
* 该标志将用于支持 LAM/LASS 虚拟化。
---
* 新增模拟标志 `X86EMUL_F_INVLPG`
```diff
diff --git a/arch/x86/kvm/kvm_emulate.h b/arch/x86/kvm/kvm_emulate.h
index 65fc7ef5ca3d..8bd9b23543cc 100644
--- a/arch/x86/kvm/kvm_emulate.h
+++ b/arch/x86/kvm/kvm_emulate.h
@@ -92,6 +92,7 @@ struct x86_instruction_info {
 #define X86EMUL_F_WRITE            BIT(0)
 #define X86EMUL_F_FETCH            BIT(1)
 #define X86EMUL_F_IMPLICIT     BIT(2)
+#define X86EMUL_F_INVLPG       BIT(3)
```
* `invlpga` 不需要额外的信息来进行模拟。只需将标志传递给 `em_invlpg()` 即可。
* 如果逻辑地址线性化成功，`__linearize()` 返回 `X86EMUL_CONTINUE`。
  * 然而对于 `X86EMUL_F_INVLPG`，将来我们可以看到 `vmx_get_untagged_addr()` 直接返回的是 GVA，并未 untagged，因为 LAM 仅针对数据访问。这种情况 `__linearize()` 并不需要因为 LAM 做过多的特殊处理
  * 但用 `linearize()` 是不适合的，因为它会将 `@flags = write ? X86EMUL_F_WRITE : 0`；
    * 如果按修改前 `@write = false` 会导致 `@flags = 0`，对此 flag，`vmx_get_untagged_addr()` 会模拟 untagged 的过程，这是不符合 SDM 的
    * 改用 `__linearize()` 并传入 `X86EMUL_F_INVLPG` 标志，不做 untagged 处理就对了
```diff
diff --git a/arch/x86/kvm/emulate.c b/arch/x86/kvm/emulate.c
index 87ee1802166a..ceec8c5f9687 100644
--- a/arch/x86/kvm/emulate.c
+++ b/arch/x86/kvm/emulate.c
@@ -3440,8 +3440,10 @@ static int em_invlpg(struct x86_emulate_ctxt *ctxt)
 {
    int rc;
    ulong linear;
+   unsigned int max_size;

-   rc = linearize(ctxt, ctxt->src.addr.mem, 1, false, &linear);
+   rc = __linearize(ctxt, ctxt->src.addr.mem, &max_size, 1, ctxt->mode,
+            &linear, X86EMUL_F_INVLPG);
    if (rc == X86EMUL_CONTINUE)
        ctxt->ops->invlpg(ctxt, linear);
    /* Disable writeback. */
```

## [PATCH 04/12] KVM: x86/mmu: Drop non-PA bits when getting GFN for guest's PGD
* 当使用 guest `MAXPHYADDR` 的最大理论掩码获取 guest PGD 的 GFN 时，丢弃非 PA 的比特位。
```diff
diff --git a/arch/x86/kvm/mmu/mmu.c b/arch/x86/kvm/mmu/mmu.c
index 59b026b6ad2a..73070650b143 100644
--- a/arch/x86/kvm/mmu/mmu.c
+++ b/arch/x86/kvm/mmu/mmu.c
@@ -3806,7 +3806,7 @@ static int mmu_alloc_shadow_roots(struct kvm_vcpu *vcpu)
    hpa_t root;

    root_pgd = kvm_mmu_get_guest_pgd(vcpu, mmu);
-   root_gfn = root_pgd >> PAGE_SHIFT;
+   root_gfn = (root_pgd & __PT_BASE_ADDR_MASK) >> PAGE_SHIFT;

    if (!kvm_vcpu_is_visible_gfn(vcpu, root_gfn)) {
        mmu->root.hpa = kvm_mmu_get_dummy_root();
```
* 无条件地执行此操作，因为它对于 32 位 guest 无害，而查询 64 位模式会更昂贵；并且对于 EPT，掩码不与 guest 模式绑定。
* 使用 `PT_BASE_ADDR_MASK` 在技术上是错误的（PAE 分页具有 64 位元素，*除了* `CR3`，它只有 32 个有效位），但在实践中这并不重要。
```diff
diff --git a/arch/x86/kvm/mmu/paging_tmpl.h b/arch/x86/kvm/mmu/paging_tmpl.h
index c85255073f67..4d4e98fe4f35 100644
--- a/arch/x86/kvm/mmu/paging_tmpl.h
+++ b/arch/x86/kvm/mmu/paging_tmpl.h
@@ -62,7 +62,7 @@
 #endif

 /* Common logic, but per-type values.  These also need to be undefined. */
-#define PT_BASE_ADDR_MASK  ((pt_element_t)(((1ULL << 52) - 1) & ~(u64)(PAGE_SIZE-1)))
+#define PT_BASE_ADDR_MASK  ((pt_element_t)__PT_BASE_ADDR_MASK)
 #define PT_LVL_ADDR_MASK(lvl)  __PT_LVL_ADDR_MASK(PT_BASE_ADDR_MASK, lvl, PT_LEVEL_BITS)
 #define PT_LVL_OFFSET_MASK(lvl)    __PT_LVL_OFFSET_MASK(PT_BASE_ADDR_MASK, lvl, PT_LEVEL_BITS)
 #define PT_INDEX(addr, lvl)    __PT_INDEX(addr, lvl, PT_LEVEL_BITS)
```
* *批注*：修改前的操作也是想构造一个 `52~12` 位为 `1` 的掩码，现在改用 `GENMASK_ULL(51, 12)` 直接生成了
* 有机会性就使用 `GENMASK_ULL()` 来定义 `__PT_BASE_ADDR_MASK`。
```diff
diff --git a/arch/x86/kvm/mmu/mmu_internal.h b/arch/x86/kvm/mmu/mmu_internal.h
index b66a7d47e0e4..0669a8a668ca 100644
--- a/arch/x86/kvm/mmu/mmu_internal.h
+++ b/arch/x86/kvm/mmu/mmu_internal.h
@@ -13,6 +13,7 @@
 #endif

 /* Page table builder macros common to shadow (host) PTEs and guest PTEs. */
+#define __PT_BASE_ADDR_MASK GENMASK_ULL(51, 12)
 #define __PT_LEVEL_SHIFT(level, bits_per_level)    \
    (PAGE_SHIFT + ((level) - 1) * (bits_per_level))
 #define __PT_INDEX(address, level, bits_per_level) \
```

## [PATCH 05/12] KVM: x86: Add & use kvm_vcpu_is_legal_cr3() to check CR3's legality
* 添加并使用 `kvm_vcpu_is_legal_cr3()` 检查 `CR3` 的合法性，以明确区分 `CR3` 和 `GPA` 的检查。
* 这将允许从 `kvm_vcpu_is_legal_cr3()` 中免除（exempting）位，而不影响一般 GPA 检查，例如 对于即将推出的功能，将使用 `CR3` 中的高位来启用功能。
* 无意进行功能改变。
---
* 引入新函数 `kvm_vcpu_is_legal_cr3()` 的最初形态，现在还无法用于区分 `CR3` 和 `GPA` 的检查
```cpp
static inline bool kvm_vcpu_is_legal_cr3(struct kvm_vcpu *vcpu, unsigned long cr3)
{
    return kvm_vcpu_is_legal_gpa(vcpu, cr3);
}
```
* `kvm_vcpu_is_legal_gpa()` 的实现如下，GPA 如果没有用到 vCPU 的 arch 规定的 GPA 的保留位则认为 GPA 合法
```cpp
static inline bool kvm_vcpu_is_legal_gpa(struct kvm_vcpu *vcpu, gpa_t gpa)
{
    return !(gpa & vcpu->arch.reserved_gpa_bits);
}
```
* 本 patch 的剩余部分都是如下类似的改动，即把新定义的 `kvm_vcpu_is_legal_cr3()` 在语义合适的地方用上，不贴了
```diff
diff --git a/arch/x86/kvm/svm/nested.c b/arch/x86/kvm/svm/nested.c
index 3fea8c47679e..90ca9489aab6 100644
--- a/arch/x86/kvm/svm/nested.c
+++ b/arch/x86/kvm/svm/nested.c
@@ -311,7 +311,7 @@ static bool __nested_vmcb_check_save(struct kvm_vcpu *vcpu,
    if ((save->efer & EFER_LME) && (save->cr0 & X86_CR0_PG)) {
        if (CC(!(save->cr4 & X86_CR4_PAE)) ||
            CC(!(save->cr0 & X86_CR0_PE)) ||
-           CC(kvm_vcpu_is_illegal_gpa(vcpu, save->cr3)))
+           CC(!kvm_vcpu_is_legal_cr3(vcpu, save->cr3)))
            return false;
    }
```

## [PATCH 06/12] KVM: x86: Remove kvm_vcpu_is_illegal_gpa()

* 删除 `kvm_vcpu_is_illegal_gpa()` 并使用 `!kvm_vcpu_is_legal_gpa()` 代替。
* “illegal” helper 实际上早于 “legal” helper，commit 4bda0e97868a（"KVM: x86:Add a helper to check for a legal GPA"）未删除“illegal”变体的唯一原因是避免代码流失（churn）。
* 既然 `CR3` 有了专门的 helper（判断 `CR3` 里的 GPA 是否合法），调用者就少了，因此代码改动并没有那么大的威慑力（deterrent）。
* 无意进行功能改变。
---
* 代码改动为删除 `kvm_vcpu_is_illegal_gpa()` 及所有对其的引用
```diff
diff --git a/arch/x86/kvm/cpuid.h b/arch/x86/kvm/cpuid.h
index 5fd3f2d1ca14..fa04a00090a2 100644
--- a/arch/x86/kvm/cpuid.h
+++ b/arch/x86/kvm/cpuid.h
@@ -47,11 +47,6 @@ static inline bool kvm_vcpu_is_legal_gpa(struct kvm_vcpu *vcpu, gpa_t gpa)
    return !(gpa & vcpu->arch.reserved_gpa_bits);
 }

-static inline bool kvm_vcpu_is_illegal_gpa(struct kvm_vcpu *vcpu, gpa_t gpa)
-{
-   return !kvm_vcpu_is_legal_gpa(vcpu, gpa);
-}
-
```
## [PATCH 07/12] KVM: x86: Introduce get_untagged_addr() in kvm_x86_ops and call it in emulator
* 向 `kvm_x86_ops` 引入新接口 `get_untagged_addr()` 以从线性地址 untag 元数据的标记。
  * 调用 64-bit 模式指令模拟器的线性化接口。
* 当启用 Intel Linear Address Masking（LAM）或 AMD Upper Address Ignore（UAI）等功能时，线性地址可能会被 tagged 一些在规范性检查之前需要删除的元数据，即忽略元数据。
* 将 `get_untagged_addr()` 引入 `kvm_x86_ops` 以隐藏供应商特定代码，遗憾的是 LAM 和 UAI 具有不同的语义。
* 传递模拟器标志，允许供应商特定的实现，以精确识别访问类型（LAM 不会 untag 某些访问）。
---
* 新增 `struct x86_emulate_ops` 的 `get_untagged_addr` 域，是一个函数指针
```diff
diff --git a/arch/x86/kvm/kvm_emulate.h b/arch/x86/kvm/kvm_emulate.h
index 8bd9b23543cc..e6d149825169 100644
--- a/arch/x86/kvm/kvm_emulate.h
+++ b/arch/x86/kvm/kvm_emulate.h
@@ -230,6 +230,9 @@ struct x86_emulate_ops {
    int (*leave_smm)(struct x86_emulate_ctxt *ctxt);
    void (*triple_fault)(struct x86_emulate_ctxt *ctxt);
    int (*set_xcr)(struct x86_emulate_ctxt *ctxt, u32 index, u64 xcr);
+
+   gva_t (*get_untagged_addr)(struct x86_emulate_ctxt *ctxt, gva_t addr,
+                  unsigned int flags);
 };

 /* Type, address-of, and value of an instruction's operand. */
```
* 函数指针 `get_untagged_addr` 指向函数 `emulator_get_untagged_addr()`，它调用 KVM x86 operations 特定的 `get_untagged_addr` 实现
```diff
diff --git a/arch/x86/kvm/x86.c b/arch/x86/kvm/x86.c
index 3a648b6e632d..557872c0331a 100644
--- a/arch/x86/kvm/x86.c
+++ b/arch/x86/kvm/x86.c
@@ -8445,6 +8445,15 @@ static void emulator_vm_bugged(struct x86_emulate_ctxt *ctxt)
        kvm_vm_bugged(kvm);
 }

+static gva_t emulator_get_untagged_addr(struct x86_emulate_ctxt *ctxt,
+                   gva_t addr, unsigned int flags)
+{
+   if (!kvm_x86_ops.get_untagged_addr)
+       return addr;
+
+   return static_call(kvm_x86_get_untagged_addr)(emul_to_vcpu(ctxt), addr, flags);
+}
+
 static const struct x86_emulate_ops emulate_ops = {
    .vm_bugged           = emulator_vm_bugged,
    .read_gpr            = emulator_read_gpr,
@@ -8489,6 +8498,7 @@ static const struct x86_emulate_ops emulate_ops = {
    .leave_smm           = emulator_leave_smm,
    .triple_fault        = emulator_triple_fault,
    .set_xcr             = emulator_set_xcr,
+   .get_untagged_addr   = emulator_get_untagged_addr,
 };

 static void toggle_interruptibility(struct kvm_vcpu *vcpu, u32 mask)
```
* 新增 KVM x86 operations 的 `get_untagged_addr()` 声明
```diff
diff --git a/arch/x86/include/asm/kvm-x86-ops.h b/arch/x86/include/asm/kvm-x86-ops.h
index 26b628d84594..756791665117 100644
--- a/arch/x86/include/asm/kvm-x86-ops.h
+++ b/arch/x86/include/asm/kvm-x86-ops.h
@@ -135,6 +135,7 @@ KVM_X86_OP(msr_filter_changed)
 KVM_X86_OP(complete_emulated_msr)
 KVM_X86_OP(vcpu_deliver_sipi_vector)
 KVM_X86_OP_OPTIONAL_RET0(vcpu_get_apicv_inhibit_reasons);
+KVM_X86_OP_OPTIONAL(get_untagged_addr)

 #undef KVM_X86_OP
 #undef KVM_X86_OP_OPTIONAL
diff --git a/arch/x86/include/asm/kvm_host.h b/arch/x86/include/asm/kvm_host.h
index a565a2e70f30..8d042f27d644 100644
--- a/arch/x86/include/asm/kvm_host.h
+++ b/arch/x86/include/asm/kvm_host.h
@@ -1762,6 +1762,8 @@ struct kvm_x86_ops {
     * Returns vCPU specific APICv inhibit reasons
     */
    unsigned long (*vcpu_get_apicv_inhibit_reasons)(struct kvm_vcpu *vcpu);
+
+   gva_t (*get_untagged_addr)(struct kvm_vcpu *vcpu, gva_t gva, unsigned int flags);
 };
```
* 在逻辑地址线性化的时候，调用模拟器 operation 的 `get_untagged_addr` 实现，最终调用到 KVM x86 的 `get_untagged_addr` 实现
```diff
diff --git a/arch/x86/kvm/emulate.c b/arch/x86/kvm/emulate.c
index ceec8c5f9687..e223043ef5b2 100644
--- a/arch/x86/kvm/emulate.c
+++ b/arch/x86/kvm/emulate.c
@@ -701,7 +701,7 @@ static __always_inline int __linearize(struct x86_emulate_ctxt *ctxt,
    *max_size = 0;
    switch (mode) {
    case X86EMUL_MODE_PROT64:
-       *linear = la;
+       *linear = la = ctxt->ops->get_untagged_addr(ctxt, la, flags);
        va_bits = ctxt_virt_addr_bits(ctxt);
        if (!__is_canonical_address(la, va_bits))
            goto bad;
```

## [PATCH 08/12] KVM: x86: Untag addresses for LAM emulation where applicable
* 在 `vmx_get_untagged_addr()` 中打桩，并连接来自模拟器的调用（通过 `get_untagged_addr()`）以及来自 VMX 中各种 VM-Exit handler 的“直接”调用，这些地方应该应用 LAM untagging。
* 将 `vmx_get_untagged_addr()` 的核心内容推迟到未来的补丁中实现纯粹是为了使更改更易于使用。
* LAM 仅对 64 位线性地址有效，并且几种类型的访问被豁免。
- 需要 untag 地址标记的情况（在 `get_vmx_mem_address()` 中处理）
  * VMX 指令和 `INVPCID` 的操作数。
  * SGX ENCLS 的操作数。
- LAM 不适用的情况（无需更改）
  * `INVLPG` 的操作数。
  * `INVPCID` 描述符中的线性地址。
  * `INVVPID` 描述符中的线性地址。
  * `ECREATE` 的 SECS 中指定的 `BASEADDR`。
* **注意**：
  - LAM 不适用于写入控制寄存器或 MSR
  - 需在遍历页表之前应用 LAM 掩码，即 `CR2` 中的 faulting 线性地址不包含元数据。
  - VMCS 中保存的 guest 线性地址不包含元数据。
---
### 新增 VMX 的 `get_untagged_addr` 操作的实现 `vmx_get_untagged_addr()`
* 添加 VMX 的 `get_untagged_addr` 操作的实现 `vmx_get_untagged_addr()` 的声明和定义（该 patch 只打桩）
```diff
diff --git a/arch/x86/kvm/vmx/vmx.h b/arch/x86/kvm/vmx/vmx.h
index c2130d2c8e24..45cee1a8bc0a 100644
--- a/arch/x86/kvm/vmx/vmx.h
+++ b/arch/x86/kvm/vmx/vmx.h
@@ -420,6 +420,8 @@ void vmx_enable_intercept_for_msr(struct kvm_vcpu *vcpu, u32 msr, int type);
 u64 vmx_get_l2_tsc_offset(struct kvm_vcpu *vcpu);
 u64 vmx_get_l2_tsc_multiplier(struct kvm_vcpu *vcpu);

+gva_t vmx_get_untagged_addr(struct kvm_vcpu *vcpu, gva_t gva, unsigned int flags);
+
 static inline void vmx_set_intercept_for_msr(struct kvm_vcpu *vcpu, u32 msr,
                         int type, bool value)
 {
diff --git a/arch/x86/kvm/vmx/vmx.c b/arch/x86/kvm/vmx/vmx.c
index 69072e418c73..d7ff06d77720 100644
--- a/arch/x86/kvm/vmx/vmx.c
+++ b/arch/x86/kvm/vmx/vmx.c
@@ -8205,6 +8205,11 @@ static void vmx_vm_destroy(struct kvm *kvm)
    free_pages((unsigned long)kvm_vmx->pid_table, vmx_get_pid_table_order(kvm));
 }

+gva_t vmx_get_untagged_addr(struct kvm_vcpu *vcpu, gva_t gva, unsigned int flags)
+{
+   return gva;
+}
+
 static struct kvm_x86_ops vmx_x86_ops __initdata = {
    .name = KBUILD_MODNAME,

@@ -8345,6 +8350,8 @@ static struct kvm_x86_ops vmx_x86_ops __initdata = {
    .complete_emulated_msr = kvm_complete_insn_gp,

    .vcpu_deliver_sipi_vector = kvm_vcpu_deliver_sipi_vector,
+
+   .get_untagged_addr = vmx_get_untagged_addr,
 };
```
### 对 `invpcid` 的模拟
* 对于 `invpcid` 指令导致的 VM-exit，`handle_invpcid() -> kvm_handle_invpcid()`
```cpp
static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
     ...
     [EXIT_REASON_INVPCID]                 = handle_invpcid,
     ...
}
```
* LAM 不会应用于 TLB invalidation 的输入地址，因此 `invpcid` 的 LAM 模拟无需特殊处理
```diff
diff --git a/arch/x86/kvm/x86.c b/arch/x86/kvm/x86.c
index 557872c0331a..ecfe97aa35c2 100644
--- a/arch/x86/kvm/x86.c
+++ b/arch/x86/kvm/x86.c
@@ -13563,6 +13563,10 @@ int kvm_handle_invpcid(struct kvm_vcpu *vcpu, unsigned long type, gva_t gva)

    switch (type) {
    case INVPCID_TYPE_INDIV_ADDR:
+       /*
+        * LAM doesn't apply to addresses that are inputs to TLB
+        * invalidation.
+        */
        if ((!pcid_enabled && (operand.pcid != 0)) ||
            is_noncanonical_address(operand.gla, vcpu)) {
            kvm_inject_gp(vcpu, 0);
```
### 对 `invvpid` 的模拟
* 对于嵌套虚拟化未开启的场景，`invvpid` 导致的 VM-exit 会给 guest 产生一个 `#UD` 异常
```cpp
/*
 * When nested=0, all VMX instruction VM Exits filter here.  The handlers
 * are overwritten by nested_vmx_setup() when nested=1.
 */
static int handle_vmx_instruction(struct kvm_vcpu *vcpu)
{
    kvm_queue_exception(vcpu, UD_VECTOR);
    return 1;
}
...
static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
     ...
     [EXIT_REASON_INVVPID]                 = handle_vmx_instruction,
     ...
}
```
* 对于嵌套虚拟化开启的场景，`invvpid` 的处理函数被重载为 `handle_invvpid()`
```cpp
__init int nested_vmx_hardware_setup(int (*exit_handlers[])(struct kvm_vcpu *))
{
    ...
    exit_handlers[EXIT_REASON_INVVPID]  = handle_invvpid;
    ...
}
```
* 然而，LAM 不会应用于 TLB invalidation 的输入地址，因此 `invvpid` 的 LAM 模拟也无需特殊处理
```diff
diff --git a/arch/x86/kvm/vmx/nested.c b/arch/x86/kvm/vmx/nested.c
index 51622878d6e4..4ba46e1b29d2 100644
--- a/arch/x86/kvm/vmx/nested.c
+++ b/arch/x86/kvm/vmx/nested.c
@@ -5797,6 +5798,10 @@ static int handle_invvpid(struct kvm_vcpu *vcpu)
    vpid02 = nested_get_vpid02(vcpu);
    switch (type) {
    case VMX_VPID_EXTENT_INDIVIDUAL_ADDR:
+       /*
+        * LAM doesn't apply to addresses that are inputs to TLB
+        * invalidation.
+        */
        if (!operand.vpid ||
            is_noncanonical_address(operand.gla, vcpu))
            return nested_vmx_fail(vcpu,
```
### 对 VMX 指令和 `INVPCID` 操作数的 untag 处理
* `ret` 为指向通过传址来返回的 GVA 的指针，在 `get_vmx_mem_address()` 对其进行统一处理
```diff
diff --git a/arch/x86/kvm/vmx/nested.c b/arch/x86/kvm/vmx/nested.c
index 51622878d6e4..4ba46e1b29d2 100644
--- a/arch/x86/kvm/vmx/nested.c
+++ b/arch/x86/kvm/vmx/nested.c
@@ -4980,6 +4980,7 @@ int get_vmx_mem_address(struct kvm_vcpu *vcpu, unsigned long exit_qualification,
        else
            *ret = off;

+       *ret = vmx_get_untagged_addr(vcpu, *ret, 0);
        /* Long mode: #GP(0)/#SS(0) if the memory address is in a
         * non-canonical form. This is the only check on the memory
         * destination for long mode!
```
* 看看 `get_vmx_mem_address()` 调用处就知道它的影响了：
```cpp
1 arch/x86/kvm/vmx/nested.c|5095| <<nested_vmx_get_vmptr>> if (get_vmx_mem_address(vcpu, vmx_get_exit_qual(vcpu),
2 arch/x86/kvm/vmx/nested.c|5447| <<handle_vmread>> if (get_vmx_mem_address(vcpu, exit_qualification,
3 arch/x86/kvm/vmx/nested.c|5521| <<handle_vmwrite>> if (get_vmx_mem_address(vcpu, exit_qualification,
4 arch/x86/kvm/vmx/nested.c|5689| <<handle_vmptrst>> if (get_vmx_mem_address(vcpu, exit_qual, instr_info,
5 arch/x86/kvm/vmx/nested.c|5737| <<handle_invept>> if (get_vmx_mem_address(vcpu, vmx_get_exit_qual(vcpu),
6 arch/x86/kvm/vmx/nested.c|5820| <<handle_invvpid>> if (get_vmx_mem_address(vcpu, vmx_get_exit_qual(vcpu),
7 arch/x86/kvm/vmx/nested.h|34| <<handle_invvpid>> int get_vmx_mem_address(struct kvm_vcpu *vcpu, unsigned long exit_qualification,
8 arch/x86/kvm/vmx/vmx.c|5961| <<handle_invpcid>> if (get_vmx_mem_address(vcpu, vmx_get_exit_qual(vcpu),
```
* 1-7 是对嵌套虚拟化的影响，操作数需要 untag
* 8 是在 `handle_invpcid()` 调用 `kvm_handle_invpcid()` 之前调用的 `get_vmx_mem_address()`，而且不只是嵌套虚拟化
  * 虽然 `kvm_handle_invpcid()` 没对 LAM 做处理，但在之前的 `get_vmx_mem_address()` 已经处理过了

### 对 SGX ENCLS 的操作数的处理
* 修改了 `sgx_get_encls_gva()`，对 GVA 进行 untag 处理
```diff
diff --git a/arch/x86/kvm/vmx/sgx.c b/arch/x86/kvm/vmx/sgx.c
index 3e822e582497..6fef01e0536e 100644
--- a/arch/x86/kvm/vmx/sgx.c
+++ b/arch/x86/kvm/vmx/sgx.c
@@ -37,6 +37,7 @@ static int sgx_get_encls_gva(struct kvm_vcpu *vcpu, unsigned long offset,
    if (!IS_ALIGNED(*gva, alignment)) {
        fault = true;
    } else if (likely(is_64_bit_mode(vcpu))) {
+       *gva = vmx_get_untagged_addr(vcpu, *gva, 0);
        fault = is_noncanonical_address(*gva, vcpu);
    } else {
        *gva &= 0xffffffff;
```
* 同样，通过 `sgx_get_encls_gva()` 的调用处可以了解它的使用场景
```cpp
1 arch/x86/kvm/vmx/sgx.c|228| <<handle_encls_ecreate>> if (sgx_get_encls_gva(vcpu, kvm_rbx_read(vcpu), 32, 32, &pageinfo_gva) ||
2 arch/x86/kvm/vmx/sgx.c|229| <<handle_encls_ecreate>> sgx_get_encls_gva(vcpu, kvm_rcx_read(vcpu), 4096, 4096, &secs_gva))
3 arch/x86/kvm/vmx/sgx.c|247| <<handle_encls_ecreate>> if (sgx_get_encls_gva(vcpu, pageinfo.metadata, 64, 64, &metadata_gva) ||
4 arch/x86/kvm/vmx/sgx.c|248| <<handle_encls_ecreate>> sgx_get_encls_gva(vcpu, pageinfo.contents, 4096, 4096,
5 arch/x86/kvm/vmx/sgx.c|305| <<handle_encls_einit>> if (sgx_get_encls_gva(vcpu, kvm_rbx_read(vcpu), 1808, 4096, &sig_gva) ||
6 arch/x86/kvm/vmx/sgx.c|306| <<handle_encls_einit>> sgx_get_encls_gva(vcpu, kvm_rcx_read(vcpu), 4096, 4096, &secs_gva) ||
7 arch/x86/kvm/vmx/sgx.c|307| <<handle_encls_einit>> sgx_get_encls_gva(vcpu, kvm_rdx_read(vcpu), 304, 512, &token_gva))
```
* Guest 执行 SGX ENCLS `ECREATE` 和 `EINIT` 的时候会导致 VM-exit，这时会通过 `sgx_get_encls_gva()` 对操作数进行 untag 处理
```cpp
static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
    [EXIT_REASON_ENCLS]           = handle_encls,
}
handle_encls()
   if (leaf == ECREATE)
   -> handle_encls_ecreate()
   if (leaf == EINIT)
   -> handle_encls_einit()
```
* `ECREATE` 的 SECS 中指定的 `BASEADDR` 无需进行 LAM 处理
  * 译注：为什么？是因为这完全是 guest 的内部事务吗？

## [PATCH 09/12] KVM: x86: Virtualize LAM for supervisor pointer
* 添加支持以允许 guest 为 LAM 设置新的 `CR4` 控制位，并添加实现以获取 supervisor 指针的 untagged 地址。
* LAM 修改了应用于数据访问的 64 位线性地址的规范性检查，允许软件将未转换的地址位用于元数据，并在将元数据位用作访问内存的线性地址之前屏蔽元数据位。
  * LAM 使用 `CR4.LAM_SUP`（bit `28`）来配置和启用 supervisor 指针的 LAM。
  * 它还更改了 `VMENTER`，以允许在 VMCS 的 `HOST_CR4` 和 `GUEST_CR4` 中设置该位以支持虚拟化。
  * 注意，即使不是 64 位模式，`CR4.LAM_SUP` 也可以设置，但不会生效，因为 LAM 仅适用于 64 位线性地址。
* 将 `CR4.LAM_SUP` 移出 `CR4_RESERVED_BITS`，其保留取决于 vCPU 是否支持 LAM。
  * 将其保留为拦截状态，以防止在 LAM 未暴露给 guest 的情况下 guest 设置该位，并避免每次 KVM 获取其值时进行 `vmread`，以期 guest 不会频繁切换该位。
* 在虚拟机的模拟 `IA32_VMX_CR4_FIXED1` MSR 中设置 `CR4.LAM_SUP` 位，以允许虚拟机在嵌套 VMX 操作中为 supervisor 指针启用 LAM。
* 当 `CR4.LAM_SUP` 切换时，硬件不需要执行 TLB 刷新，KVM 不需要基于它来模拟 TLB 刷新。
  * `{kvm,vmx}_set_cr4()` 中没有其他功能或 `vmx_exec_controls` 连接，也不需要其他代码。
* 跳过指令 fetches（包括分支目标）、`INVLPG` 指令的操作数和隐式系统访问（`X86EMUL_F_IMPLICIT`）的地址 untag，所有这些都不受 untagging 的影响。
  * 请注意，不会为隐式系统访问调用 `get_untagged_addr()`，因为没有理由这样做，但出于文档目的无论如何都要检查该标志。
---
* 将 `CR4.LAM_SUP` 移出 `CR4_RESERVED_BITS`，
```diff
diff --git a/arch/x86/include/asm/kvm_host.h b/arch/x86/include/asm/kvm_host.h
index 8d042f27d644..f96988f283d5 100644
--- a/arch/x86/include/asm/kvm_host.h
+++ b/arch/x86/include/asm/kvm_host.h
@@ -133,7 +133,8 @@
#define CR4_RESERVED_BITS                                               \
    (~(unsigned long)(X86_CR4_VME | X86_CR4_PVI | X86_CR4_TSD | X86_CR4_DE\
              | X86_CR4_PSE | X86_CR4_PAE | X86_CR4_MCE     \
              | X86_CR4_PGE | X86_CR4_PCE | X86_CR4_OSFXSR | X86_CR4_PCIDE \
              | X86_CR4_OSXSAVE | X86_CR4_SMEP | X86_CR4_FSGSBASE \
              | X86_CR4_OSXMMEXCPT | X86_CR4_LA57 | X86_CR4_VMXE \
-             | X86_CR4_SMAP | X86_CR4_PKE | X86_CR4_UMIP))
+             | X86_CR4_SMAP | X86_CR4_PKE | X86_CR4_UMIP \
+             | X86_CR4_LAM_SUP))
```
* `CR4.LAM_SUP` 保留与否取决于 vCPU 是否支持 LAM。
```diff
diff --git a/arch/x86/kvm/x86.h b/arch/x86/kvm/x86.h
index 5184fde1dc54..2f7e19166658 100644
--- a/arch/x86/kvm/x86.h
+++ b/arch/x86/kvm/x86.h
@@ -530,6 +530,8 @@ bool kvm_msr_allowed(struct kvm_vcpu *vcpu, u32 index, u32 type);
        __reserved_bits |= X86_CR4_VMXE;        \
    if (!__cpu_has(__c, X86_FEATURE_PCID))          \
        __reserved_bits |= X86_CR4_PCIDE;       \
+   if (!__cpu_has(__c, X86_FEATURE_LAM))           \
+       __reserved_bits |= X86_CR4_LAM_SUP;     \
    __reserved_bits;                                \
 })
```
* 在虚拟机的模拟 `IA32_VMX_CR4_FIXED1` MSR 中设置 `CR4.LAM_SUP` 位，以允许虚拟机在嵌套 VMX 操作中为 supervisor 指针启用 LAM。
```diff
diff --git a/arch/x86/kvm/vmx/vmx.c b/arch/x86/kvm/vmx/vmx.c
index d7ff06d77720..3bbe5a9362a9 100644
--- a/arch/x86/kvm/vmx/vmx.c
+++ b/arch/x86/kvm/vmx/vmx.c
@@ -7673,6 +7673,9 @@ static void nested_vmx_cr_fixed1_bits_update(struct kvm_vcpu *vcpu)
    cr4_fixed1_update(X86_CR4_UMIP,       ecx, feature_bit(UMIP));
    cr4_fixed1_update(X86_CR4_LA57,       ecx, feature_bit(LA57));

+   entry = kvm_find_cpuid_entry_index(vcpu, 0x7, 1);
+   cr4_fixed1_update(X86_CR4_LAM_SUP,    eax, feature_bit(LAM));
+
 #undef cr4_fixed1_update
 }
```
* 真正实现 `get_untagged_addr` 函数指针在 VMX 时的回调函数 `vmx_get_untagged_addr()` 的内核态部分。下一个 patch 实现用户态部分，这里把代码一次贴完整了。
* **注意**，SDM 规定线性地址在修改后的规范性检查 *之后* 被屏蔽，而 KVM 屏蔽（untags）地址，然后执行“正常”规范性检查。
  * 从功能上讲，这两种方法是相同的，并且当发生相对于规范性检查的屏蔽时，软件看不到，即 KVM 的行为不会违反 SDM。
```cpp
/*
 * Note, the SDM states that the linear address is masked *after* the modified
 * canonicality check, whereas KVM masks (untags) the address and then performs
 * a "normal" canonicality check.  Functionally, the two methods are identical,
 * and when the masking occurs relative to the canonicality check isn't visible
 * to software, i.e. KVM's behavior doesn't violate the SDM.
 */
gva_t vmx_get_untagged_addr(struct kvm_vcpu *vcpu, gva_t gva, unsigned int flags)
{
    int lam_bit;
    unsigned long cr3_bits;
    //跳过指令 fetches（包括分支目标）、INVLPG 指令的操作数和隐式系统访问的地址 untag，所有这些都不受 untagging 的影响
    if (flags & (X86EMUL_F_FETCH | X86EMUL_F_IMPLICIT | X86EMUL_F_INVLPG))
        return gva;
    //LAM 只支持 64 位模式
    if (!is_64_bit_mode(vcpu))
        return gva;
    //bit 63 确定该地址是否应被视为用户态地址或 supervisor 地址
    /*
     * Bit 63 determines if the address should be treated as user address
     * or a supervisor address.
     */
    if (!(gva & BIT_ULL(63))) { //bit 63 未设置，是用户态地址
        cr3_bits = kvm_get_active_cr3_lam_bits(vcpu); //获取 guest CR3 中的 LAM_U57 和 LAM_U48 开关
        if (!(cr3_bits & (X86_CR3_LAM_U57 | X86_CR3_LAM_U48))) //如果都没开，直接返回 GVA
            return gva;
        //如果打开了 LAM_U57，忽略 LAM_U48
        /* LAM_U48 is ignored if LAM_U57 is set. */
        lam_bit = cr3_bits & X86_CR3_LAM_U57 ? 56 : 47;
    } else { //bit 63 已设置，是内核态地址
        if (!kvm_is_cr4_bit_set(vcpu, X86_CR4_LAM_SUP)) //获取 guest CR4 中的 LAM_SUP 开关
            return gva; //如果没开，直接返回 GVA
        //获取 guest CR4 中的五级分页开关，如果开启，guest 支持 LAM57；否则支持 LAM48
        lam_bit = kvm_is_cr4_bit_set(vcpu, X86_CR4_LA57) ? 56 : 47;
    }
    //通过对 lam_bit 进行符号扩展来 untag 地址，但不要扩展到 bit 63。bit 63 保留在原始虚拟地址中，因此 untag 不会将用户访问权限更改为管理员访问权限，反之亦然。
    /*
     * Untag the address by sign-extending the lam_bit, but NOT to bit 63.
     * Bit 63 is retained from the raw virtual address so that untagging
     * doesn't change a user access to a supervisor access, and vice versa.
     */
    return (sign_extend64(gva, lam_bit) & ~BIT_ULL(63)) | (gva & BIT_ULL(63));
}
```

## [PATCH 10/12] KVM: x86: Virtualize LAM for user pointer
* 添加支持以允许 guest 为 Linear Address Masking（LAM）设置新的 `CR3` 控制位，并添加实现以获取用户态指针的 untagged 地址。
* LAM 修改了 64 位线性地址的规范检查，允许软件将屏蔽/忽略的地址位用于元数据。
  * 在使用线性地址访问内存之前，硬件会屏蔽元数据位。 
  * LAM 使用两个新的 `CR3` 非地址位 `LAM_U48`（bit `62`）和 `LAM_U57`（bit `61`）来为用户态指针配置 LAM。
  * LAM 还更改了 `VMENTER`，以允许在 VMCS 的 `HOST_CR3` 和 `GUEST_CR3` 中设置这两个位以进行虚拟化。
```diff
@@ -8230,8 +8232,12 @@ gva_t vmx_get_untagged_addr(struct kvm_vcpu *vcpu, gva_t gva, unsigned int flags
     * or a supervisor address.
     */
    if (!(gva & BIT_ULL(63))) {
-       /* KVM doesn't yet virtualize LAM_U{48,57}. */
-       return gva;
+       cr3_bits = kvm_get_active_cr3_lam_bits(vcpu);
+       if (!(cr3_bits & (X86_CR3_LAM_U57 | X86_CR3_LAM_U48)))
+           return gva;
+
+       /* LAM_U48 is ignored if LAM_U57 is set. */
+       lam_bit = cr3_bits & X86_CR3_LAM_U57 ? 56 : 47;
    } else {
        if (!kvm_is_cr4_bit_set(vcpu, X86_CR4_LAM_SUP))
            return gva;
```
* 当 EPT 打开时，`CR3` 不会被 KVM 捕获（trapped），并且由 guest 来设置两个 LAM 控制位中的任何一个。
  * 但是，当 EPT 关闭时，guest 使用的实际 `CR3` 是从 shadow MMU root 生成的，这与 guest *设置* 的 `CR3` 不同，并且 KVM 需要基于 guest *看到* 的 cached 的 `CR3`，手动将任何激活的控制位应用于 VMCS 的 `GUEST_CR3`。
  * 译注：由于 EPT 开启时写 `CR3` 不会 VM-exit，这样就没法截获 guest 的进程是否开启了 LAM。为了弥补这个虚拟化漏洞，只有在加载 guest `CR3` 前读取一下 cached 的 guest `CR3` 看是否设置了 `LAM_U48` 或者 `LAM_U57` 了
```diff
diff --git a/arch/x86/kvm/vmx/vmx.c b/arch/x86/kvm/vmx/vmx.c
index 3bbe5a9362a9..3825ab316be9 100644
--- a/arch/x86/kvm/vmx/vmx.c
+++ b/arch/x86/kvm/vmx/vmx.c
@@ -3400,7 +3400,8 @@ static void vmx_load_mmu_pgd(struct kvm_vcpu *vcpu, hpa_t root_hpa,
            update_guest_cr3 = false;
        vmx_ept_load_pdptrs(vcpu);
    } else {
-       guest_cr3 = root_hpa | kvm_get_active_pcid(vcpu);
+       guest_cr3 = root_hpa | kvm_get_active_pcid(vcpu) |
+               kvm_get_active_cr3_lam_bits(vcpu);
    }

    if (update_guest_cr3)
```
* KVM 手动检查 guest 的 `CR3` 以确保它指向有效的 guest 物理地址（即支持 guest 中较小的 `MAXPHYSADDR`）。
  * 扩展此检查以允许设置两个 LAM 控制位。
  * 检查后，guest `CR3` 的 LAM bit 将被剥离以提取 guest 物理地址（GPA）。
  * 译注：当 KVM CPU 有 LAM 这个特性且 guest 的 `cpuid` 也有 LAM 这个特性时，清除 `CR3` 中 `LAM_U48`（bit `62`） 和 `LAM_U57`（bit `61`）的 bits
```diff
diff --git a/arch/x86/kvm/cpuid.h b/arch/x86/kvm/cpuid.h
index fa04a00090a2..e4bd59c15a2c 100644
--- a/arch/x86/kvm/cpuid.h
+++ b/arch/x86/kvm/cpuid.h
@@ -276,6 +276,10 @@ static __always_inline bool guest_can_use(struct kvm_vcpu *vcpu,

 static inline bool kvm_vcpu_is_legal_cr3(struct kvm_vcpu *vcpu, unsigned long cr3)
 {
+   if (kvm_cpu_cap_has(X86_FEATURE_LAM) &&
+       guest_cpuid_has(vcpu, X86_FEATURE_LAM))
+       cr3 &= ~(X86_CR3_LAM_U48 | X86_CR3_LAM_U57);
+
    return kvm_vcpu_is_legal_gpa(vcpu, cr3);
 }
```
* 在嵌套的情况下，对于支持 LAM 的 guest 虚拟机，VMCS12 的 `HOST_CR3` 和 `GUEST_CR3` 都允许设置新的 LAM 控制位，即
  * 当 L0 进入 L1 模拟从 L2 到 L1 的 VMEXIT 时或
  * 当 L0 直接进入 L2 时。
* KVM 还手动检查 VMCS12 的 `HOST_CR3` 和 `GUEST_CR3` 是否为有效物理地址。扩展此类检查以允许新的 LAM 控制位。
* **注意**，LAM 没有全局控制位来完全打开/关闭 LAM，而是完全取决于硬件的 `CPUID` 来确定是否可以启用它。
  * 这意味着，当 EPT 打开时，即使 KVM 没有向 guest 透露 LAM，guest 仍然可以在 `CR3` 中设置 LAM 控制位，而不会导致问题。这是一个不幸的虚拟化漏洞。
  * 在这种情况下，KVM 可以选择拦截 CR3 并注入 fault，但这会在运行不带 LAM 支持的正常 VM 时损害性能。这是不希望的。
  * 只是选择让 guest 做这种非法的事情，因为最坏的情况是当 KVM 最终发现这种非法行为并且 guest 行为不端时，guest 被杀死。
---
* 新增函数读取 vCPU cached 的 `CR3` 的值里是否激活了 `LAM_U48` 或 `LAM_U57`
```cpp
static inline unsigned long kvm_get_active_cr3_lam_bits(struct kvm_vcpu *vcpu)
{
    if (!guest_can_use(vcpu, X86_FEATURE_LAM))
        return 0;

    return kvm_read_cr3(vcpu) & (X86_CR3_LAM_U48 | X86_CR3_LAM_U57);
}
```

## [PATCH 11/12] KVM: x86: Advertise and enable LAM (user and supervisor)
* LAM 由 `CPUID.7.1:EAX.LAM[bit 26]` 枚举。向用户空间通告该功能并启用它，作为对 supervisor 和用户指针的 LAM 虚拟化支持之后的最后一步。
* SGX LAM 支持尚未公布（advertised）。SGX LAM 支持在 SGX 自己的 `CPUID` 中枚举，并且没有硬性要求在 CPUID leaf `0x7` 中报告 LAM 时必须支持它。
---
* 通过以下路径设置 vCPU 的能力
```cpp
vmx_init()
-> kvm_x86_vendor_init(&vmx_init_ops)
   -> __kvm_x86_vendor_init(ops)
      -> ops->hardware_setup()
      => hardware_setup()
         -> vmx_set_cpu_caps()
            -> kvm_set_cpu_caps()
               -> kvm_cpu_cap_mask(CPUID_7_1_EAX, ...| F(LAM))
                     kvm_cpu_caps[leaf] &= mask;
                  -> __kvm_cpu_cap_mask(leaf)
```
* 向 guest 的用户空间通告 LAM 的功能
```diff
diff --git a/arch/x86/kvm/cpuid.c b/arch/x86/kvm/cpuid.c
index dda6fc4cfae8..40d68fef748a 100644
--- a/arch/x86/kvm/cpuid.c
+++ b/arch/x86/kvm/cpuid.c
@@ -671,7 +671,7 @@ void kvm_set_cpu_caps(void)
    kvm_cpu_cap_mask(CPUID_7_1_EAX,
        F(AVX_VNNI) | F(AVX512_BF16) | F(CMPCCXADD) |
        F(FZRM) | F(FSRS) | F(FSRC) |
-       F(AMX_FP16) | F(AVX_IFMA)
+       F(AMX_FP16) | F(AVX_IFMA) | F(LAM)
    );

    kvm_cpu_cap_init_kvm_defined(CPUID_7_1_EDX,
```

## [PATCH 12/12] KVM: x86: Use KVM-governed feature framework to track "LAM enabled"
* 使用 governed feature framework 来跟踪 Linear Address Masking（LAM）是否已“启用”，即 guest 是否可以使用 LAM。
* 使用该框架可以避免在 `cr3` 和 vmexit 处理 LAM 路径期间相对昂贵的 `guest_cpuid_has()` 调用。
* 无意进行功能改变。
---
* 向 governed feature framework 添加 LAM 的特性
```diff
diff --git a/arch/x86/kvm/governed_features.h b/arch/x86/kvm/governed_features.h
index 423a73395c10..ad463b1ed4e4 100644
--- a/arch/x86/kvm/governed_features.h
+++ b/arch/x86/kvm/governed_features.h
@@ -16,6 +16,7 @@ KVM_GOVERNED_X86_FEATURE(PAUSEFILTER)
 KVM_GOVERNED_X86_FEATURE(PFTHRESHOLD)
 KVM_GOVERNED_X86_FEATURE(VGIF)
 KVM_GOVERNED_X86_FEATURE(VNMI)
+KVM_GOVERNED_X86_FEATURE(LAM)

 #undef KVM_GOVERNED_X86_FEATURE
 #undef KVM_GOVERNED_FEATURE
diff --git a/arch/x86/kvm/vmx/vmx.c b/arch/x86/kvm/vmx/vmx.c
index 3825ab316be9..d30df9b3fe3e 100644
--- a/arch/x86/kvm/vmx/vmx.c
+++ b/arch/x86/kvm/vmx/vmx.c
@@ -7763,6 +7763,7 @@ static void vmx_vcpu_after_set_cpuid(struct kvm_vcpu *vcpu)
        kvm_governed_feature_check_and_set(vcpu, X86_FEATURE_XSAVES);

    kvm_governed_feature_check_and_set(vcpu, X86_FEATURE_VMX);
+   kvm_governed_feature_check_and_set(vcpu, X86_FEATURE_LAM);

    vmx_setup_uret_msrs(vmx);
```
* 将 `guest_cpuid_has()` 改为开销更低的 `guest_can_use()` 调用
```diff
diff --git a/arch/x86/kvm/cpuid.h b/arch/x86/kvm/cpuid.h
index e4bd59c15a2c..856e3037e74f 100644
--- a/arch/x86/kvm/cpuid.h
+++ b/arch/x86/kvm/cpuid.h
@@ -276,8 +276,7 @@ static __always_inline bool guest_can_use(struct kvm_vcpu *vcpu,

 static inline bool kvm_vcpu_is_legal_cr3(struct kvm_vcpu *vcpu, unsigned long cr3)
 {
-   if (kvm_cpu_cap_has(X86_FEATURE_LAM) &&
-       guest_cpuid_has(vcpu, X86_FEATURE_LAM))
+   if (guest_can_use(vcpu, X86_FEATURE_LAM))
        cr3 &= ~(X86_CR3_LAM_U48 | X86_CR3_LAM_U57);

    return kvm_vcpu_is_legal_gpa(vcpu, cr3);
diff --git a/arch/x86/kvm/mmu.h b/arch/x86/kvm/mmu.h
index 580e2414b88c..60f21bb4c27b 100644
--- a/arch/x86/kvm/mmu.h
+++ b/arch/x86/kvm/mmu.h
@@ -148,8 +148,7 @@ static inline unsigned long kvm_get_active_pcid(struct kvm_vcpu *vcpu)

 static inline unsigned long kvm_get_active_cr3_lam_bits(struct kvm_vcpu *vcpu)
 {
-   if (!kvm_cpu_cap_has(X86_FEATURE_LAM) ||
-       !guest_cpuid_has(vcpu, X86_FEATURE_LAM))
+   if (!guest_can_use(vcpu, X86_FEATURE_LAM))
        return 0;

    return kvm_read_cr3(vcpu) & (X86_CR3_LAM_U48 | X86_CR3_LAM_U57);
```

## References
- [[GIT PULL] KVM: x86: LAM support for 6.8](https://lore.kernel.org/kvm/20240104193303.3175844-4-seanjc@google.com/)
- [[PATCH v11 00/16] LAM and LASS KVM Enabling](https://lore.kernel.org/kvm/20230913124227.12574-1-binbin.wu@linux.intel.com/)
