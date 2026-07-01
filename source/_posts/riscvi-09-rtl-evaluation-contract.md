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

做 ISA 研究，最容易把话说过头。看到动态指令数少了，就说性能更好；看到某个程序周期少了，就说 ISA 更快；看到分支错误少了，就说分支预测改善了。这些说法如果没有控制变量，很容易误导。

RISC-VI 项目里专门冻结了一份 RTL 评测口径。它的核心原则可以概括成一句话：

```text
所有“RISC-VI 比 RV32R 更好”的结论，
都必须建立在同前端、同预测器、同 LSU、同 memory、同 workload 的对比上。
```

![RTL 对比的控制变量](/images/mermaid-svg/riscvi-09-rtl-evaluation-contract/evaluation-contract.svg)

## 为什么 C 模拟器不能直接证明 IPC

C 模拟器是 reference model，它能证明：

- 指令语义是否正确；
- 程序是否按约定停机；
- 动态提交了多少条指令；
- RISC-VI 指令是否真的被执行；
- load/store/branch 等提交级事件是否发生。

但它不能证明：

- 每条指令花多少周期；
- load-use stall 是否发生；
- 分支预测器如何更新；
- 双发射是否真的同时发出两条；
- 三源指令是否造成读端口阻塞。

所以模拟器数据是进入 RTL 评测的前置证据，而不是最终性能结论。

## 三层 workload 口径

第一层是 ISA 功能闭环。使用 cpu-test、AM 测试和 C 模拟器，主要证明 LLVM 发出的指令能执行，程序返回值正确，动态统计字段合理。

第二层是同源 RTL 功能/计数。使用 `compare_*` 程序对，在同一 RTL 配置下比较：

- retired count；
- cycle count；
- IPC；
- RAW stall；
- load-use stall；
- branch mispredict；
- read port pressure；
- LSU access。

第三层是更长 workload，例如更大的排序、搜索或类 CoreMark 程序。这一层只有在前两层口径稳定后才有意义，否则数据越大，归因越混乱。

## 指标要先定义清楚

`retired_count` 是提交指令数。它最接近动态指令数，但不等于周期。

项目把这些字段冻结在 `riscv-vi-research/docs/rtl_eval_contract.md:84` 开始的列表中。`retired_count`、`cycle_count`、`ipc_x1000`、`riscvi_count`、`stall_*`、`branch_*`、`three_source_count` 和 `lsu_access_count` 都在同一个口径下解释。

`cycle_count` 是从复位到停机的周期数。它会受流水线、memory、分支、阻塞和 workload 长度共同影响。

`ipc_x1000` 是整数化 IPC：

```text
ipc_x1000 = retired_count * 1000 / cycle_count
```

`riscvi_count` 表示提交了多少条 RISC-VI 指令。它说明命中密度，但不自动等价于收益。

`load_count/store_count/lsu_access_count` 用来防止误读 `lwxs/swxs`。indexed memory 能减少地址生成链，但真实 load/store 次数未必减少。

`three_source_count/read_port_pressure_count/stall_read_port_count` 用来观察 `csel`、`swxs` 这类多源指令的结构代价。

`branch_count/branch_mispredict_count/branch_flush_count` 要继续把 `bchkltu` 算作分支。它压缩了边界检查周边指令，但没有消灭控制流。

这个边界在 `riscv-vi-research/docs/rtl_eval_contract.md:160` 明确写出：`bchkltu` 继续计入 branch。

## 同源对比比绝对数字更重要

ISA 评测里，绝对数字很容易误导。比如 RISC-VI 版周期更少，可能是因为：

- 指令数真的减少；
- 分支路径不同；
- workload 输入不同；
- 链接布局不同；
- 预测器配置不同；
- memory 模型不同；
- 统计起止点不同。

所以项目强调同源对比：RV32R 和 RISC-VI32R 使用同一批 workload、同一前端、同一 LSU、同一 memory、同一预测器配置。只有这样，差异才有机会归因到 ISA 和指令序列。

## 分支预测不能混着比

项目里有 static 和 BHT 两类预测器报告。正确比较方式是：

```text
static RV32R vs static RISC-VI
BHT RV32R vs BHT RISC-VI
```

不能拿 static RV32R 和 BHT RISC-VI 比，否则预测器收益和 ISA 收益会混在一起。

`bchkltu` 也必须进入同一预测器口径。它仍然是条件分支，仍然可能被预测正确或错误，仍然可能带来 flush。

## 项目里的验证入口

RTL 口径相关报告包括双发射同源对比、分支预测器对比和读端口限制对比。这些报告不是文章主线，而是把研究口径落成文件和表格的工程手段。读者更应该关注：每个字段是什么、哪些变量被控制、哪些结论不能越界。

## 能说什么，不能说什么

当前可以稳妥地说：

1. RISC-VI 指令是否能正确执行。
2. 同源 workload 下动态提交数是否减少。
3. 同源 RTL 下 retired/cycle/stall 等指标如何变化。
4. `csel` 是否增加三源读端口压力。
5. `bchkltu` 是否减少边界检查周边指令但仍保留分支代价。

当前不能直接说：

1. RISC-VI 一定最终 IPC 更高。
2. RISC-VI 天然改善分支预测器。
3. RISC-VI 一定减少访存次数。
4. 小 workload 结果代表完整软件生态。

## 小结

评测口径是研究可信度的一部分。没有口径，数据越多越容易误导。下一篇基于这套口径，复盘当前双发射实验中 RISC-VI 到底减少了什么，又付出了什么代价。
