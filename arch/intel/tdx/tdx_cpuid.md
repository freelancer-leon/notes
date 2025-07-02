# TDX CPUID

* TDX 对 `CPUID` 的虚拟化大致分为四种类型：
  * **Native**：平台的原生值，由 TDX module 初始化时遍历得到，存在 global metadata 域中
  * **As Configured**：Per VM 初始化的时候由 Qemu 以 native 的值作为蓝本，结合用户的配置加工而成，会下发给 KVM
    * As Configured（if Native）: 原生的值为 `0`，则它也为 `0`；否则由 VMM 配置
  * **Calculated**：KVM 最后会将 configured 后的值下发给 TDX module，TDX module 计算后存入 per VM 的 `CPUID` 数组中
  * **Fixed**：固定的值，例如 `CPUID.(EAX=0x21)`
* Guest 执行 `cpuid` 指令时，VM-Exit 到 TDX module，TDX module 查询 `CPUID` 数组，
  * 如果合法，则返回给 TD
    * 此时还有再次计算 `CPUID` 的值的机会，见 `tdx-module/src/td_dispatcher/vm_exits/td_cpuid.c::td_cpuid_exit()`
  * 如果不合法，则 `#VE` 弹射回 TD
    * TD 发起 tdvmcall `VP.VMCALL<Instruction.CPUID>` 将请求发给 host，由 host 将模拟的结果返回给 TD
  * 除此之外还有一些其他的情况，参考 TDX module spec CPUID 虚拟化章节

## TDX Module 初始化时遍历 CPUID
* TDX module 初始化的时候，会调用 seamcall `TDH_SYS_INIT`，该实现会用 `cpuid` 遍历预定义的 leaves/sub-leaves 放在 global metadata 域中，作为 **native** CPUID 值
```cpp
//tdx-module/src/vmm_dispatcher/api_calls/tdh_sys_init.c
tdh_sys_init()
-> check_platform_config_and_cpu_enumeration()
   -> check_cpuid_configurations(tdx_global_data_ptr)
```
* 根据 `cpuid_lookup[]` 数组，执行 `cpuid` 命令，将结果存入 `global_data_ptr->cpuid_values[]` 作为蓝本
```cpp
_STATIC_INLINE_ api_error_type check_cpuid_configurations(tdx_module_global_t* global_data_ptr)
{
...
    uint32_t last_base_leaf, last_extended_leaf;
    uint32_t ebx, ecx, edx;
    //运行 cpuid 0 返回的 $eax 的返回值是最大的基本 cpuid 范围
    // Last base leaves need to be taken before
    ia32_cpuid(CPUID_MAX_INPUT_VAL_LEAF, 0, &last_base_leaf, &ebx, &ecx, &edx);
    if (last_base_leaf < CPUID_MIN_LAST_CPU_BASE_LEAF)
    {
        return api_error_with_operand_id(TDX_CPUID_LEAF_NOT_SUPPORTED, CPUID_MIN_LAST_CPU_BASE_LEAF);
    }
    global_data_ptr->cpuid_last_base_leaf = CPUID_LAST_BASE_LEAF;
    //运行 cpuid 0x80000000 返回的 $eax 的返回值是最大的扩展 cpuid 范围
    ia32_cpuid(CPUID_MAX_EXTENDED_VAL_LEAF, 0, &last_extended_leaf, &ebx, &ecx, &edx);
    if (last_extended_leaf < CPUID_LAST_EXTENDED_LEAF)
    {
        return api_error_with_operand_id(TDX_CPUID_LEAF_NOT_SUPPORTED, CPUID_LAST_EXTENDED_LEAF);
    }
    tdx_sanity_check(last_extended_leaf >= CPUID_MAX_EXTENDED_VAL_LEAF, SCEC_SEAMCALL_SOURCE(TDH_SYS_INIT_LEAF), 4);
    global_data_ptr->cpuid_last_extended_leaf = last_extended_leaf;
    //cpuid_lookup[] 是 tdx-module/include/auto_gen/cpuid_configurations.c 中用工具生成的一个大数组，记录了要遍历的 cpuid
    for (uint32_t i = 0; i < MAX_NUM_CPUID_LOOKUP; i++)
    {   //如果数组元素无效则跳过
        if (!cpuid_lookup[i].valid_entry)
        {
            continue;
        }
        //运行基本 cpuid 和 扩展 cpuid 范围内的 cpuid
        cpuid_config.leaf_subleaf = cpuid_lookup[i].leaf_subleaf;
        if ((cpuid_config.leaf_subleaf.leaf <= last_base_leaf) ||
            ((cpuid_config.leaf_subleaf.leaf >= CPUID_FIRST_EXTENDED_LEAF) &&
             (cpuid_config.leaf_subleaf.leaf <= last_extended_leaf)))
        {
            ia32_cpuid(cpuid_config.leaf_subleaf.leaf, cpuid_config.leaf_subleaf.subleaf,
                    &cpuid_config.values.eax, &cpuid_config.values.ebx,
                    &cpuid_config.values.ecx, &cpuid_config.values.edx);
        }
        else
        {   //不在范围内的 cpuid 用全 0 值作为缺省值
            // Leaf is not supported by the CPU, use all-0 values as default
            cpuid_config.values.low = 0;
            cpuid_config.values.high = 0;
        }
        //对特定的 cpuid 进行一系列的检测和判断
...
        //将 cpuid 条目存到 global metadata 域中
        global_data_ptr->cpuid_values[i] = cpuid_config;
    }

    return TDX_SUCCESS;
}
```
## KMV 初始化 TDX Module 时读取 CPUID
* KVM 中 `tdx_info->cpuid_configs[i]` 中的 `CPUID` 数据的来源于 TDX module 初始化时用 seamcall `TDH_SYS_RD` 从 global metadata 域中读取到的值
```cpp
//arch/x86/kvm/vmx/main.c
module_init(vt_init)
-> vt_init()
      //arch/x86/kvm/x86.c
   -> kvm_x86_vendor_init(&vt_init_ops)
      -> kvm_x86_vendor_init(ops)
         -> __kvm_x86_vendor_init(ops)
            -> ops->hardware_setup()
               //arch/x86/kvm/vmx/main.c
            => vt_hardware_setup()
                  //arch/x86/kvm/vmx/tdx.c
               -> tdx_hardware_setup(&vt_x86_ops)
                  -> tdx_module_setup()
                        struct tdx_md_map mds[] = {
                           TDX_MD_MAP(NUM_CPUID_CONFIG, &num_cpuid_config),
                           TDX_MD_MAP(TDCS_BASE_SIZE, &tdcs_base_size),
                           TDX_MD_MAP(TDVPS_BASE_SIZE, &tdvps_base_size),
                        };
                     -> tdx_md_read(mds, ARRAY_SIZE(mds)) //seamcall TDH_SYS_RD 读取以上 global metadata 域
                        -> tdx_sys_metadata_field_read()
                           -> seamcall_prerr_ret(TDH_SYS_RD, &args)
                     -> tdx_info = kzalloc(sizeof(*tdx_info) +  //给 TDX module 定义的 CPUID 数组分配空间
                              sizeof(*tdx_info->cpuid_configs) * num_cpuid_config, GFP_KERNEL);
                        tdx_info->num_cpuid_config = num_cpuid_config;
                        for (i = 0; i < num_cpuid_config; i++) { //根据上面得到的 CPUID 条目数遍历
                            struct kvm_tdx_cpuid_config *c = &tdx_info->cpuid_configs[i];
                            u64 leaf, eax_ebx, ecx_edx;
                            struct tdx_md_map cpuids[] = {
                                TDX_MD_MAP(CPUID_CONFIG_LEAVES + i, &leaf),
                                TDX_MD_MAP(CPUID_CONFIG_VALUES + i * 2, &eax_ebx),
                                TDX_MD_MAP(CPUID_CONFIG_VALUES + i * 2 + 1, &ecx_edx),
                            };
                            //seamcall TDH_SYS_RD 读取一条 CPUID 配置，写入 tdx_info->cpuid_configs[i]
                            ret = tdx_md_read(cpuids, ARRAY_SIZE(cpuids));
                            if (ret)
                                goto error_out;
               
                            c->leaf = (u32)leaf;
                            c->sub_leaf = leaf >> 32;
                            c->eax = (u32)eax_ebx;
                            c->ebx = eax_ebx >> 32;
                            c->ecx = (u32)ecx_edx;
                            c->edx = ecx_edx >> 32;
                        } 
```
## TDX Module 提供给 VMM 可配置的 CPUID 数据
* TDX Module 对应 seamcall `TDH_SYS_RD` 的路径如下：
```cpp
//tdx-module/src/vmm_dispatcher/api_calls/tdh_sys_rd.c
tdh_sys_rd()
   //tdx-module/src/common/metadata_handlers/metadata_generic.c
-> md_read_element()
   case MD_CTX_SYS:
      //tdx-module/src/common/metadata_handlers/metadata_sys.c
   -> md_sys_read_element()
      -> md_sys_get_elements()
```
* 对应 `TDX_MD_MAP(CPUID_CONFIG_LEAVES + i, &leaf)` 和 `TDX_MD_MAP(CPUID_CONFIG_VALUES, ...)` 的读取在这里
  * VMM 可配置 CPUID 条目在生成的数组 `const cpuid_configurable_t cpuid_configurable[MAX_NUM_CPUID_CONFIG]` 中规定，包括索引都是生成的，不在此数组内的 CPUID leaf 和 sub-leaf，VMM 无法读取
  * tdx-module/src/common/metadata_handlers/metadata_sys.c
```cpp
static bool_t md_sys_get_elements(md_field_id_t field_id, const md_lookup_t* entry,
                                  uint64_t* element_array, uint64_t* array_size)
{
...
    uint32_t field_num = (field_id.field_code - entry->field_id.field_code) / entry->num_of_elem;

    switch (entry->field_id.class_code)
    {
...
            else if (entry->field_id.field_code == MD_SYS_CPUID_CONFIG_LEAVES_FIELD_CODE)
            {
                if (field_num >= MAX_NUM_CPUID_CONFIG)
                { //如果 VMM 读取的可配置 CPUID 条目超出范围，返回数据里写入全 1
                    *element_array = ~(0ULL);
                }
                else
                { //否则返回指定的 leaf 和 sub-leaf
                    lfence(); // Prevent speculative access to non-existant array field
                    *element_array = cpuid_configurable[field_num].leaf_subleaf.raw;
                }
            }
            else if (entry->field_id.field_code == MD_SYS_CPUID_CONFIG_VALUES_FIELD_CODE)
            {
                if (field_num >= MAX_NUM_CPUID_CONFIG)
                { //如果 VMM 读取的可配置 CPUID 条目超出范围，返回数据里写入 0
                    // No configurable bits
                    element_array[0] = 0;
                    element_array[1] = 0;
                }
                else
                {
                    lfence(); // Prevent speculative access to non-existant array field
                    //数组 cpuid_configurable[] 中元素的 .lookup_index 是其在数组 cpuid_lookup[] 中对应的元素的索引
                    uint32_t lookup_index = cpuid_configurable[field_num].lookup_index;

                    *array_size = 2 * sizeof(uint64_t);
                    //将 CPUID 数据从蓝本取出，即 global metadata 的 CPUID 的 native 值，仅暴露可配置的位给 VMM
                    element_array[0] = cpuid_configurable[field_num].config_direct.low |
                            (cpuid_configurable[field_num].allow_direct.low &
                             global_data->cpuid_values[lookup_index].values.low);

                    element_array[1] = cpuid_configurable[field_num].config_direct.high |
                            (cpuid_configurable[field_num].allow_direct.high &
                             global_data->cpuid_values[lookup_index].values.high);
                }
            }
...
    }
    return true;
}
```

## TD VM 初始化时的 CPUID

