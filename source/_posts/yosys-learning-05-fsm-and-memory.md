---
title: "Yosys 学习笔记（五）：FSM 与 Memory 如何变成状态表和寄存器阵列"
date: 2026-07-23 10:52:28
tags:
  - "Yosys"
  - "Verilog"
  - "RTLIL"
  - "FSM"
  - "Memory"
categories:
  - "Computer Architecture"
---

## 前言

组合表达式经过 `proc` 和 `opt` 后，通常会变成 mux、比较器和算术 cell。但状态机和数组存储包含更高层的设计意图：

```text
状态机：状态集合、转移条件、复位状态、控制输出
memory：容量、端口数量、读写时序、碰撞语义
```

如果综合器只把它们当成普通寄存器和 mux，就很难执行状态重编码，也很难把数组映射到 SRAM macro、FPGA block RAM 或 LUT RAM。Yosys 因此提供了两组专门的处理流程：`fsm` 和 `memory`。

不过，观察这两个宏命令时很容易遇到一个反直觉现象：

```text
运行 fsm 后，为什么 RTLIL 里没有 $fsm？
运行 memory 后，为什么 RTLIL 里也没有 $mem_v2？
```

答案不是 pass 没运行，而是两个宏默认都会继续完成后续映射：

```text
fsm 默认最后调用 fsm_map，把 $fsm 重新展开成触发器和逻辑
memory 默认最后调用 memory_map，把 $mem_v2 展开成寄存器和地址译码
```

本文用一个包含四状态控制器和 `16 x 8` memory 的小设计，完整观察下面这条路径：

```text
Verilog always / case / reg array
  -> proc：触发器、mux、独立 memory 端口
  -> fsm -nomap：$fsm 状态表与 one-hot 重编码
  -> fsm_map：重新生成状态寄存器和转移逻辑
  -> memory -nomap：统一的 $mem_v2
  -> memory_map：16 个字级寄存器、读 mux 树和写地址译码
```

重点不只是记住命令，而是理解每一个中间表示为什么存在、何时会消失，以及 pass 的顺序为什么会影响 FSM 检测结果。

## 完整实验设计

先看完整 Verilog：

```verilog
module control_and_mem(
    input        clk,
    input        rst,
    input        start,
    input  [3:0] addr,
    input  [7:0] din,
    output reg   done,
    output reg [7:0] dout
);
    localparam IDLE = 2'b00;
    localparam LOAD = 2'b01;
    localparam WAIT = 2'b10;
    localparam DONE = 2'b11;

    reg [1:0] state;
    reg [1:0] next_state;
    reg [7:0] mem [0:15];

    always @* begin
        next_state = state;
        done = 1'b0;

        case (state)
            IDLE: if (start) next_state = LOAD;
            LOAD: next_state = WAIT;
            WAIT: next_state = DONE;
            DONE: begin
                done = 1'b1;
                next_state = IDLE;
            end
            default: next_state = IDLE;
        endcase
    end

    always @(posedge clk or posedge rst) begin
        if (rst)
            state <= IDLE;
        else
            state <= next_state;
    end

    always @(posedge clk) begin
        if (state == LOAD)
            mem[addr] <= din;
        dout <= mem[addr];
    end
endmodule
```

这个模块里有两个相对独立、又通过 `state == LOAD` 联系起来的部分。

## 先读懂 FSM 行为

状态转移关系是：

```text
                 start=0
              +-----------+
              |           |
              v           |
            IDLE ----------+
              |
              | start=1
              v
            LOAD
              |
              v
            WAIT
              |
              v
            DONE --done=1--> IDLE
```

更精确地列成表：

| 当前状态 | 条件 | 下一状态 | `done` |
|---|---|---|---:|
| `IDLE` | `start=0` | `IDLE` | `0` |
| `IDLE` | `start=1` | `LOAD` | `0` |
| `LOAD` | 无条件 | `WAIT` | `0` |
| `WAIT` | 无条件 | `DONE` | `0` |
| `DONE` | 无条件 | `IDLE` | `1` |

组合块开头的默认赋值非常重要：

```verilog
next_state = state;
done = 1'b0;
```

它们保证所有组合路径都有赋值。否则某些分支没有给 `next_state` 或 `done` 赋值时，综合器必须推断 latch 来保存旧值。

