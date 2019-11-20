# chisel-learning
记录chisel学习过程中遇到的问题
## switch报错
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
