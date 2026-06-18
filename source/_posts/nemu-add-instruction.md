---
title: "NEMU 指令扩展入门：INSTPAT 与执行逻辑"
date: 2024-08-14 11:22:33
tags:
  - "NEMU"
  - "RISC-V"
  - "instruction set"
categories:
  - "Computer Architecture"
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

手册的话，我看的是 Chapter 24 RV32/64G Instruction Set Listings，这一章会列出很多指令、指令类型。

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

li s0,0 指令的含义是:addi a0, s0, 0。也就是 a0 = s0 + src1，因此我们只需查一下 addi 的指令格式：

| 31  | 27    | 26    | 25    | 24    | 23-20 | 19-15 | 14-12 | 11-7 | 6-0    |
|-----|-------|-------|-------|-------|-------|-------|-------|------|--------|
|     | imm[11:0] |       |       |   |    | rs1   | 000   | rd   | 0010011 |


然后就可以模仿示例进行解析和执行了：

```c
INSTPAT("??????? ????? ????? 000 ????? 00100 11", addi, I,
            R(rd) = src1 + imm);
```

重新编译运行：

```bash
(nemu) c
invalid opcode(PC = 0x8000000c):
        ef 00 c0 00 13 05 00 00 ...
        00c000ef 00000513...
There are two cases which will trigger this unexpected exception:
1. The instruction at PC = 0x8000000c is not implemented.
2. Something is implemented incorrectly.
```

可以看到，接下来是 00c000ef 指令无法解析，这个指令是：jal	ra,80000018。说明上一个指令已经解析成功了。接着解析 jal。

## jal


在 RISC-V 指令集中，`jal` 指令代表 "Jump and Link"。这是一种用于实现无条件跳转的控制流指令，同时它会将跳转后的下一条指令的地址保存到一个寄存器中，通常用于函数调用的实现。`jal` 的作用是将程序的执行从当前指令跳转到一个指定的目标地址，并且将返回地址（即跳转指令后的下一条指令的地址）保存在一个寄存器中。

### 指令格式
`jal` 指令在 RISC-V 中是一种 J-type 格式的指令。其操作和细节如下：

- **操作**：`jal rd, offset`
  - `rd`：这是目标寄存器，用于保存跳转后应该返回的地址（下一条指令的地址）。
  - `offset`：这是一个立即数，表示相对于当前指令的字节偏移量，用于计算跳转目标地址。

### 使用场景
`jal` 指令常用于函数调用。在调用函数前，它将返回地址保存到寄存器（通常是 `ra`，即返回地址寄存器），然后跳转到函数的起始地址执行。函数执行完毕后，可以通过跳转回保存的返回地址来继续执行调用后的指令。

指令格式为：

| 31  | 27    | 26    | 25    | 24    | 23-20 | 19-15 | 14-12 | 11-7 | 6-0    |
|-----|-------|-------|-------|-------|-------|-------|-------|------|--------|
|     | imm[20\|10:1\|11\|19:12] |       |       |   |    |    |    | rd   | 1101111 |


在执行指令的时候，首先是 fetch:
```c

int isa_exec_once(Decode* s) {
    s->isa.inst.val = inst_fetch(&s->snpc, 4);
    return decode_exec(s);
}

static inline uint32_t inst_fetch(vaddr_t *pc, int len) {
  uint32_t inst = vaddr_ifetch(*pc, len);
  (*pc) += len;
  return inst;
}
```

因此，在进行 decode 的时候，这时候 snpc 已经得到了更新，并且解析的时候，已经先将 dnpc 设置为 snpc，后面根据实际情况，比如是否跳转再由指令语义做决定修改。

返回地址的地址就是 cpu.pc or s->pc(当前执行的地址)+4；而跳转的指令就是 cpu.pc or s->pc+offset。这里建议统一访问 s而不是 cpu。

因此，指令的执行分为两步：

- 保存返回地址到特定寄存器：s->dnpc = s->pc + imm;
- 设置跳转地址：R(rd) = s->pc + 4;

这里还需要设置取 imm 的宏，由于这里是 J-type,因此得自己设置。

---

### 设置宏定义

首先看看示例指令auipc 是如何做到的：

