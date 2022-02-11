# 内存屏障

* **barrier**：可能重新排序和写的处理器提供了机器指令来确保顺序要求，这些确保顺序的指令称为 *屏障*。
* `volatile`关键字可以 **防止编译器的优化**，主要体现在以下三个方面：
  * 不会从寄存器中读取变量，而是重新从内存中取。
  * 不要对变量进行各种激进的优化，甚至将变量直接消除，保证程序员写在代码中的指令，一定会被执行。
  * `volatile`变量间的操作，不会被编译器交换顺序。
* `volatile`防止编译器优化的目的是确保数据能够被按顺序地无误地写入到内存，而不会因为编译器的原因产生偏差，设想某个变量其实对应的是通过`ioremap()`到内存的某个硬件寄存器，或者 x86 里变量要通过`in`或`out`指令访问的 I/O 空间。
* 即便是用`volatile`声明来防止编译器对变量操作的重新排序，仍然有可能会因为处理器优化 pipeline 的原因对指令重新排序，因此内存屏障仍然是很重要的。
* 对于 SMP，`smp_rmb()` , `smp_wmb()` , `smp_mb()` 和 `smp_read_barrier_depends()`是常用的 *内存屏障*（如 x86 的`mfence`指令）；对于 UP，则被定义为 *编译器屏障*，如 gcc 的`#define barrier() __asm__ __volatile__("": : :"memory")`。
* *内存屏障* 可以完成 *编译器屏障* 的功能，但编译器屏障要比处理器屏障 **轻量**（它实际上是轻快的）得多。
* 实际上，编译器屏障几乎是空闲的，因为它只防止编译器可能的重排指令。
* 不同体系架构，屏障的实际效果差别很大。如 x86 不会打乱存储，`wmb()`就什么也不做。
* 为最坏的情况（即排序能力最弱的处理器）使用恰当的内存屏障，代码才能在编译时执行针对体系结构的优化。

#### Memory and Compiler Barrier Methods

Barrier | Description
---|---
rmb() | Prevents loads from being reordered across the barrier
read_barrier_depends() | Prevents data-dependent loads from being reordered across the barrier
wmb() | Prevents stores from being reordered across thebarrier
mb() | Prevents load or stores from being reordered across the barrier
smp_rmb() | Provides an rmb() on SMP, and on UP provides a barrier()
smp_read_barrier_depends() | Provides a read_barrier_depends() on SMP, and provides a barrier() on UP
smp_wmb() | Provides a wmb() on SMP, and provides a barrier() on UP
smp_mb() | Provides an mb() on SMP, and provides a barrier() on UP
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
* 考虑以下观点

> * store buffer 的作用是为了解决`store`操作执行 invalid 操作需要等待其它 CPU 响应 invalidate response 造成的 cpu stall 问题。
> * 所有架构的 store buffer 都是 FIFO，但是 FIFO 并不能保证`store`操作的顺序。
>   * 如果 CPU 是某内存地址对应 cache line 的 owner(modified/exclusive 状态)，CPU 在执行该地址的`store`操作时不需要 invalid 其它 CPU 的 cache line，因此不需要将该操作放入 store buffer 中直接更新 cache line 就行了，这样就导致了该 cache line 先于 store buffer 中的地址可见，可能会造成乱序。
>   * wb 的作用是不管当前CPU是否是 cache line 的 owner，在执行对应物理地址的`store`操作时将其排在 store buffer 中，由该 buffer 的 FIFO 性质来保证写入的顺序。

## 保障
### 对一个 CPU 可以期望的最小保障
* 在任何给定的 CPU，对它自己的有依赖的内存访问将会按顺序发送
  ```c
  Q = READ_ONCE(P); D = READ_ONCE(*Q);
  ```
* 在同一 CPU 上，对同一位置的 load 和 store 的顺序在该 CPU 上是顺序的
  ```c
  a = READ_ONCE(*X); WRITE_ONCE(*X, b);
  ```
### “一定”或“一定不要”假设的事情
* 对于未被`READ_ONCE()`和`WRITE_ONCE()`保护的内存引用，**一定不要** 假设编译器会帮你做保护。缺乏他们，编译器有权利做任何它认为“有创造性”的变换。
* **一定不要** 假设无关联的 load 和 store 会按给定的顺序发送。
* **一定要** 假设对同一位置的内存访问会被合并或丢弃
  * **例一**
    ```c
    X = *A; Y = *(A + 4);
    ```
    可能会按以下次序之一运行：
    ```c
    X = LOAD *A; Y = LOAD *(A + 4);
    Y = LOAD *(A + 4); X = LOAD *A;
    {X, Y} = LOAD {*A, *(A + 4) };
    ```
  * **例二**
    ```c
    *A = X; *(A + 4) = Y;
    ```
    可能会按以下次序之一运行：
    ```c
    STORE *A = X; STORE *(A + 4) = Y;
    STORE *(A + 4) = Y; STORE *A = X;
    STORE {*A, *(A + 4) } = {X, Y};
    ```
### 无保障的事情
* 这些保证不适用于位域，因为编译器经常生成非原子读取-修改-写入序列代码来修改他们，因此，不要尝试使用位域来同步并行算法。 
* 即使在位域受锁保护的情况下，给定位域中的所有字段也必须由一个锁保护。如果给定位域中的两个字段由不同的锁保护，编译器的非原子读-修改-写序列可能会导致对一个字段的更新破坏相邻字段的值。
* 这些保证仅适用于正确对齐和大小标准的变量。“适当大小”目前是指与`char`、`short`、`int`和`long`大小相同的变量。“正确对齐”是指自然对齐，因此`char`没有约束，`short`是两字节对齐，`int`是四字节对齐，`long`在 32 位和 64 位系统上分别是四字节或八字节对齐。请注意，这些保证已被引入 C11 标准，因此在使用旧的 C11 之前的编译器（例如 gcc 4.6）时要小心。包含此保证的标准部分是第 3.14 节，其中定义“内存位置”如下：
  * 内存位置：一个标量类型的对象或相邻位域的最大序列都有非零的宽度
  * NOTE 1: 两个执行线程可以更新和访问单独的内存位置，而不会相互干扰
  * NOTE 2: 一个位域和一个相邻的非位域成员位于不同的内存位置。如果两个位域，一个在嵌套结构声明中声明而另一个没有，或者如果两者由零长度位域声明分隔，或者如果它们由非位域成员声明，这同样适用。无论这些中间位域的大小恰好是什么，如果在它们之间声明的所有成员也是位域，则同时更新同一结构中的两个位域是不安全的

## References
* [当我们在谈论cpu指令乱序的时候，究竟在谈论什么？](https://zhuanlan.zhihu.com/p/45808885)
