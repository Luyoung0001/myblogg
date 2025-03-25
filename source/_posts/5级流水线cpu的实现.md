---
title: 5 级流水线 CPU 的实现
date: 2024-10-10 20:20:40
tags:
    - riscv32
    - Chisel
categories:
    - PA2
    - CPU
---

<!--more-->

## 前言

在 PA2 中，要求实现一个单周期的 CPU，也就是 NPC。这个工作做完了之后，我发现了一本好书，名字叫做《CPU 制作入门：基于 RISC-V 和 Chisel》，是一个日本作者写的。内容非常适合入门，我花了一天时间就看到了单周期，被作者的思路完全影响了，第二天就立即开始重构 NPC，重构好了之后，整个项目看起来很清晰，把能在 ID 阶段做的事情都做了，一共分了 5 个阶段：IF、ID、EXE、MEM、WB。

之后作者开始做流水线，解决了指令冒险、数据冒险之后，5 级流水就做的差不多了。我按照作者的思路，也为我的 NPC 扩展了 5 级流水，可以通过所有测试案例:

```txt
[   bubble-sort] PASS
[           fib] PASS
[   select-sort] PASS
[         dummy] PASS
[   shuixianhua] PASS
[        pascal] PASS
[        string] PASS
[      mersenne] PASS
[           div] PASS
[           sum] PASS
[       if-else] PASS
[           add] PASS
[     leap-year] PASS
[    quick-sort] PASS
[        switch] PASS
[        wanshu] PASS
[  sub-longlong] PASS
[           max] PASS
[  add-longlong] PASS
[  mul-longlong] PASS
[         shift] PASS
[    matrix-mul] PASS
[         movsx] PASS
[           bit] PASS
[         mov-c] PASS
[     hello-str] PASS
[     recursion] PASS
[    load-store] PASS
[         crc32] PASS
[ to-lower-case] PASS
[      goldbach] PASS
[       unalign] PASS
[          fact] PASS
[          min3] PASS
[         prime] PASS
```

这篇文章记录一下我的思考。

## 架构

就像前面说的，一共五个阶段：IF、ID、EXE、MEM、WB。本文用这 5 个阶段阐述。

## IF
取指，就是往内存模块输送正确的指令地址，期望接收到指令。由于 PC 会受到反馈信号影响，因此这里必须提供一个寄存器：
```Scala
   val if_pc_reg = RegInit(START_ADDR)
    val if_pc_plus4 = if_pc_reg + 4.U(WORD_LEN.W)

    val exe_br_flg   = Wire(Bool())
    val exe_jmp_flg =  Wire(Bool())
    val exe_br_target = Wire(UInt(WORD_LEN.W))

    val exe_alu_out = Wire(UInt(WORD_LEN.W))

    val stall_flg = Wire(Bool())


    // 连接 PC
    // 这里使用组合电路
    io.imem.addr := if_pc_reg
    val if_inst = io.imem.inst

    val if_pc_next = MuxCase(if_pc_plus4, Seq(
        exe_br_flg -> exe_br_target,
        exe_jmp_flg -> exe_alu_out,
        (if_inst === ECALL) -> csr_regfile(0x305), // ecall 跳转到这里进行异常处理
        stall_flg -> if_pc_reg // 数据冒险: 保持pc不变
    ))

    if_pc_reg := if_pc_next // 下一个周期更新

```

同时还得给出 PC 的来源。PC 不是随便给的，这很好理解。程序如果不跳转，那么直接运行就好了，每次+4就行。但是程序的执行过程中，跳转、返回等等是必不可少的。因此 PC 有以下几个来源：
- if_pc_reg + 4 // 顺序执行
- exe_br_target // 分支
- exe_alu_out // 跳转
- if_pc_reg // 停顿

分支是指遇到了 B 指令，跳转则是 jal/jalr 引起的，至于停顿，这里牵扯到了数据冒险，后面会提到。

总结一下就是，IF 阶段就负责干一件事：指出下一条指令的地址。至于拿到指令，这是内存自动给出的。

## ID

指令解码的含义就是，拿到一条指令，对指令进行解码，分析它的位模式。同时求出操作数、寄存器地址。当然，这些信息来源于一条指令的位模式。

怎么拿到指令？

如果直接从 IF 阶段往下传，那么下一个周期就会拿到：

```Scala
id_reg_pc := Mux(stall_flg, id_reg_pc, if_pc_reg) // 如果数据冒险：保持

    // 指令冒险：if_inst 置为 BUBBLE
    id_reg_inst := MuxCase(if_inst, Seq(
        (exe_br_flg || exe_jmp_flg) -> BUBBLE,
        stall_flg -> id_reg_inst // 如果数据冒险：保持
    ))


    // 重置指令：如果是冒险指令、冒险数据，那么在当前周期重置为 BUBBLE
    val id_inst = Mux((exe_br_flg || exe_jmp_flg || stall_flg), BUBBLE, id_reg_inst)

    // 解析寄存器地址
    val id_rs1_addr = id_inst(19,15)
    val id_rs2_addr = id_inst(24,20)
    val id_wb_addr  = id_inst(11,7)
    val mem_wb_data = Wire(UInt(WORD_LEN.W))
```