这个设计中的默认 `next_state = state` 还会影响后面的 FSM 检测：它在网表中表现为状态寄存器的自反馈路径，而普通 `opt` 可能把这种“保持旧值”的行为吸收到触发器使能中。

## 再读懂 memory 行为

数组声明：

```verilog
reg [7:0] mem [0:15];
```

表示：

```text
字宽 WIDTH = 8 bit
深度 SIZE  = 16 word
地址宽度   = 4 bit
总存储容量 = 16 x 8 = 128 bit
```

写入条件是：

```verilog
if (state == LOAD)
    mem[addr] <= din;
```

所以 memory 有一个同步写端口：只有时钟上升沿到来且当前状态为 `LOAD` 时，才把 `din` 写入 `addr` 指定的 word。

读取也是同步的：

```verilog
dout <= mem[addr];
```

`dout` 只在时钟上升沿更新。由于使用非阻塞赋值，同一时钟沿同时读写相同地址时，`dout` 读取的是写入前的旧值。这类行为通常称为 read-first 或 non-transparent read。

这里没有给 memory 写初始化内容，因此它的初始 128 bit 都是不确定值。

## 第一版脚本与一个诚实的意外

最直接的实验脚本可能写成：

```text
read_verilog -sv control_and_mem.v
hierarchy -check -top control_and_mem

proc
opt
write_rtlil 01_after_proc_opt.il

fsm
opt
write_rtlil 02_after_fsm.il

memory
opt
stat
write_rtlil 03_after_memory.il
write_verilog -noattr control_and_mem_netlist.v
```

运行后会观察到：

```text
01_after_proc_opt.il 与 02_after_fsm.il 完全相同
02_after_fsm.il 中没有 $fsm
03_after_memory.il 中没有任何 memory cell
```

如果只看命令名称，很容易得出“`fsm` 没用，`memory` 把 memory 弄丢了”的错误结论。真实原因需要分别分析。

## `proc; opt` 后 FSM 变成了什么

`proc` 会把三个 `always` 过程全部转换成 RTLIL cell。

状态寄存器最初是一个带异步复位的 `$adff`：

```text
cell $adff $state_ff
  parameter \WIDTH 2
  parameter \CLK_POLARITY 1'1
  parameter \ARST_POLARITY 1'1
  parameter \ARST_VALUE 2'00
  connect \CLK \clk
  connect \ARST \rst
  connect \D \next_state
  connect \Q \state
end
```

`case (state)` 则变成：

```text
状态常量比较器
  + $mux / $pmux 组成的 next_state 选择树
  + $mux / $pmux 组成的 done 逻辑
```

执行普通 `opt` 后，Yosys 注意到某些情况下 `next_state` 等于 `state`，于是把“保持旧状态”变成触发器使能，状态寄存器从 `$adff` 变成 `$adffe`：

```text
cell $adffe $state_ff
  parameter \WIDTH 2
  parameter \ARST_VALUE 2'00
  connect \CLK \clk
  connect \ARST \rst
  connect \EN $state_should_update
  connect \D \next_state
  connect \Q \state
end
```

功能没有变化，但结构形态变化了。这个变化恰好会影响 `fsm_detect`。

## FSM 检测不是看到 case 就算成功

Yosys 不是在 Verilog AST 中看到 `case (state)` 就直接贴上 FSM 标签。`fsm_detect` 在 RTLIL 网表上寻找满足特定形状的状态信号。

一个典型候选需要满足：

1. 状态信号是多 bit 内部信号，而不是模块输出端口；
2. 它由单个 `$dff` 或 `$adff` 驱动；
3. 触发器 `D` 端由 mux 树驱动；
4. mux 树叶子只能是状态常量或旧状态本身；
5. 状态值只用于这棵转移 mux 树，或用于与常量进行简单比较；
6. 没有妨碍安全重编码的初始化、复杂数据用途或自复位行为。

普通 `opt` 把当前实验中的状态寄存器变成 `$adffe` 后，它不再属于检测器所使用的 `$dff/$adff` 子集。因此第一版脚本运行 `fsm` 时，日志只有各个 `fsm_*` 子 pass 的标题，却没有：

```text
Found FSM state register ...
```

这就是 `01_after_proc_opt.il` 和 `02_after_fsm.il` 完全相同的直接原因。

