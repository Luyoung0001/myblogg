---
title: biriscv 处理器 npcg 设计
date: 2025-12-22 15:03:13
tags: 超标量处理器设计
categories: 计算机体系结构
mermaid: true
---

## 前言

biriscv 的 issue 单元一旦接受到错误的指令，将会把错误的信号，包括目标 pc 发射到前端，提示前端取这个 pc 对应的指令。

前端需要把这个 pc 发射给 icache 单元，返回数据后，将这个 pc 的预测信息 next_taken 和 pc、inst 对齐，一块儿发给 decode 单元，实际上这就是 fetch 单元在做的事情。

但是一旦 npc 做成了两个周期，为了适配 npc 的工作，并能将 npc 流水线化，就得构思出一种架构。

boom 的做法是，在 F0 阶段将 pc 一起发射到 icache、bpd 中，同时 bpd 的单周期预测器立即给出预测信息并锁存以供下一个周期使用：

```scala
  // faubtb.scala / predictor.scala
  val s0_idx = fetchIdx(io.f0_pc)
  val s1_idx = RegNext(s0_idx)    // ← 关键：寄存器打断组合路径
  // frontend.scala:418-425：
  when (s1_valid) {
    s0_valid     := true.B
    s0_tsrc      := BSRC_1        // 标记预测来源是 F1 阶段
    s0_vpc       := f1_predicted_target  // 下一拍的 PC
    s0_ghist     := f1_predicted_ghist
  }
```

boom 的单周期预测器这样做很自然，但是我要在 biriscv 中将 npc 改成 2 周期预测器，这就很不自然了。

## biriscv 中 fetch 中的数据对齐

### T0

处理器刚启动的时候，biriscv 中的 branch_request_i 拉高，同时会有一个 vec_pc（branch_pc_w） 输送到 pc_f_q。与此同时，active_q 也将在下一个周期被拉高。

同时通往 npc 的输出信号 pc_f_o，pc_accept_o 也会有值，npc 会在本周期立即给出预测信息。

### T1

到了下一个周期的时候，active_q 被激活，pc_f_q 也锁存了有效数据。此时 icache 就可以访问内存了：

```verilog
assign icache_rd_o         = active_q & fetch_accept_i & !icache_busy_w;
assign icache_pc_o         = {icache_pc_w[31:3],3'b0};
```

与此同时，为了将icache 返回的数据对齐发送到 decode 模块，这里要对 icache 的请求信号进行保存：

```verilog
always @ (posedge clk_i or posedge rst_i)
if (rst_i)
    pc_d_q <= 32'b0;
else if (icache_rd_o && icache_accept_i)
    pc_d_q <= icache_pc_w;

always @ (posedge clk_i or posedge rst_i)
if (rst_i)
    pred_d_q <= 2'b0;
else if (icache_rd_o && icache_accept_i)
    pred_d_q <= next_taken_f_i;
else if (icache_valid_i)
    pred_d_q <= 2'b0;
```

pred_d_q 保存的是 vec_pc 的预测信息，如何得到这个信息呢？这个信息实际上是将 vec_pc 送到 npc 中得到的。也就是说，输入信息 {`vec_pc`}，输出信息 {`vec_pc+8`,`next_taken`}。可以看出，需要对齐的信息是 `vec_pc、next_taken`。


### T2

此时，数据从 icache 返回，如果没有返回，说明 icache 未命中，只需要等待几个周期即可。返回的时候，pc_d_q、pred_d_q 已经准备好，这时候和 icache 返回的数据 instr 对齐发射到 decode 单元。

对齐的数据是：
- vec
- vec 作为 npc 输入得到的 taken 信息
- vec 作为 icache 的返回数据

biriscv 中的做法就是将 vec 输入到 npc，本周期返回的 taken 数据直接用，没有跨周期，这很容易实现。但是要将 npc 做成双周期的，就得设计一下各个信号的时序了。

biriscv 的性能如下：

```bash
CoreMark/MHz: 4.109206

========================================
   Branch Predictor Statistics
========================================

--- Overall Statistics ---
Total Branches:      66423
Total Predictions:   66423

--- Direction Prediction ---
Correct:             59188
Incorrect:           7235
Accuracy:            89.107689%
```
改造成功的标志应该是分支预测正确率和 coremark 分数应该相差很小。

## npcg
我的做法是，给 fetch 单元加了新的一级流水线 npcg，意思就是专门用于产生 next_pc 的单元。这其实是将 fetch 单元的 pc 产生单元解耦到 npcg 中了。

### T0
当 branch_request_i 信号拉高的时候，将 branch_pc_i 的数据送进 npc 以及 pc_r，此时 fetch 单元不接收数据。

### T1
此时 pc_r 已经将 pc 锁存，并通过组合电路将锁存器的 pc 拉到 fetch 单元，而且此时 vec 作为 npc 输入信号的输出信号 vec+8，next_taken 已经输出。

这些输出信号继续存入到锁存器：

```verilog
reg [1:0] next_taken_f_i_r;
always @(posedge clk_i or posedge rst_i) begin
    if (rst_i) begin
        pc_r <= 32'd0;
    end
    else if (fetch_allowin && (active_q || branch_request_i)) begin
        pc_r <= pc_r_q;
    end
end
```
对于 npc，输出的信号继续进去 npc 进行预测。

### T2

这个周期，fetch 单元就接收数据了：

```verilog
// biriscv_npcg
assign npcg_valid = !first_active && active_q && !just_branch;
assign fetch_pc_o = pc_r;
assign next_taken_o = next_taken_f_i_r;

// biriscv_fetch
always @(posedge clk_i or posedge rst_i) begin
    // branch_request_i 拉高，意味着现在的所有过程都是徒劳的，需要刷掉
    if (rst_i || branch_request_i) begin
        fetch_valid_r <= 1'b0;
    end
    else if (fetch_allowin) begin
        fetch_valid_r <= npcg_valid;
    end
    // branch_request_i 时不更新，因为此时 npcg 的数据还是旧的
    if (npcg_valid && fetch_allowin && !branch_request_i) begin
        next_pc_f_i_r <= next_pc_f_i;
        next_taken_f_i_r <= next_taken_f_i;
    end
end
```

### T3
这一个周期 fetch 的指令缓存已经保存好了 npcg 发射的信息，包括一个 pc，以及和这个 pc 一起出现的 next_taken。

因此这里就有一个问题，next_taken 应该和进入 npc 的信号 pc 对齐，不应该和 next_taken 一起输出的 pc 对齐。

这里应该怎么做呢？这要从 npcg 说起。

将 branch_pc_i 输出到 npc，这样就能获取 branch_pc_i 作为输入的 next_taken。同时，将 branch_pc_i 输入到 pc_r，到了下一个周期就得到了：
- pc_r：branch_pc_i
- branch_pc_i 作为 npc 输入的输出 next_taken 信息

同时 npc 也立即输入了 branch_pc_i + 8。

同时将这两个信号直接送给 fetch 单元，这样自然就是对齐的。

运行了一下，性能明显下降：

```bash
CoreMark/MHz: 2.950479

========================================
   Branch Predictor Statistics
========================================

--- Overall Statistics ---
Total Branches:      66438
Total Predictions:   66438

--- Direction Prediction ---
Correct:             58357
Incorrect:           8081
Accuracy:            87.836780%
```










