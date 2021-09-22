# issue-unit-age-ordered
顺序发射单元。

对于每个新派遣的指令，总是放到队列中尽可能靠后的位置（即“顺序”）。而在每个周期中，这些指令都会向前移动（即“紧密”）。 这样就使得最早的指令拥有最高的发射优先级。 这种设计代码显然更复杂，面积和功耗较大，但性能更好。

在IssueUnit抽象类的基础上实现了以下功能：

* 计算出发射后发射槽和派遣过来的每一条指令需要的移位数，然后进行移位。
* 移位完成后给派遣级正确的ready信号，并实现发射逻辑。

```scala
package boom.exu
import chisel3._
import chisel3.util.{log2Ceil, PopCount}
import freechips.rocketchip.config.Parameters
import freechips.rocketchip.util.Str
import FUConstants._
import boom.common._
```



### IssueUnitCollapsing 类

紧密（顺序）发射单元。由IssueUnit 抽象类继承。

```scala
/**
 * Specific type of issue unit
 *
 * @param params issue queue params
 * @param numWakeupPorts number of wakeup ports for the issue queue
 */
  class IssueUnitCollapsing(
    params: IssueParams,
    numWakeupPorts: Int)
    (implicit p: Parameters)
  extends IssueUnit(params.numEntries, params.issueWidth, numWakeupPorts, params.iqType, params.dispatchWidth)
  {
      
      
    //-------------------------------------------------------------
    // Figure out how much to shift entries by  计算移位数

  val maxShift = dispatchWidth
    // 最大移位数，等于派遣宽度（只要下一次派遣不会溢出即可，总是移位到紧密状态反而会浪费性能？）
  val vacants = issue_slots.map(s => !(s.valid)) ++ io.dis_uops.map(_.valid).map(!_.asBool)
    // 指示发射槽中和派遣来的各指令位置是否为空。
    // “++”表示连接；当该位置未valid时为空。
  val shamts_oh = Array.fill(numIssueSlots+dispatchWidth) {Wire(UInt(width=maxShift.W))}
    // track how many to shift up this entry by by counting previous vacant spots
    // 根据之前的空位，跟踪每个条目需要移位的数量。
    // 为每一个发射槽和派遣端口中的位置提供一个独热码形式的移位数。
      
    // 移位数使用独热码的原因：
    // 容易判断是否已移满（只看最高位是否为1）、“+1”的实现只需要简单的左移、比较相同时只需要1位
      
      
  // 定义计数函数，更新当前遍历到的移位数的值
  def SaturatingCounterOH(count_oh:UInt, inc: Bool, max: Int): UInt = {
      
     val next = Wire(UInt(width=max.W))
     next := count_oh
      
     when (count_oh === 0.U && inc) {	// 前一个位置不用移位，但它是空的，则当前位置需要移1位
       next := 1.U     
     .elsewhen (!count_oh(max-1) && inc) {	// 前一个位置需要移位，但没移到最大上限，且是空的，
       next := (count_oh << 1.U)			//  则当前位置需要移位，且位数比它还要多一（独热码形式左移即+1）
     }
         
     next
  }
      
  shamts_oh(0) := 0.U	// 第一个发射槽（最接近发射的那个）显然不用移位
      
  for (i <- 1 until numIssueSlots + dispatchWidth) {	// 遍历剩余发射槽中和派遣进来的所有指令（的移位数）
    shamts_oh(i) := SaturatingCounterOH(shamts_oh(i-1), vacants(i-1), maxShift) // 用上面的函数更新移位数
  }
      
      
      
  //-------------------------------------------------------------
  // 移位逻辑

  // which entries' uops will still be next cycle? (not being issued and vacated)
  // 哪些条目的微指令下周期还在？（不被发射而腾出位置）
      
  val will_be_valid = (0 until numIssueSlots).map(i => issue_slots(i).will_be_valid) ++
                      (0 until dispatchWidth).map(i => io.dis_uops(i).valid &&
                                                        !dis_uops(i).exception &&
                                                        !dis_uops(i).is_fence &&
                                                        !dis_uops(i).is_fencei)
  val uops = issue_slots.map(s=>s.out_uop) ++ dis_uops.map(s=>s)
      // 发射槽+派遣来的指令的 将要valid的信号 和 微指令
      
  for (i <- 0 until numIssueSlots) {	// 遍历每个发射槽
      
    issue_slots(i).in_uop.valid := false.B		// 先不向发射槽中输入微指令
    issue_slots(i).in_uop.bits  := uops(i+1)	// 但是仍然在输入端口中进行移位（移一位，只发射一条？）
      
    for (j <- 1 to maxShift by 1) {		// 对当前发射槽，看看后面第j个槽是否刚好要移j位
        
      when (shamts_oh(i+j) === (1 << (j-1)).U) {
        issue_slots(i).in_uop.valid := will_be_valid(i+j)
        issue_slots(i).in_uop.bits  := uops(i+j)
      }
        
    }
      
    issue_slots(i).clear        := shamts_oh(i) =/= 0.U	// 如果这个发射槽已经发生移位，则移位后清空
      
  }
      
      
      
  //-------------------------------------------------------------
  // Dispatch/Entry Logic	派遣/条目逻辑
  // did we find a spot to slide the new dispatched uops into?
  // 查看是否找到了新派遣指令能去的地方（下个周期不会被占据，或者这个周期清空，并且没有输入到这个槽的valid指令）

  val will_be_available = (0 until numIssueSlots).map(i =>
                            (!issue_slots(i).will_be_valid || issue_slots(i).clear) && 
                             !(issue_slots(i).in_uop.valid))
      // 可用信号。 当 不将valid 或 已清空，且 未valid， 则认为将可用。
      
  val num_available = PopCount(will_be_available)	// 对上面信号计数
      
  for (w <- 0 until dispatchWidth) {	// 给派遣级的ready信号数不能超过可用数
    io.dis_uops(w).ready := RegNext(num_available > w.U)
  }
      
      
      
  //-------------------------------------------------------------
  // Issue Select Logic
  // 发射选择逻辑

  // set default	初始化，同乱序的对应部分
  for (w <- 0 until issueWidth) {
    io.iss_valids(w) := false.B
    io.iss_uops(w)   := NullMicroOp
    // unsure if this is overkill
    io.iss_uops(w).prs1 := 0.U
    io.iss_uops(w).prs2 := 0.U
    io.iss_uops(w).prs3 := 0.U
    io.iss_uops(w).lrs1_rtype := RT_X
    io.iss_uops(w).lrs2_rtype := RT_X
  }

      
  val requests = issue_slots.map(s => s.request)	// 发射槽输出的请求
  val port_issued = Array.fill(issueWidth){Bool()}	// 记录已经发射的数量，用下面的循环初始化为0
  for (w <- 0 until issueWidth) {
    port_issued(w) = false.B
  }

      
  for (i <- 0 until numIssueSlots) {	// 遍历每个发射槽
    issue_slots(i).grant := false.B	// 同意发射信号，默认拒绝
    var uop_issued = false.B	// 变量，记录发射情况

    for (w <- 0 until issueWidth) {	// issueWidth次发射尝试
        
      val can_allocate = (issue_slots(i).uop.fu_code & io.fu_types(w)) =/= 0.U
    
      when (requests(i) && !uop_issued && can_allocate && !port_issued(w)) {
          // 发射槽有请求，未发射，能分配，且发射带宽足够时
        issue_slots(i).grant := true.B			// 同意发射
        io.iss_valids(w) := true.B				// 发射valid
        io.iss_uops(w) := issue_slots(i).uop	// 发射uop
      }
        
      val was_port_issued_yet = port_issued(w)	// 临时保存这次发射尝试的结果
        // 能发射时应该是0，因为下个周期时port_issued(w)才会变成1
        
      port_issued(w) = (requests(i) && !uop_issued && can_allocate) | port_issued(w)
		// 第一次有请求，之前未发射，能分配时置为1；如果上一个周期能发射，那么后面的周期也都能发射        
      uop_issued = (requests(i) && can_allocate && !was_port_issued_yet) | uop_issued
        // 第一次出现有请求，能分配，并且发射成功不超过一个周期你，则置1；如果上一个周期为1，那么后面也都是1
    }
      
  }     
}
```