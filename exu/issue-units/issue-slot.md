# issue-slot

（指令）发射槽。发射单元中发射队列的基本组成单元。

一个发射槽的示意图如下：

<img src=".\IssueSlot.jpg" alt="IssueSlot" style="zoom: 67%;" />

微指令被派遣到发射队列中，它们将等待所有操作数准备就绪（“p”代表存在位，它表示操作数何时在寄存器堆中可用）。一旦准备就绪，发射槽将发出request信号，并等待发射。

每个发射选择逻辑端口都是一个静态的优先编码器，用于选取发射队列中的第一个valid的微指令。每个端口（连接到某个执行单元）将只调度其可以处理的微指令（例如，浮点指令将只调度到拥有浮点单元的端口）。这将为端口创建级联的优先级编码器，这些端口可以彼此调度相同的微指令。

如果检测到某个功能单元（发射槽中微指令需要的功能单元）不可用，发射槽将取消valid信号，并且不会向其该功能单元发出指令（例如，尝试发射到正在计算的一个非流水线化除法器）。

```scala
// Note: stores (and AMOs) are "broken down" into 2 uops, but stored within a single issue-slot.
// TODO XXX make a separate issueSlot for MemoryIssueSlots, and only they break apart stores.
// TODO Disable ldspec for FP queue.
// 注意：储存和AMO指令被视作两条微指令，但是会保存在同一个发射槽中
// 还未实现：为访存发射槽实现一个独立的发射槽，只有这些槽能够分解储存操作；为浮点发射队列禁用ldspec。
package boom.exu
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import boom.common._
import boom.util._
import FUConstants._
```



### IssueSlotIO 类

发射槽的IO。

核心端口：输出槽中指令的valid状态，以及输出发射请求。输入发射同意信号。

```scala
/**
 * IO bundle to interact with Issue slot
 *
 * @param numWakeupPorts number of wakeup ports for the slot
 */
  class IssueSlotIO(val numWakeupPorts: Int)(implicit p: Parameters) extends BoomBundle
{
    
    val valid         = Output(Bool())
    val will_be_valid = Output(Bool()) // TODO code review, do we need this signal so explicitely?
    	// 【输出】有效信号。第二个"将要有效"的信号计划优化掉。
    val request       = Output(Bool())
    val request_hp    = Output(Bool())
    	// 【输出】请求信号
    val grant         = Input(Bool())
    	// 【输入】同意信号

  val brupdate        = Input(new BrUpdateInfo())
  val kill          = Input(Bool()) // pipeline flush
  val clear         = Input(Bool()) // entry being moved elsewhere (not mutually exclusive with grant)
  val ldspec_miss   = Input(Bool()) // Previous cycle's speculative load wakeup was mispredicted.
    // 【输入】分支更新信号、流水线清除信号、清除信号（转移到其他地方的条目，与同意不排斥）、上一个周期推测的加载唤醒的误判信号

  val wakeup_ports  = Flipped(Vec(numWakeupPorts, Valid(new IqWakeup(maxPregSz))))
  val pred_wakeup_port = Flipped(Valid(UInt(log2Ceil(ftqSz).W)))
  val spec_ld_wakeup = Flipped(Vec(memWidth, Valid(UInt(width=maxPregSz.W))))
    // 【输入】唤醒信息，关于唤醒的相关内容见本文最后
    
    // 和微指令相关的IO
  val in_uop        = Flipped(Valid(new MicroOp())) 
    // if valid, this WILL overwrite an entry!
    // 【输入】如果valid，就覆盖当前发射槽中的微指令
  val out_uop   = Output(new MicroOp()) 
    // the updated slot uop; will be shifted upwards in a collasping queue.
    // 【输出】发射槽中已更新好的微指令，用于实现微指令在队列中的移动
  val uop           = Output(new MicroOp()) 
    // the current Slot's uop. Sent down the pipeline when issued.
    // 【输出】发射槽当前的微指令。发射时发送到流水线中

  val debug = {	// 【debug】
    val result = new Bundle {
      val p1 = Bool()
      val p2 = Bool()
      val p3 = Bool()
      val ppred = Bool()
      val state = UInt(width=2.W)
    }
    Output(result)
  }
}
```