### VMM 初始化时 CPUID 读取源自 TDX Module 的数据作为基础
* Qemu 初始化 VM 时，通过 `ioctl(KVM_TDX_CAPABILITIES)` 获取一份 KVM 提供的 TDX capabilities `struct kvm_tdx_capabilities *tdx_caps`，其中就包含 `CPUID`
```c
(gdb) bt
                          tdx_vm_ioctl(KVM_TDX_CAPABILITIES, 0, caps)
#0  0x00005555559602b6 in get_tdx_capabilities () at ../target/i386/kvm/tdx.c:1129
#1  0x00005555559602b6 in tdx_kvm_init (ms=<optimized out>, errp=0x7fffffffc9b8) at ../target/i386/kvm/tdx.c:1135
#2  0x000055555595bb17 in kvm_confidential_guest_init (errp=0x7fffffffc9b8, ms=<optimized out>) at ../target/i386/kvm/kvm.c:2538
#3  0x000055555595bb17 in kvm_arch_init (ms=ms@entry=0x555556a02e70, s=s@entry=0x555556a7f090) at ../target/i386/kvm/kvm.c:2558
#4  0x0000555555af7582 in kvm_init (ms=0x555556a02e70) at ../accel/kvm/kvm-all.c:2619
#5  0x0000555555921d12 in accel_init_machine (accel=accel@entry=0x555556a7f090, ms=0x555556a02e70) at ../accel/accel-softmmu.c:39
#6  0x00005555558bb37e in do_configure_accelerator (opaque=0x7fffffffcdf0, opts=0x5555567587b0, errp=<optimized out>)
    at ../softmmu/vl.c:2222
#7  0x0000555555c77b8d in qemu_opts_foreach
    (list=<optimized out>, func=func@entry=0x5555558bb2b0 <do_configure_accelerator>, opaque=opaque@entry=0x7fffffffcdf0, errp=0x5555566df9d0 <error_fatal>) at ../util/qemu-option.c:1135
#8  0x00005555558bff97 in configure_accelerators (progname=<optimized out>) at ../softmmu/vl.c:2288
#9  0x00005555558bff97 in qemu_init (argc=<optimized out>, argv=<optimized out>) at ../softmmu/vl.c:3552
#10 0x00005555556f39a9 in main (argc=<optimized out>, argv=<optimized out>) at ../softmmu/main.c:47
(gdb) c
```
* KVM 对应 `ioctl(KVM_TDX_CAPABILITIES)` 的实现是 `tdx_get_capabilities()` 函数
```cpp
static int tdx_get_capabilities(struct kvm_tdx_cmd *cmd)
{
    struct kvm_tdx_capabilities __user *user_caps;
    struct kvm_tdx_capabilities *caps = NULL;
    int ret = 0;

    if (cmd->flags)
        return -EINVAL;

    caps = kmalloc(sizeof(*caps), GFP_KERNEL);
    if (!caps)
        return -ENOMEM;

    user_caps = (void __user *)cmd->data;
    if (copy_from_user(caps, user_caps, sizeof(*caps))) {
        ret = -EFAULT;
        goto out;
    }

    if (caps->nr_cpuid_configs < tdx_info->num_cpuid_config) {
        ret = -E2BIG;
        goto out;
    }

    *caps = (struct kvm_tdx_capabilities) {
        .attrs_fixed0 = tdx_info->attributes_fixed0,
        .attrs_fixed1 = tdx_info->attributes_fixed1,
        .xfam_fixed0 = tdx_info->xfam_fixed0,
        .xfam_fixed1 = tdx_info->xfam_fixed1,
        .supported_gpaw = TDX_CAP_GPAW_48 |
        ((kvm_get_shadow_phys_bits() >= 52 &&
          cpu_has_vmx_ept_5levels()) ? TDX_CAP_GPAW_52 : 0),
          cpu_has_vmx_ept_5levels()) ? TDX_CAP_GPAW_52 : 0),
        .nr_cpuid_configs = tdx_info->num_cpuid_config,
        .padding = 0,
    };

    if (copy_to_user(user_caps, caps, sizeof(*caps))) {
        ret = -EFAULT;
        goto out;
    }
    if (copy_to_user(user_caps->cpuid_configs, &tdx_info->cpuid_configs,
             tdx_info->num_cpuid_config *
             sizeof(tdx_info->cpuid_configs[0]))) {
        ret = -EFAULT;
    }

out:
    /* kfree() accepts NULL. */
    kfree(caps);
    return ret;
}
```
* 可见它将 `tdx_info->cpuid_configs[i]` 中的 `CPUID` 数据拷贝了一份给 Qemu `struct kvm_tdx_capabilities *tdx_caps` 指向的 buffer

### VMM 初始化 vCPU 时传入配置过的 CPUID
* Qemu 初始化 vCPU 时先根据虚拟机的配置和运行平台的 `CPUID` 传给 KVM，KVM 再传给 TDX module。之后，Qemu 还会给每个 vCPU 准备一份 `CPUID` 数据
  * 调用路径如下：
```cpp
kvm_init_vcpu()
-> kvm_arch_pre_create_vcpu()
   -> tdx_pre_create_vcpu()
      -> init_vm->cpuid.nent = kvm_x86_arch_cpuid(env, init_vm->cpuid.entries, 0)
      -> tdx_vm_ioctl(KVM_TDX_INIT_VM, 0, init_vm)
-> kvm_arch_init_vcpu()
   -> cpuid_i = kvm_x86_arch_cpuid(env, cpuid_data.entries, cpuid_i)
   -> kvm_vcpu_ioctl(cs, KVM_SET_CPUID2, &cpuid_data)
```
* 其中 `kvm_x86_arch_cpuid()` 可能会通过 `tdx_cap_cpuid_config()` 函数从 `tdx_caps` 取得 KVM 提供的源自 TDX module 返回的 `CPUID` 数据，比如 `cpuid 0x1f` 的数据
* Kernel 中 `ioctl(KVM_TDX_INIT_VM)` 传递的 `init_vm` 中的 `cpuid.entries` 来自用户态的输入
  * arch/x86/kvm/vmx/tdx.c
```cpp
static int tdx_td_init(struct kvm *kvm, struct kvm_tdx_cmd *cmd)
{
    struct kvm_tdx_init_vm *init_vm = NULL;
...
    init_vm = kzalloc(struct_size(init_vm, cpuid.entries, KVM_MAX_CPUID_ENTRIES), GFP_KERNEL);
...
    if (copy_from_user(init_vm, (void __user *)cmd->data, sizeof(*init_vm))) {
        ret = -EFAULT;
        goto out;
    }
    if (init_vm->cpuid.nent > KVM_MAX_CPUID_ENTRIES) {
        ret = -E2BIG;
        goto out;
    }
    if (copy_from_user(init_vm->cpuid.entries,
               (void __user *)cmd->data + sizeof(*init_vm),
               flex_array_size(init_vm, cpuid.entries, init_vm->cpuid.nent))) {
        ret = -EFAULT;
        goto out;
    }
...
    td_params = kzalloc(sizeof(struct td_params), GFP_KERNEL);
...
    ret = setup_tdparams(kvm, td_params, init_vm);
...
    ret = __tdx_td_init(kvm, td_params, &cmd->error);
}
```
* `td_params` 是入参，需要填入将来 `__tdx_td_init() -> tdh_mng_init(kvm_tdx->tdr_pa, __pa(td_params), &out)` 调用 seamcall `TDH_MNG_INIT` 需要的各种 TD 参数
```cpp
static int setup_tdparams(struct kvm *kvm, struct td_params *td_params,
            struct kvm_tdx_init_vm *init_vm)
{
    struct kvm_cpuid2 *cpuid = &init_vm->cpuid;
...
    setup_tdparams_cpuids(kvm, cpuid, td_params);
    ret = setup_tdparams_xfam(cpuid, td_params);
...
}
```
* 根据 `tdsysinfo.cpuid_configs[]` 和用户输入的 `cpuid->entries` 计算配给 TD VM 的 `CPUID` 配置，填入 `td_params->cpuid_values[i]`
```cpp
static void setup_tdparams_cpuids(struct kvm *kvm, struct kvm_cpuid2 *cpuid,
                  struct td_params *td_params)
{
    struct kvm_tdx *kvm_tdx = to_kvm_tdx(kvm);
    int i;

    /*
     * td_params.cpuid_values: The number and the order of cpuid_value must
     * be same to the one of struct tdsysinfo.{num_cpuid_config, cpuid_configs}
     * It's assumed that td_params was zeroed.
     */
    kvm_tdx->cpuid_nent = 0;
    for (i = 0; i < tdx_info->num_cpuid_config; i++) {
        const struct kvm_tdx_cpuid_config *c = &tdx_info->cpuid_configs[i];
        /* KVM_TDX_CPUID_NO_SUBLEAF means index = 0. */
        u32 index = c->sub_leaf == KVM_TDX_CPUID_NO_SUBLEAF ? 0 : c->sub_leaf;
        const struct kvm_cpuid_entry2 *entry =
            kvm_find_cpuid_entry2(cpuid->entries, cpuid->nent,
                          c->leaf, index);
        struct tdx_cpuid_value *value = &td_params->cpuid_values[i];

        if (!entry)
            continue;
        //entry 来自用户输入，c 来自 TDX module，计算结果写入 value，即 td_params->cpuid_values[i]
        /*
         * tdsysinfo.cpuid_configs[].{eax, ebx, ecx, edx}
         * bit 1 means it can be configured to zero or one.
         * bit 0 means it must be zero.
         * Mask out non-configurable bits.
         */
        value->eax = entry->eax & c->eax;
        value->ebx = entry->ebx & c->ebx;
        value->ecx = entry->ecx & c->ecx;
        value->edx = entry->edx & c->edx;

        /* Remember the setting to check for KVM_SET_CPUID2. */
        kvm_tdx->cpuid[kvm_tdx->cpuid_nent] = *entry;
        kvm_tdx->cpuid_nent++;
    }
}
```
* 以上构造的就是 VMM **Configured** 的 CPUID 的值
* TDX module 在处理 seamcall `TDH_MNG_INIT` 时，在 `read_and_set_cpuid_configurations()` 函数中会在 `global_data_ptr->cpuid_values[cpuid_index]` 的基础上将 `CPUID` 配置的值存入 per TDVM 的 `tdcs_ptr->cpuid_config_vals[cpuid_index]` 数组中，即最终 TD 执行 `cpuid` 指令时查询的数组

## Guest 执行 `CPUID` 时 TDX Module 的处理
* 当 guest 执行 `CPUID` 指令时，会 VM exit 到 TDX module，根据情况
  * 将结果返回给 TD
    * 这里会用到从 TDCS 获取由 `TDH_MNG_INIT` 计算（**calculated**）出的 `CPUID` 值，即之前的 seamcall `TDH_MNG_INIT` 时填充的 `tdcs_ptr->cpuid_config_vals[cpuid_index]` 数组
  * 还是给 TD 注入一个 `#VE`（即 `#VE` 弹射）
    * TD 的 `#VE` handler 发现是 `ve->exit_reason` 是 `EXIT_REASON_CPUID`，通过 tdvmcall `VP.VMCALL<Instruction.CPUID>` 将请求发给 host，由 host 将模拟的结果返回给 TD
```c
tdx_tdexit_entry_point
-> callq tdx_td_dispatcher
      switch (vm_exit_reason.basic_reason)
      case VMEXIT_REASON_CPUID_INSTRUCTION:
           //tdx-module/src/td_dispatcher/vm_exits/td_cpuid.c
           td_cpuid_exit()
           -> index = get_cpuid_lookup_entry(leaf, subleaf) //在 cpuid_lookup[] 中查询 leaf，subleaf，返回数组索引
              if ((index == CPUID_LOOKUP_IDX_NA) || cpuid_lookup[index].faulting) {
              -> tdx_inject_ve(VMEXIT_REASON_CPUID_INSTRUCTION, 0, vp_ctx->tdvps, 0, 0)
                 return;
              }
              return_values = vp_ctx->tdcs->cpuid_config_vals[index] //从 TDCS 获取由 TDH_MNG_INIT 计算出的 CPUID 值
              // Special CPUID Leaves/Sub-Leaves Handling
              switch (leaf) {...}
              // Write the CPUID return values into the guest TD's GPR image，将 CPUID 返回值写入 guest TD 的 GPR 镜像
              vp_ctx->tdvps->guest_state.gpr_state.rax = return_values.eax;
              vp_ctx->tdvps->guest_state.gpr_state.rbx = return_values.ebx;
              vp_ctx->tdvps->guest_state.gpr_state.rcx = return_values.ecx;
              vp_ctx->tdvps->guest_state.gpr_state.rdx = return_values.edx;
    -> tdx_return_to_td(true, false, &tdx_local_data_ptr->vp_ctx.tdvps->guest_state.gpr_state) //从 TDX module 返回 TD
```

