---
title: "NEMU 代码导读：项目构建、调试入口与核心模块"
date: 2024-08-12 11:22:33
tags:
  - "NEMU"
  - "ysyx"
  - "emulator"
categories:
  - "Computer Architecture"
---

<!--more-->

## 项目构建

首先得了解一些 make 的行为，重要的就是查看make 的行为以及make 的子进程如何与系统进行交互，比如 :
- strace -f make
- make -nB # 只打印命令不执行 B强制构建
- make -d
- make -debug=v

具体RTFM。

- nemu/Makefile
- - SRCS: 和YEMU差不多, 是需要编译的源文件
- - CFLAGS: 刚才看到的编译选项
- - include $(NEMU_HOME)/scripts/native.mk: 包含其他文件
- nemu/scripts/native.mk
- - 一些用于运行和清除编译结果的伪目标
- nemu/scripts/build.mk
- - 编译规则
- - 包含源文件与头文件的依赖关系(由gcc的-MMD选项生成, 并通过fixdep工具处理)

## 代码选讲

### gdb

使用 gdb 来查看程序入口，是一种很好的方法，并且可以根据 gdb 的操作可以看到程序依次执行的各个函数。

```bash
(gdb) b main
Breakpoint 1 at 0x5080: file src/nemu-main.c, line 59.
(gdb)
```

可以使用 GDB 来 RTFSC：

```bash
(gdb) layout src   # layout split还可以看到汇编代码, 但目前不需要
```

一些常用的命令：

- r - 重新开始执行程序
- s - 单步执行一行源代码 / n - 类似但不进入函数(可用于跳过库函数)
- finish - 执行直到当前函数返回
- p - 打印变量或寄存器的值
- x - 扫描内存
- bt - 查看调用栈
- b - 设置断点 / watch - 设置监视点
- help xxx - 查看xxx命令的帮助


### RTFSC

```c

int main(int argc, char* argv[]) {
    /* Initialize the monitor. */
#ifdef CONFIG_TARGET_AM
    am_init_monitor();
#else

    init_monitor(argc, argv);
    // test_expr();

#endif

    /* Start engine. */
    engine_start();

    return is_exit_status_bad();
}


```

首先，执行的是init_monitor(argc, argv):

```c
void init_monitor(int argc, char* argv[]) {
    /* Perform some global initialization. */

    /* Parse arguments. */
    parse_args(argc, argv);

    /* Set random seed. */
    init_rand();

    /* Open the log file. */
    init_log(log_file);

    /* Initialize memory. */
    init_mem();

    /* Initialize devices. */
    IFDEF(CONFIG_DEVICE, init_device());

    /* Perform ISA dependent initialization. */
    init_isa();

    /* Load the image to memory. This will overwrite the built-in image. */
    long img_size = load_img();

    /* Initialize differential testing. */
    init_difftest(diff_so_file, img_size, difftest_port);

    /* Initialize the simple debugger. */
    init_sdb();

#ifndef CONFIG_ISA_loongarch32r
    IFDEF(CONFIG_ITRACE,
          init_disasm(
              MUXDEF(CONFIG_ISA_x86, "i686",
                     MUXDEF(CONFIG_ISA_mips32, "mipsel",
                            MUXDEF(CONFIG_ISA_riscv,
                                   MUXDEF(CONFIG_RV64, "riscv64", "riscv32"),
                                   "bad"))) "-pc-linux-gnu"));
#endif

    /* Display welcome message. */
    welcome();
}
```
通过init_monitor初始化NEMU的大部分功能。

首先是解析参数，nemu 是一个命令行工具，它在启动的时候，可以接受参数。
```c
static int parse_args(int argc, char* argv[]) {
    const struct option table[] = {
        {"batch", no_argument, NULL, 'b'},
        {"log", required_argument, NULL, 'l'},
        {"diff", required_argument, NULL, 'd'},
        {"port", required_argument, NULL, 'p'},
        {"help", no_argument, NULL, 'h'},
        {0, 0, NULL, 0},
    };
    int o;
    while ((o = getopt_long(argc, argv, "-bhl:d:p:", table, NULL)) != -1) {
        switch (o) {
            case 'b':
                sdb_set_batch_mode();
                break;
            case 'p':
                sscanf(optarg, "%d", &difftest_port);
                break;
            case 'l':
                log_file = optarg;
                break;
            case 'd':
                diff_so_file = optarg;
                break;
            case 1:
                img_file = optarg;
                return 0;
            default:
                printf("Usage: %s [OPTION...] IMAGE [args]\n\n", argv[0]);
                printf("\t-b,--batch              run with batch mode\n");
                printf("\t-l,--log=FILE           output log to FILE\n");
                printf(
                    "\t-d,--diff=REF_SO        run DiffTest with reference "
                    "REF_SO\n");
                printf(
                    "\t-p,--port=PORT          run DiffTest with port PORT\n");
                printf("\n");
                exit(0);
        }
    }
    return 0;
}

```
接着是，init_rand()，初始化随机数。
```c
void init_rand() {
  srand(get_time_internal());
}

```

