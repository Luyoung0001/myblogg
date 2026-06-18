---
title: "LA 挑战赛记录（4）：五级流水线实现与握手阻塞"
date: 2025-04-15 13:03:13
tags:
  - "LoongArch"
  - "pipeline"
  - "LA challenge"
categories:
  - "Computer Architecture"
---

## 阻塞技术解决相关引发的冲突

### 概述

阻塞技术，以 5 级流水线为例，就是将这个 5 个模块设计成单独的模块。比如对于 IDU，IFU 就是它的上游，EXU 就是它的下游。IDU 的数据来自上游，同时它要给下游 EXU 提供数据。

阻塞技术会使得每一个模块都会相互解耦，成了一个单独的模块。

模块之间如何握手呢？

设想一下，对于 IDU 而言，假设它遇到了点事情，需要阻塞，那么就会通知 IFU 不要发数据了，同时通知 EXU IDU 还没有准备好。因此，这是一个状态机。各个模块中控制握手的所有的触发器组成了整个 CPU 的状态。

事实上，参考 loongson 的开源库中的一个实现，在 [设计文档](https://gitee.com/loongson-edu/open-la500/blob/9f42fa69b7210693ba3aa663fce33a78de3824b8/doc/%E8%AE%BE%E8%AE%A1%E6%A6%82%E8%BF%B0.md) 的最后有这样一段话：

流水级之间的交互逻辑中，需要维护以下关键信号：

- `stage_valid`：由触发器维护，表示当前流水级是否在处理指令。
- `stage_ready_go`：表示流水级是否需要被阻塞。
- `stage_allowin`：表示当前流水级是否允许上一级流水进入，该信号传递至上一级，与上一级流水交互，两种情况下该信号置起，当前流水无正在处理的指令；或者当前流水级中的指令已处理完毕且下一级流水允许进入。该信号会由最后一级流水向前传递，只要某一流水级阻塞，其`stage_allowin` 拉低，同一时刻，前面所有流水级的 `stage_allowin` 都会拉低（假设所有流水级中都有指令在处理）。
- `stage_to_nextstage_valid`：表示当前流水级中是否有处理完毕的指令，该信号传递至下一级，与下一级流水交互。下一级流水在收到高电平信号时，且其下一级流水的 `stage_allowin` 为高电平，则会在下一拍置起下一级流水的 `stage_valid`，实现流水级间指令的流动。`stage_to_nextstage_bus` 总线信号中传递缓存信号，会随着`stage_to_nextstage_valid` 流动。

### 握手信号

举个例子，对于指令：

```asm
1c010000 <locate>:
locate():
1c010000:	157f5fe4 	lu12i.w	$r4,-263425(0xbfaff)
1c010004:	02810084 	addi.w	$r4,$r4,64(0x40)
1c010008:	157f5fe5 	lu12i.w	$r5,-263425(0xbfaff)
1c01000c:	0280c0a5 	addi.w	$r5,$r5,48(0x30)
```

当 IDU 译码 `addi.w	$r4,$r4,64(0x40)` 的时候，发现此时遇到了数据相关（r4），那么它就会给上游发信号，让 IFU 暂停。此时 IFU 的 PC 为 `1c010008`，它会一直保持 PC 为 `1c010008`。

但是 IDU 的 PC 呢？它现在遇到了冲突，因此，它会阻塞，此时它的处理的数据为 PC 为 `1c010004` 的数据，它可能会阻塞好几个周期，直到阻塞信号消失。

当它阻塞的时候，会发生什么呢？IDU 可以看做是一个状态机，寄存器保存着当前的 inst，只要它不 load 上游的新的 inst，那么它就能一直保持。换句话说，IDU 不管怎么操作多少次，状态都不会变，每一个模块都具有幂等性。因此，阻塞的 IDU 会一直保持状态，指导冲突信号解除。

对于 IDU，当检测到阻塞的时候，给上游发射 IDU 阻塞（不允许再发送数据）的信号，也就是置 IDU 的 allowin 为 0。此时，fs_allowin 也会被置为 0（这是组合电路信号）。

这意味着什么呢？？？

这意味着，上游也会跟着阻塞。没错，这种阻塞信号会沿着当前模块向上游传递，阻塞上游所有的模块（几乎，不考虑 reset 刚结束）。

此时 IFU 的 PC 为 `1c010008`且保持，一直到 IDU 解除阻塞。

IDU 的核心控制代码：
```Verilog
    reg   ds_valid;    // 表示当前流水级是否在处理指令
    wire  ds_ready_go; // 是否需要阻塞

    assign ds_ready_go    =  ds_valid  & !conflict;
    assign ds_allowin     = !ds_valid || ds_ready_go && es_allowin; // 表示当前阶段是否允许上游进入，通知上游模块
    assign ds_to_es_valid = ds_valid && ds_ready_go;

    always @(posedge clk ) begin
        if (rst) begin
            ds_valid <= 1'b0;
        end
        else if (ds_allowin) begin
            ds_valid <= fs_to_ds_valid;
        end
    end

    always @(posedge clk) begin
        if (fs_to_ds_valid && ds_allowin) begin
            // 卸货，暂时不使用 bus
            pc_reg <= in_pc;
            // 如果当前是跳转，那么下一条指令置空
            inst_sram_rdata_reg <= br_taken ? 32'd0 :inst_sram_rdata;
        end
    end
```

IFU 核心代码：

```Verilog
    // pre-IF stage
    assign to_fs_valid  = ~rst; // 复位时，IFU 阶段无效 这里表示上游指令已经处理完毕
    assign seq_pc       = pc + 32'h4;
    assign nextpc       = br_taken ? br_target : seq_pc;

    // IF stage
    assign fs_ready_go    = 1'b1; // 不需要阻塞
    assign fs_allowin     = !fs_valid || (fs_ready_go && ds_allowin); // 表示当前阶段是否允许上游进入
    assign fs_to_ds_valid = fs_valid && fs_ready_go; // 如果不阻塞且当前在处理指令（事实上，
                                                    // 当fs_valid有效，数据已经处理完成，可以提前发射fs_to_ds_valid）完成


    always @(posedge clk) begin
        if(rst) begin
            fs_valid <= 1'b0; // 复位时，IFU 阶段无效
        end
        else if(fs_allowin) begin
            fs_valid <= to_fs_valid; // 上游指令处理完毕且当前阶段允许上游进入，那么当前阶段开始处理数据
        end
    end

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            pc <= 32'h1bfffffc;
        end
        else if(to_fs_valid && fs_allowin) begin
            pc <= nextpc;
        end
    end
    // 数据的处理和 fs_valid 均在下一个相同的周期完成
```

这是波形图的示例：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/pipeline.png)

