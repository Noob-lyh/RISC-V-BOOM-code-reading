# RV32I简介

施工中。对RISC-V官方文档《The RISC-V Instruction Set Manual, Volume I: Unprivileged ISA》中 “RV32I Base Integer Instruction Set,
Version 2.1”即RV32I指令集部分的翻译，以及和程序中对应的一些说明。

---

RV32I的设计足以形成编译器目标并支持现代操作系统系统环境。ISA的设计还可以将所需的硬件减少到最低限度。RV32I包含40条独特的指令，尽管有可能用一个简单的实现，使用一个总是trap且可能可以把FENCE指令当做NOP的SYSTEM硬件指令覆盖ECALL/EBREAK指令，使得把基础指令数减少到38。RV32I几乎可以模拟任何其他ISA扩展（除了A扩展，它需要对原子性的额外硬件支持）。

实际上，包括机器模式特权体系结构的硬件实现还需要6条CSR指令。

基本整数ISA的子集可能对教学目的有用，但基本整数ISA的定义应确保：除了忽略对未对齐内存访问的支持和将所有系统指令视为单个陷阱之外，不存在将实际硬件实现子集的动机。

---



### 基本整数ISA的程序员模型



### 基本指令格式

![InstructionType](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/InstructionType.jpg)

### 整数计算指令



### 控制跳转指令



### 加载和储存指令

RV32I是一种加载存储体系结构(load-store architecture)，即只有加载和存储指令访问内存，算术指令仅在CPU寄存器上运行。RV32I提供一个字节寻址(byte-addressed)的32位地址空间。EEI将定义地址空间的哪些部分可以被哪些指令合法访问（例如，某些地址可能是只读的，或者只支持字访问）。目标为x0的加载指令必须仍然引发任意的异常，并导致任何的副作用，即使该加载的值已被丢弃。

EEI将定义内存系统是小端(little-endian)还是大端(big-endian)。在RISC-V中，无端(endianness)的字节地址是不变的。

注：EEI即Execution Environment Interface，RISC-V执行环境接口。它定义了程序的初始状态、环境中hart的数量和类型，包括hart支持的特权模式、内存和I/O区域的可访问性和属性、在每个hart上执行的所有合法指令的行为（例如ISA就是EEI的一个组成部分），以及对执行期间引发的任何中断或异常（包括环境调用）的处理。

> 在无端的系统中，如果某个字节在某个地址存储到内存中，则从该地址加载字节大小时将返回存储的值。
>
> 在小端配置中，多字节存储器在最低内存字节地址写入最低有效寄存器字节，然后按其有效性的升序写入其他寄存器字节。加载将较小内存字节地址的内容传输到较低有效寄存器字节。
>
> 在大端配置中，多字节存储器在最低内存字节地址写入最高有效寄存器字节，然后按重要性降序写入其他寄存器字节。加载将较大内存字节地址的内容传输到较低有效寄存器字节。

![LoadStore](https://github.com/Noob-lyh/RISC-V-BOOM-code-reading/blob/main/pics/LoadStore.jpg)

加载和存储指令在寄存器和内存之间传输值。加载以I-type编码，存储为S-type。有效地址通过将寄存器rs1与符号扩展后的12位偏移量相加获得。加载时，将值从内存复制到寄存器rd。储存时，将寄存器rs2中的值复制到内存。

LW指令将32位值从内存加载到rd。LH从内存加载16位值，然后符号扩展到32位，然后再存储到rd。LHU从内存加载16位值，然后零扩展到32位，然后再存储到rd。LB和LBU则是对8位值的对应扩展存储操作。SW、SH和SB指令将寄存器rs2低位的32位、16位和8位值存储到内存中。

无论EEI如何，有效地址自然对齐的加载和存储都不会引发地址未对齐异常。有效地址与引用数据类型不自然对齐的加载和存储（即，对于32位访问，在四字节边界上，对于16位访问，在两字节边界上）时，实现的效果取决于EEI。

EEI可以保证完全支持未对齐的加载和存储，因此在执行环境中运行的软件永远不会遇到包含的(contained)或致命(fatal)的地址未对齐陷阱(trap)。在这种情况下，可以处理未对齐的加载和存储可以在硬件中，或者通过执行环境实现中的不可见陷阱，或者通过硬件和不可见陷阱的组合（取决于地址）解决。

EEI可能无法保证不对齐的负载和存储被不可见地处理。在这种情况下，未自然对齐的加载和存储可能会成功完成执行或引发异常。引发的异常可能是地址未对齐异常或访问错误异常。对于除未对齐外可以完成的内存访问（原句：For a memory access that would otherwise be able to complete except for the misalignment），如果未对齐的访问不应被访问，则会引发访问异常而不是地址未对齐异常，比如对内存区域的访问有副作用的时候。当EEI不能保证不可见地处理未对齐的加载和存储时，EEI必须定义由地址未对齐引起的异常是否会导致包含的陷阱（允许在执行环境中运行的软件处理该陷阱）或致命陷阱（终止执行）。

> Misaligned accesses are occasionally required when porting legacy code, and help performance on
> applications when using any form of packed-SIMD extension or handling externally packed data
> structures. Our rationale for allowing EEIs to choose to support misaligned accesses via the
> regular load and store instructions is to simplify the addition of misaligned hardware support.
> One option would have been to disallow misaligned accesses in the base ISA and then provide
> some separate ISA support for misaligned accesses, either special instructions to help software
> handle misaligned accesses or a new hardware addressing mode for misaligned accesses. Special
> instructions are difficult to use, complicate the ISA, and often add new processor state (e.g.,
> SPARC VIS align address offset register) or complicate access to existing processor state (e.g.,
> MIPS LWL/LWR partial register writes).
> In addition, for loop-oriented packed-SIMD code,
> the extra overhead when operands are misaligned motivates software to provide multiple forms
> of loop depending on operand alignment, which complicates code generation and adds to loop
> startup overhead. New misaligned hardware addressing modes take considerable space in the
> instruction encoding or require very simplified addressing modes (e.g., register indirect only).

即使未对齐的加载和存储成功完成，这些访问也可能运行得非常慢，具体取决于实现（例如，通过不可见陷阱实现时）。此外，虽然自然对齐的加载和存储保证以原子方式执行，但未对齐的加载和存储可能不会，因此需要额外的同步以确保原子性。

> We do not mandate atomicity for misaligned accesses so execution environment implementa-
> tions can use an invisible machine trap and a software handler to handle some or all misaligned
> accesses. If hardware misaligned support is provided, software can exploit this by simply using
> regular load and store instructions. Hardware can then automatically optimize accesses depend-
> ing on whether runtime addresses are aligned.



### Memory Ordering 指令



### Environment Call and Breakpoints



### HINT 指令

