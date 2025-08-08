---
title: GLCC万众一芯单元验证（三）
date: 2025-08-06 13:03:13
tags: 万众一芯
categories: 体系结构
---

## 前言

本来应该做收集功能覆盖率，但是感觉难度有点高，还是先做一下 Toffee 的官方的 example，快速熟悉一下。

## Adder 模块

这个例子是一个加法器，它是一个组合电路：

```Verilog
module Adder #(
    parameter WIDTH = 64
) (
    input  [WIDTH-1:0] io_a,
    input  [WIDTH-1:0] io_b,
    input              io_cin,
    output [WIDTH-1:0] io_sum,
    output             io_cout
);

assign {io_cout, io_sum}  = io_a + io_b + io_cin;

endmodule
```

## 设计 Bundle

```python
class AdderBundle(Bundle):
    a, b, cin, sum, cout = Signals(5)
```

这个 bundle 设计得非常简单，因为 Adder 很简单。定义完成后，就可以通过 AdderBundle 类的实例来访问这些信号，例如：

```python
adder_bundle = AdderBundle()

adder_bundle.a.value = 1
adder_bundle.b.value = 2
adder_bundle.cin.value = 0
print(adder_bundle.sum.value)
print(adder_bundle.cout.value)
```

上面的 bundle 看似进行了某些操作，实际上这个目前和 DUT 还是分离的，要想通过 bundle 真正操作 DUT 的引脚，必须进行引脚绑定。

```python
# 实例化 DUT
adder = DUTAdder()
# 实例化 bundle
adder_bundle = AdderBundle()
# 绑定 bundle 和 DUT
adder_bundle.bind(adder)
```

但是，如果 DUT 的接口名称与 Bundle 中定义的名称不同，直接使用 bind 则无法正确绑定。在 Bundle 中，我们提供多种绑定方法，以适应不同的绑定需求。

这里不再赘述。

## Agent

使用 Agent 来编写对该接口的驱动方法，事实上，这里有一个问题：将基本操作放在 Bundle 层还是 Agent 层？

Bundle 层：负责实现底层的信号操作逻辑，例如 enqueue 和 dequeue。Bundle 通常是与硬件信号直接交互的层，专注于信号的定义和基本操作。因此，将信号操作的具体实现放在 Bundle 层，可以清晰地分离职责，使得 Bundle 层只负责底层的信号管理。

Agent 层：负责更高层次的操作，通常包含测试或仿真控制逻辑。Agent 是与外部接口交互的层，通常会通过 @driver_method 提供一些方法来驱动和监控信号。Agent 层将 Bundle 层的操作包装成方便调用的接口，并实现控制和调度逻辑。

这里需要提前认识这一点，但对于 Adder 这个例子来说，放到任何一层都差不多。

因此，这里就爱那个方法放在 Agent 层：

```python
# 创建 Agent
# 继续对 bundle 进行封装，使其变成更抽象的驱动方法和观测方法
class AdderAgent(Agent):
    @driver_method()
    async def exec_add(self, a, b, cin):
        self.bundle.a.value = a
        self.bundle.b.value = b
        self.bundle.cin.value = cin
        await self.bundle.step()
        return self.bundle.sum.value, self.bundle.cout.value

    @monitor_method()
    async def monitor_once(self):
        return self.bundle.as_dict()
```

## 打包验证环境

它的主要职责包括：

- 实例化组件：在 Env 内部实例化验证环境中所需的所有 Agent 组件。
- 管理接口连接：负责确保每个 Agent 都获得了正确的 Bundle 接口实例，Bundle 通常代表了与待验证设计（DUT）交互的物理或逻辑接口。
- 定义参考模型规范：Env 的结构（即其包含的 Agent 及其方法）隐式地定义了参考模型（Reference Model）需要遵循的接口规范。
- 集成与同步参考模型：对于遵循规范编写的参考模型，Env 提供了附加（attach）机制，并负责在运行时自动将测试激励和监测数据同步给这些模型。

