---
title: "LA 挑战赛记录（3）：FPGA 下板测试流程"
date: 2025-04-06 13:03:13
tags:
  - "LoongArch"
  - "FPGA"
  - "LA challenge"
categories:
  - "Computer Architecture"
---

## 使用 verilator 进行测试

### 生成测试案例

编译测试文件，之后会生成 coe 文件、mif 文件。而且，mif 文件也可以由 vivado 生成，更具体地：
- mif 文件是根据 coe 文件生成的。
- coe 文件只会在生成 rom 模块时起作用，其作用就是根据文件内容生成相应的 mif 文件，而 rom 真正使用的是 mif 文件。
- 直接编辑 .mif 文件的方式不可取，因为在重新生成其他模块 generate output products 后会根据 coe 文件重新生成.mif，可能与设计不一致。

问题是当我们引入 coe 文件的时候，重制 inst_ram 会非常耗时，笔者观察 vivado 花费了 20 分钟还没有完成，因此最好是我们直接使用编译生成的现有的 mif 文件。

前面讲到的重新定制 inst_ram 比较耗时，如果只进行仿真的话可以选择直接替换 Vivado 项目中所用 mif 文件的方法来达到更改 inst_ram 初始值的目的。以单周期CPU的实验开发环境为例，在启动仿真界面后，会发现工程文件中出现下面的目录 `soc_verify/soc_dram/run_vivado/mycpu_prj1/mycpu.sim/sim_1/behave/xsim/`，将该目录中的 `inst_ram.mif` 替换成软件环境 `obj` 目录下的 `inst_ram.mif`
（CPU_CDE/func/obj/inst_ram.mif），再点击 `Restart` 回到零时刻，点击 `run all` 开始仿真，此时 `inst_ram`的初始值就会变成更新后的`inst_ram.mif` 中的数据。具体参考 [这里](https://bookdown.org/loongson/_book3/chapter-single-cycle-cpu.html#subsubsec-recusomize-inst-ram)。

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/restart001.png)

上图中，第一个按钮（仿真之后）就是 Restart，紧接着的按钮就是 Run_all。偷梁换柱之后，点击 Restart，接着点击 Run_all 就可以了：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/tcl001.png)

最后，在生成 `golden_race.txt`，这很关键。

### 初始化工程

接着将完成好的 [cdp_ede_local](https://gitee.com/loongson-edu/cdp_ede_local) 框架中的 `myCPU_env` 复制到装有 vivado 的电脑中。

然后打开 vivado，在 tcl 命令行中 cd 到目标文件夹：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/cd_tcl.png)

接着，继续在 Tcl Console 中输入命令 `source ./create_project.tcl`。接下来 vivado 将根据 create_project.tcl 的内容创建工程。

### 升级 IP

在创建工程之后，IP 如果需要升级，那么它会被锁定：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/IP_update.png)


这时候就得升级，升级很简单，直接参考 [这里](https://bookdown.org/loongson/_book3/appendix-vivado-advanced-usage.html#sec-upgrade-project-ip)。

### 仿真

这一步，很关键，需要明白仿真的原理，这里的话，应该要明白顶层文件是什么。

点击添加文件，将 `mycpu_env/soc_verify/soc_dram/testbench/mycpu_tb.v` 中添加到项目，它是顶层文件。

如果观察的仔细，发现右上角会有这个：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/right_co.png)

直接点击取消，接着点击 Run Simulation，这时候就会发现，什么都没有：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/nothing.png)

这时候，就可以偷梁换柱了。换成已经生成好的 mif 文件后，就可以快速开始了：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/tcl001.png)

## 仿真的架构

### 顶层模块

顶层文件：mycpu_tb.v，这是一个 test_bench，它模拟各种信号给子模块，比如：

```Verilog
reg resetn;
reg clk;

//goio
wire [15:0] led;
wire [1 :0] led_rg0;
wire [1 :0] led_rg1;
wire [7 :0] num_csn;
wire [6 :0] num_a_g;
wire [7 :0] switch;
wire [3 :0] btn_key_col;
wire [3 :0] btn_key_row;
wire [1 :0] btn_step;
assign switch      = 8'hff;
assign btn_key_row = 4'd0;
assign btn_step    = 2'd3;

initial
begin
    clk = 1'b0;
    resetn = 1'b0;
    #2000;
    resetn = 1'b1;
end
always #5 clk=~clk;
```

需要注意的时，gpio 中，它直接赋值 switch 为 0xff 等，至于输出信号，连起来忽略就行。

然后就读取 golden_trace.txt 的行和子模块 soc_lite 中的信号进行对比从而实现 difftest 了。

### 子模块 soc_lite_top

模块 soc_lite_top 就是被验证的模块，它由更复杂的模块组成。

#### clk_pll

这是一个 ip，目的是给原始的 clk 降频。

#### mycpu_top

这就是核心的 CPU 模块了。

#### inst_ram

这是 inst_ram IP，存放指令。

#### data_ram

这是 data_ram IP，可以存放数据。

#### confreg

这是外设配置的模块，详细描述：

```C
//   > Description : Control module of
//                   16 red leds, 2 green/red leds,
//                   7-segment display,
//                   switchs,
//                   key board,
//                   bottom STEP,
//                   timer.
```

以上就是这个仿真的整体概况了。

## fpga 下板子测试

下板子之后，要将 switch 置 全部 1。这是为了能在地址 `0xbfaff090` 读出一个关键的 `0x0000aaaa`。

将测试的顶层模块删除掉之后，就可以下板子测试了，直接一键三连。这个过程比较费时间。经过测试，光一个 ram 综合就花费了一小时 40 分钟。

下板之后，可以通过 switch 控制寄存器 `0xbfaff090` 的值，从而控制空循环的时长：

```asm
1c010158 <idle_1s>:
idle_1s():
1c010158:	157f5fec 	lu12i.w	$r12,-263425(0xbfaff)
1c01015c:	0282418c 	addi.w	$r12,$r12,144(0x90)
1c010160:	1400016d 	lu12i.w	$r13,11(0xb)
1c010164:	02aaa9ad 	addi.w	$r13,$r13,-1366(0xaaa)
1c010168:	2880018e 	ld.w	$r14,$r12,0
1c01016c:	0015b5ce 	xor	$r14,$r14,$r13
1c010170:	0040a5cf 	slli.w	$r15,$r14,0x9
1c010174:	028005ef 	addi.w	$r15,$r15,1(0x1)

1c010178 <sub1>:
sub1():
1c010178:	02bffdef 	addi.w	$r15,$r15,-1(0xfff)
1c01017c:	2880018e 	ld.w	$r14,$r12,0
1c010180:	0015b5ce 	xor	$r14,$r14,$r13
1c010184:	0040a5ce 	slli.w	$r14,$r14,0x9
1c010188:	0012b9f0 	sltu	$r16,$r15,$r14
1c01018c:	5c000a00 	bne	$r16,$r0,8(0x8) # 1c010194 <sub1+0x1c>
1c010190:	028001cf 	addi.w	$r15,$r14,0
1c010194:	5fffe5e0 	bne	$r15,$r0,-28(0x3ffe4) # 1c010178 <sub1>
1c010198:	4c000020 	jirl	$r0,$r1,0
1c01019c:	00000000 	0x00000000
```























