### BITS_TO_LONGS()
* 除法，余数向上取整
  ```c
  ...
  #define __KERNEL_DIV_ROUND_UP(n, d) (((n) + (d) - 1) / (d))
  #define DIV_ROUND_UP __KERNEL_DIV_ROUND_UP
  ...___```
  ```
* include/linux/bitops.h
  ```c
  #define BITS_PER_TYPE(type) (sizeof(type) * BITS_PER_BYTE)
  #define BITS_TO_LONGS(nr)       DIV_ROUND_UP(nr, BITS_PER_TYPE(long))
  ```
* `BITS_PER_BYTE` 按规定每字节为 8 位
* `BITS_PER_TYPE(long)` 在 64 位机器上为`64`
* `BITS_TO_LONGS(nr)` 为`nr`除以`64`向上取整的结果

### cpu_bit_bitmap[]
* kernel/cpu.c
  ```c
  /* cpu_bit_bitmap[0] is empty - so we can back into it */
  #define MASK_DECLARE_1(x)       [x+1][0] = (1UL << (x))
  #define MASK_DECLARE_2(x)       MASK_DECLARE_1(x), MASK_DECLARE_1(x+1)
  #define MASK_DECLARE_4(x)       MASK_DECLARE_2(x), MASK_DECLARE_2(x+2)
  #define MASK_DECLARE_8(x)       MASK_DECLARE_4(x), MASK_DECLARE_4(x+4)

  const unsigned long cpu_bit_bitmap[BITS_PER_LONG+1][BITS_TO_LONGS(NR_CPUS)] = {

          MASK_DECLARE_8(0),      MASK_DECLARE_8(8),
          MASK_DECLARE_8(16),     MASK_DECLARE_8(24),
  #if BITS_PER_LONG > 32
          MASK_DECLARE_8(32),     MASK_DECLARE_8(40),
          MASK_DECLARE_8(48),     MASK_DECLARE_8(56),
  #endif
  };
  EXPORT_SYMBOL_GPL(cpu_bit_bitmap);
  ```
* `cpu_bit_bitmap`为二维数组，元素类型为`unsigned long`
  - 一维有`65`项，一维第一项为空
  - 二维由`NR_CPUS`决定，例如 256 核 CPU 二维数组会有 4 个元素，总共有`65 x 4 = 260`个`unsigned long`元素

#### 展开`MASK_DECLARE_8(0)`
* `MASK_DECLARE_8(0)`
* `MASK_DECLARE_4(0), MASK_DECLARE_4(0+4)`
* `MASK_DECLARE_2(0), MASK_DECLARE_2(0+2), MASK_DECLARE_2(0+4), MASK_DECLARE_2(0+4+2)`
* `MASK_DECLARE_1(0), MASK_DECLARE_1(0+1),`
  `MASK_DECLARE_1(0+2), MASK_DECLARE_1(0+2+1),`
  `MASK_DECLARE_1(0+4), MASK_DECLARE_1(0+4+1),`
  `MASK_DECLARE_1(0+4+2), MASK_DECLARE_1(0+4+2+1)`
* `[1][0] = (1UL << (0)), [2][0] = (1UL << (1)),`
  `[3][0] = (1UL << (2)), [4][0] = (1UL << (3)),`
  `[5][0] = (1UL << (4)), [6][0] = (1UL << (5)),`
  `[7][0] = (1UL << (6)), [8][0] = (1UL << (7))`
* `MASK_DECLARE_8(0)`就是给数组的一维的 1~8 项，二维的第 0 项，赋初值为 `1UL << n`

### cpumask_of()
* include/linux/types.h
  ```c
  #define DECLARE_BITMAP(name,bits) \
        unsigned long name[BITS_TO_LONGS(bits)]
  ```
* include/linux/cpumask.h
  ```c
  /**
   * to_cpumask - convert an NR_CPUS bitmap to a struct cpumask *
   * @bitmap: the bitmap
   *
   * There are a few places where cpumask_var_t isn't appropriate and
   * static cpumasks must be used (eg. very early boot), yet we don't
   * expose the definition of 'struct cpumask'.
   *
   * This does the conversion, and can be used as a constant initializer.
   */
  #define to_cpumask(bitmap)                                              \
          ((struct cpumask *)(1 ? (bitmap)                                \
                              : (void *)sizeof(__check_is_bitmap(bitmap))))

  static inline int __check_is_bitmap(const unsigned long *bitmap)
  {
          return 1;
  }

  /* Don't assign or return these: may not be this big! */
  typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
  ...
  /*
   * Special-case data structure for "single bit set only" constant CPU masks.
   *
   * We pre-generate all the 64 (or 32) possible bit positions, with enough
   * padding to the left and the right, and return the constant pointer
   * appropriately offset.
   */
  extern const unsigned long
          cpu_bit_bitmap[BITS_PER_LONG+1][BITS_TO_LONGS(NR_CPUS)];

  static inline const struct cpumask *get_cpu_mask(unsigned int cpu)
  {
          const unsigned long *p = cpu_bit_bitmap[1 + cpu % BITS_PER_LONG];
          p -= cpu / BITS_PER_LONG;
          return to_cpumask(p);
  }
  ...
  /**
   * cpumask_of - the cpumask containing just a given cpu
   * @cpu: the cpu (<= nr_cpu_ids)
   */
  #define cpumask_of(cpu) (get_cpu_mask(cpu))
  ...*```
  ```
