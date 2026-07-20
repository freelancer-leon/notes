# 负载量化 PELT (Per-Entity Load Tracking)

* PELT 的核心思想是为每个调度实体（`sched_entity`）独立追踪其负载贡献，而不是只看整个 CPU 的运行队列长度
* 这个算法在 Linux 3.8 版本中引入，其代码主要位于 `kernel/sched/pelt.c` 文件中

## 基础理论

### 核心基石：指数加权移动平均数 (EWMA)
* PELT 使用 **指数加权移动平均数 (EWMA)** 来跟踪负载。其核心数学模型如下：
  * **衰减周期**：每个周期为 **1024µs** (约 `1ms`)
  * **衰减系数 (y)**：满足 `y^32 = 0.5`
    * 这意味着，一个负载贡献经过 **32 个周期（约 `32ms`）** 后，其权重会衰减为原来的一半
    * 近期（`32ms` 内）的负载贡献占了一半权重，而更久远的历史贡献占了另一半
* 其数学表达式可以理解为：
  * **加权和**：`ewma_sum(u) := u_0 + u_1*y + u_2*y^2 + …`
  * **平均值**：`ewma(u) = ewma_sum(u) / ewma_sum(1)`

* **关键特性：可组合性 (Composability)**
  * 这个几何级数的一个重要特性是**可组合的**，即 `ewma(A) + ewma(B) = ewma(A+B)`
  * 这个特性至关重要，它使得当任务在不同 CPU 间迁移时，可以方便地通过加减操作来更新源 CPU 和目标 CPU 的负载总和

### 追踪的核心指标
* PELT 主要追踪两个核心指标：
1. **`running` (运行时间)**：实体实际在 CPU 上执行的时间
2. **`runnable` (可运行时间)**：实体处于可运行状态（在运行队列中等待或正在运行）的时间
* **区别**：当系统只有一个任务时，这两个值相同。但当存在 CPU 竞争时，`running` 会减少（反映每个任务分到的CPU时间片），而 `runnable` 会增加（反映排队等待的时间）

### 关键数据结构：`struct sched_avg`
* 每个调度实体（`sched_entity`）和 CPU 运行队列（`cfs_rq`）都内嵌了一个 `struct sched_avg` 结构来存储 PELT 相关的统计信息。其关键字段包括：
  * `last_update_time`：上次更新时间戳，用于计算时间差（delta）
  * `load_sum`：负载贡献的累加和（加权总和）
  * `util_sum`：利用率贡献的累加和
  * `period_contrib`：当前不完整周期（小于 `1024us`）内已累计的时间
  * `load_avg`：**平均负载**。它是 `load_sum` 除以一个固定 divider 后的结果。这个值同时考虑了 `runnable` 和 `blocked` 状态的任务，主要用于负载均衡决策
  * `runnable_avg`：**平均可运行负载**。只考虑 `runnable` 状态的任务
  * `util_avg`：**平均利用率**。它是 `util_sum` 除以 divider 后的结果。这个值主要反映任务对 CPU 算力的真实需求，与任务权重无关，常用于 **EAS (Energy-Aware Scheduling)** 和 **schedutil** 调频 governor

### 核心实现：衰减与累积
* PELT 的实现精髓在于如何高效地计算历史和当前负载的加权和

#### 1. 衰减计算：`decay_load()`
* `decay_load(val, n)` 函数实现了 `val * y^n` 的快速计算。
* 它的优化策略是查表法：
  * 因为 `y^32 = 0.5`，所以 `y^n` 可以分解为 `(1/2)^(n/32) * y^(n%32)`
  * 对于 `y^(n%32)` 这部分，内核预计算了一个包含 32 个值的查找表 `runnable_avg_yN_inv`
* 后面会展开分析

#### 2. 累积更新：`___update_load_avg()`
* 这是 PELT 最核心的函数，它负责将新过去的时间（`delta`）累积到 `sched_avg` 结构中
* 它将 `delta` 时间分成了三个部分来处理：
  * `d1`：上个不完整周期的剩余部分
  * `d2`：中间完整的周期数
  * `d3`：当前不完整周期中已经过去的部分
* 其累积公式为：
```c
u' = (u + d1) * y^p + 1024 * (y^1 + y^2 + ... + y^(p-1)) + d3 * y^0
```
* 其中，`u` 是历史累积值，`p` 是 `d2` 包含的完整周期数
* `__accumulate_pelt_segments(periods, d1, d3)` 函数就是用来计算上述公式中 `d1`, `d2`, `d3` 这三部分贡献总和的

### 何时触发更新：`update_load_avg()`

* `update_load_avg()` 是触发 PELT 更新计算的入口函数，它在以下关键事件中被调用：
  * **周期性调度**：每次 `scheduler_tick()` 时钟中断时
  * **任务状态变化**：
    * 任务入队（`enqueue_task`）时
    * 任务出队（`dequeue_task`）时
  * **任务创建与退出**
  * **负载均衡**：在进行任务迁移时，需要更新相关 CPU 和任务的负载信息

### 进阶概念：频率/CPU 不变性 (Frequency/CPU Invariance)
* 为了让负载指标在不同频率和不同性能的 CPU 核心之间具有可比性，PELT 引入了“不变性”的概念
* **问题**：在 1GHz 下运行 50% 的时间，与在 2GHz 下运行 50% 的时间，完成的“工作量”是不同的
* **解决方案**：通过引入两个缩放因子，对时间 delta 进行修正：
  * **DVFS 比率 (`r_dvfs`)**：*当前运行频率* 与 *最高频率* 的比值
  * **微架构比率 (`r_cpu`)**：*当前 CPU 性能* 与 *系统中最高性能 CPU* 的比值
* **总比率**：`r_tot = r_dvfs * r_cpu`
* 通过应用 `r_tot` 缩放时间，PELT 计算出的 `running` 和 `runnable` 指标就变得与频率和 CPU 类型无关了，可以在不同 CPU 之间公平地比较和转移

### 阻塞任务的处理
* PELT 一个重要的设计是，**阻塞 (blocked) 任务依然会为负载统计做出贡献**
  * 当一个任务因等待 I/O 而睡眠时，它的负载贡献并不会立即消失，而是会按照衰减系数随时间逐渐衰减
  * 这能更准确地反映系统即将面临的负载，因为任务可能在不久后醒来继续运行

### 总结
* PELT 算法的实现细节可归纳为：
1.  **基于 EWMA**：采用指数加权移动平均，使得负载贡献随时间指数级衰减
2.  **细粒度追踪**：为每个调度实体（任务或任务组）独立计算负载
3.  **高效实现**：通过查表法和分段累积，避免了复杂的浮点运算
4.  **关键指标**：提供了 `load_avg`（负载均衡）、`runnable_avg` 和 `util_avg`（频率缩放和EAS）等不同维度的指标
5.  **平台无关**：通过频率/CPU不变性设计，使得指标在不同硬件条件下具有可比性
* PELT 算法的精妙之处在于，它用相对较小的计算开销，为调度器提供了精确且可比较的负载信息，是整个调度器实现智能负载均衡的基础


## 衰减计算详解
* 之前我们提到 `y^n` 可以分解为 `(1/2)^(n/32) * y^(n%32)`，现在我们来把这个步骤彻底展开，它其实是利用了指数的运算性质，并结合了 Linux 内核中 **整数除法** 的特性

### 1. 数学上的完整推导（引入整数除法）
* 设总衰减周期数为 \( n \)。我们将 \( n \) 分解为 **商** 和 **余数**：

\[
n = 32 \times k + r
\]

* 其中：
  - \( k = \left\lfloor \frac{n}{32} \right\rfloor \)（即整数除法，**向下取整**）
  - \( r = n \bmod 32 \)（余数，范围 0 ~ 31）
* 那么，\( y^n \) 可以写成：

\[
y^n = y^{(32k + r)} = (y^{32})^k \times y^r
\]

* 因为 PELT 算法规定了 **\( y^{32} = 0.5 \)**，代入后得到：

\[
y^n = (0.5)^k \times y^r
\]

