From c4f5c92849b6648e30ed3930982813e094119245 Mon Sep 17 00:00:00 2001
From: Scott Wood <swood@redhat.com>
Date: Sat, 12 Oct 2019 01:52:13 -0500
Subject: [PATCH] sched: Lazy migrate_disable processing

[ Upstream commit 425c5b38779a860062aa62219dc920d374b13c17 ]

Avoid overhead on the majority of migrate disable/enable sequences by
only manipulating scheduler data (and grabbing the relevant locks) when
the task actually schedules while migrate-disabled.  A kernel build
showed around a 10% reduction in system time (with CONFIG_NR_CPUS=512).

通过仅在任务在 *禁用迁移的情况下进行实际调度* 时才操作调度器数据（并获取相关的锁），从而避免了大多数迁移禁用/启用序列的开销。在有的内核上显示系统时间减少了约10％（CONFIG_NR_CPUS = 512）。

Instead of cpuhp_pin_lock, CPU hotplug is handled by keeping a per-CPU
count of the number of pinned tasks (including tasks which have not
scheduled in the migrate-disabled section); takedown_cpu() will
wait until that reaches zero (confirmed by take_cpu_down() in stop
machine context to deal with races) before migrating tasks off of the
cpu.

取代了 `cpuhp_pin_lock`，通过维持一个 per-CPU pinned task 的计数（包括尚未在“禁用迁移”区间中被调度的任务）来处理 CPU 热插拔；`takedown_cpu()`将等到该计数达到零（由`stop_cpu_down()`在 stop machine 上下文中确认以处理竞争），然后再将任务迁移出该 CPU。

To simplify synchronization, updating cpus_mask is no longer deferred
until migrate_enable().  This lets us not have to worry about
migrate_enable() missing the update if it's on the fast path (didn't
schedule during the migrate disabled section).  It also makes the code
a bit simpler and reduces deviation from mainline.

为了简化同步，更新`cpus_mask`不再推迟到`migration_enable()`。 这使我们不必担心如果位于快速路径上的`migrate_enable()`缺少更新（在“禁用迁移”区间未被调度）。 这还使代码更简单，并减少了与主线的偏差。

While the main motivation for this is the performance benefit, lazy
migrate disable also eliminates the restriction on calling
migrate_disable() while atomic but leaving the atomic region prior to
calling migrate_enable() -- though this won't help with local_bh_disable()
(and thus rcutorture) unless something similar is done with the recently
added local_lock.

尽管这样做的主要动机是提高性能，但延迟迁移禁用还消除了在原子上下文调用`migration_disable()`的限制，而是把原子区域留在调用`migrate_enable()`之前－尽管这对`local_bh_disable()`（并因此而被撤销）无济于事，除非对最近添加的 local_lock 也做了类似的操作。

Signed-off-by: Scott Wood <swood@redhat.com>
Signed-off-by: Sebastian Andrzej Siewior <bigeasy@linutronix.de>
Signed-off-by: Steven Rostedt (VMware) <rostedt@goodmis.org>

* `p->migrate_disable_scheduled`为 1 用于表示禁止迁移的进程被调度了，这就意味着被延迟修改调度器的操作已经发生过，因此在`migrate_enable()`的时候需要真正地恢复被修改过的 cpumask 数据。
