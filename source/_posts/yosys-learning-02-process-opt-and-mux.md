---
title: "Yosys 学习笔记（二）：从 process 到 MUX 和 ADFFE，读懂 proc 与 opt"
date: 2026-07-13 15:03:13
tags:
  - "Yosys"
  - "Verilog"
  - "RTLIL"
  - "logic synthesis"
categories:
  - "Computer Architecture"
---

## 前言

上一篇建立了从 Verilog、RTLIL、通用网表到技术映射网表的整体坐标。这一篇进入带状态的设计，研究一个更具体也更容易产生错觉的问题：

```verilog
always @(posedge clk or posedge rst)
```

它在综合后到底是什么？`if/else` 如何变成 MUX？`opt` 为什么会把 `$adff` 变成 `$adffe`？把优化结果手工写回 RTL 是否有意义？交换 `load` 和 `en` 的优先级，为什么 cell 数量不变但功能已经不同？

围绕 `seq_counter` 的几个直接对照，最后得到两条很有价值的判断：

```text
功能等价 != RTLIL 结构或文本相同
结构统计相同 != 连接关系或功能相同
```

本文输出实测于 Yosys 0.66+（git `b35b6706f`）。自动实例名和编号可能随版本变化，阅读时应核对 cell 类型、参数和端口，而不是依赖 `$auto$...$12` 这类完整名字。

## 组合逻辑为什么看起来没有和时钟对齐

先回答最根本的疑惑：组合电路不需要“等到 clk 才运算”。加法器、比较器和 MUX 一直存在，只要输入变化，电信号就持续传播。

时钟控制的是寄存器何时采样，而不是组合逻辑何时开始工作：

```text
t0 时钟沿：上一级寄存器 Q 更新
  -> 经过 clock-to-Q 延迟
  -> 信号穿过加法器、MUX 等组合路径
  -> 在 t1 到来前满足 setup 要求
t1 时钟沿：下一级寄存器采样 D
```

![同步电路中的组合传播与时钟采样](/images/mermaid-svg/yosys-learning-02-process-opt-and-mux/clocked-data-path-timing.svg)

典型寄存器到寄存器路径满足：

```text
Tclk >= Tcq + Tcomb(max) + Tsetup + clock uncertainty
```

因此，“组合电路和 clk 对齐”的准确含义不是组合逻辑在时钟沿才计算，而是它的输出必须在目标采样沿前后满足寄存器的 setup/hold 要求。

异步复位则是另一条控制路径。`rst` 连接触发器的 ARST 端，可以不等待正常时钟沿就把 Q 拉到复位值；复位释放仍要满足 recovery/removal 等时序要求。

## 从源 RTL 写出状态方程

先把下面的完整模块保存为 `seq_counter.v`：

```verilog
module seq_counter(
    input        clk,
    input        rst,
    input        en,
    input        load,
    input  [7:0] load_value,
    output reg [7:0] count,
    output       at_max
);
    assign at_max = (count == 8'hff);

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            count <= 8'h00;
        end else if (load) begin
            count <= load_value;
        end else if (en) begin
            count <= count + 8'h01;
        end
    end
endmodule
```

时序块描述的优先级是：

1. `rst=1`，异步清零。
2. 否则 `load=1`，装载 `load_value`。
3. 否则 `en=1`，计数加一。
4. 否则保持原值。

忽略异步复位的特殊实现，普通数据状态方程可以写成：

```text
count_next = load ? load_value :
             en   ? count + 1  :
                    count
```

这里没有显式写 `count <= count`，但触发器在没有更新条件时天然保持。综合器必须把这个隐含保持关系表达成硬件。

## `proc` 之前：RTLIL 中仍然存在 process

用下面的命令在三个阶段分别保存 RTLIL：

```bash
yosys -p "
read_verilog -sv seq_counter.v;
hierarchy -check -top seq_counter;
write_rtlil counter_01_before_proc.il;
proc;
write_rtlil counter_02_after_proc.il;
opt;
stat;
write_rtlil counter_03_after_opt.il;
"
```

`read_verilog` 已经把连续表达式变成 cell，所以 `at_max` 的比较器和 `count+1` 的加法器已经出现。但 `always` 仍保留为 RTLIL process。`counter_01_before_proc.il` 中的关键部分如下：

```text
process $proc$...
  assign $0\count[7:0] \count
  switch \rst
    case 1'1
      assign $0\count[7:0] 8'00000000
    case
      switch \load
        case 1'1
          assign $0\count[7:0] \load_value
        case
          switch \en
            case 1'1
              assign $0\count[7:0] $add$..._Y
  sync posedge \clk
    update \count $0\count[7:0]
  sync posedge \rst
    update \count $0\count[7:0]
end
```

