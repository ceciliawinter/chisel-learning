# chisel-learning

- [chisel-learning](#chisel-learning)
  - [报错解决](#报错解决)
    - [switch 报错](#switch报错)
    - [数据类型错误](#数据类型错误)
    - [模块间传递数据位宽参数](#模块间传递数据位宽参数)
    - [Bool 类型赋值](#bool类型赋值)
    - [无隐式时钟域复位信号报错](#无隐式时钟域复位信号报错)
    - [在 when 中对 io 端口数据进行修改](#在when中对io端口数据进行修改)
    - [scala 版本导致的错误](#scala版本导致的错误)
    - [reset 类型错误](#reset类型错误)
    - [引用的包包含同名字段](#引用的包包含同名字段)
    - [io 中使用数组](#io中使用数组)
  - [学习问题](#学习问题)

    - [溢出](#溢出)
    - [修改 UInt 某一位](#修改uint某一位)
    - [状态无法保持](#状态无法保持)
    - [端口错误优化](#端口错误优化)
    - [memory 载入数据](#memory载入数据)
    - [使用 Analog](#使用analog)
    - [模块名称参数化](#模块名称参数化)
    - [使用 Queue](#使用queue)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

记录 chisel 学习过程中遇到的问题

## 报错解决

### switch 报错

部分报错内容：

```
import chisel3._

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

使用参数定义数据长度（注意，val对应的值是固定的，不可更改的）

```
val addrWidth = 4
val addr = Wire(UInt(addrWidth.W))
```

### 模块间传递数据位宽参数

错误代码 1：

在此处，signal 被定义为 Int 类型

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

错误代码 2：

在此处，signal 定义的位宽大小参数 sigsize 数据类型为 UInt

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

注意，如果数据类型是 Int 类型，Int 类型不具有参数

数据定义为 UInt 类型，可以通过参数来定义数据的大小，该参数必须为 Int 类型，不可为 UInt 类型

正确示例：

```
class ResIO(val sigsize : Int) extends Bundle{
   val signal = Input(UInt(sigsize.W))
   val res = Output(Bool())
}
```

### Bool 类型赋值

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

stackoverflow 上查找到解决方案：https://stackoverflow.com/questions/41658288/comparing-two-bits-type-values-in-chisel-3

Boolean 为 scala 数据类型，在 chisel 中所需要的是 Bool 类型

修改代码，将类型转化为 Bool

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

之前使用 BlackBox 用 verilog 代码实现部分功能，使用 chisel 语言重新编写时，未改变为 Module,故产生不存在隐式时钟域 reset 信号报错

### 在 when 中对 io 端口数据进行修改

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

在之前的chisel版本中会出现报错信息：

```
[error] (run-main-0) firrtl.FIRRTLException: Internal Error! Please file an issue at https://github.com/ucb-bar/firrtl/issues
[error] firrtl.FIRRTLException: Internal Error! Please file an issue at https://github.com/ucb-bar/firrtl/issues
[error]         at firrtl.Utils$.error(Utils.scala:396)
......
```

注意：scala 是一个脚本型语言，他会从上至下进行编译，所以如果你在后面赋值，他会直接覆盖掉之前的全部逻辑。
```
  val res = Reg(Bool())
  io.res := false.B
  when(signal_pre =/= io.signal){
      io.res := true.B
  }
```

### scala 版本导致的错误

在尝试使用 scala 命令行时，使用 scala 2.11.x 试，命令行仅显示输出，不显示输入，因此将 scala 版本更换为 2.12.x

https://stackoverflow.com/questions/49788781/ubuntu-scala-repl-commands-not-typed-on-console

更换了 scala 版本后，原本正常运行的 chisel3 程序不能正常运行，产生报错 \*\*\* is not a member of chisel3.Bundle

https://stackoverflow.com/questions/58365679/value-is-not-a-member-of-chisel3-bundle

scala 可以安装多个版本，使用 scala2.12.x 的命令行，在 chisel 工程中，build.sbt 中选择 2.11.x 即可
如今在最新的稳定版 6.6 中，可以使用 scala2.13.12

### reset 类型错误

报错代码：

```
dram.io.clk_and_rst.sys_rst := reset
```

dram.io.clk_and_rst.sys_rs 定义：

```
val sys_rst = Input(Bool())
```

报错信息

```
[error] chisel3.internal.ChiselException: Connection between sink (Bool(IO clk_and_rst_sys_rst in my_mig_ddr2)) and source (Reset(IO in unelaborated dramtest)) failed @: Sink (Bool(IO clk_and_rst_sys_rst in my_mig_ddr2)) and Source (Reset(IO in unelaborated dramtest)) have different types.
[error]         ...
```

再将 reset 信号接出时，对应的类型应为 Reset(),但是用 withClockAndReset 多时钟域时，对应的 reset 信号的数据类型应为 Bool，修改后代码：

```
val sys_rst = Input(Reset())
val ui_clk_sync_rst = Output(Bool())
...

dram.io.clk_and_rst.sys_rst := reset
withClockAndReset(dram.io.clk_and_rst.ui_clk, dram.io.clk_and_rst.ui_clk_sync_rst)
...
```

### 引用的包包含同名字段

在使用 withClockAndReset 时，根据查阅到的资料为

> 注意，在编写代码时不能写成“import chisel3.core._”，这会扰乱“import chisel3._”的导入内容。正确做法是用“import chisel3.experimental.\_”导入 experimental 对象，它里面用同名字段引用了单例对象 chisel3.core.withClockAndReset，这样就不需要再导入 core 包。

但是实际使用中依旧会出现报错

```
[error] reference to withClockAndReset is ambiguous;
[error] it is imported twice in the same scope by
[error] import chisel3.experimental._
[error] and import chisel3._
[error]     withClockAndReset(dram.io.ui_clk, dram.io.ui_clk_sync_rst) {
```

TODO 解决方案

因为端口连接需要 attach，必须引用 chisel3.experimental._ ，尝试在使用 attach 前再引用 chisel3.experimental._ , 不在文件开头引用，可行

### io 中使用数组

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

修改端口定义方式，使用 Chisel 的特性 Vec 来定义数组，不使用 Scala 类型

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

### 修改 UInt 某一位

[chisel3 wiki](https://www.chisel-lang.org/docs/cookbooks/cookbook#how-do-i-do-subword-assignment-assign-to-some-bits-in-a-uint)

chisel3 不支持修改子词(subword)，可以使用 Bundles 和 Vecs 来表达

将 UInt 转为 Vec

```
import chisel3._

class Foo extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(10.W))
    val bit = Input(Bool())
    val out = Output(UInt(10.W))
  })
  val bools = VecInit(io.in.asBools)
  bools(0) := io.bit
  io.out := bools.asUInt
}
```


### 状态无法保持

代码中包含数据 visited_map,visited_map_bitmap
bitmap 用于对每一位的修改

```
val visited_map = RegInit(0.U(32.W))
val visited_map_bitmap = VecInit(visited_map.asBools)
```

修改前

```
dinbReg := visited_map_bitmap.asUInt
```

将数据连接到 bram 上时，直接将 visited_map_bitmap.asUInt 连接到 bram 的 dinb 接口，visited_map_bitmap 值在下一次状态跳变时没有保持，修改为:

```
visited_map := visited_map_bitmap.asUInt
dinbReg := visited_map
```

查看波形图，visited_map_bitmap 状态可以持续保持

### 端口错误优化

在生成 verilog 代码时，单独综合某一模块，所有的端口与 chisel 代码保持一致，但是在综合 Top 模块时，调用的某些模块端口被优化，优化的部分内容缺失后无法实现设置的功能，可以取消相关优化
[chisel dontTouch](https://www.chisel-lang.org/api/latest/chisel3/dontTouch$.html)

```
import chisel3.dontTouch
dontTouch(io)
val a = dontTouch(...)
```

### memory 载入数据

需要初始化 memory 中的数据可以使用 loadMemoryFromFile 来实现
[chisel loadMemoryFromFile](https://www.chisel-lang.org/api/latest/chisel3/util/experimental/loadMemoryFromFile$.html)

```
    val ram = Mem(65535, UInt(64.W))
    loadMemoryFromFile(ram, "./mem.txt")
```

可以使用 2 进制或格式进制数据，注意文件格式如下：

```
0
1
2
```

### 使用 Analog

使用 Analog 声明位宽，以实现在 blackbox 中使用 verilog 的 inout 端口

需要 chisel 版本在 3.1 以上且引用 import chisel3.experimental.\_

```
import chisel3.experimental._
···
val ddr2_dq = Analog(8.W)
```

### 模块名称参数化

Chisel2 内有 setName 功能，但是 Chisel3 没有，使用 desiredName 来实现
[chisel3 wiki](https://github.com/freechipsproject/chisel3/wiki/Cookbook#how-can-i-dynamically-setparametrize-the-name-of-a-module)

在 chisel3 wiki 中讲解了使用 desiredName 来参数化模块名称，但是该名称仍是固定的

尝试使用与 setName 类似的形式实现，用“+”连接参数

实现代码内部有两个 kernel，两个 kernel 需要使用个不同的 bram ip 核（即 blk_mem_gen 模块）

在 blk_mem_gen 模块内部使用 desiredName 使不同的 kernel 内部的 bram 具有不同的名称

主要传进去的参数 num 为 Int 类型，不要使用 UInt

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

生成的 Verilog 代码:

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

可以看到两个 testbram 模块的 Verilog 代码没有复用

### 使用 Queue

**接收数据使用 Queue 缓存**

io：

```
val a = Flipped(Decoupled(UInt(4.W)))
```

使用 Queue 存储

```
val qa = Queue(io.a,32)
qa.ready := false.B // equivalent to qa.nodeq()
when(qa.valid){
    qa.ready = true.B
    data := qa.bits // or data := qa.deq()
}
```

**发送数据使用 Queue 缓存**

io：

```
val b = Flipped(Decoupled(UInt(4.W)))
```

使用 Queue 存储

```
val qb = Module(Queue(io.b,32))
io.b <> qb.io.deq
qb.io.enq.valid := false.B
qb.io.enq.bits := DontCare
when(qb.io.enq.ready){
    qb.io.enq.valid "= true.B
    qb.io.enq.bits := data
}
```
