# Ftrace

* 旧版本的 kernel 在使用 ftrace 之前需挂载 debugfs
	```
	# mkdir /debug
	# mount -t debugfs nodev /debug
	```
* 新版本的 kernel 无需挂载 debugfs，控制接口在`/sys/kernel/debug/tracing`目录下。
* 检查可用的跟踪器`available_tracers`和当前的跟踪器`current_tracer`
	```
	# cat /sys/kernel/debug/tracing/available_tracers
	blk mmiotrace branch function_graph wakeup_dl wakeup_rt wakeup preemptirqsoff preemptoff irqsoff function nop
	# cat /sys/kernel/debug/tracing/available_tracers
	nop
	```

## 内核选项
* 务必使`CONFIG_HAVE_FUNCTION_TRACER=y`
* 关闭`CONFIG_DYNAMIC_FTRACE`会影响性能，务必保持开启
* 通过 menuconfig 使能不同的跟踪器（tracer）

# 传统 tracer

## function tracer

#### 设定函数跟踪器
```sh
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo function > /sys/kernel/debug/tracing/current_tracer
```

#### 设定跟踪特定的函数
```sh
echo do_page_fault > /sys/kernel/debug/tracing/set_ftrace_filter
```
* 当然可以不设置，那样会跟踪所有能跟踪的函数。

#### 指定跟踪特定的CPU
```sh
echo "cpu-id" > /sys/kernel/debug/tracing/tracing_cpumask
```

#### 开始跟踪
```sh
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe | tee /tmp/ftrace.log
```

#### 输出示例
```c
            TASK-PID CPU irqs-off need-resched hardirq/softirq preempt-depth delay TIMESTAMP FUNCTION
            lttng-879   [000] d..1 2437395.004331: do_page_fault <-do_PrefetchAbort
            lttng-879   [000] d..1 2437395.004331: do_page_fault <-do_PrefetchAbort
            lttng-879   [000] d..1 2437395.004346: do_page_fault <-do_DataAbort
            lttng-879   [000] d..1 2437395.004362: do_page_fault <-do_PrefetchAbort
            lttng-879   [000] d..1 2437395.004377: do_page_fault <-do_PrefetchAbort
```

#### 关闭跟踪
```sh
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```
* 注意，`tracing_on`只是关闭了一个开关，将`current_tracer`设置为`nop`才能相对彻底地关闭动态跟踪，`mcount`函数内容被替换成`nop`指令，不再额外占用过多的 CPU 开销。

### 控制接口文件
* **set_graph_function** 设置要清晰显示调用关系的函数，显示的信息结构类似于 C 语言代码，这样在分析内核运作流程时会更加直观一些。在使用 `function_graph` 跟踪器时使用；缺省为对所有函数都生成调用关系序列，可以通过写该文件来指定需要特别关注的函数。
* **buffer_size_kb** 用于设置单个 CPU 所使用的跟踪缓存的大小。跟踪器会将跟踪到的信息写入缓存，每个 CPU 的跟踪缓存是一样大的。跟踪缓存实现为环形缓冲区的形式，如果跟踪到的信息太多，则旧的信息会被新的跟踪信息覆盖掉。注意，要更改该文件的值需要先将 `current_tracer` 设置为 `nop` 才可以。
* **available_filter_functions** 只读文件，记录了当前可以跟踪的内核函数。对于不在该文件中列出的函数，无法跟踪其活动。
* **set_ftrace_filter** 和 **set_ftrace_notrace** 在编译内核时配置了动态 ftrace （选中 `CONFIG_DYNAMIC_FTRACE` 选项）后使用。
	* 前者用于显示指定要跟踪的函数，后者则作用相反，用于指定不跟踪的函数。
	* 如果一个函数名同时出现在这两个文件中，则这个函数的执行状况不会被跟踪。
	* 这些文件还支持简单形式的含有通配符的表达式，这样可以用一个表达式一次指定多个目标函数。
	* 注意，要写入这两个文件的函数名必须可以在文件 `available_filter_functions` 中看到。
	* 缺省为可以跟踪所有内核函数，文件 `set_ftrace_notrace` 的值则为空。

## function_graph tracer
#### 设定函数跟踪器
```sh
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo function_graph > /sys/kernel/debug/tracing/current_tracer
```

#### 设定跟踪特定的进程
```sh
echo 822 > /sys/kernel/debug/tracing/set_ftrace_pid
```
* 当然可以不设置，那样会跟踪所有能跟踪的进程。

#### 开始跟踪
```sh
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe | tee /tmp/ftrace.log
```

