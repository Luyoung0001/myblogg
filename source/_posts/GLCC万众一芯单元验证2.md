---
title: GLCC万众一芯单元验证（二）
date: 2025-07-27 13:03:13
tags: 万众一芯
categories: 体系结构
---

## 练习0: 冒烟测试

这个练习是[第二章](https://open-verify.cc/beginner/task/picker/) 的练习，我改成异步的了：
```python
from SyncFIFO import *
import asyncio
    #创建test_reset_dut 函数作为的测试用例：把rst_n信号拉低 5 个周期后，再拉高 2 个周期，导出波形信号
async def test_reset_dut(dut: DUTSyncFIFO):
    # 复位
    dut.rst_n.value = 0
    await dut.AStep(1)
    dut.rst_n.value = 1
    await dut.AStep(1)

    # 向 FIFO 分别写入两个数据：
    dut.wr_en.value = 1
    dut.wdata.value = 0x14
    await dut.AStep(1)
    dut.we_i.value = 0
    await dut.AStep(1)

    assert dut.empty_o.value == 0,"Mismatch"
    assert dut.full_o.value == 0,"Mismatch"

    # 写第二个数据
    dut.we_i.value = 1
    dut.data_i.value = 0x514
    await dut.AStep(1)
    dut.we_i.value = 0
    await dut.AStep(1)
    # 从 FIFO 读出两个数据：
    # 给we_i置低、re_i置高，保持一个周期，之后读取data_o，判断结果是否为0x114
    # 给we_i置低、re_i置高，保持一个周期，之后读取data_o，判断结果是否为0x514、empty_o是否为 1
    dut.wr_en.value = 0
    dut.rd_en.value = 1
    await dut.AStep(1)
    dut.re_i.value = 1
    await dut.AStep(1)

    assert dut.data_o.value == 0x114,"Mismatch"
    # 读取第二个数据
    dut.we_i.value = 0
    dut.re_i.value = 1
    await dut.AStep(1)
    dut.we_i.value = 0
    dut.re_i.value = 0
    await dut.AStep(1)
    assert dut.data_o.value == 0x514,"Mismatch"
    assert dut.empty_o.value == 1,"Mismatch"

async def main(dut: DUTSyncFIFO):
    asyncio.create_task(test_reset_dut(dut))
    await asyncio.create_task(dut.RunStep(10))  # 让时钟持续推进 10 个周期

if __name__ == "__main__":
    #创建同步 FIFO 的 DUT 类
    dut = DUTSyncFIFO()
    dut.InitClock("clk")
    asyncio.run(main(dut))
    dut.Finish()
```

这里一定要注意的是，就拿写数据来说，如果写完后推进一个周期，那么 empty_o 刚位于信号的下降沿，如图蓝色箭头所指，此时进行 assert 断言会错误。
![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/pytest1.png)

因此一个写的事务，应该是这样的：
```python
    dut.wr_en.value = 1
    dut.wdata.value = 0x14
    await dut.AStep(1)
    dut.we_i.value = 0
    await dut.AStep(1)
```
也就是说，写完之后，将 we_i 的信号拉低，再推进一个周期，这样断言就能成功。

这个案例是一个异步控制。\_\_main\_\_  在实例化 dut 之后，通过异步发起调用 main()。由于 main() 也是一个异步函数，它构造了两个并发的任务：

```python
asyncio.create_task(test_reset_dut(dut))  # 启动 test_reset_dut(dut)，在后台异步运行
task = asyncio.create_task(dut.RunStep(10))  # 启动 dut.RunStep(10)，也是异步运行
await task  # 等待 RunStep(10) 执行完
```

这样就解耦了 clk 和 其他信号，clk 在后台推进，另一个协程做驱动和观测。另外，只有 RunStep() 或类似主控协程才能推进时钟，所有 ASTep() 的协程只是“等着推进的结果”，clk 就是利用了异步的特性推进。

## 练习1: 用 toffee-test 管理测试用例

这个任务就是利用 toffee-test 重新测试 练习0：

```python
from SyncFIFO import DUTSyncFIFO
import toffee
import toffee_test

from toffee_test import ToffeeRequest

@toffee_test.testcase
async def test_with_ref(dut: DUTSyncFIFO):
    # 复位
    dut.rst_n.value = 0
    await dut.AStep(1)
    dut.rst_n.value = 1
    await dut.AStep(1)

    # 向 FIFO 分别写入两个数据

    # 写第一个数据
    dut.we_i.value = 1
    dut.data_i.value = 0x114
    await dut.AStep(1)
    dut.we_i.value = 0
    await dut.AStep(1)

    assert dut.empty_o.value == 0,"Mismatch"
    assert dut.full_o.value == 0,"Mismatch"
    # 写第二个数据
    dut.we_i.value = 1
    dut.data_i.value = 0x514
    await dut.AStep(1)
    dut.we_i.value = 0
    await dut.AStep(1)

    # 从 FIFO 读出两个数据
    # 读第一个数据
    dut.we_i.value = 0
    dut.re_i.value = 1
    await dut.AStep(1)
    dut.re_i.value = 1
    await dut.AStep(1)

    assert dut.data_o.value == 0x114,"Mismatch"
    # 读取第二个数据
    dut.we_i.value = 0
    dut.re_i.value = 1
    await dut.AStep(1)
    dut.we_i.value = 0
    dut.re_i.value = 0
    await dut.AStep(1)
    assert dut.data_o.value == 0x514,"Mismatch"
    assert dut.empty_o.value == 1,"Mismatch"


@toffee_test.fixture
async def dut(toffee_request: ToffeeRequest):
    rand_dut = toffee_request.create_dut(DUTSyncFIFO, "clk")

    toffee.start_clock(rand_dut)  # 让toffee驱动时钟，只能在异步函数中调用
    return rand_dut
```

练习0 有几个缺点，为了让 clk 分离，专门给他写了一个异步进程。而且必须使用 `python3 smoke_test.py` 来执行整个脚本才能知道测试是否通过。

虽然 Picker 提供了底层的硬件交互能力，但要构建一个结构化、可复用、易于维护的验证环境，还需要更完善的框架和方法学支持。toffee 它是一个基于 picker 的硬件验证框架，旨在提供一套更高效、更规范的验证解决方案。

toffee_test 这个插件非常好用，它能自动识别监测点，还能首先执行 fixture。还能给出整个测试报告，它太方便了。toffee-test 插件简化了测试用例的编写、执行、管理和报告生成，支持 fixture 等高级测试特性。

```bash
pytest . -sv
================================== test session starts ==================================
platform linux -- Python 3.10.12, pytest-8.4.0, pluggy-1.6.0 -- /usr/bin/python3
cachedir: .pytest_cache
rootdir: /home/luyoung/chip_functional_verification/chapter3/practice0
configfile: pyproject.toml
plugins: allure-pytest-2.14.3, asyncio-1.0.0, reporter-html1-0.9.3, xdist-3.7.0, anyio-4.8.0, reporter-0.5.3, toffee-test-0.3.1.dev18+g7ee81a0
asyncio: mode=strict, asyncio_default_fixture_loop_scope=function, asyncio_default_test_loop_scope=function
collected 1 item

tests/test_smoke.py::test_with_ref PASSED
```
它能自动识别监测点，并进行测试。如果测试通过，会给出 PASSED。

默认情况下，每个测试用例都会获得由 Fixture 重新执行并返回的独立实例。这意味着 test_with_ref 使用的 dut 实例与另一个测试用例 test_another(dut) 使用的 dut 实例是不同的对象，它们之间状态隔离，确保了测试的独立性。

## 练习2：使用 Bundle 封装 DUT

### 封装 FIFO 的端口

在 bundle/write_bundle.py 中编辑：
```python
from toffee import Bundle, Signals

class WriteBundle(Bundle):
    we, data = Signals(2)

    async def enqueue(self, data):
        self.data.value = data
        self.we.value = 1
        await self.step()
        self.we.value = 0
        await self.step()

```
在 bundle/read_bundle.py 中编辑：
```python
from toffee import Bundle, Signals

class ReadBundle(Bundle):
    re, data = Signals(2)

    async def dequeue(self):
        self.re.value = 1
        await self.step()
        self.re.value = 0
        await self.step()
        return self.data.value

```
在 bundle/internal_bundle.py 中编辑：
```python
from toffee import Bundle, Signals

class InternalBundle(Bundle):
    full, empty = Signals(2)
```

### 创建测试用例：

在 bundle/`__init__`.py 中编辑：
```python
from .write_bundle import WriteBundle
from .read_bundle import ReadBundle
from .internal_bundle import InternalBundle

__all__ = ["WriteBundle", "ReadBundle", "InternalBundle"]
```
在 ../tests/test_smoke.py 中编辑：
```python
from SyncFIFO import DUTSyncFIFO
import toffee
import toffee_test

from toffee_test import ToffeeRequest
from bundle.write_bundle import WriteBundle
from bundle.read_bundle import ReadBundle
from bundle.internal_bundle import InternalBundle


@toffee_test.testcase
async def test_bundle(dut):
    # 复位
    dut.rst_n.value = 0
    await dut.AStep(1)
    dut.rst_n.value = 1
    await dut.AStep(1)

    write_bundle = WriteBundle.from_dict({
        "we": "we_i",
        "data": "data_i"
    })
    read_bundle = ReadBundle.from_dict({
        "re": "re_i",
        "data": "data_o"
    })
    internal_bundle = InternalBundle.from_dict({
        "full": "full_o",
        "empty": "empty_o"
    })

    # 然后分别绑定：
    write_bundle.bind(dut)
    read_bundle.bind(dut)
    internal_bundle.bind(dut)

    await write_bundle.enqueue(42)
    val = await read_bundle.dequeue()
    assert val == 42, f"Expected 42, got {val}"


@toffee_test.testcase
async def test_full_empty(dut):
    # 复位
    dut.rst_n.value = 0
    await dut.AStep(1)
    dut.rst_n.value = 1
    await dut.AStep(1)

    write_bundle = WriteBundle.from_dict({
        "we": "we_i",
        "data": "data_i"
    })
    read_bundle = ReadBundle.from_dict({
        "re": "re_i",
        "data": "data_o"
    })
    internal_bundle = InternalBundle.from_dict({
        "full": "full_o",
        "empty": "empty_o"
    })

    # 然后分别绑定：
    write_bundle.bind(dut)
    read_bundle.bind(dut)
    internal_bundle.bind(dut)

    input_seq = list(range(16))

    # 装满 FIFO
    for val in input_seq:
        await write_bundle.enqueue(val)

    assert internal_bundle.full.value == 1, "FIFO should be full"

    # 清空 FIFO
    output_seq = []
    for _ in input_seq:
        val = await read_bundle.dequeue()
        output_seq.append(val)

    assert internal_bundle.empty.value == 1, "FIFO should be empty"

    assert input_seq == output_seq, f"Mismatch: {input_seq} != {output_seq}"

@toffee_test.fixture
async def dut(toffee_request: ToffeeRequest):
    dut = toffee_request.create_dut(DUTSyncFIFO, "clk")
    toffee.start_clock(dut) # 异步驱动时钟
    return dut
```

绑定引脚之后，驱动：
```bash
pytest . -sv
================================== test session starts ==================================
platform linux -- Python 3.10.12, pytest-8.4.0, pluggy-1.6.0 -- /usr/bin/python3
cachedir: .pytest_cache
rootdir: /home/luyoung/chip_functional_verification/chapter3/practice1
configfile: pyproject.toml
plugins: allure-pytest-2.14.3, asyncio-1.0.0, reporter-html1-0.9.3, xdist-3.7.0, anyio-4.8.0, reporter-0.5.3, toffee-test-0.3.1.dev18+g7ee81a0
asyncio: mode=strict, asyncio_default_fixture_loop_scope=function, asyncio_default_test_loop_scope=function
collected 2 items

tests/test_smoke.py::test_bundle PASSED
tests/test_smoke.py::test_full_empty PASSED
```

## 练习3：使用 Agent 进一步封装

Agent 在 Toffee 中提供了一个更高层次的抽象，它通常封装一个或多个 Bundle，并定义与这些接口相关的 事务级 （Transaction-Level） 操作。

由于异步的操作很难写，而且在更高层次我不在乎是不是异步，我只要效果。因此可以对 bundle 进一步封装，将其封装成普通函数，能直接调用。而且 Agent 将驱动和观测分离：
- 驱动方法 （Driver Method）：主动向 DUT 发起操作，通过控制 Bundle 信号实现。通常带有参数（输入数据/配置）和返回值（操作结果/读取的数据）。
- 监测方法 （Monitor Method）：被动地观察 Bundle 信号，当满足特定条件时，捕获接口上的活动或状态，并可能生成表示该活动的事务对象或数据。

总之，Agent 屏蔽了 bundle 的复杂性，并提供了更简单的接口。

### 定义 Agent

在 fifo_agent.py 中编辑：
```python
from toffee import Agent, driver_method, monitor_method

class FIFOAgent(Agent):
    def __init__(self, read_bundle, write_bundle, internal_bundle):
        # 将 read_bundle 传给父类以获取时钟
        super().__init__(read_bundle)
        self.read = read_bundle
        self.write = write_bundle
        self.internal = internal_bundle

    @driver_method()
    async def reset(self):
        self.internal.resetn.value = 0
        await self.internal.step()
        await self.internal.step()
        self.internal.resetn.value = 1
        await self.internal.step()

    @driver_method()
    async def enqueue(self, data):
        await self.write.enqueue(data)

    @driver_method()
    async def dequeue(self):
        return await self.read.dequeue()

    @monitor_method()
    async def monitor_once(self):
        return {
            "full": self.internal.full_o.value,
            "empty": self.internal.empty_o.value
        }

```

在 `__init__.py` 中编辑：
```python
from .fifo_agent import FIFOAgent
__all__ = ["FIFOAgent"]
```

新的 smoke_test.py 大差不差。都是实例化 bundle，然后绑定dut，接着传给 Agent 构造 Agent 实例，然后就是快乐的调用了：

```python

from SyncFIFO import DUTSyncFIFO
import toffee
import toffee_test
from toffee_test import ToffeeRequest

from bundle.write_bundle import WriteBundle
from bundle.read_bundle import ReadBundle
from bundle.internal_bundle import InternalBundle
from agent.fifo_agent import FIFOAgent  # 你写的 FIFOAgent

@toffee_test.testcase
async def test_agent(dut):
    write_bundle = WriteBundle.from_dict({
        "we": "we_i",
        "data": "data_i"
    })
    read_bundle = ReadBundle.from_dict({
        "re": "re_i",
        "data": "data_o"
    })
    internal_bundle = InternalBundle.from_dict({
        "full": "full_o",
        "empty": "empty_o",
        "resetn": "rst_n"
    })

    # 然后分别绑定：
    write_bundle.bind(dut)
    read_bundle.bind(dut)
    internal_bundle.bind(dut)

    agent = FIFOAgent(read_bundle,write_bundle,internal_bundle)

    await agent.reset()
    await agent.enqueue(42)
    val = await agent.dequeue()
    assert val == 42, f"Expected 42, got {val}"


@toffee_test.testcase
async def test_full_empty(dut):
    write_bundle = WriteBundle.from_dict({
        "we": "we_i",
        "data": "data_i"
    })
    read_bundle = ReadBundle.from_dict({
        "re": "re_i",
        "data": "data_o"
    })
    internal_bundle = InternalBundle.from_dict({
        "full": "full_o",
        "empty": "empty_o",
        "resetn": "rst_n"
    })

    # 然后分别绑定：
    write_bundle.bind(dut)
    read_bundle.bind(dut)
    internal_bundle.bind(dut)

    agent = FIFOAgent(read_bundle,write_bundle,internal_bundle)

    await agent.reset()

    input_seq = list(range(16))

    for val in input_seq:
        await agent.enqueue(val)

    assert internal_bundle.full.value == 1, "FIFO should be full"

    output_seq = []
    for _ in input_seq:
        val = await agent.dequeue()
        output_seq.append(val)

    assert internal_bundle.empty.value == 1, "FIFO should be empty"
    assert input_seq == output_seq, f"Mismatch: {input_seq} != {output_seq}"


@toffee_test.fixture
async def dut(toffee_request: ToffeeRequest):
    dut = toffee_request.create_dut(DUTSyncFIFO, "clk")
    toffee.start_clock(dut)
    return dut
```

测试结果也是没问题：
```bash
pytest . -sv
================================== test session starts ==================================
platform linux -- Python 3.10.12, pytest-8.4.0, pluggy-1.6.0 -- /usr/bin/python3
cachedir: .pytest_cache
rootdir: /home/luyoung/chip_functional_verification/chapter3/practice1
configfile: pyproject.toml
plugins: allure-pytest-2.14.3, asyncio-1.0.0, reporter-html1-0.9.3, xdist-3.7.0, anyio-4.8.0, reporter-0.5.3, toffee-test-0.3.1.dev18+g7ee81a0
asyncio: mode=strict, asyncio_default_fixture_loop_scope=function, asyncio_default_test_loop_scope=function
collected 2 items

tests/test_smoke.py::test_agent PASSED
tests/test_smoke.py::test_full_empty PASSED
```

以上就是关于官方教程的练习的前三个。


















