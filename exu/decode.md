# decode

译码模块。



RISC-V指令类型：

<img src=".\InstructionType.jpg" alt="InstructionType" style="zoom:67%;" />




```scala
package boom.exu
import chisel3._
import chisel3.util._
import freechips.rocketchip.config.Parameters
import freechips.rocketchip.rocket.Instructions._
import freechips.rocketchip.rocket.RVCExpander
import freechips.rocketchip.rocket.{CSR,Causes}
import freechips.rocketchip.util.{uintToBitPat,UIntIsOneOf}
import FUConstants._
import boom.common._
import boom.util._
```



### DecodeConstants 抽象特质

```scala
// scalastyle:off 关闭scala格式
/**
 * Abstract trait giving defaults and other relevant values to different Decode constants
 * 为不同的译码常数提供默认值和其他相关值的抽象特质
 */
abstract trait DecodeConstants
  extends freechips.rocketchip.rocket.constants.ScalarOpConstants
  with freechips.rocketchip.rocket.constants.MemoryOpConstants
{
    val xpr64 = Y // TODO inform this from xLen （由xLen确定）
    val DC2 = BitPat.dontCare(2) // Makes the listing below more readable （增加可读性）
    def decode_default: List[BitPat] = // 共45位
      List(N, 		// 是否是有效指令
           N, 		// 是否是浮点指令
           X, 		// is single-prec?
           uopX,	// 微代码，即指令类型，7位
           IQT_INT, // 发射队列类型，3位
           FU_X, 	// 功能单元常量，10位（实际上是独热码，即只有1,2,4等）
           RT_X, 	// 目的寄存器类型，2位
           DC2,		// 源寄存器1类型，2位
           DC2,		// 源寄存器2类型，2位
           X, 		// 浮点指令源寄存器3使能信号
           IS_X,	// 立即数拓展选择，3位
           X,		// 使用加载队列
           X, 		// 使用储存队列
           X, 		// is amo?
           X, 		// is fence?
           N, 		// is fencei?
           M_X,   	// 内存操作常量，2位
           DC2, 	// 唤醒延迟
           X, 		// 是否可旁路（即是否有已知/固定延迟）
           X, 		// 是否是分支
           N, 		// 是否是断点或ecall
           N, 		// 是否unique（即需要为它清空流水线）
           X, 		// 是否提交时刷新
           CSR.X)	// CSR命令，3位
  val table: Array[(BitPat, List[BitPat])]
}
```
其中
```scala
  def X = BitPat("b?")
  def N = BitPat("b0")
  def Y = BitPat("b1")
```



### CtrlSigs 类

```scala
// scalastyle:on 开启scala格式
/**
 * Decoded control signals
 * 译码控制信号
 */
  class CtrlSigs extends Bundle
  {
    val legal           = Bool()
    val fp_val          = Bool()
    val fp_single       = Bool()
    val uopc            = UInt(UOPC_SZ.W)
    val iq_type         = UInt(IQT_SZ.W)
    val fu_code         = UInt(FUC_SZ.W)
    val dst_type        = UInt(2.W)
    val rs1_type        = UInt(2.W)
    val rs2_type        = UInt(2.W)
    val frs3_en         = Bool()
    val imm_sel         = UInt(IS_X.getWidth.W)
    val uses_ldq        = Bool()
    val uses_stq        = Bool()
    val is_amo          = Bool()
    val is_fence        = Bool()
    val is_fencei       = Bool()
    val mem_cmd         = UInt(freechips.rocketchip.rocket.M_SZ.W)
    val wakeup_delay    = UInt(2.W)
    val bypassable      = Bool()
    val is_br           = Bool()
    val is_sys_pc2epc   = Bool()
    val inst_unique     = Bool()
    val flush_on_commit = Bool()
    val csr_cmd         = UInt(freechips.rocketchip.rocket.CSR.SZ.W)
      // 以上和decode_default一一对应
    val rocc            = Bool()
      // 指示是否是rocc指令

      // 定义译码方法
      // 参数为UInt形式的指令，以及一个译码表
  def decode(inst: UInt, table: Iterable[(BitPat, List[BitPat])]) = {
    val decoder = freechips.rocketchip.rocket.DecodeLogic(inst, XDecode.decode_default, table)
      // 调用rocketchip中DecodeLogic的一种apply方法
      // 实现代码见补充部分
      
    val sigs =
      Seq(legal, fp_val, fp_single, uopc, iq_type, fu_code, dst_type, rs1_type,
          rs2_type, frs3_en, imm_sel, uses_ldq, uses_stq, is_amo,
          is_fence, is_fencei, mem_cmd, wakeup_delay, bypassable,
          is_br, is_sys_pc2epc, inst_unique, flush_on_commit, csr_cmd)
      // 将上述成员集合到一个序列中
      
      sigs zip decoder map {case(s,d) => s := d} // 得到解码结果
      rocc := false.B // 解码未涉及，置0
      
      this // 返回自身，但是成员已经赋值好了
  }
}
```



### X32Decode 对象 （由 DecodeConstants 继承）

```scala
// scalastyle:off 关闭scala格式
/**
 * Decode constants for RV32
 * RV32译码常数
 */
 object X32Decode extends DecodeConstants
 {
    val table: Array[(BitPat, List[BitPat])] = Array(
        
    SLLI_RV32-> List(Y, N, X, uopSLLI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
        
    SRLI_RV32-> List(Y, N, X, uopSRLI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
        
    SRAI_RV32-> List(Y, N, X, uopSRAI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N)
        
    )
     // table 中多了相应的指令，这里只标出与decode_default不同的部分，下同
     // 第一项为Y，表示指令有效
     // 第四项表示对应指令类型
     // FU项为ALU，目的和第一个源寄存器类型为定点数
     // 指令类型为I-type
     // 不适用加载或储存队列，不是amo或fence
     // 唤醒延迟1
     // 可旁路（固定延时）
     // 提交时不需刷新
 }
```



### X64Decode 对象 （由 DecodeConstants 继承）

