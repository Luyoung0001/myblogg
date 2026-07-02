---
title: "AI2026 NPU 设计笔记（一）：从神经网络计算到 NPU 数据通路"
date: 2026-04-28 15:03:13
tags:
  - "AI2026"
  - "NPU"
  - "RTL"
  - "matrix multiplication"
categories:
  - "Computer Architecture"
---

## 前言

很多人第一次看 NPU，容易把它理解成“更快的矩阵乘法器”。这句话不算错，但太短了。真正值得学习的是：为什么神经网络推理会被改写成一组重复、规则、可复用的数据流，以及硬件如何把这组数据流固定下来。

AI2026 这个仓库的定位很清楚：它是一组 AI 芯片教学和仿真 demo，主线是用 Verilator 展示 NPU/TPU/Coolda SoC 的基本工作方式，不做 FPGA 下板验证，见 `AI2026/README.md:3`。其中 `npu-project` 专门覆盖 PE、16x16 阵列、控制器、Unified Buffer、激活和池化，见 `AI2026/README.md:8`。

所以这一组 NPU 文章不把重点放在“怎么运行某个命令”，而是从硬件设计角度回答一个更通用的问题：一个神经网络算子如何被拆成数据通路、片上存储、控制状态机和验证层。

![从矩阵乘法到 NPU 数据通路](/images/mermaid-svg/ai2026-npu-01-matrix-to-npu/matrix-to-npu.svg)

## 神经网络算子为什么总绕不开矩阵乘法

全连接层、卷积展开后的 `im2col`、attention 里的 QK/AV，本质上都会把大量工作压到乘加上：

```text
C[i][j] = sum_k A[i][k] * B[k][j]
```

CPU 当然也能算这件事，但 CPU 的强项是复杂控制流、分支预测、缓存层次和通用指令集。NPU 的设计目标更窄：把一类高度重复的乘加循环变成空间上的并行结构。

这意味着硬件关注点会从“我能执行哪些指令”变成：

1. 每个周期能做多少个 MAC。
2. A、B、C 三类数据如何复用。
3. 中间结果放在哪里。
4. 主机如何告诉 NPU 当前要做 `LOAD`、`MATMUL`、`ACT`、`POOL` 还是 `STORE`。

`npu-project/README.md` 也把这个 demo 的覆盖范围写得很直白：它包含 `INT8` 乘加 PE、`16x16` PE 阵列、命令 FIFO、控制状态机、Unified Buffer，以及 `MATMUL -> ACT -> POOL -> STORE` 数据处理链路，见 `AI2026/npu-project/README.md:7` 到 `:12`。

## NPU 数据通路的四个角色

从顶层看，NPU 不是一个孤零零的乘法器。`npu_top.sv` 文件开头直接列出了它整合的子模块：Control Unit、PE Array、Unified Buffer、Activation Unit 和 Pooling Unit，见 `AI2026/npu-project/rtl/top/npu_top.sv:4` 到 `:9`。

这五个模块可以按职责分成四类。

第一类是计算核心。PE Array 负责把矩阵乘法展开成多个 PE 的并行乘加。项目默认参数里 `ARRAY_SIZE` 是 16，输入是 INT8，累加是 32 位，见 `AI2026/npu-project/rtl/top/npu_top.sv:12` 到 `:16`。

第二类是片上存储。Unified Buffer 存权重、激活、中间 partial sum 和输出，文件注释在 `AI2026/npu-project/rtl/control/unified_buffer.sv:4` 到 `:8`。这不是装饰性模块，而是决定数据能不能复用的边界。

第三类是控制。Control Unit 解析主机命令、控制数据流、协调各模块，见 `AI2026/npu-project/rtl/control/control_unit.sv:4` 到 `:7`。它把复杂的软件意图压成一组硬件状态。

