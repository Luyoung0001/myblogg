---
title: MIPS 处理器的设计
date: 2025-03-10 13:03:13
tags: MIPS
categories: CPU 设计
---

## Why MIPS?

学校的课程设计需要做一个 MIPS 的五级流水线单周期处理器，这是 [项目链接](https://github.com/XUPTF4/CPU_based_on_MIPS)。

MIPS 有一个特点，那就是延迟槽。如果在编译参数中将延迟槽关掉，那么每一个分支指令后面的延迟槽就会变成空指令，这为流水线的指令相关解决提供了便利。

这个项目提供了简单的测试环境，但是遗憾的是由于时间有限，并没有提供 difftest。

## 程序编译流程

- mips 交叉编译工具链编译成目标文件；
- 将目标文件的 .data 抽取到 dataMem 中，将.text 抽取到 instMem 中，这个过程自动化的；
- 通过链接脚本控制编译的数据段、程序段虚拟地址（CPU 发出的地址），以便正确访存；
- 可以将目标文件反汇编成 .txt 文件，利于读取查看；
- 就可以放到 vivado 中进行下板验证了；

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/Picture1.png)

## 程序运行逻辑

- 首先是_start，设置一个 sp，由于 instMem 是 reg [31:0] mem [0:1023]，大小为 4KB，因此 sp 设置为 4KB;
- 跳转到 _trm_init，这是一个监视 main 函数是否成功执行的环境，它调用 main 函数；
- main 函数就是各种测试了，比如 add.c sub.c 等等；

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/pic2.png)

## 让 MIPS 跑一个简单的 OS?

这个 OS 来源于 ysyx 项目的 yieldOS。功能是切换上下文。

原理很简单，只需要设置一个上下文结构体，A 进程调用yield()，yield() 是一个内联汇编：

```C
// 系统调用
void yield() {
    asm volatile(
        "syscall\n\t"
        "nop"
        :
        :
        :);
}
```

MIPS 执行到 syscall 指令的时候，就会跳转到 __am_asm_trap：

```asm

#define MAP(c, f) c(f)

#define REGS(f) \
      f( 1) f( 2) f( 3) f( 4) f( 5) f( 6) f( 7) f( 8) f( 9) \
f(10) f(11) f(12) f(13) f(14) f(15) f(16) f(17) f(18) f(19) \
f(20) f(21) f(22) f(23) f(24) f(25)             f(28)       \
f(30) f(31)

#define PUSH(n) sw $n, (n * 4)($sp);
#define POP(n)  lw $n, (n * 4)($v0);

#define CONTEXT_SIZE ((33) * 4)
#define OFFSET_SP     (29 * 4)



.set noat
.globl __am_asm_trap
__am_asm_trap:
  move $k1, $sp
  addiu $sp, $sp, -CONTEXT_SIZE

  MAP(REGS, PUSH)

  sw $k1, OFFSET_SP($sp)

  move $a0, $sp

  jal __am_irq_handle

  # 异常返回

  lw $a0, 16($v0)
  lw $t0, 128($v0) # 载入返回地址
  nop
  nop
  nop
  nop
  nop
  mtc0 $t0, $0      # 将返回地址载入 epc
  addiu $sp, $sp, CONTEXT_SIZE
  eret

```

保存上下文之后，调用调度函数，该函数返回一个新的上下文：
```C
int tag = 0;
Context* user_handler(Context* prev) {
    // 切换两个线程
    Context* c;
    tag++;
    if (tag % 2 == 0) {
        c = pcb[0].cp;
    } else {
        c = pcb[1].cp;
    }
    return c;
}

Context* __am_irq_handle(Context* c) {
    c = user_handler(c);

    return c;
}
```

之后，将这个上下文中的关键寄存器中的值，恢复到现场，然后 eret，返回到上次打断的地方，就可以继续运行了。

这个简单的 OS 和其它 OS 的上下文基本一致，区别无非就是CPU 结构的区别了。

