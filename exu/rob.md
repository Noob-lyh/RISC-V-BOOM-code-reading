# rob

**重排序缓存器(Reorder Buffer, ROB)**跟踪在流水线中运行的微指令的状态。ROB的作用就是给程序员一种“他的程序是按顺序执行的”的错觉。指令在被译码和重命名后，将被派遣到ROB和**发射队列**并被标记为繁忙。指令完成执行后将告知ROB并且被重新标记为不繁忙。一旦ROB的“头”不再繁忙，就代表这条指令被提交了，它的体系结构状态现在是可见的。如果有一个异常发生，并且引发异常的指令在ROB的头部，流水线将被清除，并且在异常指令可见后不会出现任何体系结构的更改，然后ROB会重定向PC到相应的异常处理程序。

<img src=".\ROB.jpg" alt="ROB" style="zoom: 67%;" />

上图是一个三发射双宽度(two-wide)BOOM的ROB。派遣的微指令将被写在ROB的底部（或尾部，tail），而提交的微指令将在顶部，即ROB头部提交，然后更新重命名阶段的相关信息。完成执行的微指令（等待写回的微指令，wb uops）会清除他们的繁忙位。注意：派遣的微指令组会同时被写进同一个ROB的行，在内存中连续存放，使得可以用一个PC代表整个行。

ROB在概念上是一个循环的缓冲区，顺序地追踪所有运行中的指令。最老（最先进入流水线）的指令将由commit head指向，最新的指令将在rob tail处插入。

为了方便超标量派遣和提交，ROB被实现为一个有W栏(bank)的循环缓冲区，其中W是机器派遣和提交的宽度。（如上图中W=2）

派遣时，最多w条指令将从取指包裹（或取指数据包，Fetch Packet，见下面引用部分）被写到一个ROB的行中，其中每条指令都会被写到不同的栏（列）中。由于一个取指包裹中的指令组在内存中都是连续存放的，可以用一个PC来联系一个取指包裹（包裹内指令的位置提供了它自己PC的低位）。这也意味着分支将在ROB中留下气泡（空指令占的位置），这使得向ROB添加更多指令的开销变小，因为本来很大的开销被每个ROB行平摊了。

> 取指包裹(Fetch Packet)：前端返回的一个包，其中包含一组连续的指令，带有一个掩码，表示哪些指令有效，以及与指令提取和分支预测相关的其他元数据。Fetch PC将指向取指包裹中的第一条有效指令，因为它是前端用来获取取指包裹的PC。

```scala
// Bank the ROB, such that each "dispatch" group gets its own row of the ROB,
// and each instruction in the dispatch group goes to a different bank.
// We can compress out the PC by only saving the high-order bits!
// ROB以栏（bank）为单位储存，使得每个派遣组都有自己在ROB中的行，派遣组中每条指令都去往不同的栏。
// 我们可以只保存高位来压缩储存PC需要的空间。
//
// ASSUMPTIONS:
//    - dispatch groups are aligned to the PC.
// 假设：派遣组与PC对齐
//
// NOTES:
//    - Currently we do not compress out bubbles in the ROB.
//    - Exceptions are only taken when at the head of the commit bundle --
//      this helps deal with loads, stores, and refetch instructions.
// 注意：当前并不压缩ROB中的气泡，除非在提交端口的头部，这将有助于处理加载、储存、重取的指令。
// 
package boom.exu
import scala.math.ceil
import chisel3._
import chisel3.util._
import chisel3.experimental.chiselName
import freechips.rocketchip.config.Parameters
import freechips.rocketchip.util.Str
import boom.common._
import boom.util._
```



### RobIO 类

```scala
/**
 * IO bundle to interact with the ROB
 *
 * @param numWakeupPorts number of wakeup ports to the rob
 * @param numFpuPorts number of fpu ports that will write back fflags
 */
  class RobIo(
    val numWakeupPorts: Int,	// 唤醒端口数量
    val numFpuPorts: Int		// 将写回fflags的FPU端口数量
    )(implicit p: Parameters)  extends BoomBundle
  {
      
      
    // Decode Stage (Allocate, write instruction to ROB).
    // 译码阶段（将指令写进ROB的输入端口）
    val enq_valids       = Input(Vec(coreWidth, Bool()))
    val enq_uops         = Input(Vec(coreWidth, new MicroOp()))
    val enq_partial_stall= Input(Bool()) 
      // we're dispatching only a partial packet, and stalling on the rest of it 
      // (don't advance the tail ptr) 表示只派遣一个包裹的一部分，剩余部分停顿（不用向前移动尾指针）

  val xcpt_fetch_pc = Input(UInt(vaddrBitsExtended.W))
      // 输入发射异常的指令的PC

  val rob_tail_idx = Output(UInt(robAddrSz.W))
  val rob_pnr_idx  = Output(UInt(robAddrSz.W))
  val rob_head_idx = Output(UInt(robAddrSz.W))
      // 输出指向ROB的尾/不可返回点/头的“指针”

  // Handle Branch Misspeculations 输入分支更新信息，处理误判
  val brupdate = Input(new BrUpdateInfo())

      
      
  // Write-back Stage (Update of ROB)  写回阶段（更新ROB）
  // Instruction is no longer busy and can be committed 指令不再繁忙，能被提交
  val wb_resps = Flipped(Vec(numWakeupPorts, Valid(new ExeUnitResp(xLen max fLen+1))))
      // 输入写回级的回复

  // Unbusying ports for stores.
  // +1 for fpstdata
  val lsu_clr_bsy      = Input(Vec(memWidth + 1, Valid(UInt(robAddrSz.W))))

  // Port for unmarking loads/stores as speculation hazards..
  val lsu_clr_unsafe   = Input(Vec(memWidth, Valid(UInt(robAddrSz.W))))


  // Track side-effects for debug purposes.
  // Also need to know when loads write back, whereas we don't need loads to unbusy.
  val debug_wb_valids = Input(Vec(numWakeupPorts, Bool()))
  val debug_wb_wdata  = Input(Vec(numWakeupPorts, Bits(xLen.W)))

  val fflags = Flipped(Vec(numFpuPorts, new ValidIO(new FFlagsResp())))
  val lxcpt = Flipped(new ValidIO(new Exception())) // LSU

      
      
  // Commit stage (free resources; also used for rollback).
  // 输出提交信号。一个自由的信号源，可以用于回滚。
  val commit = Output(new CommitSignals())

  // tell the LSU that the head of the ROB is a load
  // (some loads can only execute once they are at the head of the ROB).
  val com_load_is_at_rob_head = Output(Bool())

  // Communicate exceptions to the CSRFile
  val com_xcpt = Valid(new CommitExceptionSignals())

  // Let the CSRFile stall us (e.g., wfi).
  val csr_stall = Input(Bool())

  // Flush signals (including exceptions, pipeline replays, and memory ordering failures)
  // to send to the frontend for redirection.
  // 清除流水线信号（异常、恢复、内存排序失败），一个提交异常信号类，发送给前端用于重定向
  val flush = Valid(new CommitExceptionSignals)

  // Stall Decode as appropriate	视情况停顿译码
  val empty = Output(Bool())
  val ready = Output(Bool()) // ROB is busy unrolling rename state...	ROB正在回滚重命名阶段

  // Stall the frontend if we know we will redirect the PC	当知道将重定向PC时使前端停顿
  val flush_frontend = Output(Bool())


  val debug_tsc = Input(UInt(xLen.W))
}
```



