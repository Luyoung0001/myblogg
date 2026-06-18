---
title: "异常响应机制笔记：从 ECALL 到控制流切换"
date: 2024-10-18 16:20:40
tags:
  - "exception"
  - "ysyx"
  - "CPU"
categories:
  - "Computer Architecture"
---

<!--more-->

## 前言

在 riscv 中，当发生异常的时候，需要调用 ecall 指令来陷入异常:

```C
#define ECALL(dnpc)                                                       \
    {                                                                     \
        bool success;                                                     \
        dnpc = (isa_raise_intr(isa_reg_str2val("$a7", &success), s->pc)); \
    }

word_t isa_raise_intr(word_t NO, vaddr_t epc) {

    IFDEF(CONFIG_ETRACE, etrace(NO, epc));
    cpu.csr.mcause = NO;
    cpu.csr.mepc = epc+4;
    return cpu.csr.mtvec;
}
```

同时，ecall 会将当前的异常号保存到 mcause、将下一条指令的地址保存到 mepc(事实上，这个是否+4 取决于异常类型，后面会说到)以及设置异常处理入口，之后就进入到了异常处理阶段了。事实上，整个过程令人惊叹~

## 一个例子

```C
#include <amtest.h>

void (*entry)() = NULL; // mp entry

static const char *tests[256] = {
  ['h'] = "hello",
  ['H'] = "display this help message",
  ['i'] = "interrupt/yield test",
  ['d'] = "scan devices",
  ['m'] = "multiprocessor test",
  ['t'] = "real-time clock test",
  ['k'] = "readkey test",
  ['v'] = "display test",
  ['a'] = "audio test",
  ['p'] = "x86 virtual memory test",
};

int main(const char *args) {
  switch (args[0]) {
    CASE('h', hello);
    CASE('i', hello_intr, IOE, CTE(simple_trap));
    CASE('d', devscan, IOE);
    CASE('m', mp_print, MPE);
    CASE('t', rtc_test, IOE);
    CASE('k', keyboard_test, IOE);
    CASE('v', video_test, IOE);
    CASE('a', audio_test, IOE);
    CASE('p', vm_test, CTE(vm_handler), VME(simple_pgalloc, simple_pgfree));
    case 'H':
    default:
      printf("Usage: make run mainargs=*\n");
      for (int ch = 0; ch < 256; ch++) {
        if (tests[ch]) {
          printf("  %c: %s\n", ch, tests[ch]);
        }
      }
  }
  return 0;
}

```
这个是主程序，我们需要关注这个程序的 i,也就是异常测试程序。它运行这个程序之前，它会 调用宏CTE(simple_trap)，这是在做：

```C
#define CTE(h) ({ Context *h(Event, Context *); cte_init(h); })
```
事实上：

```C
static Context* (*user_handler)(Event, Context*) = NULL;

bool cte_init(Context* (*handler)(Event, Context*)) {
    // initialize exception entry
    asm volatile("csrw mtvec, %0" : : "r"(__am_asm_trap));

    // register event handler
    user_handler = handler;

    return true;
}
```

这里把回调函数user_handler 设置为用户传入 simple_trap 为处理函数。

