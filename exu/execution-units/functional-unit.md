# functional-unit
功能单元。

如果寄存器堆的旁路被禁用，功能单元必须在写回阶段自己实现旁路。

还未实现：探索条件IO的可能性（当有分支时给子类添加额外的IO端口之类的）。

```scala
// If regfile bypassing is disabled, then the functional unit must do its own
// bypassing in here on the WB stage (i.e., bypassing the io.resp.data)
// TODO: explore possibility of conditional IO fields? if a branch unit... how to add extra to IO in subclass?

package boom.exu
import chisel3._
import chisel3.util._
import chisel3.experimental.chiselName
import freechips.rocketchip.config.Parameters
import freechips.rocketchip.rocket.ALU._
import freechips.rocketchip.util._
import freechips.rocketchip.tile
import freechips.rocketchip.rocket.{PipelinedMultiplier,BP,BreakpointUnit,Causes,CSR}
import boom.common._
import boom.ifu._
import boom.util._
```



## 功能单元常量

### FUConstants 对象

因为一个执行流水线可以支持多功能单元，因此需要一个“标识”来指示一个执行单元中都有哪些功能单元。

这里定义了10位的掩码，每一位分别代表ALU、JMP、MEM、MUL、DIV、CSR、FPU、FDV、I2F、F2I。

在ExecutionUnit类的IO中就有如下代码，表示其中都有哪些功能单元。

val fu_types = Output(Bits(FUC_SZ.W))

```scala
/**
 * Functional unit constants
 */
  object FUConstants
  {
    // bit mask, since a given execution pipeline may support multiple functional units
    val FUC_SZ = 10
    val FU_X   = BitPat.dontCare(FUC_SZ)
    val FU_ALU =   1.U(FUC_SZ.W)
    val FU_JMP =   2.U(FUC_SZ.W)
    val FU_MEM =   4.U(FUC_SZ.W)
    val FU_MUL =   8.U(FUC_SZ.W)
    val FU_DIV =  16.U(FUC_SZ.W)
    val FU_CSR =  32.U(FUC_SZ.W)
    val FU_FPU =  64.U(FUC_SZ.W)
    val FU_FDV = 128.U(FUC_SZ.W)
    val FU_I2F = 256.U(FUC_SZ.W)
    val FU_F2I = 512.U(FUC_SZ.W)

  // FP stores generate data through FP F2I, and generate address through MemAddrCalc
  val FU_F2IMEM = 516.U(FUC_SZ.W)
}
import FUConstants._

```
### SupportedFuncUnits 类

告诉FU译码器它需要支持哪些功能单元。

此类作为RegisterRead类及其译码器类的参数类型。同时ExecutionUnit类已定义了一个函数返回一个此类型的数据，表示它支持哪些功能单元。

```scala
/**
 * Class to tell the FUDecoders what units it needs to support
 *
 * @param alu support alu unit?
 * @param bru support br unit?
 * @param mem support mem unit?
 * @param muld support multiple div unit?
 * @param fpu support FP unit?
 * @param csr support csr writing unit?
 * @param fdiv support FP div unit?
 * @param ifpu support int to FP unit?
 */
 class SupportedFuncUnits(
    val alu: Boolean  = false,
    val jmp: Boolean  = false,
    val mem: Boolean  = false,
    val muld: Boolean = false,
    val fpu: Boolean  = false,
    val csr: Boolean  = false,
    val fdiv: Boolean = false,
    val ifpu: Boolean = false)
 {
 }
```



## 功能单元的请求与回复

### FuncUnitReq 类

功能单元的输入，包括微指令uop（特质）、3个源寄存器的数据、pred数据和清除信号。

```scala
/**
 * Bundle for signals sent to the functional unit
 *
 * @param dataWidth width of the data sent to the functional unit
 */
  class FuncUnitReq(val dataWidth: Int)(implicit p: Parameters) extends BoomBundle
    with HasBoomUOP
  {
    val numOperands = 3

  val rs1_data = UInt(dataWidth.W)
  val rs2_data = UInt(dataWidth.W)
  val rs3_data = UInt(dataWidth.W) // only used for FMA units
  val pred_data = Bool()

  val kill = Bool() // kill everything
}
```
### FuncUnitResp 类

功能单元的输出，包括微指令uop（特质），预测位（此回复是从一个被predicated-off的指令来的？），数据，浮点标志回复（见execution-units.scala），地址和访存异常信息（只用于访存->LSU）,sfence（？）。

