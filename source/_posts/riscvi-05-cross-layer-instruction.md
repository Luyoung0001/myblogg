---
title: "RISC-VI 研究笔记（五）：新增一条指令要改多少层"
date: 2026-04-17 15:03:13
tags:
  - "RISC-VI"
  - "LLVM"
  - "simulator"
  - "RTL"
categories:
  - "Computer Architecture"
---

## 前言

很多人第一次做 ISA 扩展，会以为“加一条指令”就是在模拟器里写一个 `case`，或者在 LLVM 里写一个 TableGen `def`。真正的工程现实要复杂得多。

一条指令要成为系统里稳定存在的能力，至少要被这些层同时认识：

- LLVM feature 和 predicate；
- TableGen 指令编码、操作数和属性；
- 指令选择 pattern 或 C++ selector；
- MC 汇编器和编码器；
- C 模拟器 decode/execute；
- AM/BSP 或低层 wrapper；
- cpu-test/AM workload；
- 动态统计和报告；
- RTL decoder、execute、hazard、counter。

![新增指令的跨层修改矩阵](/images/mermaid-svg/riscvi-05-cross-layer-instruction/cross-layer-matrix.svg)

这篇用 `min` 和 `bchkltu` 两条指令说明：为什么 ISA 扩展是跨层协议，而不是单点补丁。

## `min`：最干净的跨层样板

`min` 没有内存访问，不是分支，也没有副作用。它的语义很简单：

```c
rd = (int32_t)rs1 < (int32_t)rs2 ? rs1 : rs2;
```

即便如此，它也要经过多个层次。

LLVM TableGen 层定义编码和属性：

```tablegen
def XVI_MIN  : XVI_ALU_rr<0b0000000, 0b010, "min", Commutable=1>,
               Sched<[WriteIMinMax, ReadIMinMax, ReadIMinMax]>;
```

指令选择层定义语义匹配：

```tablegen
def : PatGprGpr<smin, XVI_MIN, i32>;
```

模拟器 decode 层通过 `funct3` 识别：

```c
case 2:
  inst->op = OP_XVI_MIN;
  break;
```

这对应 `riscv-vi-research/sim/common/decode.c:59` 开始的 v0.1 decode。`funct3=2` 被映射成 `OP_XVI_MIN`，而 `funct3=7` 会额外解出 B-type immediate 和目标地址。

模拟器 execute 层实现提交语义：

```c
case OP_XVI_MIN:
  write_rd(cpu, inst, s32(rs1) < s32(rs2) ? rs1 : rs2);
  cpu->pc = inst->snpc;
  break;
```

执行语义对应 `riscv-vi-research/sim/common/exec.c:197` 开始的 RISC-VI case 段。`min/max/csel/bchkltu` 在这里都被写成提交级架构状态更新。

这四段代码分别回答四个问题：

1. LLVM 如何描述这条机器指令？
2. LLVM 什么时候选择它？
3. 模拟器如何从 raw instruction 认出它？
4. 模拟器如何修改架构状态？

任何一层漏掉，都会出现“能编译不能跑”或“能跑但编译器不发”的断裂。

## `bchkltu`：分支指令要改更多地方

`bchkltu` 比 `min` 更能体现跨层复杂度。它不是普通 ALU，而是 terminator：

```text
bchkltu index, length, fail
```

TableGen 层要声明它是分支：

```tablegen
let isBranch = 1;
let isTerminator = 1;
```

pattern 层要匹配失败路径：

```tablegen
def : Pat<(riscv_brcc (i32 GPR:$idx), GPR:$len, SETUGE, bb:$target),
          (XVI_BCHKLTU GPR:$idx, GPR:$len, bare_simm13_lsb0_bb:$target)>;
```

RISC-V 后端 C++ 还要把它纳入分支条件分析：

```cpp
case RISCV::XVI_BCHKLTU:
  return RISCVCC::COND_GEU;
```

分支 offset 范围也要被认识：

```cpp
case RISCV::XVI_BCHKLTU:
  return isInt<13>(BrOffset);
```

模拟器 execute 层再实现实际跳转：

```c
case OP_XVI_BCHKLTU:
  taken = rs1 >= rs2;
  execute_branch(cpu, inst, taken);
  break;
```

这就是分支类 ISA 扩展的典型特点：它不只是一条机器码，还会进入编译器的控制流理解、基本块布局、分支反转、范围扩展和 RTL 前端统计。

## MC 层的意义

很多实验项目会忽略 MC 层，直接用 `.word` 写编码。`.word` 很适合早期验证，但它绕过了 LLVM 对汇编语法和编码的检查。

真正进入编译器后端后，MC 层至少要保证：

1. 汇编器知道 `min rd, rs1, rs2` 这种语法。
2. 未开启 `+xvi32r` 时拒绝 RISC-VI 指令。
3. 编码位域和模拟器 decode 位域一致。
4. 反汇编或 objdump 时能把机器码还原成可读指令。

如果 MC 编码和模拟器 decode 不一致，最麻烦的情况是“编译器、汇编器都认为成功，模拟器却执行成另一条语义”。所以新增指令时，MC 层不是可有可无的漂亮包装，而是跨层协议的一部分。

## AM wrapper 的定位

在 LLVM 发射完全稳定前，AM 里可以保留 `.word` wrapper。它的作用是绕过编译器，直接构造指令编码，用来验证模拟器和 RTL。

这是一种很实用的分层调试手段：

- wrapper 通过，说明编码、模拟器和 RTL 大概率一致；
- LLVM codegen 不通过，问题更可能在 feature、pattern、selector 或 MC；
- LLVM 能发但 wrapper/RTL 不一致，说明编码协议可能分叉。

但 wrapper 不能替代后端工作。最终目标仍然是让 C 代码经过 LLVM 自然生成 RISC-VI 指令。

## 动态统计也属于协议

新增指令后，还要让统计和报告认识它。比如 `riscvi_count` 表示动态提交的 RISC-VI 指令数量；`load_count/store_count/branch_count` 要把 `lwxs/swxs/bchkltu` 纳入正确类别；`three_source_count` 要观察 `csel` 这类多源指令。

这一步看起来不像“功能实现”，但对研究项目非常关键。没有统计字段，后面就无法解释收益到底来自动态指令减少、RAW 缩短、分支减少，还是只是 workload 偶然变化。

## 项目里的验证入口

项目里把跨层检查拆成几类入口：LLVM codegen、MC smoke、sim test 和 AM test。它们对应不同层次：LLVM 选择、MC 编码、模拟器语义、裸机 runtime。端到端测试很重要，但在开发过程中，分层定位更重要。

## 小结

新增一条指令，本质上是在多个系统之间建立协议：LLVM 要知道它什么时候可用、MC 要知道它怎么编码、模拟器要知道它怎么执行、RTL 要知道它怎么进流水线、报告要知道它怎么计数。下一篇回到设计源头，看 RISC-VI 为什么选择这 8 条指令，而不是随手添加一组看起来酷的操作。
