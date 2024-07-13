---
title: 搭建verilator仿真环境（双控开关）
date: 2024-07-12 21:57:40
tags:
    - ysyx
    - verilator
    - 双控开关
categories: ysyx
---

<!--more-->

## 一、Verilator
### 什么是 Verilator
Verilator 是一个开源的 Verilog HDL（硬件描述语言）仿真和转换工具，它将 Verilog 代码转换为高性能的 C++ 或 SystemC 代码。Verilator 主要用于对 RTL（寄存器传输级）设计进行仿真，尤其擅长处理大型的复杂数字电路。以下是 Verilator 的一些关键特性和用途：

### 特性
1. **高性能**：
   - Verilator 生成的是编译型的 C++/SystemC 代码，相比其他解释型或事件驱动的仿真器（如 ModelSim 或 VCS）通常能提供更高的仿真效率。

2. **开源**：
   - Verilator 是完全开源的，遵循 Perl Artistic License，可以免费使用和修改。这使得它在学术界和工业界得到广泛的应用和社区支持。

3. **专注于合成后的代码**：
   - 它更多地用于对可合成的 Verilog 代码进行仿真，尤其是那些旨在实际硅片上实现的设计。

4. **广泛的语言支持**：
   - 虽然 Verilator 不完全支持 Verilog 2001 的所有构造，它仍然支持大多数常用的 RTL 设计功能，包括但不限于模块、基本门级逻辑、寄存器、始终边缘触发等。

### 用途
1. **快速仿真**：
   - Verilator 被广泛用于开发周期中的快速仿真，帮助设计师快速验证他们的 Verilog 或 SystemVerilog RTL 设计。

2. **集成测试**：
   - 生成的 C++ 或 SystemC 代码可以很容易地与其他测试框架（如 Google Test）集成，使其非常适合自动化测试和持续集成环境。

3. **学术和教学**：
   - 在学术环境中，Verilator 由于其无成本和开源的特性，常用于教学和研究项目，使学生能够理解和实现复杂的数字逻辑设计。

4. **硬件验证**：
   - 通过与 SystemC 的集成，Verilator 也用于硬件与系统级模型（如处理器、总线和复杂外设）的联合仿真和验证。

说得简单点，Verilator 可以将 Verilog 代码转化为高性能的 C++ 或者 SystemC 代码，然后编译成二进制可执行文件，高效验证 RTL 代码。需要注意的是，SystemC 可不是 C语言，而是用于系统级建模、设计和验证的C++库。

## 二、双控开关
### 程序
双控开关就是用两个输入表示两个开关，一个输出表示一个灯泡的亮灭，Verilog 代码如下：
```verilog
module our_OnOff(
    input a,
    input b,
    output f
);
  assign f = a ^ b;
endmodule
```
有了这个代码之后，就可以利用下面的命令将其转化为 C++代码模型：
```v
verilator -Wall --cc --trace our_OnOff.v
```
这样就在默认目录 obj_dir 下面生成了如下的代码：
```bash
ls -l
total 56
-rw-rw-r-- 1 luyoung luyoung 5278 Jul 11 18:55 Vour_OnOff___024root__DepSet_hd6714a0a__0.cpp
-rw-rw-r-- 1 luyoung luyoung 5566 Jul 11 18:55 Vour_OnOff___024root__DepSet_hd6714a0a__0__Slow.cpp
-rw-rw-r-- 1 luyoung luyoung 1459 Jul 11 18:55 Vour_OnOff___024root__DepSet_he4849518__0.cpp
-rw-rw-r-- 1 luyoung luyoung  893 Jul 11 18:55 Vour_OnOff___024root__DepSet_he4849518__0__Slow.cpp
-rw-rw-r-- 1 luyoung luyoung 1068 Jul 11 18:55 Vour_OnOff___024root.h
-rw-rw-r-- 1 luyoung luyoung  686 Jul 11 18:55 Vour_OnOff___024root__Slow.cpp
-rw-rw-r-- 1 luyoung luyoung 1669 Jul 11 18:55 Vour_OnOff_classes.mk
-rw-rw-r-- 1 luyoung luyoung 3228 Jul 11 18:55 Vour_OnOff.cpp
-rw-rw-r-- 1 luyoung luyoung 2769 Jul 11 18:55 Vour_OnOff.h
-rw-rw-r-- 1 luyoung luyoung 1507 Jul 11 18:55 Vour_OnOff.mk
-rw-rw-r-- 1 luyoung luyoung  786 Jul 11 18:55 Vour_OnOff__Syms.cpp
-rw-rw-r-- 1 luyoung luyoung  975 Jul 11 18:55 Vour_OnOff__Syms.h
```

