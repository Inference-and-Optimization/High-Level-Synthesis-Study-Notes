## 3.10 C++ Classes and Templates
Vivado HLS综合完全支持C++类。**综合的顶层必须是一个函数。类不能是综合的顶层**。要综合类成员函数，请**将类本身实例化为函数**。**不要简单地将顶层类实例化到测试平台中**。下面的代码示例演示如何在顶层函数cpp_FIR中实例化CFir类（在下面讨论的头文件中定义），并用于实现FIR滤波器。
```c
#include "cpp_FIR.h"
// Top-level function with class instantiated
data_t cpp_FIR(data_t x)
{
 static CFir<coef_t, data_t, acc_t> fir1;
 cout << fir1;
 return fir1(x);
}
```
:star: 重要！类和类成员函数不能成为综合的顶层。而在顶层函数中实例化该类。

在检查上面的C++ FIR滤波器示例中用于实现设计的类之前，**值得注意的是Vivado HLS在综合过程中会忽略标准输出流cout**。综合后，Vivado HLS发出以下警告：
```
INFO [SYNCHK-101] Discarding unsynthesizable system call: 
'std::ostream::operator<<' (cpp_FIR.h:108)
INFO [SYNCHK-101] Discarding unsynthesizable system call: 
'std::ostream::operator<<' (cpp_FIR.h:108)
INFO [SYNCHK-101] Discarding unsynthesizable system call: 'std::operator<< 
<std::char_traits<char> >' (cpp_FIR.h:110)
```
以下代码示例显示了头文件cpp_FIR.h，其中包括类CFir的定义及其关联的成员函数。在此示例中，运算符成员函数（）和<<是重载运算符，它们分别用于执行主算法，并与cout一起使用以格式化数据以在C仿真期间显示。

```c++

#include <fstream>
#include <iostream>
#include <iomanip>
#include <cstdlib>
using namespace std;
#define N 85
typedef int coef_t;
typedef int data_t;
typedef int acc_t;

// Class CFir definition
template<class coef_T, class data_T, class acc_T>
class CFir {
 protected:
   static const coef_T c[N];
   data_T shift_reg[N-1];
 private:
 public:
   data_T operator()(data_T x);
   template<class coef_TT, class data_TT, class acc_TT>
   friend ostream &
   operator << (ostream& o, const CFir <coef_TT, data_TT, acc_TT> &f);
};

// Load FIR coefficients
template<class coef_T, class data_T, class acc_T>
const coef_T CFir<coef_T, data_T, acc_T>::c[N] = {
 #include "cpp_FIR.h"
};
// FIR main algorithm
template<class coef_T, class data_T, class acc_T>
data_T CFir<coef_T, data_T, acc_T>::operator()(data_T x) {
 int i;
 acc_t acc = 0;
 data_t m;
 loop: for (i = N-1; i >= 0; i--) {
   if (i == 0) {
      m = x;
      shift_reg[0] = x;
   } else {
      m = shift_reg[i-1];
      if (i != (N-1))
      shift_reg[i] = shift_reg[i - 1];
   }
   acc += m * c[i];
 }
 return acc;
}
// Operator for displaying results
template<class coef_T, class data_T, class acc_T>
ostream& operator<<(ostream& o, const CFir<coef_T, data_T, acc_T> &f) {
 for (int i = 0; i < (sizeof(f.shift_reg)/sizeof(data_T)); i++) {
    o << shift_reg[ << i << ]=  << f.shift_reg[i] << endl;
 }
 o << ______________ << endl;
 return o;
}
data_t cpp_FIR(data_t x);
```

下面的代码示例显示了C++ FIR过滤器示例中的测试平台，并演示了如何调用和验证cpp_FIR上的顶层函数。此示例突出显示了Vivado HLS综合良好测试平台的一些重要要点：
- 根据已知的良好值检查输出结果。
- 如果确认结果正确，则测试台将返回0。

