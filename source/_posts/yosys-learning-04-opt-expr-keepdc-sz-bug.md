---
title: "Yosys 学习笔记（四）：一个 Sz 漏检如何让 opt_expr 把 xxxx 优化成 11x1"
date: 2026-07-20 16:03:54
tags:
  - "Yosys"
  - "C++"
  - "RTLIL"
  - "logic synthesis"
categories:
  - "Computer Architecture"
---

## 前言

如果要给一个大型开源项目提交第一个 PR，最合适的目标通常不是增加一个新 pass，而是找到一条已经存在的优化规则，构造一个很小的边界条件，再证明它在这个边界上改变了电路语义。

这次研究的是 Yosys `opt_expr -fine` 中的一条减法优化：

```text
(2^k - 1) - B  ->  ~B
```

在普通二值逻辑中，这是一条漂亮而且正确的等价变换。但原实现只拦截了输入中的 `x`，漏掉了 `z`。结果是：一个原本应当得到 `xxxx` 的四位减法，在 `-keepdc` 模式下竟然被改写成 `$not`，输出变成了 `11x1` 或 `11x0`。

最终修复只有两个条件判断：让 `Sz` 和 `Sx` 走相同的不确定值传播路径。代码很小，但理解它需要同时弄清四件事：

1. 为什么全 1 减法能够变成按位取反；
2. 为什么算术运算与按位运算对 `x/z` 的传播不同；
3. `-keepdc` 到底承诺了什么；
4. 怎样用结构断言和 SAT 证明把这个 bug 锁进回归测试。

本文从最小例子开始，完整复盘这个 bug 的来源、错误改写、修复方式和验证过程。

## 先理解正确的优化：为什么 15 - B 等于 ~B

对一个宽度为 `k` 的无符号数 `B`，它的 `k` 位按位取反可以写成：

```text
~B = (2^k - 1) - B
```

原因是 `2^k - 1` 的低 `k` 位全部为 `1`。例如 `k = 4`：

```text
2^4 - 1 = 15 = 4'b1111
```

令 `B = 4'b0010`，两种计算得到相同结果：

```text
  1111
- 0010
------
  1101

~0010 = 1101
```

因此下面的 `$sub`：

```text
Y = 4'b1111 - B
```

可以改写成：

```text
Y = ~B
```

减法器通常比一组反相器昂贵，所以这条规则有真实的综合价值。Yosys 的实现还支持更一般的掩码形式：只要 `A` 是低 `k` 位为 `1`、更高位为 `0` 的 `(2^k - 1)`，并且能够证明 `B` 不会超出这 `k` 位，就可以只对 `B` 的低 `k` 位取反。

核心逻辑可以简化成下面这样：

```cpp
// A 是否形如 000...00111...111，也就是 2^k - 1？
int k = 0;
while (k < y_width && a_val[k] == State::S1)
    k++;

bool a_is_mask = k > 0;
for (int i = k; a_is_mask && i < y_width; i++)
    if (a_val[i] != State::S0)
        a_is_mask = false;

// B 的高位是否全为 0，即 B < 2^k？
bool b_fits = true;
for (int i = k; b_fits && i < y_width; i++)
    if (sig_b[i] != State::S0)
        b_fits = false;

if (a_is_mask && b_fits) {
    RTLIL::SigSpec sig_y = module->Not(NEW_ID, sig_b.extract(0, k));
    module->connect(cell->getPort(ID::Y), sig_y);
    module->remove(cell);
}
```

到这里为止，这个模式没有问题。问题出在它前面的不确定值检查漏掉了一种状态。

## `Sx`、`Sz` 和 `-keepdc` 分别是什么

Verilog 不只有 `0` 和 `1`，还有两种常见的四态值：

| Verilog 状态 | Yosys 内部状态 | 含义 |
|---|---|---|
| `0` | `State::S0` | 确定的逻辑 0 |
| `1` | `State::S1` | 确定的逻辑 1 |
| `x` | `State::Sx` | 未知或不确定 |
| `z` | `State::Sz` | 高阻态 |

`Sx`、`Sz` 只是 Yosys C++ 代码中表示 `x`、`z` 的枚举值。对于连线和三态连接，`z` 有自己的意义；但当 `z` 进入加减乘除这类算术表达式时，它不能被当成一个确定的二进制数字，因而会按不确定值参与运算。

这里必须限定讨论范围：**不是任何地方出现 `z`，整个结果都必须变成 `x`。** 本文的结论针对 `$sub` 等相关算术 cell。三态总线、连接和逐位逻辑各有自己的四态语义，不能套用同一条规则。

