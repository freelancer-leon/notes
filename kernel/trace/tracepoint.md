# Tracepoint

## 一个Tracepoint的例子

### sched_switch
* include/trace/events/sched.h
```c
...
#include <linux/tracepoint.h>
...
/*
 * Tracepoint for task switches, performed by the scheduler:
 */
TRACE_EVENT(sched_switch,

        TP_PROTO(bool preempt,
                 struct task_struct *prev,
                 struct task_struct *next),

        TP_ARGS(preempt, prev, next),

        TP_STRUCT__entry(
                __array(        char,   prev_comm,      TASK_COMM_LEN   )   
                __field(        pid_t,  prev_pid                        )   
                __field(        int,    prev_prio                       )   
                __field(        long,   prev_state                      )   
                __array(        char,   next_comm,      TASK_COMM_LEN   )   
                __field(        pid_t,  next_pid                        )   
                __field(        int,    next_prio                       )   
        ),  

        TP_fast_assign(
                memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
                __entry->prev_pid       = prev->pid;
                __entry->prev_prio      = prev->prio;
                __entry->prev_state     = __trace_sched_switch_state(preempt, prev);
                memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
                __entry->next_pid       = next->pid;
                __entry->next_prio      = next->prio;
                /* XXX SCHED_DEADLINE */
        ),

        TP_printk("prev_comm=%s prev_pid=%d prev_prio=%d prev_state=%s%s ==> next_comm=%s next_pid=%d next_prio=%d",
                __entry->prev_comm, __entry->prev_pid, __entry->prev_prio,

                (__entry->prev_state & (TASK_REPORT_MAX - 1)) ?
                  __print_flags(__entry->prev_state & (TASK_REPORT_MAX - 1), "|",
                                { 0x01, "S" }, { 0x02, "D" }, { 0x04, "T" },
                                { 0x08, "t" }, { 0x10, "X" }, { 0x20, "Z" },
                                { 0x40, "P" }, { 0x80, "I" }) :
                  "R",

                __entry->prev_state & TASK_STATE_MAX ? "+" : "",
                __entry->next_comm, __entry->next_pid, __entry->next_prio)
);
...
```
