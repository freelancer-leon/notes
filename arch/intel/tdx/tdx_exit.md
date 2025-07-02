# TD Exit

## Host

### vt_handle_exit_irqoff()

#### tdx_handle_exit_irqoff()

##### `EXIT_REASON_EXCEPTION_NMI`
* vmx_handle_exception_nmi_irqoff()

##### `EXIT_REASON_EXTERNAL_INTERRUPT`
* vmx_handle_external_interrupt_irqoff()
  * vmx_do_interrupt_irqoff(gate_offset(desc))

### vt_handle_exit()

#### tdx_handle_exit()

##### `EXIT_REASON_TRIPLE_FAULT`
* tdx_handle_triple_fault()

##### `EXIT_REASON_EXCEPTION_NMI`
* tdx_handle_exception()

##### `EXIT_REASON_EXTERNAL_INTERRUPT`
* tdx_handle_external_interrupt()

##### `EXIT_REASON_TDCALL`
* handle_tdvmcall()
  * `EXIT_REASON_CPUID`: tdx_emulate_cpuid()
  * `EXIT_REASON_HLT`: tdx_emulate_hlt()
  * `EXIT_REASON_IO_INSTRUCTION`: tdx_emulate_io()
  * `EXIT_REASON_EPT_VIOLATION`: tdx_emulate_mmio()
  * `EXIT_REASON_MSR_READ`: tdx_emulate_rdmsr()
  * `EXIT_REASON_MSR_WRITE`: tdx_emulate_wrmsr()
  * `TDG_VP_VMCALL_GET_TD_VM_CALL_INFO`: tdx_get_td_vm_call_info()
  * `TDG_VP_VMCALL_REPORT_FATAL_ERROR`: tdx_vp_vmcall_to_user()
  * `TDG_VP_VMCALL_MAP_GPA`: tdx_map_gpa()
  * `TDG_VP_VMCALL_SETUP_EVENT_NOTIFY_INTERRUPT`: tdx_setup_event_notify_interrupt()
  * `TDG_VP_VMCALL_GET_QUOTE`: tdx_get_quote()
  * `TDG_VP_VMCALL_SERVICE`:  tdx_handle_service()
  * default and *Unknown VMCALL*: tdx_vp_vmcall_to_user()

##### `EXIT_REASON_EPT_VIOLATION`
* tdx_handle_ept_violation()

##### `EXIT_REASON_EPT_MISCONFIG`
* tdx_handle_ept_misconfig()

##### `EXIT_REASON_OTHER_SMI`
* return 1

##### `EXIT_REASON_BUS_LOCK`
* tdx_handle_bus_lock_vmexit()
  * return 1

##### `EXIT_REASON_DR_ACCESS`
* tdx_handle_dr_exit()

##### default
* return 0

## Guest `#VE`

### exc_virtualization_exception()

#### tdx_handle_virt_exception()

##### virt_exception_user()
* `EXIT_REASON_CPUID`: handle_cpuid()
* default: return -EIO

##### virt_exception_kernel()
* `EXIT_REASON_HLT`: handle_halt()
* `EXIT_REASON_MSR_READ`: read_msr()
* `EXIT_REASON_MSR_WRITE`: write_msr()
* `EXIT_REASON_CPUID`: handle_cpuid()
* `EXIT_REASON_EPT_VIOLATION`: handle_mmio()
* `EXIT_REASON_IO_INSTRUCTION`: handle_io()
* default: return -EIO