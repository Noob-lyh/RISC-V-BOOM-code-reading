# execution-unit
执行单元。



### BOOM的执行单元

在BOOM中，一个执行单元(Execution Unit)连接到一个发射端口上，接收一个发射的指令（即三发射的BOOM需要有三个执行单元），其内部拥有一个或多个功能单元(Functional Unit)。 比如一个执行单元可能仅包括一个整数ALU，也可能是整数ALU、整数乘法的组合。本文件中，先定义了一个抽象的执行单元类ExecutionUnit，然后派生出两个子类ALUExeUnit和FPUExeUnit，其中可含有的功能单元不同，而且含有什么样的功能单元可由构造时传入的参数决定。执行单元的示意图如下：

![](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/exeunit.jpg)



### 代码说明部分

发射窗口将计划把微指令发送到一个特定的执行流水线。

一个给定的执行流水线可能包含多个功能单元；有一个或多个的读端口、一个或多个的写端口。

```scala
// The issue window schedules micro-ops onto a specific execution pipeline
// A given execution pipeline may contain multiple functional units; one or more
// read ports, and one or more writeports.
```
```scala
package boom.exu
import scala.collection.mutable.{ArrayBuffer}
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.{Parameters}
import freechips.rocketchip.rocket.{BP}
import freechips.rocketchip.tile.{XLen, RoCCCoreIO}
import freechips.rocketchip.tile
import FUConstants._
import boom.common._
import boom.ifu.{GetPCFromFtqIO}
import boom.util.{ImmGen, IsKilledByBranch, BranchKillableQueue, BoomCoreStringPrefix}
```



### ExeUnitResp 类

执行单元回复。
```scala
/**
 * Response from Execution Unit. Bundles a MicroOp with data
 * （执行单元的回复，将微指令和数据绑定到一个端口）
 * @param dataWidth width of the data coming from the execution unit
 */
  class ExeUnitResp(val dataWidth: Int)(implicit p: Parameters) 
	extends BoomBundle
    with HasBoomUOP // （特质，只有一个微指令成员uop）
  {
    val data = Bits(dataWidth.W)
      // 数据
    val predicated = Bool() // Was this predicated off?
      // 指示微指令是否是被预测的
    val fflags = new ValidIO(new FFlagsResp) // write fflags to ROB // TODO: Do this better
      // Valid端口，其中的类型见下方。
      // 将fflags写到ROB
  }
```



### FFlagsResp 类

浮点标志回复。
```scala
/**
 * Floating Point flag response
 */
  class FFlagsResp(implicit p: Parameters) extends BoomBundle
  {
    val uop = new MicroOp() // 微指令
    val flags = Bits(tile.FPConstants.FLAGS_SZ.W) // 标志
  }
```



### ExecutionUnit 抽象类

执行单元抽象类，将底层功能单元（functional units）包装成一个多功能执行单元（execution unit）。

在本文件中有两个子类ALUExeUnit和FPUExeUnit。

