# rename-freelist

—— 记录并维护物理寄存器的使用情况。当译码级的指令提出分配物理寄存器的要求时，Freelist选出空闲的物理寄存器并更新状态。当提交级的指令返回提交或是回滚信号时，Freelist将释放掉占用的物理寄存器并更新状态。此外，Freelist还维护着分支预测快照表，遇到新开辟的分支槽时，则将该分支用到的所有寄存器记录于对应的列表内，如果预测失误将一并释放。



```scala
package boom.exu
import chisel3._
import chisel3.util._
import boom.common._
import boom.util._
import freechips.rocketchip.config.Parameters
```



### RenameFreeList 类（freelist主体）

重命名空闲列表。

#### 1.参数

```scala
class RenameFreeList(	// 1.参数
  val plWidth: Int,		// 指令个数
  val numPregs: Int,	// 物理寄存器个数
  val numLregs: Int)	// 逻辑寄存器个数
  (implicit p: Parameters) extends BoomModule
{
  private val pregSz = log2Ceil(numPregs)	// 物理寄存器二进制地址位数
  private val n = numPregs					// 物理寄存器个数
     
  // 2.IO端口
    
  // 3.成员
    
  // 4.逻辑功能
    
}
```
#### 2.IO端口

```scala
  val io = IO(new BoomBundle()(p) {
      
    // Physical register requests.（物理寄存器请求组）
    val reqs          = Input(Vec(plWidth, Bool()))
      // 【输入】请求组。plWidth个Bool值，分别代表每一条指令的请求是否有效。
    val alloc_pregs   = Output(Vec(plWidth, Valid(UInt(pregSz.W))))
      // 【输出】分配出去的plWidth个物理寄存器的二进制地址。

      
    // Pregs returned by the ROB.（ROB返回的物理寄存器组）
    val dealloc_pregs = Input(Vec(plWidth, Valid(UInt(pregSz.W))))
      // 【输入】释放的plWidth个物理寄存器的二进制地址组。
    
      
    // Branch info for starting new allocation lists.（启动新分配列表的分支信息）
    val ren_br_tags   = Input(Vec(plWidth, Valid(UInt(brTagSz.W))))
      // 【输入】分支标签信息。plWidth个Valid端口，内含分支标签的二进制序号，valid为指示是否有效。
      
      
    // Mispredict info for recovering speculatively allocated registers.（恢复特定已分配寄存器需要的分支误判信息）
    val brupdate        = Input(new BrUpdateInfo)
      // 【输入】分支更新信息，BrUpdataInfo见最后。
    
      
    val debug = new Bundle {
      val pipeline_empty = Input(Bool()) // 【输入】指示流水线是否为空
      val freelist = Output(Bits(numPregs.W)) // 【输出】debug空闲列表
      val isprlist = Output(Bits(numPregs.W)) // 【输出】？
    }
  })
```
#### 3.成员
```scala
  // The free list register array and its branch allocation lists.（空闲列表寄存器数组和它的分支分配列表）
  val free_list = RegInit(UInt(numPregs.W), ~(1.U(numPregs.W)))
	// 空闲列表。每一位代表一个物理寄存器，初始值为111...10，即0号寄存器永远“不空闲”，不可被分配。
  val br_alloc_lists = Reg(Vec(maxBrCount, UInt(numPregs.W)))
	// 分支分配列表组。为每一条分支储存对应的空闲列表（快照）。


  // Select pregs from the free list.（选择空闲列表的物理寄存器）
	// 这两个变量用于生成sel_mask
  val sels = SelectFirstN(free_list, plWidth)
	// 已选择的plWidth个物理寄存器的独热码地址组。
  val sel_fire  = Wire(Vec(plWidth, Bool()))
	// 指示对应物理寄存器是否能真正分配出去的信号。plWidth个Bool值。


  // Allocations seen by branches in each pipeline slot.（每个流水线槽中分支看到的分配）
  val allocs = io.alloc_pregs map (a => UIntToOH(a.bits))
	// 将输出的已分配的物理寄存器地址组转换为独热码形式，方便处理。
  val alloc_masks = (allocs zip io.reqs).scanRight(0.U(n.W)) { case ((a,r),m) => m | a & Fill(n,r) }
	// 分配物理寄存器的掩码组，(plWidth+1)行，第i行
	//   储存上一批指令已处理到第i条时，第i条和之后的指令分配出去的物理寄存器的独热码地址集合，用于恢复
	//   最后一行总是全为0
	// 原理见下方


  // Masks that modify the freelist array.（修正空闲列表数组的掩码）
  val sel_mask = (sels zip sel_fire) map { case (s,f) => s & Fill(n,f) } reduce(_|_)
	// 已选择的plWidth个物理寄存器中，有效的独热码地址的总和。
	// 原理见下方
  val br_deallocs = br_alloc_lists(io.brupdate.b2.uop.br_tag) & Fill(n, io.brupdate.b2.mispredict)
	// 分支释放列表。
	// 在分支分配列表组中定位并保存预测出错的指令对应的空闲列表快照。
  val dealloc_mask = io.dealloc_pregs.map(d => UIntToOH(d.bits)(numPregs-1,0) & Fill(n,d.valid)).reduce(_|_) | br_deallocs
	// 释放物理寄存器的掩码。
	// 取输入的plWidth个要释放的物理寄存器的二进制地址，全部转换为独热码后舍去最后一位（0号？），并根据
	//  valid是否有效选择是否将其置为0，然后把所有独热码地址合起来，最后和分支中要释放的物理寄存器独热码
	//  地址取并集，得到*最终*所有要释放（dealloc）的物理寄存器的独热码地址集合。


  val br_slots = VecInit(io.ren_br_tags.map(tag => tag.valid)).asUInt
	// 分支槽指示器。储存重命名分支标签组中的有效位信息。（asUInt？）
```
alloc_masks原理图：

