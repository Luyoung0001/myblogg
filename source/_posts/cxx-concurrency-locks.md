---
title: "C++ 并发编程：锁、死锁与 std::lock"
date: 2024-05-14 17:38:26
tags:
  - "C++"
  - "concurrency"
  - "mutex"
categories:
  - "Programming Languages"
---

<!--more-->

## 一、前言
C++中的锁和同步原语的多样化选择使得程序员可以根据具体的线程和数据保护需求来选择最合适的工具。这些工具的正确使用可以大大提高程序的稳定性和性能，本文讨论了部分锁。
## 二、std::lock

在C++中，`std::lock` 是一个用于一次性锁定两个或多个互斥量（mutexes）的函数，而且还保证不会发生死锁。**这是通过采用一种称为“死锁避免算法”的技术来实现的，该技术能够保证多个互斥量按照一定的顺序加锁**。

### 使用场景
当需要同时锁定多个互斥量，而且希望避免因为锁定顺序不一致而引起死锁时，使用`std::lock` 是非常合适的。它通常与 `std::unique_lock` 或 `std::lock_guard` 配合使用，以提供灵活的锁定管理或自动锁定和解锁功能。

### 基本用法
以下是`std::lock`的一个基本示例，展示如何使用它来安全地锁定两个互斥量：

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx1, mtx2;

void process_data() {
    // 使用std::lock来同时锁定两个互斥量
    std::lock(mtx1, mtx2);
    
    // 确保两个互斥量都已锁定，使用std::lock_guard进行管理，不指定std::adopt_lock参数
    std::lock_guard<std::mutex> lk1(mtx1, std::adopt_lock);
    std::lock_guard<std::mutex> lk2(mtx2, std::adopt_lock);
    
    // 执行一些操作
    std::cout << "Processing shared data." << std::endl;
}

int main() {
    std::thread t1(process_data);
    std::thread t2(process_data);
    
    t1.join();
    t2.join();
    
    return 0;
}
```

### 说明
1. **std::lock**：这个函数尝试锁定所有提供的互斥量，不返回直到所有的互斥量都成功锁定。它使用一个特殊的锁定算法来避免死锁。
   
2. **std::lock_guard**：此范例中用 `std::lock_guard` 来自动管理互斥量的锁定状态。由于互斥量已经被 `std::lock` 锁定，所以我们使用 `std::adopt_lock` 标记，告诉 `std::lock_guard` 对象互斥量已经被锁定，并且在 `std::lock_guard` 的生命周期结束时释放它们。

3. **std::adopt_lock**：这是一个构造参数，告诉 `std::lock_guard` 或 `std::unique_lock` 对象该互斥量已经被当前线程锁定了，**对象不应该尝试再次锁定互斥量，而是在析构时解锁它**。

通过使用 `std::adopt_lock` 参数，正确地指示了 `std::lock_guard` 对象（在这个例子中是 `lk1` 和 `lk2`），互斥量已经被当前线程锁定。**这样，`std::lock_guard` 不会在构造时尝试锁定互斥量，而是会在其析构函数中释放它们。**

这意味着，当 `lk1` 和 `lk2` 的作用域结束时（例如，当 `process_data` 函数执行完毕时），`lk1` 会自动释放 `mtx1`，`lk2` 会自动释放 `mtx2`。这是 `std::lock_guard` 的典型用法，通过在构造时获取锁并在析构时释放锁，它提供了一种方便的资源管理方式，这种方式常被称为 **RAII**（Resource Acquisition Is Initialization）。

## 三、std::lock_guard
上面的实例中已经用到了 `std::lock_guard`，主要是想利用它的 **RAII** 特性。下面详细介绍 `std::lock_guard`。

`std::lock_guard` 是 C++ 中一个非常有用的同步原语，用于在作用域内自动管理互斥量的锁定和解锁。它是一个模板类，提供了一种方便的方式来实现作用域内的锁定保护，确保在任何退出路径（包括异常退出）上都能释放锁，从而帮助避免死锁。

### 基本用法
`std::lock_guard` 的基本用法很简单：在需要保护的代码块前创建一个 `std::lock_guard` 对象，将互斥量作为参数传递给它。`std::lock_guard` 会在构造时自动锁定互斥量，在其析构函数中自动解锁互斥量。

### 示例代码
这里是一个使用 `std::lock_guard` 的简单示例：

```cpp
#include <iostream>
#include <mutex>
#include <thread>

