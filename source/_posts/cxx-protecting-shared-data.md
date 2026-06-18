---
title: "C++ 并发编程：保护共享数据"
date: 2024-05-14 21:57:45
tags:
  - "C++"
  - "concurrency"
  - "shared data"
categories:
  - "Programming Languages"
---

<!--more-->

## 一、前言
本文将会通过保护一个数据讨论：互斥锁、双重检查锁、 `std::once_flag` 类、  `std::call_once()` 函数、单例模式、使用局部静态变量实现单例模式等。

## 二、保护共享数据
假设我们需要某个共享数据，而它创建起来开销不菲。因为创建它可能需要建立数据库连接或分配大量内存，所以等到必要时才真正着手创建。这种方式称为延迟初始化（lazy initialization），常见于单线程代码。**对于需要利用共享资源的每一项操作，要先在执行前判别该数据是否已经初始化，若没有，则及时初始化，然后方可使用。**

```cpp
std::shared_ptr<some_resource> resource_ptr;
void foo()
{
    if(!resource_ptr)
    {
        resource_ptr.reset(new some_resource);    ⇽---  ①
    }
    resource_ptr->do_something();
}
```
在多线程环境中，如上代码执行存在显著的线程安全问题。主要问题是在检查 `resource_ptr` 是否为空和可能重新赋值（即初始化）这两个操作之间存在一个竞态条件。

### （一） 问题分析
1. **竞态条件**：
   - 当多个线程同时调用 `foo()` 时，每个线程都会检查 `resource_ptr` 是否为 `nullptr`。
   - 如果两个（或多个）线程几乎同时到达检查点①，并且发现 `resource_ptr` 是空的，它们可能都会尝试创建 `new some_resource` 并设置 `resource_ptr`。
   - 这将导致多个 `some_resource` 实例被创建，而只有最后一个被创建的实例会被保留在 `resource_ptr` 中。这不仅浪费资源，还可能导致之前被创建实例的内存泄露。

2. **非原子操作**：
   - `if(!resource_ptr)` 到 `resource_ptr.reset(new some_resource)` 的执行不是原子操作。在多线程环境中，即使单个操作（如检查或赋值）是原子的，组合操作通常也不是原子的。

### （二） 解决方案
为了确保线程安全，你可以使用以下几种策略之一：

#### 1、使用互斥锁
在执行检查和初始化操作时使用互斥锁来保证这两个操作的原子性：

```cpp
#include <mutex>
std::mutex mtx;
std::shared_ptr<some_resource> resource_ptr;

void foo()
{
    {
        std::lock_guard<std::mutex> lock(mtx);
        if (!resource_ptr) {
            resource_ptr.reset(new some_resource);
        }
    }
    resource_ptr->do_something();
}
```
这里是关于多线程环境中使用 `std::lock_guard` 来保护 `std::shared_ptr` 初始化过程的一个重要限制。虽然 `std::lock_guard` 解决了多线程中数据竞争和资源初始化的线程安全问题，**但它引入了另一个潜在的性能问题，即线程在尝试获取锁时的阻塞和串行化**。

##### 问题详解
- **性能瓶颈**：当将资源的检查和初始化操作放在互斥锁保护的区域内，确保了线程安全，但这也意味着每次调用 `foo()` 函数时，即便 `resource_ptr` 已经被初始化，每个线程仍需依次等待获取锁以进入临界区域，进行资源的检查。**这导致了不必要的性能开销，特别是在资源初始化之后，多个线程频繁访问此函数时。**
- **锁的粒度**：锁的粒度太大，锁定的代码块包括了检查和初始化操作。理想情况下，只有初始化部分需要被严格保护以避免多次初始化。

##### 改进方案
要改善这种情况，可以使用“**双重检查锁定模式**”，这种模式可以减少锁的争用，提高程序的并发执行效率。但请注意，这种模式在C++中实现时需要特别小心，因为它涉及到内存模型和可能的编译器重排序，我们通常需要使用原子操作和/或内存屏障来正确实现。

#### 2、双重检查锁定模式示例
```cpp
#include <mutex>
#include <memory>

std::mutex mtx;
std::shared_ptr<some_resource> resource_ptr;

void foo()
{
    // 首先，不加锁地检查资源是否已初始化
    if (!resource_ptr) {
        std::lock_guard<std::mutex> lock(mtx);
        // 再次检查，确保资源未被初始化
        if (!resource_ptr) {
             resource_ptr.reset(new some_resource);
        }
    }
    local_ptr->do_something();
}
```

##### 代码解释
- **外层检查**：首先检查 `resource_ptr` 是否已经被初始化，这一检查是在没有加锁的情况下进行的，如果已经初始化，就直接使用资源，这样大多数情况下避免了锁的开销。
- **锁内检查**：如果初次检查指示 `resource_ptr` 未初始化，进入锁保护的区域后，**需要再次检查。这是因为在当前线程获取锁之前，可能已有其他线程初始化了资源。**

##### 总结
使用**双重检查锁定模式**可以在确保线程安全的同时，减少锁的争用和提高性能。然而，正确实现这种模式需要对C++内存模型有深入理解，避免由于编译器优化或CPU重排序引起的问题。在C++11及更高版本中，通过合理使用原子变量和内存序可以更安全地实现这种模式。