![freelist2](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/freelist2.png)

sel_mask原理图：

![freelist1](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/freelist1.jpg)

#### 4.逻辑功能
##### ①填写分支分配列表

根据输入的分支标签（及有效信息），结合已分配的和要释放的寄存器，填写分支分配列表用于恢复。

```scala
   // Create branch allocation lists.（创建分支分配列表）
  for (i <- 0 until maxBrCount) {	// 遍历每个分支
    val list_req = VecInit(io.ren_br_tags.map(tag => UIntToOH(tag.bits)(i))).asUInt & br_slots	// i=1~4列
    val new_list = list_req.orR	// 最后一行
    br_alloc_lists(i) := Mux(new_list, Mux1H(list_req, alloc_masks.slice(1, plWidth+1)),
                                       br_alloc_lists(i) & ~br_deallocs | alloc_masks(0))
      // 见下文
  }  
```
list_req和new_list原理图：（`plWidth=5, maxBrCoun=4`，且图中 i 应为 0->3 ）

![freelist3](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/freelist3.png)

br_alloc_lists原理图：（再贴一遍代码）

```scala
	val br_alloc_lists = Reg(Vec(maxBrCount, UInt(numPregs.W)))
	// 分支分配列表组。为每一条分支储存对应的空闲列表（快照）。
	......
	br_alloc_lists(i) := Mux(new_list, Mux1H(list_req, alloc_masks.slice(1, plWidth+1)),
                                       br_alloc_lists(i) & ~br_deallocs | alloc_masks(0))
	// 将其第i行：
	//     若new_list为1（因为是分支入口，有了新的分配列表），
	//         根据list_req（定位第n条指令），置为alloc_masks中的第n行（第n条和之后用到的寄存器）。
	//     若new_list为0（不是分支入口，无需保存分配列表），
	//         将原值去掉已释放的，再加上本批plWidth个指令新分配的寄存器（即作为普通指令看待）。
```

![freelist4](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/freelist4.png)

##### ②更新空闲列表

根据已选择的和已释放的物理寄存器，更新空闲列表（仍然考虑0号寄存器的特殊情况）。

```scala
  // Update the free list.（更新空闲列表）
  free_list := (free_list & ~sel_mask | dealloc_mask) & ~(1.U(numPregs.W))
```
##### ③分配物理寄存器作为输出

根据分配物理寄存器的请求和空闲列表情况，做出对请求的回复，作为输出。

首先再贴一遍前面的代码：

```scala
  val reqs          = Input(Vec(plWidth, Bool()))
      // 【输入】请求组。plWidth个Bool值，分别代表每一条指令的请求是否有效。
  ......
  val alloc_pregs   = Output(Vec(plWidth, Valid(UInt(pregSz.W))))
      // 【输出】分配出去的plWidth个物理寄存器的二进制地址。
  ......
  val sels = SelectFirstN(free_list, plWidth)
	// 已选择的plWidth个物理寄存器的独热码地址组。
  val sel_fire  = Wire(Vec(plWidth, Bool()))
	// 指示对应物理寄存器是否能真正分配出去的信号。plWidth个Bool值。
```

