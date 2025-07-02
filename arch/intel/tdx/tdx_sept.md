# TDX Secure EPT
![Guest TD Control Structures Overview](pic/guest_td_control_struct_ov.png)
* Secure EPT 的根页表在 TDCS 中，TDCS 由多个 TDCX 页面组成，Secure EPT 的根页表只是其中一个

## 如何设置 Secure EPTP
### 1. VMM 初始化 TDX Module
* `struct kvm_x86_init_ops` 类型的操作 `vt_init_ops` 变量：
  * arch/x86/kvm/vmx/main.c
  ```c
  struct kvm_x86_init_ops vt_init_ops __initdata = {
      .hardware_setup = vt_hardware_setup,
      .handle_intel_pt_intr = NULL,

      .runtime_ops = &vt_x86_ops,
      .pmu_ops = &intel_pmu_ops,
  };
  ```
* `tdsysinfo` 是个 `struct tdsysinfo_struct` 类型的全局变量
  ```c
  static DECLARE_PADDED_STRUCT(tdsysinfo_struct, tdsysinfo,
                   TDSYSINFO_STRUCT_SIZE, TDSYSINFO_STRUCT_ALIGNMENT);
  ```
  * `tdx_get_sysinfo()` 返回指向这个全局变量的地址
* TDX module 经过全局初始化 `TDH.SYS.INIT` 和 LP-Scope 初始化 `TDH.SYS.LP.INIT` 之后，一旦一个 LP 已经被初始化了，host VMM 可以通过 `TDH_SYS_INFO` seamcall 得到 `tdsysinfo` 的值
```c
module_init(vt_init)
-> vt_init()
   -> kvm_x86_vendor_init(&vt_init_ops)
      -> __kvm_x86_vendor_init(ops)
         -> ops->hardware_setup()
         => vt_hardware_setup()
            -> vmx_hardware_setup()
            -> enable_tdx && !tdx_hardware_setup(&vt_x86_ops)
               arch/x86/kvm/vmx/tdx.c
               -> tdx_module_setup()
                  +-> tdx_enable()
                  |      switch (tdx_module_status)
                  |      case TDX_MODULE_UNKNOWN:
                  |      -> __tdx_enable()
                  |         -> init_tdx_module()
                  |               struct tdsysinfo_struct *sysinfo = &PADDED_STRUCT(tdsysinfo)
                  |            -> seamcall(TDH_SYS_INIT, 0, 0, 0, 0, NULL, NULL)
                  |            -> tdx_module_init_cpus()
                  |                 for_each_online_cpu(cpu)
                  |                 -> smp_call_function_single(cpu, do_lp_init, &err, true)
                  |                       csd->func = func;
                  |                    -> generic_exec_single(cpu, csd)
                  |                          smp_call_func_t func = csd->func;
                  |                       -> func(info)
                  |                       => do_lp_init()
                  |                          -> seamcall(TDH_SYS_LP_INIT, 0, 0, 0, 0, NULL, NULL)
                  |                    -> __smp_call_single_queue(cpu, &csd->node.llist)
                  |            -> __tdx_get_sysinfo(sysinfo, cmr_array)
                  |                  sysinfo_pa = slow_virt_to_phys(sysinfo)
                  |               -> seamcall(TDH_SYS_INFO, sysinfo_pa, TDSYSINFO_STRUCT_SIZE, cmr_array_pa, MAX_CMRS, NULL, &out)
                  |               -> pr_info("TDX module: atributes 0x%x, vendor_id 0x%x, major_version %u, minor_version %u, build_date %u, build_num %u", ...)
                  |               -> print_cmrs(cmr_array, out.r9)
                  +-> tdsysinfo = tdx_get_sysinfo();
                      -> tdx_caps = (struct tdx_capabilities) {
                             .tdcs_nr_pages = tdsysinfo->tdcs_base_size / PAGE_SIZE,}
   -> vmx_init()
   -> tdx_init()
   -> kvm_init()
```
* 有一个不太确定的行为，`TDH.SYS.INIT` 和 `TDH_SYS_INFO` 等 seamcall 一上来就可以被调用，我猜测是因为 seamldr 在 `SEAMLDR.INSTALL` 的时候，把 `SEAM_SIGSTRUCT` 结构中 `Intel TDX MODULE CONFIGURATION PARAMETERS` 部分的 `RIP_OFFSET` 域的值作为 seamldr 给 TDX module 的 VMCS 构造的入口点
  * 这个入口点就是 `src/td_dispatcher/tdx_td_transitions.S::tdx_tdexit_entry_point`，它会 `callq tdx_td_dispatcher` 来分派发给 TDX modules 的 seamcall
