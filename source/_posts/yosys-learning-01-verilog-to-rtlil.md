---
title: "Yosys 学习笔记（一）：从 Verilog 到 RTLIL，第一次看懂逻辑综合"
date: 2026-07-12 15:03:13
tags:
  - "Yosys"
  - "Verilog"
  - "RTLIL"
  - "logic synthesis"
categories:
  - "Computer Architecture"
---

## 前言

第一次接触 Yosys 时，我最先困惑的不是某条命令，而是几个看似简单的问题：Verilog 到底是在写程序还是在画电路？Yosys 输出的 `.v` 文件为什么仍像 RTL？RTLIL 是不是类似编译器 IR？技术映射又把什么映射成了什么？

这些问题其实指向同一件事：**逻辑综合不是把一段程序翻译成另一段程序，而是在不同抽象层次上不断改写一张电路图。**

下面直接从两个完整的小模块出发，展示 Yosys 0.66+ 中实际可见的转换。目标不是背完命令，而是建立一条可以反复使用的主线：

```text
Verilog 行为/结构描述
  -> RTLIL 中间表示
  -> 通用 cell 网络
  -> 技术映射后的目标器件网表
  -> 布局布线与物理签核
```

![从 Verilog 到物理实现的 Yosys 综合流水线](/images/mermaid-svg/yosys-learning-01-verilog-to-rtlil/yosys-synthesis-pipeline.svg)

## Verilog 描述的是硬件，不是普通程序

首先要建立的认识是：

```verilog
assign y = a + b;
```

它不是 CPU 运行到这里时做一次加法，而是描述一个一直存在的组合加法器。只要 `a` 或 `b` 改变，信号就会沿加法器传播到 `y`。

同样：

```verilog
always @(posedge clk)
    q <= d;
```

也不是注册了一个软件回调。综合器会据此推断触发器：组合逻辑在两个时钟沿之间持续传播，触发器在有效时钟沿采样 `D`，然后更新 `Q`。

编译器类比可以帮助入门，但必须知道边界：

| 软件工具链 | 数字电路工具链 | 类比的边界 |
|---|---|---|
| C/C++ 源码 | Verilog RTL | RTL 描述并行电路和状态，不是指令序列 |
| 编译器 IR | RTLIL / 通用网表 | 两者都便于统一分析和改写 |
| 目标 ISA 汇编 | 工艺单元网表 | 网表是连接图，没有逐条执行的 PC |
| 机器运行 | 芯片随时工作 | 电路单元天然并行活动 |

因此，说“RTLIL 像 IR、技术映射网表像目标相关代码”是有用的；说“网表就是硬件汇编”则容易让人误以为门会按顺序执行。

## 两种不同语境下的前端和后端

讨论 Yosys 时，“前端/后端”经常混用。至少要区分两套边界。

### Yosys 软件内部的 frontend、pass 和 backend

Yosys frontend 负责把输入格式转换成内存里的 RTLIL design，例如 `read_verilog`。大多数 pass 随后读取或修改 RTLIL。Yosys backend 则把当前 design 写成 Verilog、RTLIL、JSON、BLIF 等格式，例如 `write_verilog` 和 `write_rtlil`。

从 Yosys 源码目录也能看到这层分工：`frontends/` 读入设计，`passes/` 改写 RTLIL，`backends/` 输出设计。

这里的 backend 只是“Yosys 的输出模块”，不等于芯片物理后端。

### IC 设计流程里的前端和后端

行业里常见的划分大致是：

```text
IC 前端：规格 -> 微架构 -> RTL -> 仿真/lint/formal/CDC -> 逻辑综合
IC 后端：floorplan -> placement -> CTS -> routing -> RCX/STA -> DRC/LVS -> GDSII
```

不同公司的边界会略有差异，但逻辑综合通常位于 IC 前端的后半段。Yosys 主要解决这一段的 RTL 综合、网表变换和部分形式验证问题；它不是完整的 place-and-route 或 signoff 工具。

