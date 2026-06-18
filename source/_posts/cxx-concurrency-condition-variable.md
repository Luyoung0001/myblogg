---
title: "C++ 并发编程：使用 condition_variable 等待条件成立"
date: 2024-05-15 16:10:02
tags:
  - "C++"
  - "concurrency"
  - "condition variable"
categories:
  - "Programming Languages"
---

<!--more-->

## 一、用std::condition_variable等待处理数据
伪代码如下：

```cpp
std::mutex mut;
std::queue<data_chunk> data_queue;    ⇽---  ①
std::condition_variable data_cond;
void data_preparation_thread()            // 由线程乙运行
{
    while(more_data_to_prepare())
    {
        data_chunk const data=prepare_data();
        {
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push(data);    ⇽---  ②
        }
        data_cond.notify_one();    ⇽---  ③
    }
}
void data_processing_thread()           // 由线程甲运行
{
    while(true)
    {
        std::unique_lock<std::mutex> lk(mut);    ⇽---  ④
        data_cond.wait(
            lk,[]{return !data_queue.empty();});    ⇽---  ⑤
        data_chunk data=data_queue.front();
        data_queue.pop();
        lk.unlock();    ⇽---  ⑥
        process(data);
        if(is_last_chunk(data))
            break;
    }
}
```
在C++中，使用`std::condition_variable`与互斥锁（如`std::mutex`）结合时，其行为如下，特别是关于锁的处理：

1. **锁的获取与释放**：
   - `std::unique_lock<std::mutex>`对象`lk`在创建时获取`mut`互斥锁。
   - 当调用`data_cond.wait(lk, ...)`时，这个函数会自动释放`lk`所持有的互斥锁，并使当前线程（在这里是数据处理线程）进入等待状态。
   - 等待期间，线程会阻塞在`wait`调用上，直到被唤醒（通常是由另一线程调用`notify_one()`或`notify_all()`）。

2. **条件变量检查**：
   - 当线程被唤醒（如由`notify_one()`），`data_cond.wait()`自动重新获取互斥锁`lk`。
   - 一旦重新获得互斥锁，`wait()`将执行传入的**lambda**函数（在这里是`[]{return !data_queue.empty();}`）以检查等待条件是否满足。
   - 如果**lambda**函数返回`true`（即队列不为空），`wait()`结束，线程继续执行。
   - 如果**lambda**函数返回`false`，`wait()`会再次释放锁，并将线程置回等待状态。

3. **循环等待和锁的管理**：
   - 这种等待和检查的过程可能会发生多次，直到条件最终满足。每次在检查条件之前，线程必须重新获得互斥锁。
   - 一旦条件成立，线程将继续持有互斥锁并从`wait()`返回，执行后续代码。

4. **锁的释放**：
   - 在示例中，线程在从队列中取出数据后，手动调用`lk.unlock()`来释放互斥锁，然后处理数据。
   - 这是为了避免在执行可能耗时的`process(data)`操作时持有互斥锁，从而增加其他线程操作共享资源的机会。

### 简单总结
`std::condition_variable::wait()`的工作方式如下：
- 在调用`wait()`时，线程会释放其持有的互斥锁，并进入阻塞状态。
- 线程被唤醒后，`wait()`自动重新获取互斥锁，并检查指定的条件。
- 如果条件不满足，线程再次释放互斥锁并进入阻塞状态。
- 如果条件满足，`wait()`返回，线程继续执行并持有互斥锁。

这种机制确保了资源的安全访问和高效管理，使得多线程编程更为可靠和易于维护。

## 二、使用 `std::unique_lock<std::mutex>` 而非 `std::lock_guard<std::mutex>` 
在这个示例中，选择使用 `std::unique_lock<std::mutex>` 而非 `std::lock_guard<std::mutex>` 是因为 `std::unique_lock` 提供了更多的灵活性，尤其是在与条件变量结合使用时。以下是详细的原因和对两者差异的解释：

### 1. **锁的可解锁性**
`std::unique_lock` 允许显式地锁定和解锁操作。这一特性在使用条件变量时非常关键，因为：
- `std::condition_variable::wait()` 函数需要在等待过程中释放锁，并在条件满足、线程被唤醒后重新获取锁。`std::unique_lock` 可以自动处理这些操作。

### 2. **与条件变量的兼容性**
`std::condition_variable` 只能与 `std::unique_lock` 一起使用，**因为它依赖于 `std::unique_lock` 的能力来：**
- 在进入 `wait()` 时释放锁。
- 在从 `wait()` 返回时重新获得锁。

`std::lock_guard` 不具备这种灵活性，因为它仅在构造时获取锁，并在析构时释放锁，没有提供中间释放和重新获得锁的机制。

### 3. **灵活的锁管理**
`std::unique_lock` 同时支持延迟锁定（不在构造时立即锁定），尝试锁定，以及在某个作用域中手动控制锁的释放和再次锁定。这为编程提供了更高的灵活性，可以根据需要进行精细的锁控制，这在处理复杂的多线程逻辑时非常有用。

### 4. **性能考量**
尽管 `std::unique_lock` 相比 `std::lock_guard` 在灵活性上有优势，它可能在性能上略逊一筹，因为它需要处理更多的状态和潜在的操作。但在需要使用条件变量或复杂锁管理的场景中，这种性能损失通常是可以接受的。

## 三、伪唤醒
伪唤醒（spurious wakeups）是多线程编程中一个已知的现象，它可以在使用条件变量时发生。伪唤醒指的是线程在没有接收到显式通知的情况下，从等待状态（如 `std::condition_variable` 的 `wait()` 方法）中意外醒来。这种情况在C++标准中被允许发生，而且发生的频率和数量都是不确定的。

### 伪唤醒的产生原因

伪唤醒的具体内部机制依赖于操作系统的线程调度和管理策略，它**可能与操作系统如何实现线程间的信号传递和等待/通知机制有关**。通常，这些理由包括但不限于：

1. **优化和性能**：在某些操作系统实现中，允许伪唤醒可以使得内核的等待/通知机制实现更加简单和高效。避免在唤醒时处理复杂的条件检查可以减少线程切换的开销。

2. **资源竞争**：在高负载的系统中，处理信号可能涉及竞争和复杂的状态同步，伪唤醒可能是系统尝试在高并发状态下减少锁的使用或避免死锁的副作用。

3. **中断和信号处理**：有时系统中断或外部信号可能导致线程提前从等待状态返回。

### 应对伪唤醒的策略

由于伪唤醒是允许发生的，C++标准库设计了 `std::condition_variable` 的 `wait()` 函数，允许它接受一个谓词（predicate）。这个设计确保即使在伪唤醒发生后，只有当谓词为真时，等待循环才会停止。这就是为什么在使用条件变量时，通常推荐使用如下模式：

```cpp
std::unique_lock<std::mutex> lk(mut);
data_cond.wait(lk, []{ return !data_queue.empty(); });  // 谓词检查队列非空
```

在这个模式中，即使发生了伪唤醒，谓词函数将检查条件是否真正满足，如果不满足，线程将会再次进入等待状态。

**故此，若判定函数有副作用，则不建议选取它来查验条件。** 倘若真的要这么做，就有可能多次产生副作用，所以必须准备好应对方法。譬如，每次被调用时，判定函数就顺带提高所属线程的优先级，该提升动作即产生的副作用。结果，多次伪唤醒可“意外地”令线程优先级变得非常高。