* `SEAMLDR.INSTALL` 的一段处理：

> 1. Check input `SEAMLDR_PARAMS` address (in `RCX`).
> 2. Read `SEAMLDR_PARAMS` (pointed by `RCX`) and check the structure's contents.
> 3. Read `SEAM_SIGSTRUCT` (pointed by `SEAMLDR_PARAMS.SEAM_SIGSTRUCT_PA`), check the structure's contents, and authenticate it.
> ...
> 10. Create mappings for the new Intel TDX module's code, data, and stacks.
> 11. Create transfer VMCSs for the new Intel TDX module.
> ...
> -- Intel® Trust Domain Extensions - SEAM Loader (SEAMLDR) Interface Specification

### 2. TDX Module 返回 `struct td_sys_info_s`
* 存放 TDCS 所需的大小（byte）记录在 `td_sys_info_s.tdcs_base_size`
* tdx-module/include/tdx_api_defs.h
```c
typedef struct PACKED td_sys_info_s
{
...
    /**
     * Control Struct Info
     */
    uint16_t tdcs_base_size; /**< Base value for the number of bytes required to hold TDCS. */
...
}
```
* VMM 调用 `SEAMCALL(TDH.SYS.INFO)` 时路径如下：
```c
tdx_tdexit_entry_point
   call tdx_td_dispatcher
   -> tdx_td_dispatcher()
      case TDH_SYS_INFO_LEAF:
      -> tdh_sys_info()
```
* `td_sys_info_s.tdcs_base_size` 是 hard-coded 的 `_4KB * MAX_NUM_TDCS_PAGES`
* tdx-module/src/vmm_dispatcher/api_calls/tdh_sys_info.c
```cpp
api_error_type tdh_sys_info(uint64_t tdhsysinfo_output_pa,
        uint64_t num_of_bytes_in_buffer, uint64_t target_cmr_info_pa,
        uint64_t num_of_cmr_info_entries)
{
...
    pa_t tdhsysinfo_pa = {.raw = tdhsysinfo_output_pa};
...
    tdhsysinfo_output_la = (td_sys_info_t *) map_pa(tdhsysinfo_pa.raw_void, TDX_RANGE_RW);
    basic_memset_to_zero(tdhsysinfo_output_la, sizeof(td_sys_info_t));
...
    tdhsysinfo_output_la->tdcs_base_size = _4KB * MAX_NUM_TDCS_PAGES; //_4KB * TDCS_PAGES;
...
}
```
* 目前 TDCS pages 就分了 4 页，`MAX_NUM_TDCS_PAGES`，其中第 3 个用作 secure EPT 的根页表
* tdx-module/src/common/data_structures/td_control_structures.h
```c
/**
 * @brief Indices of TDCS pages
 */
typedef enum
{
    MSR_BITMAPS_PAGE_INDEX = 1,
    SEPT_ROOT_PAGE_INDEX   = 2,
    ZERO_PAGE_INDEX        = 3,
    MAX_NUM_TDCS_PAGES     = 4   /**< Total number of TDCS pages */
} tdcs_page_index_t;
```

