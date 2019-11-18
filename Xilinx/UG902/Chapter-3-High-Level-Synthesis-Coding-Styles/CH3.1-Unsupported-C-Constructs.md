## 3.1 Unsupported C Constructs
尽管Vivado® HLS支持广泛的C语言，但某些结构无法综合，否则可能导致设计流程中的错误。本节讨论必须对编码进行更改的区域，以便在设备中综合和实现函数。

可综合的：
- C函数必须包含设计的整体功能。
- 没有任何函数执行操作系统的系统调用。
- C构造必须为固定或有限大小。
- 这些构造的实现必须明确。

### System Calls
系统调用无法综合，因为它们是与在运行C程序的操作系统上执行某些任务有关的活动。

Vivado® HLS忽略仅显示数据且对算法的执行没有影响的常用系统调用，例如printf()和fprintf(stdout,)。通常，无法综合对系统的调用，应在综合之前将其从函数中删除。此类调用的其他示例是getc()，time()，sleep()，所有这些都对操作系统进行调用。

当执行综合时，**Vivado HLS定义宏`__SYNTHESIS__`**。这允许`__SYNTHESIS__`宏从设计中排除不可综合的代码。

:star: 注意：仅在要综合的代码中使用`__SYNTHESIS__`宏。**请勿在测试平台中使用此宏**，因为C仿真或C RTL协同仿真不遵循该宏。

:warning: 警告！**您不得在代码中或在编译器优化程序中定义或取消定义__SYNTHESIS__宏**，否则编译可能会失败。(个人注：一会说用这个宏注释代码，一会又说不能用，什么鬼？)

在下面的代码示例中，来自子函数的中间结果被保存到硬盘驱动器上的文件中。宏`__SYNTHESIS__`用于确保在综合过程中忽略不可综合的文件写入。
```c++
#include "hier_func4.h"
int sumsub_func(din_t *in1, din_t *in2, dint_t *outSum, dint_t *outSub)
{
 *outSum = *in1 + *in2;
 *outSub = *in1 - *in2;
}
int shift_func(dint_t *in1, dint_t *in2, dout_t *outA, dout_t *outB)
{
 *outA = *in1 >> 1;
 *outB = *in2 >> 2;
}
void hier_func4(din_t A, din_t B, dout_t *C, dout_t *D)
{
 dint_t apb, amb;
 sumsub_func(&A,&B,&apb,&amb);
#ifndef __SYNTHESIS__
 FILE *fp1; // The following code is ignored for synthesis
 char filename[255];
 sprintf(filename,Out_apb_%03d.dat,apb);
 fp1=fopen(filename,w);
 fprintf(fp1, %d \n, apb);
 fclose(fp1);
#endif
 shift_func(&apb,&amb,C,D);
}
```
__SYNTHESIS__宏是一种便捷的方法，可以将不可综合的代码排除在外，而无需从C函数中删除代码本身。使用这样的宏意味着用于仿真的C代码和用于综合的C代码是不同的。

:warning: 警告！如果使用__SYNTHESIS__宏来更改C代码的功能，则可能导致C仿真与C综合之间的结果不同。这种代码中的错误本质上很难调试。不要使用__SYNTHESIS__宏来更改功能。
### Dynamic Memory Usage
任何管理系统内存分配的系统调用，例如malloc()，alloc()和free()，都将使用操作系统内存中存在并在运行时创建和释放的资源 ：要能够综合设计的硬件实现，必须完全独立，并指定所有必需的资源。

综合之前必须从设计代码中删除系统调用中的内存分配。由于动态内存操作用于定义设计的功能，因此必须将其转换为等效的有界表示。
下面的代码示例演示如何将使用malloc()的设计转换为可综合的版本，并重点介绍两种有用的编码样式技术：
- 设计不使用`__SYNTHESIS__`宏。

  用户定义的宏`NO_SYNTH`用于在可综合版本和不可综合版本之间进行选择。这样可以确保在C中仿真相同的代码，并在Vivado® HLS中进行综合。

- 使用malloc()的原始设计中的指针使用固定大小的元素不需要重写。

  可以创建固定大小的资源，并且可以简单地使现有指针指向固定大小的资源。此技术可以防止对现有设计进行手动编码。

```c++
#include "malloc_removed.h"
#include <stdlib.h>
//#define NO_SYNTH
dout_t malloc_removed(din_t din[N], dsel_t width) {  
#ifdef NO_SYNTH
 long long *out_accum = malloc (sizeof(long long));
 int* array_local = malloc (64 * sizeof(int));
#else
 long long _out_accum;
 long long *out_accum = &_out_accum;
 int _array_local[64];
 int* array_local = &_array_local[0];
#endif
 int i,j;
  
 LOOP_SHIFT:for (i=0;i<N-1; i++) {
 if (i<width) 
  *(array_local+i)=din[i];
 else 
  *(array_local+i)=din[i]>>2;
 }
 *out_accum=0;
 LOOP_ACCUM:for (j=0;j<N-1; j++) {
  *out_accum += *(array_local+j);
 }
 return *out_accum;
} 
```
由于此处的编码更改会影响设计的功能，因此Xilinx不建议使用__SYNTHESIS__宏。Xilinx建议您执行以下步骤：
1. 将用户定义的宏NO_SYNTH添加到代码中，然后修改代码。
2. 启用宏NO_SYNTH，执行C仿真，然后保存结果。
3. 禁用宏NO_SYNTH，并执行C仿真以验证结果是否相同。
4. 在禁用用户定义的宏的情况下执行综合。

这种方法论可确保使用C仿真对更新后的代码进行验证，然后再对相同的代码进行综合。与C语言中动态内存使用的限制一样，Vivado HLS不支持（用于综合）动态创建或销毁的C++对象。这包括动态多态性和调用中的动态虚拟函数调用。

以下代码无法综合，因为它会在运行时创建一个新的函数。
```c++
Class A {  
public:
 virtual void bar() {}; 
}; 
void fun(A* a) {  
 a->bar();  
}


A* a = 0;
if (base) 
 a = new A(); 
else 
 a = new B();
fun(a); //原著是foo(a)
```

### Pointer Limitations

**General Pointer Casting**

Vivado HLS不支持通用指针转换，但是支持在自然C类型之间进行指针转换。

**Pointer Arrays**

Vivado HLS支持指针数组进行综合，前提是每个指针都指向一个标量或一个标量数组。指针数组不能指向附加指针。

**Function Pointers**

不支持函数指针。

### Recursive Functions

递归函数无法综合。这适用于可以形成无穷递归的函数，其中无穷无尽：
```c++
unsigned foo (unsigned n) 
{  
    if (n == 0 || n == 1) return 1;  
    return (foo(n-2) + foo(n-1)); 
} 
```
Vivado® HLS不支持尾部递归，在尾部递归中调用次数有限。
```c++
unsigned foo (unsigned m, unsigned n)  
{  
    if (m == 0) return n;  
    if (n == 0) return m; 
    return foo(n, m%n); 
} 
```
在C++中，模板可以实现尾部递归。接下来介绍它。

#### Standard Template Libraries
许多C++标准模板库（STL）包含递归功能，并使用动态内存分配。因此，无法综合STL。STL的解决方案是创建具有相同功能的局部功能，该功能不具有递归，动态内存分配或动态创建和析构的这些特征。

:star: 注意：综合支持标准数据类型，例如std::complex。

