---
title: "RISC-VI 研究笔记（十）：双发射实验里收益和代价要一起看"
date: 2026-04-23 15:03:13
tags:
  - "RISC-VI"
  - "dual issue"
  - "RTL"
  - "performance"
categories:
  - "Computer Architecture"
---

## 前言

前面几篇已经把链路讲完了：LLVM 后端能发，模拟器能跑，AM/BSP 能启动裸机程序，RTL 对比也有了评测口径。现在可以看结果了。

但这一篇我不想写成“RISC-VI 大胜 RV32R”。更准确的说法是：

```text
RISC-VI 在当前同源双发射 RTL 功能模型中，
通常能减少动态提交数和部分短依赖，
但也会暴露三源读端口、分支和 LSU 口径上的真实代价。
```

![RISC-VI 双发射实验的收益与代价](/images/mermaid-svg/riscvi-10-dual-issue-results/benefit-cost-matrix.svg)

这张图要表达三件事：

1. `lwxs/swxs` 减少地址生成链，但不减少真实 LSU 访问。
2. `csel` 可以减少小分支，但会增加三源读取压力。
3. `bchkltu` 压缩边界检查，但仍然是条件分支。

## 当前同源对比

当前双发射报告里有多组 `compare_*` 程序对，例如：

- `compare_array_max`
- `compare_bounds_sum`
- `compare_clip_accumulate`
- `compare_filter_minmax`
- `compare_dispatch_table`
- `compare_select_minmax`
- `compare_memcopy_checksum`
- `compare_sort_insert`
- `compare_sort_bubble`

这些程序对的要求是：RV32R 版和 RISC-VI 版计算同一个结果，只是指令序列不同。

重新生成报告：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make dual-issue-compare-report
make branch-predictor-report
make read-port-limit-report
make design-evidence
```

查看：

```bash
sed -n '1,220p' docs/dual_issue_compare_report.md
sed -n '1,180p' docs/branch_predictor_compare_report.md
sed -n '1,220p' docs/riscvi_design_evidence_report.md
```

## 几个典型 case

`compare_array_max` 很适合展示 `lwxs/max/bchkltu` 的组合。RISC-VI 版用 indexed load 压缩地址生成，用 `max` 替代条件更新，用 `bchkltu` 覆盖边界检查失败路径。这个 case 往往能看到提交数、RAW 和分支预测错误都有改善。

`compare_clip_accumulate` 更像混合热点。它同时使用：

```text
bchkltu / lwxs / min / max / csel / swxs
```

所以它适合观察指令组合带来的整体效果，也适合观察代价是否被放大。

`compare_select_minmax` 则提醒我们不要只看提交数。某些局部情况下，减少分支之后，数据依赖或三源压力可能变得更明显。

`compare_sort_insert` 和 `compare_sort_bubble` 属于访存密集型程序。它们能展示 indexed memory 的价值，也能说明：减少地址生成链不等于减少 LSU 访问次数。

## 分支预测不能混着比

分支预测报告里有 static 和 BHT 两种配置。正确的比较方式是：

```text
static RV32R vs static RISC-VI
BHT RV32R vs BHT RISC-VI
```

不能拿 static RV32R 去和 BHT RISC-VI 比。否则就把预测器收益和 ISA 收益混在一起了。

`bchkltu` 也必须进入同一预测器口径。它不是“无控制代价”的指令，它仍然是条件分支。

## 读端口压力

`csel` 是三源一目的指令。`swxs` 在固定 32 位编码下也会使用类似第三源的字段语义。它们可能减少提交数，但也会增加读端口需求。

所以报告里要看：

- `source_operand_delta`
- `three_source_delta`
- `read_port_pressure_delta`
- `stall_read_port_delta`

这类字段能防止我们只看到“指令变少”，却忽略“硬件读端口更紧张”。

## 可以写的结论

当前比较稳妥的结论是：

1. RISC-VI 在同源小 workload 中通常能减少动态提交数。
2. `lwxs/swxs` 是比较稳定的收益来源，主要压缩地址生成链。
3. `min/max/csel` 能减少部分比较选择和小分支。
4. `bchkltu` 能压缩边界检查周边指令，但仍有分支代价。
5. 多源指令的读端口压力必须作为设计代价保留在报告里。

不能写成：

1. RISC-VI 已经证明最终 IPC 一定更高。
2. RISC-VI 天然改善分支预测。
3. RISC-VI 一定减少访存次数。
4. 当前小 workload 结果可以代表完整软件生态。

## 系列小结

这个系列从 C 程序一路走到了双发射 RTL 计数。真正有价值的地方不只是“加了几条指令”，而是形成了一条研究闭环：

```text
问题数据 -> ISA 设计 -> LLVM 后端 -> 模拟器 -> AM/BSP -> cpu-test -> RTL 口径 -> 实验报告
```

这条闭环让后续每一次修改都有地方落、有命令跑、有数据看，也有边界可守。对一个实验 ISA 项目来说，这比单独展示某条指令更重要。
