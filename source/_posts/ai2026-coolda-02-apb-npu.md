---
title: "CoolDA 设计仿真笔记（二）：APB 外设与 4x4 INT8 矩阵乘核"
date: 2026-05-08 15:03:13
tags:
  - "AI2026"
  - "CoolDA"
  - "APB"
  - "RTL"
categories:
  - "Computer Architecture"
---

## 前言

CoolDA 的硬件核很小，但它讲清楚了一个特别重要的接口问题：CPU 看到的不是“矩阵乘法器内部细节”，而是一组可读写的寄存器。APB 外设的本质就是把硬件能力包装成寄存器契约。

这一篇从 `coolda_npu.v` 和 BSP 头文件出发，看一个 4x4 INT8 matmul 小核如何被暴露给 CPU。

![CoolDA APB 外设路径](/images/mermaid-svg/ai2026-coolda-02-apb-npu/apb-npu.svg)

## APB 接口先定义访问动作

`coolda_npu.v` 的端口很克制：时钟、复位、APB 读写方向、片选、使能、地址、写数据、读数据和 ack，见 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:1` 到 `:13`。

读写动作由三个内部信号定义：

1. `write_en = apb_psel && apb_enab && apb_rw`
2. `read_en = apb_psel && apb_enab && !apb_rw`
3. `reg_sel = apb_addr[7:2]`

这些逻辑在 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:51` 到 `:56`。

这说明寄存器文件不是随意 if/else。APB 地址先被规整到 word offset，然后读写逻辑按 offset 分发。CPU 访问基地址加偏移，最终在 RTL 里变成 `reg_sel`。

## 寄存器契约是什么

CoolDA 的寄存器非常适合教学：

![CoolDA 寄存器映射](/images/mermaid-svg/ai2026-coolda-02-apb-npu/register-map.svg)

- `REG_CTRL`：控制启动、ReLU、清零。
- `REG_STATUS`：返回 busy/done。
- `REG_INFO`：返回 `"C4x4"` 信息字。
- `REG_A0..A3`：输入矩阵 A 的四行。
- `REG_B0..B3`：输入矩阵 B 的四行。
- `REG_C_BASE`：输出矩阵 C 的 16 个 int32 结果窗口。

RTL 里的寄存器 offset 定义在 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:15` 到 `:28`。BSP 头文件用 CPU 侧偏移重新表达同一件事，见 `AI2026/coolda-soc/bsp/include/coolda.h:8` 到 `:19`。

这就是软硬件共同维护的 ABI。RTL 改了 offset，BSP 不改，软件就会往错地方写；BSP 改了 offset，RTL 不改，硬件就收不到任务。

## 32 位行寄存器为什么能放 4 个 int8

`coolda_npu` 的 A/B 输入不是 16 个独立 8 位寄存器，而是 4 个 32 位行寄存器。每个 32 位 word 里打包 4 个 INT8 lane。

RTL 用 `get_lane()` 从 32 位 word 里取第 0、1、2、3 个 lane，逻辑在 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:58` 到 `:70`。

BSP 侧则用 `pack_row()` 把四个 `int8_t` 打包成一个 `uint32_t`，见 `AI2026/coolda-soc/bsp/src/coolda.c:15` 到 `:20`。

这对函数正好构成一个数据格式合约：

1. BSP 规定 lane 顺序。
2. RTL 按同样顺序拆 lane。
3. 矩阵乘法的行列语义建立在这个 lane 顺序上。

这类小合约非常重要。很多硬件加速器 bug 不在乘法器，而在 pack/unpack、大小端、stride 和 transpose 上。

## start、busy、done 是最小控制协议

CoolDA 的启动协议由 `CTRL` 和 `STATUS` 两个寄存器完成。

BSP 头文件定义了控制位：`START`、`RELU`、`CLEAR`，见 `AI2026/coolda-soc/bsp/include/coolda.h:21` 到 `:23`；状态位是 `BUSY` 和 `DONE`，见同文件 `:25` 到 `:26`。

BSP 的 `bsp_coolda_start()` 会根据参数设置 start 和可选 ReLU bit，再写入 `CTRL`，见 `AI2026/coolda-soc/bsp/src/coolda.c:48` 到 `:54`。`bsp_coolda_wait_done()` 则轮询 `STATUS`，直到看到 done 或超时，见同文件 `:56` 到 `:65`。

RTL 侧在检测到 `start_req && !busy` 后拉高 busy、清掉 done、锁存 ReLU，并把 `k_idx` 清零，见 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:149` 到 `:160`。当 `k_idx == 3`，硬件拉低 busy、拉高 done，见同文件 `:193` 到 `:198`。

这就是最小硬件协处理器协议：

```text
write inputs
write START
poll DONE
read outputs
```

它很原始，但也很清楚。

## 4x4 matmul 的 RTL 主循环

真正的乘加发生在 `busy` 分支里。RTL 对 `row` 和 `col` 双层循环，每个位置计算：

```text
acc[row][col] += A[row][k_idx] * B[k_idx][col]
```

对应代码在 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:161` 到 `:175`。当 `k_idx` 到最后一项时，结果写入 `c_regs[row][col]`，如果 `relu_enable` 打开就经过 `relu32()`，见同文件 `:176` 到 `:189`。

`relu32()` 本身也很直白：负数变 0，非负保持原值，见 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:72` 到 `:81`。

从硬件角度看，这个核是一个固定小 kernel：

1. 输入：A 的 4 行、B 的 4 行。
2. 内部：`k_idx = 0..3` 的四拍累加。
3. 输出：C 的 16 个 32 位结果。
4. 可选：最后写 C 时做 ReLU。

## 读路径如何返回状态和结果

APB 读 mux 在 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:83` 到 `:116`。`STATUS` 返回 `{done, busy}`，`INFO` 返回 `"C4x4"`，A/B 行寄存器可以读回，C 结果窗口从 `6'h10` 到 `6'h1f` 展开成 16 个位置。

BSP 的 `bsp_coolda_read_matrix_c()` 按 row/col 遍历这 16 个结果寄存器，见 `AI2026/coolda-soc/bsp/src/coolda.c:67` 到 `:77`。

这让读者能看清一个完整闭环：

1. 软件 pack A/B。
2. BSP 写 A/B 寄存器。
3. RTL 用 lane 做乘加。
4. RTL 写 C 寄存器。
5. BSP 读 C 窗口并还原成矩阵。

## 当前硬件核的真实边界

这个硬件核没有 DMA，没有内部大 buffer，没有命令 FIFO，也没有中断完成。README 里也明确写了当前没有 DMA、命令队列、中断完成和真正后台硬件 async，见 `AI2026/coolda-soc/README.md:153` 到 `:161`。

但它恰好适合讲“寄存器级加速器”的本质。很多真实 SoC 里的小外设也是这样开始的：先把功能挂成 MMIO 外设，跑通软件栈，再逐步加入 DMA、队列、中断和更复杂的内存模型。

## 这篇的核心结论

CoolDA NPU 的硬件契约可以概括成四句话：

1. APB 地址被解码成寄存器 offset。
2. A/B 行寄存器用 32 位 word 打包 4 个 int8。
3. `START/BUSY/DONE` 构成最小启动协议。
4. 4x4 矩阵乘核用 `k_idx` 循环完成 int8 到 int32 的乘加。

下一篇我们往上一层：BSP 如何把寄存器细节包起来，runtime 又如何把 4x4 小核调度成更大的 tiled matmul。