第四类是后处理。Activation Unit 支持 ReLU、LeakyReLU、Sigmoid、Tanh，见 `AI2026/npu-project/rtl/control/activation_unit.sv:4` 到 `:8`；Pooling Unit 支持 Max Pooling 和 Average Pooling，见 `AI2026/npu-project/rtl/control/pooling_unit.sv:4` 到 `:6`。

把它们连起来，就是一个最小但完整的推理数据通路。

## 为什么选择 INT8 输入和 32 位累加

AI 推理硬件常用 INT8，不是因为 INT8 天生“高级”，而是因为推理阶段通常可以通过量化把浮点权重/激活压到更低位宽。低位宽的价值很直接：

1. 单个乘法器更小。
2. 同样面积能放更多 PE。
3. 片上 buffer 和片外带宽压力降低。
4. 能耗下降。

但累加不能也随便用 8 位。矩阵乘法的每个结果是很多项相加，哪怕单项是 INT8，累加结果也很容易超过 8 位范围。项目里的单 PE 明确把输入位宽设为 `DATA_WIDTH = 8`，累加位宽设为 `ACCUM_WIDTH = 32`，见 `AI2026/npu-project/rtl/common/pe.sv:6` 到 `:8`。

这正是硬件设计里常见的格式分工：输入压窄，乘法便宜；累加放宽，保证数值不那么快溢出。

## 从命令链看一个算子如何落地

项目没有把主机接口设计成“随便写几个信号”。Control Unit 里定义了非常清楚的命令 opcode：

- `0x10` 表示 `LOAD_W`。
- `0x11` 表示 `LOAD_A`。
- `0x20` 表示 `MATMUL`。
- `0x30` 表示 `ACT`。
- `0x40` 表示 `POOL`。
- `0x50` 表示 `STORE`。

这些定义在 `AI2026/npu-project/rtl/control/control_unit.sv:68` 到 `:75`，命令格式说明在同文件 `:303` 到 `:317`。

这套命令链有一个很重要的教学价值：它把 NPU 的工作拆成了“搬数据、计算、后处理、搬结果”。真实 NPU 当然会有更复杂的 DMA、descriptor、queue、interrupt 和 runtime，但最小 mental model 并没有变。

换句话说，`MATMUL` 不是凭空发生的。它之前必须有权重和激活到达片上存储；之后还可能有激活、池化和结果写回。把这个过程分开讲，读者才能理解 NPU 是系统，不是一个运算符。

## 顶层接口的工程含义

`npu_top.sv` 的主机接口采用了类似 AXI-Stream 的三组信号：命令输入、数据输入、数据输出，见 `AI2026/npu-project/rtl/top/npu_top.sv:24` 到 `:37`。

这意味着顶层把外部世界抽象成三件事：

1. 主机送来命令。
2. 主机送来输入数据。
3. NPU 流出输出数据。

它没有在这个 demo 里引入完整 AXI master、真实 DMA 或 cache coherency。这个边界也写在项目 README 里：当前 demo 没有商业 NPU 所需的完整 DMA、cache coherency、多核调度、编译器图优化、量化工具链或大模型算子库，见 `AI2026/npu-project/README.md:45` 到 `:49`。

这不是缺点，而是教学项目很重要的取舍。把系统缩小到这个尺度以后，读者能看清数据通路本身：命令进来，数据进入 UB，PE Array 计算，结果被后处理，再从输出通道流出去。

## 这篇的核心结论

NPU 的第一性原理不是“神经网络需要矩阵乘法”这一句话，而是四个工程事实：

1. 神经网络推理有大量规则 MAC，适合空间并行。
2. INT8 输入和 32 位累加是一种面积、带宽和数值范围之间的折中。
3. 片上 buffer 决定数据复用边界。
4. 控制命令把软件意图翻译成硬件状态。

AI2026 的 `npu-project` 刚好把这四件事压缩到一个可以阅读、可以仿真的 RTL demo 里。下一篇我们进入最核心的一层：PE 和 16x16 阵列到底在算什么。
