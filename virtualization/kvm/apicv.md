# APIC 虚拟化

## APIC Access Page
* CPU 的每个逻辑核都有一个 local APIC，而每个 local APIC 内部都有个 `4 KB` 大小的页来存放各种 APIC 寄存器
* 对于采用 xAPIC mode 的 local APIC 来说，可以用 MMIO 的方式来访问 APIC 寄存器
  * APIC 寄存器最初映射到 **物理地址空间的相同 `4 KB` 区域**，对于每个逻辑核来说都一样，上电或复位后，该字段设置为 `0xFEE0 0000`
  * `IA32_APIC_BASE` MSR 的 **APIC Base field, bits 12 到 35** 来指定 APIC 寄存器空间基地址
  * 可以通过修改 `IA32_APIC_BASE` MSR 的 *APIC Base field* 来指定一个其他的物理地址
  * 对于每个核可以修改 `IA32_APIC_BASE` MSR 指定自己的 APIC 寄存器基地址
  * 对于 P6 系列、Pentium 4 和 Intel Xeon 处理器，APIC 在内部处理对 `4 KB` APIC 寄存器空间内的地址的所有内存访问，并且不产生外部总线周期
* APIC access page 的概念是类似的，只是它是在 host 侧给 VM 准备的，可以只分配一个，由同一个 VM 中的 vCPU 公用
* 可以看到这里创建了一个有专属 slot id `APIC_ACCESS_PAGE_PRIVATE_MEMSLOT` 的 memory region
* arch/x86/kvm/lapic.c
```cpp
int kvm_alloc_apic_access_page(struct kvm *kvm)
{
    struct page *page;
    void __user *hva;
    int ret = 0;

    mutex_lock(&kvm->slots_lock);
    if (kvm->arch.apic_access_memslot_enabled ||
        kvm->arch.apic_access_memslot_inhibited)
        goto out;

    hva = __x86_set_memory_region(kvm, APIC_ACCESS_PAGE_PRIVATE_MEMSLOT,
                      APIC_DEFAULT_PHYS_BASE, PAGE_SIZE);
    if (IS_ERR(hva)) {
        ret = PTR_ERR(hva);
        goto out;
    }

    page = gfn_to_page(kvm, APIC_DEFAULT_PHYS_BASE >> PAGE_SHIFT);
    if (is_error_page(page)) {
        ret = -EFAULT;
        goto out;
    }

    /*
     * Do not pin the page in memory, so that memory hot-unplug
     * is able to migrate it.
     */
    put_page(page);
    kvm->arch.apic_access_memslot_enabled = true;
out:
    mutex_unlock(&kvm->slots_lock);
    return ret;
}
```
* `gfn_to_page()` 会导致 faultin 将对应的物理页分配出来，但并不会建立 GPA 到 HPA 的映射关系
```c
gfn_to_page()
-> gfn_to_pfn()
   -> return gfn_to_pfn_memslot(gfn_to_memslot(kvm, gfn), gfn);
      -> return __gfn_to_memslot(kvm_memslots(kvm), gfn);
         -> return ____gfn_to_memslot(slots, gfn, false);
            -> try_get_memslot(slot, gfn)
            -> search_memslots()
               return slot;
         -> return __gfn_to_pfn_memslot(slot, gfn, false, false, NULL, true, NULL, NULL);
            -> __gfn_to_hva_many()
            -> return hva_to_pfn(addr, atomic, interruptible, async, write_fault, writable);
               -> hva_to_pfn_fast()
                  -> get_user_page_fast_only(addr, FOLL_WRITE, page)
                        *pfn = page_to_pfn(page[0]);
               -> hva_to_pfn_slow()
                  -> get_user_pages_unlocked()
```
* 于是，在 guest 访问 `0xFEE00000` 发生 EPT violation 的时候，不是按常规的 MMIO 访问需要 VMM 模拟，而是通过这个专属的 slot 让缺页处理去填充这个 EPT entry 到 APIC access page。这是一个值得注意的地方。
```c
kvm_faultin_pfn()
-> __kvm_faultin_pfn()
   -> fault->pfn = __gfn_to_pfn_memslot()
```
* VMCS 是 per vCPU 的，但这里大家都公用 GPA `APIC_DEFAULT_PHYS_BASE = 0xFEE00000` 对应的同一个 HPA，写到 VMCS 的 `APIC_ACCESS_ADDR` 域中
* arch/x86/kvm/vmx/vmx.c
```cpp
static void vmx_set_apic_access_page_addr(struct kvm_vcpu *vcpu)
{
    struct page *page;

    /* Defer reload until vmcs01 is the current VMCS. */
    if (is_guest_mode(vcpu)) {
        to_vmx(vcpu)->nested.reload_vmcs01_apic_access_page = true;
        return;
    }

    if (!(secondary_exec_controls_get(to_vmx(vcpu)) &
        SECONDARY_EXEC_VIRTUALIZE_APIC_ACCESSES))
        return;

    page = gfn_to_page(vcpu->kvm, APIC_DEFAULT_PHYS_BASE >> PAGE_SHIFT);
    if (is_error_page(page))
        return;

    vmcs_write64(APIC_ACCESS_ADDR, page_to_phys(page));
    vmx_flush_tlb_current(vcpu);

    /*
     * Do not pin apic access page in memory, the MMU notifier
     * will call us again if it is migrated or swapped out.
     */
    put_page(page);
}
```
* 当用 EPT 将 `0xFEE00000` 转为 VMCS 的 `APIC_ACCESS_ADDR` 域中的 HPA 的时候，开始 APIC access page 虚拟化的处理
* `vmx_set_apic_access_page_addr()` 作为 `struct kvm_x86_ops vmx_x86_ops` 中的一个操作（方法），仅由 `kvm_vcpu_reload_apic_access_page()` 调用
* arch/x86/kvm/x86.c
```cpp
static void kvm_vcpu_reload_apic_access_page(struct kvm_vcpu *vcpu)
{
    if (!lapic_in_kernel(vcpu))
        return;

    static_call_cond(kvm_x86_set_apic_access_page_addr)(vcpu);
}
```
* 而 `kvm_vcpu_reload_apic_access_page()` 仅在 `vcpu_enter_guest()` 的时候，发现有 `KVM_REQ_APIC_PAGE_RELOAD` KVM request 的时候调用
```cpp
static int vcpu_enter_guest(struct kvm_vcpu *vcpu)
{
...
        if (kvm_check_request(KVM_REQ_APIC_PAGE_RELOAD, vcpu))
            kvm_vcpu_reload_apic_access_page(vcpu);
...
}
```
* 我们可以看一下会生成 `KVM_REQ_APIC_PAGE_RELOAD` KVM request 的场景：
```cpp
arch/x86/kvm/vmx/nested.c|4849| <<nested_vmx_vmexit>> kvm_make_request(KVM_REQ_APIC_PAGE_RELOAD, vcpu);
arch/x86/kvm/vmx/vmx.c|4869| <<vmx_vcpu_reset>> kvm_make_request(KVM_REQ_APIC_PAGE_RELOAD, vcpu);
arch/x86/kvm/vmx/vmx.c|6687| <<vmx_set_virtual_apic_mode>> kvm_make_request(KVM_REQ_APIC_PAGE_RELOAD, vcpu);
arch/x86/kvm/x86.c|10445| <<kvm_arch_mmu_notifier_invalidate_range>> kvm_make_all_cpus_request(kvm, KVM_REQ_APIC_PAGE_RELOAD);
```
* 其中 `vmx_set_virtual_apic_mode()` 将 guest APIC mode 变为 `LAPIC_MODE_XAPIC` 是会导致 reload access page 的

