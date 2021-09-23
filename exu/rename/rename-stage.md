# rename-stage
—— 重命名逻辑的整体连接。描述了整个重命名阶段的操作流程，包括顶层的接口（与系统以及其他级的互联信号）、子模块的互联（Maptable、Freelist以及Busytable）以及流水线的时序逻辑等。其中，流水线的状态、线网以及逻辑主要在抽象类中定义，而子模块互联以及BypassAllocations的逻辑在RenameStage类中给出了具体的实现。整个阶段的示意图如下：

![stage1](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/stage1.jpg)



工作流程简述：在流水线中占用两个时钟周期，第一个时钟周期中 FreeList 和 MapTable 模块将译码级传入的微指令中的逻辑寄存器映射为物理寄存器，第二个时钟周期通过 BusyTable 模块判断源操作数寄存器是否可读。



### 显式重命名与隐式重命名

BOOM 处理器采用的是显式重命名方案。

> 1. map_table 记录逻辑寄存器与物理寄存器之间的映射关系；
>
> 2. free_list记录物理寄存器的空闲状态；
>
> 3. busy_table 记录寄存器是否可读;
>
> 4. ROB 不记录指令的结果，即将提交的数据和处于推测状态的数据都保存在物理寄存器中，因此物理寄存器数目要高于逻辑寄存器数目。
>
> 5. 当一条指令发起重命名请求时，通过索引map_table获取其源操作数逻辑寄存器对应的物理寄存器，由free_list分配一个空闲的物理寄存器作为指令的目的寄存器，最后通过busy_table 判断指令的源操作数寄存器是否可读，如果可读指令将被发射(Issue)。
>
>    ![rename1](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/rename1.jpg)

隐式重命名方案见于 Pentium 3 ，Pentium Pro 等处理器。

> 1. ROB 保存正在执行、尚未提交的指令的结果；
>
> 2. ARF(ISA Register File) 保存已经提交的指令中即将写入寄存器中的值。 ARF 只保存已经提交的指令的值，处于 “推测” 状态的指令的值由 ROB 保存，因此需要的物理寄存器数量与逻辑寄存器数量相同。
>
> 3. 建立一个映射表，记录操作数在 ROB 中的位置。由于流水线中后续指令与已经提交的指令可能有相同的目的寄存器(意味着该寄存器将被修改)，映射表需要增加一个表项，记录对应寄存器的最新值保存在 ROB 还是 ARF 中，这一设计为实现数据前馈、消除 RAW 冲突创造了条件。
>
> 4. 隐式重命名方案不需要 free_list 来记录物理寄存器状态，指令被写进 ROB 即完成重命名。
>
> 5. 相比于显式重命名，隐式重命名需要的物理寄存器数目更少，但每个操作数在其生命周期中需要保存在 ROB 和 ARF 两个位置，读取数据的复杂度较高、功耗更高
>
>    ![rename2](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/rename2.jpg)



### 映射表的两种结构

对于显式的重命名方法，必须设计一个寄存器阵列用于保存逻辑寄存器和物理寄存器之间的映射关系，称为映射表。映射表的结构又可以分为两种：

- RAM：Random Addressed Memory，保存每个逻辑寄存器的映射关系，表项数目等于逻辑寄存器数目 。
- CAM：Conent Addressed Memory，保存每个物理寄存器的映射关系，表项数目等于物理寄存器数目。

在 CAM 结构中，由于一个逻辑寄存器可能与多个物理寄存器存在映射关系，必须对表中的每一项加标志位，表示是否为最新的映射关系。RAM 结构的重命名映射表表项数目更少(因为物理寄存器个数大于逻辑寄存器)，且更适合乱序执行的处理器。 BOOM 的重命名阶段使用的是 RAM 结构的映射表。



### 代码开头的说明部分

支持1或2个周期的延迟（也就是说，ren1和ren2之间的直接通路与寄存器）。

 - ren1：读映射表，在空闲列表中分配一个新的物理寄存器。

 - ren2：为操作数物理寄存器读繁忙列表。

Ren1数据作为一个输出直接提供给ROB。

```scala
// Supports 1-cycle and 2-cycle latencies. (aka, passthrough versus registers between ren1 and ren2).
//    - ren1: read the map tables and allocate a new physical register from the freelist.
//    - ren2: read the busy table for the physical operands.
//
// Ren1 data is provided as an output to be fed directly into the ROB.
```
```scala
package boom.exu
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import boom.common._
import boom.util._
```