```c
INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc, U,
            R(rd) = s->pc + imm);

...

#define immU()                                  \
    do {                                        \
        *imm = SEXT(BITS(i, 31, 12), 20) << 12; \
    } while (0)

#define SEXT(x, len) ({ struct { int64_t n : len; } __x = { .n = x }; (uint64_t)__x.n; })


#define BITS(x, hi, lo) (((x) >> (lo)) & BITMASK((hi) - (lo) + 1))


#define BITMASK(bits) ((1ull << (bits)) - 1)

```

首先 BITMASK(bits) 顾名思义就是求掩码，比如 BITMASK(3)，那就是 1 << 3 ==> 1000-1==>111，很好理解。

接着 BITS(x,hi,lo) 就是取一个字中的某些位，因为这涉及到和掩码与的操作，比如解析auipc	sp,0x9 的时候，x 为 0x00009117，二进制就是：00000000 00000000 10010001 00010111。其中[31:12]为 imm去要提取出来。那么 BITS(00000000 00000000 10010001 00010111, 31, 12)就是：

- x 左移 12：00000000 00000000 00000000 00001001
- 和掩码与运算：00000000 00000000 00000000 00001001 & 00000000 00000000 00000011 1111 1111 = 00000000 00000000 00000000 00001001
- 接着扩展，SETX(00000000 00000000 00000000 00001001,20): 值在 32 位中不变。
- 接着右移补零：00000000 00000000 00000000 00001001 << 12: 00000000 00000000 10010000 00000000

那么就成功得提取出了 imm：00000000 00000000 10010000 00000000。可见，取 imm 的最终就是将指令字中的imm 取出拼凑成无符号数。


那么 J-type 就很好设置了，J-type 的指令格式为：imm[20\|10:1\|11\|19:12]，那么就可以如此提取：

```c
*imm = SEXT(((BITS(i, 31, 31) << 19) | \
               BITS(i, 30, 21) |  \
               (BITS(i, 20, 20) << 10) | \
               (BITS(i, 19, 12) << 11)) << 1,21); \
```


---

因此指令为：

```c
    INSTPAT("??????? ????? ????? ??? ????? 11011 11", jal, J,
            s->dnpc = s->pc + imm; R(rd) = s->pc + 4);
```

继续编译和运行：

```bash
(nemu) c
invalid opcode(PC = 0x80000024):
        23 26 11 00 ef f0 9f fe ...
        00112623 fe9ff0ef...
There are two cases which will trigger this unexpected exception:
1. The instruction at PC = 0x80000024 is not implemented.
2. Something is implemented incorrectly.
```

可以看到，接下来是 00112623 指令无法解析，这个指令是：sw	ra,12(sp)。说明上一个指令已经解析成功了。接着解析 sw。

## sw

在 RISC-V 指令集中，`sw` 指令表示“Store Word”。这是一种存储指令，用于将一个寄存器的内容写入到内存中。`sw` 是 S-type（存储类型）指令的一部分，用于处理内存的写操作。

### 指令格式
`sw` 指令的基本格式是：
```
sw rs2, offset(rs1)
```
- `rs1`：基址寄存器，提供存储地址的基础部分。
- `offset`：一个立即数偏移量，与基址寄存器的内容相加得到最终的内存地址。
- `rs2`：源寄存器，其内容将被写入到由 `rs1` 和 `offset` 计算出的内存地址中。

### 操作
`sw` 指令将寄存器 `rs2` 中的内容存储到从寄存器 `rs1` 读取的地址加上 `offset` 的位置。存储的数据大小是一个字（word），在 RISC-V 标准中，一个字通常是 32 位（4 字节）。

### 使用场景
`sw` 指令常用于将计算结果或变量的值保存回内存，或在函数调用中保存寄存器的内容到栈中（如果使用基于栈的调用约定）。这对于实现复杂的数据处理、函数调用以及操作系统功能等都是必需的。

指令格式为：

| 31  | 27    | 26    | 25    | 24    | 23-20 | 19-15 | 14-12 | 11-7 | 6-0    |
|-----|-------|-------|-------|-------|-------|-------|-------|------|--------|
|imm[11:15] |    |   |  |         rs2 |  |  rs1  |  000  | imm[4:0]   | 0100011


