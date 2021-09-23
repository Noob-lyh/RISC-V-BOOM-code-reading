# execution-units
构建执行单元组。

将同类（都是非FPU或FPU）的执行单元放在一个“collection”类ExecutionUnits中。根据参数配置执行单元，检查执行单元中功能单元的完备性，完善调试输出信息以及定义执行单元组总的一些参数。

```scala
package boom.exu
import scala.collection.mutable.{ArrayBuffer}
import chisel3._
import freechips.rocketchip.config.{Parameters}
import boom.common._
import boom.util.{BoomCoreStringPrefix}
```



### BOOM的执行单元层次

从官方文档上面找到的图，只看执行单元组这一部分：                       →                      →                         ↓

![BOOMppl](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/BOOMppl.jpg)

一个方框就代表一个执行单元，而方框中难以描述的形状就代表一个功能单元；不同的功能单元组合，可以实现不同的功能。

根据各个类的包含与继承关系，可以大概画出如下的层次图：

![exeunits](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/exeunits.jpg)

其中，灰色方框就是执行单元，而灰色方框连接的线上的就是功能单元。虚线表示继承关系（如FunctionalUnit抽象类由BoomModule类继承），而导图主体表示可选的包含关系（比如ALUExeUnit类中可以包含ALUUnit类、RoCCShim类等中的若干类，注意有些功能单元组合是不被允许的）。

可以看出，不少的功能单元最后都包含有Rocket的功能单元（加粗的类），但需要注意的是，这功能单元是适用在Rocket这种顺序流水线上的，而在BOOM中，由于使用乱序流水线，被发射的指令有可能在中途被kill掉，那些功能单元不能兼容这种场景。 因此为了复用Rocket的功能单元，BOOM在这些FunctionalUnit抽象类中将Rocket上的功能单元包装起来（增加相关逻辑和端口等），以适应需要。



### ExecutionUnits 类

