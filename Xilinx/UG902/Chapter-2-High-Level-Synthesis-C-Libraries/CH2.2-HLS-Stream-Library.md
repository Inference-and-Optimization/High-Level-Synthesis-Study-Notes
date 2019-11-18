## 2.1 HLS Stream Library
流数据是一种数据传输，其中数据样本是从第一个样本开始按顺序发送的。流传输无需地址管理。

使用流数据的建模设计在C语言中可能很难。使用指针执行多个读取和/或写入访问的方法可能会带来问题，因为类型限定符和测试台的构造方式有牵连。

Vivado HLS提供了一个C++模板类`hls::stream<>`，用于对流数据结构进行建模。使用`hls::stream<>`类实现的流具有以下属性。
- 在C代码中，`hls::stream<>`的行为类似于无限深度的FIFO。不需要定义`hls::stream<>`的大小。
- 依次读取和写入它们。即，从`hls::stream<>`读取的数据，将**无法再次读取**。
- 默认情况下，顶层接口上的`hls::stream<>`是用`ap_fifo`接口实现的。
- 设计内部的`hls::stream<>`被实现为**深度为2的FIFO**。优化指令STREAM用于更改此默认大小。

本节说明`hls::stream<>`类如何更轻松地使用流数据为设计建模。本节中的主题提供：
- 使用流进行建模的概述以及流的RTL实现。
- 全局流变量的规则。
- 如何使用流。
- 阻塞读取和写入。
- 非阻塞读取和写入。
- 控制FIFO深度。

:star: 注意：`hls::stream`类应始终在函数之间作为C++**引用参数传递**。例如，&my_stream。  

:star: 重要！`hls::stream`类仅在C++设计中使用。**不支持流数组**。

### C Modeling and RTL Implementation
在**软件**（和RTL协同仿真期间的测试平台）中，将**流建模为无限队列**。无需指定任何深度即可在C++中模拟流。可以在函数内部以及函数接口上使用流。内部流可以作为函数传递给参数。

流只能在基于C++的设计中使用。每个`hls::stream<>`对象**必须由单个进程写入并由单个进程读取**。

如果在顶层接口上使用`hls::stream`，则默认情况下在RTL中将其实现为**FIFO接口(ap_fifo)**，但也可以将其最佳实现为**握手接口(ap_hs)或AXI-Stream接口(axis)**。

如果在设计函数内部使用`hls::stream`并综合到**硬件**中，则**将其实现为FIFO，默认深度为2**。在某些情况下，例如，使用插值时，FIFO的深度可能增加以确保FIFO可以容纳硬件产生的所有元素。**如果无法确保FIFO足够大以容纳硬件产生的所有数据样本，则会导致设计停滞**（在C/RTL协同仿真以及硬件实现中可见）。FIFO的深度可以使用STREAM指令在深度优化的情况下进行调整。示例设计`hls_stream`中提供了一个示例。

:star: 重要！在默认的非DATAFLOW区域中使用`hls::stream`变量时，请确保其大小正确。

如果使用`hls::stream`在任务（子功能或循环）之间传输数据，则应立即考虑在`DATAFLOW`区域中实现任务，在该区域中数据从一个任务流向下一个任务。默认（非DATAFLOW）行为是在开始下一个任务之前完成每个任务，**在这种情况下，用于实现`hls::stream`变量的FIFO必须确定大小**，以确保它们足够大以容纳所有生成的数据样本通过生产者任务。无法增加`hls::stream`变量的大小会导致以下错误：
```
ERROR: [XFORM 203-733] An internal stream xxxx.xxxx.V.user.V' with default size is used in a non-dataflow region, which may result in deadlock. Please consider to resize the stream using the directive 'set_directive_stream' or the 'HLS stream' pragma.
```
此错误通知您，在非DATAFLOW区域（默认FIFO深度为2）中，可能不足以容纳生产者任务将所有数据样本写入FIFO的大小。

#### Global and Local Streams
流可以在本地或全局定义。本地流始终实现为内部FIFO。全局流可以实现为内部FIFO或端口：
- 只能**读取或写入**的全局定义的流被推断为顶层**RTL模块的外部端口**。
- 全局定义的流（在顶层函数下面的层次结构中）都可以**读取和写入**，都被实现为**内部FIFO**。

