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
* 拆成 \( (0.5)^k \times y^r \) 不是为了数学好看，而是为了**极致地避免浮点运算和循环**
* 在 PELT 的实现中，计算 \( y^n \) 的衰减系数需要在一个时钟中断内完成，不能做耗时的大循环，于是内核采用了**查表 + 移位**的策略：
1. **对于 \( (0.5)^k \)**：乘以 `0.5` 等于**右移一位**（`>> 1`）。如果有 \( k \) 个 `0.5` 相乘，就是右移 \( k \) 位（`>> k`）
   * 移位操作在 CPU 中是一条极快的指令
2. **对于 \( y^r \)**：因为 \( r \) 只能是 0 到 31，内核预先在内存中计算好了一个包含 32 个元素的 **查找表**，直接查出 \( y^r \) 的值（这是一个大整数，用于定点数运算）
* 这样一来，计算 \( y^n \) 就变成了 **“一次查表 + 一次移位”**，时间复杂度从 `O(n)` 降到了 `O(1)`
* **数学上**：正确的分解是 \( y^n = (0.5)^{\lfloor n/32 \rfloor} \times y^{n \bmod 32} \)
* **代码里**：这里的“n 除以 32”被计算机解释为**整数除法（取下整）**，而“乘以 0.5 的 k 次方”被转换为 **右移 k 位**

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

## 负载贡献的累加总和
* `__accumulate_pelt_segments()` 函数是 PELT 算法的 **数学核心**，它负责计算过去一段时间内，负载贡献的 **累加总和**

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
  * 这是函数中最精妙、最难理解的一行，它计算的是**中间完整周期 `d2` 的贡献总和**

##### 理解 `LOAD_AVG_MAX`
* `LOAD_AVG_MAX` 是**等比级数的极限和**，定义为：
  
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
* 内核用了非常巧妙的**差集**方法来计算：

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