再看 `-keepdc`。其中 `dc` 是 don't care。Yosys 默认允许某些优化利用不确定位，因为最终制造出来的数字电路只有 0 和 1，`x` 往往也被用于表达“不关心”。但有些优化会改变四态仿真或形式验证中 `x` 的传播方式。

一个典型例子是：

```text
A + 0  ->  A
```

二值逻辑中它当然成立。可是如果 `A` 某一位为 `x`，算术加法可能把整个结果变成不确定值；直接连接 `A` 却只在原位置保留 `x`。`-keepdc` 的作用就是禁用这种会改变 don't-care 行为的优化。

因此，`-keepdc` 不是“把结果强制设成 `xxxx`”的开关。它表达的约束是：**优化前后必须保留相应的不确定值传播语义。**

## 最小 bug：原本的 xxxx 被优化成 11x1

构造下面的四位减法：

```text
A = 4'b1111
B = {3'b00z, b}
Y = A - B
```

拼接 `{3'b00z, b}` 表示：

```text
b = 0 时，B = 4'b00z0
b = 1 时，B = 4'b00z1
```

### 优化前：算术减法传播到整个结果

`B` 不是一个确定的二进制整数，因为其中含有 `z`。对 `$sub` 而言，Yosys 应把结果视为全宽不确定：

| `b` | `B` | `1111 - B` |
|---:|---:|---:|
| `0` | `00z0` | `xxxx` |
| `1` | `00z1` | `xxxx` |

注意这里不是在试图猜测 `z` 是 `0` 还是 `1`。算术表达式只知道操作数不确定，所以四个结果位都不再可靠。

### 错误优化后：按位取反只污染一个位置

旧实现没有拦住 `z`，于是继续匹配：

```text
1111 - B  ->  ~B
```

按位取反的四态规则与算术减法不同：

```text
~0 = 1
~1 = 0
~x = x
~z = x
```

因此：

```text
~00z0 = 11x1
~00z1 = 11x0
```

最终出现了清楚的前后差异：

| `b` | 优化前的 `$sub` | 错误的 `$not` | 是否等价 |
|---:|---:|---:|---|
| `0` | `xxxx` | `11x1` | 否 |
| `1` | `xxxx` | `11x0` | 否 |

这个 bug 的关键不是“`z` 经过 `$not` 变成了 `x`”。`~z = x` 本身是正确的。真正的问题是：**算术减法的不确定性应传播到全部结果位，而逐位取反只把对应的那一位变成 `x`。** 两个表达式在二值域中等价，在四态域中却不等价。

## bug 的来源：入口只检查了 Sx

在执行后续算术重写之前，`opt_expr` 已经有一段提前传播不确定值的保护逻辑。旧代码的核心如下：

```cpp
for (auto &bit : sig_a.to_sigbit_vector())
    if (bit == RTLIL::State::Sx)
        goto found_the_x_bit;

for (auto &bit : sig_b.to_sigbit_vector())
    if (bit == RTLIL::State::Sx)
        goto found_the_x_bit;

if (0) {
found_the_x_bit:
    replace_cell(
        assign_map, module, cell,
        "x-bit in input", ID::Y,
        RTLIL::SigSpec(RTLIL::State::Sx,
                       GetSize(cell->getPort(ID::Y)))
    );
    goto next_cell;
}
```

这段控制流做了三件事：

1. 扫描经过信号映射后的 `A`、`B` 端口；
2. 一旦发现常量 `Sx`，跳到 `found_the_x_bit`；
3. 用一个与 `Y` 等宽的全 `Sx` 常量替换 cell，然后跳过当前 cell 的其余优化。

`goto` 在这里承担的是局部的“两个循环共同提前退出”功能。真正的错误不在控制流形式，而在判断条件：它只识别 `Sx`，没有识别同样会使算术操作数不确定的 `Sz`。

于是输入含 `x` 时的路径是：

```text
发现 Sx
  -> Y 直接替换为 xxxx
  -> 删除 $sub
  -> 跳过后续所有重写
```

输入含 `z` 时却变成：

```text
没有发现 Sx
  -> Sz 穿过保护逻辑
  -> 继续检查细粒度优化模式
  -> 命中 (2^k - 1) - B
  -> 错误生成 $not
```

为什么 `B = 00zb` 仍然会通过 `b_fits`？在这个四位例子中，`A = 1111`，所以 `k = 4`。检查 `B` 高位的循环从索引 `4` 开始，而输出宽度也是 `4`，循环根本不会执行。它只能证明“没有第 4 位以上的数值”，并不检查低四位是否含 `x/z`。这个职责本来就属于前面的不确定值保护，但那里恰好漏掉了 `Sz`。

