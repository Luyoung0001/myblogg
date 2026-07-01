---
title: "RISC-VI 研究笔记（三）：用 TableGen 定义一条真实机器指令"
date: 2026-04-15 15:03:13
tags:
  - "RISC-VI"
  - "LLVM"
  - "TableGen"
  - "backend"
categories:
  - "Computer Architecture"
---

## 前言

LLVM 后端里，一条机器指令不是一个汇编字符串。它至少要描述这些信息：

- 指令采用哪种编码格式；
- opcode、funct3、funct7 等位域如何落到 32 位机器码；
- 有哪些输入和输出操作数；
- 是否读写内存；
- 是否是分支或基本块 terminator；
- 是否有副作用；
- 对调度模型来说读写哪些资源；
- 哪些 IR/DAG pattern 可以选择到它。

这些信息大多写在 TableGen 里。RISC-VI 的集中入口是：

```text
llvm/lib/Target/RISCV/RISCVInstrInfoXVI.td
```

这份文件的结构很清楚：指令模板从 `llvm/lib/Target/RISCV/RISCVInstrInfoXVI.td:18` 开始，真实指令定义从 `:73` 开始，SelectionDAG pattern 从 `:93` 开始。

![TableGen 机器指令的组成](/images/mermaid-svg/riscvi-03-tablegen-instruction/tablegen-anatomy.svg)

## TableGen 不是“更高级的宏”

刚接触 LLVM 后端时，很容易把 TableGen 理解成一种生成 C++ 的模板语言。这个理解只对了一半。

TableGen 更像是 LLVM 后端的机器描述数据库。它生成的不只是枚举和打印表，还会参与：

1. 指令编码和反汇编。
2. asm parser 和 asm printer。
3. SelectionDAG pattern 匹配。
4. MachineInstr 属性判断。
5. 调度模型和资源读写。
6. 分支分析、内存依赖等后端 pass 的保守假设。

所以一条指令的 `mayLoad`、`mayStore`、`isBranch` 不是装饰性字段。写错它们，后端可能会做出错误优化。

## 从 `min` 看 ALU 指令模板

`min` 是最适合入门的一条 RISC-VI 指令。它是双源一目的整数 ALU 指令：

```text
min rd, rs1, rs2
```

语义是：

```c
rd = (int32_t)rs1 < (int32_t)rs2 ? rs1 : rs2;
```

在 TableGen 中，RISC-VI 抽出了一个通用 ALU 模板：

```tablegen
let hasSideEffects = 0, mayLoad = 0, mayStore = 0 in
class XVI_ALU_rr<bits<7> funct7, bits<3> funct3, string opcodestr,
                 bit Commutable = 0>
    : RVInstR<funct7, funct3, OPC_CUSTOM_0, (outs GPR:$rd),
              (ins GPR:$rs1, GPR:$rs2), opcodestr, "$rd, $rs1, $rs2"> {
  let isCommutable = Commutable;
}
```

这段模板同时说明了三件事：

1. 它复用 RISC-V 的 R-type 指令格式。
2. 它使用 `OPC_CUSTOM_0` 作为实验编码空间。
3. 它明确声明没有内存访问、没有副作用，因此可以被普通 ALU 优化逻辑处理。

真实指令定义则很短：

```tablegen
def XVI_MIN  : XVI_ALU_rr<0b0000000, 0b010, "min", Commutable=1>,
               Sched<[WriteIMinMax, ReadIMinMax, ReadIMinMax]>;
```

`Commutable=1` 表示两个源操作数可以交换。对 `min/max` 来说这成立，对减法或比较分支就不一定成立。

## load/store 指令为什么要单独建模板

`lwxs/swxs` 看起来也只是多了一个 scale 字段，但它们的后端含义和 ALU 完全不同。

`lwxs` 是 indexed load：

```text
lwxs rd, base, index, scale
```

语义近似为：

```c
rd = *(uint32_t *)(base + (index << scale));
```

所以它的模板必须声明：

