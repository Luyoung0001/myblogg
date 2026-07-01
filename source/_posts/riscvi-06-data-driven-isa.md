---
title: "RISC-VI 研究笔记（六）：用静态依赖数据推导 ISA 设计"
date: 2026-04-18 15:03:13
tags:
  - "RISC-VI"
  - "ISA"
  - "dependency analysis"
  - "RISC-V"
categories:
  - "Computer Architecture"
---

## 前言

ISA 设计最怕变成“我觉得这条指令有用”。RISC-VI 的做法是反过来：先看 LLVM 输出的 RV32I 汇编到底卡在哪里，再从瓶颈里推导指令。

当前项目的静态分析基于 15 个用户态整数 C 用例。报告里有几个非常关键的数据：

- 总静态指令数 1162；
- RAW / 指令约 1.198；
- 距离 1 或 2 的短 RAW 占 52.9%；
- address-data 依赖 393 条；
- load-use 依赖 180 条；
- branch-data / branch 达到 85.1%。

这些数据说明：问题不只是“某条指令太慢”，而是高层语言模式被拆成了短距离依赖链。

![从静态依赖数据到 RISC-VI 指令设计](/images/mermaid-svg/riscvi-06-data-driven-isa/evidence-chain.svg)

这张图要表达三件事：

1. 先有 C case 和 RV32I 汇编，再有依赖分析。
2. RISC-VI 指令是从高频依赖模式里推出来的。
3. 静态估算只是设计输入，不等于最终性能结论。

## 三类核心瓶颈

第一类是地址生成链。典型序列是：

```asm
slli t0, index, 2
add  t0, base, t0
lw   rd, 0(t0)
```

这直接对应 `lwxs`：

```asm
lwxs rd, base, index, 2
```

store 版本对应 `swxs`。

第二类是比较选择链。比如 array max 这类程序里，经常出现比较、条件更新和小分支。RISC-VI 用 `min/max/minu/maxu/csel` 去覆盖这些模式。

第三类是边界检查。很多安全访问会先检查 `idx < len`，失败就跳出。RISC-VI 用 `bchkltu` 把失败路径检查收成一条分支。

## 运行分析命令

可以重新生成这些报告：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make verify
make dependency-report-data
make static-estimate
make issue-pressure-report
```

查看报告：

```bash
sed -n '1,220p' docs/riscv32r_dependency_report.md
sed -n '1,180p' docs/issue_pressure_report.md
sed -n '1,220p' docs/riscvi32r_static_estimate_report.md
```

## 静态替换估算

当前估算器识别到 39 个可替换机会，预计减少 63 条静态指令。模式包括：

- `bounds_check_indexed_load`
- `compare_branch_select`
- `scaled_index_load_store`

这支持了第一版 RISC-VI 指令选择：

```text
lwxs / swxs
min / max / minu / maxu
csel
bchkltu
```

当然，静态估算有边界。它不做动态执行频率统计，不做路径敏感分析，也不重新调度完整程序。所以它适合作为 ISA 设计依据，而不是最终性能证明。

## 为什么不是加更多指令

例如除法取模融合、load 后 ALU 融合、更复杂的 bit-mix 指令，都可以想象。但第一阶段不适合全塞进去。

好的实验 ISA 应该先满足几个条件：

1. LLVM IR 中模式稳定；
2. 硬件代价可控；
3. 模拟器和 RTL 容易验证；
4. 能形成端到端闭环；
5. 可以通过数据解释收益来源。

RISC-VI v0.1 的 8 条指令正是按这个标准筛出来的。

## 小结

这一篇的核心是：RISC-VI 的设计不是拍脑袋，而是从 RV32I 输出中的依赖压力推出来的。下一篇看 C 模拟器，它负责把这些指令的语义变成可执行、可对拍的 reference model。