```scala
/**
 * Top level class to wrap all execution units together into a "collection"
 * 顶层类，将所有执行单元打包到一个“collection”中
 *
 * @param fpu using a FPU?
 */
    class ExecutionUnits(val fpu: Boolean)(implicit val p: Parameters) extends HasBoomCoreParameters
{
    // 总发射宽度
    val totalIssueWidth = issueParams.map(_.issueWidth).sum

    
  //*******************************
  // Instantiate the ExecutionUnits 例化执行单元组，私有成员

  private val exe_units = ArrayBuffer[ExecutionUnit]()

    
  //*******************************
  // Act like a collection 像一个collection一样运作，即定义ExecutionUnits类的方法

  def length = exe_units.length

  def apply(n: Int) = exe_units(n)

  def map[T](f: ExecutionUnit => T) = {
    exe_units.map(f)
  }

  def withFilter(f: ExecutionUnit => Boolean) = {
    exe_units.withFilter(f)
  }

  def foreach[U](f: ExecutionUnit => U) = {
    exe_units.foreach(f)
  }

  def zipWithIndex = {
    exe_units.zipWithIndex
  }

  def indexWhere(f: ExecutionUnit => Boolean) = {
    exe_units.indexWhere(f)
  }

  def count(f: ExecutionUnit => Boolean) = {
    exe_units.count(f)
  }

    // lazy变量，调用时才会实例化
  lazy val memory_units = {
    exe_units.filter(_.hasMem)
  }

  lazy val alu_units = {
    exe_units.filter(_.hasAlu)
  }

  lazy val csr_unit = {
    require (exe_units.count(_.hasCSR) == 1)
    exe_units.find(_.hasCSR).get
  }

  lazy val ifpu_unit = {
    require (usingFPU)
    require (exe_units.count(_.hasIfpu) == 1)
    exe_units.find(_.hasIfpu).get
  }

  lazy val fpiu_unit = {
    require (usingFPU)
    require (exe_units.count(_.hasFpiu) == 1)
    exe_units.find(_.hasFpiu).get
  }


  lazy val jmp_unit_idx = {
    exe_units.indexWhere(_.hasJmpUnit)
  }

  lazy val rocc_unit = {
    require (usingRoCC)
    require (exe_units.count(_.hasRocc) == 1)
    exe_units.find(_.hasRocc).get
  }

    
    
  if (!fpu) { // 非浮点单元
    val int_width = issueParams.find(_.iqType == IQT_INT.litValue).get.issueWidth

    for (w <- 0 until memWidth) { // memWidth个带访存功能的ALU执行单元
      val memExeUnit = Module(new ALUExeUnit(
        hasAlu = false,
        hasMem = true))
    
      memExeUnit.io.ll_iresp.ready := DontCare
    
      exe_units += memExeUnit
    }
    
    for (w <- 0 until int_width) { // int_width个整数ALU执行单元？
      def is_nth(n: Int): Boolean = w == ((n) % int_width)
      val alu_exe_unit = Module(new ALUExeUnit(
        hasJmpUnit     = is_nth(0),
        hasCSR         = is_nth(1),
        hasRocc        = is_nth(1) && usingRoCC,
        hasMul         = is_nth(2),
        hasDiv         = is_nth(3),
        hasIfpu        = is_nth(4) && usingFPU))
      exe_units += alu_exe_unit
    }
  } 
    
    
  else { // 浮点单元
    val fp_width = issueParams.find(_.iqType == IQT_FP.litValue).get.issueWidth
    for (w <- 0 until fp_width) { // fp_width个FPU执行单元，第一个有fp转int，可能有浮点除法
      val fpu_exe_unit = Module(new FPUExeUnit(hasFpu = true,
                                             hasFdiv = usingFDivSqrt && (w==0),
                                             hasFpiu = (w==0)))
      exe_units += fpu_exe_unit
    }
  }

    
    
    // 输出拥有的功能
  val exeUnitsStr = new StringBuilder
  for (exe_unit <- exe_units) {
    exeUnitsStr.append(exe_unit.toString)
  }

  override def toString: String =
    (BoomCoreStringPrefix("===ExecutionUnits===") + "\n"
    + (if (!fpu) {
      BoomCoreStringPrefix(
        "==" + coreWidth + "-wide Machine==",
        "==" + totalIssueWidth + " Issue==")
    } else {
      ""
    }) + "\n"
    + exeUnitsStr.toString)

    
    
    // 检查功能单元是否完备。
    // 非浮点需要至少一个men、mul、div，浮点需要至少一个fpu
  require (exe_units.length != 0)
  if (!fpu) {
    // if this is for FPU units, we don't need a memory unit (or other integer units).
    require (exe_units.map(_.hasMem).reduce(_|_), "Datapath is missing a memory unit.")
    require (exe_units.map(_.hasMul).reduce(_|_), "Datapath is missing a multiplier.")
    require (exe_units.map(_.hasDiv).reduce(_|_), "Datapath is missing a divider.")
  } else {
    require (exe_units.map(_.hasFpu).reduce(_|_),
      "Datapath is missing a fpu (or has an fpu and shouldnt).")
  }

    
    // 计算执行单元组一共需要各种端口的数量
  val numIrfReaders       = exe_units.count(_.readsIrf)
  val numIrfReadPorts     = exe_units.count(_.readsIrf) * 2 // 整数指令2个操作数
  val numIrfWritePorts    = exe_units.count(_.writesIrf)
  val numLlIrfWritePorts  = exe_units.count(_.writesLlIrf)
  val numTotalBypassPorts = exe_units.withFilter(_.bypassable).map(_.numBypassStages).foldLeft(0)(_+_)

  val numFrfReaders       = exe_units.count(_.readsFrf)
  val numFrfReadPorts     = exe_units.count(_.readsFrf) * 3 // 浮点指令可以有3个操作数
  val numFrfWritePorts    = exe_units.count(_.writesFrf)
  val numLlFrfWritePorts  = exe_units.count(_.writesLlFrf)

    
    
  // The mem-unit will also bypass writes to readers in the RRD stage.
  // NOTE: This does NOT include the ll_wport
  // 内存单元同样会由旁路写到读寄存器级。这不包括ll_wport(?).
  val bypassable_write_port_mask = exe_units.withFilter(x => x.writesIrf).map(u => u.bypassable)
}
```
