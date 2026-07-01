---
title: "RISC-VI 研究笔记（二）：LLVM 后端里为什么先做 +xvi32r"
date: 2026-04-14 15:03:13
tags:
  - "RISC-VI"
  - "LLVM"
  - "backend"
  - "RISC-V"
categories:
  - "Computer Architecture"
---

## 前言

如果要让 LLVM 支持一种“新架构”，直觉上很容易想到：那就新建一个 target，比如 `riscvi32r`。但 RISC-VI 的第一阶段没有这么做，而是选择继续使用 RISC-V target，并通过 `+xvi32r` 打开实验指令。

这不是绕路，而是一个非常典型的后端工程取舍：先把变量收窄，把目标放在指令价值验证上，而不是过早承担新 triple、新 ABI、新 driver、新 object 生态的维护成本。

![LLVM RISC-V 后端中的 +xvi32r 扩展分层](/images/mermaid-svg/riscvi-02-llvm-extension-xvi32r/extension-stack.svg)

## LLVM 后端里的三层概念

理解这个选择前，需要先区分三个概念。

`target` 是 LLVM 后端的大类，例如 `RISCV`、`X86`、`AArch64`。一个 target 里面有寄存器文件、指令格式、调用约定、栈帧处理、MC 编码、汇编打印、指令选择等基础设施。

`triple` 是编译目标描述，例如 `riscv32`、`riscv64`、`x86_64-unknown-linux-gnu`。它会影响 ABI、数据布局、平台约定和 driver 行为。

`subtarget feature` 是 target 内部的能力开关。RISC-V 后端天然适合这种模型，因为 RISC-V 本身就是 base ISA 加扩展的组合。`+m`、`+a`、`+c` 是扩展，vendor extension 也可以被建模成 feature。

RISC-VI 当前就放在第三层：它不是另起炉灶，而是在 RISC-V target 里增加一个可开关的实验扩展。

## 源码里的 feature 定义

在 LLVM 源码中，RISC-VI feature 定义位于：

```text
llvm/lib/Target/RISCV/RISCVFeatures.td
```

当前项目中对应的是 `llvm/lib/Target/RISCV/RISCVFeatures.td:1207` 开始的 RISC-VI vendor extension 段。

核心定义是：

```tablegen
def FeatureVendorXVI32R
    : RISCVExtension<0, 1, "RISC-VI32R LLVM-friendly user-mode integer operations",
                     [], "HasVendorXRiscVI">;
def HasVendorXRiscVI : Predicate<"Subtarget->hasVendorXRiscVI()">,
                       AssemblerPredicate<(all_of FeatureVendorXVI32R),
                           "'XVI32R' (RISC-VI32R LLVM-friendly integer operations)">;
```

这段代码同时服务两件事：

1. 后端 C++ 可以通过 `Subtarget->hasVendorXRiscVI()` 判断这个 feature 是否开启。
2. 汇编器可以通过 `AssemblerPredicate` 拒绝未开启 feature 时出现的 RISC-VI 指令。

也就是说，`+xvi32r` 不是一个字符串约定，而是进入了 LLVM 的 subtarget 和 MC predicate 体系。

## 指令文件如何接入 RISC-V 后端

RISC-V 后端的主指令描述文件是：

```text
llvm/lib/Target/RISCV/RISCVInstrInfo.td
```

RISC-VI 的 include 落点在 `llvm/lib/Target/RISCV/RISCVInstrInfo.td:2425`。这个位置和其他 vendor extension 并列，而不是侵入主干 RV32I/RV64I 指令描述。

RISC-VI 通过 include 接入 vendor extension 区域：

```tablegen
include "RISCVInstrInfoXVI.td"
```

这意味着 RISC-VI 指令没有散落在主文件里，而是被集中放在 `RISCVInstrInfoXVI.td`。这种组织方式很重要：实验扩展可以独立迁移、审查和回滚，也方便后续比较 v0.1/v0.2 编码设计。

## 为什么不直接新建 `riscvi32r` target

新建 target 看起来干净，但它会立刻引出一整套问题：

- clang driver 是否认识新 triple；
- ABI 是否完全复用 RV32I；
- calling convention 是否要复制一份；
- frame lowering、register info、instruction info 如何维护；
- MC 层 relocation、object、disassembler 是否要分叉；
- libc、linker script、runtime 如何命名；
- 测试体系如何避免和 RISC-V 后端重复。

RISC-VI v0.1 真正想验证的是另一件事：

```text
少量面向 LLVM 常见代码模式和双发射流水线的整数指令，
能不能减少短距离依赖、动态提交数和局部控制流代价。
```

在这个阶段，复用 RISC-V 后端能让研究问题更清楚。变量越少，实验结论越容易归因。

## feature 不是护身符，predicate 才是边界

定义 feature 只是第一步。真正防止“未开启扩展也生成新指令”的，是每条指令和 pattern 上的 predicate。

在 `RISCVInstrInfoXVI.td` 中，真实指令被包在：

```tablegen
let Predicates = [HasVendorXRiscVI] in {
  def XVI_LWXS : ...
  def XVI_MIN  : ...
  def XVI_CSEL : ...
}
```

指令选择 pattern 进一步限定在 RV32：

```tablegen
let Predicates = [HasVendorXRiscVI, IsRV32] in {
  def : PatGprGpr<smin, XVI_MIN, i32>;
}
```

这就是工程边界：打开 `+xvi32r` 时，LLVM 可以选择 RISC-VI 指令；没有打开时，后端和汇编器都应该把它们挡住。

## 这种路线的边界

把 RISC-VI 做成 vendor extension，不代表永远不做独立 triple。它只是说明当前阶段还不该过早扩大系统边界。

后续如果满足这些条件，就可以重新评估 `riscvi32r` triple：

1. 指令语义稳定。
2. 编码版本稳定。
3. LLVM、MC、模拟器、AM、RTL 都对同一套编码闭环。
4. ABI 和用户态边界明确。
5. 更大 workload 上有足够实验数据。

在那之前，`+xvi32r` 是一个务实路线：让项目专注在 ISA 实验本身。

## 项目里的验证入口

项目提供了 feature、TableGen 和 codegen 的检查入口，例如 source tree report、tblgen report、real codegen check。它们的意义不是让读者记命令，而是表达一个工程原则：feature 定义、TableGen 生成物、正向 codegen 和负向 predicate 检查应当同时存在。只要缺一层，`+xvi32r` 就可能从“受控扩展”变成“到处泄漏的实验代码”。

## 小结

`+xvi32r` 的核心价值是小步快跑：复用 RISC-V 后端已经成熟的基础设施，把研究焦点放在指令是否真的覆盖 LLVM 常见模式、是否真的改善双发射压力。下一篇继续深入 TableGen，看一条机器指令在 LLVM 里到底由哪些信息构成。
