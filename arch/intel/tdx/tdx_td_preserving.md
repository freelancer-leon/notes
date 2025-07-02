

```c
reload_store()
-> tdx_module_update(force_load)
   -> tdx_module_update_start()
      -> raw_notifier_call_chain_robust(&update_chain_head, TDX_UPDATE_START, TDX_UPDATE_ABORT, data)
         -> notifier_call_chain_robust(&nh->head, val_up, val_down, v)
            -> notifier_call_chain(nl, val_up, v, -1, &nr)
               -> nb->notifier_call(nb, val, v)
               => tdx_update_notifier()
                     case TDX_UPDATE_START:
                  -> kvm_start_update_fw(tdx_module)
                     -> init_completion(&fw->completion) //初始化 completion，在这之后检查该 completion 的例程会等待
                        fw->update = true;
                     -> kvm_arch_start_update_fw(kvm)
                        -> kvm_make_all_cpus_request(kvm, KVM_REQ_FW_UPDATE)
                     -> synchronize_srcu(&fw->srcu) //如果有其他 KVM 组件操作该 fw，更新例程会阻塞在这，等待其他例程退出临界区
                     if (atomic_add_return(bias, &nr_running_td) != bias) {
                        if ((unsigned long)data == 0) {
                           end_update = true;
                           ret = -EBUSY;
                           break;
                        }
                     }
                     if (end_update) {
                     -> atomic_sub(bias, &nr_running_td);
                     -> kvm_end_update_fw(tdx_module);
                           fw->update = false;
                        -> synchronize_srcu(&fw->srcu)
                        -> complete_all(&fw->completion) //完成 completion，等待的例程会被唤醒
                     }
   -> tdx_module_update_action(TDX_UPDATE_PREPARE, event_data)
      -> raw_notifier_call_chain(&update_chain_head, action, data)
   -> do_tdx_module_update()
   -> WARN_ON_ONCE(tdx_module_update_action(update_status, NULL))
```

```cpp
kvm_vm_ioctl()
-> fw_idx = kvm_get_fw(kvm)
   -> wait_for_completion(&kvm->fw->completion) //检查更新是否完成的 completion，如果没完成则等待
   -> idx = srcu_read_lock(&kvm->fw->srcu) //进入临界区
   default:
   -> kvm_arch_vm_ioctl()
         case KVM_MEMORY_ENCRYPT_OP:
         -> static_call(kvm_x86_mem_enc_ioctl)(kvm, argp)
         => vt_mem_enc_ioctl()
            -> tdx_vm_ioctl()
               case KVM_TDX_INIT_VM:
               -> tdx_td_init()
               -> __tdx_td_init()
                  -> ret = atomic_inc_unless_negative(&nr_running_td) ? 0 : -EBUSY;
-> kvm_put_fw(kvm, fw_idx)
   -> srcu_read_unlock(&kvm->fw->srcu, idx) //退出临界区
```

```c
kvm_destroy_vm()
-> fw_idx = kvm_get_fw(kvm)
   -> wait_for_completion(&kvm->fw->completion)  //检查更新是否完成的 completion，如果没完成则等待
   -> idx = srcu_read_lock(&kvm->fw->srcu) //进入临界区
-> kvm_arch_destroy_vm(kvm)
   -> static_call_cond(kvm_x86_vm_destroy)(kvm)
   => vt_vm_free()
      -> tdx_vm_free(kvm)
         -> __tdx_vm_free(kvm)
            -> tdx_vm_free_tdr(kvm_tdx)
               -> atomic_dec(&nr_running_td)
-> kvm_put_fw(kvm, fw_idx)
   -> srcu_read_unlock(&kvm->fw->srcu, idx) //退出临界区
```

```c
CPU 0                                            CPU 1
-------------------------                        -------------------------
kvm_vm_ioctl()
-> fw_idx = kvm_get_fw(kvm)
   -> wait_for_completion(&kvm->fw->completion) //CPU 1 上还未初始化 completion，CPU 0 不会阻塞

                                                 kvm_start_update_fw(tdx_module)
                                                 -> init_completion(&fw->completion)
                                                    fw->update = true;
                                                 -> kvm_arch_start_update_fw(kvm)
                                                    -> kvm_make_all_cpus_request(kvm, KVM_REQ_FW_UPDATE)
                                                 -> synchronize_srcu(&fw->srcu) //CPU 0 没进入临界区，所以这里不会阻塞

   -> idx = srcu_read_lock(&kvm->fw->srcu) //进入临界区
   default:
   -> kvm_arch_vm_ioctl()
         case KVM_MEMORY_ENCRYPT_OP:
         -> static_call(kvm_x86_mem_enc_ioctl)(kvm, argp)
         => vt_mem_enc_ioctl()
            -> tdx_vm_ioctl()
               case KVM_TDX_INIT_VM:
               -> tdx_td_init()
               -> __tdx_td_init()
                  -> ret = atomic_inc_unless_negative(&nr_running_td) ? 0 : -EBUSY; //CPU 0 先完成 +1

                                                 if (atomic_add_return(bias, &nr_running_td) != bias) { //CPU 1 把 nr_running_td 变成负数
                                                    if ((unsigned long)data == 0) {
                                                       end_update = true; //结束更新
                                                       ret = -EBUSY;      //设置错误码
                                                       break;
                                                    }
                                                 }
                                                 if (end_update) {
                                                 -> atomic_sub(bias, &nr_running_td); //CPU 1 把之前施加在 nr_running_td 上的操作还原
                                                 -> kvm_end_update_fw(tdx_module);
                                                       fw->update = false;
                                                    -> synchronize_srcu(&fw->srcu)   //等待 CPU 0 退出临界区

-> kvm_put_fw(kvm, fw_idx)
   -> srcu_read_unlock(&kvm->fw->srcu, idx) //CPU 0 退出临界区，CPU 1 可以继续运行

                                                    -> complete_all(&fw->completion) //CPU 0 不受此影响
                                                 }
```