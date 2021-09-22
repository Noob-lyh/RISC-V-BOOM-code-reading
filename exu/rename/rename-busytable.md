# rename-busytable
—— 根据新的指令组和已写回的物理寄存器信息，维护物理寄存器繁忙列表，并输出新指令组中各指令源操作数物理寄存器的繁忙信息。



```scala
package boom.exu
import chisel3._
import chisel3.util._
import boom.common._
import boom.util._
import freechips.rocketchip.config.Parameters
```



### BusyResp 类

繁忙列表回复。

包括三个Bool值，分别指示一条指令的三个（非浮点指令为两个）源操作数物理寄存器是否繁忙。

```scala
class BusyResp extends Bundle
{
  val prs1_busy = Bool()
  val prs2_busy = Bool()
  val prs3_busy = Bool()
}
```



### RenameBusyTable 类（busytable主体）

#### 1.参数

```scala
class RenameBusyTable(	// 1.参数
  val plWidth: Int,
  val numPregs: Int,
  val numWbPorts: Int,	// 写回端口数量
  val bypass: Boolean,
  val float: Boolean)
  (implicit p: Parameters) extends BoomModule
{
  val pregSz = log2Ceil(numPregs)
    
    // 2.IO端口
    
    // 3.成员
    
    // 4.逻辑功能
}
```
#### 2. IO端口
```scala
 val io = IO(new BoomBundle()(p) {
     
    val ren_uops = Input(Vec(plWidth, new MicroOp))
     // 【输入】 plWidth个指令。
    val busy_resps = Output(Vec(plWidth, new BusyResp))
     // 【输出】 繁忙列表回复组。plWidth个繁忙列表回复。
    val rebusy_reqs = Input(Vec(plWidth, Bool()))
     // 【输入】 重新变为繁忙的请求。plWidth个Bool值。

    val wb_pdsts = Input(Vec(numWbPorts, UInt(pregSz.W)))
     // 【输入】 已写回目的物理寄存器组。numWbPort个物理寄存器的二进制地址。
    val wb_valids = Input(Vec(numWbPorts, Bool()))
     // 【输入】 已写回的有效信号组。numWbPort个Bool值。
    
    val debug = new Bundle { val busytable = Output(Bits(numPregs.W)) }
     // 【Debug】 Bundle类，内含一个输出busytable，长度为物理寄存器个数。
  })
```
#### 3.成员

```scala
  val busy_table = RegInit(0.U(numPregs.W))
	// 繁忙列表。长度为物理寄存器个数，每位代表一个物理寄存器。初始化为0.

  // Unbusy written back registers.（将已写回的物理寄存器组修改为非繁忙）
  val busy_table_wb = busy_table & ~(io.wb_pdsts zip io.wb_valids)
    .map {case (pdst, valid) => UIntToOH(pdst) & Fill(numPregs,valid.asUInt)}.reduce(_|_)
	// 去掉已写回的物理寄存器后的繁忙列表。 
	// 与运算的第二个参数是已写回的物理寄存器地址组（独热码）的取反，与运算后就将busy_table中对应的位
	//  置0，实现去掉已写回的物理寄存器。

  // Rebusy newly allocated registers.（将新分配的物理寄存器重新置为繁忙）
  val busy_table_next = busy_table_wb | (io.ren_uops zip io.rebusy_reqs)
    .map {case (uop, req) => UIntToOH(uop.pdst) & Fill(numPregs, req.asUInt)}.reduce(_|_)
	// 下一步的繁忙列表。
	// 去掉已写回的物理寄存器后的繁忙列表 加上 新指令组要分配的物理寄存器。
```
#### 4.逻辑功能

##### ①更新繁忙列表

```scala
  busy_table := busy_table_next
```
##### ②读繁忙列表，得到输出
```scala
  // Read the busy table. （读繁忙列表）
  for (i <- 0 until plWidth) {	// 遍历每条指令
    val prs1_was_bypassed = (0 until i).map(j =>
      io.ren_uops(i).lrs1 === io.ren_uops(j).ldst && io.rebusy_reqs(j)).foldLeft(false.B)(_||_)
    val prs2_was_bypassed = (0 until i).map(j =>
      io.ren_uops(i).lrs2 === io.ren_uops(j).ldst && io.rebusy_reqs(j)).foldLeft(false.B)(_||_)
    val prs3_was_bypassed = (0 until i).map(j =>
      io.ren_uops(i).lrs3 === io.ren_uops(j).ldst && io.rebusy_reqs(j)).foldLeft(false.B)(_||_)
      // 指示prs1~3被旁路传播的信号。
      // 示例：对于prs1一行，i=3（第四条指令）时，(0 until i)即(0,1,2)经过map后得到同样三个元素的
      //  List，值为0或1，为1的条件为：这条指令的ldst与第四条指令的lrs1相等 且 这条指令会导致繁忙，即
      //  RAW冲突，因此foldLeft后，只要有一个元素为1，就会使得这个指示prs1被旁路传播的信号置为1。

    io.busy_resps(i).prs1_busy := busy_table(io.ren_uops(i).prs1) || prs1_was_bypassed && bypass.B
    io.busy_resps(i).prs2_busy := busy_table(io.ren_uops(i).prs2) || prs2_was_bypassed && bypass.B
    io.busy_resps(i).prs3_busy := busy_table(io.ren_uops(i).prs3) || prs3_was_bypassed && bypass.B
      // 得到输出的 繁忙列表回复组 中各个prs是否繁忙的指示位
      // 注：(output)busy_resps与(input)ren_uops一一对应
      // 指示为1的条件为：busy_table中对应的物理寄存器确实繁忙（A要写但A还没写好） 或 被旁路传播
      //  且确实有旁路逻辑 （B要读但A还没写好）
      
    if (!float) io.busy_resps(i).prs3_busy := false.B
      // 非浮点指令无第三个源操作数，因此对应的繁忙信息不关心。
  }
```
##### ③调试信息
```scala
  io.debug.busytable := busy_table
	// busy_table只是“中间变量”，这里作为debug输出便于调试。

}
```