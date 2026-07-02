---
title: "CoolDA 设计仿真（二）：APB 外设与 4x4 INT8 矩阵乘核"
date: 2026-05-08 15:03:13
tags:
  - "CoolDA"
  - "APB"
  - "RTL"
  - "MMIO"
categories:
  - "Computer Architecture"
---

## 前言

CPU 眼里的 NPU，不是一堆乘法器，而是一组寄存器。只要寄存器契约设计清楚，软件就能驱动硬件；只要契约设计混乱，硬件再能算也很难用。

这一篇拆 CoolDA 的 APB NPU：寄存器怎么排，A/B 数据怎么打包，start/busy/done 如何工作，4x4 乘法核如何完成计算。

![CoolDA APB 外设路径](/images/mermaid-svg/coolda-design-02-apb-npu/apb-npu.svg)

## APB 外设的基本形状

一个 APB 外设大致会接收这些信号：

```systemverilog
input  wire        apb_psel;
input  wire        apb_enab;
input  wire        apb_rw;       // 1 = write, 0 = read
input  wire [19:0] apb_addr;
input  wire [31:0] apb_datai;
output reg  [31:0] apb_datao;
output wire        apb_ack;
```

读写动作可以简化成：

```systemverilog
reg_sel  = apb_addr[7:2];              // word offset
write_en = apb_psel && apb_enab && apb_rw;
read_en  = apb_psel && apb_enab && !apb_rw;
apb_ack  = apb_enab;
```

CPU 访问的是 byte address，硬件寄存器通常按 32 位 word 对齐，所以 `addr[7:2]` 正好用来选择寄存器。

## 寄存器表

CoolDA 的寄存器可以设计成这样：

![CoolDA 寄存器映射](/images/mermaid-svg/coolda-design-02-apb-npu/register-map.svg)

| 偏移 | 名称 | 方向 | 含义 |
|---|---|---|---|
| `0x00` | `CTRL` | 写 | start、relu、clear |
| `0x04` | `STATUS` | 读 | busy、done |
| `0x08` | `INFO` | 读 | 硬件标识，比如 `"C4x4"` |
| `0x10` | `A0` | 写/读 | A 第 0 行，4 个 int8 |
| `0x14` | `A1` | 写/读 | A 第 1 行 |
| `0x18` | `A2` | 写/读 | A 第 2 行 |
| `0x1C` | `A3` | 写/读 | A 第 3 行 |
| `0x20` | `B0` | 写/读 | B 第 0 行 |
| `0x24` | `B1` | 写/读 | B 第 1 行 |
| `0x28` | `B2` | 写/读 | B 第 2 行 |
| `0x2C` | `B3` | 写/读 | B 第 3 行 |
| `0x40~0x7C` | `C00~C33` | 读 | C 的 16 个 int32 输出 |

控制位可以定义成：

```c
#define CTRL_START  (1u << 0)
#define CTRL_RELU   (1u << 1)
#define CTRL_CLEAR  (1u << 2)

#define STATUS_BUSY (1u << 0)
#define STATUS_DONE (1u << 1)
```

这张表就是软硬件之间的 ABI。软件和 RTL 必须完全一致。

## 一个 32 位寄存器如何装 4 个 int8

A 和 B 是 4x4 INT8 矩阵。每一行有 4 个元素，刚好可以打包进一个 32 位寄存器：

```text
bits [7:0]    lane0
bits [15:8]   lane1
bits [23:16]  lane2
bits [31:24]  lane3
```

BSP 侧打包：

```c
static uint32_t pack_row(const int8_t row[4]) {
  return ((uint32_t)(uint8_t)row[0]) |
         ((uint32_t)(uint8_t)row[1] << 8) |
         ((uint32_t)(uint8_t)row[2] << 16) |
         ((uint32_t)(uint8_t)row[3] << 24);
}
```

RTL 侧拆 lane：

```systemverilog
function [7:0] get_lane;
  input [31:0] word;
  input integer lane;
  case (lane)
    0: get_lane = word[7:0];
    1: get_lane = word[15:8];
    2: get_lane = word[23:16];
    3: get_lane = word[31:24];
    default: get_lane = 8'h00;
  endcase
endfunction
```

这两个函数必须互相匹配。否则硬件乘法器本身没错，结果也会错。

## start / busy / done 协议

最小控制流程是：

```text
1. 软件写 A0~A3
2. 软件写 B0~B3
3. 软件写 CTRL.START
4. 硬件 busy = 1, done = 0
5. 硬件计算
6. 硬件 busy = 0, done = 1
7. 软件读 C00~C33
```

RTL 的启动逻辑可以写成：

```systemverilog
if (clear_req) begin
  busy <= 1'b0;
  done <= 1'b0;
end else if (start_req && !busy) begin
  busy <= 1'b1;
  done <= 1'b0;
  relu_enable <= ctrl_write_data[1];
  k_idx <= 2'd0;
  clear_accumulators();
end
```

软件侧轮询可以写成：

```c
int wait_done(unsigned timeout) {
  while (timeout--) {
    uint32_t status = read_reg(STATUS);
    if (status & STATUS_DONE) return 0;
  }
  return -1;
}
```

这是一种很原始但很清晰的协处理器协议。

## 4x4 乘法核

硬件核内部做的是：

```text
for k in 0..3:
  for row in 0..3:
    for col in 0..3:
      acc[row][col] += A[row][k] * B[k][col]
```

RTL 风格可以写成：

```systemverilog
if (busy) begin
  for (row = 0; row < 4; row++) begin
    for (col = 0; col < 4; col++) begin
      acc[row][col] <= acc[row][col] +
        $signed(get_lane(a_regs[row], k_idx)) *
        $signed(get_lane(b_regs[k_idx], col));

      if (k_idx == 2'd3) begin
        c_regs[row][col] <= relu_enable
          ? relu32(acc_next[row][col])
          : acc_next[row][col];
      end
    end
  end

  if (k_idx == 2'd3) begin
    busy <= 1'b0;
    done <= 1'b1;
  end else begin
    k_idx <= k_idx + 1'b1;
  end
end
```

这里的 `relu32()` 很简单：

```systemverilog
function signed [31:0] relu32;
  input signed [31:0] value;
  relu32 = value[31] ? 32'sd0 : value;
endfunction
```

注意：如果把这个 4x4 核用于大矩阵 tiled matmul，通常不能在每个 partial tile 后立即 ReLU。ReLU 应该在所有 K tile 累加完成以后再做，否则会改变数学语义。

## 读结果窗口

C 的 16 个 int32 输出可以展开成 16 个寄存器：

```text
C00 C01 C02 C03
C10 C11 C12 C13
C20 C21 C22 C23
C30 C31 C32 C33
```

BSP 读回时按 row-major 顺序：

```c
for (int row = 0; row < 4; row++) {
  for (int col = 0; col < 4; col++) {
    c[row][col] = read_reg(C_BASE + (row * 4 + col) * 4);
  }
}
```

这里同样有 layout 合约：RTL 怎么展开 C，软件就必须怎么读。

## 小结

一个 APB NPU 的本质是寄存器契约：

```text
输入寄存器 A/B
  -> CTRL.START
  -> STATUS.BUSY/DONE
  -> 输出寄存器 C
```

只要这张表清楚，CPU 就能使用硬件。下一篇继续往上看：BSP 如何把这张表包装成函数，runtime 又如何把 4x4 小核调度成更大的矩阵任务。