* 现在回头看之前写的 **\( (1/2)^{(n/32)} \)**：
  * 在纯数学实数除法下，\( (1/2)^{(n/32)} \) 不等于 \( (0.5)^k \)（除非 `n` 刚好能被 `32` 整除）
  * 但在 Linux 内核的 C 语言代码中，如果 `n` 是整型变量，**`n/32` 就是整数除法（自动向下取整）**
    * 因此，之前的表述其实是 **借用了代码语境下的整数除法语义**，在数学推导上不严谨
    * 正确的拆分应该是上面带向下取整符号 \( \lfloor \rfloor \) 的形式

### 2. 为什么要这么拆？（性能优化的精髓）
* 拆成 \( (0.5)^k \times y^r \) 不是为了数学好看，而是为了 **极致地避免浮点运算和循环**
* 在 PELT 的实现中，计算 \( y^n \) 的衰减系数需要在一个时钟中断内完成，不能做耗时的大循环，于是内核采用了 **查表 + 移位** 的策略：
1. **对于 \( (0.5)^k \)**：乘以 `0.5` 等于 **右移一位**（`>> 1`）。如果有 \( k \) 个 `0.5` 相乘，就是右移 \( k \) 位（`>> k`）
   * 移位操作在 CPU 中是一条极快的指令
2. **对于 \( y^r \)**：因为 \( r \) 只能是 0 到 31，内核预先在内存中计算好了一个包含 32 个元素的 **查找表**，直接查出 \( y^r \) 的值（这是一个大整数，用于定点数运算）
* 这样一来，计算 \( y^n \) 就变成了 **“一次查表 + 一次移位”**，时间复杂度从 `O(n)` 降到了 `O(1)`
* **数学上**：正确的分解是 \( y^n = (0.5)^{\lfloor n/32 \rfloor} \times y^{n \bmod 32} \)
* **代码里**：这里的“n 除以 32”被计算机解释为 **整数除法（取下整）**，而“乘以 0.5 的 k 次方”被转换为 **右移 k 位**

### 3. 内核源码：`decay_load()` 函数
* 我们直接看 Linux 内核 `kernel/sched/pelt.c` 中的 `decay_load` 函数，它是如何实现这个分解的：

```c
/*
 * Approximate:
 *   val * y^n,    where y^32 ~= 0.5 (~1 scheduling period)
 */
static u64 decay_load(u64 val, u64 n)
{
    unsigned int local_n;

    if (unlikely(n > LOAD_AVG_PERIOD * 63))
        return 0;

    /* after bounds checking we can collapse to 32-bit */
    local_n = n;
    //经过一定时间后，负载会衰减为 0
    /*  
     * As y^PERIOD = 1/2, we can combine
     *    y^n = 1/2^(n/PERIOD) * y^(n%PERIOD)
     * With a look-up table which covers y^n (n<PERIOD)
     *
     * To achieve constant time decay_load.
     */
    if (unlikely(local_n >= LOAD_AVG_PERIOD)) {
        val >>= local_n / LOAD_AVG_PERIOD; //处理 k = n / 32，直接用右移实现乘以 (0.5)^k
        local_n %= LOAD_AVG_PERIOD;        //得到余数
    }   
    //处理 r = n % 32，用查表法乘以 y^r，runnable_avg_yN_inv[] 是预计算好的 32 个值的数组
    val = mul_u64_u32_shr(val, runnable_avg_yN_inv[local_n], 32); //得到 val * y^n
    return val;
}
```

### 4. 举个具体的数字例子

* 假设我们要计算 \( n = 35 \) 个周期前的负载贡献衰减值（即 \( y^{35} \)）：
1.  **分解**：\( 35 = 32 \times 1 + 3 \)，所以 \( k=1, r=3 \)
2.  **应用公式**：\( y^{35} = (0.5)^1 \times y^3 = 0.5 \times y^3 \)
3.  **代码执行**：
    - `val >>= 1` （负载值减半，相当于乘以 `0.5`）。
    - 查表取出预计算好的 `runnable_avg_yN_inv[3]`（这个值就是 \( y^3 \) 的定点数表示）
    - 两者相乘，得到精确的 \( val \times y^{35} \)

### 问题：`mul_u64_u32_shr(val, runnable_avg_yN_inv[local_n], 32)` 的右移 32 位是为什么？
* 我们看 `mul_u64_u32_shr()` 的实现知道第三个参数是“右移的位数”，`runnable_avg_yN_inv[]` 数组不是预先计算好的 `y^(n%32)` 的值吗？为什么 `val * runnable_avg_yN_inv[n]` 后还要右移 `32` 位？
  * include/linux/math64.h
```cpp
#if defined(CONFIG_ARCH_SUPPORTS_INT128) && defined(__SIZEOF_INT128__)

#ifndef mul_u64_u32_shr
static __always_inline u64 mul_u64_u32_shr(u64 a, u32 mul, unsigned int shift)
{
    return (u64)(((unsigned __int128)a * mul) >> shift);
}
#endif /* mul_u64_u32_shr */
...
#else

#ifndef mul_u64_u32_shr
static __always_inline u64 mul_u64_u32_shr(u64 a, u32 mul, unsigned int shift)
{
    u32 ah = a >> 32, al = a;
    u64 ret;

    ret = mul_u32_u32(al, mul) >> shift;
    if (ah)
        ret += mul_u32_u32(ah, mul) << (32 - shift);
    return ret;
}
#endif /* mul_u64_u32_shr */
..
#endif
```
* 准确说，`runnable_avg_yN_inv[]` 数组还真不是预先计算好的 `y^(n%32)` 的值，而是把这些值又左移 `32` 位的值
  * kernel/sched/sched-pelt.h
```cpp
static const u32 runnable_avg_yN_inv[] __maybe_unused = {
    0xffffffff, 0xfa83b2da, 0xf5257d14, 0xefe4b99a, 0xeac0c6e6, 0xe5b906e6,
    0xe0ccdeeb, 0xdbfbb796, 0xd744fcc9, 0xd2a81d91, 0xce248c14, 0xc9b9bd85,
    0xc5672a10, 0xc12c4cc9, 0xbd08a39e, 0xb8fbaf46, 0xb504f333, 0xb123f581,
    0xad583ee9, 0xa9a15ab4, 0xa5fed6a9, 0xa2704302, 0x9ef5325f, 0x9b8d39b9,
    0x9837f050, 0x94f4efa8, 0x91c3d373, 0x8ea4398a, 0x8b95c1e3, 0x88980e80,
    0x85aac367, 0x82cd8698,
};

#define LOAD_AVG_PERIOD 32
#define LOAD_AVG_MAX 47742
```
* 这么做的直接原因在于，内核为了 **避免使用浮点数（float/double）**，将小数 `y^n` 放大了 `2^32` 倍，存储为一个大整数。**右移 32 位** 正是为了抵消这个放大倍数
* 这么做的意义：
  1.  **性能极致**：调度器在时钟中断和上下文切换中频繁调用，**浮点运算（FPU）** 在 Linux 内核中是被尽量避免的（因为保存/恢复 FPU 寄存器开销很大，且在某些体系架构下内核态根本不允许使用）
  2.  **确定性**：整数运算在所有的 CPU 架构上结果完全一致，而浮点数存在精度舍入差异，不利于调度行为的一致性
#### 为什么恰好是右移 32 位？
* 这是 Linux 内核定点数的经典约定（被称为 `SCHED_FIXEDPOINT_SHIFT`）。
  - 选择 `2^32` 是因为它足够大，能保证 `y^n`（0.97 左右）放大后依然有极高的精度（误差在 1/40 亿以内）。
  - 同时，`val`（负载值）通常也是按照 `2^32` 缩放过的。所以 `val * y^n` 做完乘法，再右移 32 位，结果依然保持了 `2^32` 的精度标准，可以继续参与后续的累加运算，而不会丢失有效数字

## PELT 负载的累加
* `__accumulate_pelt_segments()` 函数是 PELT 算法的 **数学核心**，它负责计算过去一段时间内，负载贡献的 **累加**

### 函数背景与输入参数
* 在 PELT 更新时，时间被分成了三段（`d1`, `d2`, `d3`），这个函数的作用就是计算这三段时间对负载总和的贡献

