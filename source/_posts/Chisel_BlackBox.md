---
title: Chisel 中的 BlackBox 机制
date: 2024-09-24 18:20:40
tags:
    - DPI-C
    - Chisel
categories:
    - PA2
    - 模拟器
---

<!--more-->

### 什么是 Chisel 中的 BlackBox？

在硬件设计中，**BlackBox** 是一种用于集成外部硬件模块的方法。在 Chisel 中，BlackBox 允许设计者将已经存在的硬件描述（如 Verilog 或 SystemVerilog 编写的模块）引入到 Chisel 项目中，而无需用 Chisel 重新实现这些模块。这种方法特别适用于以下情况：

1. **重用现有模块**：你可能已经有一些经过验证的 Verilog/SystemVerilog 模块，希望在新的 Chisel 项目中重用它们。
2. **使用第三方 IP 核**：许多硬件设计中会使用第三方提供的知识产权（IP）核，如处理器核心、存储控制器等。
3. **集成特定功能**：某些功能可能需要用特定语言或工具实现，BlackBox 提供了一个桥梁，将这些功能集成到 Chisel 项目中。

### 为什么使用 BlackBox？

Chisel 是一个基于 Scala 的硬件描述语言，它提供了强大的抽象能力和灵活性。然而，在某些情况下，使用 Chisel 重新实现所有模块可能既费时又不实际。BlackBox 允许你在 Chisel 项目中无缝集成其他语言（如 Verilog/SystemVerilog）编写的模块，从而结合了 Chisel 的高层次抽象和传统硬件描述语言的广泛支持。

### 如何在 Chisel 中使用 BlackBox？

以下是使用 BlackBox 的基本步骤：

1. **定义 BlackBox 类**：在 Chisel 中创建一个继承自 `BlackBox` 的类，定义其输入和输出端口。
2. **实例化 BlackBox**：在你的 Chisel 模块中实例化 BlackBox 并连接相应的信号。
3. **生成 Verilog**：使用 Chisel 的工具生成最终的 Verilog 文件，确保外部模块被正确引用。

### 详细步骤解析

#### 0. 定义 BlackBox

这里需要注意的是，这个黑盒子使用 Verilog 实现的，它 将会在里面做一些 Chisel 做不到的事情，比如DPI-C 机制。因此我们要利用这个黑科技，让它在 Chisel 中发挥作用。

假设我们的黑盒子长这个样子（`./aaa/MyBlackBox.sv`）：

```Verilog
module MyBlackBox(
    input  logic        clock,
    input  logic        reset,
    input  logic [31:0] input1,
    output logic [31:0] output1
);
    import "DPI-C" function void call_c_function(
        input  logic [31:0] input1,
        output logic [31:0] output1
    );

    logic [31:0] temp_output1;

    always_ff @(posedge clock or posedge reset) begin
        if (reset) begin
            output1 <= 32'b0;
        end else begin
            call_c_function(input1, temp_output1);
            output1 <= temp_output1;
        end
    end
endmodule


```

这样黑盒子就定义完成了，接下来就是把它接入到 Chisel 中。

#### 1. 定义 BlackBox 类（包装器） (`./MyBlackBox.scala`)

这个类是用来包装黑盒子的，给它包装一下，这样就能在 Chisel 中使用了。注意包装的时候，一定要将这里的引脚名字、类型、数量等严格与被包装的黑盒子相同，不然会在编译（verilater）的时候报错。比如你不能在黑盒子中定义`input1`，又在包装器（Wrapper）中定义`input2`。

这里再插一嘴，这里的这个`包装器`（我起的名字）是一个特殊的类，它扩展于 BlackBox，因此它不需要连线就可以编译到 `hdl` 成功。

```scala
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

- **`extends BlackBox`**：这表示 `MyBlackBox` 是一个黑盒模块，不在 Chisel 代码中实现其具体逻辑。
- **`with HasBlackBoxPath`**：这允许你指定 BlackBox 的文件路径，Chisel 在生成 Verilog 时会知道去哪里找到相应的外部模块。
- **`val io = IO(new Bundle { ... })`**：定义了 BlackBox 的输入和输出端口，这些端口需要与外部模块（如 `MyBlackBox.sv`）的端口相匹配。


#### 2. 实例化 BlackBox (`./MyModule.scala`)

```scala
import chisel3._

