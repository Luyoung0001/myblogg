---
title: "Yosys 学习笔记（四）：RTLIL 中 z 信号与三态门的处理"
date: 2026-07-20 16:03:54
updated: 2026-07-21 10:50:48
tags:
  - "Yosys"
  - "Verilog"
  - "RTLIL"
  - "ASIC"
categories:
  - "Computer Architecture"
---

## 前言

Verilog 有四种基本逻辑状态：`0`、`1`、`x` 和 `z`。前两种很直观，`x` 可以理解为未知值，但 `z` 经常被误解成“另一种未知值”。事实上，`z` 的本意不是一个可以参与计算的数字，而是：**这个驱动器现在没有驱动线路。**

这个区别直接决定了综合器应当如何处理它：

```text
x：数据值未知，可能进入普通逻辑表达式
z：驱动器释放线路，应当用三态结构表达
```

对于 Yosys 来说，问题还要再细分一层。RTLIL 的常量系统确实能够表示 `z`，内部名字是 `State::Sz`；经典 Verilog 前端也可能暂时把 `z` 放在 `$mux` 的一个输入上。但这不等于 `Sz` 可以像普通数据位一样进入 `$sub`、`$add` 或 `$mul`。

本文围绕一条完整的数据路径展开：

```text
Verilog 中的三态赋值
  -> 带 z 输入的 RTLIL $mux
  -> tribuf 识别三态模式
  -> 字级 $tribuf
  -> 逐位 $_TBUF_
  -> ASIC 三态单元、I/O PAD，或 FPGA 中的普通逻辑
```

除了 Yosys 的转换过程，本文还会从 CMOS 晶体管开始解释三态门的物理实现、多驱动总线为什么会冲突、bus keeper 有什么作用，以及为什么“RTLIL 能写 `z`”与“任意 RTLIL cell 都支持 `z`”是两件完全不同的事。

## 先区分 0、1、x 和 z

Verilog 的四态值可以先用下面这张表建立直觉：

| 状态 | Yosys 内部状态 | 主要含义 | 是否代表普通二进制数据 |
|---|---|---|---|
| `0` | `State::S0` | 确定的逻辑低电平 | 是 |
| `1` | `State::S1` | 确定的逻辑高电平 | 是 |
| `x` | `State::Sx` | 未知值或 don't-care | 否，但可以传播进普通表达式 |
| `z` | `State::Sz` | 高阻、未连接或未驱动 | 否，主要描述驱动关系 |

`x` 和 `z` 都不是芯片制造出来的一种稳定逻辑电平，但它们表达的抽象不同。

### x 表示不知道数据是什么

例如：

```verilog
wire [3:0] value = 4'b10x1;
```

这里第三位存在一个驱动者，只是仿真或综合分析无法确定它驱动的是 `0` 还是 `1`。

### z 表示这个驱动器没有参与驱动

例如：

```verilog
assign bus = enable ? data : 1'bz;
```

当 `enable=0` 时，并不是有人向 `bus` 写入了一个名为 `z` 的数据，而是当前输出级断开，让其他设备有机会驱动 `bus`。

因此，下面两句话不能混为一谈：

```text
bus = x：有人在驱动，但值无法确定
bus = z：当前没有有效驱动者
```

## 三态门到底是什么

三态门通常指三态缓冲器。它除了数据输入 `A` 和输出 `Y`，还有一个输出使能 `E`：

```text
          +----------------+
A ------->|                |-------> Y
E ------->| 三态缓冲器     |
          +----------------+
```

它的真值表是：

| `E` | `A` | `Y` |
|---:|---:|---:|
| `0` | `0` | `z` |
| `0` | `1` | `z` |
| `1` | `0` | `0` |
| `1` | `1` | `1` |

用 Verilog 表示就是：

```verilog
assign Y = E ? A : 1'bz;
```

所谓“三态”是指输出级能够表现为：

```text
驱动 0
驱动 1
高阻 Z
```

`x` 是仿真中的未知状态，不是三态门的第三种物理输出。

## CMOS 如何实现三态门

先看普通 CMOS 反相器：