```c
|<- d1 ->|<- 完整的 periods 个周期 (d2) ->|<- d3 ->|
|        |                               |        |
^        ^                               ^        ^
旧时间   周期边界                        周期边界  当前时间
```
- `periods`：中间完整的周期数（即 `d2 / 1024`）
- `d1`：上一个不完整周期剩余的时间（微秒）
- `d3`：当前不完整周期中已经过去的时间（微秒）

### 代码逐行解析
* kernel/sched/pelt.c
```cpp
static u32 __accumulate_pelt_segments(u64 periods, u32 d1, u32 d3) 
{
    u32 c1, c2, c3 = d3; /* y^0 == 1 */

    /*  
     * c1 = d1 y^p
     */
    c1 = decay_load((u64)d1, periods);

    /*  
     *            p-1
     * c2 = 1024 \Sum y^n
     *            n=1
     *
     *              inf        inf
     *    = 1024 ( \Sum y^n - \Sum y^n - y^0 )
     *              n=0        n=p
     */
    c2 = LOAD_AVG_MAX - decay_load(LOAD_AVG_MAX, periods) - 1024;

    return c1 + c2 + c3; 
}
```
#### 第三段时间 `c3` 的计算
* `u32 c1, c2, c3 = d3; /* y^0 == 1 */`
  - `c3 = d3`：当前的贡献
  - 为什么 `y^0 == 1`？
    * `d3` 是当前周期内的时间，距离现在 **0个周期**，因此衰减系数是 `y^0 = 1`，**无需衰减**，直接累加

#### 第一段时间 `c1` 的计算
* `c1 = decay_load((u64)d1, periods);`
  - 数学含义：`c1 = d1 * y^periods`
  - 解释：`d1` 是发生在 `periods` 个完整周期之前的时间。它需要承受 `periods` 个周期的 **完整衰减**
  - 例子：如果 `periods=3`，`d1` 是 3 个周期前的贡献，它的衰减系数就是 `y^3`

#### 第二段时间 `c2` 的计算
* `c2 = LOAD_AVG_MAX - decay_load(LOAD_AVG_MAX, periods) - 1024;`
  * 这是函数中最精妙、最难理解的一行，它计算的是 **中间完整周期 `d2` 的贡献总和**

##### 理解 `LOAD_AVG_MAX`
* `LOAD_AVG_MAX` 是 **等比级数的极限和**，定义为：
  
  \[
  \text{LOAD_AVG_MAX} = 1024 \times \sum_{n=0}^{\infty} y^n
  \]
  
* 根据等比数列求和公式（当 `0 < y < 1` 时）：
  
  \[
  \sum_{n=0}^{\infty} y^n = \frac{1}{1 - y}
  \]
  
* `LOAD_AVG_MAX` 就是这个极限值的 `1024` 倍（因为每个周期贡献 `1024us`）

##### 为什么要这样计算 `c2`？
* `d2` 是中间 `periods` 个 **完整周期** 的贡献，每个完整周期贡献 `1024us`，数学表达式为：

\[
c2 = 1024 \times (y^1 + y^2 + ... + y^{periods-1})
\]

* 这个和式从 `n=1` 到 `n=periods-1`，**不包含** `y^0`，也不包含 `n >= periods`
* 内核用了非常巧妙的 **差集** 方法来计算：

\[
c2 = 1024 \times \sum_{n=1}^{p-1} y^n
   = 1024 \times ( \sum_{n=0}^{\infty} y^n - \sum_{n=p}^{\infty} y^n - y^0)
\]

1. 从无限和开始：`LOAD_AVG_MAX = 1024 * (y^0 + y^1 + y^2 + ...)`
2. 减去从 `periods` 到无穷的部分：`decay_load(LOAD_AVG_MAX, periods)`
3. 减去 `y^0` 项（即 `1024`）
* 所以：
```c
c2 = LOAD_AVG_MAX                      //全部无限项
   - decay_load(LOAD_AVG_MAX, periods) //减去 n>=periods 的项
   - 1024                              //减去 n=0 的项（即 y^0 * 1024）
   = 1024 * (y^1 + y^2 + ... + y^(periods-1))
```

###### 为什么 `decay_load(LOAD_AVG_MAX, periods)` 表示“从 `periods` 到无穷”的部分？
* 因为 `LOAD_AVG_MAX` 是一个无限等比级数的总和，对它进行 `periods` 周期的衰减，相当于级数中每一项都乘以 `y^periods`：

\[
\text{decay_load}(LOAD\_AVG\_MAX, periods) = 1024 \times \sum_{n=0}^{\infty} y^n \times y^{periods} = 1024 \times \sum_{n=0}^{\infty} y^{n + periods} = 1024 \times \sum_{m=periods}^{\infty} y^m
\]

##### 举个例子验证
* 假设 `periods = 3`：
  - `LOAD_AVG_MAX = 1024 * (1 + y + y^2 + y^3 + ...)`
  - `decay_load(LOAD_AVG_MAX, 3) = 1024 * (y^3 + y^4 + y^5 + ...)`
  - 相减：`1024 * (1 + y + y^2)`
  - 再减去 `1024`（即 `n=0` 项）：`1024 * (y + y^2)`
* 正好等于 `c2 = 1024 * (y^1 + y^2)`

#### 返回总贡献
* `return c1 + c2 + c3;` 将三段贡献相加，得到总贡献：
\[
\text{总贡献} = d1 \times y^{periods} + 1024 \times \sum_{n=1}^{periods-1} y^n + d3
\]
* 此即 PELT 更新公式的完整数学表达

### 为什么不用循环直接累加？
* 为什么不直接用循环把 `periods` 个周期的贡献加起来？
* 原因：性能因素
1. **O(1) vs O(n)**：这个函数在每次时钟中断（每 `1ms`）都可能被调用。如果直接循环累加，当系统空闲很久后，`periods` 可能非常大（比如几万），循环开销不可接受
2. **利用几何级数性质**：通过 `LOAD_AVG_MAX - decay_load(LOAD_AVG_MAX, periods) - 1024`，用常数次运算（`decay_load` 只有一次查表+移位）就计算出了完整的级数和

## 负载贡献的累加总和
* PELT 更新时间被分成了三段 `d1`、`d2`、`d3` 的概念之已讨论过不再赘述，`accumulate_sum()` 函数是 PELT 更新流程中的 **总调度中心**
* 优化了一下示意图：
```c
     d3'  d1          d2          d3
      ^   ^           ^           ^
      |   |           |           |
    |---|--|<----------------->|----|
... |---x--|------| ... |------|----x (now)
        |               |<---->|    |
        |         1024us = 1 period |
        |<-----------delta--------->|
```
* 它完成了两件核心工作：**老化（衰减）历史负载** 和 **累加新产生的负载**
```cpp
/*
 * Accumulate the three separate parts of the sum; d1 the remainder
 * of the last (incomplete) period, d2 the span of full periods and d3
 * the remainder of the (incomplete) current period.
 *
 *           d1          d2           d3
 *           ^           ^            ^
 *           |           |            |
 *         |<->|<----------------->|<--->|
 * ... |---x---|------| ... |------|-----x (now)
 *
 *                           p-1
 * u' = (u + d1) y^p + 1024 \Sum y^n + d3 y^0
 *                           n=1
 *
 *    = u y^p +					(Step 1)
 *
 *                     p-1
 *      d1 y^p + 1024 \Sum y^n + d3 y^0		(Step 2)
 *                     n=1
 */
static __always_inline u32
accumulate_sum(u64 delta, struct sched_avg *sa,
	       unsigned long load, unsigned long runnable, int running)
{
	u32 contrib = (u32)delta; /* p == 0 -> delta < 1024 */
	u64 periods;
    //sa->period_contrib 是上一次的 d3，和这一次的 d1（含在 delta 中）凑成一个周期
	delta += sa->period_contrib;
	periods = delta / 1024; /* A period is 1024us (~1ms) */

	/*
	 * Step 1: decay old *_sum if we crossed period boundaries.
	 */
	if (periods) {
		sa->load_sum = decay_load(sa->load_sum, periods);
		sa->runnable_sum =
			decay_load(sa->runnable_sum, periods);
		sa->util_sum = decay_load((u64)(sa->util_sum), periods);

		/*
		 * Step 2
		 */
		delta %= 1024; //此时 delta 被重赋值为 当前不完整周期中已过去的时间（即数学公式里的 d3）
		if (load) {
			/*
			 * This relies on the:
			 *
			 * if (!load)
			 *	runnable = running = 0;
			 *
			 * clause from ___update_load_sum(); this results in
			 * the below usage of @contrib to disappear entirely,
			 * so no point in calculating it.
			 */
			contrib = __accumulate_pelt_segments(periods,
					1024 - sa->period_contrib, delta);
		}
	}
	sa->period_contrib = delta; //余数 d3 留待下次使用

	if (load)
		sa->load_sum += load * contrib;
	if (runnable)
		sa->runnable_sum += runnable * contrib << SCHED_CAPACITY_SHIFT;
	if (running)
		sa->util_sum += contrib << SCHED_CAPACITY_SHIFT;

	return periods;
}
```