这里顺便解释 CDC。CDC 是 Clock Domain Crossing，即信号跨越不同或无固定相位关系的时钟域。单比特控制信号常用同步器，多比特数据常用握手或异步 FIFO。CDC 检查属于前端验证的重要工作，但它和“组合逻辑是否需要时钟”不是一回事：组合逻辑本身不需要时钟，跨时钟域采样才需要专门处理亚稳态与数据一致性。

## 一个可以直接运行的最小例子

以下命令假设使用 Yosys 0.66+，并且 `yosys` 已加入 `PATH`。可以先运行 `yosys -V` 确认版本；自动对象名称和少量规范化结果可能随版本变化。请在一个新目录中保存本文代码并执行命令，结尾使用的 `rg` 只负责搜索文本，不参与综合。

先把下面代码保存为 `tiny_adder.v`：

```verilog
module tiny_adder(
    input  [3:0] a,
    input  [3:0] b,
    input        sub,
    output [4:0] y
);
    assign y = sub
        ? ({1'b0, a} - {1'b0, b})
        : ({1'b0, a} + {1'b0, b});
endmodule
```

然后运行一条独立的 Yosys 命令：

```bash
yosys -p "read_verilog -sv tiny_adder.v; hierarchy -check -top tiny_adder; proc; opt; stat; write_rtlil tiny_adder.il; write_verilog -noattr tiny_adder_netlist.v"
```

这行同时包含 shell 语法和 Yosys 语法：

| 片段 | 所属层 | 含义 |
|---|---|---|
| `yosys` | shell | 启动 Yosys；若不在 `PATH` 中，可替换为实际二进制路径 |
| `-p "..."` | Yosys CLI | 直接执行一整段 Yosys command string |
| `;` | Yosys script | 依次分隔多个 pass，不是此处 shell 的命令分隔符 |

引号中的每一步都有清晰职责：

```text
read_verilog -sv ...       读取 SystemVerilog，生成 RTLIL design
hierarchy -check -top ...  指定顶层并检查模块层次
proc                       将 process 降为 mux、FF、latch 等 cell
opt                        迭代运行一组简单优化与清理 pass
stat                       统计当前 wire、bit 和 cell
write_rtlil ...            将当前 design 写为 RTLIL 文本
write_verilog -noattr ...  将当前 design 写为 Verilog，不输出 attribute
```

`stat` 只能描述“当前结构里有什么”，不能证明功能正确。两个错误设计可以有完全相同的 cell 统计；反过来，两个功能等价设计也可能具有不同的中间结构。

如果希望减少终端日志，可以加 `-q`；如果希望保存完整日志，可以加 `-l yosys.log`。如果想输出 Graphviz DOT，可以在命令串中加入 `show -format dot -prefix tiny_adder`。`xdot` 只是 DOT 图的交互查看器；没有它仍可生成 DOT 文本，也不影响综合。将 DOT 渲染为 SVG 还需要 Graphviz 的 `dot` 程序。

## tiny_adder：第一次把表达式还原成 cell 图

这个模块的核心是一条条件表达式：

```verilog
assign y = sub
    ? ({1'b0, a} - {1'b0, b})
    : ({1'b0, a} + {1'b0, b});
```

`a` 和 `b` 是 4 位，输出 `y` 是 5 位。显式拼接 `1'b0` 把两个操作数扩展为 5 位，使无符号位宽意图不依赖外层上下文。

一个值得保留的严谨性是：不能简单断言“只要删掉 `{1'b0, a}`，这个版本一定丢进位”。在当前 5 位赋值上下文中，Yosys 仍可能把结果扩为 5 位。真正稳妥的结论是：显式扩展让意图明确；如果先把结果存入 4 位中间 wire，再扩到 5 位，最高位就已经丢失。

运行后，`tiny_adder.il` 中能直接找到三个 cell：

- 一个 `$sub`
- 一个 `$add`
- 一个 `$mux`

`stat` 对这个模块的核心统计是：