## CPUID - AVX 的支持
* 比如 `cpuid 0x1` 里 AVX 的支持
```sh
# cpuid -1 -l 0x1 | grep -i avx
      AVX: advanced vector extensions         = true
```
* `CPUID Bit or Field Virtualization` 类型是 `XFAM & Configured & Native`，含义就是，虚拟位值为 `1`，当且仅当配置：
  * `TD_PARAMS.XFAM<bits>` 设置后（详见 `setup_tdparams_xfam()`），
  * `TD_PARAMS.CPUID_CONFIG` 中的 bit 配置为 `1`（Qemu 配置没禁用 AVX 的支持），
  * 执行 `CPUID` 返回的原生 bit 值为 `1`（Host CPU 得支持啊）。
* `TD_PARAMS.XFAM<bits>` 和 `TD_PARAMS.XSS<bits>` 相关的值获取自 `setup_tdparams_xfam()`
  * arch/x86/kvm/vmx/tdx.c 
```cpp
static int setup_tdparams_xfam(struct kvm_cpuid2 *cpuid, struct td_params *td_params)
{
    const struct kvm_cpuid_entry2 *entry;
    u64 guest_supported_xcr0;
    u64 guest_supported_xss;

    /* Setup td_params.xfam */
    entry = kvm_find_cpuid_entry2(cpuid->entries, cpuid->nent, 0xd, 0);
    if (entry)
        guest_supported_xcr0 = (entry->eax | ((u64)entry->edx << 32));
    else
        guest_supported_xcr0 = 0;
    guest_supported_xcr0 &= kvm_caps.supported_xcr0;

    entry = kvm_find_cpuid_entry2(cpuid->entries, cpuid->nent, 0xd, 1);
    if (entry)
        guest_supported_xss = (entry->ecx | ((u64)entry->edx << 32));
    else
        guest_supported_xss = 0;
    //如果 KVM 也配置支持 XSS
    /*
     * PT and CET can be exposed to TD guest regardless of KVM's XSS, PT
     * and, CET support.
     */
    guest_supported_xss &=
        (kvm_caps.supported_xss | XFEATURE_MASK_PT | TDX_TD_XFAM_CET);
    //根据用户态投喂给的 cpuid 0xd 的值计算得到 TD param 的 XFAM 值
    td_params->xfam = guest_supported_xcr0 | guest_supported_xss;
    if (td_params->xfam & XFEATURE_MASK_LBR) {
        /*
         * TODO: once KVM supports LBR(save/restore LBR related
         * registers around TDENTER), remove this guard.
         */
#define MSG_LBR "TD doesn't support LBR yet. KVM needs to save/restore IA32_LBR_DEPTH properly.\n"
        pr_warn(MSG_LBR);
        return -EOPNOTSUPP;
    }

    return 0;
}
```
* 由上面的分析可知 `cpuid->entries` 来自用户的输入，Qemu 会查看的 `cpuid -l 0xd -s 0` 和 `cpuid -l 0xd -s 1` 的支持情况
```sh
# cpuid -1 -l 0xd -s 0
CPU:
   XSAVE features (0xd/0):
      XCR0 valid bit field mask               = 0x00000000000602e7
         x87 state                            = true
         SSE state                            = true
         AVX state                            = true
         MPX BNDREGS                          = false
         MPX BNDCSR                           = false
         AVX-512 opmask                       = true
         AVX-512 ZMM_Hi256                    = true
         AVX-512 Hi16_ZMM                     = true
         PKRU state                           = true
         XTILECFG state                       = true
         XTILEDATA state                      = true
      bytes required by fields in XCR0        = 0x00002b00 (11008)
      bytes required by XSAVE/XRSTOR area     = 0x00002b00 (11008)
# cpuid -1 -l 0xd -s 1
CPU:
      XSAVEOPT instruction                    = true
      XSAVEC instruction                      = true
      XGETBV instruction                      = true
      XSAVES/XRSTORS instructions             = true
      XFD: extended feature disable supported = true
      SAVE area size in bytes                 = 0x00002d40 (11584)
      IA32_XSS valid bit field mask           = 0x000000000000dd00
         PT state                             = true
         PASID state                          = true
         CET_U user state                     = true
         CET_S supervisor state               = true
         HDC state                            = false
         UINTR state                          = true
         LBR state                            = true
         HWP state                            = false
```

## TDX Module 对于 XFAM 的处理

### 初始化 TD 的时候
* 设置 `TD.VMCS` 中执行控制域的的 `xfam`
```cpp
//src/vmm_dispatcher/tdx_seamcall_entry_point.S
tdx_seamcall_entry_point
   //src/vmm_dispatcher/tdx_vmm_dispatcher.c
-> tdx_vmm_dispatcher()
      //src/vmm_dispatcher/api_calls/tdh_mng_init.c
   -> tdh_mng_init()
         // Map the TD PARAMS address
         td_params_ptr = (td_params_t *)map_pa((void*)td_params_pa.raw, TDX_RANGE_RO);
         // Read the TD configuration input and set TDCS fields
      -> read_and_set_td_configurations(tdr_ptr, tdcs_ptr, td_params_ptr)
            // Read and verify XFAM
            tmp_xfam.raw = td_params_ptr->xfam;
            if (!check_xfam(tmp_xfam)) {
              return_val = api_error_with_operand_id(TDX_OPERAND_INVALID, OPERAND_ID_XFAM);
              goto EXIT;
            }
            tdcs_ptr->executions_ctl_fields.xfam = tmp_xfam.raw;
         // Handle CPUID Configuration
      -> read_and_set_cpuid_configurations(tdcs_ptr, td_params_ptr, global_data_ptr, local_data_ptr)
            xfam.raw = tdcs_ptr->executions_ctl_fields.xfam;
            for (cpuid_index = 0; cpuid_index < MAX_NUM_CPUID_LOOKUP; cpuid_index++) {
               // Start with the native CPUID value, collected on TDHSYSINIT
               final_tdcs_values.low = global_data_ptr->cpuid_values[cpuid_index].values.low;
               final_tdcs_values.high = global_data_ptr->cpuid_values[cpuid_index].values.high;
               if (cpuid_leaf_subleaf.leaf == CPUID_VER_INFO_LEAF) { //当 leaf = 0x1 时
                  // Leaf 0x1 has ECX bits configurable by AVX (XFAM[2]).
                  // If XFAM[2] is 0, the applicable bits are cleared.
                  if (!xfam.avx) //根据 XFAM AVX bit 的状态决定 $ecx 寄存器中受 XFAM[2] 影响的 bit 值
                     final_tdcs_values.ecx &= ~(xfam_mask_0x1_0xffffffff[2].ecx);
               }
               else if (cpuid_leaf_subleaf.leaf == CPUID_EXT_STATE_ENUM_LEAF)//当 leaf = 0xd 时
               {
                  if (cpuid_leaf_subleaf.subleaf == 0) { //当 subleaf = 0x0 时
                      apply_cpuid_xfam_masks(&final_tdcs_values, xfam.raw, xfam_mask_0xd_0x0);
                      final_tdcs_values.ecx = calculate_xsave_area_max_size(xfam);
                  }
                  else if (cpuid_leaf_subleaf.subleaf == 1) //当 subleaf = 0x1 时
                      apply_cpuid_xfam_masks(&final_tdcs_values, xfam.raw, xfam_mask_0xd_0x1);
               }
               // Write the CPUID values to TDCS and set the CPUID_VALID flag
               tdcs_ptr->cpuid_config_vals[cpuid_index].low = final_tdcs_values.low;
               tdcs_ptr->cpuid_config_vals[cpuid_index].high = final_tdcs_values.high;
               tdcs_ptr->executions_ctl_fields.cpuid_valid[cpuid_index] = cpuid_lookup[cpuid_index].valid_entry? !cpuid_lookup[cpuid_index].faulting : false;
            }
```

### 初始化 VCPU 的时候拷贝 XFAM
* 将 `XFAM` 的值从 `VMCS` 执行控制域拷贝至 TD per vCPU 的 `TDVPS`
```cpp
//src/vmm_dispatcher/api_calls/tdh_vp_init.c
tdh_vp_init()
   // Copy XFAM to TDVPS; in DEBUG mode the debugger is allowed to change it per VCPU
   tdvps_ptr->management.xfam = tdcs_ptr->executions_ctl_fields.xfam;
```

### TD Entry 时设置 Guest 的 XFAM
```cpp
//src/td_transitions/tdh_vp_enter.c
tdh_vp_enter()
   tdx_module_local_t  * local_data_ptr = get_local_data();
   local_data_ptr->vp_ctx.xfam = tdvps_ptr->management.xfam;
   local_data_ptr->vp_ctx.xfd_supported = tdcs_ptr->executions_ctl_fields.cpuid_flags.xfd_supported;
   // Restore other Guest state (GPRs, DRs, MSRs) in TDVPS
-> restore_guest_td_state_before_td_entry(tdcs_ptr, tdvps_ptr);
      // Restore CPU extended state, based on XFAM
   -> restore_guest_td_extended_state(tdcs_ptr, tdvps_ptr);
         // Set Guest XCR0 and XSS context for restoring the state
      -> ia32_xsetbv(0, tdvps_ptr->management.xfam & XCR0_USER_BIT_MASK); //设置 TDX module 上下文的 XCR0
      -> ia32_wrmsr(IA32_XSS_MSR_ADDR, tdvps_ptr->management.xfam & XCR0_SUPERVISOR_BIT_MASK);
      // Extended state control
   -> ia32_xsetbv(0, tdvps_ptr->guest_state.xcr0); //恢复 guest 上下文的 XCR0
   -> safe_wrmsr(IA32_XSS_MSR_ADDR, tdvps_ptr->guest_msr_state.ia32_xss, tdcs_ptr);
```