### RenameStageIO 类

与重命名逻辑交互的IO端口。

```scala
/**
 * IO bundle to interface with the Register Rename logic
 *
 * @param plWidth pipeline width
 * @param numIntPregs number of int physical registers
 * @param numFpPregs number of FP physical registers
 * @param numWbPorts number of int writeback ports
 * @param numWbPorts number of FP writeback ports
 */
    class RenameStageIO(
    val plWidth: Int,		// 流水线宽度（指令条数）
    val numPhysRegs: Int,	// 物理寄存器个数
    val numWbPorts: Int)	// 写回端口的数量（整数和浮点的相同？）
    (implicit p: Parameters) extends BoomBundle
```



### DebugRenameStageIO 类

用于重命名调试的IO端口。

```scala
/**
 * IO bundle to debug the rename stage
 */
    class DebugRenameStageIO(val numPhysRegs: Int)(implicit p: Parameters) extends BoomBundle
    {
    val freelist  = Bits(numPhysRegs.W)	// 空闲列表
    val isprlist  = Bits(numPhysRegs.W)	// 神秘端口
    val busytable = UInt(numPhysRegs.W)	// 繁忙列表
    }
```

两个Bits类型，一个UInt类型的原因：？



### AbstractRenameStage 抽象类

#### 1.参数

```scala
abstract class AbstractRenameStage( // 1.参数
  plWidth: Int,
  numPhysRegs: Int,
  numWbPorts: Int)
  (implicit p: Parameters) extends BoomModule
{
    // 2.IO端口
    
    // 3.BypassAllocations虚拟函数
    
    // 4.成员和功能
}
```
#### 2.IO端口

两个子类的IO端口均与此相同。

共3个输出，但在逻辑功能中只实现了ren2_mask。

```scala
  val io = IO(new Bundle {
    val ren_stalls = Output(Vec(plWidth, Bool()))
      // 【输出】重命名停顿信号。plWidth个Bool值，分别代表每条指令是否停顿。
      
      
    val kill = Input(Bool())
      // 【输入】从系统输入的清除信号
    
      
    val dec_fire  = Input(Vec(plWidth, Bool())) // will commit state updates（将提交状态更新）
      // 【输入】译码级输出信号
    val dec_uops  = Input(Vec(plWidth, new MicroOp()))
      // 【输入】译码级输出的微指令
      
    
    // physical specifiers available AND busy/ready status available.（可用的物理标识符和繁忙/就绪状态）
    val ren2_mask = Vec(plWidth, Output(Bool())) // mask of valid instructions（可用结构的掩码）
      // 【输出】重命名级输出的微指令组的有效位信息
    val ren2_uops = Vec(plWidth, Output(new MicroOp()))
      // 【输出】重命名2级输出的微指令组
    
      
    // branch resolution (execute)       （执行阶段产生的分支）
    val brupdate = Input(new BrUpdateInfo())
      // 【输入】分支更新信息
    
      
    val dis_fire  = Input(Vec(coreWidth, Bool()))
      // 【输入】派遣阶段各条指令的就绪信号
      // coreWidth指解码、整数重命名、ROB和提交的宽度
      // parameter.scala 中有定义 val coreWidth = decodeWidth，根据规模可为1到5
    val dis_ready = Input(Bool())
      // 【输入】来自派遣阶段的输入的总就绪信号
    
      
    // wakeup ports （唤醒的端口组）
    val wakeups = Flipped(Vec(numWbPorts, Valid(new ExeUnitResp(xLen))))
      // 【输入】连接到执行单元的回复，用于唤醒寄存器
      // 共numWbPort个Valid端口，端口内容为ExeUnitResp类，包括一个宽度为xLen的数据，一个指示这条指令
      //  是否是预测的信号，还有一个浮点标志响应（内含微指令和一个标志）
    
      
    // commit stage （来自提交阶段的输入）
    val com_valids = Input(Vec(plWidth, Bool()))
      // 【输入】提交阶段中微指令组的有效信号
    val com_uops = Input(Vec(plWidth, new MicroOp()))
      // 【输入】提交阶段中的微指令组
    val rbk_valids = Input(Vec(plWidth, Bool()))
      // 【输入】提交阶段中指令组的回滚有效信号
    val rollback = Input(Bool())
      // 【输入】回滚信号
    
      
    val debug_rob_empty = Input(Bool())
      // 【输入】debug用，指示ROB是否为空的信号
    val debug = Output(new DebugRenameStageIO(numPhysRegs))
      // 【输出】debug用，类型为上文中定义的debugIO端口类
  })
```
#### 3.BypassAllocations 虚拟函数

