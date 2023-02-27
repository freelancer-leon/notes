# NOP
* 清除由上一个算术逻辑指令设置的 flag 位
* 1 字节 `nop` 指令是 `XCHG (E)AX, (E)AX` 的别名助记词
* 2~9 字节的 `nop` 指令见 SDM Vol.2
* 从软件上来说，`nop` 的两种应用场景
  * 制造错误
  * 辅助编程
* 硬件上来看，用 `nop` 的目的主要是 *占位* 和 *对齐*
  * 占位是为了方便软件编写
  * 对齐是为了提升译码效率（部分 CPU）
* 但是 `nop` 本身也需要被执行，因此会降低 CPU 的有效执行效率
  * 设计上, `nop` 作为一条有效的指令是需要完成整个 CPU 流水线过程的。它需要 Decode，Dispatch，Execution，Retire。执行它的单元一般是 ALU, 但是它不改变任何 ARF 的值. 所以 `nop` 的出现会影响 CPU 效率
* 但是现代 CPU 已经有良好的乱序处理能力，并不需要人为使用 `nop` 来打断流水线

# References
- [NOP指令会打断CPU流水线吗？](https://www.zhihu.com/question/21122634/answer/22763700)