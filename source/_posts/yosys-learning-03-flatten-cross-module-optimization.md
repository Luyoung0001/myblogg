---
title: "Yosys 学习笔记（三）：flatten 与 opt 如何联手把整个设计化成常量"
date: 2026-07-14 15:03:13
tags:
  - "Yosys"
  - "Verilog"
  - "RTLIL"
  - "logic synthesis"
categories:
  - "Computer Architecture"
---

## 前言

设想这样一个层次化设计：

- 子模块 A 只在输入为奇数时输出 `1`；
- 子模块 B 只在输入为偶数时输出 `1`；
- 顶层把两个结果做或运算。

对任何普通二进制输入，一个数不是奇数就是偶数，所以顶层输出必然为 `1`。源码里明明写了两个子模块、一个取反和一个或运算，综合器最终能不能发现“整个设计其实只是常量 `1`”？

答案是：**能，但不是 `flatten` 单独完成，也不是第一次 `opt` 就能完成。**

真正发生的是一组分工明确的结构变换：

```text
保留层次
  -> opt：只清理各模块内部，跨模块关系仍不可见
  -> flatten：复制子模块实现，消除模块边界
  -> opt：识别 x | ~x = 1，删除 $or 和 $not
  -> clean -purge：删除残余层次元数据和无用公开线网
  -> 顶层只剩 always_one = 1，功能 cell 数量为 0
```

![flatten 与 opt 的五阶段结构演化](/images/mermaid-svg/yosys-learning-03-flatten-cross-module-optimization/flatten-opt-stage-evolution.svg)

这个例子看上去很小，却把逻辑综合中的三个关键概念连在了一起：**层次是优化边界，展平是在改变优化视野，`opt` 才是在改写布尔结构。**

本文结果实测于 Yosys 0.66+（git `b35b6706f`）。不同版本生成的自动名称和编号可能变化，阅读时应关注 cell 类型、连接关系和常量，而不是依赖完整的 `$auto$...` 名字。

## 完整的层次化 Verilog

先把下面代码保存为 `parity_cover.v`：

```verilog
module odd_detector(
    input  [7:0] value,
    output       is_odd
);
    assign is_odd = value[0];
endmodule

module even_detector(
    input  [7:0] value,
    output       is_even
);
    assign is_even = ~value[0];
endmodule

module parity_cover(
    input  [7:0] value,
    output       always_one
);
    wire odd_hit;
    wire even_hit;

    odd_detector odd_part(
        .value(value),
        .is_odd(odd_hit)
    );

    even_detector even_part(
        .value(value),
        .is_even(even_hit)
    );

    assign always_one = odd_hit | even_hit;
endmodule
```

这里不需要真的计算八位整数的除法余数。对二进制整数而言，最低位已经决定奇偶性：

| `value[0]` | `odd_hit` | `even_hit` | `always_one` |
|---:|---:|---:|---:|
| `0` | `0` | `1` | `1` |
| `1` | `1` | `0` | `1` |

令 `x = value[0]`，顶层逻辑就是：

```text
odd_hit    = x
even_hit   = ~x
always_one = x | ~x = 1
```

从人的视角看，这个结论一眼就能得到。问题在于，第一次读入设计时，这三行关系分散在三个模块里。Yosys 是否能做出同样的判断，取决于当前 pass 能看见多大的结构范围。

## 一次保存五个阶段的 Yosys 脚本

再把下面脚本保存为 `parity_cover.ys`。它会把每个关键阶段都写成 RTLIL 和 Verilog，便于直接比较：

```text
read_verilog -sv parity_cover.v
hierarchy -check -top parity_cover
stat
write_rtlil 01_hierarchy.il
write_verilog -noattr 01_hierarchy.v

opt
stat
write_rtlil 02_hierarchy_opt.il
write_verilog -noattr 02_hierarchy_opt.v

flatten
stat
write_rtlil 03_flattened.il
write_verilog -noattr 03_flattened.v

opt
stat
write_rtlil 04_flattened_opt.il
write_verilog -noattr 04_flattened_opt.v

clean -purge
stat
sat -verify -prove always_one 1 -show value
write_rtlil 05_constant.il
write_verilog -noattr 05_constant.v
```

