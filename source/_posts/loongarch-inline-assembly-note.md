---
title: "LoongArch32R 内联汇编笔记：CSR 写入的一个细节"
date: 2026-01-07 15:03:13
tags:
  - "LoongArch"
  - "inline assembly"
  - "CSR"
categories:
  - "Computer Architecture"
mermaid: true
---

## 前言
前置知识：“CSRWR 指令将通用寄存器 rd 中的旧值写入到指定 CSR 中，同时将指定CSR的旧值更新到通用寄存器 rd 中。“

LoongArch 的调用规则：

  | 寄存器 | 别名 | 用途        |
  |--------|------|-------------|
  | $r4    | a0   | 第 1 个参数 |
  | $r5    | a1   | 第 2 个参数 |
  | $r6    | a2   | 第 3 个参数 |
  | ...    | ...  | ...         |


我在初始化 csr_eentry 的时候，打算将 `trap_entry` 这个函数地址保存到 `csr_eentry`，将`ecfg` 也更新。之后打印出日志：

```C
#define write_csr_eentry(v) csr_write(CSR_EENTRY, v)
static inline void csr_write(uint32_t csr, uint32_t val) {
    asm volatile (
        "csrwr %0, %1"
        :
        : "r"(val),"i"(csr)
    );
}


void trap_init(void) {
    for (int i = 0; i < IRQ_MAX; i++) {
        irq_handlers[i] = default_irq_handler;
    }
    // bypass GOT
    uint32_t entry_addr;
    asm volatile ("la.local %0, trap_entry" : "=r"(entry_addr));

    /* Set exception entry point */
    write_csr_eentry(entry_addr);
    /* Enable UART and PS2 interrupts in ECFG */
    uint32_t ecfg = ECFG_LIE_HWI1 | ECFG_LIE_HWI2; /* UART + PS2 */
    write_csr_ecfg(ecfg);

    myprintf("[TRAP] Initialized, eentry=0x%08x\n", entry_addr);
}
```

编译出来的汇编是这样的：

```asm
00000a10 <trap_init>:
     a10:	1c00002c 	pcaddu12i	$r12,1(0x1)
     a14:	02b1d18c 	addi.w	$r12,$r12,-908(0xc74)
     a18:	1c00000e 	pcaddu12i	$r14,0
     a1c:	02b9a1ce 	addi.w	$r14,$r14,-408(0xe68)
     a20:	0280d18d 	addi.w	$r13,$r12,52(0x34)
     a24:	2980018e 	st.w	$r14,$r12,0
     a28:	0280118c 	addi.w	$r12,$r12,4(0x4)
     a2c:	5ffff98d 	bne	$r12,$r13,-8(0x3fff8) # a24 <trap_init+0x14>
     a30:	1dffffe5 	pcaddu12i	$r5,-1(0xfffff)
     a34:	029800a5 	addi.w	$r5,$r5,1536(0x600)
     a38:	04003025 	csrwr	$r5,0xc
     a3c:	0280600c 	addi.w	$r12,$r0,24(0x18)
     a40:	0400102c 	csrwr	$r12,0x4
     a44:	1c000024 	pcaddu12i	$r4,1(0x1)
     a48:	02a58084 	addi.w	$r4,$r4,-1696(0x960)
     a4c:	5001a400 	b	420(0x1a4) # bf0 <myprintf>
```
可以看到，csrwr 操作了 r5 以后，r5 的值就会被改变。而 `myprintf` 函数的参数是: `r4`、`r5`，前者放一个字符串，后者就是改变了值的变量 `entry_addr`。

这也导致我打印了错的 eentry：

```bash
xOS Booting...
[TRAP] Initialized, eentry=0x00000000
[INIT] Interrupts enabled

========================================
  xOS - Simple Operating System
  for LoongArch32R SoC
========================================

Type 'help' for available commands.

xos>
```

## 解决

```C
void trap_init(void) {
    for (int i = 0; i < IRQ_MAX; i++) {
        irq_handlers[i] = default_irq_handler;
    }
    uint32_t entry_addr;
    asm volatile ("la.local %0, trap_entry" : "=r"(entry_addr));

    uint32_t saved_entry = entry_addr;

    write_csr_eentry(entry_addr);

    uint32_t ecfg = ECFG_LIE_HWI1 | ECFG_LIE_HWI2; /* UART + PS2 */
    write_csr_ecfg(ecfg);

    myprintf("[TRAP] Initialized, eentry=0x%08x\n", saved_entry);
}
```

这样就能解决了吗？不行，编译出来的代码没变：

