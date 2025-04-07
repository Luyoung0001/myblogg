---
title: LA 挑战赛：加上 difftest
date: 2025-04-05 13:03:13
tags:
    - LoongArch
    - CPU
categories: CPU
---

## difftest

之前做 ysyx 中的处理器设计的时候，使用了 基于 nemu 的 difftest。那个 difftest 的原理是：
- nemu 被编译成一个库，暴露出了一些接口供 CPU 调用；
- 仿真程序将 img 装到各自的内存；
- 当 CPU 执行一步的时候，REF CPU 也执行一步；
- 接着对比两者的所有的 寄存器包括 PC；
- 如果寄存器不同，那么就发出报告信息，程序员可以进行 debug。

这种 difftest 的思路和方法不错，每执行完同一条指令就立即进行对比的。

但是我见过一种更好的 difftest，也就是我使用的 [cdp_ede_local](https://gitee.com/loongson-edu/cdp_ede_local)，它效率更高。它的思路是，有一个 REF CPU，它先跑一遍 test_func 用例。接着 REF 生成 golden_trace.txt，它记录信息很有讲究，它记录了 4 个信息： we、pc、wnum、value。

we 这条指令是否修改通用寄存器，pc 是这条指令的地址，wnum 是修改的寄存器编号，value 是修改的值，下面就是一段 trace：

```txt
1 1c000000 0c ffffffff
1 1c000004 0c ffffffff
1 1c010000 04 bfaff000
1 1c010004 04 bfaff040
1 1c010008 05 bfaff000
1 1c01000c 05 bfaff030
1 1c010010 06 bfaff000
1 1c010014 06 bfaff020
1 1c010018 18 bfaff000
1 1c01001c 18 bfaff050
1 1c010020 0d 00000000
...
```

可以看到，这里的 we 几乎都是 1，这意味着我们事实上只需要 trace 修改寄存器的指令就行了，这种思路很好诠释了 CPU 系统是状态机，它只需要记录每条指令执行后 CPU 状态的改变就行了，看到这里，我已经想到了优化 nemu 的 difftest 的思路（每次只需比较要回写的寄存器就行了，没必要比较所有的寄存器）。

接着，我的 CPU 就可以在执行完后，当执行的指令要回写寄存器的时候，就读取一行 golden_trace.txt，接着作比较就行了：

```C
int difftest() {
    uint32_t we = top->rootp->verilator_top->debug_wb_rf_we;
    uint32_t wnum = top->rootp->verilator_top->debug_wb_rf_wnum;
    uint32_t pc = top->rootp->verilator_top->debug_wb_pc;
    uint32_t value = top->rootp->verilator_top->debug_wb_rf_wdata;
    if (we)
        printf("-->CPU %d %08x %02x %08x\n", we == 15, pc, wnum, value);

    int temp_we = (we == 15 ? 1 : 0);
    if (temp_we == 1) {
        read_ref();
        return wnum == ref_struct.wnum && pc == ref_struct.pc &&
               value == ref_struct.value;
    }
    return -1;
}
```

## 对 cdp_ede_local 项目的改造

是的，我要将 cdp_ede_local 改造成基于 verilator 的仿真框架，这还是挺有挑战性的，但是想到可以更快地进行推进，很值。毕竟 cdp_ede_local 每次difftest 要启动 vivado，时间成本很高。

### ip 核的模拟

原本的框架提供了 ram 的核，我要使用 verilator 的话没办法用它，因此我可以利用 DPI-C 来模拟 data_ram：
```Verilog
module data_ram(
    input clk,
    input we,
    input  [31:0] a,
    input  [31:0] d,
    output [31:0] spo
);
    initial begin
    end

    // dpi-c
    import "DPI-C" function int data_ram_read(input int addr);
    import "DPI-C" function void data_ram_write(input int addr, input int wdata);

    always @(clk) begin
        if (we) begin
            data_ram_write(a, d);
        end
    end

    assign spo = data_ram_read(a);


endmodule
```
inst_ram：
```Verilog
module inst_ram(
    input clk,
    input we,
    input  [31:0] a,
    input  [31:0] d,
    output [31:0] spo
);
    // dpi-c
    import "DPI-C" function int inst_ram_read(input int addr);

        assign spo = inst_ram_read(a);

endmodule

```

接着，实现访存函数：

```C
extern "C" int data_ram_read(int addr) {
    // gpio
    if (addr == SW_INTER_ADDR) {
        return 0x0000aaaa;
    }
    return paddr_read(addr);
}
extern "C" int inst_ram_read(int addr) {
    return paddr_read(addr);
}

extern "C" void data_ram_write(int addr, int wdata) {
    printf("data_write_addr:%08x--->wdata:%08x\n", addr, wdata);
    paddr_write(addr, wdata);
    // 写完验证一下
    uint32_t ans = data_ram_read(addr);
    printf("data_wrote:%08x\n", ans);
}
```

至于底层，分配内存、加载镜像，就可以启动 verilator 了：

```C
void init_system() {
    // 分配内存
    init_mem();
    // 加载镜像
    load_inst();
}
```

### difftest

从 CPU 中拉出四根 debug 信号，verilator_top.v：

```Verilog
    //debug
    wire [31:0] debug_wb_pc/* verilator public */;
    wire [ 3:0] debug_wb_rf_we/* verilator public */;
    wire [ 4:0] debug_wb_rf_wnum/* verilator public */;
    wire [31:0] debug_wb_rf_wdata/* verilator public */;
```

mycpu_top.v：

```Verilog
    // debug info generate
    assign debug_wb_pc       = pc;
    assign debug_wb_rf_we   = {4{rf_we}};
    assign debug_wb_rf_wnum  = dest;
    assign debug_wb_rf_wdata = final_result;
```

这样，就可以了：

```C
// 读取 trace，与拉到的信号进行比对
int difftest() {
    uint32_t we = top->rootp->verilator_top->debug_wb_rf_we;
    uint32_t wnum = top->rootp->verilator_top->debug_wb_rf_wnum;
    uint32_t pc = top->rootp->verilator_top->debug_wb_pc;
    uint32_t value = top->rootp->verilator_top->debug_wb_rf_wdata;
    if (we)
        printf("-->CPU %d %08x %02x %08x\n", we == 15, pc, wnum, value);

    int temp_we = (we == 15 ? 1 : 0);
    if (temp_we == 1) {
        read_ref();
        return wnum == ref_struct.wnum && pc == ref_struct.pc &&
               value == ref_struct.value;
    }
    return -1;
}

```

## 运行效果

接下来就是 debug 了，cdp_ede_local 留了很多坑，原本是让读者基于 vivado 的 difftest 进行 debug 的，但是有了 verilator，很快就找到了所有的 bug：

```C
void cpu_exec(uint64_t n) {
    while (n) {
        // 如果为 0 说明不一致，应该打印一些信息
        if (difftest() == 0) {
            printf("Error!\n");
            break;
        }
        // 停机信号
        uint32_t pc = top->rootp->verilator_top->__PVT__cpu__DOT__pc;
        uint32_t stop_pc = 0x1c000100;
        if (pc == stop_pc) {
            break;
        }
        stepi();
        n--;
    }
    top->final();
    tfp->close();
}
```

如果执行到 0x1c000100，说明 EXP=6 已经通过了测试，这 20 条指令的 bug 已经修复完成：

```bash
make clean && make run
...
data_write_addr:bfaff040--->wdata:00000001
data_wrote:00000001
data_write_addr:bfaff040--->wdata:00000001
data_wrote:00000001
i:11990
data_write_addr:bfaff030--->wdata:00000001
data_wrote:00000001
data_write_addr:bfaff030--->wdata:00000001
data_wrote:00000001
i:11991
-->CPU 1 1c010150 04 00000000
-->REF 1 1c010150 04 00000000
i:11992
-->CPU 1 1c010154 01 1c010158
-->REF 1 1c010154 01 1c010158
i:11993
-->CPU 1 1c000100 0c bfaff091
-->REF 1 1c000100 0c bfaff091

```

这样，就完成了改造，虽然经过艰难险阻，但结果是成功的。