### 创建 Env

对于 Adder，它需要一个 Bundle 对象参数，正如前面说的，它的职责就是实例化 Agent，它需要 Bundle 对象来实例化 Agent.

然后使用 attach 方法集成合同步参考模型，AdderEnv 创建完成后，整个验证环境的结构也随之确定。

```python
from .agent import AdderAgent
from .agent import AdderBundle
from .refmodel import AdderModelWithDriverHook
from .refmodel import AdderModelWithMonitorHook
from .refmodel import AdderModelWithPort
from toffee import *


class AdderEnv(Env):
    def __init__(self, adder_bundle):
        super().__init__()
        self.add_agent = AdderAgent(adder_bundle)

        self.attach(AdderModelWithDriverHook())
        self.attach(AdderModelWithMonitorHook())
        self.attach(AdderModelWithPort())
```

### 定义参考模型

参考模型用于模拟待验证设计的行为，以便在验证过程中对设计进行验证，有点类似于 difftest。在 toffee 验证环境中，参考模型需要遵循 Env 的接口规范，以便能够附加到 Env 上，由 Env 来完成参考模型的自动同步。

#### 函数调用模式

函数调用模式即是将参考模型的对外接口定义为一系列的函数，通过调用这些函数来驱动参考模型的行为。此时，我们通过输入参数向参考模型发送数据，并通过返回值获取参考模型的输出数据，参考模型通过函数体的逻辑来更新内部状态。

#### 独立执行流模式

独立执行流模式即是将参考模型的行为定义为一个独立的执行流，它不再受外部主动调用函数控制，而拥有了主动获取输入数据的能力。当外部给参考模型发送数据时，参考模型不会立即响应，而是将这一数据保存起来，等待其执行逻辑主动获取该数据。

### 函数匹配

将参考模型定义好后，要能匹配上 Agent 中的驱动函数以及检测函数。这里的 AdderAgent 定义了两个函数：

```python
class AdderAgent(Agent):
    @driver_method()
    async def exec_add(self, a, b, cin):
        self.bundle.a.value = a
        self.bundle.b.value = b
        self.bundle.cin.value = cin
        await self.bundle.step()
        return self.bundle.sum.value, self.bundle.cout.value

    @monitor_method()
    async def monitor_once(self):
        return self.bundle.as_dict()
```


#### 驱动函数匹配

那么如果我们想要编写与之对应的参考模型，自然地，我们需要定义这四个驱动函数被调用时参考模型的行为。也就是说为每一个驱动函数编写一个对应的函数，这些函数将会在驱动函数被调用时被框架自动调用。

如何让参考模型中定义的函数能够与某个驱动函数匹配呢？首先应该使用 @driver_hook 装饰器来表示这个函数是一个驱动函数的匹配函数。接着，为了建立对应关系，我们需要在装饰器中指定其对应的 Agent 和驱动函数的名称。最后，只需要保证函数的参数与驱动函数的参数一致，两个函数便能够建立对应关系。

对于 AdderAgent 中的驱动函数 exec_add，我们应该这样写才能匹配上：

```python
class AdderModelWithDriverHook(Model):
    @driver_hook(agent_name="add_agent", driver_name="exec_add")
    def exec_add(self, a, b, cin):
        result = a + b + cin
        sum = result & ((1 << 64) - 1)
        cout = result >> 64
        return sum, cout
```

此时，驱动函数与参考模型的对应关系已经建立，当 Env 中的某个驱动函数被调用时，参考模型中对应的函数将会被自动调用，并自动对比两者的返回值是否一致。


#### 监测函数匹配

检测函数也是一样的，和上面类似：

```python
class AdderModelWithMonitorHook(Model):
    @monitor_hook(agent_name="add_agent", driver_name="monitor_once")
    def monitor_once(self, item):
        sum = item["a"] + item["b"] + item["cin"]
        assert sum & ((1 << 64) - 1) == item["sum"]
        assert sum >> 64 == item["cout"]
```


