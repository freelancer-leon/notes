# SGX 虚拟化

## 虚拟 EPC
* 还有一个虚拟 EPC 驱动来支持 guest 中的 SGX enclave，与 SGX 驱动不同，由虚拟 EPC 驱动分配的 EPC 页面没有与之关联的特定 enclave
  * 这是因为 KVM 不跟踪 guest 如何使用 EPC 页面
  * 因此，SGX 核心页面回收器不支持通过虚拟 EPC 驱动回收分配给 KVM guest 的 EPC 页面
* 如果用户想在同一台机器上的 host 和 guest 中部署 SGX 应用程序，用户应该为 host SGX 应用程序预留足够的 EPC（通过从 *物理 EPC 大小* 中取出 *所有 SGX VM 总的虚拟 EPC 大小*），以便它们可以以可接受的性能运行
* 架构行为是将所有 EPC 页面恢复到未初始化状态，即便在 guest 重新启动后也一样
  * 因为这个状态只能通过特权`ENCLS[EREMOVE]`指令来达到，所以`/dev/sgx_vepc`提供了`SGX_IOC_VEPC_REMOVE_ALL`的`ioctl` 来在虚拟 EPC 中的所有页面上执行指令
* `EREMOVE`可能因三个原因而失败，用户空间必须注意预期的失败并按如下方式处理它们：
1. 当页面所属的 enclave 中运行任何线程时，页面删除将始终失败。
   * 在这种情况下，`ioctl`将返回`EBUSY`，而与它是否已成功删除某些页面无关；
   * 用户空间可以通过 阻止 *映射这些虚拟 EPC 的* 任何 vcpu 执行来避免失败
2. 如果两个对`EREMOVE`的调用同时发生在引用相同“SECS”元数据页面的页面上，页面删除将导致通用保护缺陷（general protection fault）
   * 如果同时调用`SGX_IOC_VEPC_REMOVE_ALL`，或者 guest 中的`/dev/sgx_vepc`文件描述符与`SGX_IOC_VEPC_REMOVE_ALL`同时关闭，则可能会发生这种情况；
   * 这也将被报告为`EBUSY`。这可以通过序列化对`ioctl()`和`close()`的调用在用户空间中规避，但一般来说这不成问题
3. 最后，对于仍有子页面的 SECS 元数据页面，页面删除将失败
   * 可以通过对映射到 guest 的所有`/dev/sgx_vepc`文件描述符执行`SGX_IOC_VEPC_REMOVE_ALL`来删除子页面
   * 这意味着必须调用`ioctl()`两次：删除子页面的初始调用集和删除 SECS 页面的后续调用集
   * 只有那些从第一次调用返回非零值的映射才需要第二组调用
   * 如果第二轮`SGX_IOC_VEPC_REMOVE_ALL`调用中的任何一个返回代码不是`0`，则表明内核或用户空间的客户端存在错误
* 要使用`/dev/sgx_vepc`分配具有特定大小的虚拟 EPC 实例，hypervisor 打开`/dev/sgx_vepc`，并使用预期大小的`mmap()`来获取虚拟 EPC 的地址范围。然后可以使用该地址范围创建一个 KVM 内存插槽作为 guest 的虚拟 EPC。
* Provisioning key 是敏感的。SGX 驱动程序仅允许在 enclave 创建者有权打开`/dev/sgx_provision`时创建可以访问 provisioning key 的 enclave。
  * 这也应该适用于 VM，因为 provisioning key 是特定于平台的，因此不受限制的 VM 也可能会破坏配置密钥。