std::mutex mtx;  // 全局互斥量

void print_data(const std::string& data) {
    std::lock_guard<std::mutex> guard(mtx);  // 创建时自动锁定mtx
    // 以下代码在互斥锁保护下执行
    std::cout << data << std::endl;
    // guard 在离开作用域时自动解锁mtx
}

int main() {
    std::thread t1(print_data, "Hello from Thread 1");
    std::thread t2(print_data, "Hello from Thread 2");

    t1.join();
    t2.join();

    return 0;
}
```

### 说明
1. **自动锁定与解锁**：在 `print_data` 函数中，`std::lock_guard` 的实例 `guard` 在创建时自动对 `mtx` 进行锁定，并在函数结束时（`guard` 的生命周期结束时）自动对 `mtx` 进行解锁。这确保了即使在发生异常的情况下也能释放锁，从而防止死锁。
   
2. **作用域控制**：`std::lock_guard` 的作用范围限制于它被定义的代码块内。一旦代码块执行完毕，`std::lock_guard` 会被销毁，互斥量会被自动释放。

3. **不支持手动控制**：与 `std::unique_lock` 不同，`std::lock_guard` 不提供锁的手动控制（如调用 `lock()` 和 `unlock()`）。**它仅在构造时自动加锁，在析构时自动解锁。**

通过使用 `std::lock_guard`，你可以确保即使面对多个返回路径和异常，互斥锁的管理也是安全的，从而简化多线程代码的编写。这使得 `std::lock_guard` 成为处理互斥量时的首选工具之一，尤其是在简单的锁定场景中。

## 四、std::unique_lock
`std::unique_lock` 是 C++ 标准库中的一个灵活的同步工具，用于管理互斥量（mutex）。与 `std::lock_guard` 相比，`std::unique_lock` 提供了更多的控制能力，包括延迟锁定、尝试锁定、条件变量支持和手动锁定与解锁的能力。这使得 `std::unique_lock` 在需要复杂锁定逻辑的情况下非常有用。

### 基本用法
`std::unique_lock` 的基本用法包括自动管理互斥量的锁定和解锁，但它也支持手动操作和条件变量。

### 示例代码
下面是一些展示 `std::unique_lock` 使用方式的示例：

#### 基本的自动锁定与解锁
```cpp
#include <iostream>
#include <mutex>
#include <thread>

std::mutex mtx;  // 全局互斥量

void print_data(const std::string& data) {
    std::unique_lock<std::mutex> lock(mtx);  // 在构造时自动锁定mtx
    std::cout << data << std::endl;
    // lock 在离开作用域时自动解锁mtx
}

int main() {
    std::thread t1(print_data, "Thread 1");
    std::thread t2(print_data, "Thread 2");

    t1.join();
    t2.join();

    return 0;
}
```

#### 延迟锁定
`std::unique_lock` 允许延迟锁定，即创建锁对象时不立即锁定互斥量。

```cpp
void delayed_lock_example() {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock);  // 创建时不锁定
    // 进行一些不需要互斥量保护的操作
    lock.lock();  // 现在需要锁定
    std::cout << "Locked and safe" << std::endl;
    // lock 在离开作用域时自动解锁mtx
}
```

#### 手动控制锁定与解锁
`std::unique_lock` 提供了 `lock()` 和 `unlock()` 方法，允许在其生命周期内多次锁定和解锁。

```cpp
void manual_lock_control() {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock);
    // 决定什么时候锁定
    lock.lock();
    std::cout << "Processing data" << std::endl;
    lock.unlock();
    // 可以再次锁定
    lock.lock();
    std::cout << "Processing more data" << std::endl;
    // lock 在离开作用域时自动解锁mtx
}
```

#### 与条件变量结合使用
`std::unique_lock` 通常与条件变量一起使用，因为它支持在等待期间解锁和重新锁定。

```cpp
std::condition_variable cv;
bool data_ready = false;

void data_preparation_thread() {
    {
        std::unique_lock<std::mutex> lock(mtx);
        // 准备数据
        data_ready = true;
    }
    cv.notify_one();  // 通知等待线程
}