```scala
/**
 * Bundle for the signals sent out of the function unit
 *
 * @param dataWidth data sent from the functional unit
 */
 class FuncUnitResp(val dataWidth: Int)(implicit p: Parameters) extends BoomBundle
    with HasBoomUOP
 {
    val predicated = Bool() // Was this response from a predicated-off instruction
    val data = UInt(dataWidth.W)
    val fflags = new ValidIO(new FFlagsResp)
    val addr = UInt((vaddrBits+1).W) // only for maddr -> LSU
    val mxcpt = new ValidIO(UInt((freechips.rocketchip.rocket.Causes.all.max+2).W)) //only for maddr->LSU
    val sfence = Valid(new freechips.rocketchip.rocket.SFenceReq) // only for mcalc
 }
```



## 功能单元与分支

### BrResolutionInfo 类

已解决的分支信息。

包含微指令uop，valid位，指示是否错判的信号mispredict，指示分支跳转方向的信号taken，cfi_type，用于重新计算这条分支的pc的信息pc_sel（2位），jalr指令的目标地址以及地址的偏置。

```scala
/**
 * Branch resolution information given from the branch unit
 */
  class BrResolutionInfo(implicit p: Parameters) extends BoomBundle
  {
    val uop        = new MicroOp
    val valid      = Bool()
    val mispredict = Bool()
    val taken      = Bool()                     // which direction did the branch go?
    val cfi_type   = UInt(CFI_SZ.W)

  // Info for recalculating the pc for this branch
  val pc_sel     = UInt(2.W)

  val jalr_target = UInt(vaddrBitsExtended.W)
  val target_offset = SInt()
}
```
### BrUpdateInfo 类

分支更新信息。包含上下两个类。

```scala
class BrUpdateInfo(implicit p: Parameters) extends BoomBundle
{
  // On the first cycle we get masks to kill registers
  val b1 = new BrUpdateMasks
  // On the second cycle we get indices to reset pointers
  val b2 = new BrResolutionInfo
}
```
### BrUpdateMasks 类

分支更新掩码。独热码，每一位代表一个分支tag。两个变量分别是已解决（确认是否跳转）和误判的分支掩码。

```scala
class BrUpdateMasks(implicit p: Parameters) extends BoomBundle
{
  val resolve_mask = UInt(maxBrCount.W)
  val mispredict_mask = UInt(maxBrCount.W)
}
```



## 功能单元的层次

### FunctionalUnit 抽象类

抽象的顶层功能单元类，包裹着一个低层的手工功能单元。

只实现了IO端口。可根据参数选择是否为FPU、分支单元、内存地址计算单元提供额外的端口。

```scala
/**
 * Abstract top level functional unit class that wraps a lower level hand made functional unit
 *
 * @param isPipelined is the functional unit pipelined?
 * @param numStages how many pipeline stages does the functional unit have
 * @param numBypassStages how many bypass stages does the function unit have
 * @param dataWidth width of the data being operated on in the functional unit
 * @param hasBranchUnit does this functional unit have a branch unit?
 */
 abstract class FunctionalUnit(
    val isPipelined: Boolean,	// （内部的）功能单元是流水化的？
    val numStages: Int,			// 功能单元在流水线中的级数
    val numBypassStages: Int,	// 功能单元有多少旁路？
    val dataWidth: Int,			// 功能单元操作的数据宽度
    val isJmpUnit: Boolean = false,
    val isAluUnit: Boolean = false,
    val isMemAddrCalcUnit: Boolean = false,
    val needsFcsr: Boolean = false)
 (implicit p: Parameters) extends BoomModule
 {
     
    val io = IO(new Bundle {
    
        // 请求与回复
    	val req    = Flipped(new DecoupledIO(new FuncUnitReq(dataWidth)))
    	val resp   = (new DecoupledIO(new FuncUnitResp(dataWidth)))
		
        // 分支更新信息
    	val brupdate = Input(new BrUpdateInfo())

        // 旁路
   		val bypass = Output(Vec(numBypassStages, Valid(new ExeUnitResp(dataWidth))))

    	// only used by the fpu unit
    	val fcsr_rm = if (needsFcsr) Input(UInt(tile.FPConstants.RM_SZ.W)) else null

    	// only used by branch unit
    	val brinfo     = if (isAluUnit) Output(new BrResolutionInfo()) else null
    	val get_ftq_pc = if (isJmpUnit) Flipped(new GetPCFromFtqIO()) else null
    	val status     = if (isMemAddrCalcUnit) Input(new freechips.rocketchip.rocket.MStatus()) else null

    	// only used by memaddr calc unit
    	val bp = if (isMemAddrCalcUnit) Input(Vec(nBreakpoints, new BP)) else null

  })
     
}
```
#### PipelinedFunctionalUnit 抽象类

抽象顶层流水线化的功能单元，流水线长度由参数决定，其中的指令也因此有固定的需求周期数。由FunctionalUnit抽象类继承。

