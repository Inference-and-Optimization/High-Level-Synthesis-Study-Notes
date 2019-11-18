## 3.2 C Test Bench
综合任何块的第一步是验证C函数是否正确。该步骤由测试台执行。编写好的测试台可以大大提高您的生产率。

C函数比RTL仿真的执行速度快几个数量级。与在RTL进行开发相比，在综合之前使用C开发和验证算法的效率更高。
- 利用C语言开发的关键是要有一个测试台，以根据已知的良好结果检查函数的结果。因为已知该算法是正确的，所以可以在综合之前验证任何代码更改。
- Vivado® HLS重用C测试台以验证RTL设计。使用Vivado HLS时无需创建RTL测试台。如果测试台检查顶层函数的结果，则可以通过仿真验证RTL。

:star: 注意：要为测试台提供输入参数，请选择Project→Project Settings，单击Simulation，然后使用Input Arguments选项。测试台不能执行交互用户输入。Vivado HLS GUI没有命令控制台，并且在执行测试台时不能接受用户输入。

Xilinx建议您从测试台中分离用于综合的顶层函数，并使用头文件。以下代码示例显示了一种设计，其中hier_func调用两个子函数：
- sumsub_func执行加法和减法。
- shift_func执行移位。

数据类型在头文件（hier_func.h）中定义，该文件也有描述：
```c++
#include "hier_func.h"
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
void hier_func(din_t A, din_t B, dout_t *C, dout_t *D)
{
 dint_t apb, amb;
 sumsub_func(&A,&B,&apb,&amb);
 shift_func(&apb,&amb,C,D);
}
```
顶层函数可以包含多个子函数。只能有一个用于综合的顶层函数。要综合多个函数，请将它们组合为一个顶层函数。

要综合函数hier_func：
1. 将以上示例所示的文件作为设计文件添加到Vivado HLS项目中。
2. 将顶层函数指定为hier_func。

综合后：
- 顶层函数的参数（在上面的示例中为A，B，C和D）被综合为RTL端口。
- 顶层函数（上例中的sumsub_func和shift_func）内的函数被综合为分层的块。

上面的示例中的头文件（hier_func.h）显示了如何使用宏以及typedef语句使代码更具可移植性和可读性。稍后的部分将展示typedef语句如何优化类型以及变量的位宽，以便在最终的FPGA实现中改善面积和性能。
```c
#ifndef _HIER_FUNC_H_
#define _HIER_FUNC_H_
#include <stdio.h>
#define NUM_TRANS 40
typedef int din_t;
typedef int dint_t;
typedef int dout_t;
void hier_func(din_t A, din_t B, dout_t *C, dout_t *D);
#endif
```
此示例中的头文件包含设计文件中不需要的一些定义（例如NUM_TRANS）。这些定义由测试台使用，该测试台还包括相同的头文件。