```scala
/**
 * Decode constants for RV64
 * RV译码常数
 */
  object X64Decode extends DecodeConstants
  {
    val table: Array[(BitPat, List[BitPat])] = Array(
        
    LD      -> List(Y, N, X, uopLD   , IQT_MEM, FU_MEM , RT_FIX, RT_FIX, RT_X  , N, IS_I, Y, N, N, N, N, M_XRD, 3.U, N, N, N, N, N, CSR.N),
    LWU     -> List(Y, N, X, uopLD   , IQT_MEM, FU_MEM , RT_FIX, RT_FIX, RT_X  , N, IS_I, Y, N, N, N, N, M_XRD, 3.U, N, N, N, N, N, CSR.N),
    SD      -> List(Y, N, X, uopSTA  , IQT_MEM, FU_MEM , RT_X  , RT_FIX, RT_FIX, N, IS_S, N, Y, N, N, N, M_XWR, 0.U, N, N, N, N, N, CSR.N),

  SLLI    -> List(Y, N, X, uopSLLI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SRLI    -> List(Y, N, X, uopSRLI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SRAI    -> List(Y, N, X, uopSRAI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),

  ADDIW   -> List(Y, N, X, uopADDIW, IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SLLIW   -> List(Y, N, X, uopSLLIW, IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SRAIW   -> List(Y, N, X, uopSRAIW, IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SRLIW   -> List(Y, N, X, uopSRLIW, IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),

  ADDW    -> List(Y, N, X, uopADDW , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SUBW    -> List(Y, N, X, uopSUBW , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SLLW    -> List(Y, N, X, uopSLLW , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SRAW    -> List(Y, N, X, uopSRAW , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SRLW    -> List(Y, N, X, uopSRLW , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N)
  )
}
```



### XDecode 对象（由 DecodeConstants 继承）