注意：这有助于跟踪哪些uop在中间阶段被清除，但检查和consumption同周期的清除是consumer的工作。（？）

在FunctionalUnit抽象类的基础上：

* 因为流水线化，所以始终为ready。
* 若其中的功能单元需要多个流水线级，则实现流水线的队列和旁路功能；若只需要一个流水线级，直接生成回复，只修改uop中的分支掩码。

```scala
/**
 * Abstract top level pipelined functional unit
 *
 * Note: this helps track which uops get killed while in intermediate stages,
 * but it is the job of the consumer to check for kills on the same cycle as consumption!!!
 *
 * @param numStages how many pipeline stages does the functional unit have
 * @param numBypassStages how many bypass stages does the function unit have
 * @param earliestBypassStage first stage that you can start bypassing from
 * @param dataWidth width of the data being operated on in the functional unit
 * @param hasBranchUnit does this functional unit have a branch unit?
 */
  abstract class PipelinedFunctionalUnit(
    numStages: Int,
    numBypassStages: Int,
    earliestBypassStage: Int,	// 可以开始旁路的第一个流水线级
    dataWidth: Int,
    isJmpUnit: Boolean = false,
    isAluUnit: Boolean = false,
    isMemAddrCalcUnit: Boolean = false,
    needsFcsr: Boolean = false)
(implicit p: Parameters) extends FunctionalUnit(
    isPipelined = true,
    numStages = numStages,
    numBypassStages = numBypassStages,
    dataWidth = dataWidth,
    isJmpUnit = isJmpUnit,
    isAluUnit = isAluUnit,
    isMemAddrCalcUnit = isMemAddrCalcUnit,
    needsFcsr = needsFcsr)
{
    // Pipelined functional unit is always ready. 流水线化的功能单元总是ready
    io.req.ready := true.B

    
  if (numStages > 0) {	// 功能单元需要多个周期的情况
      
    // 储存功能单元中所有流水线级上的valid和uop
    val r_valids = RegInit(VecInit(Seq.fill(numStages) { false.B }))
    val r_uops   = Reg(Vec(numStages, new MicroOp()))

      
    // handle incoming request 处理进来的请求
    r_valids(0) := io.req.valid && !IsKilledByBranch(io.brupdate, io.req.bits.uop) && !io.req.bits.kill
    r_uops(0)   := io.req.bits.uop
    r_uops(0).br_mask := GetNewBrMask(io.brupdate, io.req.bits.uop)
    
    // handle middle of the pipeline 处理流水线中间部分
    for (i <- 1 until numStages) {
      r_valids(i) := r_valids(i-1) && !IsKilledByBranch(io.brupdate, r_uops(i-1)) && !io.req.bits.kill
      r_uops(i)   := r_uops(i-1)
      r_uops(i).br_mask := GetNewBrMask(io.brupdate, r_uops(i-1))
    
      if (numBypassStages > 0) {
        io.bypass(i-1).bits.uop := r_uops(i-1) // 将所有流水线级中的uop保存到旁路中
      }
    }
    // 流水线就是一个队列，索引从0到(numStage-1)，0段是最新进来的
    // 由于是并行的，所以不用跟写一般的程序一样把两个for的顺序颠倒
      
      
    // handle outgoing (branch could still kill it)
    // consumer must also check for pipeline flushes (kills)
    // 处理输出（但仍可以被分支清除）。使用者必须检查流水线清除。
    io.resp.valid    := r_valids(numStages-1) && !IsKilledByBranch(io.brupdate, r_uops(numStages-1))
    io.resp.bits.predicated := false.B
    io.resp.bits.uop := r_uops(numStages-1)
    io.resp.bits.uop.br_mask := GetNewBrMask(io.brupdate, r_uops(numStages-1))
    // 均使用最后一级(numStage-1)即要出列的一项
      
    
    // bypassing (TODO allow bypass vector to have a different size from numStages)
    // 输出旁路。还未实现：让旁路的向量能有与numStage不同的大小。
    // 似乎强制要求earliestBypassStage == 0，然后把（下一个周期开始时）流水线中的所有信息全部作为旁路输出。
    if (numBypassStages > 0 && earliestBypassStage == 0) {
      io.bypass(0).bits.uop := io.req.bits.uop
      for (i <- 1 until numBypassStages) {
        io.bypass(i).bits.uop := r_uops(i-1)
      }
      // 由于并行性质，此时r_uop仍是上一个周期的值，所以bypass(0)要使用req中的值，bypass(i)要用r_uop(i-1)的值。
    }
  } 
    
    
  else {	// 功能单元只要一个周期的情况
    require (numStages == 0)
    // pass req straight through to response 
    // 直接将请求通给回复。这里指的是直接将req里的uop复制到resp中，只更新其中的分支掩码。（最后两行）

    // valid doesn't check kill signals, let consumer deal with it.
    // The LSU already handles it and this hurts critical path.
    // valid信号不会检查kill信号，而是让consumer处理。LSU已经处理了它，这会影响关键路径。
    io.resp.valid    := io.req.valid && !IsKilledByBranch(io.brupdate, io.req.bits.uop) // 传递valid信号
    io.resp.bits.predicated := false.B	// 这条回复不从predicted-off的指令来
    io.resp.bits.uop := io.req.bits.uop
    io.resp.bits.uop.br_mask := GetNewBrMask(io.brupdate, io.req.bits.uop)
  }
}
```
##### ALUUnit 类