在全局范围内定义的流遵循与任何其他全局变量相同的规则。
### Using HLS Streams
要使用`hls::stream<>`对象，请包含头文件`hls_stream.h`。通过指定类型和变量名称来定义流数据对象。在此示例中，定义了128位无符号整数类型，并将其用于创建名为`my_wide_stream`的流变量。
```c++
#include "ap_int.h"
#include "hls_stream.h"
typedef ap_uint<128> uint128_t;  // 128-bit user defined type
hls::stream<uint128_t> my_wide_stream;  // A stream declaration
```
流必须使用范围命名。Xilinx建议使用上面示例中显示的带范围的`hls::`命名。但是，如果要使用`hls`命名空间，则可以将前面的示例重写为：
```c++
#include <ap_int.h>
#include <hls_stream.h>
using namespace hls;
typedef ap_uint<128> uint128_t;  // 128-bit user defined type
stream<uint128_t> my_wide_stream;  // hls:: no longer required
```
给定一个指定为`hls::stream<T>`的流，类型`T`可以是：
- 任何C++自然数据类型
- Vivado HLS任意精度类型（例如ap_int<>，ap_ufixed<>）
- 一个用户定义的包含以上两种类型任何一个的结构体

注意：包含方法（成员函数）的常规用户定义类（或结构体）不应用作流变量的类型（T）。

流可以被可选地命名。为流提供名称允许在报告中使用该名称。例如，Vivado HLS自动检查以确保在仿真期间读取输入流中的所有元素。给出以下两个流：
```c++
stream<uint8_t> bytestr_in1;
stream<uint8_t> bytestr_in2("input_stream2");
```
关于流中剩余元素的所有警告均报告如下，其中很明显哪个消息与`bytetr_in2`有关：
```
WARNING: Hls::stream 'hls::stream<unsigned char>.1' contains leftover data, which may result in RTL simulation hanging.
WARNING: Hls::stream 'input_stream2' contains leftover data, which may result in RTL simulation hanging.
```
当流传入和传出函数时，必须按引用传递它们，如以下示例所示：
```c++
void stream_function (
  hls::stream<uint8_t> &strm_out,
  hls::stream<uint8_t> &strm_in,
  uint16_t strm_len
)
```
Vivado HLS支持阻塞和非阻塞访问方法。
- **非阻塞**访问只能实现为**FIFO接口**。
- 被实现为**ap_fifo端口**且由**AXI4Stream**资源定义的流端口**不得使用非阻塞访问**。
Vivado HLS示例中提供了使用流的完整设计示例。请参阅GUI欢迎屏幕上可用的设计示例中的`hls_stream`示例。
#### Blocking Reads and Writes
对hls::stream<>对象的基本访问正在阻止读取和写入。这些是使用类方法完成的。如果在空stream FIFO上尝试进行读取，对full stream FIFO进行写操作，或者直到映射到ap_hs接口协议的流完成了完整握手，这些方法就会stall（执行）。

可以在C/RTL协同仿真中观察到stall，当仿真器连续执行，而传输没有任何进展。下图显示了一个stall情况的经典示例，其中，RTL仿真时间持续增长，但是Inter或Intra事务没有任何进展： 
```
// RTL Simulation : "Inter-Transaction Progress" ["Intra-Transaction 
Progress"] @ 
"Simulation Time"
/////////////////////////////////////////////////////////////////////////////
//////
// RTL Simulation : 0 / 1 [0.00%] @ "110000"
// RTL Simulation : 0 / 1 [0.00%] @ "202000"
// RTL Simulation : 0 / 1 [0.00%] @ "404000"
```
##### Blocking Write Methods
在此示例中，变量src_var的值被推送到流中。
```c++
// Usage of void write(const T & wdata)
hls::stream<int> my_stream;
int src_var = 42;
my_stream.write(src_var);
```
<< 操作符已重载，因此可以类似于C++流（例如iostream和filestream）的流插入操作符的方式使用它。将被写入的`hls::stream<>`对象作为左侧参数提供，将被写入的值作为右侧参数提供。
```c++
// Usage of void operator << (T & wdata)
hls::stream<int> my_stream;
int src_var = 42;
my_stream << src_var;
```
##### Blocking Read Methods
此方法从流的开头读取并将值赋值给变量dst_var。
```c++
// Usage of void read(T &rdata)
hls::stream<int> my_stream;
int dst_var;
my_stream.read(dst_var);
```
另外，可以通过将流赋值给（例如使用=，+=）左侧的对象来读取流中的下一个对象：
```c++
// Usage of T read(void)
hls::stream<int> my_stream;
int dst_var = my_stream.read();
```
“ >>”运算符已重载，以允许使用类似于C++流（例如iostream和filestream）的流提取操作符。hls::stream作为LHS参数提供，而目标变量作为RHS提供。
```c++
// Usage of void operator >> (T & rdata)
hls::stream<int> my_stream;
int dst_var;
my_stream >> dst_var;
```
#### Non-Blocking Reads and Writes
还提供了非阻塞的写入和读取方法。即使在empty stream上尝试读取或对full stream进行写操作，这些操作也可以继续执行。