#### 输出示例
```c
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 0)   1.623 us    |                      } /* io_serial_out */
 0)   1.633 us    |                      io_serial_out();
 0)   1.633 us    |                      io_serial_out();
 0)   1.623 us    |                      io_serial_out();
 0)   1.633 us    |                      io_serial_out();
 0)   1.633 us    |                      io_serial_out();
 0)   1.634 us    |                      io_serial_out();
 0) + 43.280 us   |                    } /* serial8250_tx_chars */
 0)               |                    _raw_spin_unlock_irqrestore() {
 0)   0.150 us    |                      preempt_count_sub();
 0)   1.332 us    |                    }
 0) + 55.423 us   |                  } /* serial8250_handle_irq.part.13 */
 0) + 59.165 us   |                } /* serial8250_default_handle_irq */
 0)               |                serial8250_default_handle_irq() {
 0)   1.623 us    |                  io_serial_in();
 0)   2.736 us    |                }
 0)               |                _raw_spin_unlock() {
 0)   0.135 us    |                  preempt_count_sub();
 0)   1.257 us    |                }
 0) + 68.529 us   |              } /* serial8250_interrupt */
 0)   0.195 us    |              add_interrupt_randomness();
 0)   0.380 us    |              note_interrupt();
 0) + 72.356 us   |            } /* handle_irq_event_percpu */
 0)               |            _raw_spin_lock() {
 0)   0.141 us    |              preempt_count_add();
 0)   1.273 us    |            }
 0) + 77.977 us   |          } /* handle_irq_event */
 0)               |          _raw_spin_unlock() {
 0)   0.170 us    |            preempt_count_sub();
 0)   1.397 us    |          }
 0) + 88.338 us   |        } /* handle_edge_irq */
 0) + 90.883 us   |      } /* handle_irq */
 0)               |      irq_exit() {
 0)               |        vtime_gen_account_irq_exit() {
 0)               |          _raw_spin_lock() {
 0)   0.140 us    |            preempt_count_add();
 0)   1.257 us    |          }
 0)               |          __vtime_account_system() {
 0)   0.356 us    |            get_vtime_delta();
 0)               |            account_system_time() {
 0)               |              cpuacct_account_field() {
 0)   0.125 us    |                __rcu_read_lock();
 0)   0.140 us    |                __rcu_read_unlock();
 0)   2.374 us    |              }
 0)               |              acct_account_cputime() {
 0)               |                __acct_update_integrals() {
 0)   0.135 us    |                  jiffies_to_timeval();
 0)   1.312 us    |                }
 0)   2.455 us    |              }
 0)   6.938 us    |            }
 0)   9.328 us    |          }
 0)               |          _raw_spin_unlock() {
 0)   0.140 us    |            preempt_count_sub();
 0)   1.563 us    |          }
 0) + 15.345 us   |        }
 0)   0.205 us    |        preempt_count_sub();
 0)   0.136 us    |        idle_cpu();
 0)   0.245 us    |        rcu_irq_exit();
 0) + 20.360 us   |      }
 0) ! 133.912 us  |    } /* do_IRQ */
 0)   <========== |
 0)   ==========> |
 0)               |    smp_apic_timer_interrupt() {
 0)               |      irq_enter() {
 0)   0.280 us    |        rcu_irq_enter();
 0)               |        vtime_common_account_irq_enter() {
 0)               |          vtime_account_user() {
 0)               |            _raw_spin_lock() {
 0)   0.206 us    |              preempt_count_add();
 0)   1.588 us    |            }
 0)   0.446 us    |            get_vtime_delta();
 0)               |            account_user_time() {
 0)               |              cpuacct_account_field() {
 0)   0.125 us    |                __rcu_read_lock();
 0)   0.150 us    |                __rcu_read_unlock();
 0)   2.725 us    |              }
 0)               |              acct_account_cputime() {
 0)               |                __acct_update_integrals() {
 0)   0.135 us    |                  jiffies_to_timeval();
 0)   1.293 us    |                }
 0)   2.400 us    |              }
 0)   8.076 us    |            }
 0)               |            _raw_spin_unlock() {
 0)   0.195 us    |              preempt_count_sub();
 0)   1.347 us    |            }
 0) + 15.825 us   |          }
 0) + 17.008 us   |        }
 0)   0.176 us    |        preempt_count_add();
 0) + 20.871 us   |      }
 0)   0.125 us    |      exit_idle();
 ...
```

# Non-Tracer Tracer
* 从 2.6.30 开始，ftrace 还支持几种 Non-tracer tracer，所谓 Non-tracer tracer 主要包括以下几种：
	* Max Stack Tracer
	* Profiling (branches / unlikely / likely / Functions)
	* Event tracing

## Max Stack Tracer
* 这个 tracer 记录内核函数的堆栈使用情况
* 用如下命令打开该 tracer
	```
	echo 1 > /proc/sys/kernel/stack_tracer_enabled
	```
* 从此，ftrace 便留心记录内核函数的堆栈使用。 Max Stack Tracer 的输出在 stack_trace 文件中：
	```c
	# cat /sys/kernel/debug/tracing/stack_trace
	        Depth    Size   Location    (25 entries)
	        -----    ----   --------
	  0)     2912     216   update_group_capacity+0x33/0x2b0
	  1)     2696      64   msecs_to_jiffies+0x5/0x20
	  2)     2632     120   update_group_capacity+0x33/0x2b0
	  3)     2512     368   find_busiest_group+0x103/0x9a0
	  4)     2144     288   load_balance+0x1bd/0xaa0
	  5)     1856     128   pick_next_task_fair+0x2df/0x4f0
	  6)     1728     112   __schedule+0x6c4/0xf60
	  7)     1616      32   schedule+0x35/0xa0
	  8)     1584     176   schedule_timeout+0x234/0x300
	  9)     1408      64   io_schedule_timeout+0xa4/0x110
	 10)     1344      96   wait_for_common_io.constprop.2+0x94/0x110
	 11)     1248      16   wait_for_completion_io+0x18/0x20
	 12)     1232     176   blk_execute_rq+0x7c/0x100
	 13)     1056      80   scsi_execute+0x131/0x1b0
	 14)      976     112   scsi_execute_req_flags+0x8e/0x100
	 15)      864     128   scsi_test_unit_ready+0x84/0x140
	 16)      736      48   sd_check_events+0x10e/0x150
	 17)      688     112   disk_check_events+0x51/0x160
	 18)      576      16   disk_events_workfn+0x16/0x20
	 19)      560     112   process_one_work+0x2e9/0x750
	 20)      448     128   worker_thread+0x420/0x800
	 21)      320       0   return_to_handler+0x0/0x28
	 22)      320     144   kthread+0xc9/0xe0
	 23)      176       0   return_to_handler+0x0/0x28
	 24)      176     176   ret_from_fork+0x42/0x70
	 ```
