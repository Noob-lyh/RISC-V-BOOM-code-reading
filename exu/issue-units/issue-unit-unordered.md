# issue-unit-unordered

乱序发射单元。

是MIPS R10k采取的方案。当指令被派遣时，会选择发射队列中最靠前的可用位置进行安排（valid优先而不是前面的优先，即“乱序”），而后续不会移动位置（即“静态”）。 这就意味着如果某条指令在派遣时“恰好”被安排到了较为靠后的位置，那它就需要等待很久（等到排在前面位置的能够发射的指令不足时）才能被发射。 这种实现方式比较简单，但性能不够高。

在IssueUnit抽象类的基础上实现了以下功能：

* 为每一个发射槽增加一个写使能的独热码信号，用于将派遣的指令安排到发射队列中。
* 实现高低两个优先级的发射尝试。

```scala
package boom.exu
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import freechips.rocketchip.util.Str
import FUConstants._
import boom.common._
```



### IssueUnitStatic 类

静态（乱序）发射单元。由IssueUnit 抽象类继承。

```scala
/**
 * Specific type of issue unit
 *
 * @param params issue queue params
 * @param numWakeupPorts number of wakeup ports for the issue queue
 */
  class IssueUnitStatic(
    params: IssueParams,
    numWakeupPorts: Int)
    (implicit p: Parameters)
    extends IssueUnit(params.numEntries, params.issueWidth, numWakeupPorts, params.iqType, params.dispatchWidth)
  {
      
      
    //-------------------------------------------------------------
    // Issue Table 发射表，在父类的基础上增加了写使能独热码信号

    // 为每个发射槽增加一串写使能的独热码信号，每一位代表那一条派遣的指令是否valid
  val entry_wen_oh = VecInit(Seq.fill(numIssueSlots){ Wire(Bits(dispatchWidth.W)) })
  for (i <- 0 until numIssueSlots) {	// 遍历每个发射槽的IO
    issue_slots(i).in_uop.valid := entry_wen_oh(i).orR 
    issue_slots(i).in_uop.bits  := Mux1H(entry_wen_oh(i), dis_uops) // 根据独热码选择填进发射槽的指令
    issue_slots(i).clear        := false.B
  }

      
      
  //-------------------------------------------------------------
  // Dispatch/Entry Logic	派遣/入口逻辑
  // find a slot to enter a new dispatched instruction  找到一个槽，输入一条新的派遣的指令

      
  val entry_wen_oh_array = Array.fill(numIssueSlots,dispatchWidth){false.B}
      // 储存所有写使能独热码的Array，初始化为全0
  var allocated = VecInit(Seq.fill(dispatchWidth){false.B}) 
      // did an instruction find an issue width?
      // 变量，派遣指令中已找到发射槽的指令对应的位置为1，初始化为0

      
  for (i <- 0 until numIssueSlots) { // 遍历每个发射槽
      
    // 变量  
    var next_allocated = Wire(Vec(dispatchWidth, Bool())) // 循环迭代到下一次时的allocated，其中的1应该越来越多
    var can_allocate = !(issue_slots(i).valid) // 仅当该发射槽不valid时才能分配

    for (w <- 0 until dispatchWidth) {	// 对第i个发射槽，遍历每条派遣的指令
      entry_wen_oh_array(i)(w) = can_allocate && !(allocated(w))  
      next_allocated(w) := can_allocate | allocated(w)
      can_allocate = can_allocate && allocated(w)       
      // 若一开始发射槽valid，即其中有微指令，则can_allocate将一直为0，独热码此行也将都为0，allocated保持原值
      // 若一开始发射槽不valid，即空，则遍历到第一条未找到发射槽的指令时，将独热码中对应位置改为1，更新next_allocated，
      //     并将can_allocate置0，进入上面的情况（即分配完成，不需要再分配了）
    }    
    allocated = next_allocated
  
  }
     
      
  // if we can find an issue slot, do we actually need it?
  // also, translate from Scala data structures to Chisel Vecs
  // 当找到发射槽时计算是否真的需要它，同时将Scala的数据转换为Chisel的Vecs
  for (i <- 0 until numIssueSlots) {	// 遍历每个发射槽
      
    val temp_uop_val = Wire(Vec(dispatchWidth, Bool()))	// 记录真正的独热码写使能信号

    for (w <- 0 until dispatchWidth) {	// 对第i个发射槽，遍历每条派遣的指令
      // TODO add ctrl bit for "allocates iss_slot"  未完成：对分配发射槽增加控制信号？
      temp_uop_val(w) := io.dis_uops(w).valid &&
                         !dis_uops(w).exception &&
                         !dis_uops(w).is_fence &&
                         !dis_uops(w).is_fencei &&
                         entry_wen_oh_array(i)(w)	// 除了可分配之外，还要考虑分配的微指令的相关信号
    }
    entry_wen_oh(i) := temp_uop_val.asUInt	// 类型转换
  }

  for (w <- 0 until dispatchWidth) {
    io.dis_uops(w).ready := allocated(w)	// 该位置的指令已分配后，对派遣器输出ready信号，表示可以接受新的指令
  }

      
      
  //-------------------------------------------------------------
  // Issue Select Logic  发射选择逻辑

  for (w <- 0 until issueWidth) {	// 遍历每个发射槽（中输出的部分端口），进行初始化
    io.iss_valids(w) := false.B
    io.iss_uops(w)   := NullMicroOp
    // unsure if this is overkill
    io.iss_uops(w).prs1 := 0.U
    io.iss_uops(w).prs2 := 0.U
    io.iss_uops(w).prs3 := 0.U
    io.iss_uops(w).lrs1_rtype := RT_X
    io.iss_uops(w).lrs2_rtype := RT_X
  }

      
  // TODO can we use flatten to get an array of bools on issue_slot(*).request? 未完成：使用flatten？
  // 将发射分为高低优先级
  val lo_request_not_satisfied = Array.fill(numIssueSlots){Bool()}
  val hi_request_not_satisfied = Array.fill(numIssueSlots){Bool()}

      
  for (i <- 0 until numIssueSlots) {  // 遍历每个发射槽，填写两个优先级的请求情况
    lo_request_not_satisfied(i) = issue_slots(i).request
    hi_request_not_satisfied(i) = issue_slots(i).request_hp
    issue_slots(i).grant := false.B // default 默认不同意发射
  }

      
  for (w <- 0 until issueWidth) {  // 遍历每次发射尝试
    
    var port_issued = false.B	// 变量，表示当前发射尝试是否完成

      
    // first look for high priority requests 先考虑高优先级的请求
    for (i <- 0 until numIssueSlots) {	// 遍历每个发射槽
      
      val can_allocate = (issue_slots(i).uop.fu_code & io.fu_types(w)) =/= 0.U
        // 当功能单元有效时为1
    
      when (hi_request_not_satisfied(i) && can_allocate && !port_issued) {
          // （高优先级）请求还未满足 且 可以分配 且 此次发射还未完成时，发出同意信号，并且输出该位置valid的微指令
        issue_slots(i).grant := true.B
        io.iss_valids(w)     := true.B
        io.iss_uops(w)       := issue_slots(i).uop
      }
    
      val port_already_in_use     = port_issued	// 记录发射情况
        
      port_issued                 = (hi_request_not_satisfied(i) && can_allocate) | port_issued
      // deassert lo_request if hi_request is 1. 更新发射情况，防止低优先级中出现错误
        
      lo_request_not_satisfied(i) = (lo_request_not_satisfied(i) && !hi_request_not_satisfied(i))
      // if request is 0, stay 0. only stay 1 if request is true and can't allocate
      // 只保留高优先级中0位置（即下面可分配但未能发射的情况）对应的低优先级请求
        
      hi_request_not_satisfied(i) = (hi_request_not_satisfied(i) && (!can_allocate || port_already_in_use))
      // 可分配但未能发射时，高优先级对应位置改为0
    }
    
      
    // now look for low priority requests 再考虑低优先级的请求
    for (i <- 0 until numIssueSlots) {
    
      val can_allocate = (issue_slots(i).uop.fu_code & io.fu_types(w)) =/= 0.U
    
      when (lo_request_not_satisfied(i) && can_allocate && !port_issued) {
        issue_slots(i).grant := true.B
        io.iss_valids(w)     := true.B
        io.iss_uops(w)       := issue_slots(i).uop
      }
    
      val port_already_in_use     = port_issued
        
      port_issued                 = (lo_request_not_satisfied(i) && can_allocate) | port_issued
      // if request is 0, stay 0. only stay 1 if request is true and can't allocate or port already in use
      // 以上逻辑和高优先级的完全相同
      
      lo_request_not_satisfied(i) = (lo_request_not_satisfied(i) && (!can_allocate || port_already_in_use))
      // 可分配但未能发射时，低优先级对应位置改为0
        
    }
  }
}
```