```scala
/**
 * Overall Decode constants
 * 总体译码常数
 */
  object XDecode extends DecodeConstants
  {
    val table: Array[(BitPat, List[BitPat])] = Array(
        
    LW      -> List(Y, N, X, uopLD   , IQT_MEM, FU_MEM , RT_FIX, RT_FIX, RT_X  , N, IS_I, Y, N, N, N, N, M_XRD, 3.U, N, N, N, N, N, CSR.N),
    LH      -> List(Y, N, X, uopLD   , IQT_MEM, FU_MEM , RT_FIX, RT_FIX, RT_X  , N, IS_I, Y, N, N, N, N, M_XRD, 3.U, N, N, N, N, N, CSR.N),
    LHU     -> List(Y, N, X, uopLD   , IQT_MEM, FU_MEM , RT_FIX, RT_FIX, RT_X  , N, IS_I, Y, N, N, N, N, M_XRD, 3.U, N, N, N, N, N, CSR.N),
    LB      -> List(Y, N, X, uopLD   , IQT_MEM, FU_MEM , RT_FIX, RT_FIX, RT_X  , N, IS_I, Y, N, N, N, N, M_XRD, 3.U, N, N, N, N, N, CSR.N),
    LBU     -> List(Y, N, X, uopLD   , IQT_MEM, FU_MEM , RT_FIX, RT_FIX, RT_X  , N, IS_I, Y, N, N, N, N, M_XRD, 3.U, N, N, N, N, N, CSR.N),

  SW      -> List(Y, N, X, uopSTA  , IQT_MEM, FU_MEM , RT_X  , RT_FIX, RT_FIX, N, IS_S, N, Y, N, N, N, M_XWR, 0.U, N, N, N, N, N, CSR.N),
  SH      -> List(Y, N, X, uopSTA  , IQT_MEM, FU_MEM , RT_X  , RT_FIX, RT_FIX, N, IS_S, N, Y, N, N, N, M_XWR, 0.U, N, N, N, N, N, CSR.N),
  SB      -> List(Y, N, X, uopSTA  , IQT_MEM, FU_MEM , RT_X  , RT_FIX, RT_FIX, N, IS_S, N, Y, N, N, N, M_XWR, 0.U, N, N, N, N, N, CSR.N),

  LUI     -> List(Y, N, X, uopLUI  , IQT_INT, FU_ALU , RT_FIX, RT_X  , RT_X  , N, IS_U, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),

  ADDI    -> List(Y, N, X, uopADDI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  ANDI    -> List(Y, N, X, uopANDI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  ORI     -> List(Y, N, X, uopORI  , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  XORI    -> List(Y, N, X, uopXORI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SLTI    -> List(Y, N, X, uopSLTI , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SLTIU   -> List(Y, N, X, uopSLTIU, IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),

  SLL     -> List(Y, N, X, uopSLL  , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  ADD     -> List(Y, N, X, uopADD  , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SUB     -> List(Y, N, X, uopSUB  , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SLT     -> List(Y, N, X, uopSLT  , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SLTU    -> List(Y, N, X, uopSLTU , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  AND     -> List(Y, N, X, uopAND  , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  OR      -> List(Y, N, X, uopOR   , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  XOR     -> List(Y, N, X, uopXOR  , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SRA     -> List(Y, N, X, uopSRA  , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_I, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),
  SRL     -> List(Y, N, X, uopSRL  , IQT_INT, FU_ALU , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 1.U, Y, N, N, N, N, CSR.N),

  MUL     -> List(Y, N, X, uopMUL  , IQT_INT, FU_MUL , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  MULH    -> List(Y, N, X, uopMULH , IQT_INT, FU_MUL , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  MULHU   -> List(Y, N, X, uopMULHU, IQT_INT, FU_MUL , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  MULHSU  -> List(Y, N, X, uopMULHSU,IQT_INT, FU_MUL , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  MULW    -> List(Y, N, X, uopMULW , IQT_INT, FU_MUL , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  DIV     -> List(Y, N, X, uopDIV  , IQT_INT, FU_DIV , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  DIVU    -> List(Y, N, X, uopDIVU , IQT_INT, FU_DIV , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  REM     -> List(Y, N, X, uopREM  , IQT_INT, FU_DIV , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  REMU    -> List(Y, N, X, uopREMU , IQT_INT, FU_DIV , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  DIVW    -> List(Y, N, X, uopDIVW , IQT_INT, FU_DIV , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  DIVUW   -> List(Y, N, X, uopDIVUW, IQT_INT, FU_DIV , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  REMW    -> List(Y, N, X, uopREMW , IQT_INT, FU_DIV , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  REMUW   -> List(Y, N, X, uopREMUW, IQT_INT, FU_DIV , RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  AUIPC   -> List(Y, N, X, uopAUIPC, IQT_INT, FU_JMP , RT_FIX, RT_X  , RT_X  , N, IS_U, N, N, N, N, N, M_X  , 1.U, N, N, N, N, N, CSR.N), // use BRU for the PC read
  JAL     -> List(Y, N, X, uopJAL  , IQT_INT, FU_JMP , RT_FIX, RT_X  , RT_X  , N, IS_J, N, N, N, N, N, M_X  , 1.U, N, N, N, N, N, CSR.N),
  JALR    -> List(Y, N, X, uopJALR , IQT_INT, FU_JMP , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 1.U, N, N, N, N, N, CSR.N),
  BEQ     -> List(Y, N, X, uopBEQ  , IQT_INT, FU_ALU , RT_X  , RT_FIX, RT_FIX, N, IS_B, N, N, N, N, N, M_X  , 0.U, N, Y, N, N, N, CSR.N),
  BNE     -> List(Y, N, X, uopBNE  , IQT_INT, FU_ALU , RT_X  , RT_FIX, RT_FIX, N, IS_B, N, N, N, N, N, M_X  , 0.U, N, Y, N, N, N, CSR.N),
  BGE     -> List(Y, N, X, uopBGE  , IQT_INT, FU_ALU , RT_X  , RT_FIX, RT_FIX, N, IS_B, N, N, N, N, N, M_X  , 0.U, N, Y, N, N, N, CSR.N),
  BGEU    -> List(Y, N, X, uopBGEU , IQT_INT, FU_ALU , RT_X  , RT_FIX, RT_FIX, N, IS_B, N, N, N, N, N, M_X  , 0.U, N, Y, N, N, N, CSR.N),
  BLT     -> List(Y, N, X, uopBLT  , IQT_INT, FU_ALU , RT_X  , RT_FIX, RT_FIX, N, IS_B, N, N, N, N, N, M_X  , 0.U, N, Y, N, N, N, CSR.N),
  BLTU    -> List(Y, N, X, uopBLTU , IQT_INT, FU_ALU , RT_X  , RT_FIX, RT_FIX, N, IS_B, N, N, N, N, N, M_X  , 0.U, N, Y, N, N, N, CSR.N),

  // I-type, the immediate12 holds the CSR register.
  CSRRW   -> List(Y, N, X, uopCSRRW, IQT_INT, FU_CSR , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.W),
  CSRRS   -> List(Y, N, X, uopCSRRS, IQT_INT, FU_CSR , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.S),
  CSRRC   -> List(Y, N, X, uopCSRRC, IQT_INT, FU_CSR , RT_FIX, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.C),

  CSRRWI  -> List(Y, N, X, uopCSRRWI,IQT_INT, FU_CSR , RT_FIX, RT_PAS, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.W),
  CSRRSI  -> List(Y, N, X, uopCSRRSI,IQT_INT, FU_CSR , RT_FIX, RT_PAS, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.S),
  CSRRCI  -> List(Y, N, X, uopCSRRCI,IQT_INT, FU_CSR , RT_FIX, RT_PAS, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.C),

  SFENCE_VMA->List(Y,N, X, uopSFENCE,IQT_MEM, FU_MEM , RT_X  , RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N,M_SFENCE,0.U,N, N, N, Y, Y, CSR.N),
  SCALL   -> List(Y, N, X, uopERET  ,IQT_INT, FU_CSR , RT_X  , RT_X  , RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, Y, Y, Y, CSR.I),
  SBREAK  -> List(Y, N, X, uopERET  ,IQT_INT, FU_CSR , RT_X  , RT_X  , RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, Y, Y, Y, CSR.I),
  SRET    -> List(Y, N, X, uopERET  ,IQT_INT, FU_CSR , RT_X  , RT_X  , RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.I),
  MRET    -> List(Y, N, X, uopERET  ,IQT_INT, FU_CSR , RT_X  , RT_X  , RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.I),
  DRET    -> List(Y, N, X, uopERET  ,IQT_INT, FU_CSR , RT_X  , RT_X  , RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.I),

  WFI     -> List(Y, N, X, uopWFI   ,IQT_INT, FU_CSR , RT_X  , RT_X  , RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, Y, Y, CSR.I),

  FENCE_I -> List(Y, N, X, uopNOP  , IQT_INT, FU_X   , RT_X  , RT_X  , RT_X  , N, IS_X, N, N, N, N, Y, M_X  , 0.U, N, N, N, Y, Y, CSR.N),
  FENCE   -> List(Y, N, X, uopFENCE, IQT_INT, FU_MEM , RT_X  , RT_X  , RT_X  , N, IS_X, N, Y, N, Y, N, M_X  , 0.U, N, N, N, Y, Y, CSR.N), // TODO PERF make fence higher performance
  // currently serializes pipeline （当前序列化流水线？）
        
  // A-type
  AMOADD_W-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_ADD, 0.U,N, N, N, Y, Y, CSR.N), // TODO make AMOs higherperformance
  AMOXOR_W-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_XOR, 0.U,N, N, N, Y, Y, CSR.N),
  AMOSWAP_W->List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_SWAP,0.U,N, N, N, Y, Y, CSR.N),
  AMOAND_W-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_AND, 0.U,N, N, N, Y, Y, CSR.N),
  AMOOR_W -> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_OR,  0.U,N, N, N, Y, Y, CSR.N),
  AMOMIN_W-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_MIN, 0.U,N, N, N, Y, Y, CSR.N),
  AMOMINU_W->List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_MINU,0.U,N, N, N, Y, Y, CSR.N),
  AMOMAX_W-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_MAX, 0.U,N, N, N, Y, Y, CSR.N),
  AMOMAXU_W->List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_MAXU,0.U,N, N, N, Y, Y, CSR.N),

  AMOADD_D-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_ADD, 0.U,N, N, N, Y, Y, CSR.N),
  AMOXOR_D-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_XOR, 0.U,N, N, N, Y, Y, CSR.N),
  AMOSWAP_D->List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_SWAP,0.U,N, N, N, Y, Y, CSR.N),
  AMOAND_D-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_AND, 0.U,N, N, N, Y, Y, CSR.N),
  AMOOR_D -> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_OR,  0.U,N, N, N, Y, Y, CSR.N),
  AMOMIN_D-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_MIN, 0.U,N, N, N, Y, Y, CSR.N),
  AMOMINU_D->List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_MINU,0.U,N, N, N, Y, Y, CSR.N),
  AMOMAX_D-> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_MAX, 0.U,N, N, N, Y, Y, CSR.N),
  AMOMAXU_D->List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XA_MAXU,0.U,N, N, N, Y, Y, CSR.N),

  LR_W    -> List(Y, N, X, uopLD    , IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_X  , N, IS_X, Y, N, N, N, N, M_XLR   , 0.U,N, N, N, Y, Y, CSR.N),
  LR_D    -> List(Y, N, X, uopLD    , IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_X  , N, IS_X, Y, N, N, N, N, M_XLR   , 0.U,N, N, N, Y, Y, CSR.N),
  SC_W    -> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XSC   , 0.U,N, N, N, Y, Y, CSR.N),
  SC_D    -> List(Y, N, X, uopAMO_AG, IQT_MEM, FU_MEM, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, Y, Y, N, N, M_XSC   , 0.U,N, N, N, Y, Y, CSR.N)
  )
}
```