以下代码示例显示了第一个示例中所示设计的测试台。
```c
#include "hier_func.h"
int main() {
 // Data storage
 int a[NUM_TRANS], b[NUM_TRANS];
 int c_expected[NUM_TRANS], d_expected[NUM_TRANS];
 int c[NUM_TRANS], d[NUM_TRANS];
  //Function data (to/from function)
 int a_actual, b_actual;
 int c_actual, d_actual;
  // Misc
 int retval=0, i, i_trans, tmp;
 FILE *fp;
 // Load input data from files
 fp=fopen(tb_data/inA.dat,r);
 for (i=0; i<NUM_TRANS; i++){
  fscanf(fp, %d, &tmp);
  a[i] = tmp;
 } 
 fclose(fp);
 fp=fopen(tb_data/inB.dat,r);
 for (i=0; i<NUM_TRANS; i++){
  fscanf(fp, %d, &tmp);
  b[i] = tmp;
 } 
 fclose(fp);
 // Execute the function multiple times (multiple transactions)
 for(i_trans=0; i_trans<NUM_TRANS-1; i_trans++){
  //Apply next data values
  a_actual = a[i_trans];
  b_actual = b[i_trans];
  
  hier_func(a_actual, b_actual, &c_actual, &d_actual);
  
  //Store outputs
  c[i_trans] = c_actual;
  d[i_trans] = d_actual;
 }
 // Load expected output data from files
 fp=fopen(tb_data/outC.golden.dat,r);
 for (i=0; i<NUM_TRANS; i++){
  fscanf(fp, %d, &tmp);
  c_expected[i] = tmp;
 } 
 fclose(fp);
 fp=fopen(tb_data/outD.golden.dat,r);
 for (i=0; i<NUM_TRANS; i++){
  fscanf(fp, %d, &tmp);
  d_expected[i] = tmp;
 } 
 fclose(fp);
 // Check outputs against expected
 for (i = 0; i < NUM_TRANS-1; ++i) {
  if(c[i] != c_expected[i]){
    retval = 1;
  }
  if(d[i] != d_expected[i]){
    retval = 1;
  }
 }
 // Print Results
 if(retval == 0){
  printf(    *** *** *** *** \n); 
  printf(    Results are good \n); 
  printf(    *** *** *** *** \n); 
 } else {
  printf(    *** *** *** *** \n); 
  printf(    Mismatch: retval=%d \n, retval); 
  printf(    *** *** *** *** \n);  
 }
 // Return 0 if outputs are corre
 return retval;
}
```

### Productive Test Benches
该测试台示例突出显示了一个高效测试台的一些特征，例如：
- 综合的顶层函数（hier_func）对多个事务执行，如宏NUM_TRANS所定义。该执行允许应用和验证许多不同的数据值。测试台仅取决于其执行的各种测试。
- 将函数输出与已知的值进行比较。在此示例中，从文件读取了已知的值，但也可以将其作为测试台的一部分进行计算。
- main()函数的返回值设置为：
  - 零：结果正确。
  - 非零值：结果不正确。

:star: 注意：测试台可以返回任何非零值。复杂的测试台可以根据不同的类型或故障返回不同的值。如果测试台在C仿真或C/RTL协同仿真之后返回非零值，则Vivado® HLS报告错误，并且仿真失败。

:white_check_mark: 推荐：因为系统环境（例如Linux，Windows或Tcl）会解释main()函数的返回值，所以Xilinx建议您将返回值限制在8位范围内，以实现可移植性和安全性。

:warning: 警告！您有责任确保测试台检查结果。如果测试台未检查结果但返回零，则Vivado HLS表示即使未实际检查结果，模拟测试也已通过。即使输出数据正确且有效，如果测试台没有将main()的值返回零，Vivado HLS也会报告仿真失败。

具有这些属性的测试平台可以在综合之前快速测试并验证对C函数所做的任何更改，并且可以在RTL上重用，从而可以更轻松地验证RTL。

### Design Files and Test Bench Files
由于Vivado® HLS将C测试台重新用于RTL验证，因此，当将它们添加到Vivado HLS项目时，要求将测试台和任何相关文件表示为测试台文件。与测试台关联的文件是以下任何文件：
- 由测试台访问
- 测试台正确运行所必需。

此类文件的示例包括测试台示例中的数据文件inA.dat和inB.dat。您必须将它们作为测试台文件添加到Vivado HLS项目中。

在Vivado HLS项目中标识测试台文件的需求不要求设计和测试台位于单独的文件中（尽管建议使用单独的文件）。

在下面的示例中，重复了C测试台的相同设计。唯一的区别是顶层函数已重命名为hier_func2，以区分示例。

使用相同的头文件和测试台（除了从hier_func更改为hier_func2以外），在Vivado HLS中**将sumsub_func综合为顶层函数**所需的唯一更改是： 
- 将sumsub_func设置为Vivado HLS项目中的顶层函数。
- 在以下示例中将其添加为设计文件和项目文件。sumsub_func（函数hier_func2）以上的级别现在是测试台的一部分。它必须包含在RTL模拟中。

