---
title: C++并发：构建线程安全的队列
date: 2024-05-15 17:44:36
tags: c++
categories: C++
---

<!--more-->

## 正文
线程安全队列的完整的类定义，其中采用了条件变量：

```cpp
#include <condition_variable>
#include <memory>
#include <mutex>
#include <queue>
template <typename T> class threadsafe_queue {
  private:
    mutable std::mutex mut;
    std::queue<T> data_queue;
    std::condition_variable data_cond;

  public:
    threadsafe_queue() {}
    threadsafe_queue(threadsafe_queue const &other) {
        std::lock_guard<std::mutex> lk(other.mut);
        data_queue = other.data_queue;
    }
    void push(T new_value) {
        std::lock_guard<std::mutex> lk(mut);
        data_queue.push(new_value);
        data_cond.notify_one();
    }
    void wait_and_pop(T &value) {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(lk, [this] { return !data_queue.empty(); });
        value = data_queue.front();
        data_queue.pop();
    }
    std::shared_ptr<T> wait_and_pop() {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(lk, [this] { return !data_queue.empty(); });
        std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
        data_queue.pop();
        return res;
    }
    bool try_pop(T &value) {
        std::lock_guard<std::mutex> lk(mut);
        if (data_queue.empty())
            return false;
        value = data_queue.front();
        data_queue.pop();
        return true;
    }
    std::shared_ptr<T> try_pop() {
        std::lock_guard<std::mutex> lk(mut);
        if (data_queue.empty())
            return std::shared_ptr<T>();
        std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
        data_queue.pop();
        return res;
    }
    bool empty() const {
        std::lock_guard<std::mutex> lk(mut);
        return data_queue.empty();
    }
};
```

这个队列的设计允许多个生产者和消费者线程安全地向队列中添加或移除元素，而无需担心数据竞争或其他并发错误。通过 `std::condition_variable` 的使用，消费者线程可以有效地等待直到队列中有数据可用，从而优化资源使用和线程调。

在多线程环境中，使用 `mutable` 关键字修饰 `std::mutex` 类型的成员变量是一种常见的做法，特别是在类设计中涉及到需要保护类成员不被多个线程同时修改的情况下。下面我们详细解释一下 `mutable` 的使用背景、意义以及为什么在 `threadsafe_queue` 类中应用它。

### mutable的作用

`mutable` 修饰符用于C++中，表示即使在一个 `const` 成员函数中，该成员变量仍可被修改。`const` 成员函数承诺不修改对象的任何数据成员（不包括由 `mutable` 修饰的成员）。这个特性在处理需要修改类成员但又不改变对象状态的设计模式（如缓存、锁等）时非常有用。

### 应用于 threadsafe_queue

在 `threadsafe_queue` 类中，成员函数 `empty` 被声明为 `const`，意味着这个函数不应修改对象的任何数据成员。然而，这个函数内部需要使用 `mutex` 来保证线程安全性，即使它只是检查队列是否为空。由于 `mutex` 通常会在锁定和解锁时修改其内部状态，所以正常情况下你不能在 `const` 函数中进行这些操作。

为了解决这一问题，`mutex` 成员变量被声明为 `mutable`。这允许即使在 `const` 成员函数中，我们也可以锁定和解锁互斥量，而不违反函数的 `const` 性质。这样做确保了即使在多线程环境中，`empty` 函数执行时，队列的状态检查是线程安全的。

### 在构造函数中的应用

在 `threadsafe_queue` 的拷贝构造函数中，尽管传入的 `other` 对象是一个 `const` 引用，我们仍然需要从这个 `const` 对象中复制数据。拷贝构造函数需要访问 `other` 对象的 `data_queue`，而为了线程安全，必须先锁定 `other` 的互斥量。由于 `mut` 是 `mutable` 的，即使在 `const` 上下文中，也能执行锁定操作。

### 运行结果
写一个多线程的测试程序：

```cpp
void producer(threadsafe_queue<int> &queue, int start_value) {
    for (int i = 0; i < 5; ++i) {
        queue.push(start_value + i);
        std::this_thread::sleep_for(
            std::chrono::milliseconds(100)); // 模拟耗时操作
    }
}

std::mutex print_mutex; // 保证打印有序，方便观察
void consumer(threadsafe_queue<int> &queue) {
    for (int i = 0; i < 5; ++i) {
        int value;
        queue.wait_and_pop(value);

        std::lock_guard<std::mutex> lock(print_mutex);
        std::cout << "Consumer " << std::this_thread::get_id()
                  << " popped: " << value << std::endl;
    }
}

int main() {
    threadsafe_queue<int> queue;

    std::thread producers[3];
    std::thread consumers[3];

    // 启动生产者线程
    for (int i = 0; i < 3; ++i) {
        producers[i] = std::thread(producer, std::ref(queue),
                                   i * 10); // 每个生产者推送不同范围的数字
    }

    // 启动消费者线程
    for (int i = 0; i < 3; ++i) {
        consumers[i] = std::thread(consumer, std::ref(queue));
    }

    // 等待所有生产者线程完成
    for (int i = 0; i < 3; ++i) {
        producers[i].join();
    }

    // 等待所有消费者线程完成
    for (int i = 0; i < 3; ++i) {
        consumers[i].join();
    }

    return 0;
}
```
运行结果：

```bash
./main 
Consumer 0x16b333000 popped: 0
Consumer 0x16b333000 popped: 20
Consumer 0x16b3bf000 popped: 10
Consumer 0x16b44b000 popped: 1
Consumer 0x16b333000 popped: 11
Consumer 0x16b3bf000 popped: 21
Consumer 0x16b44b000 popped: 12
Consumer 0x16b333000 popped: 2
Consumer 0x16b3bf000 popped: 22
Consumer 0x16b44b000 popped: 13
Consumer 0x16b333000 popped: 3
Consumer 0x16b3bf000 popped: 23
Consumer 0x16b44b000 popped: 14
Consumer 0x16b3bf000 popped: 4
Consumer 0x16b44b000 popped: 24
```
这样，就实现了一个线程安全的队列。



