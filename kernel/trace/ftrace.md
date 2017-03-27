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
```
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo function > /sys/kernel/debug/tracing/current_tracer
```

#### 设定跟踪特定的函数
```
echo do_page_fault > /sys/kernel/debug/tracing/set_ftrace_filter
```
* 当然可以不设置，那样会跟踪所有能跟踪的函数。

#### 指定跟踪特定的CPU
```
echo "cpu-id" > /sys/kernel/debug/tracing/tracing_cpumask
```

#### 开始跟踪
```
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe | tee /tmp/ftrace.log
```

#### 输出示例
```
            TASK-PID CPU irqs-off need-resched hardirq/softirq preempt-depth delay TIMESTAMP FUNCTION
            lttng-879   [000] d..1 2437395.004331: do_page_fault <-do_PrefetchAbort
            lttng-879   [000] d..1 2437395.004331: do_page_fault <-do_PrefetchAbort
            lttng-879   [000] d..1 2437395.004346: do_page_fault <-do_DataAbort
            lttng-879   [000] d..1 2437395.004362: do_page_fault <-do_PrefetchAbort
            lttng-879   [000] d..1 2437395.004377: do_page_fault <-do_PrefetchAbort
```

#### 关闭跟踪
```
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
```
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo function_graph > /sys/kernel/debug/tracing/current_tracer
```

#### 设定跟踪特定的进程
```
echo 822 > /sys/kernel/debug/tracing/set_ftrace_pid
```
* 当然可以不设置，那样会跟踪所有能跟踪的进程。

#### 开始跟踪
```
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe | tee /tmp/ftrace.log
```

#### 输出示例
```
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
```
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

# Reference

* [ftrace 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/)
* [使用 ftrace 调试 Linux 内核，第 1 部分](http://www.ibm.com/developerworks/cn/linux/l-cn-ftrace1/)
* [使用 ftrace 调试 Linux 内核，第 2 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace2/)
* [使用 ftrace 调试 Linux 内核，第 3 部分](http://www.ibm.com/developerworks/cn/linux/l-cn-ftrace3/)
* [Debugging the kernel using Ftrace - part 1](https://lwn.net/Articles/365835/)
* [Debugging the kernel using Ftrace - part 2](https://lwn.net/Articles/366796/)
* [Secrets of the Ftrace function tracer](https://lwn.net/Articles/370423/)
