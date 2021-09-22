# dispatch
指令的派遣。

定义派遣器xxxDispatcher，xxx根据其参数/派遣策略可以是Basic或者Compacting。从重命名级输入coreWidth条微指令，输出派遣的微指令组到各发射队列（到不同发射队列的指令组长度可能不同）。注意这里的输入输出都用的是DecoupledIO，因此在处理得到派遣的指令时，“反向传播”ready信号至重命名级，以获取下一批指令。

```scala
package boom.exu
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import boom.common._
import boom.util._
```



### DispatchIO 类

派遣的IO端口类。包含coreWidth条从重命名2阶段传来的微指令，以及发送到N个发射队列的微指令组。

到每个发射队列的微指令组长度dispatchWidth可能是不同的，因此使用MixedVec，见文件最后。尽管在这里长度都是1。

```scala
class DispatchIO(implicit p: Parameters) extends BoomBundle
{
  // incoming microops from rename2
  val ren_uops = Vec(coreWidth, Flipped(DecoupledIO(new MicroOp)))

  // outgoing microops to issue queues
  // N issues each accept up to dispatchWidth uops
  // dispatchWidth may vary between issue queues
  val dis_uops = MixedVec(issueParams.map(ip=>Vec(ip.dispatchWidth, DecoupledIO(new MicroOp))))
}
```
其中issueParams在parameters.scala的BoomCoreParams case类中有如下定义：

```scala
...
issueParams: Seq[IssueParams] = Seq(
    IssueParams(issueWidth=1, numEntries=16, iqType=IQT_MEM.litValue, dispatchWidth=1),
    IssueParams(issueWidth=2, numEntries=16, iqType=IQT_INT.litValue, dispatchWidth=1),
    IssueParams(issueWidth=1, numEntries=16, iqType=IQT_FP.litValue , dispatchWidth=1)),
...
```

序列中的每一项都是issue-unit.scala中定义的IssueParams类。



### Dispatcher 抽象类

派遣器抽象类，只定义了IO端口。有下面两种具体实现。

```scala
abstract class Dispatcher(implicit p: Parameters) extends BoomModule
{
  val io = IO(new DispatchIO)
}
```



#### BasicDispatcher 类

基础派遣器类，考虑了最坏的情况，即所有被派遣的uop都到同一个发射队列中去了。这和BOOMv2的行为是等价的。

特点：每个发射队列的dispatchWidth都与coreWidth相等，所以一个发射队列才能够刚好容纳所有被派遣的uop。

```scala
/**
 * This Dispatcher assumes worst case, all dispatched uops go to 1 issue queue
 * This is equivalent to BOOMv2 behavior
 */
  class BasicDispatcher(implicit p: Parameters) extends Dispatcher
  {
    issueParams.map(ip=>require(ip.dispatchWidth == coreWidth))

  val ren_readys = io.dis_uops.map(d=>VecInit(d.map(_.ready)).asUInt).reduce(_&_)
      // 输出给派遣级的微指令中的总ready信号作为传给重命名级的readys信号。
      // 只有所有队列中相同位置的微指令都ready时，对应的readys才为1。

  for (w <- 0 until coreWidth) {
    io.ren_uops(w).ready := ren_readys(w) // 将上面的值连接到输出
  }	// 不同发射队列的同一个位置的微指令都ready时，这个位置才能从重命名级接受新的微指令

  for {i <- 0 until issueParams.size	// 对第i条发射队列
       w <- 0 until coreWidth} {	// 遍历所有重命名级传来的微指令    （scala中for这么用相当于简单的嵌套）
    val issueParam = issueParams(i)	// 获取第i条发射队列的参数
    val dis        = io.dis_uops(i)	// 选择要派遣到第i条发射队列的微指令Vec
		// 选择第w条微指令，用重命名级传来的微指令修改其valid和bits
      	// 体现了dispatchWidth==coreWidth
    dis(w).valid := io.ren_uops(w).valid && ((io.ren_uops(w).bits.iq_type & issueParam.iqType.U) =/= 0.U)
    dis(w).bits  := io.ren_uops(w).bits
  }
}
```



#### CompactingDispatcher 类

紧密(compacting)派遣器类，尝试尽可能多地向发射队列派遣指令，因而可以忍受每个周期较低的coreWidth。

特点：每个发射队列的dispatchWidth都大于等于coreWidth。

当dispatchWidth > coreWidth时，一个派遣队列不可能只被一批要派遣的微指令占满。

当dispatchWidth = coreWidth时，它的行为也与基础派遣器不同：只会在微指令请求的一个发射队列满了的时候才停顿发射。

