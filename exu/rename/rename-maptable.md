# rename-maptable
—— 实现指令中的逻辑寄存器到物理寄存器的映射。输入映射请求组和重映射请求组，以及分支和回滚的相关信息，实现映射表的维护（包括保存和恢复快照），并输出对应指令的寄存器的最新映射信息。



```scala
package boom.exu
import chisel3._
import chisel3.util._
import boom.common._
import boom.util._
import freechips.rocketchip.config.Parameters
```



### MapReq 类
映射请求。

包括三个源操作数逻辑寄存器和一个目的逻辑寄存器的二进制地址。

整数指令只有两个源操作数，运行浮点指令时才会使用第三个源操作数寄存器。

每条指令对应一个`MapReq`。

```scala
class MapReq(val lregSz: Int) extends Bundle
{
  val lrs1 = UInt(lregSz.W) // lregSz是为了定位逻辑寄存器所需的二进制地址的最小位数
  val lrs2 = UInt(lregSz.W)
  val lrs3 = UInt(lregSz.W)
  val ldst = UInt(lregSz.W)
}
```



### MapResp 类

映射回复。

包括三个源操作数物理寄存器（对应MapReq中的三个源操作数逻辑寄存器）和一个僵死目的物理寄存器（指令提交之后不再被读取，寄存器将被返回到`Freelist`，对应MapReq中的目的逻辑寄存器）的二进制地址。

每条指令对应一个`MapResp`。

```scala
class MapResp(val pregSz: Int) extends Bundle
{
  val prs1 = UInt(pregSz.W)
  val prs2 = UInt(pregSz.W)
  val prs3 = UInt(pregSz.W)
  val stale_pdst = UInt(pregSz.W)
}
```



### RemapReq 类

重映射请求。

保存一对物理目的寄存器和逻辑目的寄存器，以及用于指示是否需要进行重映射的`valid`信号。

```scala
class RemapReq(val lregSz: Int, val pregSz: Int) extends Bundle
{
  val ldst = UInt(lregSz.W)
  val pdst = UInt(pregSz.W)
  val valid = Bool()
}
```



### RenameMapTable类（maptable主体）
#### 1.参数

```scala
class RenameMapTable(	// 1.参数	
  val plWidth: Int,		// 同时处理的指令数
  val numLregs: Int,	// 整个表中逻辑（指令）寄存器个数
  val numPregs: Int,	// 整个表中物理寄存器个数
  val bypass: Boolean,	// 指示是否有旁路逻辑
  val float: Boolean)	// 指示是否有浮点操作
  (implicit p: Parameters) extends BoomModule
{
  val pregSz = log2Ceil(numPregs)	//为了定位物理寄存器需要的二进制地址的最小位数
	
  // 2.IO端口
    
  // 3.成员
    
  // 4.逻辑功能
}
```
#### 2.IO端口

```scala
val io = IO(new BoomBundle()(p) {		//输入输出端口
         
    
    // Logical sources -> physical sources. （逻辑源映射到物理源）
    val map_reqs    = Input(Vec(plWidth, new MapReq(lregSz)))
    	// 【输入】 plWidth个映射请求（映射请求组）
    val map_resps   = Output(Vec(plWidth, new MapResp(pregSz)))
    	// 【输出】 plWidth个映射回复（映射回复组）

    
    // Remapping an ldst to a newly allocated pdst? （需要将一个逻辑目的寄存器重新映射吗？）
    val remap_reqs  = Input(Vec(plWidth, new RemapReq(lregSz, pregSz)))
    	// 【输入】 plWidth个重映射请求（重映射请求组）
    
    
    // Dispatching branches: need to take snapshots of table state. （保存映射表快照）
    val ren_br_tags = Input(Vec(plWidth, Valid(UInt(brTagSz.W))))
    	// 【输入】 plWidth个分支tag
    	// 分支tag用于标记指令与分支的依赖关系。当一个分支指令通过Rename时，寄存器Rename表和空闲列表 
    	//   的副本被复制。在预测失误时，将终止流水线中依赖于该分支的全部指令，并还原保存的处理器状态。
   
    
    // Signals for restoring state following misspeculation.（误判后恢复状态的信号）
    val brupdate      = Input(new BrUpdateInfo)
    	// 【输入】分支tag更新信息
    val rollback    = Input(Bool())
    	// 【输入】指示是否回滚的信号。回滚状态下不能提交（commit）。
    
  })
```
#### 3.成员