### CommitSignals 类

传输提交信息的一组信号。其中retireWidth为一次能提交的最大指令数。

```scala
/**
 * Bundle to send commit signals across processor
 */
  class CommitSignals(implicit p: Parameters) extends BoomBundle
  {
      
    val valids      = Vec(retireWidth, Bool()) 
      // These instructions may not correspond to an architecturally executed insn
      // 这些指令可能不是在体系结构上被执行的
    val arch_valids = Vec(retireWidth, Bool())
    val uops        = Vec(retireWidth, new MicroOp())
    val fflags      = Valid(UInt(5.W))

  // These come a cycle later  用于调试，上一个周期的指令
  val debug_insts = Vec(retireWidth, UInt(32.W))

  // Perform rollback of rename state (in conjuction with commit.uops).  用于重命名阶段的回滚信号（与微指令结合）
  val rbk_valids = Vec(retireWidth, Bool())
  val rollback   = Bool()

  val debug_wdata = Vec(retireWidth, UInt(xLen.W))
}
```



### CommitExceptionSignals 类

提交异常信号，与机器状态寄存器堆通讯的信号。

```scala
/**
 * Bundle to communicate exceptions to CSRFile
 *
 * TODO combine FlushSignals and ExceptionSignals (currently timed to different cycles).
 */
 class CommitExceptionSignals(implicit p: Parameters) extends BoomBundle
 {
    val ftq_idx    = UInt(log2Ceil(ftqSz).W)
    val edge_inst  = Bool()
    val is_rvc     = Bool()
    val pc_lob     = UInt(log2Ceil(icBlockBytes).W)
    val cause      = UInt(xLen.W)
    val badvaddr   = UInt(xLen.W)
 // The ROB needs to tell the FTQ if there's a pipeline flush (and what type)
 // so the FTQ can drive the frontend with the correct redirected PC.
 // ROB需要告诉取指队列是否有清除流水线信号（以及其类型），这样取值队列才可以用正确的重定向PC驱动前端
    val flush_typ  = FlushTypes()
 }
```



### FlushTypes 对象

告诉前端清除（flush）的种类，以正确地建立下一个PC。

```scala
/**
 * Tell the frontend the type of flush so it can set up the next PC properly.
 */
  object FlushTypes
  {
    def SZ = 3
    def apply() = UInt(SZ.W)
    def none = 0.U
    def xcpt = 1.U // An exception occurred.
    def eret = (2+1).U // Execute an environment return instruction.
    def refetch = 2.U // Flush and refetch the head instruction.
    def next = 4.U // Flush and fetch the next instruction.

  def useCsrEvec(typ: UInt): Bool = typ(0) // typ === xcpt.U || typ === eret.U
      // 异常或者环境返回指令，使用CSREven..？
  def useSamePC(typ: UInt): Bool = typ === refetch
      // 清除流水线并重新取指，使用相同的PC
  def usePCplus4(typ: UInt): Bool = typ === next
      // 清除流水线并取下一条指令，使用PC+4

  def getType(valid: Bool, i_xcpt: Bool, i_eret: Bool, i_refetch: Bool): UInt = {
    val ret =
      Mux(!valid, none,
      Mux(i_eret, eret,
      Mux(i_xcpt, xcpt,
      Mux(i_refetch, refetch,
        next))))
    ret
  }
}
```



### Exception 类

异常类。包括引发异常的微指令，异常原因以及发生异常的指令的地址。

```scala
/**
 * Bundle of signals indicating that an exception occurred
 */
 class Exception(implicit p: Parameters) extends BoomBundle
 {
    val uop = new MicroOp()
    val cause = Bits(log2Ceil(freechips.rocketchip.rocket.Causes.all.max+2).W)
    val badvaddr = UInt(coreMaxAddrBits.W)
 }
```



### DebugRobSignals 类

用于调试ROB的信号，不被综合。

```scala
/**
 * Bundle for debug ROB signals
 * These should not be synthesized!
 */
 class DebugRobSignals(implicit p: Parameters) extends BoomBundle
 {
    val state = UInt()
    val rob_head = UInt(robAddrSz.W)
    val rob_pnr = UInt(robAddrSz.W)
    val xcpt_val = Bool()
    val xcpt_uop = new MicroOp()
    val xcpt_badvaddr = UInt(xLen.W)
 }
```



### Rob 类

``` scala
/**
 * Reorder Buffer to keep track of dependencies and inflight instructions
 *
 * @param numWakeupPorts number of wakeup ports to the ROB
 * @param numFpuPorts number of FPU units that will write back fflags
 */
  @chiselName
  class Rob(
    val numWakeupPorts: Int,
    val numFpuPorts: Int
    )(implicit p: Parameters) extends BoomModule
  {
    val io = IO(new RobIo(numWakeupPorts, numFpuPorts))
    ......
```


#### 基本成员