这也解释了为什么问题不是出在数学恒等式或 `b_fits` 的位宽判断中，而是出在进入模式匹配之前的状态分类中。

## 修复：让 Sz 与 Sx 走同一条提前退出路径

最小修复是在 `A`、`B` 两个循环中同时识别 `Sz`：

```cpp
for (auto &bit : sig_a.to_sigbit_vector())
    if (bit == RTLIL::State::Sx || bit == RTLIL::State::Sz)
        goto found_the_x_bit;

for (auto &bit : sig_b.to_sigbit_vector())
    if (bit == RTLIL::State::Sx || bit == RTLIL::State::Sz)
        goto found_the_x_bit;
```

修复后的路径变为：

```text
发现 Sx 或 Sz
  -> Y 直接替换为等宽的 Sx 向量
  -> 删除原算术 cell
  -> 跳过 (2^k - 1) - B -> ~B 等后续重写
```

对本文的例子，最终网表不会保留 `$sub`，也不会生成 `$not`，而是直接建立常量连接：

```text
connect \y 4'xxxx
```

这就是“拦截”的准确含义：不是简单保留原 `$sub` 不动，而是在已经确定结果全为不确定值时，立即完成正确的常量传播，并阻止后续二值优化模式误用。

这里没有把条件写成“不是 `S0` 且不是 `S1`”，因为 `SigBit` 还可能表示一根普通信号线。普通动态输入当然不能被当成 undef。明确匹配 `Sx || Sz`，范围最小，也最符合原代码的意图。

## 一次修复解决了两个不同表现

回归测试同时把 `z` 放到 `$sub` 的 `A` 端和 `B` 端，但两者在旧实现中的表现并不完全相同。

### A 端含 z：漏掉可以完成的 undef 常量传播

```text
A = {3'b00z, b}
B = 4'b0101
```

这里 `A` 既不是确定常量，也不是 `(2^k - 1)` 掩码，所以不会命中 `$sub -> $not`。旧实现通常会留下 `$sub`。它的输出语义仍然是 `xxxx`，所以主要问题是错过了一个本来可以完成的常量传播和 cell 删除。

### B 端含 z：触发错误重写，改变输出语义

```text
A = 4'b1111
B = {3'b00z, b}
```

这里 `A` 正好是 `(2^4 - 1)`，旧实现会错误地产生 `$not`，把 `xxxx` 改成 `11x1` 或 `11x0`。这是一个真实的正确性 bug，而不只是漏优化。

因此，这个小补丁同时带来两个结果：

| 场景 | 修复前 | 修复后 |
|---|---|---|
| `A` 含 `z` | 可能残留 `$sub` | 直接传播全 `x`，删除 `$sub` |
| `B` 含 `z` 且 `A=2^k-1` | 错误生成 `$not` | 直接传播全 `x`，不生成 `$not` |

称它“一箭双雕”没有问题，但写 PR 时应把主次说清楚：**核心是修复错误的四态语义，顺带补全 A 端的 undef 常量传播。**

## 用一个脚本直接观察修复前的错误

下面的 Yosys 脚本包含两个设计。第一个证明正常二值输入确实应该从 `$sub` 变成 `$not`；第二个复现含 `z` 时的错误行为。

```text
# Case 1: 正常二值优化，4'b1111 - B -> ~B
read_rtlil <<EOF
module \normal
  wire width 4 input 1 \b
  wire width 4 output 2 \y

  cell $sub \sub
    parameter \A_WIDTH 4
    parameter \B_WIDTH 4
    parameter \Y_WIDTH 4
    parameter \A_SIGNED 0
    parameter \B_SIGNED 0
    connect \A 4'1111
    connect \B \b
    connect \Y \y
  end
end
EOF

check
eval -set b 2 -show y
opt_expr -fine -keepdc
select -assert-none t:$sub
select -assert-count 1 t:$not
stat
dump
eval -set b 2 -show y

design -reset

# Case 2: 含 z 的算术输入，复现旧实现错误生成 $not
read_rtlil <<EOF
module \bug
  wire input 1 \b
  wire width 4 output 2 \y

  cell $sub \sub
    parameter \A_WIDTH 4
    parameter \B_WIDTH 4
    parameter \Y_WIDTH 4
    parameter \A_SIGNED 0
    parameter \B_SIGNED 0
    connect \A 4'1111
    connect \B { 3'00z \b }
    connect \Y \y
  end
end
EOF

check
eval -set b 0 -show y
eval -set b 1 -show y
opt_expr -fine -keepdc
select -assert-none t:$sub
select -assert-count 1 t:$not
stat
dump
eval -set b 0 -show y
eval -set b 1 -show y
```

