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

很多人第一次做 ISA 扩展，会以为“加一条指令”就是在模拟器里写一个 case，或者在 LLVM 里写一个 TableGen def。实际上，只改一层通常都不够。

一条指令要进入真实工程闭环，至少要被这些层认识：

- LLVM 后端；
- MC 汇编和编码；
- C 模拟器；
- AM/BSP wrapper 或编译器发射路径；
- cpu-test 或 AM 测试；
- 动态统计和报告；
- RTL decoder 和执行逻辑。

![新增指令的跨层修改矩阵](/images/mermaid-svg/riscvi-05-cross-layer-instruction/cross-layer-matrix.svg)

这张图要表达三件事：

1. 指令不是一个点修改，而是跨层协议。
2. 某一层不同步，就会出现“能编译但不能跑”。
3. 最稳的开发方式是按层做单点验证，再做端到端验证。

## 以 `min` 为例

`min` 是最干净的样板。它没有内存访问，不是分支，语义也简单：

```c
rd = (int32_t)rs1 < (int32_t)rs2 ? rs1 : rs2;
```

但即便如此，它也需要跨层落地。

## LLVM 层

在 `RISCVInstrInfoXVI.td` 中定义：

```tablegen
def XVI_MIN  : XVI_ALU_rr<0b0000000, 0b010, "min", Commutable=1>,
               Sched<[WriteIMinMax, ReadIMinMax, ReadIMinMax]>;
```

再用 pattern 让 `smin` 选择到它：

```tablegen
def : PatGprGpr<smin, XVI_MIN, i32>;
```

验证：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make llvm-codegen-cases
```

## 模拟器层

模拟器需要三件事。

第一，op enum 里要有 `OP_XVI_MIN`。

第二，decode 要能从 raw instruction 识别出它。v0.1 中 RISC-VI 指令都在 `custom-0`，通过 `funct3` 区分。

第三，execute 要实现语义：

```c
case OP_XVI_MIN:
  write_rd(cpu, inst, s32(rs1) < s32(rs2) ? rs1 : rs2);
  cpu->pc = inst->snpc;
  break;
```

验证：

```bash
make sim-test
```

## AM/BSP 层

在 LLVM 发射完全稳定之前，AM 里可以通过 `.word` wrapper 直接构造指令。编码宏类似：

```c
#define XVI_MIN(RD, RS1, RS2) \
  XVI_ENCODE_R(0, RS2, RS1, XVI_F3_MIN, RD, XVI_OPCODE_ALU)
```

这样 AM 应用可以先绕过编译器路径，直接验证模拟器和 RTL。

验证：

```bash
make -C am APP=riscvi_ops test
```

## cpu-test 层

最终还是要回到 C 程序：

```bash
make -C cpu-test asm CASE=min3 MODE=riscvi32r
make -C cpu-test run CASE=min3 MODE=riscvi32r
```

如果汇编里出现 `min`，并且模拟器运行返回值正确，说明 LLVM 到模拟器这条链路是通的。

## 常见同步失败

LLVM 能发，但模拟器报 `unimplemented op`：说明 decode 或 exec 没同步。

模拟器能跑 `.word`，但 LLVM 不发：说明 TableGen pattern 或 `+xvi32r` predicate 有问题。

RV32R 模式也出现新指令：说明 predicate 漏了。

动态统计里没有 `min`：说明 stats 或 report 没同步，或者测试程序没有真正执行到目标路径。

## 小结

新增指令最重要的工程习惯是“跨层对齐”。不要只看某个单点是否成功。真正的成功是：

```text
LLVM 能发 -> MC 能汇编 -> 模拟器能执行 -> AM/cpu-test 能验证 -> 报告能统计 -> RTL 能对齐
```

下一篇回到设计源头：这些指令为什么是这 8 条，而不是随便挑出来的。