ALU单元类，包裹RocketChips的ALU。从PipelinedFunctionalUnit抽象类继承。

```scala
/**
 * Functional unit that wraps RocketChips ALU
 *
 * @param isBranchUnit is this a branch unit?
 * @param numStages how many pipeline stages does the functional unit have
 * @param dataWidth width of the data being operated on in the functional unit
 */
  @chiselName
class ALUUnit(  isJmpUnit: Boolean = false, 
                numStages: Int = 1,	// 流水线队列中可有一条指令
                dataWidth: Int)
  (implicit p: Parameters)
  extends PipelinedFunctionalUnit(
    numStages = numStages,
    numBypassStages = numStages,
    isAluUnit = true,
    earliestBypassStage = 0,	// 可旁路，一个旁路端口
    dataWidth = dataWidth,
    isJmpUnit = isJmpUnit)
    with boom.ifu.HasBoomFrontendParameters
  {
    
    // 保存输入的uop
    val uop = io.req.bits.uop

    // immediate generation 生成立即数
    val imm_xprlen = ImmGen(uop.imm_packed, uop.ctrl.imm_sel)

      
  // operand 1 select
  var op1_data: UInt = null
  if (isJmpUnit) {
    // Get the uop PC for jumps
    val block_pc = AlignPCToBoundary(io.get_ftq_pc.pc, icBlockBytes)
    val uop_pc = (block_pc | uop.pc_lob) - Mux(uop.edge_inst, 2.U, 0.U)

    op1_data = Mux(uop.ctrl.op1_sel.asUInt === OP1_RS1 , io.req.bits.rs1_data,
               Mux(uop.ctrl.op1_sel.asUInt === OP1_PC  , Sext(uop_pc, xLen),
                                                         0.U))
  } else {
    op1_data = Mux(uop.ctrl.op1_sel.asUInt === OP1_RS1 , io.req.bits.rs1_data,
                                                         0.U)
  }

      
  // operand 2 select
  val op2_data = Mux(uop.ctrl.op2_sel === OP2_IMM,  Sext(imm_xprlen.asUInt, xLen),
                 Mux(uop.ctrl.op2_sel === OP2_IMMC, io.req.bits.uop.prs1(4,0),
                 Mux(uop.ctrl.op2_sel === OP2_RS2 , io.req.bits.rs2_data,
                 Mux(uop.ctrl.op2_sel === OP2_NEXT, Mux(uop.is_rvc, 2.U, 4.U),
                                                    0.U))))

  val alu = Module(new freechips.rocketchip.rocket.ALU())

  alu.io.in1 := op1_data.asUInt
  alu.io.in2 := op2_data.asUInt
  alu.io.fn  := uop.ctrl.op_fcn
  alu.io.dw  := uop.ctrl.fcn_dw


  // Did I just get killed by the previous cycle's branch, or by a flush pipeline?
  // 
  val killed = WireInit(false.B)
  when (io.req.bits.kill || IsKilledByBranch(io.brupdate, uop)) {
    killed := true.B
  }

  val rs1 = io.req.bits.rs1_data
  val rs2 = io.req.bits.rs2_data
  val br_eq  = (rs1 === rs2)
  val br_ltu = (rs1.asUInt < rs2.asUInt)
  val br_lt  = (~(rs1(xLen-1) ^ rs2(xLen-1)) & br_ltu |
                rs1(xLen-1) & ~rs2(xLen-1)).asBool

  val pc_sel = MuxLookup(uop.ctrl.br_type, PC_PLUS4,
                 Seq(   BR_N   -> PC_PLUS4,
                        BR_NE  -> Mux(!br_eq,  PC_BRJMP, PC_PLUS4),
                        BR_EQ  -> Mux( br_eq,  PC_BRJMP, PC_PLUS4),
                        BR_GE  -> Mux(!br_lt,  PC_BRJMP, PC_PLUS4),
                        BR_GEU -> Mux(!br_ltu, PC_BRJMP, PC_PLUS4),
                        BR_LT  -> Mux( br_lt,  PC_BRJMP, PC_PLUS4),
                        BR_LTU -> Mux( br_ltu, PC_BRJMP, PC_PLUS4),
                        BR_J   -> PC_BRJMP,
                        BR_JR  -> PC_JALR
                        ))

  val is_taken = io.req.valid &&
                   !killed &&
                   (uop.is_br || uop.is_jalr || uop.is_jal) &&
                   (pc_sel =/= PC_PLUS4)

  // "mispredict" means that a branch has been resolved and it must be killed
  val mispredict = WireInit(false.B)

  val is_br          = io.req.valid && !killed && uop.is_br && !uop.is_sfb
  val is_jal         = io.req.valid && !killed && uop.is_jal
  val is_jalr        = io.req.valid && !killed && uop.is_jalr

  when (is_br || is_jalr) {
    if (!isJmpUnit) {
      assert (pc_sel =/= PC_JALR)
    }
    when (pc_sel === PC_PLUS4) {
      mispredict := uop.taken
    }
    when (pc_sel === PC_BRJMP) {
      mispredict := !uop.taken
    }
  }

  val brinfo = Wire(new BrResolutionInfo)

  // note: jal doesn't allocate a branch-mask, so don't clear a br-mask bit
  brinfo.valid          := is_br || is_jalr
  brinfo.mispredict     := mispredict
  brinfo.uop            := uop
  brinfo.cfi_type       := Mux(is_jalr, CFI_JALR,
                           Mux(is_br  , CFI_BR, CFI_X))
  brinfo.taken          := is_taken
  brinfo.pc_sel         := pc_sel

  brinfo.jalr_target    := DontCare


  // Branch/Jump Target Calculation
  // For jumps we read the FTQ, and can calculate the target
  // For branches we emit the offset for the core to redirect if necessary
  val target_offset = imm_xprlen(20,0).asSInt
  brinfo.jalr_target := DontCare
  if (isJmpUnit) {
    def encodeVirtualAddress(a0: UInt, ea: UInt) = if (vaddrBitsExtended == vaddrBits) {
      ea
    } else {
      // Efficient means to compress 64-bit VA into vaddrBits+1 bits.
      // (VA is bad if VA(vaddrBits) != VA(vaddrBits-1)).
      val a = a0.asSInt >> vaddrBits
      val msb = Mux(a === 0.S || a === -1.S, ea(vaddrBits), !ea(vaddrBits-1))
      Cat(msb, ea(vaddrBits-1,0))
    }


    val jalr_target_base = io.req.bits.rs1_data.asSInt
    val jalr_target_xlen = Wire(UInt(xLen.W))
    jalr_target_xlen := (jalr_target_base + target_offset).asUInt
    val jalr_target = (encodeVirtualAddress(jalr_target_xlen, jalr_target_xlen).asSInt & -2.S).asUInt
    
    brinfo.jalr_target := jalr_target
    val cfi_idx = ((uop.pc_lob ^ Mux(io.get_ftq_pc.entry.start_bank === 1.U, 1.U << log2Ceil(bankBytes), 0.U)))(log2Ceil(fetchWidth),1)
    
    when (pc_sel === PC_JALR) {
      mispredict := !io.get_ftq_pc.next_val ||
                    (io.get_ftq_pc.next_pc =/= jalr_target) ||
                    !io.get_ftq_pc.entry.cfi_idx.valid ||
                    (io.get_ftq_pc.entry.cfi_idx.bits =/= cfi_idx)
    }
  }

  brinfo.target_offset := target_offset


  io.brinfo := brinfo



// Response
// TODO add clock gate on resp bits from functional units
//   io.resp.bits.data := RegEnable(alu.io.out, io.req.valid)
//   val reg_data = Reg(outType = Bits(width = xLen))
//   reg_data := alu.io.out
//   io.resp.bits.data := reg_data

  val r_val  = RegInit(VecInit(Seq.fill(numStages) { false.B }))
  val r_data = Reg(Vec(numStages, UInt(xLen.W)))
  val r_pred = Reg(Vec(numStages, Bool()))
  val alu_out = Mux(io.req.bits.uop.is_sfb_shadow && io.req.bits.pred_data,
    Mux(io.req.bits.uop.ldst_is_rs1, io.req.bits.rs1_data, io.req.bits.rs2_data),
    Mux(io.req.bits.uop.uopc === uopMOV, io.req.bits.rs2_data, alu.io.out))
  r_val (0) := io.req.valid
  r_data(0) := Mux(io.req.bits.uop.is_sfb_br, pc_sel === PC_BRJMP, alu_out)
  r_pred(0) := io.req.bits.uop.is_sfb_shadow && io.req.bits.pred_data
  for (i <- 1 until numStages) {
    r_val(i)  := r_val(i-1)
    r_data(i) := r_data(i-1)
    r_pred(i) := r_pred(i-1)
  }
  io.resp.bits.data := r_data(numStages-1)
  io.resp.bits.predicated := r_pred(numStages-1)
  // Bypass
  // for the ALU, we can bypass same cycle as compute
  require (numStages >= 1)
  require (numBypassStages >= 1)
  io.bypass(0).valid := io.req.valid
  io.bypass(0).bits.data := Mux(io.req.bits.uop.is_sfb_br, pc_sel === PC_BRJMP, alu_out)
  for (i <- 1 until numStages) {
    io.bypass(i).valid := r_val(i-1)
    io.bypass(i).bits.data := r_data(i-1)
  }

  // Exceptions
  io.resp.bits.fflags.valid := false.B
}
```
##### MemAddrCalcUnit 类

