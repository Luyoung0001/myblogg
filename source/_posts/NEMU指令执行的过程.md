---
title: NEMU 指令执行过程
date: 2024-08-13 11:22:33
tags:
    - 模拟器
    - NEMU
categories:
    - OS
    - RISCV
---

<!--more-->

## NEMU 架构

NEMU 是一个CPU模拟器，它使用 C语言 完成了模拟取指令、指令解析、指令执行、寄存器、初始化等，可以执行很多平台的任何二进制指令。

## 初始化

nemu main 中进行了一些初始化，这里只研究关键的初始化：

```c
    init_mem();

    /* Initialize devices. */
    IFDEF(CONFIG_DEVICE, init_device());

    /* Perform ISA dependent initialization. */
    init_isa();

    /* Load the image to memory. This will overwrite the built-in image. */
    long img_size = load_img();

    /* Initialize differential testing. */
    init_difftest(diff_so_file, img_size, difftest_port);
```

### init_mem()


```c

#if   defined(CONFIG_PMEM_MALLOC)
static uint8_t *pmem = NULL;
#else // CONFIG_PMEM_GARRAY
static uint8_t pmem[CONFIG_MSIZE] PG_ALIGN = {};
#endif

void init_mem() {
#if   defined(CONFIG_PMEM_MALLOC)
  pmem = malloc(CONFIG_MSIZE);
  assert(pmem);
#endif
  IFDEF(CONFIG_MEM_RANDOM, memset(pmem, rand(), CONFIG_MSIZE));
  Log("physical memory area [" FMT_PADDR ", " FMT_PADDR "]", PMEM_LEFT, PMEM_RIGHT);
}

```

可以看到，这里提供了两种内存分配方式，如果定义了CONFIG_PMEM_MALLOC，就用 malloc 的方式，否则就用静态数组的方式。然后初始化为随机数：memset(pmem, rand(), CONFIG_MSIZE)。

### 内存结构

首先，nemu 运行的时候，充当的是 host；host 中可以运行一些程序，比如 YEMU 中运行的 prog.bin。

x86的物理内存是从0开始编址的， 但对于一些ISA来说却不是这样， 例如mips32和riscv32的物理地址均从0x80000000开始。因此对于mips32和riscv32，其CONFIG_MBASE将会被定义成0x80000000。

将来CPU访问内存时, 我们会将CPU将要访问的内存地址映射到pmem中的相应偏移位置, 这是通过nemu/src/memory/paddr.c中的guest_to_host()函数实现的. 例如如果mips32的CPU打算访问内存地址0x80000000, 我们会让它最终访问pmem[0], 从而可以正确访问客户程序的第一条指令。

这种机制有一个专门的名字, 叫地址映射。

假如 nemu 要运行 prog.bin文件，首先要被加载到 host 提供的内存中，比如 pmem。假设 bin 程序中由要访问 paddr 的地址的行为，那么 C 语言应该怎么模拟这一过程？（一般编译的时候，会加上起始地址。编译好之后，地址均为绝对地址，具体看平台）

首先要将 paddr 的地址转化为 host 能访问的地址，对于 riscv，内存是从0x80000000 开始的。因此，paddr 是物理地址，它相对于0x80000000的相对地址为paddr - CONFIG_MBASE，那么它自然就位于 pmem + paddr - CONFIG_MBAS中。


同样，host 中 的某一个地址怎么映射到 paddr 呢？比如 haddr,首先 haddr - pmem，这是相对地址，然后再加上0x80000000。

---





### 客户机与宿主机
现在，再来看看几个重要的函数：

```c
uint8_t* guest_to_host(paddr_t paddr) { return pmem + paddr - CONFIG_MBASE; }
paddr_t host_to_guest(uint8_t *haddr) { return haddr - pmem + CONFIG_MBASE; }

```


paddr - CONFIG_MBASE：从传入的物理地址 paddr 中减去基址 CONFIG_MBASE。这个步骤是为了将客户机的物理地址转换为相对于 pmem 数组起始位置的偏移量。

