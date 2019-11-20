# chisel-learning
记录chisel学习过程中遇到的问题
- **switch报错**

部分报错内容：
```
[error] /home/cecilia/risc5/chisel-examples/hello-world/src/main/scala/Hello.scala:28:3: not found: value switch
[error]   switch(io.num){
[error]   ^
[error] /home/cecilia/risc5/chisel-examples/hello-world/src/main/scala/Hello.scala:30:5: not found: value is
[error]     is("b0000".U) {io.seg := "b11000000".U}
[error]     ^
```
错误原因： 未引用库文件，需要添加
```
import chisel3.util._
```
- **数据类型错误**

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