在两个子类RenameStage和PredRenameStage中有不同的具体实现，见下文。

输入：一条微指令uop，一个微指令序列older_uops，一个Bool值序列alloc_reqs。（两个序列长度应当相等）

输出：一条微指令。

```scala
  def BypassAllocations(uop: MicroOp, older_uops: Seq[MicroOp], alloc_reqs: Seq[Bool]): MicroOp
```
#### 4.成员和逻辑功能

实现的功能：

第1级：接收译码级的输出。

第2级：先回应是否清除或派遣，再对第1级的指令组进行旁路逻辑和分支的处理。

```scala
  //-------------------------------------------------------------
  // Pipeline State & Wires （流水线状态与线网）

  // Stage 1 （第一级）  功能见下方
  val ren1_fire       = Wire(Vec(plWidth, Bool()))
  val ren1_uops       = Wire(Vec(plWidth, new MicroOp))


  // Stage 2 （第二级）
  val ren2_fire       = io.dis_fire		// 【输入】派遣阶段各条指令的就绪信号
  val ren2_ready      = io.dis_ready	// 【输入】来自派遣阶段的输入的总就绪信号
  val ren2_valids     = Wire(Vec(plWidth, Bool()))	// 向【输出】ren2_mask的寄存器更新信号的线网
	// 这两个是由临时保存寄存器引出的线网，见下方
  val ren2_uops       = Wire(Vec(plWidth, new MicroOp))
  val ren2_alloc_reqs = Wire(Vec(plWidth, Bool()))


  //-------------------------------------------------------------
  // pipeline registers （流水线寄存器）

  // 两个第一级的变量，接收译码级的输出
  for (w <- 0 until plWidth) {	// 遍历每条指令
    ren1_fire(w)          := io.dec_fire(w)
    ren1_uops(w)          := io.dec_uops(w)
  }

  // 第二级的变量，
  for (w <- 0 until plWidth) {	// 遍历每条指令
    val r_valid  = RegInit(false.B)	// 临时保存输出微指令有效信号（初始为0）
    val r_uop    = Reg(new MicroOp)	// 临时保存输出微指令的寄存器
    val next_uop = Wire(new MicroOp) // 上面寄存器引出的线网

    next_uop := r_uop // 默认情况下r_uop一直保持
    
    when (io.kill) {			// 有系统清除信号
      r_valid := false.B		//     -> r_valid置0
    } .elsewhen (ren2_ready) {	//   无系统清除信号，来自派遣阶段的输入就绪  
      r_valid := ren1_fire(w)	//     -> 下一条微指令的有效信号置为译码的有效信号     
      next_uop := ren1_uops(w)	//     -> 下一条微指令置为译码完成的指令
    } .otherwise {				//  无系统清除信号，来自派遣阶段的输入未就绪
      r_valid := r_valid && !ren2_fire(w) // clear bit if uop gets dispatched（如果微指令派遣，清除）
        					  //     -> 派遣即fire为1，valid强制置0，否则保持
      next_uop := r_uop		   //     -> 下条指令仍保持
    }
    
    r_uop := GetNewUopAndBrMask(BypassAllocations(next_uop, ren2_uops, ren2_alloc_reqs), io.brupdate)
      // 通过两个辅助函数，将next_uop经过旁路处理和分支处理后，赋给r_uop
      // GetNewUopAndBrMask在util.scala中有定义，输入一条微指令和一个分支更新信息，返回一条新的微指令（包含	  //  新的分支掩码）
    
    ren2_valids(w) := r_valid	// 由寄存器引出线网，下同
    ren2_uops(w)   := r_uop		// 经过旁路和分支处理后，得到第2级的微指令
  }

  //-------------------------------------------------------------
  // Outputs

  io.ren2_mask := ren2_valids


}	// 整个AbstractRenameStage抽象类的结束
```



### RenameStage 类 （由AbstractRenameStage抽象类继承）

在抽象类的基础上