pmem + (paddr - CONFIG_MBASE)：将上一步计算的偏移量与 pmem 的起始地址相加，得到相应的主机内存地址。这是实际指向客户机物理地址在主机内存中对应位置的指针。

haddr - pmem：计算 haddr 指针与 pmem 起始地址之间的差值，即从模拟的物理内存起始位置到当前 haddr 指针位置的偏移量。

haddr - pmem + CONFIG_MBASE：将上述计算出的偏移量加上基址 CONFIG_MBASE。这样做的目的是将主机内存中的地址转换为客户机的物理地址。通过加上 CONFIG_MBASE，实际上是恢复了这个地址在客户机内存空间中的原始位置。

这个过程上面已经提到过了。

```c
static inline word_t host_read(void *addr, int len) {
  switch (len) {
    case 1: return *(uint8_t  *)addr;
    case 2: return *(uint16_t *)addr;
    case 4: return *(uint32_t *)addr;
    IFDEF(CONFIG_ISA64, case 8: return *(uint64_t *)addr);
    default: MUXDEF(CONFIG_RT_CHECK, assert(0), return 0);
  }
}

static inline void host_write(void *addr, int len, word_t data) {
  switch (len) {
    case 1: *(uint8_t  *)addr = data; return;
    case 2: *(uint16_t *)addr = data; return;
    case 4: *(uint32_t *)addr = data; return;
    IFDEF(CONFIG_ISA64, case 8: *(uint64_t *)addr = data; return);
    IFDEF(CONFIG_RT_CHECK, default: assert(0));
  }
}
```

这两个函数非常重要，前者是返回 host 中的 addr 处的数据，后者是往 host 中的 addr 处的地址写数据。




```c
static word_t pmem_read(paddr_t addr, int len) {
  word_t ret = host_read(guest_to_host(addr), len);
  return ret;
}

static void pmem_write(paddr_t addr, int len, word_t data) {
  host_write(guest_to_host(addr), len, data);
}

```

而这两个函数做的就是这样的事，顶层是 pmem_read，底层是 host_read。





```c

static void out_of_bound(paddr_t addr) {
  panic("address = " FMT_PADDR " is out of bound of pmem [" FMT_PADDR ", " FMT_PADDR "] at pc = " FMT_WORD,
      addr, PMEM_LEFT, PMEM_RIGHT, cpu.pc);
}

word_t paddr_read(paddr_t addr, int len) {
  if (likely(in_pmem(addr))) return pmem_read(addr, len);
  IFDEF(CONFIG_DEVICE, return mmio_read(addr, len));
  out_of_bound(addr);
  return 0;
}

void paddr_write(paddr_t addr, int len, word_t data) {
  if (likely(in_pmem(addr))) { pmem_write(addr, len, data); return; }
  IFDEF(CONFIG_DEVICE, mmio_write(addr, len, data); return);
  out_of_bound(addr);
}
```

还有一些宏定义：

```c
#define CONFIG_MSIZE 0x8000000
#define CONFIG_PC_RESET_OFFSET 0x0
#define CONFIG_MBASE 0x80000000
```

其中，CONFIG_MSIZE 是host内存大小、CONFIG_PC_RESET_OFFSET host pc 的初始值、#define CONFIG_MBASE 0x80000000 是host的base。

### init_isa()

init_isa 就是做了一些状态机的初始化工作：

```c
// this is not consistent with uint8_t
// but it is ok since we do not access the array directly
static const uint32_t img [] = {
  0x00000297,  // auipc t0,0
  0x00028823,  // sb  zero,16(t0)
  0x0102c503,  // lbu a0,16(t0)
  0x00100073,  // ebreak (used as nemu_trap)
  0xdeadbeef,  // some data
};

static void restart() {
  /* Set the initial program counter. */
  cpu.pc = RESET_VECTOR;

  /* The zero register is always 0. */
  cpu.gpr[0] = 0;
}

void init_isa() {
  /* Load built-in image. */
  memcpy(guest_to_host(RESET_VECTOR), img, sizeof(img));

  /* Initialize this virtual computer system. */
  restart();
}
```