对于 IDU，那么如何检测阻塞呢？

### 如何检测数据相关

IDU 需要读取寄存器，但是有一种可能就是 IDU 下游的模块并没有将数据回写，因此 IDU 读取到的数据是不正确的数据。

一个可行的办法就是，在 EXU 中发射三个信号：

```Verilog
    assign exu_regWr = gr_we;
    assign exu_data = alu_result;
    assign exu_regAddr = wire_in_dest;
```

含义是寄存器写使能、写的数据、写的地址。后面的模块也是同理，将这三组信号通过组合电路延伸到 IDU 中，进行判断：

```Verilog
    wire conflict; // 冲突信号
    assign conflict = conflict_regaData_tag || conflict_regbData_tag;

    // 当检测到冲突以后，就得阻塞

    always @(*) begin
        if (exu_regWr && (rf_raddr1 == exu_regAddr)) begin
            conflict_regaData_tag = 1'b1;
        end
        else if (mem_regWr && (rf_raddr1 == mem_regAddr)) begin
            conflict_regaData_tag = 1'b1;
        end
        else if (wbu_regWr && (rf_raddr1 == wbu_regAddr)) begin
            conflict_regaData_tag = 1'b1;
        end
        else begin
            conflict_regaData_tag = 1'b0;
        end
    end

    always @(*) begin
        if (exu_regWr && (rf_raddr2 == exu_regAddr)) begin
            conflict_regbData_tag = 1'b1;
        end
        else if (mem_regWr && (rf_raddr2 == mem_regAddr)) begin
            conflict_regbData_tag = 1'b1;
        end
        else if (wbu_regWr && (rf_raddr2 == wbu_regAddr)) begin
            conflict_regbData_tag = 1'b1;
        end
        else begin
            conflict_regbData_tag = 1'b0;
        end
    end
```

这样就拿到了冲突信号。

### 如何消除冲突信号

实现阻塞控制的核心就是如何检测阻塞信号，以及如何取消阻塞。以 IDU 为例，当 IDU 阻塞的时候，给下游会发送信号，那就是 `ds_to_es_valid`，这个信号会被置 0，告诉下游模块上游没有产生数据。因此，下游模块并不会接收到新的数据，状态机信号保持。EXU 的 `es_valid` 会在下一个周期被置为 0，同时，`es_to_ms_valid` 也会被置为 0，但至少在这一个周期，数据完成了交接。

但是，这里有一个问题。

如果 EXU 给 IDU 发送的数据相关一直保持，那么就会一直有 conflict 信号，也就是说，IDU 将会一直阻塞。

我们并不希望一直阻塞，我们希望的是，EXU 拿到数据之后，要控制写信号的保持时间。

控制寄存器的写保持时间很关键，因为我们不希望一个寄存器的写信号一直存在，虽然在 EXU、MEM 阶段无关痛痒，但是在 WBU 阶段，写信号的保持将会带来功耗和性能问题。

因此统一处理方式是，在 gr_we 的输出上 和 es_valid 信号绑定。这时候，数据相关的信号消失，刚好，可以和 es_valid 配合：