* 实现了辅助函数BypassAllocations以完成旁路逻辑；
* 添加了三个实例化的模块——重命名表、空闲列表、繁忙列表，功能如下：
  * 重命名表——对于第1级传入的微指令，对三个源逻辑寄存器和一个目的逻辑寄存器进行映射译码。再根据回滚情况，选择重映射的方式，最终更新第1级指令的源物理寄存器和僵死目的物理寄存器。
  * 空闲列表——接收来自提交和回滚的释放物理寄存器请求，为第2级的微指令分配目的物理寄存器。
  * 繁忙列表——分配好目的物理寄存器后（空闲列表任务完成后），为第2级的微指令填写原物理寄存器繁忙信息。
* 实现抽象类未实现的debug端口和剩下两个输出端口。

#### 0.注释

连接映射表、空闲列表、繁忙列表的重命名阶段。

在浮点数流水线和正常执行的流水线中都可以使用。

```scala
/**	0.注释
 * Rename stage that connets the map table, free list, and busy table.
 * Can be used in both the FP pipeline and the normal execute pipeline.
 *
 * @param plWidth pipeline width
 * @param numWbPorts number of int writeback ports
 * @param numWbPorts number of FP writeback ports
 */
```
####  1.参数、继承、常用变量

 ```scala
 class RenameStage(	// 1.参数、继承、常用变量
    plWidth: Int,
    numPhysRegs: Int,
    numWbPorts: Int,
    float: Boolean)
    (implicit p: Parameters) extends AbstractRenameStage(plWidth, numPhysRegs, numWbPorts)(p)
 {
    val pregSz = log2Ceil(numPhysRegs)
    val rtype = if (float) RT_FLT else RT_FIX	// 解码阶段的控制信号，分别为01和00
     
     // 2.BypassAllocationss函数的具体实现
     
     // 3.重命名结构（实例化、提交与回滚）
     
     // 4.重命名表（重映射表）的连接
     
     // 5.空闲列表的连接
     
     // 6.繁忙列表的连接
     
     // 7.整体的输出信号
     
     // 8.整体的调试信号
 }
 ```
#### 2.BypassAllocationss函数的具体实现

输入一个新指令和一个旧的指令组（当前ren2中的部分指令），检测是否存在旁路情况（新指令相关的寄存器是否被旧指令组中的某一条写），若存在，修改新指令相关成员（物理寄存器及其繁忙信息）以完成旁路关系。

```scala
  //-------------------------------------------------------------
  // Helper Functions （辅助函数）

  def BypassAllocations(uop: MicroOp, older_uops: Seq[MicroOp], alloc_reqs: Seq[Bool]): MicroOp = {
    val bypassed_uop = Wire(new MicroOp)
    bypassed_uop := uop	// 用线网连接当前函数输入的新微指令（要处理的指令）

      
    val bypass_hits_rs1 = (older_uops zip alloc_reqs) map { case (r,a) => a && r.ldst === uop.lrs1 }
    val bypass_hits_rs2 = (older_uops zip alloc_reqs) map { case (r,a) => a && r.ldst === uop.lrs2 }
    val bypass_hits_rs3 = (older_uops zip alloc_reqs) map { case (r,a) => a && r.ldst === uop.lrs3 }
    val bypass_hits_dst = (older_uops zip alloc_reqs) map { case (r,a) => a && r.ldst === uop.ldst }
      // 每个变量都是plWidth个Bool值，每一位代表：新指令的寄存器（rs1~3、dst）是否与ren2中对应指令的
      //  目的寄存器相等（还需要对应指令的alloc_reqs为1，即这条指令要求分配了这个目的寄存器）。
      // 相当于检测RAW和WAW。
    
      
    val bypass_sel_rs1 = PriorityEncoderOH(bypass_hits_rs1.reverse).reverse
    val bypass_sel_rs2 = PriorityEncoderOH(bypass_hits_rs2.reverse).reverse
    val bypass_sel_rs3 = PriorityEncoderOH(bypass_hits_rs3.reverse).reverse
    val bypass_sel_dst = PriorityEncoderOH(bypass_hits_dst.reverse).reverse
      // 将对应的hit变量反转，保留最右一位1，再反转。相当于保留对应的hit变量的最左一位1（最近的指令的）。
    
      
    val do_bypass_rs1 = bypass_hits_rs1.reduce(_||_)
    val do_bypass_rs2 = bypass_hits_rs2.reduce(_||_)
    val do_bypass_rs3 = bypass_hits_rs3.reduce(_||_)
    val do_bypass_dst = bypass_hits_dst.reduce(_||_)
      // 将对应的hit变量归约。指示新指令的这个寄存器（rs1~3、dst）是否被旧指令组中的至少一个写。
    
      
    val bypass_pdsts = older_uops.map(_.pdst)
      // 旧指令组的目的物理寄存器
      
    
    when (do_bypass_rs1) { bypassed_uop.prs1       := Mux1H(bypass_sel_rs1, bypass_pdsts) }
    when (do_bypass_rs2) { bypassed_uop.prs2       := Mux1H(bypass_sel_rs2, bypass_pdsts) }
    when (do_bypass_rs3) { bypassed_uop.prs3       := Mux1H(bypass_sel_rs3, bypass_pdsts) }
    when (do_bypass_dst) { bypassed_uop.stale_pdst := Mux1H(bypass_sel_dst, bypass_pdsts) }
      // 存在旁路情况时，将返回的微指令的对应寄存器（rs1~3、dst）置为触发旁路的旧指令的目的寄存器
    
      
    bypassed_uop.prs1_busy := uop.prs1_busy || do_bypass_rs1
    bypassed_uop.prs2_busy := uop.prs2_busy || do_bypass_rs2
    bypassed_uop.prs3_busy := uop.prs3_busy || do_bypass_rs3
      // 返回的微指令的对应寄存器繁忙的条件： 这条指令传进来时就已经繁忙（？） 或者 有旁路（别的指令要写）
    
    if (!float) {	// 不是浮点数指令
      bypassed_uop.prs3      := DontCare	// 第三个物理寄存器不关心
      bypassed_uop.prs3_busy := false.B		// 且视为空闲
    }
    
    bypassed_uop	// 返回微指令
  }
```
#### 3.重命名结构（实例化、提交与回滚）