```c++
#include "cpp_FIR.h"
int main() {
 ofstream result;
 data_t output;
 int retval=0;
 // Open a file to saves the results
 result.open(result.dat);
 // Apply stimuli, call the top-level function and saves the results
 for (int i = 0; i <= 250; i++)
 {
    output = cpp_FIR(i);
    result << setw(10) << i;
    result << setw(20) << output;
    result << endl;
 }
 result.close();
 // Compare the results file with the golden results
 retval = system(diff --brief -w result.dat result.golden.dat);
 if (retval != 0) {
    printf(Test failed  !!!\n); 
    retval=1;
 } else {
    printf(Test passed !\n);
 }
 // Return 0 if the test
 return retval;
}
```
cpp_FIR的C++测试平台

将指令应用于类中定义的对象：
1. 打开定义该类的文件（通常是头文件）。
2. 使用“Directives”选项卡应用该指令。

与函数一样，一个类的所有实例都对其应用了相同的优化。

### Constructors, Destructors, and Virtual Functions
每当声明类对象时，都将包含并综合类构造函数和析构函数。

Vivado HLS支持虚函数（包括抽象功能）以进行综合，前提是它可以在详细说明过程中静态确定该函数。在以下情况下，Vivado HLS不支持虚函数综合： 
- 可以在多层继承类层次结构中定义虚拟函数，但只使用单个继承。
- 仅当可以在编译时确定指针对象时才支持动态多态。例如，此类指针不能在if-else或loop构造中使用。
- STL容器**不能包含对象的指针**，并且不能调用多态函数。例如：
  ```c++
  vector<base *> base_ptrs(10);
  //Push_back some base ptrs to vector.
  for (int i = 0; i < base_ptrs.size(); ++i) {
    //Static elaboration cannot resolve base_ptrs[i] to actual data type.
    base_ptrs[i]->virtual_function(); 
  }
  ```
- Vivado HL**S不支持基础对象指针是全局变量的情况**。例如：
  ```c++
  Base *base_ptr; 
  void func()
  {
  ...
  base_prt->virtual_function();
  ...
  }
  ```
- 基础对象指针**不能是类定义中的成员变量**。例如：
  ```c++
  // Static elaboration cannot bind base object pointer with correct data 
  type.
  class A
  {
    ...
    Base *base_ptr;
    void set_base(Base *base_ptr);
    void some_func();
    ...
  };
  void A::set_base(Base *ptr)
  {
    this.base_ptr = ptr;
  }
  void A::some_func()
  {
    â¦.
    base_ptr->virtual_function();
    â¦.
  }
  ```
- 如果基础对象指针或引用位于构造函数的功能参数列表中，则Vivado HLS不会对其进行转换。ISO C++标准在第12.7节中对此进行了描述：有时行为未定义。
  ```c++
  class A {
    A(Base *b) {
      b-> virtual _ function ();
    }
  };
  ```
