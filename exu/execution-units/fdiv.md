# fdiv
浮点除法单元。

BOOM通过一个FDiv/Sqrt模块（或者适用于short的fdiv模块）来完全支持浮点除法和开方操作。BOOM例化了hardfloat库中的一个双精度单元来实现这一点，这个单元有如下的特征与约束：

* 65bit的重编码双精度输入与输出
* 能同时执行一条除法操作和一条开方操作
* 操作是非流水线的，并且延迟不确定
* 提供一个*不稳定*的FIFO接口

单精度操作将其操作数放大到双精度（然后将输出缩小）。

尽管这个单元是非流水线的，它并不完全适合使用FunctionalUnit抽象类派生的两个流水线/非流水线子类，而是直接从FunctionalUnit抽象类继承（见execution-units.scala）。这是因为它提供了一个不稳定的FIFO接口：尽管在第i个周期此单元ready，也不能保证在第i+1个周期仍然是ready，即使没有新的操作入列。这对于实现是一个挑战，因为发射队列可能尝试发射一条指令到一个单元，但不能保证在后面的周期里，指令到达这个单元时是否能被接受。

解决方法是在这个单元内增加额外的缓冲区以保存指令，直到这些指令能直接释放到单元中。如果单元的缓冲区已满，可以安全地向发射队列发送反压（back pressure）信号。

```scala
package boom.exu
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import freechips.rocketchip.tile.FPConstants._
import freechips.rocketchip.tile
import boom.common._
import boom.util._
```



### UOPCodeFDivDecoder 类

微指令编码浮点除法的解码器，解码FPU的除法和开方信号。

```scala
/**
 * Decoder for FPU divide and square root signals
 */
  class UOPCodeFDivDecoder extends Module
  {
    val io = IO(new Bundle {
    val uopc = Input(Bits(UOPC_SZ.W))
    val sigs = Output(new tile.FPUCtrlSigs())
    })

  val N = BitPat("b0")
  val Y = BitPat("b1")
  val X = BitPat("b?")

  val decoder = freechips.rocketchip.rocket.DecodeLogic(io.uopc,
    // Note: not all of these signals are used or necessary, but we're
    // constrained by the need to fit the rocket.FPU units' ctrl signals.
    //                                       swap12         fma
    //                                       | swap32       | div
    //                                       | | singleIn   | | sqrt
    //                            ldst       | | | singleOut| | | wflags
    //                            | wen      | | | | from_int | | |
    //                            | | ren1   | | | | | to_int | | |
    //                            | | | ren2 | | | | | | fast | | |
    //                            | | | | ren3 | | | | | |  | | | |
    //                            | | | | |  | | | | | | |  | | | |
    /* Default */            List(X,X,X,X,X, X,X,X,X,X,X,X, X,X,X,X),
    Array(
      BitPat(uopFDIV_S)  -> List(X,X,Y,Y,X, X,X,Y,Y,X,X,X, X,Y,N,Y),
      BitPat(uopFDIV_D)  -> List(X,X,Y,Y,X, X,X,N,N,X,X,X, X,Y,N,Y),
      BitPat(uopFSQRT_S) -> List(X,X,Y,N,X, X,X,Y,Y,X,X,X, X,N,Y,Y),
      BitPat(uopFSQRT_D) -> List(X,X,Y,N,X, X,X,N,N,X,X,X, X,N,Y,Y)
    ))

  val s = io.sigs
  val sigs = Seq(s.ldst, s.wen, s.ren1, s.ren2, s.ren3, s.swap12,
                 s.swap23, s.singleIn, s.singleOut, s.fromint, s.toint, s.fastpipe, s.fma,
                 s.div, s.sqrt, s.wflags)
  sigs zip decoder map {case(s,d) => s := d}
}
```



### FDivSqrtUnit 类

浮点除法开方单元。

* 双精度。

* 必须根据精度需要选择是否对输入进行上转换（upconvert）和对输出进行下转换（downconvert）。

* 必须等到被清除的uop完成后再ready。
* fdiv/fsqrt单元使用不稳定的FIFO接口，因此我们必须花一个周期缓冲uop，以提供发射队列和fdiv/fsqrt单元之间的空闲时间。FDivUnit**直接从FunctionalUnit继承**，因为UnpipelinedFunctionalUnit只能处理1个inflight（处理中？）的uop，而FDivUnit最多包含2个inflight的uop；而且因为fdiv单元使用不稳定的FIFO接口，需要缓冲输入。

还未实现：扩展UnpipedFunctionalUnit以在处理多个inflight的uops。

