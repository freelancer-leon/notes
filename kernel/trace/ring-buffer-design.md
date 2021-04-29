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

## 通用 Ring Buffer

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
This is very much like a writer being preempted by an interrupt and the interrupt doing a write as well.
这非常类似于写者被中断抢占，而中断也执行写入操作。

Readers can happen at any time. But no two readers may run at the same time, nor can a reader preempt/interrupt another reader. A reader cannot preempt/interrupt a writer, but it may read/consume from the buffer at the same time as a writer is writing, but the reader must be on another processor to do so. A reader may read on its own processor and can be preempted by a writer.
读操作可以随时发生。但是，任何两个读者都不能同时运行，一个读者也不能抢占/中断其他读者。虽然读者无法抢占/中断读者，但它可以在写者写入的同时从缓冲区读取/消费，且读者必须在另一个处理器上才能这样做。读者可以在自己的处理器上读，并且可以被写者抢占。

A writer can preempt a reader, but a reader cannot preempt a writer. But a reader can read the buffer at the same time (on another processor) as a writer.
写者可以抢占读者，但读者不能抢占写者。但是，读者可以与写者同时（在另一个处理器上）读取缓冲区。

The ring buffer is made up of a list of pages held together by a linked list.
Ring buffer 由链接列表所保持的 pages 列表组成。

At initialization a reader page is allocated for the reader that is not part of the ring buffer.
在初始化时，为读者分配一个 **读者 page**，该 page 不属于 ring buffer。

The head_page, tail_page and commit_page are all initialized to point to the same page.
head_page，tail_page 和 commit_page 都被初始化为指向同一页面。

The reader page is initialized to have its next pointer pointing to the head page, and its previous pointer pointing to a page before the head page.
将 *读者 page* 初始化为使其 **next 指针** 指向 head page，其 **previous 指针** 指向 head page 之前的页面。

The reader has its own page to use. At start up time, this page is allocated but is not attached to the list. When the reader wants to read from the buffer, if its page is empty (like it is on start-up), it will swap its page with the head_page. The old reader page will become part of the ring buffer and the head_page will be removed. The page after the inserted page (old reader_page) will become the new head page.
读者可以使用自己的页面。在启动时，已分配此页面，但并未将其附加到列表中。当读者要从缓冲区读取时，如果其页面为空（如启动时一样），它将与 head_page 交换其页面。旧的 *读者 page* 将成为 ring buffer 的一部分，并且 head_page 将被删除。插入页面之后的页面（旧的 reader_page）将成为新的 head page。

Once the new page is given to the reader, the reader could do what it wants with it, as long as a writer has left that page.
一旦将新页面提供给 *读者*，只要 *写者* 离开了该页面，*读者* 就可以用它来做自己想做的事情。

A sample of how the reader page is swapped: Note this does not show the head page in the buffer, it is for demonstrating a swap only.
一个 *读者 page* 如何交换的示例如下：请注意，这不会在缓冲区中显示 head page，它仅用于演示交换。
```
+------+
|reader|          RING BUFFER
|page  |
+------+
                +---+   +---+   +---+
                |   |-->|   |-->|   |
                |   |<--|   |<--|   |
                +---+   +---+   +---+
                 ^ |             ^ |
                 | +-------------+ |
                 +-----------------+


+------+
|reader|          RING BUFFER
|page  |--------next-------+
+------+                   v
  |             +---+   +---+   +---+
  |             |   |-->|   |-->|   |
  |             |   |<--|   |<--|   |<-+
  |             +---+   +---+   +---+  |
  |              ^ |             ^ |   |
  |              | +---previous--+ |   |
  |              +-------next------+   |
  +-------------previous---------------+

+------+
|reader|          RING BUFFER
|page  |--------next-------+
+------+ <----previous---+ v
  |  ^          +---+   +---+   +---+
  |  |          |   |-->|   |-->|   |
  |  |          |   |   |   |<--|   |<-+
  |  |          +---+   +---+   +---+  |
  |  |             |             ^ |   |
  |  |             +---previous--+ |   |
  |  +----------next---------------+   |
  +------------previous----------------+

+------+
|buffer|          RING BUFFER
|page  |-------------------+
+------+ <---------------+ v
  |  ^          +---+   +---+   +---+
  |  |          |   |   |   |-->|   |
  |  |  New     |   |   |   |<--|   |<-+
  |  | Reader   +---+   +---+   +---+  |
  |  |  page ----^                 |   |
  |  |                             |   |
  |  +-----------------------------+   |
  +------------------------------------+
```
It is possible that the page swapped is the commit page and the tail page, if what is in the ring buffer is less than what is held in a buffer page.
如果 ring buffer 中的内容少于缓冲区页面中保存的内容，则交换的页面可能是 commit page 和 tail page。
```
reader page    commit page   tail page
    |              |             |
    v              |             |
   +---+           |             |
   |   |<----------+             |
   |   |<------------------------+
   |   |------+
   +---+      |
              |
              v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```