运行：

```bash
yosys -V
yosys -Q -l parity_cover.log parity_cover.ys
```

`-Q` 只省略启动横幅，不会像 `-q` 那样静默 `stat`；`-l parity_cover.log` 会把完整日志保存下来。若安装了 ripgrep，可以分别定位统计、模块删除和各阶段 cell：

```bash
rg -n -A 14 '^=== parity_cover ===' parity_cover.log
rg -n 'Deleting now unused module|SAT proof finished' parity_cover.log
rg -n '\$not|\$or|\$scopeinfo|connect \\always_one' 0*.il
```

五个阶段的核心结果如下：

| 阶段 | 子模块实例 | 功能 cell | 元数据 cell | `always_one` |
|---|---:|---|---|---|
| `hierarchy` | 2 | `$not`、`$or` | 0 | 由 `$or` 驱动 |
| 第一次 `opt` | 2 | `$not`、`$or` | 0 | 仍由 `$or` 驱动 |
| `flatten` | 0 | `$not`、`$or` | 2 个 `$scopeinfo` | 仍由 `$or` 驱动 |
| 第二次 `opt` | 0 | 0 | 2 个 `$scopeinfo` | 常量 `1'1` |
| `clean -purge` | 0 | 0 | 0 | 常量 `1'1` |

这里把 `$scopeinfo` 单独列出，是因为它保存来源和层次信息，不是会被制造成门电路的功能逻辑。

## 第一阶段：hierarchy 建立模块关系，但不展开实现

`read_verilog` 把三个 Verilog 模块转换成 RTLIL module。随后：

```text
hierarchy -check -top parity_cover
```

主要完成三件事：

1. 指定 `parity_cover` 是顶层；
2. 解析并规范化实例接口，检查引用的 module 是否存在；端口宽度异常可能被调整或产生 warning；
3. 删除从顶层不可达的无用模块。

这一步不会把子模块内容复制进顶层。省略属性和部分参数后，`01_hierarchy.il` 的关键结构是：

```text
module \odd_detector
  wire width 8 input 1 \value
  wire output 2 \is_odd
  connect \is_odd \value [0]
end

module \even_detector
  wire width 8 input 1 \value
  wire output 2 \is_even

  cell $not $not$parity_cover.v:12$1
    connect \A \value [0]
    connect \Y $not$parity_cover.v:12$1_Y
  end

  connect \is_even $not$parity_cover.v:12$1_Y
end

module \parity_cover
  cell \odd_detector \odd_part
    connect \value \value
    connect \is_odd \odd_hit
  end

  cell \even_detector \even_part
    connect \value \value
    connect \is_even \even_hit
  end

  cell $or $or$parity_cover.v:32$2
    connect \A \odd_hit
    connect \B \even_hit
    connect \Y $or$parity_cover.v:32$2_Y
  end

  connect \always_one $or$parity_cover.v:32$2_Y
end
```

这段 RTLIL 揭示了一个重要事实：**模块实例在 RTLIL 中也是 cell。**

- `$not` 和 `$or` 是 Yosys 内部 cell 类型；
- `\odd_detector` 和 `\even_detector` 是用户定义的 cell 类型；
- `\odd_part` 和 `\even_part` 是两个实例名。

顶层当前只知道：`odd_hit` 来自一个 `odd_detector` 实例，`even_hit` 来自一个 `even_detector` 实例，然后二者进入 `$or`。子模块的定义虽然也存在于 design 中，但尚未成为顶层内部的一张连续布尔网络。

## 为什么第一次 opt 没有把输出化成 1

第一次执行：

```text
opt
```

