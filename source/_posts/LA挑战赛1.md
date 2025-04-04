---
title: LA 挑战赛：开篇
date: 2025-04-03 13:03:13
tags:
    - LoongArch
    - CPU
categories: CPU
---

## 环境
参考 [chiplab](https://gitee.com/loongson-edu/chiplab) 配置好环境之后，基于 [loongsonEdu](https://gitee.com/loongson-edu) 的开源项目 [cdp_ede_local](https://gitee.com/loongson-edu/cdp_ede_local)，进行前期的增量开发，同时也参考 [CPU 设计实战](https://bookdown.org/loongson/_book3/chapter-cpu-chip-design-process.html)。后期可以顺利移植到 chiplab 中进行验证。

## minicpu_env

这个实验要求阅读代码，熟悉流程，并补充代码且进行 debug（代码有的地方有问题）。初次在 vivado 上仿真之后，感觉难以调试（时间漫长），于是我在本地搭建了基于 verilator 的仿真环境，可以迅速生成波形图。

修复一些问题之后，成功通过了斐波那契数列程序的检测。现在简要分析一下 minicpu_env 中的设计、信号的生成。

环境中有一个 convert 的程序，它生成 coe、mif 文件（和 bin2coe 程序功能一致）。

由于 verilator 中无法使用 xilinx 的 ram ip，我加了两个脚本，让其自动生成 instram、data_ram 模块，但是要注意字节序的问题。loongarch 是小端字节序，因此指令读出来的时候要将其进行字节序翻转。

接下来就可以用 verilator 仿真了。

## minicpu_env 的架构

### IFU

```Verilog
    always @(posedge clk) begin
        if (reset) begin
            // pc <= 32'h1bfffffc;     //trick: to make nextpc be 0x1c000000 during reset
            pc <=32'h0;
        end
        else begin
            pc <= nextpc;
        end
    end
```

修改 nextpc 在 exu 中实现。

### IDU

众所周知，简单 CPU 实现中最关键的模块就是译码模块。

环境将 32 位指令字段划分后，利用译码器来进行指令识别：

```Verilog
    wire [ 5:0] op_31_26;
    wire [ 3:0] op_25_22;
    wire [ 1:0] op_21_20;
    wire [ 4:0] op_19_15;
    wire [63:0] op_31_26_d;
    wire [15:0] op_25_22_d;
    wire [ 3:0] op_21_20_d;
    wire [31:0] op_19_15_d;

    assign op_31_26 = inst[31:26];
    assign op_25_22 = inst[25:22];
    assign op_21_20 = inst[21:20];
    assign op_19_15 = inst[19:15];

    decoder_6_64 u_dec0(.in(op_31_26 ), .co(op_31_26_d ));
    decoder_4_16 u_dec1(.in(op_25_22 ), .co(op_25_22_d ));
    decoder_2_4  u_dec2(.in(op_21_20 ), .co(op_21_20_d ));
    decoder_5_32 u_dec3(.in(op_19_15 ), .co(op_19_15_d ));

    assign inst_add_w  = op_31_26_d[6'h00] & op_25_22_d[4'h0] & op_21_20_d[2'h1] & op_19_15_d[5'h00];
    assign inst_addi_w = op_31_26_d[6'h00] & op_25_22_d[4'ha];
    assign inst_ld_w   = op_31_26_d[6'h0a] & op_25_22_d[4'h2];
    assign inst_st_w   = op_31_26_d[6'h0a] & op_25_22_d[4'h6];
    assign inst_bne    = op_31_26_d[6'h17];
```

这个指令译码设计得挺好。另外，译码器还分别实现了以下信号。

#### src2_is_imm

这个信号针对一些第二操作数是立即数的指令，比如 addi、ld、st等指令。

#### res_from_mem

这个信号针对一些 load 指令，比如 ld。

#### gr_we

这个信号针对是否修改寄存器的指令，比如add、ld、addi等等，大部分指令都需要回写寄存器。

#### mem_we

这个信号针对修改内存的指令，比如 store 指令。

#### src_reg_is_rd

源寄存器是否是 rd。由于 loongarch 某些指令需要用 rd 寄存器作为源寄存器，比如 bne、st。

接着就可以合成访问寄存器的信号了：

```Verilog
    assign rf_raddr1 = rj;
    assign rf_raddr2 = src_reg_is_rd ? rd :rk;
    regfile u_regfile(
                .clk    (clk      ),
                .raddr1 (rf_raddr1),
                .rdata1 (rj_value),
                .raddr2 (rf_raddr2),
                .rdata2 (rkd_value),
                .we     (gr_we    ),
                .waddr  (rd       ),
                .wdata  (rf_wdata )
            );
```

#### 分支的处理

分支信号的产生在 IDU 中产生，这样距离 IFU 很近，但是需要利用加法，这不是什么问题：

```Verilog
    assign br_offs   = {{14{i16[15]}},i16[15:0],2'b00};
    assign br_target = pc + br_offs;
    assign rj_eq_rd  = (rj_value == rkd_value);
    assign br_taken  = valid && inst_bne  && !rj_eq_rd;
    assign nextpc    = br_taken ? br_target : pc + 4;
```

接下来生成两个操作数，为 ALU 做准备：

```Verilog
    assign imm      = {{20{i12[11]}},i12[11:0]};
    assign alu_src1 = rj_value;
    assign alu_src2 = src2_is_imm ? imm : rkd_value;
```

以上就是 IDU。


### EXU

目前的 EXU 很简单：

```Verilog
    assign alu_result = alu_src1 + alu_src2;
```

### MEM

访存单元：

```Verilog
    assign data_sram_we    = mem_we;
    assign data_sram_addr  = alu_result;
    assign data_sram_wdata = rkd_value;
```

### WBU

回写单元：

```Verilog
    assign rf_wdata = res_from_mem ? data_sram_rdata : alu_result;
```

可以看到，麻雀虽小，但五脏俱全。

以上 5 条指令的处理器可以运行斐波那契数列计算程序：

```asm
1c000000:       addi.w   $t0,$zero,0x0      //置第1项的0
1c000004:       addi.w   $t1,$zero,0x1      //置第2项的1
1c000008:       addi.w   $s0,$zero,0x0      //循环变量i初始化为0
1c00000c:       addi.w   $s1,$zero,0x1      //循环的步长置为1
1c000010:       ld.w     $a0,$zero,1024     //读取拨码开关输入的终止值
          loop:
1c000014:       add.w    $t2,$t0,$t1        //f(i) = f(i-2) + f(i-1)
1c000018:       addi.w   $t0,$t1,0x0        //记录f(i-1)
1c00001c:       addi.w   $t1,$t2,0x0        //记录f(i)
1c000020:       add.w    $s0,$s0,$s1        //i++
1c000024:       bne      $s0,$a0,loop       //if i!=n, goto loop
1c000028:       st.w     $t2,$zero,1028     //将f(n)的值输出到数码管上
          end:
1c00002c:       bne      $s1, $zero, end    //测试完毕，进入死循环

```