### FDecode 对象 （由 DecodeConstants 继承）

```scala
/**
 * FP Decode constants
 * 浮点译码常数
 */
  object FDecode extends DecodeConstants
  {
    val table: Array[(BitPat, List[BitPat])] = Array(
        
    FLW     -> List(Y, Y, Y, uopLD     , IQT_MEM, FU_MEM, RT_FLT, RT_FIX, RT_X  , N, IS_I, Y, N, N, N, N, M_XRD, 0.U, N, N, N, N, N, CSR.N),
    FLD     -> List(Y, Y, N, uopLD     , IQT_MEM, FU_MEM, RT_FLT, RT_FIX, RT_X  , N, IS_I, Y, N, N, N, N, M_XRD, 0.U, N, N, N, N, N, CSR.N),
    FSW     -> List(Y, Y, Y, uopSTA    , IQT_MFP,FU_F2IMEM,RT_X , RT_FIX, RT_FLT, N, IS_S, N, Y, N, N, N, M_XWR, 0.U, N, N, N, N, N, CSR.N), // sort of a lie; broken into two micro-ops
    FSD     -> List(Y, Y, N, uopSTA    , IQT_MFP,FU_F2IMEM,RT_X , RT_FIX, RT_FLT, N, IS_S, N, Y, N, N, N, M_XWR, 0.U, N, N, N, N, N, CSR.N),

  FCLASS_S-> List(Y, Y, Y, uopFCLASS_S,IQT_FP , FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCLASS_D-> List(Y, Y, N, uopFCLASS_D,IQT_FP , FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  FMV_S_X -> List(Y, Y, Y, uopFMV_S_X, IQT_INT, FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMV_D_X -> List(Y, Y, N, uopFMV_D_X, IQT_INT, FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMV_X_S -> List(Y, Y, Y, uopFMV_X_S, IQT_FP , FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMV_X_D -> List(Y, Y, N, uopFMV_X_D, IQT_FP , FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  FSGNJ_S -> List(Y, Y, Y, uopFSGNJ_S, IQT_FP , FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FSGNJ_D -> List(Y, Y, N, uopFSGNJ_D, IQT_FP , FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FSGNJX_S-> List(Y, Y, Y, uopFSGNJ_S, IQT_FP , FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FSGNJX_D-> List(Y, Y, N, uopFSGNJ_D, IQT_FP , FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FSGNJN_S-> List(Y, Y, Y, uopFSGNJ_S, IQT_FP , FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FSGNJN_D-> List(Y, Y, N, uopFSGNJ_D, IQT_FP , FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  // FP to FP
  FCVT_S_D-> List(Y, Y, Y, uopFCVT_S_D,IQT_FP , FU_FPU, RT_FLT, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_D_S-> List(Y, Y, N, uopFCVT_D_S,IQT_FP , FU_FPU, RT_FLT, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  // Int to FP
  FCVT_S_W-> List(Y, Y, Y, uopFCVT_S_X, IQT_INT,FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_S_WU->List(Y, Y, Y, uopFCVT_S_X, IQT_INT,FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_S_L-> List(Y, Y, Y, uopFCVT_S_X, IQT_INT,FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_S_LU->List(Y, Y, Y, uopFCVT_S_X, IQT_INT,FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  FCVT_D_W-> List(Y, Y, N, uopFCVT_D_X, IQT_INT,FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_D_WU->List(Y, Y, N, uopFCVT_D_X, IQT_INT,FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_D_L-> List(Y, Y, N, uopFCVT_D_X, IQT_INT,FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_D_LU->List(Y, Y, N, uopFCVT_D_X, IQT_INT,FU_I2F, RT_FLT, RT_FIX, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  // FP to Int
  FCVT_W_S-> List(Y, Y, Y, uopFCVT_X_S, IQT_FP, FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_WU_S->List(Y, Y, Y, uopFCVT_X_S, IQT_FP, FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_L_S-> List(Y, Y, Y, uopFCVT_X_S, IQT_FP, FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_LU_S->List(Y, Y, Y, uopFCVT_X_S, IQT_FP, FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  FCVT_W_D-> List(Y, Y, N, uopFCVT_X_D, IQT_FP, FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_WU_D->List(Y, Y, N, uopFCVT_X_D, IQT_FP, FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_L_D-> List(Y, Y, N, uopFCVT_X_D, IQT_FP, FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FCVT_LU_D->List(Y, Y, N, uopFCVT_X_D, IQT_FP, FU_F2I, RT_FIX, RT_FLT, RT_X  , N, IS_I, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  // "fp_single" is used for wb_data formatting (and debugging)
  FEQ_S    ->List(Y, Y, Y, uopCMPR_S , IQT_FP,  FU_F2I, RT_FIX, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FLT_S    ->List(Y, Y, Y, uopCMPR_S , IQT_FP,  FU_F2I, RT_FIX, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FLE_S    ->List(Y, Y, Y, uopCMPR_S , IQT_FP,  FU_F2I, RT_FIX, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  FEQ_D    ->List(Y, Y, N, uopCMPR_D , IQT_FP,  FU_F2I, RT_FIX, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FLT_D    ->List(Y, Y, N, uopCMPR_D , IQT_FP,  FU_F2I, RT_FIX, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FLE_D    ->List(Y, Y, N, uopCMPR_D , IQT_FP,  FU_F2I, RT_FIX, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  FMIN_S   ->List(Y, Y, Y,uopFMINMAX_S,IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMAX_S   ->List(Y, Y, Y,uopFMINMAX_S,IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMIN_D   ->List(Y, Y, N,uopFMINMAX_D,IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMAX_D   ->List(Y, Y, N,uopFMINMAX_D,IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  FADD_S   ->List(Y, Y, Y, uopFADD_S , IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FSUB_S   ->List(Y, Y, Y, uopFSUB_S , IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMUL_S   ->List(Y, Y, Y, uopFMUL_S , IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FADD_D   ->List(Y, Y, N, uopFADD_D , IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FSUB_D   ->List(Y, Y, N, uopFSUB_D , IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMUL_D   ->List(Y, Y, N, uopFMUL_D , IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),

  FMADD_S  ->List(Y, Y, Y, uopFMADD_S, IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, Y, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMSUB_S  ->List(Y, Y, Y, uopFMSUB_S, IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, Y, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FNMADD_S ->List(Y, Y, Y, uopFNMADD_S,IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, Y, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FNMSUB_S ->List(Y, Y, Y, uopFNMSUB_S,IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, Y, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMADD_D  ->List(Y, Y, N, uopFMADD_D, IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, Y, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FMSUB_D  ->List(Y, Y, N, uopFMSUB_D, IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, Y, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FNMADD_D ->List(Y, Y, N, uopFNMADD_D,IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, Y, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
  FNMSUB_D ->List(Y, Y, N, uopFNMSUB_D,IQT_FP,  FU_FPU, RT_FLT, RT_FLT, RT_FLT, Y, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N)
  )
}
```



