---
title: YPC的研究
date: 2024-09-20 18:20:40
tags:
    - YPC
    - 模拟器
categories:
    - PA2
    - 模拟器
---

<!--more-->

## 一、前言
本文尝试去分析 YSYX 中的一个实例 [YPC](https://ysyx.oscc.cc/slides/2306/06.html#/%E7%94%A8%E7%BB%84%E5%90%88%E9%80%BB%E8%BE%91%E7%94%B5%E8%B7%AF%E5%AE%9E%E7%8E%B0%E6%8C%87%E4%BB%A4%E7%9A%84%E8%AF%AD%E4%B9%89),YPC 使用 Chisel 来实现基本的指令执行过程：定义指令结构、取指、解码（操作码、操作数）、执行、更新 PC。

## 二、写被执行的程序

这个程序主要使用内嵌汇编的方法来执行一些汇编代码，我们可以确认它仅仅包含这些指令，应为 `YPC` 被设计为只能执行 `I类指令`的两种指令，`addi` 和 `ebreak`。

```C
static void ebreak(long arg0, long arg1) {
  asm volatile("addi a0, x0, %0;"
               "addi a1, x0, %1;"
               "ebreak" : : "i"(arg0), "i"(arg1));
}
static void putch(char ch) { ebreak(0, ch); }
static void halt(int code) { ebreak(1, code); while (1); }

void _start() {
  putch('H'); putch('e'); putch('l'); putch('l'); putch('o'); putch(','); putch(' ');
  putch('R'); putch('I'); putch('S'); putch('C'); putch('-'); putch('V'); putch('!');
  putch('\n');
  halt(0);
}
```

这个程序很简单，不管是 `putch` 还是 `halt`，它都会自己调用函数 `ebreak()`，其对应着 3 条指令：

```asm
addi a0, x0, %0
addi a1, x0, %1
ebreak : : "i"(arg0), "i"(arg1) ; arg0 传给寄存器 a0(10号寄存器)，arg1 传给寄存器a1(11号寄存器)
```
这里需要澄清的是，这是一个程序，它会在riscv 上自己执行， 它不会做额外的事情。

我们可以把这个程序编译一下：

```bash
rv32gcc -ffreestanding -nostdlib -static -Wl,-Ttext=0 -O2 -o prog a.c
rvobjdump -M no-aliases -d prog # 查看反汇编

prog:     file format elf32-littleriscv


Disassembly of section .text:

00000000 <_start>:
   0:   00000513                addi    a0,zero,0
   4:   04800593                addi    a1,zero,72
   8:   00100073                ebreak
   c:   00000513                addi    a0,zero,0
  10:   06500593                addi    a1,zero,101
  14:   00100073                ebreak
  18:   00000513                addi    a0,zero,0
  1c:   06c00593                addi    a1,zero,108
  20:   00100073                ebreak
  24:   00000513                addi    a0,zero,0
  28:   06c00593                addi    a1,zero,108
  2c:   00100073                ebreak
  30:   00000513                addi    a0,zero,0
  34:   06f00593                addi    a1,zero,111
  38:   00100073                ebreak
  3c:   00000513                addi    a0,zero,0
  40:   02c00593                addi    a1,zero,44
  44:   00100073                ebreak
  48:   00000513                addi    a0,zero,0
  4c:   02000593                addi    a1,zero,32
  50:   00100073                ebreak
  54:   00000513                addi    a0,zero,0
  58:   05200593                addi    a1,zero,82
  5c:   00100073                ebreak
  60:   00000513                addi    a0,zero,0
  64:   04900593                addi    a1,zero,73
  68:   00100073                ebreak
  6c:   00000513                addi    a0,zero,0
  70:   05300593                addi    a1,zero,83
  74:   00100073                ebreak
  78:   00000513                addi    a0,zero,0
  7c:   04300593                addi    a1,zero,67
  80:   00100073                ebreak
  84:   00000513                addi    a0,zero,0
  88:   02d00593                addi    a1,zero,45
  8c:   00100073                ebreak
  90:   00000513                addi    a0,zero,0
  94:   05600593                addi    a1,zero,86
  98:   00100073                ebreak
  9c:   00000513                addi    a0,zero,0
  a0:   02100593                addi    a1,zero,33
  a4:   00100073                ebreak
  a8:   00000513                addi    a0,zero,0
  ac:   00a00593                addi    a1,zero,10
  b0:   00100073                ebreak
  b4:   00100513                addi    a0,zero,1
  b8:   00000593                addi    a1,zero,0
  bc:   00100073                ebreak
  c0:   0000006f                jal     zero,c0 <_start+0xc0>


```

可以看到这个程序真的只含有两条指令（最后一条 jal 指令是跳转指令，目的是制造死循环，不用管）。


但是我们可以手动写一个解释器，它可以用来`执行` riscv 代码，甚至我们可以根据 ebreak 的参数来确定接下来的行为。比如，当调用函数 `ebreak( 0,ch)`的时候，这时候 a0 就是 0，a11 就是 ch，我们可以定义它为输出字符；当调用函数 `ebreak( 1,code)`的时候，这时候 a0 就是 1，a11 就是 code，我们甚至可以根据不同的 code 来触发不同的行为。

由于我只需要二进制程序 prog 中的汇编指令，因此我们可以利用命令将一个可执行二进制文件中的汇编指令抽离出来：

```bash
riscv64-linux-gnu-objcopy -j .text -O binary prog prog.bin
```

这样里面就仅仅包含了我们想要的二进制指令，然后就可以放在解释器上执行它了。


在 PYC 中，我们定义了两种有限的行为：
- 打印 ch
- 停机

## YPC

```Scala
import chisel3._
import chisel3.util._
import chisel3.util.experimental.loadMemoryFromFileInline

class YPC extends Module {
  val io = IO(new Bundle { val halt = Output(Bool()) })
  val R = Mem(32, UInt(32.W))
  val PC = RegInit(0.U(32.W))
  val M = Mem(1024 / 4, UInt(32.W))
  def Rread(idx: UInt) = Mux(idx === 0.U, 0.U(32.W), R(idx))

  val Ibundle = new Bundle {
    val imm11_0 = UInt(12.W)
    val rs1 = UInt(5.W)
    val funct3 = UInt(3.W)
    val rd = UInt(5.W)
    val opcode = UInt(7.W)
  }
  def SignEXT(imm11_0: UInt) = Cat(Fill(20, imm11_0(11)), imm11_0)

  val inst = M(PC(31, 2)).asTypeOf(Ibundle)
  val isAddi = (inst.opcode === "b0010011".U) && (inst.funct3 === "b000".U)
  val isEbreak = inst.asUInt === "x00100073".U
  assert(isAddi || isEbreak, "Invalid instruction 0x%x", inst.asUInt)

  val rs1Val = Rread(Mux(isEbreak, 10.U(5.W), inst.rs1))
  val rs2Val = Rread(Mux(isEbreak, 11.U(5.W), 0.U(5.W)))
  when(isAddi) { R(inst.rd) := rs1Val + SignEXT(inst.imm11_0) }
  when(isEbreak && (rs1Val === 0.U)) { printf("%c", rs2Val(7, 0)) }
  io.halt := isEbreak && (rs1Val === 1.U)
  PC := PC + 4.U
}
object YPC extends App {
  (new chisel3.stage.ChiselStage).emitVerilog(new YPC)
}

```

这个程序定一个了 YPC 模块，它有 4 个组件，分别是：
- io.halt: 这是一个输出，它会输出是否停机的信号；
- R: 这是寄存器，一共 32 个，每一个 32 位；
- M: 内存单元，每个单元都是 32 位，换句话说，这些单元的下标对应着的地址是 4 字节对齐的；
- PC: 初始化为 0，标记着指令在 M 中的地址；

- Ibundle: 这是一个 I 型指令的bundle；
- inst: 指令；
- isAddi、isEbreak: 指令判断，bool 类型；


接下来的操作是YPC 的核心：
- 如果是 ebreak 类型的指令，分别读取 a0、a1，根据 a0、a1 来赋予指令语义，比如是输出字符还是终止；
- 如果是 addi 类型的指令，分别读取 源寄存器、零寄存器，它们会被当做两个源寄存器，以便后面进行加法操作；

- 当是 ebreak 指令的时候， R(inst.rd) := rs1Val + SignEXT(inst.imm11_0)；
- 当是 ebreak && a0 是 0，就输出字符 a1中的值，也就是传给的 ch；
- 当是 ebreak && a0 是 1，就标记输出 io.halt 为 1。这个输出在运行时环境中会被用来判断是否停机。
- 最后更新 pc。



## Chisel 到 Verilog 再到C++仿真

写好了 Chsel，想要利用 C++ 进行仿真，还得将它翻译成 Verilog，然后借助 verilator 将 HDL 翻译成 C++ 代码，然后我们可以写一个运行时环境，给它装载程序、控制停机、执行等：

```cpp
#include <stdio.h>
#include "VYPC.h"
#include "VYPC___024root.h"
static VYPC* top = NULL;
void step() {
    top->clock = 0;
    top->eval();
    top->clock = 1;
    top->eval();
}
void reset(int n) {
    top->reset = 1;
    while (n--) {
        step();
    }
    top->reset = 0;
}
void load_prog(const char* bin) {
    FILE* fp = fopen(bin, "r");
    fread(&top->rootp->YPC__DOT__M, 1, 1024, fp);
    fclose(fp);
}
int main(int argc, char* argv[]) {
    top = new VYPC;
    load_prog(argv[1]);
    reset(20);
    while (!top->io_halt) {
        step();
    }
    return 0;
}

```

假设已经生成了 HDL ，也就是 YPC.v，现在我们将它翻译成 C++。为了方便操作，可以写一个简单的 Makefile 来简化操作：

```Makefile
VERILATOR = verilator
SRC = main.cpp YPC.v
OBJ_SRC = hello.c
OBJ_BIN = prog.bin
OBJ_TARGET = prog
OBJ_DIR = obj_dir
RV32GCC = riscv64-linux-gnu-gcc -march=rv32g -mabi=ilp32
CFLAGS = -ffreestanding -nostdlib -static -Wl,-Ttext=0 -O2

RV64OBJCOPY = riscv64-linux-gnu-objcopy
RV64CP_FLAGS = -j .text -O binary

EXE = $(OBJ_DIR)/VYPC

build: $(EXE)

$(EXE): $(SRC)
	bear -- $(VERILATOR) --cc --trace --exe --build $^
	make -C $(OBJ_DIR) -f VYPC.mk

run: $(EXE) $(OBJ_BIN)
	@$(EXE) $(OBJ_BIN)

$(OBJ_TARGET): $(OBJ_SRC)
	$(RV32GCC) $(CFLAGS) -o $@ $^

$(OBJ_BIN): $(OBJ_TARGET)
	$(RV64OBJCOPY) $(RV64CP_FLAGS) $^ $@
clean:
	@rm -rf $(OBJ_DIR) prog*

.PHONY: build run clean

```

然后直接make run：

```bash
$ make run
Hello, RISC-V!

```

看来，YPC 已经达到了预期效果。

事实上，这个 YPC 可以很强大，只要实现所有的命令，它就可以运行所有的 riscv 程序！

## 思考

第一个问题：注意到 `YPC.scala` 中的一些调用，比如 `assert、printf`，这其实是 `Scala` 的函数调用，当然它依然会被 `sbt` 转化为 `HDL`:

```Verilog
$fwrite(32'h80000002,
            "Assertion failed: Invalid instruction 0x%x\n    at YPC.scala:25 assert(isAddi || isEbreak, \"Invalid instruction 0x%%x\", inst.asUInt)\n"
            ,_isEbreak_T); // @[YPC.scala 25:9]
$fwrite(32'h80000002,"%c",rs2Val[7:0]); // @[YPC.scala 30:46]
```

可以看到，这并不是真正的 HDL，他依然是函数调用，需要借助于仿真环境，它这次就会被 verilator 转化为 C++ 的系统调用（in verilated.h）：

```cpp
VL_FWRITEF(0x80000002U,"%c",8,((0U == ((0x100073U
                                                == vlSelf->YPC__DOT__M_inst_MPORT_data)
                                                ? 0xbU
                                                : 0U))
                                        ? 0U : (0xffU
                                                & vlSelf->YPC__DOT__R
                                                [((0x100073U
                                                   == vlSelf->YPC__DOT__M_inst_MPORT_data)
                                                   ? 0xbU
                                                   : 0U)])));

VL_FWRITEF(0x80000002U,"Assertion failed: Invalid instruction 0x%x\n    at YPC.scala:25 assert(isAddi || isEbreak, \"Invalid instruction 0x%%x\", inst.asUInt)\n",
                   32,vlSelf->YPC__DOT__M_inst_MPORT_data);
```

第二个问题：程序中有 jal 指令，按理说会打印出 `Assertion failed: Invalid instruction...`，因为我并没有实现 jal 指令，但是并没有注意到这个信息，为什么？

答案很简答，因为这个调用：
```c
static void halt(int code) { ebreak(1, code); while (1); }
```
首先是 halt 了，然后才死循环调用 jal 了，事实上当：

```Scala
  io.halt := isEbreak && (rs1Val === 1.U)
  PC := PC + 4.U
```
的时候，运行时环境已经结束了二进制程序的继续推进：

```C++
int main(int argc, char* argv[]) {
    top = new VYPC;
    load_prog(argv[1]);
    reset(20);
    while (!top->io_halt) {
        step();
    }
    return 0;
}
```
