首先这里定义了一个结构体 img，这就是模拟出来的img，里面设置了一些 riscv 指令。init_isa 将这个结构体中的数据拷贝到 host 机中，并且设置了 pc 的初始值，以及零寄存器。

### load_img

```c
static long load_img() {
    if (img_file == NULL) {
        Log("No image is given. Use the default build-in image.");
        return 4096;  // built-in image size
    }

    FILE* fp = fopen(img_file, "rb");
    Assert(fp, "Can not open '%s'", img_file);

    fseek(fp, 0, SEEK_END);
    long size = ftell(fp);

    Log("The image is %s, size = %ld", img_file, size);

    fseek(fp, 0, SEEK_SET);
    int ret = fread(guest_to_host(RESET_VECTOR), size, 1, fp);
    assert(ret == 1);

    fclose(fp);
    return size;
}
```

如果没有 img ，那么默认使用上面的 img 结构体，否则直接覆盖掉，这里只从参数载入的 img。

### init_sdb

```c
void init_sdb() {
    /* Compile the regular expressions. */
    init_regex();

    /* Initialize the watchpoint pool. */
    init_wp_pool();
}
```

这是为了表达式求职以及设置监视点做了一点初始化工作。

### 回到 main

初始化工作做完以后，就开始启动了nemu 了。我们可以写一些小程序，来让 nemu 执行。

## 执行一条命令

上面回顾了 nemu 启动 的过程，接下来就可以研究下 nemu 执行一条指令的过程。

### 启动 nenu

```bash
make run
+ CC src/memory/paddr.c
+ LD /home/luyoung/ysyx-workbench/nemu/build/riscv32-nemu-interpreter
make[1]: Entering directory '/home/luyoung/ysyx-workbench'
make[1]: Leaving directory '/home/luyoung/ysyx-workbench'
make[1]: Entering directory '/home/luyoung/ysyx-workbench'
make[1]: Leaving directory '/home/luyoung/ysyx-workbench'
/home/luyoung/ysyx-workbench/nemu/build/riscv32-nemu-interpreter --log=/home/luyoung/ysyx-workbench/nemu/build/nemu-log.txt
[src/utils/log.c:30 init_log] Log is written to /home/luyoung/ysyx-workbench/nemu/build/nemu-log.txt
[src/memory/paddr.c:55 init_mem] physical memory area [0x80000000, 0x87ffffff]
[src/monitor/monitor.c:55 load_img] No image is given. Use the default build-in image.
[src/monitor/monitor.c:28 welcome] Trace: ON
[src/monitor/monitor.c:31 welcome] If trace is enabled, a log file will be generated to record the trace. This may lead to a large log file. If it is not necessary, you can disable it in menuconfig
[src/monitor/monitor.c:34 welcome] Build time: 17:56:50, Aug 13 2024
Welcome to riscv32-NEMU!
For help, type "help"
[src/monitor/monitor.c:38 welcome] Exercise: Please remove me in the source code and compile NEMU again.
(nemu) si
0x80000000: 00 00 02 97 auipc   t0, 0
(nemu) si 2
0x80000004: 00 02 88 23 sb      zero, 16(t0)
0x80000008: 01 02 c5 03 lbu     a0, 16(t0)
(nemu) si
0x8000000c: 00 10 00 73 ebreak
[src/cpu/cpu-exec.c:162 cpu_exec] nemu: HIT GOOD TRAP at pc = 0x8000000c
[src/cpu/cpu-exec.c:119 statistic] host time spent = 10,428 us
[src/cpu/cpu-exec.c:120 statistic] total guest instructions = 4
[src/cpu/cpu-exec.c:122 statistic] simulation frequency = 383 inst/s
(nemu) si
Program execution has ended. To restart the program, exit NEMU and run again.
(nemu) q

```

可以看到，它执行了一些指令之后，就退出了，这些指令很眼熟，研究室 img 结构体中定义的那些指令，为了方便得解决问题，我们就研究这些个指令执行的过程。

