# issue-unit

定义了IssueUnit抽象类，issue-unit-age-ordered.scala和issue-unit-unordered.scala中定义的顺序/乱序发射单元类都由它派生。

**发射队列**保存着已派遣但又未执行的微指令。当微指令的所有操作数都就绪时，保存该微指令的发射槽将把request位置为1。然后发射选择逻辑将选择一个发射槽进行发射。一条微指令被发射时，它将在发射队列中被删除，以腾出空间给更多派遣的指令。

BOOM使用彼此不同的一组发射队列，对应不同的指令类型（整数、浮点数、访存）。

尽管还没有实现，未来的设计可能会为了提高性能而发射推测状态的微指令（例如，推测一条加载指令会命中缓存，所以在假设加载的数据会通过旁路从而可用的情况下，选择发射有数据依赖的指令）。在这种情况下，发射队列将不能删除已发射的处于推测状态的微指令，直到其状态被确定。如果已发射的微指令的推测出错，所有从这个推测窗口发射的微指令都必须被清除，并从发射队列中重试。另外更多先进的技术也已经可用。

```scala
package boom.exu
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import freechips.rocketchip.util.{Str}
import boom.common._
import boom.exu.FUConstants._
import boom.util.{BoolToChar}
```



### IssueParams case类

用于配置的一组发射参数。

```scala
/**
 * Class used for configurations
 *
 * @param issueWidth amount of things that can be issued
 * @param numEntries size of issue queue
 * @param iqType type of issue queue
 */
 case class IssueParams(
    dispatchWidth: Int = 1,	// 单次派遣的指令数
    issueWidth: Int = 1,	// 单次可发射的指令数
    numEntries: Int = 8,	// 发射队列的大小
    iqType: BigInt		// 发射队列的种类
 )
```



### IssueUnitConstants 特质

发射单元常量，包括用于了解微指令状态的常量s_invalid，其可能的值与对应的意义如下：

* invalid : 发射槽没有valid的微指令。
* s_valid1 : 发射槽有一条valid的微指令。
* s_valid2 : 发射槽有一条store-like（？）的微指令，它可能被分成两条微指令。（在issue-slot.scala中有涉及这种状况的处理）

```scala
/**
 * Constants for knowing about the status of a MicroOp
 */
 trait IssueUnitConstants
 {
    // invalid  : slot holds no valid uop.
    // s_valid_1: slot holds a valid uop.
    // s_valid_2: slot holds a store-like uop that may be broken into two micro-ops.
    val s_invalid :: s_valid_1 :: s_valid_2 :: Nil = Enum(3)
 }
```



### IqWakeup 类

给发射队列的唤醒信息。

包括广播队列被唤醒的物理寄存器地址pdst，以及这个物理寄存器是否被污染（即被一个推测的发射唤醒）的指示信号poisoned。

```scala
/**
 * What physical register is broadcasting its wakeup?
 * Is the physical register poisoned (aka, was it woken up by a speculative issue)?
 *
 * @param pregSz size of physical destination register
 */
 class IqWakeup(val pregSz: Int) extends Bundle
 {
    val pdst = UInt(width=pregSz.W)
    val poisoned = Bool()
 }
```



### IssueUnitIO 类

发射单元的IO端口。

```scala
/**
 * IO bundle to interact with the issue unit
 *
 * @param issueWidth amount of operations that can be issued at once
 * @param numWakeupPorts number of wakeup ports for issue unit
 */
  class IssueUnitIO(
    val issueWidth: Int,		// 一次可以发射的微指令数量
    val numWakeupPorts: Int,	// 唤醒端口数量
    val dispatchWidth: Int)
    (implicit p: Parameters) extends BoomBundle
  {
    val dis_uops         = Vec(dispatchWidth, Flipped(Decoupled(new MicroOp)))
      // 来自派遣器的输入，见dispatch.scala

  val iss_valids       = Output(Vec(issueWidth, Bool()))
  val iss_uops         = Output(Vec(issueWidth, new MicroOp()))
      // 输出发射的微指令及其valid信号
      
  val wakeup_ports     = Flipped(Vec(numWakeupPorts, Valid(new IqWakeup(maxPregSz))))
  val pred_wakeup_port = Flipped(Valid(UInt(log2Ceil(ftqSz).W)))
      // 输入唤醒和pred唤醒的信号

  val spec_ld_wakeup   = Flipped(Vec(memWidth, Valid(UInt(width=maxPregSz.W))))
      // 输入推测加载的唤醒信号（？）

  // tell the issue unit what each execution pipeline has in terms of functional units
  val fu_types         = Input(Vec(issueWidth, Bits(width=FUC_SZ.W)))
	// 输入，告诉发射单元（实际上是每个发射位？）每个执行流水线都有什么功能单元
      
  val brupdate         = Input(new BrUpdateInfo())
  val flush_pipeline   = Input(Bool())
  val ld_miss          = Input(Bool())
      // 分支更新信息，清除流水线信号，加载信号

  val event_empty      = Output(Bool()) // used by HPM events; is the issue unit empty?
      // 指示发射队列是否为空。HPM？

  val tsc_reg          = Input(UInt(width=xLen.W))
      // ？
}
```



### IssueUnit 抽象类

抽象的顶层发射单元。

仅实现以下功能：

* 将派遣过来的指令保存在dis_uops并做基本的处理。
* 定义所有发射槽与其IO端口，并连接指示信号。
* 定义队列空闲信号、valid计数和警告、打印字符串等基本功能