### 1. 时间分片（边界对齐）
```c
delta += sa->period_contrib;
periods = delta / 1024; /* A period is 1024us (~1ms) */
```
* `sa->period_contrib` 为上次更新时，当前 `1024us` 周期内 **已经过去** 的时间（即上次 `delta % 1024` 的余数）
* `delta` 为这次的时间差
* `delta += sa->period_contrib`：将上次的余数加到这次的时间差上。这样，我们就得到了 **从上一个完整周期的起始边界** 到当前时刻的总时间
* `periods = delta / 1024`：计算这段时间跨越了多少个 **完整的 1024us 周期**
* 为什么要加 `sa->period_contrib`？
  * 因为 PELT 必须按 `1024us` 周期边界进行分段累加。如果不加，`d1`（上个周期剩余部分）就会被丢弃，导致负载计算不准确

### 2. Step 1 — 老化历史贡献（衰减旧的 `*_sum`）
```cpp
if (periods) {
    sa->load_sum = decay_load(sa->load_sum, periods);
    sa->runnable_sum = decay_load(sa->runnable_sum, periods);
    sa->util_sum = decay_load((u64)(sa->util_sum), periods);
    // ...
}
```
* 这里对应注释中公式的 Step 1：`u * y^p`
* 作用：将过去累积的 **所有** 贡献（`load_sum`、`runnable_sum`、`util_sum`）**整体** 衰减 `periods` 个周期
* 含义：距离现在越久远的负载，其权重越小。这是指数衰减在数据结构上的直接体现
* 注意：此时 `sa->*_sum` 已经变成了“老化后的历史值”，还 **不包含本次新增的负载**

### 3. Step 2 — 计算本次新增贡献（`contrib`）
```cpp
delta %= 1024;
if (load) {
    contrib = __accumulate_pelt_segments(periods,
                    1024 - sa->period_contrib, delta);
}
```
* 这里对应注释中公式的 **Step 2**：`d1 * y^p + 1024 * sum(...) + d3 * y^0`
* `delta %= 1024`：此时 `delta` 被重赋值为 **当前不完整周期中已过去的时间**（即数学公式里的 `d3`）
* `1024 - sa->period_contrib`：这是上个周期剩余的时间，即数学公式里的 `d1`
* 调用 `__accumulate_pelt_segments(periods, d1, d3)`：利用之前的几何级数差集算法，一次性算出本次新产生的负载贡献总值（`contrib`）
* **关键优化 `if (load)`**：注释里提到 `if (!load) runnable = running = 0`
  * 如果 `load` 为 `0`（即任务权重为 `0`，通常发生在空队列或系统空闲时），那么后续 `load * contrib` 依然为 `0`
  * 此时计算 `contrib` 完全是浪费 CPU，所以直接跳过

### 4. 累加回三个独立的 `*_sum` 字段

```c
sa->period_contrib = delta;

if (load)
    sa->load_sum += load * contrib;
if (runnable)
    sa->runnable_sum += runnable * contrib << SCHED_CAPACITY_SHIFT;
if (running)
    sa->util_sum += contrib << SCHED_CAPACITY_SHIFT;
```

* `sa->period_contrib = delta`：保存本次 `d3`（当前周期已过去的时间），作为下次更新的起始余数
* `load_sum` 累加：乘以 `load`（任务的权重 `load.weight`，通常与 nice 值相关）。因为不同 nice 值的任务对“负载”的贡献权重不同（高 nice 值权重低）
* `runnable_sum` 与 `util_sum` 累加：这里都左移了 `SCHED_CAPACITY_SHIFT`（通常为 `10`，即乘以 `1024`）。为什么要移位？
  * 因为 `contrib` 是基于 `1024us` 时间单位计算的，而 `util_sum` 需要以 **算力容量（capacity）** 为单位进行缩放（频率不变性和 CPU 不变性）
  * 左移 `10` 位相当于将 *时间贡献* 映射到 `[0, 1024]` 的 *容量刻度* 上，以便后续直接除以周期数得到 `util_avg`
* `runnable` 和 `running` 的标志位：分别控制是否更新 *可运行负载* 和 *利用率*
  * 它们在调用时由 `___update_load_sum()` 根据任务状态（是否在队列、是否在运行）设置

### 5. 返回值 `periods`
* 函数返回本次跨越的完整周期数
* 这个返回值主要用于 `___update_load_avg()` 中判断是否需要进行 **`load_avg` 的更新**，以及触发 **负载均衡** 的时机判断
  * 例如，如果 `periods` 大于 1，意味着系统可能经历了较长的空闲期

### 完整流程图（新旧合并）
* 把 Step 1 和 Step 2 合并，就得到了完整的更新公式：

\[
\text{new_sum} = \underbrace{\text{old_sum} \times y^{periods}}_{\text{Step 1: 老化历史}} + \underbrace{\text{__accumulate_pelt_segments}(periods, d1, d3)}_{\text{Step 2: 新增贡献}}
\]

* 这与 PELT 论文中的标准递推公式完全一致

### 总结：这个函数的精妙之处
1. **解耦历史与新增**：先把老的贡献全部衰减，再独立计算新的贡献，最后相加
2. **按需计算**：通过 `if (load)` 和 `if (runnable)` 等标志，避免在空闲时进行不必要的乘法运算
3. **统一的缩放处理**：将 `runnable_sum` 和 `util_sum` 左移 `SCHED_CAPACITY_SHIFT`，为后续除以周期数（`LOAD_AVG_MAX`）得到 `avg` 值做好了准备，同时保证了频率/CPU 不变性
4. **状态持久化**：通过 `period_contrib` 保存断点，使得中断性的更新也能完美拼接
* ==简单来说，`accumulate_sum()` 就像是一个精准的“计数器”，它把不连续的时间增量，通过周期边界对齐和指数衰减，精确地累加到了三个独立的负载统计维度（权重负载、可运行时间、利用率）上==

