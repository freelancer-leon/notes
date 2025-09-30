# Hardware Latency Detector
* `hwlat` tracer 是 ftrace 中一个非常有用的跟踪器，专门用于检测和测量由于硬件（如 CPU、内存、电源管理等）引入的系统延迟。
* 这些延迟通常被称为“系统管理模式”或“电源管理中断”等造成的，表现为在关中断的情况下，内核代码执行被硬件事件“抢占”了一小段时间。

## `hwlat` 工作原理简介
* 启用此跟踪器后，系统将根据 `cpumask` 文件的设置创建一个或多个内核线程；每个线程会以循环方式运行，用于检测由内核以外的因素引发的中断。
  * 例如，若系统管理中断（SMI）的执行耗时达到可察觉的程度，此跟踪器便能将其检测出来。
* 该跟踪器可用于测试系统是否能稳定支持实时任务（Real Time tasks）。
* 启用此跟踪器后，跟踪目录（tracing directory）下会生成以下文件：
  * `hwlat_detector/width`：循环运行的持续时间，单位为微秒（usecs）。
  * `hwlat_detector/window`：每次循环迭代开始的时间间隔，单位为微秒（usecs）。
* 系统会创建一个内核线程，该线程在每个“`window`”周期内，会以“**禁用中断**”的状态循环运行“`width`”微秒；其余“`window - width`”微秒内不进行循环运行，此期间系统可正常执行其他操作。
* 跟踪结果将输出到 `trace` 和 `trace_pipe` 文件中。
* 当跟踪器未运行时，它对系统无任何影响；但在运行状态下，可能导致系统周期性失去响应。**请勿在生产系统上运行此跟踪器**。

## `hwlat` 的使用方法
* 在 `/sys/kernel/debug/tracing/tracing_thresh` 文件中，可以设置延迟阈值。
  * 只有当检测到的延迟超过这个阈值时，才会被记录到跟踪缓冲区中
* 启用此跟踪器的方法：向 `current_tracer` 文件中写入“`hwlat`”（命令格式示例：`echo hwlat > current_tracer`）。
* 每当延迟时间超过` tracing_thresh`（跟踪阈值）时，该延迟事件会被记录到环形缓冲区（ring buffer）中。

```c
# cat  trace
# tracer: hwlat
#
# entries-in-buffer/entries-written: 87/87   #P:8
#
#                                _-----=> irqs-off/BH-disabled
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
           <...>-12494   [001] d.... 13926.739078: #1     inner/outer(us):   96/83    ts:1759213128.708484095 count:40
           <...>-12494   [002] d.... 13928.039575: #2     inner/outer(us):   97/102   ts:1759213129.992901100 count:40
           <...>-12494   [003] d.... 13929.040825: #3     inner/outer(us):   83/73    ts:1759213130.993780714 count:38
           <...>-12494   [004] dn... 13930.048923: #4     inner/outer(us):   85/84    ts:1759213132.001614057 count:40
```
* **TIMESTAMP**：该条采样数据记录到跟踪缓冲区的时间戳
* **FUNCTION**：检测到延迟事件的采用窗口的序列号。表示从跟踪开始以来，这是第几次检测到的延迟的采样窗口的计数，而不是延迟事件的计数。
* **inner**：==**内部延迟**，`t2 - t1`，检测硬件在极短时间内的干扰，这个是观察的重点，直接反映了延迟的严重程度==
* **outer**：外部延迟，`本次 t1 - 上次 t2`，检测采样循环之间的干扰，包含循环处理采样的时间
* **ts**：第一次超阈值时记录时间戳，一个原始的、架构相关的高精度时间戳（例如 TSC）。在进行非常精细的分析时可能有用，但对于一般性解读，上面的时间戳已经足够
* **count**：在一个采样窗口内检查到的延迟事件的计数

## 核心逻辑
* 双时间戳检测：通过连续读取 `t1` 和 `t2` 来检测：
  * 内部延迟：`t2 - t1`，检测硬件在极短时间内的干扰
  * 外部延迟：`本次 t1 - 上次 t2`，检测采样循环之间的干扰，包含处理采样的时间