```scala
  //-------------------------------------------------------------
  // Rename Structures （重命名结构）

  // 实例化三个模块
  val maptable = Module(new RenameMapTable(
    plWidth,
    32,
    numPhysRegs,
    false,
    float))
  val freelist = Module(new RenameFreeList(
    plWidth,
    numPhysRegs,
    if (float) 32 else 31))
  val busytable = Module(new RenameBusyTable(
    plWidth,
    numPhysRegs,
    numWbPorts,
    false,
    float))


  val ren2_br_tags    = Wire(Vec(plWidth, Valid(UInt(brTagSz.W))))
	// 重命名第二阶段分支标签组。

  // Commit/Rollback （提交/回滚）
  val com_valids      = Wire(Vec(plWidth, Bool()))
	// 每条指令的提交有效信号
  val rbk_valids      = Wire(Vec(plWidth, Bool()))
	// 每条指令的回滚有效信号

  for (w <- 0 until plWidth) {	// 遍历每条指令
    ren2_alloc_reqs(w)    := ren2_uops(w).ldst_val && ren2_uops(w).dst_rtype === rtype && ren2_fire(w)
      // 第2级的第w条指令需要重新分配物理寄存器，当且仅当：
      //   指令有目的寄存器、目的寄存器类型正确、这条指令可以派遣。
    ren2_br_tags(w).valid := ren2_fire(w) && ren2_uops(w).allocate_brtag
      // 第2级的第w条指令的分支标签有效，当且仅当：
      //   这条指令可以派遣、这条资料确实需要分配一个分支标签。

    com_valids(w)         := io.com_uops(w).ldst_val && io.com_uops(w).dst_rtype === rtype && io.com_valids(w)
      // 提交信号，需要有目的寄存器、类型正确、输入信号中指示可以提交
    rbk_valids(w)         := io.com_uops(w).ldst_val && io.com_uops(w).dst_rtype === rtype && io.rbk_valids(w)
      // 回滚信号，类似提交信号
      
    ren2_br_tags(w).bits  := ren2_uops(w).br_tag
      // 保存这条指令的分支标签
  }
```
上面for循环中涉及到的两个微指令成员的解释：

ldst_val : Bool值，指示指令是否有目的寄存器。对 储存 和 目的寄存器为x0 的指令，其值为0。

allocate_brtag : Bool值，值为 (is_br && !is_sfb) || is_jalr ，指示是否需要为这条指令分配一个分支标签。

​                          是分支但不用sfb优化 或者 是跳转指令 时，其值为1（sfb分支不会得到掩码，而是得到一个预测位）