## 正确准备 FSM 检测输入

Yosys 给出的推荐准备方式是：

```text
opt -nosdff -nodffe
```

其中 `-nodffe` 阻止 `opt_dff` 把 mux 反馈吸收到触发器 enable，`-nosdff` 则避免形成检测器暂不处理的同步复位触发器变体。

因此更适合观察 FSM 的前半段脚本是：

```text
read_verilog -sv control_and_mem.v
hierarchy -check -top control_and_mem

proc
opt -nosdff -nodffe
write_rtlil 01_after_proc.il

fsm -nomap
write_rtlil 02_with_fsm.il
```

这里还有另一个关键选项：

```text
-nomap
```

`fsm` 是宏命令，默认最后调用 `fsm_map`。如果不加 `-nomap`，即使 `$fsm` 成功生成，也会在写 RTLIL 之前重新展开成触发器和组合逻辑。

所以：

```text
opt -nosdff -nodffe：让检测器看见合适的 FF + mux 结构
fsm -nomap：让生成的 $fsm 暂时留在 RTLIL 中供我们观察
```

## `fsm` 宏内部做了什么

`fsm` 不是一个单动作 pass，而是一组子 pass：

```text
fsm_detect
fsm_extract
fsm_opt
opt_clean
fsm_opt
fsm_recode
fsm_info
fsm_map        # 除非使用 -nomap
```

它们分别承担不同职责：

| 子 pass | 作用 |
|---|---|
| `fsm_detect` | 找到状态寄存器，并标记 `fsm_encoding="auto"` |
| `fsm_extract` | 沿转移 mux 树收集状态、输入、输出和转移表，生成 `$fsm` |
| `fsm_opt` | 删除无用输入输出，合并等价转移条件 |
| `fsm_recode` | 为状态分配新编码 |
| `fsm_info` | 输出状态数量、编码和转移表 |
| `fsm_map` | 把 `$fsm` 重新实现为触发器和组合逻辑 |

`fsm_encoding` 也可以由 RTL 属性控制：

```verilog
(* fsm_encoding = "auto" *) reg [1:0] state;
```

或者明确阻止检测：

```verilog
(* fsm_encoding = "none" *) reg [1:0] state;
```

但强制属性不是修复任意复杂状态机的万能方法。如果状态寄存器还承担数据运算、部分 bit 直接泄漏到外部，或者转移树不适合重编码，强制提取可能增加面积，甚至造成仿真与综合语义风险。应先让 RTL 和 pass 顺序满足检测不变量。

## 实际提取出的五条状态转移

修正脚本后，日志首先确认：

```text
Found FSM state register control_and_mem.state.
```

随后从两位原始状态编码中提取五条转移：

```text
2'00 + start=0 -> 2'00
2'00 + start=1 -> 2'01
2'01 + start=- -> 2'10
2'10 + start=- -> 2'11
2'11 + start=- -> 2'00
```

这里 `-` 表示 don't-care：处于 `LOAD`、`WAIT` 或 `DONE` 时，`start` 不影响下一状态。

为什么四个状态却有五条转移？因为 `IDLE` 必须根据 `start` 分成两种情况，其余三个状态各有一条无条件转移：

```text
IDLE：2 条
LOAD：1 条
WAIT：1 条
DONE：1 条
总计：5 条
```

## 两位编码为什么变成四位 one-hot

原始 Verilog 使用两位二进制编码：

```text
IDLE = 00
LOAD = 01
WAIT = 10
DONE = 11
```

`fsm_recode` 的自动策略为这个小状态机选择了 one-hot，并输出：

```text
00 -> ---1
10 -> --1-
01 -> -1--
11 -> 1---
```

这里 `-` 是编码表中的 don't-care 展示。映射后的有效状态分别只有一个 bit 为 `1`，复位状态对应最低位：

```text
IDLE -> 0001
WAIT -> 0010
LOAD -> 0100
DONE -> 1000
```

one-hot 使用四个状态 bit，而原设计只用两个。它不是单纯“节省触发器”的优化，而是用更多触发器换取更简单的状态译码和转移逻辑：

| 编码 | 状态寄存器数量 | 状态比较逻辑 | 常见适用方向 |
|---|---:|---|---|
| 二进制编码 | 少 | 相对复杂 | ASIC、状态较多、重视面积 |
| one-hot | 多 | 相对简单 | FPGA、小 FSM、重视速度 |

