---
title: DPI-C 传递参数的问题
date: 2024-09-25 20:20:40
tags:
    - DPI-C
    - Verilog
categories:
    - PA2
    - 模拟器
---

<!--more-->

## 前言
DPI-C 是 Verilator 提供的一种机制，可以在 Verilog 代码中调用 C/C++ 中定义的 C 语言函数。这为 Verilog 与仿真环境（C/C++）的交互提供了方便。

---

## DPI-C 的基本使用步骤

1. **在 Verilog 中声明 C 函数**

   使用 `import "DPI-C"` 语句在 Verilog 中声明要调用的 C 函数。语法如下：

   ```verilog
   import "DPI-C" function <返回类型> <函数名>(<参数列表>);
   ```

   - `<返回类型>`：函数的返回类型，可以是 `void` 或基本数据类型。
   - `<函数名>`：C 函数的名称。
   - `<参数列表>`：函数的参数，包括 `input`、`output`、`inout`，以及对应的类型。

2. **在 Verilog 中调用 C 函数**

   在需要的位置，直接调用已声明的 C 函数：

   ```verilog
   <函数名>(<参数列表>);
   ```


3. **在 C/C++ 中实现该函数**

   在 C 或 C++ 文件中，按照声明的参数和返回类型，实现对应的函数：

   ```cpp
   extern "C" <返回类型> <函数名>(<参数列表>) {
       // 函数实现
   }
   ```

   - `extern "C"`：防止 C++ 编译器对函数名进行重整（Name Mangling），确保函数名在链接时与 Verilog 中的声明一致。

---

## 示例：`call_c_function` 函数的应用

### 1. 在 Verilog 中声明 `call_c_function`

在文件声明一个名为 `call_c_function` 的 C 函数：

```verilog
module MyBlackBox(
    input  logic [31:0] input1,
    input  logic        clock,
    input  logic        reset,
    output logic [31:0] output1
);
    import "DPI-C" function void call_c_function(input int in, output int out);
endmodule

```

### 2. 在 Verilog 中调用 `call_c_function`

在同一文件中，通过以下方式调用该函数：

```verilog
always_ff @(posedge clock or posedge reset) begin
        if (reset) begin
            output1 <= 0;
        end else begin
            int temp;
            call_c_function(input1, temp);
            output1 <= temp;
        end
    end
```

- **说明**：
  - 在组合逻辑块中，每当输入信号位非 reset 时调用 `call_c_function`。
  - input1、temp 都是参数，其中 temp 类似于 C++ 中的引用 ，在函数中对temp修改，它的值就会改变，这点非常关键，一定要注意。

### 3. 在 C++ 中实现 `call_c_function`

在文件 `main.cpp` 中，实现该函数：

```cpp
extern "C" void call_c_function(uint32_t input1, uint32_t* output1) {
    uint32_t out_val = input1 + 1; // 示例处理
    *output1 = out_val;
    printf("output1 in func:%u\n",out_val);
}
```

- **函数逻辑**：
  - 可以看到，这里的形参和声明这个函数的时候，参数并不一样，这是什么现象呢？后面会讨论；
  - 将 input+1 赋值给 out_val；
  - 再将 output 修改为 outval；
  - 打印一些信息。

### 4. 仿真过程中的调用

在仿真过程中，每次调用 `eval()` 方法评估 Verilog 设计时，如果 `call_c_function` 被触发，就会调用对应的 C++ 函数。这实现了 Verilog 代码对 C 函数的调用，使得硬件仿真能够与软件逻辑进行交互。

---

## 测试

这里需要准备：
`./main.cpp`:

```cpp
#include <stdint.h>
#include <stdio.h>
#include "VMyModule.h"
#include "verilated.h"

extern "C" void call_c_function(uint32_t input1, uint32_t* output1) {
    uint32_t out_val = input1 + 1; // 示例处理
    *output1 = out_val;
    printf("output1 in func:%u\n",out_val);
}

static VMyModule* top = NULL;

void step() {
    top->clock = 0;
    top->eval();
    top->clock = 1;
    top->eval();
}

void reset(int n) {
    top->reset = 1;
    while (n--) {
        step();
    }
    top->reset = 0;
}

int main(int argc, char* argv[]) {
    Verilated::commandArgs(argc, argv);
    top = new VMyModule;

    reset(1);  // 初始化

    // 设置输入信号
    for (int i = 0; i < 10; ++i) {
        top->io_in = i; // 更新输入信号
        step(); // 进行一步仿真
        printf("Input: %u, Output: %u\n", top->io_in, top->io_out);

    }

    delete top; // 释放资源
    return 0;
}

```

`./MyModule.scala`:

```Scala
import chisel3._

// 定义的模块可以直接在里面引入黑盒
// 这个也算是顶层模块

class MyModule extends Module {
    val io = IO(new Bundle {
    val in = Input(UInt(32.W))
    val out = Output(UInt(32.W))
  })

  val blackBox = Module(new MyBlackBox)

  // 在这里引入我们刚包装好的黑盒，连接好引脚即可
  blackBox.io.clock := clock
  blackBox.io.reset := reset
  blackBox.io.input1 := io.in
  io.out := blackBox.io.output1
}

object MyModule extends App {
  (new chisel3.stage.ChiselStage).emitVerilog(new MyModule)
}
```
`./MyBlackBox.scala`:

```Scala
import chisel3._
import chisel3.util._
import chisel3.experimental._

class MyBlackBox extends BlackBox with HasBlackBoxPath {
  val io = IO(new Bundle {
    val clock = Input(Clock())
    val reset = Input(Bool())
    val input1 = Input(UInt(32.W))
    val output1 = Output(UInt(32.W))
  })

}

```

`./aaa/MyBlackBox.sc`:
```Verilog
module MyBlackBox(
    input  logic [31:0] input1,
    input  logic        clock,
    input  logic        reset,
    output logic [31:0] output1
);
    import "DPI-C" function void call_c_function(input int in, output int out);

    always_ff @(posedge clock or posedge reset) begin
        if (reset) begin
            output1 <= 0;
        end else begin
            int temp;
            call_c_function(input1, temp);
            output1 <= temp;
        end
    end
endmodule

```

最后一个就是 `./Makefile` 了：

```Makefile
hdl:
	@sbt clean
	@sbt run

build:
	bear -- verilator --cc MyModule.v --exe --build main.cpp -I/home/luyoung/Test/BlackBox/aaa

clean:
	@rm -rf obj_dir project target

run:
	./obj_dir/VMyModule

.PHONY: hdl build clean

```

直接运行：