生成结果的顶层部分仍是下面这样；两个子模块定义也仍在输出文件中，这里没有重复展开：

```verilog
module parity_cover(
    input  [7:0] value,
    output       always_one
);
    wire odd_hit;
    wire even_hit;

    odd_detector  odd_part  (.value(value), .is_odd(odd_hit));
    even_detector even_part (.value(value), .is_even(even_hit));

    assign always_one = odd_hit | even_hit;
endmodule
```

一些匿名中间 wire 会被合并，但两个实例、`$not` 和 `$or` 仍然存在。这不是 `opt` 不够聪明，而是它此时面对的是三个分别优化的 module：

```text
odd_detector 内部：   is_odd  = value[0]
even_detector 内部：  is_even = ~value[0]
parity_cover 内部：   always_one = odd_hit | even_hit
```

在顶层模块内部，`odd_hit` 和 `even_hit` 只是两个子模块输出。仅看顶层连接，任何下面这些实现都可能藏在子模块里：

```text
odd_hit = x      even_hit = ~x    -> 输出恒为 1
odd_hit = x      even_hit = x     -> 输出等于 x
odd_hit = 0      even_hit = 0     -> 输出恒为 0
```

如果不把实例实现带入顶层，`opt` 就不能擅自假设两个端口互补。模块边界在这里形成了一个真实的优化边界。

### opt -hier 能不能替代 flatten

`opt -hier` 会额外调用 `opt_hier`，可以做一些层次相关清理。例如当前设计的两个检测器都只使用 `value[0]`，它可以断开实例输入端口中没有被子模块使用的高七位。

但在当前 Yosys 版本中，下面这组结构仍会保留：

```text
2 个子模块实例 + 1 个 $not + 1 个 $or
```

原因是 `opt_hier` 不是通用的模块内联器。它能利用部分层次信息做端口和参数层面的清理，却不会像 `flatten` 那样把两个子模块的实现完整复制到顶层。因此，对本例来说，`opt -hier` 仍不能替代 `flatten`。

## flatten 到底做了什么

现在执行：

```text
flatten
```

Yosys 的帮助把它描述为：用 cell 的 implementation 替换 cell。它与 `techmap` 有些相似，区别是 `flatten` 把**当前 design 本身**当作映射库。

对 `odd_part`，它把 `odd_detector` 内部的：

```text
is_odd = value[0]
```

复制到顶层；对 `even_part`，它把：

```text
is_even = ~value[0]
```

复制到顶层。完成替换以后，原来的两个实例已经不存在，而两个子模块定义也不再被任何实例使用，所以日志会出现：

```text
Deleting now unused module even_detector.
Deleting now unused module odd_detector.
```

这两行很容易让人误解。它们的意思不是“奇偶判断逻辑已经被优化掉”，而是“子模块定义已经被内联，旧的独立 module 定义不再需要”。

`03_flattened.il` 仍然明确包含 `$not` 和 `$or`：

```text
module \parity_cover
  wire width 8 input 1 \value
  wire output 2 \always_one

  wire \odd_part.is_odd
  wire width 8 \odd_part.value
  wire \even_part.is_even
  wire width 8 \even_part.value

  cell $or $or$parity_cover.v:32$2
    connect \A \odd_hit
    connect \B \even_hit
    connect \Y \always_one
  end

  cell $not $flatten\even_part.$not$parity_cover.v:12$1
    connect \A \even_part.value [0]
    connect \Y \even_part.is_even
  end

  connect \odd_part.is_odd \odd_part.value [0]
  connect \odd_hit \odd_part.is_odd
  connect \odd_part.value \value
  connect \even_hit \even_part.is_even
  connect \even_part.value \value
end
```

生成的 Verilog 也能直观看见同一件事：