接下来进行日志功能，打开日志文件：

```c
void init_log(const char *log_file) {
  log_fp = stdout;
  if (log_file != NULL) {
    FILE *fp = fopen(log_file, "w");
    Assert(fp, "Can not open '%s'", log_file);
    log_fp = fp;
  }
  Log("Log is written to %s", log_file ? log_file : "stdout");
}

```

接着是 init_mem()，初始化状态机的 M
```c
void init_mem() {
#if   defined(CONFIG_PMEM_MALLOC)
  pmem = malloc(CONFIG_MSIZE);
  assert(pmem);
#endif
  IFDEF(CONFIG_MEM_RANDOM, memset(pmem, rand(), CONFIG_MSIZE));
  Log("physical memory area [" FMT_PADDR ", " FMT_PADDR "]", PMEM_LEFT, PMEM_RIGHT);
}
```

这是将内存初始化为随机数，可以暴露访问初始化内存的未定义行为。

接下来是初始化状态机的状态：init_isa():
```c

void init_isa() {
  /* Load built-in image. */
  memcpy(guest_to_host(RESET_VECTOR), img, sizeof(img));

  /* Initialize this virtual computer system. */
  restart();
}

static void restart() {
  /* Set the initial program counter. */
  cpu.pc = RESET_VECTOR;

  /* The zero register is always 0. */
  cpu.gpr[0] = 0;
}
```
这个过程是将 img 中的一些指令复制到 host 地址中执行。接着设置客户机的 pc 为 0，以及 cpu.gpr[0] = 0。零寄存器很好用，可以方便得让 CPU 产生 0.

接着为load_img，但是参数中并没有给出，因此直接略过：


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

接着就是初始化 sdb 了。

```c
void init_sdb() {
    /* Compile the regular expressions. */
    init_regex();

    /* Initialize the watchpoint pool. */
    init_wp_pool();
}
```

接着就是，welcome()
- 输出欢迎信息
- 以及trace的状态信息
- 还输出了编译的时间和日期


```c
static void welcome() {
    Log("Trace: %s", MUXDEF(CONFIG_TRACE, ANSI_FMT("ON", ANSI_FG_GREEN),
                            ANSI_FMT("OFF", ANSI_FG_RED)));
    IFDEF(CONFIG_TRACE,
          Log("If trace is enabled, a log file will be generated "
              "to record the trace. This may lead to a large log file. "
              "If it is not necessary, you can disable it in menuconfig"));
    Log("Build time: %s, %s", __TIME__, __DATE__);
    printf("Welcome to %s-NEMU!\n",
           ANSI_FMT(str(__GUEST_ISA__), ANSI_FG_YELLOW ANSI_BG_RED));
    printf("For help, type \"help\"\n");
    Log("Exercise: Please remove me in the source code and compile NEMU "
        "again.");
    assert(1);
}

```

接着回到 main，执行engine_start()-->sdb_mainloop():
```c
void sdb_mainloop() {
    if (is_batch_mode) {
        cmd_c(NULL);
        return;
    }

    for (char* str; (str = rl_gets()) != NULL;) {
        char* str_end = str + strlen(str);

        /* extract the first token as the command */
        char* cmd = strtok(str, " ");
        if (cmd == NULL) {
            continue;
        }

        /* treat the remaining string as the arguments,
         * which may need further parsing
         */
        char* args = cmd + strlen(cmd) + 1;
        if (args >= str_end) {
            args = NULL;
        }

#ifdef CONFIG_DEVICE
        extern void sdl_clear_event_queue();
        sdl_clear_event_queue();
#endif

        int i;
        for (i = 0; i < NR_CMD; i++) {
            if (strcmp(cmd, cmd_table[i].name) == 0) {
                if (cmd_table[i].handler(args) < 0) {
                    return;
                }
                break;
            }
        }

        if (i == NR_CMD) {
            printf("Unknown command '%s'\n", cmd);
        }
    }
}

```

这里就是 PA1 的 sdb 了。

### 指令执行

执行指令和 YEMU 中的 inst_cycle 很相似，类似于 CPU 执行一个一个指令。

```c

int isa_exec_once(Decode *s) {
  s->isa.inst.val = inst_fetch(&s->snpc, 4);
  return decode_exec(s);
}
```

其中 decode 解码和执行：
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

以上，就是 nemu 的一个大概的过程，当然要想更全面理解 nemu，需要深入仔细阅读源码。