对于本例，这里依然有一个不容忽视的问题：尽管当前线程能够看见其他线程写入指针，却有可能无视新实例some_resource的创建，结果 `do_something()` 的调用就会对不正确的值进行操作。**C++标准将此例定义为数据竞争（data race）**，是条件竞争的一种，其将导致未定义行为，所以我们肯定要防范。

#### 4、 使用 `std::call_once` 和 `std::once_flag`

这种方法保证初始化代码只执行一次，即使在多线程环境中：

```cpp
#include <mutex>
std::once_flag flag;
std::shared_ptr<some_resource> resource_ptr;

void init_resource()
{
    resource_ptr.reset(new some_resource);
}

void foo()
{
    std::call_once(flag, init_resource);
    resource_ptr->do_something();
}
```

`std::call_once()` 是 C++11 引入的一个函数，用于确保某个函数只被调用一次，即使在多线程环境中也是如此。这通常用于资源或服务的懒惰初始化。`std::call_once()` 通常与 `std::once_flag` 配合使用，后者用来跟踪函数是否已经被调用。

##### 工作原理
`std::call_once()` 保证无论多少线程尝试调用指定的可调用对象，该对象的调用只会执行一次。它通过 `std::once_flag` 来控制，**这个标志协调不同线程对函数调用的访问，确保目标函数只执行一次。**

##### 应用场景
1. **系统级资源初始化**：适用于需要确保全局或静态资源只初始化一次的场景。
2. **一次性配置读取**：适用于配置数据或环境设置，这些数据在程序运行期间不应该改变，只需要加载一次。
3. **单例模式**：在创建单例对象时确保构造函数只被调用一次。

##### 示例代码
下面是一些使用 `std::call_once()` 的示例代码。

###### 示例1: 懒惰初始化
```cpp
#include <iostream>
#include <mutex>
#include <thread>

std::once_flag flag;
int config;

void init_config() {
    config = 42;  // 假设这是从文件或数据库加载的配置
    std::cout << "Configuration initialized to " << config << std::endl;
}

void process(int id) {
    std::call_once(flag, init_config);
    std::cout << "Thread " << id << " sees configuration as: " << config << std::endl;
}

int main() {
    std::thread threads[5];
    for (int i = 0; i < 5; ++i) {
        threads[i] = std::thread(process, i);
    }
    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```
以上代码，虽然有 5 个线程在尝试执行 `process()` ，但不管如何，只有一个线程能运行`init_config()`。

###### 示例2: 单例模式
单例模式确保一个类只有一个实例，并提供一个访问它的全局访问点。
```cpp
#include <iostream>
#include <mutex>
#include <memory>

class Singleton {
private:
    static std::unique_ptr<Singleton> instance;
    static std::once_flag onceFlag;
public:
	Singleton() = default;
    ~Singleton() = default;
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;

    static Singleton* getInstance() {
        std::call_once(onceFlag, [] {
            instance.reset(new Singleton);
        });
        return instance.get();
    }

    void doSomething() {
        std::cout << "Doing something" << std::endl;
    }
};

std::unique_ptr<Singleton> Singleton::instance;
std::once_flag Singleton::onceFlag;

int main() {
    Singleton::getInstance()->doSomething();
    return 0;
}
```

在这些示例中，`std::call_once()` 确保了无论多少个线程试图执行初始化代码或创建单例实例，初始化逻辑只执行一次，从而避免了资源浪费和潜在的数据竞争问题。这种机制是线程安全的，为多线程程序提供了一种简单而有效的同步解决方案。

对于单例模式，我们还有一种方法，就是 C++11的规定：**这个特性确保了函数局部静态变量的线程安全初始化，即使在并发执行的多线程环境中，这个变量也只会被初始化一次，并且所有其他线程都将等待这个初始化过程完成才继续执行**。
## 三、C++11 前的情况与改进

在 C++11 之前，静态局部变量的线程安全初始化不是由语言标准保证的。如果多个线程同时首次调用一个包含静态局部变量的函数，可能会引起条件竞争，从而导致变量被多次初始化或者在完全初始化之前被另一个线程使用。

C++11 标准通过规定，**静态局部变量的初始化将在第一次遇到变量定义时原子性地进行，确保了这种初始化的线程安全性**。这意味着如果有多个线程同时到达初始化语句，**只有一个线程会执行初始化，其他线程将会阻塞，直到初始化完成**。

### （一）应用示例

这个特性特别适用于实现单例模式。单例模式要求一个类只有一个实例，并且提供一个全局的访问点。使用 C++11 的特性，**我们可以安全地用局部静态变量实现单例，而不需要额外的锁或其他同步机制**。

#### 示例：使用局部静态变量实现单例模式

```cpp
#include <iostream>

class Singleton {
public:
    // 提供一个获取单例实例的方法
    static Singleton& getInstance() {
        static Singleton instance;  // 局部静态变量
        return instance;
    }

    void doSomething() {
        std::cout << "Doing something" << std::endl;
    }

private:
    // 私有构造和析构防止外部创建和删除实例
    Singleton() {}
    ~Singleton() {}
    // 禁止复制和赋值
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
};

int main() {
    // 获取单例实例并使用
    Singleton& singleton = Singleton::getInstance();
    singleton.doSomething();

    return 0;
}
```