```scala
  // ROB Finite State Machine    ROB有限状态机常量。正常/回滚/等待清空/无
  val s_reset :: s_normal :: s_rollback :: s_wait_till_empty :: Nil = Enum(4)
  val rob_state = RegInit(s_reset)


  // commit entries at the head, and unwind exceptions from the tail
  // 在头部提交，从尾部展开异常
  val rob_head     = RegInit(0.U(log2Ceil(numRobRows).W))
	// ROB的头指针
  val rob_head_lsb = RegInit(0.U((1 max log2Ceil(coreWidth)).W)) 
	// ROB的头指针的最低位。当前总是0，精确地追踪头指针LSB还未实现。
	// TODO: Accurately track head LSB (currently always 0)
  val rob_head_idx = if (coreWidth == 1) rob_head else Cat(rob_head, rob_head_lsb)
	// ROB的头指针索引，在头指针后面接上头指针最低位

  val rob_tail     = RegInit(0.U(log2Ceil(numRobRows).W))
  val rob_tail_lsb = RegInit(0.U((1 max log2Ceil(coreWidth)).W))
  val rob_tail_idx = if (coreWidth == 1) rob_tail else Cat(rob_tail, rob_tail_lsb)

  val rob_pnr      = RegInit(0.U(log2Ceil(numRobRows).W))
  val rob_pnr_lsb  = RegInit(0.U((1 max log2Ceil(coreWidth)).W))
  val rob_pnr_idx  = if (coreWidth == 1) rob_pnr  else Cat(rob_pnr , rob_pnr_lsb)
	// ROB的尾指针和不可返回点，定义和头指针类似


  val com_idx = Mux(rob_state === s_rollback, rob_tail, rob_head)
	// 提交索引。回滚状态时为ROB尾指针，其他状态为头指针。


	// 填充状态指示信号
  val maybe_full   = RegInit(false.B)
  val full         = Wire(Bool())
  val empty        = Wire(Bool())


	// 提交信号
  val will_commit         = Wire(Vec(coreWidth, Bool()))
  val can_commit          = Wire(Vec(coreWidth, Bool()))
  val can_throw_exception = Wire(Vec(coreWidth, Bool()))


	// 储存ROB各栏中指令的状态
  val rob_pnr_unsafe      = Wire(Vec(coreWidth, Bool())) 
	// are the instructions at the pnr unsafe?
  val rob_head_vals       = Wire(Vec(coreWidth, Bool())) 
	// are the instructions at the head valid?
  val rob_tail_vals       = Wire(Vec(coreWidth, Bool())) 
	// are the instructions at the tail valid? (to track partial row dispatches)
  val rob_head_uses_stq   = Wire(Vec(coreWidth, Bool()))
  val rob_head_uses_ldq   = Wire(Vec(coreWidth, Bool()))
  val rob_head_fflags     = Wire(Vec(coreWidth, UInt(freechips.rocketchip.tile.FPConstants.FLAGS_SZ.W)))


  val exception_thrown = Wire(Bool())

  // exception info		异常信息
  // TODO compress xcpt cause size. Most bits in the middle are zero. 
  // 未实现：将异常原因的长度缩小，因为中间大部分位是0
  val r_xcpt_val       = RegInit(false.B)
  val r_xcpt_uop       = Reg(new MicroOp())
  val r_xcpt_badvaddr  = Reg(UInt(coreMaxAddrBits.W))
  io.flush_frontend := r_xcpt_val
```

#### 获取行和栏索引的函数

从rob_idx中获取row_idx和bank_idx。

在rob_idx中，bank_idx是低位，位数和派遣宽度相同（即coreWidth，刚好可以索引每个栏），剩下的高位是row_idx。

```scala
  //--------------------------------------------------
  // Utility

  def GetRowIdx(rob_idx: UInt): UInt = {
    if (coreWidth == 1) 
      return rob_idx
    else 
      return rob_idx >> log2Ceil(coreWidth).U
  }

  def GetBankIdx(rob_idx: UInt): UInt = {
    if(coreWidth == 1) 
      { return 0.U }
    else           
      { return rob_idx(log2Ceil(coreWidth)-1, 0).asUInt }
  }
```


#### Debug端口

```scala
  // **************************************************************************
  // Debug

  class DebugRobBundle extends BoomBundle	// Debug端口类
  {
    val valid      = Bool()
    val busy       = Bool()
    val unsafe     = Bool()
    val uop        = new MicroOp()
    val exception  = Bool()
  }

  val debug_entry = Wire(Vec(numRobEntries, new DebugRobBundle))
	// 为每个ROB入口构造一个Debug端口类（线网）
  debug_entry := DontCare // override in statements below 先置为不关心，后面会重写
```


#### ROB逻辑

```scala
  // **************************************************************************
  // --------------------------------------------------------------------------
  // **************************************************************************


  // Contains all information the PNR needs to find the oldest instruction which can't be safely speculated past.
  // 包含PNR查找无法安全推测过去的最旧指令所需的所有信息。
  val rob_unsafe_masked = WireInit(VecInit(Seq.fill(numRobRows << log2Ceil(coreWidth)){false.B}))


  // Used for trace port, for debug purposes only
  // 只用于debug的跟踪端口
  val rob_debug_inst_mem   = SyncReadMem(numRobRows, Vec(coreWidth, UInt(32.W)))
  val rob_debug_inst_wmask = WireInit(VecInit(0.U(coreWidth.W).asBools))
  val rob_debug_inst_wdata = Wire(Vec(coreWidth, UInt(32.W)))
  rob_debug_inst_mem.write(rob_tail, rob_debug_inst_wdata, rob_debug_inst_wmask)
  val rob_debug_inst_rdata = rob_debug_inst_mem.read(rob_head, will_commit.reduce(_||_))


  for (w <- 0 until coreWidth) {	// 遍历ROB的每一栏
      
    def MatchBank(bank_idx: UInt): Bool = (bank_idx === w.U)
      // 定义栏匹配函数，当输入的值与当前遍历的栏号相同时返回1
      // bank_idx由上面定义的GetBankIdx函数得到

    // one bank 一个栏中每个行的指令的信号
    val rob_val       = RegInit(VecInit(Seq.fill(numRobRows){false.B}))
    val rob_bsy       = Reg(Vec(numRobRows, Bool()))
    val rob_unsafe    = Reg(Vec(numRobRows, Bool()))
    val rob_uop       = Reg(Vec(numRobRows, new MicroOp()))
    val rob_exception = Reg(Vec(numRobRows, Bool()))
    val rob_predicated = Reg(Vec(numRobRows, Bool())) // Was this instruction predicated out?
    val rob_fflags    = Mem(numRobRows, Bits(freechips.rocketchip.tile.FPConstants.FLAGS_SZ.W))
    
    val rob_debug_wdata = Mem(numRobRows, UInt(xLen.W))
    ......
```
##### 将派遣的指令添加到ROB尾部

```scala
    //-----------------------------------------------
    // Dispatch: Add Entry to ROB
    
    rob_debug_inst_wmask(w) := io.enq_valids(w)
    rob_debug_inst_wdata(w) := io.enq_uops(w).debug_inst
    
    when (io.enq_valids(w)) {	// 当要派遣到这一栏的指令valid时，尝试将指令放入ROB尾部
        
      rob_val(rob_tail)       := true.B
      rob_bsy(rob_tail)       := !(io.enq_uops(w).is_fence ||
                                   io.enq_uops(w).is_fencei)
      rob_unsafe(rob_tail)    := io.enq_uops(w).unsafe
      rob_uop(rob_tail)       := io.enq_uops(w)
      rob_exception(rob_tail) := io.enq_uops(w).exception
      rob_predicated(rob_tail)   := false.B
      rob_fflags(rob_tail)    := 0.U
    
      assert (rob_val(rob_tail) === false.B, "[rob] overwriting a valid entry.")
        // 写入前尾部已经有valid的指令，发出警告
      assert ((io.enq_uops(w).rob_idx >> log2Ceil(coreWidth)) === rob_tail)
        // 索引右移后应该就是尾部指针的值
    } 

	.elsewhen (io.enq_valids.reduce(_|_) && !rob_val(rob_tail)) {	// 指令未valid，该位置也没有valid指令
      rob_uop(rob_tail).debug_inst := BUBBLE // just for debug purposes
        // 标记为气泡位，仅用于debug
    }
```
##### 处理唤醒端口中的写回信息