内存地址计算单元类，传入base和imm以计算地址，然后把要储存的数据传递给LSU。

65bit的浮点数据在储存时需要被译码为64bit形式。

由PipelinedFunctionalUnit抽象类继承。

```scala
/**
 * Functional unit that passes in base+imm to calculate addresses, and passes store data
 * to the LSU.
 * For floating point, 65bit FP store-data needs to be decoded into 64bit FP form
 */
  class MemAddrCalcUnit(implicit p: Parameters)
    extends PipelinedFunctionalUnit(
    numStages = 0,			// 流水线级数为0，可在一个周期内完成计算。
    numBypassStages = 0,
    earliestBypassStage = 0,
    dataWidth = 65, // TODO enable this only if FP is enabled? 还未实现：仅当FP使能时启用这一项
    isMemAddrCalcUnit = true)
    with freechips.rocketchip.rocket.constants.MemoryOpConstants
    with freechips.rocketchip.rocket.constants.ScalarOpConstants
  {
      
      
    // perform address calculation
    val sum = (io.req.bits.rs1_data.asSInt + io.req.bits.uop.imm_packed(19,8).asSInt).asUInt
    val ea_sign = Mux(sum(vaddrBits-1), ~sum(63,vaddrBits) === 0.U,
                                       sum(63,vaddrBits) =/= 0.U)
    val effective_address = Cat(ea_sign, sum(vaddrBits-1,0)).asUInt

  val store_data = io.req.bits.rs2_data

  io.resp.bits.addr := effective_address
  io.resp.bits.data := store_data

  if (dataWidth > 63) {
    assert (!(io.req.valid && io.req.bits.uop.ctrl.is_std &&
      io.resp.bits.data(64).asBool === true.B), "65th bit set in MemAddrCalcUnit.")

    assert (!(io.req.valid && io.req.bits.uop.ctrl.is_std && io.req.bits.uop.fp_val),
      "FP store-data should now be going through a different unit.")
  }

  assert (!(io.req.bits.uop.fp_val && io.req.valid && io.req.bits.uop.uopc =/=
          uopLD && io.req.bits.uop.uopc =/= uopSTA),
          "[maddrcalc] assert we never get store data in here.")

  // Handle misaligned exceptions
  val size = io.req.bits.uop.mem_size
  val misaligned =
    (size === 1.U && (effective_address(0) =/= 0.U)) ||
    (size === 2.U && (effective_address(1,0) =/= 0.U)) ||
    (size === 3.U && (effective_address(2,0) =/= 0.U))

  val bkptu = Module(new BreakpointUnit(nBreakpoints))
  bkptu.io.status := io.status
  bkptu.io.bp     := io.bp
  bkptu.io.pc     := DontCare
  bkptu.io.ea     := effective_address

  val ma_ld  = io.req.valid && io.req.bits.uop.uopc === uopLD && misaligned
  val ma_st  = io.req.valid && (io.req.bits.uop.uopc === uopSTA || io.req.bits.uop.uopc === uopAMO_AG) && misaligned
  val dbg_bp = io.req.valid && ((io.req.bits.uop.uopc === uopLD  && bkptu.io.debug_ld) ||
                                (io.req.bits.uop.uopc === uopSTA && bkptu.io.debug_st))
  val bp     = io.req.valid && ((io.req.bits.uop.uopc === uopLD  && bkptu.io.xcpt_ld) ||
                                (io.req.bits.uop.uopc === uopSTA && bkptu.io.xcpt_st))

  def checkExceptions(x: Seq[(Bool, UInt)]) =
    (x.map(_._1).reduce(_||_), PriorityMux(x))
  val (xcpt_val, xcpt_cause) = checkExceptions(List(
    (ma_ld,  (Causes.misaligned_load).U),
    (ma_st,  (Causes.misaligned_store).U),
    (dbg_bp, (CSR.debugTriggerCause).U),
    (bp,     (Causes.breakpoint).U)))

  io.resp.bits.mxcpt.valid := xcpt_val
  io.resp.bits.mxcpt.bits  := xcpt_cause
  assert (!(ma_ld && ma_st), "Mutually-exclusive exceptions are firing.")

  io.resp.bits.sfence.valid := io.req.valid && io.req.bits.uop.mem_cmd === M_SFENCE
  io.resp.bits.sfence.bits.rs1 := io.req.bits.uop.mem_size(0)
  io.resp.bits.sfence.bits.rs2 := io.req.bits.uop.mem_size(1)
  io.resp.bits.sfence.bits.addr := io.req.bits.rs1_data
  io.resp.bits.sfence.bits.asid := io.req.bits.rs2_data
}
```
##### FPUUnit 类