在这个例子中：
- `Singleton` 类的构造函数被设为私有，防止外部直接创建实例。
- `getInstance()` 方法中定义了一个静态局部变量 `instance`。根据 C++11 的规定，这个实例的创建是线程安全的，无论多少线程同时调用这个方法，实例只会被创建一次。
- 任何需要使用单例的代码都可以通过 `Singleton::getInstance()` 来安全地访问单例实例，无需担心并发环境下的线程安全问题。

**这种方式比使用 `std::call_once` 更简洁，因为它无需显式定义 `std::once_flag` 或编写额外的初始化函数。** 编译器自动为我们处理了所有的线程安全细节。


## 四、 递归加锁

在C++中，递归锁（也称为可重入锁）是一种特殊类型的互斥锁，它允许同一个线程多次对同一把锁进行加锁。与普通的互斥锁不同，如果一个线程已经持有了锁，它仍可以再次请求这个锁而不会导致死锁。

### 为什么要引入递归锁？

递归锁的主要应用场景是在复杂的应用程序中，其中某个函数可能会被多个其他函数调用，这些函数又直接或间接地调用了原始函数。如果使用普通锁，在这种情况下可能会导致死锁，因为一个线程试图多次获取同一资源的独占访问权。

### 应用场景

递归锁通常用于以下几种情况：
1. **递归函数**：如果递归函数中需要保护共享资源，使用递归锁可以防止在递归过程中发生死锁。
2. **类的成员函数**：当一个类的成员函数需要调用另一个需要同一把锁的成员函数时，递归锁可以简化编程。
3. **复杂控制流**：在复杂的控制流程中，一个锁可能在多个地方需要被重复获取，递归锁可以避免死锁的风险。

### 示例

下面的示例展示了如何在C++中使用`std::recursive_mutex`来实现递归锁：

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::recursive_mutex rec_mtx;

void recursive_function(int n) {
    if (n <= 0) return;

    rec_mtx.lock();
    std::cout << "Lock level " << n << std::endl;
    recursive_function(n-1);
    std::cout << "Unlock level " << n << std::endl;
    rec_mtx.unlock();
}

int main() {
    std::thread t(recursive_function, 3);
    t.join();
    return 0;
}
```

### 代码解释

- 这个例子中，我们定义了一个递归函数`recursive_function`，它接受一个整数`n`作为参数。
- 函数中使用了`std::recursive_mutex`类型的`rec_mtx`。在递归调用前后分别进行锁定和解锁操作。
- 由于`std::recursive_mutex`的特性，即使在同一线程中多次锁定，也不会导致死锁。

这种情况下使用普通的互斥锁将会导致线程在第二次尝试锁定互斥量时死锁，因为普通互斥锁不允许单个线程多次锁定。通过使用递归锁，线程可以安全地多次进入临界区，只要确保每次加锁都有对应的解锁操作。

递归锁的缺点也很明显，它会导致串行操作：

```cpp
#include <iostream>
#include <mutex>
#include <thread>

std::recursive_mutex rec_mtx;
int Num[5] = {0, 1, 2, 3, 4};

void recursive_function_update(int n) {
    if (n <= 0)
        return;

    rec_mtx.lock();
    std::cout << "updating: " << n << std::endl;
    Num[n] = Num[n] * Num[n];
    recursive_function_update(n - 1);
    std::cout << "Unlock level " << n << std::endl;
    rec_mtx.unlock();
}

void recursive_function_read(int n) {
    if (n <= 0)
        return;

    rec_mtx.lock();
    std::cout << "reading: " << n << std::endl;
    std::cout << Num[n] << std::endl;
    recursive_function_read(n - 1);
    std::cout << "Unlock level " << n << std::endl;
    rec_mtx.unlock();
}

int main() {
    std::thread u(recursive_function_update, 3);
    std::thread u1(recursive_function_update, 3);
    std::thread r(recursive_function_read, 3);
    std::thread r1(recursive_function_read, 3);

    u.join();
    u1.join();
    r.join();
    r1.join();
    return 0;
}

```

运行结果：

```bash
./main
updating: 3
updating: 2
updating: 1
Unlock level 1
Unlock level 2
Unlock level 3
updating: 3
updating: 2
updating: 1
Unlock level 1
Unlock level 2
Unlock level 3
reading: 3
81
reading: 2
16
reading: 1
1
Unlock level 1
Unlock level 2
Unlock level 3
reading: 3
81
reading: 2
16
reading: 1
1
Unlock level 1
Unlock level 2
Unlock level 3
```

尽管 `std::recursive_mutex` 允许单个线程多次获得锁，但它并不允许多个线程同时持有锁。这**意味着在多线程环境中，其他线程仍需等待当前持有锁的线程完全释放锁**，才能继续执行，这可能导致执行的串行化，减少了多线程的效益。

## 五、参考
《C++并发编程实战》（第二版）。
