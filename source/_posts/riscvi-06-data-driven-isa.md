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

ISA 设计最怕变成“我觉得这条指令有用”。RISC-VI 的做法是反过来：先看 LLVM 生成的 RV32I 汇编到底暴露了哪些瓶颈，再从瓶颈里推导指令。

这种方法的重点不是跑几个命令，而是建立一条证据链：

```text
C workload
  -> LLVM 生成 RV32I 汇编
  -> 静态指令和依赖分析
  -> 双发射压力分类
  -> 候选 ISA 模式
  -> RISC-VI 指令集草案
```

![从静态依赖数据到 RISC-VI 指令设计](/images/mermaid-svg/riscvi-06-data-driven-isa/evidence-chain.svg)

## 为什么看静态依赖

RISC-VI 面向的是一个双发射语境。双发射处理器的理想情况是每周期提交两条互不冲突的指令，但现实里经常被这些因素限制：

- 相邻 RAW 数据依赖；
- load-use 紧邻消费；
- 分支依赖前序比较或加载结果；
- load/store 地址生成链；
- 结构资源冲突；
- 控制流导致的 fetch/issue 空洞。

静态依赖分析不能替代 RTL 性能评测，但它可以帮助我们发现“编译器经常生成什么形状的短链”。ISA 扩展如果能直接压缩这些短链，就有机会改善双发射压力。

## 项目里的关键观察

当前 RISC-VI 研究基于 15 个用户态整数 C 样例，对 RV32I 汇编做静态分析。项目报告中的几个数字很有代表性：

- 总静态指令数为 1162。
- RAW / 指令为 1.198。
- 距离 1 或 2 的短 RAW 占全部 RAW 的 52.9%。
- address-data 依赖为 393 条。
- load-use 依赖为 180 条。
- branch-data / branch 为 85.1%。

这些数字来自 `riscv-vi-research/docs/riscv32r_dependency_report.md:90` 的总体指标表；双发射前置压力报告也在 `riscv-vi-research/docs/issue_pressure_report.md:5` 给出了同样的摘要。

这些数字说明，瓶颈不是某一条“慢指令”，而是高层语言模式被拆成短距离依赖链。双发射最怕这种链：指令数量看似不大，但后一条马上等前一条结果，第二发射槽很难被填满。

## 第一类：地址生成链

数组访问在 RV32I 上经常长这样：

```asm
slli t0, index, 2
add  t0, base, t0
lw   rd, 0(t0)
```

这里有两层问题：

1. `slli -> add -> lw` 形成紧密 RAW。
2. load/store 本身没有减少，真正被浪费的是地址生成 ALU 指令和它们造成的依赖距离。

因此 RISC-VI 设计了：

```text
lwxs / swxs
```

它们的目标不是减少内存访问次数，而是把常见的 `base + (index << scale)` 地址生成折叠进 load/store。这个目标很克制，也容易被 LLVM 的 address selection 接住。

## 第二类：比较选择链

很多整数程序里有这类模式：

```c
if (x > best) best = x;
```

RV32I 上可能变成比较、分支、移动或条件路径更新。对双发射来说，这类小控制流会同时带来 branch-data 和 fetch/issue 压力。

RISC-VI 用两组指令覆盖：

```text
min / max / minu / maxu
csel
```

`min/max` 对应 LLVM 中稳定的 `smin/smax/umin/umax` 节点，是最自然的编译器落点。`csel` 覆盖更一般的条件选择，但它带来三源读端口代价。所以它不是“无条件越多越好”，而是要在后续 RTL 中观察收益和压力。

## 第三类：边界检查失败分支

安全访问经常包含：

```c
if (idx >= len) goto fail;
value = array[idx];
```

或者从正向写法看：

```c
if (idx < len) {
  value = array[idx];
}
```

这类边界检查会把比较、分支和后续地址生成绑在一起。RISC-VI 设计 `bchkltu`，把失败路径检查稳定编码成单条分支：

```text
bchkltu idx, len, fail
```

它的价值是压缩边界检查周边指令，不是消灭分支。后续评测必须继续把它算作 branch，并进入同一预测器口径。

## 静态估算能说明什么

项目中的静态替换估算识别了若干可替换模式，例如：

- `scaled_index_load_store`
- `compare_branch_select`
- `bounds_check_indexed_load`

项目状态报告在 `riscv-vi-research/docs/project_status_report_v02.md:20` 记录了这组估算：识别 39 个机会，预计减少 63 条静态指令。这里要注意它是设计输入，不是最终性能结论。

这些模式支持第一版 RISC-VI 指令选择：

```text
lwxs / swxs
min / max / minu / maxu
csel
bchkltu
```

但静态估算不是最终性能结论。它不知道动态执行频率，不知道分支预测状态，不知道 RTL 旁路和读端口限制，也不做完整调度。因此它适合作为 ISA 设计输入，而不是最终胜负证明。

## 为什么不加更多指令

当然，还可以想象很多指令：除法取模融合、load 后 ALU 融合、bit-mix、rotate、复杂字符串操作。但第一阶段不能什么都加。

一个实验 ISA 的第一版最好满足这些条件：

1. LLVM IR/DAG 中有稳定模式。
2. 编码和模拟器实现简单。
3. 硬件代价可控。
4. 可以形成端到端闭环。
5. 收益来源可以被数据解释。

RISC-VI v0.1 的 8 条指令就是按这个标准收敛出来的。它们不是最华丽的一组，而是最容易建立证据链的一组。

## 项目里的验证入口

项目里有静态依赖报告、静态替换估算和 issue pressure 报告。它们的作用是把“为什么设计这些指令”变成可检查的数据，而不是把读者带进命令细节。真正值得关注的是报告中依赖类型、替换机会和 ISA 设计之间的对应关系。

## 小结

这一篇的核心是：RISC-VI 的设计不是从指令表开始，而是从 RV32I 输出里的短距离依赖、地址生成链和分支数据依赖开始。下一篇进入 C 模拟器，看这些 ISA 语义如何变成可执行、可对拍的 reference model。