```bash
$ make hdl
[info] welcome to sbt 1.10.2 (Private Build Java 1.8.0_422)
[info] loading project definition from /home/luyoung/Test/BlackBox/project
[info] loading settings for project blackbox from build.sbt ...
[info] set current project to blackbox (in build file:/home/luyoung/Test/BlackBox/)
[success] Total time: 0 s, completed Sep 25, 2024 9:28:55 PM
[info] welcome to sbt 1.10.2 (Private Build Java 1.8.0_422)
[info] loading project definition from /home/luyoung/Test/BlackBox/project
[info] loading settings for project blackbox from build.sbt ...
[info] set current project to blackbox (in build file:/home/luyoung/Test/BlackBox/)
[info] compiling 2 Scala sources to /home/luyoung/Test/BlackBox/target/scala-2.12/classes ...
[info] running MyModule
[success] Total time: 6 s, completed Sep 25, 2024 9:29:04 PM

luyoung at luyoung-desktop in ~/Test/BlackBox
$ make build
bear -- verilator --cc MyModule.v --exe --build main.cpp -I/home/luyoung/Test/BlackBox/aaa
make[1]: Entering directory '/home/luyoung/Test/BlackBox/obj_dir'
ccache g++  -I.  -MMD -I/usr/local/share/verilator/include -I/usr/local/share/verilator/include/vltstd -DVM_COVERAGE=0 -DVM_SC=0 -DVM_TRACE=0 -DVM_TRACE_FST=0 -DVM_TRACE_VCD=0 -faligned-new -fcf-protection=none -Wno-bool-operation -Wno-sign-compare -Wno-uninitialized -Wno-unused-but-set-variable -Wno-unused-parameter -Wno-unused-variable -Wno-shadow       -Os -c -o main.o ../main.cpp
ccache g++  -I.  -MMD -I/usr/local/share/verilator/include -I/usr/local/share/verilator/include/vltstd -DVM_COVERAGE=0 -DVM_SC=0 -DVM_TRACE=0 -DVM_TRACE_FST=0 -DVM_TRACE_VCD=0 -faligned-new -fcf-protection=none -Wno-bool-operation -Wno-sign-compare -Wno-uninitialized -Wno-unused-but-set-variable -Wno-unused-parameter -Wno-unused-variable -Wno-shadow       -Os -c -o verilated.o /usr/local/share/verilator/include/verilated.cpp
ccache g++  -I.  -MMD -I/usr/local/share/verilator/include -I/usr/local/share/verilator/include/vltstd -DVM_COVERAGE=0 -DVM_SC=0 -DVM_TRACE=0 -DVM_TRACE_FST=0 -DVM_TRACE_VCD=0 -faligned-new -fcf-protection=none -Wno-bool-operation -Wno-sign-compare -Wno-uninitialized -Wno-unused-but-set-variable -Wno-unused-parameter -Wno-unused-variable -Wno-shadow       -Os -c -o verilated_dpi.o /usr/local/share/verilator/include/verilated_dpi.cpp
ccache g++  -I.  -MMD -I/usr/local/share/verilator/include -I/usr/local/share/verilator/include/vltstd -DVM_COVERAGE=0 -DVM_SC=0 -DVM_TRACE=0 -DVM_TRACE_FST=0 -DVM_TRACE_VCD=0 -faligned-new -fcf-protection=none -Wno-bool-operation -Wno-sign-compare -Wno-uninitialized -Wno-unused-but-set-variable -Wno-unused-parameter -Wno-unused-variable -Wno-shadow       -Os -c -o verilated_threads.o /usr/local/share/verilator/include/verilated_threads.cpp
/usr/bin/python3 /usr/local/share/verilator/bin/verilator_includer -DVL_INCLUDE_OPT=include VMyModule.cpp VMyModule___024root__DepSet_h95824ea7__0.cpp VMyModule___024root__DepSet_hba0a4a14__0.cpp VMyModule__Dpi.cpp VMyModule___024root__Slow.cpp VMyModule___024root__DepSet_h95824ea7__0__Slow.cpp VMyModule___024root__DepSet_hba0a4a14__0__Slow.cpp VMyModule__Syms.cpp > VMyModule__ALL.cpp
ccache g++  -I.  -MMD -I/usr/local/share/verilator/include -I/usr/local/share/verilator/include/vltstd -DVM_COVERAGE=0 -DVM_SC=0 -DVM_TRACE=0 -DVM_TRACE_FST=0 -DVM_TRACE_VCD=0 -faligned-new -fcf-protection=none -Wno-bool-operation -Wno-sign-compare -Wno-uninitialized -Wno-unused-but-set-variable -Wno-unused-parameter -Wno-unused-variable -Wno-shadow       -Os -c -o VMyModule__ALL.o VMyModule__ALL.cpp
echo "" > VMyModule__ALL.verilator_deplist.tmp
Archive ar -rcs VMyModule__ALL.a VMyModule__ALL.o
g++    main.o verilated.o verilated_dpi.o verilated_threads.o VMyModule__ALL.a    -pthread -lpthread -latomic   -o VMyModule
rm VMyModule__ALL.verilator_deplist.tmp
make[1]: Leaving directory '/home/luyoung/Test/BlackBox/obj_dir'

luyoung at luyoung-desktop in ~/Test/BlackBox
$ make run
./obj_dir/VMyModule
output1 in func:1
Input: 0, Output: 1
output1 in func:2
Input: 1, Output: 2
output1 in func:3
Input: 2, Output: 3
output1 in func:4
Input: 3, Output: 4
output1 in func:5
Input: 4, Output: 5
output1 in func:6
Input: 5, Output: 6
output1 in func:7
Input: 6, Output: 7
output1 in func:8
Input: 7, Output: 8
output1 in func:9
Input: 8, Output: 9
output1 in func:10
Input: 9, Output: 10
```