lsu的写回信息需要独立处理。（待研究）

```scala
    //-----------------------------------------------
    // Writeback
    

    for (i <- 0 until numWakeupPorts) {	// 遍历唤醒端口
        
      val wb_resp = io.wb_resps(i)
      val wb_uop = wb_resp.bits.uop
      val row_idx = GetRowIdx(wb_uop.rob_idx)
      when (wb_resp.valid && MatchBank(GetBankIdx(wb_uop.rob_idx))) {	// 当写回回复valid且栏匹配时
        rob_bsy(row_idx)      := false.B	// 已写回，不再繁忙
        rob_unsafe(row_idx)   := false.B	// 写回但未提交，置为不安全？
        rob_predicated(row_idx)  := wb_resp.bits.predicated
      }
        
      // TODO check that fflags aren't overwritten
      // TODO check that the wb is to a valid ROB entry, give it a time stamp
      // 未完成部分代码：确认fflags没被重写、确认写回是给到一个valid的ROB条目并且有时间戳        
//        assert (!(wb_resp.valid && MatchBank(GetBankIdx(wb_uop.rob_idx)) &&
//                  wb_uop.fp_val && !(wb_uop.is_load || wb_uop.is_store) &&
//                  rob_exc_cause(row_idx) =/= 0.U),
//                  "FP instruction writing back exc bits is overriding an existing exception.")    
    }


    // Stores have a separate method to clear busy bits
	// 储存指令清除数据需要独立的方法
    for (clr_rob_idx <- io.lsu_clr_bsy) {
      when (clr_rob_idx.valid && MatchBank(GetBankIdx(clr_rob_idx.bits))) {
        val cidx = GetRowIdx(clr_rob_idx.bits)
        rob_bsy(cidx)    := false.B
        rob_unsafe(cidx) := false.B
        assert (rob_val(cidx) === true.B, "[rob] store writing back to invalid entry.")
        assert (rob_bsy(cidx) === true.B, "[rob] store writing back to a not-busy entry.")
      }
    }
    for (clr <- io.lsu_clr_unsafe) {
      when (clr.valid && MatchBank(GetBankIdx(clr.bits))) {
        val cidx = GetRowIdx(clr.bits)
        rob_unsafe(cidx) := false.B
      }
    }
```
##### 积累fflags（待研究）

```scala
    //-----------------------------------------------
    // Accruing fflags
    for (i <- 0 until numFpuPorts) {	// 遍历每个FPU端口
      val fflag_uop = io.fflags(i).bits.uop
      when (io.fflags(i).valid && MatchBank(GetBankIdx(fflag_uop.rob_idx))) {
        rob_fflags(GetRowIdx(fflag_uop.rob_idx)) := io.fflags(i).bits.flags
      }
    }
```
##### 更新异常信息

发生异常时，将ROB中指令的异常位置1，并更新can_throw_exception的值。

```scala
    //-----------------------------------------------------
    // Exceptions
    // (the cause bits are compressed and stored elsewhere)
    
    when (io.lxcpt.valid && MatchBank(GetBankIdx(io.lxcpt.bits.uop.rob_idx))) {
        
      rob_exception(GetRowIdx(io.lxcpt.bits.uop.rob_idx)) := true.B
        
      when (io.lxcpt.bits.cause =/= MINI_EXCEPTION_MEM_ORDERING) {          
        // In the case of a mem-ordering failure, the failing load will have been marked safe already.
        // 在内存排序失败的情况下，失败的加载指令将已经被标记为安全。因此不考虑内存排序失败的情况。
        assert(rob_unsafe(GetRowIdx(io.lxcpt.bits.uop.rob_idx)),
          "An instruction marked as safe is causing an exception")
          // 对于其他情况，当被标记为安全的指令造成异常时发出警告
      }
        
    }

    can_throw_exception(w) := rob_val(rob_head) && rob_exception(rob_head)
	  // 仅当能引发异常的指令到达ROB头部时才抛出异常
```
##### 处理提交或回滚
```scala
    //-----------------------------------------------
    // Commit or Rollback
    

    // Can this instruction commit? (the check for exceptions/rob_state happens later).
	// 先判断这栏中，头指针选中的指令能否提交。只要求valid、不繁忙，并且不被csr停顿。
	// 检查异常和rob状态将在后面进行。
    can_commit(w) := rob_val(rob_head) && !(rob_bsy(rob_head)) && !io.csr_stall


    // use the same "com_uop" for both rollback AND commit
    // Perform Commit
	// 输出给提交级
    io.commit.valids(w) := will_commit(w)
    io.commit.arch_valids(w) := will_commit(w) && !rob_predicated(com_idx)
    io.commit.uops(w)   := rob_uop(com_idx)
    io.commit.debug_insts(w) := rob_debug_inst_rdata(w)
    

    // We unbusy branches in b1, but its easier to mark the taken/provider src in b2,
    // when the branch might be committing
	// ？
    when (io.brupdate.b2.mispredict &&
      MatchBank(GetBankIdx(io.brupdate.b2.uop.rob_idx)) &&
      GetRowIdx(io.brupdate.b2.uop.rob_idx) === com_idx) {
      io.commit.uops(w).debug_fsrc := BSRC_C
      io.commit.uops(w).taken      := io.brupdate.b2.taken
    }


    // Don't attempt to rollback the tail's row when the rob is full.
	// 当ROB满时，不尝试回滚尾部那一行 ？
    val rbk_row = rob_state === s_rollback && !full
    
    io.commit.rbk_valids(w) := rbk_row && rob_val(com_idx) && !(enableCommitMapTable.B)
	  // 若要为1，除了rbk_row为1外，还需要提交的那条指令valid，以及禁用提交映射表
    io.commit.rollback := (rob_state === s_rollback)
    
    assert (!(io.commit.valids.reduce(_||_) && io.commit.rbk_valids.reduce(_||_)),
      "com_valids and rbk_valids are mutually exclusive")
	// 一条指令不可能在提交和回滚上同时valid
    
    when (rbk_row) {
      rob_val(com_idx)       := false.B
      rob_exception(com_idx) := false.B
    }
    
    if (enableCommitMapTable) {			// 启用提交映射表时，在异常的下一个周期，清空ROB（全部置为“气泡”）
      when (RegNext(exception_thrown)) {
        for (i <- 0 until numRobRows) {
          rob_val(i) := false.B
          rob_bsy(i) := false.B
          rob_uop(i).debug_inst := BUBBLE
        }
      }
    }
```
##### 清除分支误判的推测条目

也会同时更新没有误判的指令的分支掩码。