### IssueSlot 类

单个发射槽。在发射队列中持有一条微指令。

```scala
/**
 * Single issue slot. Holds a uop within the issue queue
 *
 * @param numWakeupPorts number of wakeup ports
 */
  class IssueSlot(val numWakeupPorts: Int)(implicit p: Parameters)
    extends BoomModule
    with IssueUnitConstants
  {
    val io = IO(new IssueSlotIO(numWakeupPorts))

      
      
  // 定义方法，返回发射槽当前状态，见issue-unit.scala中的IssueUnitConstants特质
  // slot invalid?
  // slot is valid, holding 1 uop
  // slot is valid, holds 2 uops (like a store)
  def is_invalid = state === s_invalid
  def is_valid = state =/= s_invalid

  
      
  // 线网，用于实现队列的移动。其值都是对应的当前值（state以及slot_uop中对应成员）寄存器输出经过逻辑选择后的值。
  // 这四个线网以及next_brmask最后都作为输出out_uop的一部分。
  val next_state      = Wire(UInt()) 
      // the next state of this slot (which might then get moved to a new slot)
  val next_uopc       = Wire(UInt()) 
      // the next uopc of this slot (which might then get moved to a new slot)
  val next_lrs1_rtype = Wire(UInt()) 
      // the next reg type of this slot (which might then get moved to a new slot)
  val next_lrs2_rtype = Wire(UInt()) 
      // the next reg type of this slot (which might then get moved to a new slot)

      
  // 内部寄存器
  val state = RegInit(s_invalid)
  val p1    = RegInit(false.B)
  val p2    = RegInit(false.B)
  val p3    = RegInit(false.B)
  val ppred = RegInit(false.B)

      
  // 两个物理寄存器的污染信号。当被一个推测的加载唤醒时即被污染。
  // 污染持续一个周期，而加载miss会在下一个周期出现，所以如果污染位是1，设置为0.（？）
  // 输出的uop与out_uop中，使用的都是p1/2_poisoned，原因是（？）
  // Poison if woken up by speculative load.
  // Poison lasts 1 cycle (as ldMiss will come on the next cycle).
  // SO if poisoned is true, set it to false!
  val p1_poisoned = RegInit(false.B)
  val p2_poisoned = RegInit(false.B)
  p1_poisoned := false.B
  p2_poisoned := false.B
  val next_p1_poisoned = Mux(io.in_uop.valid, io.in_uop.bits.iw_p1_poisoned, p1_poisoned)
  val next_p2_poisoned = Mux(io.in_uop.valid, io.in_uop.bits.iw_p2_poisoned, p2_poisoned)

      
  // 发射槽中的微指令
  val slot_uop = RegInit(NullMicroOp)
  val next_uop = Mux(io.in_uop.valid, io.in_uop.bits, slot_uop)
      
      
      
  //-----------------------------------------------------------------------------
  // next slot state computation
  // compute the next state for THIS entry slot (in a collasping queue, the
  // current uop may get moved elsewhere, and a new uop can enter
  // 更新此发射槽的当前状态

  when (io.kill) {
    state := s_invalid
  } .elsewhen (io.in_uop.valid) {
    state := io.in_uop.bits.iw_state
  } .elsewhen (io.clear) {
    state := s_invalid
  } .otherwise {
    state := next_state
  }

      
      
  //-----------------------------------------------------------------------------
  // "update" state
  // compute the next state for the micro-op in this slot. This micro-op may
  // be moved elsewhere, so the "next_state" travels with it.
  // 计算此发射槽的下一个状态
      
  // defaults  默认情况，与当前状态相同
  next_state := state
  next_uopc := slot_uop.uopc
  next_lrs1_rtype := slot_uop.lrs1_rtype
  next_lrs2_rtype := slot_uop.lrs2_rtype

  when (io.kill) {	// 清除的情况，将置为invalid
    next_state := s_invalid
  } 
  .elsewhen ((io.grant && (state === s_valid_1)) ||				// 同意发射且有一条valid微指令 或
    (io.grant && (state === s_valid_2) && p1 && p2 && ppred)) {	// 同意发射且有一条可分valid微指令且各寄存器就绪
    // try to issue this uop.			
    when (!(io.ldspec_miss && (p1_poisoned || p2_poisoned))) {	// ldspec为0 或 p1p2都未被污染时，尝试发射
      next_state := s_invalid	// 发射后置为invalid
    }
  } 
  .elsewhen (io.grant && (state === s_valid_2)) {	// 同意发射且有一条可分valid微指令，但有寄存器未就绪
    when (!(io.ldspec_miss && (p1_poisoned || p2_poisoned))) {	// ldspec为0 或 p1p2都未被污染时，
      next_state := s_valid_1	// 暂时不发射
      when (p1) {	// p1就绪
        slot_uop.uopc := uopSTD	// STD：store data generation
        next_uopc := uopSTD
        slot_uop.lrs1_rtype := RT_X
        next_lrs1_rtype := RT_X
      } .otherwise {	// p2就绪？
        slot_uop.lrs2_rtype := RT_X
        next_lrs2_rtype := RT_X
      }
    }
  }

  // 进来的微指令valid时覆盖发射槽中的微指令，若原微指令还是valid（未发射）则发出警告
  when (io.in_uop.valid) {
    slot_uop := io.in_uop.bits
    assert (is_invalid || io.clear || io.kill, "trying to overwrite a valid issue slot.")
  }

      
  // Wakeup Compare Logic 唤醒的比较逻辑

  // 用于对应信号在队列中的移动
  // these signals are the "next_p*" for the current slot's micro-op.
  // they are important for shifting the current slot_uop up to an other entry.
  val next_p1 = WireInit(p1)
  val next_p2 = WireInit(p2)
  val next_p3 = WireInit(p3)
  val next_ppred = WireInit(ppred)

  when (io.in_uop.valid) {	// 用进来的uop更新内部的寄存器（储存微指令寄存器状态）
    p1 := !(io.in_uop.bits.prs1_busy)
    p2 := !(io.in_uop.bits.prs2_busy)
    p3 := !(io.in_uop.bits.prs3_busy)
    ppred := !(io.in_uop.bits.ppred_busy)
  }

  // 检查是否有将x0标记为被污染的指令
  when (io.ldspec_miss && next_p1_poisoned) {
    assert(next_uop.prs1 =/= 0.U, "Poison bit can't be set for prs1=x0!")
    p1 := false.B
  }
  when (io.ldspec_miss && next_p2_poisoned) {
    assert(next_uop.prs2 =/= 0.U, "Poison bit can't be set for prs2=x0!")
    p2 := false.B
  }

  for (i <- 0 until numWakeupPorts) {	// 遍历唤醒端口，将真正唤醒的寄存器标记为就绪
    when (io.wakeup_ports(i).valid &&
         (io.wakeup_ports(i).bits.pdst === next_uop.prs1)) {
      p1 := true.B
    }
    when (io.wakeup_ports(i).valid &&
         (io.wakeup_ports(i).bits.pdst === next_uop.prs2)) {
      p2 := true.B
    }
    when (io.wakeup_ports(i).valid &&
         (io.wakeup_ports(i).bits.pdst === next_uop.prs3)) {
      p3 := true.B
    }
  }
  when (io.pred_wakeup_port.valid && io.pred_wakeup_port.bits === next_uop.ppred) {
    ppred := true.B
  }

  // 检查是否有指令尝试推测性地唤醒x0
  for (w <- 0 until memWidth) {
    assert (!(io.spec_ld_wakeup(w).valid && io.spec_ld_wakeup(w).bits === 0.U),
      "Loads to x0 should never speculatively wakeup other instructions")
  }

  // TODO disable if FP IQ. 还未实现：为浮点队列禁用以下代码
  for (w <- 0 until memWidth) {
    when (io.spec_ld_wakeup(w).valid &&					// 对于推测唤醒的微指令（仅整数）
      io.spec_ld_wakeup(w).bits === next_uop.prs1 &&	
      next_uop.lrs1_rtype === RT_FIX) {
      p1 := true.B								// 置为就绪，并且标记为污染
      p1_poisoned := true.B
      assert (!next_p1_poisoned)
    }
    when (io.spec_ld_wakeup(w).valid &&
      io.spec_ld_wakeup(w).bits === next_uop.prs2 &&
      next_uop.lrs2_rtype === RT_FIX) {
      p2 := true.B
      p2_poisoned := true.B
      assert (!next_p2_poisoned)
    }
  }


  // Handle branch misspeculations 更新分支信息
  val next_br_mask = GetNewBrMask(io.brupdate, slot_uop)

      
  // was this micro-op killed by a branch? if yes, we can't let it be valid if
  // we compact it into an other entry
  when (IsKilledByBranch(io.brupdate, slot_uop)) {
    next_state := s_invalid			// 被分支清除时强制invalid
  }

  when (!io.in_uop.valid) {
    slot_uop.br_mask := next_br_mask	// 没有新指令进来（队列没动）时，照常更新分支掩码
  }

      
      
  //-------------------------------------------------------------
  // Request Logic  请求逻辑
      
  // 有效且各寄存器就绪且未被清除时，输出的request信号置1    
  io.request := is_valid && p1 && p2 && p3 && ppred && !io.kill
  // 优先请求信号。（分支/跳转优先）
  val high_priority = slot_uop.is_br || slot_uop.is_jal || slot_uop.is_jalr
  io.request_hp := io.request && high_priority

  // 每个周期更新request信号
  when (state === s_valid_1) {
    io.request := p1 && p2 && p3 && ppred && !io.kill
  } .elsewhen (state === s_valid_2) {
    io.request := (p1 || p2) && ppred && !io.kill
  } .otherwise {
    io.request := false.B
  }

      
      
  // assign outputs 连接其他的输出
  io.valid := is_valid
    // 连接到发射槽当前持有的微指令相关信息。
  io.uop := slot_uop
  io.uop.iw_p1_poisoned := p1_poisoned
  io.uop.iw_p2_poisoned := p2_poisoned

  // micro-op will vacate due to grant.
  val may_vacate = io.grant && ((state === s_valid_1) || (state === s_valid_2) && p1 && p2 && ppred)
  val squash_grant = io.ldspec_miss && (p1_poisoned || p2_poisoned)
  io.will_be_valid := is_valid && !(may_vacate && !squash_grant)

    // 连接到给队列中下一个发射槽的微指令相关信息。
  io.out_uop            := slot_uop
  io.out_uop.iw_state   := next_state
  io.out_uop.uopc       := next_uopc
  io.out_uop.lrs1_rtype := next_lrs1_rtype
  io.out_uop.lrs2_rtype := next_lrs2_rtype
  io.out_uop.br_mask    := next_br_mask
  io.out_uop.prs1_busy  := !p1
  io.out_uop.prs2_busy  := !p2
  io.out_uop.prs3_busy  := !p3
  io.out_uop.ppred_busy := !ppred
  io.out_uop.iw_p1_poisoned := p1_poisoned
  io.out_uop.iw_p2_poisoned := p2_poisoned

  when (state === s_valid_2) {	// 对于持有一条可分微指令的情况
    when (p1 && p2 && ppred) {
      ; // send out the entire instruction as one uop  当寄存器都就绪，不处理，当做一条微指令送出
    } .elsewhen (p1 && ppred) {
      io.uop.uopc := slot_uop.uopc
      io.uop.lrs2_rtype := RT_X
    } .elsewhen (p2 && ppred) {
      io.uop.uopc := uopSTD
      io.uop.lrs1_rtype := RT_X
    }
  }

      
      
  // debug outputs 连接debug输出
  io.debug.p1 := p1
  io.debug.p2 := p2
  io.debug.p3 := p3
  io.debug.ppred := ppred
  io.debug.state := state
}
```



### Wake-Up（唤醒）相关

Boom有两种类型的唤醒——快唤醒和慢唤醒（也称长延迟唤醒）。

由于在ALU中的微指令可以通过旁路网络传输它们将要写回的数据，被发射的ALU微指令将在发射时向发射队列广播唤醒信息。

但是，浮点操作、加载和可变延迟操作不会通过旁路网络发送，因此唤醒信号将在写回阶段从寄存器文件端口发出。