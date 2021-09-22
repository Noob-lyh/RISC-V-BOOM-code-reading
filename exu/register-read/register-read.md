# register-read
—— 读寄存器。

工作流程：从指令发射（ISS）阶段接受valid信号和微指令uop信息，以及读寄存器的地址（ISS阶段末）；在读寄存器（RDD）阶段的开始通过func-unit-decode中定义的译码器得到功能单元译码、考虑分支后的微指令uop；RDD阶段，从连接的寄存器堆读取数据；在RDD阶段末，用从ALU过来的旁路信息更新数据，最终传给执行（EXE）阶段。

当前实现的寄存器堆静态地为所有已发射指令提供需要的所有寄存器读取端口。例如，如果发射端口#0对应于一个整数ALU，发射端口#1对应于内存单元，那么前两个寄存器读取端口将静态地为ALU服务，下两个寄存器读取端口将为内存单元服务，总共四个读取端口。

未来的设计可以通过提供更少的寄存器读取端口和使用动态调度来仲裁(arbitrate)这些端口来提高区域效率。这尤其有用，因为大多数指令只需要一个操作数。然而，它确实增加了设计的额外复杂性，这通常表现为需要额外的流水线级来仲裁或者检测结构冲突。它还需要能够终止已发射的微指令，并在以后的周期中从发射队列重新发射它们。

```scala
package boom.exu
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import boom.common._
import boom.util._
```



### RegisterRead 类

处理乱序后端接口的寄存器读取和旁路网络，包括入列端的发射窗口和出列端的执行流水线。