```scala
  // Pipeline logic | hookup outputs.（流水线逻辑 | 连接输出）
  for (w <- 0 until plWidth) {	// 遍历每条指令
    val can_sel = sels(w).orR	// 指示是否为这条指令选择到了（一个）物理寄存器
    val r_valid = RegInit(false.B)	// 见下文
    val r_sel   = RegEnable(OHToUInt(sels(w)), sel_fire(w))	
      // 为第w条指令分配的物理寄存器的二进制地址，使能信号为sel_fire的第w位。

    r_valid := r_valid && !io.reqs(w) || can_sel
      // 指示到当前序号为止的指令选中的所有寄存器是否全部有效。
      //  为1的条件为：这条指令确实选到了寄存器（有足够多的空闲寄存器） 
      //            或者 上一条指令选到了寄存器且这条指令并不请求寄存器
      //            （因为selectFirstN的性质，不存在前面没选到而这条选到的情况）
    sel_fire(w) := (!r_valid || io.reqs(w)) && can_sel
      // 将sel_fire的第w位：
      //     若can_sel为0（没选到，空闲寄存器不够了），置为0。
      //     若can_sel为1（选到了，空闲寄存器够），
      //         当r_valid为0 或者 reqs第w位是1（第w条指令确实请求寄存器），置为1，否则置为0。
      //         (第一个条件不关心，因为此情况下必为1，但写上去的意义？)
    
    io.alloc_pregs(w).bits  := r_sel	// 二进制地址
    io.alloc_pregs(w).valid := r_valid	// 有效位（只能是1后面接若干个0，可以全是1）
  }
```
##### ④调试与警告

```scala
  io.debug.freelist := free_list | io.alloc_pregs.map(p => UIntToOH(p.bits) & Fill(n,p.valid)).reduce(_|_)
	// 输出，从整个rename模块接出去。
	// 当前空闲列表中物理寄存器 和 分配出去的物理寄存器 的 并集，即上一个周期空闲列表的内容。
  io.debug.isprlist := 0.U  // TODO track commit free list. （）
	// 输出，从整个rename模块接出去。
	// 未研究。注释说置为0可以跟踪空闲列表的提交。

  assert (!(io.debug.freelist & dealloc_mask).orR, "[freelist] Returning a free physical register.")
	// 上一周期的所有空闲寄存器 和 （上一周期）要求释放的物理寄存器有重叠。
	// 警告正在返回（释放）一个空闲的物理寄存器。
  assert (!io.debug.pipeline_empty || PopCount(io.debug.freelist) >= (numPregs - numLregs - 1).U,
    "[freelist] Leaking physical registers.")
	// 流水线为空 而且 上一周期空闲物理寄存器数量 < 物理寄存器数量 - 逻辑寄存器数量 (正常情况下应相等)
	// 警告物理寄存器泄露（有物理寄存器应当被释放而实际上没有）

} // 整个RenameFreeList类的结束
```





### 相关语法和模块

#### 1.SelectFirstN 方法

在`util.scala`中定义如下：

```scala
object SelectFirstN
{
	def apply(in: UInt, n: Int) = {
 		val sels = Wire(Vec(n, UInt(in.getWidth.W)))
 		var mask = in

 		for (i <- 0 until n) {
   			sels(i) := PriorityEncoderOH(mask)
   			mask = mask & ~sels(i)
 		}
 	sels
	}
}
```

其中```PriorityEncoderOH```方法在```OneHot.scala```中有定义，输入一个向量，将其保留最低一位（最靠右）的1后返回，如输入false.B, true.B, true.B, false.B，则返回false.B, false.B, true.B, false.B

因此这里```SelectFirstN```的原理是，将输入的```free_list```保存在```mask```中，每次提取一个独热码，作为返回值的一行，并在```mask```中抹去这一位，循环n次。



#### 2.scanRight 方法（例子）

```scala
List(1,2,3,4).scanRight(0)(_+_) == List(10,9,7,4,0) 
```



#### 3.BrUpdataInfo 模块

见functional-unit.scala。

```scala
class BrUpdateInfo(implicit p: Parameters) extends BoomBundle
{
// On the first cycle we get masks to kill registers（第一个周期得到释放寄存器的掩码）
val b1 = new BrUpdateMasks
// On the second cycle we get indices to reset pointers（第二个周期得到索引以重置指针）
val b2 = new BrResolutionInfo
}
```

```scala
class BrUpdateMasks(implicit p: Parameters) extends BoomBundle
{
val resolve_mask = UInt(maxBrCount.W)
val mispredict_mask = UInt(maxBrCount.W)
}
```

```scala
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