### FDivSqrtDecode 对象 （由 DecodeConstants 继承）

```scala
/**
 * FP Divide SquareRoot Constants
 * 浮点除法和平方根常数
 */
 object FDivSqrtDecode extends DecodeConstants
 {
    val table: Array[(BitPat, List[BitPat])] = Array(
        
    FDIV_S    ->List(Y, Y, Y, uopFDIV_S , IQT_FP, FU_FDV, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    FDIV_D    ->List(Y, Y, N, uopFDIV_D , IQT_FP, FU_FDV, RT_FLT, RT_FLT, RT_FLT, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    FSQRT_S   ->List(Y, Y, Y, uopFSQRT_S, IQT_FP, FU_FDV, RT_FLT, RT_FLT, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    FSQRT_D   ->List(Y, Y, N, uopFSQRT_D, IQT_FP, FU_FDV, RT_FLT, RT_FLT, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N)
        
    )
 }
```



### RoCCDecode 对象 （由 DecodeConstants 继承）

```scala
//scalastyle:on 打开scala格式

/**
 * RoCC initial decode
 * RoCC初始化译码
 */
 object RoCCDecode extends DecodeConstants
 {
    // Note: We use FU_CSR since CSR instructions cannot co-execute with RoCC instructions
	// 注意：我们使用FU_CSR，因为CSR指令不能和RoCC指令一起执行
    val table: Array[(BitPat, List[BitPat])] = Array(
    CUSTOM0            ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_X  , RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM0_RS1        ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_FIX, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM0_RS1_RS2    ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM0_RD         ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_X  , RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM0_RD_RS1     ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_FIX, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM0_RD_RS1_RS2 ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM1            ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_X  , RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM1_RS1        ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_FIX, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM1_RS1_RS2    ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM1_RD         ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_X  , RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM1_RD_RS1     ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_FIX, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM1_RD_RS1_RS2 ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM2            ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_X  , RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM2_RS1        ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_FIX, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM2_RS1_RS2    ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM2_RD         ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_X  , RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM2_RD_RS1     ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_FIX, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM2_RD_RS1_RS2 ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM3            ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_X  , RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM3_RS1        ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_FIX, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM3_RS1_RS2    ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_X  , RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM3_RD         ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_X  , RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM3_RD_RS1     ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_FIX, RT_X  , N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N),
    CUSTOM3_RD_RS1_RS2 ->List(Y, N, X, uopROCC   , IQT_INT, FU_CSR, RT_FIX, RT_FIX, RT_FIX, N, IS_X, N, N, N, N, N, M_X  , 0.U, N, N, N, N, N, CSR.N)
    )
 }
```



### DecodeUnitIo 类

```scala
/**
 * IO bundle for the Decode unit
 * 译码单元IO端口
 */
  class DecodeUnitIo(implicit p: Parameters) extends BoomBundle
  {
    val enq = new Bundle { val uop = Input(new MicroOp()) } 	//【输入】入列的微指令
    val deq = new Bundle { val uop = Output(new MicroOp()) } 	//【输出】出列的微指令

  // from CSRFile （从CSR堆来的【输入】信号）
  val status = Input(new freechips.rocketchip.rocket.MStatus())
      // 机器模式状态寄存器（不是其真的一部分，但是使用起来方便）
  val csr_decode = Flipped(new freechips.rocketchip.rocket.CSRDecodeIO)
      // 与CSR译码的IO端口连接
  val interrupt = Input(Bool()) // 中断信号
  val interrupt_cause = Input(UInt(xLen.W)) // 中断原因
}
```



### DecodeUnit 类

