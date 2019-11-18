## 3.8 C Builtin Functions
Vivado HLS支持以下C内置函数：
- __builtin_clz（unsigned int x）：返回x中从最高有效位开始的前导0位的数目。如果x为0，则结果不确定。
- __builtin_ctz（unsigned int x）：返回x中从最低有效位开始的尾随0位的数量。如果x为0，则结果不确定。

以下示例显示了可以使用的这些函数。此示例返回in0中前导零和in1中尾后零的数量之和：
```c
int foo (int in0, int in1) {
 int ldz0 = __builtin_clz(in0);
 int ldz1 = __builtin_ctz(in1);
 return (ldz0 + ldz1);
}
```