但是生成了模型之后，我们还要利用它做些别的事，比如使用它生成的接口来测试或者使用，这时候就得再写一个 C/C++程序：
```cpp
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include "./obj_dir/Vour_OnOff.h"
#include "verilated_vcd_c.h"

vluint64_t main_time = 0;  // initial 仿真时间

double sc_time_stamp() {
    return main_time;
}

int main(int argc, char** argv) {
    Verilated::commandArgs(argc, argv);
    Verilated::traceEverOn(true);  //导出vcd波形需要加此语句

    VerilatedVcdC* tfp = new VerilatedVcdC();  //导出vcd波形需要加此语句

    Vour_OnOff* top = new Vour_OnOff("top");
    top->trace(tfp, 0);
    tfp->open("wave.vcd");  //打开vcd

    while (sc_time_stamp() < 20 && !Verilated::gotFinish()) {
        int a = rand() & 1;
        int b = rand() & 1;
        top->a = a;
        top->b = b;
        top->eval();
        printf("a = %d, b = %d, f = %d\n", a, b, top->f);
        tfp->dump(main_time);  // dump wave
        main_time++;           //推动仿真时间
    }
    top->final();
    tfp->close();
    delete top;
    return 0;
}
```

上述代码是一个用于测试 Verilator 生成的 C++ 模型的测试台（testbench）示例。它模拟了 `Vour_OnOff` 模块的操作，同时生成 VCD (Value Change Dump) 波形文件用于后续的波形查看。以下是代码的详细解释：

### 包含头文件和全局变量
- `#include` 语句包括了必要的标准库文件和 Verilator 生成的模块头文件。
- `vluint64_t main_time` 是仿真的全局时间计数器，用于跟踪仿真进度。

### 时间戳函数
- `sc_time_stamp()`：这个函数返回当前的仿真时间。Verilator 在生成 VCD 波形时会调用此函数以获取时间戳。

### 主函数 `main`
- `Verilated::commandArgs(argc, argv);`：处理命令行参数，使得 Verilator 能识别如 `--vcd` 等仿真特定的参数。
- `Verilated::traceEverOn(true);`：开启波形跟踪功能。这对于生成 VCD 文件是必须的。
- `VerilatedVcdC* tfp = new VerilatedVcdC();`：创建一个 VCD 文件生成器对象。
- `top->trace(tfp, 0);`：将 VCD 生成器与顶层模块关联，并设置跟踪的层级。
- `tfp->open("wave.vcd");`：打开一个名为 "wave.vcd" 的文件用于写入波形数据。

### 仿真循环
- `while (sc_time_stamp() < 20 && !Verilated::gotFinish())`。循环直到达到20个时间单位或接收到结束信号。
- `int a = rand() & 1; int b = rand() & 1;`：为模块的输入 `a` 和 `b` 生成随机的二进制值。
- `top->a = a; top->b = b;`：设置顶层模块的输入端口。
- `top->eval();`：评估模块逻辑。这是在每个仿真步骤中调用，用于更新输出。
- `printf("a = %d, b = %d, f = %d\n", a, b, top->f);`：在控制台输出当前的输入和输出状态。
- `tfp->dump(main_time);`：将当前时刻的所有信号状态写入 VCD 文件。
- `main_time++;`：时间前进。