* `BITS_PER_LONG` 在 64 位机器上为 64
* 用如下函数可以 dump `cpu_bit_bitmap`数组
  ```c
  void dump_cpu_bit_bitmap(void)
  {
      int i = 0 , j = 0;
      const unsigned long *p = NULL;
      size_t row = sizeof(cpu_bit_bitmap)/sizeof(cpu_bit_bitmap[0]);
      size_t column = sizeof(cpu_bit_bitmap[0])/sizeof(unsigned long);

      printf("cpu_bit_bitmap[%zd][%zd]\n", row, column);

      for (i = 0; i < row; i++)
      {   
          printf("[%02d] ", i);
          for (j = 0; j < column; j++)
          {   
              p = &cpu_bit_bitmap[i][j];
              printf("%p(%016lx) ", p, *p);
          }   
          printf("\n");
      }   
  }
  ```
* 假设有 256 个逻辑核，得到的数组如下：

  bitmap | 0 | 1 | 2 | 3
  ---|---|---|---|---
  0  | x |192|128|64
  1  | 0 |193|129|65
  2  | 1 |194|130|66
  ...|...|...|...|...
  63 |62 |255|   |127
  64 |63 | x | x | x
  * 第一列为一维数组索引，第一行为二维数组索引
  * 中间的数为 cpu 取不同的值在数组中的位置

### _find_next_bit()
* include/linux/bitmap.h
  ```c
  #define BITMAP_FIRST_WORD_MASK(start) (~0UL << ((start) & (BITS_PER_LONG - 1)))
  ```
* lib/find_bit.c
  ```c
  /*
   * This is a common helper function for find_next_bit, find_next_zero_bit, and
   * find_next_and_bit. The differences are:
   *  - The "invert" argument, which is XORed with each fetched word before
   *    searching it for one bits.
   *  - The optional "addr2", which is anded with "addr1" if present.
   */
  static inline unsigned long _find_next_bit(const unsigned long *addr1,
                  const unsigned long *addr2, unsigned long nbits,
                  unsigned long start, unsigned long invert)
  {
          unsigned long tmp;
          /*健全性检查，超过 cpu 数的起始位置不合法，返回有效 cpu 数表示没有交集*/
          if (unlikely(start >= nbits))
                  return nbits;
          /*addr1 和 addr2 分别指向由多个 unsigned long 元素组成的数组，每个数组元素的一
            个 bit 代表一个 cpu。
            那么 addr[start/BITS_PER_LONG] 就是操作起始的第一个数组元素。*/
          tmp = addr1[start / BITS_PER_LONG];
          /*得到 addr1 和 addr2 第一个数组元素的交集*/
          if (addr2)
                  tmp &= addr2[start / BITS_PER_LONG];
          /*这里利用二进制与 0 异或，位不变；与 1 异或，位反转。
            find_next_zero_bit() invert 就是全 1*/
          tmp ^= invert;

          /* Handle 1st word. */
          /*然而，start 可能并不是第一个数组元素的第一个位，所以 BITMAP_FIRST_WORD_MASK()
            把 start 之前的位清零，之后的位构成掩码。
            至此，第一个元素中的位就处理完了。*/
          tmp &= BITMAP_FIRST_WORD_MASK(start);
          start = round_down(start, BITS_PER_LONG); /*start 做下舍入 BITS_PER_LONG*/
          /*如果第一个元素有交集，则无需之后的寻找。
            之后的循环处理也是类似的，有交集则退出循环。*/
          while (!tmp) {
                  start += BITS_PER_LONG; /*start 移至下一数组元素*/
                  if (start >= nbits)     /*再次检测越界的可能*/
                          return nbits;
                  /*基本上和之前的处理是一样的*/
                  tmp = addr1[start / BITS_PER_LONG];
                  if (addr2)
                          tmp &= addr2[start / BITS_PER_LONG];
                  tmp ^= invert;
          }
          /*__ffs()通常是 arch 相关的实现，比如说有指令来寻找第一个置 1 的位。这就找到了。*/
          return min(start + __ffs(tmp), nbits);
  }
  ```