## Virtual APIC Page
### 分配 Virtual APIC Page
* arch/x86/kvm/lapic.c
```cpp
int kvm_create_lapic(struct kvm_vcpu *vcpu, int timer_advance_ns)
{
    struct kvm_lapic *apic;

    ASSERT(vcpu != NULL);

    apic = kzalloc(sizeof(*apic), GFP_KERNEL_ACCOUNT);
    if (!apic)
        goto nomem;

    vcpu->arch.apic = apic;

    apic->regs = (void *)get_zeroed_page(GFP_KERNEL_ACCOUNT);
    if (!apic->regs) {
        printk(KERN_ERR "malloc apic regs error for vcpu %x\n",
               vcpu->vcpu_id);
        goto nomem_free_apic;
    }
    ...
}
```
### 设置 Virtual APIC Page
* arch/x86/kvm/vmx/vmx.c
```cpp
static void init_vmcs(struct vcpu_vmx *vmx)
{
...
    if (cpu_has_vmx_tpr_shadow()) {
        vmcs_write64(VIRTUAL_APIC_PAGE_ADDR, 0);
        if (cpu_need_tpr_shadow(&vmx->vcpu))
            vmcs_write64(VIRTUAL_APIC_PAGE_ADDR,
                     __pa(vmx->vcpu.arch.apic->regs));
        vmcs_write32(TPR_THRESHOLD, 0);
    }
...
}
```