## 使用 toffee-test 管理测试用例

### fixture
我们使用基于 Toffee 的插件 toffee-test 提供的方法来管理测试用例。首先创建一个 adder_env 的 Fixture，用于在每个测试用例之前初始化验证环境。在该 Fixture 中，我们使用了 toffee_request.create_dut 方法来创建加法器实例，从而 DUT 的波形、覆盖率文件及测试报告会由 toffee-test 生成并收集到指定文件夹。

```python
@toffee_test.fixture
async def adder_env(toffee_request: toffee_test.ToffeeRequest):
    dut = toffee_request.create_dut(DUTAdder)
    start_clock(dut)
    return AdderEnv(AdderBundle.from_prefix("io_").bind(dut))
```

### 添加测试用例

```python
@toffee_test.testcase
async def test_random(adder_env):
    for _ in range(1000):
        a = random.randint(0, 2**64 - 1)
        b = random.randint(0, 2**64 - 1)
        cin = random.randint(0, 1)
        await adder_env.add_agent.exec_add(a, b, cin)


@toffee_test.testcase
async def test_boundary(adder_env):
    for cin in [0, 1]:
        for a in [0, 2**64 - 1]:
            for b in [0, 2**64 - 1]:
                await adder_env.add_agent.exec_add(a, b, cin)
```

这两个测试用例中的参数 adder_env 是独立互不干扰的。

### 添加功能检查点

功能检查点(Cover Point) 在验证中用于检验待测设计的某种情况是否被验证到。同一类功能检查点可以被组织成一个测试组(Cover Group)或者也可以成为覆盖组，用于统计某一类功能检查点的覆盖率。在 Toffee 中，我们使用 CovGroup 和 add_watch_point 来定义覆盖组并添加功能检查点。

对于 Adder，我们只需要检查它的加法是否正确，因此可以给这个测试组或者覆盖组起名叫做 “Adder addition function”，意思就是专门测加法模块的加法功能。这个组里包含了很多的测试点，我们使用 add_cover_point 方法来添加测试点或者功能检查点：

```python
    g.add_cover_point(adder.io_cout, {"io_cout is 0": fc.Eq(0)}, name="Cout is 0")
    g.add_cover_point(adder.io_cout, {"io_cout is 1": fc.Eq(1)}, name="Cout is 1")
    g.add_cover_point(adder.io_cin, {"io_cin is 0": fc.Eq(0)}, name="Cin is 0")
    g.add_cover_point(adder.io_cin, {"io_cin is 1": fc.Eq(1)}, name="Cin is 1")
    g.add_cover_point(adder.io_a, {"a > 0": fc.Gt(0)}, name="signal a set")
    g.add_cover_point(adder.io_b, {"b > 0": fc.Gt(0)}, name="signal b set")
    g.add_cover_point(adder.io_sum, {"sum > 0": fc.Gt(0)}, name="signal sum set")
```

顾名思义，之所以要设计功能检查点，是因为上面的两个测试可能覆盖不掉，比如只测试了 io_cin 是 0 的情况，因此这里的
```python
    g.add_cover_point(adder.io_cin, {"io_cin is 0": fc.Eq(0)}, name="Cin is 0")
    g.add_cover_point(adder.io_cin, {"io_cin is 1": fc.Eq(1)}, name="Cin is 1")
```

就在检查测试是否包含了 0、1 两种情况，都要包含才是合格的测试。可以在 fixture 中添加该测试组到验证环境中：

```python
@toffee_test.fixture
async def adder_env(toffee_request: toffee_test.ToffeeRequest):
    dut = toffee_request.create_dut(DUTAdder)
    toffee_request.add_cov_groups(adder_cover_point(dut))  # 添加测试组
    start_clock(dut)
    return AdderEnv(AdderBundle.from_prefix("io_").bind(dut))
```

可以看到，一个测试组基本上就是在验证一个功能，而要保证这个功能是够正确就得写足够多的测试点通过设计极端情况来尽量覆盖掉所有可能。
