### TD Exit 时 XFAM 的处理
* TD Exit 时保存 Geust 的 XFAM 恢复 TDX module 的 XFAM
* `td_vmexit_to_vmm()` 是 TD-Exit 到 VMM 时调用的接口函数，两个调用者分别对应 *同步的 TD VMCALL 场景* 和 *TD 异步退出的场景*
```cpp
1 src/td_dispatcher/vm_exits/tdg_vp_vmcall.c|120| <<tdg_vp_vmcall>> td_vmexit_to_vmm(VCPU_READY, LAST_EXIT_TDVMCALL, scrub_mask, control.      xmm_select, false, true);
2 src/td_transitions/td_exit.c|403| <<async_tdexit_internal>> td_vmexit_to_vmm(vcpu_state, last_td_exit, scrub_mask, ...);
```
* 保存 guest 状态，恢复 TDX module 状态
```cpp
ia32_xcr0_t xcr0 = { .raw = tdx_local_data_ptr->vp_ctx.xfam };
xcr0.sse = 1;
uint64_t scrub_mask = xcr0.raw;
//tdx-module/src/td_transitions/td_exit.c
td_vmexit_to_vmm(..., scrub_mask, ...)
   // Local data
   tdx_module_local_t* tdx_local_data_ptr = get_local_data();
   // Avoid accessing TD private memory and control structures (except TDR).
   // The TD is dead, no need so save its state.
   if (!is_td_dead)
      // 1.  Save any guest state that it has not saved as part of the common guest-side operation, e.g.,
      //     the extended state per TDCS.XFAM
      -> save_guest_td_state_before_td_exit(tdcs_ptr, tdx_local_data_ptr);
            tdvps_t* tdvps_ptr = local_data_ptr->vp_ctx.tdvps;
            // Extended state control
            if (tdcs_ptr->executions_ctl_fields.cpuid_flags.xfd_supported)  {
               tdvps_ptr->guest_msr_state.ia32_xfd = ia32_rdmsr(IA32_XFD_MSR_ADDR);
               tdvps_ptr->guest_msr_state.ia32_xfd_err = ia32_rdmsr(IA32_XFD_ERROR_MSR_ADDR);
            }
            // XCR0 state。运行 TD 时的 XCR0 状态保存到 guest 状态域中
            tdvps_ptr->guest_state.xcr0 = ia32_xgetbv(0);
      // Save Guest-TD XCR0 and XSS
         //src/common/helpers/helpers.h
      -> save_guest_td_extended_state(tdvps_ptr, tdx_local_data_ptr->vp_ctx.xfam)
            // Set Guest XCR0 and XSS context for saving the state
         -> ia32_xsetbv(0, xfam & XCR0_USER_BIT_MASK); //设置 TDX module 上下文的 XFAM
         -> ia32_wrmsr(IA32_XSS_MSR_ADDR, xfam & XCR0_SUPERVISOR_BIT_MASK);
         -> ia32_xsaves(&tdvps_ptr->guest_extension_state.xbuf, xfam);
   // 5.  Load any host VMM state, which is not part of the output of TDHVPENTER, and that
   //     might have been modified by TDX-SEAM itself and by the guest TD, with synthetic INIT values.
   //
   // This is done before loading VMM state - to save additional XCR0/XSS save-restore
-> initialize_extended_state(scrub_mask)
   -> ia32_xsetbv(0, xfam & XCR0_USER_BIT_MASK); //设置 TDX module 上下文的 XFAM
   -> ia32_wrmsr(IA32_XSS_MSR_ADDR, xfam & XCR0_SUPERVISOR_BIT_MASK);
   -> ia32_xrstors(&get_global_data()->xbuf, xfam);
   // 6.  Restore host VMM CPU state from the TDX-SEAM LP-scope state save area.
   //     At this point, any flow that needed to update the host VMM state, e.g.,
   //     write to RDX as an output operand, has done so by updating the state save area.
-> load_vmm_state_before_td_exit(tdx_local_data_ptr)
      if (local_data_ptr->vp_ctx.xfd_supported) {
      -> init_msr_opt(IA32_XFD_MSR_ADDR, local_data_ptr->vp_ctx.tdvps->guest_msr_state.ia32_xfd);
      -> init_msr_opt(IA32_XFD_ERROR_MSR_ADDR, local_data_ptr->vp_ctx.tdvps->guest_msr_state.ia32_xfd_err);
      }
```

### SEAMRET 到 KVM 时 XFAM 的处理
* TDX Module `SEAMRET` 到 KVM 后，KVM 需要根据 host 的情况决定怎么恢复 `XCR0`
```cpp
//arch/x86/kvm/vmx/tdx.c
tdx_vcpu_run()
-> tdx_vcpu_enter_exit(tdx)
-> tdx_restore_host_xsave_state(vcpu)
```
* 如果仅当 TD 与 host `XCR0` 不一致时才恢复 `host_xcr0` 原来的值
  * arch/x86/kvm/vmx/tdx.c
```cpp
static void tdx_restore_host_xsave_state(struct kvm_vcpu *vcpu)
{
    struct kvm_tdx *kvm_tdx = to_kvm_tdx(vcpu->kvm);

    if (static_cpu_has(X86_FEATURE_XSAVE) &&
        host_xcr0 != (kvm_tdx->xfam & kvm_caps.supported_xcr0))
        xsetbv(XCR_XFEATURE_ENABLED_MASK, host_xcr0);
    if (static_cpu_has(X86_FEATURE_XSAVES) &&
        /* PT can be exposed to TD guest regardless of KVM's XSS support */
        host_xss != (kvm_tdx->xfam &
             (kvm_caps.supported_xss | XFEATURE_MASK_PT | TDX_TD_XFAM_CET)))
        wrmsrl(MSR_IA32_XSS, host_xss);
    if (static_cpu_has(X86_FEATURE_PKU) &&
        (kvm_tdx->xfam & XFEATURE_MASK_PKRU))
        write_pkru(vcpu->arch.host_pkru);
}
```

# TD Guest 枚举拓扑虚拟化
* 请先阅读 TDX Module Spec 相关章节

## x2APIC ID 的分配
* `CPUID(0xB)` 和 `CPUID(0x1F)` 只是提供了拓扑的信息，但具体到 `/proc/cpuinfo` 里看到的 `processor`、`core id`、`physical id` 和 `initial apicid` 最根本的源头来自 Qemu 分配的 x2APIC ID 以及用户配置的拓扑信息，算出以上数值

### Qemu 分配 x2APIC ID
* `x86_cpu_properties[]` 数组定义了 `X86CPU` 类型的属性（Property）和域（Field）的映射关系，比如，设置 `"apic-id"` 属性即设置 `X86CPU.apic_id` 域
* qemu/target/i386/cpu.c
```c
static Property x86_cpu_properties[] = {
#ifdef CONFIG_USER_ONLY
    /* apic_id = 0 by default for *-user, see commit 9886e834 */
    DEFINE_PROP_UINT32("apic-id", X86CPU, apic_id, 0),
    DEFINE_PROP_INT32("thread-id", X86CPU, thread_id, 0),
    DEFINE_PROP_INT32("core-id", X86CPU, core_id, 0),
    DEFINE_PROP_INT32("die-id", X86CPU, die_id, 0),
    DEFINE_PROP_INT32("socket-id", X86CPU, socket_id, 0),
#else
    DEFINE_PROP_UINT32("apic-id", X86CPU, apic_id, UNASSIGNED_APIC_ID),
    DEFINE_PROP_INT32("thread-id", X86CPU, thread_id, -1),
    DEFINE_PROP_INT32("core-id", X86CPU, core_id, -1),
    DEFINE_PROP_INT32("die-id", X86CPU, die_id, -1),
    DEFINE_PROP_INT32("socket-id", X86CPU, socket_id, -1),
#endif
    DEFINE_PROP_INT32("node-id", X86CPU, node_id, CPU_UNSET_NUMA_NODE_ID),
    DEFINE_PROP_BOOL("pmu", X86CPU, enable_pmu, false),
...
}
```
* 当初始化 vCPU 时有以下路径 `possible_cpus = mc->possible_cpu_arch_ids(ms)` 即 `x86_possible_cpu_arch_ids()`
```c
(gdb) bt
#0  x86_possible_cpu_arch_ids (ms=0x5555570fc990) at ../hw/i386/x86.c:496
    possible_cpus = mc->possible_cpu_arch_ids(ms)
#1  0x0000555555b70f99 in x86_cpus_init (x86ms=x86ms@entry=0x5555570fc990, default_cpu_version=1) at ../hw/i386/x86.c:155
#2  0x0000555555b7e527 in pc_q35_init (machine=0x5555570fc990) at ../hw/i386/pc_q35.c:183
#3  0x000055555591045b in machine_run_board_init (machine=<optimized out>, mem_path=<optimized out>, errp=<optimized out>,
    errp@entry=0x555556d85f78 <error_fatal>) at ../hw/core/machine.c:1546
#4  0x0000555555ac2b86 in qemu_init_board () at ../system/vl.c:2614
#5  qmp_x_exit_preconfig (errp=<optimized out>) at ../system/vl.c:2705
#6  0x0000555555ac68f0 in qmp_x_exit_preconfig (errp=<optimized out>) at ../system/vl.c:2700
#7  qemu_init (argc=<optimized out>, argv=<optimized out>) at ../system/vl.c:3737
#8  0x000055555587bc89 in main (argc=<optimized out>, argv=<optimized out>) at ../system/main.c:47
```
* 在这个函数里 Qemu 将要分配的 x2APIC ID 确定了 `ms->possible_cpus->cpus[i].arch_id`
```c
static void init_topo_info(X86CPUTopoInfo *topo_info,
                           const X86MachineState *x86ms)
{
    MachineState *ms = MACHINE(x86ms);

    topo_info->dies_per_pkg = ms->smp.dies;
    topo_info->cores_per_die = ms->smp.cores;
    topo_info->threads_per_core = ms->smp.threads;
}
...
/*
 * Calculates initial APIC ID for a specific CPU index
 *
 * Currently we need to be able to calculate the APIC ID from the CPU index
 * alone (without requiring a CPU object), as the QEMU<->Seabios interfaces have
 * no concept of "CPU index", and the NUMA tables on fw_cfg need the APIC ID of
 * all CPUs up to max_cpus.
 */
uint32_t x86_cpu_apic_id_from_index(X86MachineState *x86ms,
                                    unsigned int cpu_index)
{
    X86CPUTopoInfo topo_info;
    //好吧，又来一遍，因为该函数其他调用者不需要 topo_info
    init_topo_info(&topo_info, x86ms);
    //根据用户配置的拓扑和 CPU 索引，构建 initial APIC ID
    return x86_apicid_from_cpu_idx(&topo_info, cpu_index);
}
...
const CPUArchIdList *x86_possible_cpu_arch_ids(MachineState *ms)
{
    X86MachineState *x86ms = X86_MACHINE(ms);
    unsigned int max_cpus = ms->smp.max_cpus;
    X86CPUTopoInfo topo_info;
    int i;
    //初始化后 ms->possible_cpus->len 和 max_cpus 必须一致
    if (ms->possible_cpus) {
        /*
         * make sure that max_cpus hasn't changed since the first use, i.e.
         * -smp hasn't been parsed after it
         */
        assert(ms->possible_cpus->len == max_cpus);
        return ms->possible_cpus;
    }
    //初始化时，分配 ms->possible_cpus 的空间
    ms->possible_cpus = g_malloc0(sizeof(CPUArchIdList) +
                                  sizeof(CPUArchId) * max_cpus);
    ms->possible_cpus->len = max_cpus;
    //根据用户配置设置本地变量 topo_info 的 thread、core、die 信息
    init_topo_info(&topo_info, x86ms);
    //逐个初始化 possible_cpus.cpu[] 数组的信息
    for (i = 0; i < ms->possible_cpus->len; i++) {
        X86CPUTopoIDs topo_ids;

        ms->possible_cpus->cpus[i].type = ms->cpu_type;
        ms->possible_cpus->cpus[i].vcpus_count = 1;
        ms->possible_cpus->cpus[i].arch_id =      //根据 vCPU 索引得到 apic_id
            x86_cpu_apic_id_from_index(x86ms, i); //比如 cores=90,sockets=2 时，i=90 的 apicid 为 128，二进制 1000_0000
        x86_topo_ids_from_apicid(ms->possible_cpus->cpus[i].arch_id,
                                 &topo_info, &topo_ids); //根据用户配置的拓扑和 APIC ID 计算出 thread/core/package IDs
        ms->possible_cpus->cpus[i].props.has_socket_id = true;
        ms->possible_cpus->cpus[i].props.socket_id = topo_ids.pkg_id;
        if (ms->smp.dies > 1) {
            ms->possible_cpus->cpus[i].props.has_die_id = true;
            ms->possible_cpus->cpus[i].props.die_id = topo_ids.die_id;
        }
        ms->possible_cpus->cpus[i].props.has_core_id = true;
        ms->possible_cpus->cpus[i].props.core_id = topo_ids.core_id;
        ms->possible_cpus->cpus[i].props.has_thread_id = true;
        ms->possible_cpus->cpus[i].props.thread_id = topo_ids.smt_id;
    }
    return ms->possible_cpus;
}
```
* 上面用到的最关键的两个函数来自 qemu/include/hw/i386/topology.h
```c
/*
 * APIC IDs can be 32-bit, but beware: APIC IDs > 255 require x2APIC support
 */
typedef uint32_t apic_id_t;

typedef struct X86CPUTopoIDs {
    unsigned pkg_id;
    unsigned die_id;
    unsigned core_id;
    unsigned smt_id;
} X86CPUTopoIDs;

typedef struct X86CPUTopoInfo {
    unsigned dies_per_pkg;
    unsigned cores_per_die;
    unsigned threads_per_core;
} X86CPUTopoInfo;

/* Return the bit width needed for 'count' IDs */
static unsigned apicid_bitwidth_for_count(unsigned count)
{
    g_assert(count >= 1); //每个 domain 必然至少有 1 个
    count -= 1; //因为最终要提供的是一个位宽，所以 -1，比如 90 - 1 = 89
    return count ? 32 - clz32(count) : 0; //32 -（32-bit 值中的前导零的个数）即为对齐到 2 次幂 -1 的掩码的位宽
} //比如 clz32(90) = 25，返回值就是 7
//以下三个函数返回各个 domain 的位宽
/* Bit width of the SMT_ID (thread ID) field on the APIC ID */
static inline unsigned apicid_smt_width(X86CPUTopoInfo *topo_info)
{
    return apicid_bitwidth_for_count(topo_info->threads_per_core);
}

/* Bit width of the Core_ID field */
static inline unsigned apicid_core_width(X86CPUTopoInfo *topo_info)
{
    return apicid_bitwidth_for_count(topo_info->cores_per_die);
}

/* Bit width of the Die_ID field */
static inline unsigned apicid_die_width(X86CPUTopoInfo *topo_info)
{
    return apicid_bitwidth_for_count(topo_info->dies_per_pkg);
}
//以下三个函数返回各个 domain 在 xAPIC ID 中的偏移
/* Bit offset of the Core_ID field */
static inline unsigned apicid_core_offset(X86CPUTopoInfo *topo_info)
{ //core ID 的偏移是在 thread 位宽的基础上得到的
    return apicid_smt_width(topo_info);
}

/* Bit offset of the Die_ID field */
static inline unsigned apicid_die_offset(X86CPUTopoInfo *topo_info)
{ //die ID 的偏移是在 core 位宽 + 偏移的基础上得到的
    return apicid_core_offset(topo_info) + apicid_core_width(topo_info);
}

/* Bit offset of the Pkg_ID (socket ID) field */
static inline unsigned apicid_pkg_offset(X86CPUTopoInfo *topo_info)
{ //package ID 的偏移是在 die 位宽 + 偏移的基础上得到的
    return apicid_die_offset(topo_info) + apicid_die_width(topo_info);
}
//根据组装好的 topo_info 得到 APIC ID
/*
 * Make APIC ID for the CPU based on Pkg_ID, Core_ID, SMT_ID
 *
 * The caller must make sure core_id < nr_cores and smt_id < nr_threads.
 */
static inline apic_id_t x86_apicid_from_topo_ids(X86CPUTopoInfo *topo_info,
                                                 const X86CPUTopoIDs *topo_ids)
{ //比如 cores=90,sockets=2 时，cpu_index=90 的 apicid 为 128，二进制 1000_0000
    return (topo_ids->pkg_id  << apicid_pkg_offset(topo_info)) |
           (topo_ids->die_id  << apicid_die_offset(topo_info)) |
           (topo_ids->core_id << apicid_core_offset(topo_info)) |
           topo_ids->smt_id;
} //比如 cores=180,sockets=1 时，cpu_index=90 的 apicid 为 90，二进制 0101_1010
//根据用户配置的拓扑信息和 CPU 索引得到 top_ids
static inline void x86_topo_ids_from_idx(X86CPUTopoInfo *topo_info,
                                         unsigned cpu_index,
                                         X86CPUTopoIDs *topo_ids)
{   //用户配置的拓扑信息
    unsigned nr_dies = topo_info->dies_per_pkg;
    unsigned nr_cores = topo_info->cores_per_die;
    unsigned nr_threads = topo_info->threads_per_core;
    //当配置 180 个 LPs 时，为什么 cores=90,sockets=2 时 initial apicid 会分段，cores=180,sockets=1 时是连续的，原因就在这里！！！
    topo_ids->pkg_id = cpu_index / (nr_dies * nr_cores * nr_threads); //在第几个 package？比如 cores=90,sockets=2 时，cpu_index=90 的 pkg_id 为 1
    topo_ids->die_id = cpu_index / (nr_cores * nr_threads) % nr_dies; //在第几个 die？
    topo_ids->core_id = cpu_index / nr_threads % nr_cores; //在第几个 core？比如 cores=90,sockets=2 时，cpu_index=90 的 core_id 为 0
    topo_ids->smt_id = cpu_index % nr_threads; //在 core 上的第几个 HT？
}
//根据用户配置的拓扑和 APIC ID 计算出 thread/core/package IDs
/*
 * Calculate thread/core/package IDs for a specific topology,
 * based on APIC ID
 */
static inline void x86_topo_ids_from_apicid(apic_id_t apicid,
                                            X86CPUTopoInfo *topo_info,
                                            X86CPUTopoIDs *topo_ids)
{
    topo_ids->smt_id = apicid &
            ~(0xFFFFFFFFUL << apicid_smt_width(topo_info));
    topo_ids->core_id =
            (apicid >> apicid_core_offset(topo_info)) &
            ~(0xFFFFFFFFUL << apicid_core_width(topo_info));
    topo_ids->die_id =
            (apicid >> apicid_die_offset(topo_info)) &
            ~(0xFFFFFFFFUL << apicid_die_width(topo_info));
    topo_ids->pkg_id = apicid >> apicid_pkg_offset(topo_info);
}
//根据用户配置的拓扑和 CPU 索引，得到 APIC ID
/*
 * Make APIC ID for the CPU 'cpu_index'
 *
 * 'cpu_index' is a sequential, contiguous ID for the CPU.
 */
static inline apic_id_t x86_apicid_from_cpu_idx(X86CPUTopoInfo *topo_info,
                                                unsigned cpu_index)
{
    X86CPUTopoIDs topo_ids;
    x86_topo_ids_from_idx(topo_info, cpu_index, &topo_ids); //输入是用户配置的拓扑和 CPU 索引，输出是组装好的 top_ids
    return x86_apicid_from_topo_ids(topo_info, &topo_ids); //根据组装好的 topo_info 得到 apic_id，并返回
}
```
* 随后 `x86_cpus_init()` 会调用 `x86_cpu_new()` 设置 `"apic-id"` 属性即设置 `X86CPU.apic_id` 域
```c
void x86_cpu_new(X86MachineState *x86ms, int64_t apic_id, Error **errp)
{
    Object *cpu = object_new(MACHINE(x86ms)->cpu_type);
    //设置 "apic-id" 属性即设置 X86CPU.apic_id 域
    if (!object_property_set_uint(cpu, "apic-id", apic_id, errp)) {
        goto out;
    }
    qdev_realize(DEVICE(cpu), NULL, errp);

out:
    object_unref(cpu);
}
```

