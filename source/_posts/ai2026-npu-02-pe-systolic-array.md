---
title: "AI2026 NPU 设计笔记（二）：PE 与脉动阵列到底在算什么"
date: 2026-04-29 15:03:13
tags:
  - "AI2026"
  - "NPU"
  - "systolic array"
  - "RTL"
categories:
  - "Computer Architecture"
---

## 前言

NPU 的核心不是“一个很大的乘法器”，而是一堆小 PE 按某种数据流组织起来。PE 负责最小乘加，阵列负责复用数据并展开并行度。

AI2026 的 NPU demo 同时保留了两个层面的材料：单 PE 的最小实现，以及 `16x16` 阵列的外积式矩阵乘法实现。前者适合讲清楚一个计算单元，后者适合讲清楚数据如何在空间结构里复用。

![PE 与阵列的数据复用](/images/mermaid-svg/ai2026-npu-02-pe-systolic-array/pe-systolic-array.svg)

## 单个 PE：最小的 MAC 合约

单 PE 的合约非常短：

![单 PE 的乘加合约](/images/mermaid-svg/ai2026-npu-02-pe-systolic-array/single-pe-contract.svg)

```text
y_out = y_in + a * b
```

这句话直接写在 `AI2026/npu-project/rtl/common/pe.sv:2` 到 `:4`。模块参数里输入是 `DATA_WIDTH = 8`，累加是 `ACCUM_WIDTH = 32`，见 `AI2026/npu-project/rtl/common/pe.sv:6` 到 `:8`。

PE 的输入端口包括：

1. `valid_in`：本拍输入是否有效。
2. `a`：激活值。
3. `b`：权重值。
4. `y_in`：已有累加值。
5. `clear_acc`：清空累加路径。

这些端口定义在 `AI2026/npu-project/rtl/common/pe.sv:13` 到 `:24`。

真正容易被忽略的是有符号处理。`a` 和 `b` 是 8 位，但乘法前要做符号扩展，项目在 `AI2026/npu-project/rtl/common/pe.sv:31` 到 `:34` 显式使用 `$signed` 和高位扩展。随后 product 再扩展到 32 位，与 `y_in` 相加，见 `AI2026/npu-project/rtl/common/pe.sv:40` 到 `:44`。

这就把一个 PE 的正确性拆成三件事：

1. INT8 输入要按有符号数解释。
2. 乘积要扩展后进入 32 位累加。
3. `valid_out` 要跟随 `valid_in`，并反映流水延迟。

`valid_out` 的一拍延迟写在 `AI2026/npu-project/rtl/common/pe.sv:50` 到 `:60`。这类时序细节决定了 PE 能不能被阵列稳定组合。

## 阵列不是复制 PE 这么简单

如果只是把 PE 复制 256 个，还不能叫一个好用的 NPU 阵列。关键问题是数据怎么喂进去。

项目里的 `pe_array_16x16.sv` 采用外积方法。文件开头把数据流讲得很清楚：周期 `k` 发送 A 的第 `k` 列和 B 的第 `k` 行，每个 `PE(row, col)` 累加 `C[row][col] += A[k][row] * B[k][col]`，见 `AI2026/npu-project/rtl/common/pe_array_16x16.sv:6` 到 `:10`。

数学上这就是：

```text
C = sum_k A[:, k] x B[k, :]
```

外积数据流的好处是，一个 A 元素会被同一行/列的一组 PE 复用，一个 B 元素也会被另一方向复用。硬件不需要每个 PE 独立去读完整矩阵，而是把当前 `k` 的一列 A 和一行 B 广播给阵列。

在 RTL 里，A 输入接口一次接收一列，B 输入接口一次接收一行，见 `AI2026/npu-project/rtl/common/pe_array_16x16.sv:31` 到 `:41`。A 和 B 的当前值分别存进 `a_storage` 和 `b_storage`，见 `AI2026/npu-project/rtl/common/pe_array_16x16.sv:56` 到 `:65`。

## 数据 layout 是硬件正确性的组成部分

矩阵乘法公式看起来简单，但 RTL 真正出 bug 的地方往往是数据 layout。

`pe_array_16x16.sv` 开头专门写了小端数据格式：`a_data` 里每个 8 位 lane 对应 A 的一项，`a_storage[row] = A[k][row]`，见 `AI2026/npu-project/rtl/common/pe_array_16x16.sv:12` 到 `:15`。