```scala
/**
 * Decode unit that takes in a single instruction and generates a MicroOp.
 * 译码单元，输入一条指令，生成一条微指令（微操作，MicroOp）
 */
  class DecodeUnit(implicit p: Parameters) extends BoomModule
    with freechips.rocketchip.rocket.constants.MemoryOpConstants
  {
    val io = IO(new DecodeUnitIo) // IO端口为上面定义的类

  val uop = Wire(new MicroOp())
  uop := io.enq.uop // 记录输入的微指令

  var decode_table = XDecode.table
  if (usingFPU) decode_table ++= FDecode.table
  if (usingFPU && usingFDivSqrt) decode_table ++= FDivSqrtDecode.table
  if (usingRoCC) decode_table ++= RoCCDecode.table
  decode_table ++= (if (xLen == 64) X64Decode.table else X32Decode.table)
      // 根据使用的资源，扩展译码表的内容

  val inst = uop.inst // 记录输入的指令

  val cs = Wire(new CtrlSigs()).decode(inst, decode_table) // 得到控制信号

      
  // Exception Handling （异常处理）
  io.csr_decode.csr := inst(31,20)
  val csr_en = cs.csr_cmd.isOneOf(CSR.S, CSR.C, CSR.W)
  val csr_ren = cs.csr_cmd.isOneOf(CSR.S, CSR.C) && uop.lrs1 === 0.U
  val system_insn = cs.csr_cmd === CSR.I
  val sfence = cs.uopc === uopSFENCE

  val cs_legal = cs.legal
	//   dontTouch(cs_legal)  ，防止被优化掉

      
    // ？？非法指令
  val id_illegal_insn = !cs_legal ||
    cs.fp_val && io.csr_decode.fp_illegal || 
      // TODO check for illegal rm mode: (io.fpu.illegal_rm)
    cs.rocc && io.csr_decode.rocc_illegal ||
    cs.is_amo && !io.status.isa('a'-'a')  ||
    (cs.fp_val && !cs.fp_single) && !io.status.isa('d'-'a') ||
    csr_en && (io.csr_decode.read_illegal || !csr_ren && io.csr_decode.write_illegal) ||
    ((sfence || system_insn) && io.csr_decode.system_illegal)
	  // cs.div && !csr.io.status.isa('m'-'a') || TODO check for illegal div instructions

      
      // 定义检查异常的方法
      // 输入一个元素为Bool值（是否有异常）和UInt值（异常原因）原组的序列
      // 得到指示是否有异常的Bool值，和第一个异常的原因
  def checkExceptions(x: Seq[(Bool, UInt)]) =
    (x.map(_._1).reduce(_||_), PriorityMux(x))

      // 用这个方法得到微指令的异常情况（有效位和原因）
  val (xcpt_valid, xcpt_cause) = checkExceptions(List(
    (io.interrupt && !io.enq.uop.is_sfb, io.interrupt_cause),  // Disallow interrupts while we are handling a SFB （正在进行SFB优化时，禁用异常）
    (uop.bp_debug_if,                    (CSR.debugTriggerCause).U),
    (uop.bp_xcpt_if,                     (Causes.breakpoint).U),
    (uop.xcpt_pf_if,                     (Causes.fetch_page_fault).U),
    (uop.xcpt_ae_if,                     (Causes.fetch_access).U),
    (id_illegal_insn,                    (Causes.illegal_instruction).U)))
      // 依次列出所有可能的异常情况

  uop.exception := xcpt_valid // 将异常信息写入uop
  uop.exc_cause := xcpt_cause

      
  //-------------------------------------------------------------
  // 将译码得到的控制信号cs中的信息写入uop
      
  uop.uopc       := cs.uopc
  uop.iq_type    := cs.iq_type
  uop.fu_code    := cs.fu_code

  // x-registers placed in 0-31, f-registers placed in 32-63.
  // This allows us to straight-up compare register specifiers and not need to
  // verify the rtypes (e.g., bypassing in rename).
  // 整数寄存器放在0-31，浮点寄存器放在32-63，这样就可以直接通过寄存器标号直接得知其类型，而不需rtypes.
  // 参数在Const.scala中的一个特质中有定义
  uop.ldst       := inst(RD_MSB,RD_LSB) // 11-7
  uop.lrs1       := inst(RS1_MSB,RS1_LSB) // 19-15
  uop.lrs2       := inst(RS2_MSB,RS2_LSB) // 24-20
  uop.lrs3       := inst(RS3_MSB,RS3_LSB) // 31-27

  uop.ldst_val   := cs.dst_type =/= RT_X && !(uop.ldst === 0.U && uop.dst_rtype === RT_FIX) 
      // 逻辑目的寄存器有效： 有明确类型 且 不是x0或者是定点数
  uop.dst_rtype  := cs.dst_type
  uop.lrs1_rtype := cs.rs1_type
  uop.lrs2_rtype := cs.rs2_type
  uop.frs3_en    := cs.frs3_en

  uop.ldst_is_rs1 := uop.is_sfb_shadow
      // 目的寄存器和第一个源寄存器相同 即 这条指令被预测 ？
      
      
  // SFB optimization     SFB优化
  when (uop.is_sfb_shadow && cs.rs2_type === RT_X) { // 被预测，无第二个操作数
    uop.lrs2_rtype  := RT_FIX
    uop.lrs2        := inst(RD_MSB,RD_LSB) // 将第二个操作数改为目的寄存器
    uop.ldst_is_rs1 := false.B
  } .elsewhen (uop.is_sfb_shadow && cs.uopc === uopADD && inst(RS1_MSB,RS1_LSB) === 0.U) { 
      // 被预测，ADD指令，第一个操作数是0
    uop.uopc        := uopMOV	// 改为移动指令
    uop.lrs1        := inst(RD_MSB, RD_LSB) // 第一个操作数改为目的寄存器
    uop.ldst_is_rs1 := true.B
  }
  when (uop.is_sfb_br) { // 是预测？
    uop.fu_code := FU_JMP // 功能单元改为跳转单元
  }


  uop.fp_val     := cs.fp_val
  uop.fp_single  := cs.fp_single // TODO use this signal instead of the FPU decode's table signal?

  uop.mem_cmd    := cs.mem_cmd
  uop.mem_size   := Mux(cs.mem_cmd.isOneOf(M_SFENCE, M_FLUSH_ALL), Cat(uop.lrs2 =/= 0.U, uop.lrs1 =/= 0.U), inst(13,12))
  uop.mem_signed := !inst(14)
  uop.uses_ldq   := cs.uses_ldq
  uop.uses_stq   := cs.uses_stq
  uop.is_amo     := cs.is_amo
  uop.is_fence   := cs.is_fence
  uop.is_fencei  := cs.is_fencei
  uop.is_sys_pc2epc   := cs.is_sys_pc2epc
  uop.is_unique  := cs.inst_unique
  uop.flush_on_commit := cs.flush_on_commit || (csr_en && !csr_ren && io.csr_decode.write_flush)

  uop.bypassable   := cs.bypassable

  //-------------------------------------------------------------
  // immediates （立即数）

  // repackage the immediate, and then pass the fewest number of bits around
  // (根据指令类型)重新打包立即数，然后传递最少的位
  val di24_20 = Mux(cs.imm_sel === IS_B || cs.imm_sel === IS_S, inst(11,7), inst(24,20))
  uop.imm_packed := Cat(inst(31,25), di24_20, inst(19,12))

  //-------------------------------------------------------------

  uop.is_br          := cs.is_br
  uop.is_jal         := (uop.uopc === uopJAL)
  uop.is_jalr        := (uop.uopc === uopJALR)
  // uop.is_jump        := cs.is_jal || (uop.uopc === uopJALR)
  // uop.is_ret         := (uop.uopc === uopJALR) &&
  //                       (uop.ldst === X0) &&
  //                       (uop.lrs1 === RA)
  // uop.is_call        := (uop.uopc === uopJALR || uop.uopc === uopJAL) &&
  //                       (uop.ldst === RA)

  //-------------------------------------------------------------

      
  io.deq.uop := uop // 把处理后的uop作为输出
}
```



### BranchDecodeSignals 类

分支译码信号。

```scala
/**
 * Smaller Decode unit for the Frontend to decode different
 * branches.
 * Accepts EXPANDED RVC instructions
 * 前端译码不同分支的小型译码单元。支持扩展RVC指令。
    */

class BranchDecodeSignals(implicit p: Parameters) extends BoomBundle
{
  val is_ret   = Bool() // 是否返回
  val is_call  = Bool() // 是否调用
  val target   = UInt(vaddrBitsExtended.W) // 目标地址
  val cfi_type = UInt(CFI_SZ.W) // 控制流指令种类（X,BR,JAL,JALR），3位


  // Is this branch a short forwards jump? （这个分支是小的前向跳转？）
  val sfb_offset = Valid(UInt(log2Ceil(icBlockBytes).W)) 
  // Is this instruction allowed to be inside a sfb? （这条指令同意被收入sfb？）
  val shadowable = Bool()
}
```



### BranchDecode 类

分支译码。

