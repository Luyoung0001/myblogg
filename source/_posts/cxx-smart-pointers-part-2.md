---
title: "C++ 语言笔记：智能指针（二）shared_ptr 与所有权"
date: 2024-04-21 23:52:29
tags:
  - "C++"
  - "smart pointer"
  - "shared_ptr"
categories:
  - "Programming Languages"
---

<!--more-->

## 正文
为什么要设计智能指针？看看下面的例子：

```cpp
#include <iostream>
#include <memory>
#include <string>

using namespace std;

void memoryLeak1() {
    string *str = new string("动态分配内存！");
    delete str;
    return;
}

int memoryLeak2() {
    string *str = new string("内存泄露！");

    if (1) {
        throw "异常";
    }
    delete str;
    return 0;
}

int main(void) {
    memoryLeak1();

    try {
        memoryLeak2();
    } catch (...) {
        // 处理异常
    }
    return 0;
}


```
以上是两个极端且常见的例子，例子展示了两种可能导致内存泄漏的情况：

1. `memoryLeak1()` 函数中，动态分配了一个字符串对象 `str`，但在函数结束时没有释放该内存，因为没有调用 `delete str;`。即使这个函数返回，没有释放内存的操作也会造成内存泄漏。

2. `memoryLeak2()` 函数中，同样动态分配了一个字符串对象 `str`，但在函数中途就可能发生某种异常，导致函数提前返回而没有释放内存。即使在代码中写了释放内存的语句 `delete str;`，但由于函数在中途返回，该语句不会被执行，仍然会造成内存泄漏。

对于第一种情况，我们知道函数返回后，会销毁栈上的自由值，因此指针 `str` 被释放了，但是指针 `str` 指向的内存空间却没有被释放，这就导致了第一种内存泄漏。
对于第二种情况，由于栈回退机制，抛出异常后，会清空所有的对象，但是也没有释放 `str` 指向的内存。

我们的美好想法是，在栈上释放自由变量的时候，如果也能顺势执行一下自身的类似于析构函数该多好。

很好，这时候就可以想到我们如果将所有的指针都设置成一个对象，那么栈上的对象在释放时就会调用是够函数，那么我们只要将这个析构函数定义得足够好，我们就再也不用担心内存泄漏问题了。**这个想法就是智能指针的来源！**


有了这个思想之后，我们只要经过精心设计，就能达到这个目的，事实上，C++中提供了这几种智能指针，每一种都有各自的特点。C++98 提供了 `auto_ptr` 模板的解决方案，C++11 增加`unique_ptr`、`shared_ptr` 和 `weak_ptr`。
当使用C++编程时，管理动态内存是一个重要的任务，因为内存泄漏和悬挂指针等问题可能导致严重的错误。为了解决这些问题，C++标准库提供了几种智能指针，包括`unique_ptr`、`shared_ptr`和`weak_ptr`。

下面是对每种智能指针的简要总结：

1. **`unique_ptr`**：
   - `unique_ptr` 是 C++11 引入的智能指针，提供了独占所有权的语义。每个 `unique_ptr` 对象拥有对其指向对象的唯一所有权。
   - 不允许复制构造和复制赋值，这意味着它不能与其他 `unique_ptr` 共享所有权。
   - 可以使用移动语义转移所有权，从而避免昂贵的复制操作。
   - 当 `unique_ptr` 超出范围时，它所管理的资源会被自动释放，从而避免了内存泄漏的风险。

2. **`shared_ptr`**：
   - `shared_ptr` 允许多个指针共享对同一对象的所有权。它使用引用计数来跟踪共享的次数，并在不再需要时自动释放资源。
   - 允许复制构造和复制赋值，每个新的 `shared_ptr` 对象都增加了引用计数。
   - 当最后一个 `shared_ptr` 对象超出范围时，引用计数为零，资源被释放。

3. **`weak_ptr`**：
   - `weak_ptr` 是对 `shared_ptr` 的一种补充，它允许观察由 `shared_ptr` 管理的对象，而不会增加引用计数。
   - `weak_ptr` 不拥有资源的所有权，因此不会影响对象的生命周期。
   - 可以通过 `lock()` 方法获取一个 `shared_ptr` 对象，如果底层资源仍然存在的话。

以下是 `unique_ptr` 的实现示例：

```cpp
#include <iostream>

template<typename T>
class unique_ptr {
private:
    T* ptr; // 指向动态分配的内存的指针

public:
    explicit unique_ptr(T* p = nullptr) : ptr(p) {} // 构造函数

    ~unique_ptr() { // 析构函数
        delete ptr; // 释放动态分配的内存
    }

    unique_ptr(const unique_ptr&) = delete;
    unique_ptr& operator=(const unique_ptr&) = delete;

    unique_ptr(unique_ptr&& other) noexcept // 移动构造函数
        : ptr(other.ptr) {
        other.ptr = nullptr; // 置空源对象的指针，防止重复释放
    }

    unique_ptr& operator=(unique_ptr&& other) noexcept { // 移动赋值运算符
        if (this != &other) {
            delete ptr; // 释放当前对象指向的内存
            ptr = other.ptr; // 转移资源所有权
            other.ptr = nullptr; // 置空源对象的指针，防止重复释放
        }
        return *this;
    }

    T* release() { // 释放资源所有权
        T* temp = ptr;
        ptr = nullptr; // 置空指针，防止析构函数重复释放
        return temp;
    }

    T* get() const { // 获取指针
        return ptr;
    }

    T* operator->() const { // 重载箭头运算符
        return ptr;
    }

    T& operator*() const { // 解引用操作符
        return *ptr;
    }
};

int main() {
    unique_ptr<int> up(new int(42));
    std::cout << *up << std::endl; // 输出：42

    unique_ptr<int> up2 = std::move(up); // 移动构造
    std::cout << *up2 << std::endl; // 输出：42
    std::cout << (up.get() == nullptr) << std::endl; // 输出：1，up已经转移所有权，不再指向任何对象

    unique_ptr<int> up3(new int(100));
    up3 = std::move(up2); // 移动赋值
    std::cout << *up3 << std::endl; // 输出：42
    std::cout << (up2.get() == nullptr) << std::endl; // 输出：1，up2已经转移所有权，不再指向任何对象

    return 0;
}

```
运行结果：

```bash
g++ unique_ptr2.cxx -o unique_ptr2 -std=c++11
./unique_ptr2
42
42
1
42
1
```
可以看到，我们自己写的智能指针还能起点作用。

智能指针的选择取决于所需的所有权模型和内存管理策略。使用适当的智能指针可以大大简化代码，提高程序的可维护性和可靠性。

