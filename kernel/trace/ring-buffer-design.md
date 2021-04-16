# 无锁环形缓冲区的设计（Lockless Ring Buffer Design）
Copyright 2009 Red Hat Inc.

Author
Steven Rostedt <srostedt@redhat.com>

License
The GNU Free Documentation License, Version 1.2 (dual licensed under the GPL v2)

Reviewers
Mathieu Desnoyers, Huang Ying, Hidetoshi Seto, and Frederic Weisbecker.

Written for: 2.6.31

## 本文使用的术语
**尾部 tail**
* 新的写入发生在 ring buffer 中的位置。

**头部 head**
* 新的读取发生在 ring buffer 中的位置。

**生产者 producer**
* 往 ring buffer 写的任务（与“写者”相同）

**写者 writer**
* 与“生产者”相同

**消费者 consumer**
* 从 ring buffer 中读的任务（与“读者”相同）

**读者 reader**
* 与“消费者”相同

**reader_page**
* 一个在 ring buffer 之外的页面，（大部分）由读者单独使用

**head_page**
* 指向读者下一个使用的页面的指针

**tail_page**
* 指向下一个会被写入的页面的指针

**commit_page**
* 指向最后完成的非嵌套写入的页面的指针。

**cmpxchg**
* 需借助硬件实现的原子操作，执行以下操作：
  ```c
  如果之前的 A == C，令 A = B
  R = cmpxchg(A, C, B) 意思是说，当且仅当 A 等于 C 时，用 B 替换 A，并且我们把旧的（当前的） A 放入 R
  无论 A 是否用 B 更新，R 都会获得前一个 A 的值。
  ```
  查看更新是否成功，可以比较`R == C`。（译注：SMP 场景时，`R == C`不相等意味着 A 已经在其他核上被更新）

## 范型 Ring Buffer

The ring buffer can be used in either an overwrite mode or in producer/consumer mode.
Ring buffer 可用于 *覆盖模式* 或 *生产者/消费者模式*。

Producer/consumer mode is where if the producer were to fill up the buffer before the consumer could free up anything, the producer will stop writing to the buffer. This will lose most recent events.
在 **生产者/消费者模式** 下，如果生产者在消费者可以释放任何东西之前填满缓冲区，生产者将停止写入缓冲区。这将 **丢失最近发生的事件**。

Overwrite mode is where if the producer were to fill up the buffer before the consumer could free up anything, the producer will overwrite the older data. This will lose the oldest events.
在 **覆盖模式** 下，如果生产者要在消费者可以释放任何内容之前填满缓冲区，则生产者将覆盖较早的数据。这将 **丢失最早的事件**。

No two writers can write at the same time (on the same per-cpu buffer), but a writer may interrupt another writer, but it must finish writing before the previous writer may continue. This is very important to the algorithm. The writers act like a “stack”. The way interrupts works enforces this behavior:
没有两个写者可以同时（在同一 per-cpu 缓冲区）写，但是一个写者可能会中断另一个写者，且它必须先完成写操作，然后之前的写者才能继续写操作。这一点对算法非常重要。写者的行为就像一个“栈”。中断的工作方式会强制执行以下行为：
```
writer1 start
   <preempted> writer2 start
       <preempted> writer3 start
                   writer3 finishes
               writer2 finishes
writer1 finishes
```