可以看到，cte_init() 把__am_asm_trap 设为异常处理函数：
```C
#define concat_temp(x, y) x ## y
#define concat(x, y) concat_temp(x, y)
#define MAP(c, f) c(f)

#if __riscv_xlen == 32
#define LOAD  lw
#define STORE sw
#define XLEN  4
#else
#define LOAD  ld
#define STORE sd
#define XLEN  8
#endif

#define REGS_LO16(f) \
      f( 1)       f( 3) f( 4) f( 5) f( 6) f( 7) f( 8) f( 9) \
f(10) f(11) f(12) f(13) f(14) f(15)
#ifndef __riscv_e
#define REGS_HI16(f) \
                                    f(16) f(17) f(18) f(19) \
f(20) f(21) f(22) f(23) f(24) f(25) f(26) f(27) f(28) f(29) \
f(30) f(31)
#define NR_REGS 32
#else
#define REGS_HI16(f)
#define NR_REGS 16
#endif

#define REGS(f) REGS_LO16(f) REGS_HI16(f)

#define PUSH(n) STORE concat(x, n), (n * XLEN)(sp);
#define POP(n)  LOAD  concat(x, n), (n * XLEN)(sp);

#define CONTEXT_SIZE  ((NR_REGS + 3) * XLEN)
#define OFFSET_SP     ( 2 * XLEN)
#define OFFSET_CAUSE  ((NR_REGS + 0) * XLEN)
#define OFFSET_STATUS ((NR_REGS + 1) * XLEN)
#define OFFSET_EPC    ((NR_REGS + 2) * XLEN)

.align 3
.globl __am_asm_trap
__am_asm_trap:
  addi sp, sp, -CONTEXT_SIZE

  MAP(REGS, PUSH)

  csrr t0, mcause
  csrr t1, mstatus
  csrr t2, mepc

  STORE t0, OFFSET_CAUSE(sp)
  STORE t1, OFFSET_STATUS(sp)
  STORE t2, OFFSET_EPC(sp)

  # set mstatus.MPRV to pass difftest
  li a0, (1 << 17)
  or t1, t1, a0
  csrw mstatus, t1

  mv a0, sp
  jal __am_irq_handle

  LOAD t1, OFFSET_STATUS(sp)
  LOAD t2, OFFSET_EPC(sp)
  csrw mstatus, t1
  csrw mepc, t2

  MAP(REGS, POP)

  addi sp, sp, CONTEXT_SIZE
  mret

```
这个程序做了几件事：
- 为 context 开辟空间；
- 将 通用寄存器保存到栈上；
- 将 csr 寄存器保存到栈上；
- 跳转到异常处理函数；
- 处理完后返回；
- 恢复 status 到 mstatus；
- 恢复 dnpc 到 mepc
- 恢复 sp
- 处理完成，返回（接着执行 dnpc）

我们看看__am_irq_handle():

```C
Context* __am_irq_handle(Context* c) {
    if (user_handler) {
        Event ev = {0};
        switch (c->mcause) {
            case -1:
                ev.event = EVENT_YIELD;
                break;
            default:
                ev.event = EVENT_ERROR;
                break;
        }

        c = user_handler(ev, c);
        assert(c != NULL);
    }

    return c;
}

```

这一节可谓是惊为天人，它把 sp 栈上的数据排布的完全和 context 一样，然后通过指针转型获取传入的 context。

然后就是根据 a7 的异常号，再次确定事件编号。然后根据事件编号，传入到回调函数 user_handler()，就可以处理了。处理好了后，返回到 trap.S，然后返回，整个过程就结束了。

我们来看看用户程序：

```C
#include <amtest.h>

Context* simple_trap(Event ev, Context* ctx) {
    switch (ev.event) {
        case EVENT_IRQ_TIMER:
            putch('t');
            break;
        case EVENT_IRQ_IODEV:
            putch('d');
            break;
        case EVENT_YIELD:
            putch('y');
            break;
        default:
            panic("Unhandled event");
            break;
    }
    return ctx;
}

void hello_intr() {
    printf("Hello, AM World @ " __ISA__ "\n");
    printf("  t = timer, d = device, y = yield\n");
    io_read(AM_INPUT_CONFIG);
    iset(1);
    while (1) {
        for (volatile int i = 0; i < 10000000; i++)
            ;
        yield();
    }
}

```

这个是一个main 的 子程序，它实现了trap 处理函数 simple_trap，这个函数会根据应用程序执行不同的陷入来执行不同的操作，比如输出 t、d、y 等。

hello_intr() 就很简单了，首先它会 io_read、iset(1)，事实上没做什么，这里不用考虑。接着为了防止 yield()太快，它加入了一段空循环，然后死循环。

yield()做了两件事:

```C
void yield() {
#ifdef __riscv_e
    asm volatile("li a5, -1; ecall");
#else
    asm volatile("li a7, -1; ecall");
#endif
}
```

它将 -1 加载到 a7 寄存器，然后 ecall，这里是的 ecall 实现如下：