```verilog
module parity_cover(value, always_one);
  input [7:0] value;
  output always_one;
  wire even_hit;
  wire odd_hit;
  wire \even_part.is_even ;
  wire [7:0] \even_part.value ;
  wire \odd_part.is_odd ;
  wire [7:0] \odd_part.value ;

  assign \even_part.is_even  = ~ \even_part.value [0];
  assign always_one = odd_hit | even_hit;
  assign \odd_part.is_odd  = \odd_part.value [0];
  assign odd_hit = \odd_part.is_odd ;
  assign \odd_part.value  = value;
  assign even_hit = \even_part.is_even ;
  assign \even_part.value  = value;
endmodule
```

这里名称中的 `odd_part.` 和 `even_part.` 是原层次被编码进展平后对象名的结果。模块边界已经消失，但 Yosys 尽量保留名称，以便追溯来源。

所以必须把结论说准确：

> `flatten` 不是布尔化简 pass。它的作用是消除用户模块实例边界，让原来分散在多个 module 中的逻辑出现在同一个 module 里。

## 第二次 opt 为什么突然能把整个逻辑删掉

展平以后，顶层内部出现了一条连续关系：

```text
value[0] -----------------------> odd_hit ----+
                                                OR ---> always_one
value[0] ---> NOT ---> even_hit --------------+
```

也就是：

```text
always_one = value[0] | ~value[0]
```

现在第二次执行 `opt`，`opt_expr` 可以直接识别互补输入的或运算：

```text
x | ~x -> 1
```

`opt` 不是一个单独算法，而是按有用顺序调用多个 `opt_*` pass 的包装器。与本例最相关的循环可以简化理解为：

```text
opt_expr
opt_merge

do
  opt_muxtree
  opt_reduce
  opt_merge
  opt_dff
  opt_clean
  opt_expr
while design changed
```

本例中的实际分工非常清楚：

1. `opt_expr` 把 `$or` 的结果改接到常量 `1'1`；
2. 原 `$or` cell 消失；
3. `$not` 的输出已经无人使用；
4. `opt_clean` 删除无用的 `$not` 和相关 wire；
5. `opt` 再跑一轮，直到设计不再变化。

如果沿主脚本执行到 `flatten` 后，暂时只运行 `opt_expr` 并立即写出 RTLIL，可以看到 `$or` cell 已经不在了，输出被直接接成常量，但暂时无用的 `$not` 还没清理：

```text
cell $not $flatten\even_part.$not$parity_cover.v:12$1
  connect \A \even_part.value [0]
  connect \Y \even_part.is_even
end

connect \always_one 1'1
```

完整 `opt` 结束后，`04_flattened_opt.il` 的功能部分已经只剩：

```text
module \parity_cover
  wire width 8 input 1 \value
  wire output 2 \always_one

  cell $scopeinfo \odd_part
    parameter \TYPE "module"
  end

  cell $scopeinfo \even_part
    parameter \TYPE "module"
  end

  connect \always_one 1'1
end
```

此时 `$not` 和 `$or` 都没有了。输入端口 `value` 仍存在，因为它属于顶层接口；只是任何输入位都不再影响输出。

这就是为什么源码规模不能直接代表综合后的面积。RTL 中写了多少层模块、多少个表达式，不等于芯片上一定留下多少逻辑。综合器关心的是最终可观察功能。

## `$scopeinfo` 为什么还被 `stat` 算成 cell

对本例中 `odd_part`、`even_part` 这类具有公开名称的实例，`flatten` 默认会创建 `$scopeinfo` cell，保存已经删除的实例和模块来源，例如：

```text
attribute \cell_src "parity_cover.v:22.18-25.6"
attribute \module_src "parity_cover.v:1.1-6.10"
attribute \module "odd_detector"
cell $scopeinfo \odd_part
  parameter \TYPE "module"
end
```

它记录的信息包括：

- 原实例来自源码的什么位置；
- 被内联的模块定义来自什么位置；
- 原模块和实例叫什么名字。

`$scopeinfo` 没有参与功能计算的输入输出端口，不代表门、触发器或工艺单元。它之所以留在 RTLIL 中，是为了让后续报告和调试仍能追溯展平前的层次。