从多路复用其可以看到，如果不遇到意外，比如没有跳转、数据冒险，就能直接接过接力棒。但是如果遇到了意外，比如分支、数据冒险，那么就不能简单的直接从 IF 拿指令了。

如果遇到跳转，等等，谁在跳转？

是 EXE，如果检测到 EXE 在执行一条跳转指令，那么前面处理的 IF、ID不都是白费力气了？

因此，一旦检测到 EXE 在执行分支、跳转，那么在这个周期，就可以截断 IF 的指令传送，直接让它为 BUBBLE 指令，也就是空指令。下一个周期，这个空指令就会传送到 EXE，这不会改变状态机的任何有效状态。

如果遇到数据冒险呢？这件事情很常见，也很广泛。以下是一个简单的例子：

```asm
80000000:	00000413          	li	s0,0
80000004:	00009117          	auipc	sp,0x9
80000008:	ffc10113          	addi	sp,sp,-4 # 80009000 <_end>
```

当 EXE 开始执行 auipc 的时候，ID 刚好正在处理 addi。此时，ID 要将操作数 sp 准备好，但是 auipc 执行时，并没有将数据放到寄存器中供给 ID 读取。因为 EX 执行后，数据甚至还没到 MEM，后面还有 WB 阶段。

这时候，可能会想到，等一个周期，等到 MEM 阶段，将数据直接传送到 ID，而 ID 阶段重新执行一下 addi（之前执行的其实是 BUBBLE）。就好了，这个方法就是 直通 + 停顿。

## 等一个周期

一个状态机，怎么实现等待？

这是一个好问题，答案是保持输入不变。

ID 阶段需要向 EXE 提供的底层输入有哪些？

- id_reg_pc
- id_reg_inst

没错，就这两个：

```Scala
id_reg_pc := Mux(stall_flg, id_reg_pc, if_pc_reg)

id_reg_inst := MuxCase(if_inst, Seq(
    (exe_br_flg || exe_jmp_flg) -> BUBBLE,
    stall_flg -> id_reg_inst // 如果数据冒险：保持
))
```

当数据冒险的时候，就保持 id_reg_pc、id_reg_inst。

然后呢？ID 阶段还有一些操作，这些操作是由组合电路自动完成的，不能乱操作，不能一直经过 EXE、MEM、WB 后影响整个状态机的状态。因此还必须让它空转：

```Scala
// 重置指令：如果是冒险指令、冒险数据，那么在当前周期重置为 BUBBLE
val id_inst = Mux((exe_br_flg || exe_jmp_flg || stall_flg), BUBBLE, id_reg_inst)

```

这样，如果是冒险指令、冒险数据，让它decode BUBBLE，当前的 ID、后面的 EX、MEM、WB 自然就不会影响有效状态。


## 直通

直通就是当 MEM、WB 阶段的时候，ID 要访问寄存器，可以不必等到写好之后再访问，可以直接访问：

```Scala
val id_rs1_data = MuxCase(regfile(id_rs1_addr), Seq(
        (id_rs1_addr === 0.U) -> 0.U(WORD_LEN.W),
        // 从 MEM 直通
        ((id_rs1_addr === mem_reg_wb_addr) && (mem_reg_rf_wen === REN_S)) -> mem_wb_data,
        // 从 WB 直通
        ((id_rs1_addr === wb_reg_wb_addr) && (wb_reg_rf_wen === REN_S)) -> wb_reg_wb_data

    ))
val id_rs2_data = MuxCase(regfile(id_rs2_addr), Seq(
        (id_rs2_addr === 0.U) -> 0.U(WORD_LEN.W),
        // 从 MEM 直通
        ((id_rs2_addr === mem_reg_wb_addr) && (mem_reg_rf_wen === REN_S)) -> mem_wb_data,
        // 从 WB 直通
        ((id_rs2_addr === wb_reg_wb_addr) && (wb_reg_rf_wen === REN_S)) -> wb_reg_wb_data
    ))
```

这样就完成了指令冒险，数据冒险的处理。

总结：如果遇到指令冒险，那么就等待 EXE 执行一周期，这样，EXE 将会再接下来的两个周期执行两次 BUBBLE，第三个周期才会执行到跳转的目标指令。原因就是 ID 在当前 decode 了 BUBBLE化的指令，同时 ID 的下一个周期还是 BUBBLE，下下个周期才是 IF 传上来的 目标指令。至于 IF，当前传送的指令被BUBBLE 重置，下一个周期自然就是 BUBBLE 了，当前传送的指令也没用，但是下一个周期传送的指令就是目标指令了。

如果是数据冒险，那么情况就有点复杂。

如果可以直通，那么就直通；如果不能直通，那么就等待一个周期后直通。等待的含义就是保持状态不变，IF 等待、ID 等待。

IF 等待只需要保持 if_pc_reg 不变就行了，也就是将 if_pc_next 置为 if_pc_reg。
ID 保持，只需要保持 id_reg_pc、id_reg_inst 不变就行了。另外再执行一下 BUBBLE化的指令，这是幂等指令，执行 N 次（N >= 0）都不会改变状态机有效状态。


至于 MEM、WB 就很简单了，没什么可讲的了。


