因此，sw 这样执行：M(R[rs1] + offset) = R[rs2]。

我们有接口，因此可以直接调用：

```c
void vaddr_write(vaddr_t addr, int len, word_t data) {
  paddr_write(addr, len, data);
}
```

即为：vaddr_write(rs1+imm,4,rs2)。但是这个函数名太长，我们直接定义宏：

```c
#define Mr vaddr_read
#define Mw vaddr_write
```

因此可以这样解析和执行：

```c
INSTPAT("??????? ????? ????? 010 ????? 01000 11", sw, S,
            Mw(src1 + imm, 4, src2));
```

继续编译和执行：

```bash
(nemu) c
invalid opcode(PC = 0x80000014):
        67 80 00 00 13 01 01 ff ...
        00008067 ff010113...
There are two cases which will trigger this unexpected exception:
1. The instruction at PC = 0x80000014 is not implemented.
2. Something is implemented incorrectly.
```

可以看到，接下来是 00008067 指令无法解析，这个指令是：ret。说明上一个指令已经解析成功了。接着解析 ret.

## ret

在 RISC-V 指令集中，`ret` 指令是一个伪指令，用于从一个函数返回。它通常被解释为 `jalr` 指令的一个特定形式，具体来说就是 `jalr x0, 0(ra)`。这个操作意味着它会从寄存器 `ra`（返回地址寄存器）进行跳转，其中 `x0` 是一个永远为零的寄存器，因此不会对返回地址进行任何修改。

### 功能解释

- **`jalr` 指令**: `jalr` 是一个跳转并链接寄存器指令，它将当前指令的下一个地址（即跳转指令的地址 + 4）保存到指定的寄存器中，并跳转到目标寄存器加上一个立即数的地址。在 `ret` 指令的情况下，这个操作简化为从 `ra` 寄存器指定的地址进行跳转，因为立即数为0。

- **`x0` 寄存器**: 这是一个特殊的寄存器，其值永远是0。在 `ret` 指令中使用 `x0` 作为目标寄存器意味着跳转指令的返回地址不会被保存，这是因为在函数返回时，我们不需要在 `ra` 寄存器之外再保存一个返回地址。

### 使用场景

`ret` 指令通常用在函数的末尾，实现从函数调用返回到调用者的代码位置。在大多数高级语言中，这对应于函数的结束括号或者 `return` 语句。

### 示例

如果有一个函数 `foo` 调用了另一个函数 `bar`，并且 `bar` 函数执行完毕后需要返回到 `foo`，那么在 `bar` 的末尾就会有一个 `ret` 指令，如下所示：

```assembly
foo:
    ...
    jal ra, bar  # 调用 bar 函数
    ...
bar:
    ...
    ret          # 返回到 foo 函数的调用后的下一条指令
```

这样，`ret` 指令确保程序从 `bar` 函数跳回 `foo` 函数中 `jal` 指令之后的位置继续执行。这是现代计算机体系结构中实现函数调用和返回机制的一个基本组成部分。

因此，我们要实现 jalr 指令的解析和执行：

- 下一条指令的地址即 s->pc + 4 保存：R(rd) = s->pc + 4);
- 并跳转到目标寄存器加上一个立即数(x0=0)的地址：s->dnpc = (src1 + imm) & ~(word_t)1; // 在 RISC-V 架构中，指令地址必须对齐到偶数地址


因此，ret 地址可以这样被解析和执行：

```c
    INSTPAT("??????? ????? ????? 000 ????? 11001 11", jalr, I,
            s->dnpc = (src1 + imm) & ~(word_t)1; // 在 RISC-V 架构中，指令地址必须对齐到偶数地址
            R(rd) = s->pc + 4);
```

编译和执行：

```bash
(nemu) c
[src/cpu/cpu-exec.c:164 cpu_exec] nemu: HIT GOOD TRAP at pc = 0x80000030
[src/cpu/cpu-exec.c:121 statistic] host time spent = 196 us
[src/cpu/cpu-exec.c:122 statistic] total guest instructions = 13
[src/cpu/cpu-exec.c:124 statistic] simulation frequency = 66,326 inst/s
```

根据讲义，做到这里，说明第一阶段已经成功了~