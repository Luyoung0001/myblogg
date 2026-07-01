---
title: "RISC-VI 研究笔记（九）：从模拟器到 RTL，如何建立可信评测口径"
date: 2026-04-21 15:03:13
tags:
  - "RISC-VI"
  - "RTL"
  - "performance"
  - "difftest"
categories:
  - "Computer Architecture"
---

## 前言

做 ISA 研究，最容易把话说过头。比如看到某个程序的周期少了，就说“这个 ISA 更快”；看到分支错误少了，就说“分支预测更好了”。这些说法如果没有控制变量，很容易失真。

RISC-VI 项目里专门冻结了一份 RTL 评测口径。核心原则是：

```text
所有“RISC-VI 比 RISC-V 好”的结论，
都必须建立在同前端、同预测器、同 LSU、同 memory、同 workload 的对比上。
```

![RTL 对比的控制变量](/images/mermaid-svg/riscvi-09-rtl-evaluation-contract/evaluation-contract.svg)

这张图要表达三件事：

1. RV32R 和 RISC-VI32R 必须走同源对比。
2. 预测器、LSU、memory 不一致时，不能把差异归因于 ISA。
3. C 模拟器和 RTL 证明的东西不同。

## 三层 workload

第一层是 ISA/功能闭环层。使用 `cpu-test` 和 C 模拟器，主要证明：

- LLVM 发出的指令能执行；
- 程序返回值正确；
- 动态指令数和 RISC-VI 命中数可统计。

这一层不证明 IPC。

第二层是同源 RTL 功能/计数层。使用 `compare_*` 程序对，才开始看：

- retired count；
- cycle count；
- IPC；
- RAW stall；
- load-use stall；
- branch predictor；
- read port pressure。

第三层是更长 workload。比如更大的排序、搜索、类 CoreMark 程序。这一层必须在前两层口径稳定后再扩展。

## 关键指标

`retired_count` 是提交指令数。它最接近动态指令数。

`cycle_count` 是从复位到停机的总周期。

`ipc_x1000` 定义为：

```text
retired_count * 1000 / cycle_count
```

`riscvi_count` 只是 RISC-VI 指令提交数量。它表示新指令命中密度，不等于性能收益。

`load_count/store_count/lsu_access_count` 很重要，因为：

```text
lwxs/swxs 能减少地址生成链，不代表一定减少真实 load/store 次数。
```

`three_source_count` 和 `read_port_pressure_count` 用来观察 `csel/swxs` 这类多源指令的结构代价。

## 运行报告

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make sim-trace-smoke
make riscvi-trace-align-check
make dual-issue-compare-report
make branch-predictor-report
make read-port-limit-report
```

可以查看：

```bash
sed -n '1,220p' docs/rtl_eval_contract.md
sed -n '1,220p' docs/dual_issue_compare_report.md
sed -n '1,180p' docs/branch_predictor_compare_report.md
```

## 能说什么，不能说什么

当前可以说：

- RISC-VI 指令能否正确执行；
- 同源 workload 下动态指令是否减少；
- 同源 RTL 下 retired/cycle/stall 等指标怎么变化；
- `csel` 是否增加三源压力；
- `bchkltu` 是否减少检查周边指令但仍保留分支代价。

当前不能直接说：

- RISC-VI 一定更高 IPC；
- RISC-VI 天然让分支预测更好；
- RISC-VI 一定减少访存次数；
- 当前结果已经代表完整软件生态。

## 小结

评测口径是研究可信度的一部分。没有口径，数据越多越容易误导。下一篇我们基于这个口径，复盘当前双发射实验中 RISC-VI 到底减少了什么，又付出了什么代价。