class MyModule extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(32.W))
    val out = Output(UInt(32.W))
  })

  // 实例化 BlackBox
  val blackBox = Module(new MyBlackBox)

  // 连接信号
  blackBox.io.clock := clock
  blackBox.io.reset := reset
  blackBox.io.input1 := io.in
  io.out := blackBox.io.output1
}

object MyModule extends App {
  (new chisel3.stage.ChiselStage).emitVerilog(new MyModule)
}
```

- **实例化 BlackBox**：使用 `Module(new MyBlackBox)` 创建一个 BlackBox 实例。
- **连接信号**：将 Chisel 模块的信号连接到 BlackBox 的端口。例如，将顶层模块的 `io.in` 连接到 BlackBox 的 `input1`，并将 BlackBox 的 `output1` 连接到顶层模块的 `io.out`。

#### 3. 生成 Verilog (`MyModule` 对象)

运行 `MyModule` 对象的 `App` 将使用 Chisel 的 `ChiselStage` 生成最终的 Verilog 文件。生成的 Verilog 文件将包含对 `MyBlackBox.sv` 的引用，确保在综合和仿真时能够正确找到并使用外部模块。

```shell
sbt run
```

运行上述命令后，Chisel 会生成包含 `MyModule` 的 Verilog 文件，通常命名为 `MyModule.v`。确保在综合工具或仿真环境中将 `MyBlackBox.sv` 文件包含进去，以便正确解析 BlackBox 模块。


#### 4. 编译(`./main.cpp`)

main.cpp:
```Cpp
#include <stdint.h>
#include <stdio.h>
#include "VMyModule.h"
#include "verilated.h"

extern "C" void call_c_function(uint32_t input1, uint32_t* output1) {
    uint32_t in_val = input1;
    uint32_t out_val = in_val + 1; // 示例处理
    *output1 = out_val;
    printf("hello, this is a test func.\n");
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
        top->io_in = uint32_t(i); // 更新输入信号
        step(); // 进行一步仿真

        // 打印输出结果
        printf("Input: %u, Output: %d\n", top->io_in, top->io_out); // 这里其实在打印地址
    }

    delete top; // 释放资源
    return 0;
}

```

此时就可以编译了，但是编译要注意使用`-I` 将黑盒子连接起来，这里就不使用在包装器中重定路径的方法了，不能用，不知道为什么。

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

直接生成 hdl、编译、运行，就好了。

```bash
make hdl
make build
make run

hello, this is a test func.
Input: 0, Output: -609878311
hello, this is a test func.
Input: 1, Output: -609878311
hello, this is a test func.
Input: 2, Output: -609878311
hello, this is a test func.
Input: 3, Output: -609878311
hello, this is a test func.
Input: 4, Output: -609878311
hello, this is a test func.
Input: 5, Output: -609878311
hello, this is a test func.
Input: 6, Output: -609878311
hello, this is a test func.
Input: 7, Output: -609878311
hello, this is a test func.
Input: 8, Output: -609878311
hello, this is a test func.
Input: 9, Output: -609878311

```

### BlackBox 的使用场景

1. **集成现有模块**：例如，你有一个经过验证的加密模块或通信接口模块，直接在 Chisel 项目中复用它们。
2. **混合语言设计**：某些模块可能用不同的语言编写，如 Verilog、SystemVerilog 或 VHDL，BlackBox 允许这些模块与 Chisel 代码协同工作。
3. **第三方 IP 核**：许多芯片设计中使用第三方提供的 IP 核（如存储控制器、处理器核心），通过 BlackBox 可以轻松集成这些 IP 核。
4. **特定功能实现**：某些功能可能需要用特定的工具或库实现，BlackBox 提供了一个桥梁，将这些功能集成到 Chisel 设计中。



### 总结

以上是一个包装器包装黑盒子的最佳实践，也算是 BlackBox 的入门。



