## 3.5 Loops
循环提供了一种非常直观和简洁的方式来捕获算法的行为，并经常在C代码中使用。综合可以很好地支持循环：循环可以流水线化，展开，部分展开，合并和展平。

优化可以有效地展开，部分展开，展平和合并，从而对循环结构进行更改，就像更改了代码一样。这些优化确保优化循环时仅需要有限的编码更改。某些优化只能在某些条件下应用。可能需要更改一些代码。

:white_check_mark: 推荐：避免将全局变量用于循环索引变量，因为这可能会抑制某些优化。

### Variable Loop Bounds
当循环具有变量边界时，将阻止Vivado® HLS可以应用的某些优化。在下面的代码示例中，循环边界由变量`width`确定，变量`width`由顶层输入驱动。在这种情况下，**循环被认为具有变量边界，因为Vivado HLS无法知道循环何时完成**。

```c
#include "ap_cint.h"
#define N 32
typedef int8 din_t;
typedef int13 dout_t;
typedef uint5 dsel_t;
dout_t code028(din_t A[N], dsel_t width) {  
 dout_t out_accum=0;
 dsel_t x;
 LOOP_X:for (x=0;x<width; x++) {
  out_accum += A[x];
 }
 return out_accum;
}
```
上面示例中试图优化设计的过程揭示了可变循环边界所产生的问题。可变循环边界的第一个问题是，它们会阻止Vivado HLS确定循环的延迟。Vivado HLS可以确定完成一个循环的延迟，但是由于无法静态确定变量`width`的确切值，因此它不知道执行了多少次迭代，因此无法报告循环延迟（执行完每个循环迭代的完整时钟周期）。

当存在可变循环界限时，Vivado HLS会将延迟报告为问号（？），而不是使用精确值。下面显示了以上示例的综合结果。
```
+ Summary of overall latency (clock cycles): 
 * Best-case latency:    ?
 * Worst-case latency:   ?
+ Summary of loop latency (clock cycles): 
 + LOOP_X: 
 * Trip count: ?
 * Latency:    ?
```
可变循环界限的另一个问题是设计的性能未知。解决此问题的两种方法如下：
- 使用Tripcount指令。在此说明有关此方法的详细信息。
- 在C代码中使用assert宏。

tripcount指令允许为循环指定最小和/或最大tripcount。tripcount是循环迭代的次数。如果在第一个示例中将最大tripcount 32应用于LOOP_X，则报告将更新为以下内容： 
```
+ Summary of overall latency (clock cycles): 
 * Best-case latency:    2
 * Worst-case latency:   34
+ Summary of loop latency (clock cycles): 
 + LOOP_X: 
 * Trip count: 0 ~ 32
 * Latency:    0 ~ 32 
```
Tripcount指令对综合结果没有影响，仅对报告有影响。用户为Tripcount指令提供的值**仅用于报告**。Tripcount值使Vivado HLS可以报告报告中的数量，从而可以比较来自不同解决方案的报告。**为了使综合拥有相同的循环绑定信息，必须更新C代码**。

Tripcount指令对综合结果没有影响，仅对报告有影响。

在较短的启动间隔内优化第一个示例的下一步是：
- 展开循环并允许累加并行发生。
- 通过单个内存端口对数组输入进行分区，或限制并行累加。

如果应用了这些优化，则Vivado HLS的输出将突出显示变量绑定循环中最明显的问题： 

```
@W [XFORM-503] Cannot unroll loop 'LOOP_X' in function 'code028': cannot 
completely 
unroll a loop with a variable trip count. //这里显示可变tripcount
```
因为可变边界循环无法展开，所以它们不仅阻止unroll指令的应用，而且还阻止了循环以上级别的流水化。

:star: 重要！**当对循环或函数进行流水线处理**时，Vivado HLS会在函数或循环下方的层次结构中**展开所有循环**。**如果在此层次结构中存在一个带有变量边界的循环，则它将阻止流水线操作**。

解决带有可变边界的循环的方案是使循环迭代的次数固定在一个固定值上，并在循环内部条件执行。可以重写可变循环边界示例中的代码，如以下代码示例中所示。此处，将循环界限明确设置为可变宽度的最大值，并条件执行循环主体：

