# 内存屏障

* 谈及内存屏障，不理解 Store Buffer 和 Write Combine Buffer 不行，这两个 buffer 涉及处理器微架构，搬出两张微架构块图镇楼
#### Intel Skylake
![https://mdsattacks.com/images/skylake-color.svg](pic/skylake-color.svg)
* Skylake (Client) Microarchitecture
  * [https://en.wikichip.org/w/images/7/7e/skylake_block_diagram.svg](pic/skylake_block_diagram.svg)
* Skylake (Server) Microarchitecture
  * [https://en.wikichip.org/w/images/e/ee/skylake_server_block_diagram.svg](pic/skylake_server_block_diagram.svg)
#### AMD ZEN 5
![https://en.wikichip.org/w/images/0/02/zen_block_diagram.svg](pic/zen_block_diagram.svg)
* Zen 2 - Microarchitecture
  * [https://en.wikichip.org/w/images/f/f2/zen_2_core_diagram.svg](pic/zen_2_core_diagram.svg)

* **barrier**：可能重新排序和写的处理器提供了机器指令来确保顺序要求，这些确保顺序的指令称为 *屏障*。
* `volatile`关键字可以 **防止编译器的优化**，主要体现在以下三个方面：
  * 不会从寄存器中读取变量，而是重新从内存中取。
  * 不要对变量进行各种激进的优化，甚至将变量直接消除，保证程序员写在代码中的指令，一定会被执行。
  * `volatile`变量间的操作，不会被编译器交换顺序。
* `volatile`防止编译器优化的目的是确保数据能够被按顺序地无误地写入到内存，而不会因为编译器的原因产生偏差，设想某个变量其实对应的是通过`ioremap()`到内存的某个硬件寄存器，或者 x86 里变量要通过`in`或`out`指令访问的 I/O 空间。
* 即便是用`volatile`声明来防止编译器对变量操作的重新排序，仍然有可能会因为处理器优化 pipeline 的原因对指令重新排序，因此内存屏障仍然是很重要的。
* 对于 SMP，`smp_rmb()` , `smp_wmb()` , `smp_mb()` 和 `smp_read_barrier_depends()`是常用的 *内存屏障*（如 x86 的`mfence`指令）；对于 UP，则被定义为 *编译器屏障*，如 gcc 的`#define barrier() __asm__ __volatile__("": : :"memory")`。
* *内存屏障* 可以完成 *编译器屏障* 的功能，但编译器屏障要比处理器屏障 **轻量**（它实际上是轻快的）得多。
* 实际上，编译器屏障几乎是空的，因为它只防止编译器可能的重排指令。
* 不同体系架构，屏障的实际效果差别很大。如 x86 不会打乱存储，`wmb()`就什么也不做。
* 为最坏的情况（即排序能力最弱的处理器）使用恰当的内存屏障，代码才能在编译时执行针对体系结构的优化。

#### Memory and Compiler Barrier Methods

Barrier   | Description
----------|-------------
rmb()     | Prevents loads from being reordered across the barrier
read_barrier_depends() | Prevents data-dependent loads from being reordered across the barrier
wmb()     | Prevents stores from being reordered across the barrier
mb()      | Prevents load or stores from being reordered across the barrier
smp_rmb() | Provides an rmb() on SMP, and on UP provides a barrier()
smp_read_barrier_depends() | Provides a read_barrier_depends() on SMP, and provides a barrier() on UP
smp_wmb() | Provides a wmb() on SMP, and provides a barrier() on UP
smp_mb()  | Provides an mb() on SMP, and provides a barrier() on UP
barrier() | Prevents the compiler from optimizing stores or loads across the barrier

## 编译器屏障
* `volatile`关键字
* `READ_ONCE()/WRITE_ONCE()`其实用的也是`volatile`关键字，但没有采用`lock`前缀，并不能保证原子性
* `barrier()`宏/编译器扩展
  * include/linux/compiler.h
  ```c
  /* Optimization barrier */
  #ifndef barrier
  /* The "volatile" is due to gcc bugs */
  # define barrier() __asm__ __volatile__("": : :"memory")
  #endif
  ```

## 处理器内存屏障
* 谈及乱序谈论的是 **顺序性**，一般会涉及 *不同的存储位置* 和 *多个处理器*
* 乱序的起因是因为处理器的 store buffer，而不是 cache，因为 *高速缓存一致性协议（cache consistency protocol）* 能保证同一地址里的数据在多核的 cache 之间保持一致
* 内存屏障保证的只是本 CPU 上的指令执行顺序，进而间接保证其他 CPU 观察到的数据变化顺序，而不保证其他 CPU 上的执行顺序
* 内存屏障的使用应该成对，例如`smp_wmb`必须配对`smp_rmb`或者`smp_mb`，单独使用`smp_wmb`是达不到顺序效果的。同样`smp_rmb`必须配对使用`smp_wmb`或者`smp_mb`。
* 考虑以下观点：

> * store buffer 的作用是为了解决`store`操作执行 invalid 操作需要等待其它 CPU 响应 invalidate response 造成的 cpu stall 问题。
> * 所有架构的 store buffer 都是 FIFO，但是 FIFO 并不能保证`store`操作的顺序。
>   * 如果 CPU 是某内存地址对应 cache line 的 owner（modified/exclusive 状态），CPU 在执行该地址的`store`操作时不需要 invalid 其它 CPU 的 cache line，因此不需要将该操作放入 store buffer 中直接更新 cache line 就行了，这样就导致了该 cache line 先于 store buffer 中的地址可见，可能会造成乱序。
>   * `wb` 的作用是不管当前 CPU 是否是 cache line 的 owner，在执行对应物理地址的`store`操作时将其排在 store buffer 中，由该 buffer 的 FIFO 性质来保证写入的顺序。


## 抽象内存访问模型
* 考虑以下系统抽象模型：
```
               :                :
               :                :
               :                :
   +-------+   :   +--------+   :   +-------+
   |       |   :   |        |   :   |       |
   |       |   :   |        |   :   |       |
   | CPU 1 |<----->| Memory |<----->| CPU 2 |
   |       |   :   |        |   :   |       |
   |       |   :   |        |   :   |       |
   +-------+   :   +--------+   :   +-------+
       ^       :       ^        :       ^
       |       :       |        :       |
       |       :       |        :       |
       |       :       v        :       |
       |       :   +--------+   :       |
       |       :   |        |   :       |
       |       :   |        |   :       |
       +---------->| Device |<----------+
               :   |        |   :
               :   |        |   :
               :   +--------+   :
               :                :
```
* 每个 CPU 都会执行一个生成内存访问操作的程序。
  * 在抽象的 CPU 中，内存操作顺序非常宽松，CPU 实际上可以按照它喜欢的任何顺序执行内存操作，只要程序因果关系看起来得到了维持。
  * 同样，编译器也可以按照它喜欢的任何顺序排列它发出的指令，只要它不影响程序的明显操作。
* 因此，在上图中，当操作跨越 CPU 与系统其余部分之间的接口（虚线）时，CPU 执行的内存操作的影响会被系统其余部分感知到。
* 例如，考虑以下事件序列：
  ```c
  CPU 1           CPU 2
  =============== ===============
  { A == 1; B == 2 }
  A = 3;          x = B;
  B = 4;          y = A;
  ```
* 中间的内存系统所看到的访问集可以排列成 24 种不同的组合：
  ```c
  STORE A=3,      STORE B=4,      y=LOAD A->3,    x=LOAD B->4
  STORE A=3,      STORE B=4,      x=LOAD B->4,    y=LOAD A->3
  STORE A=3,      y=LOAD A->3,    STORE B=4,      x=LOAD B->4
  STORE A=3,      y=LOAD A->3,    x=LOAD B->2,    STORE B=4
  STORE A=3,      x=LOAD B->2,    STORE B=4,      y=LOAD A->3
  STORE A=3,      x=LOAD B->2,    y=LOAD A->3,    STORE B=4
  STORE B=4,      STORE A=3,      y=LOAD A->3,    x=LOAD B->4
  STORE B=4, ...
  ...
  ```
* 因此可以得到四种不同的值组合：
  ```c
  x == 2, y == 1 //二者都读到旧值
  x == 2, y == 3 //x 旧 y 新
  x == 4, y == 1 //x 新 y 旧
  x == 4, y == 3 //二者都读到新值
  ```
* 此外，一个 CPU 提交给内存系统的 stores 可能无法被另一个 CPU 的 loads 感知其提交时 stores 的顺序。
* 再举一个例子，考虑以下事件序列：
  ```c
  CPU 1           CPU 2
  =============== ===============
  { A == 1, B == 2, C == 3, P == &A, Q == &C }
  B = 4;          Q = P;
  P = &B;         D = *Q;
  ```
* 这里存在明显的地址依赖性，因为加载到 `D` 中的值取决于 CPU 2 从 `P` 获取到的地址。在序列结束时，可能出现以下任何结果：
  ```c
  (Q == &A) and (D == 1) //CPU2 上的指令先于 CPU1 执行
  (Q == &B) and (D == 2) //CPU2 上观察到 CPU1 上的 stroe 是乱序的
  (Q == &B) and (D == 4) //CPU2 上观察到 CPU1 上的 store 是顺序的
  ```
* 注意，CPU 2 永远不会尝试将 `C` 加载到 `D` 中，因为 CPU 会在发出 `*Q` 的加载之前将 `P` 加载到 `Q` 中。
  * **译注**：对于可见的数据依赖，是不会产生乱序的。CPU1 上的 stores 在 CPU1 看来并不构成数据依赖，所以可能乱序。

### 设备操作
* 某些设备将其控制接口显示为内存位置的集合，但访问控制寄存器的顺序非常重要。
* 例如，假设有一张以太网卡，它有一组内部寄存器，可通过地址 port 寄存器（`A`）和数据 port 寄存器（`D`）进行访问。要读取内部寄存器 `5`，可以使用以下代码：
  ```c
  *A = 5;
  x = *D;
  ```
* 但这可能会显示为以下两个序列之一：
  ```c
  STORE *A = 5, x = LOAD *D //期望的顺序
  x = LOAD *D, STORE *A = 5 //不期望出现的乱序
  ```
* 第二种情况几乎肯定会导致故障，因为它在尝试读取寄存器 *之后* 设置了地址。

### 保障
#### 对一个 CPU 可以期望的最小保障
* 在任何给定的 CPU，对它自己的 *有依赖的* 内存访问将会按顺序发送，这意味着：
  ```c
  Q = READ_ONCE(P); D = READ_ONCE(*Q);
  ```
  CPU 将发出以下内存操作：
  ```c
  Q = LOAD P, D = LOAD *Q
  ```
  并且始终按照该顺序执行。
  * 然而，在 DEC Alpha 上，`READ_ONCE()` 还会发出内存屏障指令，因此 DEC Alpha CPU 将改为发出以下内存操作：
  ```c
  Q = LOAD P, MEMORY_BARRIER, D = LOAD *Q, MEMORY_BARRIER
  ```
  无论是否在 DEC Alpha 上，`READ_ONCE()` 也能防止编译器的恶作剧。
* 在同一 CPU 上，对同一位置的 load 和 store 的顺序在该 CPU 上是顺序的，这意味着：
  ```c
  a = READ_ONCE(*X); WRITE_ONCE(*X, b);
  ```
  CPU 只会发出以下一系列的内存操作：
  ```c
  a = LOAD *X, STORE *X = b
  ```
  以及：
  ```c
  WRITE_ONCE(*X, c); d = READ_ONCE(*X);
  ```
  CPU 只会发出：
  ```c
  STORE *X = c, d = LOAD *X
  ```
  （如果 load 和 store 的目标是同一内存位置，则它们会重叠）。

#### “一定”或“一定不要”假设的事情
* 对于未被`READ_ONCE()`和`WRITE_ONCE()`保护的内存引用，**一定不要** 假设编译器会帮你做保护。缺乏他们，编译器有权利做任何它认为“有创造性”的变换。
* **一定不要** 假设 **无关联的** load 和 store 会按给定的顺序发送。这意味着：
  ```c
  X = *A; Y = *B; *D = Z;
  ```
  我们可能会得到以下任意一个序列：
  ```c
  X = LOAD *A,  Y = LOAD *B,  STORE *D = Z //最直观的顺序
  X = LOAD *A,  STORE *D = Z, Y = LOAD *B
  Y = LOAD *B,  X = LOAD *A,  STORE *D = Z
  Y = LOAD *B,  STORE *D = Z, X = LOAD *A
  STORE *D = Z, X = LOAD *A,  Y = LOAD *B
  STORE *D = Z, Y = LOAD *B,  X = LOAD *A
  ```
* **一定要** 假设对同一位置的内存访问会被合并或丢弃。这意味着：
  ```c
  X = *A; Y = *(A + 4);
  ```
  可能会按以下次序之一运行（译注：load-load 乱序？）：
  ```c
  X = LOAD *A; Y = LOAD *(A + 4);
  Y = LOAD *(A + 4); X = LOAD *A;
  {X, Y} = LOAD {*A, *(A + 4) };
  ```
  以及：
  ```c
  *A = X; *(A + 4) = Y;
  ```
  可能会按以下次序之一运行（译注：store-store 乱序？）：
  ```c
  STORE *A = X; STORE *(A + 4) = Y;
  STORE *(A + 4) = Y; STORE *A = X;
  STORE {*A, *(A + 4) } = {X, Y};
  ```
#### 无保障的事情
* 这些保证不适用于位域，因为编译器经常生成 *非原子的读取-修改-写入序列* 的代码来修改他们，因此，不要尝试使用位域来同步并行算法。 
* 即使在位域受锁保护的情况下，给定位域中的所有字段也必须由一个锁保护。
  * 如果给定位域中的两个字段由不同的锁保护，编译器的 *非原子读-修改-写序列* 可能会导致对一个字段的更新破坏相邻字段的值。
* 这些保证仅适用于正确对齐和大小标准的变量。
  * “适当大小”目前是指与 `char`、`short`、`int` 和 `long` 大小相同的变量。
  * “正确对齐”是指自然对齐，因此 `char` 没有约束，`short` 是两字节对齐，`int` 是四字节对齐，`long` 在 32 位和 64 位系统上分别是四字节或八字节对齐。
  * 注意，这些保证已被引入 C11 标准，因此在使用旧的 C11 之前的编译器（例如 gcc 4.6）时要小心。包含此保证的标准部分是第 3.14 节，其中定义“内存位置”如下：
* **内存位置**：一个标量类型的对象或相邻位域的最大序列都有非零的宽度
  * **注意 1**: 两个执行线程可以分别更新和访问不同的内存位置，而不会相互干扰
  * **注意 2**: 一个位域和一个相邻的非位域成员位于不同的内存位置。
    * 如果两个位域，一个在嵌套结构声明中声明而另一个没有，或者如果两者由零长度位域声明分隔，或者如果它们由非位域成员声明，这同样适用。
    * 无论这些中间位域的大小恰好是什么，如果在它们之间声明的所有成员也是位域，则同时更新同一结构中的两个位域是不安全的。

## 什么是内存屏障？
* 如上所示，独立内存操作实际上是以随机顺序执行的，但这对于 CPU-CPU 交互和 I/O 来说可能是一个问题。需要某种干预方式来指示编译器和 CPU 来限制（访问内存的）顺序。
* 内存屏障就是这样的干预。它们对屏障两侧的内存操作施加可感知的部分排序。
* 这种强制执行很重要，因为系统中的 CPU 和其他设备可以使用各种技巧来提高性能，包括重新排序、延迟和组合内存操作；推测性加载；推测性分支预测和各种类型的缓存。
  * 内存屏障用于覆盖或抑制这些技巧，允许代码合理地控制多个 CPU 和/或设备的交互。

### 内存屏障的种类
* 内存屏障有四种基本类型：
1. 写（或 store）内存屏障
   * 写屏障保证了相对于系统的其他组件而言，屏障之前指定的所有 store 操作都发生在屏障之后指定的所有 store 操作之前。
   * 写屏障仅对 stores 进行部分排序；它不需要对 loads 产生任何影响。
   * 可以将 CPU 视为随着时间的推移将 stores 操作序列提交给内存系统。写屏障 *之前* 的所有 stores 都将发生在写屏障之后的所有 stores *之前*。
   * **注意**，写屏障通常应与 *读* 或 *地址依赖* 屏障配对；请参阅“SMP 屏障配对”小节。
2. 地址依赖屏障（历史）
   * 本节被标记为 *历史*：它涵盖了早已过时的 `smp_read_barrier_depends()` 宏，该宏的语义现在隐含在所有标记的访问中。有关更多最新信息，包括编译器转换有时如何破坏地址依赖关系，请参阅 Documentation/RCU/rcu_dereference.rst。
   * 地址依赖屏障是读屏障的较弱形式。如果执行两次 loads，而第二次 load 依赖于第一次 load 的结果（例如：第一次 load 获取第二次 load 将指向的地址），则需要使用地址依赖屏障来确保在访问第一次 load 获得的地址后更新第二次 load 的目标。
   * 地址依赖屏障仅对相互依赖的 loads 进行部分排序；它不需要对 stores、独立的 loads 或重叠 loads 产生任何影响。
   * 如（1）中所述，系统中的其他 CPU 可以被视为将 stores 序列提交给内存系统，然后所考虑的 CPU 可以感知这些 stores 序列。
     * 所考虑的 CPU 发出的地址依赖屏障保证，对于它之前的任何 loads，如果该 load 接触来自另一个 CPU 的 stores 序列之一，那么在屏障完成时，在 *地址依赖屏障之后发出的任何 loads* 都可以感知到该 loads 接触的 stores 之前的所有 stores 的影响。
   * 请参阅“内存屏障序列示例”小节，查看显示排序约束的图表。
   * **注意**，第一次 load 实际上必须具有 *地址* 依赖性，而不是控制依赖性。如果第二次 load 的地址依赖于第一次 load，但依赖性是通过条件而不是实际 load 地址本身，那么它就是 *控制* 依赖性，需要完全读屏障或更好的屏障。有关更多信息，请参阅“控制依赖性”小节。
   * **注意**，地址依赖屏障通常应与写屏障配对；请参阅“SMP 屏障配对”小节。
   * 内核版本 v5.9 删除了显式地址依赖屏障的内核 API。如今，用于标记来自共享变量的加载的 API（例如 `READ_ONCE()` 和 `rcu_dereference()`）提供了隐式地址依赖屏障。
3. 读（或 load）内存屏障
   * 读屏障是地址依赖屏障加上保证，即相对于系统的其他组件，屏障之前指定的所有 load 操作看起来都发生在屏障之后指定的所有 load 操作之前。
   * 读屏障仅对 loads 进行部分排序；它不需要对 stores 产生任何影响。
   * 读内存屏障意味着地址依赖屏障，因此可以替代它们。
   * **注意**，读屏障通常应与写屏障配对；请参阅“SMP 屏障配对”小节。
4. 通用内存屏障
   * 通用内存屏障保证了屏障之前指定的所有 load 和 store 操作相对于系统的其他组件而言，看起来都发生在屏障之后指定的所有 load 和 store 操作之前。
   * 通用内存屏障是对 load 和 store 的部分排序。
   * 通用内存屏障意味着读取和写入内存屏障，因此可以替代其中任何一个。
* 还有几种隐含的变体：
5. `ACQUIRE` 操作
   * 这充当了单向可渗透屏障。它保证了 `ACQUIRE` 操作 **之后** 的所有内存操作相对于系统的其他组件来说，看起来都发生在 `ACQUIRE` 操作 **之后**。
   * `ACQUIRE` 操作包括 `LOCK` 操作以及 `smp_load_acquire()` 和 `smp_cond_load_acquire()` 操作。
   * `ACQUIRE` 操作 **之前** 发生的内存操作可能看起来发生在 `ACQUIRE` 操作完成 **之后**。
   * `ACQUIRE` 操作几乎总是应该与 `RELEASE` 操作配对。
6. `RELEASE` 操作
   * 这也充当了单向可渗透屏障。它保证了 `RELEASE` 操作 **之前** 的所有内存操作相对于系统的其他组件来说，看起来都发生在 `RELEASE` 操作 **之前**。
   * `RELEASE` 操作包括 `UNLOCK` 操作和 `smp_store_release()` 操作。
   * `RELEASE` 操作 **之后** 发生的内存操作可能看起来发生在操作完成 **之前**。
   * 使用 `ACQUIRE` 和 `RELEASE` 操作通常可以避免使用其他类型的内存屏障。
     * 此外，`RELEASE+ACQUIRE` 对不能保证充当完整的内存屏障。
     * 但是，在对给定变量执行 `ACQUIRE` 之后，保证在对该变量执行任何先前的 `RELEASE` 之前的所有内存访问都是可见的。换句话说，在给定变量的临界区内，保证对该变量的所有先前临界区的所有访问都已完成。
   * 这意味着 `ACQUIRE` 充当最小的“获取”操作，而 `RELEASE` 充当最小的“释放”操作。
* atomic_t.txt 中描述的原子操作子集除了完全有序和宽松（无屏障语义）定义外，还具有 `ACQUIRE` 和 `RELEASE` 变体。
  * 对于执行 load 和 store 的复合 atomics，`ACQUIRE` 语义仅适用于 load，`RELEASE` 语义仅适用于操作的 store 部分。
* **仅在两个 CPU 之间或 CPU 与设备之间可能存在交互时才需要内存屏障**。如果可以保证在任何特定代码段中不会发生任何此类交互，则该代码段中不需要内存屏障。
* **注意**，这些是最低限度的保证。不同的架构可能会提供更实质性的保证，但在特定架构的代码之外，它们可能不被依赖。

### 关于内存屏障，哪些是不能假设的？
* Linux 内核内存屏障不能保证某些事情：
* 无法保证在内存屏障之前指定的任何内存访问在内存屏障指令完成时会 *完成*；可以认为屏障在该 CPU 的访问队列中划了一条线，相应类型的访问不得越过该线。
* 无法保证在一个 CPU 上发出内存屏障会对另一个 CPU 或系统中的任何其他硬件产生任何直接影响。间接影响将是第二个 CPU 看到第一个 CPU 访问发生的影响的顺序，但请参见下一点：
* 无法保证 CPU 会看到第二个 CPU 访问的正确影响顺序，即使 *如果* 第二个 CPU 使用内存屏障，除非第一个 CPU *也* 使用匹配的内存屏障（请参阅“SMP 屏障配对”小节）。
* 无法保证某些中间的 CPU 外硬件不会重新排序内存访问。CPU 缓存一致性机制应该在 CPU 之间传播内存屏障的间接影响，但可能不会按顺序传播。
  * 有关总线控制 DMA 和一致性的信息，请阅读：
    - Documentation/driver-api/pci/pci.rst
    - Documentation/core-api/dma-api-howto.rst
    - Documentation/core-api/dma-api.rst

### 地址依赖屏障（历史）
* 本节被标记为历史：它涵盖了早已过时的 `smp_read_barrier_depends()` 宏，其语义现在隐含在所有标记的访问中。有关更多最新信息，包括编译器转换有时如何破坏地址依赖关系，请参阅 Documentation/RCU/rcu_dereference.rst。
* 从 Linux 内核 v4.15 开始，`smp_mb()` 已添加到 DEC Alpha 的 `READ_ONCE()` 中，这意味着只有那些使用 DEC Alpha 架构特定代码的人和那些使用 `READ_ONCE()` 本身的人才需要关注本节。对于那些需要它的人，以及对历史感兴趣的人，这里是地址依赖屏障的故事。
* 虽然在 load-to-load 和 load-to-store 关系中都观察到地址依赖性，但地址依赖屏障对于 load-to-store 情况不是必需的。
* 地址依赖屏障的要求有点微妙，而且并不总是很明显。为了说明这一点，请考虑以下事件序列：
  ```c
  CPU 1                 CPU 2
  ===============       ===============
  { A == 1, B == 2, C == 3, P == &A, Q == &C }
  B = 4;
  <write barrier>
  WRITE_ONCE(P, &B);
                        Q = READ_ONCE_OLD(P);
                        D = *Q;
  ```
  `READ_ONCE_OLD()` 对应于 4.15 之前内核的 `READ_ONCE()`，它不暗示地址依赖性障碍。
  这里有明显的地址依赖性，并且似乎在序列结束时，`Q` 必须是 `&A` 或 `&B`，并且：
  ```c
  (Q == &A) implies (D == 1) //CPU2 上的指令先于 CPU1 的执行
  (Q == &B) implies (D == 4) //CPU1 上的指令先于 CPU2 的执行
  ```
  但是！CPU 2 对 `P` 的感知可能在对 `B` 的感知 *之前* 更新，从而导致以下情况：
  ```c
  (Q == &B) and (D == 2) ????
  ```
  虽然这看起来像是一致性或因果关系维护的失败，但事实并非如此，并且这种行为可以在某些真实 CPU（例如 DEC Alpha）上观察到。
* 为了解决这个问题，自内核版本 v4.15 以来，`READ_ONCE()` 提供了一个隐式地址依赖屏障：
  ```c
    CPU 1                 CPU 2
  ===============       ===============
  { A == 1, B == 2, C == 3, P == &A, Q == &C }
  B = 4;
  <write barrier>
  WRITE_ONCE(P, &B);
                        Q = READ_ONCE(P);
                        <implicit address-dependency barrier>
                        D = *Q;
  ```
  这会强制发生两种含义之一，并阻止第三种可能性的出现。
* **注意**，这种极其违反直觉的情况最容易出现在具有 split caches 的机器上，例如，一个 cache bank 处理偶数 cache lines，另一个 cache bank 处理奇数  cache lines。
  * 指针 `P` 可能存储在奇数 cache lines 中，变量 `B` 可能存储在偶数 cache lines 中。
  * 然后，如果读取 CPU 的 cache 的偶数组非常繁忙，而奇数组空闲，则可以看到指针 `P` 的新值（`&B`），但变量 `B` 的旧值（`2`）。
* 不需要地址依赖屏障来对依赖性写入进行排序，因为 Linux 内核支持的 CPU 只有在确定（1）写入确实会发生、（2）写入的位置和（3）要写入的值时，才会执行写入。
* 但请仔细阅读“控制依赖关系”部分和 Documentation/RCU/rcu_dereference.rst 文件：编译器可以并且确实会以许多极具创意的方式打破依赖关系。
  ```c
    CPU 1                 CPU 2
  ===============       ===============
  { A == 1, B == 2, C = 3, P == &A, Q == &C }
  B = 4;
  <write barrier>
  WRITE_ONCE(P, &B);
                        Q = READ_ONCE_OLD(P);
                        WRITE_ONCE(*Q, 5);
  ```
  因此，不需要地址依赖屏障来将读入 `Q` 与存储到 `*Q` 进行排序。换句话说，即使没有现代 `READ_ONCE()` 的隐式地址依赖屏障，这种结果是被禁止的：
  ```c
  (Q == &B) && (B == 4)
  ```
* 请注意，这种模式应该很少见。毕竟，依赖性排序的整个目的是 **防止** 写入数据结构，以及与这些写入相关的昂贵的 cache misses。
  * 此模式可用于记录罕见的错误情况等，而 CPU 自然发生的排序可防止此类记录丢失。
* **注意**，地址依赖性提供的顺序对于包含它的 CPU 来说是本地的。有关更多信息，请参阅“多副本原子性”部分。
* 例如，地址依赖性屏障对 RCU 系统非常重要。请参阅 include/linux/rcupdate.h 中的 `rcu_assign_pointer()` 和 `rcu_dereference()`。
  * 这允许将 RCU 指针的当前目标替换为新的修改目标，而不会使替换目标看起来未完全初始化。
* 另请参阅“Cache 一致性”小节以获取更详尽的示例。

### 控制依赖
* 控制依赖关系可能有点棘手，因为当前的编译器无法理解它们。本节的目的是帮助您防止编译器的无知破坏您的代码。
* load-load 控制依赖关系需要完整的读内存屏障，而不仅仅是（隐式）地址依赖屏障才能使其正常工作。
* 请考虑以下代码：
  ```c
  q = READ_ONCE(a);
  <implicit address-dependency barrier>
  if (q) {
          /* BUG: No address dependency!!! */
          p = READ_ONCE(b);
  }
  ```
  这不会产生预期的效果，因为没有实际的地址依赖关系，而是一种控制依赖关系，CPU 可能会通过尝试提前预测结果来短路，以至于其他 CPU 认为来自 `b` 的 load 发生在来自 `a` 的 load 之前。在这种情况下，实际需要的是：
  ```c
  q = READ_ONCE(a);
  if (q) {
          <read barrier>
          p = READ_ONCE(b);
  }
  ```
* 但是，stores 不是推测的。这意味着为 load-store 控制依赖关系提供了排序，如下例所示：
  ```c
  q = READ_ONCE(a);
  if (q) {
          WRITE_ONCE(b, 1);
  }
  ```
* 控制依赖关系通常与其他类型的屏障配对。也就是说，请注意 `READ_ONCE()` 和 `WRITE_ONCE()` 都不是可选的！
  * 如果没有 `READ_ONCE()`，编译器可能会将来自“`a`”的 load 与来自“`a`”的其他 loads 相结合。
  * 如果没有 `WRITE_ONCE()`，编译器可能会将对“`b`”的 store 与对“`b`”的其他 stores 相结合。
* 两者都会对排序产生非常违反直觉的影响。
* 更糟糕的是，如果编译器能够证明（比如说）变量“`a`”的值始终为非零，那么它完全有权通过消除“`if`”语句来优化原始示例，如下所示：
  ```c
  q = a;
  b = 1;  /* BUG: Compiler and CPU can both reorder!!! */
  ```
  所以不要忽略 `READ_ONCE()`。
* 尝试在“`if`”语句的两个分支上强制对相同的 stores 进行排序是很有吸引力的，如下所示：
  ```c
  q = READ_ONCE(a);
  if (q) {
          barrier();
          WRITE_ONCE(b, 1);
          do_something();
  } else {
          barrier();
          WRITE_ONCE(b, 1);
          do_something_else();
  }
  ```
  不幸的是，当前的编译器会在高优化级别下进行如下转换：
  ```c
  q = READ_ONCE(a);
  barrier();
  WRITE_ONCE(b, 1);  /* BUG: No ordering vs. load from a!!! */
  if (q) {
          /* WRITE_ONCE(b, 1); -- moved up, BUG!!! */
          do_something();
  } else {
          /* WRITE_ONCE(b, 1); -- moved up, BUG!!! */
          do_something_else();
  }
  ```
* 现在，从“`a`” load 和 store 到“`b`”之间没有条件，这意味着 CPU 有权对它们进行重新排序：条件是绝对必要的，并且即使在应用了所有编译器优化之后，也必须存在于汇编代码中。
* 因此，如果您需要在此示例中进行排序，则需要显式内存屏障，例如 `smp_store_release()`：
  ```c
  q = READ_ONCE(a);
  if (q) {
          smp_store_release(&b, 1);
          do_something();
  } else {
          smp_store_release(&b, 1);
          do_something_else();
  }
  ```
* 相反，如果没有显式的内存屏障，则仅当 stores *不同* 时才能保证仅有两个条件的 `if` 控制顺序，例如：
  ```c
  q = READ_ONCE(a);
  if (q) {
          WRITE_ONCE(b, 1);
          do_something();
  } else {
          WRITE_ONCE(b, 2);    //store 的值是 2，不是 1
          do_something_else();
  }
  ```
  初始的 `READ_ONCE()` 仍然是必需的，以防止编译器证明“`a`”的值。
* 此外，您需要小心处理局部变量“`q`”，否则编译器可能会猜测该值并再次删除所需的条件。例如：
  ```c
  q = READ_ONCE(a);
  if (q % MAX) {
          WRITE_ONCE(b, 1);
          do_something();
  } else {
          WRITE_ONCE(b, 2);
          do_something_else();
  }
  ```
  如果 `MAX` 定义为 `1`，则编译器知道 `(q % MAX)` 等于零，在这种情况下，编译器有权将上述代码转换为以下内容：
  ```c
  q = READ_ONCE(a);
  WRITE_ONCE(b, 2);
  do_something_else();
  ```
* 鉴于这种转换，CPU 无需遵守从变量“`a`” load 和 store 到变量“`b`”之间的顺序。添加一个 `barrier()` 很诱人，但这无济于事。条件消失了，屏障也无法将其恢复。
* 因此，如果您依赖此顺序，则应确保 `MAX` 大于 `1`，可能如下所示：
  ```c
  q = READ_ONCE(a);
  BUILD_BUG_ON(MAX <= 1); /* Order load from a with store to b. */
  if (q % MAX) {
          WRITE_ONCE(b, 1);
          do_something();
  } else {
          WRITE_ONCE(b, 2);
          do_something_else();
  }
  ```
* 请再次注意，对“`b`”的 store 有所不同。如果它们相同，如前所述，编译器可以将此 stores 拉出“`if`”语句之外。
* 你还必须小心，不要过分依赖布尔短路求值。考虑这个例子：
  ```c
  q = READ_ONCE(a);
  if (q || 1 > 0)
          WRITE_ONCE(b, 1);
  ```
  因为第一个条件不会出错（cannot fault）并且第二个条件始终为 `true`，所以编译器可以将此示例转换为如下所示，从而克服控制依赖性：
  ```c
  q = READ_ONCE(a);
  WRITE_ONCE(b, 1);
  ```
* 此示例强调了确保编译器无法猜透您的代码的必要性。更一般地说，尽管 `READ_ONCE()` 确实会强制编译器针对给定 load 实际发出代码，但它不会强制编译器使用结果。
  * **译注**：意思是这里编译器完全略过了对 `q` 的值的判断，因为它发现第二个条件总是为 `true`，对第一个条件的求值就可以被优化掉
* 此外，控制依赖关系仅适用于相关 `if` 语句的 `then` 子句和 `else` 子句。具体而言，它不一定适用于 `if` 语句后面的代码：
  ```c
  q = READ_ONCE(a);
  if (q) {
          WRITE_ONCE(b, 1);
  } else {
          WRITE_ONCE(b, 2);
  }
  WRITE_ONCE(c, 1);  /* BUG: No ordering against the read from 'a'. */
  ```
  人们很容易认为事实上是有序的，因为编译器无法对 `volatile` 访问进行重新排序，也无法根据条件对“`b`”的写入进行重新排序。不幸的是，对于这种推理，编译器可能会将对“`b`”的两次写入编译为条件移动（conditional-move）指令，就像这种假象的伪汇编语言一样：
  ```asm
  ld r1,a       ;变量 a 的值 load 到寄存器 r1
  cmp r1,$0     ;r1 的值和立即数 0 比较
  cmov,ne r4,$1 ;如果不相等，把立即数 1 的值存入寄存器 r4
  cmov,eq r4,$2 ;如果相等，把立即数 2 的值存入寄存器 r4
  st r4,b       ;将寄存器 r4 的值 store 到变量 b
  st $1,c       ;将立即数 1 的值 store 到变量 c，但 CPU 可能会将该指令与第一条指令调换
  ```
* 弱顺序 CPU 在从“`a`” load 和 store 到“`c`”之间没有任何依赖关系。控制依赖关系将仅扩展到 `cmov` 指令对和依赖于它们的 stores。
* 简而言之，控制依赖关系仅适用于相关 `if` 语句的 `then` 子句和 `else` 子句中的 stores（包括这两个子句调用的函数），而不适用于该 `if` 语句后面的代码。
* **注意**，控制依赖项提供的顺序对于包含它的 CPU 来说是本地的。有关更多信息，请参阅“多副本原子性”部分。

#### 小结
* 控制依赖关系可以对前面的 loads 和后面 stores 进行排序。但是，它 **不** 保证任何其他类型的顺序：
  * 既不保证前面 loads 和后续 loads 的顺序，也不保证前面是 stores 和后续任何内容的顺序。
  * 如果您需要这些其他形式的顺序，请使用 `smp_rmb()`、`smp_wmb()`，或者在前面 stores 和后面 loads 的情况下使用 `smp_mb()`。
* 如果“`if`”语句的 *两个分支* 都以对 *同一变量* 的 *相同 stores* 开始，则必须对这些 stores 进行排序，方法是在它们前面加上 `smp_mb()`，或者使用 `smp_store_release()` 来执行 stores。
  * 请注意，在“`if`”语句的每个分支的开头使用 `barrier()` 是不够的，因为如上例所示，优化编译器可以在遵守 `barrier()` 定律的同时破坏控制依赖关系。
* 控制依赖关系要求在先前 loads 和后续 stores 之间至少有一个运行时条件，并且该条件必须涉及先前 loads。
  * 如果编译器能够优化条件，那么它也会优化顺序。谨慎使用 `READ_ONCE()` 和 `WRITE_ONCE()` 有助于保留所需的条件。
* 控制依赖关系要求编译器避免将依赖关系重新排序为不存在。谨慎使用 `READ_ONCE()` 或 `atomic{,64}_read()` 有助于保留控制依赖关系。
  * 请参阅“编译器屏障”部分了解更多信息。
* 控制依赖关系仅适用于包含控制依赖关系的 `if` 语句的 `then` 子句和 `else` 子句，包括这两个子句调用的任何函数。
  * 控制依赖关系不适用于包含控制依赖关系的 `if` 语句后面的代码。
* 控制依赖关系通常与其他类型的屏障配对。
* 控制依赖关系不提供多副本原子性。如果您需要所有 CPU 同时查看给定的 stores，请使用 `smp_mb()`。
* 编译器不理解控制依赖关系。因此，您的工作是确保他们不会破坏您的代码。

### SMP 屏障配对
* 在处理 CPU-CPU 交互时，某些类型的内存屏障应始终成对出现。缺乏适当的配对几乎肯定是错误的。
* 通用屏障相互配对，但它们也与大多数其他类型的屏障配对，尽管没有多副本原子性。
* 一个 acquire 屏障与一个 release 屏障配对，但两者都可以与其他屏障配对，当然包括通用屏障。
* 一个写屏障与一个地址依赖屏障、一个控制依赖屏障、一个 acquire 屏障、一个 realease 屏障、一个读屏障或一个通用屏障配对。
* 类似地，一个读屏障、控制依赖屏障或一个地址依赖屏障与一个写屏障、一个 acquire 屏障、一个 release 屏障或一个通用屏障配对：
  ```c
  CPU 1                 CPU 2
  ===============       ===============
  WRITE_ONCE(a, 1);
  <write barrier>
  WRITE_ONCE(b, 2);     x = READ_ONCE(b);
                        <read barrier>
                        y = READ_ONCE(a);
  ```
  或
  ```c
  CPU 1                 CPU 2
  ===============       ===============================
  a = 1;
  <write barrier>
  WRITE_ONCE(b, &a);    x = READ_ONCE(b);
                        <implicit address-dependency barrier>
                        y = *x;
  ```
  甚至（译注：假设初始值 `x = y = 0`？）：
  ```c
  CPU 1                 CPU 2
  ===============       ===============================
  r1 = READ_ONCE(y);
  <general barrier>
  WRITE_ONCE(x, 1);     if (r2 = READ_ONCE(x)) {
                            <implicit control dependency>
                            WRITE_ONCE(y, 1);
                        }

  assert(r1 == 0 || r2 == 0);
  ```
* 基本上，读屏障始终必须存在，即使它可以是“较弱”的类型。
* **注意**，写屏障之前的 stores 通常应该与读屏障或地址依赖屏障之后的 loads 相匹配，反之亦然：
  ```c
  CPU 1                               CPU 2
  ===================                 ===================
  WRITE_ONCE(a, 1);    }----   --->{  v = READ_ONCE(c);
  WRITE_ONCE(b, 2);    }    \ /    {  w = READ_ONCE(d);
  <write barrier>            \        <read barrier>
  WRITE_ONCE(c, 3);    }    / \    {  x = READ_ONCE(a);
  WRITE_ONCE(d, 4);    }----   --->{  y = READ_ONCE(b);
  ```

### 内存屏障序列示例
* 首先，写屏障在 store 操作中充当部分排序。考虑以下事件序列：
  ```c
  CPU 1
  =======================
  STORE A = 1
  STORE B = 2
  STORE C = 3
  <write barrier>
  STORE D = 4
  STORE E = 5
  ```
* 这一系列事件被提交给内存一致性系统，其顺序是系统其余部分可能认为的无序集合 `{ STORE A, STORE B, STORE C }` 都发生在无序集合 `{ STORE D, STORE E}` 之前：
  ```c
  +-------+       :      :
  |       |       +------+
  |       |------>| C=3  |     }     /\
  |       |  :    +------+     }-----  \  -----> 系统其他部分可感知的事件
  |       |  :    | A=1  |     }        \/
  |       |  :    +------+     }
  | CPU 1 |  :    | B=2  |     }
  |       |       +------+     }
  |       |   wwwwwwwwwwwwwwww }   <--- 此时，写屏障要求屏障之前的所有 stores
  |       |       +------+     }        操作必须先提交，之后才能进行更多的
  |       |  :    | E=5  |     }        stores 操作
  |       |  :    +------+     }
  |       |------>| D=4  |     }
  |       |       +------+
  +-------+       :      :
                      |
                      | 这是 CPU 1 向内存系统提交 stores 操作的顺序
                      |
                      V
  ```
* 其次，地址依赖屏障对存在地址依赖的 loads 操作起到部分排序的作用。考虑以下事件序列：
  ```c
  CPU 1                   CPU 2
  ======================= =======================
          { B = 7; X = 9; Y = 8; C = &Y }
  STORE A = 1
  STORE B = 2
  <write barrier>
  STORE C = &B            LOAD X
  STORE D = 4             LOAD C (获得 &B)
                          LOAD *C (读取 B)
  ```
* 如果不进行干预，尽管 CPU 1 发出了写屏障，CPU 2 仍可能以某种实际随机的顺序感知 CPU 1 上的事件：
  ```c
  +-------+       :      :                :       :
  |       |       +------+                +-------+  | CPU 2 上感知到的
  |       |------>| B=2  |-----       --->| Y->8  |  | 更新的顺序
  |       |  :    +------+     \          +-------+  |
  | CPU 1 |  :    | A=1  |      \     --->| C->&Y |  V
  |       |       +------+       |        +-------+
  |       |   wwwwwwwwwwwwwwww   |        :       :
  |       |       +------+       |        :       :
  |       |  :    | C=&B |---    |        :       :       +-------+
  |       |  :    +------+   \   |        +-------+       |       |
  |       |------>| D=4  |    ----------->| C->&B |------>|       |
  |       |       +------+       |        +-------+       |       |
  +-------+       :      :       |        :       :       |       |
                                 |        :       :       |       |
                                 |        :       :       | CPU 2 |
                                 |        +-------+       |       |
      Apparently incorrect --->  |        | B->7  |------>|       |
      perception of B (!)        |        +-------+       |       |
                                 |        :       :       |       |
                                 |        +-------+       |       |
      The load of X holds --->    \       | X->9  |------>|       |
      up the maintenance           \      +-------+       |       |
      of coherence of B             ----->| B->2  |       +-------+
                                          +-------+
                                          :       :
  ```
* 在上述示例中，尽管对 `*C`（其值应为 `B`）的 load 操作是在对 `C` 的 load 操作之后进行的，但 CPU 2 感知到的 `B` 值却是 `7`。
* 然而，如果在 CPU 2 上对 `C` 的 load 操作与对 `*C`（即 `B`）的 load 操作之间设置一个地址依赖屏障：
  ```c
  CPU 1                   CPU 2
  ======================= =======================
          { B = 7; X = 9; Y = 8; C = &Y }
  STORE A = 1
  STORE B = 2
  <write barrier>
  STORE C = &B            LOAD X
  STORE D = 4             LOAD C (获得 &B)
                          <address-dependency barrier>
                          LOAD *C (读取 B)
  ```
* 那么会发生如下事情：
  ```c
  +-------+       :      :                :       :
  |       |       +------+                +-------+
  |       |------>| B=2  |-----       --->| Y->8  |
  |       |  :    +------+     \          +-------+
  | CPU 1 |  :    | A=1  |      \     --->| C->&Y |
  |       |       +------+       |        +-------+
  |       |   wwwwwwwwwwwwwwww   |        :       :
  |       |       +------+       |        :       :
  |       |  :    | C=&B |---    |        :       :       +-------+
  |       |  :    +------+   \   |        +-------+       |       |
  |       |------>| D=4  |    ----------->| C->&B |------>|       |
  |       |       +------+       |        +-------+       |       |
  +-------+       :      :       |        :       :       |       |
                                 |        :       :       |       |
                                 |        :       :       | CPU 2 |
                                 |        +-------+       |       |
                                 |        | X->9  |------>|       |
                                 |        +-------+       |       |
    Makes sure all effects --->   \   aaaaaaaaaaaaaaaaa   |       |
    prior to the store of C        \      +-------+       |       |
    are perceptible to              ----->| B->2  |------>|       |
    subsequent loads                      +-------+       |       |
                                          :       :       +-------+
  ```
* 第三，读屏障在 load 上充当部分排序。考虑以下事件顺序：
  ```c
  CPU 1                   CPU 2
  ======================= =======================
          { A = 0, B = 9 }
  STORE A=1
  <write barrier>
  STORE B=2
                          LOAD B
                          LOAD A
  ```
* 在没有干预的情况下，CPU 2 可以选择以某种有效的随机顺序感知 CPU 1 上的事件，尽管 CPU 1 发出了写屏障：
  ```c
  +-------+       :      :                :       :
  |       |       +------+                +-------+
  |       |------>| A=1  |------      --->| A->0  |
  |       |       +------+      \         +-------+
  | CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
  |       |       +------+        |       +-------+
  |       |------>| B=2  |---     |       :       :
  |       |       +------+   \    |       :       :       +-------+
  +-------+       :      :    \   |       +-------+       |       |
                               ---------->| B->2  |------>|       |
                                  |       +-------+       | CPU 2 |
                                  |       | A->0  |------>|       |
                                  |       +-------+       |       |
                                  |       :       :       +-------+
                                   \      :       :
                                    \     +-------+
                                     ---->| A->1  |
                                          +-------+
                                          :       :
  ```
* 但是，如果在 CPU 2 上的 `B` 和 `A` 的 loads 之间放置一个读屏障：
  ```c
  CPU 1                   CPU 2
  ======================= =======================
          { A = 0, B = 9 }
  STORE A=1
  <write barrier>
  STORE B=2
                          LOAD B
                          <read barrier>
                          LOAD A
  ```
* 那么 CPU 1 施加的部分排序将被 CPU 2 正确感知：
  ```c
  +-------+       :      :                :       :
  |       |       +------+                +-------+
  |       |------>| A=1  |------      --->| A->0  |
  |       |       +------+      \         +-------+
  | CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
  |       |       +------+        |       +-------+
  |       |------>| B=2  |---     |       :       :
  |       |       +------+   \    |       :       :       +-------+
  +-------+       :      :    \   |       +-------+       |       |
                               ---------->| B->2  |------>|       |
                                  |       +-------+       | CPU 2 |
                                  |       :       :       |       |
                                  |       :       :       |       |
    At this point the read ---->   \  rrrrrrrrrrrrrrrrr   |       |
    barrier causes all effects      \     +-------+       |       |
    prior to the storage of B        ---->| A->1  |------>|       |
    to be perceptible to CPU 2            +-------+       |       |
                                          :       :       +-------+
  ```
* 为了更完整地说明这一点，考虑一下如果代码在读屏障的任何一侧都包含 `A` 的 load 会发生什么：
  ```c
  CPU 1                   CPU 2
  ======================= =======================
          { A = 0, B = 9 }
  STORE A=1
  <write barrier>
  STORE B=2
                          LOAD B
                          LOAD A [A 的第一次 load]
                          <read barrier>
                          LOAD A [A 的第二次 load]
  ```
* 即使 `A` 的两次 loads 都发生在 `B` 的 load 之后，它们可能都得到不同的值：
  ```c
  +-------+       :      :                :       :
  |       |       +------+                +-------+
  |       |------>| A=1  |------      --->| A->0  |
  |       |       +------+      \         +-------+
  | CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
  |       |       +------+        |       +-------+
  |       |------>| B=2  |---     |       :       :
  |       |       +------+   \    |       :       :       +-------+
  +-------+       :      :    \   |       +-------+       |       |
                               ---------->| B->2  |------>|       |
                                  |       +-------+       | CPU 2 |
                                  |       :       :       |       |
                                  |       :       :       |       |
                                  |       +-------+       |       |
                                  |       | A->0  |------>| 1st   |
                                  |       +-------+       |       |
    At this point the read ---->   \  rrrrrrrrrrrrrrrrr   |       |
    barrier causes all effects      \     +-------+       |       |
    prior to the storage of B        ---->| A->1  |------>| 2nd   |
    to be perceptible to CPU 2            +-------+       |       |
                                          :       :       +-------+
  ```
* 但可能是 CPU 1 对 `A` 的更新在读取屏障完成之前就被 CPU 2 感知到了：
  ```c
  +-------+       :      :                :       :
  |       |       +------+                +-------+
  |       |------>| A=1  |------      --->| A->0  |
  |       |       +------+      \         +-------+
  | CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
  |       |       +------+        |       +-------+
  |       |------>| B=2  |---     |       :       :
  |       |       +------+   \    |       :       :       +-------+
  +-------+       :      :    \   |       +-------+       |       |
                               ---------->| B->2  |------>|       |
                                  |       +-------+       | CPU 2 |
                                  |       :       :       |       |
                                   \      :       :       |       |
                                    \     +-------+       |       |
                                     ---->| A->1  |------>| 1st   |
                                          +-------+       |       |
                                      rrrrrrrrrrrrrrrrr   |       |
                                          +-------+       |       |
                                          | A->1  |------>| 2nd   |
                                          +-------+       |       |
                                          :       :       +-------+
  ```
* 可以保证的是，如果 `B` 的 load 得到 `B == 2`，则第二次 load 总是得到 `A == 1`。
  * `A` 的第一次 load 不存在这样的保证；可能会得到 `A == 0` 或 `A == 1`。

### 读屏障 vs load 投机
* 许多 CPU 会对 load 操作进行推测执行：也就是说，CPU 会预见到自己需要从内存中 load 某个数据项，然后会找一个没有用总线进行其他 load 操作的时机，提前执行该 load 操作 —— 即便此时在指令执行流程中还没有真正到达该 load 操作的位置。这使得实际的 load 指令有可能立即完成，因为 CPU 已经手头有了所需的值。
* 实际情况可能是，CPU 其实并不需要这个值 —— 或许是因为某个分支跳转绕过了该 load 操作 —— 在这种情况下，CPU 可以丢弃这个值，或者只是将其缓存起来供日后使用。
* 考虑以下例子：
  ```c
  CPU 1                   CPU 2
  ======================= =======================
                          LOAD B
                          DIVIDE          } 除法指令通常用
                          DIVIDE          } 很长的时间执行
                          LOAD A
  ```
* 可能会像这样
  ```c
                                          :       :       +-------+
                                          +-------+       |       |
                                      --->| B->2  |------>|       |
                                          +-------+       | CPU 2 |
                                          :       :DIVIDE |       |
                                          +-------+       |       |
  The CPU being busy doing a --->     --->| A->0  |~~~~   |       |
  division speculates on the              +-------+   ~   |       |
  LOAD of A                               :       :   ~   |       |
                                          :       :DIVIDE |       |
                                          :       :   ~   |       |
  Once the divisions are complete -->     :       :   ~-->|       |
  the CPU can then perform the            :       :       |       |
  LOAD with immediate effect              :       :       +-------+
  ```
* 在第二次 load 之前放置读屏障或地址依赖屏障：
  ```c
  CPU 1                   CPU 2
  ======================= =======================
                          LOAD B
                          DIVIDE
                          DIVIDE
                          <read barrier>
                          LOAD A
  ```
* 将迫使投机获得的任何值在一定程度上被重新考虑，这取决于所使用的屏障类型。如果推测的内存位置没有变化，那么推测的值就会被使用：
  ```c
                                          :       :       +-------+
                                          +-------+       |       |
                                      --->| B->2  |------>|       |
                                          +-------+       | CPU 2 |
                                          :       :DIVIDE |       |
                                          +-------+       |       |
  The CPU being busy doing a --->     --->| A->0  |~~~~   |       |
  division speculates on the              +-------+   ~   |       |
  LOAD of A                               :       :   ~   |       |
                                          :       :DIVIDE |       |
                                          :       :   ~   |       |
                                          :       :   ~   |       |
                                      rrrrrrrrrrrrrrrr~   |       |
                                          :       :   ~   |       |
                                          :       :   ~-->|       |
                                          :       :       |       |
                                          :       :       +-------+
  ```
* 但如果有来自另一个 CPU 的挂起的更新或使无效，那么推测将被取消并重新 load 值：
  ```c
                                          :       :       +-------+
                                          +-------+       |       |
                                      --->| B->2  |------>|       |
                                          +-------+       | CPU 2 |
                                          :       :DIVIDE |       |
                                          +-------+       |       |
  The CPU being busy doing a --->     --->| A->0  |~~~~   |       |
  division speculates on the              +-------+   ~   |       |
  LOAD of A                               :       :   ~   |       |
                                          :       :DIVIDE |       |
                                          :       :   ~   |       |
                                          :       :   ~   |       |
                                      rrrrrrrrrrrrrrrrr   |       |
                                          +-------+       |       |
  The speculation is discarded --->   --->| A->1  |------>|       |
  and an updated value is                 +-------+       |       |
  retrieved                               :       :       +-------+
  ```

### 多副本原子性（MULTICOPY ATOMICITY）
* **多副本原子性（Multicopy atomicity ）** 是一种关于顺序的极具直觉性的概念，但实际的计算机系统并非总能提供这种特性，即：
  * 某个给定的 store 操作会同时对所有 CPU 可见；
  * 或者说，所有 CPU 对于所有 stores 操作变为可见的顺序持一致看法。
* 然而，对完全多副本原子性的支持会排除一些有价值的硬件优化，因此一种被称为 “其他多副本原子性（other multicopy atomicity）” 的较弱形式应运而生，它仅保证某个给定的 store 操作会同时对所有其他 CPU 可见。
* 本文档的其余部分将讨论这种较弱形式的原子性，但为简洁起见，仅称之为 “多副本原子性”。
* 以下示例将阐释多副本原子性：
  ```c
  CPU 1                   CPU 2                   CPU 3
  ======================= ======================= =======================
          { X = 0, Y = 0 }
  STORE X=1               r1=LOAD X (读到 1)      LOAD Y (读到 1)
                          <general barrier>       <read barrier>
                          STORE Y=r1              LOAD X
  ```
* 假设 CPU 2 从 `X` load 的值为 `1`，随后将其存储到 `Y` 中，而 CPU 3 从 `Y` load 的值为 `1`。
  * 这表明 CPU 1 对 `X` 的 store 操作发生在 CPU 2 对 `X` 的 load 操作之前，且 CPU 2 对 `Y` 的 store 操作发生在 CPU 3 对 `Y` 的 load 操作之前。
  * 此外，内存屏障确保 CPU 2 先执行 load 操作，再执行 store 操作，
  * 且 CPU 3 先从 `Y` load 数据，再从 `X` load 数据。
* 那么问题来了：“CPU 3 从 `X` 加载的值可能为 `0` 吗？”
* 从某种意义上说，由于 CPU 3 对 `X` 的 load 操作发生在 CPU 2 对 `X` 的 load 操作之后，人们自然会认为 CPU 3 从 `X` load 的值也必定为 `1`。
  * 这种预期源于 **多副本原子性**：在多副本原子性系统中，如果 CPU B 上执行的 load 操作晚于 CPU A 上对同一变量的 load 操作（且 CPU A 最初并未 store 它所读取的值），那么 CPU B 的 load 操作返回的值要么与 CPU A 的 load 操作返回的值相同，要么是某个更新的值。
  * 然而，Linux 内核并不要求系统具备多副本原子性。
* 在上述示例中，通用内存屏障的使用弥补了任何多副本原子性的缺失。在该示例中，如果 CPU 2 从 `X` load 的值为 `1`，且 CPU 3 从 `Y` load 的值为 `1`，那么 CPU 3 从 `X` load 的值确实也必须为 `1`。
* 但是，依赖关系、读屏障和写屏障并非总能弥补非多副本原子性的问题。例如，假设从上述示例中移除 CPU 2 的通用屏障，仅留下如下所示的数据依赖：
  ```c
  CPU 1                   CPU 2                   CPU 3
  ======================= ======================= =======================
          { X = 0, Y = 0 }
  STORE X=1               r1=LOAD X (读到 1)      LOAD Y (读到 1)
                          <data dependency>       <read barrier>
                          STORE Y=r1              LOAD X (读到 0)
   ```
* 这种替换会让非多副本原子性的问题变得不受控制：在这个示例中，CPU 2 从 `X` load 的值为 `1`、CPU 3 从 `Y` load 的值为 `1`，而 CPU 3 从 `X` load 的值为 `0`，这种情况是完全合法的。
* 关键在于，尽管 CPU 2 的数据依赖关系对其 load 和 store 操作进行了排序，但它并不能保证对 CPU 1 的 store 操作进行排序。
  * 因此，如果该示例运行在一个非多副本原子性的系统上，且 CPU 1 和 CPU 2 共享一个 store buffer 或某一级 cache，那么 CPU 2 可能会提前访问到 CPU 1 的写入内容。
  * 因此，需要通用屏障来确保所有 CPU 对多个访问操作的组合顺序达成一致。
* ==通用屏障不仅能弥补非多副本原子性的不足，还能生成额外的排序，确保 **所有** CPU 都能感知到 **所有** 操作的相同顺序==。
* 相比之下，一系列 *释放 - 获取对（release-acquire pairs）* 无法提供这种额外的排序，这意味着只有该链条上的 CPU 才能保证对访问操作的组合顺序达成一致。
* 例如，为了致敬赫尔曼・霍勒瑞斯（Herman Hollerith）的精神，我们切换到 C 语言代码来举例：
  ```c
  int u, v, x, y, z;

  void cpu0(void)
  {
      r0 = smp_load_acquire(&x);
      WRITE_ONCE(u, 1);
      smp_store_release(&y, 1);
  }

  void cpu1(void)
  {
      r1 = smp_load_acquire(&y);
      r4 = READ_ONCE(v);
      r5 = READ_ONCE(u);
      smp_store_release(&z, 1);
  }

  void cpu2(void)
  {
      r2 = smp_load_acquire(&z);
      smp_store_release(&x, 1);
  }

  void cpu3(void)
  {
      WRITE_ONCE(v, 1);
      smp_mb();
      r3 = READ_ONCE(u);
  }
  ```
* 由于 `cpu0()`、`cpu1()` 和 `cpu2()` 在一个 `smp_store_release()`/`smp_load_acquire()` 对链中，因此不允许出现以下结果：
  ```c
  r0 == 1 && r1 == 1 && r2 == 1
  ```
  * **译注**：`r1 == 1 && r2 == 1` 意味着 `cpu2()` 的 `smp_store_release(&x, 1)` 必然在 `cpu0()` 的 `r0 = smp_load_acquire(&x)` 之后执行，因此不可能看到 `r0 == 1`
* 此外，由于 `cpu0()` 和 `cpu1()` 之间的 release-acquire 关系，`cpu1()` 必须看到 `cpu0()` 的写操作，因此禁止出现以下结果：
  ```c
  r1 == 1 && r5 == 0
  ```
  * **译注**：能看到 `r1 == 1` 说明 `cpu0()` 的 `smp_store_release(&y, 1)` 已经执行过了，在它之前的 `WRITE_ONCE(u, 1)` 更是已经执行过了，因此必然有 `r5 == 0`
* 然而，release-acquire 链提供的排序对于参与该链的 CPU 来说是本地的，而不适用于 `cpu3()`，至少除了 store 之外。因此，以下结果是可能的：
  ```c
  r0 == 0 && r1 == 1 && r2 == 1 && r3 == 0 && r4 == 0
  ```
* 顺便说一句，以下结果也是可能的：
  ```c
  r0 == 0 && r1 == 1 && r2 == 1 && r3 == 0 && r4 == 0 && r5 == 1
  ```
* 尽管 `cpu0()`、`cpu1()` 和 `cpu2()` 会按顺序看到各自的读写操作，但未参与 release-acquire 链的 CPU 很可能会对操作顺序存在分歧。
  * 这种分歧源于一个事实：用于实现 `smp_load_acquire()` 和 `smp_store_release()` 的弱内存屏障指令，并非在所有情况下都要求将先前的 store 操作与后续的 load 操作进行排序。
  * 这意味着，`cpu3()` 可能会认为 `cpu0()` 对 `u` 的 store 操作发生在 `cpu1()` 从 `v` load 数据 **之后**，即便 `cpu0()` 和 `cpu1()` 都一致认为这两个操作是按预期顺序发生的。
* 但请注意，`smp_load_acquire()` 并非什么神奇的指令。具体来说，它只是对其参数进行带有序性的读取操作，**并不能** 保证一定会读取到某个特定的值。因此，可能会出现以下结果：
  ```c
  r0 == 0 && r1 == 0 && r2 == 0 && r5 == 0
  ```
* 需要注意的是，即便在一个不存在任何重排序的假想的顺序一致性系统中，这种结果也可能发生。
* 再次强调，**如果你的代码要求所有操作都完全有序，请在整个过程中使用通用屏障**。

## 显式内核屏障
* Linux 内核有各种不同的屏障，它们在不同的层次上起作用：
  * 编译器屏障。
  * CPU 内存屏障。

### 编译器屏障
* Linux 内核有一个显式的编译器屏障函数，它能阻止编译器将屏障两侧的内存访问操作移到另一侧：
  ```c
  barrier();
  ```
* 这是一个通用屏障 —— 不存在针对 read-read 或 write-write 操作的 `barrier()` 变体。
* 不过，`READ_ONCE()` 和 `WRITE_ONCE()` 可被视为 `barrier()` 的弱形式，它们仅对由 `READ_ONCE()` 或 `WRITE_ONCE()` 标记的特定访问操作产生影响。
* `barrier()` 函数具有以下作用：
  * 防止编译器将 `barrier()` 之后的访问操作重排序到 `barrier()` 之前的任何访问操作之前。此特性的一个使用场景是简化中断处理程序代码与被中断代码之间的通信。
  * 在循环中，强制编译器在每次循环迭代时加载循环条件中使用的变量。
* `READ_ONCE()` 和 `WRITE_ONCE()` 函数能够阻止多种优化操作，这些优化在单线程代码中是完全安全的，但在并发代码中可能会导致致命错误。以下是这类优化的一些示例：

#### 示例 1
* 编译器有权对 **同一变量** 的 loads 和 stores 操作进行重排序，在某些情况下，CPU 也有权对 **同一变量** 的 loads 操作进行重排序。这意味着以下代码：
  ```c
  a[0] = x;
  a[1] = x;
  ```
* 可能会导致存储在 `a[1]` 中的 `x` 值比存储在 `a[0]` 中的 `x` 值更旧。
  * **译注**：这意味着有其他 CPU 在并发地修改 `x` 的值
* 可以通过以下方式同时阻止编译器和 CPU 进行此类操作：
  ```c
  a[0] = READ_ONCE(x);
  a[1] = READ_ONCE(x);
  ```
* 简言之，`READ_ONCE()` 和 `WRITE_ONCE()` 为多个 CPU 对 **单个变量** 的访问提供了 cache 一致性。
* **译注**：`READ_ONCE()` 和 `WRITE_ONCE()` 本质上是加了 `volatile` 关键字，并没有插入内存屏障指令，如何能阻止 CPU 的重排序？！

#### 示例 2
* 编译器有权合并来自同一变量的连续 loads。这样的合并会导致编译器“优化”以下代码：
  ```c
  while (tmp = a)
      do_something_with(tmp);
  ```
  * 成为下面的代码，虽然在某种意义上对单线程代码是合法的，但几乎肯定不是开发人员想要的：
  ```c
  if (tmp = a)
     for (;;)
        do_something_with(tmp);
  ```
  * **译注**：注意，这里是“赋值”不是“等于”，如果编译器在编译时认定 `a` 的值恒等于 `true` 就有可能优化成上面的代码，然而 `a` 可能在别的 CPU 通过地址引用的方式修改
* 使用 `READ_ONCE()` 来防止编译器对你（的代码）这样做：
  ```c
  while (tmp = READ_ONCE(a))
        do_something_with(tmp);
  ```

#### 示例 3
* 编译器有权重新加载变量，例如，在寄存器压力较大的情况下，编译器无法将所有关注的数据都保留在寄存器中，就可能会这样做。因此，在我们前面的示例中，编译器可能会优化掉变量“`tmp`”：
  ```c
  while (tmp = a)
      do_something_with(tmp);
  ```
* 这可能会生成以下代码，这种代码在单线程中是完全安全的，但在并发代码中可能会导致致命错误：
  ```c
  while (a)
      do_something_with(a);
  ```
* 例如，在“`while`”语句和调用 `do_something_with()` 之间，若变量 `a` 被其他 CPU 修改，那么经过优化的代码可能会向 `do_something_with()` 传递一个零值。
* 同样，可以使用 `READ_ONCE()` 来阻止编译器进行此类优化：
  ```c
  while (tmp = READ_ONCE(a))
      do_something_with(tmp);
  ```
* 需要注意的是，如果编译器的寄存器不足，它可能会将 `tmp` 保存到栈中。这种保存和后续恢复操作所带来的开销，正是编译器重新加载变量的原因。
* 对于单线程代码来说，这样做完全安全，因此在不安全的场景下，你需要明确告知编译器。

#### 示例 4

* 如果编译器知道某个变量的值，它有权完全省略对该变量的 load 操作。例如，若编译器能证明变量“`a`”的值始终为零，它会将以下代码：
  ```c
  while (tmp = a)
      do_something_with(tmp);
  ```
  * 优化为：
  ```c
  do { } while (0);
  ```
* 这种转换对单线程代码是有益的，因为它省去了一次 load 操作和一个分支判断。
  * 但问题在于，编译器进行这种推理的前提是，假设当前 CPU 是唯一会更新变量“`a`”的 CPU。如果变量“`a`”是共享变量，那么编译器的这种推理就会出现错误。
  * 此时可以使用 `READ_ONCE()` 来告知编译器，它并没有自己想象中那么了解情况：
  ```c
  while (tmp = READ_ONCE(a))
      do_something_with(tmp);
  ```
* 但需要注意的是，编译器还会密切关注你在 `READ_ONCE()` 之后对该值的处理。
* 例如，假设你编写了以下代码，且 `MAX` 是一个值为 `1` 的预处理宏：
  ```c
  while ((tmp = READ_ONCE(a)) % MAX)
      do_something_with(tmp);
  ```
  * 那么编译器会知道，对 `MAX` 进行“`%`”运算的结果始终为零，因此又会将这段代码优化得几乎不存在（但它仍然会从变量“`a`”中加载值）。

#### 示例 5

* 同样，若编译器知道变量已具有将要 store 的值，它有权完全省略该 store 操作。
* 此外，编译器会假设当前 CPU 是唯一会向该变量执行 store 操作的 CPU，这可能导致编译器在处理共享变量时出错。
* 例如，假设你有如下代码：
  ```c
  a = 0;
  ... 不向变量 a 执行 store 操作的代码 ...
  a = 0;
  ```
* 编译器会发现变量“`a`”的值已经是 `0`，因此很可能会省略第二个 store 操作。如果在此期间其他某个 CPU 可能已经向变量“`a`”执行了 store 操作，这种情况就会带来致命问题。
* 使用 `WRITE_ONCE()` 可以防止编译器做出此类错误判断：
  ```c
  WRITE_ONCE(a, 0);
  ... 不向变量 a 执行 store 操作的代码 ...
  WRITE_ONCE(a, 0);
  ```

#### 示例 6

* 编译器有权对内存访问操作进行重排序，除非你明确告知它不要这样做。例如，考虑以下进程级代码与中断处理程序之间的交互：
  ```c
  void process_level(void)
  {
      msg = get_message();
      flag = true;
  }

  void interrupt_handler(void)
  {
      if (flag)
         process_message(msg);
  }
  ```
* 没有什么东西阻止编译器将 `process_level()` 转换为以下形式，实际上，对于单线程代码而言，这种转换甚至可能是有益的：
  ```c
  void process_level(void)
  {
      flag = true;
      msg = get_message();
  }
  ```
* 如果中断发生在这两条语句之间，那么 `interrupt_handler()` 可能会接收到一个错乱的 `msg` 值。可以通过 `WRITE_ONCE()` 防止这种情况，如下所示：
  ```c
  void process_level(void)
  {
      WRITE_ONCE(msg, get_message());
      WRITE_ONCE(flag, true);
  }

  void interrupt_handler(void)
  {
      if (READ_ONCE(flag))
         process_message(READ_ONCE(msg));
  }
  ```
* 需要注意的是，如果该中断处理程序可能被其他同样访问 `flag` 和 `msg` 的操作（例如嵌套中断或不可屏蔽中断 NMI）中断，那么 `interrupt_handler()` 中的 `READ_ONCE()` 和 `WRITE_ONCE()` wrappers 是必需的。
  * 否则，`interrupt_handler()` 中的 `READ_ONCE()` 和 `WRITE_ONCE()` 仅出于文档说明目的才需要（另请注意，现代 Linux 内核中通常不会发生嵌套中断，实际上，如果中断处理程序在中断启用的情况下返回，会触发 `WARN_ONCE()` 警告）。
* 你应当假设，编译器可以将 `READ_ONCE()` 和 `WRITE_ONCE()` 移动到不包含 `READ_ONCE()`、`WRITE_ONCE()`、`barrier()` 或类似原语的代码前后。
  * 这种效果也可以通过 `barrier()` 实现，但 `READ_ONCE()` 和 `WRITE_ONCE()` 的针对性更强：
    * 使用 `READ_ONCE()` 和 `WRITE_ONCE()` 时，编译器只需忘记指定内存位置的内容；
    * 而使用 `barrier()` 时，编译器必须丢弃当前在任何机器寄存器中 cache 的所有内存位置的值。
  * 当然，编译器必须遵守 `READ_ONCE()` 和 `WRITE_ONCE()` 的执行顺序，不过 CPU 显然不必如此。

#### 示例 7
* 编译器有权凭空生成对变量的 store 操作，如下例所示：
  ```c
  if (a)
     b = a;
  else
     b = 42;
  ```
* 编译器可能会通过如下优化来省去一个分支判断：
  ```c
  b = 42;
  if (a)
     b = a;
  ```
* 在单线程代码中，这不仅安全，还能减少一次分支判断。但遗憾的是，在并发代码中，这种优化可能导致其他 CPU 在 load 变量 `b` 时，看到一个虚假的 `42` 值 —— 即便变量 `a` 从未为零。
* 可以使用 `WRITE_ONCE()` 来防止这种情况，如下所示：
  ```c
  if (a)
     WRITE_ONCE(b, a);
  else
     WRITE_ONCE(b, 42);
  ```
* 编译器也可能凭空生成 load 操作。这类操作通常破坏性较小，但可能导致缓存行颠簸（cache-line bouncing），进而影响性能和可扩展性。
* 使用 `READ_ONCE()` 可以防止编译器生成这类凭空的 load 操作。

#### 示例 8
* 对于那些对齐的内存位置（其大小允许通过单条内存引用指令访问），防止“加载撕裂”（load tearing）和“存储撕裂”（store tearing）—— 即单个大型访问操作被拆分为多个小型访问操作的情况。
* 例如，在某架构中，16-bit store 指令的立即数字段为 7-bit，编译器可能会倾向于使用两条 16-bit store-immediate 指令来实现以下 32-bit store 操作：
  ```c
  p = 0x00010002;
  ```
* 需要注意的是，GCC 确实会采用这类优化，这并不奇怪，因为构建该常量再进行 store 可能需要不止两条指令。因此，在单线程代码中，这种优化可能是有益的。
* 事实上，最近曾出现一个 bug（现已修复），导致 GCC 在 `volatile` store 操作中错误地使用了这种优化。
* 在没有此类 bug 的情况下，使用 `WRITE_ONCE()` 可以防止以下示例中的 store tearing：
  ```c
  WRITE_ONCE(p, 0x00010002);
  ```
* 压缩结构体（packed structures）也可能导致 load 和 store tearing，如下例所示：
  ```c
  struct __attribute__((__packed__)) foo {
          short a;
          int b;
          short c;
  };
  struct foo foo1, foo2;
  ...

  foo2.a = foo1.a;
  foo2.b = foo1.b;
  foo2.c = foo1.c;
  ```
* 由于没有使用 `READ_ONCE()` 或 `WRITE_ONCE()` 包装，也没有 `volatile` 标记，编译器完全有权将这三条赋值语句实现为两次 32-bit loads 操作， 接着两次 32-bit stores 操作。这会导致对 `foo1.b` 的 load tearing 和对 `foo2.b` 的 store tearing。
* 在这个示例中，`READ_ONCE()` 和 `WRITE_ONCE()` 同样可以防止撕裂：
  ```c
  foo2.a = foo1.a;
  WRITE_ONCE(foo2.b, READ_ONCE(foo1.b));
  foo2.c = foo1.c;
  ```
---
* 除此之外，对于已标记为 `volatile` 的变量，永远不需要使用 `READ_ONCE()` 和 `WRITE_ONCE()`。
  * 例如，由于 `jiffies` 被标记为 `volatile`，因此永远不必写成 `READ_ONCE(jiffies)`。
  * 原因是 `READ_ONCE()` 和 `WRITE_ONCE()` 是通过 `volatile` 强制转换实现的，当它们的参数已被标记为 `volatile` 时，这种转换就不会产生任何效果。
* 请注意，这些编译器屏障对 CPU 没有直接影响，CPU 仍可以按照自己的意愿对操作进行重排序。

### CPU 内存屏障
* Linux 内核有 7 个基本的 CPU 内存屏障：

类型    | MANDATORY | SMP CONDITIONAL
--------|-----------|-----------------------
通用    | mb()      | smp_mb()
写      | wmb()     | smp_wmb()
读      | rmb()     | smp_rmb()
地址依赖 | .         | READ_ONCE()

* **除地址依赖屏障外，所有内存屏障都隐含着编译器屏障**。地址依赖不会施加任何额外的编译器排序约束。
* 补充说明：在存在地址依赖的情况下，编译器理应按正确顺序执行 load 操作（例如，`a[b]` 必须先 load `b` 的值，再 load`a[b]`），
  * 但 C 语言规范并未保证编译器不会对 `b` 的值进行推测（例如，推测 `b` 的值为 `1`）并在 load `b`之前就 load `a[b]`（例如，`tmp = a[1]; if (b != 1) tmp = a[b];`）。
* 此外，还存在一种问题：编译器可能在 load `a[b]` 之后重新 load `b`，导致 `b` 的副本比 `a[b]` 更新。
* 关于这些问题尚未达成共识，但`READ_ONCE()`宏是一个值得着手研究的方向。
* 在单处理器编译的系统中，SMP 内存屏障会被简化为编译器屏障，因为假设单个 CPU 会呈现出自一致性，并且会相对于自身正确排序重叠的访问操作。不过，请参见下文“[虚拟机 Guest](#虚拟机-guest)”小节。
* **注意**：在 SMP 系统上，必须使用 SMP 内存屏障来控制对共享内存的引用顺序，不过使用锁机制来替代也足够。
* 不应使用强制屏障（Mandatory Barriers）来控制 SMP 相关的效果，因为强制屏障会给 SMP 和 UP 系统都带来不必要的开销。
  * 不过，它们可用于控制通过 relaxed memory I/O 窗口进行的访问所产生的 MMIO 效应。
  * 即使在非 SMP 系统上，这些屏障也是必需的，因为它们通过禁止编译器和 CPU 对内存操作进行重排序，来控制内存操作呈现给设备的顺序。
---
* 还有一些更高级的屏障函数：
  ```c
  smp_store_mb(var, value)
  ```
* 这将值赋给变量，然后在它 **之后** 插入一个完整的内存屏障。它不能保证在 UP 编译中插入除了编译器屏障之外的任何东西。
---
  ```c
  smp_mb__before_atomic();
  smp_mb__after_atomic();
  ```
* 这些屏障用于那些不隐含内存屏障但代码中需要内存屏障的 *原子读-改-写（RMW）函数*。
* 不隐含内存屏障的原子 RMW 函数示例
  * 包括加法（add）、减法（subtract）、（failed）条件操作、`_relaxed` 函数等，
  * 但不包括 `atomic_read` 或 `atomic_set`。
* 原子操作用于引用计数时，通常就是需要内存屏障的常见场景。
* 这些屏障也用于那些不隐含内存屏障的原子 RMW 位操作函数（如 `set_bit` 和 `clear_bit`）。
* 例如，考虑一段将某个对象标记为“已销毁”，然后递减该对象引用计数的代码：
  ```c
  obj->dead = 1;
  smp_mb__before_atomic();
  atomic_dec(&obj->ref_count);
  ```
* 这能确保该对象的“已销毁”标记被感知为在引用计数递减 **之前** 设置。
* 更多信息请参见 Documentation/atomic_{t,bitops}.txt。
---
  ```c
  dma_wmb();
  dma_rmb();
  dma_mb();
  ```
* 这些屏障用于一致性内存，以保证 CPU 和具备 DMA 能力的设备均可访问的共享内存的读写顺序。
* 有关一致性内存的更多信息，请参见 Documentation/core-api/dma-api.rst文件。
* 例如，考虑一个与设备共享内存的设备驱动程序，它使用描述符状态值来指示描述符属于设备还是 CPU，并使用门铃（doorbell）在新描述符可用时通知设备：
```c
if (desc->status != DEVICE_OWN) {
   /* 在获得描述符所有权之前不读取数据 */
   dma_rmb();

   /* 读取/修改数据 */
   read_data = desc->data;
   desc->data = write_data;

   /* 在更新状态前刷新修改 */
   dma_wmb();

   /* 分配所有权 */
   desc->status = DEVICE_OWN;

   /* 先让设备看到描述符状态，再通知设备有新描述符 */
   writel(DESC_NOTIFY, doorbell);
}
```
* `dma_rmb()` 确保我们在从描述符读取数据之前，设备已经释放了所有权；
* `dma_wmb()` 确保在设备看到自己获得所有权之前，数据已写入描述符。
* `dma_mb()` 同时具备 `dma_rmb()` 和 `dma_wmb()` 的功能。
* 请注意，`dma_*()` 屏障不保证对 MMIO（内存映射 I/O）区域访问的顺序。有关 I/O 访问函数和 MMIO 排序的更多信息，请参见后文的“KERNEL I/O BARRIER EFFECTS”小节。
---
```c
pmem_wmb();
```
* 这用于持久性内存（persistent memory），以确保那些修改已写入持久性存储的 stores 操作已到达平台的持久性域（platform durability domain）。
* 例如，在对持久性内存区域执行非临时写入（non-temporal write）后，我们使用 `pmem_wmb()` 来确保 stores 操作已到达平台的持久性域。
  * 这能保证在后续指令引发的任何数据访问或数据传输开始之前，stores 操作已完成对持久性存储的更新。
  * 这是在 `wmb()` 所做排序之外的额外保障。
* 对于从持久性内存进行的 load 操作，现有的读内存屏障足以确保读取顺序。
---
```c
io_stop_wc();
```
* 对于具有 **写合并（write-combining）** 属性的内存访问（例如由 `ioremap_wc()` 返回的内存区域），CPU 可能会等待先前的访问与后续访问合并。
* 当这种等待存在性能影响时，`io_stop_wc()` 可用于 **防止** 该宏 **之前** 的写合并内存访问与 **之后**的写合并内存访问发生 **合并**。

## 隐式内核内存屏障
* Linux 内核中的其他一些函数也隐含着内存屏障，其中包括加锁函数和调度函数。
* 本规范仅提供 **最低限度** 的保证；任何特定的架构都可能提供更全面的保证，但在架构特定代码之外，不能依赖这些额外保证。

### 锁获取函数
* Linux内核有很多锁结构：
  * spin locks
  * R/W spin locks
  * mutexes
  * semaphores
  * R/W semaphores
* 在所有情况下，每种结构都有“`ACQUIRE`（获取）”操作和“`RELEASE`（释放）”操作的变体。这些操作都隐含着特定的屏障：
1. `ACQUIRE` 操作的隐含意义：
   - 在 `ACQUIRE` 之后发出的内存操作，将在 `ACQUIRE` 操作完成后才会完成。
   - 在 `ACQUIRE` 之前发出的内存操作，可能在 `ACQUIRE` 操作完成后才完成。
2. `RELEASE` 操作的隐含意义：
   - 在 `RELEASE` 之前发出的内存操作，将在 `RELEASE` 操作完成前完成。
   - 在 `RELEASE` 之后发出的内存操作，可能在 `RELEASE` 操作完成前就完成。
3. `ACQUIRE` 与 `ACQUIRE` 的隐含关系：
   - 所有在另一个 `ACQUIRE` 操作之前发出的 `ACQUIRE` 操作，将在该 `ACQUIRE` 操作完成前完成。
4. `ACQUIRE` 与 `RELEASE` 的隐含关系：
   - 所有在 `RELEASE` 操作之前发出的 `ACQUIRE` 操作，将在该 `RELEASE` 操作完成前完成。
5. 失败条件性的 `ACQUIRE` 的隐含意义：
   - 某些 `ACQUIRE` 操作的加锁变体可能会失败，原因可能是无法立即获得锁，或者在等待锁可用时休眠期间收到了解除阻塞的信号。
   - 失败的锁操作不隐含任何形式的屏障。
* **注意**：锁的 `ACQUIRE` 和 `RELEASE` 只是 **单向屏障**，其后果之一是，临界区外部指令的影响可能会渗透到临界区内部。
* 不能假设一个 `ACQUIRE` 之后跟着一个 `RELEASE` 就是完整的内存屏障，因为 `ACQUIRE` 之前的访问可能在 `ACQUIRE` 之后发生，`RELEASE` 之后的访问可能在 `RELEASE` 之前发生，而且这两个访问本身可能会交叉：
  ```c
  *A = a;
  ACQUIRE M
  RELEASE M
  *B = b;
  ```
  * 可能会以如下顺序执行：
  ```c
  ACQUIRE M, STORE *B, STORE *A, RELEASE M
  ```
* 当 `ACQUIRE` 和 `RELEASE` 分别是锁的获取和释放时，如果锁的 `ACQUIRE` 和 `RELEASE` 针对的是同一个锁变量，那么从不持有该锁的其他 CPU 的角度来看，也可能发生同样的重排序。
* ==简言之，**不能** 假设一个 `ACQUIRE` 之后跟着一个 `RELEASE` 就是完整的内存屏障==。
* 同样，释放（`RELEASE`）之后跟着获取（`ACQUIRE`）的情况也不意味着完整的内存屏障。因此，CPU 执行与 `RELEASE` 和 `ACQUIRE` 对应的临界区时可能会出现交叉，以至于：
  ```c
  *A = a;
  RELEASE M
  ACQUIRE N
  *B = b;
  ```
* 可能会以如下顺序执行：
  ```c
  ACQUIRE N, STORE *B, STORE *A, RELEASE M
  ```
* 这看起来似乎这种重排序可能会导致死锁。但实际上不会发生，因为如果存在死锁风险，`RELEASE` 操作会正常完成，从而避免死锁。
  * **译注**：比如其他 CPU 上的执行顺序是 `ACQUIRE N, ACQUIRE M` 并且先执行

#### 为什么会这样呢？
* 一个关键点是，我们这里只讨论 CPU 进行的重排序，而非编译器的重排序。如果是编译器（或者说开发者）交换了操作顺序，那么死锁 **可能** 会发生。
* 但假设是 CPU 对操作进行了重排序。在这种情况下，汇编代码中 **解锁** 操作是在 **加锁** 操作 **之前** 的。CPU 只是选择先尝试执行后面的加锁操作。
* 如果存在死锁，这个加锁操作会进入自旋状态（或者尝试休眠，不过这一点后面再详细说明）。最终，CPU 会执行解锁操作（该操作在汇编代码中位于加锁操作之前），这会解除潜在的死锁，使加锁操作能够成功执行。
* 但如果是睡眠锁（sleeplock）呢？在这种情况下，代码会尝试进入调度器，在那里最终会遇到一个内存屏障，该屏障会强制前面的解锁操作完成，同样能解除死锁。
  * 可能会出现睡眠-解锁（sleep-unlock）竞争的情况，但无论如何，加锁原语需要能够正确解决这类竞争。
---
* 在单处理器（UP）编译系统中，锁和信号量可能无法提供任何排序保证，因此在这种情况下，它们根本无法确保实现任何功能 —— 尤其是对于 I/O 访问而言 —— 除非与中断禁用操作结合使用。
* 另请参见“Inter-CPU acquiring barrier effects”部分。
* 例如，考虑以下代码：
  ```c
  *A = a;
  *B = b;
  ACQUIRE
  *C = c;
  *D = d;
  RELEASE
  *E = e;
  *F = f;
  ```
* 以下事件序列是可接受的：
  ```c
  ACQUIRE, {*F,*A}, *E, {*C,*D}, *B, RELEASE
  ```
* **注意**：`{*F,*A}` 表示组合访问。
* 但以下序列均不可接受：
  ```c
  {*F,*A}, *B,    ACQUIRE, *C, *D,        RELEASE, *E           //*F 不可跨越 ACQUIRE
  *A, *B, *C,     ACQUIRE, *D,            RELEASE, *E, *F       //*C 不可跨越 ACQUIRE
  *A, *B,         ACQUIRE, *C,            RELEASE, *D, *E, *F   //*D 不可跨越 RELEASE
  *B,             ACQUIRE, *C, *D,        RELEASE, {*F,*A}, *E  //*A 不可跨越 RELEASE
  ```

### 中断禁用函数
* 禁用中断的函数（相当于 `ACQUIRE` 操作）和启用中断的函数（相当于 `RELEASE` 操作）仅充当编译器屏障。
* 因此，若在此类场景中需要内存屏障或 I/O 屏障，则必须通过其他方式来提供。

### 睡眠和唤醒函数

* 在全局数据中标记的事件上进行休眠和唤醒，可以看作是两部分数据之间的交互：等待事件的任务的任务状态，以及用于指示事件的全局数据。
* 为确保这些操作按正确顺序发生，开始进入休眠过程的原语和发起唤醒的原语隐含着特定的屏障。
* 首先，休眠进程通常遵循如下事件序列：
  ```c
  for (;;) {
          set_current_state(TASK_UNINTERRUPTIBLE);
          if (event_indicated)
                  break;
          schedule();
  }
  ```
  * `set_current_state()` 在修改任务状态后，会自动插入一个通用内存屏障：
  ```c
  CPU 1
  ===============================
  set_current_state();
    smp_store_mb();
      STORE current->state
      <通用屏障>
  LOAD event_indicated
  ```
  * `set_current_state()` 可能会被以下函数包装：
  ```c
  prepare_to_wait();
  prepare_to_wait_exclusive();
  ```
  * 因此，这些函数在设置状态后也隐含一个通用内存屏障。
  * 上述完整序列有多种现成形式，它们都在正确的位置插入了内存屏障：
  ```c
  wait_event();
  wait_event_interruptible();
  wait_event_interruptible_exclusive();
  wait_event_interruptible_timeout();
  wait_event_killable();
  wait_event_timeout();
  wait_on_bit();
  wait_on_bit_lock();
  ```
* 其次，执行唤醒操作的代码通常遵循如下形式：
  ```c
  event_indicated = 1;
  wake_up(&event_wait_queue);
  ```
  * 或者：
  ```c
  event_indicated = 1;
  wake_up_process(event_daemon);
  ```
* `wake_up()` 如果确实唤醒了某个进程，会执行一个通用内存屏障。
  * 如果没有唤醒任何进程，则可能执行也可能不执行内存屏障，你不能对此抱有依赖。
  * 该屏障在访问任务状态之前发生，具体来说，它位于 *指示事件的存储操作（STORE）* 和 *设置 `TASK_RUNNING` 的存储操作（STORE）* 之间：
  ```c
  CPU 1（休眠进程）                 CPU 2（唤醒进程）
  =============================== ===============================
  set_current_state();            STORE event_indicated
    smp_store_mb();               wake_up();
      STORE current->state          ...
      <通用屏障>                     <通用屏障>
  LOAD event_indicated              if ((LOAD task->state) & TASK_NORMAL)
                                      STORE task->state
  ```
* 其中，“`task`”是被唤醒的线程，且等同于 CPU 1 的“`current`”。
* 需要重申的是，如果 `wake_up()` 确实唤醒了某个进程，那么它保证会执行一个通用内存屏障，否则不做此保证。
* 为理解这一点，考虑以下事件序列，其中 `X` 和 `Y` 的初始值均为 `0`：
  ```c
  CPU 1                           CPU 2
  =============================== ===============================
  X = 1;                          Y = 1;
  smp_mb();                       wake_up();
  LOAD Y                          LOAD X
  ```
* 如果发生了唤醒操作，两个 loads 操作（`LOAD`）中至少有一个必须看到 `1`。
  * **译注**：如果 CPU 2 先执行，CPU 1 的 `Y` 必然能看到 `1`；如果 CPU 1 先执行，CPU 2 的 `X` 必然看到 `1`
* 反之，如果没有发生唤醒操作，两个 loads 操作都可能看到 `0`。
  * **译注**：如果 CPU 2 在 CPU 1 的 `X = 1` 之前先执行，`LOAD X` 看到 `0`，而 `Y = 1` 写到 store buffer 里，对于 CPU 1 的 `LOAD Y` 不可见
* `wake_up_process()` **始终** 会执行一个通用内存屏障。该屏障同样在访问任务状态之前发生。
  * 具体来说，如果将前面代码片段中的 `wake_up()` 替换为 `wake_up_process()` 调用，那么两个 loads 操作中必有一个保证能看到 `1`。
* 可用的唤醒函数包括：
  ```c
  complete();
  wake_up();
  wake_up_all();
  wake_up_bit();
  wake_up_interruptible();
  wake_up_interruptible_all();
  wake_up_interruptible_nr();
  wake_up_interruptible_poll();
  wake_up_interruptible_sync();
  wake_up_interruptible_sync_poll();
  wake_up_locked();
  wake_up_locked_poll();
  wake_up_nr();
  wake_up_poll();
  wake_up_process();
  ```
* 就内存排序而言，这些函数都提供与 `wake_up()` 相同（或更强）的保证。
* **注意**：休眠进程和唤醒进程所隐含的内存屏障，**不能** 对唤醒操作之前的多个 stores 操作，与休眠进程调用 `set_current_state()` 之后对这些被存储值的 loads 操作进行排序。
* 例如，如果休眠进程执行：
  ```c
  set_current_state(TASK_INTERRUPTIBLE); //含有 smp_store_mb() 写屏障
  if (event_indicated)
      break;
  __set_current_state(TASK_RUNNING); //只含一个 WRITE_ONCE() 编译器屏障
  do_something(my_data);
  ```
  * 而唤醒进程执行：
  ```c
  my_data = value;
  event_indicated = 1;
  wake_up(&event_wait_queue);
  ```
  * 无法保证休眠进程会将 `event_indicated` 的变更视为发生在 `my_data` 的变更之后。
  * **译注**：比如唤醒进程的 `event_indicated = 1` 已经在 cache 可见，而 `my_data = value` 还在 store buffer，而休眠进程在 `if (event_indicated)` 和 `do_something(my_data)` 之间没有读屏障，导致休眠进程看到了 `event_indicated = 1` 然后对过期的 `do_something(my_data)`
  * 在这种情况下，双方代码必须在单独的数据访问之间插入自己的内存屏障。因此，上述休眠进程应改为：
  ```c
  set_current_state(TASK_INTERRUPTIBLE);
  if (event_indicated) {
      smp_rmb(); //保证如果看到 event_indicated == 1，必然看到 my_data 的新值
      do_something(my_data);
  }
  ```
  * 而唤醒进程应改为：
  ```c
  my_data = value;
  smp_wmb(); //排空 store buffer 里的值到 cache
  event_indicated = 1;
  wake_up(&event_wait_queue);
  ```

### 杂项函数
* 其他包含屏障的函数：
  * `schedule()` 和类似的函数隐私含有完全的内存屏障

## CPU 间获取屏障效应
* 在对称多处理器（SMP）系统上，加锁原语提供了一种更有效的屏障形式：在任何特定锁存在竞争的情况下，这种屏障会影响其他 CPU 上的内存访问顺序。

### `ACQUIRES` VS 内存访问
* 考虑以下情况：系统有一对自旋锁（`M`）和（`Q`），以及三个 CPU；此时可能会发生以下事件序列：
  ```c
  CPU 1                           CPU 2
  =============================== ===============================
  WRITE_ONCE(*A, a);              WRITE_ONCE(*E, e);
  ACQUIRE M                       ACQUIRE Q
  WRITE_ONCE(*B, b);              WRITE_ONCE(*F, f);
  WRITE_ONCE(*C, c);              WRITE_ONCE(*G, g);
  RELEASE M                       RELEASE Q
  WRITE_ONCE(*D, d);              WRITE_ONCE(*H, h);
  ```
* 那么，对于 CPU 3 而言，无法保证它看到的对 `*A` 到 `*H` 的访问顺序，唯一的限制来自不同 CPU 上各自锁所施加的约束。
* 例如，CPU 3 可能会看到如下顺序：
  ```c
  *E, ACQUIRE M, ACQUIRE Q, *G, *C, *F, *A, *B, RELEASE Q, *D, *H, RELEASE M
  ```
* 但它绝不会看到以下任何一种情况：
  - `*B`、`*C` 或 `*D` 出现在 `ACQUIRE M` 之前
  - `*A`、`*B` 或 `*C` 出现在 `RELEASE M` 之后
  - `*F`、`*G` 或 `*H` 出现在 `ACQUIRE Q` 之前
  - `*E`、`*F` 或 `*G` 出现在 `RELEASE Q` 之后

## 哪里需要内存屏障？

* 在正常操作中，内存操作重排序通常不会成为问题，因为即使在 SMP 内核中，单线程的线性代码看起来仍然能正常工作。然而，在以下四种情况下，重排序 **肯定可能** 会引发问题：
  - 处理器间交互
  - 原子操作
  - 设备访问
  - 中断

### 处理器间交互
* 当系统中存在多个处理器时，系统中的多个 CPU 可能会同时处理相同的数据集。
  * 这可能会导致同步问题，处理这类问题的常规方法是使用锁。
  * 然而，锁的开销相当大，因此在可能的情况下，不使用锁进行操作会更为理想。
  * 在这种情况下，影响多个 CPU 的操作必须经过精心排序，以防止出现故障。
* 例如，考虑读写信号量（R/W semaphore）的慢路径。在这里，等待进程会在信号量上排队，其实现方式是将进程栈的一部分链接到信号量的等待进程列表中：
  ```c
  struct rw_semaphore {
          ...
          spinlock_t lock;
          struct list_head waiters;
  };

  struct rwsem_waiter {
          struct list_head list;
          struct task_struct *task;
  };
  ```
* 要唤醒某个特定的等待进程，`up_read()` 或 `up_write()` 函数必须执行以下操作：
1. 从该等待进程的记录中读取下一个指针，以确定下一个等待进程记录的位置；
2. 读取指向等待进程任务结构的指针；
3. 清空任务指针，以此告知等待进程它已获得信号量；
4. 对该任务调用 `wake_up_process()`；
5. 释放对等待进程任务结构所持有的引用。
* 换句话说，它必须执行以下事件序列：
  ```c
  LOAD waiter->list.next;
  LOAD waiter->task;
  STORE waiter->task;
  CALL wakeup
  RELEASE task
  ```
* 如果这些步骤中的任何一步出现顺序错乱，整个操作都可能出现故障。
* 一旦等待进程将自身排入队列并释放信号量锁后，它就不会再获取该锁；相反，它只会等待自己的任务指针被清空后再继续执行。
  * **译注**：这里说的“信号量锁”应该指的是 `struct rw_semaphore.lock` 这个自旋锁
* 由于该记录（`struct rwsem_waiter waiter`）位于等待进程的栈上，这意味着如果在读取列表中的下一个指针（`LOAD waiter->list.next`）**之前** 就清空了任务指针（`STORE waiter->task`），那么另一个 CPU 可能会开始处理该等待进程，并且可能在 `up*()` 函数有机会读取下一个指针之前就破坏了该等待进程的栈。
* 不妨考虑上述事件序列可能发生的情况：
  ```c
  CPU 1                           CPU 2
  =============================== ===============================
                                  down_xxx()
                                  将等待进程入队 //waiter 实例在当前栈上分配
                                  睡眠
  up_yyy()
  LOAD waiter->task;
  STORE waiter->task;
                                  被其他事件唤醒
  <抢占>
                                  恢复处理 //比如条件是 waiter-> task 为空就恢复处理
                                  down_xxx() 返回 //栈指针回到调用前的位置，waiter 的内容还在
                                  调用 foo()
                                  foo() 破坏 *waiter
  </抢占>
  LOAD waiter->list.next;
  --- 发生错误 ---
  ```
* 这可以通过使用信号量锁来解决，但这样一来，`down_xxx()`函数在被唤醒后就不得不不必要地再次获取自旋锁。
* 处理此问题的方法是插入一个通用的 SMP 内存屏障：
  ```c
  LOAD waiter->list.next;
  LOAD waiter->task;
  smp_mb();
  STORE waiter->task;
  CALL wakeup
  RELEASE task
  ```
* 在这种情况下，该屏障保证，对于系统中的其他 CPU 而言，屏障之前的所有内存访问看起来都发生在屏障之后的所有内存访问之前。
  * 但它 **不** 保证在屏障指令本身完成时，屏障之前的所有内存访问都已完成。
* 在单处理器（UP）系统中 —— 这种情况原本不会成为问题 —— `smp_mb()`仅仅是一个编译器屏障，因此它只会确保编译器按正确顺序生成指令，而不会实际干预 CPU 的操作。由于只有一个 CPU，该 CPU 的依赖排序逻辑会处理其他所有事情。

### 原子操作

* 虽然从技术角度而言，原子操作属于处理器间交互需要考虑的范畴，但它们仍被单独列出，因为有些原子操作隐含完整的内存屏障，而有些则不。
* 不过，作为一个整体，原子操作在整个内核中被广泛依赖。
* 更多信息请参见 Documentation/atomic_t.txt。

### 设备访问

* 许多设备可以进行内存映射，因此对 CPU 而言，它们就像是一组内存位置。要控制这类设备，驱动程序通常必须按精确的顺序执行正确的内存访问操作。
* 然而，聪明的 CPU 或聪明的编译器可能会带来一个潜在问题：如果 CPU 或编译器认为重新排序、组合或合并访问操作更高效，那么驱动程序代码中精心安排顺序的访问操作可能无法按必要的顺序到达设备——这会导致设备出现故障。
* 在 Linux 内核中，I/O 操作应通过适当的访问函数（如 `inb()` 或 `writel()`）来完成，这些函数知道如何使此类访问以适当的顺序进行。
* 虽然在大多数情况下，这使得显式使用内存屏障变得不必要，但如果使用这些访问函数来访问具有 relaxed memory access 属性的 I/O 内存窗口，则需要 **强制** 内存屏障来确保顺序。
* 更多信息请参见 Documentation/driver-api/device-io.rst。

### 中断

* 驱动程序可能会被其自身的中断服务程序中断，因此驱动程序的这两部分可能会在尝试控制或访问设备时相互干扰。
* 这种情况至少可以部分缓解 —— 通过禁用本地中断（一种加锁形式），使所有关键操作都包含在驱动程序中禁用中断的部分内。
  * 当驱动程序的中断程序执行时，驱动程序的核心部分不会在同一个 CPU 上运行，而且在当前中断处理完成之前，不允许再次发生该中断，因此中断处理程序无需为此进行加锁防护。
* 然而，考虑这样一个驱动程序：它与一个配备（sports）地址寄存器和数据寄存器的以太网网卡进行通信。
  * 如果该驱动程序的核心部分在禁用中断的情况下与网卡通信，之后驱动程序的中断处理程序被调用：
  ```c
  本地中断禁用
  writew(ADDR, 3);
  writew(DATA, y);
  本地中断启用
  <中断发生>
  writew(ADDR, 4);
  q = readw(DATA);
  </中断结束>
  ```
* 如果排序规则足够宽松，对数据寄存器的存储操作可能会在第二次对地址寄存器的存储操作之后发生：
  ```c
  STORE *ADDR = 3，STORE *ADDR = 4，STORE *DATA = y，q = LOAD *DATA
  ```
* 如果排序规则宽松，除非使用隐式或显式屏障，否则必须假设在禁用中断的部分内执行的访问可能会泄露到该部分之外，并且可能与中断中执行的访问交错进行，反之亦然。
* 通常情况下，这不会成为问题，因为在这类部分中执行的 I/O 访问会包含对严格排序的 I/O 寄存器的同步 loads 操作，这些操作构成了隐式的 I/O 屏障。
* 中断程序与在不同 CPU 上运行且相互通信的两个程序之间，可能会出现类似的情况。如果这类情况有可能发生，就应使用禁用中断的锁来保证操作顺序。

### 内核 I/O 屏障效果

* 通过 I/O 访问与外设交互在很大程度上依赖于具体的架构和设备。
  * 因此，本质上不具备可移植性的驱动程序可能会依赖其目标系统的特定行为，以尽可能轻量的方式实现同步。
  * 对于旨在在多种架构和总线实现之间移植的驱动程序，内核提供了一系列具有不同程度排序保证的访问函数：

#### `readX()`、`writeX()`：

* `readX()` 和 `writeX()` 这两个 MMIO 访问函数接收一个指向被访问外设的指针作为 `__iomem *` 参数。
* 对于使用默认 I/O 属性映射的指针（例如由 `ioremap()` 返回的指针），其排序保证如下：
1. 对同一外设的所有 `readX()` 和 `writeX()` 访问都相对于彼此有序。这确保了同一 CPU 线程对特定设备的 MMIO 寄存器访问会按程序顺序到达。
2. 持有自旋锁的 CPU 线程执行的 `writeX()`，会先于另一个 CPU 线程在之后获取同一自旋锁后对同一外设执行的 `writeX()`。这确保了持有自旋锁时对特定设备的 MMIO 寄存器写入，会按与锁的获取顺序一致的顺序到达。
3. CPU 线程对外设执行的 `writeX()`，会先等待所有先前由该线程发出或传播到该线程的内存写入完成。这确保了当 CPU 写入 DMA 引擎的 MMIO 控制寄存器以触发传输时，CPU 对由 `dma_alloc_coherent()` 分配的出站 DMA 缓冲区的写入，能被 DMA 引擎看到。
4. CPU 线程从外设执行的 `readX()` 会完成后，同一线程对内存的所有后续读取才能开始。这确保了在从 DMA 引擎的 MMIO 状态寄存器读取以确认 DMA 传输已完成后，CPU 从由 `dma_alloc_coherent()` 分配的入站 DMA 缓冲区读取时，不会看到陈旧数据。
5. CPU 线程从外设执行的 `readX()` 会完成后，同一线程上的所有后续 `delay()` 循环才能开始执行。这确保了如果 CPU 对外设的两次 MMIO 寄存器写入中，第一次写入后立即用 `readX()` 读回，且在第二次 `writeX()` 之前调用 `udelay(1)`，那么这两次写入到达外设的时间至少间隔 `1` 微秒：
   ```c
   writel(42, DEVICE_REGISTER_0); // 到达设备...
   readl(DEVICE_REGISTER_0);
   udelay(1);
   writel(42, DEVICE_REGISTER_1); // ...至少比这次早 1 微秒。
   ```
* 使用非默认属性获取的 `__iomem` 指针（例如由 `ioremap_wc()` 返回的指针）的排序特性特定于底层架构，因此对于这些类型映射的访问，通常不能依赖上述保证。

#### `readX_relaxed()`、`writeX_relaxed()`：
* 这些函数与 `readX()` 和 `writeX()` 类似，但提供更弱的内存排序保证。
* 具体来说，它们不保证与锁、普通内存访问或 `delay()` 循环相关的排序（即上述第 2-5 点），但当操作使用默认I/O属性映射的 `__iomem` 指针时，仍保证相对于同一 CPU 线程对同一外设的其他访问是有序的。

#### `readsX()`、`writesX()`：
* `readsX()` 和 `writesX()` 这两个 MMIO 访问函数设计用于访问位于外设上的基于寄存器的内存映射 FIFO，这些 FIFO 无法执行 DMA。
* 因此，它们仅提供上述 `readX_relaxed()` 和 `writeX_relaxed()` 的排序保证。

#### `inX()`、`outX()`：
* `inX()` 和 `outX()` 访问函数用于访问传统的端口映射 I/O 外设，在某些架构（特别是 x86）上可能需要特殊指令。被访问外设的端口号作为参数传递。
* 由于许多 CPU 架构最终通过内部虚拟内存映射访问这些外设，因此 `inX()` 和 `outX()` 提供的可移植排序保证，分别与访问具有默认 I/O 属性的映射时 `readX()` 和 `writeX()` 提供的保证相同。
* 设备驱动程序可能期望 `outX()` 发出一个 non-posted 写入事务，在返回前等待来自 I/O 外设的完成响应。
  * 但并非所有架构都保证这一点，因此这不属于可移植排序语义的一部分。

#### `insX()`、`outsX()`：
* 与上述类似，`insX()` 和 `outsX()` 访问函数在访问具有默认 I/O 属性的映射时，分别提供与 `readsX()` 和 `writesX()` 相同的排序保证。

#### `ioreadX()`、`iowriteX()`：
* 这些函数会根据其实际执行的访问类型（无论是 `inX()`/`outX()` 还是 `readX()`/`writeX()`）进行适当的操作。
* 除了字符串访问函数（`insX()`、`outsX()`、`readsX()` 和 `writesX()`）之外，上述所有函数都假设底层外设是小端序的，因此会在大端序架构上执行字节交换操作。

## 假设最小执行排序模型

* 必须假设概念上的 CPU 是弱序的，但其会维持自身程序因果关系的表象。
  * 有些 CPU（如 i386 或 x86_64）比其他 CPU（如 powerpc 或 frv）具有更多约束，因此在架构特定代码之外，必须假设最宽松的情况（即 DEC Alpha）。
* 这意味着必须考虑到，CPU 会以其认为合适的任何顺序执行指令流，甚至可能并行执行
  * 前提是，如果指令流中的某条指令依赖于前面的某条指令，那么前面的那条指令必须足够完成，后面的指令才能继续执行；
    * 有些指令具有多种效果 —— 例如改变条件码、改变寄存器或改变内存 —— 不同的指令可能依赖于不同的效果。
  * 换句话说，前提是要维持因果关系的表象。
* CPU 也可能会丢弃最终没有任何效果的指令序列。
  * 例如，如果两条相邻的指令都将一个立即数加载到同一个寄存器中，那么第一条指令可能会被丢弃。
* 同样，必须假设编译器可能会以其认为合适的任何方式重新排序指令流，同样是以维持因果关系的表象为前提。

## CPU Cache 的效果
* Cache 的内存操作在整个系统中的感知方式，在一定程度上受到 CPU 与内存之间的 cache 以及维持系统状态一致性的内存 coherence（一致性）系统的影响。
* 就 CPU 通过 caches 与系统其他部分进行交互的方式而言，内存系统必须包含 CPU 的 caches，而内存屏障在很大程度上作用于 CPU 与其 cache 之间的接口
  * 内存屏障在逻辑上作用于下图中的虚线处：
  ```c
      <--- CPU --->         :       <----------- Memory ----------->
                            :
  +--------+    +--------+  :   +--------+    +-----------+
  |        |    |        |  :   |        |    |           |    +--------+
  |  CPU   |    | Memory |  :   | CPU    |    |           |    |        |
  |  Core  |--->| Access |----->| Cache  |<-->|           |    |        |
  |        |    | Queue  |  :   |        |    |           |--->| Memory |
  |        |    |        |  :   |        |    |           |    |        |
  +--------+    +--------+  :   +--------+    |           |    |        |
                            :                 | Cache     |    +--------+
                            :                 | Coherency |
                            :                 | Mechanism |    +--------+
  +--------+    +--------+  :   +--------+    |           |    |        |
  |        |    |        |  :   |        |    |           |    |        |
  |  CPU   |    | Memory |  :   | CPU    |    |           |--->| Device |
  |  Core  |--->| Access |----->| Cache  |<-->|           |    |        |
  |        |    | Queue  |  :   |        |    |           |    |        |
  |        |    |        |  :   |        |    |           |    +--------+
  +--------+    +--------+  :   +--------+    +-----------+
                            :
                            :
  ```
* 尽管任何特定的加载（load）或存储（store）操作实际上可能不会出现在发出该操作的 CPU 外部（因为它可能已在 CPU 自身的 cache 中得到满足），但就其他 CPU 而言，它仍然会表现得如同完整的内存访问已经发生
  * 这是由于 cache 一致性机制会在发生冲突时，将 cache line 迁移到进行访问的 CPU，并传播相关影响。
* CPU core 可以按照其认为合适的任何顺序执行指令，前提是要维持预期的程序因果关系表象。
  * 其中一些指令会生成 load 和 store 操作，这些操作随后会进入待执行的内存访问队列。
  * Core 可以按照其期望的任何顺序将这些操作放入队列，并继续执行，直到被迫等待某条指令完成。
* ==内存屏障所关注的是控制访问从 CPU 侧跨越到内存侧的顺序，以及系统中其他观察者感知这些影响发生的顺序。==
* ⚠️ 在单个 CPU 内部 **不需要** 内存屏障，因为 CPU 总是会将自己的 load 和 store 操作视为按程序顺序发生的。
* ⚠️ MMIO（内存映射 I/O）或其他设备访问可能会绕过 cache 系统。这取决于访问设备所通过的内存窗口的属性，以及/或者 CPU 可能具有的任何特殊设备通信指令的使用情况。

### Cache 一致性 VS DMA
* 并非所有系统都能维持 cache 与执行 DMA 的设备之间的一致性。
  * 在这种情况下，尝试执行 DMA 的设备可能会从 RAM 中获取陈旧数据，因为脏 cacheline 可能驻留在各个 CPU 的 cache 中，且尚未写回至 RAM。
  * 为解决此问题，内核的相应部分必须刷新每个 CPU 上 cache 的重叠部分（overlapping bits）（或许还需将其无效化）。
* 此外，设备 DMA 到 RAM 的数据可能会被以下情况覆盖：
  * 设备写入自身数据后，CPU cache 中的脏 cacheline 被写回至 RAM；
  * 或者，CPU cache 中存在的 cache lines 可能会掩盖 RAM 已更新这一事实，直到该 cacheline 从 CPU cache 中被丢弃并重新加载时，这种掩盖（obscure）才会消失。
  * 为应对此问题，内核的相应部分必须将每个 CPU 上 cache 的重叠的部分（overlapping bits）无效化。
* 有关缓存管理的更多信息，请参见 Documentation/core-api/cachetlb.rst。

### Cache 一致性 VS MMIO
* 内存映射 I/O 通常通过 CPU 内存空间中某个窗口内的内存位置进行，该窗口被分配的属性与常规指向 RAM 的窗口不同。
* 这些属性中通常包括这样一个特点：此类访问会完全绕过 cache，直接通往设备总线。这意味着，实际上，MMIO 访问可能会超过（先于）之前发出的对 cache 内存的访问。
* 在这种情况下，仅靠内存屏障是不够的，相反，如果 cache 内存写入操作与 MMIO 访问之间存在任何依赖关系，则必须在两者之间刷新 cache。

## CPU 要做的事情
* 程序员可能想当然地认为，CPU 会严格按照指定的顺序执行内存操作，例如，如果给 CPU 一段如下代码去执行：
  ```c
  a = READ_ONCE(*A);
  WRITE_ONCE(*B, b);
  c = READ_ONCE(*C);
  d = READ_ONCE(*D);
  WRITE_ONCE(*E, e);
  ```
* 他们会期望 CPU 在执行下一条指令之前，完成每条指令的内存操作，从而让系统中的外部观察者看到一个确定的操作序列：
  ```c
  LOAD *A, STORE *B, LOAD *C, LOAD *D, STORE *E。
  ```
* 当然，实际情况要复杂得多。对于许多 CPU 和编译器来说，上述假设并不成立，原因如下：
  - loads 通常需要立即完成以保证执行进度，而 stores 则往往可以延迟执行且不会有问题；
  - loads 可能会进行推测执行，若发现结果不必要，就会将其丢弃；
  - loads 的推测执行可能导致在预期的事件序列中，结果在错误的时间被获取；
  - 内存访问的顺序可能会被重新排列，以更好地利用 CPU 总线和 cache；
  - 当与能够对相邻位置进行批量访问的内存或 I/O 硬件交互时，loads 和 stores 可能会被合并以提高性能，从而降低事务建立成本（setup costs）（内存和 PCI 设备都可能支持这种操作）；
  - CPU 的数据 cache 可能会影响操作顺序，虽然 cache 一致性机制可能会缓解这一问题
    * 但一旦 store 操作实际写入 cache，也无法保证一致性管理会按顺序传播到其他 CPU。
* 因此，另一个 CPU（假设存在）从上述代码中实际观察到的序列可能是：
  ```c
  LOAD *A, ..., LOAD {*C,*D}, STORE *E, STORE *B
  ```
  * （其中“`LOAD {*C,*D}`”表示合并后的 load 操作）
* 不过，可以保证的是，CPU 会保持自我一致性：它会认为自己的访问是按正确顺序进行的，无需内存屏障。例如，对于以下代码：
  ```c
  U = READ_ONCE(*A);
  WRITE_ONCE(*A, V);
  WRITE_ONCE(*A, W);
  X = READ_ONCE(*A);
  WRITE_ONCE(*A, Y);
  Z = READ_ONCE(*A);
  ```
  * 在假设没有外部因素干扰的情况下，可以认为最终结果会呈现为：
  ```c
  U == *A 的原始值
  X == W
  Z == Y
  *A == Y
  ```
  * 上述代码可能会导致 CPU 生成完整的内存访问序列：
  ```c
  U=LOAD *A, STORE *A=V, STORE *A=W, X=LOAD *A, STORE *A=Y, Z=LOAD *A
  ```
* 且按此顺序执行，但在没有干预的情况下，只要程序对世界的认知保持一致，该序列中的元素几乎可以以任何组合方式被合并或丢弃。
* 需要注意的是，在上述示例中，`READ_ONCE()` 和 `WRITE_ONCE()` **并非** 可有可无，因为在某些架构中，特定 CPU 可能会对同一位置的连续 loads 操作进行重排序。
  * 在这类架构上，`READ_ONCE()` 和 `WRITE_ONCE()` 会采取必要措施防止这种情况，例如，在安腾（Itanium）架构中，`READ_ONCE()` 和 `WRITE_ONCE()` 所使用的 `volatile` 强制转换会导致 GCC 分别生成特殊的 `ld.acq` 和 `st.rel` 指令，以阻止此类重排序。
* 编译器甚至可能在 CPU 看到指令序列之前，就对其元素进行合并、丢弃或延迟处理。例如：
  ```c
  *A = V;
  *A = W;
  ```
  * 可能会被简化为：
  ```c
  *A = W;
  ```
* 因为如果没有写屏障或 `WRITE_ONCE()`，可以认为向 `*A` 存储 `V` 的效果会被覆盖。同样地：
  ```c
  *A = Y;
  Z = *A;
  ```
  * 在没有内存屏障或 `READ_ONCE()` 和 `WRITE_ONCE()` 的情况下，可能会被简化为：
  ```c
  *A = Y;
  Z = Y; //没有真正去 CPU 外部去 load
  ```
  * 而加载（`LOAD`）操作永远不会出现在 CPU 外部。

### 然后是 Alpha

* DEC Alpha CPU 是现有 CPU 中排序约束最宽松（relaxed）的一种。
* 不仅如此，某些版本的 Alpha CPU 还采用了分离式（split）数据 cache，这使得两个语义相关的 cache lines 可以在不同时间更新。
  * 这正是地址依赖屏障真正发挥作用的地方 —— 它会将两个 caches 与内存一致性系统同步（synchronises），从而确保指针变更和新数据的出现顺序是正确的。
* Alpha 定义了 Linux 内核的内存模型，不过自 v4.15 版本起，Linux 内核在 Alpha 架构的 `READ_ONCE()` 中添加了 `smp_mb()`，这大大降低了该 CPU 对内核内存模型的影响。

### 虚拟机 Guest

* 在虚拟机中运行的 guest 可能会受到 SMP（对称多处理器）效应的影响，即使 guest 本身是在未启用 SMP 支持的情况下编译的。
  * 这是在运行 UP（单处理器）内核时与 SMP host 交互产生的问题。针对这种情况，可以使用强制屏障，但这通常不是最优选择。
* 为了最优地处理这种情况，内核提供了底层的 `virt_mb()` 等宏。
  * 当启用 SMP 时，这些宏与 `smp_mb()` 等宏具有相同的效果，但在 SMP 系统和非 SMP 系统上会生成相同的代码。
  * 例如，虚拟机 guest 在与（可能是 SMP 的）host 进行同步时，应使用 `virt_mb()` 而非 `smp_mb()`。
    * **译注**：因为 `smp_mb()` 会因为 UP 而被忽略，然而事实是此处即使是 UP 也不应该忽略，`virt_mb()` 则不会因为 UP 而不去插入屏障
* 这些宏在其他所有方面都与对应的 `smp_mb()` 等宏等效，特别是它们不控制 MMIO（内存映射 I/O）效应；
  * 若要控制 MMIO 效应，请使用强制屏障（见 [CPU 内存屏障](#cpu-内存屏障)）。

## 示例使用
* 内存屏障可用于实现循环缓冲，而无需使用锁来实现生产者与消费者的串行化。详情请参见：
  * Documentation/core-api/circular-buffers.rst

## References
* [当我们在谈论cpu指令乱序的时候，究竟在谈论什么？](https://zhuanlan.zhihu.com/p/45808885)
* [Weak vs. Strong Memory Models](https://preshing.com/20120930/weak-vs-strong-memory-models/)
* [Memory Barriers Are Like Source Control Operations](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
* [x86 - Are memory barriers needed because of cpu out of order execution or because of cache consistency problem_ - Stack Overflow](https://stackoverflow.com/questions/63970362/are-memory-barriers-needed-because-of-cpu-out-of-order-execution-or-because-of-c)
* [multithreading - Does an x86 CPU reorder instructions_ - Stack Overflow](https://stackoverflow.com/questions/50307693/does-an-x86-cpu-reorder-instructions)
* [concurrency - x86 memory ordering_ Loads Reordered with Earlier Stores vs. Intra-Processor Forwarding - Stack Overflow](https://stackoverflow.com/questions/20907811/x86-memory-ordering-loads-reordered-with-earlier-stores-vs-intra-processor-for/20908626)
* [x86 - Globally Invisible load instructions - Stack Overflow](https://stackoverflow.com/questions/50609934/globally-invisible-load-instructions)
* [arm - How is load-_store reordering possible with in-order commit_ - Stack Overflow](https://stackoverflow.com/questions/52215031/how-is-load-store-reordering-possible-with-in-order-commit)
* [x86 - How do the store buffer and Line Fill Buffer interact with each other_ - Stack Overflow](https://stackoverflow.com/questions/61129773/how-do-the-store-buffer-and-line-fill-buffer-interact-with-each-other)
* [intel - Where is the Write-Combining Buffer located_ x86 - Stack Overflow](https://stackoverflow.com/questions/49959963/where-is-the-write-combining-buffer-located-x86/49961612#49961612)
* [MDS Attacks_ Microarchitectural Data Sampling](https://mdsattacks.com/)
* [C语言实现泛型 C11关键字_Generic](https://blog.csdn.net/qq_37151416/article/details/113203701)