```scala
/**
 * Abstract Top level Execution Unit that wraps lower level functional units to make a
 * multi function execution unit.
 *
 * @param readsIrf does this exe unit need a integer regfile port
 * @param writesIrf does this exe unit need a integer regfile port
 * @param readsFrf does this exe unit need a integer regfile port
 * @param writesFrf does this exe unit need a integer regfile port
 * @param writesLlIrf does this exe unit need a integer regfile port
 * @param writesLlFrf does this exe unit need a integer regfile port
 * @param numBypassStages number of bypass ports for the exe unit
 * @param dataWidth width of the data coming out of the exe unit
 * @param bypassable is the exe unit able to be bypassed
 * @param hasMem does the exe unit have a MemAddrCalcUnit
 * @param hasCSR does the exe unit write to the CSRFile
 * @param hasBrUnit does the exe unit have a branch unit
 * @param hasAlu does the exe unit have a alu
 * @param hasFpu does the exe unit have a fpu
 * @param hasMul does the exe unit have a multiplier
 * @param hasDiv does the exe unit have a divider
 * @param hasFdiv does the exe unit have a FP divider
 * @param hasIfpu does the exe unit have a int to FP unit
 * @param hasFpiu does the exe unit have a FP to int unit
 */
    abstract class ExecutionUnit(
    val readsIrf         : Boolean       = false, 	// ----------------
    val writesIrf        : Boolean       = false, 	//
    val readsFrf         : Boolean       = false, 	// 需要整数寄存器端口？
    val writesFrf        : Boolean       = false, 	//
    val writesLlIrf      : Boolean       = false, 	//
    val writesLlFrf      : Boolean       = false, 	// ----------------
    val numBypassStages  : Int,						// 旁路端口数量 
    val dataWidth        : Int,						// 数据宽度
    val bypassable       : Boolean       = false,   
        // TODO make override def for code clarity （为了代码易读性还需要做重载定义） 指示执行单元有无旁路逻辑
    val alwaysBypassable : Boolean       = false,
    val hasMem           : Boolean       = false,	// 有内存地址计算（访存）单元？
    val hasCSR           : Boolean       = false,	// 可写到CSRFile？
    val hasJmpUnit       : Boolean       = false,	
    val hasAlu           : Boolean       = false,
    val hasFpu           : Boolean       = false,
    val hasMul           : Boolean       = false,
    val hasDiv           : Boolean       = false,
    val hasFdiv          : Boolean       = false,
    val hasIfpu          : Boolean       = false,	// int转FP
    val hasFpiu          : Boolean       = false,	// FP转int
    val hasRocc          : Boolean       = false
    )(implicit p: Parameters) extends BoomModule
    {

        
        
  val io = IO(new Bundle {
    val fu_types = Output(Bits(FUC_SZ.W)) 
      // 【输出】拥有的功能单元类型。10位独热码，分别代表是否拥有10种功能单元。

    val req      = Flipped(new DecoupledIO(new FuncUnitReq(dataWidth)))
      // 【输入】接受功能单元请求
    
    val iresp    = if (writesIrf)   new DecoupledIO(new ExeUnitResp(dataWidth)) else null
    val fresp    = if (writesFrf)   new DecoupledIO(new ExeUnitResp(dataWidth)) else null
    val ll_iresp = if (writesLlIrf) new DecoupledIO(new ExeUnitResp(dataWidth)) else null
    val ll_fresp = if (writesLlFrf) new DecoupledIO(new ExeUnitResp(dataWidth)) else null
      // 【输出】根据参数需求，搭建对应的输出写端口


    val bypass   = Output(Vec(numBypassStages, Valid(new ExeUnitResp(dataWidth))))
      // 【输出】旁路。numBypassStages个执行单元回复（Valid端口）。
    val brupdate = Input(new BrUpdateInfo())
      // 【输入】分支更新信息


    // only used by the rocc unit （为rocc单元提供额外的IO端口）
    val rocc = if (hasRocc) new RoCCShimCoreIO else null
    
      
    // only used by the branch unit （分支单元）
    val brinfo     = if (hasAlu) Output(new BrResolutionInfo()) else null
      // 【输出】有ALU的分支单元，可以计算后输出分支解析信息。
    val get_ftq_pc = if (hasJmpUnit) Flipped(new GetPCFromFtqIO()) else null
      // 【输入】有跳转单元的分支单元，从取队列得到指令的PC（JALR指令则为下条指令的PC）
    val status     = Input(new freechips.rocketchip.rocket.MStatus())
      // 【输入】机器状态
    
      
    // only used by the fpu unit （浮点运算单元）
    val fcsr_rm = if (hasFcsr) Input(Bits(tile.FPConstants.RM_SZ.W)) else null
      // 【输入】？
    
      
    // only used by the mem unit（访存单元）
    val lsu_io = if (hasMem) Flipped(new boom.lsu.LSUExeIO) else null
      // 加载储存单元的IO端口
    val bp = if (hasMem) Input(Vec(nBreakpoints, new BP)) else null
      // 【输入】断点信息
    
      
    // TODO move this out of ExecutionUnit （这部分需要被移出）
    val com_exception = if (hasMem || hasRocc) Input(Bool()) else null
      // 【输入】 有访存或rocc单元时，用于输入提交阶段的异常信息。
  })
        
        
        
      // 根据配置，修改有效位、预测信息（？）
  if (writesIrf)   {
    io.iresp.bits.fflags.valid := false.B
    io.iresp.bits.predicated := false.B
    assert(io.iresp.ready)
  }
  if (writesLlIrf) {
    io.ll_iresp.bits.fflags.valid := false.B
    io.ll_iresp.bits.predicated := false.B
  }
  if (writesFrf)   {
    io.fresp.bits.fflags.valid := false.B
    io.fresp.bits.predicated := false.B
    assert(io.fresp.ready)
  }
  if (writesLlFrf) {
    io.ll_fresp.bits.fflags.valid := false.B
    io.ll_fresp.bits.predicated := false.B
  }

        
        
        
  // TODO add "number of fflag ports", so we can properly account for FPU+Mem combinations （还需要增加“fflag端口的数量”，这样就可以正确地解释FPU+Mem组合）
  def hasFFlags     : Boolean = hasFpu || hasFdiv

        
        
  require ((hasFpu || hasFdiv) ^ (hasAlu || hasMul || hasMem || hasIfpu),
    "[execute] we no longer support mixing FP and Integer functional units in the same exe unit.") // 不可混合FP和int单元，否则报错
  
        
        
  def hasFcsr = hasIfpu || hasFpu || hasFdiv

        
        
  require (bypassable || !alwaysBypassable,
    "[execute] an execution unit must be bypassable if it is always bypassable")
        // 检查简单的逻辑错误

        
        
  def supportedFuncUnits = { // 定义一个方法，返回支持的功能单元类型
    new SupportedFuncUnits(
      alu = hasAlu,
      jmp = hasJmpUnit,
      mem = hasMem,
      muld = hasMul || hasDiv,
      fpu = hasFpu,
      csr = hasCSR,
      fdiv = hasFdiv,
      ifpu = hasIfpu)
  }
}
```



