# 时间管理

# 概念

* 内核需要在硬件的帮助下才能计算和管理时间
* **系统定时器** 是一种可编程硬件芯片，以固定频率产生中断。该中断即所谓的 **定时器中断**。
* **定时器中断处理程序** 负责 **更新系统时间**，也负责 **执行需要周期性运行的任务**。
* **动态定时器** 一种用来推迟执行的工具。
* 定时器以某种频率自行触发（击中（hitting）或射中（popping）），该频率可通过编程预定，称作 **节拍率（tick rate）**。
* 因为预编的节拍率对内核来说是可知的，所以内核知道连续两次时钟中断的间隔时间，该间隔时间成为 **节拍（tick）**。
* 节拍 = 1/节拍率 秒，tick = 1/(tick rate) 秒，可见，*节拍是一个时间概念*。
* 系统定时器频率（节拍率 tick rate）是通过静态预处理定义的，即 **HZ（赫兹）**。
  * 在系统启动时，按照HZ值对硬件进行设置。
  * 体系架构不同，HZ的值也不同。
  * 如x86体系结构中，系统定时器频率默认值为100，时钟中断频率就为100HZ，即该处理器上每秒时钟中断100次（即10ms一次）。
  * 注意，HZ的值是可调的。不要在代码中hard-coded HZ的值。

# HZ

## 理想的HZ值
* 更高的时钟中断解析度（resolution）可提高时间驱动事件的解析度
* 提高了时间驱动事件的准确度（accuracy）

## 高HZ的优劣
* 依赖定时执行的系统调用，如 poll() 和 select()，能够以更高的精度运行
* 对诸如资源消耗和系统运行时间的测量会有更精细的解析度
* 提高进程抢占的准确度

## 高HZ的劣势
* 节拍率高 -> 时钟中断频率高 -> 系统负担越重
* 中断处理程序占用时间越多，处理器处理其他工作的时间就越少
* 频繁打乱处理器的高速缓存并增加耗电

## 动态时钟
* 启用 **动态时钟** 的系统也称为 **无时钟系统（tickless system）**
* 内核配置选项为`CONFIG_NO_HZ`
* 动态时钟只有在有些任务需要实际执行时才激活时钟周期。否则会临时禁用时钟周期。
* 内核如何判定系统无事可做？
  * 运行队列没有活动进程，内核将选取一个特别的idle进程（调度器类为idle_task）来运行。
  * 每当选中idle进程运行时，都将禁用周期时钟，直至如下情况再重新启用时钟周期：
    * 下一个定时器即将到器为止
    * 或者有中断发生
  * 其间CPU进入不受打扰的睡眠状态。

### 注意以下几点
* 只有经典定时器需要考虑此用法。高分辨率定时器不绑定到时钟频率，也并非基于周期时钟实现。
* 单触发时钟是实现动态时钟的先决条件。纯粹的周期性的定时器根本就不是适用于该机制。

# jiffies

* 全局变量`jiffies`记录自系统启动以来产生的节拍的总数。
  * 启动时初始化为0。
  * 每次时钟中断处理程序会增加该变量的值（因此可以认为其单位为（次）），意味着时间过去了一个节拍（tick）。
  * 每秒内`jiffies`的值会增加`HZ`。
  * 系统的运行时间为`jiffies/HZ`秒（`HZ`的单位为（次/秒））。

* include/linux/jiffies.h
```c
/* some arch's have a small-data section that can be accessed register-relative
 * but that can only take up to, say, 4-byte variables. jiffies being part of
 * an 8-byte variable may not be correctly accessed unless we force the issue
 */
#define __jiffy_data  __attribute__((section(".data")))

/*
 * The 64-bit value is not atomic - you MUST NOT read it
 * without sampling the sequence number in jiffies_lock.
 * get_jiffies_64() will do this for you as appropriate.
 */
extern u64 __jiffy_data jiffies_64;
extern unsigned long volatile __jiffy_data jiffies;

#if (BITS_PER_LONG < 64)
u64 get_jiffies_64(void);
#else
static inline u64 get_jiffies_64(void)
{
    return (u64)jiffies;
}
#endif
...
```

* kernel/time/jiffies.c
```c
__cacheline_aligned_in_smp DEFINE_SEQLOCK(jiffies_lock);

#if (BITS_PER_LONG < 64)
u64 get_jiffies_64(void)
{
    unsigned long seq;
    u64 ret;

    do {
        seq = read_seqbegin(&jiffies_lock);
        ret = jiffies_64;
    } while (read_seqretry(&jiffies_lock, seq));
    return ret;
}   
EXPORT_SYMBOL(get_jiffies_64);
#endif

EXPORT_SYMBOL(jiffies);
```
* 32位体系结构不能原子地一次访问64位变量中的两个32位数值。因此在读取`jiffies`时需用seq锁对变量`jiffies`进行锁定。

* 这里用`jiffies_64`变量的初值覆盖`jiffies`变量
  * arch/x86/kernel/vmlinux.lds.S
  ```c
  #ifdef CONFIG_X86_32
  OUTPUT_ARCH(i386)
  ENTRY(phys_startup_32)
  jiffies = jiffies_64;
  #else
  OUTPUT_ARCH(i386:x86-64)
  ENTRY(phys_startup_64)
  jiffies_64 = jiffies;
  #endif
  ```
