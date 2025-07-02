# KVM MMU Notifier

## KVM MMU Invalidation 的几种场景
* `kvm_mmu_invalidate_begin()` 的调用有以下几种场景：
  1. 更新 MTRR 时调用 `kvm_zap_gfn_range()`
  2. KVM MMU notifier 作为 MMU 的观察者，当 MMU 某个 range 发生改变时收到通知
  3. 设置内存属性
     ```c
     kvm_vm_ioctl()
     case KVM_SET_MEMORY_ATTRIBUTES:
     -> kvm_vm_ioctl_set_mem_attributes()
     ```
  4. KVM memory slot 注册一个 restrictedmem 的 notifier，当 restrictedmem 被打洞时收到通知
     * `fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE, start, length)`

## Invalidation 相关操作对 KVM mmu_lock 的使用
* 以下几个函数需要持有 `kvm->mmu_lock` 的 **写锁**
  * `kvm_mmu_invalidate_begin()`
  * `kvm_mmu_invalidate_range_add(kvm, start, end)`
  * `kvm_mmu_invalidate_end()`
* 以下函数需要持有 `kvm->mmu_lock` 的 **读锁**
  * `mmu_invalidate_retry_gfn()`
* 以下函数不需要持有 `kvm->mmu_lock` 的 **读锁**
  * `mmu_invalidate_retry()`
* 为了避免长时间持有 `kvm->mmu_lock` 的 *写锁*，引入了 `kvm->mmu_invalidate_in_progress` 和 `kvm->mmu_invalidate_seq` 两个计数
  * 持有写锁的时候更新这两个计数，随即释放，不会长时间持有写锁
  * 其他订阅者只需查看这两个计数再决定要不要继续读相关的操作还是 retry
  * 如果订阅者持有读锁，意欲 invalidate 内存的线程就会在争用写锁的地方等待读锁的释放，所以订阅者读期间数据是安全的
  * 由于写锁持有的时间并不长，因此不会出现多个读者饥饿的情况，如果读到计数不对，就 retry
* `kvm->mmu_invalidate_in_progress` 计数增加必须在解锁时可见，因为如果不获取 `mmu_lock` 就无法建立 spte，并且计数也在 `mmu_lock` 临界区内读取
```c
void kvm_mmu_invalidate_begin(struct kvm *kvm)
{
    /*
     * The count increase must become visible at unlock time as no
     * spte can be established without taking the mmu_lock and
     * count is also read inside the mmu_lock critical section.
     */
    kvm->mmu_invalidate_in_progress++;

    if (likely(kvm->mmu_invalidate_in_progress == 1)) {
        kvm->mmu_invalidate_range_start = INVALID_GPA;
        kvm->mmu_invalidate_range_end = INVALID_GPA;
    }
}
```
* `kvm->mmu_invalidate_seq` 增加将通知 kvm 缺页，即将映射到 spte 中的页面可能已被释放
* 上面的 `kvm->mmu_invalidate_seq` 增加必须在下面的 `kvm->mmu_invalidate_in_progress` 计数减少之前可见，这是由上面的 `smp_wmb` 和 `mmu_invalidate_retry()` 中的 `smp_rmb` 一起确保的
```c
void kvm_mmu_invalidate_end(struct kvm *kvm)
{
    /*
     * This sequence increase will notify the kvm page fault that
     * the page that is going to be mapped in the spte could have
     * been freed.
     */
    kvm->mmu_invalidate_seq++;
    smp_wmb();
    /*
     * The above sequence increase must be visible before the
     * below count decrease, which is ensured by the smp_wmb above
     * in conjunction with the smp_rmb in mmu_invalidate_retry().
     */
    kvm->mmu_invalidate_in_progress--;
}
```
* 确保 `mmu_invalidate_in_progress` 的读取发生在 `mmu_invalidate_seq` 的读取之前
  * 这与 `mmu_notifier_invalidate_range_end()` 中的 `smp_wmb()` 交互，以确保调用者看到 `mmu_invalidate_in_progress` 的旧（非零）值或 `mmu_invalidate_seq` 的新（增加）值
```c
static inline int mmu_invalidate_retry(struct kvm *kvm, unsigned long mmu_seq)
{
    if (unlikely(kvm->mmu_invalidate_in_progress))
        return 1;
    /*
     * Ensure the read of mmu_invalidate_in_progress happens before
     * the read of mmu_invalidate_seq.  This interacts with the
     * smp_wmb() in mmu_notifier_invalidate_range_end to make sure
     * that the caller either sees the old (non-zero) value of
     * mmu_invalidate_in_progress or the new (incremented) value of
     * mmu_invalidate_seq.
     *
     * PowerPC Book3s HV KVM calls this under a per-page lock rather
     * than under kvm->mmu_lock, for scalability, so can't rely on
     * kvm->mmu_lock to keep things ordered.
     */
    smp_rmb();
    if (kvm->mmu_invalidate_seq != mmu_seq)
        return 1;
    return 0;
}
```
* 如果 `mmu_invalidate_in_progress` 不为零，则 `kvm_mmu_notifier_invalidate_range_start()` 维护的范围包含所有可能无效的地址
  * 请注意，由于处理并发 invalidations 时的捷径，它可能包含一些误报
```c
static inline int mmu_invalidate_retry_gfn(struct kvm *kvm,
                       unsigned long mmu_seq,
                       gfn_t gfn)
{
    lockdep_assert_held(&kvm->mmu_lock);
    /*
     * If mmu_invalidate_in_progress is non-zero, then the range maintained
     * by kvm_mmu_notifier_invalidate_range_start contains all addresses
     * that might be being invalidated. Note that it may include some false
     * positives, due to shortcuts when handing concurrent invalidations.
     */
    if (unlikely(kvm->mmu_invalidate_in_progress)) {
        /*
         * Dropping mmu_lock after bumping mmu_invalidate_in_progress
         * but before updating the range is a KVM bug.
         */
        if (WARN_ON_ONCE(kvm->mmu_invalidate_range_start == INVALID_GPA ||
                 kvm->mmu_invalidate_range_end == INVALID_GPA))
            return 1;

        if (gfn >= kvm->mmu_invalidate_range_start &&
            gfn < kvm->mmu_invalidate_range_end)
            return 1;
    }

    if (kvm->mmu_invalidate_seq != mmu_seq)
        return 1;
    return 0;
}
```
* `mmu_seq` 的值来自 `kvm->mmu_invalidate_seq`
  * 当有其他核在 invalidate KVM MMU 的一个范围时（`kvm->mmu_invalidate_in_progress`），本核需要 retry
  * 如果没有其他核在 invalidate KVM MMU 的一个范围，
    * `kvm->mmu_invalidate_seq == mmu_seq`，说明没有其他核和我们竞争，我们可以继续做我们想做的事情
    * `kvm->mmu_invalidate_seq != mmu_seq`，说明我们持有的数据可能过期了，需要 retry
* `kvm->mmu_invalidate_in_progress` 和 `kvm->mmu_invalidate_seq` 涉及两个核对两个内存位置的读写，需要屏障来保证顺序性
  * 这个设计就是为了避免等待数据同步的开销，查看两个计数就可以做出决定
  * 因为 `mmu_invalidate_retry()` 并未要求持有读锁，因此需要内存屏障来保证读取两个计数的顺序；
  * `mmu_invalidate_retry_gfn()` 要求持有 `kvm->mmu_lock` 读锁，因此在判断期间，争用写锁的进程会等待，不会干扰读锁这边进行判断，所以不需要内存屏障