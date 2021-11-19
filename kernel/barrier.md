
* 内存屏障的使用应该成对，例如`smp_wmb`必须配对`smp_rmb`或者`smp_mb`，单独使用`smp_wmb`是达不到顺序效果的。同样`smp_rmb`必须配对使用`smp_wmb`或者`smp_mb`。

* 考虑以下观点

> * store buffer 的作用是为了解决`store`操作执行 invalid 操作需要等待其它 CPU 响应 invalidate response 造成的 cpu stall 问题。
> * 所有架构的 store buffer 都是 FIFO，但是 FIFO 并不能保证`store`操作的顺序。
>   * 如果 CPU 是某内存地址对应 cache line 的 owner(modified/exclusive 状态)，CPU 在执行该地址的`store`操作时不需要 invalid 其它 CPU 的 cache line，因此不需要将该操作放入 store buffer 中直接更新 cache line 就行了，这样就导致了该 cache line 先于 store buffer 中的地址可见，可能会造成乱序。
>   * wb 的作用是不管当前CPU是否是 cache line 的 owner，在执行对应物理地址的`store`操作时将其排在 store buffer 中，由该 buffer 的 FIFO 性质来保证写入的顺序。

# References
* [当我们在谈论cpu指令乱序的时候，究竟在谈论什么？](https://zhuanlan.zhihu.com/p/45808885)