```c
#include "ap_cint.h"
#define N 32
typedef int8 din_t;
typedef int13 dout_t;
typedef uint5 dsel_t;
dout_t loop_max_bounds(din_t A[N], dsel_t width) {  
 dout_t out_accum=0;
 dsel_t x;
 LOOP_X:for (x=0;x<N; x++) {
  if (x<width) {
    out_accum += A[x];
  }
 }
 return out_accum;
}
```
上面示例中的for循环（LOOP_X）可以展开。由于循环具有固定的上限，因此Vivado HLS知道要创建多少硬件。RTL设计中有N（32）个循环体副本。循环主体的每个副本都具有与之关联的条件逻辑，并根据变量`width`的值执行。

### Loop Pipelining
在对循环进行流水线处理时，通常通过对最内部的循环进行流水线处理来找到面积与性能之间的最佳平衡。这也导致最快的运行时间。下面的代码示例演示了循环和函数流水化时的权衡取舍。
```c
#include "loop_pipeline.h"
dout_t loop_pipeline(din_t A[N]) {  
 int i,j;
 static dout_t acc;
 LOOP_I:for(i=0; i < 20; i++){
  LOOP_J: for(j=0; j < 20; j++){
    acc += A[i] * j;
  }
 }
 return acc;
}
```
如果**最里面的（LOOP_J）被流水线处理**，则硬件中有一个LOOP_J副本（一个乘法器）。在这种情况下，Vivado® HLS会尽可能自动展平循环，在这种情况下，有效地创建了一个**20*20迭代的新单循环**。只需要调度**一个乘法操作和一个数组访问**，然后就可以将循环迭代调度为单个循环实体（20x20循环迭代）（这里的意思应该是把两层循环变成了单层循环即400个）。

:bulb: 提示：当对循环或函数进行流水线处理时，必须展开循环或函数下方的层次结构中的任何循环。

如果对**外部循环（LOOP_I）进行了流水线处理**，则将展开内部循环（LOOP_J），以创建循环体的20个副本：现在必须安排**20个乘法器和20个数组访问**。然后，可以将LOOP_I的每次迭代安排为单个实体。

如果**顶层函数已流水化**，则必须展开两个循环：现在必须**调度400个乘法器和400个数组访问**。Vivado HLS不太可能产生具有400个乘法的设计，因为在大多数设计中，**数据相关性通常会阻止最大并行度**，例如，即使将双端口RAM用于A [N]，该设计也可以在任何时钟周期内，仅访问A[N]的两个值。

在选择流水线层次结构的哪个级别时要理解的概念是，对最内部的循环流水线化**可以用最小的硬件提供大多数应用程序通常可接受的吞吐量**。对层次结构的高层进行流水线化将**展开所有子循环，并且可以创建更多要调度的操作**（这可能会影响运行时间和内存容量），但是通常在吞吐量和延迟方面提供最高性能的设计。

总结以上选项：
- Pipeline LOOP_J：延迟大约为400个周期（20x20），需要少于100个LUT和寄存器（始终存在I/O控制和FSM）。
- Pipeline LOOP_I：延迟约为20个周期，但需要数百个LUT和寄存器。使用的逻辑大约是第一次优化，减去可以进行的任何逻辑优化的20倍。
- Pipeline function loop_pipeline：延迟大约为10（20个双端口访问），但需要数千个LUT和寄存器（逻辑大约是第一个优化减去可以进行的任何优化的400倍）。

#### Imperfect Nested Loops
当对循环层次结构的内部循环进行流水线处理时，Vivado® HLS对嵌套循环进行展平，以通过消除由循环传递（在进入和退出循环时对循环索引执行的检查）来减少延迟并提高整体吞吐量。从一个循环过渡到下一个循环（进入和/或退出）时，此类检查可能会导致时钟延迟。

不完善的循环嵌套，或者无法展平的循环嵌套，**会导致额外的时钟周期进入和退出循环**。当设计包含嵌套循环时，请分析结果以确保展平了尽可能多的嵌套循环：针对该case查看日志文件或在综合报告中查看，如上所示，其中循环标签已合并（现在将LOOP_I和LOOP_J报告为LOOP_I_LOOP_J）。

### Loop Parallelism
Vivado® HLS要尽早调度逻辑和函数以减少延迟。为此，它并行调度尽可能多的逻辑操作和函数。它不会调度循环以并行执行。