```scala
  // The map table register array and its branch snapshots.（映射表和他的分支快照）
  val map_table = RegInit(VecInit(Seq.fill(numLregs){0.U(pregSz.W)}))
	// 映射表。含numLregs个宽度为pregSz的物理寄存器地址。初始化为0.
  val br_snapshots = Reg(Vec(maxBrCount, Vec(numLregs, UInt(pregSz.W))))
	// 分支快照组。包含maxBrCount个快照，每个快照包含numLregs个宽度为pregSz的物理寄存器地址，即一个映射表。


  // The intermediate states of the map table following modification by each pipeline slot. （每个流水线槽修改后映射表的中间状态？）
  val remap_table = Wire(Vec(plWidth+1, Vec(numLregs, UInt(pregSz.W))))
	// 重映射表。保存(plWidth+1)条指令（为了配合scanLeft函数）中逻辑与物理寄存器的映射关系。
	// remap_table的主要作用是记录被修改过的映射关系，最终用于更新map_table,因此不需要使用寄存器储存     
    //   它的值，而是将其定义为Wire数据类型，经逻辑电路后连接到map_table对应的寄存器组中即可。
	// 共(plWidth+1)列，每列对应一条指令。每一列的行数与逻辑寄存器个数相同。内容是物理寄存器二进制地址。
	// 索引时，remap_table(j)(i)表示其第i行第j列。


  // Uops requesting changes to the map table.（）
  val remap_pdsts = io.remap_reqs map (_.pdst)
	// 重映射物理目的寄存器组。保存了重映射请求组对应的物理目的寄存器二进制地址（组）。
  val remap_ldsts_oh = io.remap_reqs map (req => UIntToOH(req.ldst) & Fill(numLregs, req.valid.asUInt))
	// 重映射逻辑目的寄存器组（独热码形式）。如果对应请求有效，则保存该请求的逻辑寄存器独热码地址。
```
#### 4.逻辑功能

##### ①更新重映射表

```scala
  // Figure out the new mappings seen by each pipeline slot.（找出每个流水线槽看到的新映射）
  for (i <- 0 until numLregs) {	// 对每个逻辑寄存器进行遍历
    if (i == 0 && !float) {
      for (j <- 0 until plWidth+1) {
        remap_table(j)(i) := 0.U  // 对第0个逻辑寄存器，对每一列（每个j）置零
      }
    } else { // 对第1到第numLregs个逻辑寄存器
      val remapped_row = (remap_ldsts_oh.map(ldst => ldst(i)) zip remap_pdsts)
        .scanLeft(map_table(i)) {case (pdst, (ldst, new_pdst)) => Mux(ldst, new_pdst, pdst)}
        // 更新重映射后的一行，在下一个for循环中用来更新重映射表的一行。
        // 原理见下图
      for (j <- 0 until plWidth+1) {
        remap_table(j)(i) := remapped_row(j) // 更新重映射表的第i行
      }
    }
  }
```
更新重映射表原理示意图：（假设`plWidth = 6`，`numLregs = 4`，`numPregs = 8 = 2^3`，且此时i取1）

![maptable1](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/maptable1.jpg)

##### ②创建和恢复映射表的快照

```scala
  // Create snapshots of new mappings.（创建新映射的快照）
  for (i <- 0 until plWidth) { // 遍历每条指令
    when (io.ren_br_tags(i).valid) { // valid值为1表示当前指令在分支中被预测执行
      br_snapshots(io.ren_br_tags(i).bits) := remap_table(i+1) 
        // 索引+1，使得第0到第(plWidth-1)条指令可以保存到重映射表的第1到第plWidth列（见上图）
    }
  }

  when (io.brupdate.b2.mispredict) { // 此值为1表示分支预测发生失误
    // Restore the map table to a branch snapshot.（将映射表保存到分支快照）
    map_table := br_snapshots(io.brupdate.b2.uop.br_tag) // 将预测失误指令对应的快照恢复
  } .otherwise { // 预测成功
    // Update mappings.（更新映射表）
    map_table := remap_table(plWidth) // 将完成流水线所有指令后最终映射关系更新到映射表中
  }
```

##### ③得到输出的映射回复组

