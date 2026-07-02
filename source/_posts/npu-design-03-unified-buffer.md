---
title: "NPU 设计（三）：Unified Buffer 与片上数据复用"
date: 2026-04-30 15:03:13
tags:
  - "NPU"
  - "memory hierarchy"
  - "RTL"
  - "buffer"
categories:
  - "Computer Architecture"
---

## 前言

NPU 的性能瓶颈经常不是“算不动”，而是“喂不饱”。PE Array 可以很大，但如果每一拍都在等数据，面积再多也只是摆设。

Unified Buffer 是 NPU 里的片上数据仓库。它的价值不只是“存一些数”，而是定义数据本地性：哪些数据已经在片上，哪些结果可以原地处理，哪些数据需要输出到外部。

![Unified Buffer 与本地性边界](/images/mermaid-svg/npu-design-03-unified-buffer/unified-buffer.svg)

## 为什么片上 buffer 很重要

矩阵乘法中，同一个 A 元素会参与多个输出，同一个 B 元素也会参与多个输出：

```text
C[i][0] += A[i][k] * B[k][0]
C[i][1] += A[i][k] * B[k][1]
C[i][2] += A[i][k] * B[k][2]
...
```

如果 `A[i][k]` 每用一次都从外部内存读一次，带宽会非常浪费。更好的方式是：先把一块 A/B 数据搬到片上，然后在 PE Array 内多次复用。

所以 NPU 里常见的数据路径是：

```text
外部输入
  -> Unified Buffer
  -> PE Array
  -> Unified Buffer
  -> 激活/池化
  -> 输出
```

Unified Buffer 是这个路径的中心。

## 一个 4KB 教学级地址分区

为了让数据生命周期清晰，可以把 4KB 片上 buffer 分成四段：

![Unified Buffer 地址分区](/images/mermaid-svg/npu-design-03-unified-buffer/ub-address-map.svg)

| 地址范围 | 用途 | 大小 |
|---|---|---|
| `0x000 - 0x3FF` | Weight Buffer | 1KB |
| `0x400 - 0x7FF` | Activation Buffer | 1KB |
| `0x800 - 0xBFF` | Partial Sum | 1KB |
| `0xC00 - 0xFFF` | Output Buffer | 1KB |

这个划分的目的不是凑整，而是让数据职责分开：

- 权重区保存模型参数；
- 激活区保存输入特征；
- partial sum 区保存中间累加；
- output 区保存最终结果。

当设计变复杂以后，这种分区会演化成更严肃的 tile layout、bank layout、DMA descriptor 和 double buffering。

## 三个访问端口对应三类角色

一个简单 Unified Buffer 可以提供三个逻辑端口：

| 端口 | 使用者 | 典型动作 |
|---|---|---|
| Port A | PE Array | 读取 A/B 数据 |
| Port B | 数据搬运通道 | 写入输入，读出结果 |
| Port C | 控制/后处理 | 写回 C，执行 ACT/POOL |

简化 RTL 形状如下：

```systemverilog
reg [31:0] mem [0:4095];

always_ff @(posedge clk) begin
  if (a_en && !a_we) a_dout <= mem[a_addr];
  if (a_en &&  a_we) mem[a_addr] <= a_din;
end

always_ff @(posedge clk) begin
  if (b_en && !b_we) b_dout <= mem[b_addr];
  if (b_en &&  b_we) mem[b_addr] <= b_din;
end

always_ff @(posedge clk) begin
  if (c_en && !c_we) c_dout <= mem[c_addr];
  if (c_en &&  c_we) mem[c_addr] <= c_din;
end
```

这个片段有一个隐含重点：读是同步读。地址在这一拍给进去，数据通常下一拍才稳定。

## 同步读会改变状态机设计

算法里我们经常写：

```text
a = buffer[addr_a]
b = buffer[addr_b]
send_to_pe(a, b)
```

RTL 里不能这么随意。同步 SRAM 需要拆节拍：

```text
第 1 拍：给出 addr_a / addr_b
第 2 拍：捕获 a_dout / b_dout
第 3 拍：拉高 valid，送入 PE Array
```

所以一个矩阵乘法控制状态机常常会有这些阶段：

| 状态 | 含义 |
|---|---|
| `CLEAR` | 清空 PE 累加器 |
| `CAPTURE` | 等待并捕获 buffer 读数据 |
| `SEND` | 向 PE Array 发送一组 A/B |
| `DRAIN` | 等阵列内部流水排空 |
| `COMMIT` | 把结果写回 buffer |

这就是为什么存储模块会影响控制模块。硬件不是抽象数学，读写延迟会塑造整个状态机。

## 结果为什么要写回 buffer

一个直觉设计是：

```text
PE Array -> 直接输出
```

但如果后面还要做 ReLU 或 Pooling，直接输出就不方便了。更实用的方式是：

```text
PE Array -> 写回 result 区
ACT      -> 从 result 区读，原地写回
POOL     -> 从 result 区读窗口，写回压缩结果
STORE    -> 从 result 区输出
```

这样做有两个好处：

1. 中间结果不用离开片上 buffer。
2. 后处理可以复用同一套地址和存储协议。

这也是 Unified Buffer 叫 “unified” 的原因：它不是某一个模块的私有 RAM，而是计算、后处理和输出之间共享的数据边界。

## 什么时候需要更复杂的 buffer

教学级 Unified Buffer 可以先用一个朴素三端口模型，但真实 NPU 会遇到更多问题：

1. PE Array 每拍要读很多数据，单端口不够。
2. DMA 写入和 PE 读取可能冲突。
3. ACT/POOL 读写结果区会和 STORE 竞争。
4. 大矩阵需要 tile 分块，地址生成更复杂。
5. 为了隐藏搬运延迟，需要 double buffering。
6. 多 bank 设计要处理 bank conflict。

所以 buffer 设计不是“把容量调大”这么简单。容量、带宽、端口、bank、地址映射、仲裁策略，都会影响 NPU 的有效吞吐。

## 小结

Unified Buffer 是 NPU 的本地性边界。

这一篇可以记住三点：

1. 片上 buffer 的核心价值是数据复用，而不是单纯存储。
2. 地址分区表达了权重、激活、中间结果和输出的生命周期。
3. 同步读延迟会直接影响 MATMUL、ACT、POOL、STORE 的状态机设计。

下一篇继续往控制层走：NPU 如何用命令 FIFO 和状态机把 `LOAD -> MATMUL -> ACT -> POOL -> STORE` 串成一条可执行算子链。
