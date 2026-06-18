---
title: "LA 挑战赛记录（10）：在 FPGA 上启动 Linux Kernel"
date: 2025-07-24 13:03:13
tags:
  - "LoongArch"
  - "FPGA"
  - "Linux"
categories:
  - "Computer Architecture"
---

## 前言
从 7月1号 到现在在 FPGA 启动 linux_kernel 已经过了 23 天了，虽然中间考试和休息占了 5 天左右，这也说明在 chiplab 中启动 linux_kernel 和在 FPGA 上启动 linux_kernel 有很大的跨度以及很多的工作量。事实上，工作量主要集中在 debug 上，下面细说一下常见的 bug 以及 cacop 的实现细节。

## cacop
没错，想要使用双 cache 来启动 linux_kernel，cacop 指令绝对绕不过去，这也是 Loongarch32r 中最复杂的指令，它真的很复杂，我光实现这个指令加上构思，花了整整两天（20h工作量）。

因为我刚开始做的是 4-way dcache\icache，但是我测试《CPU 设计实战》给的 EXP=23 的时候，发现过不去，经过排查发现它测试的对象是 2-way cache，所以我把 dcache 改成 2—way 的，icache 不影响，因为它没有 write_back 的逻辑，我只需要极端一点将 index 指定的 cache_line 全部无效就行了。也是顺利通过了 EXP=23。

但是这里我写了一个很严重的 bug，后面排查的时候才发现：dcache 在 replace_v 有效且 mode2 是回写（之前没有判断 replace_v）。这个 bug 很难排查，花了很多时间和精力，因此写的每一行处理逻辑，都要思考全面，不然后面 debug 的时候将会花费十倍百倍的精力和时间。

## 其它 bug

就要就是各种细节了，比如忘了清空异常信号的缓存，br_taken 之后上一个异常信号还在，这里不细说了，主要说一下排除 bug 的方法。

因为我在 chiplab 中能启动 linux_kernel 成功，但是无法在 fpga 上成功启动，这种就很难排除，因为在 fpga 上无法 debug。所以我把注意力放在了 chiplab 提供的 run_random 上，也就是进行随机测试。

随机测试真的是一个很好的东西，我认为就算你能启动世界上所有的操作系统，能通过所有的功能测试，都无法证明你的 CPU 设计是正确的。事实上，CPU 功能验证是一个很复杂的问题，不是说跑一下 linux、功能测试就能证明这个核没有问题。

像 Intel Atom C2000 系列（如 C2750）中的一个严重 bug：某个极少数但可预测的指令序列（如某种条件跳转+loop循环+call/ret）会触发前端跳转预测逻辑中的一个 定时器（clock gating / enable）异常路径。导致该部分逻辑无法重新上电或无法恢复工作，某些内部 oscillator/clock gating 电路完全关闭。原因就是 RTL 设计中存在“前端状态控制逻辑 + clock enable信号”的组合路径，但未覆盖极端指令组合下的状态转换。

Intel 在流片测试前肯定跑了成千上万亿条随机指令测试，但是这都没有覆盖到这种情况，更别说只在 chiplab 中仅仅跑了几个功能测试。这给我们的启示就是，“即使万亿次验证通过，也不能代表系统没有潜藏死锁和状态机陷阱。”换句话说，尽管跑随机指令测试并不能组合出所有情况，但这也是我们保证 CPU “实现正确”的最接近的手段了。因此随机测试一定要做，根据我的经验，只要过跑一个随机测试案例，基本上就能在 FPGA 中启动 linux_kernel 了。

如果在跑随机测试的时候遇到错误而暂停，实在找不到 bug 就可以利用 OpenLA500 来进行指纹级别的肉眼比对了。具体的做法就是重新复制一个 chiplab，换上 OpenLA500，让它也跑随机测试，并设置 time_limit 让它在你的 CPU 出错的地方附近暂停。然后就可以仔细对比波形图了。这个过程很花费时间和精力，但是很有效。我就是这样找出了 3 个 bug 并解决后，成功在 fpga 上启动了 linux_kernel。