### Global Variables and Classes
Xilinx不建议在类中使用全局变量。它们可以防止发生某些优化情况。在下面的代码示例中，一个类用于创建过滤器的组件（polyd_cell类用作执行移位，乘法和累加操作的组件）。
```c++
typedef long long acc_t;
typedef int mult_t;
typedef char data_t;
typedef char coef_t;
#define TAPS 3
#define PHASES 4
#define DATA_SAMPLES 256
#define CELL_SAMPLES 12
// Use k on line 73 static int k;
template <typename T0, typename T1, typename T2, typename T3, int N>
class polyd_cell {
private:
public:
  T0 areg;
  T0 breg;
  T2 mreg;
  T1 preg;
  T0 shift[N];
  int k;   //line 73
  T0 shift_output;
  void exec(T1 *pcout, T0 *dataOut, T1 pcin, T3 coeff, T0 data, int col)
  {
  Function_label0:;
    if (col==0) {
      SHIFT:for (k = N-1; k >= 0; --k) {
        if (k > 0) 
          shift[k] = shift[k-1];
        else 
          shift[k] = data;
      }
      *dataOut = shift_output;
      shift_output = shift[N-1];
    }
    *pcout = (shift[4*col]* coeff) + pcin;
  }
};
// Top-level function with class instantiated
void cpp_class_data (
 acc_t *dataOut,
 coef_t coeff1[PHASES][TAPS],
 coef_t  coeff2[PHASES][TAPS],
 data_t  dataIn[DATA_SAMPLES],
 int  row
) {
 
 acc_t pcin0 = 0;
 acc_t pcout0, pcout1;
 data_t dout0, dout1;
 int col;
 static acc_t accum=0;
 static int sample_count = 0;
 static polyd_cell<data_t, acc_t, mult_t, coef_t, CELL_SAMPLES> polyd_cell0;
 static polyd_cell<data_t, acc_t, mult_t, coef_t, CELL_SAMPLES> polyd_cell1;
 
 COL:for (col = 0; col <= TAPS-1; ++col) {
  polyd_cell0.exec(&pcout0,&dout0,pcin0,coeff1[row][col],dataIn[sample_count],col);
  polyd_cell1.exec(&pcout1,&dout1,pcout0,coeff2[row][col],dout0,col);
  if ((row==0) && (col==2)) {
      *dataOut = accum;
      accum = pcout1;
  } else {
      accum = pcout1 + accum;
  }
 }
 sample_count++;
}
```
在polyd_cell类中，有一个SHIFT循环用于移位数据。如果删除了循环SHIFT中使用的循环索引k并用k的全局索引替换（在示例中已显示，但注释为static int k），则Vivado HLS无法对使用了polyd_cell类的任何循环或函数进行流水化处理。Vivado HLS将发出以下消息：
```
@W [XFORM-503] Cannot unroll loop 'SHIFT' in function 'polyd_cell<char, 
long long, 
int, char, 12>::exec' completely: variable loop bound.
```
使用局部非全局变量进行循环索引可确保Vivado HLS可以执行所有优化。
### Templates
Vivado HLS支持使用C++中的模板进行综合。Vivado HLS不支持模板用于顶层函数。

:star: IMPORTANT! The top-level function cannot be a template.

#### Using Templates to Create Unique Instances
对于模板参数的每个不同值，模板函数中的静态变量都有独立的拷贝。
```c++
template<int NC, int K>
void startK(int* dout) {
 static int acc=0;
 acc  += K;
 *dout = acc;
}
void foo(int* dout) {
 startK<0,1> (dout);
}
void goo(int* dout) {
 startK<1,1> (dout);
}
int main() {
 int dout0,dout1;
 for (int i=0;i<10;i++) {
 foo(&dout0);
 goo(&dout1);
   cout <<"dout0/1 = "<<dout0<<" / "<<dout1<<endl;
 }
    return 0;
}
```
输出
```
dout0/1 = 1 / 1
dout0/1 = 2 / 2
dout0/1 = 3 / 3
dout0/1 = 4 / 4
dout0/1 = 5 / 5
dout0/1 = 6 / 6
dout0/1 = 7 / 7
dout0/1 = 8 / 8
dout0/1 = 9 / 9
dout0/1 = 10 / 10
```
**Using Templates for Recursion**
也可以使用模板来实现标准C综合（递归函数）中不支持的递归形式。

以下代码示例显示了使用模板化结构实现尾递归斐波那契算法的情况。进行综合的关键是使用一个类来实现递归中的最终调用，其中使用的模板大小为1。
```c++
//Tail recursive call
template<data_t N> struct fibon_s {
  template<typename T>
    static T fibon_f(T a, T b) {
       return fibon_s<N-1>::fibon_f(b, (a+b));
 }
};
// Termination condition
template<> struct fibon_s<1> {
  template<typename T>
    static T fibon_f(T a, T b) {
       return b;
 }
};
void cpp_template(data_t a, data_t b, data_t &dout){
 dout = fibon_s<FIB_N>::fibon_f(a,b);
}
```