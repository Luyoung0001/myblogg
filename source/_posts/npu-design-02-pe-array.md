---
title: "NPU 设计（二）：PE 与阵列如何展开矩阵乘法"
date: 2026-04-29 15:03:13
tags:
  - "NPU"
  - "systolic array"
  - "RTL"
  - "matrix multiplication"
categories:
  - "Computer Architecture"
---

## 前言

NPU 的核心计算单元叫 PE，也就是 Processing Element。一个 PE 并不神秘，它做的事情通常就是一次乘加：

```text
y_out = y_in + a * b
```

难点不在单个 PE，而在怎样组织成阵列。矩阵乘法的每个输出都需要沿着 K 维累加，如果让每个 PE 自己去取完整数据，带宽会爆炸；如果让数据按规律流过阵列，同一份 A、B 数据就能被多个 PE 复用。

![PE 与阵列的数据复用](/images/mermaid-svg/npu-design-02-pe-array/pe-systolic-array.svg)

## 单个 PE 的完整合约

一个 PE 的输入输出可以画成这样：

![单 PE 的乘加合约](/images/mermaid-svg/npu-design-02-pe-array/single-pe-contract.svg)

它至少需要这些信号：

| 信号 | 作用 |
|---|---|
| `a` | INT8 激活值 |
| `b` | INT8 权重值 |
| `y_in` | 已有 INT32 累加值 |
| `valid_in` | 本拍输入是否有效 |
| `clear_acc` | 是否清空累加 |
| `y_out` | 新的 INT32 累加值 |
| `valid_out` | 输出是否有效 |

简化 RTL 可以写成：

```systemverilog
wire signed [7:0]  a_s = a;
wire signed [7:0]  b_s = b;
wire signed [15:0] product = a_s * b_s;
wire signed [31:0] product_ext = {{16{product[15]}}, product};

always_ff @(posedge clk) begin
  if (!rst_n || clear_acc) begin
    y_out     <= 32'sd0;
    valid_out <= 1'b0;
  end else begin
    y_out     <= y_in + product_ext;
    valid_out <= valid_in;
  end
end
```

这里有三个细节非常重要：

1. `a` 和 `b` 必须按有符号数解释。
2. 16 位乘积要符号扩展到 32 位。
3. `valid_out` 通常会比 `valid_in` 晚一拍。

很多 PE bug 都出在这些“看起来很小”的地方。

## 矩阵乘法可以怎样拆

矩阵乘法公式是：

```text
C[i][j] = sum_k A[i][k] * B[k][j]
```

如果我们固定某一个 `k`，就能得到一组外积：

```text
A[:, k] 是 A 的第 k 列
B[k, :] 是 B 的第 k 行

本轮贡献：
C += A[:, k] x B[k, :]
```

例如 4x4 矩阵乘法可以按 `k = 0, 1, 2, 3` 分四轮：

```text
第 0 轮：C += A[:,0] x B[0,:]
第 1 轮：C += A[:,1] x B[1,:]
第 2 轮：C += A[:,2] x B[2,:]
第 3 轮：C += A[:,3] x B[3,:]
```

这样设计阵列时，每一轮只需要把 A 的一列和 B 的一行送进去。阵列里的每个 PE `(row, col)` 计算：

```text
C[row][col] += A[row][k] * B[k][col]
```

## 4x4 阵列的一拍计算

假设某一拍输入：

```text
A[:,k] = [a0, a1, a2, a3]^T
B[k,:] = [b0, b1, b2, b3]
```

那么 4x4 PE 阵列这一拍会产生 16 个乘积：

```text
PE(0,0): C00 += a0*b0
PE(0,1): C01 += a0*b1
PE(0,2): C02 += a0*b2
PE(0,3): C03 += a0*b3

PE(1,0): C10 += a1*b0
PE(1,1): C11 += a1*b1
...
PE(3,3): C33 += a3*b3
```

四个 `k` 全部走完以后，C 矩阵才算完整。

这种外积式数据流的优点是数据复用非常直观：

- `a0` 被第 0 行所有 PE 使用；
- `b0` 被第 0 列所有 PE 使用；
- 每个 PE 本地保存自己的 partial sum。

## 阵列 RTL 的核心形状

一个教学级阵列可以这样写：

```systemverilog
for (row = 0; row < ARRAY_ROWS; row++) begin
  for (col = 0; col < ARRAY_COLS; col++) begin
    always_ff @(posedge clk) begin
      if (!rst_n || clear_acc) begin
        acc[row][col] <= 32'sd0;
      end else if (a_valid && b_valid) begin
        acc[row][col] <= acc[row][col]
          + signed(a_storage[row]) * signed(b_storage[col]);
      end
    end
  end
end
```

这里 `a_storage[row]` 保存当前 A 列的第 `row` 个元素，`b_storage[col]` 保存当前 B 行的第 `col` 个元素。所有 PE 在同一拍使用同一组 `k` 的数据。

这不是唯一的数据流。真实 NPU 还会使用更复杂的 weight-stationary、output-stationary、row-stationary 或 systolic wavefront 方案。但对入门来说，外积数据流非常适合解释“矩阵乘法如何变成阵列并行”。

## 数据 layout 比公式更容易出错

矩阵公式只有一行，数据 layout 却能让硬件悄悄算错。

假设一个 32 位 word 打包 4 个 INT8：

```text
word[7:0]    = lane0
word[15:8]   = lane1
word[23:16]  = lane2
word[31:24]  = lane3
```

那么软件写入、片上 buffer 读取、阵列拆 lane，都必须遵守同一个顺序。否则 A 的一列可能被当成一行，或者第 0 个元素被当成第 3 个元素。

写 NPU 时，下面这些问题要显式回答：

1. A 是按行送，还是按列送？
2. B 是按行送，还是按列送？
3. 一个 32 位 word 里 lane 顺序是什么？
4. 阵列输出按 row-major 还是 column-major 展开？

这些不是文档细节，而是硬件接口的一部分。

## 为什么小阵列测试很重要

即使目标是 16x16 阵列，也应该先用 4x4 测。原因很简单：

1. 4x4 可以手算。
2. 波形更容易看。
3. 数据 layout 错误更容易暴露。
4. 单个 PE 和阵列调度能分层验证。

比如使用：

```text
A = [1 2 3 4]   B = 单位矩阵
    [5 6 7 8]
    [9 ...  ]
```

如果 `B` 是单位矩阵，理论上 `C = A`。这类输入非常适合验证 A/B layout 和输出顺序，因为正确答案一眼就能看出来。

## 小结

PE 是最小计算合约，阵列是这个合约的空间展开。

这一篇的核心是三句话：

1. 单 PE 做 `y_in + a*b`，但要处理符号扩展、累加宽度和 valid 时序。
2. 外积数据流把矩阵乘法拆成多轮 `A[:,k] x B[k,:]`。
3. 数据 layout 是硬件正确性的一部分，不能只看数学公式。

下一篇进入 Unified Buffer：如果没有片上数据复用，再多 PE 也只能饿着等数据。