### 3. VMM 添加 `TDH.MNG.ADDCX` 函数添加控制结构页面
* host VMM 可以通过 `TDH.MNG.ADDCX` 函数添加控制结构页面来构建 TD Control Structure（TDCS）
* TDCS 页面的数目通过 `struct tdx_capabilities tdx_caps` 的 `.tdcs_nr_pages` 域得到
  * 在 `tdx_module_setup()` 被赋予了 `tdsysinfo->tdcs_base_size / PAGE_SIZE`，见上面
* 通过 `ioctl(KVM_MEMORY_ENCRYPT_OP, tdx_cmd.id = KVM_TDX_INIT_VM)` 在初始化 TDVM 的时候会添加该 VM 的 TDCS 页面
```c
//virt/kvm/kvm_main.c
kvm_vm_ioctl()
default:
   //arch/x86/kvm/x86.c
-> kvm_arch_vm_ioctl()
   case KVM_MEMORY_ENCRYPT_OP:
   -> static_call(kvm_x86_mem_enc_ioctl)(kvm, argp)
      //arch/x86/kvm/vmx/main.c
   => vt_mem_enc_ioctl()
         //arch/x86/kvm/vmx/tdx.c
      -> tdx_vm_ioctl()
         switch (tdx_cmd.id)
         case KVM_TDX_INIT_VM:
         -> tdx_td_init(kvm, &tdx_cmd)
            -> setup_tdparams(kvm, td_params, init_vm)
                   td_params->eptp_controls = VMX_EPTP_MT_WB;
                   if (cpu_has_vmx_ept_5levels() && max_pa > 48)
                      td_params->eptp_controls |= VMX_EPTP_PWL_5;
                   else
                      td_params->eptp_controls |= VMX_EPTP_PWL_4
            -> __tdx_td_init(kvm, td_params)
```

```cpp
static int __tdx_td_init(struct kvm *kvm, struct td_params *td_params)
{
...
    // 先在 host VMM 侧分配 TDCS 页面
    tdcs_pa = kcalloc(tdx_caps.tdcs_nr_pages, sizeof(*kvm_tdx->tdcs_pa),
              GFP_KERNEL_ACCOUNT | __GFP_ZERO);
    if (!tdcs_pa)
        goto free_tdr;
    for (i = 0; i < tdx_caps.tdcs_nr_pages; i++) {
        va = __get_free_page(GFP_KERNEL_ACCOUNT);
        if (!va)
            goto free_tdcs;
        tdcs_pa[i] = __pa(va);
    }
...
    // 用 seamcall(TDH_MNG_ADDCX) 将这些页面塞给 TDX module
    kvm_tdx->tdcs_pa = tdcs_pa;
    for (i = 0; i < tdx_caps.tdcs_nr_pages; i++) {
        err = tdh_mng_addcx(kvm_tdx->tdr_pa, tdcs_pa[i]);
        if (WARN_ON_ONCE(err)) {
            pr_tdx_error(TDH_MNG_ADDCX, err, NULL);
            for (i++; i < tdx_caps.tdcs_nr_pages; i++) {
                free_page((unsigned long)__va(tdcs_pa[i]));
                tdcs_pa[i] = 0;
            }
            ret = -EIO;
            goto teardown;
        }
    }
    // seamcall(TDH_MNG_INIT) 时会设置 secure EPT 的根页面
    err = tdh_mng_init(kvm_tdx->tdr_pa, __pa(td_params), &out);
...
}
```