### ALUExeUnit  类

通用计算执行单元。

```scala
/**
 * ALU execution unit that can have a branch, alu, mul, div, int to FP,
 * and memory unit.
 * ALU执行单元，可以有分支、ALU、乘法、除法、int转FP和内存单元。
 *
 * @param hasBrUnit does the exe unit have a branch unit
 * @param hasCSR does the exe unit write to the CSRFile
 * @param hasAlu does the exe unit have a alu
 * @param hasMul does the exe unit have a multiplier
 * @param hasDiv does the exe unit have a divider
 * @param hasIfpu does the exe unit have a int to FP unit
 * @param hasMem does the exe unit have a MemAddrCalcUnit
 */
  class ALUExeUnit(		// 参数
    hasJmpUnit     : Boolean = false,
    hasCSR         : Boolean = false,
    hasAlu         : Boolean = true,
    hasMul         : Boolean = false,
    hasDiv         : Boolean = false,
    hasIfpu        : Boolean = false,
    hasMem         : Boolean = false,
    hasRocc        : Boolean = false)
  (implicit p: Parameters)
  extends ExecutionUnit( 	// 从执行单元类继承时的参数
    readsIrf         = true,
    writesIrf        = hasAlu || hasMul || hasDiv, // ALU、乘除需要写整数regfile
    writesLlIrf      = hasMem || hasRocc,          // mem、rocc需要加载整数端口
    writesLlFrf      = (hasIfpu || hasMem) && p(tile.TileKey).core.fpu != None,
      // ？
    numBypassStages  =
      if (hasAlu && hasMul) 3 //TODO XXX p(tile.TileKey).core.imulLatency
      else if (hasAlu) 1 else 0,
      // ?
    dataWidth        = p(tile.XLen) + 1,
    bypassable       = hasAlu,
    alwaysBypassable = hasAlu && !(hasMem || hasJmpUnit || hasMul || hasDiv || hasCSR || hasIfpu || hasRocc),
    hasCSR           = hasCSR,
    hasJmpUnit       = hasJmpUnit,
    hasAlu           = hasAlu,
    hasMul           = hasMul,
    hasDiv           = hasDiv,
    hasIfpu          = hasIfpu,
    hasMem           = hasMem,
    hasRocc          = hasRocc)
    with freechips.rocketchip.rocket.constants.MemoryOpConstants
  {
      
      
      // 检测不支持的单元组合
    require(!(hasRocc && !hasCSR),
    "RoCC needs to be shared with CSR unit") 
      // 有RoCC就必须有CSR
    require(!(hasMem && hasRocc),
    "We do not support execution unit with both Mem and Rocc writebacks") 
      // mem和rocc不共存
    require(!(hasMem && hasIfpu),
    "TODO. Currently do not support AluMemExeUnit with FP")
      // mem和int转fp不共存，当前还不支持

      
      
      // 打印功能单元信息
  val out_str =
    BoomCoreStringPrefix("==ExeUnit==") +
    (if (hasAlu)  BoomCoreStringPrefix(" - ALU") else "") +
    (if (hasMul)  BoomCoreStringPrefix(" - Mul") else "") +
    (if (hasDiv)  BoomCoreStringPrefix(" - Div") else "") +
    (if (hasIfpu) BoomCoreStringPrefix(" - IFPU") else "") +
    (if (hasMem)  BoomCoreStringPrefix(" - Mem") else "") +
    (if (hasRocc) BoomCoreStringPrefix(" - RoCC") else "")

  override def toString: String = out_str.toString

      // 除法、int转FP的繁忙信号
  val div_busy  = WireInit(false.B)
  val ifpu_busy = WireInit(false.B)

      
  // The Functional Units --------------------
  // Specifically the functional units with fast writeback to IRF
  // 功能单元，特别是能快速写回到整数寄存器堆的功能单元
  val iresp_fu_units = ArrayBuffer[FunctionalUnit]() // 整数功能单元阵列

  io.fu_types := Mux(hasAlu.B, FU_ALU, 0.U) |
                 Mux(hasMul.B, FU_MUL, 0.U) |
                 Mux(!div_busy && hasDiv.B, FU_DIV, 0.U) |
                 Mux(hasCSR.B, FU_CSR, 0.U) |
                 Mux(hasJmpUnit.B, FU_JMP, 0.U) |
                 Mux(!ifpu_busy && hasIfpu.B, FU_I2F, 0.U) |
                 Mux(hasMem.B, FU_MEM, 0.U)
      // 输出含有的功能
      // 对于ALU、MUL乘法单元、CSR单元、JMP单元和MEN单元，只要在该执行单元中存在，其ready信号就为1， 
      //   说明这些单元是流水线的，每个周期都能接收一个指令； 
      // 而对于DIV单元和Int转FP单元来说，由于其内部不是流水线的，因此需要有一个busy信号，当该单元空闲时
      //   ready信号才会置位。

      
  // ALU Unit ------------------------------- ALU 功能单元
  var alu: ALUUnit = null // 变量
  if (hasAlu) {
    alu = Module(new ALUUnit(isJmpUnit = hasJmpUnit,  // 使用rocketchip的ALUUnit例化
                             numStages = numBypassStages,
                             dataWidth = xLen))
    alu.io.req.valid := (
      io.req.valid &&
      (io.req.bits.uop.fu_code === FU_ALU ||
       io.req.bits.uop.fu_code === FU_JMP ||
      (io.req.bits.uop.fu_code === FU_CSR && io.req.bits.uop.uopc =/= uopROCC)))
    //ROCC Rocc Commands are taken by the RoCC unit
    // alu的valid需要req的valid，且指令的功能编码为ALU或JMO，或不是ROCC的CSR

    alu.io.req.bits.uop      := io.req.bits.uop
    alu.io.req.bits.kill     := io.req.bits.kill
    alu.io.req.bits.rs1_data := io.req.bits.rs1_data
    alu.io.req.bits.rs2_data := io.req.bits.rs2_data
    alu.io.req.bits.rs3_data := DontCare
    alu.io.req.bits.pred_data := io.req.bits.pred_data
    alu.io.resp.ready := DontCare
    alu.io.brupdate := io.brupdate
    
    iresp_fu_units += alu // 填充进阵列
    
    // Bypassing only applies to ALU （旁路只应用于ALU）
    io.bypass := alu.io.bypass
    
    // branch unit is embedded inside the ALU （分支单元被嵌入ALU）
    io.brinfo := alu.io.brinfo
    if (hasJmpUnit) { // 如果有跳转功能单元，用get_ftq_pc得到当前pc
      alu.io.get_ftq_pc <> io.get_ftq_pc
    }
  }

      
  var rocc: RoCCShim = null
  if (hasRocc) {
    rocc = Module(new RoCCShim)
    rocc.io.req.valid         := io.req.valid && io.req.bits.uop.uopc === uopROCC
    rocc.io.req.bits          := DontCare
    rocc.io.req.bits.uop      := io.req.bits.uop
    rocc.io.req.bits.kill     := io.req.bits.kill
    rocc.io.req.bits.rs1_data := io.req.bits.rs1_data
    rocc.io.req.bits.rs2_data := io.req.bits.rs2_data
    rocc.io.brupdate          := io.brupdate // We should assert on this somewhere
    rocc.io.status            := io.status
    rocc.io.exception         := io.com_exception
    io.rocc                   <> rocc.io.core

    rocc.io.resp.ready        := io.ll_iresp.ready
    io.ll_iresp.valid         := rocc.io.resp.valid
    io.ll_iresp.bits.uop      := rocc.io.resp.bits.uop
    io.ll_iresp.bits.data     := rocc.io.resp.bits.data
  }


  // Pipelined, IMul Unit ------------------ 流水线化的整数乘法单元
  var imul: PipelinedMulUnit = null
  if (hasMul) {
    imul = Module(new PipelinedMulUnit(imulLatency, xLen))
    imul.io <> DontCare
    imul.io.req.valid         := io.req.valid && io.req.bits.uop.fu_code_is(FU_MUL)
    imul.io.req.bits.uop      := io.req.bits.uop
    imul.io.req.bits.rs1_data := io.req.bits.rs1_data
    imul.io.req.bits.rs2_data := io.req.bits.rs2_data
    imul.io.req.bits.kill     := io.req.bits.kill
    imul.io.brupdate := io.brupdate
    iresp_fu_units += imul
  }

      
  var ifpu: IntToFPUnit = null
  if (hasIfpu) {
    ifpu = Module(new IntToFPUnit(latency=intToFpLatency))
    ifpu.io.req        <> io.req
    ifpu.io.req.valid  := io.req.valid && io.req.bits.uop.fu_code_is(FU_I2F)
    ifpu.io.fcsr_rm    := io.fcsr_rm
    ifpu.io.brupdate   <> io.brupdate
    ifpu.io.resp.ready := DontCare

    // buffer up results since we share write-port on integer regfile.
    // 因为共用整数寄存器堆的写端口，需要把结果先缓存起来
    val queue = Module(new BranchKillableQueue(new ExeUnitResp(dataWidth),
      entries = intToFpLatency + 3)) // TODO being overly conservative（过于保守）
      // 定义一个可分支清除的队列，队列中的元素为执行单元的回复，长度为int转FP延迟数+3
    queue.io.enq.valid       := ifpu.io.resp.valid
    queue.io.enq.bits.uop    := ifpu.io.resp.bits.uop
    queue.io.enq.bits.data   := ifpu.io.resp.bits.data
    queue.io.enq.bits.predicated := ifpu.io.resp.bits.predicated
    queue.io.enq.bits.fflags := ifpu.io.resp.bits.fflags
    queue.io.brupdate := io.brupdate
    queue.io.flush := io.req.bits.kill
    
    io.ll_fresp <> queue.io.deq
    ifpu_busy := !(queue.io.empty)
    assert (queue.io.enq.ready) // ready为0说明队列已满？
  }

      
  // Div/Rem Unit ----------------------- 除法/取余单元
  var div: DivUnit = null
  val div_resp_val = WireInit(false.B)
  if (hasDiv) {
    div = Module(new DivUnit(xLen))
    div.io <> DontCare
    div.io.req.valid           := io.req.valid && io.req.bits.uop.fu_code_is(FU_DIV) && hasDiv.B
    div.io.req.bits.uop        := io.req.bits.uop
    div.io.req.bits.rs1_data   := io.req.bits.rs1_data
    div.io.req.bits.rs2_data   := io.req.bits.rs2_data
    div.io.brupdate            := io.brupdate
    div.io.req.bits.kill       := io.req.bits.kill

    // share write port with the pipelined units （和流水线化单元共用写端口）
    div.io.resp.ready := !(iresp_fu_units.map(_.io.resp.valid).reduce(_|_))
      // 仅当其他功能单元valid均为0时，此单元valid才为1
    
    div_resp_val := div.io.resp.valid
    div_busy     := !div.io.req.ready ||
                    (io.req.valid && io.req.bits.uop.fu_code_is(FU_DIV))
      // 当ready为0（不空闲） 或 valid且新指令需要用到（可用且有请求）时为busy状态
    
    iresp_fu_units += div
  }

      
  // Mem Unit -------------------------- 访存单元
  if (hasMem) {
    require(!hasAlu) // 有mem，则不能有alu功能模块
    val maddrcalc = Module(new MemAddrCalcUnit) // 访存——内存地址计算
    maddrcalc.io.req        <> io.req
    maddrcalc.io.req.valid  := io.req.valid && io.req.bits.uop.fu_code_is(FU_MEM)
    maddrcalc.io.brupdate     <> io.brupdate
    maddrcalc.io.status     := io.status
    maddrcalc.io.bp         := io.bp
    maddrcalc.io.resp.ready := DontCare
    require(numBypassStages == 0)

    io.lsu_io.req := maddrcalc.io.resp
    
    io.ll_iresp <> io.lsu_io.iresp
    if (usingFPU) {
      io.ll_fresp <> io.lsu_io.fresp
    }
  }

      
      
  // Outputs (Write Port #0)  --------------- 输出（写端口#0）
  if (writesIrf) { // 如果有写整数寄存器堆的需求
    io.iresp.valid     := iresp_fu_units.map(_.io.resp.valid).reduce(_|_)
    io.iresp.bits.uop  := PriorityMux(iresp_fu_units.map(f =>
      (f.io.resp.valid, f.io.resp.bits.uop)))
    io.iresp.bits.data := PriorityMux(iresp_fu_units.map(f =>
      (f.io.resp.valid, f.io.resp.bits.data)))
    io.iresp.bits.predicated := PriorityMux(iresp_fu_units.map(f =>
      (f.io.resp.valid, f.io.resp.bits.predicated)))
      // 取出所有功能单元中第一个valid的uop、data、predicated

      
      
    // pulled out for critical path reasons （由于关键路径的原因退出）
    // TODO: Does this make sense as part of the iresp bundle? （也许可以放到iresp里？）
    if (hasAlu) {
      io.iresp.bits.uop.csr_addr := ImmGen(alu.io.resp.bits.uop.imm_packed, IS_I).asUInt
      io.iresp.bits.uop.ctrl.csr_cmd := alu.io.resp.bits.uop.ctrl.csr_cmd
    }
  }

  assert ((PopCount(iresp_fu_units.map(_.io.resp.valid)) <= 1.U && !div_resp_val) ||
          (PopCount(iresp_fu_units.map(_.io.resp.valid)) <= 2.U && (div_resp_val)),
          "Multiple functional units are fighting over the write port.")
      // 回复valid的大于1 或 除法回复有效
      // 回复valid的大于2 或 除法回复无效
      // 以上两条同时满足时，警告多个功能单元抢夺写端口（其实就是DIV外的valid大于1，不知道为啥要这么写）
}
```