如果从一开始就不需要这些信息，可以运行：

```text
flatten -noscopeinfo
```

如果希望先保留来源信息，等最终输出前再清理，可以使用：

```text
clean -purge
```

`clean` 与 `opt_clean` 等价，用于删除无用 cell 和 wire。这里的 `-purge` 有两个相关效果：

- 允许删除无用但具有公开名称的内部 net；
- 取消普通模式对 `$scopeinfo` cell 的专门保留。

因此，本例的清理日志会报告删除两个无用 cell 和四条无用 wire：两个 cell 正是 `odd_part`、`even_part` 对应的 `$scopeinfo`。

最终 `05_constant.il` 非常短：

```text
module \parity_cover
  wire width 8 input 1 \value
  wire output 2 \always_one
  connect \always_one 1'1
end
```

对应 Verilog 是：

```verilog
module parity_cover(value, always_one);
  input [7:0] value;
  wire [7:0] value;
  output always_one;
  wire always_one;
  assign always_one = 1'h1;
endmodule
```

统计结果是 **0 cells**。这不代表模块本身消失了：顶层 module、输入端口和输出端口仍然存在，只是实现这个输入输出关系不再需要任何逻辑单元。

如果后续映射到标准单元库，常量通常会通过 tie-high cell 或目标流程规定的常量网络实现；“Yosys 通用 RTLIL 中是 0 个逻辑 cell”不等于版图里一定完全没有与常量分发有关的物理资源。

## 用 SAT 分别验证优化前后的输出性质

结构变得过于简单时，最危险的反应有两个：

- 看到 cell 变少就无条件相信；
- 看到模块消失就认定工具出错。

更可靠的做法是检查完整输出性质，并明确它是在优化前还是优化后证明的。主脚本中的命令运行在最终常量网表上：

```text
sat -verify -prove always_one 1 -show value
```

含义分别是：

- `sat`：把当前选中模块转换成 SAT 问题；
- `-prove always_one 1`：尝试证明 `always_one` 对所有输入都等于 `1`；
- `-show value`：如果存在反例，显示对应的输入；
- `-verify`：证明失败时让 Yosys 返回错误，而不是只在日志里报告失败。

成功时会看到：

```text
Import proof-constraint: \always_one = 1'1
Final proof equation: \always_one = 1'1
SAT proof finished - no model found: SUCCESS!
```

“no model found”在这里不是求解器失败，而是找不到任何令证明条件为假的输入模型，也就是最终网表不存在反例。

其实不必等优化完成。只要先 `flatten`，默认二值 SAT 就已经能直接证明原 `$not + $or` 网络恒为 `1`：

```bash
yosys -Q -p 'read_verilog -sv parity_cover.v; hierarchy -top parity_cover; flatten; sat -prove always_one 1 -show value,always_one'
```

于是本例有两份独立证据：

- `flatten` 后、第二次 `opt` 前的 `$not + $or` 网络，对所有二值输入满足 `always_one = 1`；
- 第二次 `opt` 和 `clean -purge` 后的常量网络，也满足同一性质。

因为这是一个只有单个输出的组合设计，而且该性质完整规定了这个输出，所以前后分别证明它已经很有说明力。但必须知道：**后网表上的一条 `sat -prove` 本身不是通用的前后等价证明。** 对多输出或时序设计，应使用 `equiv_make`、`equiv_simple`、`equiv_induct`、`equiv_status -assert` 等等价检查流程直接比较两个设计。

结构优化和性质证明仍是两件不同的事：前者把实现改得更小，后者检查所声明的逻辑结论。

## 必须说清楚的 X/Z 与 undef 语义边界

上面的真值表只列出了 `0` 和 `1`。Verilog 仿真还有 `X` 和 `Z`。如果 `value[0]` 是 `X`：

```text
~X     = X
X | X  = X
```