```text
=== tiny_adder ===
3 cells
1   $add
1   $mux
1   $sub
```

![tiny_adder 从 Verilog 表达式到 RTLIL cell 网络](/images/mermaid-svg/yosys-learning-01-verilog-to-rtlil/tiny-adder-rtlil-network.svg)

Yosys `$mux` 的连接规则是：

```text
Y = S ? B : A
```

这里的连接是：

```text
A = add_result
B = sub_result
S = sub
Y = y
```

所以它准确还原为 `y = sub ? sub_result : add_result`。读 RTLIL 不能只看 cell 名称，还要一起看参数和 `connect`。

## 逐行读 RTLIL

tiny_adder 的 RTLIL 头部是：

```text
autoidx 4
attribute \src "tiny_adder.v:1.1-10.10"
attribute \top 1
module \tiny_adder
```

这些字段分别表示：

- `autoidx 4`：后续创建自动对象时使用的编号起点。
- `attribute \src ...`：源码位置元数据，方便日志、调试和交叉定位。
- `attribute \top 1`：标记当前顶层模块。
- `module \tiny_adder`：开始一个模块；反斜杠开头的是 RTLIL 公开标识符。

接下来是 wire：

```text
wire width 4 input 1 \a
wire width 5 output 4 \y
wire width 5 $sub$tiny_adder.v:8$1_Y
```

前两个通常保留用户源码名字。美元符号开头的名字通常由 Yosys 自动生成。

`$sub$tiny_adder.v:8$1_Y` 可以这样拆：

```text
$sub          来自减法表达式
tiny_adder.v  源文件名
:8            来源大致在源码第 8 行
$1            frontend 分配的唯一自动编号
_Y            这是相应 cell 的 Y 输出临时 wire
```

`$1_Y` 不是“第 1 位 Y”，也不是循环执行次数，只是自动命名的一部分。

cell 本体则由类型、实例名、参数和端口连接构成。下面为了突出结构省略了部分参数，并用 `...` 缩短自动名字，因此它是阅读用删节，不是可以直接交给 `read_rtlil` 的完整文件：

```text
cell $add $add$...
  parameter \A_WIDTH 5
  parameter \B_WIDTH 5
  parameter \Y_WIDTH 5
  connect \A { 1'0 \a }
  connect \B { 1'0 \b }
  connect \Y $add$..._Y
end
```

### RTLIL 是否必须和原 Verilog 放在一起

不需要。`attribute \src` 是来源信息，不是运行时依赖。只要 RTLIL 文件完整，就可以用 `read_rtlil` 重新加载其中的模块、wire、cell、参数和连接，即使原始 `.v` 文件已经不存在，综合后的逻辑仍然成立。

但“可以重建逻辑”不等于“可以恢复原始源码”。从 RTLIL 可以重新输出功能等价的 Verilog，却通常无法恢复原注释、排版、变量命名、宏、循环、函数和原始 `always` 写法。综合是多对一变换，很多不同 RTL 都会收敛到相同或等价的网络。

## 为什么 netlist Verilog 看起来仍像 RTL

`tiny_adder_netlist.v` 中仍然能看到：

```verilog
assign _0_ = { 1'h0, a } + { 1'h0, b };
assign _1_ = { 1'h0, a } - { 1'h0, b };
assign y = sub ? _1_ : _0_;
```

它看起来和 RTL 很像，有两个原因。

第一，**网表是一张 cell 与 net 的连接图，不是一种特定文件后缀。** Verilog 既能表达高层 RTL，也能承载结构网表。

第二，`write_verilog` 默认会把许多内部 cell 重新打印成 Verilog expression。`-noattr` 只是移除 attribute，不会强制输出门实例。如果加上 `write_verilog -noexpr`，内部 cell 会更直接地以结构形式出现。

这个例子仍是 `$add/$sub/$mux` 组成的通用字级网表，还没有绑定 NAND、AOI 或某个标准单元库，所以它本来就不会像大规模门级网表那样复杂。

