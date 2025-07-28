# 微架构学习

[DecodeZ](https://decodezp.github.io) 的系列文章《Skylake微架构(Microarchitecture)剖析》言简意赅，是学习微架构很好的起步资料

* [Skylake (Client) Microarchitecture](https://en.wikichip.org/w/images/7/7e/skylake_block_diagram.svg)
![Skylake (Client) Microarchitecture](../kernel/pic/skylake_block_diagram.svg)    
* [Skylake (Server) Microarchitecture](https://en.wikichip.org/w/images/e/ee/skylake_server_block_diagram.svg)
![Skylake (Server) Microarchitecture](../kernel/pic/skylake_server_block_diagram.svg)

## 流水线总览
![common arch](pic/428px-intel_common_arch_post_ucache.svg.png)
* `Allocation Queue`，它就还有一个名字叫做 `Instruction Decode Queue`，同时它还有可能被叫做 `IDQ` 或 `AQ`

### 前端（Front-End）
* 上图中蓝色部分就代表流水线的前端。它的主要作用就是获取指令、解码（Decode）指令。
* 为了最大限度的发挥 CPU 的能力，前端就需要尽可能高效率地把程序指令输送给后端。这里就面临两个挑战：
  * 如何更快更准确地取得要执行的指令
  * 如何将取得的指令更快地解码为 **微指令（micro-ops/uops）**
* 有了更多的微指令输送给后端（执行单元），后端的工作量才能饱和。而前端的所有组件和机制，都是围绕这两个挑战进行的。

### 后端（Back-End）
* 上图中红色的部分就代表流水线的后端。一般来讲绿色的部分是存储子系统，虽然与后端交互，但严格讲不算在后端里面。
* 后端的主要任务就是执行前端送过来的指令。和前端类似，后端除了“来料加工”之外，也有它自己需要面对的挑战：
  * 如何提高指令的并行程度
  * 如何充分利用已有的 CPU 能力

## 前端
* 处理器在前端这一部分的时候还是顺序（in-order）处理的，主要是也确实没什么乱序的空间。
  * 虽然说是顺序，但前端因为贴近业务，所以受人写的代码的影响也比较大。
  * 如果仅仅只是“取指令->解码”，恐怕需要写程序的人是个非常聪明的程序员。前端很多组件的工作其实都是在填程序员的坑，这也是作者比较心疼前端的地方。

### Fetch
![Front-end-fetch](pic/300px-skylake_fetch.svg.webp)

* 前端的任务，首先是从内存中取得指令。同读取数据类似，CPU 通过查询页表获得指令所在的内存地址，同时把指令塞到 CPU 的 L1 指令缓存里。
* 具体要把哪个地址上的指令数据送到 L1 iCache 里，这是 **分支预测器（Branch predictor）** 的工作。
  * 作为 CPU 的核心技术，Intel 并没有透露太多信息，我们这里也只好一笔带过。不过它的细节也许很复杂，但它的脾气很好掌握：和我们很多人不喜欢自己的工作一样，它的工作就是处理分支，但它最不喜欢分支。
* 在 Skylake 架构里，L1 iCache 大小为 `32KB`，组织形式为 8-way set associative（关于 CPU 缓存组织形式的讲解可以参照[这篇](https://decodezp.github.io/2018/11/25/quickwords2-cacheassociativity/)）
* 每个 cycle 可以取 `16 Byte` 长度（**fetch window**）的指令。
  * 如果你开了 Hyper-thread，那么同一个物理核上的两个逻辑核均分这个 fetch window，每个 cycle 各占用一次。
* 在 L1 iCache 里的指令还都是变长的 x86 macro-ops，也就是我们看到的那些编译之后的汇编指令。如果熟悉这些指令的话，就会知道这些指令的长度（就是那些二进制数字）都不一样，同时一条指令有时可以由好几个操作组成。
* 这种指令对 CPU 的执行单元来说是很不友好的，同时如果想要通过乱序执行提高指令的并行度，减小指令的粒度也是必须的步骤。因此需要把这些 `marco-ops` “**解码**”为 “`micro-ops`”。
* 当然具体的解码工作还在后面。从 L1 iCache 中取得指令数据后，首先要进入“**预解码**”阶段，
  * 在这里需要识别出在一个 fetch window 中取得的这 `16` 个 Byte 的数据里面有多少个指令。
  * 除此之外，还需要对一些特殊指令，比如分支转跳打上一些标记。
* 但因为指令变长的原因，`16` 个 Byte 往往并不对应固定的指令数，还有可能最后一个指令有一半在这 `16 Byte` 里，另一边还在后面。
* 另外就是 pre-decode 在一个 cycle 最多识别出 `6` 个指令，或者这 `16 Byte` 的数据都解析完。
  * 如果这 `16` 个 Byte 里包含有 `7` 个指令，那么第一个 cycle 识别出前 `6` 个之后，还需要第二个 cycle 识别最后一个，然后才能再读取后面 `16 Byte`。
  * 那么 pre-decode 的效率就变成了 `3.5 instruction / cycle`，比最理想的情况 `6 instruction / cycle` 降低了 `41%`，现实就是这么残酷。
* 经过 pre-decode 之后，才真正从 `16 Byte` 的二进制数据中识别出了指令，这些指令下一步要被塞到一个队列里（Instruction Queue）看看有没有什么能被优化的地方。
  * 一个最常见的优化方式就是 **macro-op fusion**，就是把两个相邻的，且能被一个指令表示的指令，用那一个指令替换掉。比如：
```s
cmp eax, [mem]
jne loop
```
直接换成
```s
cmpjne eax, [mem], loop
```
* 当然既然决定这么替换，新指令对流水线的开销肯定小于被替换的两个指令之和。如此可以减轻一部分后端执行单元的工作负荷和资源开销。
* OK, 在取得了指令数据，识别出了数据中的指令，并对指令做了一些优化合并之后，就该开始正儿八经地解码了。

### 解码

![Front-end Decode](pic/425px-skylake_decode.svg.webp)

* 在拿到了经过“预解码”的 `macro-ops` 之后，开始正式进入解码过程。`marco-ops` 进入 Instruction Decode 组件解码，最终的输出为定长的 `micro-ops`。
* Insturction Decode 组件也有入口带宽限制，每个 cycle 最多取
  * `3` 个 unfused 指令 + `2` 个 fused 指令，
  * 或者 `5` 个 unfused 指令（这里指 macro ops）。
* 所以说 fused 多了也不好，一个 cycle 最多取两个。
* 同时如果开了 Hyper Thread，则两个 Thread 按 cycle 交替使用 Instruction Decode。
* 在 Instruction Decode 组件里面的就是各个具体的 Decoder。Decoder类型可以分类两类，
  * 一类是 Simple Decoder，
  * 一类是 Complex Decoder。
* 顾名思义，**Simple Decoder** 处理的是解码之后的输出为 `1` 个 `fused-uop` 的指令；**Complex Decoder** 处理的是解码之后的输出为 `1` 个至 `4` 个 `fused-uop` 的指令。

#### Fused-uop
* **注意**：这里说的是 fused-`uop`，不是 fused-`marco`。在这里所有输出的 uop 都是做过 fused 处理的，目的是减少后续资源的占用。
* 但这里有一个比较容易混淆的概念，就是 `fused-uop` 并非专指那些两个 uop 合并之后生成的“合并 uop”，而是指所有经过了 uop fusion 处理的 uop
  * 有些指令可能两个 uop 变一个，但也有一些是一个还是一个，即便如此，输出的那一个也叫 `fused-uop`
* 为了进一步澄清这个概念，我们稍微需要涉及一点后端的概念。
  * 在前端这里，生成 `fused-uop` 的部分还属于 CPU 流水线中的 **uops fused domain**，
  * 而在后端需要将指令发射到执行单元去的时候，是不能执行 `fused uop` 的，所以 `fused uop` 还需要再分解为 `unfused uop` 才可以执行，这一部分就属于 CPU 流水线中的 **uops unfused domain**。
* 有了这些概念之后，我们可以看一下 [Instruction Tables.pdf](https://www.agner.org/optimize/instruction_tables.pdf) 这份文档。
  * 在 P244 中有对 skylake 指令的说明，上面有对一些概念的解释，下面是一张表格：

![Fusion uop vs Unfused uop](pic/FjnbCV.png)

* 在这张表格里是最常见的 `mov` 命令的说明。但因为操作数（operands）的不同在真正执行的时候也会有细节上的差别。
  * 第一行中的 `mov` 的两个操作数一个是 `register`，另外一个是一个立即数。在 uops fused domain 和 uops unfused domain 两栏中的计数都是 `1`。这种指令也算在 uops fused domain 经过了 fusion 处理。只不过其实前后没什么区别。
  * 但如果我们看一下所有在 uops unfused domain 里计数为 `2` 的 `mov` 指令，它们在 uops fused domain 中的计数都是 `1`。这种 `mov` 指令就是真正做过`2` 条 uop 合并的 `mov` 指令。
* Skylake 有 `4` 个 Simple Decoder 和 `1` 个 Complex Decoder。但从表里我们可以看到 uops fused domain 计数为 `1`，也就是可以被 Simple Decoder 处理的指令在所有指令中所占的比例似乎并没有达到 `4/5` 那么高。
* 这里需要说明的是，输出大于 `4` 个 uop 的指令，既不由 Simple Decoder 处理，也不由 Complex Decoder 处理，而是直接去查 **Microcode Sequencer（MS）**，这是一块类似于缓存的 ROM。
* Complex Decoder 的数量始终为 `1` 的原因是：
  * Complex Decoder 解码出来的 uop 都很难在执行时进行并行化处理，
  * 同时 Complex Decoder 每多解码一个 uop，就要有一个 Simple Decoder 处于不能工作的状态。
* 对 CPU 来说，它最希望的就是它要做的工作，它需要的数据，它要执行的指令，都已经在一块缓存里准备就绪了。这是 CPU 上班摸鱼的主要方法，但摸出了风格，摸出了水平。

### MSROM

![MSROM](pic/425px-skylake_msrom.svg.webp)

* MSROM（Micro-code sequencer ROM）就是上一节提到的专门处理输出大于 `4` 个 uop 的那块类似缓存的 ROM。
* 很多文档里面也直接将其称为 `MS`，具体叫什么多需要结合上下文语境，知道是一回事就好了。
* 在 Instruction Decoder 收到一个输出要大于 `4` 个 `uop` 的指令之后，它就会将请求转发给 MSROM。
* MSROM 虽然是专门解码/查询大于 `4` 个 `uop` 的指令的组件，但它最大的传输效率是 `4 uop/cycle`。同时在它工作的时候，所有的 Instruction Decoder 都要处于 Disable 的状态。因此虽然它的工作不太需要“动脑子”，但却仍要尽量避免。

### Stack Engine
* **Stack Engine** 是专门处理栈操作指令的专用组件。类似 `push`、`pop`、`call`、`ret` 这样的指令都算栈操作指令。
* Stack Engine 不算什么新鲜的黑科技，自从 Pentium M 时代起就已经出现在 Intel 的 CPU 中。
* 它的主要目的是 **避免栈操作指令对后端资源的占用，从而为其他计算任务提供出更多的资源**。为此，Stack Engine 提供 **栈操作指令专用的加法器** 和 *其他所需的逻辑* 完成这一任务。
* Stack Engine 在 Instruction Decoder 之后，监控所有流出的 uop，并且从中提取出栈操作指令，进而直接执行，从而减轻栈操作指令对后端资源的占用。
* 这也可能是为什么有些时候 `inline` 的函数性能还不如不 `inline` 的原因吧:D（不负责任猜测）

### Decoded Stream Buffer（DSB）

![Decoded Stream Buffer](pic/400px-skylake_ucache.svg.webp)

#### 别名
* 像 DSB 这种组件，首先要说明的就是它也叫 uop cache 或 decoded iCache
#### 作用
* 无论是用 Instruction Decoder 还是用 MSROM，终究还是要做一次“解码”的操作。但同所有 Cache 加速的原理一样，如果能把解码之后的结果（uop）存下来，下次再出现的时候直接使用，那么就可以显著提高解码速度，DSB 就是这个目的。
#### 参数
* DSB 的组织形式是 `32` 个 set，每个 set 有 `8` 条 cache line，每条 cache line 最多保存 `6` 个 uop。
* 每次 cache hit 可以传输最大 6 个 `uop/cycle`，这 `6` 个 uop 最大可以对应到 `64 byte` 的前端 fetch window size，并且完全不需要任何 Instruction decoder 参与，也没有繁琐的解码过程。
* 在实际应用中，DSB 的 cache hit rate 在 `80%` 或以上。
#### 与 `iCache` 的关系
* CPU 的 iCache 一般存储的是最原始的从内存里读进来的程序的汇编指令（marco instruction）。
* 而 DSB 或者 uop cache 虽然也是存 instruction 的cache，但如前所述，它存的是已经解码好的 uop，所以这玩意有时候又被称为“**decoded iCache**”。
  * 当然了，这些 uop 都是 CPU 的 iCache 中的指令解码之后得到的。
#### 与 MSROM 的关系
* 输出大于 `4` 个 uop 的指令依然只能由 MSROM 解码。
* DSB 保存的也是那些小于等于 `4` 个 uop 指令的 uop。

#### MITE Path 和 DSB Path
* 这两个概念主要用于区分最终需要执行的 uop 是通过什么方式来的。
* 在上一节 Decoded Stream Buffer 之前的所有内容，都算是 **MITE Path**。
  * MITE 是 Micro-instruction Translation Engine 的缩写，同时它在有些文档里也被称作 legacy decode pipeline 或 legacy path。
  * 这条线路上过来的 uop 都是从 marco instruction 一步一步解码来的。
* **DSB path** 就是直接从 DSB 那条道上过来的 uop。
* 当 CPU 需要在 MITE Path、DSB Path 以及 MSROM 之间切换（switch）以便取得所需的 uop 时，需要花费一定的 CPU cycle 完成这一工作。

## References
- [几句话说清楚9：Skylake微架构(Microarchitecture)剖析(1)](https://decodezp.github.io/2019/01/07/quickwords9-skylake-pipeline-1/)
- [几句话说清楚10：Skylake微架构(Microarchitecture)剖析(2) ](https://decodezp.github.io/2019/01/10/quickwords10-skylake-pipeline-2/)
- [几句话说清楚11：Skylake微架构(Microarchitecture)剖析(3)](https://decodezp.github.io/2019/01/12/quickwords11-skylake-pipeline-3/)
- [几句话说清楚12：Skylake微架构(Microarchitecture)剖析(4)](https://decodezp.github.io/2019/01/20/quickwords12-skylake-pipeline-4/)