```scala
    // -----------------------------------------------
    // Kill speculated entries on branch mispredict

    for (i <- 0 until numRobRows) {		// 遍历ROB当前栏中的每一行
      val br_mask = rob_uop(i).br_mask
    
      //kill instruction if mispredict & br mask match
      // 若这行的指令是分支误判，清除；若没误判且指令有效，则更新其分支掩码
      when (IsKilledByBranch(io.brupdate, br_mask))
      {
        rob_val(i) := false.B
        rob_uop(i.U).debug_inst := BUBBLE
      } .elsewhen (rob_val(i)) {
        // clear speculation bit even on correct speculation
        rob_uop(i).br_mask := GetNewBrMask(io.brupdate, br_mask)
      }
    }


    // Debug signal to figure out which prediction structure or core resolved a branch correctly
	// 调试信号，用于找出哪个预测结构/核计算出了这个分支
    when (io.brupdate.b2.mispredict && MatchBank(GetBankIdx(io.brupdate.b2.uop.rob_idx))) {
      rob_uop(GetRowIdx(io.brupdate.b2.uop.rob_idx)).debug_fsrc := BSRC_C
      rob_uop(GetRowIdx(io.brupdate.b2.uop.rob_idx)).taken      := io.brupdate.b2.taken
    }
```
##### 将提交时将头部指令valid置0
```scala
    // -----------------------------------------------
    // Commit
    when (will_commit(w)) {
      rob_val(rob_head) := false.B
    }
```
##### 保存需要输出的值
```scala
    // -----------------------------------------------
    // Outputs
    rob_head_vals(w)     := rob_val(rob_head)
    rob_tail_vals(w)     := rob_val(rob_tail)
    rob_head_fflags(w)   := rob_fflags(rob_head)
    rob_head_uses_stq(w) := rob_uop(rob_head).uses_stq
    rob_head_uses_ldq(w) := rob_uop(rob_head).uses_ldq
```
##### 更新安全状态

更新栏中所有指令的不安全掩码，以及读取不可返回点指令的安全状态。

```scala
    //------------------------------------------------
    // Invalid entries are safe; thrown exceptions are unsafe.

    for (i <- 0 until numRobRows) {	// 遍历当前栏的每一行
      rob_unsafe_masked((i << log2Ceil(coreWidth)) + w) := rob_val(i) && (rob_unsafe(i) || rob_exception(i))
        // 索引是完整的ROB索引；当指令有效，且不安全/有异常时，将不安全掩码中对应位置置1
    }

    // Read unsafe status of PNR row.
    rob_pnr_unsafe(w) := rob_val(rob_pnr) && (rob_unsafe(rob_pnr) || rob_exception(rob_pnr))
```
##### 不综合的debug写端口
```scala
    // -----------------------------------------------
    // debugging write ports that should not be synthesized
    when (will_commit(w)) {
      rob_uop(rob_head).debug_inst := BUBBLE
    } 
	.elsewhen (rbk_row) {
      rob_uop(rob_tail).debug_inst := BUBBLE
    }
```
##### 跟踪目的寄存器副作用的debug
```scala
    //--------------------------------------------------
    // Debug: for debug purposes, track side-effects to all register destinations
    
    for (i <- 0 until numWakeupPorts) {	// 遍历每个唤醒端口
        
      val rob_idx = io.wb_resps(i).bits.uop.rob_idx
      when (io.debug_wb_valids(i) && MatchBank(GetBankIdx(rob_idx))) {
        rob_debug_wdata(GetRowIdx(rob_idx)) := io.debug_wb_wdata(i)
      }
      val temp_uop = rob_uop(GetRowIdx(rob_idx))
    
      assert (!(io.wb_resps(i).valid && MatchBank(GetBankIdx(rob_idx)) &&
               !rob_val(GetRowIdx(rob_idx))),
               "[rob] writeback (" + i + ") occurred to an invalid ROB entry.")
      assert (!(io.wb_resps(i).valid && MatchBank(GetBankIdx(rob_idx)) &&
               !rob_bsy(GetRowIdx(rob_idx))),
               "[rob] writeback (" + i + ") occurred to a not-busy ROB entry.")
      assert (!(io.wb_resps(i).valid && MatchBank(GetBankIdx(rob_idx)) &&
               temp_uop.ldst_val && temp_uop.pdst =/= io.wb_resps(i).bits.uop.pdst),
               "[rob] writeback (" + i + ") occurred to the wrong pdst.")
    }

    io.commit.debug_wdata(w) := rob_debug_wdata(rob_head)

  } //for (w <- 0 until coreWidth)	接着处理下一栏
```
#### 提交逻辑

根据can_commit信号，让第一个can_commits提交。

旧指令可能会阻挡提交端口中新指令的提交（比如异常或者同时valid且busy）。

如果前面有指令想提交，则不抛出异常（仅在头端口抛出异常）。

> 当指令在提交头且不再繁忙（而且不会引发异常）时，它可能会被提交，也就是说，它对体系结构的改变将变得可见。对于超标量提交，会考虑整个ROB行指令的繁忙情况，因此一个周期最多能提交一整行的ROB指令。ROB会尽可能多地提交这行中的指令，以尽可能快地释放资源。然而，ROB当前还不能在多行中寻找能提交的指令。
>
> 储存指令仅当在提交时才会被送往内存。对于储存的超标量提交，加载/储存单元（LSU）将被告知有多少储存被标记为提交，然后在合适的时候将储存提交到内存。
>
> 当一条写寄存器的指令提交，它将释放僵死的物理目的寄存器（stale pdst），使其之后能被重分配给新的指令。

