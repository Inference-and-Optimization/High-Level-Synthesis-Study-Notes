## 3.11 Assertions
当用assert范围信息时，HLS支持C中的断言宏进行综合。例如，变量和循环界限的上限。

当存在可变的循环边界时，Vivado HLS无法确定循环的所有迭代的延迟，而是通过“?”标记来报告延迟。Tripcount指令可以将循环界限通知给Vivado HLS，但是此信息仅用于报告目的，不会影响综合结果（创建相同大小的硬件，无论是否使用Tripcount指令）。

下面的代码示例显示断言如何将变量的最大范围告知Vivado HLS，以及如何将这些断言用于产生更优的硬件。

在使用断言之前，必须包含定义断言宏的头文件。在此示例中，它包含在头文件中。
```c++
#ifndef _loop_sequential_assert_H_
#define _loop_sequential_assert_H_
#include <stdio.h>
#include <assert.h> 
#include ap_cint.h
#define N 32
typedef int8 din_t;
typedef int13 dout_t;
typedef uint8 dsel_t;
void loop_sequential_assert(din_t A[N], din_t B[N], dout_t X[N], dout_t Y[N], dsel_t xlimit, dsel_t ylimit);
#endif
```
在主代码中，两个assert语句放置在每个循环之前。
```c
assert(xlimit<32);
...
assert(ylimit<16);
...
```
这些断言：
- 保证如果断言为假与值大于规定的值，则C仿真将失败。这也说明了为什么在综合之前仿真C代码很重要：确认设计在综合之前有效。
- 通知Vivado HLS该变量的范围不会超过该值，并且这个事实可以优化RTL中的变量大小，在这种情况下，可以优化循环迭代。

以下代码示例显示了这些断言。
```c
#include "loop_sequential_assert.h"
void loop_sequential_assert(din_t A[N], din_t B[N], dout_t X[N], dout_t 
Y[N], dsel_t 
xlimit, dsel_t ylimit) {  
  dout_t X_accum=0;
  dout_t Y_accum=0;
  int i,j;
  assert(xlimit<32);
  SUM_X:for (i=0;i<=xlimit; i++) {
      X_accum += A[i];
      X[i] = X_accum;
  }
  assert(ylimit<16);
  SUM_Y:for (i=0;i<=ylimit; i++) {
      Y_accum += B[i];
      Y[i] = Y_accum;
  }
}
```
除了断言宏，此代码与“Loop Parallelism”中显示的代码相同。在综合之后的综合报告中，有两个重要区别。

如果没有assert宏，报告将如下所示，表明循环tripcount可以在1到256之间变化，因为循环边界的变量是d_sel数据类型的8位变量。

```
* Loop Latency: 
    +----------+-----------+----------+
    |Target II |Trip Count |Pipelined |
    +----------+-----------+----------+
    |- SUM_X   |1 ~ 256    |no        |
    |- SUM_Y   |1 ~ 256    |no        |
    +----------+-----------+----------+
```

在具有assert宏的版本中，报告显示了循环SUM_X和SUM_Y报告的Tripcount为32和16。因为断言声明值永远不会大于32和16，因此Vivado HLS可以在报告中使用该值。

```
* Loop Latency: 
    +----------+-----------+----------+
    |Target II |Trip Count |Pipelined |
    +----------+-----------+----------+
    |- SUM_X   |1 ~ 32     |no        |
    |- SUM_Y   |1 ~ 16     |no        |
    +----------+-----------+----------+
```
此外，与使用Tripcount指令不同，assert语句可以提供更优化的硬件。在没有断言的情况下，最终硬件将使用大小最大为256个循环迭代的变量和计数器。
```
* Expression: 
    +----------+------------------------+-------+---+----+
    |Operation |Variable Name           |DSP48E |FF |LUT |
    +----------+------------------------+-------+---+----+
    |+         |X_accum_1_fu_182_p2     |0      |0  |13  |
    |+         |Y_accum_1_fu_209_p2     |0      |0  |13  |
    |+         |indvar_next6_fu_158_p2  |0      |0  |9   |
    |+         |indvar_next_fu_194_p2   |0      |0  |9   |
    |+         |tmp1_fu_172_p2          |0      |0  |9   |
    |+         |tmp_fu_147_p2           |0      |0  |9   |
    |icmp      |exitcond1_fu_189_p2     |0      |0  |9   |
    |icmp      |exitcond_fu_153_p2      |0      |0  |9   |
    +----------+------------------------+-------+---+----+
    |Total     |                        |0      |0  |80  |
    +----------+------------------------+-------+---+----+
```
断言变量范围小于最大可能范围的代码将导致更小的RTL设计。
```
* Expression: 
    +----------+------------------------+-------+---+----+
    |Operation |Variable Name           |DSP48E |FF |LUT |
    +----------+------------------------+-------+---+----+
    |+         |X_accum_1_fu_176_p2     |0      |0  |13  |
    |+         |Y_accum_1_fu_207_p2     |0      |0  |13  |
    |+         |i_2_fu_158_p2           |0      |0  |6   |
    |+         |i_3_fu_192_p2           |0      |0  |5   |
    |icmp      |tmp_2_fu_153_p2         |0      |0  |7   |
    |icmp      |tmp_9_fu_187_p2         |0      |0  |6   |
    +----------+------------------------+-------+---+----+
    |Total     |                        |0      |0  |50  |
    +----------+------------------------+-------+---+----+
```
断言可以指示设计中任何变量的范围。在使用断言时执行涵盖所有可能情况的C仿真非常重要。这将确认Vivado HLS使用的断言是有效的。