可以看到，顺利得传递了参数，并且类似于引用一样，修改了黑盒子处的值 `temp`，它影响了 `output1`:

```Verilog
import "DPI-C" function void call_c_function(input int in, output int out);

    always_ff @(posedge clock or posedge reset) begin
        if (reset) begin
            output1 <= 0;
        end else begin
            int temp;
            call_c_function(input1, temp);
            output1 <= temp;
        end
    end
```

## 讨论类似于引用一样的传值

这是一件很少见的事情，考虑到这是 C++，我试图用类型隐式转换来解释。

这里的函数声明有什么不对吗：

```Verilog
...
input  logic [31:0] input1,
...

import "DPI-C" function void call_c_function(input int in, output int out);

...
        int temp;
        call_c_function(input1, temp);
...
```

这里都声明为 int 类型，但是调用的时候，很明显往里面传送了一个 32 位的 input 类型。

另外，函数的定义更为奇怪：

```cpp
extern "C" void call_c_function(uint32_t input1, uint32_t* output1) {
    uint32_t out_val = input1 + 1; // 示例处理
    *output1 = out_val;
    printf("output1 in func:%u\n",out_val);
}
```

函数的定义，形参直接变成 `uint32_t`、`uint32_t*` 了。

怎么理解这件事呢？

经过查阅手册，SystemVerilog 和 Verilog 数据类型：

| **数据类型**   | **描述**                           | **位宽**   | **有符号/无符号** | **两态/四态** |
|----------------|------------------------------------|------------|-------------------|----------------|
| `shortint`     | 16 位有符号整数                    | 16 位      | 有符号            | 两态           |
| `int`          | 32 位有符号整数                    | 32 位      | 有符号            | 两态           |
| `longint`      | 64 位有符号整数                    | 64 位      | 有符号            | 两态           |
| `byte`         | 8 位有符号整数或 ASCII 码字符      | 8 位       | 有符号            | 两态           |
| `bit`          | 用户定义的向量尺寸                 | 用户定义   | 无符号            | 两态           |
| `logic`        | 用户定义的向量尺寸                 | 用户定义   | 无符号            | 四态           |
| `reg`          | Verilog-2001 用户定义的向量尺寸    | 用户定义   | 无符号            | 四态           |
| `integer`      | Verilog-2001 32 位有符号整数       | 32 位      | 有符号            | 四态           |
| `time`         | Verilog-2001 64 位无符号整数       | 64 位      | 无符号            | 四态           |


可以看到，在 Verilog 中声明的 `int` 类型是 32 位的，这就很好理解了。

看到的类型不匹配实际上在位宽和内存布局上是兼容的。 Verilog 的 int 和 C 的 uint32_t 在 DPI 传递过程中能够正确映射。使用固定宽度的整数类型（如 uint32_t）是确保跨平台兼容性的良好实践。

也就是说，只要宽度一样，就可以做良好的映射。另外这里的指针操作很奇怪，明明传送的是一个变量，怎么就修改了呢？