#### 当配置 180 个 LPs，`cores=90,sockets=2` 时 `initial apicid` 会分段，`cores=180,sockets=1` 时是连续的
* `cores=90,sockets=2`，`0~89` LP 在 package `0`，`90~179` LP 在 package `1`，`180` 个 LP 在 `2` 个 sockets 上平均分布
  * `initial apicid` 跳变的源头就是上面分析的 Qemu 的代码
```sh
# grep -e processor -e "core id" -e "physical id" -e "initial apicid" /proc/cpuinfo
...
processor       : 89
physical id     : 0
core id         : 44
initial apicid  : 89

processor       : 90
physical id     : 1   # 结果：package id 从这里开始跳变
core id         : 0
initial apicid  : 128 # 原因：initial apicid 从这里开始跳变
...
```
* `cores=180,sockets=1`，`0~127` LP 在 package `0`，`128~179` LP 在 package `1`，尽管只配了一个 socket，`180` 个 LP 在 `2` 个 sockets 上不均匀分布
```sh
# grep -e processor -e "core id" -e "physical id" -e "initial apicid" /proc/cpuinfo
...
processor       : 127
physical id     : 0
core id         : 63
initial apicid  : 127

processor       : 128
physical id     : 1   # 结果 2：physical id 从这里跳变
core id         : 0   # 结果 2：core 从这里跳变
initial apicid  : 128 # 原因：initial apicid 这里是连续的，没有跳变；且 CPUID.(EAX=0x1F,ECX=1):EAX[04-00] == 0x7
...
```
* 这应该是 guest kernel 在 `detect_extended_topology()` 时计算的结果导致的
```c
//对于 BSP 是以下路径
start_kernel()
-> arch_cpu_finalize_init()
   -> identify_boot_cpu()
      -> identify_cpu(&boot_cpu_data)
         -> generic_identify()
            -> cpu_detect(c)
            c->apicid = apic->phys_pkg_id(c->initial_apicid, 0)
         -> this_cpu->c_init(c)
         => init_intel()
            -> detect_extended_topology_leaf()
         -> detect_ht()
            -> detect_ht_early()
-> arch_call_rest_init()
   -> rest_init()
      -> user_mode_thread(kernel_init, NULL, CLONE_FS);
                              |
     +------------------------+
     |
     v
kernel_init()
-> kernel_init_freeable()
   -> smp_init()
      -> pr_info("Bringing up secondary CPUs ...\n")
      -> bringup_nonboot_cpus(setup_max_cpus)
      -> pr_info("Brought up %d node%s, %d CPU%s\n", ...)
      -> smp_cpus_done(setup_max_cpus)
         -> smp_ops.smp_cpus_done(max_cpus)
         => native_smp_cpus_done()
            -> calculate_max_logical_packages()
               -> pr_info("Max logical packages: %u\n", __max_logical_packages)
            -> impress_friends()
               -> pr_info("Total of %d processors activated (%lu.%02lu BogoMIPS)\n", ...)

//对于 AP 是以下路径
start_secondary()
-> ap_starting()
   -> smp_store_cpu_info()
      -> identify_secondary_cpu(c)
         -> identify_cpu(c)
            -> generic_identify()
               -> cpu_detect(c)
               c->apicid = apic->phys_pkg_id(c->initial_apicid, 0)
            -> this_cpu->c_init(c)
            => init_intel()
               -> early_init_intel()
                     if (detect_extended_topology_early(c) < 0)
                         -> detect_extended_topology_leaf(c)
                            -> check_extended_topology_leaf(0x1f)
                            -> check_extended_topology_leaf(0xb)
                     -> detect_ht_early(c)
               -> detect_extended_topology_leaf()
            -> detect_ht()
               -> detect_ht_early()
      -> validate_apic_and_package_id()
         -> topology_update_package_map(c->phys_proc_id, cpu)
         -> topology_update_die_map(c->cpu_die_id, cpu)
            -> topology_phys_to_logical_die(die, cpu)
            -> pr_info("CPU %u Converting physical %u to logical die %u\n", cpu, die, new)

//cat /proc/cpuinfo
show_cpuinfo_core()
```
* 在一台 `2` packages，每个 package `48` 个 cores，支持超线程的机器上，总共有 `2 x 48 x 2 = 192` 个 logical processors。在 guest 中执行 `cpuid -1 -l 0x1f` 命令：
  * **注意**：这里 Qemu 没有配置 `dies > 2`，因此 guest `CPUID(0x1F)` 用的是 native 的值，而不是根据 Qemu 配置的拓扑制作的值