在未修复版本上，第一组输出保持等价：

```text
优化前：Y = 1101
优化后：Y = 1101
结构变化：$sub -> $not
```

第二组则暴露 bug：

```text
优化前：b=0，Y=xxxx
优化前：b=1，Y=xxxx

优化后：b=0，Y=11x1
优化后：b=1，Y=11x0
结构变化：$sub -> $not
```

修复后，第二组脚本中“必须存在一个 `$not`”的旧行为断言会失败，这是预期的，因为正确结构已经变成零个 cell 和一个 `xxxx` 常量连接。演示 bug 时应观察旧版本；验证修复时则应使用下面的回归测试。

## 正式回归测试：同时检查结构和语义

一个好的综合器回归测试不能只看日志中的某一句话。日志可能调整，cell 名字也可能变化。更可靠的测试应该回答两个问题：

1. 优化后的结构是否还残留算术 cell，或者错误生成了其他 cell？
2. 输出在 undef 语义下是否真的是不确定值？

完整测试如下：

```text
# Arithmetic cells must treat z inputs as undef before applying further rewrites.
read_rtlil <<EOF
module \test
  wire input 1 \b
  wire width 4 output 2 \y_a
  wire width 4 output 3 \y_b

  cell $sub \sub_z_a
    parameter \A_WIDTH 4
    parameter \B_WIDTH 4
    parameter \Y_WIDTH 4
    parameter \A_SIGNED 0
    parameter \B_SIGNED 0
    connect \A { 3'00z \b }
    connect \B 4'0101
    connect \Y \y_a
  end

  cell $sub \sub_z_b
    parameter \A_WIDTH 4
    parameter \B_WIDTH 4
    parameter \Y_WIDTH 4
    parameter \A_SIGNED 0
    parameter \B_SIGNED 0
    connect \A 4'1111
    connect \B { 3'00z \b }
    connect \Y \y_b
  end
end
EOF

check
opt_expr -fine -keepdc
select -assert-none t:*
sat -verify -enable_undef -set b 0 -prove y_a[0] 1'x
sat -verify -enable_undef -set b 0 -prove y_b[0] 1'x

design -reset
```

逐行看最后几条命令：

```text
check
```

先检查设计结构是否合法，避免端口宽度或连接错误让测试失去意义。

```text
opt_expr -fine -keepdc
```

`-fine` 启用包括减法转取反在内的细粒度重写；`-keepdc` 要求保留不确定值行为。两者组合正好覆盖出错路径。

```text
select -assert-none t:*
```

`t:*` 选择所有 cell 类型。这条断言要求优化后一个 cell 都不能剩下：A 端的 `$sub` 应被全 `x` 常量连接替代，B 端也不能再出现错误的 `$not`。

```text
sat -verify -enable_undef -set b 0 -prove y_a[0] 1'x
sat -verify -enable_undef -set b 0 -prove y_b[0] 1'x
```

`-enable_undef` 让 SAT 流程显式追踪 undef；`-set b 0` 固定最小反例所需的输入；`-prove ... 1'x` 证明两个结果的最低位都是不确定值。

为什么只证明最低位就能抓到 B 端 bug？因为旧的错误结果在 `b=0` 时是：

```text
11x1
   ^
   y_b[0] = 1
```

正确结果要求 `y_b[0] = x`，所以这一位已经足以区分正确与错误实现。测试不必为证明 bug 而堆满重复断言。`b=1` 同样能构成反例，此时错误的最低位是 `0`。

结构断言和 SAT 证明各自承担不同职责：

| 检查 | 能发现什么 |
|---|---|
| `select -assert-none t:*` | 残留 `$sub`、错误生成 `$not` 或其他 cell |
| `sat -enable_undef` | 即使结构改变，输出的 undef 语义仍必须正确 |

两种检查组合起来，既锁住了这次修复期望的常量传播结果，也锁住了真正重要的功能语义。

## 修复前后的完整对比

把关键结果集中起来看：

