---
title: biriscv 处理器 npc 的周期扩增
date: 2025-12-015 15:03:13
tags: 超标量处理器设计
categories: 计算机体系结构
mermaid: true
---

## fetch

fetch 单元在周期 T 向 npc 发射请求信号包括一个 pc 信号（pc_f_q），它能在同一周期 T 内获取到 next_pc 在下一个周期更新 pc_f_q。同时利用该 pc_f_q 向 icache 发出请求，在周期 T+1 拿到数据后将数据传送给 decode 单元。

fetch 的接口 io 如下：

```verilog
module biriscv_fetch
//-----------------------------------------------------------------
// Params
//-----------------------------------------------------------------
#(
     parameter SUPPORT_MMU      = 1
)
//-----------------------------------------------------------------
// Ports
//-----------------------------------------------------------------
(
    // Inputs
     input           clk_i
    ,input           rst_i
    ,input           fetch_accept_i
    ,input           icache_accept_i
    ,input           icache_valid_i
    ,input           icache_error_i
    ,input  [ 63:0]  icache_inst_i
    ,input           icache_page_fault_i
    ,input           fetch_invalidate_i
    ,input           branch_request_i
    ,input  [ 31:0]  branch_pc_i
    ,input  [  1:0]  branch_priv_i
    ,input  [ 31:0]  next_pc_f_i
    ,input  [  1:0]  next_taken_f_i

    // Outputs
    ,output          fetch_valid_o
    ,output [ 63:0]  fetch_instr_o
    ,output [  1:0]  fetch_pred_branch_o
    ,output          fetch_fault_fetch_o
    ,output          fetch_fault_page_o
    ,output [ 31:0]  fetch_pc_o
    ,output          icache_rd_o
    ,output          icache_flush_o
    ,output          icache_invalidate_o
    ,output [ 31:0]  icache_pc_o
    ,output [  1:0]  icache_priv_o
    ,output [ 31:0]  pc_f_o
    ,output          pc_accept_o
);
```

如果将 npc 进行扩增一个周期，这时候就必须思考如何将 fetch 单元也扩增一个周期以对齐原来的io数据。如果将 fetch 单元进行重写设计，这将会带来很大的复杂度，甚至对 frontend 都要进行修改。

而我们只想要让 fetch 单元提前发出 npc 请求信号，并在下一个周期返回 npc 并和原来的 fetch io数据对齐。思路就是针对这两个信号向前分裂出一级流水线：

```verilog
    ,output [ 31:0]  pc_f_o
    ,output          pc_accept_o
```

新的一级流水线做的事情很简单，就是发出这两个信号。这两个信号需要在 fetch0 （原来 fetch 的上一拍）发出来，得考虑以下两个问题：

- pc_f_o、pc_accept_o 和哪些输入信号相关；
- 返回的数据 next_pc_f_i、next_taken_f_i 和哪些数据要对齐。

针对第一个问题，直接查看 fetch 单元就行，思路很简单就是复制一个 biriscv_fetch.v 把不相干的信号全部删除。

针对第二个问题，查看 fetch 单元，发现这三个信号不仅和 pc_f_o、pc_accept_o 相关，而且还要和 next_pc_f_i、next_taken_f_i 对齐：

```verilog
    ,input           branch_request_i
    ,input  [ 31:0]  branch_pc_i
    ,input  [  1:0]  branch_priv_i
```

因此在 fetch0 中利用它们发射 npc 请求，并对它加拍，这样就能在 fetch1 中形成和原 fetch 单元**完全等价的信号组**。这就是我们的目的———**形成等价的信号组**。

换句话说，我们做的一切工作，目的就是要提前一个周期发射 npc 请求并保持 fetch1 和原 fetch 单元完全等价，这样对其它部件不会造成任何影响。

开辟一个新的流水线 fetch0，需要关注 fetch0 的数据通路。fetch0 要发射 pc_f_o、pc_accept_o 信号，需要下游的一些信号：

```verilog
    ,input           fetch_accept_i
    ,input           icache_accept_i
    ,input           icache_valid_i
    ,input           fetch_invalidate_i
```

这些信号是下游的数据旁路，提前从下游 fetch1 传递上来配合 fetch0 发射 npc 请求。

fetch0 是流水线的某一级，它的流水线输入信号是：

```verilog
    ,input           branch_request_i
    ,input  [ 31:0]  branch_pc_i
    ,input  [  1:0]  branch_priv_i
```

流水线输出信号是：

```verilog
    ,output [ 31:0]  pc_f_o
    ,output          pc_accept_o
    ,output reg         branch_request_r
    ,output reg [ 31:0] branch_pc_r
    ,output reg [  1:0] branch_priv_r
```

## npc

由于 fetch 改成了 fetch0、fetch1 的流水线结构，因此 npc 也必须改成流水线结构。

在每一个周期 fetch0 都会发出 npc 请求，npc 中必须有流水线结构来存储新的请求。与此同时，返回信号也必须配合 npc 流水线更新，比如 npc 在 T 周期发出了读请求，与此同时反馈信号拉高要写 sram，这时候就应该 bypass 到读请求以检查是否和写数据 hit；甚至在 T+1 周期返回的时候，反馈信号也可能拉高，这时候返回的 npc 也要 mux 一下。

还要检查读写端口是否冲突，如果冲突，就把写先暂存到 buffer 中。

这里的流水线相当复杂，要好好思考设计问题，我再看看 boom 是如何处理的。