#### 4.重命名表（映射表）的连接

```scala
  //-------------------------------------------------------------
  // Rename Table （重命名表）

  // Maptable inputs. （映射表输入）
  val map_reqs   = Wire(Vec(plWidth, new MapReq(lregSz))) // 映射请求，lrs1~3和ldst
  val remap_reqs = Wire(Vec(plWidth, new RemapReq(lregSz, pregSz))) // 重映射请求，prs1~3和stale_pdst

  // Generate maptable requests. (生成映射表请求)
  for ((((ren1,ren2),com),w) <- ren1_uops zip ren2_uops zip io.com_uops.reverse zipWithIndex) {
      // 可以看做同时遍历重命名两级中的指令
    map_reqs(w).lrs1 := ren1.lrs1
    map_reqs(w).lrs2 := ren1.lrs2
    map_reqs(w).lrs3 := ren1.lrs3
    map_reqs(w).ldst := ren1.ldst
      // 根据第一级填写映射请求组

    remap_reqs(w).ldst := Mux(io.rollback, com.ldst      , ren2.ldst)
    remap_reqs(w).pdst := Mux(io.rollback, com.stale_pdst, ren2.pdst)
      // 若回滚，则要重命名的逻辑寄存器是commit级传过来的微指令中的目的寄存器，需要将其重映射到僵死的
      //     目的寄存器（即恢复之前的映射关系）
      // 若不回滚，则要重命名的是当前译码级指令的目的寄存器，重映射的对象由Freelist在第二级（ren2）
      //     一开始写到ren2.pdst中。     
  }
  ren2_alloc_reqs zip rbk_valids.reverse zip remap_reqs map {
    case ((a,r),rr) => rr.valid := a || r}
	// 重映射有效 需要以下两个条件之一：
	// 当前译码级传来的指令需要分配新的物理寄存器 或 当前提交级传来的指令需要回滚操作 

  // Hook up inputs. （连接到映射表的输入）
  maptable.io.map_reqs    := map_reqs
  maptable.io.remap_reqs  := remap_reqs
  maptable.io.ren_br_tags := ren2_br_tags
  maptable.io.brupdate    := io.brupdate
  maptable.io.rollback    := io.rollback

  // Maptable outputs. （映射表输出）
  for ((uop, w) <- ren1_uops.zipWithIndex) { // 遍历每条指令
    val mappings = maptable.io.map_resps(w) // 储存映射表输出的映射回复组

    uop.prs1       := mappings.prs1
    uop.prs2       := mappings.prs2
    uop.prs3       := mappings.prs3 // only FP has 3rd operand
    uop.stale_pdst := mappings.stale_pdst
      // 完成重命名第一级中指令的寄存器映射
  }
```
#### 5.空闲列表的连接

```scala
  //-------------------------------------------------------------
  // Free List （空闲列表）

  // Freelist inputs. （空闲列表输入）
  freelist.io.reqs := ren2_alloc_reqs // 请求组为重命名第2级分配寄存器的请求
  
  freelist.io.dealloc_pregs zip com_valids zip rbk_valids map
    {case ((d,c),r) => d.valid := c || r}
	// 当需要提交或者回滚时，对应释放物理寄存器请求的valid位置1
  
  freelist.io.dealloc_pregs zip io.com_uops map
    {case (d,c) => d.bits := Mux(io.rollback, c.pdst, c.stale_pdst)}
	// 释放物理寄存器请求中的物理寄存器 是 提交级微指令的 物理目的寄存器（回滚时，因为无效）
	//  或僵死物理目的寄存器（不回滚时，因为这样指令提交，对应的寄存器就空闲了）
  
  freelist.io.ren_br_tags := ren2_br_tags // 分支标签组，使用重命名第2级的分支标签组
  freelist.io.brupdate := io.brupdate // 分支更新信息
  freelist.io.debug.pipeline_empty := io.debug_rob_empty // 流水线是否为空，使用ROB是否为空的信号

  assert (ren2_alloc_reqs zip freelist.io.alloc_pregs map {case (r,p) => !r || p.bits =/= 0.U} reduce (_&&_),  "[rename-stage] A uop is trying to allocate the zero physical register.")
	// 当要求分配的寄存器中有0号时发出相应警告

  // Freelist outputs. （空闲列表输出）
  for ((uop, w) <- ren2_uops.zipWithIndex) { // 遍历重命名第2级的指令
    val preg = freelist.io.alloc_pregs(w).bits // 保存空闲列表输出的已分配物理寄存器的地址
    uop.pdst := Mux(uop.ldst =/= 0.U || float.B, preg, 0.U)
      // 将重命名第2级中指令的物理目的寄存器：
      // 	若这个寄存器不是0号寄存器 或 这条指令是浮点指令，置为空闲列表分配的物理寄存器
      //    否则置为0号寄存器
  } 
```
#### 6.繁忙列表的连接

