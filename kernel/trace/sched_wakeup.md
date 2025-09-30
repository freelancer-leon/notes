# Wakeup Tracers
* 跟踪点注册
  * include/linux/tracepoint.h
```c
#define __DECLARE_TRACE_COMMON(name, proto, args, data_proto)       \
    extern int __traceiter_##name(data_proto);          \
    DECLARE_STATIC_CALL(tp_func_##name, __traceiter_##name);    \
    extern struct tracepoint __tracepoint_##name;           \
    extern void rust_do_trace_##name(proto);            \
    static inline int                       \
    register_trace_##name(void (*probe)(data_proto), void *data)    \
    {                               \
        return tracepoint_probe_register(&__tracepoint_##name,  \
                        (void *)probe, data);   \   
    }                               \
    static inline int                       \
    register_trace_prio_##name(void (*probe)(data_proto), void *data,\
                   int prio)                \
    {                               \
        return tracepoint_probe_register_prio(&__tracepoint_##name, \
                          (void *)probe, data, prio); \
    }                               \
...
```
* 唤醒跟踪点和新进程唤醒跟踪点的定义
  * include/trace/events/sched.h
```c
/*
 * Tracepoint called when the task is actually woken; p->state == TASK_RUNNING.
 * It is not always called from the waking context.
 */
DEFINE_EVENT(sched_wakeup_template, sched_wakeup,
         TP_PROTO(struct task_struct *p),
         TP_ARGS(p));

/*
 * Tracepoint for waking up a new task:
 */
DEFINE_EVENT(sched_wakeup_template, sched_wakeup_new,
         TP_PROTO(struct task_struct *p),
         TP_ARGS(p));
```
* 插入跟踪点
  * kernel/sched/core.c
```c
/*
 * Mark the task runnable.
 */
static inline void ttwu_do_wakeup(struct task_struct *p)
{
    WRITE_ONCE(p->__state, TASK_RUNNING);
    trace_sched_wakeup(p);
}
...
/*
 * wake_up_new_task - wake up a newly created task for the first time.
 *
 * This function will do some initial scheduler statistics housekeeping
 * that must be done for every newly created context, then puts the task
 * on the runqueue and wakes it.
 */
void wake_up_new_task(struct task_struct *p)
{
...
    activate_task(rq, p, ENQUEUE_NOCLOCK | ENQUEUE_INITIAL);
    trace_sched_wakeup_new(p);
    wakeup_preempt(rq, p, wake_flags);
...
}
```
* 注册唤醒时的 tracer
  * kernel/trace/trace_sched_wakeup.c
```c
static void start_wakeup_tracer(struct trace_array *tr)
{
    int ret;

    ret = register_trace_sched_wakeup(probe_wakeup, NULL);
    if (ret) {
        pr_info("wakeup trace: Couldn't activate tracepoint"
            " probe to kernel_sched_wakeup\n");
        return;
    }   

    ret = register_trace_sched_wakeup_new(probe_wakeup, NULL);
    if (ret) {
        pr_info("wakeup trace: Couldn't activate tracepoint"
            " probe to kernel_sched_wakeup_new\n");
        goto fail_deprobe;
    }   

    ret = register_trace_sched_switch(probe_wakeup_sched_switch, NULL);
    if (ret) {
        pr_info("sched trace: Couldn't activate tracepoint"
            " probe to kernel_sched_switch\n");
        goto fail_deprobe_wake_new;
    }   

    ret = register_trace_sched_migrate_task(probe_wakeup_migrate_task, NULL);
    if (ret) {
        pr_info("wakeup trace: Couldn't activate tracepoint"
            " probe to kernel_sched_migrate_task\n");
        goto fail_deprobe_sched_switch;
    }

    wakeup_reset(tr);

    /*
     * Don't let the tracer_enabled = 1 show up before
     * the wakeup_task is reset. This may be overkill since
     * wakeup_reset does a spin_unlock after setting the
     * wakeup_task to NULL, but I want to be safe.
     * This is a slow path anyway.
     */
    smp_wmb();

    if (start_func_tracer(tr, is_graph(tr)))
        printk(KERN_ERR "failed to start wakeup tracer\n");

    return;
fail_deprobe_sched_switch:
    unregister_trace_sched_switch(probe_wakeup_sched_switch, NULL);
fail_deprobe_wake_new:
    unregister_trace_sched_wakeup_new(probe_wakeup, NULL);
fail_deprobe:
    unregister_trace_sched_wakeup(probe_wakeup, NULL);
}
```
* 记录唤醒事件
  * kernel/trace/trace_sched_wakeup.c