```text
        VDD
         |
       PMOS
         |
         +------ Y
         |
       NMOS
         |
        GND
```

输入为 `0` 时，PMOS 导通，把 `Y` 拉向 `VDD`；输入为 `1` 时，NMOS 导通，把 `Y` 拉向 `GND`。正常情况下，输出总有一条有效的上拉或下拉路径。

三态反相器会在上拉、下拉路径中各增加一个使能晶体管：

```text
             VDD
              |
     P_EN，门极接 E_N
              |
      P_A，门极接 A
              |
              +------ Y
              |
      N_A，门极接 A
              |
     N_EN，门极接 E
              |
             GND
```

其中：

```text
E_N = ~E
```

当 `E=1` 时：

```text
P_EN 导通
N_EN 导通
电路表现为普通反相器
Y = ~A
```

当 `E=0` 时：

```text
P_EN 关闭
N_EN 关闭
Y 到 VDD 的路径断开
Y 到 GND 的路径也断开
```

此时 `Y` 就是高阻状态。非反相三态缓冲器可以在前面增加一级反相器，或者用标准单元库中经过优化的等效晶体管结构实现。

最重要的物理认识是：

> `Z` 不是第三种电压，而是输出级同时关闭上拉和下拉。

## 驱动器输出 z，不一定代表整根线悬空

假设两个三态缓冲器连接到同一根总线：

```text
data_a -> TBUF_A --+
                   +---- BUS
data_b -> TBUF_B --+
```

对应 Verilog：

```verilog
wire bus;

assign bus = enable_a ? data_a : 1'bz;
assign bus = enable_b ? data_b : 1'bz;
```

如果 A 输出 `z`、B 输出 `1`：

```text
TBUF_A：释放总线
TBUF_B：驱动总线为 1
最终 BUS = 1
```

只有在所有驱动器都输出 `z`，并且线路上也没有上拉、下拉或 bus keeper 时，整根总线才真正浮空。

| A 驱动 | B 驱动 | 总线结果 |
|---:|---:|---:|
| `z` | `z` | `z`，可能浮空 |
| `0` | `z` | `0` |
| `z` | `1` | `1` |
| `0` | `0` | `0` |
| `1` | `1` | `1` |
| `0` | `1` | `x`，存在驱动冲突 |

最后一行在真实芯片中尤其危险。一个驱动器试图通过上拉网络连接 `VDD`，另一个驱动器试图通过下拉网络连接 `GND`，可能形成近似直通电流：

```text
VDD
  -> 驱动器 A 的上拉晶体管
  -> BUS
  -> 驱动器 B 的下拉晶体管
  -> GND
```

这会带来大电流、IR drop、电迁移和不确定电压。因此共享三态总线必须保证输出使能互斥，并在驱动权切换时采用 break-before-make：先关闭旧驱动器，再开启新驱动器。

## 为什么 ASIC 还需要 bus keeper

所有驱动器都关闭后，线路上的寄生电容可能暂时保留上一次电平，但漏电和噪声会让电压逐渐漂移。浮空输入还可能停留在 CMOS 门的中间电压区间，使 PMOS、NMOS 同时部分导通，增加静态功耗。

ASIC 中常加入驱动能力很弱的 bus keeper：

```text
BUS -> 弱反馈结构 -> BUS
```

它会弱保持总线最近一次的逻辑值：

```text
总线最后是 1 -> 弱保持为 1
总线最后是 0 -> 弱保持为 0
```

正常三态驱动器开启后，驱动能力远大于 keeper，可以覆盖它。keeper 解决的是“无人驱动时如何避免浮空”，不能解决多个强驱动器互相冲突的问题。

## Verilog 中 z 的三种不同语境

同一个字符 `z` 出现在不同语法位置时，含义可能不同。

### 合法的三态驱动模式

```verilog
assign bus = output_enable ? output_data : 1'bz;
```

这里 `z` 表示释放线路，综合器应识别出三态缓冲器。

### casez 中的匹配通配符

```verilog
casez (opcode)
    4'b10??: result = a;
    4'b01??: result = b;
endcase
```

