# Notes
***
# Architectures
## ARM
- [ARM64 的一些基本概念](arch/arm64/basic.md)

## Intel
- [AMX](arch/intel/amx.md)
- [APIC](arch/intel/apic.md)
- [APIC 虚拟化和虚拟中断](arch/intel/apicv_vintr.md)
- [Buslock](arch/intel/buslock.md)
- [Memory Cache Control](arch/intel/cache_control.md)
- [CPU 拓扑枚举](arch/intel/cpu_topo.md)
- [Multi-Processors Management](arch/intel/MP_management.md)
- [Linear Address Masking (LAM)](arch/intel/lam)
- [TDX](arch/intel/tdx)
- [SGX](arch/intel/sgx)
- [RDT](arch/intel/rdt)

### VT-d
- [Vt-d Overview](arch/intel/vt-d/vt-d-ov.md)

# Computer Science
- [Cache](computer_science/cache.md)
- [TLB](computer_science/tlb.md)
- [浮点数](computer_science/float.md)
- [MESI 协议](computer_science/MESI.md)

# Kernel
- [Bottom Half 下半部](kernel/Bottom_Half.md)
- [cmwq](kernel/cmwq.md)
- [时间和定时器](kernel/time.md)
- [虚拟文件系统（VFS）](kernel/vfs.md)
- [页缓存和页回写](kernel/page_cache.md)
- [块I/O层（BIO）](kernel/bio.md)
- [设备模型](kernel/dev_model.md)
- [Linux 图形](kernel/graphic/Linux-Graphic.md)
- [中断](kernel/irq.md)
- [x86 内核栈](kernel/stack_x86-64.md)
- [系统调用](kernel/syscall.md)
- [NMI](kernel/nmi/nmi.md)
- [IPI - x86](kernel/ipi.md)
- [Kexec](kernel/kexec.md)
- [Kexec - x86](kernel/kexec_x86.md)
- [Kdump](kernel/kdump.md)
- [MCE](kernel/mce.md)
- [内核线程及 Stop Machine](kernel/stop_machine.md)
- [CPU 热插拔](kernel/cpu_hotplug.md)
- [段寻址](kernel/segmentation.md)
- [基于作用域的资源管理](kernel/cleanup.md)
- [cpumask](kernel/cpumask.md)
- [一些内核配置](kernel/config.md)

## 锁
- [锁的基本知识](kernel/lock/Lock-1.md)
- [Linux 自旋锁的实现](kernel/lock/Lock-2-Linux_x86_Spin_Lock.md)
- [Linux 信号量（Semaphore）的实现](kernel/lock/Lock-3-Linux_Semaphore.md)
- [Linux Mutex 的实现](kernel/lock/Lock-4-Linux_Mutex.md)
- [自旋锁的新实现 - qspinlock](kernel/lock/qspinlock.md)
- [乐观自旋锁 osq_lock 的实现](kernel/lock/osq_lock.md)

## 进程调度
- [进程调度](kernel/sched/sched_intro.md)
- [UNIX进程调度](kernel/sched/sched_unix.md)
- [Linux进程调度（Linux Process Scheduling）](kernel/sched/sched_linux.md)
- [完全公平调度（Completely Fair Scheduler）](kernel/sched/sched_cfs.md)
- [CFS - 1](kernel/sched/sched_cfs-1.md)
- [实时调度](kernel/sched/sched_rt.md)
- [实时调度负载均衡](kernel/sched/sched_rt_load_balance.md)
- [EEVDF](kernel/sched/sched_eevdf.md)
- [Deadline](kernel/sched/sched_dl.md)
- [内核抢占](kernel/sched/sched_kernel_preempt.md)

## 内存管理
- [内存管理](kernel/mm/mm.md)
- [分页管理](kernel/mm/mm_pagetable.md)
- [进程地址空间](kernel/mm/mm-1-process_addr_spc.md)
- [slab](kernel/mm/slab.md)
- [slub](kernel/mm/slub.md)
- [学习 /proc/meminfo](kernel/mm/meminfo.md)
- [内核中的直接映射](kernel/mm/direct_map.md)
- [缺页处理](kernel/mm/fault.md)
- [Folio](kernel/mm/folio.md)
- [Vmalloc](kernel/mm/vmalloc.md)
- [Per-CPU](kernel/mm/percpu.md)
- [异常修正表](kernel/mm/fixup.md)

## 网络
- [Monitoring and Tuning the Linux Networking Stack: Receiving Data](kernel/networking/monitoring-tuning-linux-networking-stack-receiving-data.md)
- [TCP](kernel/networking/TCP.md)
- [macvlan](kernel/networking/macvlan.md)
- [ARP](kernel/networking/arp.md)
- [GARP](kernel/networking/garp.md)

### Netfilter
- [netfilter](kernel/networking/netfilter/netfilter.md)
- [conntrack](kernel/networking/netfilter/conntrack.md)

## 初始化
- [initcall](kernel/init/initcall.md)

## 性能诊断
- [Perf](kernel/profiling/perf.md)

## 跟踪
- [Ftrace](kernel/trace/ftrace.md)
- [Ftrace设计](kernel/trace/ftrace-design.md)

## 调试
- [Debug](kernel/debug/debug.md)
- [kprobes](kernel/debug/kprobes.md)

## 主题
- [KPTI](kernel/issues/kpti/kpti.md)

# Virtualization
- [docker](virtualization/docker/docker.md)
## KVM
- [APIC 虚拟化](virtualization/kvm/apicv.md)
- [虚拟化异常](virtualization/kvm/exception.md)
- [Posted Interrupt](virtualization/kvm/posted_interrupt.md)
- [TDP MMU 缺页](virtualization/kvm/tdp_page_fault.md)
- [Guest First Memory](virtualization/kvm/gmem.md)