menu 中 si 命令可以一次执行多条命令，默认是 1 条命令，我们可以利用 si 来研究此过程。

### si

首先进入到 si：

```c
static int cmd_si(char* args) {
    if (args == NULL)
        cpu_exec(1);
    else
        cpu_exec((uint64_t)(atoi(args)));
    return 0;
}
```

然后进入到 cpu_exec():

```c
/* Simulate how the CPU works. */
void cpu_exec(uint64_t n) {
    g_print_step = (n < MAX_INST_TO_PRINT);
    switch (nemu_state.state) {
        case NEMU_END:
        case NEMU_ABORT:
            printf(
                "Program execution has ended. To restart the program, exit "
                "NEMU and run again.\n");
            return;
        default:
            nemu_state.state = NEMU_RUNNING;
    }

    uint64_t timer_start = get_time();

    execute(n);

    uint64_t timer_end = get_time();
    g_timer += timer_end - timer_start;

    switch (nemu_state.state) {
        case NEMU_RUNNING:
            nemu_state.state = NEMU_STOP;
            break;

        case NEMU_END:
        case NEMU_ABORT:
            Log("nemu: %s at pc = " FMT_WORD,
                (nemu_state.state == NEMU_ABORT
                     ? ANSI_FMT("ABORT", ANSI_FG_RED)
                     : (nemu_state.halt_ret == 0
                            ? ANSI_FMT("HIT GOOD TRAP", ANSI_FG_GREEN)
                            : ANSI_FMT("HIT BAD TRAP", ANSI_FG_RED))),
                nemu_state.halt_pc);
            // fall through
        case NEMU_QUIT:
            statistic();
    }
}
```
结合 gdb，此时cpu 的状态定义为：nemu_state.state = NEMU_RUNNING。

然后进入 execute(n)这个函数，这个函数就开始执行这个指令了：

```c
static void execute(uint64_t n) {
    Decode s;
    for (; n > 0; n--) {
        exec_once(&s, cpu.pc);
        g_nr_guest_inst++;
        trace_and_difftest(&s, cpu.pc);
        if (nemu_state.state != NEMU_RUNNING)
            break;
        IFDEF(CONFIG_DEVICE, device_update());
    }
}
```

n 为 1，传入之后，真正执行命令的是exec_once(&s, cpu.pc):

```c
static void exec_once(Decode* s, vaddr_t pc) {
    s->pc = pc;
    s->snpc = pc;
    isa_exec_once(s);
    cpu.pc = s->dnpc;
#ifdef CONFIG_ITRACE
    char* p = s->logbuf;
    p += snprintf(p, sizeof(s->logbuf), FMT_WORD ":", s->pc);
    int ilen = s->snpc - s->pc;
    int i;
    uint8_t* inst = (uint8_t*)&s->isa.inst.val;
    for (i = ilen - 1; i >= 0; i--) {
        p += snprintf(p, 4, " %02x", inst[i]);
    }
    int ilen_max = MUXDEF(CONFIG_ISA_x86, 8, 4);
    int space_len = ilen_max - ilen;
    if (space_len < 0)
        space_len = 0;
    space_len = space_len * 3 + 1;
    memset(p, ' ', space_len);
    p += space_len;

#ifndef CONFIG_ISA_loongarch32r
    void disassemble(char* str, int size, uint64_t pc, uint8_t* code,
                     int nbyte);
    disassemble(p, s->logbuf + sizeof(s->logbuf) - p,
                MUXDEF(CONFIG_ISA_x86, s->snpc, s->pc),
                (uint8_t*)&s->isa.inst.val, ilen);
#else
    p[0] = '\0';  // the upstream llvm does not support loongarch32r
#endif
#endif
}
```

这里必须说一下，什么是静态 npc，什么是动态 npc。如果一个命令序列不跳转，那么静态npc，或snpc 就是当前指令的下一条指令地址。如果跳转，那么 dnpc 就是下一条真正执行的指令地址。如果不跳转，那么dnpc和 snpc 的值是相同的。当然，对于 cpu，dnpc 是更重要的。