void data_processing_thread() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return data_ready; });  // 等待数据准备好
    // 处理数据
    std::cout << "Data processed" << std::endl;
}
```

#### 转移互斥归属权到函数调用者
转移有一种用途：准许函数锁定互斥，然后把互斥的归属权转移给函数调用者，好让它在同一个锁的保护下执行其他操作。下面的代码片段就此做了示范：`get_lock()` 函数先锁定互斥，接着对数据做前期准备，再将归属权返回给调用者：

```cpp
std::unique_lock<std::mutex> get_lock()
{
    extern std::mutex some_mutex;
    std::unique_lock<std::mutex> lk(some_mutex);
    prepare_data();
    return lk;    ⇽---  ①
}
void process_data()
{
    std::unique_lock<std::mutex> lk(get_lock());    ⇽---  ②
    do_something();
}
```
 ①处通过移动构造创建返回值，该值为右值。然后右值在②处移动构造 `lk` 。我们关注的是，这里的 `std::unique_lock` 的移动语义特性。这使得 `std::unique_lock` 对象可以在函数或其他作用域之间传递互斥体的所有权，而不是仅仅通过复制来共享所有权。这一点尤其重要，因为 `std::unique_lock` 管理的互斥体锁定状态需要保持一致性和独占性，复制操作会破坏这一点。
 
 `std::unique_lock`类十分灵活，允许它的实例在被销毁前解锁。其成员函数 `unlock()` 负责解锁操作，这与互斥一致。




## 五、std::scoped_lock（C++17）
前面的实例中，有些复杂，我们可以使用更简单的 `std::scoped_lock`。因为它自动处理了多个互斥量的锁定和解锁，而不需要显式指定 `std::adopt_lock`。C++17提供了新的**RAII**类模板`std::scoped_lock<>`。它封装了多互斥体的锁定功能，确保无死锁，且使用方便。

`std::scoped_lock` 自动锁定其构造函数中传递的所有互斥体，并在作用域结束时释放它们，因此非常适合用于替代 `std::lock` 加 `std::lock_guard` 的组合使用。

### 示例
以下是一个使用 `std::scoped_lock` 的例子，处理两个互斥量：

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx1, mtx2;

void process_shared_data() {
    // 使用std::scoped_lock同时锁定两个互斥量
    std::scoped_lock lock(mtx1, mtx2); //<------①

    // 执行一些操作
    std::cout << "Processing shared data safely." << std::endl;
}

int main() {
    std::thread t1(process_shared_data);
    std::thread t2(process_shared_data);

    t1.join();
    t2.join();

    return 0;
}
```

### 说明
在这个例子中：
1. **std::scoped_lock**: 构造时自动锁定传递给它的所有互斥量（在这里是 `mtx1` 和 `mtx2`）。这样的锁定是原子的，这意味着它使用死锁避免算法来避免在尝试锁定多个互斥量时可能发生的死锁问题。
2. **自动解锁**：当 `std::scoped_lock` 的实例 `lock` 的作用域结束时，它自动以安全的顺序释放所有互斥体。这在函数 `process_shared_data` 结束时发生。
3. **简洁性和安全性**：与 `std::lock` 和 `std::lock_guard` 结合使用相比，`std::scoped_lock` 更简洁且不易出错，因为不需要使用 `std::adopt_lock` 或担心锁定的顺序。


C++17具有隐式类模板参数推导（implicit class template parameter deduction）机制，依据传入构造函数的参数对象自动匹配，选择正确的互斥型别。①处的语句等价于下面完整写明的版本：

```cpp
std::scoped_lock<std::mutex,std::mutex> lock(mtx1, mtx2);
```
## 六、std::shared_mutex（C++17）
 `std::shared_mutex` 的工作原理和使用模式，特别是在涉及共享锁（读锁）和排他锁（写锁）的情形。这种机制被设计用于处理读多写少的场景，允许多个读取者同时访问数据，但保证写入者有独占访问权。

### 读写锁的工作原理：
- **共享锁（读锁）**：多个线程可以同时持有共享锁。当共享锁被持有时，其他线程仍然可以获取共享锁，但不能获取排他锁。
- **排他锁（写锁）**：当排他锁被一个线程持有时，其他线程不能获取共享锁也不能获取排他锁。排他锁确保锁的持有者可以安全地写数据，无需担心数据一致性问题。

### 示例
下面的示例展示了如何在实践中使用 `std::shared_mutex` 和 `std::shared_lock`（用于共享锁）以及 `std::unique_lock`（用于排他锁）：