## 组合 ALU：连续赋值何时变成 RTLIL cell

再看一个完整的组合 ALU，把下面代码保存为 `alu_slice.v`：

```verilog
module alu_slice(
    input  [7:0] a,
    input  [7:0] b,
    input  [1:0] op,
    output [7:0] y,
    output       is_zero
);
    wire [7:0] add_y  = a + b;
    wire [7:0] and_y  = a & b;
    wire [7:0] xor_y  = a ^ b;
    wire [7:0] pass_y = a;

    assign y =
        (op == 2'b00) ? add_y :
        (op == 2'b01) ? and_y :
        (op == 2'b10) ? xor_y :
                        pass_y;

    assign is_zero = (y == 8'h00);
endmodule
```

可以在 `hierarchy`、`proc`、`opt` 后各保存一次 RTLIL：

```bash
yosys -p "read_verilog -sv alu_slice.v; hierarchy -check -top alu_slice; write_rtlil alu_01_hierarchy.il; proc; write_rtlil alu_02_proc.il; opt; stat; write_rtlil alu_03_opt.il"
```

最重要的观察是：`hierarchy` 后的第一份 RTLIL 已经包含 `$add`、`$and`、`$xor`、比较器和三级 `$mux`。这说明连续表达式在 `read_verilog` 阶段就已经下降成 RTLIL cell；`proc` 并不负责所有表达式 lowering，它主要针对 `process`。

这个设计没有 `always`，所以 `proc` 没有 process 可以转换成触发器。不过也不能说“proc 前后完全不变”。`proc` 是一个宏命令，末尾还会运行 `opt_expr`。本版本会把 `op == 0` 和 `y == 0` 从 `$eq` 规范化为 `$logic_not`。

例如 `op == 2'b00` 在较早快照中可能是：

```text
cell $eq $eq$alu_slice.v:14$4
  connect \A \op
  connect \B 2'00
  connect \Y $eq$alu_slice.v:14$4_Y
end
```

经过 `proc` 尾部的 `opt_expr` 后变成：

```text
cell $logic_not $eq$alu_slice.v:14$4
  connect \A \op
  connect \Y $eq$alu_slice.v:14$4_Y
end
```

对 2 位信号来说，“等于零”就是“所有位都不为 1”，因此可以用逻辑非表达。cell 的实例名仍可能保留 `$eq$...` 的来源痕迹，但 cell 类型已经是 `$logic_not`。

这引出一个比背命令更重要的习惯：

> 不要只问“运行了哪个 pass”，还要问“当前 design 中有没有它处理的对象，它的子 pass 实际改了什么”。

另外，`op` 是运行时输入，`op == 2'b01` 不可能在综合时直接“算完”；它必须成为比较逻辑。`y == 0` 同理，它依赖前面的 MUX 结果。把数据位宽从 8 改成 16 时，字级 `$add/$and/$xor/$mux` 的数量也不一定翻倍，首先改变的是 cell 的宽度参数；映射到位级门后，资源数量才通常随位宽增长。

## 技术映射：输入和输出分别是什么

到目前为止，我们看到的仍是通用 RTLIL cell。技术映射解决的是：如何用目标器件真正提供的资源实现这些通用运算。

可以把输入和输出写成下面的集合。具体使用哪些输入取决于映射阶段，并不是每条映射命令都同时需要模板、Liberty 和约束：

```text
输入：通用 cell 网络
    + 映射模板或目标器件规则
    + Liberty 标准单元库
    + 延迟/负载等约束

输出：由目标标准单元或 FPGA LUT/FF 组成的网表
    + 每个实例的类型、参数和连接
```

三个常见命令各有不同职责：

- `techmap`：用 Verilog/RTLIL 模板替换匹配的 cell。无 `-map` 时使用 Yosys 内置映射，把较粗粒度内部 cell 降为内部门库；模板来自 Yosys 自带 `techlibs`、厂商库或用户提供的 map 文件。
- `dfflibmap -liberty cells.lib`：专门把内部触发器映射到 Liberty 中可用的触发器，必要时还可能引入反相器。
- `abc -liberty cells.lib`：调用 ABC 对抽取出的组合逻辑做优化，并映射到 Liberty 描述的组合标准单元。

