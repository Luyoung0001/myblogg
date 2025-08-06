---
title: GLCC万众一芯单元验证（一）
date: 2025-07-25 13:03:13
tags: 万众一芯
categories: 体系结构
---

## 问题

在[官方教程](https://open-verify.cc/beginner/course/3-toffee/)中有这样一个事例：

```Python
# test_with_toffee.py
from RandomGenerator import DUTRandomGenerator
import random
from toffee import start_clock, create_dut
from toffee_test import ToffeeRequest


# 定义参考模型
class LFSR_16:
    def __init__(self, seed):
        self.state = seed & ((1 << 16) - 1)

    def Step(self):
        new_bit = (self.state >> 15) ^ (self.state >> 14) & 1
        self.state = ((self.state << 1) | new_bit) & ((1 << 16) - 1)


@toffee_test.testcase
async def test_with_ref(dut: DUTRandomGenerator):
    ref = LFSR_16(seed)                  # 创建参考模型用于对比

    for i in range(65536):  # 循环65536次
        dut.Step()          # dut 推进一个时钟周期，生成随机数
        ref.Step()          # ref 推进一个时钟周期，生成随机数
        # 对比DUT和参考模型生成的随机数
        rand = dut.random_number.value
        assert rand == ref.state, "Mismatch"


@toffee_test.fixture
async def dut(toffee_request: ToffeeRequest):
    # 使用toffee创建DUT并绑定时钟
    dut = create_dut(DUTRandomGenerator, "clk")
    seed = random.randint(0, 2**16 - 1)  # 生成随机种子
    dut.seed.value = seed                # 设置DUT种子
    # reset DUT
    dut.reset.value = 1  # reset 信号置1
    dut.Step()           # 推进一个时钟周期（DUTRandomGenerator是时序电路，需要通过Step推进）
    dut.reset.value = 0  # reset 信号置0
    dut.Step()           # 推进一个时钟周期
    return dut
```
这个在用 `pytest . -sv` 的时候会报错，因为：
```bash
>>> from toffee import start_clock, create_dut
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ImportError: cannot import name 'create_dut' from 'toffee' (/home/luyoung/.local/lib/python3.10/site-packages/toffee/__init__.py)
```
create_dut 定义在 ToffeeRequest 类中（toffee_test.request模块里），它是类的实例方法，不是模块的顶层函数

因此需要先获得 ToffeeRequest 的实例，再调用它的 create_dut 方法：

```python
from toffee import start_clock
from toffee_test import ToffeeRequest
import toffee_test

@toffee_test.fixture
async def dut(toffee_request: ToffeeRequest):
    # toffee_request 已经是 ToffeeRequest 实例
    dut = toffee_request.create_dut(DUTRandomGenerator, "clk")
    ...
    return dut
```

这样例子就能运行通了：

```bash
pytest . -sv
================================== test session starts ==================================
platform linux -- Python 3.10.12, pytest-8.4.0, pluggy-1.6.0 -- /usr/bin/python3
cachedir: .pytest_cache
rootdir: /home/luyoung/chip_functional_verification/chapter2/example
plugins: allure-pytest-2.14.3, asyncio-1.0.0, reporter-html1-0.9.3, xdist-3.7.0, anyio-4.8.0, reporter-0.5.3, toffee-test-0.3.1.dev18+g7ee81a0
asyncio: mode=strict, asyncio_default_fixture_loop_scope=function, asyncio_default_test_loop_scope=function
collected 1 item

test_with_toffee.py::test_with_ref PASSED

=================================== warnings summary ====================================
<frozen importlib._bootstrap>:241
<frozen importlib._bootstrap>:241
  <frozen importlib._bootstrap>:241: DeprecationWarning: builtin type SwigPyPacked has no __module__ attribute

<frozen importlib._bootstrap>:241
<frozen importlib._bootstrap>:241
  <frozen importlib._bootstrap>:241: DeprecationWarning: builtin type SwigPyObject has no __module__ attribute

<frozen importlib._bootstrap>:241
  <frozen importlib._bootstrap>:241: DeprecationWarning: builtin type swigvarlink has no __module__ attribute

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
============================= 1 passed, 5 warnings in 0.24s =============================
sys:1: DeprecationWarning: builtin type swigvarlink has no __module__ attribute
```

还有一些警告，这是 Python 给出的弃用警告（DeprecationWarning），意思是：

这些类型（SwigPyPacked、SwigPyObject、swigvarlink）是由 SWIG 生成的 Python C++ 绑定类型（SWIG 是一种用来自动生成 Python 和 C/C++ 代码接口的工具）

它们是“内置类型”（builtin type），但是缺少 __module__ 属性

__module__ 属性用于标明某个对象属于哪个模块

Python 计划将来不再支持这种缺失 __module__ 属性的内置类型行为，所以提示你这是个弃用的情况。