# regfile
—— 寄存器堆。

BOOM使用统一的物理寄存器堆(Physical Register File, PRF)设计。寄存器堆同时保持提交状态和推测状态。此外，有两个寄存器堆：一个用于整数，另一个用于浮点寄存器值。重命名映射表跟踪与ISA寄存器对应的物理寄存器。

```scala
// Register File (Abstract class and Synthesizable RegFile)
// 一个抽象的寄存器堆类，以及一个可综合的寄存器堆类

package boom.exu
import scala.collection.mutable.ArrayBuffer
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import boom.common._
import boom.util.{BoomCoreStringPrefix}
```



### BOOM的寄存器堆

BOOM用片上SRAM做寄存器堆，包括整数寄存器堆和浮点寄存器堆，整数寄存器堆有6个读端口，3个写端口，而浮点寄存器堆有3个读端口，2个写端口（典型值，可通过参数进行配置）。 寄存器堆中存储的值既包括已经提交的值，也包括处于推测状态下的值。

BOOM复用[berkeley-hardfloat](https://link.zhihu.com/?target=https%3A//github.com/ucb-bar/berkeley-hardfloat)作为FPU，因此所有的物理浮点寄存器均为65位。

BOOM采用的是显式重命名，因此物理寄存器的数量要多于逻辑寄存器的数量，具体数量可以配置。对于默认配置中三发射的`LargeBoom` 来说，整数物理寄存器有100个，浮点物理寄存器有96个。



### RegisterFileReadPortIO 类

寄存器堆读端口的IO。输入地址，输出数据。

```scala
/**
 * IO bundle for a register read port
 *
 * @param addrWidth size of register address in bits
 * @param dataWidth size of register in bits
 */
 class RegisterFileReadPortIO(val addrWidth: Int, val dataWidth: Int)(implicit p: Parameters) extends BoomBundle
 {
    val addr = Input(UInt(addrWidth.W))
    val data = Output(UInt(dataWidth.W))
 }
```

register-read.scala中，RegisterRead类的IO端口中就有以下两项：

```scala
val rf_read_ports = Flipped(Vec(numTotalReadPorts, new RegisterFileReadPortIO(maxPregSz, registerWidth)))
val prf_read_ports = Flipped(Vec(issueWidth, new RegisterFileReadPortIO(log2Ceil(ftqSz), 1)))
```





### RegisterFileWritePort 类

寄存器堆写端口。两个成员，地址和数据。

不在名字后面加IO的原因是：实际的写IO端口是本类再加一个Valid信号。

```scala
/**
 * IO bundle for the register write port
 *
 * @param addrWidth size of register address in bits
 * @param dataWidth size of register in bits
 */
 class RegisterFileWritePort(val addrWidth: Int, val dataWidth: Int)(implicit p: Parameters) extends BoomBundle
 {
    val addr = UInt(addrWidth.W)
    val data = UInt(dataWidth.W)
 }
```



### WritePort 对象

定义一个函数，将执行单元的回复与寄存器堆的写端口IO匹配起来。

输入：一个作为DecoupledIO的执行单元回复、地址宽度、数据宽度和寄存器类型。

输出：wire类型的一个带valid信号的寄存器堆写端口，内部成员已被置为相应的值。

```scala
/**
 * Utility function to turn ExeUnitResps to match the regfile's WritePort I/Os.
 */
 object WritePort
 {
    def apply(enq: DecoupledIO[ExeUnitResp], addrWidth: Int, dataWidth: Int, rtype: UInt)
    (implicit p: Parameters): Valid[RegisterFileWritePort] = {
     val wport = Wire(Valid(new RegisterFileWritePort(addrWidth, dataWidth)))

     wport.valid     := enq.valid && enq.bits.uop.dst_rtype === rtype
     wport.bits.addr := enq.bits.uop.pdst
     wport.bits.data := enq.bits.data
     enq.ready       := true.B
     wport
    }
 }
```



### RegisterFile 抽象类

寄存器堆。

抽象类，只定义了IO端口和输出内部信息的功能。

```scala
/**
 * Register file abstract class
 *
 * @param numRegisters number of registers
 * @param numReadPorts number of read ports
 * @param numWritePorts number of write ports
 * @param registerWidth size of registers in bits
 * @param bypassableArray list of write ports from func units to the read port of the regfile
 */
  abstract class RegisterFile(
    numRegisters: Int,	// （物理）寄存器（组）数量
    numReadPorts: Int,	// 读端口数量
    numWritePorts: Int,	// 写端口数量
    registerWidth: Int,	// 寄存器（组）的大小
    bypassableArray: Seq[Boolean]) 
		// which write ports can be bypassed to the read ports?
		// 旁路，指示哪些端口可以旁路连接到写端口
    (implicit p: Parameters) extends BoomModule
  {
      
    	// IO
    val io = IO(new BoomBundle {
    val read_ports = Vec(numReadPorts, new RegisterFileReadPortIO(maxPregSz, registerWidth))
    val write_ports = Flipped(Vec(numWritePorts, Valid(new RegisterFileWritePort(maxPregSz, registerWidth))))
        // 写端口内没有注明Input或Output，所以需要用Flipped的方式连接
    })

      
      // 输出寄存器堆信息
  private val rf_cost = (numReadPorts + numWritePorts) * (numReadPorts + 2*numWritePorts)
      // 寄存器堆开销 = （读端口+写端口）*（读端口+2*写端口） ？
  private val type_str = if (registerWidth == fLen+1) "Floating Point" else "Integer"
  override def toString: String = BoomCoreStringPrefix(
    "==" + type_str + " Regfile==",
    "Num RF Read Ports     : " + numReadPorts,
    "Num RF Write Ports    : " + numWritePorts,
    "RF Cost (R+W)*(R+2W)  : " + rf_cost,
    "Bypassable Units      : " + bypassableArray)
}
```



### RegisterFileSynthesizable 类

可综合的寄存器堆类。

继承自上面的抽象类，增加了具体的寄存器堆，以及输入、旁路、输出、多写一时警告的逻辑。

```scala
/**
 * A synthesizable model of a Register File. You will likely want to blackbox this for more than modest port counts.
 *
 * @param numRegisters number of registers
 * @param numReadPorts number of read ports
 * @param numWritePorts number of write ports
 * @param registerWidth size of registers in bits
 * @param bypassableArray list of write ports from func units to the read port of the regfile
 */
  class RegisterFileSynthesizable(
   numRegisters: Int,
   numReadPorts: Int,
   numWritePorts: Int,
   registerWidth: Int,
   bypassableArray: Seq[Boolean])
   (implicit p: Parameters)
   extends RegisterFile(numRegisters, numReadPorts, numWritePorts, registerWidth, bypassableArray)
  {
      
      
  // --------------------------------------------------------------
  val regfile = Mem(numRegisters, UInt(registerWidth.W))
  	// 寄存器堆，调用Mem

      
      
  // --------------------------------------------------------------
  // Read ports.

  val read_data = Wire(Vec(numReadPorts, UInt(registerWidth.W)))
      // 读端口，输入引出来的线网

  // Register the read port addresses to give a full cycle to the RegisterRead Stage (if desired).
  // 注册读端口地址，为RegisterRead阶段提供一个完整的周期（如果需要）
  val read_addrs = io.read_ports.map(p => RegNext(p.addr))
      // 一个周期后，read_addrs寄存器中将存放读端口传来的地址

  for (i <- 0 until numReadPorts) {
    read_data(i) := regfile(read_addrs(i))
  }
      // 用地址做索引，将数据写到regfile中对应位置

      
      
  // --------------------------------------------------------------
  // Bypass out of the ALU's write ports.
  // We are assuming we cannot bypass a writer to a reader within the regfile memory
  //  for a write that occurs at the end of cycle S1 and a read that returns data on cycle S1.
  // But since these bypasses are expensive, and not all write ports need to bypass their data,
  //  only perform the w->r bypass on a select number of write ports.
  // ALU写端口的旁路。
  // 我们假设：考虑周期S1，在此周期末尾有一个写操作，而这个周期中要求读的时候，我们不能在寄存器
  //  堆中通过旁路直接将写入者传给读取者。但既然这些旁路很昂贵，而且不是所有读端口都需要旁路，因此我们只将读到写的
  //  旁路应用在一些经过选择的写端口。

  require (bypassableArray.length == io.write_ports.length) // 旁路信息长度要与写端口数量匹配

  if (bypassableArray.reduce(_||_)) {	// 存在需要旁路的写端口时
    val bypassable_wports = ArrayBuffer[Valid[RegisterFileWritePort]]()
    io.write_ports zip bypassableArray map { case (wport, b) => if (b) { bypassable_wports += wport} }
      // 将需要旁路的写端口加到这个序列中

    for (i <- 0 until numReadPorts) {	// 遍历写端口
      val bypass_ens = bypassable_wports.map(x => x.valid && x.bits.addr === read_addrs(i))
        // 在有旁路的写端口序列中，寻找valid并且地址匹配的写端口
    
      val bypass_data = Mux1H(VecInit(bypass_ens), VecInit(bypassable_wports.map(_.bits.data)))
        // 得到需要旁路的数据。这里可以保证ens信号是独热码，因为有重命名。（？）
    
      io.read_ports(i).data := Mux(bypass_ens.reduce(_|_), bypass_data, read_data(i))
    }
  } else {	// 无旁路
    for (i <- 0 until numReadPorts) {
      io.read_ports(i).data := read_data(i)	// 将线网上的数据传给输出
    }
  }

      
      
  // --------------------------------------------------------------
  // Write ports.

  for (wport <- io.write_ports) {	// 遍历写端口
    when (wport.valid) {
      regfile(wport.bits.addr) := wport.bits.data	//	valid时，写数据
    }
  }

  // ensure there is only 1 writer per register (unless to preg0)
  // 保证每个寄存器只有一个写入者（对第0个物理寄存器无效）
  if (numWritePorts > 1) {
    for (i <- 0 until (numWritePorts - 1)) {
      for (j <- (i + 1) until numWritePorts) {
        assert(!io.write_ports(i).valid ||
               !io.write_ports(j).valid ||
               (io.write_ports(i).bits.addr =/= io.write_ports(j).bits.addr) ||
               (io.write_ports(i).bits.addr === 0.U), // note: you only have to check one here
          "[regfile] too many writers a register")
          // 第i和第j个写端口都valid，写地址相同且不为0时，发出警告
      }
    }
  }
}
```