最终优劣取决于工艺库、目标频率、状态图和后续优化，不能只看状态 bit 数量。

## `$fsm` cell 中保存了什么

`fsm -nomap` 后，整个状态机被压缩成一个 `$fsm`：

```text
cell $fsm $state_fsm
  parameter \NAME "\\state"
  parameter \CTRL_IN_WIDTH 1
  parameter \CTRL_OUT_WIDTH 4
  parameter \STATE_BITS 4
  parameter \STATE_NUM 4
  parameter \STATE_RST 0
  parameter \TRANS_NUM 5

  connect \CLK \clk
  connect \ARST \rst
  connect \CTRL_IN \start
  connect \CTRL_OUT $state_decodes
end
```

重要参数是：

```text
1 个控制输入：start
4 个状态
4 个重编码后的状态 bit
5 条状态转移
异步复位状态：状态 0
```

`CTRL_OUT` 不只是顶层的 `done`。状态比较结果还被 memory 写使能等外部逻辑使用。提取器会把这些依赖状态编码的比较结果变成 `$fsm` 的控制输出，这样状态编码改变后，外部逻辑仍能获得正确含义。

## 为什么最终网表里通常又没有 `$fsm`

完成状态分析和重编码后，硬件仍需要由触发器和组合门实现。执行：

```text
fsm_map
opt
```

会把 `$fsm` 展开为：

```text
一个 4 bit $adff 状态寄存器
复位值 4'b0001
由 start 和状态 bit 构成的转移逻辑
由状态 bit 产生的控制输出逻辑
```

关键 RTLIL 类似：

```text
cell $adff $state_ff
  parameter \WIDTH 4
  parameter \ARST_VALUE 4'0001
  connect \CLK \clk
  connect \ARST \rst
  connect \D $next_one_hot_state
  connect \Q \state
end
```

所以最终看不到 `$fsm` 不代表状态机没有被识别。判断方法应当是查看 `fsm_detect`、`fsm_extract`、`fsm_recode` 的日志，或者用 `fsm -nomap` 在中间阶段暂停。

## `proc; opt` 后 memory 变成了什么

与 FSM 不同，memory 在 `proc` 后仍保留一个 RTLIL memory 对象：

```text
memory width 8 size 16 \mem
```

读写端口暂时是独立 cell。

读取路径表现为一个异步 `$memrd` 加输出 `$dff`：

```text
cell $memrd $read_port
  parameter \MEMID "\\mem"
  parameter \ABITS 4
  parameter \WIDTH 8
  parameter \CLK_ENABLE 0
  connect \ADDR \addr
  connect \DATA $dout_next
end

cell $dff $dout_ff
  parameter \WIDTH 8
  connect \CLK \clk
  connect \D $dout_next
  connect \Q \dout
end
```

这并不表示原设计是异步读。原 Verilog 的同步读语义被分成了两层：memory 本体先组合读出，外面的 `$dff` 再在时钟沿采样。

写入路径是 `$memwr_v2`：

```text
cell $memwr_v2 $write_port
  parameter \MEMID "\\mem"
  parameter \ABITS 4
  parameter \WIDTH 8
  parameter \CLK_ENABLE 1'1
  parameter \CLK_POLARITY 1'1
  connect \ADDR $write_addr
  connect \DATA $write_data
  connect \EN $write_enable_per_bit
  connect \CLK \clk
end
```

`EN` 宽度与数据宽度相同，因为 Yosys 的写端口支持逐 bit 写使能。当前设计一次写整个字，所以八个 enable bit 都连接到同一个 `state == LOAD` 条件。

## `memory` 也是一个宏命令

默认 `memory` 依次调用：

```text
opt_mem
opt_mem_priority
opt_mem_feedback
memory_bmux2rom
memory_dff
opt_clean
memory_share
opt_mem_widen
opt_clean
memory_collect
memory_map          # 除非使用 -nomap
```

可以把它们分为三组：

| 阶段 | 代表 pass | 目的 |
|---|---|---|
| 端口优化 | `opt_mem*`、`memory_share` | 删除无效端口、整理优先级、共享资源 |
| 结构收集 | `memory_dff`、`memory_collect` | 吸收端口寄存器，把分散端口合成 `$mem_v2` |
| 实现映射 | `memory_map` 或目标专用映射 | 变成寄存器逻辑、block RAM、LUT RAM 或 SRAM macro |

