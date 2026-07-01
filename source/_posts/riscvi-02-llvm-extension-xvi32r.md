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

如果要让 LLVM 支持一种“新架构”，直觉上很容易想到：那就新建一个 target，比如 `riscvi32r`。但在 RISC-VI 这个项目里，第一阶段我没有这么做，而是选择：

```bash
llc -mtriple=riscv32 -mattr=+xvi32r
```

这不是偷懒，而是一个很重要的工程取舍。

![LLVM RISC-V 后端中的 +xvi32r 扩展分层](/images/mermaid-svg/riscvi-02-llvm-extension-xvi32r/extension-stack.svg)

这张图要表达三件事：

1. RISC-VI 当前仍然复用 LLVM 的 RISC-V target。
2. `+xvi32r` 是 subtarget feature，用来控制实验指令是否可用。
3. 第一阶段目标是验证指令价值，不是立刻维护一整套新 ABI 和新工具链生态。

## target、triple 和 feature

LLVM 后端里有几个层次需要先分清。

`target` 是一个后端大类，比如 RISC-V、AArch64、X86。它包含寄存器定义、指令格式、调用约定、栈帧处理、汇编打印、MC 编码等一整套基础设施。

`triple` 是目标平台描述，比如：

```text
riscv32
riscv64
x86_64-unknown-linux-gnu
```

`subtarget feature` 是某个 target 内部可开关的扩展能力，比如 RISC-V 里的各种扩展。RISC-VI 当前就是通过这个机制接入：

```tablegen
def FeatureVendorXVI32R
    : SubtargetFeature<"xvi32r", "HasVendorXRiscVI", "true",
                       "'XVI32R' (RISC-VI32R LLVM-friendly integer operations)">;
```

后续指令定义就可以用：

```tablegen
let Predicates = [HasVendorXRiscVI] in {
  // RISC-VI instructions here.
}
```

也就是说，不打开 `+xvi32r`，这些指令就不应该被选择出来。

## 为什么不直接新建 `riscvi32r`

新建 target 看起来很干净，但代价很大。它意味着你要回答一连串问题：

- 新 triple 怎么被 clang driver 接受；
- 新 ABI 是否和 RV32I 完全一致；
- 寄存器、栈帧、调用约定是否复用；
- 汇编器、反汇编器、relocation、object 格式怎么处理；
- 测试树怎么组织；
- 后续和 RISC-V 上游代码如何同步。

而 RISC-VI 第一阶段真正想验证的是另一件事：

```text
少量面向 LLVM 常见模式和双发射流水线的指令，能否减少短距离依赖和动态指令数。
```

所以更稳妥的路线是：先借住 RISC-V 后端，把 RISC-VI 做成实验扩展。

## 代码入口

这个阶段主要看三个文件：

```text
llvm/lib/Target/RISCV/RISCVFeatures.td
llvm/lib/Target/RISCV/RISCVInstrInfo.td
llvm/lib/Target/RISCV/RISCVInstrInfoXVI.td
```

`RISCVFeatures.td` 定义 `+xvi32r`。  
`RISCVInstrInfo.td` include 新的指令文件。  
`RISCVInstrInfoXVI.td` 定义 RISC-VI 指令本身。

研究仓库里还保留了 patch 原型：

```text
riscv-vi-research/llvm_patches/
```

这很重要。博客里讲工程能力时，不只讲“我改了哪里”，还要讲“我怎么让改动可以复现、检查和迁移”。

## 验证 feature 是否工作

先跑项目里的报告入口：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make llvm-source-tree-report
make llvm-tblgen-report
make llvm-plan
```

再直接调用 `llc`：

```bash
/home/luyoung/llvm-project/build-riscvi-llvm/bin/llc \
  -mtriple=riscv32 -mattr=+xvi32r \
  cases/llvm_codegen/xvi_minmax.ll -o -
```

负向测试也很关键：

```bash
/home/luyoung/llvm-project/build-riscvi-llvm/bin/llc \
  -mtriple=riscv32 \
  cases/llvm_codegen/xvi_minmax.ll -o -
```

如果没开 `+xvi32r` 也出现 `min/max/lwxs`，那说明 predicate 漏了，属于很严重的问题。

## 这种路线的边界

这种做法并不意味着永远不做 `riscvi32r` triple。它只是说明，当前阶段还没到那个点。

我会等这些条件更稳定：

- 指令语义稳定；
- 编码稳定；
- LLVM、模拟器、AM、RTL 都对同一套编码闭环；
- 用户态 ABI 边界明确；
- 更大 workload 有验证结果。

到了那时，再评估新 triple 才更合理。

## 小结

`+xvi32r` 的价值在于小步快跑：复用 RISC-V 后端已经成熟的基础设施，把注意力集中在“这些指令是否真的对 LLVM 输出和双发射流水线有帮助”。下一篇继续往下走，看 TableGen 里一条真实机器指令到底要描述哪些信息。
