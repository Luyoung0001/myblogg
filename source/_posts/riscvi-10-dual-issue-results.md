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

前面几篇已经把链路铺完整：LLVM 后端能发，MC 层能编码，模拟器能作为 reference model，AM/BSP 能承载裸机程序，RTL 评测也有了控制变量。现在可以讨论结果，但仍然要保持克制。

更准确的总结是：

```text
RISC-VI 在当前同源双发射 RTL 功能模型中，
通常能减少动态提交数和部分短依赖，
但也会暴露三源读端口、load-use 和分支预测口径上的真实代价。
```

这组结论对应项目里的三份报告：同源双发射对比见 `riscv-vi-research/docs/dual_issue_compare_report.md:14`，BHT 预测器口径见 `riscv-vi-research/docs/branch_predictor_compare_report.md:5`，4 读端口限制实验见 `riscv-vi-research/docs/read_port_limit_compare_report.md:5`。

![RISC-VI 双发射实验的收益与代价](/images/mermaid-svg/riscvi-10-dual-issue-results/benefit-cost-matrix.svg)

## 不要只看“指令变少”

ISA 扩展最容易讲成一个简单故事：新指令更强，所以指令数少，所以性能更好。这个故事太粗糙。

在双发射里，真正要同时看四件事：

1. 动态提交数是否减少。
2. 短距离 RAW 是否减少。
3. 新指令是否引入结构压力。
4. 周期变化是否能在同源配置下解释。

`lwxs` 是好例子。它能把 `slli/add/lw` 这样的地址生成链压成 indexed load，通常会减少 ALU 指令和 address-data RAW。但它不减少真实 load 次数，也可能让 load-use 更集中。

`csel` 也一样。它能减少小分支，但会变成三源一目的指令。对读端口有限的双发射设计来说，三源不是免费能力。

## 几类 workload 分别说明什么

`compare_array_max` 适合观察 `lwxs/max/bchkltu` 的组合。数组遍历、边界检查和最大值更新混在一起，能体现地址生成、比较选择和分支周边指令压缩。

`compare_clip_accumulate` 更像混合热点。它同时使用：

```text
bchkltu / lwxs / min / max / csel / swxs
```

这种程序适合观察组合收益，也适合暴露代价是否被放大。

例如 `riscv-vi-research/docs/dual_issue_evidence_notes.md:9` 记录了 indexed memory、边界检查和比较选择的正信号，`:15` 开始记录 load-use、三源和单 LSU 这些代价信号。

`compare_select_minmax` 用来提醒我们：消除分支不一定无条件改善。减少控制流后，数据依赖和三源读压力可能变得更明显。

`compare_sort_insert` 和 `compare_sort_bubble` 更偏访存密集。它们能展示 indexed memory 压缩地址链的价值，也能说明 load-use 和 LSU 访问次数不能被忽略。

## `lwxs/swxs`：稳定但有边界的收益来源

从设计证据看，indexed memory 是最稳定的一类收益来源。它对应的源模式非常常见：

```text
base + (index << scale)
```

LLVM 也有明确的地址选择入口，RTL 语义相对直接。它主要减少：

- 地址生成 ALU 指令；
- `slli -> add -> load/store` 的短 RAW；
- 部分动态提交数。

但它不减少：

- 程序真实读取或写入的元素数量；
- LSU 访问次数；
- load 结果被马上消费导致的 load-use 风险。

所以写博客时不能说“`lwxs` 减少访存”，更准确的说法是“`lwxs` 减少访存地址生成链”。

## `min/max/csel`：控制流和读端口的交换

`min/max/minu/maxu` 的落点比较干净。它们对应 LLVM 中稳定的 min/max 语义，也不需要第三源。

`csel` 更微妙。它把一些小分支改成条件选择，可能减少分支条数和错误预测代价；但它需要同时读取 true value、false value 和 condition。RTL 评测里必须看：

- `three_source_delta`
- `read_port_pressure_delta`
- `stall_read_port_delta`

如果只写“分支少了”，就漏掉了 ISA 对硬件资源的要求。

## `bchkltu`：压缩检查，不消灭分支

`bchkltu` 适合边界检查失败路径：

```text
if (idx >= len) goto fail;
```

它的收益来自压缩检查周边指令，并让编译器有一个稳定的失败路径分支形式。但它仍然是条件分支：

- 要进入分支预测器；
- 可能预测错误；
- taken 时可能 flush；
- offset 范围仍按分支指令处理。

所以它不应该被描述成“消除边界检查开销”，而应该描述成“降低边界检查周边的指令和依赖成本”。

## 可以写的阶段性结论

当前比较稳妥的结论是：

1. RISC-VI 在同源小 workload 中通常能减少动态提交数。
2. `lwxs/swxs` 是比较稳定的收益来源，主要压缩地址生成链。
3. `min/max` 能直接覆盖比较选择类语义。
4. `csel` 能减少部分小分支，但读端口代价必须保留。
5. `bchkltu` 能压缩边界检查周边指令，但仍然是分支。
6. 双发射收益必须结合 RAW、load-use、branch、LSU 和读端口指标一起看。

不应该写成：

1. RISC-VI 已经证明最终 IPC 一定更高。
2. RISC-VI 天然改善分支预测。
3. RISC-VI 一定减少访存次数。
4. 当前小 workload 结果可以代表完整软件生态。

## 项目里的验证入口

项目把结果复盘落在几类报告里：同源双发射对比、预测器口径、读端口结构代价和设计证据汇总。它们的价值不是“命令能跑”，而是让每个结论都能回到一个明确指标。

## 系列小结

这个系列真正想表达的不是“加了 8 条指令”，而是一条比较完整的 ISA 研究闭环：

```text
静态依赖数据
  -> ISA 候选指令
  -> LLVM feature / TableGen / SelectionDAG
  -> MC 编码
  -> C 模拟器 reference model
  -> AM/BSP 裸机 workload
  -> RTL 同源评测口径
  -> 收益和代价复盘
```

这条闭环让后续每一次修改都有地方落、有指标看、有边界可守。对实验 ISA 来说，这比单独展示某条指令更重要。