默认 `memory` 会一直执行到 `memory_map`，所以普通脚本写完 RTLIL 时，`$mem_v2` 已经消失。

要观察统一 memory cell，应使用：

```text
memory -nomap
write_rtlil 03_with_mem_v2.il
```

## `memory_dff` 如何恢复同步读取

`memory_dff` 会检查 memory 端口附近的触发器，尝试把寄存器吸收到 memory 端口中。

对当前设计，它识别到：

```text
$memrd -> $dff -> dout
```

可以合并成一个同步 read port，并报告：

```text
merging output FF to cell
Write port 0: non-transparent
```

合并后的 read port 在时钟上升沿更新 `dout`，不再需要外部 `$dff`。`non-transparent` 表示同周期同地址读写时，读端不会直接看到新写入数据，对应原始非阻塞赋值的 read-first 行为。

这一步说明 memory inference 不只看数组声明，还必须理解端口周围的时序逻辑。同步读、异步读、read-first、write-first 和 no-change 会直接影响能否映射到目标 memory primitive。

## `memory_collect` 生成统一的 `$mem_v2`

执行 `memory -nomap` 后，独立的 memory 对象、读端口和写端口被收集为一个 `$mem_v2`：

```text
cell $mem_v2 \mem
  parameter \WIDTH 8
  parameter \SIZE 16
  parameter \ABITS 4

  parameter \RD_PORTS 1
  parameter \RD_CLK_ENABLE 1'1
  parameter \RD_CLK_POLARITY 1'1
  parameter \RD_TRANSPARENCY_MASK 1'0
  parameter \RD_COLLISION_X_MASK 1'0

  parameter \WR_PORTS 1
  parameter \WR_CLK_ENABLE 1'1
  parameter \WR_CLK_POLARITY 1'1

  parameter \INIT 128'x

  connect \RD_CLK \clk
  connect \RD_EN 1'1
  connect \RD_ADDR \addr
  connect \RD_DATA \dout

  connect \WR_CLK \clk
  connect \WR_EN $write_enable_per_bit
  connect \WR_ADDR $write_addr
  connect \WR_DATA $write_data
end
```

这个 cell 完整描述了目标映射真正关心的信息：

```text
几行、每行多宽
有几个读写端口
端口是否同步
时钟极性
写使能粒度
读写碰撞行为
初始内容
```

`$mem_v2` 是一个很重要的架构边界。它暂时保留“这是 memory”的意图，让后续 target-specific pass 有机会选择真正的存储资源。

## `memory_map` 为什么生成 16 个寄存器

当前实验没有提供 FPGA RAM 规则或 ASIC SRAM macro，所以手动执行：

```text
memory_map
opt
```

会采用通用逻辑实现。日志给出非常具体的结果：

```text
created 16 $dff cells of width 8
read interface: 1 $dff and 15 $mux cells
write interface: 16 write mux blocks
```

第一行表示 memory 的 16 个 word 分别变成一个 8 bit 字级触发器：

```text
mem[0]  -> 8 bit FF
mem[1]  -> 8 bit FF
...
mem[15] -> 8 bit FF
```

总存储 bit 仍然是：

```text
16 x 8 = 128 bit
```

经过后续 `opt_dff`，每个 word 的“地址命中时更新，否则保持”被吸收到 enable，因此最终是 16 个 8 bit `$dffe`。这不是只有 16 个单比特触发器；每个 `$dffe` 的 `WIDTH=8`，继续技术映射后才可能拆成 128 个单比特标准触发器。

## 为什么 16 个 word 需要 15 个读 mux

读端要从 16 个 word 中选择一个：

```text
                 addr[3]
              +-----------+
              |           |
            8-word      8-word
             子树         子树
              |           |
              +-----MUX---+
                    |
                   dout
```

一棵拥有 16 个叶子的二叉选择树需要 15 个二选一 mux：

```text
第 1 层：8 个 mux
第 2 层：4 个 mux
第 3 层：2 个 mux
第 4 层：1 个 mux
总计：8 + 4 + 2 + 1 = 15
```

