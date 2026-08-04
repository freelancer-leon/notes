
## Guest 带 C-bit 访问共享页面

### KVM 的处理
```cpp
//arch/x86/kvm/svm/svm.c
npf_interception()
   if (is_sev_snp_guest(vcpu) && (error_code & PFERR_GUEST_ENC_MASK))
      error_code |= PFERR_PRIVATE_ACCESS;
   -> trace_kvm_page_fault(vcpu, gpa, error_code);
      //arch/x86/kvm/mmu/mmu.c
   -> kvm_mmu_page_fault(vcpu, gpa, error_code, ...)
         //arch/x86/kvm/mmu/mmu_internal.h
      -> kvm_mmu_do_page_fault(vcpu, cr2_or_gpa, error_code, false, &emulation_type, NULL)
         -> kvm_tdp_page_fault(vcpu, &fault)
               //arch/x86/kvm/mmu/mmu.c
            -> kvm_mmu_faultin_pfn(vcpu, fault, ACC_ALL)
                  //第一遍时 #VMEXIT(NPF) 与 gmem 的记录不一致，upcall 到 Qemu 去转换内存
                  if (fault->is_private != kvm_mem_is_private(kvm, fault->gfn)) {
                     //include/linux/kvm_host.h
                     kvm_mmu_prepare_memory_fault_exit(vcpu, fault);
                        vcpu->run->exit_reason = KVM_EXIT_MEMORY_FAULT;
                        if (is_private)
                           vcpu->run->memory_fault.flags |= KVM_MEMORY_EXIT_FLAG_PRIVATE;
                     return -EFAULT;
                  }
                  //第二遍时正常 #NPF 的 fault in 处理，并更新 RMP 表的记录为私有页面
               -> __kvm_mmu_faultin_pfn(vcpu, fault)
                     if (fault->is_private || kvm_memslot_is_gmem_only(fault->slot))
                        return kvm_mmu_faultin_pfn_gmem(vcpu, fault);
                        -> kvm_gmem_get_pfn(kvm, slot, gfn, &pfn, &page, &order)
                           -> __kvm_gmem_get_pfn(file, slot, index, pfn, max_order)
                              //virt/kvm/guest_memfd.c
                           -> kvm_gmem_prepare_folio(kvm, slot, gfn, folio)
                              -> return __kvm_gmem_prepare_folio(kvm, slot, index, folio);
                                    //arch/x86/kvm/x86.c
                                 -> kvm_arch_gmem_prepare(kvm, gfn, pfn, folio_order(folio))
                                    -> kvm_x86_call(gmem_prepare)(kvm, pfn, gfn, max_order)
                                       //arch/x86/kvm/svm/sev.c
                                    => sev_gmem_prepare()
                                       -> snp_lookup_rmpentry(pfn, &assigned, &level)
                                       -> rmp_make_private(pfn_aligned, gfn_to_gpa(gfn_aligned), level, sev->asid, false)
                                          -> adjust_direct_map(pfn, level)
                                          -> return rmpupdate(pfn, &state);
            //arch/x86/kvm/mmu/tdp_mmu.c
         -> kvm_tdp_mmu_map(vcpu, fault)
            -> tdp_mmu_map_handle_target_level(vcpu, fault, &iter)
   if (rc > 0 && error_code & PFERR_GUEST_RMP_MASK)
   -> sev_handle_rmp_fault(vcpu, gpa, error_code)
      -> kvm_gmem_get_pfn(kvm, slot, gfn, &pfn, &page, &order)
      -> snp_lookup_rmpentry(pfn, &assigned, &rmp_level)
      -> snp_rmptable_psmash(pfn)
      -> kvm_zap_gfn_range(kvm, gfn, gfn + PTRS_PER_PMD)
      -> trace_kvm_rmp_fault(vcpu, gpa, pfn, error_code, rmp_level, ret)
```
### Qemu 相应的处理
* 第一遍时 `#VMEXIT(NPF)` 与 gmem 的记录不一致，upcall 到 Qemu 去转换内存
* 这给 Qemu 一个机会去监听 ramblock 的转变，如果是 DMA 内存被从共享转为私有，需要清除 IOMMU 的表项
```cpp
//accel/kvm/kvm-all.c
kvm_cpu_exec()
-> run_ret = kvm_vcpu_ioctl(cpu, KVM_RUN, 0)
   case KVM_EXIT_MEMORY_FAULT:
   -> kvm_convert_memory(run->memory_fault.gpa, run->memory_fault.size, run->memory_fault.flags & KVM_MEMORY_EXIT_FLAG_PRIVATE)
```