在 case pattern 的语境中，`z` 或 `?` 可以参与 don't-care 匹配。这不是在构造三态硬件。

### 不应使用的普通算术操作数

```verilog
assign y = 4'b1111 - 4'b00z0;
```

`z` 不是一个数位，不能解释成一个确定的二进制整数。可综合 Verilog 只定义少数三态驱动形态，不应把 `z` 当普通数据参与加减乘除等表达式。

因此阅读含 `z` 的代码时，第一件事不是问“它会变成几个 `x`”，而是先问：

```text
这里是在描述一个驱动器释放线路，
还是错误地把高阻状态当成了数据？
```

## RTLIL 确实能够表示 z

RTLIL 文本常量允许使用：

```text
0  1  x  z  m  -
```

例如：

```text
1'z
4'zzzz
8'0000zzzz
```

在 Yosys C++ 数据结构中，它们对应：

```cpp
State::S0
State::S1
State::Sx
State::Sz
```

其中 `State::Sz` 的含义是 high-impedance / not-connected。

但是必须区分两个命题：

```text
命题一：RTLIL 的 SigBit 和 Const 可以保存 Sz。
命题二：每一种 RTLIL cell 都接受 Sz 作为合法操作数。
```

命题一成立，命题二不成立。`Sz` 对三态驱动、未连接状态和特定 pattern 有意义，但它不是 `$sub`、`$mul` 等普通算术 cell 的常规输入域。

手工写出下面的 RTLIL 在语法上可以被解析：

```text
cell $sub \invalid_sub
  parameter \A_WIDTH 4
  parameter \B_WIDTH 4
  parameter \Y_WIDTH 4
  parameter \A_SIGNED 0
  parameter \B_SIGNED 0
  connect \A 4'1111
  connect \B 4'00z0
  connect \Y \y
end
```

但“解析器能装进数据结构”并不等于“这种 cell 组合属于综合流程承诺支持的语义”。正确的三态结构应当由专门的 `$tribuf` 或 `$_TBUF_` 表达。

## 一个完整的 Yosys 三态实验

使用下面这个四位三态端口：

```verilog
module tristate_demo(
    input  wire       oe,
    input  wire [3:0] data,
    inout  wire [3:0] bus
);
    assign bus = oe ? data : 4'bzzzz;
endmodule
```

准备脚本，分别保存三个阶段的 RTLIL：

```text
read_verilog tristate_demo.v
hierarchy -check -top tristate_demo
write_rtlil tristate_01_read.il

tribuf
write_rtlil tristate_02_tribuf.il

simplemap
write_rtlil tristate_03_simplemap.il

select -assert-count 4 t:$_TBUF_
select -assert-none t:$mux t:$tribuf
```

运行：

```bash
yosys -Q -l tristate_demo.log tristate_demo.ys
```

经典 Verilog 前端会给出一条提醒：

```text
Warning: Yosys has only limited support for tri-state logic at the moment.
```

这不是说简单三态缓冲器一定无法综合，而是提醒使用者：复杂多驱动解析、内部三态和目标器件支持都有边界，应当检查最终网表。

## 第一阶段：前端先生成带 z 输入的 `$mux`

刚执行完 `read_verilog` 后，关键 RTLIL 是：

```text
cell $mux $ternary$1
  parameter \WIDTH 4
  connect \A 4'zzzz
  connect \B \data
  connect \S \oe
  connect \Y $ternary$1_Y
end

connect \bus $ternary$1_Y
```

这是三目运算符的直接结构化表示：

```text
oe = 0 -> 选择 A = zzzz
oe = 1 -> 选择 B = data
```

所以“`z` 不应该出现在任何 RTLIL 中”是不准确的。当前经典前端生成的中间 RTLIL 里，`Sz` 可以暂时存在于 `$mux` 输入；关键是后续 pass 要识别这个受支持的三态语法形状，不让它继续被当作普通数据传播。

## 第二阶段：`tribuf` 把模式变成 `$tribuf`

执行：

```text
tribuf
```

之后，RTLIL 变为：