常见顺序是先用 `dfflibmap` 处理寄存器，再用 `abc -liberty` 处理组合路径，因为触发器映射可能引入需要继续映射的反相器。

技术映射后的网表已经面向目标工艺，但还不是芯片版图。后续通常还要进行网表等价检查、约束检查、DFT、布局、时钟树综合、布线、寄生提取、STA、DRC 和 LVS。

## 频率、面积和功耗在综合阶段如何估算

综合阶段的 PPA 是早期估计，不是最终签核数字。前面的两个例子没有读取 Liberty，也没有时钟与 IO 约束，因此只能观察通用 cell 结构，不能得到可信面积、Fmax 或功耗；基础 `stat` 也不是完整 STA 或功耗分析器。

### 面积

映射后可以根据 Liberty 中的 cell area 累加组合门和触发器面积。它适合比较不同实现，但没有完整考虑宏单元形状、布线拥塞、tap/filler、时钟树和最终 core utilization，因此不能直接等同于芯片面积。

### 频率

频率来自最长时序路径。综合器利用目标库中的 timing arc、输入驱动、输出负载和时序约束估计组合延迟，典型寄存器到寄存器约束是：

```text
Tclk >= Tcq + Tcomb(max) + Tsetup + clock uncertainty
Fmax ≈ 1 / Tcritical
```

布局前缺少真实线长和寄生 RC，所以这是 pre-layout timing。布局布线、RCX 以后才能得到更可信的 post-layout STA。

### 功耗

动态功耗近似依赖：

```text
Pdynamic ≈ activity × capacitance × voltage^2 × frequency
```

还要加上 cell leakage 和 internal power。若没有真实切换活动率、频率、电压、负载以及库中的功耗模型，只能做粗略估计。Yosys 的基础 `stat` 更不能单独给出 signoff power。

## 这一阶段最值得保留的判断标准

到这里可以形成几条不会轻易过时的判断：

1. Verilog 语法像程序，不代表硬件按语句顺序执行。
2. RTLIL 是 Yosys 的统一设计表示，主要 pass 围绕它工作。
3. 网表由 cell/net 图的语义定义，不由 `.v`、`.il` 或代码长相定义。
4. 通用网表类似 IR，技术映射网表绑定目标库，但两者都是并行电路图。
5. `stat` 适合观察结构变化，不负责证明功能正确。
6. `.il` 可以独立恢复综合后的网络，不能无损恢复原始 RTL。
7. PPA 必须结合目标库和约束；布局前的结果仍是估计。

## 完整运行命令

```bash
yosys -p "read_verilog -sv tiny_adder.v; hierarchy -check -top tiny_adder; proc; opt; stat; write_rtlil tiny_adder.il; write_verilog -noattr tiny_adder_netlist.v"

yosys -p "read_verilog -sv alu_slice.v; hierarchy -check -top alu_slice; write_rtlil alu_01_hierarchy.il; proc; write_rtlil alu_02_proc.il; opt; stat; write_rtlil alu_03_opt.il"

rg -n '\$add|\$sub|\$mux' tiny_adder.il
rg -n '\$add|\$and|\$xor|\$eq|\$logic_not|\$mux' alu_*.il
```

## 小结

这一篇建立了逻辑综合的坐标系：Verilog 是电路描述入口，RTLIL 是 Yosys 统一处理的中间表示，pass 持续改写 cell 与 wire 网络，backend 再把当前 design 序列化为不同格式。技术映射把通用网络绑定到目标器件，而布局布线和签核仍在后面。

下一篇进入真正有状态的设计：观察 `always @(posedge clk or posedge rst)` 如何经 `proc` 变成 MUX 和 `$adff`，`opt` 又为什么能把反馈保持路径提取成 `$adffe.EN`。