```sh
# cpuid -1 -l 0x1f -s 0
CPU:
   V2 extended topology (0x1f):
      x2APIC ID of logical processor = 0x94 (148)
      --- level 0 ---
      level number                          = 0x0 (0)    # 输入参数 -s 0，ECX[07-00] 的值
      level type                            = thread (1) # ECX[15-08] 的值为 1
      bit width of level & previous levels  = 0x1 (1)    # EAX[04-00] 的值为 1，x2APIC ID 必须向右移动的位数
      number of logical processors at level = 0x2 (2)    # EBX[15-00] 的值为 2，对 core domain 来说有 2 个 logical processors
# cpuid -1 -l 0x1f -s 0 -r
CPU:
   0x0000001f 0x00: eax=0x00000001 ebx=0x00000002 ecx=0x00000100 edx=0x0000000b

# cpuid -1 -l 0x1f -s 1
CPU:
      --- level 1 ---
      level number                          = 0x1 (1)   # 输入参数 -s 1，ECX[07-00] 的值为 1
      level type                            = core (2)  # ECX[15-08] 的值为 2
      bit width of level & previous levels  = 0x7 (7)   # EAX[04-00] 的值为 7，x2APIC ID 必须向右移动的位数
      number of logical processors at level = 0x60 (96) # EBX[15-00] 的值为 0x60，对 package domain 来说有 96 个 logical processors
# cpuid -1 -l 0x1f -s 1 -r
CPU:
   0x0000001f 0x01: eax=0x00000007 ebx=0x00000060 ecx=0x00000201 edx=0x00000082
```
* 因为 initial apicid 是连续的，没有跳变；且 `CPUID.(EAX=0x1F,ECX=1):EAX[04-00] == 0x7`，在 `detect_extended_topology()` 计算的时候，对于 x2APIC ID 为 `128` 的 vCPU
  * `ht_mask_width` 是 `1`，即 x2APIC ID 右移 `1` 位并和 `core_select_mask` 进行 **与运算** 可以得到 `core id`
  * `pkg_mask_width` 是 `7`，即 x2APIC ID 右移 `7` 位得到 `physical id`，因此尽管配置是 `cores=180,sockets=1`，**依然被划分成了两个 packages**
  * `core_select_mask` 的计算公式为  `core_select_mask = (~(-1 << pkg_mask_width)) >> ht_mask_width`
  * `bit 0` 作为 core domain 下的第几个 LP 的索引
  * `bit 6~1` 作为 package domain 下的第几个 core 的索引。对于 xAPIC ID `127` 而言就是 `63`，对于 xAPIC ID `128` 而言就是 `0`
  * `bit 7+` 作为 package 的索引。对于 xAPIC ID `127` 而言就是 `0`，对于 xAPIC ID `128` 而言就是 `1`
* 最后看到的就是 `sockets=1` 时看到的 LPs 不均匀分布在 packages 上的情况，尽管 x2APIC ID 在 Qemu --> KVM --> TDX Module --> Guest kernel 传递的通道是正常工作的。

### Qemu 创建 vCPU 时的传递含义 x2APIC ID 的信息
* Qemu 创建 vCPU 线程时传递 `CPUState *cpu` 的指针包含之前组装好的数据 
```c
(gdb) bt
#0  kvm_start_vcpu_thread (cpu=0x555557249a80) at ../accel/kvm/kvm-accel-ops.c:66
    cpus_accel->create_vcpu_thread(cpu)
#1  0x0000555555ab5a6c in qemu_init_vcpu (cpu=cpu@entry=0x555557249a80) at ../system/cpus.c:669
#2  0x0000555555baebdf in x86_cpu_realizefn (dev=0x555557249a80, errp=0x7fffffffce40) at ../target/i386/cpu.c:7431
#3  0x0000555555cdf8bb in device_set_realized (obj=<optimized out>, value=<optimized out>, errp=0x7fffffffcec0) at ../hw/core/qdev.c:510
#4  0x0000555555ce3f16 in property_set_bool(obj=0x555557249a80, v=<optimized out>, name=<optimized out>, opaque=0x555556e9ade0, errp=0x7fffffffcec0)
    at ../qom/object.c:2358
#5  0x0000555555ce7398 in object_property_set (obj=obj@entry=0x555557249a80, name=name@entry=0x555556004522 "realized", v=v@entry=    0x5555572521d0, errp=0x7fffffffcec0, errp@entry=0x555556d85f78 <error_fatal>)
    at ../qom/object.c:1472
#6  0x0000555555ceacf0 in object_property_set_qobject(obj=obj@entry=0x555557249a80, name=name@entry=0x555556004522 "realized", value=value@entry=0x555556e98c70, errp=errp@entry=0x555556d85f78 <error_fatal>) at ../qom/qom-qobject.c:28
#7  0x0000555555ce7a55 in object_property_set_bool(obj=0x555557249a80, name=name@entry=0x555556004522 "realized", value=value@entry=true, errp=errp@entry=0x555556d85f78 <error_fatal>)
    at ../qom/object.c:1541
#8  0x0000555555ce027e in qdev_realize (dev=<optimized out>, bus=bus@entry=0x0, errp=errp@entry=0x555556d85f78 <error_fatal>)
    at ../hw/core/qdev.c:292
#9  0x0000555555b70ed1 in x86_cpu_new (x86ms=<optimized out>, apic_id=0, errp=0x555556d85f78 <error_fatal>) at ../hw/i386/x86.c:106
#10 0x0000555555b70fce in x86_cpus_init (x86ms=x86ms@entry=0x5555570fc990, default_cpu_version=<optimized out>) at ../hw/i386/x86.c:157
#11 0x0000555555b7e527 in pc_q35_init (machine=0x5555570fc990) at ../hw/i386/pc_q35.c:183
#12 0x000055555591045b in machine_run_board_init (machine=<optimized out>, mem_path=<optimized out>, errp=<optimized out>,    errp@entry=0x555556d85f78 <error_fatal>)
    at ../hw/core/machine.c:1546
#13 0x0000555555ac2b86 in qemu_init_board () at ../system/vl.c:2614
#14 qmp_x_exit_preconfig (errp=<optimized out>) at ../system/vl.c:2705
#15 0x0000555555ac68f0 in qmp_x_exit_preconfig (errp=<optimized out>) at ../system/vl.c:2700
#16 qemu_init (argc=<optimized out>, argv=<optimized out>) at ../system/vl.c:3737
#17 0x000055555587bc89 in main (argc=<optimized out>, argv=<optimized out>) at ../system/main.c:47
(gdb)
```
* vCPU 线程函数是 `kvm_vcpu_thread_fn()`，参数是 `CPUState *cpu` 类型的指针
```c
static void kvm_start_vcpu_thread(CPUState *cpu)
{
    char thread_name[VCPU_THREAD_NAME_SIZE];

    cpu->thread = g_malloc0(sizeof(QemuThread));
    cpu->halt_cond = g_malloc0(sizeof(QemuCond));
    qemu_cond_init(cpu->halt_cond);
    snprintf(thread_name, VCPU_THREAD_NAME_SIZE, "CPU %d/KVM",
             cpu->cpu_index);
    qemu_thread_create(cpu->thread, thread_name, kvm_vcpu_thread_fn,
                       cpu, QEMU_THREAD_JOINABLE);
}
```
* 创建 vCPU 时 Qemu 调用 `ioctl(KVM_CREATE_VCPU)` 的调用路径如下：
```c
//qemu/accel/kvm/kvm-accel-ops.c
kvm_vcpu_thread_fn(void *arg)
   CPUState *cpu = arg;
   //qemu/accel/kvm/kvm-all.c
-> kvm_init_vcpu(cpu, &error_fatal)
   -> trace_kvm_init_vcpu(cpu->cpu_index, kvm_arch_vcpu_id(cpu))
   -> kvm_get_vcpu(s, kvm_arch_vcpu_id(cpu))
      -> kvm_arch_vcpu_id(cpu)
         -> X86CPU *cpu = X86_CPU(cs);
         -> return cpu->apic_id;  //kvm_arch_vcpu_id() 的返回值就是 vcpu_id
      -> kvm_vm_ioctl(s, KVM_CREATE_VCPU, (void *)vcpu_id)
```
* 内核处理 `ioctl(KVM_CREATE_VCPU)` 和 `ioctl(KVM_MEMORY_ENCRYPT_OP)` 的调用路径如下：
```cpp
//virt/kvm/kvm_main.c
kvm_vm_ioctl(struct file *filp, unsigned int ioctl, unsigned long arg)
   case KVM_CREATE_VCPU:
   -> kvm_vm_ioctl_create_vcpu(kvm, arg) //arg 就是 Qemu 传过来的 vcpu_id 了
      -> kvm_vcpu_init(vcpu, kvm, id)    //id 就是 arg，即 vcpu_id
            vcpu->vcpu_id = id;          //就在这里把 Qemu 构造好的 apic_id 传给 KVM 了！！！
      -> create_vcpu_fd(vcpu)
         -> snprintf(name, sizeof(name), "kvm-vcpu:%d", vcpu->vcpu_id)
         -> return anon_inode_getfd(name, &kvm_vcpu_fops, vcpu, O_RDWR | O_CLOEXEC)
   default:
      //arch/x86/kvm/x86.c
   -> kvm_arch_vm_ioctl(filp, ioctl, arg)
      case KVM_MEMORY_ENCRYPT_OP:
      -> static_call(kvm_x86_mem_enc_ioctl)(kvm, argp)
         //arch/x86/kvm/vmx/main.c
      => vt_vcpu_mem_enc_ioctl()
            //arch/x86/kvm/vmx/tdx.c
         -> tdx_vcpu_ioctl(vcpu, argp)
            -> tdx_td_vcpu_init(vcpu, (u64)cmd.data)
                  //arch/x86/kvm/vmx/tdx_ops.h
               -> tdh_vp_init_apicid(tdx->tdvpr_pa, vcpu_rcx, vcpu->vcpu_id) //这个函数见后面详解
```
* 调用 `ioctl(KVM_MEMORY_ENCRYPT_OP)` 的最后调用的 `tdh_vp_init_apicid(..., vcpu->vcpu_id)` 是新加的，其参数 `vcpu->vcpu_id` 正是来自之前 `ioctl(KVM_CREATE_VCPU)` 传递的 vCPU 的 x2APIC ID

## VMM 支持拓扑枚举虚拟化

### Qemu 支持 TD 拓扑枚举虚拟化
* https://github.com/intel-sandbox/fyin.qemu-64K/commit/2ad0aebfcc1e86f34d77697ce453150dbf7c1473