```cpp
#include <iostream>
#include <shared_mutex>
#include <thread>
#include <vector>

std::shared_mutex rw_mutex;
int data = 0;  // 示例共享数据

void reader(int id) {
    std::shared_lock<std::shared_mutex> lock(rw_mutex);
    // 多个读取者可以同时执行下面的代码
    std::cout << "Reader " << id << " sees data = " << data << std::endl;
    // shared_lock 在离开作用域时自动释放
}

void writer(int id, int value) {
    std::unique_lock<std::shared_mutex> lock(rw_mutex);
    // 只有一个写入者可以执行下面的代码
    data = value;
    std::cout << "Writer " << id << " updated data to " << data << std::endl;
    // unique_lock 在离开作用域时自动释放
}

int main() {
    std::thread readers[5];
    std::thread writers[2];

    // 启动读取者线程
    for (int i = 0; i < 5; ++i) {
        readers[i] = std::thread(reader, i);
    }

    // 启动写入者线程
    writers[0] = std::thread(writer, 0, 100);
    writers[1] = std::thread(writer, 1, 200);

    // 等待所有线程完成
    for (int i = 0; i < 5; ++i) {
        readers[i].join();
    }
    for (int i = 0; i < 2; ++i) {
        writers[i].join();
    }

    return 0;
}
```
运行结果：

```bash
./main
Reader 0 sees data = 0
Reader 2 sees data = 0
Reader 4 sees data = 0
Reader 1 sees data = 0
Writer 0 updated data to 100
Writer 1 updated data to 200
Reader 3 sees data = 200
```

### 解释
1. **共享锁的行为**：当一个或多个读者线程通过 `std::shared_lock` 持有共享锁时，它们可以安全地读取共享数据。此时，如果另一个线程尝试通过 `std::unique_lock` 获取排他锁以写入数据，它将会阻塞，直到所有共享锁被释放。
2. **排他锁的行为**：当一个写者线程持有排他锁时，任何其他试图通过 `std::shared_lock` 或 `std::unique_lock` 获取锁的线程都将阻塞，直到排他锁被释放。这确保写入操作的独占性和安全性。

这种锁的设计非常适合数据读取远多于数据修改的应用场景，可以显著提高并发性和性能。


## 七、防范死锁的补充准则
防范死锁的准则最终可归纳成一个思想：**只要另一线程有可能正在等待当前线程，那么当前线程千万不能反过来等待它。**
- 第一条准则最简单：假如已经持有锁，就不要试图获取第二个锁。
- 一旦持锁，就须避免调用由用户提供的程序接口。
- 依从固定顺序获取锁。
- 按照层接加锁。


### 按照层级加锁
这一块儿比较重要，需要展开讨论。思路是，我们把应用程序分层，并且明确每个互斥位于哪个层级。若某线程已对低层级互斥加锁，则不准它再对高层级互斥加锁。以下伪代码示范了两个线程如何运用层级互斥：

```cpp
hierarchical_mutex high_level_mutex(10000);    ⇽---  ①
hierarchical_mutex low_level_mutex(5000);    ⇽---  ②
hierarchical_mutex other_mutex(6000);    ⇽---  ③
int do_low_level_stuff();
int low_level_func()
{
    std::lock_guard<hierarchical_mutex> lk(low_level_mutex);    ⇽---  ④
    return do_low_level_stuff();
}
void high_level_stuff(int some_param);
void high_level_func()
{
    std::lock_guard<hierarchical_mutex> lk(high_level_mutex);    ⇽---  ⑥
    high_level_stuff(low_level_func());    ⇽---  ⑤
}
void thread_a()    ⇽---  ⑦
{
    high_level_func();
}

void do_other_stuff();
void other_stuff()
{
    high_level_func();    ⇽---  ⑩
    do_other_stuff();
}
void thread_b()    ⇽---  ⑧
{
    std::lock_guard<hierarchical_mutex> lk(other_mutex);    ⇽---  ⑨
    other_stuff();
}
```
显然，⑧处的代码不符合规范，因为目前持有的锁是 `other_mutex`，其标号是 6000，而底层调用的代码 `other_stuff()` 中却持有了一个 `high_level_mutex`，其标号为 10000。这没有遵守底层调用持有底层锁，`hierarchical_mutex`会抛出异常。

## 八、参考
《C++并发编程实战》（第二版）。





