---
title: "NPU 设计（四）：控制状态机、激活和池化"
date: 2026-05-01 15:03:13
tags:
  - "NPU"
  - "control unit"
  - "post-processing"
  - "RTL"
categories:
  - "Computer Architecture"
---

## 前言

PE Array 负责算，Unified Buffer 负责存，但 NPU 还缺一个“指挥系统”。这个系统就是 Control Unit。

Control Unit 不做乘法，却决定：

1. 什么时候接收命令；
2. 什么时候从 buffer 读数据；
3. 什么时候启动 PE Array；
4. 什么时候做激活和池化；
5. 什么时候把结果输出。

![控制状态机与后处理链路](/images/mermaid-svg/npu-design-04-control-postprocess/control-postprocess.svg)

## 32 位命令格式

一个最小 NPU 可以使用 32 位命令：

![NPU 命令字段与状态机](/images/mermaid-svg/npu-design-04-control-postprocess/command-fsm.svg)

```text
31          24 23          12 19    16 11           0
+-------------+--------------+--------+--------------+
|   opcode    |  addr/base   | extra  |   len/rows   |
+-------------+--------------+--------+--------------+
```

几个典型 opcode：

| opcode | 命令 | 语义 |
|---|---|---|
| `0x10` | `LOAD_W` | 把权重写入 buffer |
| `0x11` | `LOAD_A` | 把激活写入 buffer |
| `0x20` | `MATMUL` | 启动矩阵乘法 |
| `0x30` | `ACT` | 对结果做激活 |
| `0x40` | `POOL` | 对结果做池化 |
| `0x50` | `STORE` | 输出结果 |

这套命令格式把软件意图压缩成硬件字段。软件不需要知道 PE 的每根线，只需要告诉 NPU：要做什么、数据在哪、长度是多少、额外模式是什么。

## 为什么需要命令 FIFO

主机发命令和 NPU 执行命令不一定同速。

例如 `LOAD` 可能持续多个数据拍，`MATMUL` 可能持续多个阵列周期，`POOL` 还要读 2x2 窗口。主机如果连续发命令，NPU 需要一个地方暂存。

命令 FIFO 的语义可以写成：

```systemverilog
cmd_ready = !fifo_full;

if (cmd_valid && cmd_ready) begin
  fifo.push(cmd_data);
end
```

这样主机看到 `cmd_ready = 0` 时，就知道 NPU 暂时不能接新命令。这就是最小 backpressure。

## 主状态机

控制状态机可以这样组织：

```text
IDLE
  -> FETCH
  -> DECODE
  -> DMA_LOAD   (LOAD_W / LOAD_A)
  -> MATMUL     (MATMUL)
  -> ACT        (ACT)
  -> POOL       (POOL)
  -> DMA_STORE  (STORE)
  -> DONE
  -> IDLE
```

简化伪代码：

```systemverilog
case (state)
  IDLE:
    if (!fifo_empty) state <= FETCH;

  FETCH:
    state <= DECODE;

  DECODE:
    case (opcode)
      LOAD_W, LOAD_A: state <= DMA_LOAD;
      MATMUL:         state <= MATMUL;
      ACT:            state <= ACT;
      POOL:           state <= POOL;
      STORE:          state <= DMA_STORE;
      default:        state <= IDLE;
    endcase

  MATMUL:
    if (pe_done) state <= DONE;

  ACT, POOL:
    if (post_done) state <= DONE;

  DMA_LOAD, DMA_STORE:
    if (dma_done) state <= DONE;

  DONE:
    state <= IDLE;
endcase
```

这里的 `FETCH -> DECODE` 两拍很重要。很多 FIFO 或 SRAM 是同步读，不能在发起读取的同一拍就稳定拿到命令。状态机多一拍，是为了让硬件时序成立。

## `MATMUL` 如何启动 PE Array

当 decode 到 `MATMUL` 时，控制单元通常会产生：

```text
pe_start = 1
pe_a_addr = base
pe_b_addr = base + offset_B
pe_c_addr = base + offset_C
pe_rows   = rows
pe_cols   = cols
```

这几个信号不是矩阵乘法本身，而是告诉顶层数据通路：

1. 从哪里读 A；
2. 从哪里读 B；
3. 结果写到哪里；
4. 本次矩阵大小是多少。

真正的读 buffer、送 PE、写回 C，会由顶层 MATMUL 子状态机完成。

## 激活函数：让线性计算变成神经网络

只有矩阵乘法是不够的。多层线性变换叠起来仍然是线性变换，神经网络需要非线性。

最常见的激活函数是 ReLU：

```text
ReLU(x) = max(0, x)
```

RTL 可以非常简单：

```systemverilog
assign relu_out = x[31] ? 32'd0 : x;
```

LeakyReLU 保留负半轴的一点斜率：

```text
LeakyReLU(x) = x,       if x >= 0
             = alpha*x, if x < 0
```

Sigmoid 和 Tanh 更复杂，真实硬件一般不会直接做昂贵的指数函数，而会采用查表、分段近似或专用近似单元。

一个激活选择器可以写成：

```systemverilog
case (act_type)
  ACT_NONE:  y = x;
  ACT_RELU:  y = x[31] ? 0 : x;
  ACT_LEAKY: y = x[31] ? (x >>> slope_shift) : x;
  ACT_SIGM:  y = sigmoid_lut[index];
  ACT_TANH:  y = tanh_lut[index];
endcase
```

## 池化：压缩局部窗口

Pooling 用来把局部窗口压缩成一个值。2x2 max pooling 的公式是：

```text
out = max(v00, v01, v10, v11)
```

2x2 average pooling 则是：

```text
out = (v00 + v01 + v10 + v11) / 4
```

硬件上，POOL 通常需要多拍读取窗口：

```text
读 v00
读 v01
读 v10
读 v11
计算 max/avg
写回 out
```

简化函数：

```systemverilog
function [31:0] pool2x2;
  input [31:0] v0, v1, v2, v3;
  input        avg_mode;
  begin
    if (avg_mode)
      pool2x2 = (v0 + v1 + v2 + v3) >>> 2;
    else
      pool2x2 = max(max(v0, v1), max(v2, v3));
  end
endfunction
```

注意 pooling 的复杂点不是公式，而是地址。对于二维结果矩阵，2x2 窗口的四个地址通常是：

```text
base + row*stride + col
base + row*stride + col + 1
base + (row+1)*stride + col
base + (row+1)*stride + col + 1
```

如果地址错了，pooling 结果会看似合理但对应错窗口。

## 后处理为什么常常原地写回

激活和池化都可以在矩阵结果区上原地处理：

```text
MATMUL  写 C 区
ACT     读 C 区，写回 C 区
POOL    读 C 区窗口，写回 C 区前半部分
STORE   输出 C 区有效结果
```

这样可以避免中间结果频繁离开片上 buffer。对 NPU 来说，减少数据搬运通常比节省几个简单 ALU 操作更重要。

## 小结

控制单元把 NPU 从“计算电路”变成“可被主机驱动的协处理器”。

这一篇的核心是：

1. 命令格式是软硬件之间的最小协议。
2. FIFO 和状态机负责把命令变成稳定时序。
3. 激活和池化属于后处理链，最好复用片上结果区。

下一篇收束到验证：一个 NPU 设计怎样用分层 testbench 和波形把这些部件验证起来。