几个细节很关键：

- 开头先把 next value 默认设为当前 `count`，这就是隐含保持。
- `rst -> load -> en` 被保存为嵌套的 `switch/case`。
- `clk` 和 `rst` 的边沿事件被保存为两个 `sync` rule。

此时的 RTLIL 已经不是 Verilog AST，但仍含有高级过程结构。后续门级映射更适合处理普通 cell 和 wire，因此需要 `proc` 继续 lowering。

## `proc` 不是一个动作，而是一组子 pass

运行 `help proc` 可以看到它按常用顺序调用：

```text
proc_clean
proc_rmdead
proc_prune
proc_init
proc_arst
proc_rom
proc_mux
proc_dlatch
proc_dff
proc_memwr
proc_clean
opt_expr -keepdc
```

在本例中最关键的是：

| 子 pass | 本例作用 |
|---|---|
| `proc_arst` | 从事件和分支中识别异步复位 `rst` |
| `proc_mux` | 把 `if/else` 决策树变成 MUX 网络 |
| `proc_dff` | 为 `count` 创建带异步复位的 `$adff` |
| `proc_clean` | 删除已经转换完的空 process |
| `opt_expr` | 做紧随过程转换的表达式简化 |

Yosys 日志也会逐项记录这条路径：

```text
Found async reset \rst in ...
Creating decoders for process ...
Creating register for signal \count ...
  created $adff cell ... with positive edge clock and positive level reset.
Removing empty process ...
```

## `proc` 之后：MUX 加 `$adff`

`proc` 后 process 已经完全消失，普通数据路径变成：

```text
m_en = en ? count + 1 : count
D    = load ? load_value : m_en
```

`counter_02_after_proc.il` 中，内层 MUX 是：

```text
cell $mux $procmux$4
  connect \A \count
  connect \B $add$..._Y
  connect \S \en
  connect \Y $procmux$4_Y
end
```

因为 `$mux` 的规则是 `Y = S ? B : A`，这正是 `en ? count+1 : count`。

外层 MUX 是：

```text
cell $mux $procmux$7
  connect \A $procmux$4_Y
  connect \B \load_value
  connect \S \load
  connect \Y $procmux$7_Y
end
```

它实现 `load ? load_value : m_en`。最终 `$adff` 的端口关系是：

```text
D    = $procmux$7_Y
Q    = count
ARST = rst
CLK  = clk
```

`$adff` 可以读作 asynchronous-reset D flip-flop：带异步复位的 D 触发器，但还没有显式 EN 端口。

## `opt` 如何把保持路径提取成 EN

`opt` 也不是单一算法，而是 `opt_expr`、`opt_merge`、`opt_muxtree`、`opt_reduce`、`opt_dff`、`opt_clean` 等 pass 的迭代组合。不同设计会触发不同优化，本例最醒目的变化来自 `opt_dff`。

`proc` 后存在反馈关系：

```text
en=0 且 load=0 时，MUX 把 count(Q) 重新送回 D
```

工具识别出这并不是需要真实更新的数据，而是在描述“寄存器保持”。日志会明确写道：

```text
Adding EN signal on ... ($adff) from module seq_counter
```

随后 `$adff` 被改为 `$adffe`：

```text
$adff   = async-reset D flip-flop
$adffe  = async-reset D flip-flop with enable
```

最终 EN 来自：

```text
EN = bool({load, en})
```

在当前 1 位二值输入下等价于：

```text
EN = load || en
```

`counter_03_after_opt.il` 中，`$adffe` 和使能逻辑直接表现为：

```text
cell $adffe $auto$ff.cc:337:slice$12
  parameter \CLK_POLARITY 1
  parameter \EN_POLARITY 1
  parameter \ARST_POLARITY 1
  parameter \ARST_VALUE 8'00000000
  parameter \WIDTH 8
  connect \CLK \clk
  connect \EN $auto$opt_dff.cc:319:make_patterns_logic$13
  connect \ARST \rst
  connect \D $0\count[7:0]
  connect \Q \count
end

cell $reduce_bool $auto$opt_dff.cc:320:make_patterns_logic$14
  parameter \A_WIDTH 2
  parameter \Y_WIDTH 1
  connect \A { \load \en }
  connect \Y $auto$opt_dff.cc:319:make_patterns_logic$13
end
```

沿着 `connect \Y` 和 `connect \EN` 可以看出，`$reduce_bool` 的输出就是 `$adffe.EN`。

数据方程也随之变成：

```text
EN = load || en
D  = load ? load_value : (en ? count+1 : X)
```

