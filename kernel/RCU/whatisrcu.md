# What is RCU, Fundamentally?

Paul E. McKenney, IBM Linux Technology Center
Jonathan Walpole, Portland State University Department of Computer Science

## 引言
读-拷贝更新（RCU，Read-copy update）是 2002 年 10 月加入 Linux 内核的一种同步机制。RCU 通过允许读操作与更新操作并发执行，提升了系统的可扩展性。
传统锁原语会对所有并发线程（无论读者还是更新者）强制互斥；读写锁虽然支持并发读，但一旦存在更新就不允许读。和它们不同，RCU 支持 **一个更新者与多个读者并发运行**。
RCU 通过维护对象的多个版本、并且保证旧版本不会被释放，直到所有先前启动的读侧临界区全部执行完毕，以此保证读操作的一致性。
RCU 定义并使用一套高效、高可扩展的机制：用于发布、读取对象的新版本，同时延迟（deferring）回收旧版本。
这套机制把开销分摊到读路径和更新路径上，让 **读路径的开销极低**。在部分场景下（不可抢占内核），RCU 的读侧原语开销为零。

**快速自测题1：** 那顺序锁（seqlock）不也允许读端和更新端并发执行吗？

这就引出一个问题：“RCU 到底是什么？”，或许还会附带另一个问题：“RCU 这东西凭什么能正常工作？”（也经常会看到一种论调，说 RCU 根本就不可能正常工作）。
本文从底层原理角度解答这些问题；后续篇章会从使用方式和API接口的角度展开说明。最后一篇还会附上参考文献列表。