```text
cell $tribuf $ternary$1
  parameter \WIDTH 4
  connect \A \data
  connect \EN \oe
  connect \Y $ternary$1_Y
end

connect \bus $ternary$1_Y
```

`$tribuf` 是字级三态缓冲器：

```text
输入：A[WIDTH-1:0]
使能：EN
输出：Y[WIDTH-1:0]
```

它定义的函数是：

```verilog
assign Y = EN ? A : 'bz;
```

现在 `z` 的语义不再藏在一个普通 `$mux` 输入里，而是由 cell 类型明确表达：“`EN=0` 时不驱动输出”。

## `tribuf` 是怎样识别模式的

`tribuf` pass 会检查 `$mux` 和 `$_MUX_` 的两个数据输入。

逻辑可以概括为：

```cpp
if (A 全部是 z && B 全部是 z) {
    删除 mux;
}

if (A 全部是 z) {
    data   = B;
    enable = S;
    mux    -> tribuf;
}

if (B 全部是 z) {
    data   = A;
    enable = ~S;
    mux    -> tribuf;
}
```

为什么第二、第三种情况的使能极性不同？因为 `$mux` 的基本语义是：

```text
S=0 -> Y=A
S=1 -> Y=B
```

若 `A=z`，只有 `S=1` 时才选择有效数据 `B`，所以 `enable=S`。若 `B=z`，只有 `S=0` 时才选择有效数据 `A`，所以 `enable=~S`。

这个识别过程依赖语法形状，因此三态驱动最好写成清晰、常见的形式：

```verilog
assign bus = oe ? data : 'z;
```

不要把 `z` 嵌入复杂算术或经过多层难以恢复的组合表达式。

## 第三阶段：`simplemap` 拆成逐位 `$_TBUF_`

`$tribuf` 是字级 cell，而 `$_TBUF_` 是单比特内部 cell。执行：

```text
simplemap
```

四位 `$tribuf` 会被拆成四个 `$_TBUF_`：

```text
cell $_TBUF_ $tbuf0
  connect \A \data [0]
  connect \E \oe
  connect \Y $ternary$1_Y [0]
end

cell $_TBUF_ $tbuf1
  connect \A \data [1]
  connect \E \oe
  connect \Y $ternary$1_Y [1]
end

cell $_TBUF_ $tbuf2
  connect \A \data [2]
  connect \E \oe
  connect \Y $ternary$1_Y [2]
end

cell $_TBUF_ $tbuf3
  connect \A \data [3]
  connect \E \oe
  connect \Y $ternary$1_Y [3]
end
```

两种 cell 的对应关系是：

| cell | 粒度 | 数据端口 | 使能端口 | 输出端口 |
|---|---|---|---|---|
| `$tribuf` | 字级、参数化宽度 | `A` | `EN` | `Y` |
| `$_TBUF_` | 单比特 | `A` | `E` | `Y` |

脚本最后两条结构断言：

```text
select -assert-count 4 t:$_TBUF_
select -assert-none t:$mux t:$tribuf
```

证明最终恰好存在四个逐位三态缓冲器，而且原来的 `$mux` 和字级 `$tribuf` 都已消失。

## `tribuf` 的四种工作模式

`tribuf` pass 除了识别基本模式，还有三个重要选项：

| 命令 | 作用 |
|---|---|
| `tribuf` | 把带全 `z` 输入的 mux 转成三态 buffer |
| `tribuf -merge` | 合并驱动同一网络的多个三态 buffer |
| `tribuf -logic` | 将不驱动模块输出端口的内部三态转换成普通逻辑，并隐含 `-merge` |
| `tribuf -formal` | 将所有三态转换成普通逻辑，并加入多驱动互斥断言，也隐含 `-merge` |

### `-merge`：合并多个三态驱动器

多个三态 buffer 驱动同一网络时，`-merge` 会收集它们的数据和使能，用 `$pmux` 表示“哪个驱动器被选中”，再用所有使能的 OR 表示合并后三态 buffer 的总使能。

概念上：

