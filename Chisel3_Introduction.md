# Chisel3简介

[Chisel/FIRRTL: Introduction (chisel-lang.org)](https://www.chisel-lang.org/chisel3/docs/introduction.html)

对Chisel3/FIRRTL介绍文档中重点内容的翻译。



### Chisel的数据类型

#### Bits、SInt/UInt和Bool

`Bits` 单纯的比特位的集合。

`SInt/UInt` 有符号和无符号整数，是定点数（fixed-point numbers）的一个子集。

`Bool`  布尔类型值。

> 需要注意，这些和scala内建的`Int`和`Boolen`类型是不同的。



数据类型示例如下：

```scala
1.U       	// decimal 1-bit lit from Scala Int.
"ha".U    	// hexadecimal 4-bit lit from string.
"o12".U   	// octal 4-bit lit from string.
"b1010".U 	// binary 4-bit lit from string.

"h_dead_beef".U   // 32-bit lit of type UInt.   "_"可用于分割，增加可读性。

5.S    // signed decimal 4-bit lit from Scala Int.
-8.S   // negative decimal 4-bit lit from Scala Int.
5.U    // unsigned decimal 3-bit lit from Scala Int.

8.U(4.W) 		// 4-bit unsigned decimal, value 8.
-152.S(32.W) 	// 32-bit signed decimal, value -152.

true.B 		// Bool lits from Scala lits.
false.B
```



默认情况下，Chisel编译器会根据量的值选择最小的位数，同时加上符号位。但位数也可以指定，如下：

（注意，`.W`将Scala的Int类型转换为Chisel的Width类型）

```scala
"ha".asUInt(8.W)     // hexadecimal 8-bit lit of type UInt
"o12".asUInt(6.W)    // octal 6-bit lit of type UInt
"b1010".asUInt(12.W) // binary 12-bit lit of type UInt

5.asSInt(7.W) // signed decimal 7-bit lit of type SInt
5.asUInt(8.W) // unsigned decimal 8-bit lit of type UInt
```

将字面量作为UInt储存时，将进行零拓展；作为SInt时将进行符号拓展。若给定的位数不足以储存字面量，则将生成Chisel错误。



这些数据类型也可以相互强制转换：

```scala
val sint = 3.S(4.W)             // 4-bit SInt
val uint = sint.asUInt          // cast SInt to UInt
uint.asSInt                     // cast UInt to SInt

val bool: Bool = false.B        // always-low wire
val clock = bool.asClock        // always-low clock
clock.asUInt                    // convert clock to UInt (width 1)
clock.asUInt.asBool             // convert clock to Bool (Chisel 3.2+)
clock.asUInt.toBool             // convert clock to Bool (Chisel 3.0 and 3.1 only)
```

注意：在Chisel数据类型的相互转换中，`asUInt`和`asSInt`不可加参数指定位数。Chisel会在连接对象时根据需要增减位数。



#### Bundle和Vecs

`Bundle` 用于整合不同的值，并使这些值有同样的命名空间。类似于其他语言的`Structs`。

注意，根据Scala管理，类的名字一般用大驼峰命名法。

```scala
import chisel3._

class MyFloat extends Bundle {	// 定义一个Bundle
  val sign        = Bool()
  val exponent    = UInt(8.W)
  val significand = UInt(23.W)
}

class ModuleWithFloatWire extends RawModule {	// 定义一个RawModule，其中包含这个Bundle
  val x  = Wire(new MyFloat)
  val xs = x.sign
}
```


`Input`和`Output`可以指定`Bundle`中的输入和输出项，从而将这个类变成一个端口(Ports)。

```scala
class ABBundle extends Bundle {
  val a = Input(Bool())
  val b = Output(Bool())
}
```


`Flipped`函数可以递归地将`Bundle`中的输入输出反转，以实现端口的连接（常见于ready和valid这些握手信号）。

```scala
class MyFlippedModule extends RawModule {

  val normalBundle = IO(new ABBundle)
  normalBundle.b := normalBundle.a

  val flippedBundle = IO(Flipped(new ABBundle))
  flippedBundle.a := flippedBundle.b			// 此时a是输入，b是输出
}
```



这将生成以下的Verilog代码：

```verilog
module MyFlippedModule(
  input   normalBundle_a,
  output  normalBundle_b,
  output  flippedBundle_a,
  input   flippedBundle_b
);
  assign normalBundle_b = normalBundle_a; // @[bundles-and-vecs.md 61:18]
  assign flippedBundle_a = flippedBundle_b; // @[bundles-and-vecs.md 66:19]
endmodule
```



`Vecs` 用于有索引的整合起来的值。（还有`MixedVecs`，见？）

```scala
class ModuleWithVec extends RawModule {
  val myVec = Wire(Vec(5, SInt(23.W)))	// 5个23位SInt组成的向量线网
  val reg3 = myVec(3)					// 连接到向量的一个元素
}
```



### 组合电路与线网

‎简单的表达式可以构建树状电路，但要构建任意定向环形图 （DAG） 形状的电路，我们需要描述扇出。在 Chisel 中，我们通过命名具有次表达式的线网来做到这一点，而且可以在随后的表达式中多次引用该线网。我们通过声明变量来命名Chisel的线网。例如：

```scala
val sel = a | b
val out = (sel & in1) | (~sel & in0)
```

关键词`sel`是Scala的部分，被用于命名不会改变的值。这里用于命名Chisel线网`sel`，保持住或运算的结果，这样就可以在后面的表达式中多次使用。



‎Chisel还支持线网作为硬件节点，可以分配值或连接其他节点。‎

```scala
val myNode = Wire(UInt(8.W))
when (input > 128.U) {
  myNode := 255.U
} .elsewhen (input > 64.U) {
  myNode := 1.U
} .otherwise {
  myNode := 0.U
}
```



### 操作符

| 操作符                                      | 解释                                                      |
| :------------------------------------------ | :-------------------------------------------------------- |
| **位运算**                                  | **用于:** SInt, UInt, Bool                                |
| `val invertedX = ~x`                        | Bitwise NOT                                               |
| `val hiBits = x & "h_ffff_0000".U`          | Bitwise AND                                               |
| `val flagsOut = flagsIn \| overflow`        | Bitwise OR                                                |
| `val flagsOut = flagsIn ^ toggle`           | Bitwise XOR                                               |
| **位归约**                                  | **用于:** SInt and UInt. 返回Bool.                        |
| `val allSet = x.andR`                       | AND reduction                                             |
| `val anySet = x.orR`                        | OR reduction                                              |
| `val parity = x.xorR`                       | XOR reduction                                             |
| **比较是否相等**                            | **用于:** SInt, UInt, and Bool. 返回Bool.                 |
| `val equ = x === y`                         | Equality                                                  |
| `val neq = x =/= y`                         | Inequality                                                |
| **移位**                                    | **用于:** SInt and UInt                                   |
| `val twoToTheX = 1.S << x`                  | Logical shift left                                        |
| `val hiBits = x >> 16.U`                    | Right shift **(logical on UInt and arithmetic on SInt)**. |
| **位域操作**                                | **用于:** SInt, UInt, and Bool.                           |
| `val xLSB = x(0)`                           | Extract single bit, LSB has index 0.                      |
| `val xTopNibble = x(15, 12)`                | Extract bit field from end to start bit position.         |
| `val usDebt = Fill(3, "hA".U)`              | Replicate a bit string multiple times.                    |
| `val float = Cat(sign, exponent, mantissa)` | Concatenates bit fields, with first argument on left.     |
| **逻辑操作符**                              | **用于:** Bool                                            |
| `val sleep = !busy`                         | Logical NOT                                               |
| `val hit = tagMatch && valid`               | Logical AND                                               |
| `val stall = src1busy || src2busy`          | Logical OR                                                |
| `val out = Mux(sel, inTrue, inFalse)`       | Two-input mux where sel is a Bool                         |
| **算术操作符**                              | **用于这些类型的数:** SInt and UInt.                      |
| `val sum = a + b` *或* `val sum = a +% b`   | Addition (without width expansion)                        |
| `val sum = a +& b`                          | Addition **(with width expansion)**                       |
| `val diff = a - b` *或* `val diff = a -% b` | Subtraction (without width expansion)                     |
| `val diff = a -& b`                         | Subtraction **(with width expansion)**                    |
| `val prod = a * b`                          | Multiplication                                            |
| `val div = a / b`                           | Division                                                  |
| `val mod = a % b`                           | Modulus                                                   |
| **算术比较符**                              | **用于这些类型的数:** SInt and UInt. Returns Bool.        |
| `val gt = a > b`                            | Greater than                                              |
| `val gte = a >= b`                          | Greater than or equal                                     |
| `val lt = a < b`                            | Less than                                                 |
| `val lte = a <= b`                          | Less than or equal                                        |

> 为了不和Scala的操作符混淆，使用了`===`和`=/=`。

Chisel操作符并不直接作为Chisel语言的一部分定义。实际上，它是由电路evaluation的顺序决定的，因此自然地遵守Scala的操作符优先级。可以用括号改变优先级。

> 在操作符优先级上，Chisel和Scala是相似的，但和Java和C不同。Verilog和C相同，但VHDL不同。Verilog有逻辑运算符的优先级，而VHDL是从左向右运算。



### 宽度推理

为了降低设计压力，Chisel提供自动宽度推理，由FIRRTL编译器提供。

For all circuit components declared with unspecified widths, the FIRRTL compiler will infer the minimum possible width that maintains the legality of all its incoming connections. Implicit here is that inference is done in a right to left fashion in the sense of an assignment statement in chisel, i.e. from the left hand side from the right hand side. If a component has no incoming connections, and the width is unspecified, then an error is thrown to indicate that the width could not be inferred.

For module input ports with unspecified widths, the inferred width is the minimum possible width that maintains the legality of all incoming connections to all instantiations of the module. The width of a ground-typed multiplexor expression is the maximum of its two corresponding input widths. For multiplexing aggregate-typed expressions, the resulting widths of each leaf subelement is the maximum of its corresponding two input leaf subelement widths. The width of a conditionally valid expression is the width of its input expression. For the full formal description see the [Firrtl Spec](https://github.com/freechipsproject/firrtl/blob/master/spec/spec.pdf).

Hardware operators have output widths as defined by the following set of rules:

| 操作                          | 位宽                         |
| :---------------------------- | :--------------------------- |
| `z = x + y` *or* `z = x +% y` | `w(z) = max(w(x), w(y))`     |
| `z = x +& y`                  | `w(z) = max(w(x), w(y)) + 1` |
| `z = x - y` *or* `z = x -% y` | `w(z) = max(w(x), w(y))`     |
| `z = x -& y`                  | `w(z) = max(w(x), w(y)) + 1` |
| `z = x & y`                   | `w(z) = max(w(x), w(y))`     |
| `z = Mux(c, x, y)`            | `w(z) = max(w(x), w(y))`     |
| `z = w * y`                   | `w(z) = w(x) + w(y)`         |
| `z = x << n`                  | `w(z) = w(x) + maxNum(n)`    |
| `z = x >> n`                  | `w(z) = w(x) - minNum(n)`    |
| `z = Cat(x, y)`               | `w(z) = w(x) + w(y)`         |
| `z = Fill(n, x)`              | `w(z) = w(x) * maxNum(n)`    |

> where for instance `w(z)` is the bit width of wire `z`, and the `&` rule applies to all bitwise logical operations.

Given a path of connections that begins with an unspecified width element (most commonly a top-level input), then the compiler will throw an exception indicating a certain width was uninferrable.

A common “gotcha” comes from truncating addition and subtraction with the operators `+` and `-`. Users who want the result to maintain the full, expanded precision of the addition or subtraction should use the expanding operators `+&` and `-&`.

> The default truncating operation comes from Chisel’s history as a microprocessor design language.



### 函数抽象

用函数包裹一个简单的组合逻辑：

```scala
def clb(a: UInt, b: UInt, c: UInt, d: UInt): UInt =
  (a & b) | (~c & d)
```

在另一个电路中可以这样使用这个模块： `scala mdoc:silent val out = clb(a,b,c,d)`



### 模块

Chisel的模块（modules）和Verilog的很相似。Chisel中，一个用户定义的模块应该是一个这样的类：

* 从`Module`继承，
* 包含一个包裹在模块中的`IO()`方法，并且一个名为`io`的端口中，
* 在它的构造器中将子电路连接起来。

一个简单的二选一MUX例子：

```scala
import chisel3._

class Mux2IO extends Bundle {	// IO从Bundle类继承
  val sel = Input(UInt(1.W))
  val in0 = Input(UInt(1.W))
  val in1 = Input(UInt(1.W))
  val out = Output(UInt(1.W))
}

class Mux2 extends Module {
  val io = IO(new Mux2IO)
  io.out := (io.sel & io.in1) | (~io.sel & io.in0)	// “:=”用于将右边的输入赋给左边的输出
}
```



模块也可以具有层次结构，如用三个二选一MUX构造四选一MUX：

```scala
class Mux4IO extends Bundle {
  val in0 = Input(UInt(1.W))
  val in1 = Input(UInt(1.W))
  val in2 = Input(UInt(1.W))
  val in3 = Input(UInt(1.W))
  val sel = Input(UInt(2.W))
  val out = Output(UInt(1.W))

class Mux4 extends Module {
  val io = IO(new Mux4IO)

  val m0 = Module(new Mux2)	// 用Module方法和Scala的new关键词构造一个新的模块
  m0.io.sel := io.sel(0)
  m0.io.in0 := io.in0
  m0.io.in1 := io.in1

  val m1 = Module(new Mux2)
  m1.io.sel := io.sel(0)
  m1.io.in0 := io.in2
  m1.io.in1 := io.in3

  val m3 = Module(new Mux2)
  m3.io.sel := io.sel(1)
  m3.io.in0 := m0.io.out
  m3.io.in1 := m1.io.out

  io.out := m3.io.out
}
```

注意：Chisel的模块有隐性的时钟信号`clock`和重置信号`reset`。



为了实现其他的特性，Chisel还实现了`MultiIOModule`和`RawModule`。`MultiIOModule`允许多个IO端口，而且不强制要求储存在名为io的val中，`RawModule`则在这个基础上去掉了隐性的时钟信号`clock`和重置信号`reset`（然后就可以手动指定了）。使用例如下：

```scala
import chisel3.{RawModule, withClockAndReset}	// import需要用的新玩意

class Foo extends Module {
  val io = IO(new Bundle{
    val a = Input(Bool())
    val b = Output(Bool())
  })
  io.b := !io.a
}

class FooWrapper extends RawModule {
  val a_i  = IO(Input(Bool()))
  val b_o  = IO(Output(Bool()))
  val clk  = IO(Input(Clock()))
  val rstn = IO(Input(Bool()))

  val foo = withClockAndReset(clk, !rstn){ Module(new Foo) }
    // 用withClockAndReset手动指定时钟和重置信号，使得现在这个模块是低电平重置

  foo.io.a := a_i
  b_o := foo.io.b
}
```



### 时序电路

最简单的例子：**边沿触发器**。注意：时钟和重置信号是全局的；`reg`的类型会根据`in`的类型自动设置；无初始值，需要重置。

```scala
val reg = RegNext(in)
```

利用这个简单的例子可以构造一个**上升沿检测器**：

```scala
def risingedge(x: Bool) = x && !RegNext(x)
```

利用`RegInit`可以构造带初始值的寄存器，因此可以如下实现一个**计数器**：

```scala
def counter(max: UInt) = {	
  val x = RegInit(0.asUInt(max.getWidth.W))	// 初始值为0，宽度利用.getWidth方法获得
  x := Mux(x === max, 0.U, x + 1.U)
  x		// 计数范围0~max，可认为上面Mux结果是x寄存器的信号输入，需要时钟信号改变后才更新寄存器中的值
}
```

利用上面的计数器又可以实现**脉冲生成器**：

```scala
def pulse(n: UInt) = counter(n - 1.U) === 0.U	// 每n个周期输出一次高电平
```

再利用脉冲生成器实现**方波生成器**：

```scala
def toggle(p: Bool) = {	
  val x = RegInit(false.B)
  x := Mux(p, !x, x)	// 每次输入1时，翻转输出
  x
}

def squareWave(period: UInt) = toggle(pulse(period >> 1))	// 每period/2个周期翻转一次信号
```



### 存储单元

#### ROM

#### 可读写的存储单元

##### 同步

###### 读写端口分开

###### 读写共用端口

##### 异步

#### 掩码



#### 初始化



### 端口的连接

#### 向量

Bundle设置Input/Output、Flipped方法见开头部分。

#### 批量连接

#### 标准ready-valid接口

ReadyValidIO / Decoupled