* 从上例中可以看到内核堆栈最满的情况如下，有 25 层函数调用，堆栈使用大小为 2912 字节。此外还可以在 Location 这列中看到整个的 calling stack 情况。

# 前端工具

## trace-cmd

### 资源
```
git://git.kernel.org/pub/scm/linux/kernel/git/rostedt/trace-cmd.git
```
### 简介
* [trace-cmd: A front-end for Ftrace](https://lwn.net/Articles/410200/)
* [trace-cmd(1) - Linux manual page](http://man7.org/linux/man-pages/man1/trace-cmd.1.html)
### 功能
#### trace-cmd
```sh
$ trace-cmd -h

trace-cmd version 2.3.1

usage:
  trace-cmd [COMMAND] ...

  commands:
     record - record a trace into a trace.dat file
     start - start tracing without recording into a file
     extract - extract a trace from the kernel
     stop - stop the kernel from recording trace data
     show - show the contents of the kernel tracing buffer
     reset - disable all kernel tracing and clear the trace buffers
     report - read out the trace stored in a trace.dat file
     hist - show a historgram of the trace.dat information
     split - parse a trace.dat file into smaller file(s)
     options - list the plugin options available for trace-cmd report
     listen - listen on a network socket for trace clients
     list - list the available events, plugins or options
     restore - restore a crashed record
     snapshot - take snapshot of running trace
     stack - output, enable or disable kernel stack tracing
     check-events - parse trace event formats
```
#### trace-cmd record
```sh
$ trace-cmd record -h

trace-cmd version 2.3.1

usage:
 trace-cmd record [-v][-e event [-f filter]][-p plugin][-F][-d][-D][-o file] \
           [-s usecs][-O option ][-l func][-g func][-n func] \
           [-P pid][-N host:port][-t][-r prio][-b size][-B buf][command ...]
           [-m max]
          -e run command with event enabled
          -f filter for previous -e event
          -p run command with plugin enabled
          -F filter only on the given process //只抓取给定进程的 trace
          -P trace the given pid like -F for the command
          -c also trace the childen of -F
          -T do a stacktrace on all events
          -l filter function name
          -g set graph function
          -n do not trace function
          -m max size per CPU in kilobytes
          -M set CPU mask to trace
          -v will negate all -e after it (disable those events)
          -d disable function tracer when running
          -D Full disable of function tracing (for all users)
          -o data output file [default trace.dat]
          -O option to enable (or disable)
          -r real time priority to run the capture threads
          -s sleep interval between recording (in usecs) [default: 1000]
          -N host:port to connect to (see listen)
          -t used with -N, forces use of tcp in live trace
          -b change kernel buffersize (in kilobytes per CPU)
          -B create sub buffer and folling events will be enabled here
          -k do not reset the buffers after tracing.
          -i do not fail if an event is not found
          --func-stack perform a stack trace for function tracer
             (use with caution)
```
### 示例
#### 函数调用
```
# trace-cmd record -e sched ls
trace-cmd record -e sched ls
/sys/kernel/debug/tracing/events/sched/filter
/sys/kernel/debug/tracing/events/*/sched/filter
trace.dat.cpu4  trace.dat trace.dat.cpu5 trace.dat.cpu0  trace.dat.cpu6
trace.dat.cpu1  trace.dat.cpu7  trace.dat.cpu2 trace.dat.cpu3
Kernel buffer statistics:
  Note: "entries" are the entries left in the kernel ring buffer and are not
        recorded in the trace data. They should all be zero.

CPU: 0
entries: 0
overrun: 0
commit overrun: 0
bytes: 0
oldest event ts: 434426.563442
now ts: 434858.825321
dropped events: 0
read events: 0

CPU: 1
entries: 0
overrun: 0
commit overrun: 0
bytes: 0
oldest event ts: 434426.867346
now ts: 434858.825347
dropped events: 0
read events: 0
...
CPU0 data recorded at offset=0x464000
    4096 bytes in size
CPU1 data recorded at offset=0x465000
    4096 bytes in size
CPU2 data recorded at offset=0x466000
    4096 bytes in size
CPU3 data recorded at offset=0x467000
    4096 bytes in size
CPU4 data recorded at offset=0x468000
    4096 bytes in size
CPU5 data recorded at offset=0x469000
    4096 bytes in size
CPU6 data recorded at offset=0x46a000
    4096 bytes in size
CPU7 data recorded at offset=0x46b000
    4096 bytes in size
#
# trace-cmd report
version = 6
cpus=8
              ls-16326 [005] 435246.915558: sched_wakeup:         migration/5:48 [0] success=1 CPU:005
              ls-16326 [005] 435246.915562: sched_stat_runtime:   comm=trace-cmd pid=16326 runtime=235329 [ns] vruntime=2669564645 [ns]
              ls-16326 [005] 435246.915564: sched_switch:         trace-cmd:16326 [120] R ==> migration/5:48 [0]
     migration/5-48    [005] 435246.915569: sched_stat_wait:      comm=trace-cmd pid=16326 delay=11537 [ns]
     migration/5-48    [005] 435246.915571: sched_migrate_task:   comm=trace-cmd pid=16326 prio=120 orig_cpu=5 dest_cpu=6
     migration/5-48    [005] 435246.915583: sched_switch:         migration/5:48 [0] S ==> swapper/5:0 [120]
       trace-cmd-16320 [007] 435246.915607: sched_stat_runtime:   comm=trace-cmd pid=16320 runtime=513676 [ns] vruntime=1873745509 [ns]
       trace-cmd-16320 [007] 435246.915613: sched_stat_wait:      comm=kworker/7:0 pid=14838 delay=1189644 [ns]
       trace-cmd-16320 [007] 435246.915616: sched_switch:         trace-cmd:16320 [120] S ==> kworker/7:0:14838 [120]
     kworker/7:0-14838 [007] 435246.915632: sched_stat_runtime:   comm=kworker/7:0 pid=14838 runtime=26527 [ns] vruntime=28831448457112 [ns]
     kworker/7:0-14838 [007] 435246.915646: sched_switch:         kworker/7:0:14838 [120] S ==> swapper/7:0 [120]
       trace-cmd-16325 [003] 435246.915786: sched_stat_runtime:   comm=trace-cmd pid=16325 runtime=477180 [ns] vruntime=533623809 [ns]
       trace-cmd-16325 [003] 435246.915798: sched_switch:         trace-cmd:16325 [120] S ==> swapper/3:0 [120]
...
```
#### 函数调用栈跟踪
```
# trace-cmd record -p function_graph -g ip_rcv
  plugin 'function_graph'
Hit Ctrl^C to stop recording
^CKernel buffer statistics:
  Note: "entries" are the entries left in the kernel ring buffer and are not
        recorded in the trace data. They should all be zero.

CPU: 0
entries: 0
overrun: 107
commit overrun: 0
bytes: 1244
oldest event ts: 435687.839645
now ts: 435689.296958
dropped events: 0
read events: 127

CPU: 1
entries: 0
overrun: 107
commit overrun: 0
bytes: 1244
oldest event ts: 435688.607176
now ts: 435689.297141
dropped events: 0
read events: 127
...

# trace-cmd report
version = 6
CPU 4 is empty
CPU 5 is empty
CPU 6 is empty
cpus=8
          <idle>-0     [002] 435670.426372: funcgraph_entry:                   |  ip_rcv() {
          <idle>-0     [002] 435670.426375: funcgraph_entry:                   |    ip_rcv_finish() {
          <idle>-0     [002] 435670.426377: funcgraph_entry:        0.566 us   |      udp_v4_early_demux();
          <idle>-0     [002] 435670.426378: funcgraph_entry:                   |      ip_route_input_noref() {
          <idle>-0     [002] 435670.426380: funcgraph_entry:                   |        fib_table_lookup() {
          <idle>-0     [002] 435670.426381: funcgraph_entry:        0.867 us   |          check_leaf.isra.6();
          <idle>-0     [002] 435670.426383: funcgraph_exit:         3.236 us   |        }
          <idle>-0     [002] 435670.426384: funcgraph_entry:                   |        fib_validate_source() {
          <idle>-0     [002] 435670.426386: funcgraph_entry:                   |          fib_table_lookup() {
          <idle>-0     [002] 435670.426386: funcgraph_entry:        0.301 us   |            check_leaf.isra.6();
          <idle>-0     [002] 435670.426388: funcgraph_exit:         1.798 us   |          }
          <idle>-0     [002] 435670.426388: funcgraph_entry:                   |          fib_table_lookup() {
          <idle>-0     [002] 435670.426390: funcgraph_entry:        0.707 us   |            check_leaf.isra.6();
          <idle>-0     [002] 435670.426391: funcgraph_exit:         2.325 us   |          }
          <idle>-0     [002] 435670.426392: funcgraph_exit:         6.979 us   |        }
          <idle>-0     [002] 435670.426393: funcgraph_entry:        0.121 us   |        __skb_dst_set_noref();
          <idle>-0     [002] 435670.426394: funcgraph_exit:       + 15.431 us  |      }
          <idle>-0     [002] 435670.426395: funcgraph_entry:                   |      ip_local_deliver() {
          <idle>-0     [002] 435670.426395: funcgraph_entry:                   |        ip_local_deliver_finish() {
          <idle>-0     [002] 435670.426396: funcgraph_entry:        0.295 us   |          raw_local_deliver();
          <idle>-0     [002] 435670.426397: funcgraph_entry:                   |          udp_rcv() {
          <idle>-0     [002] 435670.426398: funcgraph_entry:                   |            __udp4_lib_rcv() {
          <idle>-0     [002] 435670.426399: funcgraph_entry:                   |              __udp4_lib_mcast_deliver() {
          <idle>-0     [002] 435670.426399: funcgraph_entry:        0.090 us   |                _raw_spin_lock();
          <idle>-0     [002] 435670.426400: funcgraph_entry:        0.090 us   |                _raw_spin_unlock();
          <idle>-0     [002] 435670.426401: funcgraph_entry:                   |                kfree_skb() {
          <idle>-0     [002] 435670.426402: funcgraph_entry:                   |                  skb_release_all() {
          <idle>-0     [002] 435670.426402: funcgraph_entry:        0.230 us   |                    skb_release_head_state();
          <idle>-0     [002] 435670.426403: funcgraph_entry:                   |                    skb_release_data() {
          <idle>-0     [002] 435670.426404: funcgraph_entry:                   |                      skb_free_head() {
          <idle>-0     [002] 435670.426404: funcgraph_entry:                   |                        put_page() {
          <idle>-0     [002] 435670.426405: funcgraph_entry:        0.150 us   |                          put_compound_page();
          <idle>-0     [002] 435670.426406: funcgraph_exit:         1.263 us   |                        }
          <idle>-0     [002] 435670.426407: funcgraph_exit:         2.365 us   |                      }
          <idle>-0     [002] 435670.426407: funcgraph_exit:         3.311 us   |                    }
          <idle>-0     [002] 435670.426407: funcgraph_exit:         5.455 us   |                  }
          <idle>-0     [002] 435670.426408: funcgraph_entry:                   |                  kfree_skbmem() {
          <idle>-0     [002] 435670.426409: funcgraph_entry:        0.562 us   |                    kmem_cache_free();
          <idle>-0     [002] 435670.426410: funcgraph_exit:         1.838 us   |                  }
          <idle>-0     [002] 435670.426410: funcgraph_exit:         8.882 us   |                }
          <idle>-0     [002] 435670.426411: funcgraph_exit:       + 11.608 us  |              }
          <idle>-0     [002] 435670.426411: funcgraph_exit:       + 13.001 us  |            }
          <idle>-0     [002] 435670.426412: funcgraph_exit:       + 13.968 us  |          }
          <idle>-0     [002] 435670.426412: funcgraph_exit:       + 16.362 us  |        }
          <idle>-0     [002] 435670.426413: funcgraph_exit:       + 17.298 us  |      }
          <idle>-0     [002] 435670.426413: funcgraph_exit:       + 37.097 us  |    }
          <idle>-0     [002] 435670.426413: funcgraph_exit:       + 39.166 us  |  }
...
```
* 对应的 ftrace 操作
```sh
echo ip_rcv > /sys/kernel/debug/tracing/set_graph_function
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

### 图形前端-kernelshark
* [KernelShark](http://rostedt.homelinux.com/kernelshark/)
* [Using KernelShark to analyze the real-time scheduler](https://lwn.net/Articles/425583/)

# 跟踪选项
## 通用 trace_options
* `/sys/kernel/debug/tracing/trace_options`文件
	```
	# cat /sys/kernel/debug/tracing/trace_options
	print-parent
	nosym-offset
	nosym-addr
	noverbose
	noraw
	nohex
	nobin
	noblock
	nostacktrace
	trace_printk
	noftrace_preempt
	nobranch
	annotate
	nouserstacktrace
	nosym-userobj
	noprintk-msg-only
	context-info
	nolatency-format
	sleep-time
	graph-time
	record-cmd
	overwrite
	nodisable_on_free
	irq-info
	markers
	function-trace
	notest_nop_accept
	notest_nop_refuse
	```
* 通过修改`/sys/kernel/debug/tracing/options`下的文件可以使能这些`trace_options`

### latency-format
* 该选项会改变 trace 输出。使能的时候，trace 会显示关于延迟的额外的信息
	```
	# tracer: irqsoff
	#
	# irqsoff latency trace v1.1.5 on 3.8.0-test+
	# --------------------------------------------------------------------
	# latency: 259 us, #4/4, CPU#2 | (M:preempt VP:0, KP:0, SP:0 HP:0 #P:4)
	#    -----------------
	#    | task: ps-6143 (uid:0 nice:0 policy:0 rt_prio:0)
	#    -----------------
	#  => started at: __lock_task_sighand
	#  => ended at:   _raw_spin_unlock_irqrestore
	#
	#
	#                  _------=> CPU#            
	#                 / _-----=> irqs-off        
	#                | / _----=> need-resched    
	#                || / _---=> hardirq/softirq
	#                ||| / _--=> preempt-depth   
	#                |||| /     delay             
	#  cmd     pid   ||||| time  |   caller      
	#     \   /      |||||  \    |   /           
	      ps-6143    2d...    0us!: trace_hardirqs_off <-__lock_task_sighand
	      ps-6143    2d..1  259us+: trace_hardirqs_on <-_raw_spin_unlock_irqrestore
	      ps-6143    2d..1  263us+: time_hardirqs_on <-_raw_spin_unlock_irqrestore
	      ps-6143    2d..1  306us : <stack trace>
	 => trace_hardirqs_on_caller
	 => trace_hardirqs_on
	 => _raw_spin_unlock_irqrestore
	 => do_task_stat
	 => proc_tgid_stat
	 => proc_single_show
	 => seq_read
	 => vfs_read
	 => sys_read
	 => system_call_fastpath
	```

### function-trace
* 如果该选项使能（缺省情况），延迟型 tracer 会启用函数跟踪功能
* 当禁用该选项时，延迟型 tracer 不跟踪函数，从而降低执行延迟测试时的开销

### stacktrace
* 当启用该选项时，任何 **trace event** 的栈跟踪将会被记录下来

## function tracer 选项
### func_stack_trace
* 如果说设置`function_graph tracer`+`set_graph_function`可以跟踪到一个函数往下的调用路径，那么`function tracer`+`set_ftrace_filter`+`func_stack_trace`则可以跟踪到它往上的调用路径
* 需使能`CONFIG_STACKTRACE`内核选项
* 只有当`current_tracer`设置为 **function tracer** 时才会出现
* 当设置该 option 时，每个函数被记录的时候，它的栈跟踪也会被记录
* **NOTE:** 在使能该选项时，务必同时设置`set_ftrace_filter`，否则系统性能会严重下降。并且在清除过滤函数的时候记得关闭该选项。
#### Example:
```sh
$ echo rtc_timer_enqueue > /sys/kernel/debug/tracing/set_ftrace_filter
$ echo function > /sys/kernel/debug/tracing/current_tracer
$ echo 1 > /sys/kernel/debug/tracing/options/func_stack_trace
$ echo 1 > /sys/kernel/debug/tracing/tracing_on
$ cat /sys/kernel/debug/tracing/trace_pipe
rtc_test.2029-2902  [001] ....  7694.641987: rtc_timer_enqueue <-rtc_update_irq_enable
rtc_test.2029-2902  [001] ....  7694.642015: <stack trace>
=> rtc_timer_enqueue
=> rtc_update_irq_enable
=> rtc_dev_ioctl
=> do_vfs_ioctl
=> SyS_ioctl
=> system_call_fastpath
```
#### `function-trace` vs `stacktrace` vs `func_stack_trace`
* `function-trace`、`stacktrace`、`func_stack_trace`三者都可以用于栈跟踪，但适用场景是不同的
  * `function-trace`主要用于 **延迟型 tracer**，例如`irqsoff`、`preemptoff`和`preemptirqsoff`
    * kernel command line：`trace_options=function-trace ftrace=preemptoff`
  * `stacktrace`主要用于 **trace event**，也就是预设好的跟踪点
    * kernel command line eg：`trace_options=stacktrace trace_event=[event-list] trace_trigger="sched_switch.stacktrace if prev_state == 2"`
  * `func_stack_trace`主要用于 **function tracer**，用于自定义要跟踪的函数（配合`set_ftrace_filter`）
    * kernel command line eg.1：`trace_options=func_stack_trace ftrace=function ftrace_filter=[function-list]`
    * kernel command line eg.2：`trace_options=func_stack_trace ftrace=function_graph ftrace_graph_filter=[function-list]`

## function_graph tracer 选项
### funcgraph-proc
* `funcgraph-proc`：function_graph tracer 缺省不显示进程名，只有在进程切换时才显示。使能这个选项可以总是显示进程名。
### funcgraph-irqs
* `funcgraph-irqs`：当禁用时，发生在中断上下文的函数不会被跟踪
### sleep-time
* `sleep-time`：当使能时，会把任务被调度出去的时间也算作函数调用的一部份。

# 基于 kprobes 的事件跟踪
* 与基于 tracepoint 的事件跟踪相似，kprobes 事件跟踪是基于 kprobes 点的跟踪
* 可动态插入和删除 kprobes 跟踪点
* 需开启`CONFIG_KPROBE_EVENT=y`和`CONFIG_DYNAMIC_FTRACE=y`
* 无需通过`current_tracer`激活
* 通过`/sys/kernel/debug/tracing/kprobe_events`添加动态探针
* 通过`/sys/kernel/debug/tracing/events/kprobes/<EVENT>/enable`使能
* 移除所有的探针`echo > /sys/kernel/debug/tracing/kprobe_events`

## 内核启动参数
* 可以在内核启动时插入 kprobes event
* 格式为`kprobe_event=`
* events 之间用分号（semicolon）`;`隔开
* event 的参数之间用逗号（comma）`,`隔开

## kprobes事件语法参数定义
```c
p[:[GRP/]EVENT] [MOD:]SYM[+offs]|MEMADDR [FETCHARGS]  : 设置一个探针
r[MAXACTIVE][:[GRP/]EVENT] [MOD:]SYM[+0] [FETCHARGS]  : 设置一个返回探针
p:[GRP/]EVENT] [MOD:]SYM[+0]%return [FETCHARGS]       : 设置一个返回探针
-:[GRP/]EVENT                                         : 清除一个探针
```

参数      | 定义
----------|-----
GRP       | 组名，如忽略会采用用缺省值`kprobes`
EVENT     | 事件名，如忽略会基于`SYMBOL[+offs]`或`MEMADDR`生成
MOD       | 含有欲跟踪符号`SYM`的模块名
SYM[+offs]| 在 *符号+偏移* 处插入探针
SYM%return| 在 *符号的返回地址* 处插入探针
MEMADDR   | 在地址`MEMADDR`处插入探针
MAXACTIVE | 可以同时探测的指定函数的最大实例数，否则为缺省值 0，见 [Documentation/trace/kprobes.rst](https://www.kernel.org/doc/html/latest/trace/kprobes.html) section 1.3.1
FETCHARGS | 探测点的参数，每个探测点可以有最多 128 个参数

### 探测点参数 FETCHARGS 的格式
参数           | 定义
---------------|-----
`%REG`         | 获取寄存器`REG`的值
`@ADDR`        | 获取内存地址`ADDR`处的值（`ADDR`应处于内核地址空间）
`@SYM[+\|-offs]`| 获取符号偏移`SYM +\|- offs`的值（`SYM`应为数据符号）
`$stackN`      | 获取栈上的第 N 个条目（N >= 0）
`$stack`       | 获取栈顶的地址
`$argN`        | 获取函数的第 N 个参数（仅用于安置在函数上的探针）
`$retval`      | 获取函数的返回值（仅用于 *返回探针*）
`$comm`        | 获取当前进程的名字
`+\|-[u]OFFS(FETCHARG)`| 获取 FETCHARG 偏移 `+\|- OFFS` 地址处的值（可用于获取数据结构的域；`u`表示用户空间的解引用）
`\IMM`         | 将立即数存入参数
`NAME=FETCHARG`| 将`NAME`设置为 FETCHARG 的参数名
`FETCHARG:TYPE`| 将 FETCHARG 的类型设置为`TYPE`

#### FETCHARG 类型
* fetch-args 支持几种类型，Kprobe tracer 会按给定的类型访问内存。
* 类型的前缀`s`和`u`用来指明类型是 *有符号的* 还是 *无符号的*。trace 打印时以十进制的方式显示。
* 前缀`x`暗示该类型是无符号的。trace 打印时以十六进制的方式显示。
* 这些类型也可以是数组，添加`[N]`（N 为确切的数字，小于 64）到基本类型来记录数组数据。
  * 例如，`x16[4]`表示一个有 4 个元素的`x16`（2 字节 16 进制）的数组。
* 注意：数组可以被用于内存地址类型的 fetchargs，不能用于寄存器/栈类型的条目。
  * 例如，`$stack1:x8[8]`是错误的，但`+8($stack):x8[8]`是正确的。
* **字符串类型** 是一种特殊的类型，它从内核地址空间获取一个 null 结尾的字符串。这意味着如果存放该字符串的容器被 pageed-out 了，会导致取值失败并且得到 NULL。
* `ustring` 类型是另一种形式的字符串类型，用于用户空间。
* **字符串数组类型** 和其他类型有一些不同：
  * 对于其他基本类型，`<base-type>[1]`等于`<base-type>`（例如`+0(%di):x32[1]`和`+0(%di):x32`）相同。
  * 但`string[1]`不等于`string`。字符串类型本身表示 **“char array”**，字符串数组类型表示 **“char * array”**。
  * 例如：`+0(%di):string[1]`等于`+0(+0(%di)):string`。
* **位域（Bitfield）类型** 是另一个特殊类型，有 *位宽（bit-width）*,*位偏移（bit-offset）* 和 *容器大小（container-size）*（通常是 32）三个参数。
  * 语法为：`b<bit-width>@<bit-offset>/<container-size>`
* **符号类型（symbol）** 是`u32`或`u64`（取决于`BITS_PER_LONG`）类型的别名，将给定的指针显示为`symbol+offset`的风格。
* 对于`$comm`缺省类型是`string`；任何其他的类型都是非法的。
* `$argN`由 commit `a1303af5d79eb13a658633a9fb0ce3aed0f7decf tracing: probeevent: Add $argN for accessing function args`在`v4.20`引入，需要使能`CONFIG_HAVE_FUNCTION_ARG_ACCESS_API`选项。

## 例子1：probe do_fork
* 该例演示一个简单的 kprobes event 运用
### 插入探针
```sh
echo 'p:doforkprobe _do_fork clone_flags=%di stack_start=%si stack_size=%dx rsp=%sp rbp=%bp'> /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/doforkprobe/enable
cat /sys/kernel/debug/tracing/trace
```
### 查看结果
```sh
# cat trace
# tracer: nop
#
# entries-in-buffer/entries-written: 79/79   #P:2
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
         systemd-1     [000] ...2 938618.752964: doforkprobe: (_do_fork+0x0/0x3d0) clone_flags=0x1200011 stack_start=0x0 stack_size=0x0 rsp=0xffffa8d480643f20 rbp=0xffffa8d480643f28
              sh-1140  [001] ...2 938621.996818: doforkprobe: (_do_fork+0x0/0x3d0) clone_flags=0x1200011 stack_start=0x0 stack_size=0x0 rsp=0xffffa8d480c93f20 rbp=0xffffa8d480c93f28
        kthreadd-2     [000] ...2 938638.790374: doforkprobe: (_do_fork+0x0/0x3d0) clone_flags=0x800711 stack_start=0xffffffffa9096370 stack_size=0xffff8abe09841240 rsp=0xffffa8d48064bf08 rbp=0xffffa8d48064bf10
```
* 因为探针加在函数的入口，且`_do_fork()`的调用者没有用到栈，所以`%rbp + 8 = %rsp`，`8`是`call`指令压入的函数返回地址的长度（x86-64）
### 移除探针
```sh
echo 0 > /sys/kernel/debug/tracing/events/kprobes/doforkprobe/enable
echo '-:kprobes/doforkprobe' > /sys/kernel/debug/tracing/kprobe_events
```
## 例子2：probe do_sys_open
* 该例演示一个 kprobes event 结合`stacktrace`选项运用
### 插入探针
```sh
echo 'p:dosysopen do_sys_open+6 dfd=%di filename=+0(%si):string flags=%dx mode=%cx rbp=%bp stack=$stack' > /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/options/stacktrace
echo 1 > /sys/kernel/debug/tracing/events/kprobes/dosysopen/enable
cat /sys/kernel/debug/tracing/trace
```
### 查看结果
```sh
# tracer: nop
#
# entries-in-buffer/entries-written: 3/3   #P:2
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
             cat-14096 [000] ...1 1198740.336139: dosysopen: (do_sys_open+0x6/0x220) dfd=0xffffff9c filename="/etc/ld.so.cache" flags=0x88000 mode=0x0 rbp=0xffffa8d48382bf28 stack=0xffffa8d48382bf18
             cat-14096 [000] ...1 1198740.336154: <stack trace>
=> do_sys_open
=> do_syscall_64
=> entry_SYSCALL_64_after_hwframe
             cat-14096 [000] ...1 1198740.336200: dosysopen: (do_sys_open+0x6/0x220) dfd=0xffffff9c filename="/lib64/libc.so.6" flags=0x88000 mode=0x0 rbp=0xffffa8d48382bf28 stack=0xffffa8d48382bf18
             cat-14096 [000] ...1 1198740.336203: <stack trace>
=> do_sys_open
=> do_syscall_64
=> entry_SYSCALL_64_after_hwframe
             cat-14096 [000] ...1 1198740.336971: dosysopen: (do_sys_open+0x6/0x220) dfd=0xffffff9c filename="/sys/kernel/debug/tracing/trace" flags=0x8000 mode=0x0 rbp=0xffffa8d48382bf28 stack=0xffffa8d48382bf18
             cat-14096 [000] ...1 1198740.336977: <stack trace>
=> do_sys_open
=> do_syscall_64
=> entry_SYSCALL_64_after_hwframe
```
* 无法对寄存器直接取字符串内容，需借助`+|-[u]OFFS(%REG)`的格式来获得，可以看上面取`filename`的例子
* 动态插入的`kprobe_events`可以配合`/sys/kernel/debug/tracing/options/stacktrace`选项查看调用栈
### 移除探针
```sh
echo 0 > /sys/kernel/debug/tracing/events/kprobes/dosysopen/enable
echo 0 > /sys/kernel/debug/tracing/options/stacktrace
echo '-:dosysopen' > /sys/kernel/debug/tracing/kprobe_events
```

## 例子3：多个探针
* 该例演示多个 kprobes events 观察寄存器变化的运用，可以仿照出动态观测本地变量的变化的实际运用
### 插入探针
```sh
echo 'p:dosysopen1 do_sys_open+6 dfd=%di filename=+0(%si):string flags=%dx mode=%cx rbp=%bp stack=$stack' > /sys/kernel/debug/tracing/kprobe_events
echo 'p:dosysopen2 do_sys_open+17 dfd=%di filename=+0(%si):string flags=%dx mode=%cx rbp=%bp stack=$stack' >> /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/dosysopen1/enable
echo 1 > /sys/kernel/debug/tracing/events/kprobes/dosysopen2/enable
cat /sys/kernel/debug/tracing/trace
```
### 查看结果
```sh
<...>-23028 [001] ...1 1281675.534643: dosysopen1: (do_sys_open+0x6/0x220) dfd=0xffffff9c filename="/etc/ld.so.cache" flags=0x88000 mode=0x0 rbp=0xffffa8d488c4bf28 stack=0xffffa8d488c4bf18
<...>-23028 [001] ...1 1281675.534685: dosysopen2: (do_sys_open+0x11/0x220) dfd=0xffffff9c filename="/etc/ld.so.cache" flags=0x88000 mode=0x0 rbp=0xffffa8d488c4bf18 stack=0xffffa8d488c4bef8
<...>-23028 [001] ...1 1281675.534733: dosysopen1: (do_sys_open+0x6/0x220) dfd=0xffffff9c filename="/lib64/libc.so.6" flags=0x88000 mode=0x0 rbp=0xffffa8d488c4bf28 stack=0xffffa8d488c4bf18
<...>-23028 [001] ...1 1281675.534736: dosysopen2: (do_sys_open+0x11/0x220) dfd=0xffffff9c filename="/lib64/libc.so.6" flags=0x88000 mode=0x0 rbp=0xffffa8d488c4bf18 stack=0xffffa8d488c4bef8
<...>-23028 [001] ...1 1281675.535445: dosysopen1: (do_sys_open+0x6/0x220) dfd=0xffffff9c filename="trace" flags=0x8000 mode=0x0 rbp=0xffffa8d488c4bf28 stack=0xffffa8d488c4bf18
<...>-23028 [001] ...1 1281675.535448: dosysopen2: (do_sys_open+0x11/0x220) dfd=0xffffff9c filename="trace" flags=0x8000 mode=0x0 rbp=0xffffa8d488c4bf18 stack=0xffffa8d488c4bef8
```
* 对于`do_sys_open()`反汇编代码
	```nasm
	0000000000002100 <do_sys_open>:
	do_sys_open():
	/usr/src/kernel/fs/open.c:1095
	    2100:   e8 00 00 00 00          callq  2105 <do_sys_open+0x5>
	    2105:   55                      push   %rbp   
	    2106:   48 89 e5                mov    %rsp,%rbp  ;<-dosysopen1 pokes here
	    2109:   41 57                   push   %r15
	    210b:   41 56                   push   %r14
	    210d:   41 55                   push   %r13
	    210f:   41 54                   push   %r12
	    2111:   41 89 d4                mov    %edx,%r12d ;<-dosysopen2 pokes here
	```
  * `%rsp`与`$stack`是相同的
  * 我们可以看到`do_sys_open+5 = 2105`的地方有个`push %rbp`，所以加在`do_sys_open+6 = 2106`的探针运行到时，栈上除了`call`指令压入的函数返回地址，还有上一条指令压入的上一帧的`%rbp`，因此我们看到探针 dosysopen1 的`rbp - 16 = stack`（x86-64）
  * `do_sys_open+6`的`mov %rsp,%rbp`改变了`%rbp`的值，探针 dosysopen2 位置的`%rbp`随即发生了变化，变为旧`%rsp`/`stack`的值`0xffffa8d488c4bf18`
  * `2109 ~ 210f`连续四个压栈操作，所以加在`do_sys_open+17 = 2111`的探针 dosysopen2 运行到时，`rsp = 0xffffa8d488c4bf18 - 0x8 x 4 = 0xffffa8d488c4bef8`
### 移除探针
```sh
echo 0 > /sys/kernel/debug/tracing/events/kprobes/dosysopen1/enable
echo 0 > /sys/kernel/debug/tracing/events/kprobes/dosysopen2/enable
echo '-:dosysopen2' > /sys/kernel/debug/tracing/kprobe_events
echo '-:dosysopen1' > /sys/kernel/debug/tracing/kprobe_events
```

# 快照 Snapshot
* `CONFIG_TRACER_SNAPSHOT`提供了快照功能，可用于所有非延迟类型 tracer
* 延迟型 tracer，例如`irqsoff`和`wakeup`，无法使用快照功能，因为它们的内部实现已经使用了该功能
* 控制文件为`/sys/kernel/debug/tracing/snapshot`

# 实例 Instances
* 实例的 buffer 是独立于主 buffer 的
* 实例目前仅支持事件，不支持 tracer
* `trace_options`目前的影响是全局的，将来可能会支持实例独立的`trace_options`
* 控制目录为`/sys/kernel/debug/tracing/instances`，创建文件会相应地创建 buffer

# Reference

* [ftrace 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/)
* [使用 ftrace 调试 Linux 内核，第 1 部分](http://www.ibm.com/developerworks/cn/linux/l-cn-ftrace1/)
* [使用 ftrace 调试 Linux 内核，第 2 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace2/)
* [使用 ftrace 调试 Linux 内核，第 3 部分](http://www.ibm.com/developerworks/cn/linux/l-cn-ftrace3/)
* [Debugging the kernel using Ftrace - part 1](https://lwn.net/Articles/365835/)
* [Debugging the kernel using Ftrace - part 2](https://lwn.net/Articles/366796/)
* [Secrets of the Ftrace function tracer](https://lwn.net/Articles/370423/)
* [ftrace - Function Tracer — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/trace/ftrace.html)
* [Event Tracing — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/trace/events.html)
* [Kernel Probes (Kprobes) — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/trace/kprobes.html)
* [Kprobe-based Event Tracing — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/trace/kprobetrace.html)