每个 mux 都是 8 bit 宽，因此面积并不小。这也是为什么较大的 memory 应尽量映射到专用存储宏，而不是无条件展开成 FF RAM。

## 写地址译码在做什么

写端必须确保只有被 `addr` 选中的 word 更新。四位地址先被译码成 16 路 one-hot：

```text
addr = 0000 -> word_enable[0]  = 1
addr = 0001 -> word_enable[1]  = 1
...
addr = 1111 -> word_enable[15] = 1
```

然后与全局写条件相与：

```text
write_enable[i] = (state == LOAD) && (addr == i)
```

每个 `$dffe` 获得自己的 enable：

```text
if write_enable[i]:
    mem[i] <= din
else:
    mem[i] 保持原值
```

综合器会共享地址反相和部分与门，所以最终门数量不一定等于手工展开公式的直接总和。

## 修正后的完整阶段统计

采用“先暂停观察，再手动映射”的脚本，最终通用逻辑网表统计为：

```text
86 cells
 1   $adff
42   $and
 1   $dff
16   $dffe
18   $mux
 5   $not
 1   $pmux
 2   $reduce_or
```

可以按功能划分：

```text
$adff：4 bit one-hot FSM 状态寄存器
$dff：memory 同步读出的 dout 寄存器
16 x $dffe：16 个 8 bit memory word
$mux：memory 读取树及剩余数据选择
$and/$not：写地址译码和 FSM 转移逻辑
$pmux/$reduce_or：组合控制输出
```

cell 数量不是优化质量的绝对指标。这里真正值得观察的是结构意图如何经历：

```text
状态寄存器 + mux
  -> $fsm
  -> one-hot 状态寄存器 + 新转移逻辑

memory 对象 + 独立端口
  -> $mem_v2
  -> 字级寄存器 + 地址译码 + 读 mux 树
```

## 为什么真实流程通常不急着 `memory_map`

直接 `memory_map` 等于明确选择 FF RAM。对于 128 bit 小存储器，它可能可以接受；对于几 KB、几 MB 的 memory，这会造成巨大的面积和功耗。

更常见的目标相关流程是：

```text
memory -nomap
  -> 保留 $mem_v2
  -> memory_libmap 根据目标存储库选择实现
  -> techmap 映射到硬件 primitive 或 SRAM wrapper
  -> memory_map 只处理无法映射的剩余 memory
```

`memory_libmap` 在必要时还可能加入模拟逻辑，以弥补目标 macro 与 RTL 端口语义之间的差异。

### FPGA 上的候选资源

FPGA 通常有四类候选：

| 类型 | 典型资源 | 主要限制 |
|---|---|---|
| FF RAM | 触发器 + mux | 灵活但面积大 |
| LUT RAM | LUT 内部存储 | 通常支持同步写、异步读，容量较小 |
| Block RAM | 专用 BRAM tile | 端口、时钟、读写模式受硬件限制 |
| Huge RAM | UltraRAM、SPRAM 等 | 只在部分系列存在，限制更多 |

通常：

```text
异步读 -> 更可能是 LUT RAM 或 FF RAM
同步读 + 合适端口 -> 有机会使用 block RAM
很小的 memory -> 工具可能认为 LUT/FF 更划算
很大的 memory -> 更倾向专用 RAM
```

当前例子是单时钟、一个同步读端口和一个同步写端口，结构上适合许多同步 RAM primitive；但只有 128 bit，最终选择仍取决于目标系列和成本模型。

### ASIC 上的候选资源

ASIC 中通常希望映射到 memory compiler 生成的 SRAM macro。映射需要匹配：

```text
深度和字宽
读写端口数量
单口、简单双口或真双口
时钟极性
读延迟
写 mask 粒度
read-during-write 行为
初始化和复位能力
```

如果流程没有提供可用 SRAM macro 或映射规则，Yosys 只能退回寄存器和组合逻辑。对大型 memory 来说，这通常不是可接受的最终实现，因此必须在综合约束和日志中确认 macro inference 是否成功。

## 读写碰撞语义为什么重要

假设同一时钟沿同时读取和写入同一地址：

```text
read_addr  = 4
write_addr = 4
write_data = 8'hA5
```

可能存在三种常见语义：