子类有乱序和顺序两种，分别在issue-unit-unordered.scala和issue-unit-age-ordered.scala中定义。

```scala
/**
 * Abstract top level issue unit
 *
 * @param numIssueSlots depth of issue queue
 * @param issueWidth amoutn of operations that can be issued at once
 * @param numWakeupPorts number of wakeup ports for issue unit
 * @param iqType type of issue queue (mem, int, fp)
 */
abstract class IssueUnit(
    val numIssueSlots: Int,
    val issueWidth: Int,
    val numWakeupPorts: Int,
    val iqType: BigInt,
    val dispatchWidth: Int)
  (implicit p: Parameters)
  extends BoomModule
  with IssueUnitConstants
{
    val io = IO(new IssueUnitIO(issueWidth, numWakeupPorts, dispatchWidth))

    
    
  //-------------------------------------------------------------
  // Set up the dispatch uops
  // special case "storing" 2 uops within one issue slot.
  // 建立派遣的微指令。特例“storing”，一个发射槽中两条微指令。

  val dis_uops = Array.fill(dispatchWidth) {Wire(new MicroOp())}
  
    for (w <- 0 until dispatchWidth) {	// 遍历每一条派遣来的微指令
    dis_uops(w) := io.dis_uops(w).bits
    dis_uops(w).iw_p1_poisoned := false.B
    dis_uops(w).iw_p2_poisoned := false.B
    dis_uops(w).iw_state := s_valid_1
      // 用dis_uop保存输入的微指令，两个物理寄存器的状态置为未污染，发射槽的状态置为“有一条valid的微指令”

      
        
    if (iqType == IQT_MEM.litValue || iqType == IQT_INT.litValue) {
      
      // For StoreAddrGen for Int, or AMOAddrGen, we go to addr gen state
      // 对于整数储存生成地址，或AMO生成地址，进入地址生成状态
      when ((io.dis_uops(w).bits.uopc === uopSTA && io.dis_uops(w).bits.lrs2_rtype === RT_FIX) ||
             io.dis_uops(w).bits.uopc === uopAMO_AG) {
        dis_uops(w).iw_state := s_valid_2
        // For store addr gen for FP, rs2 is the FP register, and we don't wait for that here
        // 对于浮点储存生成地址，rs2就是浮点寄存器，所以不用等待
      } 
      
      .elsewhen (io.dis_uops(w).bits.uopc === uopSTA && io.dis_uops(w).bits.lrs2_rtype =/= RT_FIX) {
        dis_uops(w).lrs2_rtype := RT_X
        dis_uops(w).prs2_busy  := false.B
      }
      
      dis_uops(w).prs3_busy := false.B
    } 
        
    else if (iqType == IQT_FP.litValue) {
      
      // FP "StoreAddrGen" is really storeDataGen, and rs1 is the integer address register
      // 浮点的“储存生成地址”是真正的储存数据生成，rs1是整数的地址寄存器
      when (io.dis_uops(w).bits.uopc === uopSTA) {
        dis_uops(w).lrs1_rtype := RT_X
        dis_uops(w).prs1_busy  := false.B
      
      }
    }
    
        
    if (iqType != IQT_INT.litValue) {
        // 对于不是派遣到整数发射队列的微指令，若ppred繁忙且valid时警告
      assert(!(io.dis_uops(w).bits.ppred_busy && io.dis_uops(w).valid))
      dis_uops(w).ppred_busy := false.B
    }
  }

    
    
  //-------------------------------------------------------------
  // Issue Table    发射表

  val slots = for (i <- 0 until numIssueSlots) yield { val slot = Module(new IssueSlot(numWakeupPorts)); slot }
    // 发射队列的本体：numIssueSlots个发射槽（IssueSlot类），每个都带有numWakeupPorts个唤醒端口
  val issue_slots = VecInit(slots.map(_.io))
    // 每个发射槽的IO

  for (i <- 0 until numIssueSlots) {	// 遍历每个发射槽的IO
    issue_slots(i).wakeup_ports     := io.wakeup_ports
    issue_slots(i).pred_wakeup_port := io.pred_wakeup_port
    issue_slots(i).spec_ld_wakeup   := io.spec_ld_wakeup
      // 连接唤醒信号
    issue_slots(i).ldspec_miss      := io.ld_miss
    issue_slots(i).brupdate         := io.brupdate
    issue_slots(i).kill             := io.flush_pipeline
      // 连接加载缺失、分支更新信息、清除流水线的信号
  }

  io.event_empty := !(issue_slots.map(s => s.valid).reduce(_|_))
    // 当每个发射槽输出的valid位都为1时，发射队列空闲，event_empty置1

  val count = PopCount(slots.map(_.io.valid)) // 所有发射槽的valid计数
  dontTouch(count)	// 防止被优化掉

    
    
  //-------------------------------------------------------------
  // 当 发射队列中给总发射同意信号数量 多于 单次发射的最多指令数 时，发出警告
    
  assert (PopCount(issue_slots.map(s => s.grant)) <= issueWidth.U, "[issue] window giving out too many grants.")


    
  //-------------------------------------------------------------
  // 定义方法，返回发射队列类型（字符串）
    
  def getType: String =
    if (iqType == IQT_INT.litValue) "int"
    else if (iqType == IQT_MEM.litValue) "mem"
    else if (iqType == IQT_FP.litValue) " fp"
    else "unknown"
}
```