```text
TBUF(data_a, en_a) --+
                      +-- BUS
TBUF(data_b, en_b) --+
```

变为：

```text
selected_data = pmux(x, {data_a, data_b}, {en_a, en_b})
any_enable    = en_a | en_b
BUS           = any_enable ? selected_data : z
```

这里默认数据使用 `x`，因为没有驱动器使能时，数据端是什么已经不重要，真正决定输出高阻的是 `any_enable=0`。

### `-logic`：消除内部三态

很多目标器件不支持芯片内部三态网络。`tribuf -logic` 会把不直接驱动模块输出端口的三态结构变成 mux 等普通逻辑。仍然需要输出高阻的顶层端口可以保留三态结构，交给 I/O 映射处理。

### `-formal`：把总线仲裁条件变成断言

形式验证工具更适合处理普通逻辑，而不是物理高阻和多驱动解析。`tribuf -formal` 会删除三态 cell，建立 mux 逻辑，并为驱动冲突加入 `$assert`。

对于两个驱动器，核心约束相当于：

```text
assert(!(en_a && en_b));
```

也就是任何时候都不能同时启用两个驱动器。

## 用 `tribuf -formal` 检查多驱动冲突

准备一个共享总线例子：

```verilog
module shared_bus_demo(
    input  wire en_a,
    input  wire en_b,
    input  wire a,
    input  wire b,
    output wire bus
);
    assign bus = en_a ? a : 1'bz;
    assign bus = en_b ? b : 1'bz;
endmodule
```

运行：

```text
read_verilog shared_bus_demo.v
hierarchy -check -top shared_bus_demo
tribuf -formal
stat
write_rtlil shared_bus_formal.il

select -assert-none t:$tribuf t:$_TBUF_
select -assert-count 2 t:$assert
```

这个两驱动例子的实际结构统计是：

```text
9 cells
2   $and
2   $assert
2   $not
1   $pmux
2   $reduce_or
```

三态 cell 已经全部消失，变成选择数据的 `$pmux` 和检查冲突的组合逻辑。当前实现按每个驱动器生成一条局部冲突断言，所以两个驱动器得到两个等价方向的 `$assert`：

```text
驱动器 A 开启时，其他驱动器不能开启
驱动器 B 开启时，其他驱动器不能开启
```

这比单纯在仿真中等待 `bus=x` 更主动，因为仲裁错误被提升成了可以由形式工具证明的设计约束。

## `opt_expr` 如何优化专用三态 cell

三态结构进入 `$tribuf` 或 `$_TBUF_` 后，`opt_expr` 可以根据使能常量进行安全化简。

### 使能恒为 1

```text
E = 1
Y = E ? A : z
```

此时驱动器永远开启：

```text
Y = A
```

`opt_expr` 可以把三态 cell 替换为直接连接并删除 cell。

### 使能恒为 0

```text
E = 0
Y = E ? A : z
```

数据端 `A` 永远不会影响输出。Yosys 可以把 `A` 改成不确定常量以消除无意义的数据依赖，但仍要保留“该驱动器不驱动线路”的三态结构，直到后续 pass 根据目标流程处理它。

这说明专用 cell 的价值不仅是名字更清楚。它把数据 `A`、输出使能 `E` 和高阻行为分开，使优化器能够在不混淆 `x` 与 `z` 的前提下处理常量条件。

## ASIC 标准单元如何描述三态输出

ASIC 标准单元库通常提供类似下面的单元：

```text
TBUF_X1
TBUF_X4
BUFT
INVZ
```

在 Liberty 中，一个非反相三态缓冲器可以概念化地描述为：

```text
cell (TBUF_X1) {
    pin (A) {
        direction : input;
    }

    pin (OE) {
        direction : input;
    }

    pin (Y) {
        direction   : output;
        function    : "A";
        three_state : "!OE";
    }
}
```

这里要注意 `three_state` 表达式描述的是“什么时候输出为高阻”：

```text
OE=0 -> !OE=1 -> Y 进入高阻
OE=1 -> !OE=0 -> Y 按 function=A 驱动
```