```C
#define ECALL(dnpc)                                                       \
    {                                                                     \
        bool success;                                                     \
        dnpc = (isa_raise_intr(isa_reg_str2val("$a7", &success), s->pc)); \
    }
```

在执行 ecall 的时候，得做两件事：
- 取回 a7 中的异常号；
- 处理进入异常处理之前的事情。

也就是 前面说到的：

```C
word_t isa_raise_intr(word_t NO, vaddr_t epc) {

    IFDEF(CONFIG_ETRACE, etrace(NO, epc));
    cpu.csr.mcause = NO;
    cpu.csr.mepc = epc+4;
    return cpu.csr.mtvec;
}
```

之后，就进入了__am_asm_trap，然后保存现场、传入context 给处理函数处理，确定事件类型，然后传给用户程序处理。然后返回到 trap.S，然后恢复现场，返回 mret:

```C
#define MRET(dnpc)                                                       \
    {                                                                    \
        dnpc = cpu.csr.mepc;                                             \
    }
```

整个过程很有意思，处处体现了解耦的思想：
- 用户想要进行特殊的中断处理，必须提供 handler;
- 中断向量调用的程序，会调用一个专门的函数，这个函数会根据事件类型再次调用用户的 handler，让它自己处理。

整个过程如下：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/20241018.png)


## 后话

为什么dnpc要+4?

答案很简单，如果不加 4，那么它就不断得重复 ecall，因此它的 ftrace 就像这样：

```bash
0x8000152c:     call [ __am_irq_handle@0x800013c8
0x80001420:      call [ simple_trap@0x80000ed8
0x80000f64:       call [ putch@0x8000101c
y0x80001024:      ret [ simple_trap@0x80000ed8
0x80000f78:     ret [ __am_irq_handle@0x800013c8
0x8000152c:     call [ __am_irq_handle@0x800013c8
0x80001420:      call [ simple_trap@0x80000ed8
0x80000f64:       call [ putch@0x8000101c
y0x80001024:      ret [ simple_trap@0x80000ed8
0x80000f78:     ret [ __am_irq_handle@0x800013c8
0x8000152c:     call [ __am_irq_handle@0x800013c8
0x80001420:      call [ simple_trap@0x80000ed8
0x80000f64:       call [ putch@0x8000101c
y0x80001024:      ret [ simple_trap@0x80000ed8
0x80000f78:     ret [ __am_irq_handle@0x800013c8
0x8000152c:     call [ __am_irq_handle@0x800013c8
0x80001420:      call [ simple_trap@0x80000ed8
0x80000f64:       call [ putch@0x8000101c
y0x80001024:      ret [ simple_trap@0x80000ed8
0x80000f78:     ret [ __am_irq_handle@0x800013c8
0x8000152c:     call [ __am_irq_handle@0x800013c8
0x80001420:      call [ simple_trap@0x80000ed8
```

如果+4 那么它在处理完异常之后，就干别的事了：

```bash
0x80001014:    call [ yield@0x80001478
0x8000152c:     call [ __am_irq_handle@0x800013c8
0x80001420:      call [ simple_trap@0x80000ed8
0x80000f64:       call [ putch@0x8000101c
y0x80001024:      ret [ simple_trap@0x80000ed8
0x80000f78:     ret [ __am_irq_handle@0x800013c8
0x80001480:   ret [ hello_intr@0x80000fb4
0x80001014:    call [ yield@0x80001478
0x8000152c:     call [ __am_irq_handle@0x800013c8
0x80001420:      call [ simple_trap@0x80000ed8
0x80000f64:       call [ putch@0x8000101c
y0x80001024:      ret [ simple_trap@0x80000ed8
0x80000f78:     ret [ __am_irq_handle@0x800013c8
0x80001480:   ret [ hello_intr@0x80000fb4
0x80001014:    call [ yield@0x80001478
0x8000152c:     call [ __am_irq_handle@0x800013c8
0x80001420:      call [ simple_trap@0x80000ed8
0x80000f64:       call [ putch@0x8000101c
y0x80001024:      ret [ simple_trap@0x80000ed8
0x80000f78:     ret [ __am_irq_handle@0x800013c8
```

可以看到，他们的过程完全不同。虽然都是打印字符 'y'。