FPU单元类，包裹低层的FPU。

现在还不支持旁路。且为了方便写端口的调度，所有的FP指令都被扩展到最大延迟。

由PipelinedFunctionalUnit抽象类继承。

```scala
/**
 * Functional unit to wrap lower level FPU
 *
 * Currently, bypassing is unsupported!
 * All FP instructions are padded out to the max latency unit for easy
 * write-port scheduling.
 */
  class FPUUnit(implicit p: Parameters)
    extends PipelinedFunctionalUnit(
    numStages = p(tile.TileKey).core.fpu.get.dfmaLatency,	// 从参数获得流水线级数
    numBypassStages = 0,
    earliestBypassStage = 0,
    dataWidth = 65,
    needsFcsr = true)
  {
    val fpu = Module(new FPU())
    fpu.io.req.valid         := io.req.valid
    fpu.io.req.bits.uop      := io.req.bits.uop
    fpu.io.req.bits.rs1_data := io.req.bits.rs1_data
    fpu.io.req.bits.rs2_data := io.req.bits.rs2_data
    fpu.io.req.bits.rs3_data := io.req.bits.rs3_data
    fpu.io.req.bits.fcsr_rm  := io.fcsr_rm

  io.resp.bits.data              := fpu.io.resp.bits.data
  io.resp.bits.fflags.valid      := fpu.io.resp.bits.fflags.valid
  io.resp.bits.fflags.bits.uop   := io.resp.bits.uop
  io.resp.bits.fflags.bits.flags := fpu.io.resp.bits.fflags.bits.flags // kill me now
}
```
##### IntToFPUnit 类

