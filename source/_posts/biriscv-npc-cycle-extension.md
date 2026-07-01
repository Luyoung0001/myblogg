---
title: "biriscv 处理器研究：NPC 周期扩增与 fetch 时序"
date: 2025-12-15 15:03:13
tags:
  - "biriscv"
  - "NPC"
  - "microarchitecture"
categories:
  - "Computer Architecture"
---

## biriscv_fetch

biriscv_fetch 是 biRISC-V CPU 的取指单元，位于 ICache 和 Decode 阶段之间，负责：
- 管理 PC (Program Counter)
- 向 ICache 发起取指请求
- 缓冲 ICache 返回的指令
- 处理分支跳转请求
- 支持 MMU（可配置）

下面的分析是没有支持 MMU 的。

### branch 处理

```verilog
begin
    assign branch_w      = branch_q || branch_request_i;
    assign branch_pc_w   = (branch_q & !branch_request_i) ? branch_pc_q   : branch_pc_i;
    assign branch_priv_w = `PRIV_MACHINE; // don't care

    always @ (posedge clk_i or posedge rst_i)
    if (rst_i)
    begin
        branch_q       <= 1'b0;
        branch_pc_q    <= 32'b0;
    end
    else if (branch_request_i && (icache_busy_w || !active_q))
    begin
        branch_q       <= branch_w;
        branch_pc_q    <= branch_pc_w;
    end
    else if (~icache_busy_w)
    begin
        branch_q       <= 1'b0;
        branch_pc_q    <= 32'b0;
    end
end
```

这个时序逻辑处理了 issue 来的反馈信号，issue 可以重定向任何一条指令的下一条指令。换言之，issue 总是能找到本条指令的下一条指令，只要给 decode 的一个 pc，decode 译码后发给 issue，issue 从此就能找到所有正确序列。

因此 branch 信号给前端正确的反馈，甚至给出连续的反馈，因此这里使用了暂存器来暂存一组反馈：

```verilog
    ,input           branch_request_i
    ,input  [ 31:0]  branch_pc_i
    ,input  [  1:0]  branch_priv_i
```
branch_priv_i 无关紧要，这里做了忽略（总是机器模式）。

### 激活

```verilog
//-------------------------------------------------------------
// Active flag
//-------------------------------------------------------------
always @ (posedge clk_i or posedge rst_i)
if (rst_i)
    active_q    <= 1'b0;
else if (SUPPORT_MMU && branch_w && ~stall_w)
    active_q    <= 1'b1;
else if (!SUPPORT_MMU && branch_w)
    active_q    <= 1'b1;
```
当 branch_request_i 拉高或者拉高之后锁存，active_q 下一个周期就会被激活。一旦被激活，fetch 将会出在激活状态。

active_q 是一个"启动开关"：
- 防止复位后用无效的 PC（0x0）去取指；
- 等待外部（通常是 CSR 或复位逻辑）提供真正的启动地址；
- 一旦启动，就一直工作下去。


### 暂停



```txt
  stall_w = !fetch_accept_i || icache_busy_w || !icache_accept_i
                │                   │                  │
                ▼                   ▼                  ▼
           下游反压            等待响应           ICache忙

  | 条件             | 含义                | 场景                            |
  |------------------|---------------------|---------------------------------|
  | !fetch_accept_i  | Decode 不接受新指令 | Decode 阶段有阻塞（如依赖、满） |
  | icache_busy_w    | ICache 请求未返回   | 等待 ICache 响应中              |
  | !icache_accept_i | ICache 不接受请求   | ICache 内部忙（如 miss 处理）   |

  | 信号    | 类型     | 含义                 |
  |---------|----------|----------------------|
  | stall_w | 组合逻辑 | 当前周期是否需要暂停 |
  | stall_q | 寄存器   | 上一周期是否暂停     |

```

```verilog
//-------------------------------------------------------------
// Stall flag
//-------------------------------------------------------------
reg stall_q;

always @ (posedge clk_i or posedge rst_i)
if (rst_i)
    stall_q    <= 1'b0;
else
    stall_q    <= stall_w;
```

stall_q 锁存了当前 fetch 系统的 stall_w 状态，如果 stall_q 为 1，说明本周期继续处理这种 stall 情况。

#### stall_w 的作用 1：控制 PC 更新

```verilog
always @ (posedge clk_i or posedge rst_i)
if (rst_i)
    pc_f_q  <= 32'b0;
// Branch request
else if (SUPPORT_MMU && branch_w && ~stall_w)
    pc_f_q  <= branch_pc_w;
else if (!SUPPORT_MMU && (stall_w || !active_q || stall_q) && branch_w)
    pc_f_q  <= branch_pc_w;
// NPC
else if (!stall_w)
    pc_f_q  <= next_pc_f_i;