This case is still valid for this algorithm. When the writer leaves the page, it simply goes into the ring buffer since the reader page still points to the next location in the ring buffer.
这种情况对于此算法仍然有效。当写者离开页面时，它只是进入 ring buffer，因为 *读者 page* 仍指向 ring buffer 中的下一个位置。

主要的指针：

**reader page**
* 该页面仅由读者使用，不属于 ring buffer（可以交换入）

**head page**
* Ring buffer 中的下一个与 reader page 交换的页面。

**tail page**
* 下一次写入的页面。

**commit page**
* 最后完成写操作的页面。

The commit page only is updated by the outermost writer in the writer stack. A writer that preempts another writer will not move the commit page.
Commit page 仅由写者堆栈中最外面的写者更新。抢占另一个写者的写者将不会移动 commit page。

When data is written into the ring buffer, a position is reserved in the ring buffer and passed back to the writer. When the writer is finished writing data into that position, it commits the write.
当将数据写入 ring buffer 时，会在 ring buffer 中保留一个位置并将其传递回写者。当写程序完成将数据写入该位置时，它将提交写操作。

Another write (or a read) may take place at anytime during this transaction. If another write happens it must finish before continuing with the previous write.
在该事务处理期间的任何时候都可能发生另一次写（或读）操作。如果发生另一次写入，则它必须先完成，然后再继续上一次写入。

Write reserve:
写保留：
```
Buffer page
+---------+
|written  |
+---------+  <--- given back to writer (current commit)
|reserved |
+---------+ <--- tail pointer
| empty   |
+---------+
```
Write commit:
写提交：
```
Buffer page
+---------+
|written  |
+---------+
|written  |
+---------+  <--- next position for write (current commit)
| empty   |
+---------+
```
If a write happens after the first reserve:
如果在第一次保留后发生写操作：
```
Buffer page
+---------+
|written  |
+---------+  <-- current commit
|reserved |
+---------+  <--- given back to second writer
|reserved |
+---------+ <--- tail pointer

After second writer commits::
第二个写者提交后：

Buffer page
+---------+
|written  |
+---------+  <--(last full commit)
|reserved |
+---------+
|pending  |
|commit   |
+---------+ <--- tail pointer

When the first writer commits::

Buffer page
+---------+
|written  |
+---------+
|written  |
+---------+
|written  |
+---------+  <--(last full commit and tail pointer)
```
The commit pointer points to the last write location that was committed without preempting another write. When a write that preempted another write is committed, it only becomes a pending commit and will not be a full commit until all writes have been committed.
提交指针指向最后提交的写入位置，而不会抢占另一个写入。如果一次写入抢占另一次写入并提交，则该写入将仅成为挂起的提交，并且在提交所有写入之前不会成为完全提交。