Int转FP单元类。由PipelinedFunctionalUnit抽象类继承。

```scala
/**
 * Int to FP conversion functional unit
 *
 * @param latency the amount of stages to delay by
 */
  class IntToFPUnit(latency: Int)(implicit p: Parameters)
    extends PipelinedFunctionalUnit(
    numStages = latency,	// 流水线级数由构造时的参数决定
    numBypassStages = 0,
    earliestBypassStage = 0,
    dataWidth = 65,
    needsFcsr = true)
    with tile.HasFPUParameters
  {
    val fp_decoder = Module(new UOPCodeFPUDecoder) // TODO use a simpler decoder
    val io_req = io.req.bits
    fp_decoder.io.uopc := io_req.uop.uopc
    val fp_ctrl = fp_decoder.io.sigs
    val fp_rm = Mux(ImmGenRm(io_req.uop.imm_packed) === 7.U, io.fcsr_rm, ImmGenRm(io_req.uop.imm_packed))
    val req = Wire(new tile.FPInput)
    val tag = !fp_ctrl.singleIn

  req <> fp_ctrl

  req.rm := fp_rm
  req.in1 := unbox(io_req.rs1_data, tag, None)
  req.in2 := unbox(io_req.rs2_data, tag, None)
  req.in3 := DontCare
  req.typ := ImmGenTyp(io_req.uop.imm_packed)
  req.fmaCmd := DontCare

  assert (!(io.req.valid && fp_ctrl.fromint && req.in1(xLen).asBool),
    "[func] IntToFP integer input has 65th high-order bit set!")

  assert (!(io.req.valid && !fp_ctrl.fromint),
    "[func] Only support fromInt micro-ops.")

  val ifpu = Module(new tile.IntToFP(intToFpLatency))
  ifpu.io.in.valid := io.req.valid
  ifpu.io.in.bits := req
  ifpu.io.in.bits.in1 := io_req.rs1_data
  val out_double = Pipe(io.req.valid, !fp_ctrl.singleOut, intToFpLatency).bits

//io.resp.bits.data              := box(ifpu.io.out.bits.data, !io.resp.bits.uop.fp_single)
  io.resp.bits.data              := box(ifpu.io.out.bits.data, out_double)
  io.resp.bits.fflags.valid      := ifpu.io.out.valid
  io.resp.bits.fflags.bits.uop   := io.resp.bits.uop
  io.resp.bits.fflags.bits.flags := ifpu.io.out.bits.exc
}
```
##### PipelinedMulUnit 类

