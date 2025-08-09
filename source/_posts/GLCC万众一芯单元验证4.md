---
title: GLCC万众一芯单元验证（四）
date: 2025-08-07 13:03:13
tags: 万众一芯
categories: 体系结构
---

## 前言
个人感觉学习 Toffee 的话，还是看 Toffee 的 [官方文档](https://pytoffee.readthedocs.io/zh-cn/latest/index.html) 比较好。本文将会就我本人的理解，写一点需要重点理解的东西。

## 异步环境

异步是符合真实的硬件运行逻辑的，因为芯片中的电路在每一个时钟周期后，所有的引脚信号都会更新。

```python
import toffee
from toffee.triggers import *

async my_coro(dut):
    await ClockCycles(dut, 10)
    print("10 cycles passed")

async def start_test():
    dut = MyDUT()
    toffee.start_clock(dut)

    await my_coro(dut)

toffee.run(start_test)
```

在 toffee 框架中，通过 toffee.run() 来启动一个事件循环，其中的参数为事件循环 start_test。而事件循环是由各个异步事件组成的，事件循环用于管理多个同时运行的协程，协程之间可以相互等待并通过事件循环来进行切换。start_test() 是一个异步时钟（协程任务），它和 my_coro(dut) 是两个协程。


## Bundle 与 DUT 的绑定

一共有以下几种方法进行绑定，这一步很关键，一定要确保它们绑定成功，不然下层出问题，上层很难发现。

### 直接绑定

例如有一个简单的加法器 DUT，其接口名称与 Bundle 中定义的名称相同，就可以直接使用默认构造方法实例化 Bundle：

```python
adder = DUTAdder()

adder_bundle = AdderBundle()
adder_bundle.bind(adder)
```
这样要求 DUT 的接口和 Bundle 中接口名称要完全一致，这不是我们想要的结果，因为 DUT 的写法如果很复杂，Bundle 中也必须定义这么复杂。


### 通过字典进行绑定

使用 Bundle 提供的 from_dict 方法实例化 Bundle：

```python
adder = DUTAdder()
adder_bundle = AdderBundle.from_dict({
    'a': 'a_in',
    'b': 'b_in',
    'sum': 'sum_out',
    'cin': 'cin_in',
    'cout': 'cout_out'
})
adder_bundle.bind(adder)
```

这种优点是简单粗暴，缺点是书写麻烦。

### 通过前缀进行绑定

假设 Bundle 中的接口名称与 DUT 中的接口名称拥有如下对应关系：
```text
a    -> io_a
b    -> io_b
sum  -> io_sum
cin  -> io_cin
cout -> io_cout
```

使用 Bundle 提供的 from_prefix 方法实例化 Bundle：

```python
adder = DUTAdder()
adder_bundle = AdderBundle.from_prefix('io_')
adder_bundle.bind(adder)
```

### 通过正则表达式进行绑定
假设 Bundle 中的接口名称与 DUT 中的接口名称之间的对应关系为：

```text
a    -> io_a_in
b    -> io_b_in
sum  -> io_sum_out
cin  -> io_cin_in
cout -> io_cout_out
```

这是最好的方法，管什么妖魔鬼怪似的 DUT 接口名称，可以使用 from_regex 方法实例化 Bundle

```python
adder = DUTAdder()
adder_bundle = AdderBundle.from_regex(r'io_(.*)_.*')
adder_bundle.bind(adder)
```

对于上面代码中的正则表达式，io_a_in 会与正则表达式成功匹配，唯一的 **捕获组** 捕获到的内容为 a。a 这个名称与 Bundle 中的接口名称 a 匹配，因此 io_a_in 会被正确绑定至 a。

### 绑定子 Bundle

举一个例子：
```Verilog
module ArithmeticModule (
    input [31:0] io_add_a,    // for 'add_' prefix matching adder inputs
    input [31:0] io_add_b,
    input io_add_cin,
    output [31:0] io_add_sum,
    output io_add_cout,

    input [31:0] io_mul_a,    // for 'mul_' prefix matching multiplier inputs
    input [31:0] io_mul_b,
    output [31:0] io_mul_product,

    input io_selector        // selector for selecting between adder and multiplier
);
```

有时候，DUT 接口很复杂，我们在划分功能点的时候，需要将整个 IO 接口划分多个 Bundle，以上面的 DUT 为例，它支持加法和乘法，就得划分为三部分：
```python
from toffee import Bundle, Signal, Signals

class AdderBundle(Bundle):
    a, b, sum, cin, cout = Signals(5)

class MultiplierBundle(Bundle):
    a, b, product = Signals(3)

class ArithmeticBundle(Bundle):
    selector = Signal()

    adder = AdderBundle.from_prefix('add_')
    multiplier = MultiplierBundle.from_prefix('mul_')
```

在上面的代码中，我们定义了一个 ArithmeticBundle，它包含了自己的信号 selector。除此之外它还包含了一个 AdderBundle 和一个 MultiplierBundle，这两个子 Bundle 分别被命名为 adder 和 multiplier。

当我们需要访问 ArithmeticBundle 中的子 Bundle 时，可以通过 . 运算符来访问：
```python
arithmetic_bundle = ArithmeticBundle()

arithmetic_bundle.selector.value = 1
arithmetic_bundle.adder.a.value = 1
arithmetic_bundle.adder.b.value = 2
arithmetic_bundle.multiplier.a.value = 3
arithmetic_bundle.multiplier.b.value = 4
```

需要注意的是，子 Bundle 的创建方法去匹配的信号名称，是经过上一次 Bundle 的创建方法进行处理过后的名称。例如在上面的代码中，我们将顶层 Bundle 的匹配方式设置为 from_prefix('io_')，那么在 AdderBundle 中去匹配的信号，是去除了 io_ 前缀后的名称。

为什么要设置子 Bundle 呢？通过子 Bundle，可以实现系统的层次化设计。这有助于分解复杂的系统，使得每个模块都能独立工作，同时保证它们之间的接口清晰且易于管理。比如，我们通过设置不同的 Agent 来测试这个模块的不同功能，还可以在系统测试的时候，将所有的 Bundle 之上的 Agent 结合起来进行系统测试。

## 编写 Agent

Agent 在 toffee 验证环境中实现了对一类 Bundle 中信号的高层封装，使得上层驱动代码可以在不关心具体信号赋值的情况下，完成对 Bundle 中信号的驱动及监测。

我的想法是，所有的驱动方法全部由 Agent 提供，Bundle 只是 DUT 单纯的 IO 信号划分。换句话说，Bundle 是第一次对 DUT 的封装，Agent 是在 Bundle 之上的第二次基本功能的第二次封装。

一个 Agent 由 驱动方法(driver_method) 和 监测方法(monitor_method) 组成，其中驱动方法用于主动驱动 Bundle 中的信号，而监测方法用于被动监测 Bundle 中的信号。

我的理解是，一个 Agent 最好只涉及到一个功能，而这个功能可能涉及多个 Bundle，但是最好只涉及到一个 Bundle，这样能解耦的更彻底。

比如，DUTArithmeticModule 划分了 3 个 Bundle，但是只有两个功能，分别是乘法和加法。但是要做几个 Agent 呢？答案是三个：

- 一个加法
- 一个乘法
- 一个结合加法、乘法以及 selector

每一个 Agent 中都要写驱动方法和监测方法，这些方法可能有多个。

比如对于 AdderBundle，我们可以在它之上设计 AdderAgent：

```python
from toffee.agent import *

class AdderAgent(Agent):
    @driver_method()
    async def exec_add(self, a, b, cin):
        self.bundle.a.value = a
        self.bundle.b.value = b
        self.bundle.cin.value = cin
        await self.bundle.step()
        return self.bundle.sum.value, self.bundle.cout.value

    @monitor_method()
    async def monitor_sum(self):
        if self.bundle.sum.value > 0:
            return self.bundle.as_dict()

```

这样一个 Agent 就做好了。

## 搭建 Env
有了 Agent，似乎还不够，因为测试环境应该是一个整体，它应该包含定义的功能 Agents 以及参考模型，然后还需要我们将 Agents 中的观测方法获取来的数据和参考模型做匹配，这样就又有了覆盖组观测点的概念。总之，Env 是最后一层，它是一个有机体。

### 创建 Env

为了定义一个 Env，需要自定义一个新类，并继承 toffee 中的 Env 类。下面是一个简单的 Env 的定义示例：

```python
from toffee.env import *

class DualPortStackEnv(Env):
    def __init__(self, port1_bundle, port2_bundle):
        super().__init__()

        self.port1_agent = StackAgent(port1_bundle)
        self.port2_agent = StackAgent(port2_bundle)
```

这个 Env 中定义了两个 Agent，它们由传进来的两个 bundle 实例化：

```text
DualPortStackEnv
  - port1_agent
    - @driver_method push
    - @driver_method pop
    - @monitor_method some_monitor
  - port2_agent
    - @driver_method push
    - @driver_method pop
    - @monitor_method some_monitor
```

这样，Env 中就有了基本的 Agent。接下来就要继续添加参考模型了。

### 添加参考模型

参考模型也可以通过这些接口来进行编写，编写好的参考模型都可以使用 attach 操作直接附加到 Env 上，由 Env 来完成参考模型的自动同步，方式如下：
```python
env = DualPortStackEnv(port1_bundle, port2_bundle)
env.attach(StackRefModel())
```
一个 Env 可以附加多个参考模型，这些参考模型都将会被 Env 自动同步。

## 如何编写参考模型

参考模型很重要，没有参考模型就无法对 Agent 乃至 DUT 进行测试。

### 函数调用模式

这我认为是最简单的模式了，直接编写一个和 Agent 能绑定的函数就行，比如：

```python
class AdderRefModel:
    def add(self, a, b):
        return a + b
```

### 独立执行流模式

独立执行流模式就有点复杂了，它会主动获取 Agent 中的驱动参数以及观测参数，并主动就行对比：

```python
class AdderRefModel(Model):
    def __init__(self):
        super().__init__()

        self.add_port = DriverPort(agent_name="add_agent", driver_name="add")
        self.sum_port = MonitorPort(agent_name="add_agent", monitor_name="sum")

    async def main():
        while True:
            operands = await self.add_port()
            std_sum = operands["a"] + operands["b"]
            dut_sum = await self.sum_port()
            assert std_sum == dut_sum, f"Expected {std_sum}, but got {dut_sum}"

```

可以看到，它有两个成员 add_port、sum_port，定义了这两个接口后，上层代码在给参考模型发送数据时，并不会触发参考模型中的某个函数，而是会将数据发送到 add_port 这个驱动接口中。同时，DUT的输出数据也会被发送到 sum_port 这个监测接口中。

那么参考模型如何去使用这两个接口呢？在参考模型中，有一个 main 函数，这是参考模型执行的入口，当参考模型创建时, main 函数会被自动调用，并在后台持续运行。在上面代码中 main 函数里，参考模型通过不断重复这一过程：等待 add_port 中的数据、计算结果、获取 sum_port 中的数据、比较结果，来完成参考模型的验证工作。

参考模型会主动向 add_port 请求数据，如果 add_port 中没有数据，参考模型会等待数据的到来。当数据到来后，参考模型将会进行计算，之后参考模型再次主动等待 sum_port 中的数据到来。它的执行过程是一个独立的执行流，不受外部的主动调用控制。当参考模型变得复杂时，其将会含有众多的驱动接口和监测接口，通过独立执行流的方式，可以更好的去处理结构之间的相互关系，尤其是接口之间存在调用顺序的情况。

## 如何编写函数调用模式的参考模型

### 驱动函数匹配

假如 Env 中定义的接口如下：
```python
StackEnv
  - port_agent
    - @driver_method push
    - @driver_method pop
    - @monitor_method monitor_pop_data
```
那么如果我们想要编写与之对应的参考模型，自然地，我们需要定义这四个驱动函数被调用时参考模型的行为。也就是说为每一个驱动函数编写一个对应的函数，这些函数将会在驱动函数被调用时被框架自动调用。

如何让参考模型中定义的函数能够与某个驱动函数匹配呢？首先应该使用 @driver_hook 装饰器来表示这个函数是一个驱动函数的匹配函数。接着，为了建立对应关系，我们需要在装饰器中指定其对应的 Agent 和驱动函数的名称。最后，只需要保证函数的参数与驱动函数的参数一致，两个函数便能够建立对应关系。

class StackRefModel(Model):
    @driver_hook(agent_name="port_agent")
    def push(self, data):
        pass

    @driver_hook(agent_name="port_agent")
    def pop(self):
        pass

此时，驱动函数与参考模型的对应关系已经建立，当 Env 中的某个驱动函数被调用时，参考模型中对应的函数将会被**自动调用**，并自动对比两者的返回值是否一致。

### 监测函数匹配
Toffee 目前支持检测函数的匹配，通过 @monitor_hook 装饰器来表示这个函数是一个监测函数的匹配函数。与 @driver_hook 类似，为了建立对应关系，需要在装饰器中指定其对应的 Agent 和监测函数的名称，例如：
```python
class StackRefModel(Model):
    @monitor_hook(agent_name="port_agent")
    def monitor_pop_data(self, item):
        pass
```
monitor_hook 含有一个固定的额外参数，例如上面代码中的 item，用于接收监测函数的返回值。当 Env 中的监测函数被调用时，参考模型中对应的 monitor_hook 函数将会被自动调用，在函数体的实现中可以判断监测函数的返回值是否符合预期。

monitor_hook 支持 driver_method 所支持的所有匹配方式。

### Agent 匹配

如果在参考模型中，一个一个写，看起来会有点啰嗦，因此这里把整个 Agent 作为一个单元进行匹配，而不是 Agent 中的各个子函数。

Toffee 还提供了 agent_hook，用于一次性匹配多个驱动函数或监测函数，方式如下：
```python
class StackRefModel(Model):
    @agent_hook("port_agent")
    def port_agent(self, name, item):
        pass
```

在这个例子中，port_agent 函数将会匹配 port_agent Agent 中的所有驱动函数与监测函数。当 Agent 中的任意一个驱动函数被调用时，port_agent 都会被自动调用，并将驱动函数的名称与参数传入。当 Agent 中的任意一个监测函数被调用时，port_agent 也会被自动调用，并将监测函数的名称与返回值传入。此外，如果 agent_hook 有返回值，框架会使用此函数的返回值与驱动函数的返回值进行对比。

与驱动函数类似，@agent_hook 装饰器也支持当函数名与 Agent 名称相同时省略 agent_name 参数。
```python
class StackRefModel(Model):
    @agent_hook()
    def port_agent(self, driver_name, args):
        pass

```
如果需要同时匹配多个 Agent，可以使用 agent_hook 中的 agents 参数，例如：
```python
class StackRefModel(Model):
    @agent_hook(agents=["port_agent", "port_agent2"])
    def port_agent(self, driver_name, args):
        pass

```
如果需要同时匹配多个驱动函数或监测函数，可以使用 agent_hook 中的 methods 参数，并指定需要匹配的驱动函数或监测函数的路径，例如：

```python
class StackRefModel(Model):
    @agent_hook(methods=["port_agent.push", "port_agent.pop", "port_agent2.monitor_pop_data"])
    def port_agent(self, driver_name, args):
        pass

```

## 测试用例

测试用例就是调用 Agents 中的方法，来驱动 DUT，并自动调用和对比参考模型对应的方法，达到验证的作用。

当验证环境搭建完成后，编写测试用例用于验证设计的功能是否符合预期。对于硬件验证中的验证，两个重要的导向是：功能覆盖率、行覆盖率，功能覆盖率意味着测试用例是否覆盖了设计的所有功能，行覆盖率意味着测试用例是否触发了设计的所有代码行。

### 同时调用不同的驱动函数

为什么要这样？有的 Agent 可能有这样的需求，比如一个 AXI 接口，可能会同时发起读事务和写事务，这时候就得测试事务同时进行的场景。考虑这样一个Env:

```text
DualPortStackEnv
  - port1_agent
    - @driver_method push
    - @driver_method pop
  - port2_agent
    - @driver_method push
    - @driver_method pop
```

我们期望在测试用例中同时调用 port1_agent 和 port2_agent 的 push 函数，以便同时驱动两个接口，就可以这样做：将两个不同的 Agent 中的方法包装成一个执行块：

```python
from toffee import Executor

def test_push(env):
    async with Executor() as exec:
        exec(env.port1_agent.push(1))
        exec(env.port2_agent.push(2))

    print("result", exec.get_results())
```
我们使用 async with 来创建一个 Executor 对象，并建立一个执行块，通过直接调用 exec 可以添加需要执行的驱动函数。当 Executor 对象退出作用域时，会将所有添加的驱动函数同时执行。Executor 会自动等待所有驱动函数执行完毕。

### 同一驱动函数被多次调用

```python
from toffee import Executor

def test_push(env):
    async with Executor() as exec:
        for i in range(5):
            exec(env.port1_agent.push(1))
        exec(env.port2_agent.push(2))

    print("result", exec.get_results())
```

这个也很好理解。

## 功能检查点

当我们把 Env 设计好后，并写好了测试用例后，我就可以评估整个测试过程了。如何定量分析测试用例的完备与否呢（是否遗漏了某些情况）？

在 toffee 中，功能检查点(Cover Point) 是指对设计的某个功能进行验证的最小单元，判断该功能是否满足设计目标。测试组(Cover Croup) 是一类检查点的集合。

### 编写检查点

首先得创建一个测试组，并把测试点添加到测试组中，这里面有几个概念：

```python
import toffee.funcov as fc

g = fc.CovGroup("Group-A")

g.add_watch_point(adder.io_cout,
                  {"io_cout is 0": fc.Eq(0)},
                  name="cover_point_1")
```

g 是一个测试组，cover_point_1 是 g 的一个检查点的名称，adder.io_cout 则是检查目标。针对这个检查点，可以看一些条件，如果这些条件在测试案例测试的时候，都达到了，说明这个检查目标已经被检查了。这些条件就被称为检查条件，也被称为 Cover Bin。

```python
def add_watch_point(target,
                    bins: dict,
                    name: str = "", once=None):
        """
        @param target: 检查目标，可以是一个引脚，也可以是一个DUT对象
        @param bins: 检查条件，dict格式，key为条件名称，value为具体检查方法或者检查方法的数组。
        @param name: 检查点名称
        @param once，如果once=True，表明只检查一次，一旦该检查点满足要求后就不再进行重复条件判断。
```

当添加完所有的检查点后，需要在 DUT 的 Step 回调函数中调用 CovGroup 的 sample() 方法进行判断。在检查过程中，或者测试运行完后，可以通过 CovGroup 的 as_dict() 方法查看检查情况。




