```scala  
  // **************************************************************************
  // --------------------------------------------------------------------------
  // **************************************************************************

  // -----------------------------------------------
  // Commit Logic
  // need to take a "can_commit" array, and let the first can_commits commit
  // previous instructions may block the commit of younger instructions in the commit bundle
  // e.g., exception, or (valid && busy).
  // Finally, don't throw an exception if there are instructions in front of
  // it that want to commit (only throw exception when head of the bundle).

	// 变量。ROB不在正常状态，且前两个周期抛出过异常或者ROB正等待清空时，阻塞提交。
  var block_commit = (rob_state =/= s_normal) && 
						(rob_state =/= s_wait_till_empty) 	|| 
						RegNext(exception_thrown) 			|| 
						RegNext(RegNext(exception_thrown))
  var will_throw_exception = false.B
  var block_xcpt   = false.B


  for (w <- 0 until coreWidth) {	// 遍历每个栏
    
    // 整行的抛出异常信号，只要此行中有一个栏的指令会抛出异常就置1
    will_throw_exception = (can_throw_exception(w) && !block_commit && !block_xcpt) || will_throw_exception

    // 整行所有指令中可提交的指令
    will_commit(w)       := can_commit(w) && !can_throw_exception(w) && !block_commit
      
    // 异常可以阻塞提交（是否有阻塞了提交的信号）
    block_commit         = (rob_head_vals(w) &&
                           (!can_commit(w) || can_throw_exception(w))) || block_commit
    // ？？
    block_xcpt           = will_commit(w)
  }


  // Note: exception must be in the commit bundle.
  // Note: exception must be the first valid instruction in the commit bundle.
  // 注意：异常必须在提交端口中，且必须是提交端口中第一条valid的指令                          待研究
  exception_thrown := will_throw_exception
  val is_mini_exception = io.com_xcpt.bits.cause === MINI_EXCEPTION_MEM_ORDERING
  io.com_xcpt.valid := exception_thrown && !is_mini_exception
  io.com_xcpt.bits.cause := r_xcpt_uop.exc_cause

  io.com_xcpt.bits.badvaddr := Sext(r_xcpt_badvaddr, xLen)
  val insn_sys_pc2epc =
    rob_head_vals.reduce(_|_) && PriorityMux(rob_head_vals, io.commit.uops.map{u => u.is_sys_pc2epc})

  val refetch_inst = exception_thrown || insn_sys_pc2epc
  val com_xcpt_uop = PriorityMux(rob_head_vals, io.commit.uops)
  io.com_xcpt.bits.ftq_idx   := com_xcpt_uop.ftq_idx
  io.com_xcpt.bits.edge_inst := com_xcpt_uop.edge_inst
  io.com_xcpt.bits.is_rvc    := com_xcpt_uop.is_rvc
  io.com_xcpt.bits.pc_lob    := com_xcpt_uop.pc_lob

  val flush_commit_mask = Range(0,coreWidth).map{i => io.commit.valids(i) && io.commit.uops(i).flush_on_commit}
  val flush_commit = flush_commit_mask.reduce(_|_)
  val flush_val = exception_thrown || flush_commit

  assert(!(PopCount(flush_commit_mask) > 1.U),
    "[rob] Can't commit multiple flush_on_commit instructions on one cycle")

  val flush_uop = Mux(exception_thrown, com_xcpt_uop, Mux1H(flush_commit_mask, io.commit.uops))

  // delay a cycle for critical path considerations
  io.flush.valid          := flush_val
  io.flush.bits.ftq_idx   := flush_uop.ftq_idx
  io.flush.bits.pc_lob    := flush_uop.pc_lob
  io.flush.bits.edge_inst := flush_uop.edge_inst
  io.flush.bits.is_rvc    := flush_uop.is_rvc
  io.flush.bits.flush_typ := FlushTypes.getType(flush_val,
                                                exception_thrown && !is_mini_exception,
                                                flush_commit && flush_uop.uopc === uopERET,
                                                refetch_inst)
```
#### 浮点异常逻辑（待研究）

将fflags数据送给CSR。

```scala 
  // -----------------------------------------------
  // FP Exceptions
  // send fflags bits to the CSRFile to accrue

  val fflags_val = Wire(Vec(coreWidth, Bool()))
  val fflags     = Wire(Vec(coreWidth, UInt(freechips.rocketchip.tile.FPConstants.FLAGS_SZ.W)))

  for (w <- 0 until coreWidth) {
    fflags_val(w) :=
      io.commit.valids(w) &&
      io.commit.uops(w).fp_val &&
      !io.commit.uops(w).uses_stq

    fflags(w) := Mux(fflags_val(w), rob_head_fflags(w), 0.U)
    
    assert (!(io.commit.valids(w) &&
             !io.commit.uops(w).fp_val &&
             rob_head_fflags(w) =/= 0.U),
             "Committed non-FP instruction has non-zero fflag bits.")
    assert (!(io.commit.valids(w) &&
             io.commit.uops(w).fp_val &&
             (io.commit.uops(w).uses_ldq || io.commit.uops(w).uses_stq) &&
             rob_head_fflags(w) =/= 0.U),
             "Committed FP load or store has non-zero fflag bits.")
  }
  io.commit.fflags.valid := fflags_val.reduce(_|_)
  io.commit.fflags.bits  := fflags.reduce(_|_)
```
#### 异常跟踪逻辑（待研究）

最前面（最老）指令异常信息的跟踪逻辑，如果新收到的异常指令比正在跟踪的异常指令更“老”，则更新正在追踪的异常状态。

```scala
  // -----------------------------------------------
  // Exception Tracking Logic
  // only store the oldest exception, since only one can happen!

  val next_xcpt_uop = Wire(new MicroOp())
  next_xcpt_uop := r_xcpt_uop
  val enq_xcpts = Wire(Vec(coreWidth, Bool()))
  for (i <- 0 until coreWidth) {
    enq_xcpts(i) := io.enq_valids(i) && io.enq_uops(i).exception
  }

  when (!(io.flush.valid || exception_thrown) && rob_state =/= s_rollback) {
    when (io.lxcpt.valid) {
      val new_xcpt_uop = io.lxcpt.bits.uop

      when (!r_xcpt_val || IsOlder(new_xcpt_uop.rob_idx, r_xcpt_uop.rob_idx, rob_head_idx)) {
        r_xcpt_val              := true.B
        next_xcpt_uop           := new_xcpt_uop
        next_xcpt_uop.exc_cause := io.lxcpt.bits.cause
        r_xcpt_badvaddr         := io.lxcpt.bits.badvaddr
      }
    } .elsewhen (!r_xcpt_val && enq_xcpts.reduce(_|_)) {
      val idx = enq_xcpts.indexWhere{i: Bool => i}
    
      // if no exception yet, dispatch exception wins
      r_xcpt_val      := true.B
      next_xcpt_uop   := io.enq_uops(idx)
      r_xcpt_badvaddr := AlignPCToBoundary(io.xcpt_fetch_pc, icBlockBytes) | io.enq_uops(idx).pc_lob
    
    }
  }

  r_xcpt_uop         := next_xcpt_uop
  r_xcpt_uop.br_mask := GetNewBrMask(io.brupdate, next_xcpt_uop)
  when (io.flush.valid || IsKilledByBranch(io.brupdate, next_xcpt_uop)) {
    r_xcpt_val := false.B
  }

  assert (!(exception_thrown && !r_xcpt_val),
    "ROB trying to throw an exception, but it doesn't have a valid xcpt_cause")

  assert (!(empty && r_xcpt_val),
    "ROB is empty, but believes it has an outstanding exception.")

  assert (!(will_throw_exception && (GetRowIdx(r_xcpt_uop.rob_idx) =/= rob_head)),
    "ROB is throwing an exception, but the stored exception information's " +
    "rob_idx does not match the rob_head")
```
#### ROB的头逻辑

需要注意，不能在提交部分包裹之后就移动头指针，而需要等待整个包裹派遣进ROB。

