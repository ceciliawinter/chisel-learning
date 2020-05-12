# chisel-learning

- [chisel-learning](#chisel-learning)
  * [报错解决](#报错解决)
    + [switch报错](#switch报错)
    + [数据类型错误](#数据类型错误)
    + [模块间传递数据位宽参数](#模块间传递数据位宽参数)
    + [Bool类型赋值](#bool类型赋值)
    + [无隐式时钟域复位信号报错](#无隐式时钟域复位信号报错)
    + [在when中对io端口数据进行修改](#在when中对io端口数据进行修改)
    + [scala版本导致的错误](#scala版本导致的错误)
    + [reset类型错误](#reset类型错误)
    + [引用的包包含同名字段](#引用的包包含同名字段)
    + [io中使用数组](#io中使用数组)
  * [学习问题](#学习问题)
    + [溢出](#溢出)
    + [修改UInt某一位](#修改uint某一位)
    + [状态无法保持](#状态无法保持)
    + [端口错误优化](#端口错误优化)
    + [memory载入数据](#memory载入数据)
    + [使用Analog](#使用analog)
    + [模块名称参数化](#模块名称参数化)
    
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
```
测试发现在when语句内部修改OutPut端口值，即会出现报错，猜测chisel中不可以使用此语法 ***TODO待查寻是否语法规则如此***

### scala版本导致的错误

在尝试使用scala命令行时，使用scala 2.11.x试，命令行仅显示输出，不显示输入，因此将scala版本更换为2.12.x

https://stackoverflow.com/questions/49788781/ubuntu-scala-repl-commands-not-typed-on-console

更换了scala版本后，原本正常运行的chisel3程序不能正常运行，产生报错 *** is not a member of chisel3.Bundle

https://stackoverflow.com/questions/58365679/value-is-not-a-member-of-chisel3-bundle

scala可以安装多个版本，使用scala2.12.x的命令行，在chisel工程中，build.sbt中选择2.11.x即可

### reset类型错误

报错代码：
```
dram.io.clk_and_rst.sys_rst := reset
```
dram.io.clk_and_rst.sys_rs定义：
```
val sys_rst = Input(Bool())
```
报错信息
```
[error] chisel3.internal.ChiselException: Connection between sink (Bool(IO clk_and_rst_sys_rst in my_mig_ddr2)) and source (Reset(IO in unelaborated dramtest)) failed @: Sink (Bool(IO clk_and_rst_sys_rst in my_mig_ddr2)) and Source (Reset(IO in unelaborated dramtest)) have different types.
[error]         ...
```
再将reset信号接出时，对应的类型应为Reset(),但是用withClockAndReset多时钟域时，对应的reset信号的数据类型应为Bool，修改后代码：
```
val sys_rst = Input(Reset())
val ui_clk_sync_rst = Output(Bool())
...

dram.io.clk_and_rst.sys_rst := reset
withClockAndReset(dram.io.clk_and_rst.ui_clk, dram.io.clk_and_rst.ui_clk_sync_rst)
...
```

### 引用的包包含同名字段

在使用withClockAndReset时，根据查阅到的资料为
> 注意，在编写代码时不能写成“import chisel3.core._”，这会扰乱“import chisel3._”的导入内容。正确做法是用“import chisel3.experimental._”导入experimental对象，它里面用同名字段引用了单例对象chisel3.core.withClockAndReset，这样就不需要再导入core包。

但是实际使用中依旧会出现报错
```
[error] reference to withClockAndReset is ambiguous;
[error] it is imported twice in the same scope by
[error] import chisel3.experimental._
[error] and import chisel3._
[error]     withClockAndReset(dram.io.ui_clk, dram.io.ui_clk_sync_rst) {
```
TODO解决方案

因为端口连接需要attach，必须引用chisel3.experimental._ ，尝试在使用attach前再引用chisel3.experimental._ , 不在文件开头引用，可行

### io中使用数组

报错代码：
```
 val io = IO(new Bundle{
  val p1_finish = Array.fill (2) ( Input(Bool()))
 })
 val p1_finish_state = Wire(Bool())
 p1_finish_state := io.p1_finish(0) && io.p1_finish(1)
```

报错信息：
```
[error] (run-main-0) chisel3.core.Binding$ExpectedHardwareException: bits operated on 'chisel3.core.Bool@22' must be hardware, not a bare Chisel type. Perhaps you forgot to wrap it in Wire(_) or IO(_)?
```

修改端口定义方式，使用Chisel的特性Vec来定义数组，不使用Scala类型

https://www.cnblogs.com/JamesDYX/p/10082385.html

```
 val io = IO(new Bundle{
  val p1_finish = Input(Vec(2, Bool()))
 })
 val p1_finish_state = Wire(Bool())
 p1_finish_state := io.p1_finish(0) && io.p1_finish(1)
```



## 学习问题

### 溢出

两个操作数位数不等时，结果位数与位数高的操作数相同，会产生溢出的问题加减操作可以改为+% -%，会进行位扩展

### 修改UInt某一位

[chisel3 wiki](https://github.com/freechipsproject/chisel3/wiki/Cookbook#how-do-i-do-subword-assignment-assign-to-some-bits-in-a-uint)

chisel3不支持修改子词，可以使用Bundles和Vecs来表达

将UInt转为Vec
```
val data = RegInit(0.U(32.W))
val data_bitmap = VecInit(data.asBools))
data_bitmap(0) := true.B
data := data_bitmap.asUInt
```
ps 目前推荐使用asBools代替chisel3 wiki中提到的toBools

### 状态无法保持

代码中包含数据visited_map,visited_map_bitmap
bitmap用于对每一位的修改

```
val visited_map = RegInit(0.U(32.W))
val visited_map_bitmap = VecInit(visited_map.asBools)
```

修改前
```
dinbReg := visited_map_bitmap.asUInt
```
将数据连接到bram上时，直接将visited_map_bitmap.asUInt连接到bram的dinb接口，visited_map_bitmap值在下一次状态跳变时没有保持，修改为:

```
visited_map := visited_map_bitmap.asUInt
dinbReg := visited_map
```

查看波形图，visited_map_bitmap状态可以持续保持

### 端口错误优化

在生成verilog代码时，单独综合某一模块，所有的端口与chisel代码保持一致，但是在综合Top模块时，调用的某些模块端口被优化，优化的部分内容缺失后无法实现设置的功能，可以取消相关优化
[chisel dontTouch](https://www.chisel-lang.org/api/latest/chisel3/dontTouch$.html)

```
import chisel3.dontTouch
dontTouch(io)
val a = dontTouch(...)
```

### memory载入数据

需要初始化memory中的数据可以使用loadMemoryFromFile来实现
[chisel loadMemoryFromFile](https://www.chisel-lang.org/api/latest/chisel3/util/experimental/loadMemoryFromFile$.html)
```
    val ram = Mem(65535, UInt(64.W))
    loadMemoryFromFile(ram, "./mem.txt")
```

可以使用2进制或格式进制数据，注意文件格式如下：

```
0
1
2
```

### 使用Analog

使用Analog声明位宽，以实现在blackbox中使用verilog的inout端口

需要chisel版本在3.1以上且引用import chisel3.experimental._
```
import chisel3.experimental._
···
val ddr2_dq = Analog(8.W)
```

### 模块名称参数化


Chisel2内有setName功能，但是Chisel3没有，使用desiredName来实现
[chisel3 wiki](https://github.com/freechipsproject/chisel3/wiki/Cookbook#how-can-i-dynamically-setparametrize-the-name-of-a-module)

在chisel3 wiki中讲解了使用desiredName来参数化模块名称，但是该名称仍是固定的

尝试使用与setName类似的形式实现，用“+”连接参数

实现代码内部有两个kernel，两个kernel需要使用个不同的bram ip核（即blk_mem_gen模块）

在blk_mem_gen模块内部使用desiredName使不同的kernel内部的bram具有不同的名称

主要传进去的参数num为Int类型，不要使用UInt
```
class blk_mem_gen_IO(implicit val conf : Configuration) extends Bundle{
	val addra = Input(UInt(conf.Addr_width.W))
	val dina = Input(UInt(conf.Data_width.W))
	val douta = Output(UInt(conf.Data_width.W))
}

class blk_mem_gen(val num : Int)(implicit val conf : Configuration) extends BlackBox{
	val io = IO(new blk_mem_gen_IO)
	override def desiredName = "blk_mem_gen_" + num
}

class testbram(val num : Int)(implicit val conf : Configuration) extends Module{
    val io = IO(new Bundle {})
    val bram = Module (new blk_mem_gen(num))
}

class top extends Module{
    val io = IO(new Bundle {})
    implicit val configuration = Configuration()
    val kernel1 = Module (new testbram(0))
    val kernel2 = Module (new testbram(1))
}

case class Configuration(){
    val Data_width = 64
    val Addr_width = 16
}
```
生成的Verilog代码:
```
module testbram(
);
  wire [15:0] bram_addra; // @[top.scala 21:23]
  wire [63:0] bram_dina; // @[top.scala 21:23]
  wire [63:0] bram_douta; // @[top.scala 21:23]
  blk_mem_gen_0 bram ( // @[top.scala 21:23]
    .addra(bram_addra),
    .dina(bram_dina),
    .douta(bram_douta)
  );
  assign bram_addra = 16'h0;
  assign bram_dina = 64'h0;
endmodule
module testbram_1(
);
  wire [15:0] bram_addra; // @[top.scala 21:23]
  wire [63:0] bram_dina; // @[top.scala 21:23]
  wire [63:0] bram_douta; // @[top.scala 21:23]
  blk_mem_gen_1 bram ( // @[top.scala 21:23]
    .addra(bram_addra),
    .dina(bram_dina),
    .douta(bram_douta)
  );
  assign bram_addra = 16'h0;
  assign bram_dina = 64'h0;
endmodule
module top(
  input   clock,
  input   reset
);
  initial begin end
  testbram kernel1 ( // @[top.scala 27:26]
  );
  testbram_1 kernel2 ( // @[top.scala 28:26]
  );
endmodule
```
可以看到两个testbram模块的Verilog代码没有复用