* `jiffies`取整个64为`jiffies_64`变量的低32位，时间管理代码使用整个64位，来避免整个64位的溢出。
* 访问`jiffies`的代码仅会读取`jiffies_64`的低32位。通过`get_jiffies_64()`读取整个64位数值。

## jiffies的回绕
* `jiffies`有可能会发生回绕（wrap around），最好用提供的宏来进行判断。
* include/linux/typecheck.h
```c
...
/*
 *  These inlines deal with timer wrapping correctly. You are
 *  strongly encouraged to use them
 *  1. Because people otherwise forget
 *  2. Because if the timer wrap changes in future you won't have to
 *     alter your driver code.
 *                                                                                                                                                 
 * time_after(a,b) returns true if the time a is after time b.
 *
 * Do this with "<0" and ">=0" to only test the sign of the result. A
 * good compiler would generate better code (and a really good compiler
 * wouldn't care). Gcc is currently neither.
 */
#define time_after(a,b)     \
    (typecheck(unsigned long, a) && \
     typecheck(unsigned long, b) && \
     ((long)((b) - (a)) < 0))
#define time_before(a,b)    time_after(b,a)

#define time_after_eq(a,b)  \
    (typecheck(unsigned long, a) && \
     typecheck(unsigned long, b) && \
     ((long)((a) - (b)) >= 0))
#define time_before_eq(a,b) time_after_eq(b,a)
...
```
* 注意，`time_after`和`time_before`并不能完全解决回绕（wrap around）问题：
  * 例如，想用`time_after(a,b)`来判断`a`时间是否在`b`之后，假设，`b`的值是 2<sup>31</sup>-1 = 2147483647，HZ=1000，即一个tick是1ms，则经过 2147483649 个tick之后，`jiffies`回绕，此时`(long)(b-a)>0`，判断结果是`a`在`b`之前，然而事实上`a`确实已经是在`b`之后。
  * 再假如，想用`time_before(a,b)`来判断`a`时间是否在`b`之前，假设，`b`的值是 0，HZ=1000，即一个tick是1ms，则经过 2<sup>31</sup>-1 = 2147483648 个tick之后，`jiffies`回绕，此时`(long)(a-b)<0`，判断结果是`a`在`b`之前，然而事实上`a`确实已经是在`b`之后很久了。
  * 可见，该宏的使用前提是`a`和`b`的逻辑差值在2<sup>31</sup>（如果HZ是1000则大概是24.86天）之内，否则有可能会因为回绕而不再准确。

## 用户空间和HZ
* 如果在内核中修改了HZ的定义值，用户空间并不知道新HZ值，这就打破了用户空间的常量关系，从而导致问题。
* 

# 硬时钟和定时器
实时时钟
  ● RTC，用来持久存放系统时间的设备，系统关闭后也可靠主板上的微型电池供电，保持系统的计时
  ● PC架构中，RTC与CMOS集成在一起
  ● 系统启动时，内核通过读取RTC来初始化墙上时间，该时间存放在xtime变量中。这是它最主要的作用。

系统定时器
  ● 提供一种周期性触发中断机制
  ● 对电子晶振进行分频来实现；或者提供一个衰减测量器（decrementer）来实现（设置测量器初始值，该值以固定频率衰减，衰减到0时触发一个中断
  ● x86主要采用可编程中断时钟（PIT）
  ● x86其他时钟资源还包括本地的APIC和时间戳（TSC）等

时钟中断处理程序的功能
  ● 重新启动时钟 *
  ● 按内部定时器有计划地调用内部的内核
  ● 为核心进程和用户进程提供执行profiling的能力 *
  ● 收集系统和进程的accounting statistics *
  ● 计时 *
  ● 在有请求时，向进程发送alarm软中断信号
  ● 定期唤醒swapper进程
  ● 控制进程的调度
带 * 的每次时钟tick都会做，其他的有条件地执行或者若干个始终tick执行一次。
时钟中断处理程序运行时proccessor execution level会设的比较高，以防止中断处理程序工作期间发生其他时间（如来自外设的中断）。
因此，速度要很快。

重启时钟
由软件指令重启时钟，从而在适当的时间间隔后再次中断处理器。由机器相关的指令完成。

内部的系统超时
callout表 —— 定时器到时要执行函数与定时时间（以时钟tick为单位）的映射表
  ● 时钟中断处理程序并不会直接调用这些函数（为什么？会导致忽略后面的时钟中断）
  ● 通过触发一个软中断来调用
  ● 软中断要比其他中断优先级低，在内核处理其他中断之前处于屏蔽状态
  ● 软中断最终发生时，清楚callout中过时表项，并调用对应的函数

Profiling
时钟中断处理程序调用profile驱动程序的中断处理函数，根据pc（程序计数器）所在函数的范围，使对应的内部计数器加1。
没有计入花在执行时钟中断处理程序和屏蔽时钟级中断的代码上的时间。

计时
boot后每个时钟中断时给一个时间变量加1。系统计时以时钟tick为单位。