### 4. TDX module 接收 `TDH_MNG_ADDCX` 添加的页面
* Host VMM 通过 `seamcall(TDH_MNG_ADDCX)` 添加 TDCS 页面的 PA 会记录到 `tdr_ptr->management_fields.tdcx_pa[]`
* tdx-module/src/vmm_dispatcher/api_calls/tdh_mng_add_cx.c
```c
api_error_type tdh_mng_add_cx(uint64_t target_tdcx_pa, uint64_t target_tdr_pa)
{
...
    if (tdcx_index_num == MSR_BITMAPS_PAGE_INDEX)
    {
        fill_area_cacheline(tdcx_ptr, TDX_PAGE_SIZE_IN_BYTES, (~(uint64_t)0));
    }
    else if (tdcx_index_num == SEPT_ROOT_PAGE_INDEX)
    {
        fill_area_cacheline(tdcx_ptr, TDX_PAGE_SIZE_IN_BYTES, SEPTE_INIT_VALUE);
    }
    else
    {
        zero_area_cacheline(tdcx_ptr, TDX_PAGE_SIZE_IN_BYTES);
    }

    // Register the new TDCS page in its parent TDR
    tdr_ptr->management_fields.tdcx_pa[tdcx_index_num] = tdcx_pa.raw;
    tdr_ptr->management_fields.num_tdcx = (uint32_t)(tdcx_index_num + 1);
...
}
```

### 5. TDX module 处理 `TDH_MNG_INIT` 设置 secure EPT 根页表
* Host VMM 随后调用的 `tdh_mng_init()` 触发 `seamcall(TDH_MNG_INIT)`
* tdx-module/src/vmm_dispatcher/api_calls/tdh_mng_init.c
```c
api_error_type tdh_mng_init(uint64_t target_tdr_pa, uint64_t target_td_params_pa)
{
...
    return_val = read_and_set_td_configurations(tdcs_ptr,
                                                td_params_ptr,
                                                MAX_PA,
                                                tdr_ptr->management_fields.tdcx_pa[SEPT_ROOT_PAGE_INDEX],
                                                &virt_tsc_freq);
...
}

static api_error_type read_and_set_td_configurations(tdcs_t * tdcs_ptr,
                                                     td_params_t * td_params_ptr,
                                                     uint64_t tdx_max_pa,
                                                     uint64_t sept_root_raw_pa,
                                                     uint16_t* virt_tsc_freq)
{
    ia32e_eptp_t   target_eptp = { .raw = 0 };
    pa_t           sept_root_pa = {.raw = sept_root_raw_pa};
...
    // Read and verify EPTP_CONTROLS
    target_eptp.raw = td_params_ptr->eptp_controls.raw;
...
    /**
     *  The PA field of EPTP points to the Secure EPT root page in TDCS,
     *  which has already been initialized to 0 during TDADDCX
     */
    target_eptp.fields.base_pa = sept_root_pa.page_4k_num;

    tdcs_ptr->executions_ctl_fields.eptp.raw = target_eptp.raw;
...
}
```
* 通过以上一系列的操作，终于将 secure EPT 的 PA 设置到了 TDCS 的执行控制域的 EPTP 指针

## 如何设置 Shared EPTP

* 相对 secure EPTP，shared EPTP 的设置简单的太多了
```c
kvm_mmu_load()
   if (vcpu->arch.mmu->root_role.direct)
   -> mmu_alloc_direct_roots()
        struct kvm_mmu *mmu = vcpu->arch.mmu;
        if (is_tdp_mmu_enabled(vcpu->kvm))
           root = kvm_tdp_mmu_get_vcpu_root_hpa(vcpu, false);
           -> return __pa(kvm_tdp_mmu_get_vcpu_root(vcpu, private)->spt);
              -> root = tdp_mmu_alloc_sp(vcpu, role);
              -> tdp_mmu_init_sp(root, NULL, 0)
              -> return root;
           mmu->root.hpa = root;
   -> kvm_mmu_load_pgd()
      -> static_call(kvm_x86_load_mmu_pgd)(vcpu, root_hpa, vcpu->arch.mmu->root_role.level)
      => vt_load_mmu_pgd()
            if (is_td_vcpu(vcpu))
            -> tdx_load_mmu_pgd(vcpu, root_hpa, pgd_level)
               -> td_vmcs_write64(to_tdx(vcpu), SHARED_EPT_POINTER, root_hpa & PAGE_MASK);
```

## References
- Intel® Trust Domain Extensions - SEAM Loader (SEAMLDR) Interface Specification