The commit page points to the page that has the last full commit. The tail page points to the page with the last write (before committing).
Commit page 指向具有最后一次完整提交的页面。Tail page 指向最后写入（提交之前）的页面。

The tail page is always equal to or after the commit page. It may be several pages ahead. If the tail page catches up to the commit page then no more writes may take place (regardless of the mode of the ring buffer: overwrite and produce/consumer).
Tail page 始终等于或在 commit page 之后。可能要提前几个页。如果 tail page 赶上 commit page，则不会再进行任何写操作（无论 ring buffer 的模式如何：覆盖和产生/消费）。

The order of pages is:
页面的顺序为：
```
head page
commit page
tail page
```
Possible scenario:
可能出现的场景：
```
                             tail page
  head page         commit page  |
      |                 |        |
      v                 v        v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
```
There is a special case that the head page is after either the commit page and possibly the tail page. That is when the commit (and tail) page has been swapped with the reader page. This is because the head page is always part of the ring buffer, but the reader page is not. Whenever there has been less than a full page that has been committed inside the ring buffer, and a reader swaps out a page, it will be swapping out the commit page.
在特殊情况下，head page 位于 commit page 之后，也可能位于 tail page 之后。那就是，当 commit page（和 tail page）与 *读者 page* 交换时。这是因为 head page 始终是 ring buffer 的一部分，而 *读者 page* 则不是。只要在 ring buffer 内提交的页面不足以容纳整个页面，并且读者换出页面，此时将换出 commit page。
```
reader page    commit page   tail page
    |              |             |
    v              |             |
   +---+           |             |
   |   |<----------+             |
   |   |<------------------------+
   |   |------+
   +---+      |
              |
              v
    +---+    +---+    +---+    +---+
<---|   |--->|   |--->|   |--->|   |--->
--->|   |<---|   |<---|   |<---|   |<---
    +---+    +---+    +---+    +---+
              ^
              |
          head page
```
In this case, the head page will not move when the tail and commit move back into the ring buffer.
在这种情况下，当 tail 和 commit (page) 移回 ring buffer 时，head page 将不会移动。

The reader cannot swap a page into the ring buffer if the commit page is still on that page. If the read meets the last commit (real commit not pending or reserved), then there is nothing more to read. The buffer is considered empty until another full commit finishes.
如果 commit page 仍在（读者）页面上，则读者无法将该页面交换到 ring buffer 中。如果读取遇到最后一个提交（实际提交未挂起或保留），则没有更多内容可读取。在另一个完全提交完成之前，缓冲区被认为是空的。

When the tail meets the head page, if the buffer is in overwrite mode, the head page will be pushed ahead one. If the buffer is in producer/consumer mode, the write will fail.
当 tail 遇到 head page 时，如果缓冲区处于 *覆盖模式*，则 head page 将向前推。如果缓冲区处于 *生产者/消费者模式*，则写入将失败。

Overwrite mode:
覆盖模式：
```
              tail page
                 |
                 v
      +---+    +---+    +---+    +---+
  <---|   |--->|   |--->|   |--->|   |--->
  --->|   |<---|   |<---|   |<---|   |<---
      +---+    +---+    +---+    +---+
                          ^
                          |
                      head page


              tail page
                 |
                 v
      +---+    +---+    +---+    +---+
  <---|   |--->|   |--->|   |--->|   |--->
  --->|   |<---|   |<---|   |<---|   |<---
      +---+    +---+    +---+    +---+
                                   ^
                                   |
                               head page


                      tail page
                          |
                          v
      +---+    +---+    +---+    +---+
  <---|   |--->|   |--->|   |--->|   |--->
  --->|   |<---|   |<---|   |<---|   |<---
      +---+    +---+    +---+    +---+
                                   ^
                                   |
                               head page

```
Note, the reader page will still point to the previous head page. But when a swap takes place, it will use the most recent head page.
请注意，读者 page 仍将指向上一个 head page。但是当交换发生时，它将使用最新的 head page。