### FPUExeUnit 类

浮点执行单元。代码和ALUExeUnit的很类似。

```scala
/**
 * FPU-only unit, with optional second write-port for ToInt micro-ops.
 *
 * @param hasFpu does the exe unit have a fpu
 * @param hasFdiv does the exe unit have a FP divider
 * @param hasFpiu does the exe unit have a FP to int unit
 */
class FPUExeUnit( // 参数
    hasFpu  : Boolean = true,
    hasFdiv : Boolean = false,
    hasFpiu : Boolean = false
    )
  (implicit p: Parameters)
  extends ExecutionUnit( // 继承的参数
    readsFrf  = true,
    writesFrf = true,
    writesLlIrf = hasFpiu,
    writesIrf = false,
    numBypassStages = 0,
    dataWidth = p(tile.TileKey).core.fpu.get.fLen + 1,
    bypassable = false,
    hasFpu  = hasFpu,
    hasFdiv = hasFdiv,
    hasFpiu = hasFpiu) with tile.HasFPUParameters
{
    val out_str = // 输出拥有的功能
    BoomCoreStringPrefix("==ExeUnit==")
    (if (hasFpu)  BoomCoreStringPrefix("- FPU (Latency: " + dfmaLatency + ")") else "") +
    (if (hasFdiv) BoomCoreStringPrefix("- FDiv/FSqrt") else "") +
    (if (hasFpiu) BoomCoreStringPrefix("- FPIU (writes to Integer RF)") else "")

    
  val fdiv_busy = WireInit(false.B)
  val fpiu_busy = WireInit(false.B)

    
  // The Functional Units --------------------
  val fu_units = ArrayBuffer[FunctionalUnit]()

  io.fu_types := Mux(hasFpu.B, FU_FPU, 0.U) |
                 Mux(!fdiv_busy && hasFdiv.B, FU_FDV, 0.U) |
                 Mux(!fpiu_busy && hasFpiu.B, FU_F2I, 0.U)

    
  // FPU Unit -----------------------
  var fpu: FPUUnit = null
  val fpu_resp_val = WireInit(false.B)
  val fpu_resp_fflags = Wire(new ValidIO(new FFlagsResp()))
  fpu_resp_fflags.valid := false.B
  if (hasFpu) {
    fpu = Module(new FPUUnit())
    fpu.io.req.valid         := io.req.valid &&
                                (io.req.bits.uop.fu_code_is(FU_FPU) ||
                                io.req.bits.uop.fu_code_is(FU_F2I)) 
      // TODO move to using a separate unit （需要移动到独立的功能单元中）
    fpu.io.req.bits.uop      := io.req.bits.uop
    fpu.io.req.bits.rs1_data := io.req.bits.rs1_data
    fpu.io.req.bits.rs2_data := io.req.bits.rs2_data
    fpu.io.req.bits.rs3_data := io.req.bits.rs3_data
    fpu.io.req.bits.pred_data := false.B
    fpu.io.req.bits.kill     := io.req.bits.kill
    fpu.io.fcsr_rm           := io.fcsr_rm
    fpu.io.brupdate          := io.brupdate
    fpu.io.resp.ready        := DontCare
    fpu_resp_val             := fpu.io.resp.valid
    fpu_resp_fflags          := fpu.io.resp.bits.fflags

    fu_units += fpu
  }

    
  // FDiv/FSqrt Unit -----------------------
  var fdivsqrt: FDivSqrtUnit = null
  val fdiv_resp_fflags = Wire(new ValidIO(new FFlagsResp()))
  fdiv_resp_fflags := DontCare
  fdiv_resp_fflags.valid := false.B
  if (hasFdiv) {
    fdivsqrt = Module(new FDivSqrtUnit())
    fdivsqrt.io.req.valid         := io.req.valid && io.req.bits.uop.fu_code_is(FU_FDV)
    fdivsqrt.io.req.bits.uop      := io.req.bits.uop
    fdivsqrt.io.req.bits.rs1_data := io.req.bits.rs1_data
    fdivsqrt.io.req.bits.rs2_data := io.req.bits.rs2_data
    fdivsqrt.io.req.bits.rs3_data := DontCare
    fdivsqrt.io.req.bits.pred_data := false.B
    fdivsqrt.io.req.bits.kill     := io.req.bits.kill
    fdivsqrt.io.fcsr_rm           := io.fcsr_rm
    fdivsqrt.io.brupdate          := io.brupdate

      
    // share write port with the pipelined units （与流水线化单元共享写端口）
    fdivsqrt.io.resp.ready := !(fu_units.map(_.io.resp.valid).reduce(_|_)) // TODO PERF will get blocked by fpiu.
      // 仅当其他功能单元valid都为0时才ready
    
    fdiv_busy := !fdivsqrt.io.req.ready || (io.req.valid &&
                                            io.req.bits.uop.fu_code_is(FU_FDV))
    
    fdiv_resp_fflags := fdivsqrt.io.resp.bits.fflags
    
    fu_units += fdivsqrt
  }

    
  // Outputs (Write Port #0)  ---------------

  io.fresp.valid       := fu_units.map(_.io.resp.valid).reduce(_|_) &&
                          !(fpu.io.resp.valid && fpu.io.resp.bits.uop.fu_code_is(FU_F2I))
  io.fresp.bits.uop    := PriorityMux(fu_units.map(f => (f.io.resp.valid,
                                                         f.io.resp.bits.uop)))
  io.fresp.bits.data:= PriorityMux(fu_units.map(f =>  
                                                (f.io.resp.valid,f.io.resp.bits.data)))
  io.fresp.bits.fflags := Mux(fpu_resp_val, fpu_resp_fflags, fdiv_resp_fflags)

    
  // Outputs (Write Port #1) -- FpToInt Queuing Unit -----------------------
  // 输出（写端口#1） FP转int队列单元

  if (hasFpiu) {
    // TODO instantiate our own fpiu; and remove it from fpu.scala.
    // buffer up results since we share write-port on integer regfile.
    // 还需要例化自己的fpiu，将其从fpu.scala中移除。需要将结果缓存因为共享整数寄存器堆写端口。
    val queue = Module(new BranchKillableQueue(new ExeUnitResp(dataWidth),
      entries = dfmaLatency + 3)) // TODO being overly conservative
    queue.io.enq.valid       := (fpu.io.resp.valid &&
                                 fpu.io.resp.bits.uop.fu_code_is(FU_F2I) &&
                                 fpu.io.resp.bits.uop.uopc =/= uopSTA) 
      							// STA means store data gen for floating point
    queue.io.enq.bits.uop    := fpu.io.resp.bits.uop
    queue.io.enq.bits.data   := fpu.io.resp.bits.data
    queue.io.enq.bits.predicated := fpu.io.resp.bits.predicated
    queue.io.enq.bits.fflags := fpu.io.resp.bits.fflags
    queue.io.brupdate          := io.brupdate
    queue.io.flush           := io.req.bits.kill

    assert (queue.io.enq.ready) 
      // If this backs up, we've miscalculated the size of the queue.
    
      
    val fp_sdq = Module(new BranchKillableQueue(new ExeUnitResp(dataWidth),
      entries = 3)) // Lets us backpressure floating point store data
    fp_sdq.io.enq.valid      := io.req.valid && io.req.bits.uop.uopc === uopSTA && !IsKilledByBranch(io.brupdate, io.req.bits.uop)
    fp_sdq.io.enq.bits.uop   := io.req.bits.uop
    fp_sdq.io.enq.bits.data  := ieee(io.req.bits.rs2_data)
    fp_sdq.io.enq.bits.predicated := false.B
    fp_sdq.io.enq.bits.fflags := DontCare
    fp_sdq.io.brupdate         := io.brupdate
    fp_sdq.io.flush          := io.req.bits.kill
    
    assert(!(fp_sdq.io.enq.valid && !fp_sdq.io.enq.ready))
    
      
    val resp_arb = Module(new Arbiter(new ExeUnitResp(dataWidth), 2))
    resp_arb.io.in(0) <> queue.io.deq
    resp_arb.io.in(1) <> fp_sdq.io.deq
    io.ll_iresp       <> resp_arb.io.out
    
    fpiu_busy := !(queue.io.empty && fp_sdq.io.empty)
  }

    
  override def toString: String = out_str.toString
}
```