RTL 在 `a_valid` 时把 `a_data[i*DATA_WIDTH +: DATA_WIDTH]` 放进 `a_storage[i]`，见 `AI2026/npu-project/rtl/common/pe_array_16x16.sv:90` 到 `:94`；B 也用同样的 lane 方式写进 `b_storage[j]`，见同文件 `:100` 到 `:104`。

这说明 layout 不是软件随便定的，也不是硬件随便猜的。软件、buffer、阵列必须对 lane 顺序达成一致，否则矩阵乘法会悄悄变成转置、乱序或错误累加。

这也是为什么 AI 芯片项目里常常会把数据 layout、tile layout 和 DMA layout 单独写成协议。计算公式只有一行，数据摆放却能决定整个系统对不对。

## 16x16 阵列的并行度来自哪里

阵列内部用双层 `generate` 生成每个 `row, col` 的计算逻辑，见 `AI2026/npu-project/rtl/common/pe_array_16x16.sv:120` 到 `:151`。

每个位置取：

```text
a_val = a_storage[row]
b_val = b_storage[col]
product = a_val * b_val
```

然后在 `a_valid_r && b_valid_r` 时累加到 `y_reg[row][col]`，见 `AI2026/npu-project/rtl/common/pe_array_16x16.sv:127` 到 `:146`。

这里的并行度来自空间展开。对于一个给定的 `k`，所有 `row, col` 的 product 都可以同时计算。一个 16x16 阵列理论上每个有效周期可以覆盖 256 个乘加位置。项目文档也把 “16x16 MAC per cycle” 作为阵列能力来解释，见 `AI2026/npu-project/docs/README-cn.md:82` 附近的阵列数据流说明。

但要注意，“每周期 256 个位置参与计算”并不等于整个系统永远按这个速率跑。真实吞吐还受限于：

1. 数据是否能按时从 Unified Buffer 读出。
2. A/B 输入是否每拍有效。
3. 结果提交是否会阻塞。
4. 控制状态机有没有空泡。

所以性能分析不能只看 PE 数量。PE 是峰值，数据流和存储才决定持续利用率。

## 为什么测试里会出现 4x4 阵列

项目默认顶层是可参数化的，完整 NPU 用 16x16 作为设计目标，但 testbench 经常把阵列缩到 4x4。`npu_tb.sv` 里实例化 `npu_top` 时把 `ARRAY_SIZE` 设为 4，见 `AI2026/npu-project/sim/tb/npu_tb.sv:60` 到 `:64`。

这不是设计倒退，而是验证策略。4x4 更容易手算、更容易观察波形，也更容易在文章里解释。

`pe_array_4x4_simple_tb.sv` 里专门写了测试数据流：连续发送 A 的四列和 B 的四行，每个周期一对，见 `AI2026/npu-project/sim/tb/pe_array_4x4_simple_tb.sv:79` 到 `:109`。同一个 testbench 还说明 4x4 阵列需要等待一段周期让数据流过阵列并完成计算，见 `AI2026/npu-project/sim/tb/pe_array_4x4_simple_tb.sv:117` 到 `:121`。

这类小规模验证很重要。它不是为了证明大规模性能，而是为了先证明数据 layout、valid 时序、累加语义和输出收集都没有错。

## 单 PE 测试检查什么

单 PE testbench 检查了几类最基础但最容易出错的事情：

1. `2 * 3 = 6` 的基本乘法。
2. `2 * 3 + 6 = 12` 的累加。
3. `-2 * 3 = -6` 的有符号乘法。
4. `1*1 + 2*2 + 3*3 = 14` 的连续累加。

这些测试写在 `AI2026/npu-project/sim/tb/single_pe_tb.sv:75` 到 `:176`。它们看起来不复杂，但每一类都对应一个硬件风险：符号扩展、累加宽度、清零时机、valid 延迟。

工程上，单元测试的价值就在这里。不要等到整机 `MATMUL` 错了再猜是哪一层坏了；先把 PE 的乘加合约锁住。

## 这篇的核心结论

PE 是 NPU 的最小计算合约，阵列是这个合约的空间展开。

AI2026 的实现值得注意的地方有三点：

1. 单 PE 明确处理 INT8 有符号乘法和 32 位累加。
2. 16x16 阵列采用外积数据流，把 A 列和 B 行广播给所有 PE。
3. 4x4 testbench 用小规模手算数据锁住 layout 和时序。

下一篇我们看另一个决定 NPU 成败的部件：Unified Buffer。没有片上存储，本事再大的阵列也只能等数据。