* 阈值比较：只有超过 `tracing_thresh` 的延迟才会被记录
* NMI 统计：在采样期间启用 NMI 回调，统计不可屏蔽中断的影响
* 采样窗口：在固定的时间窗口内持续采样，持续时间为 *采样宽度*，寻找最大延迟值
* 结果记录：检测到延迟时，将详细信息记录到跟踪缓冲区用于后续分析
  * kernel/trace/trace_hwlat.c
```c
/**
 * get_sample - sample the CPU TSC and look for likely hardware latencies
 *              采样 CPU TSC 并寻找可能的硬件延迟
 * Used to repeatedly capture the CPU TSC (or similar), looking for potential
 * hardware-induced latency. Called with interrupts disabled and with
 * hwlat_data.lock held.
 * 用于重复捕获 CPU TSC（或类似计数器），寻找潜在的硬件引起的延迟。
 * 在中断禁用和持有 hwlat_data.lock 的情况下调用。
 */
static int get_sample(void)
{
	struct hwlat_kthread_data *kdata = get_cpu_data();
	struct trace_array *tr = hwlat_trace;
	struct hwlat_sample s;            //用于存储采样结果的数据结构
	time_type start, t1, t2, last_t2; //时间戳变量
	s64 diff, outer_diff, total, last_total = 0;
	u64 sample = 0;                   //记录的最大内部延迟（微秒）
	u64 thresh = tracing_thresh;      //延迟阈值（纳秒）
	u64 outer_sample = 0;             //记录的最大外部延迟（微秒）
	int ret = -1;                     //返回值：-1=错误，0=无延迟，1=检测到延迟
	unsigned int count = 0;           //超过阈值的采样点数量
    //将阈值从纳秒转换为微秒，因为后续计算使用微秒
	do_div(thresh, NSEC_PER_USEC); /* modifies interval value */
    //初始化 NMI（不可屏蔽中断）统计
	kdata->nmi_total_ts = 0;
	kdata->nmi_count = 0;
	/* Make sure NMIs see this first */
	barrier(); //内存屏障：确保 NMI 处理程序能看到上面的初始化
    //启用硬件延迟回调，允许 NMI 处理程序记录时间
	trace_hwlat_callback_enabled = true;
    //初始化上次的外部时间戳
	init_time(last_t2, 0);
	start = time_get(); /* start timestamp */ //获取采样窗口开始时间戳
	outer_diff = 0;
    //主采样循环：在指定的采样宽度内持续采样
	do {

		t1 = time_get();	/* we'll look for a discontinuity */ //第一个时间戳
		t2 = time_get();    //紧接着的第二个时间戳
        //如果不是第一次迭代，计算外部循环的延迟
		if (time_u64(last_t2)) {
			/* Check the delta from outer loop (t2 to next t1) */
			outer_diff = time_to_us(time_sub(t1, last_t2));//检查外部循环的延迟（从上次的 t2 到当前的 t1）
			/* This shouldn't happen */
			if (outer_diff < 0) { //时间不应倒流，如果发生则报错
				hwlat_err(BANNER "time running backwards\n");
				goto out;
			}
			if (outer_diff > outer_sample) //如果本次循环的外部延迟大于记录的最大的外部延迟
				outer_sample = outer_diff; //更新最大的外部延迟
		}
		last_t2 = t2; //更新上次的 t2 时间戳
        //计算从采样开始到现在的总时间（采样窗口进度）
		total = time_to_us(time_sub(t2, start)); /* sample width */
        //检查时间溢出（不应发生）
		/* Check for possible overflows */
		if (total < last_total) {
			hwlat_err("Time total overflowed\n");
			break;
		}
		last_total = total;
        //检查内部循环的延迟（t1 到 t2 的时间差）
		/* This checks the inner loop (t1 to t2) */
		diff = time_to_us(time_sub(t2, t1));     /* current diff */
        //如果内部或外部延迟超过阈值，记录相关信息
		if (diff > thresh || outer_diff > thresh) {
			if (!count)
				ktime_get_real_ts64(&s.timestamp); //第一次超阈值时记录时间戳
			count++; //增加超阈值计数
		}
        //再次检查时间不应倒流
		/* This shouldn't happen */
		if (diff < 0) {
			hwlat_err(BANNER "time running backwards\n");
			goto out;
		}
        //记录最大的内部延迟
		if (diff > sample) //如果本次循环的内部延迟大于记录的最大的内部延迟
			sample = diff; /* only want highest value */

	} while (total <= hwlat_data.sample_width); //循环直到达到采样窗口宽度
    //内存屏障：确保 NMI 处理程序能看到 trace_hwlat_callback_enabled 的更改
	barrier(); /* finish the above in the view for NMIs */
	trace_hwlat_callback_enabled = false; //禁用硬件延迟回调
	barrier(); /* Make sure nmi_total_ts is no longer updated */
    //默认返回无错误
	ret = 0;
    //如果内部或外部延迟超过阈值，说明检测到了硬件延迟
	/* If we exceed the threshold value, we have found a hardware latency */
	if (sample > thresh || outer_sample > thresh) {
		u64 latency;
        //返回检测到延迟
		ret = 1;
        //将 NMI 总时间从纳秒转换为微秒
		/* We read in microseconds */
		if (kdata->nmi_total_ts)
			do_div(kdata->nmi_total_ts, NSEC_PER_USEC);
        //填充采样数据结构
		hwlat_data.count++;
		s.seqnum = hwlat_data.count;          //序列号
		s.duration = sample;                  //最大内部延迟
		s.outer_duration = outer_sample;      //最大外部延迟
		s.nmi_total_ts = kdata->nmi_total_ts; //NMI 总时间
		s.nmi_count = kdata->nmi_count;       //NMI 计数
		s.count = count;                      //超阈值采样点数量
		trace_hwlat_sample(&s);               //将一条采样数据记录到跟踪缓冲区
        //计算总延迟（取内部和外部延迟的最大值）
		latency = max(sample, outer_sample);
        //更新跟踪器记录的最大延迟
		/* Keep a running maximum ever recorded hardware latency */
		if (latency > tr->max_latency) {
			tr->max_latency = latency;
			latency_fsnotify(tr);  //通知文件系统更新
		}
	}

out:
	return ret;
}
```
* trace 记录构造函数 `trace_hwlat_sample()`
```c
static void trace_hwlat_sample(struct hwlat_sample *sample)
{
    struct trace_array *tr = hwlat_trace;
    struct trace_buffer *buffer = tr->array_buffer.buffer;
    struct ring_buffer_event *event;
    struct hwlat_entry *entry;

    event = trace_buffer_lock_reserve(buffer, TRACE_HWLAT, sizeof(*entry),
                      tracing_gen_ctx());
    if (!event)
        return;
    entry   = ring_buffer_event_data(event);
    entry->seqnum           = sample->seqnum;
    entry->duration         = sample->duration;
    entry->outer_duration       = sample->outer_duration;
    entry->timestamp        = sample->timestamp;
    entry->nmi_total_ts     = sample->nmi_total_ts;
    entry->nmi_count        = sample->nmi_count;
    entry->count            = sample->count;

    trace_buffer_unlock_commit_nostack(buffer, event);
}
```
* 打印 trace 记录函数 `trace_hwlat_print()`
  * kernel/trace/trace_output.c
```c
trace_hwlat_print(struct trace_iterator *iter, int flags,
          struct trace_event *event)
{
    struct trace_entry *entry = iter->ent;
    struct trace_seq *s = &iter->seq;
    struct hwlat_entry *field;

    trace_assign_type(field, entry);

    trace_seq_printf(s, "#%-5u inner/outer(us): %4llu/%-5llu ts:%lld.%09ld count:%d",
             field->seqnum,
             field->duration,
             field->outer_duration,
             (long long)field->timestamp.tv_sec,
             field->timestamp.tv_nsec, field->count);

    if (field->nmi_count) {
        /*   
         * The generic sched_clock() is not NMI safe, thus
         * we only record the count and not the time.
         */
        if (!IS_ENABLED(CONFIG_GENERIC_SCHED_CLOCK))
            trace_seq_printf(s, " nmi-total:%llu",
                     field->nmi_total_ts);
        trace_seq_printf(s, " nmi-count:%u",
                 field->nmi_count);
    }    

    trace_seq_putc(s, '\n');

    return trace_handle_return(s);
}
```