```scala
class BranchDecode(implicit p: Parameters) extends BoomModule
{
  val io = IO(new Bundle {
    val inst    = Input(UInt(32.W)) // 【输入】指令
    val pc      = Input(UInt(vaddrBitsExtended.W)) // 【输入】PC

    val out = Output(new BranchDecodeSignals) // 【输出】上面定义的类
  })

  val bpd_csignals =   // 调用rocketchip的译码函数，得到BitPat控制信号。
    freechips.rocketchip.rocket.DecodeLogic(io.inst,
                  List[BitPat](N, N, N, N, X),
////                               is br?
////                               |  is jal?
////                               |  |  is jalr?
////                               |  |  |
////                               |  |  |  shadowable
////                               |  |  |  |  has_rs2
////                               |  |  |  |  |
            Array[(BitPat, List[BitPat])](
               JAL         -> List(N, Y, N, N, X),
               JALR        -> List(N, N, Y, N, X),
               BEQ         -> List(Y, N, N, N, X),
               BNE         -> List(Y, N, N, N, X),
               BGE         -> List(Y, N, N, N, X),
               BGEU        -> List(Y, N, N, N, X),
               BLT         -> List(Y, N, N, N, X),
               BLTU        -> List(Y, N, N, N, X),

               SLLI        -> List(N, N, N, Y, N),
               SRLI        -> List(N, N, N, Y, N),
               SRAI        -> List(N, N, N, Y, N),
    
               ADDIW       -> List(N, N, N, Y, N),
               SLLIW       -> List(N, N, N, Y, N),
               SRAIW       -> List(N, N, N, Y, N),
               SRLIW       -> List(N, N, N, Y, N),
    
               ADDW        -> List(N, N, N, Y, Y),
               SUBW        -> List(N, N, N, Y, Y),
               SLLW        -> List(N, N, N, Y, Y),
               SRAW        -> List(N, N, N, Y, Y),
               SRLW        -> List(N, N, N, Y, Y),
    
               LUI         -> List(N, N, N, Y, N),
    
               ADDI        -> List(N, N, N, Y, N),
               ANDI        -> List(N, N, N, Y, N),
               ORI         -> List(N, N, N, Y, N),
               XORI        -> List(N, N, N, Y, N),
               SLTI        -> List(N, N, N, Y, N),
               SLTIU       -> List(N, N, N, Y, N),
    
               SLL         -> List(N, N, N, Y, Y),
               ADD         -> List(N, N, N, Y, Y),
               SUB         -> List(N, N, N, Y, Y),
               SLT         -> List(N, N, N, Y, Y),
               SLTU        -> List(N, N, N, Y, Y),
               AND         -> List(N, N, N, Y, Y),
               OR          -> List(N, N, N, Y, Y),
               XOR         -> List(N, N, N, Y, Y),
               SRA         -> List(N, N, N, Y, Y),
               SRL         -> List(N, N, N, Y, Y)
            ))

  val (cs_is_br: Bool) :: (cs_is_jal: Bool) :: (cs_is_jalr:Bool) :: (cs_is_shadowable:Bool) :: (cs_has_rs2) :: Nil = bpd_csignals
    // 枚举，提取控制信号

    
    // 对提取的控制信号做进一步处理，得到输出
  io.out.is_call := (cs_is_jal || cs_is_jalr) && GetRd(io.inst) === RA
  io.out.is_ret  := cs_is_jalr && GetRs1(io.inst) === BitPat("b00?01") && GetRd(io.inst) === X0

  io.out.target := Mux(cs_is_br, ComputeBranchTarget(io.pc, io.inst, xLen),
                                 ComputeJALTarget(io.pc, io.inst, xLen))
  io.out.cfi_type :=
    Mux(cs_is_jalr,
      CFI_JALR,
    Mux(cs_is_jal,
      CFI_JAL,
    Mux(cs_is_br,
      CFI_BR,
      CFI_X)))

    
  val br_offset = Cat(io.inst(7), io.inst(30,25), io.inst(11,8), 0.U(1.W))
  // Is a sfb if it points forwards (offset is positive)
  // 如果是指向前（偏移量为正），是一条sfb指令
  io.out.sfb_offset.valid := cs_is_br && !io.inst(31) && br_offset =/= 0.U && (br_offset >> log2Ceil(icBlockBytes)) === 0.U
  io.out.sfb_offset.bits  := br_offset
  io.out.shadowable := cs_is_shadowable && (
    !cs_has_rs2 ||
    (GetRs1(io.inst) === GetRd(io.inst)) ||
    (io.inst === ADD && GetRs1(io.inst) === X0)
  )
}
```



### BranchMaskGenerationLogic 类

分支掩码生成逻辑。