| 模式 | 读出的值 |
|---|---|
| read-first | 写入前的旧值 |
| write-first | 新写入的 `8'hA5` |
| no-change / undefined | 保持旧输出或结果不保证 |

目标 block RAM 或 SRAM macro 不一定原生支持 RTL 要求的模式。综合器有时需要插入 bypass mux 或其他逻辑维持等价，这会影响面积和时序。

当前实验使用：

```verilog
if (write_enable)
    mem[addr] <= din;
dout <= mem[addr];
```

对应 read-first。`$mem_v2` 中：

```text
RD_TRANSPARENCY_MASK = 0
RD_COLLISION_X_MASK  = 0
```

表示读端不透明，也没有把碰撞声明为返回 `x`。

## 常见的 memory RTL 写法差异

### 异步读

```verilog
always @(posedge clk)
    if (we)
        mem[waddr] <= wdata;

assign rdata = mem[raddr];
```

地址变化后不等时钟，`rdata` 就变化。这通常不能直接映射到只支持同步读的 block RAM。

### 同步读

```verilog
always @(posedge clk) begin
    if (we)
        mem[waddr] <= wdata;
    rdata <= mem[raddr];
end
```

`rdata` 在时钟沿更新，更适合常见 BRAM 和 SRAM macro。

### write-first bypass

```verilog
always @(posedge clk) begin
    if (we)
        mem[waddr] <= wdata;

    if (we && waddr == raddr)
        rdata <= wdata;
    else
        rdata <= mem[raddr];
end
```

同地址碰撞时显式返回新数据。若硬件 memory 不是 write-first，综合器可能需要保留额外 bypass mux。

RTL 中看似只差几行，映射到目标资源时可能是完全不同的硬件约束。

## FSM 为什么可能无法识别

除了本实验的 `$adffe` pass 顺序问题，常见原因还包括：

### 状态寄存器成为模块输出

```verilog
output [2:0] state;
```

如果状态编码直接暴露到模块接口，重编码会改变可观察行为，检测器通常不会自动处理。

更好的做法是输出有语义的状态标志：

```verilog
assign busy = state != IDLE;
assign done = state == DONE;
```

### 状态值被当成普通数据运算

```verilog
assign debug_value = state + offset;
```

这使状态编码本身具有数据含义，重编码就不再自由。

### 下一状态不是常量或旧状态组成的 mux 树

```verilog
next_state = state + 1'b1;
```

虽然人可以把它理解为计数式 FSM，但对启发式检测器来说，它更像普通计数器。

### 组合逻辑漏默认赋值

```verilog
always @* begin
    case (state)
        IDLE: if (start) next_state = LOAD;
        LOAD: next_state = WAIT;
    endcase
end
```

未覆盖路径会推断 latch，状态转移结构也不再是预期的纯组合 mux 树。

### pass 顺序破坏检测形状

一些优化会把 mux、enable、同步复位或比较逻辑改写成检测器不认识的形态。遇到这种情况，应先查看 `fsm_detect` 的输入 RTLIL，而不是立刻强制添加属性。

## Memory 为什么可能无法映射到专用 RAM

常见原因包括：

```text
异步读，而目标 BRAM 只支持同步读
读写端口数量超过 macro 能力
多个写端口处于不同、不受支持的时钟域
byte enable 粒度与硬件不匹配
碰撞语义与硬件模式不匹配
memory 太小，成本模型选择 LUT/FF
memory 太大或形状特殊，没有合适 primitive
初始化或复位要求硬件无法实现
```

可以使用属性表达实现偏好，例如：

```verilog
(* ram_style = "logic" *)       reg [7:0] mem [0:15];
(* ram_style = "distributed" *) reg [7:0] mem [0:15];
(* ram_style = "block" *)       reg [7:0] mem [0:1023];
```

但属性不是魔法。如果目标器件无法实现指定模式，正确结果应当是映射失败或报错，而不是强行产生不等价硬件。

## 推荐的完整观察脚本

下面这份脚本把 FSM 和 memory 的高层表示都保留下来一次，然后再手动映射：