所以四值 RTL 仿真中，原始代码的 `always_one` 会是 `X`，而不是 `1`。在展平但尚未执行第二次 `opt` 的设计上启用 Yosys 的 undef/X-bit 建模：

```bash
yosys -Q -p 'read_verilog -sv parity_cover.v; hierarchy -top parity_cover; flatten; sat -enable_undef -prove always_one 1 -show value,always_one'
```

求解器会找到反例：

```text
SAT proof finished - model found: FAIL!

\always_one = x
\value      = 0000000x
```

如果对已经化成 `always_one = 1'1` 的最终网表再加 `-enable_undef`，证明仍会成功。SAT 选项只能改变当前网络中的未知值建模，不能恢复已经被综合删除的 X 传播结构。

还要注意，`-enable_undef` 不是完整的 Verilog 四值仿真器。它建模 undef/X-bit，不会独立区分 `Z`，也不覆盖驱动强度和事件调度等四值仿真细节。本例的 X 反例与 RTL 的未知传播一致，但不能把这个选项泛化成完整四态仿真。

![二值布尔语义与 X/undef 建模的区别](/images/mermaid-svg/yosys-learning-03-flatten-cross-module-optimization/binary-vs-undef-semantics.svg)

这不意味着 `opt` 的常量化是错误的。可综合组合硬件最终工作在布尔 `0/1` 抽象上；仿真里的 `X` 主要表达“仿真器不知道当前值”，并不是芯片中的第三种稳定数字逻辑值。综合优化通常不承诺保留所有 RTL 的 `X` 传播现象。

这条边界非常重要：

- 不要把 `X` 当成正常运行时功能状态；
- 不要依赖 `X` 传播来实现芯片行为；
- 对“不允许未知”的要求，应使用复位、断言、lint 和形式约束表达；
- 亚稳态是模拟电气现象，也不能靠四值 `X` 精确描述。

因此，本文的“恒为 `1`”准确含义是：**对所有已定义的二值硬件输入，输出恒为 `1`。**

## keep_hierarchy：有意保留优化边界

并不是所有模块都应该展平。Yosys 的 `flatten` 会跳过带有 `keep_hierarchy` 属性的 cell，或者其实现 module 带有该属性的 cell。

只保护某个实例：

```verilog
(* keep_hierarchy *)
odd_detector odd_part(
    .value(value),
    .is_odd(odd_hit)
);
```

保护某类子模块的所有实例：

```verilog
(* keep_hierarchy *)
module odd_detector(
    input  [7:0] value,
    output       is_odd
);
    assign is_odd = value[0];
endmodule
```

需要注意，单纯把 `keep_hierarchy` 标在当前顶层 `parity_cover` module 上，并不会自动为它内部的每个子实例建立保护墙。要阻止 `odd_part` 或 `even_part` 被展开，应把属性放在对应实例或被例化的子模块定义上。

这个属性只保护当前被例化边界，不会递归冻结整个子树，也不会阻止 `opt` 优化被保护 module 的内部逻辑。加入保护后，本例的跨模块互补关系再次被边界隔开，普通的 `flatten; opt` 就不会得到完全相同的常量化结果。这正是属性存在的意义：主动用一部分全局优化空间换取稳定的设计边界。

## 什么时候 flatten 有利，什么时候应该克制

展平的直接收益是让后续 pass 看见更大的连续网络。常见机会包括：

- 常量跨 wrapper 和子模块端口传播；
- 跨模块删除无用逻辑；
- 识别原本分散在不同模块里的互补或重复表达式；
- 让后续技术映射或组合逻辑优化处理更大的逻辑锥；
- 消除仅用于组织源码、没有必要保留到网表的薄封装层。

代价同样真实：

- 网表名称更长，源码与网表对象更难一一对应；
- 波形、时序报告和形式反例的调试可读性下降；
- 层次化约束和依赖实例路径的脚本可能需要调整；
- 大型设计的内存占用和优化运行时间可能增加；
- IP、复用模块、DFT 分区或增量流程可能需要稳定边界；
- 某些团队依靠模块层次划分所有权和签核范围。