流水线化乘法功能单元，包裹了RocketChip的流水线化乘法器。由PipelinedFunctionalUnit抽象类继承。

（这部分本来在文件最后，但根据继承关系就放在这里了）

```scala
/**
 * Pipelined multiplier functional unit that wraps around the RocketChip pipelined multiplier
 *
 * @param numStages number of pipeline stages
 * @param dataWidth size of the data being passed into the functional unit
 */
 class PipelinedMulUnit(numStages: Int, dataWidth: Int)(implicit p: Parameters)
    extends PipelinedFunctionalUnit(
    numStages = numStages,		// 流水线级数由构造时的参数决定
    numBypassStages = 0,
    earliestBypassStage = 0,
    dataWidth = dataWidth)
 {
    val imul = Module(new PipelinedMultiplier(xLen, numStages))
    // request
    imul.io.req.valid    := io.req.valid
    imul.io.req.bits.fn  := io.req.bits.uop.ctrl.op_fcn
    imul.io.req.bits.dw  := io.req.bits.uop.ctrl.fcn_dw
    imul.io.req.bits.in1 := io.req.bits.rs1_data
    imul.io.req.bits.in2 := io.req.bits.rs2_data
    imul.io.req.bits.tag := DontCare
    // response
    io.resp.bits.data    := imul.io.resp.bits.data
 }
```
#### IterativeFunctionalUnit 抽象类

可迭代（？）或非流水线化的功能单元，只能同时处理一条微指令。假设在请求和回复之间至少有一个寄存器（的延迟）。

还未实现：同时能支持N条微指令。

由FunctionalUnit抽象类继承。

```scala
/**
 * Iterative/unpipelined functional unit, can only hold a single MicroOp at a time
 * assumes at least one register between request and response
 *
 * TODO allow up to N micro-ops simultaneously.
 *
 * @param dataWidth width of the data to be passed into the functional unit
 */
  abstract class IterativeFunctionalUnit(dataWidth: Int)(implicit p: Parameters)
    extends FunctionalUnit(
    isPipelined = false,
    numStages = 1,
    numBypassStages = 0,
    dataWidth = dataWidth)
  {
      
      
    val r_uop = Reg(new MicroOp())

  val do_kill = Wire(Bool())
  do_kill := io.req.bits.kill // irrelevant default

  when (io.req.fire()) {
    // update incoming uop
    do_kill := IsKilledByBranch(io.brupdate, io.req.bits.uop) || io.req.bits.kill
    r_uop := io.req.bits.uop
    r_uop.br_mask := GetNewBrMask(io.brupdate, io.req.bits.uop)
  } .otherwise {
    do_kill := IsKilledByBranch(io.brupdate, r_uop) || io.req.bits.kill
    r_uop.br_mask := GetNewBrMask(io.brupdate, r_uop)
  }

  // assumes at least one pipeline register between request and response
  io.resp.bits.uop := r_uop
}
```
##### DivUnit 类

除法功能单元。由IterativeFunctionalUnit抽象类继承。

```scala
/**
 * Divide functional unit.
 *
 * @param dataWidth data to be passed into the functional unit
 */
 class DivUnit(dataWidth: Int)(implicit p: Parameters)
    extends IterativeFunctionalUnit(dataWidth)
 {

  // We don't use the iterative multiply functionality here.
  // Instead we use the PipelinedMultiplier
  val div = Module(new freechips.rocketchip.rocket.MulDiv(mulDivParams, width = dataWidth))

  // request
  div.io.req.valid    := io.req.valid && !this.do_kill
  div.io.req.bits.dw  := io.req.bits.uop.ctrl.fcn_dw
  div.io.req.bits.fn  := io.req.bits.uop.ctrl.op_fcn
  div.io.req.bits.in1 := io.req.bits.rs1_data
  div.io.req.bits.in2 := io.req.bits.rs2_data
  div.io.req.bits.tag := DontCare
  io.req.ready        := div.io.req.ready

  // handle pipeline kills and branch misspeculations
  div.io.kill         := this.do_kill

  // response
  io.resp.valid       := div.io.resp.valid && !this.do_kill
  div.io.resp.ready   := io.resp.ready
  io.resp.bits.data   := div.io.resp.bits.data
}
```