即仅当提交端口中所有valid指令都提交后，头指针才会更新。

```scala
  // -----------------------------------------------
  // ROB Head Logic

  // remember if we're still waiting on the rest of the dispatch packet, and prevent
  // the rob_head from advancing if it commits a partial parket before we
  // dispatch the rest of it.
  // update when committed ALL valid instructions in commit_bundle

  val rob_deq = WireInit(false.B)
  val r_partial_row = RegInit(false.B)

  when (io.enq_valids.reduce(_|_)) {	// 入列，即派遣进的包裹中仍有valid指令，说明包裹未全部派遣完毕
    r_partial_row := io.enq_partial_stall	// 使用停顿信号
  }

  val finished_committing_row =		
    (io.commit.valids.asUInt =/= 0.U) &&
    ((will_commit.asUInt ^ rob_head_vals.asUInt) === 0.U) &&
    !(r_partial_row && rob_head === rob_tail && !maybe_full)
	// 有valid的提交 且 valid的提交恰好就是头这一行所有的valid指令 且
	//  满足至少一条：①这个包裹已全部派遣 ②ROB头尾不重叠 ③ROB可能会填满
	// 此时才可判断头行已全部完成提交


  when (finished_committing_row) {		// 头行全部提交完成，头指针向前移动，并且说明ROB出列
    rob_head     := WrapInc(rob_head, numRobRows)
    rob_head_lsb := 0.U	// 似乎当前没有用
    rob_deq      := true.B
  } 
  .otherwise {
    rob_head_lsb := OHToUInt(PriorityEncoderOH(rob_head_vals.asUInt))
  }
```
#### ROB的PNR逻辑

PNR即Point-of-No-Return，暂且翻译为不可返回点。就像是ROB的第二个头指针，它在ROB提交头的前面，指向**位于最前面（最旧）的，有可能是错误预测，或有可能发生异常**的指令。 这意味着在提交头前面，且在PNR后面的指令一定是安全的（？），不会发生回滚的，即使还没写回。在PNR逻辑中，最核心的内容就是`do_inc_row`，它控制着PNC的增长。即当PNR当前所指向的行的所有指令的`unsafe`都为`false`时（同时PNR指向的不是tail或者ROB已满等情况），PNR会向下指向一行。

目前PNR仅仅用于RoCC的场景下。很多RoCC要求发来的指令必须是确定的，不能是处于“推测”状态下的。因此ROB仅当RoCC指令过了PNR之后，才将其发射到RoCC上。

未实现：为指针添加一个校验码？使得比较哪个异常更老的逻辑开销更小，以防到时需要用大量的PNR相比较。也不需要将ROB尾部（或头部）导出到我们想要与PNR进行比较的任何对象。

```scala  
  // -----------------------------------------------
  // ROB Point-of-No-Return (PNR) Logic
  // Acts as a second head, but only waits on busy instructions which might cause misspeculation.
  // TODO is it worth it to add an extra 'parity' bit to all rob pointer logic?
  // Makes 'older than' comparisons ~3x cheaper, in case we're going to use the PNR to do a large number of those.
  // Also doesn't require the rob tail (or head) to be exported to whatever we want to compare with the PNR.

  if (enableFastPNR) {
    val unsafe_entry_in_rob = rob_unsafe_masked.reduce(_||_)
    val next_rob_pnr_idx = Mux(unsafe_entry_in_rob,
                               AgePriorityEncoder(rob_unsafe_masked, rob_head_idx),
                               rob_tail << log2Ceil(coreWidth) | PriorityEncoder(~rob_tail_vals.asUInt))
    rob_pnr := next_rob_pnr_idx >> log2Ceil(coreWidth)
    if (coreWidth > 1)
      rob_pnr_lsb := next_rob_pnr_idx(log2Ceil(coreWidth)-1, 0)
  } else {
    // Distinguish between PNR being at head/tail when ROB is full.
    // Works the same as maybe_full tracking for the ROB tail.
    val pnr_maybe_at_tail = RegInit(false.B)

    val safe_to_inc = rob_state === s_normal || rob_state === s_wait_till_empty
    val do_inc_row  = !rob_pnr_unsafe.reduce(_||_) && (rob_pnr =/= rob_tail || (full && !pnr_maybe_at_tail))
    when (empty && io.enq_valids.asUInt =/= 0.U) {
      // Unforunately for us, the ROB does not use its entries in monotonically
      //  increasing order, even in the case of no exceptions. The edge case
      //  arises when partial rows are enqueued and committed, leaving an empty
      //  ROB.
      rob_pnr     := rob_head
      rob_pnr_lsb := PriorityEncoder(io.enq_valids)
    } .elsewhen (safe_to_inc && do_inc_row) {
      rob_pnr     := WrapInc(rob_pnr, numRobRows)
      rob_pnr_lsb := 0.U
    } .elsewhen (safe_to_inc && (rob_pnr =/= rob_tail || (full && !pnr_maybe_at_tail))) {
      rob_pnr_lsb := PriorityEncoder(rob_pnr_unsafe)
    } .elsewhen (safe_to_inc && !full && !empty) {
      rob_pnr_lsb := PriorityEncoder(rob_pnr_unsafe.asUInt | ~MaskLower(rob_tail_vals.asUInt))
    } .elsewhen (full && pnr_maybe_at_tail) {
      rob_pnr_lsb := 0.U
    }
    
    pnr_maybe_at_tail := !rob_deq && (do_inc_row || pnr_maybe_at_tail)
  }

  // Head overrunning PNR likely means an entry hasn't been marked as safe when it should have been.
  assert(!IsOlder(rob_pnr_idx, rob_head_idx, rob_tail_idx) || rob_pnr_idx === rob_tail_idx)

  // PNR overrunning tail likely means an entry has been marked as safe when it shouldn't have been.
  assert(!IsOlder(rob_tail_idx, rob_pnr_idx, rob_head_idx) || full)
```
#### ROB的尾逻辑
```scala 
  // -----------------------------------------------
  // ROB Tail Logic

  val rob_enq = WireInit(false.B)

  when (rob_state === s_rollback && (rob_tail =/= rob_head || maybe_full)) {
    // Rollback a row
    // ROB在回滚状态，且头尾不重叠（未满）或者未将被填满，则回滚一行
    rob_tail     := WrapDec(rob_tail, numRobRows)	// 尾指针-1
    rob_tail_lsb := (coreWidth-1).U	// ？
    rob_deq := true.B	// 出列信号置1？
  } 

  .elsewhen (rob_state === s_rollback && (rob_tail === rob_head) && !maybe_full) {
    // Rollback an entry
    // ROB在回滚状态，且头尾重叠，且未将被填满，则回滚一个条目？
    rob_tail_lsb := rob_head_lsb
  } 

  .elsewhen (io.brupdate.b2.mispredict) {
    // b2误判时，尾部定位到误判指令的下一行？
    rob_tail     := WrapInc(GetRowIdx(io.brupdate.b2.uop.rob_idx), numRobRows)
    rob_tail_lsb := 0.U
  } 

  .elsewhen (io.enq_valids.asUInt =/= 0.U && !io.enq_partial_stall) {
    // 有新的一批指令要派遣进来，且没有部分派遣停顿信号时，尾部+1，入列信号置1，准备接受派遣的指令
    rob_tail     := WrapInc(rob_tail, numRobRows)
    rob_tail_lsb := 0.U
    rob_enq      := true.B
  } 

  .elsewhen (io.enq_valids.asUInt =/= 0.U && io.enq_partial_stall) {
    // 有新的一批指令要派遣进来，但有部分派遣停顿信号时，？
    rob_tail_lsb := PriorityEncoder(~MaskLower(io.enq_valids.asUInt))
  }


  if (enableCommitMapTable) {			// 启用提交映射表时，在抛出异常的下个周期将各指针复位
    when (RegNext(exception_thrown)) {
      rob_tail     := 0.U
      rob_tail_lsb := 0.U
      rob_head     := 0.U
      rob_pnr      := 0.U
      rob_pnr_lsb  := 0.U
    }
  }
```
#### ROB满/空的逻辑