### KVM 支持 TD 拓扑枚举虚拟化
* 目前只有一个参考实现：[KVM: X86: TDX: Set x2apic id for TD Guest](https://github.com/intel/tdx/commit/3e5d13b368754)
* 使用 `TOPOLOGY_ENUM` 功能，VMM 可以为每个 TD vCPU 设置 x2APIC ID。
* 支持此功能。检查 TDX `FEATURES0` 是否支持此功能，如果支持，则传递 `x2apicid`。
---
* 引入新函数 `tdh_vp_init_apicid()` 用 v1 版本的 seamcall `TDH_VP_INIT`，可以用寄存器 `r8` 传递 x2APIC ID
  * arch/x86/kvm/vmx/tdx_ops.h
```c
//在 arch/x86/kvm/vmx/tdx_arch.h 中引入
#define TDX_VERSION_SHIFT              16
#define MD_FIELD_ID_FEATURES0_TOPOLOGY_ENUM BIT_ULL(20)

static inline u64 tdh_vp_init_apicid(hpa_t tdvpr, u64 rcx, u32 x2apicid)
{
    struct tdx_module_args in = {
        .rcx = tdvpr,
        .rdx = rcx,
        .r8 = x2apicid,
    };

    /* apicid requires version == 1. */
    return tdx_seamcall(TDH_VP_INIT | (1ULL << TDX_VERSION_SHIFT), &in,
                NULL);
}
```
* 在 VMM 调用 seamcall `TDH_VP_INIT` 的时候，如果 TDX 支持拓扑枚举虚拟化，则采用 v1 接口
```diff
--- a/arch/x86/kvm/vmx/tdx.c
+++ b/arch/x86/kvm/vmx/tdx.c
@@ -4218,7 +4218,10 @@ static int tdx_td_vcpu_init(struct kvm_vcpu *vcpu, u64 vcpu_rcx)
                tdx_account_ctl_page(vcpu->kvm);
        }

-       err = tdh_vp_init(tdx->tdvpr_pa, vcpu_rcx);
+       if (tdx_info->features0 & MD_FIELD_ID_FEATURES0_TOPOLOGY_ENUM)
+               err = tdh_vp_init_apicid(tdx->tdvpr_pa, vcpu_rcx, vcpu->vcpu_id);
+       else
+               err = tdh_vp_init(tdx->tdvpr_pa, vcpu_rcx);
        if (KVM_BUG_ON(err, vcpu->kvm)) {
                pr_tdx_error(TDH_VP_INIT, err, NULL);
                return -EIO;
```
* `tdh_vp_init_apicid()` 的第三个参数 `vcpu->vcpu_id` 就是 Qemu 根据用户配置的拓扑构造好的 x2APIC ID，来龙去脉见之前 [Qemu 创建 vCPU 时的传递含义 x2APIC ID 的信息](#qemu-创建-vcpu-时的传递含义-x2apic-id-的信息) 一节

## TDX Module 支持 TD 拓扑枚举虚拟化

### TDX Module 中对扩展拓扑枚举 `CPUID(0xB)` 和 V2 `CPUID(0x1F)` 的定义
* **注意**：按照 TDX module spec 规定，扩展拓扑枚举 `CPUID(0xB)` 需派生至 V2 扩展拓扑枚举 `CPUID(0x1F)`，因此 TDX 的 `CPUID(0xB)` 并不是 VMM 可配置的。
* 例如，在 `cpuid_virtualization.js` 文件中的规定：
```json
{
   "Leaf": "0xb / Sub-Leaves 0x0-0x1",
   "EAX": [
     {
       "MSB": "4",
       "LSB": "0",
       "Field Size": "5",
       "Field Name": "Shift Count",
       "Configuration Details": null,
       "Bit or Field Virtualization Type": "Special",
       "Virtualization Details": "0"
     },
     {
       "MSB": "31",
       "LSB": "5",
       "Field Size": "27",
       "Field Name": "Reserved",
       "Configuration Details": null,
       "Bit or Field Virtualization Type": "Fixed",
       "Virtualization Details": "0"
     }
   ],
   ...
}
```
* 虽然在 include/auto_gen/cpuid_configurations.c 中的 `cpuid_lookup[]` 数组有 `0xB` 和 `0x1F` 相关元素，但 `cpuid_configurable[]` 数组仅有 `0x1F` 的相关元素，`0xB` 则没有，因此，这也意味着它不会出现在 KVM 的 `tdx_info->cpuid_configs[]` 数组中
```cpp
/**
 *  This File is Automatically generated by the TDX xls extract tool
 *  Spreadsheet Format Version - '8'
 **/

#include "auto_gen/cpuid_configurations.h"

// Lookup table first MAX_NUM_CPUID_CONFIG entries will be in the same order as in cpuid_configurable table

const cpuid_lookup_t cpuid_lookup[MAX_NUM_CPUID_LOOKUP] = {
...
 [64] = { .leaf_subleaf = {.leaf = 0xb, .subleaf = 0x0},
  .valid_entry = true,
  .fixed1 = { .eax = 0x0, .ebx = 0x0, .ecx = 0x0 },
  .fixed0_or_dynamic = { .eax = 0xffffffe0, .ebx = 0xffff0000, .ecx = 0xffff0000, .edx = 0xffffffff },
  .config_index = CPUID_CONFIG_NULL_IDX
  },
 [65] = { .leaf_subleaf = {.leaf = 0xb, .subleaf = 0x1},
  .valid_entry = true,
  .fixed1 = { .eax = 0x0, .ebx = 0x0, .ecx = 0x0 },
  .fixed0_or_dynamic = { .eax = 0xffffffe0, .ebx = 0xffff0000, .ecx = 0xffff0000, .edx = 0xffffffff },
  .config_index = CPUID_CONFIG_NULL_IDX
  },
 [66] = { .leaf_subleaf = {.leaf = 0xb, .subleaf = 0x2},
  .valid_entry = true,
  .fixed1 = { .eax = 0x0, .ebx = 0x0, .ecx = 0x0 },
  .fixed0_or_dynamic = { .eax = 0xffffffff, .ebx = 0xffffffff, .ecx = 0xffffff00, .edx = 0xffffffff },
  .config_index = CPUID_CONFIG_NULL_IDX
  },
...
 [46] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x0},
  .valid_entry = true,
  .verify_same = { .eax = 0xffffffff, .ebx = 0xffffffff, .ecx = 0xffffffff },
  .fixed1 = { .eax = 0x0, .ebx = 0x0, .ecx = 0x0 },
  .fixed0_or_dynamic = { .eax = 0xffffffe0, .ebx = 0xffff0000, .ecx = 0xffff0000, .edx = 0xffffffff },
  .config_index = 12
  },
 [47] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x1},
  .valid_entry = true,
  .verify_same = { .eax = 0xffffffff, .ebx = 0xffffffff, .ecx = 0xffffffff },
  .fixed1 = { .eax = 0x0, .ebx = 0x0, .ecx = 0x0 },
  .fixed0_or_dynamic = { .eax = 0xffffffe0, .ebx = 0xffff0000, .ecx = 0xffff0000, .edx = 0xffffffff },
  .config_index = 13
  },
 [48] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x2},
  .valid_entry = true,
  .verify_same = { .eax = 0xffffffff, .ebx = 0xffffffff, .ecx = 0xffffffff },
  .fixed1 = { .eax = 0x0, .ebx = 0x0, .ecx = 0x0 },
  .fixed0_or_dynamic = { .eax = 0xffffffe0, .ebx = 0xffff0000, .ecx = 0xffff0000, .edx = 0xffffffff },
  .config_index = 14
  },
 [49] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x3},
  .valid_entry = true,
  .verify_same = { .eax = 0xffffffff, .ebx = 0xffffffff, .ecx = 0xffffffff },
  .fixed1 = { .eax = 0x0, .ebx = 0x0, .ecx = 0x0 },
  .fixed0_or_dynamic = { .eax = 0xffffffe0, .ebx = 0xffff0000, .ecx = 0xffff0000, .edx = 0xffffffff },
  .config_index = 15
  },
 [50] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x4},
  .valid_entry = true,
  .verify_same = { .eax = 0xffffffff, .ebx = 0xffffffff, .ecx = 0xffffffff },
  .fixed1 = { .eax = 0x0, .ebx = 0x0, .ecx = 0x0 },
  .fixed0_or_dynamic = { .eax = 0xffffffe0, .ebx = 0xffff0000, .ecx = 0xffff0000, .edx = 0xffffffff },
  .config_index = 16
  },
 [51] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x5},
  .valid_entry = true,
  .verify_mask = { .ecx = 0xff00 },
  .verify_value = { .ecx = 0x0 },
  .verify_same = { .eax = 0xffffffff, .ebx = 0xffffffff, .ecx = 0xffffffff },
  .fixed1 = { .eax = 0x0, .ebx = 0x0, .ecx = 0x0 },
  .fixed0_or_dynamic = { .eax = 0xffffffe0, .ebx = 0xffff0000, .ecx = 0xffff0000, .edx = 0xffffffff },
  .config_index = 17
  },
...
}

const cpuid_configurable_t cpuid_configurable[MAX_NUM_CPUID_CONFIG] = {
...
 [12] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x0},
  .valid_entry = true,
  .config_direct = { .eax = 0x1f, .ebx = 0xffff, .ecx = 0xffff },
  .lookup_index = 46
  },
 [13] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x1},
  .valid_entry = true,
  .config_direct = { .eax = 0x1f, .ebx = 0xffff, .ecx = 0xffff },
  .lookup_index = 47
  },
 [14] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x2},
  .valid_entry = true,
  .config_direct = { .eax = 0x1f, .ebx = 0xffff, .ecx = 0xffff },
  .lookup_index = 48
  },
 [15] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x3},
  .valid_entry = true,
  .config_direct = { .eax = 0x1f, .ebx = 0xffff, .ecx = 0xffff },
  .lookup_index = 49
  },
 [16] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x4},
  .valid_entry = true,
  .config_direct = { .eax = 0x1f, .ebx = 0xffff, .ecx = 0xffff },
  .lookup_index = 50
  },
 [17] = { .leaf_subleaf = {.leaf = 0x1f, .subleaf = 0x5},
  .valid_entry = true,
  .config_direct = { .eax = 0x1f, .ebx = 0xffff, .ecx = 0xffff },
  .lookup_index = 51
  },
...
}
```
* 可见，数组 `cpuid_configurable[]` 中元素的 `.lookup_index` 是其在数组 `cpuid_lookup[]` 中对应的元素的索引
* 可以看到 `CPUID(0xB)` 在 `cpuid_lookup[]` 中的 `.config_index = CPUID_CONFIG_NULL_IDX`
* 代码片段见 [TDX Module 提供给 VMM 可配置的 CPUID 数据](#tdx-module-提供给-vmm-可配置的-cpuid-数据)

### Host 初始化 TD VM 时 TDX Module 的处理
* TDX module 在处理 seamcall `TDH_MNG_INIT` 时，
  * `check_cpuid_1f_and_compute_cpuid_0b()` 检查 `CPUID(0x1F)` 的虚拟拓扑配置，并派生 `CPUID(0xB)` 的值。如果 VMM 都配成全零，则采用 native 的值
  * 在 `read_and_set_cpuid_configurations()` 函数中预设 `tdcs_ptr->executions_ctl_fields.topology_enum_configured = true`，将来 seamcall `TDH_VP_INIT` 配置不对时再清除
  * tdx-module/src/vmm_dispatcher/api_calls/tdh_mng_init.c
```c
static api_error_type read_and_set_cpuid_configurations(tdcs_t * tdcs_ptr,
                                                        td_params_t * td_params_ptr,
                                                        tdx_module_global_t * global_data_ptr,
                                                        tdx_module_local_t * local_data_ptr)
{
...
    // Check the virtual topology configuration of CPUID(0x1F) and derive CPUID(0xB).
    // If configured as all-0, use the h/w values.
    return_val = check_cpuid_1f_and_compute_cpuid_0b(tdcs_ptr, true);
    if (return_val != TDX_SUCCESS)
    {
        goto EXIT;
    }

    // May be cleared later if not configured for all VCPUs
    tdcs_ptr->executions_ctl_fields.topology_enum_configured = true;
...
}
```

### Host 初始化 vCPU 时 TDX Module 的处理
* Host 执行 seamcall `TDH.VP.INIT` 初始化每个 vCPU 时，TDX module 执行以下函数
  * 版本 0 的 `TDH.VP.INIT` 不支持枚举拓扑虚拟化，但凡有一个 vCPU 用到 v0 的接口，清除之前预设的 `tdcs_ptr->executions_ctl_fields.topology_enum_configured = true`
  * 否则将 VMM `r8` 寄存器传进来的 x2APIC ID 保存在 per-TD 的 `x2apic_ids[]` 数组
  * tdx-module/src/vmm_dispatcher/api_calls/tdh_vp_init.c
```c
api_error_type tdh_vp_init(uint64_t target_tdvpr_pa, uint64_t td_vcpu_rcx)
{
...
    uint64_t x2apic_id = get_local_data()->vmm_regs.r8; //r8 寄存器传过来的值作为 x2APIC ID
...
    //per-VM 的 num_vcpus 是 vCPU 的计数，每初始化一个 vCPU +1
    /*
     * Set the VCPU index and increment the number of VCPUs in the TD
     * MAX_VCPUS can be in the range 0x0001 to 0xFFFF.
     * Thus, VCPU_INDEX is in the range 0x0000 to 0xFFFE.
     * This assures that there in no overflow in the 16b VPID, later assigned as VCPU_INDEX + 1.
     */
    vcpu_index = _lock_xadd_32b(&tdcs_ptr->management_fields.num_vcpus, 1); //当前 vCPU 的索引
    if (vcpu_index >= tdcs_ptr->executions_ctl_fields.max_vcpus) //vCPU 的索引不能超过限制
    {
        (void)_lock_xadd_32b(&tdcs_ptr->management_fields.num_vcpus, (uint32_t)-1);
        TDX_ERROR("Max VCPUS (%d) has been exceeded\n", tdcs_ptr->executions_ctl_fields.max_vcpus);
        return_val = TDX_MAX_VCPUS_EXCEEDED;
        goto EXIT;
    }
    tdvps_ptr->management.vcpu_index = vcpu_index; //当前 vCPU 的索引记录到 per-vCPU 数据结构中的 vCPU 索引

    if (leaf_opcode.version == 0)
    {   //版本 0 的 TDH.VP.INIT 不支持枚举拓扑虚拟化
        // No x2APIC ID was provided
        tdcs_ptr->executions_ctl_fields.topology_enum_configured = false;
    }
    else
    {   //VMM 指定的 x2APIC ID 的值超出范围，回退 vCPU 计数，返回错误
        // Check and save the configured x2APIC ID.  Upper 32 bits must be 0.
        if (x2apic_id > 0xFFFFFFFF)
        {
            (void)_lock_xadd_32b(&tdcs_ptr->management_fields.num_vcpus, (uint32_t)-1);
            return_val = api_error_with_operand_id(TDX_OPERAND_INVALID, OPERAND_ID_R8);
            goto EXIT;
        }
        //per-TD 的 x2apic_ids[]数组里已经有了这个 x2APIC ID，返回错误
        for (uint32_t i = 0; i < vcpu_index; i++)
        {
            if ((uint32_t)x2apic_id == tdcs_ptr->x2apic_ids[i])
            {
                return_val = api_error_with_operand_id(TDX_X2APIC_ID_NOT_UNIQUE, tdcs_ptr->x2apic_ids[i]);
                goto EXIT;
            }
        }
        //通过以上检查，将传进来的 x2APIC ID 保存在 per-TD 的 x2apic_ids[] 数组
        tdcs_ptr->x2apic_ids[vcpu_index] = (uint32_t)x2apic_id;
    }
...
}
```

## TD Guest 需要启用 `ENUM_TOPOLOGY` 特性
* [[PATCHv6 4/4] x86/tdx: Enable CPU topology enumeration](https://lore.kernel.org/lkml/20240828093505.2359947-1-kirill.shutemov@linux.intel.com/T/#mb7e0faf194fdb31b52f2e5e0fc1d80745b9cf860)
* [[PATCHv6, RESEND 4/4] x86/tdx: Enable CPU topology enumeration](https://lore.kernel.org/lkml/20241104103803.195705-5-kirill.shutemov@linux.intel.com/)
* TDX 1.0 定义了 TDX guest 平台的基线行为。
  * TDX 1.0 在访问与拓扑相关的 `CPUID` leaf（`0xB` 和 `0x1F`）和 `X2APIC_APICID` MSR 时生成 `#VE`。
  * 内核在 `CPUID` 拓扑上返回全零。
* 实际上，这意味着内核只能使用普通拓扑进行启动。任何复杂情况都会导致问题。
* `ENUM_TOPOLOGY` 功能允许 VMM 向 guest 提供拓扑信息。启用该功能可消除与拓扑相关的 `#VE`：TDX module 虚拟化对 `CPUID` leaf 和 MSR 的访问。
* 如果可用，启用 `ENUM_TOPOLOGY`。
---
```diff
diff --git a/arch/x86/coco/tdx/tdx.c b/arch/x86/coco/tdx/tdx.c
index f969f4f5ebf8..ca162a7ce4d1 100644
--- a/arch/x86/coco/tdx/tdx.c
+++ b/arch/x86/coco/tdx/tdx.c
@@ -249,6 +249,32 @@ static void disable_sept_ve(u64 td_attr)
    return;
 }
 
+/*
+ * TDX 1.0 generates a #VE when accessing topology-related CPUID leafs (0xB and
+ * 0x1F) and the X2APIC_APICID MSR. The kernel returns all zeros on CPUID #VEs.
+ * In practice, this means that the kernel can only boot with a plain topology.
+ * Any complications will cause problems.
+ *
+ * The ENUM_TOPOLOGY feature allows the VMM to provide topology information.
+ * Enabling the feature  eliminates topology-related #VEs: the TDX module
+ * virtualizes accesses to the CPUID leafs and the MSR.
+ *
+ * Enable ENUM_TOPOLOGY if it is available.
+ */
+static void enable_cpu_topology_enumeration(void)
+{
+	u64 configured;
+
+	/* Has the VMM provided a valid topology configuration? */
+	tdg_vm_rd(TDCS_TOPOLOGY_ENUM_CONFIGURED, &configured);
+	if (!configured) {
+		pr_err("VMM did not configure X2APIC_IDs properly\n");
+		return;
+	}
+
+	tdg_vm_wr(TDCS_TD_CTLS, TD_CTLS_ENUM_TOPOLOGY, TD_CTLS_ENUM_TOPOLOGY);
+}
+
 static void tdx_setup(u64 *cc_mask)
 {
    struct tdx_module_args args = {};
@@ -280,6 +306,7 @@ static void tdx_setup(u64 *cc_mask)
    tdg_vm_wr(TDCS_NOTIFY_ENABLES, 0, -1ULL);
 
    disable_sept_ve(td_attr);
+	enable_cpu_topology_enumeration();
 }
 
 /*
diff --git a/arch/x86/include/asm/shared/tdx.h b/arch/x86/include/asm/shared/tdx.h
index fecb2a6e864b..89f7fcade8ae 100644
--- a/arch/x86/include/asm/shared/tdx.h
+++ b/arch/x86/include/asm/shared/tdx.h
@@ -23,12 +23,14 @@
 #define TDCS_CONFIG_FLAGS		0x1110000300000016
 #define TDCS_TD_CTLS			0x1110000300000017
 #define TDCS_NOTIFY_ENABLES		0x9100000000000010
+#define TDCS_TOPOLOGY_ENUM_CONFIGURED	0x9100000000000019
 
 /* TDCS_CONFIG_FLAGS bits */
 #define TDCS_CONFIG_FLEXIBLE_PENDING_VE	BIT_ULL(1)
 
 /* TDCS_TD_CTLS bits */
 #define TD_CTLS_PENDING_VE_DISABLE	BIT_ULL(0)
+#define TD_CTLS_ENUM_TOPOLOGY		BIT_ULL(1)
 
 /* TDX hypercall Leaf IDs */
 #define TDVMCALL_MAP_GPA		0x10001
```
* 其实这里对应着两个域：
  * `TDCS_TOPOLOGY_ENUM_CONFIGURED`：TD guest 不能控制的 `tdcs_ptr->executions_ctl_fields.topology_enum_configured` 域，VMM 是否已（正确）配置拓扑枚举虚拟化
  * `TD_CTLS_ENUM_TOPOLOGY`：TD guest 可以控制的 `td_ctls.enum_topology` 域，TD guest 启用拓扑枚举虚拟化

### TDX Module 对 TD Guest 启用拓扑枚举虚拟化的处理
* 如果 VMM 已（正确）配置拓扑枚举虚拟化 `topology_enum_configured`，
  * tdx-module/src/common/metadata_handlers/metadata_td.c
```c
api_error_code_e md_td_write_element(md_field_id_t field_id, const md_lookup_t* entry, md_access_t access_type,
        md_access_qualifier_t access_qual, md_context_ptrs_t md_ctx, uint64_t wr_value, uint64_t wr_request_mask,
        uint64_t* old_value)
{
...
      switch (entry->field_id.raw)
      {
            case MD_TDCS_TD_CTLS_FIELD_ID:
            {
...
                // To enable virtual topology enumeration, all VCPU must have been properly configured
                td_ctls_t td_ctls = { .raw = wr_value };
                if (td_ctls.enum_topology && !md_ctx.tdcs_ptr->executions_ctl_fields.topology_enum_configured)
                {
                    return TDX_METADATA_FIELD_VALUE_NOT_VALID;
                }
...
                break;
            }
      }
...
    // Update only the relevant bits per the write mask
    if (!write_done)
    {   //更新写入值到指定的 metadata 域
        if (!write_element_by_size(elem_ptr, read_value, wr_value, combined_wr_mask, elem_size))
        {
            return api_error_with_operand_id(TDX_OPERAND_BUSY, OPERAND_ID_METADATA_FIELD);
        }
    }
    //返回 metadata 域的旧值
    *old_value = read_value & rd_mask;

    return TDX_SUCCESS;
}
```

## TDX Module 对虚拟 CPU 拓扑枚举的处理

### Guest 执行拓扑枚举相关的 `CPUID` 指令时
* 相关指令为 `CPUID.(EAX=0x01):EBX[31:24]` 和 `CPUID.(EAX=0x0B):EDX`，`CPUID.(EAX=0x1F):EDX`
* tdx-module/src/td_dispatcher/vm_exits/td_cpuid.c
```c
void td_cpuid_exit(void)
{
...
    // Get the CPUID value calculated by TDH_MNG_INIT from TDCS
    return_values = vp_ctx->tdcs->cpuid_config_vals[index]; //从 configured/calculated 后的数组中将值取出

    // Special CPUID Leaves/Sub-Leaves Handling
    //  - XFAM-allowed
    //  - KeyLocker-allowed
    //  - Perfmon-allowed
    //  - Dynamic

    switch (leaf)
    {
    case 0x1:
        cpuid_01_ebx.raw = return_values.ebx; //先将这个值放在本地便于加工
        //如果拓扑枚举虚拟化选项开启，获取 vCPU 索引对应的 x2APIC ID 的值，放在 EBX[31:24] bits
        if (vp_ctx->tdcs->executions_ctl_fields.td_ctls.enum_topology)
        {
            // INITIAL_APIC_ID dynamically reflects x2APIC ID, but is limited to 8 bits
            cpuid_01_ebx.initial_apic_id = (uint8_t)vp_ctx->tdcs->x2apic_ids[vp_ctx->tdvps->management.vcpu_index];
        }
        else
        {   //否则直接用 vCPU 的索引
            // INITIAL_APIC_ID dynamically reflects VCPU_INDEX, but is limited to 8 bits
            cpuid_01_ebx.initial_apic_id = vp_ctx->tdvps->management.vcpu_index;
        }
        ...
        break;
...
    case 0xB:
    case 0x1F:
        if (vp_ctx->tdcs->executions_ctl_fields.td_ctls.enum_topology)
        {   //如果拓扑枚举虚拟化选项开启，获取 vCPU 索引对应的 x2APIC ID 的值，放在 EDX
            // EDX dynamically reflects x2APIC ID
            return_values.edx = vp_ctx->tdcs->x2apic_ids[vp_ctx->tdvps->management.vcpu_index];
        }
        else
        {   //否则按照 spec 要求注入 #VE
            // No topology enumeration
            tdx_inject_ve(VMEXIT_REASON_CPUID_INSTRUCTION, 0, vp_ctx->tdvps, 0, 0);
            return;
        }

        break;
...
    }
...
}
```

### Guest 执行拓扑枚举相关的 `RDMSR` 指令时
* tdx-module/src/td_dispatcher/vm_exits/td_msr_access.c
```c
td_msr_access_status_t td_rdmsr_exit(void)
{
...
    uint32_t msr_addr = (uint32_t)tdvps_p->guest_state.gpr_state.rcx;
...
   switch (msr_addr)
   {
...
       case IA32_X2APIC_APICID:
        {
            if (tdcs_p->executions_ctl_fields.td_ctls.enum_topology)
            {   //如果拓扑枚举虚拟化选项开启，获取 vCPU 索引对应的 x2APIC ID 的值，放在 EAX
                // Return the current VCPU's virtual x2APIC ID
                tdvps_p->guest_state.gpr_state.rdx = 0;
                tdvps_p->guest_state.gpr_state.rax = tdcs_p->x2apic_ids[tdvps_p->management.vcpu_index];
            }
            else
            {   //否则按照 spec 要求注入 #VE
                // No topology enumeration, #VE will be injected
                return TD_MSR_ACCESS_MSR_NON_ARCH_EXCEPTION;
            }

            break;
        }
...    
   }
...
}
```

### TD 迁移时
* tdx-module/src/common/helpers/helpers.c
```c
api_error_type check_imported_vp_state(tdr_t* tdr_p, tdcs_t* tdcs_p, tdvps_t* tdvps_p)
{
    UNUSED(tdr_p);

    if (tdcs_p->executions_ctl_fields.topology_enum_configured)
    {
        uint32_t x2apic_id = tdcs_p->x2apic_ids[tdvps_p->management.vcpu_index];

        for (uint32_t i = 0; i < tdcs_p->management_fields.num_vcpus; i++)
        {
            if (x2apic_id == tdcs_p->x2apic_ids[i])
            {
                if (i != tdvps_p->management.vcpu_index)
                {
                    return api_error_with_operand_id(TDX_X2APIC_ID_NOT_UNIQUE, x2apic_id);
                }
            }
        }
    }

    return TDX_SUCCESS;
}
```