```c
typedef struct Decode {
  vaddr_t pc;
  vaddr_t snpc; // static next pc
  vaddr_t dnpc; // dynamic next pc
  ISADecodeInfo isa;
  IFDEF(CONFIG_ITRACE, char logbuf[128]);
} Decode;
```

前面 exec_once()拿到 img 中第一条指令后，它的 pc 自动为：
```bash
│
execute (n=1) at src/cpu/cpu-exec.c:106
(gdb) n
(gdb) p cpu.pc
$3 = 2147483648
(gdb) p 0x80000000
$4 = 2147483648
(gdb)
```

也就是 0x80000000，也就是第一条指令，没变，可见 pc 的值此时还是第一条指令的地址。传入到exec_once(&s, cpu.pc);中的 pc 也是它，进入函数之后，要设置的值为：

```bash
s.pc = pc #0x80000000
s.snpc = pc #0x80000000
```

接着为isa_exec_once() 顾名思义，这是 isa 执行指令的函数：

```c

int isa_exec_once(Decode *s) {
  s->isa.inst.val = inst_fetch(&s->snpc, 4);
  return decode_exec(s);
}

```

看起来这个函数是取指，然后执行：

```c
static inline uint32_t inst_fetch(vaddr_t *pc, int len) {
  uint32_t inst = vaddr_ifetch(*pc, len);
  (*pc) += len;
  return inst;
}

```

可以看到，取指之后呢，snpc 会更新，也就是 s->snpc += 4，然后返回一个字。

接着执行指令：

```c
static int decode_exec(Decode *s) {
  int rd = 0;
  word_t src1 = 0, src2 = 0, imm = 0;
  s->dnpc = s->snpc;

#define INSTPAT_INST(s) ((s)->isa.inst.val)
#define INSTPAT_MATCH(s, name, type, ... /* execute body */ ) { \
  decode_operand(s, &rd, &src1, &src2, &imm, concat(TYPE_, type)); \
  __VA_ARGS__ ; \
}

  INSTPAT_START();
  INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc  , U, R(rd) = s->pc + imm);
  INSTPAT("??????? ????? ????? 100 ????? 00000 11", lbu    , I, R(rd) = Mr(src1 + imm, 1));
  INSTPAT("??????? ????? ????? 000 ????? 01000 11", sb     , S, Mw(src1 + imm, 1, src2));

  INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak , N, NEMUTRAP(s->pc, R(10))); // R(10) is $a0
  INSTPAT("??????? ????? ????? ??? ????? ????? ??", inv    , N, INV(s->pc));
  INSTPAT_END();

  R(0) = 0; // reset $zero to 0

  return 0;
}
```

这个函数看起来好像在指令解析和指令执行。首先，s的 dnpc = s->snpc，这意味着没有跳转指令，事实上，不管有没有，先给dnpc 赋值为 snpc，要是有跳转指令，后面会更新它。

接着定义了两个宏：
```c
INSTPAT_INST(s) ((s)->isa.inst.val)
#define INSTPAT_MATCH(s, name, type, ... /* execute body */ ({decode_operand(s, &rd, &src1, &src2, &imm, concat(TYPE_, type));__VA_ARGS__ ;}
```

前者前者获取 decode 的 指令字；后者执行指令。

执行完后，回到 exec_once()，这时候更新 cpu.pc = s->dnpc。接着：

1. **指令追踪（如果启用了 `CONFIG_ITRACE`）**：
   - 创建一个日志缓冲区 `p` 并开始格式化输出当前指令地址。
   - 计算指令长度 `ilen` 为 `s->snpc - s->pc`，即指令的字节大小。
   - 循环将指令的每个字节转换为十六进制并添加到日志缓冲区。
   - 根据最大指令长度（对于 x86 是 8 字节，其他可能是 4 字节）计算并添加必要的空格以对齐输出。

