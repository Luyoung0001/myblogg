---
title: "Chisel 速成笔记：30 分钟掌握硬件构建基础"
date: 2024-09-18 13:50:40
tags:
  - "Chisel"
  - "Scala"
  - "digital design"
categories:
  - "Digital Design"
---

<!--more-->

这针对拥有基础 Scala 知识但对 Chisel 完全零基础的入门教程。Chisel（Constructing Hardware In a Scala Embedded Language）是一个基于 Scala 的硬件描述语言，用于设计数字电路。这个教程从环境设置开始，逐步介绍 Chisel 的基本概念和实际应用。

---

## 目录

1. [Chisel 简介](#1-chisel-简介)
2. [环境设置](#2-环境设置)
3. [基本概念](#3-基本概念)
    - [模块（Module）](#模块module)
    - [IO 和 Bundle](#io-和-bundle)
4. [编写第一个 Chisel 模块](#4-编写第一个-chisel-模块)
5. [运行和仿真](#5-运行和仿真)
6. [生成 Verilog 代码](#6-生成-verilog-代码)
7. [示例项目](#7-示例项目)
8. [进一步学习资源](#8-进一步学习资源)

---

## 1. Chisel 简介

**Chisel** 是一个嵌入在 Scala 中的硬件描述语言，旨在简化数字电路的设计过程。它利用了 Scala 的强大功能，如高阶函数、抽象和类型系统，使得硬件设计更加灵活和模块化。

**Chisel 的主要特点：**

- **高度抽象**：利用 Scala 的面向对象和函数式编程特性，实现高层次的硬件描述。
- **可重用性**：模块化设计，便于重用和扩展。
- **强大的生成能力**：能够生成高效的 Verilog 代码，供 FPGA 或 ASIC 实现使用。

---

## 2. 环境设置

要开始使用 Chisel，你需要设置开发环境。以下是基本的步骤：

### 2.1 安装 Java

Chisel 基于 Scala 和 Java，因此需要先安装 Java。

1. **检查是否已安装 Java**：

    ```bash
    java -version
    ```

    如果没有安装，请前往 [Oracle 官方网站](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html) 或使用包管理器安装 OpenJDK。

2. **安装 OpenJDK（以 Ubuntu 为例）**：

    ```bash
    sudo apt update
    sudo apt install openjdk-11-jdk
    ```

### 2.2 安装 Scala 和 sbt

**sbt**（Scala Build Tool）是 Scala 的构建工具，Chisel 项目通常使用 sbt 进行管理。

1. **安装 sbt**：

    - **Ubuntu**：

        ```bash
        echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
        echo "deb https://repo.scala-sbt.org/scalasbt/debian /" | sudo tee /etc/apt/sources.list.d/sbt_old.list
        curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x99E82A75642AC823" | sudo apt-key add
        sudo apt update
        sudo apt install sbt
        ```

    - **macOS（使用 Homebrew）**：

        ```bash
        brew install sbt
        ```

    - **Windows**：

        前往 [sbt 官方下载页面](https://www.scala-sbt.org/download.html) 下载并安装。

### 2.3 创建 Chisel 项目

1. **创建项目目录**：

    ```bash
    mkdir MyChiselProject
    cd MyChiselProject
    ```

2. **初始化 sbt 项目**：

    在项目根目录下创建一个名为 `build.sbt` 的文件，内容如下：

    ```scala
    name := "MyChiselProject"

    version := "0.1"

    scalaVersion := "2.12.15"

    libraryDependencies ++= Seq(
      "edu.berkeley.cs" %% "chisel3" % "3.5.4",
      "edu.berkeley.cs" %% "chiseltest" % "0.5.4" % "test",
      "edu.berkeley.cs" %% "firrtl" % "1.5.4"
    )
    ```

    **说明**：

    - `chisel3`：Chisel 核心库。
    - `chiseltest`：Chisel 测试库，用于编写测试代码。
    - `firrtl`：Chisel 编译工具，负责将 Chisel 代码转换为 Verilog。

3. **创建目录结构**：

    ```bash
    mkdir -p src/main/scala
    mkdir -p src/test/scala
    ```

4. **添加依赖**：

    运行 sbt 命令，下载所需的依赖。

    ```bash
    sbt compile
    ```

    第一次运行时，sbt 会自动下载依赖，这可能需要一些时间。

---

## 3. 基本概念

### 模块（Module）

**模块** 是 Chisel 中的基本构建块，类似于 Verilog 中的模块或 VHDL 中的实体。每个模块可以包含输入输出端口（IO）和内部逻辑。

### IO 和 Bundle

**IO** 定义了模块的输入和输出端口，**Bundle** 是一种将多个信号组合在一起的方式，类似于结构体。

---

## 4. 编写第一个 Chisel 模块

让我们编写一个简单的 **D Flip-Flop** 模块，展示如何定义输入输出和内部逻辑。

### 4.1 创建 FlipFlop.scala

在 `src/main/scala` 目录下创建一个名为 `FlipFlop.scala` 的文件，内容如下：

```scala
import chisel3._

class FlipFlop extends Module {
  val io = IO(new Bundle {
    val d = Input(Bool())  // 输入信号 D
    val q = Output(Bool()) // 输出信号 Q
  })

  // 隐式使用默认时钟
  val reg = RegInit(false.B)  // 定义寄存器，初始值为 false
  reg := io.d                 // 在每个时钟上升沿将 D 的值赋给寄存器
  io.q := reg                 // 输出寄存器的值
}

object FlipFlop extends App {
  (new chisel3.stage.ChiselStage).emitVerilog(new FlipFlop)
}

```

### 4.2 代码解释

- **导入 Chisel 库**：

    ```scala
    import chisel3._
    ```

- **定义模块**：

    ```scala
    class FlipFlop extends Module {
      ...
    }
    ```

    `FlipFlop` 类继承自 `Module`，表示这是一个 Chisel 模块。

- **定义 IO**：

    ```scala
    val io = IO(new Bundle {
      val d = Input(Bool())
      val q = Output(Bool())
    })
    ```

    - `d`：输入信号，布尔类型。
    - `q`：输出信号，布尔类型。

- **定义寄存器和逻辑**：

    ```scala
   // 隐式使用默认时钟
  val reg = RegInit(false.B)  // 定义寄存器，初始值为 false
  reg := io.d                 // 在每个时钟上升沿将 D 的值赋给寄存器
  io.q := reg                 // 输出寄存器的值
    ```

    - 使用 `withClock` 指定模块使用的时钟信号。
    - `RegInit(false.B)` 定义一个初始值为 `false` 的寄存器。
    - 在每个时钟周期，将 `d` 的值赋给寄存器。
    - 将寄存器的值输出到 `q`。

- **生成 Verilog**：

    ```scala
    object FlipFlop extends App {
    (new chisel3.stage.ChiselStage).emitVerilog(new FlipFlop)
    }
    ```

    运行 `FlipFlop` 对象，将生成对应的 Verilog 代码。

### 4.3 生成 Verilog 代码

在项目根目录下运行以下命令：

```bash
sbt "runMain FlipFlop"
```

运行成功后，Verilog 文件将生成在 `./generated/FlipFlop.v` 路径下。

---

## 5. 运行和仿真

Chisel 提供了强大的仿真功能，允许你验证设计的正确性。以下是使用 ChiselTest 进行简单仿真的步骤。

### 5.1 编写测试代码

在 `src/test/scala` 目录下创建一个名为 `FlipFlopTest.scala` 的文件，内容如下：

```scala
import chisel3._
import chiseltest._
import org.scalatest.flatspec.AnyFlatSpec

class FlipFlopTest extends AnyFlatSpec with ChiselScalatestTester {
  "FlipFlop" should "correctly store the input on the rising edge of the clock" in {
    test(new FlipFlop) { c =>
      // 初始状态
      c.io.d.poke(false.B)   // 设置输入 d
      c.clock.step(1)        // 推进时钟
      c.io.q.expect(false.B) // 检查输出 q

      // 设置 D 为 true
      c.io.d.poke(true.B)
      c.clock.step(1)
      c.io.q.expect(true.B)

      // 设置 D 为 false
      c.io.d.poke(false.B)
      c.clock.step(1)
      c.io.q.expect(false.B)
    }
  }
}

```

### 5.2 代码解释

- **导入必要库**：

    ```scala
    import chisel3._
    import chiseltest._
    import org.scalatest.flatspec.AnyFlatSpec
    ```

- **定义测试类**：

    ```scala
    class FlipFlopTest extends AnyFlatSpec with ChiselScalatestTester {
      ...
    }
    ```

    `FlipFlopTest` 类继承自 `AnyFlatSpec` 并混入 `ChiselScalatestTester`，用于编写测试用例。

- **编写测试用例**：

    ```scala
    "FlipFlop" should "correctly store the input on the rising edge of the clock" in {
      test(new FlipFlop) { c =>
        // 初始状态
        c.io.d.poke(false.B)
        c.clock.step(1)
        c.io.q.expect(false.B)

        // 设置 D 为 true
        c.io.d.poke(true.B)
        c.clock.step(1)
        c.io.q.expect(true.B)

        // 设置 D 为 false
        c.io.d.poke(false.B)
        c.clock.step(1)
        c.io.q.expect(false.B)
      }
    }
    ```

    - **`poke`**：设置输入信号的值。
    - **`step`**：推进时钟一个周期。
    - **`expect`**：检查输出信号的值是否符合预期。

### 5.3 运行测试

在项目根目录下运行以下命令：

```bash
sbt test
```

如果一切正常，你将看到测试通过的消息。

---

## 6. 生成 Verilog 代码

Chisel 设计的最终目标通常是生成可用于 FPGA 或 ASIC 实现的 Verilog 代码。以下是生成 Verilog 的基本步骤。

### 6.1 使用 Chisel 工具生成 Verilog

假设你已经编写了一个名为 `FlipFlop` 的 Chisel 模块，并且定义了对应的 `object` 来生成 Verilog 代码。

```scala
object FlipFlop extends App {
  chisel3.Driver.execute(args, () => new FlipFlop)
}
```

### 6.2 生成 Verilog

在项目根目录下运行以下命令：

```bash
sbt "runMain FlipFlop"
```

运行成功后，Verilog 文件将生成在 `./generated/FlipFlop.v` 路径下。

### 6.3 查看生成的 Verilog

打开 `generated/FlipFlop.v`，你将看到类似以下内容：

```verilog
module FlipFlop(
  input  d,
  input  clk,
  output q
);
  reg q_reg;
  always @(posedge clk) begin
    q_reg <= d;
  end
  assign q = q_reg;
endmodule
```

---

## 7. 示例项目

让我们通过一个更复杂的示例，整合前面学到的知识。这里，我们将设计一个简单的 **计数器** 模块，并编写对应的测试。

### 7.1 创建 Counter.scala

在 `src/main/scala` 目录下创建 `Counter.scala`，内容如下：

```scala
import chisel3._
import chisel3.util.log2Ceil // 导入 log2Ceil 函数
class Counter(val max: Int = 10) extends Module {
  val io = IO(new Bundle {
    val count = Output(UInt(log2Ceil(max + 1).W))
    val done = Output(Bool())
  })

  val reg = RegInit(0.U(log2Ceil(max + 1).W))
  when(reg < (max).U) {
    reg := reg + 1.U
  }.otherwise {
    reg := 0.U
  }

  io.count := reg
  io.done := reg === (max).U
}

object Counter extends App {
  (new chisel3.stage.ChiselStage).emitVerilog(new Counter)
}


```

### 7.2 编写测试代码

在 `src/test/scala` 目录下创建 `CounterTest.scala`，内容如下：

```scala
import chisel3._
import chiseltest._
import org.scalatest.flatspec.AnyFlatSpec

class CounterTest extends AnyFlatSpec with ChiselScalatestTester {
  "Counter" should "count from 0 to max and then reset" in {
    test(new Counter(5)) { c =>
      // 初始状态
      c.io.count.expect(0.U)
      c.io.done.expect(false.B)

      // 逐步计数
      for (i <- 1 to 5) {
        c.clock.step(1)
        c.io.count.expect(i.U)
        if (i == 5) {
          c.io.done.expect(true.B)
        } else {
          c.io.done.expect(false.B)
        }
      }

      // 重置后计数
      c.clock.step(1)
      c.io.count.expect(0.U)
      c.io.done.expect(false.B)
    }
  }
}

```

### 7.3 运行测试

在项目根目录下运行：

```bash
sbt test
```

测试应通过，验证计数器的正确性。

### 7.4 生成 Verilog

运行：

```bash
sbt "runMain Counter"
```

生成的 Verilog 文件位于 `./generated/Counter.v`。



