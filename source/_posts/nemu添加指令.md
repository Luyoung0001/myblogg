---
title: NEMU 添加指令
date: 2024-08-14 11:22:33
tags:
    - 模拟器
    - NEMU
    - PA2
categories:
    - OS
    - RISCV
    - PA
---

<!--more-->

## 前言

前面讨论了 nemu 执行一条指令的过程，在源码中，可以看到它目前可以解析的指令有限：

```c
static int decode_exec(Decode* s) {
    int rd = 0;
    word_t src1 = 0, src2 = 0, imm = 0;
    s->dnpc = s->snpc;

#define INSTPAT_INST(s) ((s)->isa.inst.val)
#define INSTPAT_MATCH(s, name, type, ... /* execute body */)             \
    {                                                                    \
        decode_operand(s, &rd, &src1, &src2, &imm, concat(TYPE_, type)); \
        __VA_ARGS__;                                                     \
    }

    INSTPAT_START();
    INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc, U,
            R(rd) = s->pc + imm);
    INSTPAT("??????? ????? ????? 100 ????? 00000 11", lbu, I,
            R(rd) = Mr(src1 + imm, 1));
    INSTPAT("??????? ????? ????? 000 ????? 01000 11", sb, S,
            Mw(src1 + imm, 1, src2));

    INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak, N,
            NEMUTRAP(s->pc, R(10)));  // R(10) is $a0
    INSTPAT("??????? ????? ????? ??? ????? ????? ??", inv, N, INV(s->pc));
    INSTPAT_END();

    R(0) = 0;  // reset $zero to 0

    return 0;
}
```

因此它并不能运行某些程序，因为程序中存在的某些指令不能被 nemu 解析，我们要做到的，就是添加指令。这意味着要看手册，研究各个指令的语义，再来实现它。

手册的话，我看的是 《Chapter 24 RV32/64G Instruction Set Listings》，这一章会列出很多指令、指令类型。

## addi

```bash
luyoung at luyoungUbt in ~/ysyx-workbench/am-kernels/tests/cpu-tests (ics2021)
$ make ARCH=riscv32-nemu ALL=dummy run
...
...
(nemu) c
invalid opcode(PC = 0x80000000):
        13 04 00 00 17 91 00 00 ...
        00000413 00009117...
There are two cases which will trigger this unexpected exception:
1. The instruction at PC = 0x80000000 is not implemented.
2. Something is implemented incorrectly.
```

运行之后发现 00000413 无法解析，查看反汇编，发现是 li s0,0：

```txt
Disassembly of section .text:

80000000 <_start>:
80000000:	00000413          	li	s0,0
80000004:	00009117          	auipc	sp,0x9
80000008:	ffc10113          	addi	sp,sp,-4 # 80009000 <_end>
8000000c:	00c000ef          	jal	ra,80000018 <_trm_init>

80000010 <main>:
80000010:	00000513          	li	a0,0
80000014:	00008067          	ret

80000018 <_trm_init>:
80000018:	ff010113          	addi	sp,sp,-16
8000001c:	00000517          	auipc	a0,0x0
80000020:	01c50513          	addi	a0,a0,28 # 80000038 <_etext>
80000024:	00112623          	sw	ra,12(sp)
80000028:	fe9ff0ef          	jal	ra,80000010 <main>
8000002c:	00050513          	mv	a0,a0
80000030:	00100073          	ebreak
80000034:	0000006f          	j	80000034 <_trm_init+0x1c>
```

通过查阅，得知 li 是伪指令。它实际上会被翻译成 addi。因此，我们只需要将 li 当成 addi 的别名就行了。翻译指令的时候，nemu 只会查看 00000413 的特殊字段，只要解析了 00000413，li 自然就会执行。这里需要搞清楚的是，这里是可以实现伪指令的（但最好不要，因为要考虑到指令复杂性、正交性等等）。

li s0,0 指令的含义是:addi a0, x0, 0