## 更新负载总和的入口
* `___update_load_sum()` 的主要任务：
1. 它把杂乱的纳秒级时间戳整理成规整的“周期块”
2. 它过滤掉无效的时间倒退和异常状态
3. 只有在确认“时间确实前进了足够多（至少跨过一个 `1024us` 边界）”，它才放行数据，交给 `accumulate_sum()` 做真正的数学运算，并向上层报告“有更新，请刷新平均值”
```cpp
/*
 * We can represent the historical contribution to runnable average as the
 * coefficients of a geometric series.  To do this we sub-divide our runnable
 * history into segments of approximately 1ms (1024us); label the segment that
 * occurred N-ms ago p_N, with p_0 corresponding to the current period, e.g.
 *
 * [<- 1024us ->|<- 1024us ->|<- 1024us ->| ...
 *      p0            p1           p2
 *     (now)       (~1ms ago)  (~2ms ago)
 *
 * Let u_i denote the fraction of p_i that the entity was runnable.
 *
 * We then designate the fractions u_i as our co-efficients, yielding the
 * following representation of historical load:
 *   u_0 + u_1*y + u_2*y^2 + u_3*y^3 + ...
 *
 * We choose y based on the with of a reasonably scheduling period, fixing:
 *   y^32 = 0.5
 *
 * This means that the contribution to load ~32ms ago (u_32) will be weighted
 * approximately half as much as the contribution to load within the last ms
 * (u_0).
 *
 * When a period "rolls over" and we have new u_0`, multiplying the previous
 * sum again by y is sufficient to update:
 *   load_avg = u_0` + y*(u_0 + u_1*y + u_2*y^2 + ... )
 *            = u_0 + u_1*y + u_2*y^2 + ... [re-labeling u_i --> u_{i+1}]
 */
static __always_inline int
___update_load_sum(u64 now, struct sched_avg *sa,
		  unsigned long load, unsigned long runnable, int running)
{
	u64 delta;

	delta = now - sa->last_update_time;
    //在系统启动早期，当调度时钟（sched_clock）从不稳定源切换到 TSC（Time Stamp Counter）时，时间戳可能会出现短暂的倒退
    //如果检测到这种情况，直接重置时间戳并返回，放弃本次更新，避免产生负数负载
	/*
	 * This should only happen when time goes backwards, which it
	 * unfortunately does during sched clock init when we swap over to TSC.
	 */
	if ((s64)delta < 0) {
		sa->last_update_time = now;
		return 0;
	}
    //原始 delta 的单位是纳秒 (ns)。delta >>= 10 等价于 delta / 1024，将纳秒转换为以 1024ns（约 1us）为单位的量
    //好处：移位代替除法，计算更快。同时，1024us（约 1ms）是 PELT 的一个标准周期，用 1024 作为基数便于后续的周期对齐计算
	/*
	 * Use 1024ns as the unit of measurement since it's a reasonable
	 * approximation of 1us and fast to compute.
	 */
	delta >>= 10;
	if (!delta)
		return 0;

	sa->last_update_time += delta << 10;

	/*
	 * running is a subset of runnable (weight) so running can't be set if
	 * runnable is clear. But there are some corner cases where the current
	 * se has been already dequeued but cfs_rq->curr still points to it.
	 * This means that weight will be 0 but not running for a sched_entity
	 * but also for a cfs_rq if the latter becomes idle. As an example,
	 * this happens during sched_balance_newidle() which calls
	 * sched_balance_update_blocked_averages().
	 *
	 * Also see the comment in accumulate_sum().
	 */
	if (!load)
		runnable = running = 0;
    //是否跨越了一个周期的边界？
	/*
	 * Now we know we crossed measurement unit boundaries. The *_avg
	 * accrues by two steps:
	 *
	 * Step 1: accumulate *_sum since last_update_time. If we haven't
	 * crossed period boundaries, finish.
	 */
	if (!accumulate_sum(delta, sa, load, runnable, running))
		return 0; //未跨域周期边界，负载和利用率数值没有任何变化，无需进行除法运算
    //跨过了边界，历史负载发生了衰减和新增
	return 1;
}
```

### 为什么更新 `last_update_time` 时用累加（`+=`），而不是直接赋值 `= now`？
* 因为右移 `10` 位丢失了低 `10` 位的精度（纳秒级）
* 如果直接 `sa->last_update_time = now`，下次计算 `delta` 时会包含这次丢失的纳秒余数，导致时间被重复计算，产生累加误差
* 通过 `sa->last_update_time += delta << 10`（将微秒级增量转回纳秒并累加），`last_update_time` 始终与 `delta` 的“粗粒度”步长对齐，确保时间差是 `1024ns` 的整数倍，避免了精度漂移

### 为什么 `if (!load) runnable = running = 0`？
* 背景：`load` 通常对应调度实体的权重（`se->load.weight`）。在某些特殊场景下（比如任务已经出队，但 `cfs_rq->curr` 指针还没来得及更新），`load` 可能为 `0`，但 `runnable` 或 `running` 标志可能仍为 `1`
* 当无新的负载时，强制将 `runnable` 和 `running` 置 `0`
* 结合上一轮的 `accumulate_sum()` 来看，
  * 如果 `load == 0`，则 `if (load)` 条件不成立，`contrib` 根本不会被计算，后续的 `load_sum` 也不会累加
  * 如果此时 `runnable` 或 `running` 还是 `1`，就可能导致 `util_sum` 误累加（虽然乘数为 `0`，但为了避免逻辑混乱，内核干脆在这里统一清零，确保“无权重，则无任何负载贡献”的一致性）

### 返回值解析
* `accumulate_sum()` 返回的是本次跨越的完整周期数 `periods`
* 如果 `periods == 0`（意味着两次更新间隔不足 `1024us`，没有跨过周期边界），则 `___update_load_sum()` 返回 `0`
* 如果 `periods > 0`（跨过了边界，历史负载发生了衰减和新增），则返回 `1`
* 它的调用者根据这个返回值直接随后的 `___update_load_avg()` 是否需要重新计算 `*_avg`（即 `load_avg`、`runnable_avg`、`util_avg`）
  * 如果 `periods == 0`，说明负载和利用率数值没有任何变化，无需进行除法运算（`sum / LOAD_AVG_MAX`），直接跳过，这大大节省了 CPU 开销

## 从累加到平均值的转换
* `___update_load_avg()` 函数是整个 PELT 机制中 **“从累加到平均值”的转换器**
* 它负责将之前计算出的无标度累加值（`*_sum`），转换成直观的、限定在 `[0, 1024]` 范围内的平均利用率（`*_avg`）
* 要彻底理解它，核心在于搞懂那个 **动态除数（Divider）** 的设计逻辑
```cpp
/*
 * When syncing *_avg with *_sum, we must take into account the current
 * position in the PELT segment otherwise the remaining part of the segment
 * will be considered as idle time whereas it's not yet elapsed and this will
 * generate unwanted oscillation in the range [1002..1024[.
 *
 * The max value of *_sum varies with the position in the time segment and is
 * equals to :
 *
 *   LOAD_AVG_MAX*y + sa->period_contrib
 *
 * which can be simplified into:
 *
 *   LOAD_AVG_MAX - 1024 + sa->period_contrib
 *
 * because LOAD_AVG_MAX*y == LOAD_AVG_MAX-1024
 *
 * The same care must be taken when a sched entity is added, updated or
 * removed from a cfs_rq and we need to update sched_avg. Scheduler entities
 * and the cfs rq, to which they are attached, have the same position in the
 * time segment because they use the same clock. This means that we can use
 * the period_contrib of cfs_rq when updating the sched_avg of a sched_entity
 * if it's more convenient.
 */
static __always_inline void
___update_load_avg(struct sched_avg *sa, unsigned long load)
{
	u32 divider = get_pelt_divider(sa);

	/*
	 * Step 2: update *_avg.
	 */
	sa->load_avg = div_u64(load * sa->load_sum, divider);
	sa->runnable_avg = div_u64(sa->runnable_sum, divider);
	WRITE_ONCE(sa->util_avg, sa->util_sum / divider);
}
```

### 动态除数 `get_pelt_divider()`
* kernel/sched/pelt.h
```c
#define PELT_MIN_DIVIDER        (LOAD_AVG_MAX - 1024)

static inline u32 get_pelt_divider(struct sched_avg *avg)
{
        return PELT_MIN_DIVIDER + avg->period_contrib;
}
```

#### 为什么除数是 `LOAD_AVG_MAX - 1024 + period_contrib`，而不是直接用固定的 `LOAD_AVG_MAX`？

##### 1. 除数的数学来源
* 在 PELT 算法中，`*_sum` 的最大可能值并不是一个固定常量，而是随着当前周期内已度过的时间（`period_contrib`）变化的

###### “最大和”的场景
* 假设一个 CPU 从一开始（`-∞`）到现在，**每一刻都处于满负荷运行状态**（即每个 `1024us` 周期都跑满）
  - 当前周期（不完整）：已经过去了 `period_contrib` 微秒，贡献为 `period_contrib * y^0`（因为当前无需衰减，`y^0=1`）
  - 上一个完整周期：贡献为 `1024 * y^1`（衰减了 1 个周期）
  - 上上个完整周期：贡献为 `1024 * y^2`
  - 以此类推，直到无穷远，
* 那么，`*_sum` 能达到的 **理论最大值**（记作 `Sum_max`）就是所有这些贡献的总和：