```tablegen
let hasSideEffects = 0, mayLoad = 1, mayStore = 0 in
class XVI_LoadIndexed ...
```

`swxs` 则必须声明 `mayStore = 1`。这些字段会影响 LLVM 对内存操作的排序、依赖分析和 MachineMemOperand 保留。如果把真实 load/store 伪装成普通 ALU，后端可能会错误移动它，语义就不再可靠。

RISC-VI 的 indexed memory pattern 还特意匹配真实 load/store 节点，而不是先把地址算术变成普通 ALU 再替换。文件里的注释点出了原因：这样可以保留原始 `MachineMemOperand`，让后续内存相关 pass 仍然知道这是一条内存指令。

## `csel` 和三源操作数

`csel` 的语义是：

```text
csel rd, true_value, false_value, cond
```

当 `cond != 0` 时选择 `true_value`，否则选择 `false_value`。它的模板是三源一目的：

```tablegen
class XVI_CSel<bits<3> funct3, string opcodestr>
    : RVInstRBase<funct3, OPC_CUSTOM_0, (outs GPR:$rd),
                  (ins GPR:$rs1, GPR:$rs2, GPR:$rs3),
                  opcodestr, "$rd, $rs1, $rs2, $rs3"> {
  bits<5> rs3;
  let Inst{31} = 0;
  let Inst{30} = 0;
  let Inst{29-25} = rs3;
}
```

这段代码很适合说明 ISA 设计和硬件实现之间的张力。`csel` 可以减少小分支，但它需要第三个源寄存器。对双发射流水线来说，这可能带来读端口压力。所以后面的 RTL 报告必须统计 `three_source_count` 和 `read_port_pressure_count`。

## `bchkltu` 是分支，不是普通比较

`bchkltu` 的目标是边界检查失败路径：

```text
bchkltu index, length, fail
```

它的语义不是“如果小于就跳”，而是：

```c
if ((uint32_t)index >= (uint32_t)length) goto fail;
```

因此 TableGen 里必须把它标成分支和 terminator：

```tablegen
let isBranch = 1;
let isTerminator = 1;
let mayLoad = 0;
let mayStore = 0;
```

这会影响基本块边界、分支反转、分支范围判断和控制流分析。后续 RISC-V 后端的 C++ 代码还要显式认识 `XVI_BCHKLTU`，这就是第五篇会讲的跨层同步问题。

## pattern：让 LLVM 主动生成指令

只定义机器指令还不够。LLVM 需要知道“什么语义可以降成这条指令”。

`min/max` 的 pattern 很直接：

```tablegen
def : PatGprGpr<smin, XVI_MIN, i32>;
def : PatGprGpr<smax, XVI_MAX, i32>;
def : PatGprGpr<umin, XVI_MINU, i32>;
def : PatGprGpr<umax, XVI_MAXU, i32>;
```

这说明 RISC-VI 不是在汇编文本层做替换，而是在 SelectionDAG 语义层匹配 `smin/smax/umin/umax`。

`csel` 对应 `riscv_selectcc`，`bchkltu` 对应 `riscv_brcc` 的 `SETUGE`。这些 pattern 的共同点是：它们必须依赖 LLVM 中已经相对稳定的中间语义。指令设计如果找不到稳定 IR/DAG 模式，编译器就很难自然生成。

## 项目里的验证入口

项目中 TableGen 相关检查集中在 codegen case、real codegen check 和 MC smoke。它们对应三类风险：pattern 是否能生成指令，真实 LLVM 构建产物是否包含改动，MC 层是否能编码这些指令。读者不需要执行它们，也能从这个分层看出后端开发的基本纪律。

## 小结

TableGen 是 LLVM 后端工程的中心层之一。它不是简单的“指令表”，而是编码、操作数、属性、调度和选择规则的组合描述。下一篇进入指令选择，看 LLVM 如何从 IR/DAG 语义走到 `lwxs`、`csel` 和 `bchkltu`。