RCU 由三大基础机制构成：第一个用于插入操作，第二个用于删除操作，第三个让读端可以容忍并发的插入与删除。
下面分小节讲解这些机制，以 RCU 在链表上的应用为例展开说明：
- [发布-订阅机制](#发布-订阅机制publish-subscribe-mechanism)（Publish-Subscribe，用于插入）
- [等待已存在的 RCU 读端临界区执行完毕](#等待已启动的-rcu-读端临界区执行完毕)（用于删除）
- [为新近更新的对象维护多个版本](#为新近更新的对象维护多个版本)（服务于读端）

以上小节之后是 [总结] 以及 [快速自测题答案]。

## 发布-订阅机制（Publish-Subscribe Mechanism）

RCU 的一个核心特性：**即使数据正在被并发修改，读端依然可以安全地遍历该数据**。为支持并发插入，RCU 采用了一种可理解为发布-订阅的机制。
举个例子：有一个初始值为 `NULL` 的全局指针 `gp`，我们要修改它，让它指向一块刚分配并完成初始化的数据结构。
可以用下面这段代码片段（还需要搭配合适的锁保护）实现这个目的：
```c
struct foo {
   int a;
   int b;
   int c;
};
struct foo *gp = NULL;
 
/* . . . */

p = kmalloc(sizeof(*p), GFP_KERNEL);
p->a = 1;
p->b = 2;
p->c = 3;
gp = p;
```

但这段代码存在问题：**没有任何约束强制编译器和 CPU 按顺序执行最后四条赋值语句**。
如果对 `gp` 的赋值先于 `p` 成员字段的初始化完成，那么并发的读端就有可能读到未初始化的脏值。
这种场景需要内存屏障来保证顺序，而内存屏障本身以难用、容易出错著称。因此内核把屏障逻辑封装成原语 `rcu_assign_pointer()`，它自带 **发布语义（publication semantics）**。修改后最后四行代码如下：
```c
p->a = 1;
p->b = 2;
p->c = 3;
rcu_assign_pointer(gp, p);
```

`rcu_assign_pointer()` 负责 *发布* 这个新结构体：它会强制编译器与 CPU，保证对 `gp` 的赋值，一定在 `p` 指向的各个字段赋值完成 *之后* 执行。

但仅仅在更新端保证执行顺序还不够，**读端同样必须保证正确的访存顺序**。请看下面这段代码片段：
```c
p = gp;
if (p != NULL) {
    do_something_with(p->a, p->b, p->c);
}
```

这段代码乍一看似乎不会发生乱序问题，但不巧的是，[DEC Alpha CPU](http://www.rdrop.com/users/paulmck/scalability/paper/ordering.2007.09.19a.pdf)，以及 **值推测（value-speculation）** 类编译器优化，真的可以让 `p->a`、`p->b`、`p->c` 的读取先于指针 `p` 的读取！
用值推测优化的场景最好理解：编译器会预先猜测 `p` 的值，提前加载 `p->a`、`p->b`、`p->c`，之后再读取真实的 `p`，校验刚才的猜测是否成立。这类优化非常激进，甚至可以说近乎离谱，但在基于采样反馈的配置驱动优化（profile-driven optimization）中确实会出现。

显然，我们需要阻止编译器和 CPU 做出这类“小动作”。原语 `rcu_dereference()` 会按需插入内存屏障指令以及编译器屏障，用来达成这个目的：
```c
rcu_read_lock();
p = rcu_dereference(gp);
if (p != NULL) {
    do_something_with(p->a, p->b, p->c);
}
rcu_read_unlock();
```

因此可以把 `rcu_dereference()` 理解为 **订阅** 指定指针的取值：它能保证后续对该指针成员的解引用，一定可以看到对应发布操作（`rcu_assign_pointer()`）之前完成的全部初始化。
`rcu_read_lock()` 和 `rcu_read_unlock()` 是必不可少的：它们划定了 **RCU 读端临界区** 的范围。它们的作用会在下一节说明；重点是：这两个函数 **不会自旋、不会阻塞**，也不会阻止 `list_add_rcu()` 并发执行。实际上，在未开启 `CONFIG_PREEMPT` 的内核中，这两个调用不会生成任何机器码。

理论上，仅凭 `rcu_assign_pointer()` 和 `rcu_dereference()` 就可以构造任意受 RCU 保护的数据结构，但工程实践中，使用更高层封装会更方便。
所以这两个原语已经被封装进 Linux 链表操作 API 的 RCU 专用版本里。
Linux 提供两种双向链表：环形的 `struct list_head`，以及线性的 `struct hlist_head / struct hlist_node` 组合。
前者的布局如下，其中绿色方框代表链表头，蓝色方框代表链表节点。

![Linux List](pic/wircu_Linux_list.jpg)

把上面指针发布的示例改造为链表版本，代码如下：
```c
  1 struct foo {
  2   struct list_head list;
  3   int a;
  4   int b;
  5   int c;
  6 };
  7 LIST_HEAD(head);
  8 
  9 /* . . . */
 10 
 11 p = kmalloc(sizeof(*p), GFP_KERNEL);
 12 p->a = 1;
 13 p->b = 2;
 14 p->c = 3;
 15 list_add_rcu(&p->list, &head);
```

第 15 行需要通过某种同步机制（最常用的是锁）加以保护，避免多个 `list_add()` 并发执行。但这套同步机制 **不会阻止本次 `list_add_rcu()` 和 RCU 读端并发运行**。

订阅受 RCU 保护的链表也很简单：
```c
rcu_read_lock();
list_for_each_entry_rcu(p, head, list) {
  do_something_with(p->a, p->b, p->c);
}
rcu_read_unlock();
```

原语 `list_add_rcu()` 将一个节点发布到指定链表；它保证对应的 `list_for_each_entry_rcu()` 遍历宏可以正确订阅到这个新增节点。

**快速自测题2：** 如果 `list_for_each_entry_rcu()` 和 `list_add_rcu()` 恰好同时执行，是什么机制避免了段错误？

Linux 的另一种双向链表是哈希链表（hlist），它是线性链表。环形链表的链表头需要两个指针，而 hlist 的表头只需要一个指针。因此，在大容量哈希表的哈希桶数组中使用 hlist，可以将内存占用减少一半。

![Linux hlist](pic/wircu_Linux_hlist.jpg)

向受 RCU 保护的 hlist 发布新节点，操作和环形链表非常相似：
```c
  1 struct foo {
  2   struct hlist_node *list;
  3   int a;
  4   int b;
  5   int c;
  6 };
  7 HLIST_HEAD(head);
  8 
  9 /* . . . */
 10 
 11 p = kmalloc(sizeof(*p), GFP_KERNEL);
 12 p->a = 1;
 13 p->b = 2;
 14 p->c = 3;
 15 hlist_add_head_rcu(&p->list, &head);
```

和前面一样，第 15 行代码必须由同步机制（例如锁）保护。

遍历受 RCU 保护的 hlist 的方式也和环形链表类似：
```c
rcu_read_lock();
hlist_for_each_entry_rcu(p, q, head, list) {
  do_something_with(p->a, p->b, p->c);
}
rcu_read_unlock();
```

**快速自测题 3**：为什么 `hlist_for_each_entry_rcu()` 需要传入两个临时指针，而 `list_for_each_entry_rcu()` 只需要一个？

下表列出了全部 RCU 发布、订阅原语，另外还包含用于 “取消发布”（也就是撤销节点）的原语：

Category     | Publish              | Retract                       | Subscribe
-------------|----------------------|-------------------------------|-----------------
**Pointers** | rcu_assign_pointer() | rcu_assign_pointer(..., NULL) | rcu_dereference()
**Lists**    | list_add_rcu()</br>list_add_tail_rcu()</br>list_replace_rcu() | list_del_rcu() | list_for_each_entry_rcu()
**Hlists**   | hlist_add_after_rcu()</br>hlist_add_before_rcu()</br>hlist_add_head_rcu()</br>hlist_replace_rcu() | hlist_del_rcu() | hlist_for_each_entry_rcu()

需要注意，`list_replace_rcu()`、`list_del_rcu()`、`hlist_replace_rcu()`、`hlist_del_rcu()` 这几组接口会带来一个复杂问题：**什么时候才可以安全释放被替换或者被删除的数据节点？** 更具体一点：我们要如何确认，所有读端都已经不再持有该数据节点的引用？

下一节将解答这些问题。

## 等待已启动的 RCU 读端临界区执行完毕
从最根本的形态来看，RCU 就是一种等待任务完成的机制。当然，实现等待的方式还有很多，比如引用计数、读写锁、事件等等。
RCU 的突出优势在于：它可以同时等待（举例来说）20000 个不同的执行流结束，却不需要逐个显式追踪每一个；同时也不会出现显式追踪方案自带的性能下降、可扩展性瓶颈、复杂死锁场景以及内存泄露风险。

在 RCU 中，等待的对象叫做 **RCU 读端临界区**。一个 RCU 读端临界区以 `rcu_read_lock()` 原语开始，以配对的 `rcu_read_unlock()` 原语结束。
RCU 读端临界区支持嵌套，内部几乎可以放置任意代码，但 **不能主动阻塞或睡眠**（有一种特殊的 RCU 实现 [SRCU]，允许在它的读端临界区内睡眠）。
只要遵守这条规则，你就可以利用 RCU 等待指定代码段执行完成。

[经典 RCU] 和 [实时 RCU] 的相关文档里已经讲过：RCU 是通过间接判定的方式，获知这些读端何时执行完毕。

如下图所示：RCU 的作用，就是等待 **在删除操作之前就已经启动** 的那些 RCU 读端临界区（译注：最下面两个 Reader）全部执行完成，包括这些临界区内所有的内存访问操作。

![Grace Period Good](pic/wircu_GracePeriodGood.png)

译注：删除操作的发布即启动一个宽限期（上图中黄色区域最左边的一条线）。发布之前的 Reader 看到的是旧数据，发布之后的 Reader 看到的是新数据。然而即该宽限期需要等到最下面一个 Reader 结束的才能结束（上图中黄色区域最右边的一条线），回收旧数据。

但是请注意：**在某个宽限期（grace period）启动之后才开始的 RCU 读端临界区（译注：除最右边两个 Reader 外的其他 Reader），完全可以、也确实会运行到该宽限期结束之后**。

下面这段伪代码展示了利用 RCU 等待读端完成的基础算法流程：
1. 执行修改操作，例如替换链表中的一个节点。
2. 等待所有在宽限期开始前就已启动的 RCU 读端临界区全部执行完毕（可以使用原语 `synchronize_rcu()`）。这里的核心逻辑是：**宽限期之后新产生的读端，已经无法拿到刚才被移除节点的引用**。
3. 清理资源，比如释放上面被替换掉的节点。

下面的代码片段改编自上一节示例，演示整个流程，其中成员 `a` 作为检索键：
```c
 1 struct foo {
 2   struct list_head list;
 3   int a;
 4   int b;
 5   int c;
 6 };
 7 LIST_HEAD(head);
 8 
 9 /* . . . */
10 
11 p = search(head, key);
12 if (p == NULL) {
13   /* 执行对应处理、解锁并返回 */
14 }
15 q = kmalloc(sizeof(*p), GFP_KERNEL);
16 *q = *p;
17 q->b = 2;
18 q->c = 3;
19 list_replace_rcu(&p->list, &q->list); //GP 开始，在此之前的读端看到的是旧链表，在此之后的读端看到的是新链表
20 synchronize_rcu();                    //GP 结束，阻塞式等待在删除操作之前就已经启动的那些 RCU 读端临界区全部执行完成
21 kfree(p);
```

第 19、20、21 行，正好对应上面所说的三步。
第 16～19 行的逻辑，正是 RCU（read-copy update，读-拷贝-更新）名字的由来：它允许 *读端* 并发访问；第 16 行执行拷贝，17–19 行完成 *更新*。

`synchronize_rcu()` 这个原语初看会有点难以理解：它需要等待全部旧的 RCU 读端临界区结束，而前面我们已经看到，在未开启 `CONFIG_PREEMPT` 的内核中，用来标记读临界区边界的 `rcu_read_lock()` 和 `rcu_read_unlock()` 甚至不会生成任何机器码！

这里有一个关键诀窍：经典 RCU 中由 `rcu_read_lock()` 和 `rcu_read_unlock()` 划定的读端临界区 **不允许阻塞或睡眠**。
因此，当某个 CPU 发生上下文切换时，我们就可以断定：该CPU 上之前启动的所有 RCU 读端临界区都已经执行完毕。
也就是说，只要每一个 CPU 都至少完成一次上下文切换，那么所有在此之前启动的 RCU 读端临界区必然全部结束，此时 `synchronize_rcu()` 就可以安全返回。

因此从原理上理解，经典 RCU 的 `synchronize_rcu()` 可以简化成下面这样：
```c
1 for_each_online_cpu(cpu)
2    run_on(cpu);
```

其中 `run_on()` 会把当前线程调度到指定 CPU 上运行，强制该 CPU 产生一次上下文切换。
这个 `for_each_online_cpu()` 循环会让每个在线 CPU 都发生一次上下文切换，以此保证所有先前的 RCU 读端临界区执行完成。
这种简单思路适用于 RCU 读临界区内禁止抢占的内核，也就是 `CONFIG_PREEMPT` 和非抢占内核；**但不适用于 `CONFIG_PREEMPT_RT` 实时（-rt）内核**。所以实时 RCU 采用了一套不同的实现方案，大体上基于引用计数思想。

当然，Linux 内核里的真实实现要复杂得多：它需要处理中断、NMI、CPU 热插拔以及商用内核里的各类异常场景，同时还要保证不错的性能与可扩展性。
而 RCU 的实时版本还必须保障良好的实时响应延迟，因此像上面这种依赖关闭抢占的简易实现就不可用了。

理解 `synchronize_rcu()` 这个简化原理很有帮助，但还有别的问题待解答。例如：读端遍历一个正在被并发更新的链表时，到底会读到什么内容？下一节就来回答这个问题。

## 为新近更新的对象维护多个版本
本节将说明 RCU 如何维护链表的多版本，从而支持无需同步的读端。
文中给出两个示例，用来阐释一条核心规则：只要某个读端还处在 RCU 读端临界区内，该读端可能正在引用的节点就必须保持完好可用。
第一个示例演示链表节点的删除，第二个演示节点替换。

### 示例 1：删除操作期间维护多版本

我们从“删除”示例开始，对 [上一节示例](#等待已启动的-rcu-读端临界区执行完毕) 的第 11–21 行代码做如下修改
```c
1 p = search(head, key);
2 if (p != NULL) {
3   list_del_rcu(&p->list);
4   synchronize_rcu();
5   kfree(p);
6 }
```

链表的初始状态（包含指针 `p`）说明如下：

![MultiVersion1](pic/wircu_MultiVersion1.jpg)

每个节点里的三元组分别代表字段 `a`、`b`、`c` 的值。节点上的红色边框表示：读端可能持有该节点的引用；由于读端和更新端之间不做直接同步，读端可以和这一整套替换流程并发执行。
为了简洁，图中省略了反向指针，以及链表尾指向链表头的链接。

第 3 行的 `list_del_rcu()` 执行完毕后，节点 `(5,6,7)` 已经从链表中摘除，如下图所示。
因为读端与更新端无直接同步，读端可能正在并发遍历该链表。这些并发读端能否看到这个刚被摘除的节点，取决于时序。
但是，如果某个读端在拿到这个待摘除节点的指针之后被延迟（例如被中断、ECC 内存错误，或是在开启 `CONFIG_PREEMPT_RT` 的内核中被抢占），那么在节点被摘除之后的很长一段时间里，该读端仍然可以访问链表的旧版本。
于是此时链表就存在两个版本：一个版本包含节点 `(5,6,7)`，另一个版本不包含它。
节点 `(5,6,7)` 的边框依旧为红色，代表仍有读端可能正在引用它。

![MultiVersionDelete2](pic/wircu_MultiVersionDelete2.jpg)

请注意：读端在退出 RCU 读端临界区之后，不允许继续持有节点 `(5,6,7)` 的引用。
因此，当第 4 行的 `synchronize_rcu()` 返回时，所有在它之前启动的读端临界区都已经保证执行完成，不再有任何读端会引用该节点，对应节点边框变为黑色。此时链表回到单一版本。

![MultiVersionDelete3](pic/wircu_MultiVersionDelete3.jpg)

到这一步，就可以安全释放节点 `(5,6,7)` 的内存，如下图：

![MultiVersionDelete4](pic/wircu_MultiVersionDelete4.jpg)

至此，节点 `(5,6,7)` 的删除流程完成。下一节介绍节点替换操作。

### 示例 2：替换操作期间维护多版本

我们开始讲解替换示例，先给出上一个示例的后续代码片段：
```c
1 q = kmalloc(sizeof(*p), GFP_KERNEL);
2 *q = *p;
3 q->b = 2;
4 q->c = 3;
5 list_replace_rcu(&p->list, &q->list);
6 synchronize_rcu();
7 kfree(p);
```

链表初始状态（包含指针 `p`）和删除示例完全一致：

![rMultiVersion1](pic/wircu_rMultiVersion1.jpg)

和之前一样，每个节点中的三元组依次代表字段 `a`、`b`、`c` 的值。
节点红色边框代表读端可能持有该节点引用；由于读端与更新端之间没有直接同步，读端可以和整个替换流程并发执行。
为便于理解，这里同样省略了反向指针以及链表尾指向链表头的链接。

第 1 行：调用 `kmalloc()` 分配用于替换的新节点：

![rMultiVersion2](pic/wircu_rMultiVersion2.jpg)

第 2 行：将旧节点内容拷贝到新节点：

![rMultiVersion3](pic/wircu_rMultiVersion3.jpg)

第 3 行：把 `q->b` 修改为 `2`：

![rMultiVersion4](pic/wircu_rMultiVersion4.jpg)

第 4 行：把 `q->c` 修改为 `3`：

![rMultiVersion5](pic/wircu_rMultiVersion5.jpg)

第 5 行执行替换动作，新节点至此对读端可见。
此时链表存在两个版本：在替换之前就已经开始的老读端，仍然可以看到旧节点（`5,6,7`）；而新来的读端将看到新节点（`5,2,3`）。
但对于任意一个读端，它看到的链表一定是合法、完整的。

![rMultiVersion6](pic/wircu_rMultiVersion6.jpg)

第 6 行 `synchronize_rcu()` 返回后，宽限期已经结束。所有在 `list_replace_rcu()` 之前发起的读操作都已经完成。
特别是，所有曾经持有旧节点（`5,6,7`）引用的读端，都已经退出 RCU 读端临界区，不再允许继续持有该引用。
旧节点（`5,6,7`）的边框变为细黑边，代表已经没有读端引用它。站在读端视角，链表变回单一版本，只是新节点替换掉了旧节点。

![rMultiVersion7](pic/wircu_rMultiVersion7.jpg)

第7行 `kfree()` 执行完成后，链表状态如下：

![rMultiVersion8](pic/wircu_rMultiVersion8.jpg)

尽管 RCU 这个名字本身来源于 **替换场景**，但 Linux 内核里绝大多数 RCU 的实际使用，都是 [上一节](#示例-1删除操作期间维护多版本) 介绍的简单删除场景。


### 讨论

上面这些示例都假设：整个更新过程全程持有互斥锁，这意味着任意时刻，最多只会同时存在两个版本的链表。

**快速自测题 4**：如何修改删除示例，使得链表可以同时存在两个以上的有效版本？

**快速自测题 5**：一条链表在任意时刻，最多可以有多少个 RCU 版本同时生效？

上面这一系列流程展示了 RCU 更新是如何借助多版本机制，在存在并发读端的情况下安全完成数据修改。
当然，部分算法无法友好地处理多版本。有相关 [技术方案](http://www.rdrop.com/users/paulmck/RCU/RCUdissertation.2004.07.14e1.pdf) 可以改造这类算法以适配 RCU，但已经超出本文的讨论范围。

## 小结

本文介绍了基于 RCU 的算法的三大基础组成：
1. 用于新增数据的发布-订阅机制；
2. 等待已存在的 RCU 读端执行完毕的机制；
3. 维护多版本数据的规范，以此支持修改操作，而不会破坏并发读端，也不会不必要地阻塞读端。

**快速自测题6**：`rcu_read_lock()` 和 `rcu_read_unlock()` 原语既不忙等也不阻塞，那 RCU 更新端还有可能拖慢 RCU 读端吗？

RCU 的这三大组件，让数据在有并发读访问的场景下仍然可以被更新。它们能够以不同方式组合，实现种类非常丰富的 RCU 算法，其中一部分将在本系列《What is RCU, Really?》的下一篇中讲解。

## 致谢

本文初稿承蒙 Andy Whitcroft、Gautham Shenoy 以及 Mike Fulton 审阅，他们的意见极大完善了本文，在此深表谢意。感谢相对论编程项目（Relativistic Programming project）成员以及 PNW TEC 的各位同仁带来大量富有价值的讨论。感谢 Dan Frye 对本工作的支持。
最后，本研究内容基于美国国家科学基金会（National Science Foundation）资助项目（项目编号：CNS-0719851）完成。

本文仅代表作者观点，并不一定代表 IBM 或波特兰州立大学（Portland State University）的立场。

Linux 是 Linus Torvalds 的注册商标。

文中出现的其他公司、产品及服务名称，可能是对应主体的商标或服务商标。

## 快速自测题答案

**快速自测题 1**：seqlock 不是同样允许读端和更新端并发执行吗？

**答**：是，也不是。虽然 seqlock 的读端可以和写端并发运行，但一旦发生这种情况，`read_seqretry()` 原语会强制读端重试。也就是说，和更新端并发的 seqlock 读端此前做的所有计算都会被丢弃，需要重新执行。
因此 seqlock 读端虽然能和更新端并发跑，但这种情况下它无法真正完成有效工作。

与之不同，RCU 读端即使在更新端并发修改数据时，仍然可以完成有意义的工作。

**快速自测题 2**：如果 `list_for_each_entry_rcu()` 和 `list_add_rcu()` 恰好同时执行，为什么不会触发段错误？

**答**：在所有运行 Linux 的系统上，指针的加载与存储操作都是原子的。也就是说，当对同一个指针的读、写同时发生时，读操作只会拿到修改前的值或者修改后的值，绝不会拿到两者比特混杂的非法值。
除此之外，`list_for_each_entry_rcu()` 只会沿着链表向前遍历，不会回头访问。因此，它要么能看到 `list_add_rcu()` 新增的节点，要么看不到；无论哪种情况，它看到的链表都是合法、结构完整的。

**快速自测题 3**：为什么 `hlist_for_each_entry_rcu()` 需要传入两个指针，而 `list_for_each_entry_rcu()` 只需要一个？

**答**：因为 hlist（哈希链表）的终止条件是判断是否为 `NULL`，而不是判断是否碰到链表头。（你可以试着只用一个指针实现 `hlist_for_each_entry_rcu()`。如果你能想出优雅的实现方案，那会非常有价值！）

**快速自测题 4**：如何修改删除示例，让链表同时存在两个以上的有效版本？

**答**：一种实现方式如下：
```c
spin_lock(&mylock);
p = search(head, key);
if (p == NULL)
  spin_unlock(&mylock);
else {
  list_del_rcu(&p->list); //发布，开始一个新的 GP
  spin_unlock(&mylock);   //解锁之后其他路径可以拿到锁，然后发布，开启一个新的 GP
  synchronize_rcu();      //等待 GP 结束
  kfree(p);
}
```
注意：这样修改后，多个并发删除操作可能会阻塞在 `synchronize_rcu()` 处等待宽限期结束。

**快速自测题 5**：一条链表在任意时刻，最多可以同时存在多少个 RCU 有效版本？

**答**：取决于同步方案。如果保护更新的信号量在整个宽限期内都持有，那么最多同时存在两个版本：旧版本和新版本。

但是，如果锁只保护查找、修改以及 `list_replace_rcu()` 这一小段逻辑，那么理论上可以存在任意多个有效版本，上限仅受内存大小以及单个宽限期内能够完成的更新次数限制。
不过要注意：更新频率极高的数据结构，通常并不适合用 RCU。当然，在有需要的时候，RCU 也可以支撑很高的更新速率。

**快速自测题 6**：`rcu_read_lock()` 和 `rcu_read_unlock()` 既不忙等也不阻塞，RCU 更新端怎么还可能拖慢读端？

**答**：RCU 更新端执行的数据修改操作，会让对应 CPU invalidate 包含该数据的 cache lines，导致正在并发读 RCU 的其他 CPU 产生开销很高的 cache misses。（你能否设计一种算法，修改数据结构时 *不会* 给并发读端、以及后续读端带来昂贵的 cache misses？）

# What is RCU? Part 2: Usage

## 引言

读-拷贝-更新（RCU）是一种同步机制，于 2002 年 10 月引入 Linux 内核。RCU 最常被说成是读写锁的替代品，但它也有很多其他使用方式。
RCU 的突出特点是：RCU 读端并不与更新端直接同步，这使得 RCU 读路径极快；并且即便和更新端并发运行，读端依然可以完成有效工作。

这就引出了一个问题：**RCU 到底是什么？** 本文将从使用者的视角来解答这个问题。由于 RCU 大多用来替换已有同步机制，我们主要通过它和这些机制的对比关系来介绍：

1. [RCU 是读写锁的替代品](#rcu读写锁的替代方案)
2. [RCU 是受限版引用计数机制](#rcu-是一种受限的引用计数机制)
3. [RCU 是批量引用计数机制](#rcu-是一种批量引用计数机制)
4. [RCU 是穷人版垃圾回收器](#rcu穷人版垃圾回收器)
5. [RCU 是一种提供对象存在性保证的手段](#rcu-是一种提供存在性保证的机制)
6. [RCU 是一种等待操作完成的方式](#rcu-是一种等待操作完成的方式)

以上章节之后是 [总结](#小结-2)，以及 [快速自测题的答案](#自测题答案)。

## RCU：读写锁的替代方案
在 Linux 内核中，RCU 最常见的用途，就是在读密集场景下替代读写锁。不过，我一开始并没有立刻意识到这种用法。
事实上，早在 90 年代初，在实现通用 RCU 之前，我先打算实现一个和 brlock（大读锁）类似的原语。当初设想给这个 brlock 原型原语的所有应用场景，最后全都改用 RCU 实现了。
这个 brlock 原型原语，甚至过了三年多才迎来第一次实际使用。唉，当时真是觉得自己很蠢！

RCU 和读写锁最核心的 **共同点**：二者都支持读端临界区并行执行。在部分场景下，甚至可以直接把读写锁 API 替换成对应的 RCU API。但首先会有一个问题：这么做意义何在？

RCU 的 **优势** 包括性能更好、不会死锁、实时延迟更优。当然 RCU 也存在 **局限**：读者和更新者可以并发运行；低优先级的 RCU 读者，有可能阻塞正在等待宽限期结束的高优先级线程；宽限期的延迟最高可达数毫秒。下文会逐一讨论这些优势与局限。

### 性能
下图展示了在一台 16 核、3GHz Intel x86 机器上，RCU 相比读写锁在读端性能上的优势。

![rwlockRCUperf](pic/wircu_rwlockRCUperf.jpg)

可以看到：单核场景下，读写锁的性能比 RCU 慢数个数量级；到 16 核时，读写锁又 *额外* 慢了接近两个数量级。与之相反，RCU 的扩展性非常好。两组测试里，误差棒（error bars）代表正负一个标准差。

**快速自测题 1**：WTF??? 3GHz 主频的时钟周期都超过 300 *皮秒（picosecond）* 了，你怎么能说 RCU 的开销只有 100 飞秒（femtosecond）？

开启`CONFIG_PREEMPT`（可抢占内核）后，差距会收敛（moderate）一些，但 RCU 依旧比读写锁快 1~3 个数量级。可以注意到，CPU 数量变多时，读写锁的测试波动（variability）非常大。误差棒同样代表正负一个标准差。

![rwlockRCUperfPREEMPT](pic/wircu_rwlockRCUperfPREEMPT.jpg)

当然，当临界区内的计算开销变大时，读写锁本身的性能劣势就不再那么突出，16 核机器上的下图也印证了这点。

![rwlockRCUperfwtPREEMPT](pic/wircu_rwlockRCUperfwtPREEMPT.jpg)

但这点需要辩证（tempered ）看待：不少系统调用（以及其中包含的 RCU 读端临界区）本身就能在数微秒内执行完毕。此外，下一节会讲到，RCU 读端原语几乎不会引发死锁。

**快速自测题 2**：为什么临界区自身开销变大时，rwlock 的开销和波动都会变小？

### 免死锁特性（Deadlock Immunity）
尽管在读多写少的负载场景下 RCU 具备显著的性能优势，但当初设计 RCU 最核心的原因之一，其实是它的读端不会引发死锁。该（immunity ）特性源于（stems from）：RCU 读端原语不会阻塞、不会自旋，甚至不存在反向分支（backwards branches），执行时间是确定的。因此，读端代码不可能参与到死锁环中。

**快速自测题 3**：这种免死锁特性是否存在例外？如果有，什么样的事件序列会引发死锁？

RCU 读端免死锁带来一个很有价值的推论：可以无条件地把 RCU 读端上下文升级为更新操作。如果用 **读写锁做同类的读转写升级，就会直接死锁**。下面是 RCU 读上下文升级到更新操作的示例代码：
```c
 1 rcu_read_lock();
 2 list_for_each_entry_rcu(p, head, list_field) {
 3   do_something_with(p);
 4   if (need_update(p)) {
 5     spin_lock(&my_lock);
 6     do_update(p);
 7     spin_unlock(&my_lock);
 8   }
 9 }
10 rcu_read_unlock();
```
注意：`do_update()` 是在自旋锁保护，*同时* 也在 RCU 读端保护下执行的。

RCU 免死锁特性带来的另一个好处：它可以规避一大类优先级反转问题。例如，低优先级的 RCU 读者不会阻碍高优先级的 RCU 更新者获取更新侧锁；同理，低优先级的 RCU 更新者也不会阻止高优先级的 RCU 读者进入 RCU 读端临界区。

### 实时延迟
RCU 读端原语既不会自旋，也不会阻塞，因此具备优异的实时延迟特性。此外前面也提到过：这意味着它可以避免一类由 RCU 读端原语与锁共同引发的优先级反转问题。

但是，RCU 仍会遇到更隐蔽的优先级反转场景。举例来说：在 `-rt` 实时内核中，一个高优先级进程阻塞等待 RCU 宽限期结束时，有可能被低优先级的 RCU 读者拖住。该问题可以通过 **[RCU 优先级提升（RCU priority boosting）](http://lwn.net/Articles/220677/)** 解决。

### RCU 读者与更新者并发执行
由于 RCU 读者既不会自旋也不会阻塞，并且更新操作不存在回滚或中止语义，RCU 的读者和更新者必然是并发运行的。这意味着 RCU 读者有可能读到旧数据，甚至观察到不一致状态。以上两点，使得从读写锁迁移到 RCU 并不是一件简单的事。

不过在相当多场景下，不一致和过期数据并不会造成问题。最经典的例子就是网络路由表：路由更新本身就需要很长时间才能抵达一台主机（数秒甚至数分钟），在更新到达之前，系统其实已经在错误转发报文很长一段时间了。再多维持几毫秒的旧路由转发，通常是可以接受的。
此外，RCU 更新者不需要等待读者完成就可以提交修改，因此 RCU 读者反而有可能比批处理公平读写锁的读者更早感知到变更，如下图所示。

![rwlockRCUupdate](pic/wircu_rwlockRCUupdate.jpg)

译注：左边的事件先发生，右边的事件后发生。RCU 的写者不阻塞读者，只是读者读到的数据有可能是旧的（发布前）也有可能是新的（发布后）。

更新抵达后，读写锁的写者必须等到最后一个读者完成才能继续；而后续新来的读者，也必须等写者执行完毕才能运行。但这些后续读者一定能读到新值，对应图中的绿色背景。

与之对比，RCU 的读者和更新者互不阻塞，因此 RCU 读者可以更早看到更新后的数据。当然，因为读者和更新者的执行时间存在重叠，*所有* RCU 读者都有可能读到新值，包括那三个在更新发起前就已经启动的读者。但只有图中标为绿色背景的 RCU 读者，才被 *保证* 一定读到新值。

读写锁与 RCU 提供的保证本身就不一样：

使用读写锁时，在写者启动（译注：拿到锁）之后才开始的读者，一定可以读到新值；而在写者自旋（译注：争用锁）期间尝试启动的读者能否读到新值，则取决于该读写锁实现是读优先还是写优先。

而 RCU 的规则是：在更新操作完成之后才启动的读者，一定读到新值；在更新操作启动之后才结束的读者，能否读到新值取决于时序。

这里的核心观点是：虽然读写锁确实能保证计算机系统内部的数据一致性，但在某些场景下，这种内部一致性的代价是 **和外部世界之间的不一致性变大**。换句话说，读写锁换来了系统内部一致，代价是默默持有相对于外部世界的过期数据。

当然，也存在一些场景，系统内部的不一致和过期数据是完全不可接受的。好在已经有不少方案可以规避这类问题，相关内容可以参考 [FREENIX 那篇将 RCU 应用于 System V IPC 的论文](http://www.rdrop.com/users/paulmck/RCU/rcu.FREENIX.2003.06.14.pdf) 以及我的 [博士论文](http://www.rdrop.com/users/paulmck/RCU/RCUdissertation.2004.07.14e1.pdf)。不过对这些方案的深入讨论超出了本文的范围。

### 低优先级 RCU 读者可能阻塞高优先级回收任务
实时 RCU、[SRCU](http://lwn.net/Articles/202847/) 和 [QRCU](http://lwn.net/Articles/243851/)（这三者都会在本系列最后一篇中介绍）存在这样一个现象：一个被抢占的读者会阻止宽限期结束，即便此时有高优先级任务正在阻塞、等待该宽限期完成。
实时 RCU 可以通过用 `call_rcu()` 替换 `synchronize_rcu()`，或者使用 [RCU 优先级提升机制](http://lwn.net/Articles/220677/) 来规避该问题；截至 2007 年末，RCU 优先级提升还处于实验阶段。未来或许也需要给 SRCU 和 QRCU 增加优先级提升能力，但前提是要先证明真实场景下确实存在这个需求。

### RCU 宽限期长达数毫秒
除 QRCU 之外，RCU 的宽限期通常会持续数毫秒。虽然有不少方法可以消除这类长延迟带来的负面影响，比如在条件允许时使用异步接口（`call_rcu()`、`call_rcu_bh()`），但这也是一条经验准则的核心原因：**RCU 适合在读多写少场景使用**。

### 读写锁与 RCU 代码对比
最优情况下，从读写锁迁移到 RCU 的改造非常简单，下面这个示例取自 [维基百科](http://en.wikipedia.org/wiki/Read-copy-update)。

```c
 1 struct el {                          1 struct el {
 2   struct list_head list;             2   struct list_head list;
 3   long key;                          3   long key;
 4   spinlock_t mutex;                  4   spinlock_t mutex;
 5   int data;                          5   int data;
 6   /* Other data fields */            6   /* Other data fields */
 7 };                                   7 };
 8 rwlock_t listmutex;                  8 spinlock_t listmutex;
 9 struct el head;                      9 struct el head;

 1 int search(long key, int *result)    1 int search(long key, int *result)
 2 {                                    2 {
 3   struct list_head *lp;              3   struct list_head *lp;
 4   struct el *p;                      4   struct el *p;
 5                                      5
 6   read_lock(&listmutex);             6   rcu_read_lock();
 7   list_for_each_entry(p, head, lp) { 7   list_for_each_entry_rcu(p, head, lp) {
 8     if (p->key == key) {             8     if (p->key == key) {
 9       *result = p->data;             9       *result = p->data;
10       read_unlock(&listmutex);      10       rcu_read_unlock();
11       return 1;                     11       return 1;
12     }                               12     }
13   }                                 13   }
14   read_unlock(&listmutex);          14   rcu_read_unlock();
15   return 0;                         15   return 0;
16 }                                   16 }

 1 int delete(long key)                 1 int delete(long key)
 2 {                                    2 {
 3   struct el *p;                      3   struct el *p;
 4                                      4
 5   write_lock(&listmutex);            5   spin_lock(&listmutex);
 6   list_for_each_entry(p, head, lp) { 6   list_for_each_entry(p, head, lp) {
 7     if (p->key == key) {             7     if (p->key == key) {
 8       list_del(&p->list);            8       list_del_rcu(&p->list);
 9       write_unlock(&listmutex);      9       spin_unlock(&listmutex);
                                       10       synchronize_rcu();
10       kfree(p);                     11       kfree(p);
11       return 1;                     12       return 1;
12     }                               13     }
13   }                                 14   }
14   write_unlock(&listmutex);         15   spin_unlock(&listmutex);
15   return 0;                         16   return 0;
16 }                                   17 }
```

更复杂的读写锁替换场景不在本文讨论范围内。

## RCU 是一种受限的引用计数机制
只要还有 RCU 读端临界区正在执行，宽限期就不能结束，基于这一点，RCU 读端原语可以当作一种受限的引用计数机制来使用。请看下面这段代码：
```c
1 rcu_read_lock();  /* 获取引用 */
2 p = rcu_dereference(head);
3 /* 使用 p 做一些操作 */
4 rcu_read_unlock();  /* 释放引用 */
```
可以把 `rcu_read_lock()` 理解成对指针 `p` 获取一个引用：在 `rcu_dereference()` 读取到 `p` 之后才启动的宽限期，必须等到对应的 `rcu_read_unlock()` 执行完之后才能够结束。
这套引用计数方案是 **受限** 的：RCU 读端临界区内不允许阻塞，也不允许把一个 RCU 读端临界区从一个任务转交（hand off）给另一个任务。

即便存在这些限制，下面这段代码也可以安全地释放 `p`：
```c
1 spin_lock(&mylock);
2 p = head;
3 head = NULL;
4 spin_unlock(&mylock);
5 synchronize_rcu();  /* 等待所有引用释放 */
6 kfree(p);
```
给 `head` 赋值为空，阻止后续代码再获取 `p` 的引用；`synchronize_rcu()` 则等待之前已经拿到的引用全部释放。

**快速自测题 4**：等等！这段代码和把 RCU 当成读写锁替代品时用的代码完全一样！这该怎么解释？

当然，RCU 也可以和传统引用计数组合使用，LKML（Linux 内核邮件列表）上已有相关讨论，[Overview of Linux-Kernel Reference Counting](http://open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2167.pdf) 一文也做了总结。

但为什么要这么做（bother）？答案一部分仍然是性能，如下图所示，测试数据同样来自一台 16 核、3GHz Intel x86 机器。

![refRCUperfPREEMPT](pic/wircu_refRCUperfPREEMPT.jpg)

和读写锁的场景一样，RCU 的性能优势在临界区很短的时候最为突出，下图是 16 核机器上的测试结果。此外，和读写锁场景类似，很多系统调用（以及其中包含的 RCU 读端临界区）都能在数微秒内完成。

**快速自测题 5**：为什么引用计数的开销曲线在接近 6 核处出现一个低谷（dip）？

![refRCUperfwtPREEMPT](pic/wsrcu_refRCUperfwtPREEMPT.jpg)

但是 RCU 附带的这些限制有时会非常苛刻（onerous）。举个例子：很多场景下，RCU 读端临界区内禁止睡眠这一条，会直接让方案无法落地。下一节会介绍解决该问题的办法，部分方案同时还能降低传统引用计数的复杂度。

## RCU 是一种批量引用计数机制

上一节提到，传统引用计数器通常绑定某一个特定数据结构，或者一组特定的数据结构。但如果为大量不同的数据结构维护同一个全局引用计数器，往往会造成存放该计数值的 cache lines 频繁乒乓（bouncing）。这种 cache-line bouncing 会严重损害性能。

与之相比，RCU 的轻量读端原语支持极高频率的读操作，而性能损耗几乎可以忽略不计，因此 RCU 可以作为 **批量引用计数（bulk reference-counting）** 机制使用，且几乎不带来性能损失。
如果需要让单个任务持有引用、并且在持有引用的代码段中允许阻塞，可以使用 [可睡眠 RCU（SRCU）](http://lwn.net/Articles/202847/)。
但还有一类很常见的场景 SRCU 无法很好覆盖：引用需要从一个任务传递给另一个任务。
例如，发起 I/O 时获取引用，在对应的 I/O 完成中断处理函数中释放引用。（理论上 SRCU 实现可以支持该场景，但实际上，目前还不清楚这么做是否是合适的取舍。）

当然，SRCU 本身也自带一条限制：`srcu_read_lock()` 的返回值必须传给对应的 `srcu_read_unlock()`。
该限制究竟会带来多大麻烦、以及最优的处理方式是什么，目前尚无定论。

## RCU：穷人版垃圾回收器
刚接触 RCU 的人经常会感慨（exclamation）：“RCU 有点像垃圾回收器！”这个说法有一定道理（grain of truth），但也容易造成误解。

理解 RCU 和自动垃圾回收器（GC）之间关系比较合适的方式是：二者的相似点在于，回收 *时机* 都是自动决定的；但 RCU 和 GC 也存在两点关键区别：
1. 程序员必须手动标记某个数据结构何时可以（eligible）被回收；
2. 程序员必须手动标出 RCU 读端临界区 —— 在这些区域内，可能合法（legitimately）持有对象引用。

尽管存在这些差异，二者的相似性其实很深，至少有一篇 RCU 的理论分析文章就提到了这一点。
另外，据我所知，第一个类 RCU 机制，就是依靠垃圾回收器来实现宽限期的。不过，下一节会介绍一种对 RCU 更贴切的理解方式。

## RCU 是一种提供存在性保证的机制

[Gamsa 等人的论文](http://www.usenix.org/events/osdi99/full_papers/gamsa/gamsa.pdf) 讨论了 **存在性保证（existence guarantees）**，并介绍了如何使用类似 RCU 的机制来提供这类保证（参见论文第 7 页的第 5 节）。
其核心效果是：只要在 RCU 读端临界区内访问某个受 RCU 保护的数据元素，就可以保证该数据元素在这个 RCU 读端临界区的整个生命周期内持续存在。

细心的读者会发现，这和最初 “RCU 是等待操作完成的一种手段” 这一核心观点差别很小，下一节将继续展开这个话题。

## RCU 是一种等待操作完成的方式
正如本系列第一篇文章所述，RCU 的一个核心组成部分，就是等待 RCU 读者执行完毕。
RCU 的一大优势：它可以等待成千上万个不同的操作完成，却不需要显式跟踪每一个操作；同时也规避了显式跟踪方案与生俱来的性能衰减、扩展性瓶颈、复杂死锁场景以及内存泄漏风险。

本节将说明：`synchronize_sched()` 对应的读端上下文（包括所有关闭抢占的代码、以及关闭中断的硬件操作和原语），如何实现与不可屏蔽中断（NMI）处理函数之间的同步交互。
如果使用传统锁，这类同步会非常棘手。
我在 [博士论文](http://www.rdrop.com/users/paulmck/RCU/RCUdissertation.2004.07.14e1.pdf) 中将这套方案命名为 **纯 RCU（Pure RCU）**，Linux 内核多处都用到了该机制。

这类“纯 RCU”设计的基本流程如下：
1. 执行变更，例如修改操作系统对 NMI 的响应方式。
2. 等待所有已经启动的读端临界区全部执行完毕（例如使用 `synchronize_sched()` 原语）。这里关键结论：之后新进入的 RCU 读端临界区一定能看到本次变更。
3. 收尾清理，例如返回状态，标识变更已成功完成。

本节后续给出改编自 Linux 内核的示例代码。例子里的 `nmi_stop` 函数借助 `synchronize_sched()`，保证在释放相关资源前，所有正在处理中的（in-flight）NMI 通知都已经执行完成。
下面是简化版代码：

```c
 1 struct profile_buffer {
 2   long size;
 3   atomic_t entry[0];
 4 };
 5 static struct profile_buffer *buf = NULL;
 6 
 7 void nmi_profile(unsigned long pcvalue)
 8 {
 9   atomic_t *p = rcu_dereference(buf);
10 
11   if (p == NULL)
12     return;
13   if (pcvalue >= p->size)
14     return;
15   atomic_inc(&p->entry[pcvalue]);
16 }
17 
18 void nmi_stop(void)
19 {
20   atomic_t *p = buf;
21 
22   if (p == NULL)
23     return;
24   rcu_assign_pointer(buf, NULL);
25   synchronize_sched();
26   kfree(p);
27 }
```

第 1–4 行定义了 `profile_buffer` 结构体，包含一个大小字段和长度不定的条目数组（indefinite array of entries）。
第 5 行定义一个指向性能采样缓冲区（profile buffer）的指针，该指针会在别处初始化，指向一块动态分配的内存。

第 7–16 行是 `nmi_profile()` 函数，该函数在 NMI 处理函数内部调用。
既然运行在 NMI 上下文，它不会被抢占，也不会被普通中断处理函数打断；但依然会因为 cache misses、ECC 纠错、同核心内其他硬件线程抢占周期（cycle stealing）而产生延迟。
第 9 行通过 `rcu_dereference()` 获取缓冲区的本地指针，用来保证 DEC Alpha 平台上的内存序；
第 11、12 行：如果当前没有分配缓冲区，直接退出；
第 13、14 行：如果入参 `pcvalue` 越界，也直接退出。
否则第 15 行按照 `pcvalue` 作为下标，对缓冲区对应条目计数加一。
注意：缓冲区自身携带 `size` 字段，保证范围检查和缓冲区匹配。即便一个大缓冲区突然被替换成更小的缓冲区，也不会出现越界访问。

第 18–27 行定义 `nmi_stop()` 函数，调用方需要自行保证互斥（例如持有对应的锁）。
第 20 行取出缓冲区指针；
第 22、23 行：缓冲区为空则直接返回。
否则第 24 行把全局缓冲区指针置空（使用 `rcu_assign_pointer()`，在弱序平台上维护内存序）；
第 25 行等待 RCU-sched 宽限期结束，也就是等待所有不可抢占代码段（包括 NMI 处理函数）全部执行完成。
执行走到第 26 行时，可以保证：所有拿到旧缓冲区指针的 `nmi_profile()` 调用都已经返回。此时就可以安全释放缓冲区，这里用 `kfree()`。

简言之，RCU 让动态切换性能采样缓冲区变得简单高效（你可以试试只用原子操作做到同等效率，或者用锁来实现这套逻辑，难度会非常高！）。
不过，正如前面小节展示的，RCU 通常在更高抽象层级上使用。

**快速自测题 6**：假设 `nmi_profile()` 允许被抢占，要做哪些修改才能让这个例子正确运行？

## 小结 2
RCU 的本质，就是一套 API，它提供：
- 用于新增数据的发布-订阅机制；
- 等待已启动 RCU 读者执行完毕的手段；
- 维护多版本数据的规范（discipline），使得修改操作不会破坏并发 RCU 读者，也不会造成不必要的延迟。

在此基础上，可以基于 RCU 构建高层抽象（higher-level constructs），包括前文介绍的读写锁、引用计数、存在性保证等机制。
此外，我毫不怀疑，Linux 社区还会持续挖掘 RCU 以及其他各类同步原语（synchronization primitive）新颖且有价值的用法。

## 自测题答案
**快速自测题 1**：WTF??? 3GHz 主频的时钟周期都超过 300 皮秒了，你怎么能相信 RCU 的开销只有 100 飞秒？

答：首先，本次测试所用的内层循环代码如下：
```c
1 for (i = 0; i < CSCOUNT_SCALE; i++) {
2   rcu_read_lock();
3   rcu_read_unlock();
4 }
```
其次，我们来看 `rcu_read_lock()` 和 `rcu_read_unlock()` 的等效定义：
```c
1 #define rcu_read_lock()   do { } while (0)
2 #define rcu_read_unlock() do { } while (0)
```
再加上编译器会做简单优化，直接把整个循环优化成：
```c
i = CSCOUNT_SCALE;
```
所以这个 100 飞秒的“测量值”，本质只是计时本身的固定开销，除以循环迭代次数得到的结果。
因此这个测量结果本身是错的，误差甚至可以达到任意多个数量级。从上面宏定义就能看到，**实际开销就是 0**。

能看到 100 飞秒的计时结果反而属于高估，这种情况可不常见！

**快速自测题 2**：为什么临界区自身开销变大时，rwlock 的开销和波动都会变小？

答：因为临界区开销变大后，底层 `rwlock_t` 的竞争程度下降。但受缓存颠簸（cache-thrashing）开销影响，读写锁的开销并不会降到单核场景下的水平。

**快速自测题 3**：RCU 读端免死锁特性是否存在例外？如果有，什么样的操作序列会引发死锁？

答：下面这一段（非法）代码序列，就会形成包含 RCU 读端原语的死锁环：
```c
idx = srcu_read_lock(&srcucb);
synchronize_srcu(&srcucb);
srcu_read_unlock(&srcucb, idx);
```
`synchronize_srcu()` 需要等待所有已经启动的 SRCU 读端临界区完成才会返回；但它本身又包裹在一个 SRCU 读端临界区里面，这个读端临界区又要等 `synchronize_srcu()` 返回才能结束。最终形成经典的自死锁 —— 和持有读写锁读锁时，再去申请写锁的效果一模一样。

注意：这个自死锁场景 **不适用于经典 RCU**。经典 RCU 里 `synchronize_rcu()` 会触发上下文切换，上下文切换本身就是该 CPU 的静默状态（quiescent state），宽限期可以正常完成。
但这反而更危险：宽限期完成之后，RCU 读端临界区正在访问的数据有可能被释放。

简而言之，**不要在 RCU 读端临界区里面调用同步类 RCU 更新原语**。

**快速自测题 4**：等等！这段代码和把 RCU 当成读写锁替代品时写的代码完全一样！这该怎么解释？

答：这是“示例代码定律（Law of Toy Examples）”带来的现象：简单示例写到一定程度，代码长得都会一样。唯一区别在于我们理解代码的角度。
但这个理解角度的差异至关重要。举个例子：如果我们把 RCU 理解成受限引用计数机制，就不会误以为更新操作会排斥 RCU 读端临界区。

当然，把 RCU 理解成读写锁替代品在很多场景下仍然很有用，比如你正在把读写锁迁移为 RCU 的时候。

**快速自测题 5**：为什么引用计数的开销曲线在接近 6 核处出现低谷？

答：极有可能是 NUMA 效应。另外从误差棒可以看到，引用计数这组测量值离散度很大，部分场景下标准差甚至超过测量值的 10%。所以这个开销低谷，也很可能只是统计上的偶然偏差。

**快速自测题 6**：假设 `nmi_profile()` 函数允许被抢占，要做哪些修改才能让这个例子正确运行？

答：一种方案是在 `nmi_profile()` 中使用 `rcu_read_lock()` / `rcu_read_unlock()`，并且把 `synchronize_sched()` 替换成 `synchronize_rcu()`，修改示意大致如下：

```c
 1 struct profile_buffer {
 2   long size;
 3   atomic_t entry[0];
 4 };
 5 static struct profile_buffer *buf = NULL;
 6 
 7 void nmi_profile(unsigned long pcvalue)
 8 {
 9   atomic_t *p;
10 
11   rcu_read_lock();
12   p = rcu_dereference(buf);
13   if (p == NULL) {
14     rcu_read_unlock();
15     return;
16   }
17   if (pcvalue >= p->size) {
18     rcu_read_unlock();
19     return;
20   }
21   atomic_inc(&p->entry[pcvalue]);
22   rcu_read_unlock();
23 }
24 
25 void nmi_stop(void)
26 {
27   atomic_t *p = buf;
28 
29   if (p == NULL)
30     return;
31   rcu_assign_pointer(buf, NULL);
32   synchronize_rcu();
33   kfree(p);
34 }
```

# RCU part 3: the RCU API

## 引言
读-拷贝-更新（RCU）是一种同步机制，于 2002 年 10 月加入 Linux 内核。RCU 最常被视作读写锁的替代方案，但它还有很多其他用途。
RCU 的一大特点是：RCU 读端与更新端之间不需要直接同步，这使得 RCU 读路径的开销极低；同时，即使读端和更新端并发执行，读端依然可以正常完成有效工作。

由此引出一个问题：**RCU 究竟是什么？** 本文档将从 Linux 内核 RCU API 的角度来回答这个问题。

1. [RCU 拥有一组“等待操作完成”的 API](#rcu-拥有一组等待完成wait-to-finish系列api)
2. [RCU 提供发布-订阅以及多版本维护 API](#rcu-发布-订阅与版本维护-api)
3. [那么，RCU 的本质到底是什么？](#那么rcu-的本质究竟是什么)

上述章节之后是 [参考文献]()，以及 [快速自测题的答案]()。

## RCU 拥有一组“等待完成（Wait-to-Finish）”系列API
对于“RCU 是什么”这个问题，最直白的回答是：RCU 是 Linux 内核中使用的一套 API，本节的两张表格对其做了汇总（第一张表格展示 API 中等待 RCU 读端完成的部分，第二张表格展示发布/订阅相关的 API）。或者更准确地说，RCU 是由多张表格列出的一系列 API 构成的集合，表格中的每一列对应 RCU API 家族里的一个分支。

如果你是 RCU 初学者，可以先只聚焦下表中的某一列。举例来说，如果你主要想了解 RCU 在 Linux 内核里的实际用法，可以从 **经典 RCU（RCU Classic）** 入手，它的使用场景最多。
反之，如果你单纯想研究 RCU 本身的原理，**SRCU** 的 API 是最简单的。其余分支你可以后续再学习。

如果你已经熟悉 RCU，下面这两张表格也可以当作一份实用参考手册。

属性    | RCU Classic | RCU BH         | RCU Sched      | Realtime RCU | SRCU      | QRCU
--------|-------------|---------------|-----------------|--------------|----------|--------------------
目的     | 原本的       | 防止 DDoS 攻击 | 等待硬中断和 NMI | 实时响应      | 读者睡眠 |	读者睡眠与快速宽限期
可用性   | 2.5.43   	 | 2.6.9          | 2.6.12          | Aug 2005 -rt | 2.6.19   |
读侧原语 | <span style="background:#ddffdd;">rcu_read_lock()</br>rcu_read_unlock()</span> | rcu_read_lock_bh()</br>rcu_read_unlock_bh() | preempt_disable()</br>preempt_enable()(and friends) | rcu_read_lock()</br>rcu_read_unlock() | srcu_read_lock()</br>srcu_read_unlock() | qrcu_read_lock()</br>qrcu_read_unlock()
更新侧原语（同步）|synchronize_rcu()</br>synchronize_net()| | synchronize_sched() | synchronize_rcu()</br>synchronize_net() | synchronize_srcu() |synchronize_qrcu()
更新侧原语（异步/回调）| <span style="background:#ddffdd;">call_rcu()</span> | call_rcu_bh() | | call_rcu() | <span style="background:#ffdddd;">N/A</span> | <span style="background:#ffdddd;">N/A</span>
更新侧原语（等待回调）| rcu_barrier() | | | rcu_barrier() | <span style="background:#ffdddd;">N/A</span> | <span style="background:#ffdddd;">N/A</span>
读侧限制 | 非阻塞    | 不开启中断 | 非阻塞 | 除抢占和锁获取外非阻塞 | 无 synchronize_srcu() | 无 synchronize_qrcu()
读侧开销 | 抢占关/开（非抢占则无）| BH 关闭/开启 | 抢占关/开（非抢占则无）| 简单指令，中断关/开 | 简单指令，抢占关/开 | 原子增加和减少共享变量
异步更新侧开销（例如 `call_rcu()`）| 半微秒 | 半微秒 | | 半微秒 | <span style="background:#ffdddd;">N/A</span> | <span style="background:#ffdddd;">N/A</span>
宽限期延迟 | 10 毫秒 | 10 毫秒 | 10 毫秒 | 10 毫秒 | 10 毫秒 | 无读者时 10 纳秒
非 PREEMPT_RT 实现 | RCU Classic | RCU BH | RCU Classic | N/A | SRCU | N/A
PREEMPT_RT 实现 | N/A | 实时 RCU | 强制在所有 CPU 上调度 | 实时 RCU | SRCU | N/A

**快速自测题 1**：上表中部分单元格为什么标为绿色？

“经典 RCU（RCU Classic）”一列对应最初的 RCU 实现。
它的 RCU 读端临界区由 `rcu_read_lock()` 和 `rcu_read_unlock()` 划定，支持嵌套。
对应的同步更新原语是 `synchronize_rcu()`，以及别名 `synchronize_net()`，作用是等待所有正在执行的 RCU 读端临界区执行完毕；这段等待时间就称为 **宽限期（grace period）**。
异步更新原语 `call_rcu()` 会在后续一个宽限期结束后，以指定参数调用传入的函数。例如 `call_rcu(p,f);` 会在宽限期过后执行 RCU 回调 `f(p)`。
[有些场景下，比如卸载使用了 `call_rcu()` 的内核模块](http://lwn.net/Articles/217484/)，需要等待所有未完成的 RCU 回调执行结束，`rcu_barrier()` 就是用来做这件事的。

“RCU BH”这一列：`rcu_read_lock_bh()` / `rcu_read_unlock_bh()` 划定 RCU 读端临界区，`call_rcu_bh()` 在宽限期结束后触发回调。
注意，RCU BH 没有同步接口 `synchronize_rcu_bh()`，不过有需要的话可以很容易新增该接口。

**快速自测题 2**：混用不同 RCU 原语会发生什么？举个例子：用 `rcu_read_lock()` / `rcu_read_unlock()` 标记读临界区，但是用 `call_rcu_bh()` 提交 RCU 回调？

“RCU Sched”这一列：任何关闭抢占的场景都等价于 RCU 读端临界区，`synchronize_sched()` 等待对应的 RCU 宽限期。
这组 RCU API 是 2.6.12 内核引入的，它把旧的 `synchronize_kernel()` 拆分成现在的 `synchronize_rcu()`（给经典 RCU）和 `synchronize_sched()`（给 RCU Sched）。
RCU Sched 没有异步接口 `call_rcu_sched()`，按需可以添加。

**快速自测题 3**：混用经典 RCU 和 RCU Sched 会怎样？

“实时 RCU（Realtime RCU）”的 API 和经典 RCU 完全一致；唯一区别是它的读端临界区允许被抢占，也可以在获取自旋锁时阻塞。
实时 RCU 的设计在 LWN 文章[《The design of preemptible read-copy-update》](http://lwn.net/Articles/253651/) 中有描述。

**快速自测题 4**：混用实时 RCU 和经典 RCU 会怎样？

“SRCU”是专用 RCU 接口，允许 RCU 读端临界区内任意睡眠，LWN 文章《Sleepable RCU》有介绍。
注意：**不能在 SRCU 读临界区里调用 `synchronize_srcu()`，否则会自死锁**。
SRCU 和前面几种 RCU 实现的差别是：使用者要为每一组独立的 SRCU 场景分配一个 `srcu_struct`。这样设计可以保证一组 SRCU 的读临界区不会阻塞其他无关的 `synchronize_srcu()`。
另外，该变体中，`srcu_read_lock()` 返回一个值，必须传给配对的 `srcu_read_unlock()`。

“QRCU”的 API 结构和 SRCU 一致，但它针对 **无读者时极低延迟的宽限期** 做了优化，见 LWN 文章[《Using Promela and Spin to verify parallel algorithms》](http://lwn.net/Articles/243851/)。
和 SRCU 一样，在临界区内调用 `synchronize_qrcu()` 会造成自死锁，应当避免。
QRCU 至今还没有并入 Linux 主线内核，但值得一提——它是唯一能做到深亚微秒级（boast deep sub-microsecond）宽限期延迟的 RCU 实现。

**快速自测题 5**：为什么 SRCU 和 QRCU 都没有异步版本的 `call_srcu()`、`call_qrcu()`？
**快速自测题 6**：在什么条件下，可以在 SRCU 读临界区内部安全调用 `synchronize_srcu()`？

Linux 内核如今拥有数量相当多的 RCU API 与实现。未来有希望缩减这个数量：证据是，一份内核编译配置下，最多只会启用三套底层实现、对外暴露四组 API（经典 RCU 和实时 RCU 共用同一套API）。但这需要仔细审查与分析，和众多锁 API 的精简工作一样。

## RCU 发布-订阅与版本维护 API
幸运的是，下表中的 RCU 发布订阅、版本维护原语适用于上面介绍的所有 RCU 变体。这种通用性在部分场景下可以复用更多代码，从而避免 API 数量过度膨胀。

分类     | 原语                      | 可用性 | 开销
--------|---------------------------|--------|-------------------------------
链表遍历 | list_for_each_entry_rcu() | 2.5.59 | 简单指令（在 Alpha 上是内存屏障）
链表更新 | list_add_rcu()            | 2.5.44 | 内存屏障
.       | list_add_tail_rcu()       | 2.5.44 | 内存屏障
.       | list_del_rcu()            | 2.5.44 | 简单指令
.       | list_replace_rcu()        | 2.6.9  | 内存屏障
.       | list_splice_init_rcu()    | 2.6.21 | 宽限期延迟
Hlist 遍历 | hlist_for_each_entry_rcu() | 2.6.8  | 简单指令（在 Alpha 上是内存屏障）
Hlist 更新 | hlist_add_after_rcu()      | 2.6.14 | 内存屏障
.          | hlist_add_before_rcu()     | 2.6.14 | 内存屏障
.          | hlist_add_head_rcu()       | 2.5.64 | 内存屏障
.          | hlist_del_rcu()            | 2.5.64 | 简单指令
.          | hlist_replace_rcu()        | 2.6.15 | 内存屏障
指针遍历 | rcu_dereference()             | 2.6.9  | 简单指令（在 Alpha 上是内存屏障）
指针更新 | rcu_assign_pointer()          | 2.6.10 | 内存屏障

第一组接口用于操作 Linux 的 `struct list_head` 链表，也就是循环双向链表。
`list_for_each_entry_rcu()` 原语以类型安全的方式遍历受 RCU 保护的链表，同时保证内存序：链表遍历与新节点插入并发执行时，保证访问顺序正确。
在非 Alpha 平台上，该原语相比普通 `list_for_each_entry()` 几乎没有性能损耗。
`list_add_rcu()`、`list_add_tail_rcu()`、`list_replace_rcu()` 对应普通版本的链表操作，但在弱序架构上会额外引入内存屏障开销。
`list_del_rcu()` 同样对应普通删除接口，有意思的是它反而会略快一点：它只毒化 `prev` 指针，而普通 `list_del()` 需要同时毒化 `prev` 和 `next` 指针。
最后，`list_splice_init_rcu()` 类似普通的链表拼接接口，但会引入一整个宽限期延迟。
设置宽限期的目的：在把源链表从链表头彻底断开之前，等待 RCU 读者完成源链表的遍历；如果不这么做，读者可能永远无法结束遍历。

**快速自测题 7**：为什么 `list_del_rcu()` 不同时毒化 `next` 和 `prev` 指针？

第二组接口用于 Linux 的 `struct hlist_head`，即线性哈希链表。
相比 `struct list_head`，`hlist_head` 的一个优势是链表头只需要单个指针，在大型哈希表中可以节省大量内存。
表中 `hlist` 系列 RCU 原语和普通 hlist 接口的对应关系，和 list_head 系列 RCU 原语保持一致。

最后一组接口直接操作指针，适合用来构建非链表类 RCU 保护数据结构，例如 RCU 保护的数组、树结构。
`rcu_assign_pointer()` 原语保证：在弱序处理器上，指针赋值之前的所有初始化操作，都会排在指针赋值动作之前。
与之对应，`rcu_dereference()` 原语保证：在 Alpha CPU 上，后续解引用该指针的代码，可以看到对应 `rcu_assign_pointer()` 之前的初始化效果；在非 Alpha CPU 上，`rcu_dereference()` 主要起到注释作用，标明此处指针访问受 RCU 保护。

**快速自测题 8**：一般来说，所有交给 `rcu_dereference()` 读取的指针，都必须用 `rcu_assign_pointer()` 更新。这条规则有什么例外？
**快速自测题 9**：这些遍历、更新原语可以搭配任意一类 RCU API 使用，这一点有没有弊端？

## 那么，RCU 的本质究竟是什么？

RCU 的核心，本质上就是一套 API，它支持插入操作的发布与订阅、等待所有 RCU 读者执行完成，以及维护多版本数据。

尽管如此，我们可以在 RCU 之上构建更高层级的同步组件，包括配套文章里提到的读写锁、引用计数、存在性保证机制。
而且我毫不怀疑，Linux 社区会持续发掘 RCU 新颖的用法，就像对待内核里其他各类同步原语一样。

最后，要完整理解 RCU，还需要了解可以用这些 API 实现的全部能力。

## 参考文献
略

## 快速自测题答案
**自测题 1**：上表中部分单元格为什么标为绿色？

答：这些绿色标注的 API（`rcu_read_lock()`、`rcu_read_unlock()`、`call_rcu()`）是 Paul E. McKenney 在 90 年代中期所知的仅有的 Linux RCU 接口。那段时期，他曾误以为自己已经掌握了 RCU 的全部知识。

**自测题 2**：混用不同 RCU 原语会发生什么？举例：用`rcu_read_lock()`/`rcu_read_unlock()`标记 RCU 读端临界区，却调用`call_rcu_bh()`提交 RCU 回调？

答：如果调用 `call_rcu_bh()` 时，没有由 `rcu_read_lock_bh()`/`rcu_read_unlock_bh()` 划定的 RCU 读临界区在运行，RCU 就有权立刻执行回调，这就有可能释放一个仍被 RCU 读临界区正在使用的数据结构！
这不只是理论上的风险：一个长时间运行、由 `rcu_read_lock()`/`rcu_read_unlock()` 保护的读临界区，就会遇到这种错误。
在 -rt 实时内核中该风险消失，此时经典 RCU 与 RCU BH 底层共用同一套实现。

**自测题 3**：混用经典 RCU 与 RCU Sched 会怎样？

答：在非抢占内核或者普通抢占内核上，这种混用能跑通只是巧合——在这类内核配置里，经典 RCU 和 RCU Sched 复用同一套底层实现。
但在打上 -rt 补丁集的 PREEMPT_RT 内核中，这样混用会引发严重问题：实时 RCU 的读临界区允许被抢占，`synchronize_sched()` 有可能在读临界区执行到 `rcu_read_unlock()` 之前就提前返回，进而导致数据结构在读端还在使用时就被释放，大大增加内核崩溃的概率。
==事实上，之所以把经典 RCU 和 RCU Sched 拆分成两套接口，初衷就是为了支持可抢占的 RCU 读临界区。==

**自测题 4**：混用实时 RCU 和经典 RCU 会怎样？

答：这需要你自行修改内核代码才能实现这种混用。当前内核中，启用经典 RCU 的内核无法使用实时 RCU，反之亦然。

**自测题 5**：为什么 SRCU 和 QRCU 都没有异步版本 `call_srcu()`、`call_qrcu()` 接口？

答：如果提供异步接口，单个任务可以注册数量无上限的 SRCU/QRCU 回调，造成内存无节制增长。
而现有的同步接口 `synchronize_srcu()`、`synchronize_qrcu()` 有约束：一个任务必须等待上一次宽限期完成之后，才能发起下一次等待。

**自测题 6**：在什么条件下，可以在 SRCU 读临界区内部安全调用 `synchronize_srcu()`？

答：理论上，你可以在使用 **A 份`srcu_struct`** 的 SRCU 读临界区中，调用针对另一份独立 `srcu_struct`（B）的 `synchronize_srcu()`。
但实际开发中，几乎不建议这么写。
比如下面这段代码，依然会形成死锁：
```c
idx = srcu_read_lock(&ssa);
synchronize_srcu(&ssb);
srcu_read_unlock(&ssa, idx);

/* …… */

idx = srcu_read_lock(&ssb);
synchronize_srcu(&ssa);
srcu_read_unlock(&ssb, idx);
```

**自测题 7**：为什么 `list_del_rcu()` 不会同时毒化 `next`、`prev` 两个指针？

答：如果毒化 `next` 指针，会干扰正在遍历链表的 RCU 读者，读者必须依靠 `next` 指针向后遍历。
而 RCU 读者规范上不允许使用 `prev` 指针，所以 `prev` 指针可以安全毒化。

**自测题 8**：一般来说，所有由 `rcu_dereference()` 读取的指针，都必须用 `rcu_assign_pointer()` 更新。这条规则有什么例外？

答：一种典型场景：一个多节点链表结构在初始化阶段，其他 CPU 还无法访问它，此时可以先完成内部所有指针赋值，最后只调用一次 `rcu_assign_pointer()` 把全局指针指向这个完整建好的结构。
初始化阶段内部的指针赋值不需要 `rcu_assign_pointer()`；但一旦这个结构对全局可见之后，后续任何指针修改都必须使用 `rcu_assign_pointer()`。
不过，除非这段初始化代码是极高频的热点路径，否则即便理论上不需要，也建议统一使用`rcu_assign_pointer()`。
一旦后续代码发生微小改动，很容易打破“私有初始化、无并发访问”这个前提假设。

**自测题 9**：这些遍历、更新原语可以搭配任意一类 RCU API 使用，这一点有没有弊端？

答：自动化代码检查工具（例如 sparse），甚至开发者自己，有时很难判断一段 RCU 遍历代码对应的是哪一类 RCU 读临界区。
举个例子：
```c
rcu_read_lock();
preempt_disable();
p = rcu_dereference(global_pointer);

/* …… */

preempt_enable();
rcu_read_unlock();
```
这里的 `rcu_dereference()` 是处在经典 RCU 临界区，还是 RCU Sched 临界区？你要怎么判断？

# What is RCU?

## 1. RCU 概览
RCU 背后的基本思想是把更新拆成“移除”与“回收”两个阶段。
* **移除阶段** 把数据结构中指向某个数据项的引用去掉（也可以换成指向该数据项新版本的引用），这一阶段可以与读者并发执行。之所以能与读者并发执行，是因为现代 CPU 的语义保证读者要么看到数据结构的旧版本、要么看到新版本，而不会看到一个只更新了一半的引用。
* **回收阶段** 则负责回收（例如释放）在移除阶段已从数据结构中摘下的数据项。由于回收数据项会干扰仍在引用这些数据项的读者，回收阶段必须等到读者不再持有这些数据项的引用之后才能开始。

把更新拆成移除与回收两个阶段，使得更新者可以立即执行移除阶段，而把回收阶段推迟到移除阶段期间所有活跃读者都已完成之后 —— 既可以阻塞等待它们结束，也可以注册一个在它们结束后才被调用的回调。
这里只需考虑移除阶段期间活跃的读者，因为在移除阶段开始 **之后** 才进入的读者不可能获得被移除数据项的引用，因而也不会被回收阶段干扰。

因此，典型的 RCU 更新序列大致如下：
a. 摘除指向某个数据结构的指针，使后续读者无法再获得对它的引用。
b. 等待此前所有读者都完成各自的 RCU 读侧临界区。
c. 此时已不可能有任何读者持有该数据结构的引用，因此它可以被安全地回收（例如 `kfree()`）。

上面第 (b) 步正是 RCU 延迟销毁的关键思想。能够等到所有读者都结束，使 RCU 读者在很多场合可以使用轻得多的同步手段，有些场合甚至完全不需要任何同步。
相比之下，在更传统的基于锁的方案中，读者必须使用重量级同步来防止更新者把数据结构从它们脚下删除掉。这是因为基于锁的更新者通常原地修改数据项，因而必须排斥读者。
而 RCU 更新者通常利用这样一个事实：**在现代 CPU 上对单个对齐指针的写入是原子的，于是可以在链式结构中原子地插入、移除与替换数据项，而不干扰读者**。
并发的 RCU 读者则可以继续访问旧版本，从而省去原子操作、内存屏障以及通信缓存缺失（cache miss）的开销——这些开销在当今的 SMP 计算机系统上即便没有锁竞争也相当昂贵。

在上面这个三步过程中，更新者同时承担了移除与回收两步，但其实常常让另一个线程专门负责回收更合适，Linux 内核的目录项缓存（dcache）正是如此。
即便由同一个线程完成更新步骤（上述第 (a) 步）与回收步骤（第 (c) 步），把它们分开看待通常也更有帮助。
例如，RCU 读者与更新者根本不需要通信，但 RCU 在读者与回收者之间提供了隐式的、低开销的通信，也就是上述第 (b) 步。
那么问题来了：既然读者不做任何同步操作，回收者究竟怎么知道某个读者已经结束？继续往下读，你会看到 RCU 的 API 如何让这件事变得简单。

## 2. RCU 的核心 API 是什么？

RCU 的核心 API 相当少：
```c
a. rcu_read_lock()
b. rcu_read_unlock()
c. synchronize_rcu() / call_rcu()
d. rcu_assign_pointer()
e. rcu_dereference()
```
RCU API 还有许多其他成员，但其余的都可用这五个来表达 —— 尽管多数实现反过来用 `call_rcu()` 回调 API 来实现 `synchronize_rcu()`。

下面先描述这五个核心 RCU API，其余 18 个稍后列出。更多信息可参阅内核的 docbook 文档，或直接查看函数头部注释。

### `rcu_read_lock()`
```c
void rcu_read_lock(void);
```
读者用它告知回收者：自己正在进入 RCU 读侧临界区。
在 RCU 读侧临界区内阻塞是非法的，不过以 `CONFIG_PREEMPT_RCU` 构建的内核可以抢占 RCU 读侧临界区。
在 RCU 读侧临界区期间访问的任何受 RCU 保护的数据结构，都保证在整个临界区期间不会被回收。
若需要对数据结构保持更长期的引用，可以配合 RCU 使用引用计数。

### `rcu_read_unlock()`
```c
void rcu_read_unlock(void);
```
读者用它告知回收者：自己正在退出 RCU 读侧临界区。
请注意 RCU 读侧临界区可以嵌套，也可以相互重叠。

### `synchronize_rcu()`
```c
void synchronize_rcu(void);
```
它标记 **更新者代码的结束** 与 **回收者代码的开始**：阻塞到所有 CPU 上此前已存在的 RCU 读侧临界区全部完成为止。

注意 `synchronize_rcu()` 不必等待其后才开始的 RCU 读侧临界区完成。例如，考虑下面的事件序列：
```c
        CPU 0                  CPU 1                 CPU 2
     --------------- ------------------------- ---------------
1.   rcu_read_lock()
2.                     enters synchronize_rcu()
3.                                                rcu_read_lock()
4.   rcu_read_unlock()
5.                      exits synchronize_rcu()
6.                                               rcu_read_unlock()
```
再强调一次（To reiterate）：`synchronize_rcu()` 只等待 **正在进行中** 的 RCU 读侧临界区完成，不一定要等待在 `synchronize_rcu()` 被调用 **之后** 才开始的临界区。

当然，`synchronize_rcu()` 也不一定会在最后一个既存的 RCU 读侧临界区完成后立即返回。一方面可能存在调度延迟；另一方面，许多 RCU 实现为了提高效率而批量处理请求，这会进一步推迟 `synchronize_rcu()`。

由于 `synchronize_rcu()` 正是那个必须弄清读者何时结束的 API，它的实现是 RCU 的关键。
要让 RCU 除极端读密集的场景之外都能用得上，`synchronize_rcu()` 的开销也必须相当小。

`call_rcu()` API 是 `synchronize_rcu()` 的回调形式，将在后面详细描述。它不阻塞，而是注册一个函数及其参数，在所有进行中的 RCU 读侧临界区完成后调用。在不容许阻塞的场合，这种回调形式尤其有用。

### `rcu_assign_pointer()`
```c
typeof(p) rcu_assign_pointer(p, typeof(p) v);
```
没错，`rcu_assign_pointer()` 是以宏实现的 —— 尽管若能以这种方式声明函数会很酷。（编译器专家们想必不会同意。）

更新者用它给受 RCU 保护的指针赋新值，以便把值的变化安全地从更新者传递给读者。
该函数返回新值，并执行特定 CPU 体系结构所需的任何内存屏障指令。

或许更重要的是，它起到标记哪些指针受 RCU 保护的作用。不过，`rcu_assign_pointer()` 最常被间接使用，即通过 `_rcu` 系列链表操作原语（如 `list_add_rcu()`）来使用。

### `rcu_dereference()`
```c
typeof(p) rcu_dereference(p);
```
与 `rcu_assign_pointer()` 一样，`rcu_dereference()` 必须实现为宏。

读者用它取出受 RCU 保护的指针，返回的值随后可以被安全解引用。
注意 `rcu_dereference()` 并不真的解引用该指针，而是为该指针的后续解引用提供保护；它还会执行特定 CPU 体系结构所需的任何内存屏障指令。

目前只有 Alpha 需要在 `rcu_dereference()` 内使用内存屏障 —— 在其他 CPU 上它编译后什么也不剩，连一条编译器指示都没有。
常见的编码实践是用 `rcu_dereference()` 把受 RCU 保护的指针复制到局部变量，然后解引用该局部变量，例如：
```c
p = rcu_dereference(head.next);
return p->data;
```
不过在这种情况下，也可以干脆合并为一条语句：
```c
return rcu_dereference(head.next)->data;
```
如果你要从受 RCU 保护的结构中取多个字段，那当然还是用局部变量更好。反复调用 `rcu_dereference()` 不但难看，在 Alpha CPU 上还会带来不必要的开销。

注意 `rcu_dereference()` 返回的值只在其所在的 RCU 读侧临界区内有效。例如，下面这样是不合法的：
```c
rcu_read_lock();
p = rcu_dereference(head.next);
rcu_read_unlock();
x = p->address;
rcu_read_lock();
y = p->data;
rcu_read_unlock();
```
把一个引用从一个 RCU 读侧临界区保持到另一个临界区，和一个引用从一个基于锁的临界区保持到另一个临界区一样非法！同样，在获取引用的临界区之外使用该引用，也与普通加锁的情形一样非法。

与 `rcu_assign_pointer()` 一样，`rcu_dereference()` 的一个重要功能是标记哪些指针受 RCU 保护。而且同样地，`rcu_dereference()` 通常是通过 `_rcu` 链表操作原语（如 `list_for_each_entry_rcu()`）间接使用的。

### RCU 原语之间的关系
下图展示各个 API 如何在读者、更新者与回收者之间传递信息。

![RCU 原语之间的关系](pic/wircu_readupdreclaim.jpg)

图 1：RCU 原语之间的关系

RCU 基础设施观察 `rcu_read_lock()`、`rcu_read_unlock()`、`synchronize_kernel()` 与 `call_rcu()` 调用之间的时间顺序，从而判定

1. `synchronize_kernel()` 调用何时可以返回给调用者，
2. `call_rcu()` 回调何时可以被调用。

高效的 RCU 实现大量使用批处理，把开销分摊到相应 API 的众多使用上。

Linux 内核中的 RCU 机制不止三种；上图展示的是第一种，也是使用最广泛的一种。`rcu_dereference()` 与 `rcu_assign_pointer()` 原语在全部四种机制中都会用到，但不同机制使用的延迟与保护原语不同：

表 1：四种 RCU 机制的延迟与保护原语

.  | Defer               | Protect
---|---------------------|-----------------------------------------
a. | synchronize_rcu()   | rcu_read_lock() / rcu_read_unlock()
.  | call_rcu()          | .
b. | call_rcu_bh()       | rcu_read_lock_bh() / rcu_read_unlock_bh()
c. | synchronize_sched() | preempt_disable() / preempt_enable()
.  | .                   | local_irq_save() / local_irq_restore()
.  | .                   | hardirq enter / hardirq exit
.  | .                   | NMI enter / NMI exit
d. | synchronize_srcu()  | srcu_read_lock() / srcu_read_unlock()

这四种机制的用途分别是：
a. 用于普通数据结构的 RCU。
b. 用于可能遭受远程拒绝服务攻击的网络数据结构的 RCU。
c. 用于调度器与中断/NMI 处理任务的 RCU。
d. 允许读者阻塞的 RCU 变体。

同样，大多数用途都是 (a)。(b)、(c)、(d) 在某些特殊场合很重要，但相对少见。

## 3. 核心 RCU API 有哪些示例用法？
本节展示核心 RCU API 的一个简单用法：保护一个指向动态分配结构的全局指针。更典型的 RCU 用法见 listRCU.txt、arrayRCU.txt 与 NMI-RCU.txt。

```c
struct foo {
        int a;
        char b;
        long c;
};
DEFINE_SPINLOCK(foo_mutex);

struct foo *gbl_foo;

/*
 * 新建一个 struct foo，内容与 gbl_foo 当前所指的相同，只把字段 "a" 换成 "new_a"。
 * 让 gbl_foo 指向新结构，并在一个宽限期之后释放旧结构。
 *
 * 用 rcu_assign_pointer() 确保并发读者看到的是已初始化好的新结构。
 *
 * 用 synchronize_rcu() 确保可能持有旧结构引用的读者在旧结构被释放之前全部完成。
 */
void foo_update_a(int new_a)
{
        struct foo *new_fp;
        struct foo *old_fp;

        new_fp = kmalloc(sizeof(*fp), GFP_KERNEL);
        spin_lock(&foo_mutex);
        old_fp = gbl_foo;
        *new_fp = *old_fp;
        new_fp->a = new_a;
        rcu_assign_pointer(gbl_foo, new_fp);
        spin_unlock(&foo_mutex);
        synchronize_rcu();
        kfree(old_fp);
}

/*
 * 返回当前 gbl_foo 结构中字段 "a" 的值。用 rcu_read_lock() 与 rcu_read_unlock()
 * 确保结构不会被人从我们脚下删掉，并用 rcu_dereference() 确保我们看到的是已初始化好的结构
 *（这对 DEC Alpha 以及阅读代码的人都重要）。
 */
int foo_get_a(void)
{
        int retval;

        rcu_read_lock();
        retval = rcu_dereference(gbl_foo)->a;
        rcu_read_unlock();
        return retval;
}
```

归纳如下：
* 用 `rcu_read_lock()` 与 `rcu_read_unlock()` 守护 RCU 读侧临界区。
* 在 RCU 读侧临界区内，用 `rcu_dereference()` 解引用受 RCU 保护的指针。
* 用某种可靠的方案（如锁或信号量）防止并发更新互相干扰。
* 用 `rcu_assign_pointer()` 更新受 RCU 保护的指针。
  * 该原语保护的是并发读者不受更新者干扰，而不是并发更新之间互不干扰！
  * 因此你仍然需要加锁（或类似手段）来防止并发的 `rcu_assign_pointer()` 原语互相干扰。
* 从受 RCU 保护的数据结构中移出某个数据项之后、回收/释放该数据项之前，使用 `synchronize_rcu()`，以等待所有可能引用该数据项的 RCU 读侧临界区完成。

关于使用 RCU 时还应遵循的其他规则，见 Linux 内核源码树 Documentation/RCU 目录下的 checklist.txt。

## 4. 如果我的更新线程不能阻塞怎么办？
在上面的例子中，`foo_update_a()` 会阻塞直到一个宽限期过去。这很简单，但在某些场合你等不起这么久 —— 可能还有别的高优先级工作要做。

这种情况下应当用 `call_rcu()` 而不是 `synchronize_rcu()`。`call_rcu()` API 如下：
```c
void call_rcu(struct rcu_head * head,
              void (*func)(struct rcu_head *head));
```
该函数在一个宽限期过去后调用 `func(head)`。
这个调用可能发生在 softirq 上下文，也可能发生在进程上下文，因此该函数不允许阻塞。
`foo` 结构需要加上一个 `rcu_head` 结构，例如：
```c
struct foo {
        int a;
        char b;
        long c;
        struct rcu_head rcu;
};
```
于是 `foo_update_a()` 函数可以写成：
```c
/*
 * 新建一个 struct foo，内容与 gbl_foo 当前所指的相同，只把字段 "a" 换成 "new_a"。
 * 让 gbl_foo 指向新结构，并在一个宽限期之后释放旧结构。
 *
 * 用 rcu_assign_pointer() 确保并发读者看到的是已初始化好的新结构。
 *
 * 用 call_rcu() 确保可能持有旧结构引用的读者在旧结构被释放之前全部完成。
 */
void foo_update_a(int new_a)
{
        struct foo *new_fp;
        struct foo *old_fp;

        new_fp = kmalloc(sizeof(*fp), GFP_KERNEL);
        spin_lock(&foo_mutex);
        old_fp = gbl_foo;
        *new_fp = *old_fp;
        new_fp->a = new_a;
        rcu_assign_pointer(gbl_foo, new_fp);
        spin_unlock(&foo_mutex);
        call_rcu(&old_fp->rcu, foo_reclaim);
}
```
`foo_reclaim()` 函数可能如下：
```c
void foo_reclaim(struct rcu_head *rp)
{
        struct foo *fp = container_of(rp, struct foo, rcu);

        kfree(fp);
}
```
`container_of()` 原语是一个宏：给定指向结构内部的指针、结构的类型以及被指向的字段，返回指向该结构起始处的指针。

使用 `call_rcu()` 使 `foo_update_a()` 的调用者可以立即取回控制权，而不必再操心刚更新元素旧版本的后续处理。
它也清楚地体现了 RCU 对更新者（即 `foo_update_a()`）与回收者（即 `foo_reclaim()`）的区分。

给出的建议与上一节相同，只是现在用 `call_rcu()` 而不是 `synchronize_rcu()`：
* 从受 RCU 保护的数据结构中移出某个数据项之后，使用 `call_rcu()` 注册一个回调函数，该函数将在所有可能引用该数据项的 RCU 读侧临界区完成后被调用。

同样，关于使用 RCU 的其他规则见 checklist.txt。

## 5. RCU 有哪些简单实现？
RCU 的一大好处是它有一些极其简单的“玩具”实现，是理解 Linux 内核中生产级实现的第一步。
本节给出两个这样的“玩具”RCU 实现：一个用熟悉的加锁原语实现，另一个更接近“经典”RCU。
两者都过于简单，功能与性能都不足以实用，但它们有助于体会 RCU 的工作原理。
生产级实现见 kernel/rcupdate.c，另见：
* http://www.rdrop.com/users/paulmck/RCU

那里有描述 Linux 内核 RCU 实现的论文。
OLS'01 与 OLS'02 的两篇论文是很好的入门材料，而那篇学位论文对当前实现给出了更多细节。

### 5A. “玩具”实现之一：加锁
本节给出一个基于熟悉的加锁原语的“玩具”RCU 实现。
它的开销使它无法用于实际系统，可扩展性差也是原因之一。
它同样不适用于实时场景，因为它会让调度延迟从一个读侧临界区“渗透”到另一个读侧临界区。

不过，它大概是最容易让人产生共鸣的实现，因此是一个很好的起点。

它极其简单：
```c
static DEFINE_RWLOCK(rcu_gp_mutex);

void rcu_read_lock(void)
{
        read_lock(&rcu_gp_mutex);
}

void rcu_read_unlock(void)
{
        read_unlock(&rcu_gp_mutex);
}

void synchronize_rcu(void)
{
        write_lock(&rcu_gp_mutex);
        write_unlock(&rcu_gp_mutex);
}
```
[你可以忽略 `rcu_assign_pointer()` 与 `rcu_dereference()` 而不会有太多损失。但既然提到了，还是列在这里。无论如何，提交使用 RCU 的补丁时千万不要忘记它们！]
```c
#define rcu_assign_pointer(p, v)     ({ \
                                      smp_wmb(); \
                                      (p) = (v); \
                                 })

#define rcu_dereference(p)           ({ \
                                      typeof(p) _________p1 = p; \
                                      smp_read_barrier_depends(); \
                                      (_________p1); \
                                 })
```
`rcu_read_lock()` 与 `rcu_read_unlock()` 原语以读方式获取并释放一个全局读写锁；`synchronize_rcu()` 原语则以写方式获取同一把锁，然后立即释放。
这意味着一旦 `synchronize_rcu()` 返回，在 `synchronize_rcu()` 被调用之前就已开始的所有 RCU 读侧临界区保证都已完成——否则 `synchronize_rcu()` 根本不可能以写方式获取该锁。

`rcu_read_lock()` 可以嵌套，因为读写锁可以被递归获取。
还要注意 `rcu_read_lock()` 不会死锁（这是 RCU 的一个重要属性）。原因在于：唯一能让 `rcu_read_lock()` 阻塞的是 `synchronize_rcu()`；而 `synchronize_rcu()` 在持有 `rcu_gp_mutex` 期间不再获取任何锁，因此不存在死锁环路。

**快速测验 #1**：为什么这个论证很天真？在真实世界的 Linux 内核中使用该算法，死锁会如何发生？又该如何避免？

### 5B. “玩具”示例之二：经典 RCU
本节给出一个基于“经典 RCU”的“玩具”RCU 实现。
它在性能上（仅对更新而言）以及热插拔 CPU、可在 `CONFIG_PREEMPT` 内核中运行等特性上都有欠缺。
`rcu_dereference()` 与 `rcu_assign_pointer()` 的定义与上一节相同，故此处省略。
```c
void rcu_read_lock(void) { }

void rcu_read_unlock(void) { }

void synchronize_rcu(void)
{
        int cpu;

        for_each_cpu(cpu)
                run_on(cpu);
}
```
注意 `rcu_read_lock()` 与 `rcu_read_unlock()` 什么也不做。
这正是经典 RCU 在非抢占内核中的巨大优势：读侧开销恰好为零——至少在非 Alpha CPU 上是如此。
而且 `rcu_read_lock()` 绝对不可能卷入死锁环路！

`synchronize_rcu()` 的实现只是把自己依次调度到每个 CPU 上。`run_on()` 原语可以直接用 `sched_setaffinity()` 实现。
当然，一个不那么“玩具”的实现会在完成后恢复亲和性，而不是把所有任务都丢在最后一个 CPU 上运行；但我既然说了是“玩具”，那就是 *玩具*！

那么，这玩意儿到底（heck）是怎么工作的？？？

请记住，在 RCU 读侧临界区内阻塞是非法的。
因此，如果某个 CPU 执行了一次上下文切换，我们就知道它必然已完成此前所有的 RCU 读侧临界区。
一旦 **所有** CPU 都执行过一次上下文切换，此前 **所有的** RCU 读侧临界区就都已完成。

于是，假设我们把某个数据项从其结构中移出，然后调用 `synchronize_rcu()`。
一旦 `synchronize_rcu()` 返回，就保证了不再有任何 RCU 读侧临界区持有该数据项的引用，因此我们可以安全地回收它。

**快速测验 #2**：举一个经典 RCU 读侧开销为 **负** 的例子。

**快速测验 #3**：既然在 RCU 读侧临界区内阻塞是非法的，那么在 `PREEMPT_RT` 中普通自旋锁都可以阻塞，你又该怎么办？？？

## 6. 与读写锁的类比
尽管 RCU 可以有多种用法，但一种非常常见的用法与读写锁类似。下面这个 unified diff 展示了 RCU 与读写锁之间的关系有多么密切。
```diff
@@ -13,15 +14,15 @@
        struct list_head *lp;
        struct el *p;

-        read_lock();
-        list_for_each_entry(p, head, lp) {
+        rcu_read_lock();
+        list_for_each_entry_rcu(p, head, lp) {
                 if (p->key == key) {
                          *result = p->data;
-                         read_unlock();
+                         rcu_read_unlock();
                          return 1;
                 }
         }
-        read_unlock();
+        rcu_read_unlock();
         return 0;
    }

@@ -29,15 +30,16 @@
 {
        struct el *p;

-        write_lock(&listmutex);
+        spin_lock(&listmutex);
         list_for_each_entry(p, head, lp) {
                 if (p->key == key) {
-                        list_del(&p->list);
-                        write_unlock(&listmutex);
+                        list_del_rcu(&p->list);
+                        spin_unlock(&listmutex);
+                        synchronize_rcu();
                         kfree(p);
                         return 1;
                 }
         }
-        write_unlock(&listmutex);
+        spin_unlock(&listmutex);
         return 0;
    }
```
或者，对于喜欢并 side-by-side list 写法的人：
```c
 1 struct el {                                1 struct el {
 2   struct list_head list;                   2   struct list_head list;
 3   long key;                                3   long key;
 4   spinlock_t mutex;                        4   spinlock_t mutex;
 5   int data;                                5   int data;
 6   /* 其他数据字段 */                        6   /* 其他数据字段 */
 7 };                                         7 };
 8 spinlock_t listmutex;                      8 spinlock_t listmutex;
 9 struct el head;                            9 struct el head;

 1 int search(long key, int *result)          1 int search(long key, int *result)
 2 {                                          2 {
 3   struct list_head *lp;                    3   struct list_head *lp;
 4   struct el *p;                            4   struct el *p;
 5                                            5
 6   read_lock();                             6   rcu_read_lock();
 7   list_for_each_entry(p, head, lp) {       7   list_for_each_entry_rcu(p, head, lp) {
 8     if (p->key == key) {                   8     if (p->key == key) {
 9       *result = p->data;                   9       *result = p->data;
10       read_unlock();                      10       rcu_read_unlock();
11       return 1;                           11       return 1;
12     }                                     12     }
13   }                                       13   }
14   read_unlock();                          14   rcu_read_unlock();
15   return 0;                               15   return 0;
16 }                                         16 }

 1 int delete(long key)                       1 int delete(long key)
 2 {                                          2 {
 3   struct el *p;                            3   struct el *p;
 4                                            4
 5   write_lock(&listmutex);                  5   spin_lock(&listmutex);
 6   list_for_each_entry(p, head, lp) {       6   list_for_each_entry(p, head, lp) {
 7     if (p->key == key) {                   7     if (p->key == key) {
 8       list_del(&p->list);                  8       list_del_rcu(&p->list);
 9       write_unlock(&listmutex);            9       spin_unlock(&listmutex);
                                             10       synchronize_rcu();
10       kfree(p);                           11       kfree(p);
11       return 1;                           12       return 1;
12     }                                     13     }
13   }                                       14   }
14   write_unlock(&listmutex);               15   spin_unlock(&listmutex);
15   return 0;                               16   return 0;
16 }                                         17 }
```
无论哪种写法，差异都很小：读侧加锁变成 `rcu_read_lock()` 与 `rcu_read_unlock()`，更新侧加锁从读写锁变成简单的自旋锁，并且在 `kfree()` 之前多了一句 `synchronize_rcu()`。

不过这里有一个潜在的陷阱：现在读侧与更新侧的临界区可以并发运行。很多情况下这不是问题，但仍需仔细检查。例如，若多个彼此独立的链表更新必须被当作一次原子更新来看待，那么改用 RCU 就需要特别小心。

另外，`synchronize_rcu()` 的存在意味着 RCU 版本的 `delete()` 现在可能会阻塞。如果这是个问题，可以用一种永不阻塞的回调机制，即 `call_rcu()`，来替代 `synchronize_rcu()`。

## 7. 类比引用计数
读写锁类比（上一节已说明）并不总是理解 RCU 用法的最佳方式。另一个很有帮助的思路是：把 RCU 看作是给所有受 RCU 保护的对象施加了一套有效的引用计数机制。

引用计数一般不会阻止被引用对象内部的值发生变化，但它可以阻止 **对象类型发生改变** —— 尤其避免一种根本性的类型变更：对象内存被释放、之后重新分配给别的用途。
一旦拿到对象的类型安全引用，还需要其他机制保证访问对象数据的一致性。
可以使用自旋锁；而在 RCU 场景下，典型做法是：读侧使用具备 SMP 语义的操作（例如 `smp_load_acquire()`），更新侧使用原子读改写原语，以此提供所需的内存序。
RCU 提供了一系列封装好的辅助函数，内置了这些必要操作与内存序，上一节用到的宏 `list_for_each_entry_rcu()` 就是其中一例。

从更聚焦的角度看引用计数行为：在 `rcu_read_lock()` 与 `rcu_read_unlock()` 之间，通过 `rcu_dereference()` 对标记为 `__rcu` 的指针拿到的引用，等价于 **临时抬高了该对象的引用计数**。
这个作用是防止对象发生类型变更。
具体含义取决于该类型对象的常规约定，通常包括：自旋锁仍然可以安全上锁、普通引用计数器可以安全操作、`__rcu` 指针可以安全解引用。

持有 RCU 引用时，可以对对象执行的常见操作包括：
- 拷贝对象类型本身保证稳定的数据；
- 调用 `kref_get_unless_zero()` 或类似接口获取更长期的引用（该操作当然有可能失败）；
- 获取对象内的自旋锁，校验对象是否仍是预期对象，校验通过后即可自由操作。

RCU 只保护 **类型不变、不保证对象身份不变** 这一点，在使用 `SLAB_TYPESAFE_BY_RCU` 标志创建的 slab 缓存对象上体现得尤为明显。
RCU 有可能返回一个来自该缓存的对象引用：这块内存已经被并发释放，又重新分配给了另一个对象，**二者类型相同，但并不是同一个实例**。
这种场景下 RCU 甚至无法保护对象身份不变，仅仅保证类型不变。
也就是说，拿到的对象未必是你想要的那一个，但它一定是一个可以安全获取引用（之后再尝试拿自旋锁）的合法对象，后续代码再校验身份是否符合预期。

有人会想跳过获取引用，直接拿自旋锁，但这是不安全的：`SLAB_TYPESAFE_BY_RCU` 的对象里的自旋锁，**每次调用 `kmem_cache_alloc()` 之后都必须重新初始化**。
跳过引用直接上锁会引发问题。因此使用 `SLAB_TYPESAFE_BY_RCU` 时，必须正确使用引用计数器。
如果使用 `refcount_t`，要使用专用 API：`refcount_{add|inc}_not_zero_acquire()` 和 `refcount_set_release()`，在校验对象身份、初始化新分配对象时保证正确的内存序。
`refcount_{add|inc}_not_zero_acquire()` 自带获取屏障，**保证身份校验发生在引用计数增加之后**；
`refcount_set_release()` 需要在新对象完全初始化完毕之后调用，自带释放屏障，**保证对象新的初始化内容先对外可见**，其他用户才能成功拿到引用。
一旦调用 `refcount_set_release()`，就代表该对象已经对其他任务可见。
（如果开发者选择在 `kmem_cache` 构造函数里初始化锁，也可以使用锁机制，包括对缓存友好的顺序锁。）

传统引用计数（比如 Linux 内核的 kref）的特点：当对象最后一个引用被释放时，会执行收尾逻辑；对 kre f来说，收尾函数就是传给 `kref_put()` 的回调。
而 RCU 场景下，收尾逻辑不能立刻执行：必须等所有指向该对象的 `__rcu` 指针都更新完毕，并且 **度过一个宽限期（grace period）**。
所有全局可见、指向该对象的指针都被视作潜在的引用；只有全部指针都修改完成之后，才一般通过 `call_rcu()` 调度收尾回调。

我们可以根据保护对象的粒度，来选择这两种类比（RCU 类比读写锁 / RCU 类比引用计数）：
- 读写锁类比，适合看待链表这类由多个成员组成的复合对象：描述在增删链表节点时，RCU 如何提升并发能力；
- 引用计数类比，适合看待单个对象：描述在它所属的整体结构里，如何安全访问这个对象本身。

## 8. RCU API 全列表
RCU 的 API 在 Linux 内核源码中以 docbook 格式的头注释加以说明，但列出一份完整清单仍有帮助，因为 docbook 里似乎没办法对它们分类。下面按类别列出。

RCU 链表遍历：
```c
list_entry_rcu
list_entry_lockless
list_first_entry_rcu
list_first_or_null_rcu
list_tail_rcu
list_next_rcu
list_next_or_null_rcu
list_for_each_entry_rcu
list_for_each_entry_continue_rcu
list_for_each_entry_from_rcu
list_for_each_entry_lockless
hlist_first_rcu
hlist_next_rcu
hlist_pprev_rcu
hlist_for_each_entry_rcu
hlist_for_each_entry_rcu_notrace
hlist_for_each_entry_rcu_bh
hlist_for_each_entry_from_rcu
hlist_for_each_entry_continue_rcu
hlist_for_each_entry_continue_rcu_bh
hlist_nulls_first_rcu
hlist_nulls_next_rcu
hlist_nulls_for_each_entry_rcu
hlist_nulls_for_each_entry_safe
hlist_bl_first_rcu
hlist_bl_for_each_entry_rcu
```

RCU 指针/链表更新：
```c
rcu_assign_pointer
rcu_replace_pointer
INIT_LIST_HEAD_RCU
list_add_rcu
list_add_tail_rcu
list_del_rcu
list_replace_rcu
list_splice_init_rcu
list_splice_tail_init_rcu
hlist_add_behind_rcu
hlist_add_before_rcu
hlist_add_head_rcu
hlist_add_tail_rcu
hlist_del_rcu
hlist_del_init_rcu
hlist_replace_rcu
hlist_nulls_del_init_rcu
hlist_nulls_del_rcu
hlist_nulls_add_head_rcu
hlist_nulls_add_tail_rcu
hlist_nulls_add_fake
hlists_swap_heads_rcu
hlist_bl_add_head_rcu
hlist_bl_del_rcu
hlist_bl_set_first_rcu
```
RCU：
```c
Critical sections               Grace period            Barrier

rcu_read_lock                   synchronize_net         rcu_barrier
rcu_read_unlock                 synchronize_rcu
guard(rcu)()                    synchronize_rcu_expedited
scoped_guard(rcu)               synchronize_rcu_mult
rcu_dereference                 call_rcu
rcu_dereference_check           call_rcu_hurry
rcu_dereference_protected       kfree_rcu
rcu_read_lock_held              kvfree_rcu
rcu_read_lock_any_held          kfree_rcu_mightsleep
rcu_pointer_handoff             cond_synchronize_rcu
unrcu_pointer                   cond_synchronize_rcu_full
                                cond_synchronize_rcu_expedited
                                cond_synchronize_rcu_expedited_full
                                get_completed_synchronize_rcu
                                get_completed_synchronize_rcu_full
                                get_state_synchronize_rcu
                                get_state_synchronize_rcu_full
                                poll_state_synchronize_rcu
                                poll_state_synchronize_rcu_full
                                same_state_synchronize_rcu
                                same_state_synchronize_rcu_full
                                start_poll_synchronize_rcu
                                start_poll_synchronize_rcu_full
                                start_poll_synchronize_rcu_expedited
                                start_poll_synchronize_rcu_expedited_full
```
bh:
```c
Critical sections       Grace period            Barrier

rcu_read_lock_bh        [Same as RCU]           [Same as RCU]
rcu_read_unlock_bh
[local_bh_disable]
[and friends]
rcu_dereference_bh
rcu_dereference_bh_check
rcu_dereference_bh_protected
rcu_read_lock_bh_held
```
sched:
```c
Critical sections       Grace period            Barrier

rcu_read_lock_sched     [Same as RCU]           [Same as RCU]
rcu_read_unlock_sched
[preempt_disable]
[and friends]
rcu_read_lock_sched_notrace
rcu_read_unlock_sched_notrace
rcu_dereference_sched
rcu_dereference_sched_check
rcu_dereference_sched_protected
rcu_read_lock_sched_held
```
RCU：初始化/清理/排序：
```c
RCU_INIT_POINTER
RCU_INITIALIZER
RCU_POINTER_INITIALIZER
init_rcu_head
destroy_rcu_head
init_rcu_head_on_stack
destroy_rcu_head_on_stack
SLAB_TYPESAFE_BY_RCU
```
RCU：静默状态和控制
```c
cond_resched_tasks_rcu_qs
rcu_all_qs
rcu_softirq_qs_periodic
rcu_end_inkernel_boot
rcu_expedite_gp
rcu_gp_is_expedited
rcu_unexpedite_gp
rcu_cpu_stall_reset
rcu_head_after_call_rcu
rcu_is_watching
```
RCU 同步原语：
```c
rcu_sync_is_idle
rcu_sync_init
rcu_sync_enter
rcu_sync_exit
rcu_sync_dtor
```
RCU-Tasks：
```c
Critical sections       Grace period                    Barrier

N/A                     call_rcu_tasks                  rcu_barrier_tasks
                        synchronize_rcu_tasks
```
RCU-Tasks-Rude:
```c
Critical sections       Grace period                    Barrier

N/A                     synchronize_rcu_tasks_rude      rcu_barrier_tasks_rude
                        call_rcu_tasks_rude
```
RCU-Tasks-Trace:
```c
Critical sections       Grace period                    Barrier

rcu_read_lock_trace     call_rcu_tasks_trace            rcu_barrier_tasks_trace
rcu_read_unlock_trace   synchronize_rcu_tasks_trace
guard(rcu_tasks_trace)()
scoped_guard(rcu_tasks_trace)
```
SRCU 链表遍历：
```c
list_for_each_entry_srcu hlist_for_each_entry_srcu
```
SRCU:
```c
Critical sections               Grace period            Barrier

srcu_read_lock                  call_srcu               srcu_barrier
srcu_read_unlock                synchronize_srcu
srcu_read_lock_fast             synchronize_srcu_expedited
srcu_read_unlock_fast           get_state_synchronize_srcu
srcu_read_lock_nmisafe          start_poll_synchronize_srcu
srcu_read_unlock_nmisafe        start_poll_synchronize_srcu_expedited
srcu_read_lock_notrace          poll_state_synchronize_srcu
srcu_read_unlock_notrace
srcu_down_read
srcu_up_read
srcu_down_read_fast
srcu_up_read_fast
guard(srcu)()
scoped_guard(srcu)
srcu_read_lock_held
srcu_dereference
srcu_dereference_check
srcu_dereference_notrace
srcu_read_lock_held
```
SRCU：初始化/清理/排序：
```c
DEFINE_SRCU
DEFINE_STATIC_SRCU
DEFINE_SRCU_FAST        // for srcu_read_lock_fast() and friends
DEFINE_STATIC_SRCU_FAST // for srcu_read_lock_fast() and friends
init_srcu_struct
init_srcu_struct_fast
cleanup_srcu_struct
smp_mb__after_srcu_read_unlock
```
所有：锁依赖检查 RCU 所用 APIs：
```c
RCU_LOCKDEP_WARN
rcu_sleep_check
```
所有：不检查 RCU 保护指针访问：
```c
rcu_dereference_raw
```
所有：带禁止解引用的不检查 RCU 保护指针访问
```c
rcu_access_pointer
```

更多信息见源码中的注释头（或由它们生成的 docbook）。

不过，Linux内核里至少有四大类 RCU API，该如何选择？下面这份清单可以作为参考：

a. 读端是否需要阻塞（睡眠）？如果需要，选用 **SRCU**。
b. 读端需要阻塞，并且用于追踪场景（例如 ftrace、BPF）？如果是，选用 **RCU-tasks、RCU-tasks-rude 和/或 RCU-tasks-trace**。
c. 那 `-rt` 补丁集呢？在非实时内核中读端需要阻塞 → 选 SRCU。如果是在 `-rt` 实时内核里，只是拿自旋锁时会阻塞（非实时内核下不会阻塞），则 **不需要 SRCU**。（`-rt` 补丁会把自旋锁转化为可睡眠锁，因此才有这个区别。）
e. 是否需要把 NMI 处理函数、硬中断处理函数、以及关闭抢占的代码段（通过 `preempt_disable()`、`local_irq_save()`、`local_bh_disable()` 或其他方式）视作显式 RCU 读端？如果是，**RCU-sched 读端** 是唯一方案；不过从内核 v4.20 左右开始，你可以使用原生（vanilla）RCU 更新原语。
f. 是否要求：即便有 softirq 长时间霸占（monopolization）一个或多个CPU，RCU 宽限期依然能够正常完成？例如代码要抵御基于网络的拒绝服务攻击？如果是，你需要在读端屏蔽软中断，比如使用 `rcu_read_lock_bh()`。同样从 v4.20 起，可以搭配原生（vanilla）RCU 更新原语。
g. 业务的更新负载极高，常规 RCU 扛不住，但又不适合其他同步原语？可以考虑 **SLAB_TYPESAFE_BY_RCU**（旧名 `SLAB_DESTROY_BY_RCU`），但务必谨慎使用！
h. 是否要求读端临界区能够被正确识别：哪怕 CPU 处于深度 idle 循环、用户态进出边界、或者 CPU 已经热下线？这种场景下只有 SRCU 和 RCU Tasks Trace 可用，绝大多数场景优先选 SRCU。
i. 其他情况：直接使用标准 RCU。

当然，上面所有判断的前提是：你已经确认 RCU 本身就是适合这个场景的同步工具。

## 9. “RCU”代表什么？
缩写“RCU”代表“read-copy update”（读—复制—更新），这一说法源自 RCU 用于原地更新链式结构的方式。希望这么做的线程遵循以下步骤：
1. 新建一个结构；
2. 把旧结构中的数据复制到新结构中，并保存一个指向旧结构的指针；
3. 修改这个新的副本结构；
4. 更新全局指针，使其指向新结构；
5. 睡眠，直到操作系统内核判定已没有读者在使用旧结构 —— 例如在 Linux 内核中使用 `synchronize_rcu()`；最后
6. 醒来后释放旧结构。
于是，结构在被 **读** 取的同时，有线程正在 **复制** 它以完成 **更新**，故而得名“读—复制—更新”。缩写“RCU”是 Linux 社区众多贡献之一。
类似技术的其他名称还包括 VM/XA 程序员所称的 passive serialization 与 MP defer，以及 K42 与 Tornado 程序员所称的 generations。

## 10. 快速测验的答案
**快速测验 #1**：为什么这个论证很天真？在真实世界的 Linux 内核中使用该算法，死锁会如何发生？[指的是基于锁的那个“玩具”RCU 算法。]
**答案**：

考虑下面的事件序列：
1. CPU 0 获取某把无关的锁，称为 "problematic_lock"。
2. CPU 1 进入 `synchronize_rcu()`，以写方式获取 `rcu_gp_mutex`。
3. CPU 0 进入 `rcu_read_lock()`，但必须等待，因为 CPU 1 持有 `rcu_gp_mutex`。
4. CPU 1 被中断，其中断处理程序试图获取 problematic_lock。

此时系统就死锁了。

避免这种死锁的一种办法是采用类似 `CONFIG_PREEMPT_RT` 的方案：所有普通自旋锁都变成可阻塞锁，所有中断处理程序都在特殊任务的上下文中执行。
这样在上面的第 4 步，中断处理程序会阻塞，从而让 CPU 1 释放 `rcu_gp_mutex`，避免死锁。

即便不死锁，这个 RCU 实现也允许延迟通过 `synchronize_rcu()` 从一些读者“渗透（bleed）”到另一些读者。
要看清这一点，可以考虑任务 A 处于 RCU 读侧临界区（即读方式持有 `rcu_gp_mutex`）、任务 B 阻塞在试图以写方式获取 `rcu_gp_mutex`、任务 C 阻塞在 `rcu_read_lock()` 中试图以读方式获取 `rcu_gp_mutex`。
任务 A 的 RCU 读侧延迟挡住了任务 C，尽管是经由任务 B 间接造成的。

因此实时 RCU 实现采用基于计数器的方案：处于 RCU 读侧临界区的任务不会被正在执行 `synchronize_rcu()` 的任务阻塞。

**快速测验 #2**：举一个经典 RCU 读侧开销为 **负** 的例子。
**答案**：

设想一个单 CPU、非 `CONFIG_PREEMPT` 的内核：进程上下文的代码使用一张路由表，但该表可能被中断上下文的代码更新（例如被一个“ICMP REDIRECT”分组更新）。
处理这种情况的通常做法是让进程上下文代码在查表时关中断。

使用 RCU 则可以免去这种关中断操作。于是，不用 RCU，你要付出关中断的代价；用了 RCU，你就不必付出。

有人会主张，相对单 CPU 的关中断方案，此情形下 RCU 的开销是负的。也有人可能主张 RCU 的开销只是零，而用零开销的 RCU 方案取代关中断方案固有的正开销，并不构成负开销。

当然，现实世界要复杂得多。但一个同步原语在理论上竟可能有负开销，这本身还是有点出人意料。;-)

**快速测验 #3**：既然在 RCU 读侧临界区内阻塞是非法的，那么在 `PREEMPT_RT` 中普通自旋锁都可以阻塞，你又该怎么办？？？
**答案**：

正如 `PREEMPT_RT` 允许抢占自旋锁临界区一样，它也允许抢占 RCU 读侧临界区；它还允许在 RCU 读侧临界区内自旋锁阻塞。

为什么会有这种表面上的不一致？因为必要时可以用优先级提升（priority boosting）来保持 RCU 宽限期足够短（例如内存紧张的时候）。
相反，如果阻塞在等待（比如说）网络收包，就无从知道该提升谁。尤其考虑到我们需要提升的那个进程很可能只是个刚出门去买披萨的人。

而且，虽然电脑操控的赶牛棒（cattle prod）可能引起浓厚兴趣，也可能招致强烈反对。再说，电脑又怎么知道那个人去了哪家披萨店？？？

## 致谢
感谢帮助我把这份文档写得让人读得下去的人，包括 Jon Walpole、Josh Triplett、Serge Hallyn 与 Suzanne Wood。

更多信息见 http://www.rdrop.com/users/paulmck/RCU 以及 RCU 的 Wikipedia 条目。

# References
- [LWN - What is RCU, Fundamentally?](https://lwn.net/Articles/262464/)
- [LWN - What is RCU? Part 2: Usage](https://lwn.net/Articles/263130/)
- [LWN - RCU part 3: the RCU API](https://lwn.net/Articles/264090/)
- [rdrop - What is RCU](http://www.rdrop.com/users/paulmck/RCU/whatisRCU.html)
- [What is RCU_ -- “Read, Copy, Update” — The Linux Kernel documentation](https://origin.kernel.org/doc/html/latest/RCU/whatisRCU.html)