```scala
/**
 * Handle the register read and bypass network for the OoO backend
 * interfaces with the issue window on the enqueue side, and the execution
 * pipelines on the dequeue side.
 *
 * @param issueWidth total issue width from all issue queues
 * @param supportedUnitsArray seq of SupportedFuncUnits classes indicating what the functional units do
 * @param numTotalReadPorts number of read ports
 * @param numReadPortsArray execution units read port sequence
 * @param numTotalBypassPorts number of bypass ports out of the execution units
 * @param registerWidth size of register in bits
 */
  class RegisterRead(
    issueWidth: Int,	// 所有发射队列的总发射宽度
    supportedUnitsArray: Seq[SupportedFuncUnits],	// 支持的功能单元类型信息
    numTotalReadPorts: Int,
    numReadPortsArray: Seq[Int],	// 执行单元的读端口序列
                        // each exe_unit must tell us how many max
                        // operands it can accept (the sum should equal
                        // numTotalReadPorts)
      					// 每个执行单元必须告知它可以接受多少个操作数（操作数总个数应该和读端口数量相同）
    numTotalBypassPorts: Int,	// 有旁路的端口数量
    numTotalPredBypassPorts: Int,
    registerWidth: Int 
  )(implicit p: Parameters) extends BoomModule
{
    
    //-----------------------------------------------------------
    // IO端口
    
    val io = IO(new Bundle {
    
    // issued micro-ops   发射的valid信号和微指令
    val iss_valids = Input(Vec(issueWidth, Bool()))
    val iss_uops   = Input(Vec(issueWidth, new MicroOp()))

    // interface with register file's read ports  
    // 与寄存器堆的写端口交互。应连接到寄存器堆。
    val rf_read_ports = Flipped(Vec(numTotalReadPorts, new RegisterFileReadPortIO(maxPregSz, registerWidth)))
    val prf_read_ports = Flipped(Vec(issueWidth, new RegisterFileReadPortIO(log2Ceil(ftqSz), 1)))
        
	// 旁路端口和预测旁路
    val bypass = Input(Vec(numTotalBypassPorts, Valid(new ExeUnitResp(registerWidth))))
    val pred_bypass = Input(Vec(numTotalPredBypassPorts, Valid(new ExeUnitResp(1))))

    // send micro-ops to the execution pipelines 将微指令发送到执行流水线
    val exe_reqs = Vec(issueWidth, (new DecoupledIO(new FuncUnitReq(registerWidth))))

    // 清除信号和分支更新信息
    val kill   = Input(Bool())
    val brupdate = Input(new BrUpdateInfo())
    })

    
   //-----------------------------------------------------------
   // 成员 
    
    // 读寄存器的valid信号和微指令，用于连接Decoder的输出
  val rrd_valids       = Wire(Vec(issueWidth, Bool()))
  val rrd_uops         = Wire(Vec(issueWidth, new MicroOp()))
	// 用于临时保存要发送到执行阶段的微指令（及valid信号等）
  val exe_reg_valids   = RegInit(VecInit(Seq.fill(issueWidth) { false.B }))
  val exe_reg_uops     = Reg(Vec(issueWidth, new MicroOp()))
  val exe_reg_rs1_data = Reg(Vec(issueWidth, Bits(registerWidth.W)))
  val exe_reg_rs2_data = Reg(Vec(issueWidth, Bits(registerWidth.W)))
  val exe_reg_rs3_data = Reg(Vec(issueWidth, Bits(registerWidth.W)))
  val exe_reg_pred_data = Reg(Vec(issueWidth, Bool()))

    
    
  //-------------------------------------------------------------
  // hook up inputs	连接输出

  for (w <- 0 until issueWidth) { // 遍历每个发射窗口
    val rrd_decode_unit = Module(new RegisterReadDecode(supportedUnitsArray(w)))
      // 定义一个读寄存器解码器，使用的功能单元与w有关，似乎功能单元与发射窗口序号是固定的？
      // RegisterReadDecode类的定义在func-unit-decode.scala中
    rrd_decode_unit.io.iss_valid := io.iss_valids(w)
    rrd_decode_unit.io.iss_uop   := io.iss_uops(w)
      // 向解码器输入valid和微指令信息

    rrd_valids(w) := RegNext(rrd_decode_unit.io.rrd_valid &&
                !IsKilledByBranch(io.brupdate, rrd_decode_unit.io.rrd_uop))
    rrd_uops(w)   := RegNext(GetNewUopAndBrMask(rrd_decode_unit.io.rrd_uop, io.brupdate))
      // 下个周期时，保存解码器输出，并且经过分支信息修正后的valid和微指令信息
  }

    
    
  //-------------------------------------------------------------
  // read ports 读端口

  require (numTotalReadPorts == numReadPortsArray.reduce(_+_)) // 检查读端口总数是否正确

  val rrd_rs1_data   = Wire(Vec(issueWidth, Bits(registerWidth.W)))
  val rrd_rs2_data   = Wire(Vec(issueWidth, Bits(registerWidth.W)))
  val rrd_rs3_data   = Wire(Vec(issueWidth, Bits(registerWidth.W)))
  val rrd_pred_data  = Wire(Vec(issueWidth, Bool()))	// pred为预测？与sfb优化有关
  rrd_rs1_data := DontCare
  rrd_rs2_data := DontCare
  rrd_rs3_data := DontCare
  rrd_pred_data := DontCare

  io.prf_read_ports := DontCare

  var idx = 0 // index into flattened read_ports array （索引到平坦的读端口序列）
    // 变量，类似偏移量的功能，使得下面的for循环能正确地遍历所有读端口
  
  for (w <- 0 until issueWidth) {	// 遍历每个发射端口
    val numReadPorts = numReadPortsArray(w)	// 此发射端口需要的读端口数量

    // NOTE:
    // rrdLatency==1, we need to send read address at end of ISS stage,
    //    in order to get read data back at end of RRD stage.
    // 注意：读寄存器延迟为1，因此为了在读寄存器阶段末得到读的数据，需要在发射阶段末发送读的地址
    
    val rs1_addr = io.iss_uops(w).prs1
    val rs2_addr = io.iss_uops(w).prs2
    val rs3_addr = io.iss_uops(w).prs3
    val pred_addr = io.iss_uops(w).ppred
    
    if (numReadPorts > 0) io.rf_read_ports(idx+0).addr := rs1_addr
    if (numReadPorts > 1) io.rf_read_ports(idx+1).addr := rs2_addr
    if (numReadPorts > 2) io.rf_read_ports(idx+2).addr := rs3_addr
    if (enableSFBOpt) io.prf_read_ports(w).addr := pred_addr
    
    if (numReadPorts > 0) rrd_rs1_data(w) := Mux(RegNext(rs1_addr === 0.U), 0.U, io.rf_read_ports(idx+0).data)
    if (numReadPorts > 1) rrd_rs2_data(w) := Mux(RegNext(rs2_addr === 0.U), 0.U, io.rf_read_ports(idx+1).data)
    if (numReadPorts > 2) rrd_rs3_data(w) := Mux(RegNext(rs3_addr === 0.U), 0.U, io.rf_read_ports(idx+2).data)
    if (enableSFBOpt) rrd_pred_data(w) := Mux(RegNext(io.iss_uops(w).is_sfb_shadow), io.prf_read_ports(w).data, false.B)
      // 下个周期时，得到数据，存入rrd_xxx_data的第w位
    
    val rrd_kill = io.kill || IsKilledByBranch(io.brupdate, rrd_uops(w)) // 清除信号
    
    exe_reg_valids(w) := Mux(rrd_kill, false.B, rrd_valids(w))
    // TODO use only the valids signal, don't require us to set nullUop
    // 还需要将它变成只使用valid信号，这样就不用置为空指令。
    exe_reg_uops(w)   := Mux(rrd_kill, NullMicroOp, rrd_uops(w))
    
    exe_reg_uops(w).br_mask := GetNewBrMask(io.brupdate, rrd_uops(w))
    
    idx += numReadPorts // 修改偏移量
  }

    
    
  //-------------------------------------------------------------
  //-------------------------------------------------------------
  // BYPASS MUXES -----------------------------------------------
  // performed at the end of the register read stage

  // NOTES: this code is fairly hard-coded. Sorry.
  // ASSUMPTIONS:
  //    - rs3 is used for FPU ops which are NOT bypassed (so don't check
  //       them!).
  //    - only bypass integer registers.
    
  // 旁路的选择器，在读寄存器阶段末起作用。
  // 这段代码很hard-coded。
  // 假设rs3只能作为无旁路的FPU操作数，而且只对整数寄存器进行旁路操作（即只有两个）。

  val bypassed_rs1_data = Wire(Vec(issueWidth, Bits(registerWidth.W)))
  val bypassed_rs2_data = Wire(Vec(issueWidth, Bits(registerWidth.W)))
  val bypassed_pred_data = Wire(Vec(issueWidth, Bool()))
  bypassed_pred_data := DontCare

    
  for (w <- 0 until issueWidth) { // 遍历每个发射窗口
    val numReadPorts = numReadPortsArray(w)
      
      // 三个变量，
    var rs1_cases = Array((false.B, 0.U(registerWidth.W)))
    var rs2_cases = Array((false.B, 0.U(registerWidth.W)))
    var pred_cases = Array((false.B, 0.U(1.W)))

      // 保存译码输出的寄存器信息
    val prs1       = rrd_uops(w).prs1
    val lrs1_rtype = rrd_uops(w).lrs1_rtype
    val prs2       = rrd_uops(w).prs2
    val lrs2_rtype = rrd_uops(w).lrs2_rtype
    val ppred      = rrd_uops(w).ppred
    
    for (b <- 0 until numTotalBypassPorts)	// 遍历每个旁路端口
    {
      val bypass = io.bypass(b)
      // can't use "io.bypass.valid(b) since it would create a combinational loop on branch kills"
      // 输入bypass的每一项是一个带valid信号的执行单元回复（ExeUnitResp）
      // 不能直接使用这个valid值，因为这样会产生一个分支中止的组合循环。（？）
        
      rs1_cases ++= Array((bypass.valid && (prs1 === bypass.bits.uop.pdst) && bypass.bits.uop.rf_wen
        && bypass.bits.uop.dst_rtype === RT_FIX && lrs1_rtype === RT_FIX && (prs1 =/= 0.U), 		
                           bypass.bits.data))
      rs2_cases ++= Array((bypass.valid && (prs2 === bypass.bits.uop.pdst) && bypass.bits.uop.rf_wen
        && bypass.bits.uop.dst_rtype === RT_FIX && lrs2_rtype === RT_FIX && (prs2 =/= 0.U), 
                           bypass.bits.data))
        // 对于每个发射窗口，rs1_cases中将保存所有旁路端口的信息。
        // 其中，当 ①执行单元回复valid 
        // ②执行单元中微指令的目的寄存器就是rs1且不为0号寄存器
        // ③目的寄存器的写使能为1（只要是个寄存器就是1）
        // ④目的寄存器和译码出的lrs1种类为定点数（整数类型） 
        // 这四个条件同时满足时，信息的valid为1。（信息的data就是执行单元回复的data）
    }
    
    for (b <- 0 until numTotalPredBypassPorts) // 遍历每个pred旁路端口
    {
      val bypass = io.pred_bypass(b)
      pred_cases ++= Array((bypass.valid && (ppred === bypass.bits.uop.pdst) && bypass.bits.uop.is_sfb_br, bypass.bits.data))
    }
    
    if (numReadPorts > 0) bypassed_rs1_data(w)  := MuxCase(rrd_rs1_data(w), rs1_cases)
    if (numReadPorts > 1) bypassed_rs2_data(w)  := MuxCase(rrd_rs2_data(w), rs2_cases)
    if (enableSFBOpt)     bypassed_pred_data(w) := MuxCase(rrd_pred_data(w), pred_cases)
      // 根据（第一个）旁路得到的数据，修正读寄存器得到的数据。
      // 使用第一个旁路数据的原因？
      // MuxCase见最末尾
  }

    
    
  //-------------------------------------------------------------
  //-------------------------------------------------------------
  // **** Execute Stage **** 执行阶段
  //-------------------------------------------------------------
  //-------------------------------------------------------------

  for (w <- 0 until issueWidth) {	// 遍历每个发射窗口
    val numReadPorts = numReadPortsArray(w)
    if (numReadPorts > 0) exe_reg_rs1_data(w) := bypassed_rs1_data(w)
    if (numReadPorts > 1) exe_reg_rs2_data(w) := bypassed_rs2_data(w)
    if (numReadPorts > 2) exe_reg_rs3_data(w) := rrd_rs3_data(w)
    if (enableSFBOpt)     exe_reg_pred_data(w) := bypassed_pred_data(w)
    // ASSUMPTION: rs3 is FPU which is NOT bypassed
    // 用考虑了旁路后的到的数据作为传给执行阶段的数据。rs3不用考虑旁路因为是FPU用的（最开始提到的假设）。
  }
  // TODO add assert to detect bypass conflicts on non-bypassable things
  // TODO add assert that checks bypassing to verify there isn't something it hits rs3
  // 还需要添加检测到“给不能旁路的东西加旁路”已及“旁路中命中rs3”时发出警告的代码。

    
    
  //-------------------------------------------------------------
  // set outputs to execute pipelines  连接到输出，传输给执行阶段
  for (w <- 0 until issueWidth) {
    val numReadPorts = numReadPortsArray(w)

    io.exe_reqs(w).valid    := exe_reg_valids(w)
    io.exe_reqs(w).bits.uop := exe_reg_uops(w)
    if (numReadPorts > 0) io.exe_reqs(w).bits.rs1_data := exe_reg_rs1_data(w)
    if (numReadPorts > 1) io.exe_reqs(w).bits.rs2_data := exe_reg_rs2_data(w)
    if (numReadPorts > 2) io.exe_reqs(w).bits.rs3_data := exe_reg_rs3_data(w)
    if (enableSFBOpt)     io.exe_reqs(w).bits.pred_data := exe_reg_pred_data(w)
  }
}
```



### MuxCase

```scala
/** Given an association of values to enable signals, returns the first value with an associated
  * high enable signal.
  *
  * @example {{{
  * MuxCase(default, Array(c1 -> a, c2 -> b))
  * }}}
  */
object MuxCase {
  /** @param default the default value if none are enabled
    * @param mapping a set of data values with associated enables
    * @return the first value in mapping that is enabled */
  def apply[T <: Data] (default: T, mapping: Seq[(Bool, T)]): T = {
    var res = default
    for ((t, v) <- mapping.reverse){
      res = Mux(t, v, res)
    }
    res
  }
}
```

使用例：

```scala
result = MuxCase(111,[(0,222),
                      (1,333),
                      (1,444),
                      (0,555),
                      (1,666)])
// result = 333
```