```text
read_verilog -sv control_and_mem.v
hierarchy -check -top control_and_mem

# 保持 $dff/$adff + mux 形态，便于 FSM 检测。
proc
opt -nosdff -nodffe
write_rtlil 01_after_proc.il

# 暂停在 $fsm。
fsm -nomap
write_rtlil 02_with_fsm.il

# 把重编码后的 FSM 映回触发器和逻辑。
fsm_map
opt

# 暂停在统一的 $mem_v2。
memory -nomap
write_rtlil 03_with_mem_v2.il

# 使用通用 FF RAM 实现。
memory_map
opt
stat
write_rtlil 04_after_memory_map.il
write_verilog -noattr control_and_mem_netlist.v
```

运行：

```bash
yosys -Q -l fsm_memory.log fsm_memory.ys
```

用结构断言把观察结果变成自动检查：

```text
# 在 02_with_fsm.il 对应阶段执行：
select -assert-count 1 t:$fsm

# 在 03_with_mem_v2.il 对应阶段执行：
select -assert-count 1 t:$mem_v2

# 最终通用映射后：
select -assert-none t:$fsm t:$mem t:$mem_v2 t:$memrd t:$memwr_v2
select -assert-count 16 t:$dffe
```

断言必须放在对应阶段，不能先把 `$fsm` 或 `$mem_v2` 映射掉再要求它存在。

## 调试命令清单

检查 FSM 是否被发现：

```bash
rg -n "Found FSM|Extracting FSM|transition:|mapping auto encoding" fsm_memory.log
```

检查 `$fsm` 的关键参数：

```bash
rg -n -A 30 'cell \$fsm' 02_with_fsm.il
```

检查 memory 的分散端口：

```bash
rg -n '\$memrd|\$memwr|memory width' 01_after_proc.il
```

检查统一的 `$mem_v2`：

```bash
rg -n -A 40 'cell \$mem_v2' 03_with_mem_v2.il
```

检查最终寄存器和 mux 数量：

```bash
rg -n '\$dff|\$dffe|\$mux|addr_decode' 04_after_memory_map.il
```

真正有用的问题不是“文件里有没有某个名字”，而是：

```text
我现在处于宏命令的哪个阶段？
这个高层 cell 是还没生成，还是已经被映射掉？
当前 target flow 是否应该保留 memory 意图？
```

## 一张图总结 FSM 路径

```text
Verilog case + state register
              |
              | proc
              v
$adff + 比较器 + mux/pmux
              |
              | opt -nosdff -nodffe
              | fsm -nomap
              v
$fsm：状态集合 + 输入输出 + 转移表
              |
              | fsm_recode
              v
2 bit binary -> 4 bit one-hot
              |
              | fsm_map
              v
4 bit $adff + 新状态转移逻辑
```

## 一张图总结 Memory 路径

```text
Verilog reg [7:0] mem [0:15]
              |
              | proc
              v
RTLIL::Memory + $memrd + $memwr_v2 + 外部 $dff
              |
              | memory_dff
              v
同步 read port，保留 read-first 语义
              |
              | memory_collect
              v
$mem_v2：16 x 8、1R1W、同步端口、碰撞参数
              |
       +------+-------------------------+
       |                                |
       v                                v
target-specific mapping             memory_map
BRAM / LUT RAM / SRAM macro         FF RAM
                                        |
                                        v
                           16 x 8-bit $dffe
                           + 15 x 8-bit read mux
                           + write address decoder
```

## 总结

FSM 与 memory 的综合都遵循同一种思路：先从普通 RTL 结构中恢复设计意图，在高层 cell 上完成分析和优化，再根据目标硬件映射回可实现结构。

对 FSM：

```text
检测依赖输入网表形状
`opt -nosdff -nodffe` 可以保留适合检测的 FF 结构
`fsm -nomap` 才能在 RTLIL 中观察 `$fsm`
自动重编码可能增加状态 bit，但简化组合逻辑
默认 `fsm` 最后会用 `fsm_map` 消除 `$fsm`
```

对 memory：

```text
`proc` 后 memory 对象与读写端口暂时分开
`memory_dff` 把端口附近寄存器吸收到同步端口
`memory_collect` 生成统一的 `$mem_v2`
`memory -nomap` 为目标 RAM 映射保留高层意图
默认 `memory_map` 会展开成字级 FF、读 mux 和写译码
```

最重要的调试原则是：

> 看不到 `$fsm` 或 `$mem_v2` 时，先检查 pass 的输入形态和宏命令是否已经把它映射掉，不要仅凭最终网表判断识别是否发生过。