```scala
/**
 *  Tries to dispatch as many uops as it can to issue queues,
 *  which may accept fewer than coreWidth per cycle.
 *  When dispatchWidth == coreWidth, its behavior differs
 *  from the BasicDispatcher in that it will only stall dispatch when
 *  an issue queue required by a uop is full.
 */
  class CompactingDispatcher(implicit p: Parameters) extends Dispatcher
  {
    issueParams.map(ip => require(ip.dispatchWidth >= ip.issueWidth))

      
  val ren_readys = Wire(Vec(issueParams.size, Vec(coreWidth, Bool())))
      // 记录 发射队列数*重命名级传入的微指令数 条微指令的ready信号。

      
  for (((ip, dis), rdy) <- issueParams zip io.dis_uops zip ren_readys) {
      // 遍历每条派遣队列，ip是其参数，dis是队列中已有的微指令，rdy是重命名级传来的coreWidth条微指令的ready信号。
    
    val ren = Wire(Vec(coreWidth, Decoupled(new MicroOp)))
    ren <> io.ren_uops	// 获取重命名级传来的coreWidth条微指令
    val uses_iq = ren map (u => (u.bits.iq_type & ip.iqType.U).orR) 
      // 提取其中的发射队列类型信息，使用orR即只要有指令的发射队列类型正确，就置1
    
    // Only request an issue slot if the uop needs to enter that queue.
    // 如果这coreWidth条微指令都不能发送到当前遍历的派遣队列，则valid全部置0，否则保持。
    (ren zip io.ren_uops zip uses_iq) foreach {case ((u,v),q) =>
      u.valid := v.valid && q}
    
    // 调用Compactor类（见文件最后），输入ren（修改了valid后的原始输入），输出dis
    val compactor = Module(new Compactor(coreWidth, ip.dispatchWidth, new MicroOp))
    compactor.io.in  <> ren
    dis <> compactor.io.out
    
    // The queue is considered ready if the uop doesn't use it.
    // 如果这coreWidth条微指令都不能发送到当前遍历的派遣队列，这条队列置为ready，否则保持。
    rdy := ren zip uses_iq map {case (u,q) => u.ready || !q}
  }

      
  (ren_readys.reduce((r,i) =>
      VecInit(r zip i map {case (r,i) =>
        r && i})) zip io.ren_uops) foreach {case (r,u) =>
          u.ready := r}
      // 对于同一个位置，如果所有派遣队列中都已经ready，则传给重命名级的ready信号才为1，否则为0
      // 和BasicDispatcher中的逻辑类似，但实现方法不同
}
```



## 补充

### MixedVec

Vec类的所有元素必须是同一个类型。如果我们想要创建一个有不同类型元素的Vec，就可以用MixedVec：

```scala
import chisel3.util.MixedVec
class ModuleMixedVec extends Module {
  val io = IO(new Bundle {
    val x = Input(UInt(3.W))
    val y = Input(UInt(10.W))
    val vec = Output(MixedVec(UInt(3.W), UInt(10.W)))	// 拼接向量x和y
  })
  io.vec(0) := io.x
  io.vec(1) := io.y
}
```

也可以用这样的炫技用法:

```scala
class ModuleProgrammaticMixedVec(x: Int, y: Int) extends Module {
  val io = IO(new Bundle {
    val vec = Input(MixedVec((x to y) map { i => UInt(i.W) }))
    // ...
  })
  // ...rest of the module goes here...
}
```



### Compactor 类

在util.scala中定义。

在n（coreWidth）个valid输入中，选择k（io.dispatch）个作为输出，类型为T（MicroOp）。

```scala
val compactor = Module(new Compactor(coreWidth, ip.dispatchWidth, new MicroOp))
```
```scala
class Compactor[T <: chisel3.core.Data](n: Int, k: Int, gen: T) extends Module
{
  require(n >= k)

  val io = IO(new Bundle {	// 输入输出都是DecoupledIO
    val in  = Vec(n, Flipped(DecoupledIO(gen)))
    val out = Vec(k,         DecoupledIO(gen))
  })

  if (n == k) {	// 选了个寂寞
    io.out <> io.in
  } 
  else {
 
      // 真正的选择逻辑
    val counts = io.in.map(_.valid).scanLeft(1.U(k.W)) ((c,e) => Mux(e, (c<<1)(k-1,0), c))
    val sels = Transpose(VecInit(counts map (c => VecInit(c.asBools)))) map (col =>
                 (col zip io.in.map(_.valid)) map {case (c,v) => c && v})
    val in_readys = counts map (row => (row.asBools zip io.out.map(_.ready)) map {case (c,r) => c && r} reduce (_||_))
    val out_valids = sels map (col => col.reduce(_||_))
    val out_data = sels map (s => Mux1H(s, io.in.map(_.bits)))
	  
      // 连接输出
    in_readys zip io.in foreach {case (r,i) => i.ready := r}
    out_valids zip out_data zip io.out foreach {case ((v,d),o) => o.valid := v; o.bits := d}
  }
}
```