这些方法返回指示访问状态的布尔值（如果成功，则为true，否则为false）。包括了其他方法来测试hls::stream<>流的状态。

:star: 重要！**非阻塞行为仅在使用`ap_fifo`协议的接口上受支持**。更具体地说，`AXI-Stream`标准和Xilinx `ap_hs` IO协议不支持非阻塞访问。

在C仿真过程中，流具有无限大小。因此，无法使用C仿真来验证流是否已满。仅当定义FIFO大小（默认大小为1或由STREAM指令定义的任意大小）时，才可以在RTL仿真期间验证这些方法。

:star: 重要！如果设计被指定使用块级I/O协议`ap_ctrl_none`，并且设计包含任何采用非阻塞行为的`hls::stream`变量，则不能保证C/RTL协同仿真完成。

##### Non-Blocking Writes
此方法尝试将变量`src_var`推送到流`my_stream`中，如果成功，则返回布尔值`true`。否则，将返回`false`，并且队列不受影响。
```c++
// Usage of void write_nb(const T & wdata)
hls::stream<int> my_stream;
int src_var = 42;
if (my_stream.write_nb(src_var)) {
 // Perform standard operations
 ...
} else {
 // Write did not occur
 return;
}
```
##### Fullness Test
```c++
bool full(void)
```
当且仅当`hls::stream<>`对象已满时，才返回`true`。
```c++
// Usage of bool full(void)
hls::stream<int> my_stream;
int src_var = 42;
bool stream_full;
stream_full = my_stream.full();
```
##### Non-Blocking Read
```c++
bool read_nb(T & rdata)
```
此方法尝试从流中读取一个值，如果成功，则返回`true`。否则，将返回`false`，并且队列不受影响。
```c++
// Usage of void read_nb(const T & wdata)
hls::stream<int> my_stream;
int dst_var;
if (my_stream.read_nb(dst_var)) {
 // Perform standard operations
 ...
} else {
 // Read did not occur
 return;
}
```
##### Emptiness Test
```c++
bool empty(void)
```
如果`hls::stream<>`为空，则返回`true`。
```c++
// Usage of bool empty(void)
hls::stream<int> my_stream;
int dst_var;
bool stream_empty;
stream_empty = my_stream.empty();
```

以下示例显示了当RTL FIFO为满或空时，无阻塞访问和满/空测试的组合如何提供错误处理功能：
```c++
#include "hls_stream.h"
using namespace hls;
typedef struct {
   short    data;
   bool     valid;
   bool     invert;
} input_interface;
bool invert(stream<input_interface>& in_data_1,
            stream<input_interface>& in_data_2,
            stream<short>& output
  ) {
  input_interface in;
  bool full_n;
// Read an input value or return
  if (!in_data_1.read_nb(in))
      if (!in_data_2.read_nb(in))
          return false;
// If the valid data is written, return not-full (full_n) as true
  if (in.valid) {
    if (in.invert)
      full_n = output.write_nb(~in.data);
    else
      full_n = output.write_nb(in.data);
  }
  return full_n;
}
```

#### Controlling the RTL FIFO Depth
对于大多数使用流数据的设计，默认的RTL FIFO深度为2就足够了。流数据通常是在一个样本中一次处理的。

对于实现要求深度大于2的FIFO的多速率设计，必须确定（并使用STREAM指令设置）完成RTL仿真所需的深度。如果FIFO深度不足，则RTL协同模拟停顿。

由于无法在GUI指令面板中查看流对象，因此无法将STREAM方向直接应用于该Pane中。

右键单击在其上声明了`hls::stream<>`的对象（或在参数列表中使用或存在的）的函数，以：
- 选择STREAM指令。
- 使用流变量的名称手动填充variable字段。

或者，您可以：
- 在指令.tcl文件中手动指定STREAM指令，或
- 在source中将其作为pragma添加。

### C/RTL Co-Simulation Support
Vivado HLS C/RTL协同仿真功能**不支持在顶层接口中包含`hls::stream<>`成员的结构体或类**。Vivado HLS支持这些结构或类进行综合。
```c++
typedef struct {
   hls::stream<uint8_t> a;
   hls::stream<uint16_t> b;
} strm_strct_t;
void dut_top(strm_strct_t indata, strm_strct_t outdata) {  }
```
这些限制**适用于顶层函数参数和全局声明的对象**。如果将流的结构体用于综合，则必须使用外部RTL仿真器和用户创建的HDL测试平台来验证设计。具有严格内部链接的`hls::stream<>`对象没有此类限制。