| 输入场景 | 优化前 | 未修复的 `opt_expr` | 修复后的 `opt_expr` |
|---|---|---|---|
| `A=1111, B=0010` | `Y=1101`，一个 `$sub` | `Y=1101`，一个 `$not` | 相同，继续正确优化为 `$not` |
| `A=1111, B=00z0` | `Y=xxxx`，一个 `$sub` | `Y=11x1`，错误的 `$not` | `Y=xxxx`，直接常量连接 |
| `A=1111, B=00z1` | `Y=xxxx`，一个 `$sub` | `Y=11x0`，错误的 `$not` | `Y=xxxx`，直接常量连接 |
| `A=00zb, B=0101` | `Y=xxxx`，一个 `$sub` | 语义仍为 undef，但可能残留 `$sub` | `Y=xxxx`，直接常量连接 |

这个对比也说明修复没有粗暴禁用原优化。普通二值输入仍然能够享受 `$sub -> $not`；只有当相关算术操作数中出现显式 `Sx/Sz` 时，才提前传播 undef 并跳过后续模式。

## 为什么修复不应只加在 `$sub -> $not` 规则里

最直接的补丁似乎是：在 `(2^k - 1)-B` 模式内部额外检查一次 `B` 是否含 `z`。但这会把同一种四态语义分散到每一条具体重写规则中，而且无法解决 A 端含 `z` 时的漏传播。

把修复放在统一的早期 undef 检查中更合理，原因有三点：

1. `z` 对相关算术 cell 的影响不是 `$sub -> $not` 独有；
2. 在模式匹配前结束处理，可以保护后续所有重写；
3. 原代码已经为 `Sx` 建立了正确路径，只需补全遗漏的 `Sz`，改动范围最小。

这类补丁很适合作为开源贡献：不是重新设计优化框架，而是找到已有不变量“算术输入含显式 undef 时，先传播 undef”，证明实现漏了一种内部状态，然后用最小修改恢复不变量。

## `-keepdc` 之外还要注意什么

这次回归明确使用 `-keepdc`，因为旧实现即使在用户要求保留 don't-care 行为时仍改变了输出，问题最清楚。但不要据此推导出“不加 `-keepdc` 就一定输出 `11x1`”。

修复后的 `Sx/Sz` 检查位于共享的前置路径，并不受 `keepdc` 条件控制。因此对本文涉及的算术 cell，只要映射后的相关操作数含显式 `Sx` 或 `Sz`，都会先走全 `x` 传播，不会到达 `$sub -> $not`。

更准确的说法是：

```text
旧实现：-keepdc 没能阻止 Sz 穿过前置检查，这是 bug。
新实现：Sx 和 Sz 都在后续重写前被处理，-keepdc 合约得到恢复。
```

## 从这个小 bug 学到的代码阅读方法

寻找 `opt_expr` 的入门贡献点时，不必一开始就理解整个文件。围绕一条模式建立一条窄而完整的证据链更有效：

```text
找到一条重写规则
  -> 写出它成立的数学条件
  -> 检查 signed、位宽、零宽和 x/z 边界
  -> 构造优化前后的最小结构
  -> 用 eval 或 sat 找到可观察差异
  -> 回到模式之前追踪保护条件
  -> 做最小修复并添加结构 + 语义回归
```

这次 bug 的证据链就是：

```text
(2^k - 1) - B -> ~B 在二值域成立
  -> `$sub` 与 `$not` 的四态传播不同
  -> Sx 会提前退出，Sz 不会
  -> Sz 能到达 sub-to-not 模式
  -> xxxx 被改成 11x1/11x0
  -> 两处加入 Sz 判断
  -> 正常二值优化保留，异常四态行为修复
```

对第一次 Yosys PR 来说，这已经是一个完整的工程闭环：问题真实、复现最小、根因明确、补丁克制、回归测试能在旧版本失败并在新版本通过。

## 总结

这次问题表面上只是一个枚举值漏检：

```cpp
State::Sx
```

变成：

```cpp
State::Sx || State::Sz
```

但它背后体现了综合优化中非常重要的一条原则：**等价变换必须说明自己在哪个值域中等价。**

`(2^k - 1)-B` 与 `~B` 在固定宽度二值域中等价；一旦操作数含 `x/z`，算术传播和逐位传播不再相同。`-keepdc` 要求优化器尊重这个差异，不能把更多确定的 `0/1` 凭空带入结果。

修复后的行为可以浓缩成一句话：

```text
相关算术输入中出现显式 x 或 z 时，先传播全宽 undef，
不要让它进入只在二值条件下成立的后续重写。
```

一个两行补丁能够成为有价值的 PR，靠的不是代码量，而是它是否恢复了清晰的语义不变量，并留下一个以后不会再次退化的测试。