\[
\text{Sum_max} = period\_contrib + 1024 \times (y^1 + y^2 + y^3 + \cdots) = period\_contrib + 1024 \times (y^0 + y^1 + y^2 + \cdots) \times y = period\_contrib + 1024 \times \sum_{n=0}^{\infty} y^n \times y
\]
* 我们得到 **最大和的理论值**：`period_contrib + LOAD_AVG_MAX * y`

###### 恒等式 `LOAD_AVG_MAX * y = LOAD_AVG_MAX - 1024`
* 根据 `LOAD_AVG_MAX` 的定义：

\[
\text{LOAD_AVG_MAX} = 1024 \times \sum_{n=0}^{\infty} y^n = 1024 + 1024 \times \sum_{m=1}^{\infty} y^m = 1024 + 1024 \times \sum_{n=0}^{\infty} y^n \times y = 1024 + \text{LOAD_AVG_MAX} \times y
\]

* 我们得到 **恒等式**：`LOAD_AVG_MAX * y = LOAD_AVG_MAX - 1024`

###### 得到除数 `divider`
* 将 *恒等式* 代入 *最大和的理论值* 得到：`Sum_max = (LOAD_AVG_MAX - 1024) + period_contrib`

###### 为什么要搞出这个公式？

* 这个公式之所以重要，是因为它直接决定了 **`util_avg` 的上限精确性**
* 我们知道 `___update_load_avg()` 计算 `util_avg` 的除法是：
\[
util\_avg = \frac{util\_sum}{\text{Divider}}
\]

* 其中 `Divider = LOAD_AVG_MAX - 1024 + period_contrib`（即 `LOAD_AVG_MAX * y + period_contrib`）
* **当 CPU 真正满载时**，`util_sum` 刚好达到理论最大值 `Sum_max`
* 代入除法：\( util\_avg = \frac{Sum\_max}{Divider} = \frac{Divider}{Divider} = 1 \)
* 换算成内核的 `0~1024` 刻度，`util_avg` 正好等于 **1024**，完美地反映了 100% 繁忙的状态
* 如果不用这个动态除数，满载时算出来的值永远达不到 `1024`，会导致调度器误以为 CPU 还有空闲容量而错误地迁移任务