```scala
/**
 * fdiv/fsqrt is douple-precision. Must upconvert inputs and downconvert outputs
 * as necessary.  Must wait till killed uop finishes before we're ready again.
 * fdiv/fsqrt unit uses an unstable FIFO interface, and thus we must spend a
 * cycle buffering up an uop to provide slack between the issue queue and the
 * fdiv/fsqrt unit.  FDivUnit inherents directly from FunctionalUnit, because
 * UnpipelinedFunctionalUnit can only handle 1 inflight uop, whereas FDivUnit
 * contains up to 2 inflight uops due to the need to buffer the input as the
 * fdiv unit uses an unstable FIFO interface.
 * TODO extend UnpipelinedFunctionalUnit to handle a >1 uops inflight.
 *
 * @param isPipelined is the functional unit pipelined
 * @param numStages number of stages for the functional unit
 * @param numBypassStages number of bypass stages
 * @param dataWidth width of the data out of the functional unit
 */
  class FDivSqrtUnit(implicit p: Parameters)
    extends FunctionalUnit(
    isPipelined = false,
    numStages = 1,
    numBypassStages = 0,
    dataWidth = 65,
    needsFcsr = true)
    with tile.HasFPUParameters
  {
      
      
    //--------------------------------------
    // buffer inputs and upconvert as needed
    // 为输入提供缓冲队列，根据需要进行上转换

  // provide a one-entry queue to store incoming uops while waiting for the fdiv/fsqrt unit to become available.
  // 提供一个单条目的队列，以储存等待fdiv/fsqrt单元valid的指令。包括valid位、浮点计算请求和浮点输入
  val r_buffer_val = RegInit(false.B)
  val r_buffer_req = Reg(new FuncUnitReq(dataWidth=65))
  val r_buffer_fin = Reg(new tile.FPInput)

  // 例化译码器
  val fdiv_decoder = Module(new UOPCodeFDivDecoder)
  fdiv_decoder.io.uopc := io.req.bits.uop.uopc

  // handle branch kill on queued entry
  // 根据清除信号选择是否valid，同时根据分支更新信息，更新队列中指令的分支掩码
  r_buffer_val := !IsKilledByBranch(io.brupdate, r_buffer_req.uop) && !io.req.bits.kill && r_buffer_val
  r_buffer_req.uop.br_mask := GetNewBrMask(io.brupdate, r_buffer_req.uop)

  // handle incoming uop, including upconversion as needed, and push back if our input queue is already occupied		缓冲队列中无valid指令时，整个功能单元才ready
  io.req.ready := !r_buffer_val

  // 定义上转换函数，对要操作的数据进行上转换
  def upconvert(x: UInt) = {
    val s2d = Module(new hardfloat.RecFNToRecFN(inExpWidth = 8, inSigWidth = 24, outExpWidth = 11, outSigWidth = 53))	// 例化转换模块，8+24 -> 11+53 （指数位+数值位）
    s2d.io.in := x
    s2d.io.roundingMode := 0.U
    s2d.io.detectTininess := DontCare
    s2d.io.out
  }
  val in1_upconvert = upconvert(unbox(io.req.bits.rs1_data, false.B, Some(tile.FType.S)))
  val in2_upconvert = upconvert(unbox(io.req.bits.rs2_data, false.B, Some(tile.FType.S)))

  // ？
  when (io.req.valid && !IsKilledByBranch(io.brupdate, io.req.bits.uop) && !io.req.bits.kill) {
    r_buffer_val := true.B
    r_buffer_req := io.req.bits
    r_buffer_req.uop.br_mask := GetNewBrMask(io.brupdate, io.req.bits.uop)
    r_buffer_fin <> fdiv_decoder.io.sigs

    r_buffer_fin.rm := io.fcsr_rm
    r_buffer_fin.typ := 0.U // unused for fdivsqrt
    val tag = !fdiv_decoder.io.sigs.singleIn
    r_buffer_fin.in1 := unbox(io.req.bits.rs1_data, tag, Some(tile.FType.D))
    r_buffer_fin.in2 := unbox(io.req.bits.rs2_data, tag, Some(tile.FType.D))
    when (fdiv_decoder.io.sigs.singleIn) {
      r_buffer_fin.in1 := in1_upconvert
      r_buffer_fin.in2 := in2_upconvert
    }
  }

  assert (!(r_buffer_val && io.req.valid), "[fdiv] a request is incoming while the buffer is already full.")

      
      
  //----------------------------------------
  // fdiv/fsqrt
  // 例化浮点除法/开方单元，将其与输入的缓冲队列连接

  val divsqrt = Module(new hardfloat.DivSqrtRecF64)

  val r_divsqrt_val = RegInit(false.B)  // inflight uop?
  val r_divsqrt_killed = Reg(Bool())           // has inflight uop been killed?
  val r_divsqrt_fin = Reg(new tile.FPInput)
  val r_divsqrt_uop = Reg(new MicroOp)

  // Need to buffer output until RF writeport is available.
  val output_buffer_available = Wire(Bool())

  val may_fire_input =
    r_buffer_val &&
    (r_buffer_fin.div || r_buffer_fin.sqrt) &&
    !r_divsqrt_val &&
    output_buffer_available

  val divsqrt_ready = Mux(divsqrt.io.sqrtOp, divsqrt.io.inReady_sqrt, divsqrt.io.inReady_div)
  divsqrt.io.inValid := may_fire_input // must be setup early
  divsqrt.io.sqrtOp := r_buffer_fin.sqrt
  divsqrt.io.a := r_buffer_fin.in1
  divsqrt.io.b := Mux(divsqrt.io.sqrtOp, r_buffer_fin.in1, r_buffer_fin.in2)
  divsqrt.io.roundingMode := r_buffer_fin.rm
  divsqrt.io.detectTininess := DontCare

  r_divsqrt_killed := r_divsqrt_killed || IsKilledByBranch(io.brupdate, r_divsqrt_uop) || io.req.bits.kill
  r_divsqrt_uop.br_mask := GetNewBrMask(io.brupdate, r_divsqrt_uop)

  when (may_fire_input && divsqrt_ready) {
    // Remove entry from the input buffer.
    // We don't have time to kill divsqrt request so must track if killed on entry.
    r_buffer_val := false.B
    r_divsqrt_val := true.B
    r_divsqrt_fin := r_buffer_fin
    r_divsqrt_uop := r_buffer_req.uop
    r_divsqrt_killed := IsKilledByBranch(io.brupdate, r_buffer_req.uop) || io.req.bits.kill
    r_divsqrt_uop.br_mask := GetNewBrMask(io.brupdate, r_buffer_req.uop)
  }

      
      
  //-----------------------------------------
  // buffer output and down-convert as needed
  // 将输出缓存，根据需要进行下转换

  val r_out_val = RegInit(false.B)
  val r_out_uop = Reg(new MicroOp)
  val r_out_flags_double = Reg(Bits())
  val r_out_wdata_double = Reg(Bits())

  output_buffer_available := !r_out_val

  r_out_uop.br_mask := GetNewBrMask(io.brupdate, r_out_uop)

  when (io.resp.ready || IsKilledByBranch(io.brupdate, r_out_uop) || io.req.bits.kill) {
    r_out_val := false.B
  }
  when (divsqrt.io.outValid_div || divsqrt.io.outValid_sqrt) {
    r_divsqrt_val := false.B

    r_out_val := !r_divsqrt_killed && !IsKilledByBranch(io.brupdate, r_divsqrt_uop) && !io.req.bits.kill
    r_out_uop := r_divsqrt_uop
    r_out_uop.br_mask := GetNewBrMask(io.brupdate, r_divsqrt_uop)
    r_out_wdata_double := sanitizeNaN(divsqrt.io.out, tile.FType.D)
    r_out_flags_double := divsqrt.io.exceptionFlags
    
    assert (r_divsqrt_val, "[fdiv] a response is being generated for no request.")
  }

  assert (!(r_out_val && (divsqrt.io.outValid_div || divsqrt.io.outValid_sqrt)),
    "[fdiv] Buffered output being overwritten by another output from the fdiv/fsqrt unit.")

  val downvert_d2s = Module(new hardfloat.RecFNToRecFN(
    inExpWidth = 11, inSigWidth = 53, outExpWidth = 8, outSigWidth = 24))
  downvert_d2s.io.in := r_out_wdata_double
  downvert_d2s.io.roundingMode := r_divsqrt_fin.rm
  downvert_d2s.io.detectTininess := DontCare
  val out_flags = r_out_flags_double | Mux(r_divsqrt_fin.singleIn, downvert_d2s.io.exceptionFlags, 0.U)

  io.resp.valid := r_out_val && !IsKilledByBranch(io.brupdate, r_out_uop)
  io.resp.bits.uop := r_out_uop
  io.resp.bits.data :=
    Mux(r_divsqrt_fin.singleIn,
      box(downvert_d2s.io.out, false.B),
      box(r_out_wdata_double, true.B))
  io.resp.bits.fflags.valid := io.resp.valid
  io.resp.bits.fflags.bits.uop := r_out_uop
  io.resp.bits.fflags.bits.uop.br_mask := GetNewBrMask(io.brupdate, r_out_uop)
  io.resp.bits.fflags.bits.flags := out_flags
}