```
stall 的时候，pc_f_q 不更新，这样确保不会跳过 PC。

#### stall_w 的作用 2：控制 npc 采样
```verilog
// biriscv_fetch.v:305
assign pc_accept_o = ~stall_w;
```
告诉 NPC 模块：
- pc_accept_o = 1：当前 PC 被接受，可以更新推测状态（RAS、BHT）
- pc_accept_o = 0：暂停，不要更新推测状态

这里停止更新 npc 的 RAS。

#### stall_q 的作用 3：从 stall 的恢复过程

```verilog
assign icache_pc_w       = (branch_w & ~stall_q) ? branch_pc_w : pc_f_q;
```

如果上一周期 stall 了，那么此时 icache_pc_w 保持原来的 pc。

### icache 请求追踪
```verilog
//-------------------------------------------------------------
// Request tracking
//-------------------------------------------------------------
reg icache_fetch_q;

// ICACHE fetch tracking
always @ (posedge clk_i or posedge rst_i)
if (rst_i)
    icache_fetch_q <= 1'b0;
else if (icache_rd_o && icache_accept_i)
    icache_fetch_q <= 1'b1;
else if (icache_valid_i)
    icache_fetch_q <= 1'b0;

assign icache_busy_w       =  icache_fetch_q && !icache_valid_i;
```
```txt
  | icache_fetch_q | icache_valid_i | icache_busy_w | 含义           |
  |----------------|----------------|---------------|----------------|
  | 0              | x              | 0             | 无未完成请求   |
  | 1              | 0              | 1             | 等待响应中，忙 |
  | 1              | 1              | 0             | 响应到达，不忙 |

  时序图

  Cycle        1        2        3        4        5
             ┌────────┬────────┬────────┬────────┬────────┐
  icache_rd  │   1    │   0    │   0    │   1    │   0    │
             ├────────┼────────┼────────┼────────┼────────┤
  icache_    │   1    │        │        │   1    │        │
  accept     │        │        │        │        │        │
             ├────────┼────────┼────────┼────────┼────────┤
  icache_    │        │        │   1    │        │   1    │
  valid      │        │        │resp @1 │        │resp @4 │
             ├────────┼────────┼────────┼────────┼────────┤
  icache_    │  0→1   │   1    │  1→0   │  0→1   │  1→0   │
  fetch_q    │ 发请求 │ 等待中 │ 收响应 │ 发请求 │ 收响应 │
             ├────────┼────────┼────────┼────────┼────────┤
  icache_    │   0    │   1    │   0    │   0    │   0    │
  busy_w     │        │  忙!   │        │        │        │
             └────────┴────────┴────────┴────────┴────────┘
```
如果 icache 命中，那么下一个周期返回，busy 信号是不会拉高的，保证了连续发射请求。如果 icache 没有命中那么 busy 就会在下一个周期拉高，防止发射连续的请求。

### pc 的更新逻辑

```verilog

//-------------------------------------------------------------
// PC
//-------------------------------------------------------------
reg [31:0]  pc_f_q;

always @ (posedge clk_i or posedge rst_i)
if (rst_i)
    pc_f_q  <= 32'b0;
// Branch request
else if (SUPPORT_MMU && branch_w && ~stall_w)
    pc_f_q  <= branch_pc_w;
else if (!SUPPORT_MMU && (stall_w || !active_q || stall_q) && branch_w)
    pc_f_q  <= branch_pc_w;
// NPC
else if (!stall_w)
    pc_f_q  <= next_pc_f_i;

wire [31:0] icache_pc_w;
wire [1:0]  icache_priv_w;
wire        fetch_resp_drop_w;

assign icache_pc_w       = (branch_w & ~stall_q) ? branch_pc_w : pc_f_q;
assign icache_priv_w     = `PRIV_MACHINE; // Don't care
assign fetch_resp_drop_w = branch_w;
```

pc_f_q 的更新逻辑是如果遇见 stall 但是此时有 branch，直接更新（这是一种激进策略）；或者没有 stall 的话，采样 npc 返回值；其他情况保持。

如果当前周期有 branch_w 信号，fetch_resp_drop_w 会拉高，这样前面请求的指令在这一周期直接无效化：

```verilog
assign fetch_valid_o       = (icache_valid_i || skid_valid_q) & !fetch_resp_drop_w;
```

### last fetch address

```verilog
// Last fetch address
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

pc_d_q 是用来追踪最后发射给 icache 请求且已经成功的 pc。这样，pc_d_q 就能和当前周期返回的 icache 数据对齐。


### Response Buffer

```verilog
//-------------------------------------------------------------
// Response Buffer
//-------------------------------------------------------------
reg [99:0]  skid_buffer_q;
reg         skid_valid_q;

always @ (posedge clk_i or posedge rst_i)
if (rst_i)
begin
    skid_buffer_q  <= 100'b0;
    skid_valid_q   <= 1'b0;
end
// Instruction output back-pressured - hold in skid buffer
else if (fetch_valid_o && !fetch_accept_i)
begin
    skid_valid_q  <= 1'b1;
    skid_buffer_q <= {fetch_fault_page_o, fetch_fault_fetch_o, fetch_pred_branch_o, fetch_pc_o, fetch_instr_o};
end
else
begin
    skid_valid_q  <= 1'b0;
    skid_buffer_q <= 100'b0;
end
```

如果 fetch_valid_o 有效，但是 decode 已经不需要了，这时候缓冲这些数据。