在 Verilog 中调用 `call_c_function(input1, temp);` 时，`temp` 是一个普通的 `int` 变量，看起来是按值传递的。但是在 C 函数中，参数是 `uint32_t* output1`，即一个指针，并且在函数中通过这个指针修改了 `output1` 的值。困惑的是，为什么在 Verilog 中传递一个值，C 函数却能通过指针修改它，`temp` 的值也因此被改变。

**这是因为在 SystemVerilog DPI 中，参数的传递方式取决于参数的方向（`input`、`output`）。**

---

**详细解释如下：**

1. **Verilog 中的函数声明和调用：**

   ```verilog
   import "DPI-C" function void call_c_function(input int in, output int out);

   ...

   int temp;
   call_c_function(input1, temp);
   ```

   - `input1`：作为 `input` 参数，按值传递。
   - `temp`：作为 `output` 参数，虽然在代码中看起来是按值传递，但实际上是按引用传递。

2. **参数传递方式：**

   - **`input` 参数：** 按值传递，C 函数**接收的是该参数的副本**，无法修改调用者的变量。
   - **`output` 参数：** 按引用传递，C 函数接收的是**一个指向调用者变量的指针**，可以修改该变量的值。

3. **在 C 函数中的定义：**

   ```cpp
   extern "C" void call_c_function(uint32_t input1, uint32_t* output1) {
       uint32_t out_val = input1 + 1; // 示例处理
       *output1 = out_val;
       printf("output1 in func:%u\n", out_val);
   }
   ```

   - `input1`：接收 `input` 参数的值。
   - `output1`：接收 `output` 参数的指针，允许修改调用者的变量。

4. **为什么 `temp` 的值会被修改：**

   - 在 Verilog 中，传递了变量 `temp`，对应于 `output` 参数。
   - SystemVerilog 编译器在编译时，知道 `out` 是一个 `output` 参数，会自动将 `temp` 的地址传递给 C 函数。
   - 因此，C 函数中的 `uint32_t* output1` 实际上指向 Verilog 中的变量 `temp`。
   - 当 C 函数通过 `*output1 = out_val;` 修改了 `output1` 指向的值，`temp` 的值也随之改变。

5. **编译器的角色：**

   - SystemVerilog 编译器负责参数传递的处理，对于 `output` 和 `inout` 参数，会自动传递变量的地址（指针）。
   - 这使得在 Verilog 代码中调用函数时，**无需显式地取地址**，代码更简洁。

---

**总结：**

- **按值传递 vs 按引用传递：**

  - **按值传递（Value）：** 仅传递变量的副本，函数内部的修改不会影响原变量。
  - **按引用传递（Reference）：** 传递变量的地址，函数内部的修改会影响原变量。

- **在 DPI-C 中，`output` 参数被按引用传递，即使在 Verilog 中看起来像是按值传递。**

- **因此，C 函数可以通过指针修改 Verilog 中的变量。**

---

**示例对照：**

- **Verilog 调用：**

  ```verilog
  int temp;
  call_c_function(input1, temp); // temp 被作为 output 参数，按引用传递
  ```

- **C 函数接收：**

  ```cpp
  void call_c_function(uint32_t input1, uint32_t* output1) {
      // output1 是一个指向 temp 的指针
  }
  ```

- **C 函数内部修改：**

  ```cpp
  *output1 = input1 + 1; // 修改了 temp 的值
  ```

- **Verilog 中的结果：**

  - 调用结束后，`temp` 的值被更新为 `input1 + 1`。

---

**进一步说明：**

- **为什么不需要在 Verilog 中取地址？**

  - 在高级语言中，如 C 或 C++，当需要传递变量的地址时，需要显式地使用取地址符 `&`。
  - 但在 Verilog 中，参数传递方式由参数方向决定，编译器自动处理，无需显式取地址。
  - 这使得代码更简洁，同时避免了与硬件描述语言的风格冲突。

- **如何确认参数传递方式？**

  - 查看函数的声明，关注参数的方向（`input`、`output`）。
  - 在 C 函数中，`input` 参数对应于值，`output` 参数对应于指针。

---