ROB可以完全填满，但仅当上个周期没有派遣一行的时候。在每个周期都派遣一行、提交一行的平稳状态下，至少有一个条目是空的。

```scala
  // -----------------------------------------------
  // Full/Empty Logic
  // The ROB can be completely full, but only if it did not dispatch a row in the prior cycle.
  // I.E. at least one entry will be empty when in a steady state of dispatching and committing a row each cycle.
  // TODO should we add an extra 'parity bit' onto the ROB pointers to simplify this logic?
  // 未完成：给ROB指针添加一个额外的“校验位”以简化这部分逻辑？

  maybe_full := !rob_deq && (rob_enq || maybe_full) || io.brupdate.b1.mispredict_mask =/= 0.U
	// 没有出列 且 可能填满 或 有入列 或 有误判，则认为可能填满
  full       := rob_tail === rob_head && maybe_full
	// 头尾重叠，可能填满，则认为已满
  empty      := (rob_head === rob_tail) && (rob_head_vals.asUInt === 0.U)
	// 头尾重叠，但所有头的指令都未valid，则认为空

	// 输出各指针索引、空状态指示以及ready信号
  io.rob_head_idx := rob_head_idx
  io.rob_tail_idx := rob_tail_idx
  io.rob_pnr_idx  := rob_pnr_idx
  io.empty        := empty
  io.ready        := (rob_state === s_normal) && !full && !r_xcpt_val
```
#### ROB有限状态机

enableCommitMapTable为0时的状态图： （该参数意义见本文最后）

<img src=".\eCMT0.jpg" alt="eCMT0" style="zoom: 80%;" />

enableCommitMapTable为1时的状态图：

<img src=".\eCMT1.jpg" alt="eCMT1" style="zoom:80%;" />

相同转换用绿色标出，不同的转换用红色和蓝色标出。

```scala 
  //-----------------------------------------------
  //-----------------------------------------------
  //-----------------------------------------------

  // ROB FSM
  if (!enableCommitMapTable) {
    
    switch (rob_state) {
      
      is (s_reset) {
        rob_state := s_normal
      }
      
      is (s_normal) {
        
        // Delay rollback 2 cycles so branch mispredictions can drain
        when (RegNext(RegNext(exception_thrown))) {
          rob_state := s_rollback
        } 
          
        .otherwise {
          for (w <- 0 until coreWidth) {
            when (io.enq_valids(w) && io.enq_uops(w).is_unique) {
              rob_state := s_wait_till_empty
            }
          }
        }
      }
      
      is (s_rollback) {
        when (empty) {
          rob_state := s_normal
        }
      }
        
      is (s_wait_till_empty) {
        when (RegNext(exception_thrown)) {
          rob_state := s_rollback
        } .elsewhen (empty) {
          rob_state := s_normal
        }
      }
    }
  } 

  else {
    
    switch (rob_state) {
        
      is (s_reset) {
        rob_state := s_normal
      }
        
      is (s_normal) {
          
        when (exception_thrown) {
          ; //rob_state := s_rollback
        } 
        .otherwise {
          for (w <- 0 until coreWidth) {
            when (io.enq_valids(w) && io.enq_uops(w).is_unique) {
              rob_state := s_wait_till_empty
            }
          }
        }
      }
        
      is (s_rollback) {
        when (rob_tail_idx  === rob_head_idx) {
          rob_state := s_normal
        }
      }
        
      is (s_wait_till_empty) {
          
        when (exception_thrown) {
          ; //rob_state := s_rollback
        } 
        .elsewhen (rob_tail === rob_head) {
          rob_state := s_normal
        }
      }
    }
  }


  // -----------------------------------------------
  // Outputs

  io.com_load_is_at_rob_head := RegNext(rob_head_uses_ldq(PriorityEncoder(rob_head_vals.asUInt)) &&
                                        !will_commit.reduce(_||_))



  override def toString: String = BoomCoreStringPrefix(
    "==ROB==",
    "Machine Width      : " + coreWidth,
    "Rob Entries        : " + numRobEntries,
    "Rob Rows           : " + numRobRows,
    "Rob Row size       : " + log2Ceil(numRobRows),
    "log2Ceil(coreWidth): " + log2Ceil(coreWidth),
    "FPU FFlag Ports    : " + numFpuPorts)
}
```







## 补充



### PC存储

ROB必须知道每一条执行中指令的PC，因为这会在以下情况中用到：

- 任意指令都可能引发异常，此时必须知道异常指令的PC值（exception pc, epc）。
- 为了目标地址计算，分支和跳转指令也需要知道它们的PC。
- 跳转-寄存器指令必须同时知道它们自己的PC以及下一条指令的PC，以验证前端是否预测到了正确的JR目标。

然而想储存PC信息有着很大的开销，因此分支和跳转指令在**读寄存器**阶段获取ROB的“PC堆”用于分支单元，ROB也有以下两项优化：

* 每个ROB行只储存一个PC。
* PC堆储存在两栏中，使得一个读端口能够同时读取两条连续的条目（用于与JR指令）。   ？



### 参数化 - 回滚与单周期重置‎

重置重命名映射表的行为是可参数化的。第一个可选项是每个周期回滚ROB的一行，以放松重命名阶段（MIPS R10k的做法）。对于每条指令，僵死的物理目的寄存器将被它的逻辑目的标识符写回映射表。

有一种可用的更快的单周期重置方法，它使用了另外的**追踪重命名表提交状态的重命名快照**。这个提交映射表将在指令提交的时候更新。在代码中用enableCommitMapTable的值指示是否启用这种方法。

