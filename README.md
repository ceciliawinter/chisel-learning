# chisel-learning

- [chisel-learning](#chisel-learning)
  * [报错解决](#报错解决)
    + [switch报错](#switch报错)
    + [数据类型错误](#数据类型错误)
    + [模块间传递数据位宽参数](#模块间传递数据位宽参数)
    + [Bool类型赋值](#bool类型赋值)
    + [无隐式时钟域复位信号报错](#无隐式时钟域复位信号报错)
    + [在when中对io端口数据进行修改](#在when中对io端口数据进行修改)
  * [学习问题](#学习问题)
    + [溢出](#溢出)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


记录chisel学习过程中遇到的问题

## 报错解决
### switch报错

部分报错内容：
```
[error] Hello.scala:28:3: not found: value switch
[error]   switch(io.num){
[error]   ^
[error] Hello.scala:30:5: not found: value is
[error]     is("b0000".U) {io.seg := "b11000000".U}
[error]     ^
```
错误原因： 未引用库文件，需要添加
```
import chisel3.util._
```
### 数据类型错误

代码：
```
val addrWidth = 4
val addr = Wire(addrWidth) 
```
报错信息
```
[error] (run-main-0) chisel3.core.Binding$ExpectedChiselTypeException: wire type 'chisel3.core.UInt@2f' must be a Chisel type, not hardware
[error] chisel3.core.Binding$ExpectedChiselTypeException: wire type 'chisel3.core.UInt@2f' must be a Chisel type, not hardware
```
要实现的目的：
```
val addr = Wire(UInt(4.W))  
```
使用参数定义数据长度
```
val addrWidth = 4
val addr = Wire(UInt(addrWidth.W)) 
```

### 模块间传递数据位宽参数

错误代码1：

在此处，signal被定义为Int类型
```
class ResIO(val sigsize : Int) extends Bundle{
    val signal = Input(Int(sigsize.W))
    val res = Output(Bool())
```
报错
```
[error] Template.scala:18:25: Int.type does not take parameters
[error]   val signal = Input(Int(sigsize.W))
[error]                         ^
```

错误代码2：

在此处，signal定义的位宽大小参数sigsize数据类型为UInt
```
class ResIO(val sigsize : UInt) extends Bundle{
    val signal = Input(UInt(sigsize.W))
    val res = Output(Bool())
```
报错
```
[error] Template.scala:18:35: value W is not a member of chisel3.UInt
[error]   val signal = Input(UInt(sigsize.W))
[error]                                   ^
```
注意，如果数据类型是Int类型，Int类型不具有参数

数据定义为UInt类型，可以通过参数来定义数据的大小，该参数必须为Int类型，不可为UInt类型

正确示例：
```
class ResIO(val sigsize : Int) extends Bundle{
   val signal = Input(UInt(sigsize.W))
   val res = Output(Bool())
}
```

### Bool类型赋值

错误代码：
```
//io.res定义部分
val res = Output(Bool())
//赋值部分
io.res := true
```
报错
```
[error]  Template.scala:27:15: type mismatch;
[error]  found   : Boolean(true)
[error]  required: chisel3.core.Data
[error]     io.res := true
[error]               ^
```
stackoverflow上查找到解决方案：https://stackoverflow.com/questions/41658288/comparing-two-bits-type-values-in-chisel-3

Boolean为scala数据类型，在chisel中所需要的是Bool类型

修改代码，将类型转化为Bool
```
io.res := true.B
```

### 无隐式时钟域复位信号报错

报错信息
```
[error] (run-main-0) chisel3.internal.ChiselException: Error: No implicit clock and reset.
[error] chisel3.internal.ChiselException: Error: No implicit clock and reset.
```
https://groups.google.com/forum/#!topic/chisel-users/ixalgSaK0Gg

之前使用BlackBox用verilog代码实现部分功能，使用chisel语言重新编写时，未改变为Module,故产生不存在隐式时钟域reset信号报错

### 在when中对io端口数据进行修改

代码：
```
class ResIO(val sigsize : Int) extends Bundle{
  val signal = Input(UInt(sigsize.W))
  val res = Output(Bool())
}

class Response(val sigsize : Int) extends Module{
  val io = IO(new ResIO(sigsize))
  val signal_pre = RegInit(0.U(sigsize.W))
  signal_pre := RegNext(io.signal)
  val res = Reg(Bool())
  when(signal_pre =/= io.signal){
      io.res := true.B
  }
  io.res := false.B
}
```
报错信息：
```
[error] (run-main-0) firrtl.FIRRTLException: Internal Error! Please file an issue at https://github.com/ucb-bar/firrtl/issues
[error] firrtl.FIRRTLException: Internal Error! Please file an issue at https://github.com/ucb-bar/firrtl/issues
[error]         at firrtl.Utils$.error(Utils.scala:396)
......
```
提示报错为内部错误，尝试修改写法
```
  val res = Reg(Bool())
  when(signal_pre =/= io.signal){
      res := true.B
  }
  res := false.B
  io.res := res
}
```
测试发现在when语句内部修改OutPut端口值，即会出现报错，猜测chisel中不可以使用此语法 ***TODO待查寻是否语法规则如此***


### 溢出

两个操作数位数不等时，结果位数与位数高的操作数相同，会产生溢出的问题加减操作可以改为+% -%，会进行位扩展