```scala  
// Read out mappings.（读出映射）
  for (i <- 0 until plWidth) { //遍历每条指令
    io.map_resps(i).prs1:= (0 until i).foldLeft(map_table(io.map_reqs(i).lrs1)) ((p,k) =>
      Mux(bypass.B && io.remap_reqs(k).valid && io.remap_reqs(k).ldst ===         
        io.map_reqs(i).lrs1, io.remap_reqs(k).pdst, p))
    io.map_resps(i).prs2:= (0 until i).foldLeft(map_table(io.map_reqs(i).lrs2)) ((p,k) =>
      Mux(bypass.B && io.remap_reqs(k).valid && io.remap_reqs(k).ldst === 
        io.map_reqs(i).lrs2, io.remap_reqs(k).pdst, p))
    io.map_resps(i).prs3:= (0 until i).foldLeft(map_table(io.map_reqs(i).lrs3)) ((p,k) =>
      Mux(bypass.B && io.remap_reqs(k).valid && io.remap_reqs(k).ldst === 
        io.map_reqs(i).lrs3, io.remap_reqs(k).pdst, p))
    io.map_resps(i).stale_pdst := (0 until i).foldLeft(map_table(io.map_reqs(i).ldst)) 
      ((p,k) => Mux(bypass.B && io.remap_reqs(k).valid && io.remap_reqs(k).ldst ===    
        io.map_reqs(i).ldst, io.remap_reqs(k).pdst, p))
      // 根据映射表、映射请求组、重映射请求组，以及是否有旁路的信息，得到映射回复组。
      // 原理见下方解释

    if (!float) io.map_resps(i).prs3 := DontCare
      // !float为1表示没有浮点数操作（只需要两个源操作数寄存器），因此让第三个源操作数寄存器与DontCare连接。DontCare的作用是告诉chisel的未连接线网检测机制，该端口的读行为无需关心。
  }
```
· 得到映射回复组的实现方式：

​		遍历所有指令，对第i条指令（第i个映射回复），对`(0 until i)`调用`foldLeft`方法，初始值为第i个映射请求的对应源操作数逻辑寄存器（如`lrs1`）在`map_table`中对应的物理寄存器地址。

​		用p表示物理寄存器二进制地址（如，初始值为`map_table`中`lrs1`对应的物理寄存器），用k表示对历史指令的索引，索引范围是0~(i-1).

​		在归约时使用一个Mux的方法，判断条件为：（共三项）
​		`	bypass.B && io.remap_reqs(k).valid && io.remap_reqs(k).ldst === io.map_reqs(i).lrs1`，若为1，置为`io.remap_reqs(k).pdst`；若为0，置为当前归约结果（一个物理寄存器二进制地址）。

​		因此，当有旁路逻辑（第一项为1）时，遍历当前指令前的所有指令（i固定，k在扫描过程中从0变化至i-1），若存在一条历史指令发生了重映射（第二项为1）且其`ldst`与当前指令的`lrs1`相同（第三项为1），**即出现RAW冲突**，则当前指令的`prs1`与该历史指令中`pdst`保持一致；否则将当前指令的`prs1`是`map_table`中`lrs1`对应的物理寄存器编码。

##### ④警告信息

```scala
  // Don't flag the creation of duplicate 'p0' mappings during rollback.（回滚期间不标记重复的p0（第0个物理寄存器）映射的创建）
  // These cases may occur soon after reset, as all maptable entries are initialized to 'p0'.（这种情况可能在重置后不久发生，因为所有映射表条目都初始化为p0）
  io.remap_reqs map (req => (req.pdst, req.valid)) foreach {case (p,r) =>
    assert (!r || !map_table.contains(p) || p === 0.U && io.rollback, "[maptable] Trying to write a duplicate mapping.")}
	// 对于每个重映射请求，提示“尝试写入重复的映射”需要满足以下条件：
	// 1. 重映射有效（valid为1）
	// 2. 映射表中已经存在请求包含的物理寄存器地址
	// 3. 请求映射到的物理寄存器不是第0个 或 不在回滚状态（？）

} //（整个class RenameMapTable的结束）
```

### 相关语法（简要信息与示例）

#### 1.foldLeft （从左向右）

```scala
val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10) 

val res = numbers.foldLeft(0)((m, n) => m + n) 

println(res) // 55
```

#### 2.scanLeft（从左向右）

```scala
val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10) 

val res = numbers.scanLeft(0)( _ + _ ) 

println(res) // List(0, 1, 3, 6, 10, 15, 21, 28, 36, 45, 55)
```

#### 3.assert

```
当判断为假时输出警告。
```