Yosys 读取带 `three_state` 属性的 Liberty 输出引脚时，可以建立 `$tribuf` 结构，把库单元的高阻条件纳入 RTLIL 表示。

真正使用哪一种晶体管拓扑、尺寸和驱动强度，由工艺库提供者完成。综合和物理实现关注的是选择合适的标准单元、检查负载和时序，而不是在 RTL 中手工搭 PMOS、NMOS。

## I/O PAD 中的三态

三态最常见、也最自然的使用场景是芯片双向引脚：

```verilog
module gpio(
    input  wire data_out,
    input  wire output_enable,
    output wire data_in,
    inout  wire pad
);
    assign pad = output_enable ? data_out : 1'bz;
    assign data_in = pad;
endmodule
```

行为是：

```text
output_enable=1：芯片驱动 pad
output_enable=0：输出级断开，外部设备可以驱动 pad
data_in：始终通过输入接收器观察 pad
```

Yosys 的 I/O pad 映射可以把 `$_TBUF_` 合并到输出或双向 pad cell 中。相关映射通常需要指定：

```text
输出使能端口
芯片内部的数据输入端口
从 pad 返回的输入数据端口
外部 PAD 端口
```

例如某个 FPGA/ASIC 库可能最终使用 `OBUFT`、`IOBUF`、`TRIBUFF` 或工艺专用 PAD 单元。名称不同，但核心仍是“数据 + 输出使能 + 可选输入接收器”。

## FPGA 为什么经常消除内部三态

大多数现代 FPGA 的内部互连由固定的可编程交换网络组成，不支持任意内部线路真正进入高阻。因此内部三态通常会被转换成 mux 或普通逻辑：

```text
多个内部三态驱动器
  -> 合并使能和数据
  -> mux / pmux
  -> FPGA LUT 与布线资源
```

常见流程会调用：

```text
tribuf -logic
deminout
```

`tribuf -logic` 消除不需要保留到模块输出的内部三态，`deminout` 则根据实际驱动关系处理不再需要双向语义的端口。

芯片顶层 I/O 仍可能支持真正的输出高阻，所以三态结构可以保留到 I/O 映射阶段，再变成器件提供的 `OBUFT` 或 `IOBUF`。

可以把 ASIC 与 FPGA 的常见策略对比为：

| 场景 | 常见实现 |
|---|---|
| ASIC 内部共享总线 | 可以使用三态标准单元，但需严格检查仲裁、STA 和 DFT |
| ASIC 双向 PAD | 映射到带输出使能的 I/O PAD |
| FPGA 内部三态 | 通常转换为 mux/LUT 逻辑 |
| FPGA 顶层三态 | 映射到器件 I/O buffer |

## 为什么 Sz 不应成为算术 cell 的普通输入

现在可以重新审视下面两个例子。

### `Sx` 进入算术表达式

```text
A = 4'b00x0
Y = A - 4'b0001
```

这里有人驱动 `A`，只是其中一位无法确定。`Sx` 作为 unknown/don't-care 在普通表达式中传播，是综合器需要处理的内部情况。

### `Sz` 进入算术表达式

```text
A = 4'b00z0
Y = A - 4'b0001
```

这里的问题不是“这一位到底算 0 还是 1”，而是“谁在驱动这一位”。在完成驱动解析之前，把高阻直接交给减法器没有合理的硬件含义。

正确的建模层次应当是：

```text
多个驱动器及其 enable
  -> 三态 buffer / 驱动解析
  -> 得到真正被接收的 0、1 或未知数据
  -> 再进入普通组合逻辑
```

而不是：

```text
把 z 当成一个二进制数位
  -> 直接送进 $sub
```

所以，RTLIL 中存在 `State::Sz` 并不意味着所有优化 pass 都应该为“任意 cell 输入含 `Sz`”定义完整语义。更稳固的设计边界是：合法三态模式尽早规范化成 `$tribuf` 或 `$_TBUF_`，普通算术 cell 只处理符合其输入约定的数据。

## `-keepdc` 与三态不是同一个问题

`opt_expr -keepdc` 用于限制那些会改变 don't-care 传播方式的优化。例如：