当 `EN=0` 时，触发器保持 Q，不采样 D，因此此时 D 可以是 don't-care。RTLIL 中的 `8'x` 不是说芯片上真的生成了一团 X，而是在告诉后续综合：这个条件下的数据值不可观察，可以自由优化。

![seq_counter 从 process 到 adff 再到 adffe 的结构演化](/images/mermaid-svg/yosys-learning-02-process-opt-and-mux/proc-opt-evolution.svg)

这个例子也说明“优化”不等于 cell 数一定减少。`proc` 后可以数到 `$eq + $add + 2×$mux + $adff` 共 5 个 cell；`opt` 后变为 `$eq + $add + 2×$mux + $adffe + $reduce_bool` 共 6 个。结构更适合使用寄存器使能，但统计数量反而增加了一个。

因此，评价优化不能只看 cell 总数，还要看 cell 类型、连接、目标库是否支持相应结构，以及后续映射结果。

## 为什么 EN 要接入 `$adffe`

将 EN 接入 `$adffe` 的逻辑非常直接：

```text
EN=1：时钟沿到来时 Q <- D
EN=0：时钟沿到来时 Q 保持不变
```

原本的反馈 MUX 用 `D=count` 表示保持；带使能触发器直接在状态单元内部表达保持。对于支持 clock-enable 的 FPGA FF 或标准单元，这可能映射到专用 EN pin；如果目标库没有匹配的 enable FF，后续技术映射也可能重新分解为普通 DFF 加反馈逻辑。

所以 `$adffe` 是通用 RTLIL 结构，不保证最终物理库里一定存在一个名字和形状完全相同的单元。

## 手工把 `opt` 结果写回 RTL

现在把相同状态转移直接写成显式使能版本，保存为 `seq_counter_explicit_enable.v`：

```verilog
module seq_counter_explicit_enable(
    input        clk,
    input        rst,
    input        en,
    input        load,
    input  [7:0] load_value,
    output reg [7:0] count,
    output       at_max
);
    wire       update_en  = load || en;
    wire [7:0] next_count = load ? load_value : (count + 8'h01);

    assign at_max = (count == 8'hff);

    always @(posedge clk or posedge rst) begin
        if (rst)
            count <= 8'h00;
        else if (update_en)
            count <= next_count;
    end
endmodule
```

这就是人工写出的等价分解：

```text
update_en = load | en
next_count = load ? load_value : count+1
```

当 `update_en=0` 时，`next_count` 是什么都不会被触发器采样，所以数据端不再需要 `en ? add : X` 这个 MUX。

实际综合统计如下：

| 通用 RTLIL cell | 原始优先级 RTL | 显式使能 RTL |
|---|---:|---:|
| `$add` | 1 | 1 |
| `$adffe` | 1 | 1 |
| `$eq` | 1 | 1 |
| 数据 `$mux` | 2 | 1 |
| 使能逻辑 | 1 `$reduce_bool` | 1 `$logic_or` |
| 总数 | 6 | 5 |

显式版本综合后会出现一个 `$logic_or`、一个数据 `$mux` 和一个 `$adffe`；原版本则保留两个数据 `$mux`。

这说明两件事：

1. 工具确实理解了两种写法中的寄存器使能。
2. 功能等价的 RTL 不保证在某个中间阶段得到逐字相同的 RTLIL，甚至 cell 数都可能不同。

同时也不能仅凭这里少一个粗粒度 MUX，就断言最终芯片面积和频率一定更好。后续 `techmap`、ABC 和目标库映射可能使两者继续收敛；真实 PPA 还取决于库单元、约束和物理实现。

## 形式等价验证，而不是只用肉眼比较

仅仅看到两份网表“长得差不多”不足以证明功能相同。可以在一个全新的 Yosys 进程中读入两份 RTL，综合后执行：

```text
read_verilog -sv seq_counter.v seq_counter_explicit_enable.v
proc
opt
async2sync
opt
equiv_make seq_counter seq_counter_explicit_enable equiv
hierarchy -check -top equiv
equiv_simple -seq 2
equiv_induct -seq 4
equiv_status -assert
```

这里先运行 `async2sync`，是因为 `equiv_simple` 的 SAT backend 不能直接为 `$adffe` 异步复位建立模型。这个正规化只作用于证明副本；前面保存的综合结果仍然保留 `$adffe.ARST`。

最终日志报告：

```text
Found 9 $equiv cells in equiv:
  Of those cells 9 are proven and 0 are unproven.
  Equivalence successfully proven!
```

更精确的表述是：在二值输入、对应状态已经对齐，并采用上述异步复位正规化模型时，两份设计不会继续分叉。这里的 `equiv_induct` 证明的是状态已对齐后的归纳保持性质，并未独立建立任意初始状态下的对齐。

`async2sync` 也不是任意异步事件的精确替代。它假设异步输入实际相对时钟同步，并采用该 pass 规定的 negative hold-time 模型。若要声称“从复位开始、覆盖任意异步时刻”的完整顺序等价，还需要显式复位序列、base case 和更合适的异步事件模型。

脚本没有启用 `-undef`，因此这不是四态 X/Z-aware 证明。原过程式 `if` 的 X optimism 和显式 `?:` 的 X 合并行为可能不同。例如 `load=X, en=1` 时，不应把二值等价结论无限外推到所有 RTL 仿真语义。

## 手工模仿 `opt` 是否有意义

对这个简单例子，通常没有必要为了替工具做局部优化而重写 RTL。常量传播、无用 wire 清理、MUX 简化和使能提取，正是综合器应完成的工作。

更合理的分工是：

### 交给综合工具

- 常量折叠与布尔简化
- 删除未使用 wire/cell
- 合并重复逻辑
- 识别简单反馈保持和寄存器使能
- 根据目标库做局部逻辑优化与技术映射

### 仍由 RTL 设计者负责

- 流水级数和接口延迟
- 数据位宽与数值语义
- 串行复用还是并行计算
- RAM、DSP、clock-enable 等推断友好的编码方式
- 跨时钟域协议与复位架构
- 时序约束和功能验证

工具不能随意增加流水级，因为那会改变可观察延迟；也不一定能跨层次发现所有架构机会。正确态度不是“手写所有门级优化”，也不是“RTL 怎么写都一样”，而是用清晰、规范、利于推断的 RTL 表达架构意图，再让工具完成可证明的局部变换。

## 显式版本有组合路径，原始版本难道没有吗

显式版本把节点写了出来：

```verilog
wire update_en = load || en;
wire [7:0] next_count = load ? load_value : count + 1;
```

因此肉眼很容易看到 OR、加法器和 MUX。原始 `if/else` 看起来只在时钟沿“执行”，但这是源代码造成的错觉。综合后的原版本同样存在：

```text
count(Q) -> 加法器 -> MUX(en) -> MUX(load) -> $adffe.D
load/en  -> OR/ReduceBool -> $adffe.EN
count(Q) -> 比较器 -> at_max
```

给组合节点起名不会凭空增加一级硬件，也不会增加一个周期。`update_en` 和 `next_count` 是并行计算的两条网络，不是先算 EN、再算 D。只有在它们之间插入新的寄存器，才会真正增加流水级。

## 交换优先级：数量相同，连接和功能不同

接着只交换同步分支的顺序。为了不破坏前面的 load-first 基线，把下面模块另存为 `seq_counter_en_first.v`，不要覆盖 `seq_counter.v`：

```verilog
module seq_counter_en_first(
    input        clk,
    input        rst,
    input        en,
    input        load,
    input  [7:0] load_value,
    output reg [7:0] count,
    output       at_max
);
    assign at_max = (count == 8'hff);

    always @(posedge clk or posedge rst) begin
        if (rst)
            count <= 8'h00;
        else if (en)
            count <= count + 8'h01;
        else if (load)
            count <= load_value;
    end
endmodule
```

### 原设计：`load` 优先

原设计综合后的两级 MUX 可以简化表示为：

```text
inner = en   ? count+1   : X
D     = load ? load_value : inner
```

对应的 RTLIL 连接可以直接写成：

```text
cell $mux $inner
  connect \A 8'x
  connect \B $add_Y
  connect \S \en
  connect \Y $inner_Y
end

cell $mux $outer
  connect \A $inner_Y
  connect \B \load_value
  connect \S \load
  connect \Y $next_count
end
```

`load` 控制最靠近 `$adffe.D` 的外层 MUX，所以 `load=1` 会覆盖内层 `en` 的结果。

### 修改后：`en` 优先

修改后，两级 MUX 的输入关系变为：

```text
inner = load ? load_value : X
D     = en   ? count+1    : inner
```

对应 RTLIL 中交换的是两个 MUX 的数据和选择关系：

```text
cell $mux $inner
  connect \A 8'x
  connect \B \load_value
  connect \S \load
  connect \Y $inner_Y
end

cell $mux $outer
  connect \A $inner_Y
  connect \B $add_Y
  connect \S \en
  connect \Y $next_count
end
```

现在 `en` 控制外层 MUX，所以 `en=1` 会覆盖 `load`。

![load 优先与 en 优先的多路器结构比较](/images/mermaid-svg/yosys-learning-02-process-opt-and-mux/priority-mux-comparison.svg)

两者 cell 统计完全相同：

```text
1 × $add
1 × $adffe
1 × $eq
2 × $mux
1 × $reduce_bool
```

假设 `rst=0`，并且 `en/load` 只取 0 或 1，两者的 EN 相同：

```text
EN = load || en
```

因为无论哪个条件优先，只要任意一个为 1，寄存器都需要更新。唯一功能差异出现在两者同时为 1 时：

| `en` | `load` | load 优先 | en 优先 |
|---:|---:|---|---|
| 0 | 0 | 保持 | 保持 |
| 0 | 1 | `load_value` | `load_value` |
| 1 | 0 | `count+1` | `count+1` |
| 1 | 1 | `load_value` | `count+1` |

`en=load=1` 是唯一可能暴露优先级差异的控制组合；如果某一拍恰好满足 `load_value == count+1`，数值结果仍可能碰巧相同。

因此可以得到一个适用于当前粗粒度 RTLIL 的读图规律：

> 对同步 `if/else if` 数据分支，最高优先级条件通常控制最靠近寄存器 D 的外层 MUX。

异步复位是例外。它已经被 `proc_arst` 提取到 `$adff/$adffe.ARST`，不再属于普通数据 MUX 树。

优先级还会改变当前 RTLIL 中的数据路径深度。load 优先时，加法结果经过内外两级 MUX，`load_value` 只经过外层；en 优先时恰好相反。但后续技术映射可以重构逻辑，所以不能只凭这张粗粒度图直接宣布最终 Fmax 高低。

## 一套更可靠的 RTLIL 阅读方法

以后面对更复杂设计，可以按以下顺序阅读：

1. 先找状态单元：`$dff`、`$adff`、`$dffe`、`$adffe`。
2. 顺着 `D/Q/CLK/ARST/EN` 端口找数据与控制来源。
3. 对每个 `$mux` 使用 `Y = S ? B : A` 还原表达式。
4. 检查 `parameter` 中的宽度、极性和复位值。
5. 区分公开 `\name` 与自动 `$name`，不要被自动编号干扰。
6. 对照 `proc` 前后确认 process 是否消失，对照 `opt` 前后确认反馈、常量和别名如何变化。
7. 用 `stat` 观察结构，但用仿真或形式等价验证功能。
8. 在技术映射和 STA 之前，不根据通用 cell 数量过早下最终 PPA 结论。

## 如何运行本文代码

以下命令假设 Yosys 已加入 `PATH`。`rg` 来自 ripgrep，只用于搜索文本，也可以换成 `grep -nE`。

观察 `proc` 和 `opt` 的三个阶段：

```bash
yosys -p "read_verilog -sv seq_counter.v; hierarchy -check -top seq_counter; write_rtlil counter_01_before_proc.il; proc; write_rtlil counter_02_after_proc.il; opt; stat; write_rtlil counter_03_after_opt.il"

rg -n 'process|sync|switch' counter_01_before_proc.il
rg -n '\$adff|\$mux|\$add' counter_02_after_proc.il
rg -n '\$adffe|\$reduce_bool|\$mux' counter_03_after_opt.il
```

单独综合显式使能版本：

```bash
yosys -p "read_verilog -sv seq_counter_explicit_enable.v; hierarchy -check -top seq_counter_explicit_enable; proc; opt; stat; write_rtlil counter_explicit_after_opt.il"
```

单独综合 en-first 版本并保存结果：

```bash
yosys -l en_first.log -p "read_verilog -sv seq_counter_en_first.v; hierarchy -check -top seq_counter_en_first; proc; opt; stat; write_rtlil counter_en_first_after_opt.il"

diff -u counter_03_after_opt.il counter_en_first_after_opt.il
```

把前文的形式等价命令保存为 `equiv.ys` 后运行：

```bash
yosys -l equiv.log equiv.ys
rg -n 'Equivalence successfully proven' equiv.log
```

## 小结

`proc` 把 RTLIL process 中的事件、决策树和赋值关系降成普通 MUX、触发器和连接；`opt` 再对这张网络做迭代简化，并能把反馈保持模式提取成 `$adffe.EN`。这两步仍是通用 RTLIL 变换，不是最终标准单元映射。

更重要的是，源代码中没有显式 `wire next_count`，不代表硬件中没有组合路径；手工写出优化方程，也不保证得到逐字相同的网表。反过来，两份设计拥有相同 cell 统计，也可能因为 MUX 连接和优先级不同而具有不同功能。

读综合结果时，真正可靠的对象始终是：**cell 类型、参数、端口连接、状态转移以及验证证据。**