所以工程判断不是“展平越多越好”，而是：

> 哪些边界只是源码组织手段，哪些边界承担调试、约束、复用或流程职责？

对前者，展平通常能释放优化空间；对后者，选择性 flatten 或 `keep_hierarchy` 往往更合适。

## opt 很强，但它不是无所不知的全局证明器

这个案例会给人一种印象：只要跑 `opt`，任意复杂逻辑都能被发现并压缩到最小。这个结论过头了。

Yosys 对 `opt` 的定位是“一系列简单优化和清理”。它非常擅长：

- 常量折叠；
- 简单布尔恒等式；
- 合并相同 cell；
- 清理 MUX 树中的死分支；
- 删除无用 wire 和 cell；
- 提取触发器使能、复位和部分同步控制；
- 反复运行上述过程直到达到稳定状态。

但它不会自动把任意大型布尔网络做成全局最小实现。更复杂的逻辑还可能依赖：

- `techmap` 或 `simplemap` 把高层 cell 降到更基础的门网络；
- `abc` 对组合逻辑做分解、重写和目标库映射；
- 目标 Liberty 库中的实际 cell 集合与面积、延迟信息；
- 时序约束和面积目标；
- 设计是否已经展平，以及后续 pass 的选择范围。

本例之所以变化如此彻底，是因为 `x | ~x = 1` 是一个非常直接的局部恒等式。一旦 `flatten` 让两个操作数进入同一 module，`opt_expr` 就能轻易识别。

## 当设计突然变成常量时，应该检查什么

看到综合结果大幅缩小时，可以按下面顺序排查：

1. **顶层是否选对。** 检查 `hierarchy -check -top ...`，选错 top 会让真正设计变成不可达模块。
2. **输出是否可观察。** 没有连接到顶层输出、存储单元或有副作用对象的逻辑，本来就可以被删除。
3. **输入是否被绑成常量。** 参数、generate 条件、端口 tie-off 都可能令整个逻辑锥静态化。
4. **是否存在简单恒等式。** 例如 `x & 0`、`x | 1`、`x ^ x`、`x | ~x`。
5. **优化发生在哪一个 pass。** 在关键命令前后分别执行 `write_rtlil` 和 `stat`，不要只比较源码与最终网表。
6. **消失的是功能 cell 还是层次定义。** `Deleting now unused module` 可能只是内联后的正常清理。
7. **证明功能，而不是只数 cell。** 用 `sat` 检查组合性质，用等价检查验证较大的前后设计。
8. **确认 X/未定义值假设。** 默认二值证明与 `-enable_undef` 的结论可能不同。
9. **检查层次保护意图。** 需要稳定边界时，确认 `keep_hierarchy` 放在正确的子模块或实例上。

这个检查顺序能区分三种完全不同的情况：正确的逻辑优化、错误的综合入口，以及 RTL 本身没有把预期功能连接到可观察输出。

## 小结

这个设计最终确实会被优化到只剩：

```verilog
assign always_one = 1'b1;
```

但整个过程不能简单概括为“`flatten` 把两个模块优化掉了”。更准确的分解是：

```text
hierarchy
  建立并保留模块实例关系

第一次 opt
  清理各 module 内部，但看不到跨边界的互补关系

flatten
  用子模块实现替换实例，使 x、~x 和 OR 出现在同一 module

第二次 opt
  opt_expr 将 x | ~x 化成 1，opt_clean 删除失去用途的 $not

clean -purge
  删除无功能作用的 $scopeinfo 和无用公开内部线网

sat
  证明对所有二值输入，always_one 恒为 1
```

最值得保留的认识不是“`opt` 很神奇”，而是：**优化能力取决于工具当前能看见的结构。模块边界决定视野，`flatten` 扩大视野，`opt` 在这个视野里重写电路。**