```scala
/**
 * Track the current "branch mask", and give out the branch mask to each micro-op in Decode
 * (each micro-op in the machine has a branch mask which says which branches it
 * is being speculated under).
 * 跟踪当前的“分支掩码”，为译码级中的每条微指令分发分支掩码。
 *
 * @param pl_width pipeline width for the processor
 */
  class BranchMaskGenerationLogic(val pl_width: Int)(implicit p: Parameters) extends BoomModule
  {
    val io = IO(new Bundle {
        
    // guess if the uop is a branch (we'll catch this later)
    val is_branch = Input(Vec(pl_width, Bool()))
        // 【输入】指示指令是否是分支指令
    // lock in that it's actually a branch and will fire, so we update
    // the branch_masks.
    val will_fire = Input(Vec(pl_width, Bool()))
        // 【输入】指示指令是否将发射。如果是分支且将发射，需要更新分支掩码。

        
    // give out tag immediately (needed in rename)
    // mask can come later in the cycle
    // 【输出】
    // 分支标签需要立即被分发，以便重命名中使用
    // 而分支掩码可以在周期中晚一点实现
    val br_tag    = Output(Vec(pl_width, UInt(brTagSz.W)))
    val br_mask   = Output(Vec(pl_width, UInt(maxBrCount.W)))

        
     // tell decoders the branch mask has filled up, but on the granularity
     // of an individual micro-op (so some micro-ops can go through)
    val is_full   = Output(Vec(pl_width, Bool()))
        // 【输出】告诉译码器分支掩码是否已被填满。但取决于单个微操作的粒度（因此某些微操作可以通过）。

        
    val brupdate         = Input(new BrUpdateInfo()) // 【输入】分支更新信息
    val flush_pipeline = Input(Bool()) // 【输入】清空流水线的信号

    val debug_branch_mask = Output(UInt(maxBrCount.W)) // 【输出】debug用的分支掩码
    })

      
  val branch_mask = RegInit(0.U(maxBrCount.W)) // 初始化分支掩码寄存器

 
      
  //-------------------------------------------------------------
  // Give out the branch tag to each branch micro-op （为每条分支微指令分发分支标签）

  var allocate_mask = branch_mask // 变量，已分配的所有分支掩码，和当前的分支掩码相同。
  val tag_masks = Wire(Vec(pl_width, UInt(maxBrCount.W))) // 标签掩码组。

      
  for (w <- 0 until pl_width) { // 遍历每条指令
    // TODO this is a loss of performance as we're blocking branches based on potentially fake branches （这是一种性能损失，因为我们正在基于潜在的伪分支阻止分支）
    io.is_full(w) := (allocate_mask === ~(0.U(maxBrCount.W))) && io.is_branch(w)
      // 已达到最大分支数，又新进分支指令，则指示已满。

      
    // find br_tag and compute next br_mask （寻找分支标签并计算下一个分支掩码）
    val new_br_tag = Wire(UInt(brTagSz.W))
    new_br_tag := 0.U
    tag_masks(w) := 0.U // 初始值
    
      
    for (i <- maxBrCount-1 to 0 by -1) { 
      when (~allocate_mask(i)) { // 找到最小的分支序号，记录并生成对应的掩码
        new_br_tag := i.U
        tag_masks(w) := (1.U << i.U)
      }
    }
    
      
    io.br_tag(w) := new_br_tag
    allocate_mask = Mux(io.is_branch(w), tag_masks(w) | allocate_mask, allocate_mask)
      // 是分支指令，则将这个变量加上新分配的掩码，否则保持
  }

      
      
  //-------------------------------------------------------------
  // Give out the branch mask to each micro-op
  // (kill off the bits that corresponded to branches that aren't going to fire)
  // 给每条微指令分发分支掩码（清除与不发射的分支相对应的部分）

  var curr_mask = branch_mask // 变量，保存当前分支掩码
  for (w <- 0 until pl_width) {
    io.br_mask(w) := GetNewBrMask(io.brupdate, curr_mask)
      // 根据分支更新信息和当前分支掩码，得到输出的新的分支掩码
      // GetNewBrMask的功能是如果该分支tag已被解决，将curr_mask置为0
    curr_mask = Mux(io.will_fire(w), tag_masks(w) | curr_mask, curr_mask)
      // 指令将发射，则将这个变量加上新分配的掩码，否则保持
  }

      
      
  //-------------------------------------------------------------
  // Update the current branch_mask （更新当前分支掩码）

  when (io.flush_pipeline) {
    branch_mask := 0.U // 清空流水线，初始化为0
  } .otherwise {
    val mask = Mux(io.brupdate.b2.mispredict,
      io.brupdate.b2.uop.br_mask,  // 发生误判，记录分支更新信息中的分支掩码
      ~(0.U(maxBrCount.W))) // 未误判，全部置1
    branch_mask := GetNewBrMask(io.brupdate, curr_mask) & mask
      // 未误判，置为最新的分支掩码。发生误判，若误判的掩码与当前的相同则保留，否则置0？
  }

  io.debug_branch_mask := branch_mask
}
```



## 补充

### rocket-chip的DecodeLogic对象

```scala
object DecodeLogic
{
    
  def term(lit: BitPat) =
    new Term(lit.value, BigInt(2).pow(lit.getWidth)-(lit.mask+1))
    
    
  def logic(addr: UInt, addrWidth: Int, cache: Map[Term,Bool], terms: Seq[Term]) = {
    terms.map { t =>
      cache.getOrElseUpdate(t, (if (t.mask == 0) addr else addr & Bits(BigInt(2).pow(addrWidth)-(t.mask+1), addrWidth)) === Bits(t.value, addrWidth))
    }.foldLeft(Bool(false))(_||_)
  }
    
    
	def apply(addr: UInt, default: BitPat, mapping: Iterable[(BitPat, BitPat)]): UInt = {
    val cache = caches.getOrElseUpdate(addr, Map[Term,Bool]())
    val dterm = term(default)
    val (keys, values) = mapping.unzip
    val addrWidth = keys.map(_.getWidth).max
    val terms = keys.toList.map(k => term(k))
    val termvalues = terms zip values.toList.map(term(_))

    for (t <- keys.zip(terms).tails; if !t.isEmpty)
      for (u <- t.tail)
        assert(!t.head._2.intersects(u._2), "DecodeLogic: keys " + t.head + " and " + u + " overlap")

    Cat((0 until default.getWidth.max(values.map(_.getWidth).max)).map({ case (i: Int) =>
      val mint = termvalues.filter { case (k,t) => ((t.mask >> i) & 1) == 0 && ((t.value >> i) & 1) == 1 }.map(_._1)
      val maxt = termvalues.filter { case (k,t) => ((t.mask >> i) & 1) == 0 && ((t.value >> i) & 1) == 0 }.map(_._1)
      val dc = termvalues.filter { case (k,t) => ((t.mask >> i) & 1) == 1 }.map(_._1)

      if (((dterm.mask >> i) & 1) != 0) {
        logic(addr, addrWidth, cache, SimplifyDC(mint, maxt, addrWidth))
      } else {
        val defbit = (dterm.value.toInt >> i) & 1
        val t = if (defbit == 0) mint else maxt
        val bit = logic(addr, addrWidth, cache, Simplify(t, dc, addrWidth))
        if (defbit == 0) bit else ~bit
      }
    }).reverse)
  }
    
    
  def apply(addr: UInt, default: Seq[BitPat], mappingIn: Iterable[(BitPat, Seq[BitPat])]): Seq[UInt] = {
    val mapping = ArrayBuffer.fill(default.size)(ArrayBuffer[(BitPat, BitPat)]())
    for ((key, values) <- mappingIn)
      for ((value, i) <- values zipWithIndex)
        mapping(i) += key -> value
    for ((thisDefault, thisMapping) <- default zip mapping)
      yield apply(addr, thisDefault, thisMapping)
  }
    
    
  def apply(addr: UInt, default: Seq[BitPat], mappingIn: List[(UInt, Seq[BitPat])]): Seq[UInt] =
    apply(addr, default, mappingIn.map(m => (BitPat(m._1), m._2)).asInstanceOf[Iterable[(BitPat, Seq[BitPat])]])
 
    
  def apply(addr: UInt, trues: Iterable[UInt], falses: Iterable[UInt]): Bool =
    apply(addr, BitPat.dontCare(1), trues.map(BitPat(_) -> BitPat("b1")) ++ falses.map(BitPat(_) -> BitPat("b0"))).asBool
    
    
  private val caches = Map[UInt,Map[Term,Bool]]()
}
```

### GetNewBrMask 方法

```scala
object GetNewBrMask
{
   def apply(brupdate: BrUpdateInfo, uop: MicroOp): UInt = {
     return uop.br_mask & ~brupdate.b1.resolve_mask
   }

   def apply(brupdate: BrUpdateInfo, br_mask: UInt): UInt = {
     return br_mask & ~brupdate.b1.resolve_mask
   }
}
```