## 实现
```cpp
struct sgx_vepc {
	struct xarray page_array;
	struct mutex lock;
};

/*
 * Temporary SECS pages that cannot be EREMOVE'd due to having child in other
 * virtual EPC instances, and the lock to protect it.
 */
static struct mutex zombie_secs_pages_lock;
static struct list_head zombie_secs_pages;

static int __sgx_vepc_fault(struct sgx_vepc *vepc,
			    struct vm_area_struct *vma, unsigned long addr)
{
	struct sgx_epc_page *epc_page;
	unsigned long index, pfn;
	int ret;

	WARN_ON(!mutex_is_locked(&vepc->lock));

	/* Calculate index of EPC page in virtual EPC's page_array */
	index = vma->vm_pgoff + PFN_DOWN(addr - vma->vm_start);

	epc_page = xa_load(&vepc->page_array, index);
	if (epc_page)
		return 0;

	epc_page = sgx_alloc_epc_page(vepc, false);
	if (IS_ERR(epc_page))
		return PTR_ERR(epc_page);

	ret = xa_err(xa_store(&vepc->page_array, index, epc_page, GFP_KERNEL));
	if (ret)
		goto err_free;

	pfn = PFN_DOWN(sgx_get_epc_phys_addr(epc_page));

	ret = vmf_insert_pfn(vma, addr, pfn);
	if (ret != VM_FAULT_NOPAGE) {
		ret = -EFAULT;
		goto err_delete;
	}

	return 0;

err_delete:
	xa_erase(&vepc->page_array, index);
err_free:
	sgx_free_epc_page(epc_page);
	return ret;
}

static vm_fault_t sgx_vepc_fault(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf->vma;
	struct sgx_vepc *vepc = vma->vm_private_data;
	int ret;

	mutex_lock(&vepc->lock);
	ret = __sgx_vepc_fault(vepc, vma, vmf->address);
	mutex_unlock(&vepc->lock);

	if (!ret)
		return VM_FAULT_NOPAGE;

	if (ret == -EBUSY && (vmf->flags & FAULT_FLAG_ALLOW_RETRY)) {
		mmap_read_unlock(vma->vm_mm);
		return VM_FAULT_RETRY;
	}

	return VM_FAULT_SIGBUS;
}

static const struct vm_operations_struct sgx_vepc_vm_ops = {
	.fault = sgx_vepc_fault,
};

static int sgx_vepc_mmap(struct file *file, struct vm_area_struct *vma)
{
	struct sgx_vepc *vepc = file->private_data;

	if (!(vma->vm_flags & VM_SHARED))
		return -EINVAL;

	vma->vm_ops = &sgx_vepc_vm_ops;
	/* Don't copy VMA in fork() */
	vma->vm_flags |= VM_PFNMAP | VM_IO | VM_DONTDUMP | VM_DONTCOPY;
	vma->vm_private_data = vepc;

	return 0;
}

static int sgx_vepc_remove_page(struct sgx_epc_page *epc_page)
{
	/*
	 * Take a previously guest-owned EPC page and return it to the
	 * general EPC page pool.
	 *
	 * Guests can not be trusted to have left this page in a good
	 * state, so run EREMOVE on the page unconditionally.  In the
	 * case that a guest properly EREMOVE'd this page, a superfluous
	 * EREMOVE is harmless.
	 */
	return __eremove(sgx_get_epc_virt_addr(epc_page));
}

static int sgx_vepc_free_page(struct sgx_epc_page *epc_page)
{
	int ret = sgx_vepc_remove_page(epc_page);
	if (ret) {
		/*
		 * Only SGX_CHILD_PRESENT is expected, which is because of
		 * EREMOVE'ing an SECS still with child, in which case it can
		 * be handled by EREMOVE'ing the SECS again after all pages in
		 * virtual EPC have been EREMOVE'd. See comments in below in
		 * sgx_vepc_release().
		 *
		 * The user of virtual EPC (KVM) needs to guarantee there's no
		 * logical processor is still running in the enclave in guest,
		 * otherwise EREMOVE will get SGX_ENCLAVE_ACT which cannot be
		 * handled here.
		 */
		WARN_ONCE(ret != SGX_CHILD_PRESENT, EREMOVE_ERROR_MESSAGE,
			  ret, ret);
		return ret;
	}

	sgx_free_epc_page(epc_page);
	return 0;
}

static long sgx_vepc_remove_all(struct sgx_vepc *vepc)
{
	struct sgx_epc_page *entry;
	unsigned long index;
	long failures = 0;

	xa_for_each(&vepc->page_array, index, entry) {
		int ret = sgx_vepc_remove_page(entry);
		if (ret) {
			if (ret == SGX_CHILD_PRESENT) {
				/* The page is a SECS, userspace will retry.  */
				failures++;
			} else {
				/*
				 * Report errors due to #GP or SGX_ENCLAVE_ACT; do not
				 * WARN, as userspace can induce said failures by
				 * calling the ioctl concurrently on multiple vEPCs or
				 * while one or more CPUs is running the enclave.  Only
				 * a #PF on EREMOVE indicates a kernel/hardware issue.
				 */
				WARN_ON_ONCE(encls_faulted(ret) &&
					     ENCLS_TRAPNR(ret) != X86_TRAP_GP);
				return -EBUSY;
			}
		}
		cond_resched();
	}

	/*
	 * Return the number of SECS pages that failed to be removed, so
	 * userspace knows that it has to retry.
	 */
	return failures;
}

static int sgx_vepc_release(struct inode *inode, struct file *file)
{
	struct sgx_vepc *vepc = file->private_data;
	struct sgx_epc_page *epc_page, *tmp, *entry;
	unsigned long index;

	LIST_HEAD(secs_pages);

	xa_for_each(&vepc->page_array, index, entry) {
		/*
		 * Remove all normal, child pages.  sgx_vepc_free_page()
		 * will fail if EREMOVE fails, but this is OK and expected on
		 * SECS pages.  Those can only be EREMOVE'd *after* all their
		 * child pages. Retries below will clean them up.
		 */
		if (sgx_vepc_free_page(entry))
			continue;

		xa_erase(&vepc->page_array, index);
	}

	/*
	 * Retry EREMOVE'ing pages.  This will clean up any SECS pages that
	 * only had children in this 'epc' area.
	 */
	xa_for_each(&vepc->page_array, index, entry) {
		epc_page = entry;
		/*
		 * An EREMOVE failure here means that the SECS page still
		 * has children.  But, since all children in this 'sgx_vepc'
		 * have been removed, the SECS page must have a child on
		 * another instance.
		 */
		if (sgx_vepc_free_page(epc_page))
			list_add_tail(&epc_page->list, &secs_pages);

		xa_erase(&vepc->page_array, index);
	}

	/*
	 * SECS pages are "pinned" by child pages, and "unpinned" once all
	 * children have been EREMOVE'd.  A child page in this instance
	 * may have pinned an SECS page encountered in an earlier release(),
	 * creating a zombie.  Since some children were EREMOVE'd above,
	 * try to EREMOVE all zombies in the hopes that one was unpinned.
	 */
	mutex_lock(&zombie_secs_pages_lock);
	list_for_each_entry_safe(epc_page, tmp, &zombie_secs_pages, list) {
		/*
		 * Speculatively remove the page from the list of zombies,
		 * if the page is successfully EREMOVE'd it will be added to
		 * the list of free pages.  If EREMOVE fails, throw the page
		 * on the local list, which will be spliced on at the end.
		 */
		list_del(&epc_page->list);

		if (sgx_vepc_free_page(epc_page))
			list_add_tail(&epc_page->list, &secs_pages);
	}

	if (!list_empty(&secs_pages))
		list_splice_tail(&secs_pages, &zombie_secs_pages);
	mutex_unlock(&zombie_secs_pages_lock);

	xa_destroy(&vepc->page_array);
	kfree(vepc);

	return 0;
}

static int sgx_vepc_open(struct inode *inode, struct file *file)
{
	struct sgx_vepc *vepc;
	//每次打开 /dev/sgx_vepc 文件就创建一个 struct sgx_vepc 实例
	vepc = kzalloc(sizeof(struct sgx_vepc), GFP_KERNEL);
	if (!vepc)
		return -ENOMEM;
	mutex_init(&vepc->lock);
	xa_init(&vepc->page_array);
	//sgx_vepc 实例作为该文件的私有数据，对文件进行操作的时候用来找回对应的 vepc
	file->private_data = vepc;

	return 0;
}

static long sgx_vepc_ioctl(struct file *file,
			   unsigned int cmd, unsigned long arg)
{
	struct sgx_vepc *vepc = file->private_data;

	switch (cmd) {
	case SGX_IOC_VEPC_REMOVE_ALL:
		if (arg)
			return -EINVAL;
		return sgx_vepc_remove_all(vepc);

	default:
		return -ENOTTY;
	}
}

static const struct file_operations sgx_vepc_fops = {
	.owner		= THIS_MODULE,
	.open		= sgx_vepc_open,
	.unlocked_ioctl	= sgx_vepc_ioctl,
	.compat_ioctl	= sgx_vepc_ioctl,
	.release	= sgx_vepc_release,
	.mmap		= sgx_vepc_mmap,
};

static struct miscdevice sgx_vepc_dev = {
	.minor		= MISC_DYNAMIC_MINOR,
	.name		= "sgx_vepc",
	.nodename	= "sgx_vepc",
	.fops		= &sgx_vepc_fops,
};

int __init sgx_vepc_init(void)
{
	/* SGX virtualization requires KVM to work */
	if (!cpu_feature_enabled(X86_FEATURE_VMX))
		return -ENODEV;

	INIT_LIST_HEAD(&zombie_secs_pages);
	mutex_init(&zombie_secs_pages_lock);
	//创建了“/dev/sgx_vepc”和“/sys/class/misc/sgx_vepc”目录
	return misc_register(&sgx_vepc_dev);
}
```
* 尽可能使用 host 的用户空间地址（由 KVM 根据`ENCLS`参数的 GPA 提供）来执行`ENCLS/EINIT`。
* 源自`ENCLS`的对 EPC 和内存的访问都受到分段和分页机制的影响。虽然也可以通过解析 PFN 为 ENCLS 参数生成内核映射（比如`kmap()`），但使用`__uaccess_xx()`更简单。
```cpp
/**
 * sgx_virt_ecreate() - Run ECREATE on behalf of guest
 * @pageinfo:	Pointer to PAGEINFO structure
 * @secs:	Userspace pointer to SECS page
 * @trapnr:	trap number injected to guest in case of ECREATE error
 *
 * Run ECREATE on behalf of guest after KVM traps ECREATE for the purpose
 * of enforcing policies of guest's enclaves, and return the trap number
 * which should be injected to guest in case of any ECREATE error.
 *
 * Return:
 * -  0:	ECREATE was successful.
 * - <0:	on error.
 */
int sgx_virt_ecreate(struct sgx_pageinfo *pageinfo, void __user *secs,
		     int *trapnr)
{
	int ret;

	/*
	 * @secs is an untrusted, userspace-provided address.  It comes from
	 * KVM and is assumed to be a valid pointer which points somewhere in
	 * userspace.  This can fault and call SGX or other fault handlers when
	 * userspace mapping @secs doesn't exist.
	 *
	 * Add a WARN() to make sure @secs is already valid userspace pointer
	 * from caller (KVM), who should already have handled invalid pointer
	 * case (for instance, made by malicious guest).  All other checks,
	 * such as alignment of @secs, are deferred to ENCLS itself.
	 */
	if (WARN_ON_ONCE(!access_ok(secs, PAGE_SIZE)))
		return -EINVAL;

	__uaccess_begin();
	ret = __ecreate(pageinfo, (void *)secs);
	__uaccess_end();

	if (encls_faulted(ret)) {
		*trapnr = ENCLS_TRAPNR(ret);
		return -EFAULT;
	}

	/* ECREATE doesn't return an error code, it faults or succeeds. */
	WARN_ON_ONCE(ret);
	return 0;
}
EXPORT_SYMBOL_GPL(sgx_virt_ecreate);

static int __sgx_virt_einit(void __user *sigstruct, void __user *token,
			    void __user *secs)
{
	int ret;

	/*
	 * Make sure all userspace pointers from caller (KVM) are valid.
	 * All other checks deferred to ENCLS itself.  Also see comment
	 * for @secs in sgx_virt_ecreate().
	 */
#define SGX_EINITTOKEN_SIZE	304
	if (WARN_ON_ONCE(!access_ok(sigstruct, sizeof(struct sgx_sigstruct)) ||
			 !access_ok(token, SGX_EINITTOKEN_SIZE) ||
			 !access_ok(secs, PAGE_SIZE)))
		return -EINVAL;

	__uaccess_begin();
	ret = __einit((void *)sigstruct, (void *)token, (void *)secs);
	__uaccess_end();

	return ret;
}

/**
 * sgx_virt_einit() - Run EINIT on behalf of guest
 * @sigstruct:		Userspace pointer to SIGSTRUCT structure
 * @token:		Userspace pointer to EINITTOKEN structure
 * @secs:		Userspace pointer to SECS page
 * @lepubkeyhash:	Pointer to guest's *virtual* SGX_LEPUBKEYHASH MSR values
 * @trapnr:		trap number injected to guest in case of EINIT error
 *
 * Run EINIT on behalf of guest after KVM traps EINIT. If SGX_LC is available
 * in host, SGX driver may rewrite the hardware values at wish, therefore KVM
 * needs to update hardware values to guest's virtual MSR values in order to
 * ensure EINIT is executed with expected hardware values.
 *
 * Return:
 * -  0:	EINIT was successful.
 * - <0:	on error.
 */
int sgx_virt_einit(void __user *sigstruct, void __user *token,
		   void __user *secs, u64 *lepubkeyhash, int *trapnr)
{
	int ret;

	if (!cpu_feature_enabled(X86_FEATURE_SGX_LC)) {
		ret = __sgx_virt_einit(sigstruct, token, secs);
	} else {
		preempt_disable();
		//如果 SGX_LC 在 host 中可用，SGX 驱动程序可能会随意重写硬件值，因此 KVM 需要将硬件值更新为 guest 的虚拟 MSR 值，以确保 EINIT 以预期的硬件值执行。
		sgx_update_lepubkeyhash(lepubkeyhash);

		ret = __sgx_virt_einit(sigstruct, token, secs);
		preempt_enable();
	}

	/* Propagate up the error from the WARN_ON_ONCE in __sgx_virt_einit() */
	if (ret == -EINVAL)
		return ret;

	if (encls_faulted(ret)) {
		*trapnr = ENCLS_TRAPNR(ret);
		return -EFAULT;
	}

	return ret;
}
EXPORT_SYMBOL_GPL(sgx_virt_einit);
```
* 仅有`ENCLS/ECREATE`和`ENCLS/EINIT`是需要虚拟化支持的
* 这两条指令引发 VM-Exit 时的入口函数是 `handle_encls()`
* arch/x86/kvm/vmx/sgx.c
```c
int handle_encls(struct kvm_vcpu *vcpu)
{
    u32 leaf = (u32)kvm_rax_read(vcpu);

    if (!encls_leaf_enabled_in_guest(vcpu, leaf)) {
        kvm_queue_exception(vcpu, UD_VECTOR);
    } else if (!sgx_enabled_in_guest_bios(vcpu)) {
        kvm_inject_gp(vcpu, 0);
    } else {
        if (leaf == ECREATE)
            return handle_encls_ecreate(vcpu);
        if (leaf == EINIT)
            return handle_encls_einit(vcpu);
        WARN(1, "KVM: unexpected exit on ENCLS[%u]", leaf);
        vcpu->run->exit_reason = KVM_EXIT_UNKNOWN;
        vcpu->run->hw.hardware_exit_reason = EXIT_REASON_ENCLS;
        return 0;
    }
    return 1;
}
```