```asm
00000a10 <trap_init>:
     a10:	1c00002c 	pcaddu12i	$r12,1(0x1)
     a14:	02b1d18c 	addi.w	$r12,$r12,-908(0xc74)
     a18:	1c00000e 	pcaddu12i	$r14,0
     a1c:	02b9a1ce 	addi.w	$r14,$r14,-408(0xe68)
     a20:	0280d18d 	addi.w	$r13,$r12,52(0x34)
     a24:	2980018e 	st.w	$r14,$r12,0
     a28:	0280118c 	addi.w	$r12,$r12,4(0x4)
     a2c:	5ffff98d 	bne	$r12,$r13,-8(0x3fff8) # a24 <trap_init+0x14>
     a30:	1dffffe5 	pcaddu12i	$r5,-1(0xfffff)
     a34:	029800a5 	addi.w	$r5,$r5,1536(0x600)
     a38:	04003025 	csrwr	$r5,0xc
     a3c:	0280600c 	addi.w	$r12,$r0,24(0x18)
     a40:	0400102c 	csrwr	$r12,0x4
     a44:	1c000024 	pcaddu12i	$r4,1(0x1)
     a48:	02a58084 	addi.w	$r4,$r4,-1696(0x960)
     a4c:	5001a400 	b	420(0x1a4) # bf0 <myprintf>
```
因为编译器认为，我的内联汇编的两个输入寄存器不会变化，因此 `uint32_t saved_entry = entry_addr` 代码没有必要，于是什么都没有发生。

很显然，**编译没有识别出 `csrwr` 的内涵，造成了错误的优化。**

那么怎么修改才能达到我的代码目的？

"+r"(val) — 读写 ✓ 正确
```C
  asm volatile (
      "csrwr %0, %1"
      : "+r"(val)     // 既是输入又是输出
      : "i"(csr)
  );
```

编译器知道这个寄存器既被读取又被修改，于是正确地理解了 `myprintf("[TRAP] Initialized, eentry=0x%08x\n", saved_entry)` 的语义，确保 `saved_entry` 不会被修改，于是保护了它：

```asm
00000a10 <trap_init>:
     a10:	1c00002c 	pcaddu12i	$r12,1(0x1)
     a14:	02b2118c 	addi.w	$r12,$r12,-892(0xc84)
     a18:	1c00000e 	pcaddu12i	$r14,0
     a1c:	02b9a1ce 	addi.w	$r14,$r14,-408(0xe68)
     a20:	0280d18d 	addi.w	$r13,$r12,52(0x34)
     a24:	2980018e 	st.w	$r14,$r12,0
     a28:	0280118c 	addi.w	$r12,$r12,4(0x4)
     a2c:	5ffff98d 	bne	$r12,$r13,-8(0x3fff8) # a24 <trap_init+0x14>
     a30:	1dffffe5 	pcaddu12i	$r5,-1(0xfffff)
     a34:	029800a5 	addi.w	$r5,$r5,1536(0x600)
     a38:	001500ac 	move	$r12,$r5
     a3c:	0400302c 	csrwr	$r12,0xc
     a40:	0280600c 	addi.w	$r12,$r0,24(0x18)
     a44:	0400102c 	csrwr	$r12,0x4
     a48:	1c000024 	pcaddu12i	$r4,1(0x1)
     a4c:	02a5b084 	addi.w	$r4,$r4,-1684(0x96c)
     a50:	5001b000 	b	432(0x1b0) # c00 <myprintf>
     a54:	03400000 	andi	$r0,$r0,0x0
     a58:	03400000 	andi	$r0,$r0,0x0
     a5c:	03400000 	andi	$r0,$r0,0x0
```

可以看到，编译器用 r12 去做 csrwr，这样 r5 就不会改变，被保护起来了。

## 总结
完整的逻辑链
1. "+r"(val) 告诉编译器：这个操作数会被修改
2. 编译器分析数据流，发现：
  - entry_addr 在 csr_write 之后还要被 myprintf 使用
  - 但 csrwr 会修改操作数寄存器
3. 编译器决策：
  - 不能直接用 entry_addr 的寄存器（r5）执行 csrwr
  - 必须先复制到临时寄存器（r12）
  - 用 r12 执行 csrwr（r12 被破坏无所谓）
  - r5 保持不变，myprintf 可以正常使用

生成的代码正是这个逻辑：
```asm
  addi.w  r5, r5, 1536       # r5 = entry_addr
  move    r12, r5            # 复制到 r12（因为 csrwr 会破坏操作数）
  csrwr   r12, 0xc           # r12 被修改成 CSR 旧值，r5 安全
  ...
  myprintf(..., r5)          # r5 还是 entry_addr ✓
```
如果没有 "+r"
1. "r"(val) 告诉编译器：这只是输入，不会被修改
2. 编译器错误地认为:
  - csrwr 不会改变操作数
  - 可以直接用 r5 执行 csrwr
3. 结果：
  - csrwr r5, 0xc 执行后，r5 变成 CSR 旧值
  - myprintf 拿到的是错误的值

一句话总结："+r" 让编译器知道寄存器会被修改，从而在需要时保护后续还要使用的变量。















