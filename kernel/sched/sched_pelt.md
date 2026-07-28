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
5.  **平台无关**：通过频率/CPU 不变性设计，使得指标在不同硬件条件下具有可比性
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
  * 注意：`util_sum` 在计算的时候放大了 `1024` 倍，而除数没有放大，所以 `util_avg` 是 `0~1024` 刻度
* 如果不用这个动态除数，满载时算出来的值永远达不到 `1024`，会导致调度器误以为 CPU 还有空闲容量而错误地迁移任务

##### 2. 为什么要用动态除数？（防止振荡）
* 注释中提到的 *“unwanted oscillation in the range [1002..1024[”* 是关键
* **假设使用固定除数 `LOAD_AVG_MAX`**：
  * 如果一个 CPU 在长达 `32ms` 内一直满负荷运行（`util_sum` 达到了当前周期能累加的最大值），计算 `util_avg = util_sum / LOAD_AVG_MAX`
  * 随着 `period_contrib` 从 `0` 逐渐增长到 `1023`，分子（最大和 `Sum_max = (LOAD_AVG_MAX - 1024) + period_contrib`）也在从 `LOAD_AVG_MAX-1024` 逐渐增长到 `LOAD_AVG_MAX-1`
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
* 函数 `__update_load_avg_blocked_se()` 是 PELT 更新路径中一个 **专门用于“老化（Aging）”阻塞任务** 的入口
* 它的设计哲学是：==一个不在 CPU 上运行的任务，其负载不应凭空消失，而应随着时间指数级衰减；但它绝不能在睡眠期间‘刷存在感’（累加新的时间片）==
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

### 清除标志：`cfs_se_util_change()`
* `cfs_se_util_change()` 服务于 UTIL_EST（预估利用率） 机制
  * 当调度器需要知道一个任务的 CPU 需求时，直接读取 `util_avg` 可能不够平滑（因为 PELT 衰减可能导致值抖动）
  * UTIL_EST 机制会记录一个任务运行时的“峰值”或“稳定值”
* 为了提升性能，代码设计了 `UTIL_AVG_UNCHANGED` 标志：
  * 在任务刚入队或某些场景下，如果 `util_avg` 没有变化，就置上这个标志
  * 当 `___update_load_avg()` 因为时间跨过周期而更新了 `util_avg` 时，就调用 `cfs_se_util_change()` 清除这个标志
  * 这样，上层的 `cpu_util()` 或 EAS 算法在读取该值时，就能通过检查这个标志，判断 `util_est` 缓存是否有效，如果标志被清除（即值变了），就知道需要重新计算缓存值了
```c
static inline void cfs_se_util_change(struct sched_avg *avg)
{
    unsigned int enqueued;
    //系统是否开启能量感知调度中的预估利用率功能
    if (!sched_feat(UTIL_EST))
        return;

    /* Avoid store if the flag has been already reset */
    enqueued = avg->util_est;
    if (!(enqueued & UTIL_AVG_UNCHANGED)) //检测“未修改”标志位
        return;  //如果已经被修改了，则返回
    //清除“未修改”标志位，表示已发生变化
    /* Reset flag to report util_avg has been updated */
    enqueued &= ~UTIL_AVG_UNCHANGED;
    WRITE_ONCE(avg->util_est, enqueued);
}
```
## `__update_load_avg_cfs_rq()` 函数
* kernel/sched/pelt.c
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
```
### 调度定点数缩放
* `SCHED_FIXEDPOINT_SHIFT = 10` 这意味着 `util_avg` 和 `load_avg` 等平均值使用 `2^10 = 1024` 作为基数。因此，`util_avg` 的取值范围是 `0 ~ 1024`，其中 `1024` 代表 100% 的 CPU 算力
* `NICE_0_LOAD_SHIFT = 20` 表示任务权重（`se->load.weight`）在内部是以 `2^20 = 1,048,576` 为基数的“高精度”整数存储的
  * 因为 nice 值从 `-20` 到 `19`，其对应的权重（`sched_prio_to_weight[]`）比值很大（`15:88761 ≈ 1:5917`）
    * 如果只用 10 位精度（`1024`）来存储权重，低优先级任务的权重值太小（接近 `1`），整数运算时会丢失精度
    * 用 20 位存储，可以完美保留所有 nice 值之间的细微权重差异
* `scale_load_down(w)` 宏将“高精度（20 位）”的权重值，右移 10 位，收缩回“标准精度（10 位）”的值
  * `w >> SCHED_FIXEDPOINT_SHIFT` 相当于 `w / 1024`
  * `max(2UL, ...)` 确保即使是最低优先级的任务，缩放后的权重也不会小于 `2`，防止在后续除法中除零或归零
* include/linux/sched.h
```cpp
/*
 * Integer metrics need fixed point arithmetic, e.g., sched/fair
 * has a few: load, load_avg, util_avg, freq, and capacity.
 *
 * We define a basic fixed point arithmetic range, and then formalize
 * all these metrics based on that basic range.
 */
# define SCHED_FIXEDPOINT_SHIFT         10
...
# define NICE_0_LOAD_SHIFT      (SCHED_FIXEDPOINT_SHIFT + SCHED_FIXEDPOINT_SHIFT)
...
# define scale_load_down(w)                                     \
({                                                              \
        unsigned long __w = (w);                                \    
                                                                \
        if (__w)                                                \
                __w = max(2UL, __w >> SCHED_FIXEDPOINT_SHIFT);  \
        __w;                                                    \    
})
```

### 1. 三个标志位的传入逻辑（与 `se` 版本的差异）
```c
___update_load_sum(now, &cfs_rq->avg,
				scale_load_down(cfs_rq->load.weight),
				cfs_rq->h_nr_runnable,
				cfs_rq->curr != NULL)
```
| 参数 | `__update_load_avg_se` (任务) | **`__update_load_avg_cfs_rq` (CPU队列)** | 说明 |
| :--- | :--- | :--- | :--- |
| `load` (第3个) | `!!se->on_rq` (值为 0 或 1) | `scale_load_down(cfs_rq->load.weight)` (值为具体权重) | 这是最关键的区别！<br>对于队列，`load`直接传入该CPU上所有任务的 **总权重值** |
| `runnable` (第4个) | `se_runnable(se)` (通常为 1) | `cfs_rq->h_nr_runnable` (该CPU上的可运行任务计数) | 更新 `runnable_sum`，用于计算平均可运行时间 |
| `running` (第5个) | `cfs_rq->curr == se` (是否正在运行) | `cfs_rq->curr != NULL` (只要有任务在运行即为真) | 只要有任务在当前CPU上执行，就更新 `util_sum` |

### 2. 为什么 `___update_load_avg(&cfs_rq->avg, 1)` 里的权重是 `1`？
* 回顾 `accumulate_sum()` 中的累加逻辑：`sa->load_sum += load * contrib`
* **对于 `se`（任务实体）**：
  * 调用 `___update_load_sum()` 时，`load` 参数传的是 `!!se->on_rq`（即 **1**），所以 `se->avg.load_sum` 累加的是 **“未加权”的纯时间贡献**
  * 因此，在后续计算 `load_avg` 时，`___update_load_avg(&se->avg, se_weight(se))` 必须 **乘以任务的权重**
    * 公式为：`se->avg.load_avg = se_weight(se) * se->avg.load_sum / divider`
* **对于 `cfs_rq`（CPU运行队列）**：
  * 调用 `___update_load_sum()` 时，`load` 参数传的是 `scale_load_down(cfs_rq->load.weight)`（即该CPU上所有任务的 **总权重**）
  * 这意味着 `cfs_rq->avg.load_sum` 累加的是 **“已经加权”的总负载贡献**
  * 如果此时再乘以一个非 `1` 的权重，就会造成 **双重加权（Double Weighting）**，数据完全错误
  * 因此，在计算 `cfs_rq` 的 `load_avg` 时，`___update_load_avg()` 必须传入权重 `1`
    * 公式变为：`cfs_rq->avg.load_avg = 1 * cfs_rq->avg.load_sum / divider`

### 数据流向的全局视角
* 把这个逻辑放进 PELT 的整个流水线中，看得更清楚：
1. `cfs_rq->load.weight` 维护着该 CPU 上所有 `on_rq` 任务的权重之和（比如 2 个普通任务，权重各 `1024`，总和为 `2048`）
2. `__update_load_avg_cfs_rq()` 在时钟中断或负载均衡时被调用：
   - 它把总权重 `2048` 通过 `scale_load_down` 缩放为 `2`（因为 `2048 >> 10 = 2`），传给底层 `accumulate_sum()`
   - 底层将 `load * contrib` 累加到 `cfs_rq->avg.load_sum` 中
   - 最后，用权重 `1` 计算平均值，得到 `cfs_rq->avg.load_avg`
3. 这个 `load_avg` 直接代表了该 CPU 当前的“系统总负载压力”，是负载均衡器（`find_busiest_group()`）判断哪个 CPU 最繁忙的最核心依据

## 实时负载更新 `update_rt_rq_load_avg()` 函数
* `update_rt_rq_load_avg()` 是 PELT 机制在 **实时调度类（RT, Real-Time）** 上的适配实现
* RT 调度类有自己的负载均衡策略（通常基于 RT 任务的 `prio` 和 CPU 亲和性），完全不需要 PELT 的 `load_avg` 来做决策

### 既然 RT 不用 `load_avg` 做负载均衡，那为什么要维护这套 PELT 数据呢？
* 主要为性能调频（DVFS）和能量感知调度（EAS）服务
  * `rq->avg_rt.util_avg` 在 `schedutil` 调频 Governor 中被读取。当 RT 任务运行时，`util_avg` 会迅速升高到接近 `1024`，驱使 CPU 频率快速提升，以保证 RT 任务的低延迟
  * 在大小核（big.LITTLE）架构上，EAS 会利用 `util_avg` 来判断 RT 任务是否适合放在高性能大核上运行
  * 所以，这个函数虽然在 RT 负载均衡中没有用，但它负责将 RT 任务的“CPU 占用率”实时反映到系统的频率和功耗控制上
* 简单来说，`update_rt_rq_load_avg()` 利用 PELT 的数学框架，将 RT 任务的“二值运行状态”转化成一个平滑的 `util_avg` 曲线，供性能和功耗管理模块使用，而不涉及其自身的调度决策

### `update_rt_rq_load_avg()` 代码实现
```cpp
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
```

* `load`、`runnable`、`running` 三个标志位被传入了 **完全相同的值**

| 参数 | CFS 版本（`cfs_rq`） | **RT 版本（`rq->avg_rt`）** | 原因分析 |
| :--- | :--- | :--- | :--- |
| `load` | `scale_load_down(总权重)` | `running` | RT 任务没有优先级权重（Nice值）的概念，只有固定优先级。只要 RT 类在运行，它对系统造成的“负载压力”就是满额的（`1`），否则为 `0` |
| `runnable` | `h_nr_runnable`（任务计数） | `running` | 在标准的 RT 内核中（未开启 `CONFIG_RT_GROUP_SCHED` 组调度），RT 任务一旦就绪，几乎会立即抢占 CPU 并运行。因此“可运行”和“正在运行”在时间上几乎重合 |
| `running` | `curr != NULL` | `running` | 表示当前 CPU 上是否有 RT 任务在运行 |

* **结论**：对于 RT，PELT 只追踪一个纯粹的“**CPU 繁忙时间片**”
  * 如果当前 CPU 被 RT 任务占用（`running=1`），则三个 `*_sum` 同时累加
  * 如果空闲或运行 CFS 任务（`running=0`），则三个 `*_sum` 同时衰减
* 源码注释中给出了 RT 特有的数据关系：
```c
util_sum = cpu_scale * load_sum
runnable_sum = util_sum
load_avg 和 runnable_avg 不支持且无意义
```

#### 1. `util_sum = cpu_scale * load_sum`
* `load_sum` 在这里实际上存储的是 **未缩放的时间贡献**（由 `___update_load_sum()` 中的 `running * contrib` 累加得到）
* `util_sum` 需要表示对 CPU 算力的实际消耗。因为不同性能的 CPU 核心（`cpu_scale`）运行同样的时间，消耗的“算力总量”不同
  * 公式表明，`util_sum` 是在 `load_sum` 的基础上乘以了 CPU 的容量缩放因子（通常 `cpu_scale = 1024` 代表最高性能核心）
* 在 RT 的特定逻辑中（可能是更新 `util_sum` 的调用处），会执行这个缩放操作，将纯时间转换为算力消耗

```c
1 kernel/sched/pelt.c|486| <<update_other_load_avgs>> return update_rt_rq_load_avg(now, rq, curr_class == &rt_sched_class) |                    
2 kernel/sched/rt.c|1671| <<set_next_task_rt>> update_rt_rq_load_avg(rq_clock_pelt(rq), rq, 0);
3 kernel/sched/rt.c|1731| <<put_prev_task_rt>> update_rt_rq_load_avg(rq_clock_pelt(rq), rq, 1);
4 kernel/sched/rt.c|2443| <<switched_to_rt>> update_rt_rq_load_avg(rq_clock_pelt(rq), rq, 0);
5 kernel/sched/rt.c|2539| <<task_tick_rt>> update_rt_rq_load_avg(rq_clock_pelt(rq), rq, 1);
```
* 从 `update_rt_rq_load_avg()` 的调用者可以看到传入的 `now` 都是经过 `rq_clock_pelt(rq)` 缩放过的，但无需经过加权
  * 第一个调用者 `update_other_load_avgs()` 后面会讲，它的 `now = rq_clock_pelt(rq)`
#### 2. `runnable_sum = util_sum`
* 由于 RT 任务的“可运行”等同于“即将运行/正在运行”，两者在时间轴上无法区分，因此这两个值被设为相等

#### 3. `load_avg` 和 `runnable_avg` 不支持且无意义
* 为什么不支持 `load_avg`？
  * `load_avg` 是 CFS 用来衡量“带权重的系统压力”的，用于跨 CPU 的负载均衡（`find_busiest_group()`）
  * RT 调度器有自己的负载均衡策略（通常基于 RT 任务的 `prio` 和 CPU 亲和性），完全不需要 PELT 的 `load_avg` 来做决策
* 为什么不支持 `runnable_avg`？
  * 同上，RT 不需要它来调整抢占或时间片

### `___update_load_avg(&rq->avg_rt, 1)` 中的权重为什么是 `1`？
* 在 `update_rt_rq_load_avg()` 中：
1. **累加时**：`load` 参数传入的是 `running`（值为 `0` 或 `1`）
2. **累加公式**：`sa->load_sum += 1 * contrib`（此时 `load_sum` 是 **未加权的纯时间总和**）
3. **平均时**：如果传入 `weight = 1`，公式变为 `load_avg = 1 * (未加权总和) / divider`。**这计算出来的是一个“未加权的平均 CPU 占用时间百分比”**
* **RT 不需要 `load_avg`**：RT 使用基于固定优先级的抢占式调度。它的负载均衡逻辑（`pull_rt_task()`）依据的是任务本身的 `prio` 优先级和 CPU 亲和性（`cpumask`），与 PELT 算出的 `load_avg` 数值无关
* 既然没有任何上层代码会去读取 `rq->avg_rt.load_avg`，那么计出什么数值都无所谓。内核选择传入 `1`，只不过是为了凑齐 `___update_load_avg` 的函数参数格式，并得到一个在数学上自洽（不溢出、不除零）的占位值罢了
* RT 维护这套 PELT 数据的唯一目的，是为了得到 **`rq->avg_rt.util_avg`**
  * `util_avg` 反映的是 CPU 被 RT 任务占用的 时间比例（`0 ~ 1024`）
  * 这个值会被 `schedutil` 调频 Governor 读取，用来在 RT 任务运行时提升 CPU 频率
  * 对于 `util_avg` 的计算，RT 的 `runnable` 和 `running` 参数也传的是 `running`，因此它的值等于“纯运行时间占比”

## DL 调度负载更新 `update_dl_rq_load_avg()` 函数
* `update_dl_rq_load_avg()` 将 DL 任务的二值运行状态（`running`），通过 PELT 的指数衰减框架，转化成平滑的 CPU 利用率信号（`util_avg`），供频率缩放和功耗管理模块使用
* 它不参与 DL 任务的调度决策（那由 `dl_rq` 的带宽和死锁检测负责），而是为系统的性能调频（DVFS）和功耗控制（EAS）提供实时数据
* `load_avg` 对 DL 调度而言也是无意义的
  * DL 的决策依据是带宽（Bandwidth），即 `runtime / period`
  * DL 调度器使用 `dl_bw` 和 `dl_rq` 进行准入控制（Admission Control）和负载均衡，而完全不依赖 PELT 计算出的加权负载（`load_avg`）来决定抢占或迁移
* `runnable_avg` 对 DL 调度而言也是无意义的
  * DL 任务的运行具有严格的确定性和突发性
  * 它们要么在运行（消耗预算），要么被挂起等待下一个周期，不存在像 CFS 那样长的排队等待时间（`runnable` 状态通常极短）
  * 因此追踪 `runnable_avg` 对调度没有参考价值
* `util_sum = cpu_scale * load_sum`
  * 与 RT 完全一样：`load_sum` 存储原始时间贡献，`util_sum` 乘以 CPU 容量因子（`cpu_scale`）后，用于反映对 CPU 算力的实际消耗
```cpp
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
```

## 硬件负载更新 `update_hw_load_avg()` 函数
* `update_hw_load_avg()` 函数是 PELT 框架在 **硬件压力（Hardware Pressure）** 追踪上的应用，它代表了一种完全不同的追踪维度
* 它与之前看过的 CFS、RT、DL 函数最本质的区别在于：**RT/DL 追踪的是“时间”（是否在运行），而 HW 追踪的是“算力缺口”（Capacity loss）**

```cpp
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
```

### 理解“硬件压力”是什么
* 注释中给出了明确的计算公式：
\[
\text{delta capacity} = \text{actual capacity} - \text{capped capacity due to a HW event}
\]

* `actual capacity`：CPU 在最高频率/无干扰下的理论最大算力（通常归一化为 `1024`）
* `capped capacity`：由于硬件事件（如 **热节流 Thermal Throttling**、**电源限制 Power Limit**、或外部中断风暴）导致 CPU 实际可用的当前最大算力
* 举例：如果一颗大核最高频率算力为 `1024`，但因为 CPU 温度过高，硬件强制降频，导致当前最高可用算力只有 `512`。那么这个 CPU 的 **硬件压力（缺口）** 就是 `1024 - 512 = 512`

### 为什么三个参数全是 `capacity`？而且这个值可以大于 1？
* 回顾函数调用：
```c
___update_load_sum(now, &rq->avg_hw,
                   capacity,   //load 参数
                   capacity,   //runnable 参数
                   capacity)   //running 参数
```
* 在 CFS/RT/DL 中，这三个参数是 `0` 或 `1`（标志位）。但在这里，它们是一个 **`u64` 类型的 `capacity` 值**（取值范围通常为 `0~1024`）
* ==核心区别在于：HW 压力不是一个“是否发生”的状态，而是一个“有多严重”的连续量==

1. `load` 参数的语义变化：
   - 在 CFS 中，`load` 代表“权重”
   - 在这里，`load` 代表“当前的算力缺口大小”。如果缺口是 512，那么累加公式变为：`sa->load_sum += 512 * contrib`（而不是 `1 * contrib`）
   - 这意味着，**硬件压力造成的负载贡献，与压力的严重程度成正比**。压力越大，`load_sum` 上升得越快
2. `runnable` 和 `running` 的意义：
   - 对于 HW 压力，不存在“可运行”或“正在运行”的概念。将这三个值统一设为 `capacity`，是为了利用 PELT 的衰减框架，将这个 **连续的算力缺口值**，像累积“负载”一样进行指数加权平滑

### 为什么 `___update_load_avg(&rq->avg_hw, 1)` 中的权重是 1？
* 这与 CFS `cfs_rq` 的逻辑类似，但动机不同：
  * **累加时已量化**：因为 `load` 参数传入的是具体的 `capacity` 值（如 `512`），所以 `rq->avg_hw.load_sum` 累加的是 **已经按压力大小加权后的总和**
  * **平均时无需再加权**：如果此时传入一个非 1 的权重，会造成双重量化。因此，必须传入权重 `1`，公式为：
    ```c
    rq->avg_hw.load_avg = 1 * rq->avg_hw.load_sum / divider
    ```
* **最终计算出的 `load_avg` 含义**：在过去的某个时间窗口内，该 CPU 平均损失了多少算力（范围 0~1024）。如果值为 512，表示平均有 50% 的算力被硬件吃掉了

### 注释中对比 RT/DL 的含意
* 注释特意强调：
> *"unlike rt/dl ... util_avg is binary, HW pressure is time weighted 'delta' capacity"*

| 特性 | RT/DL（实时/截止时间） | **HW Pressure（硬件压力）** |
| :--- | :--- | :--- |
| **追踪对象** | 任务是否在运行（0 或 1） | 算力缺口大小（0~1024 连续值） |
| **输入值** | `running`（标志位） | **`capacity`（量化值）** |
| **物理意义** | 该 CPU 有多“忙” | 该 CPU 有多“被限制” |
| **产生的信号** | `util_avg`（用于调频） | **`load_avg`（用于负载均衡规避）** |

* `update_hw_load_avg()` 利用 PELT 的数学框架，创造性地将 **硬件算力限制** 转换成了 **随时间衰减的平均负载信号**
* 它的设计精髓在于：
1. **非二进制输入**：将连续的 `capacity` 值作为累加权值，精确反映压力大小
2. **利用 CFS 的负载路径**：因为 CFS 的 `load_avg` 本身就是加权累加，正好适合承载“量化后的压力”这个概念
3. **服务于隔离决策**：最终结果告诉调度器：“某些 CPU 虽然看起来很闲，但实际上被硬件卡住了脖子，别把任务放上去。”

### 这个信号用在哪里？（上层逻辑）
* `rq->avg_hw.load_avg` 最终会汇聚到调度域（`sched_domain`）的硬件压力统计中
* 当负载均衡器（`find_busiest_group()`）看到某个 CPU 的硬件压力很高（比如 `load_avg` 接近 512）时，即使该 CPU 上的任务 `util_avg` 不高，均衡器也会 **避免向该 CPU 迁移新任务**，甚至尝试把任务迁走。因为继续往一个正在“被硬件降频”的 CPU 上堆任务，只会让任务跑得更慢，毫无意义
* 这也是为什么它不叫 `util_avg` 而沿用 `load_avg` 的原因 —— **它代表系统在当前算力约束下，需要被“打折扣”看待的残余处理能力**

## 中断平均负载更新 `update_irq_load_avg()`
* `update_irq_load_avg()` 专门用于追踪**中断（IRQ）和软中断（SoftIRQ）**对 CPU 造成的负载
* 它与之前我们分析的所有函数（CFS/RT/DL/HW）有一个根本性的不同：**它在一个更新周期内，连续调用了两次 `___update_load_sum()`**
```cpp
//kernel/sched/sched.h
#define cap_scale(v, s)         ((v)*(s) >> SCHED_CAPACITY_SHIFT)

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
```

### 第一步：特殊的时间缩放（`cap_scale`）
* 在调用底层更新函数之前，首先对 `running` 时间进行了两次缩放：

```c
running = cap_scale(running, arch_scale_freq_capacity(cpu_of(rq)));
running = cap_scale(running, arch_scale_cpu_capacity(cpu_of(rq)));
```
* **为什么需要缩放？**  
* 中断处理消耗的是 CPU 的 **算力（Capacity）**。如果 CPU 当前处于降频状态（频率低）或运行在小核上（性能弱），那么同样的 `1ms` 中断处理时间，消耗的“算力总量”是不同的
* 通过 `cap_scale`（乘以频率因子和 CPU 容量因子，再右移 `SCHED_CAPACITY_SHIFT`），`running` 时间被转换为 **以 `1024` 为基数的算力消耗单位**，使得在不同频率、不同核心上追踪的 IRQ 负载具有可比性（即具备“CPU 不变性”）

### 第二步：核心机制——两次更新（双段衰减法）
* 这是函数最精妙的部分，注释中明确解释了原因：

> *"We know the time that has been used by interrupt since last update but we don't know when."*  
> （我们知道上次更新以来中断用了多少时间，但 **不知道具体发生在哪一刻**。）

* 标准 PELT 算法假设负载均匀分布在时间轴上。但对于中断，它通常是 **突发（Burst）** 的
* 为了精确老化，内核做了一个最坏情况（或最合理）的假设：**假设所有中断处理都发生在刚刚过去的这一小段时间内（即紧挨着当前时刻）**
* 为了实现这个假设，代码将整个更新周期切成了两段：

#### 第一段：衰减历史（不含中断）
```c
ret = ___update_load_sum(rq->clock - running, &rq->avg_irq,
                          0, 0, 0);
```
- **时间戳**：`rq->clock - running`。这意味着我们把历史的终点设置在“中断爆发之前”
- **标志位**：`(0, 0, 0)`。这意味着在这段从上次更新到“中断爆发前”的时间里，**只做指数衰减（老化历史）**，不累加任何新的贡献
- **效果**：`rq->avg_irq` 的 `*_sum` 值按照这段“无中断”的时间长度进行比例衰减

#### 第二段：累加中断突发
```c
ret += ___update_load_sum(rq->clock, &rq->avg_irq,
                           1, 1, 1);
```
- **时间戳**：`rq->clock`（当前时刻）
- **标志位**：`(1, 1, 1)`。注意，此时底层函数内部计算的时间差是：`delta = rq->clock - (rq->clock - running) = running`
- **效果**：这段 `running` 时间被当作“满载运行”状态，直接累加到 `load_sum`、`runnable_sum` 和 `util_sum` 中，并且因为时间很短（就在刚才），它的衰减几乎可以忽略不计

### 第三步：这样做的数学意义是什么？
* 传统的单次更新会把中断时间平均分配到整个周期里，导致中断产生的负载信号被“稀释”和平滑，响应不够灵敏
* **双段更新法等效于数学上的“分段积分”：**
1.  先将历史负载按无中断时间衰减
2.  再在时间轴的末端（`now` 时刻）瞬间加上一大块突发负载

* 这使得 **`rq->avg_irq.util_avg` 能够对最近的中断爆发做出极其快速的反应**，同时又能保持 PELT 原有的长期历史衰减特性
* 这对于依赖该信号进行调频（DVFS）的场景至关重要——如果 CPU 刚处理完大量网卡中断，`schedutil` 需要立刻知道并提高频率，以应对后续可能到来的数据包

### 第四步：为什么标志位是 `(1,1,1)` 但注释说 `load_avg` 无意义？
* 注释中提到：

> load_avg and runnable_avg are not supported and meaningless.

* 既然无意义，为什么还要传 `1,1,1`（即同时累加 load 和 runnable）？
  - **代码复用的副作用**：`___update_load_sum()` 是通用函数，它同时维护三个独立的累加器（`load_sum`, `runnable_sum`, `util_sum`）。我们无法只单独累加 `util_sum` 而不累加其他两个（除非修改函数接口）
  - **真正有用的只有 `util_sum`**：在第二段调用中，`running=1` 使得 `util_sum += contrib << SCHED_CAPACITY_SHIFT`，这是我们真正需要的 IRQ 利用率信号
  - **其他两个是无用占位**：虽然 `load_sum` 和 `runnable_sum` 也增加了，但上层逻辑（负载均衡、`schedutil`）只会读取 `rq->avg_irq.util_avg`，另外两个值仅仅是为了满足数据结构的一致性，没有任何实际用途（类似于 RT/DL 场景）

### 第五步：最终平均值计算

```c
if (ret) {
    ___update_load_avg(&rq->avg_irq, 1);
    trace_pelt_irq_tp(rq);
}
```
- 当两次调用中只要有任意一次跨过了周期边界（`ret > 0`），就调用 `___update_load_avg()` 刷新平均值
- 传入权重 `1`，因为 `load_sum` 累加时传的是 `1`（未加权），所以计算时也无需再加权

### IRQ 与其他 PELT 信号的对比

| 特性 | CFS/RT/DL | **IRQ（中断）** |
| :--- | :--- | :--- |
| **输入数据** | 运行状态（0/1）或压力大小（capacity） | **运行时间（纳秒）** |
| **时间假设** | 均匀分布在整个周期内 | **假设集中在最近时刻（突发）** |
| **更新方式** | 单次调用 `___update_load_sum()` | **两次调用（先衰减无中断段，再累加突发段）** |
| **缩放处理** | 部分在外部、部分在内部 | **显式使用 `cap_scale` 缩放频率和 CPU 容量** |
| **主要用途** | 负载均衡、调频 | **调频（DVFS）**，应对中断引起的性能需求变化 |

* 这个函数的设计：**中断既是突发的，又对延迟极其敏感。** 通过两次更新的“欺骗性”技巧，PELT 成功地在标准指数衰减框架下，实现了对突发负载的快速响应

## PELT 时钟

### 三条时钟速查

| 时钟 | 含义 |
|---|---|
| `rq->clock` | 原始的 `sched_clock_cpu()`，纯粹的墙钟时间 |
| `rq->clock_task` | 扣掉 irq/steal 时间后的任务时间 |
| `rq->clock_pelt` | 按 CPU capacity × 频率缩放过的时间轴，rq idle 时重新同步回 `clock_task` |
| `rq->clock_idle` | 进入 idle 那一刻的 `rq_clock()` 快照（墙钟轴） |
| `rq->clock_pelt_idle` | 进入 idle 那一刻的 `rq_clock_pelt()` 快照（PELT 轴） |
| `rq->lost_idle_time` | 因 `util_sum` 饱和而"无法偿还"的算力欠账累计值 |

### 路径一：即将进入空闲
* `set_next_task_idle()` 是调度器选中 idle 进程作为下一个任务，即进入空闲的入口
```c
set_next_task_idle()
-> update_idle_rq_clock_pelt(rq)
      if (util_sum >= divider) //如果队列满载
      -> rq->lost_idle_time += rq_clock_task(rq) - rq->clock_pelt; //记录算力衰减的欠账
   -> _update_idle_rq_clock_pelt(rq)
      -> rq->clock_pelt = rq_clock_task(rq); //重置量具的零点
      -> u64_u32_store(rq->clock_idle, rq_clock(rq)) //保存进入 idle 那一刻的 rq_clock() 快照（墙钟轴）
      -> u64_u32_store(rq->clock_pelt_idle, rq_clock_pelt(rq)); //保存进入 idle 那一刻的 rq_clock_pelt() 快照（PELT 轴）
                                               return rq->clock_pelt - rq->lost_idle_time;
```

### 路径二：更新队列的 PELT 时间
```c
update_rq_clock() //目前多达 49 个调用处，分布在调度器的关键路径上
   clock = sched_clock_cpu(cpu_of(rq)); //获取硬件时钟
   delta = clock - rq->clock; //增量 = 硬件时钟 - 队列记录的时钟
   rq->clock += delta;        //更新队列时钟记录
-> update_rq_clock_task(rq, delta)
      irq_delta = irq_time_read(cpu_of(rq)) - rq->prev_irq_time; //获取中断时间
      rq->prev_irq_time += irq_delta;  //更新中断时间记录，为下一次做准备
      delta -= irq_delta;              //从增量中刨去中断时间
      steal = prev_steal = paravirt_steal_clock(cpu_of(rq)); //获取虚拟化时钟
      steal -= rq->prev_steal_time_rq; //得到虚拟化偷取的时间
      rq->prev_steal_time_rq = prev_steal; //更新偷取时间的记录，为下一次做准备
      delta -= steal;                  //从增量中刨去虚拟化偷取的时间
      rq->clock_task += delta; //更新 clock_task（这是排除中断和偷取后的净任务运行时间）
   -> update_rq_clock_pelt(rq, delta)
         if (unlikely(is_idle_task(rq->curr))) { //即便是 idle 任务也要更新 PELT 时钟
         -> _update_idle_rq_clock_pelt(rq);
            -> rq->clock_pelt = rq_clock_task(rq); //CPU 空闲时，PELT 时钟与任务时钟 gap 清零，确保下次计账不重叠
            -> u64_u32_store(rq->clock_idle, rq_clock(rq)) //保存进入 idle 那一刻的 rq_clock() 快照（墙钟轴）
            -> u64_u32_store(rq->clock_pelt_idle, rq_clock_pelt(rq)); //保存进入 idle 那一刻的 rq_clock_pelt() 快照（PELT 轴）
                                                     return rq->clock_pelt - rq->lost_idle_time;
            return; //update_rq_clock_pelt() 返回
         }
         delta = cap_scale(delta, arch_scale_cpu_capacity(cpu_of(rq)));
         delta = cap_scale(delta, arch_scale_freq_capacity(cpu_of(rq)));
         rq->clock_pelt += delta;
```

### `rq_clock_pelt()` 中减去 `lost_idle_time` 的意义
* `rq_clock_pelt(rq)` 是一条按算力与频率折算的、每 `rq` 私有的虚拟时钟：
  * **忙时** 按实际吞吐比例慢走，
  * **空闲** 时以 1:1 追平 `clock_task` 并归还忙时欠下的时间，唯有 `rq` 饱和到无空闲可归还时，欠账才被记入 `lost_idle_time` 永久扣除
  * 所有 PELT 的 `last_update_time` 都记在这条轴上
1. `lost_idle_time` 记录的是因算力衰减造成的 `clock_pelt` 的欠账，在空闲时如果 `clock_pelt` 不跟随 `clock_task` 推进，那么在下次计算 `lost_idle_time` 会把空闲时间计算在 `clock_pelt` 的欠账内，并且 `+=` 会把累计 gap 反复积分，重复计账，甚至会让 `rq_clock_pelt()` 返回负数 -- 这从根本上是错误的，空闲时间与算力衰减造成的欠账是两个概念
2. 阻塞任务负载 **是否** 衰减，由 `__update_load_avg_blocked_se()` 传入的三个 `0` 决定，与 `clock_pelt`/`clock_task` 的同步无关；然而阻塞任务负载衰减 **多少**，是否正确，取决于同步语句把空闲期间的 PELT 轴速率钉在 1:1 —— 这一点与同步高度相关
   * 回顾之前 `__update_load_avg_blocked_se()` 函数给 `___update_load_sum()` 的最后三个参数是 `0`，于是 `delta` 不累加到 `*_sum` 上，只对 `*_sum` 进行衰减
3. 空闲期间衰减并非"持续"发生：tickless idle 下 `clock_pelt` 冻结，衰减由 tick、`sched_balance_update_blocked_averages()` 与 `migrate_se_pelt_lag()` 惰性追平
* kernel/sched/pelt.h
```cpp
static inline u64 rq_clock_pelt(struct rq *rq)
{
	lockdep_assert_rq_held(rq);
	assert_clock_updated(rq);

	return rq->clock_pelt - rq->lost_idle_time;
}
```
### 重置 PELT 时间 `_update_idle_rq_clock_pelt()`
* 该函数在路径一和二上都会调用
* 因为 CPU 已经空闲了，不会再产生新的计算工作，每次空闲同步都把 PELT 时钟（缩放时钟）与 任务时间的 gap 清零，重置量具的零点，使得下一次测到的 `clock_task - clock_pelt` 必然是一段全新的、与之前互不重叠的欠账
```cpp
/* The rq is idle, we can sync to clock_task */
static inline void _update_idle_rq_clock_pelt(struct rq *rq)
{	//CPU 空闲时，强制同步 PELT 时钟与任务时钟，直接覆盖，强行对齐
	rq->clock_pelt  = rq_clock_task(rq);
	//进入 idle 那一刻的 rq_clock() 快照（墙钟轴）
	u64_u32_store(rq->clock_idle, rq_clock(rq));
	/* Paired with smp_rmb in migrate_se_pelt_lag() */
	smp_wmb();
	u64_u32_store(rq->clock_pelt_idle, rq_clock_pelt(rq)); //进入 idle 那一刻的 rq_clock_pelt() 快照（PELT 轴）
}
```
* `rq->clock_idle`：进入 idle 那一刻的 `rq_clock(rq)`，即墙钟读数。作用是给远端 CPU 一个测量"这个 CPU 已经 idle 多久"的基准点：
  ```c
  sched_clock_cpu(cpu_of(rq)) - u64_u32_load(rq->clock_idle)
  ```
* `rq->clock_pelt_idle`：同一刻的 `rq_clock_pelt(rq)`（已减去 `lost_idle_time`），即 PELT 轴的读数。它是远端 CPU 推算的起点。

### 更新空闲时 PELT 时间
* 该函数仅在 **路径一：即将进入空闲** 上调用
```cpp
/*
 * When rq becomes idle, we have to check if it has lost idle time
 * because it was fully busy. A rq is fully used when the /Sum util_sum
 * is greater or equal to:
 * (LOAD_AVG_MAX - 1024 + rq->cfs.avg.period_contrib) << SCHED_CAPACITY_SHIFT;
 * For optimization and computing rounding purpose, we don't take into account
 * the position in the current window (period_contrib) and we use the higher
 * bound of util_sum to decide.
 */
static inline void update_idle_rq_clock_pelt(struct rq *rq)
{ //1. 计算一个“饱和阈值”
	u32 divider = ((LOAD_AVG_MAX - 1024) << SCHED_CAPACITY_SHIFT) - LOAD_AVG_MAX;
	u32 util_sum = rq->cfs.avg.util_sum;
	util_sum += rq->avg_rt.util_sum;
	util_sum += rq->avg_dl.util_sum;
	//2. 判定是否“超载/满负荷”
	/*
	 * Reflecting stolen time makes sense only if the idle
	 * phase would be present at max capacity. As soon as the
	 * utilization of a rq has reached the maximum value, it is
	 * considered as an always running rq without idle time to
	 * steal. This potential idle time is considered as lost in
	 * this case. We keep track of this lost idle time compare to
	 * rq's clock_task.
	 */
	if (util_sum >= divider) //成立说明在过去时间段内，该 CPU 的算力被 CFS、RT、DL 任务完全占满（100% 利用率）
		rq->lost_idle_time += rq_clock_task(rq) - rq->clock_pelt; //3. 记录算力衰减的欠账
	//4. 最后执行硬重置
	_update_idle_rq_clock_pelt(rq);
}
```
#### 1. 理解 `divider`（饱和阈值）
* 注释中解释了这是用于判断 CPU 在过去是否 **完全满载（fully busy）** 的阈值
* 为了在不进行浮点运算的前提下，快速判断 CPU 是否“完全繁忙（Fully Busy）”而构造的一个 **安全阈值（安全裕度）**

##### 第一步：回顾 PELT 的“完全繁忙”判定标准
* 在 PELT 中，`util_avg` 的取值范围是 `0~1024`（`1024` 代表 100% 算力被耗尽）。之前我们分析过，`util_sum` 和 `util_avg` 的关系是：

\[
util\_avg = \frac{util\_sum}{Divider\_Real}
\]

* 其中，**真实的除数** \( Divider\_Real \) 是动态的，取决于当前周期内已度过的时间（`period_contrib`）：

\[
Divider\_Real = LOAD\_AVG\_MAX - 1024 + period\_contrib
\]

* 因此，**CPU 完全繁忙（`util_avg >= 1024`）** 的数学条件是：

\[
util\_sum \ge 1024 \times (LOAD\_AVG\_MAX - 1024 + period\_contrib)
\]

##### 第二步：消除动态变量 `period_contrib`
* 由于 `period_contrib` 在每次更新时都在 `[0, 1024)` 范围内变化，如果直接用这个动态值做判断，每次都需要进行额外的乘法和加法。内核为了 **性能优化**，希望用一个 **固定的常量阈值** 来代替这个动态计算
* 于是，代码选择了忽略 `period_contrib`，并选取了它的 **最小值 `0`**，得到理论固定阈值：

\[
Threshold\_0 = 1024 \times (LOAD\_AVG\_MAX - 1024)
\]

* 最小值取 `0` 是因为 `period_contrib` 越小，右边的阈值就越小，条件 `util_sum >= 阈值` 就越 **容易** 满足。这是一种“宽松”的判定策略，确保不会因为舍入误差而漏掉繁忙状态

##### 第三步：引入 `- LOAD_AVG_MAX`（安全裕度）
* 如果我们直接使用 \( 1024 \times (LOAD\_AVG\_MAX - 1024) \) 作为阈值，在边界情况下，`util_sum` 可能会因为整数运算的截断误差而略微偏小，导致本该判定为繁忙的 CPU 被误判为空闲
* 为了避免这种“漏判”，内核开发者决定 **进一步降低阈值**，增加一个“安全裕度（Safety Margin）”。他们选择减去一个 `LOAD_AVG_MAX`（大约 `47742`）：

\[
divider = 1024 \times (LOAD\_AVG\_MAX - 1024) - LOAD\_AVG\_MAX
\]

* 为什么偏偏减去 `LOAD_AVG_MAX`？
  - `LOAD_AVG_MAX` 本身代表了 **单个调度实体（`sched_entity`）在完全满载时所能贡献的最大负载值**（大约 `47742`）
  - 在 `update_idle_rq_clock_pelt()` 中，`util_sum` 是三个独立部分的总和：`cfs.util_sum + rt.util_sum + dl.util_sum`。每一部分在各自的维护过程中都可能引入微小的舍入误差
  - 减去一个 `LOAD_AVG_MAX` 相当于 **把判定门槛降低了一个“最大单实体”的量级**。这极大概率地消除了因累加舍入误差导致的误判，确保只要 CPU 的真实利用率接近 100%，就一定会触发 `lost_idle_time` 的累加，从而正确地补偿空闲时间

##### 第四步：结合代码注释验证
* 再看注释：
> *"For optimization and computing rounding purpose, we don't take into account the position in the current window (period_contrib) and we use the higher bound of util_sum to decide."*

- **“higher bound”** 在这里的语义是：**更容易触发繁忙判定的上限边界**。通过主动降低阈值（减去 `LOAD_AVG_MAX`），代码实现了这一“宽松上限”
- 其工程哲学是：**宁可把 CPU 错误地判定为“完全繁忙”（导致 `lost_idle_time` 多累加一点），也绝不能把繁忙的 CPU 判定为“有空闲”**
- 因为如果漏判，会导致空闲时间未被正确补偿，进而使后续的负载衰减过快，严重影响调度准确性

##### 数值验证
* 假设 `LOAD_AVG_MAX ≈ 47742`（这是一个众所周知的常数）：
  - \( 1024 \times (47742 - 1024) = 1024 \times 46718 = 47,839,232 \)
  - 减去 `LOAD_AVG_MAX`：`47,839,232 - 47,742 = 47,791,490`
* 最终 `divider` 约为 **47,791,490**
  * 这个数值比理论上的最小阈值（`47,839,232`）还要低约 0.1%
  * 这个微小的降低，足以容纳整数运算带来的全部舍入误差，确保判定条件的健壮性

##### `divider` 表达式的推导逻辑链是：

1. **理论公式**：`util_sum >= 1024 * (LOAD_AVG_MAX - 1024 + period_contrib)`  
2. **忽略动态变量**：取 `period_contrib = 0`，得 `1024 * (LOAD_AVG_MAX - 1024)`  
3. **加入安全裕度**：减去一个 `LOAD_AVG_MAX`，以抵消多部分求和时的舍入误差，得到最终形式：  
   \[
   \boxed{((LOAD\_AVG\_MAX - 1024) \times 1024) - LOAD\_AVG\_MAX}
   \]

#### 2. 为什么“满载”时要增加 `lost_idle_time`？
- **场景分析**：CPU 在过去一直 100% 繁忙。此时，由于频率缩放（比如降频了），`clock_pelt` 的前进速度比 `clock_task` **慢**（因为 `clock_pelt` 乘以了小于 1 的系数）
- **即将发生的事**：`_update_idle_rq_clock_pelt()` 接下来会执行 `clock_pelt = clock_task`，这意味着 **`clock_pelt` 会瞬间追加上一大块之前“少走”的时间**
- **如果不做处理**：`clock_pelt` 突然向前跳了一大步，之前因为降频欠的账不还就丢失了，导致记录的 PELT 时间比真实的快，然而实际上 CPU 并没有干那么多活。这种偏差会影响到负载均衡的正确性

##### 数值证明

* 取恒定缩放因子 `0.5`（比如一个跑在半频的 CPU），忽略 irq/steal，周期为 **10ms 满载忙 + 10ms 空闲**，且忙时 `util_sum >= divider`（每次都触发计账）

| 时刻 | clock_task | clock_pelt | lost_idle_time | rq_clock_pelt() |
|---|---|---|---|---|
| 周期起点 | T | T | L | T−L |
| 忙 10ms 后 | T+10 | T+5 | L | T+5−L |
| 进 idle，计账 | T+10 | **T+10**（同步） | **L+5** | T+5−L |
| 空闲 10ms 后 | T+20 | T+20（同步） | L+5 | T+15−L |

* 两点值得注意：
1. **`rq_clock_pelt()` 在同步的那一跳上完全连续** —— `(T+5)−L` → `(T+10)−(L+5)` 恒等。这就是 `lost_idle_time` 的全部意义：**当没有真实空闲时间可供"还账"时，精确抵消掉那一跳**。同步负责推进时钟，`lost_idle_time` 负责在不该推进时把它扣回来，二者是一体的
2. N 个周期后：`lost_idle_time = 5N`，`clock_pelt = T₀+20N`，`rq_clock_pelt() = T₀+15N`。每 20ms 墙钟对应 15ms PELT 时间 = 10ms 忙 × 0.5 + 10ms 空闲 × 1.0。**语义完全正确且单调**

* **而当 CPU 处于低负载（非满载）时**：`util_sum < divider`
  * 说明 CPU 本来就有空闲时间，此时 `clock_task - clock_pelt` 的差值代表了“正常空闲导致的时钟偏差”，这部分差值不需要补偿，直接让 `clock_pelt` 跳变即可（因为历史负载本来就应该在空闲时衰减）
  * `clock_pelt` 硬同步后跳变增大，`rq_clock_pelt()` 随之增大，导致负载正常衰减（符合预期）

### 更新 PELT 时钟 `update_rq_clock_pelt()`
* **情况 1**：如果 CPU 空闲时，强制同步 PELT 时钟与任务时钟。因为空闲期间没有实际的计算工作，再使用缩放时钟会导致时间轴漂移，同步是为了重置基准，为下一次任务唤醒做准备
* **情况 2**：如果 CPU 非空闲
  * 传入 `update_rq_clock_pelt()` 的 `delta`，是“当前 CPU 真正用于执行任务（CFS/RT/DL）的时间增量”，已经剔除了中断处理和虚拟机偷取的时间
  * 含义：如果 CPU 运行在小核上（容量 `512`），或者处于降频状态（频率 `512`），那么 `1ms` 的硬件时间经过缩放后，`clock_pelt` 可能只前进 `0.5ms`
  * 效果：这使得 `clock_pelt` 的 **前进速度与真实消耗的“算力”成正比**，保证了 PELT 在不同硬件条件下的 **不变性（Invariance）**
  * 因此，`clock_pelt` 是 `clock_task` 的缩放版本，但它们之间没有简单的固定加减关系（`delta_pelt` 等于 `delta_task` 乘以两个因子）

```cpp
/*
 * The clock_pelt scales the time to reflect the effective amount of
 * computation done during the running delta time but then sync back to
 * clock_task when rq is idle.
 *
 *
 * absolute time   | 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|16
 * @ max capacity  ------******---------------******---------------
 * @ half capacity ------************---------************---------
 * clock pelt      | 1| 2|    3|    4| 7| 8| 9|   10|   11|14|15|16
 *
 */
static inline void update_rq_clock_pelt(struct rq *rq, s64 delta)
{
	if (unlikely(is_idle_task(rq->curr))) {
		_update_idle_rq_clock_pelt(rq);
		return;
	}

	/*
	 * When a rq runs at a lower compute capacity, it will need
	 * more time to do the same amount of work than at max
	 * capacity. In order to be invariant, we scale the delta to
	 * reflect how much work has been really done.
	 * Running longer results in stealing idle time that will
	 * disturb the load signal compared to max capacity. This
	 * stolen idle time will be automatically reflected when the
	 * rq will be idle and the clock will be synced with
	 * rq_clock_task.
	 */
  //缩放：乘以 CPU 容量因子（大核/小核）和频率因子（DVFS）
	/*
	 * Scale the elapsed time to reflect the real amount of
	 * computation
	 */
	delta = cap_scale(delta, arch_scale_cpu_capacity(cpu_of(rq)));
	delta = cap_scale(delta, arch_scale_freq_capacity(cpu_of(rq)));

	rq->clock_pelt += delta;
}
```
### 迁移前 PELT 衰减任务负载
* 这个函数的目的是：==当源 CPU 处于空闲状态时，为即将迁移的调度实体（`se`）精确估算“当前”的 PELT 时间（`now`），以便在迁移前准确老化（衰减）该任务的负载==
* 它之所以复杂，是因为它巧妙地绕过了一个陷阱：
  * **在空闲期间，`rq_clock_pelt(rq)` 虽然跟随 `clock_task` 推进，但它依然携带了历史累积的 `lost_idle_time`（算力欠账）**
  * 如果直接用这个全局值去计算单个任务的衰减，可能会因为历史欠账的干扰导致衰减速率不准确（尤其是在刚退出繁忙状态不久的空闲期）
  * 因此，它选择在本地重建一个“纯净”的当前 PELT 时间
```cpp
static inline void migrate_se_pelt_lag(struct sched_entity *se)
{
	u64 throttled = 0, now, lut;
	struct cfs_rq *cfs_rq;
	struct rq *rq;
	bool is_idle;
	//如果该任务的负载已经衰减到几乎为 0（低于阈值），就没有必要再浪费时间估算和更新了
	if (load_avg_is_decayed(&se->avg))
		return;

	cfs_rq = cfs_rq_of(se);
	rq = rq_of(cfs_rq);
	//只有在源 CPU 当前正在运行 idle 进程时，才执行估算。原因如下：
	//如果源 CPU 繁忙，rq->clock_pelt 会随着调度 tick 持续更新，se->avg.last_update_time
	//也保持较新，无需额外估算。只有空闲时，时间轴才需要特殊处理
	rcu_read_lock();
	is_idle = is_idle_task(rcu_dereference_all(rq->curr));
	rcu_read_unlock();

	/*
	 * The lag estimation comes with a cost we don't want to pay all the
	 * time. Hence, limiting to the case where the source CPU is idle and
	 * we know we are at the greatest risk to have an outdated clock.
	 */
	if (!is_idle)
		return;

	/*
	 * Estimated "now" is: last_update_time + cfs_idle_lag + rq_idle_lag, where:
	 *
	 *   last_update_time (the cfs_rq's last_update_time)
	 *	= cfs_rq_clock_pelt()@cfs_rq_idle
	 *      = rq_clock_pelt()@cfs_rq_idle
	 *        - cfs->throttled_clock_pelt_time@cfs_rq_idle
	 *
	 *   cfs_idle_lag (delta between rq's update and cfs_rq's update)
	 *      = rq_clock_pelt()@rq_idle - rq_clock_pelt()@cfs_rq_idle
	 *   空闲期间的真实物理增量 = 当前硬件时间 - 空闲起点的硬件时间基准值
	 *   rq_idle_lag (delta between now and rq's update)
	 *      = sched_clock_cpu() - rq_clock()@rq_idle
	 *
	 * We can then write:
	 *    从进入空闲到现在 PELT 时间推进的量 = 基准 + 物理增量 - 带宽冻结时间
	 *    now = rq_clock_pelt()@rq_idle - cfs->throttled_clock_pelt_time +
	 *          sched_clock_cpu() - rq_clock()@rq_idle
	 * Where:
	 *      rq_clock_pelt()@rq_idle is rq->clock_pelt_idle 空闲起点的 PELT 基准值
	 *      rq_clock()@rq_idle      is rq->clock_idle      空闲起点的硬件时间基准值
	 *      cfs->throttled_clock_pelt_time@cfs_rq_idle     CFS 带宽冻结时间
	 *                              is cfs_rq->throttled_pelt_idle
	 */

#ifdef CONFIG_CFS_BANDWIDTH
	throttled = u64_u32_load(cfs_rq->throttled_pelt_idle);
	/* The clock has been stopped for throttling */
	if (throttled == U64_MAX)
		return;
#endif
	now = u64_u32_load(rq->clock_pelt_idle);
	/*
	 * Paired with _update_idle_rq_clock_pelt(). It ensures at the worst case
	 * is observed the old clock_pelt_idle value and the new clock_idle,
	 * which lead to an underestimation. The opposite would lead to an
	 * overestimation.
	 */
	smp_rmb(); //配对读屏障
	lut = cfs_rq_last_update_time(cfs_rq); //该队列上次更新负载的时间

	now -= throttled;
	if (now < lut) //如果估算出的时间竟然比上次更新时间还早（极小概率事件）
		/*
		 * cfs_rq->avg.last_update_time is more recent than our
		 * estimation, let's use it.
		 */
		now = lut; //则使用上次更新时间作为保底，保证时间绝对单调递增
	else //正常路径：将硬件时间差补上
		now += sched_clock_cpu(cpu_of(rq)) - u64_u32_load(rq->clock_idle);

	__update_load_avg_blocked_se(now, se);
}
```

#### 估算公式的数学推导
* 我们想要求解当前时刻的 PELT 时间（记为 `now`）。已知进入空闲那一刻（`idle` 时刻）的快照：
  - `rq->clock_pelt_idle`**：进入空闲时的 PELT 时间
  - `rq->clock_idle`**：进入空闲时的原始硬件时间
  - **当前硬件时间**：`sched_clock_cpu()`
* 由于 CPU 在空闲期间没有算力消耗，PELT 时钟与硬件时钟是 **1:1 同步推进**的（因为 `update_rq_clock_pelt` 中的 `is_idle` 路径会执行 `clock_pelt = clock_task`）
* 因此，从进入空闲到现在，PELT 时间推进的量 **恰好等于** 硬件时间推进的量：

\[
\text{now} = \text{clock_pelt_idle} + (\text{sched_clock_cpu}() - \text{clock_idle})
\]

#### 考虑 CFS 带宽限制（`throttled`）

```cpp
#ifdef CONFIG_CFS_BANDWIDTH
    throttled = u64_u32_load(cfs_rq->throttled_pelt_idle);
    if (throttled == U64_MAX) return;
    now -= throttled;
#endif
```
- **背景**：如果 CFS 运行队列被带宽控制限制（`throttled`），PELT 时钟会停止推进（冻结）
- **补偿**：`cfs_rq->throttled_pelt_idle` 记录了进入空闲时被冻结的时间量。在估算 `now` 时，需要**减去**这部分冻结时间，因为我们只关心实际流逝的有效时间

#### 单调性与内存屏障保障
- 内存屏障（`smp_rmb`）：
  - **写端**（`_update_idle_rq_clock_pelt`）：先写 `clock_idle`，`smp_wmb()`，再写 `clock_pelt_idle`
  - **读端**（这里）：先读 `clock_pelt_idle`，`smp_rmb()`，再读 `clock_idle`
  - **为什么必须如此？** 如果读端读到的是**新的 `clock_pelt_idle`** 但配上了**旧的 `clock_idle`**，公式会高估 `now`（负载衰减过快）。反之则会低估。屏障保证读端要么读到旧/旧组合，要么新/新组合，绝无新旧混搭
- 保底逻辑（`now < lut`）：
  - `lut` 是 `cfs_rq->avg.last_update_time`，即该运行队列上次更新负载时的 PELT 时间
  - 如果估算出的 `now` 比 `lut` 还小，说明出现了时钟倒退（可能因为刚解除 `throttled` 或快照误差），此时直接取 `lut` 作为 `now`，保证传入的时间绝不倒退

#### 最终老化负载

```c
__update_load_avg_blocked_se(now, se);
```
- 使用这个精确估算出的 `now`，调用阻塞任务负载更新函数
- 回顾我们之前澄清的原理：`__update_load_avg_blocked_se` 传入的三个标志位全是 `0`，因此 **`___update_load_sum()` 只会执行 `decay_load()`（衰减历史），不会累加新贡献**
- 而这个 `now` 值决定了 `decay_load` 中的 `periods`（即 `delta / 1024`）大小，从而精确控制负载在迁移前应该衰减多少

#### 为什么不用 `rq_clock_pelt(rq)` 直接替代？
* 关键在于 `lost_idle_time`：`rq_clock_pelt(rq) = clock_pelt - lost_idle_time`
- 如果该 CPU 之前经历过长时间的高负载（降频运行），`lost_idle_time` 中累积了较大的算力欠账
- 在进入空闲后，虽然 `clock_pelt` 同步推进了，但 `lost_idle_time` 不变。因此 `rq_clock_pelt(rq)` 始终比真实物理时间“慢”一个固定的偏置量
- 如果直接用 `rq_clock_pelt(rq)` 去老化任务，`decay_load()` 会认为时间过得比实际慢，导致衰减不足
- **`migrate_se_pelt_lag()` 的估算公式完全绕开了 `lost_idle_time`**，只依赖进入空闲时的纯净快照（`clock_pelt_idle`）和真实的物理时间差，从而得到了一个不受历史欠账污染的“当前 PELT 时间”
* 通过这种方式，即使在长时间空闲的 CPU 上迁移任务，内核也能精确补偿流逝的时间，确保该任务在被迁移到新 CPU 时，其负载值已经正确地反映了它在空闲期间应有的指数衰减。这完美地践行了观点：**负载衰减由 `decay_load()` 驱动，而 `now` 必须提供精确的时间输入以保证衰减速率正确**

## 其他负载更新函数 `update_other_load_avgs()` 函数

* 简单来说，`update_other_load_avgs()` 就像一个“保洁员”：它不负责生产新的负载数据（除了判断当前类标志），而是周期性地下沉到各个子系统的 `rq->avg_*` 中，确保历史的负载值按照指数规律衰减，同时把当前的运行状态（是否运行、压力多大）正确地累积进去，为调度器提供准确的“实时压力仪表盘”
* 职责：非 CFS 类的 PELT 指标更新入口（RT、DL、HW、IRQ）
* 调用频率：与 `scheduler_tick()` 同频（通常每毫秒一次），或在负载均衡前批量更新
* 设计核心：**时间源分离**：RT/DL 用 PELT 缩放时钟（保证不变性），HW 用原始时钟（防止双重缩放），IRQ 传入 `0`（只衰减不累加）
* 输出：统一返回是否发生了有效更新，供上层控制流决策
  * kernel/sched/pelt.c
```cpp
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
### 返回值的设计意图
* 注意：返回值为四个子函数返回值的 **逻辑或（`|`）**，而不是 `||`，四个子函数都会被调用
* 每个子函数内部，只要 `___update_load_sum()` 跨过了至少一个 `1024us` 周期边界（即 `periods > 0`），就会返回 `1`
* 因此，只要 RT、DL、HW、IRQ 中有 **任意一个** 的 `*_sum` 发生了实质性的“跨周期衰减或累加”，上层函数（如 `update_blocked_averages()`）就知道本次调用确实做了有效工作，需要相应地更新更上层的聚合统计