### 清理和结束仿真
- `top->final();`：在仿真结束时调用，执行任何必要的清理操作。
- `tfp->close();`：关闭 VCD 文件。
- `delete top;`：释放顶层模块所占用的内存。

此时，就可以利用上述程序运行和测试这个模型了：
```v
verilator -Wall --cc --exe --build --trace our_OnOff.v main.cpp
```

这里事实上，和上面第一个命令的工作重复了，但是有一个好处，那就是能在写 main.cpp 的时候vscode 不会报语法错误，像下面这样：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/err_code.png)

因此为了在写代码的过程中能有一个良好的体验，还是建议写好 Verilog 代码后首先将其转化为 c++ 代码。

### 仿真

接下来就是编译工作了，很简单，直接一个make 命令就可以了：
```bash
make -C obj_dir -f Vour_OnOff.mk Vour_OnOff
```

编译好后，就可以执行了：
```bash
./obj_dir/Vour_OnOff
a = 1, b = 0, f = 1
a = 1, b = 1, f = 0
a = 1, b = 1, f = 0
a = 0, b = 0, f = 0
a = 1, b = 1, f = 0
a = 0, b = 1, f = 1
a = 0, b = 1, f = 1
a = 1, b = 0, f = 1
a = 0, b = 0, f = 0
a = 0, b = 0, f = 0
a = 1, b = 0, f = 1
a = 1, b = 1, f = 0
a = 0, b = 0, f = 0
a = 0, b = 1, f = 1
a = 1, b = 1, f = 0
a = 1, b = 0, f = 1
a = 0, b = 0, f = 0
a = 1, b = 1, f = 0
a = 1, b = 0, f = 1
a = 1, b = 0, f = 1
```

以上，就是双控开关的仿真。

## 三、总结

### 1. **Verilog 模型转换**
首先，使用 Verilator 将 Verilog 设计 (`our_OnOff.v`) 转换成 C++ 代码。这是通过以下命令完成的：
```bash
verilator -Wall --cc --trace our_OnOff.v
```
这个命令生成了 C++ 代码和一些必要的 makefile 文件 (`Vour_OnOff.mk`) 放在 `obj_dir` 目录下，同时开启了波形跟踪功能。

### 2. **编写测试程序**
为了测试生成的模型，编写了一个 C++ 程序 (`main.cpp`)。这个程序包括了模拟输入和输出的代码，以及生成 VCD 波形文件的功能，帮助验证模型的行为。

### 3. **编译和链接**
使用 Verilator 生成的 makefile (`Vour_OnOff.mk`)，你编译并链接了 C++ 模型和测试程序。这通过以下命令实现：
```bash
make -C obj_dir -f Vour_OnOff.mk Vour_OnOff
```
这个命令确保了所有由 Verilator 生成的 C++ 文件和你的 `main.cpp` 被正确编译和链接。

### 4. **运行仿真**
最后，运行编译后的程序，观察和验证输出。程序在每个仿真周期随机生成输入，根据设计的 Verilog 逻辑计算输出，并将结果打印到控制台。同时，波形数据被记录到 VCD 文件中，方便后续波形分析。

### 5. **环境和工具配置**
我还提到了在 VSCode 中写代码时遇到的问题和解决方案，包括确保 IDE 能够正确识别和索引 Verilator 生成的头文件，以提升代码编写和调试的体验。

### 6. **改进的可能性**
虽然已经使用了一系列命令来处理编译和运行过程，有一点改进的空间是将这些步骤合并，使用更简洁的命令或脚本来自动化整个过程。例如，可以创建一个脚本来一键执行从转换、编译到运行的所有步骤，减少手动输入命令的需要。

通过这个过程，不仅验证了我的 Verilog 设计，还练习了使用 Verilator 工具链、编写和调试 C++ 代码，并通过仿真检验了设计的正确性。这是一个很好的实践，帮助深入理解数字逻辑设计和仿真验证的整个工作流程。