```Verilog
    assign mem_we = wire_in_mem_we;
    assign rkd_value = wire_in_rkd_value;
    assign res_from_mem = wire_in_res_from_mem;
    assign gr_we = wire_in_gr_we & es_valid;
    assign dest = wire_in_dest;
    assign pc = in_pc_reg;

    // 解决数据相关
    assign exu_regWr = gr_we;
    assign exu_data = alu_result;
    assign exu_regAddr = wire_in_dest;
```
这样，当 exu_regWr 信号在 IDU 中为 0 的时候，就可以认为 EXU 已经不存在和 IDU 相关了。这种相关已经顺着数据流转移到了 EXU 的下一个模块 MEM 中。

当冲突消除的那一刻，可以认为，数据已经写到寄存器中了，IDU 可以从寄存器中拿到正确的数据了。

波形图可以很清楚的看到这一点：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/pipeline2.png)


测试也能通过：

```bash
-->CPU 1 1c010150 04 00000000
-->REF 1 1c010150 04 00000000
i:133841
-->CPU 1 1c010154 01 1c010158
-->REF 1 1c010154 01 1c010158
i:133842
i:133843
i:133844
i:133845
-->CPU 1 1c000100 0c bfaff091
-->REF 1 1c000100 0c bfaff091
```


## 前递技术减少部分阻塞

前递技术，也称为旁路、或者数据直通，换句话说，我们没有必要非得等到数据被写到寄存器中才推进。换句话说，我们阻塞的时间太长了！

那怎么能减少阻塞的时间呢？可以直接将数据从后面的模块 EXU、MEM、WBU 中窃取出来，直接送给 IDU 中寄存器访问的结果 readData。

### 寄存器

我们可以这样设置：

```Verilog
    always @(*) begin
        if (exu_regWr && (rf_raddr1 == exu_regAddr)) begin
            conflict_regaData = exu_data;
            conflict_regaData_tag = 1'b1;
        end
        else if (mem_regWr && (rf_raddr1 == mem_regAddr)) begin
            conflict_regaData = mem_data;
            conflict_regaData_tag = 1'b1;
        end
        else if (wbu_regWr && (rf_raddr1 == wbu_regAddr)) begin
            conflict_regaData = wbu_data;
            conflict_regaData_tag = 1'b1;
        end
        else begin
            conflict_regaData = rf_rdata1;  // 默认值
            conflict_regaData_tag = 1'b0;
        end
    end

    always @(*) begin
        if (exu_regWr && (rf_raddr2 == exu_regAddr)) begin
            conflict_regbData = exu_data;
            conflict_regbData_tag = 1'b1;
        end
        else if (mem_regWr && (rf_raddr2 == mem_regAddr)) begin
            conflict_regbData = mem_data;
            conflict_regbData_tag = 1'b1;
        end
        else if (wbu_regWr && (rf_raddr2 == wbu_regAddr)) begin
            conflict_regbData = wbu_data;
            conflict_regbData_tag = 1'b1;
        end

        else begin
            conflict_regbData = rf_rdata2;  // 默认值
            conflict_regbData_tag = 1'b0;
        end
    end

```

这样还没有结束，还要减少冲突的持续周期。在此之前，需要解决一种相关：load-use 相关。

### load-use 相关

以下是一个代码段：

```asm
1c010168:	2880018e 	ld.w	$r14,$r12,0
1c01016c:	0015b5ce 	xor	$r14,$r14,$r13
```

如果仅仅使用前递技术，当 IDU 解析 `xor	$r14,$r14,$r13` 指令时，`ld.w	$r14,$r12,0` 指令处于 EXU 单元中，对于 load 指令，EXU 正在计算 load 指令的要访问的内存地址，此时根本拿不到任何数据，因此 IDU 就会取到错误的数据。

事实上，目前阶段，解决掉 load-use 相关就万事大吉了。

我的思路是，利用一个 reg 来标记上一条指令是否是 load 指令，如果是，再检测是否和 EXU 相关。如果满足这些条件，只需要将 IDU 阻塞一个周期。

```Verilog
    // 检测上一条指令是否是 load 指令
    reg is_load;
    always @(posedge clk) begin
        if (rst) begin
            is_load <= 1'b0;
        end
        else if (fs_to_ds_valid && ds_allowin) begin
            is_load <= inst_ld_w;
        end
    end

    wire load_use_conflict;
    assign load_use_conflict = is_load && exu_regWr && (rf_raddr1 == exu_regAddr);

    assign ds_ready_go    =  ds_valid  & !load_use_conflict;

    assign rj_value  = conflict_regaData;
    assign rkd_value = conflict_regbData;
```

这样，在阻塞技术的基础上，实现了前递技术，仅仅通过解决 load-use 相关大大减少了阻塞周期，至于其他的数据相关，直接前递技术就能解决一切问题。

```bash
-->CPU 1 1c010150 04 00000000
-->REF 1 1c010150 04 00000000
i:60241
-->CPU 1 1c010154 01 1c010158
-->REF 1 1c010154 01 1c010158
i:60242
i:60243
-->CPU 1 1c000100 0c bfaff091
-->REF 1 1c000100 0c bfaff091
```

可以看到，跑同样的测试，上一个方案跑了 133845 周期，这个方案仅仅跑了 60243 走起，减少了一半！效果惊人。