如果综合了以下代码示例，则对循环SUM_X进行调度，然后对循环SUM_Y进行调度：即使循环SUM_Y无需等待循环SUM_X完成就可以开始操作，也要在SUM_X之后进行调度。
```c
#include "loop_sequential.h"
void loop_sequential(din_t A[N], din_t B[N], dout_t X[N], dout_t Y[N], 
 dsel_t xlimit, dsel_t ylimit) {  
 dout_t X_accum=0;
 dout_t Y_accum=0;
 int i,j;
 SUM_X:for (i=0;i<xlimit; i++) {
  X_accum += A[i];
  X[i] = X_accum;
 }
 SUM_Y:for (i=0;i<ylimit; i++) {
  Y_accum += B[i];
  Y[i] = Y_accum;
 }
} 
```
**由于循环具有不同的边界（xlimit和ylimit），因此无法合并**。如下面的代码示例所示，通过将循环放置在单独的函数中，可以实现相同的功能，并且可以并行调度两个循环（在函数内部）。
```c
#include "loop_functions.h"
void sub_func(din_t I[N], dout_t O[N], dsel_t limit) {
 int i;
 dout_t accum=0;
  
 SUM:for (i=0;i<limit; i++) {
  accum += I[i];
  O[i] = accum;
 }
}
void loop_functions(din_t A[N], din_t B[N], dout_t X[N], dout_t Y[N], 
 dsel_t xlimit, dsel_t ylimit) {
 sub_func(A,X,xlimit);
 sub_func(B,Y,ylimit);
}
```

如果综合了前面的示例，则延迟是顺序循环示例的延迟的一半，因为这些循环（作为函数）现在可以并行执行。

dataflow优化也可以在顺序循环示例中使用。对于无法使用数据流优化的情况，此处介绍了把**循环置于函数中以利用并行性的原理**。例如，在一个更大的示例中，数据流优化应用于顶层的所有循环和函数，以及每个顶层循环和函数之间的内存。
### Loop Dependencies
循环依赖关系是阻止循环（通常为流水线）优化的数据依赖关系。它们可以在循环的单个迭代内，也可以在循环的不同迭代之间。

理解循环依赖关系的最简单方法是研究一个极端的例子。在以下示例中，循环的结果用作循环继续或退出条件。必须先完成循环的每个迭代，然后才能开始下一个迭代。
```c
Minim_Loop: while (a != b) { 
 if (a > b) 
  a -= b; 
 else 
  b -= a;
} 
```
该循环无法进行流水线处理。循环的下一个迭代要等到上一个迭代结束后才能开始。并非所有循环依赖性都如此极端，但是此示例强调了某些操作要等到其他一些操作完成才能开始。解决方案是尝试确保尽早执行初始操作。

循环依赖性可能与任何类型的数据一起发生。它们在使用数组时特别常见。

### Unrolling Loops in C++ Classes
在C++类中使用循环时，应注意确保循环归纳变量不是该类的数据成员，因为这样会阻止循环展开。

在此示例中，循环归纳变量k是foo_class类的成员。
```c++
template <typename T0, typename T1, typename T2, typename T3, int N>
class foo_class {
private:
 pe_mac<T0, T1, T2> mac;
public:
 T0 areg;
 T0 breg;
 T2 mreg;
 T1 preg;
 T0 shift[N];
 int k;             // Class Member
 T0 shift_output;
 
 void exec(T1 *pcout, T0 *dataOut, T1 pcin, T3 coeff, T0 data, int col)
 {
  Function_label0:;
#pragma HLS inline off
  SRL:for (k = N-1; k >= 0; --k) {
#pragma HLS unroll // Loop will fail UNROLL
  if (k > 0) 
    shift[k] = shift[k-1];
  else 
    shift[k] = data;
  }
  *dataOut = shift_output;
  shift_output = shift[N-1];
 }
 *pcout = mac.exec1(shift[4*col], coeff, pcin);
};
```
为使Vivado® HLS能够按照UNROLL编译指示指定的方式展开循环，应重新编写代码以删除k作为类成员。
```c++
template <typename T0, typename T1, typename T2, typename T3, int N>
class foo_class {
private:
 pe_mac<T0, T1, T2> mac;
public:
 T0 areg;
 T0 breg;
 T2 mreg;
 T1 preg;
 T0 shift[N];
 T0 shift_output;
 void exec(T1 *pcout, T0 *dataOut, T1 pcin, T3 coeff, T0 data, int col)
 {
  Function_label0:;
  int k;             // Local variable
#pragma HLS inline off
  SRL:for (k = N-1; k >= 0; --k) {
#pragma HLS unroll // Loop will unroll
  if (k > 0) 
    shift[k] = shift[k-1];
  else 
    shift[k] = data;
  }
  *dataOut = shift_output;
  shift_output = shift[N-1];
 }
 *pcout = mac.exec1(shift[4*col], coeff, pcin);
};
```