```scala
  //-------------------------------------------------------------
  // Busy Table （繁忙列表）

  busytable.io.ren_uops := ren2_uops  // expects pdst to be set up.（此时物理目的寄存器应该已分配好）
  busytable.io.rebusy_reqs := ren2_alloc_reqs // 重繁忙请求，使用重命名第2级分配请求
  
  busytable.io.wb_valids := io.wakeups.map(_.valid) // 外部唤醒信号使能繁忙列表的写回
  busytable.io.wb_pdsts := io.wakeups.map(_.bits.uop.pdst) // 写回目的物理寄存器的地址

  
  assert (!(io.wakeups.map(x => x.valid && x.bits.uop.dst_rtype =/= rtype).reduce(_||_)),
   "[rename] Wakeup has wrong rtype.")
	// 输入的有效唤醒信息中，微指令的目的寄存器类型 和 当前重命名模块的寄存器类型 不一致时，
	// 发出警告：唤醒的寄存器类型错误。

  
  for ((uop, w) <- ren2_uops.zipWithIndex) { // 遍历重命名第2级的微指令
    val busy = busytable.io.busy_resps(w) // 保存繁忙列表输出的繁忙回复

    uop.prs1_busy := uop.lrs1_rtype === rtype && busy.prs1_busy
    uop.prs2_busy := uop.lrs2_rtype === rtype && busy.prs2_busy
    uop.prs3_busy := uop.frs3_en && busy.prs3_busy
      // 完成重命名第2级中微指令的源寄存器的繁忙信息
      // prs1和2还需要类型正确，以及prs3要求是浮点指令（？）
    
    val valid = ren2_valids(w)
    assert (!(valid && busy.prs1_busy && rtype === RT_FIX && uop.lrs1 === 0.U), "[rename] x0 is busy??")
    assert (!(valid && busy.prs2_busy && rtype === RT_FIX && uop.lrs2 === 0.U), "[rename] x0 is busy??")
      // 检测非浮点指令的源逻辑寄存器为x0时，是否分配的物理源寄存器为繁忙。
  }
```
#### 7.整体的输出信号

```scala
  //-------------------------------------------------------------
  // Outputs （输出）

  for (w <- 0 until plWidth) { // 遍历每条指令
    val can_allocate = freelist.io.alloc_pregs(w).valid //  保存空闲列表输出的分配信息的有效位

    // Push back against Decode stage if Rename1 can't proceed. （如果重命名第1级无法继续，返回解码）
    io.ren_stalls(w) := (ren2_uops(w).dst_rtype === rtype) && !can_allocate
      // 类型正确但无法分配，对应的停顿位置1。
    
    val bypassed_uop = Wire(new MicroOp) // 定义线网，上面的信息是有旁路的指令
    if (w > 0) bypassed_uop := BypassAllocations(ren2_uops(w), ren2_uops.slice(0,w), ren2_alloc_reqs.slice(0,w))
      // 对于这批指令的第w(w>0)条，传入的旧指令列表是第0到第(w-1)条，用辅助函数将这条指令进行旁路优化修改
    else       bypassed_uop := ren2_uops(w)
      // 对这批指令的第0条，直接置为重命名第2级的第0条指令（不优化）
      
      // 由上面的描述可知，只支持同一批指令间的旁路优化
    
    io.ren2_uops(w) := GetNewUopAndBrMask(bypassed_uop, io.brupdate)
      // 再根据分支更新信息生成新的重命名第2级的指令
  }
```
  #### 8.整体的调试信号
```scala
  //-------------------------------------------------------------
  // Debug signals （debug信号，这里只是将信号继续往外引而已）

  io.debug.freelist  := freelist.io.debug.freelist
  io.debug.isprlist  := freelist.io.debug.isprlist
  io.debug.busytable := busytable.io.debug.busytable

} // 整个RenameStage类的结束
```



