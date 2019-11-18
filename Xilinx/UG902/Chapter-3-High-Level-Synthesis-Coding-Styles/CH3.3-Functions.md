## 3.3 Functions
顶层函数在综合之后成为RTL设计的top-level。在RTL设计中，子函数被综合为块。

:star: 重要！顶层函数不能是静态函数。

在综合之后，设计中的每个函数都有其自己的综合报告和RTL HDL文件（Verilog和VHDL）。

### Inlining Functions
子函数可以内联，以将其逻辑与周围函数的逻辑合并。**内联函数可以带来更好的优化效果，同时也可以提高运行效率**。
更多的逻辑和更多的可能性必须保留在内存中并进行分析。

:bulb: 提示：Vivado® HLS可能会自动内嵌小型函数。要禁用小函数的自动内联，请将该函数的inline指令设置为off。

如果函数内联，则该函数没有报告或单独的RTL文件。**逻辑和循环在层次结构中与其上方的函数合并**。

### Impact of Coding Style
**编码样式对函数的主要影响的是函数的参数和接口**。

如果函数的参数大小被精确设定，则Vivado® HLS可以在设计中传播此信息。无需为每个变量创建任意精度类型。在下面的示例中，两个整数相乘，但结果仅使用底部的24位。

```c++
#include "ap_cint.h"
int24 foo(int x, int y) {  
 int tmp;
 tmp = (x * y);
 return tmp
} 
```
综合该代码后，结果是一个32位乘法器，输出被截断为24位。

如下面的代码示例所示，如果输入的大小正确设置为12-bit类型（int12），则最终RTL使用24位乘法器。
```c
#include "ap_cint.h"
typedef int12 din_t;
typedef int24 dout_t;
dout_t func_sized(din_t x, din_t y) {  
 int tmp;
 tmp = (x * y);
 return tmp
}
```

在输入的两个函数上使**用任意精度类型**足以确保Vivado HLS使用24位乘法器创建设计。**12位类型通过设计传播**。Xilinx建议您正确调整层次结构中所有函数的参数大小。

通常，当变量直接从接口上的函数驱动，尤其是从接口上的顶层函数驱动时，它们可以**阻止发生某些优化**。典型的情况是将输入用作循环索引的上限。