2. **反汇编（可选，根据不同的 ISA 和配置）**：
   - 如果不是 `CONFIG_ISA_loongarch32r`，调用 `disassemble` 函数反汇编当前指令，并将反汇编结果添加到日志缓冲区。
   - 如果是 `CONFIG_ISA_loongarch32r`，因为上游 LLVM 不支持此 ISA，仅在日志缓冲区末尾添加空字符。

这个函数的作用非常关键，它不仅执行指令，还能通过日志记录和反汇编帮助开发者理解指令的执行细节。这在调试和教学中尤为重要，可以清晰地展示模拟器在每个步骤中做了什么，以及指令如何影响模拟的处理器状态。


然后回到 cpu_exec()，继续执行：
```c
    uint64_t timer_end = get_time();
    g_timer += timer_end - timer_start;

    switch (nemu_state.state) {
        case NEMU_RUNNING:
            nemu_state.state = NEMU_STOP;
            break;

        case NEMU_END:
        case NEMU_ABORT:
            Log("nemu: %s at pc = " FMT_WORD,
                (nemu_state.state == NEMU_ABORT
                     ? ANSI_FMT("ABORT", ANSI_FG_RED)
                     : (nemu_state.halt_ret == 0
                            ? ANSI_FMT("HIT GOOD TRAP", ANSI_FG_GREEN)
                            : ANSI_FMT("HIT BAD TRAP", ANSI_FG_RED))),
                nemu_state.halt_pc);
            // fall through
        case NEMU_QUIT:
            statistic();
    }
```

在你提供的 `cpu_exec` 函数中，第二个 `switch` 语句负责根据模拟器的状态进行不同的操作，管理模拟器的状态转换并提供反馈和统计。这个 `switch` 语句处理的是模拟器运行后的状态，具体操作如下：

### switch 语句的结构与功能

```c
switch (nemu_state.state) {
    case NEMU_RUNNING:
        nemu_state.state = NEMU_STOP;
        break;

    case NEMU_END:
    case NEMU_ABORT:
        Log("nemu: %s at pc = " FMT_WORD,
            (nemu_state.state == NEMU_ABORT
                 ? ANSI_FMT("ABORT", ANSI_FG_RED)
                 : (nemu_state.halt_ret == 0
                        ? ANSI_FMT("HIT GOOD TRAP", ANSI_FG_GREEN)
                        : ANSI_FMT("HIT BAD TRAP", ANSI_FG_RED))),
            nemu_state.halt_pc);
        // fall through
    case NEMU_QUIT:
        statistic();
        break;
}
```

1. **当状态是 `NEMU_RUNNING`：**
   - 模拟器从运行状态转为停止状态（`NEMU_STOP`）。这通常表示一个正常的停止，例如执行了预定数量的指令后。

2. **当状态是 `NEMU_END` 或 `NEMU_ABORT`：**
   - 输出日志信息，说明程序已经结束或异常终止。
     - 如果是 `NEMU_ABORT`，则打印 "ABORT" 并以红色标注。
     - 如果是 `NEMU_END`，根据 `nemu_state.halt_ret` 的值判断是正常结束还是错误结束：
       - `halt_ret == 0` 表示正常结束（"HIT GOOD TRAP"），用绿色标注。
       - `halt_ret != 0` 表示异常结束（"HIT BAD TRAP"），用红色标注。
   - 输出显示的是模拟器停止时的程序计数器（PC）位置。
   - **通过 "fall through" 继续执行下面的代码块，即不使用 `break` 直接进入下一个 `case`**。

3. **当状态是 `NEMU_QUIT`：**
   - 执行 `statistic()` 函数，这个函数负责收集并打印统计信息，如执行的指令数量、时间统计等。

最后，返回：

```c
int is_exit_status_bad() {
    int good = (nemu_state.state == NEMU_END && nemu_state.halt_ret == 0) ||
               (nemu_state.state == NEMU_QUIT);
    return !good;
}

```
如果 NEMU_END && 返回值为 0，或者手动停止，那么就返回 0，正常退出。

以上就是 nemu 执行一条指令的过程。