##### 2. 为什么要用动态除数？（防止振荡）
* 注释中提到的 *“unwanted oscillation in the range [1002..1024[”* 是关键
* **假设使用固定除数 `LOAD_AVG_MAX`**：
  * 如果一个 CPU 在长达 `32ms` 内一直满负荷运行（`util_sum` 达到了当前周期能累加的最大值），计算 `util_avg = util_sum / LOAD_AVG_MAX`
  * 随着 `period_contrib` 从 `0` 逐渐增长到 `1023`，分子（最大和）也在从 `LOAD_AVG_MAX-1024` 逐渐增长到 `LOAD_AVG_MAX-1`
  * **结果**：即使 CPU 完全满载，计算出的 `util_avg` 也只能在 `1002` 到 `1023` 之间波动，**永远达不到 1024**
    * 这会导致负载均衡器认为该 CPU 还有少量余力，从而错误地迁移任务过来
* **使用动态除数 `LOAD_AVG_MAX - 1024 + period_contrib`**：
  * 分子最大和除以这个动态除数，结果恒等于 `1`。换算成 `util_avg`（`0~1024` 刻度）就是 **1024**
  * 这精确地反映了 CPU 100% 繁忙的真实状态，消除了锯齿振荡，保证了算法的收敛性

### 三类平均值的计算逻辑

| 字段 | 计算方式 | 备注 |
| :--- | :--- | :--- |
| `load_avg` | `(load * load_sum) / divider` | **`load` 作为参数传入**（来自 `se->load.weight` 或 `cfs_rq->load.weight`）。因为 `sched_avg` 结构体本身不保存权重值，需要外部提供。这里的乘法恢复了负载的“权重”属性，使得高优先级任务对系统负载的贡献更大。使用 `div_u64` 处理 64 位乘法防止溢出。 |
| `runnable_avg` | `runnable_sum / divider` | 代表处于可运行状态（就绪队列中）的平均时间占比。只除不分权重，因为它反映的是时间份额。 |
| `util_avg` | `util_sum / divider` | 代表对 CPU 算力的实际利用率。这里用了 `WRITE_ONCE()`，因为 `util_avg` 会被调度器（如 EAS 能量感知调度）和调频模块（`schedutil`）**并发读取**，需要保证读写的原子性，防止读到一个撕裂的中间值。 |

### 何时调用此函数？
* [之前](#返回值解析)说过，`___update_load_avg()` 总是在 `___update_load_sum()` 返回 `1`（即跨过了至少一个 `1024us` 周期边界，`*_sum` 发生了变化）之后被调用
  * 如果两次更新间隔小于 `1024us`，`periods == 0`，`sum` 没有衰减也没有新增有效贡献，直接跳过耗时的除法运算，**这是一种高效的性能优化**
* 典型的调用链如下：

```c
update_load_avg()   //上层入口（任务入队/出队/时钟中断）
-> ___update_load_sum()   //更新 sum，返回是否需要刷新 avg
-> ___update_load_avg()   //将 sum 转换为 avg
```

* 简单来说，`___update_load_avg()` 在 PELT 流水线中的角色 **就像一个“归一化处理器”**：

| 函数 | 职责 | 输出 |
| :--- | :--- | :--- |
| `___update_load_sum()` | 累计历史衰减 + 新增时间片贡献 | 更新 `*_sum`，返回是否跨周期 |
| `___update_load_avg()` | 将 `*_sum` 标准化为 `0~1024` 的 `*_avg` | 更新 `*_avg`（供负载均衡和调频使用）|

1. 它动态计算出当前周期下正确的除数，确保满载时能达到 `1024` 的精确上限
2. 它将无单位的大整数 `*_sum` 转换成直观的百分比（`0~1024` 刻度），供上层的负载均衡（寻找最繁忙 CPU）和频率缩放（`schedutil` 决定 CPU 频率）直接使用

## `__update_load_avg_blocked_se()` 函数
* 函数 `__update_load_avg_blocked_se` 是 PELT 更新路径中一个 **专门用于“老化（Aging）”阻塞任务** 的入口
* `__update_load_avg_blocked_se()` 函数的设计哲学是：==一个不在 CPU 上运行的任务，其负载不应凭空消失，而应随着时间指数级衰减；但它绝不能在睡眠期间‘刷存在感’（累加新的时间片）==
* 它通过传递 **全 `0`** 标志给底层 `___update_load_sum()`，精确实现了“**只衰减、不新增**”的语义
  * 这使得调度器能够记住一个进程在过去（比如 `16ms` 前）是 CPU 密集型的，从而在它醒来时给予适当的处理
  * 同时确保如果它睡得太久（超过 `32ms`），它的历史影响会衰减到几乎可以忽略不计，不会再干扰当前的负载均衡决策
```c
/*
 * sched_entity:
 *
 *   task:
 *     se_weight()   = se->load.weight
 *     se_runnable() = !!on_rq
 *
 *   group: [ see update_cfs_group() ]
 *     se_weight()   = tg->weight * grq->load_avg / tg->load_avg
 *     se_runnable() = grq->h_nr_runnable
 *
 *   runnable_sum = se_runnable() * runnable = grq->runnable_sum
 *   runnable_avg = runnable_sum
 *
 *   load_sum := runnable
 *   load_avg = se_weight(se) * load_sum
 *
 * cfq_rq:
 *
 *   runnable_sum = \Sum se->avg.runnable_sum
 *   runnable_avg = \Sum se->avg.runnable_avg
 *
 *   load_sum = \Sum se_weight(se) * se->avg.load_sum
 *   load_avg = \Sum se->avg.load_avg
 */
int __update_load_avg_blocked_se(u64 now, struct sched_entity *se)
{
	if (___update_load_sum(now, &se->avg, 0, 0, 0)) {
		___update_load_avg(&se->avg, se_weight(se));
		trace_pelt_se_tp(se);
		return 1;
	}

	return 0;
}
```

### 1. 关键参数：`___update_load_sum(now, &se->avg, 0, 0, 0)`
* 为什么三个传入的累计标志（`load`, `runnable`, `running`）全是 `0`？
* 回顾之前分析的 `___update_load_sum()`，它内部的 `accumulate_sum()` 会检查 `if (load)` 来决定是否计算新增贡献 `contrib`
* **当全为 0 时**：`accumulate_sum()` 中的 `if (periods)` 分支依然会执行，但只会执行 **Step 1（老化历史）**，即：
  ```c
  sa->load_sum = decay_load(sa->load_sum, periods);
  sa->runnable_sum = decay_load(sa->runnable_sum, periods);
  sa->util_sum = decay_load(sa->util_sum, periods);
  ```
* **不会执行 Step 2（累加新贡献）**，因为 `if (load)` 条件不成立，`contrib` 保持为 `0`，三个 `*_sum` 也不会加上新的时间片
* **结论**：这个函数 **只会对历史负载进行指数衰减，不会累加任何新的运行时间**。
  * 这正是“阻塞（Blocked）”任务应有的行为——它不在 CPU 上运行，因此不产生新负载，但它的历史负载会随着时间推移慢慢“淡出”调度器的视野。

### 2. 平均值更新：`___update_load_avg(&se->avg, se_weight(se))`
- 当 `___update_load_sum()` 返回 `1`（即跨过了至少一个 `1024us` 周期边界，`*_sum` 发生了变化），才调用此函数刷新平均值
- 传入的权重参数是 `se_weight(se)`
- 注释中的重要说明：
  - **对于普通任务（Task）**：`se_weight(se) = se->load.weight`（即任务的 nice 值权重）
  - **对于任务组（Group）**：`se_weight(se) = tg->weight * grq->load_avg / tg->load_avg`（这是组调度中复杂的权重分摊计算）

#### 为什么 `load_avg` 需要乘以权重，而 `runnable_avg` 和 `util_avg` 不乘？  
* 注释中写明了：
  - `load_sum := runnable`（这里指未加权的原始时间累积）
  - `load_avg = se_weight(se) * load_sum`
* 因为“负载（Load）”在 Linux 调度器中代表“对系统造成的压力”，高优先级任务即使运行时间相同，其对系统施加的“重量”也不同，所以必须在计算 `load_avg` 时乘以权重

### 调用场景与设计意图

#### 1. 什么时候调用？
* 主要在两个场景：
  * **任务出队（Dequeue）后**：当任务因等待 I/O、锁或主动睡眠而进入阻塞状态时，会调用此函数，确保它从离开 CPU 的那一刻起，负载就开始指数衰减
  * **周期性的负载平衡**（如 `sched_balance_update_blocked_averages()`）：系统会定期遍历所有阻塞任务，主动老化它们的负载，避免已睡眠很久的任务依然被算作“高负载”而导致负载均衡决策失误

#### 2. 为什么要独立出这个函数？
* **解耦**：将“运行中任务的负载更新”与“阻塞任务的负载老化”分开。
  * 运行中任务需要累加 `runnable` 和 `running` 时间，逻辑更复杂
  * 阻塞任务逻辑单一（只衰减），独立出来可以避免频繁的条件判断，提升性能
* **防止误累加**：如果误传了非零的 `runnable` 标志，一个睡眠任务可能会被错误地累加上可运行时间，导致 `runnable_avg` 虚高，调度器会误以为该 CPU 上有很多任务在排队

### 结合注释理解数据流

注释中定义了几个关键公式，结合 `blocked_se` 来看：

| 指标 | 计算公式（阻塞任务） | 含义 |
| :--- | :--- | :--- |
| `se->avg.load_sum` | 仅执行 `decay_load()`，不累加 | 历史负载权重的衰减和（未乘权重前）|
| `se->avg.runnable_sum` | 仅执行 `decay_load()`，不累加 | 历史可运行时间的衰减和 |
| `se->avg.util_sum` | 仅执行 `decay_load()`，不累加 | 历史 CPU 利用率的衰减和 |
| `se->avg.load_avg` | `se_weight(se) * load_sum / divider` | 反映该任务在阻塞期间 **依然占用的系统“重量”** |
| `se->avg.runnable_avg` | `runnable_sum / divider` | 反映该任务在阻塞期间 **依然“可运行”状态的时间占比**（通常很快衰减） |
| `se->avg.util_avg` | `util_sum / divider` | 反映该任务在阻塞期间 **依然需要的 CPU 算力**（用于 EAS 等） |

## `__update_load_avg_se()` 函数
* 函数 `__update_load_avg_se()` 是 PELT 更新路径中 **最核心、最通用的入口**，专门用于更新 **正在运行队列中（或正在运行）的调度实体（`sched_entity`）**
* 它与刚刚讲过的 `__update_load_avg_blocked_se()`（只老化不新增）形成了鲜明的对比：==`__update_load_avg_se()` 不仅会老化历史负载，还会累加当前这一小段时间内新产生的运行贡献==
* `__update_load_avg_se()` 函数的设计体现了 PELT 算法的 **多维负载观测** 理念：
  * 一个 **“就绪”** 的任务（`on_rq`）会产生 **“负载”压力**
  * 它 **在队列中等待** 会贡献 **“可运行”时间**
  * 只有当它真正 **抢到 CPU 执行** 时，才贡献 **“利用率”**
* 通过这三个独立的标志位，内核能够精细区分：
  * `load_avg`：用于传统的负载均衡（选择最繁忙的 CPU）
  * `runnable_avg`：用于抢占和组调度中的压力评估
  * `util_avg`：用于性能调频（DVFS）和能量感知调度（EAS），因为它只关心实实在在的 CPU 算力消耗
```cpp
int __update_load_avg_se(u64 now, struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	if (___update_load_sum(now, &se->avg, !!se->on_rq, se_runnable(se),
				cfs_rq->curr == se)) {

		___update_load_avg(&se->avg, se_weight(se));
		cfs_se_util_change(&se->avg);
		trace_pelt_se_tp(se);
		return 1;
	}

	return 0;
}
```

### 核心差异：三个标志位的精准控制
* `load`、`runnable`、`running` 这三个参数直接决定了哪些 `*_sum` 会被累加新贡献：
```c
___update_load_sum(now, &se->avg, 
                   !!se->on_rq,       //load 标志
                   se_runnable(se),   //runnable 标志
                   cfs_rq->curr == se //running 标志
)
```

| 参数位置 | 传入值 | 对应的 `accumulate_sum()` 行为 | 影响的统计量
| :--- | :--- | :--- | :---
| `load`（第3个）| `!!se->on_rq` | 如果为 `1`，则计算 `contrib`，并执行 `sa->load_sum += load * contrib` | `load_sum`（权重负载）
| `runnable`（第4个）| `se_runnable(se)` | 如果为 `1`，则执行 `sa->runnable_sum += runnable * contrib << SCHED_CAPACITY_SHIFT` | `runnable_sum`（可运行时间）
| `running`（第5个）| `cfs_rq->curr == se` | 如果为 `1`，则执行 `sa->util_sum += contrib << SCHED_CAPACITY_SHIFT` | `util_sum`（实际运行利用率）

* 源码开头的注释清晰地定义了不同层级的数据流，在这个函数中得到了完整闭环：
1. `load_sum` 的累加：用 `se->on_rq` 控制，乘以 `se_weight(se)`（权重），得到带权重的负载压力
2. `runnable_sum` 的累加：用 `se_runnable(se)` 控制，反映排队时间（对于组调度尤为重要）
3. `util_sum` 的累加：用 `cfs_rq->curr == se` 严格控制，只记录真正运行的时间

#### 1. `!!se->on_rq` — 是否在运行队列上？
* **`on_rq`** 表示该实体是否在 CFS 的就绪队列（`cfs_rq`）中，包括两种情况：
1. 正在等待调度的任务（在队列里排队）
2. 正在当前 CPU 上运行的任务（`curr` 指针指向它，它也在 `on_rq` 上，除非被特殊标记出队）
* **作用**：只要实体处于“就绪”状态（`on_rq=1`），它的 `load_sum` 就应该被累加。这代表了该实体对系统造成的“权重压力”

#### 2. `se_runnable(se)` — 是否可运行（组调度专用）
* 注释中明确说明了这个宏的区分：
  * 对于 **普通任务（`task`）**：`se_runnable() = !!on_rq`（与上面的 `load` 完全一致）
  * 对于 **任务组（`group`）**：`se_runnable() = grq->h_nr_runnable`（即该任务组中处于可运行状态的任务数量）
* **作用**：用于更新 `runnable_sum`
  * 在组调度中，一个组实体（`task_group` 的 `se`）即使其 `on_rq=1`，如果组内只有一个任务在运行，它的“可运行性”贡献应该比有 5 个任务在运行时要小
  * `h_nr_runnable` 反映了组内的实际排队压力

#### 3. `cfs_rq->curr == se` — 是否正在当前 CPU 上运行
* 这是最精确的 **“运行中”** 标志
  * 即使 `se->on_rq=1`，如果该实体在队列中排队但未获得 CPU（例如，因为有更高优先级的任务正在运行），这个条件为 `false`
* **作用**：用于更新 `util_sum`（利用率）
  * `util_avg` 用来衡量对 CPU 算力的真实消耗，只有当前这一刻真正在 CPU 上执行指令的时间才计入 `util_sum`。它在 EAS（能量感知调度）和 `schedutil` 调频中至关重要

### 与 `blocked_se` 的完整对比

| 特征 | `__update_load_avg_blocked_se` | **`__update_load_avg_se`** |
| :--- | :--- | :--- |
| **调用场景** | 任务睡眠、出队后，或周期性清理时 | 任务入队、出队时，以及每次 `scheduler_tick()` 时钟中断 |
| **`load` 标志** | `0` | `!!se->on_rq`（通常为 `1`） |
| **`runnable` 标志** | `0` | `se_runnable(se)` |
| **`running` 标志** | `0` | `cfs_rq->curr == se` |
| **是否累加新贡献（Step 2）** | **否**（只衰减历史） | **是**（衰减历史 + 累加新时间片） |
| **`*_sum` 变化** | 只降不增 | 先降（老化）后升（新增），动态平衡 |

### 后续处理：`cfs_se_util_change()`
* 这是一个非常重要的性能优化回调
* 它检查 `se->avg.util_est` 是否发生了变化
  * 如果发生了变化，它会向上冒泡更新所在的 `cfs_rq` 的 `util_avg`，并最终可能触发 **`schedutil`** 调频 Governor 重新评估 CPU 频率
* 简单说它的意图是想表明：这个任务变忙了，可能得让 CPU 跑快一点
```c
static inline void cfs_se_util_change(struct sched_avg *avg)
{
    unsigned int enqueued;

    if (!sched_feat(UTIL_EST))
        return;

    /* Avoid store if the flag has been already reset */
    enqueued = avg->util_est;
    if (!(enqueued & UTIL_AVG_UNCHANGED))
        return;

    /* Reset flag to report util_avg has been updated */
    enqueued &= ~UTIL_AVG_UNCHANGED;
    WRITE_ONCE(avg->util_est, enqueued);
}
```
---
```cpp
int __update_load_avg_cfs_rq(u64 now, struct cfs_rq *cfs_rq)
{
	if (___update_load_sum(now, &cfs_rq->avg,
				scale_load_down(cfs_rq->load.weight),
				cfs_rq->h_nr_runnable,
				cfs_rq->curr != NULL)) {

		___update_load_avg(&cfs_rq->avg, 1);
		trace_pelt_cfs_tp(cfs_rq);
		return 1;
	}

	return 0;
}

/*
 * rt_rq:
 *
 *   util_sum = \Sum se->avg.util_sum but se->avg.util_sum is not tracked
 *   util_sum = cpu_scale * load_sum
 *   runnable_sum = util_sum
 *
 *   load_avg and runnable_avg are not supported and meaningless.
 *
 */

int update_rt_rq_load_avg(u64 now, struct rq *rq, int running)
{
	if (___update_load_sum(now, &rq->avg_rt,
				running,
				running,
				running)) {

		___update_load_avg(&rq->avg_rt, 1);
		trace_pelt_rt_tp(rq);
		return 1;
	}

	return 0;
}

/*
 * dl_rq:
 *
 *   util_sum = \Sum se->avg.util_sum but se->avg.util_sum is not tracked
 *   util_sum = cpu_scale * load_sum
 *   runnable_sum = util_sum
 *
 *   load_avg and runnable_avg are not supported and meaningless.
 *
 */

int update_dl_rq_load_avg(u64 now, struct rq *rq, int running)
{
	if (___update_load_sum(now, &rq->avg_dl,
				running,
				running,
				running)) {

		___update_load_avg(&rq->avg_dl, 1);
		trace_pelt_dl_tp(rq);
		return 1;
	}

	return 0;
}

#ifdef CONFIG_SCHED_HW_PRESSURE
/*
 * hardware:
 *
 *   load_sum = \Sum se->avg.load_sum but se->avg.load_sum is not tracked
 *
 *   util_avg and runnable_load_avg are not supported and meaningless.
 *
 * Unlike rt/dl utilization tracking that track time spent by a cpu
 * running a rt/dl task through util_avg, the average HW pressure is
 * tracked through load_avg. This is because HW pressure signal is
 * time weighted "delta" capacity unlike util_avg which is binary.
 * "delta capacity" =  actual capacity  -
 *			capped capacity a cpu due to a HW event.
 */

int update_hw_load_avg(u64 now, struct rq *rq, u64 capacity)
{
	if (___update_load_sum(now, &rq->avg_hw,
			       capacity,
			       capacity,
			       capacity)) {
		___update_load_avg(&rq->avg_hw, 1);
		trace_pelt_hw_tp(rq);
		return 1;
	}

	return 0;
}
#endif /* CONFIG_SCHED_HW_PRESSURE */

#ifdef CONFIG_HAVE_SCHED_AVG_IRQ
/*
 * IRQ:
 *
 *   util_sum = \Sum se->avg.util_sum but se->avg.util_sum is not tracked
 *   util_sum = cpu_scale * load_sum
 *   runnable_sum = util_sum
 *
 *   load_avg and runnable_avg are not supported and meaningless.
 *
 */

int update_irq_load_avg(struct rq *rq, u64 running)
{
	int ret = 0;

	/*
	 * We can't use clock_pelt because IRQ time is not accounted in
	 * clock_task. Instead we directly scale the running time to
	 * reflect the real amount of computation
	 */
	running = cap_scale(running, arch_scale_freq_capacity(cpu_of(rq)));
	running = cap_scale(running, arch_scale_cpu_capacity(cpu_of(rq)));

	/*
	 * We know the time that has been used by interrupt since last update
	 * but we don't when. Let be pessimistic and assume that interrupt has
	 * happened just before the update. This is not so far from reality
	 * because interrupt will most probably wake up task and trig an update
	 * of rq clock during which the metric is updated.
	 * We start to decay with normal context time and then we add the
	 * interrupt context time.
	 * We can safely remove running from rq->clock because
	 * rq->clock += delta with delta >= running
	 */
	ret = ___update_load_sum(rq->clock - running, &rq->avg_irq,
				0,
				0,
				0);
	ret += ___update_load_sum(rq->clock, &rq->avg_irq,
				1,
				1,
				1);

	if (ret) {
		___update_load_avg(&rq->avg_irq, 1);
		trace_pelt_irq_tp(rq);
	}

	return ret;
}
#endif /* CONFIG_HAVE_SCHED_AVG_IRQ */

/*
 * Load avg and utiliztion metrics need to be updated periodically and before
 * consumption. This function updates the metrics for all subsystems except for
 * the fair class. @rq must be locked and have its clock updated.
 */
bool update_other_load_avgs(struct rq *rq)
{
	u64 now = rq_clock_pelt(rq);
	const struct sched_class *curr_class = rq->donor->sched_class;
	unsigned long hw_pressure = arch_scale_hw_pressure(cpu_of(rq));

	lockdep_assert_rq_held(rq);

	/* hw_pressure doesn't care about invariance */
	return update_rt_rq_load_avg(now, rq, curr_class == &rt_sched_class) |
		update_dl_rq_load_avg(now, rq, curr_class == &dl_sched_class) |
		update_hw_load_avg(rq_clock_task(rq), rq, hw_pressure) |
		update_irq_load_avg(rq, 0);
}
```