### PredRenameStage 类（同样由AbstractRenameStage抽象类继承）

预测重命名阶段。不例化已实现的三个表。

#### 1.参数

```scala
class PredRenameStage(	// 1.参数
  plWidth: Int,
  numPhysRegs: Int,
  numWbPorts: Int)
  (implicit p: Parameters) extends AbstractRenameStage(plWidth, numPhysRegs, numWbPorts)(p)
{
    // 2.BypassAllocations函数的具体实现
    
    // 3.私有变量与逻辑功能
}
```
#### 2.BypassAllocations函数的具体实现

完全不做旁路处理。

```scala
  def BypassAllocations(uop: MicroOp, older_uops: Seq[MicroOp], alloc_reqs: Seq[Bool]): MicroOp = {
    uop
  }
```
#### 3.成员与逻辑功能

```scala
  ren2_alloc_reqs := DontCare
	// 不关心第2级的分配请求


  // ftqSz 是 取指目标队列的长度 （一般为16）
  val busy_table = RegInit(VecInit(0.U(ftqSz.W).asBools))
	// 简易繁忙列表。长度为ftqSz的寄存器堆，每一位对应取值目标队列中的一条指令。
  val to_busy = WireInit(VecInit(0.U(ftqSz.W).asBools))
	// 简易使繁忙列表，线网。
  val unbusy = WireInit(VecInit(0.U(ftqSz.W).asBools))
	// 简易空闲列表，线网。


  // ftq_idx 是 取指队列中的索引，通过它可以计算出指令的pc
  val current_ftq_idx = Reg(UInt(log2Ceil(ftqSz).W))	// 当前取值队列索引
  var next_ftq_idx = current_ftq_idx	// 变量，初始化为当前取值队列索引


  for (w <- 0 until plWidth) {	// 遍历每条指令
    io.ren2_uops(w) := ren2_uops(w)	// 直接将不考虑旁路，仅考虑分支处理后的第2级微指令作为输出

    val is_sfb_br = ren2_uops(w).is_sfb_br && ren2_fire(w)
      // 指示这条指令是否写一个预测
      // is_sfb_br 是微指令的成员，值为 is_br && is_sfb && enableSFBOpt.B
      //   其中: is_br 指示这条指令是分支指令还是一条正常的PC+4的指令
      //         is_sfb 指示这条指令是否是sfb
      //         enableSFBOpt 指示是否打开了sfb优化
      //         sfb ： ？
    val is_sfb_shadow = ren2_uops(w).is_sfb_shadow && ren2_fire(w)
      // 指示这条指令是否被预测
      // is_sfb_shadow 同样是微指令的成员，值为 !is_br && is_sfb && enableSFBOpt.B
      
    val ftq_idx = ren2_uops(w).ftq_idx // 保存当前指令在取值队列中的索引  
      
    when (is_sfb_br) { // 写预测，有sfb优化
      io.ren2_uops(w).pdst := ftq_idx // 将索引作为分配给这条指令的目的物理寄存器的地址
      to_busy(ftq_idx) := true.B // 相应地更新简易使繁忙列表
    }
    next_ftq_idx = Mux(is_sfb_br, ftq_idx, next_ftq_idx)
      // 如果这一步进行了，将这条指令的索引作为下一个索引，否则仍保持为当前索引
        
    when (is_sfb_shadow) { // 被预测，有sfb优化
        // ppred 是微指令的成员，指sfb优化预测的物理寄存器的地址，长度与索引相同。
      io.ren2_uops(w).ppred := next_ftq_idx
      io.ren2_uops(w).ppred_busy := (busy_table(next_ftq_idx) || to_busy(next_ftq_idx)) && !unbusy(next_ftq_idx)
    }
  }


  for (w <- 0 until numWbPorts) { // 遍历写回端口（要写回的所有指令）
    when (io.wakeups(w).valid) {
      unbusy(io.wakeups(w).bits.uop.pdst) := true.B 
        // 唤醒有效（可以写回），将目的物理寄存器填到简易空闲列表中
    }
  }


  current_ftq_idx := next_ftq_idx
	// 更新当前的取值队列索引


  busy_table := ((busy_table.asUInt | to_busy.asUInt) & ~unbusy.asUInt).asBools
	// 更新简易繁忙列表


}	// 整个PredRenameStage类的结束
```