```text
A + 0 -> A
```

在二值逻辑中成立，但如果 `A` 含 `x`，直接连接与算术 cell 可能呈现不同的全宽 unknown 传播。`-keepdc` 会阻止相关优化改变这种行为。

但 `-keepdc` 不会把一个不应出现的 `Sz` 算术操作数自动变成受支持输入，也不会替代三态模式识别。两个概念的关注点不同：

| 机制 | 关注的问题 |
|---|---|
| `-keepdc` | 普通表达式优化是否保留 `x`/don't-care 行为 |
| `$tribuf` / `$_TBUF_` | 某个驱动器是否正在驱动线路 |
| `tribuf -formal` | 多个三态驱动器是否违反互斥条件 |

把 `x` 与 `z` 都叫作 undef 很方便，但在设计三态总线和阅读 RTLIL 时，必须保留这层语义区别。

## 调试 RTLIL 三态结构的实用方法

### 观察前端是否生成带 z 的 mux

```text
read_verilog design.v
write_rtlil before_tribuf.il
select -list t:$mux
```

在 RTLIL 中搜索：

```text
connect \A ...'zzzz
connect \B ...'zzzz
```

### 检查 `tribuf` 是否完成识别

```text
tribuf
write_rtlil after_tribuf.il
select -list t:$tribuf
```

理想情况下，受支持的三态赋值应从带 `z` 输入的 `$mux` 变成 `$tribuf`。

### 检查逐位映射

```text
simplemap
select -list t:$_TBUF_
stat
```

一个 `N` 位 `$tribuf` 通常会产生 `N` 个 `$_TBUF_`。

### 检查内部三态是否被消除

```text
tribuf -logic
deminout
stat
```

具体是否保留顶层三态取决于端口位置和目标架构。

### 把冲突变成形式断言

```text
tribuf -formal
select -list t:$assert
write_rtlil formalized.il
```

如果设计依赖“所有输出使能永不同时为 1”，就应当让这个条件成为可验证的显式性质，而不是只依赖仿真中偶尔出现的 `x`。

## 一张图总结整个处理边界

```text
Verilog：assign bus = oe ? data : 'z
                |
                v
前端 RTLIL：$mux(A='z, B=data, S=oe)
                |
                | tribuf
                v
字级结构：$tribuf(A=data, EN=oe, Y=bus)
                |
                | simplemap
                v
逐位结构：$_TBUF_(A=data[i], E=oe, Y=bus[i])
                |
        +-------+----------------------+
        |                              |
        v                              v
ASIC / 顶层 I/O                  FPGA 内部网络
三态标准单元或 PAD              tribuf -logic
        |                              |
        v                              v
真实输出级可断开                mux / LUT 普通逻辑
```

不应进入这条数据路径的是：

```text
$sub(A=..., B=...z...)
$add(A=...z..., B=...)
$mul(A=..., B=...z...)
```

这些结构把“是否存在驱动者”的问题错误地降成了“某个算术数位是什么”的问题。

## 总结

理解 Yosys 中的 `z`，关键不是记住 `State::Sz` 这个枚举名，而是把值语义和驱动语义分开：

```text
0/1：确定的数据
x：数据存在但未知
z：当前驱动器没有驱动
```

在当前经典 Yosys 流程中，合法的三态赋值可以先形成带全 `z` 输入的 `$mux`，再由 `tribuf` pass 规范化为 `$tribuf`，最后由 `simplemap` 拆成 `$_TBUF_`。后续流程根据目标架构选择保留三态、映射到 I/O PAD，或者转换成 mux/LUT 逻辑。

ASIC 三态门在晶体管层面通过同时关闭上拉和下拉实现高阻；多驱动总线需要互斥使能、break-before-make 和必要的 bus keeper。FPGA 内部通常没有真正三态，因此更常见的是逻辑化处理。

最后应牢记：

> RTLIL 能表示 `Sz`，不等于每一种 RTLIL cell 都支持 `Sz` 作为普通输入。`z` 最有意义的位置是三态驱动边界，而不是算术数据通路。