```c
static void
tracing_sched_wakeup_trace(struct trace_array *tr,
               struct task_struct *wakee,
               struct task_struct *curr,
               unsigned int trace_ctx)
{
    struct ring_buffer_event *event;
    struct ctx_switch_entry *entry;
    struct trace_buffer *buffer = tr->array_buffer.buffer;

    event = trace_buffer_lock_reserve(buffer, TRACE_WAKE,
                      sizeof(*entry), trace_ctx);
    if (!event)
        return;
    entry   = ring_buffer_event_data(event);
    entry->prev_pid         = curr->pid;
    entry->prev_prio        = curr->prio;
    entry->prev_state       = task_state_index(curr);
    entry->next_pid         = wakee->pid;
    entry->next_prio        = wakee->prio;
    entry->next_state       = task_state_index(wakee);
    entry->next_cpu         = task_cpu(wakee);

    trace_buffer_unlock_commit(tr, buffer, event, trace_ctx);
}
...
static void
probe_wakeup(void *ignore, struct task_struct *p) 
{
    struct trace_array_cpu *data;
    int cpu = smp_processor_id();
    long disabled;
    unsigned int trace_ctx;

    if (likely(!tracer_enabled))
        return;

    tracing_record_cmdline(p);
    tracing_record_cmdline(current);

    /*  
     * Semantic is like this:
     *  - wakeup tracer handles all tasks in the system, independently
     *    from their scheduling class;
     *  - wakeup_rt tracer handles tasks belonging to sched_dl and
     *    sched_rt class;
     *  - wakeup_dl handles tasks belonging to sched_dl class only.
     */
    if (tracing_dl || (wakeup_dl && !dl_task(p)) ||
        (wakeup_rt && !rt_or_dl_task(p)) ||
        (!dl_task(p) && (p->prio >= wakeup_prio || p->prio >= current->prio)))
        return;

    disabled = local_inc_return(&per_cpu_ptr(wakeup_trace->array_buffer.data, cpu)->disabled);
    if (unlikely(disabled != 1)) 
        goto out;

    trace_ctx = tracing_gen_ctx();

   /* interrupts should be off from try_to_wake_up */
    arch_spin_lock(&wakeup_lock);

    /* check for races. */
    if (!tracer_enabled || tracing_dl ||
        (!dl_task(p) && p->prio >= wakeup_prio))
        goto out_locked;

    /* reset the trace */
    __wakeup_reset(wakeup_trace);

    wakeup_cpu = task_cpu(p);
    wakeup_current_cpu = wakeup_cpu;
    wakeup_prio = p->prio;

    /*
     * Once you start tracing a -deadline task, don't bother tracing
     * another task until the first one wakes up.
     */
    if (dl_task(p))
        tracing_dl = true;
    else
        tracing_dl = false;

    wakeup_task = get_task_struct(p);

    data = per_cpu_ptr(wakeup_trace->array_buffer.data, wakeup_cpu);
    data->preempt_timestamp = ftrace_now(cpu);
    tracing_sched_wakeup_trace(wakeup_trace, p, current, trace_ctx);
    __trace_stack(wakeup_trace, trace_ctx, 0);

    /*
     * We must be careful in using CALLER_ADDR2. But since wake_up
     * is not called by an assembly function  (where as schedule is)
     * it should be safe to use it here.
     */
    __trace_function(wakeup_trace, CALLER_ADDR1, CALLER_ADDR2, trace_ctx);

out_locked:
    arch_spin_unlock(&wakeup_lock);
out:
    local_dec(&per_cpu_ptr(wakeup_trace->array_buffer.data, cpu)->disabled);
}
```