即使未在main()函数内部显式实例化sumsub_func上的函数，该函数的其余部分（hier_func2和shift_func）仍确认其可以正确运行，因此属于测试台。
```c++
#include "hier_func2.h"
int sumsub_func(din_t *in1, din_t *in2, dint_t *outSum, dint_t *outSub)
{
 *outSum = *in1 + *in2;
 *outSub = *in1 - *in2;
}
int shift_func(dint_t *in1, dint_t *in2, dout_t *outA, dout_t *outB) // Test Bench一部分
{
 *outA = *in1 >> 1;
 *outB = *in2 >> 2;
}
void hier_func2(din_t A, din_t B, dout_t *C, dout_t *D) // Test Bench一部分
{
 dint_t apb, amb;
 sumsub_func(&A,&B,&apb,&amb);  // 顶层函数
 shift_func(&apb,&amb,C,D);
}
```

### Combining Test Bench and Design Files
您还可以将设计和测试台纳入单个设计文件中。下面的示例具有与C测试台相同的功能，不同之处在于所有内容均在单个文件中。hier_func重命名为hier_func3以确保示例是唯一的。

:star: 重要！如果测试台和设计位于单个文件中，则必须将该文件作为设计文件和测试台文件添加到Vivado® HLS项目中。

```c++
#include <stdio.h>
#define NUM_TRANS 40
typedef int din_t;
typedef int dint_t;
typedef int dout_t;
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
void hier_func3(din_t A, din_t B, dout_t *C, dout_t *D)
{
 dint_t apb, amb;
 sumsub_func(&A,&B,&apb,&amb);
 shift_func(&apb,&amb,C,D);
}
int main() {
 // Data storage
 int a[NUM_TRANS], b[NUM_TRANS];
 int c_expected[NUM_TRANS], d_expected[NUM_TRANS];
 int c[NUM_TRANS], d[NUM_TRANS];
 //Function data (to/from function)
 int a_actual, b_actual;
 int c_actual, d_actual;
 // Misc
 int retval=0, i, i_trans, tmp;
 FILE *fp;
 // Load input data from files
 fp=fopen(tb_data/inA.dat,r);
 for (i=0; i<NUM_TRANS; i++){
  fscanf(fp, %d, &tmp);
  a[i] = tmp;
 } 
 fclose(fp);
 fp=fopen(tb_data/inB.dat,r);
 for (i=0; i<NUM_TRANS; i++){
  fscanf(fp, %d, &tmp);
  b[i] = tmp;
 } 
 fclose(fp);
 // Execute the function multiple times (multiple transactions)
 for(i_trans=0; i_trans<NUM_TRANS-1; i_trans++){
  //Apply next data values
  a_actual = a[i_trans];
  b_actual = b[i_trans];

  hier_func3(a_actual, b_actual, &c_actual, &d_actual);
      
  //Store outputs
  c[i_trans] = c_actual;
  d[i_trans] = d_actual;
 }
 // Load expected output data from files
 fp=fopen(tb_data/outC.golden.dat,r);
 for (i=0; i<NUM_TRANS; i++){
  fscanf(fp, %d, &tmp);
  c_expected[i] = tmp;
 } 
 fclose(fp);
 fp=fopen(tb_data/outD.golden.dat,r);
 for (i=0; i<NUM_TRANS; i++){
  fscanf(fp, %d, &tmp);
  d_expected[i] = tmp;
 } 
 fclose(fp);
 // Check outputs against expected
 for (i = 0; i < NUM_TRANS-1; ++i) {
  if(c[i] != c_expected[i]){
    retval = 1;
  }
  if(d[i] != d_expected[i]){
    retval = 1;
  }
 }
 // Print Results
 if(retval == 0){
  printf(    *** *** *** *** \n); 
  printf(    Results are good \n); 
  printf(    *** *** *** *** \n); 
 } else {
  printf(    *** *** *** *** \n); 
  printf(    Mismatch: retval=%d \n, retval); 
  printf(    *** *** *** *** \n); 
 }
 // Return 0 if outputs are correct
 return retval;
}
```