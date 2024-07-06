---
title: C++：再谈智能指针
date: 2024-05-18 22:22:57
tags: c++
categories: C++
---

<!--more-->

## 〇、前言
本文会讨论 `shared_ptr`、`weak_ptr`、`unique_ptr`以及智能指针的相关注意事项。
## 一、具体实现
关于 `shared_ptr`、`weak_ptr`，虽然具体的实现细节可能根据不同的编译器和版本有所不同，下面仅仅重点介绍了 `std::shared_ptr` 和 `std::weak_ptr` 析构函数的伪代码，这些代码将展示它们如何管理资源和引用计数。这些代码不是任何特定标准库实现的直接摘录，而是为了说明这些智能指针背后的一般机制。

### `std::shared_ptr` 析构函数

```cpp
template<typename T>
class shared_ptr {
private:
    T* ptr;             // 指向被管理的对象
    long* ref_count;    // 指向引用计数
    long* weak_count;   // 指向弱引用计数

public:
    ~shared_ptr() {
        if (ptr && --(*ref_count) == 0) {
            delete ptr;         // 销毁对象
            if (*weak_count == 0) {
                delete ref_count;   // 如果没有weak_ptr观察对象，则删除计数器
                delete weak_count;
            }
        }
    }
};
```

### `std::weak_ptr` 析构函数

```cpp
template<typename T>
class weak_ptr {
private:
    T* ptr;             // 指向被管理的对象
    long* ref_count;    // 指向引用计数
    long* weak_count;   // 指向弱引用计数

public:
    ~weak_ptr() {
        if (--(*weak_count) == 0 && *ref_count == 0) {
            delete ref_count;   // 如果没有shared_ptr和其他weak_ptr，则删除计数器
            delete weak_count;
        }
    }
};
```
### 解释
1. **`shared_ptr` 析构函数**：
    - 当 `shared_ptr` 实例析构时，它首先减少引用计数。
    - 如果引用计数达到0，说明没有其他 `shared_ptr` 实例指向该对象，因此对象被销毁。
    - 然后检查弱引用计数，如果没有 `weak_ptr` 实例观察这个对象（弱引用计数也为0），则控制块（包含计数器）也会被销毁。

2. **`weak_ptr` 析构函数**：
    - 当 `weak_ptr` 实例析构时，它减少弱引用计数。
    - 如果弱引用计数和引用计数都为0，说明没有任何 `shared_ptr` 或 `weak_ptr` 实例指向该对象或观察该对象，因此控制块被销毁。

## 二、循环引用

```cpp
#include <iostream>
#include <memory>
using namespace std;

class B; // 前置声明

class A {
  public:
    shared_ptr<B> b_;
    A() { cout << "A constructed!" << endl; }
    ~A() { cout << "A destructed!" << endl; }
};

class B {
  public:
    shared_ptr<A> a_;
    B() { cout << "B constructed!" << endl; }
    ~B() { cout << "B destructed!" << endl; }
};

int main() {
    auto classA = make_shared<A>();
    auto classB = make_shared<B>();
    classA->b_ = classB;
    classB->a_ = classA;
    cout << "A: " << classA.use_count() << endl;
    cout << "B: " << classB.use_count() << endl;
    return 0;
}
```

运行结果：

```bash
g++ loopP.cxx -o main -std=c++11
./main
A constructed!
B constructed!
A: 2
B: 2
```
可以看到，A、B 对象并没有被析构掉，这就造成了内存泄漏。需要注意的是：**智能指针本身的析构和智能指针管理的对象的析构。**

### 智能指针的析构

智能指针如 `std::shared_ptr` 在离开作用域时会被自动析构。这意味着智能指针作为一个对象的实例会被销毁，相关的操作包括减少其所管理对象的引用计数。

### 智能指针管理的对象的析构

智能指针管理的对象是否被析构，取决于其引用计数是否达到零：
- 如果引用计数为零，表示没有任何 `shared_ptr` 实例正在管理这个对象，对象将被析构。
- 如果引用计数不为零，即使智能指针实例被销毁，对象本身不会被析构。

### 循环引用情况

在循环引用的情形中（如示例中 `A` 和 `B` 相互持有），尽管每个 `shared_ptr` 实例（如 `classA` 和 `classB`）在 `main` 函数结束时会被析构，它们的引用计数会相应减少，但不会归零。因为每个对象（`A` 的实例和 `B` 的实例）仍被另一个对象通过 `shared_ptr` 持有。

### 析构过程
当 `classB` 析构的时候，它会将把引用计数减一，但是引用计数为 1，因此 `classB` 指向的 `B` 对象并没有析构；当 `classA` 析构的时候，它同样会将把引用计数减一，但是引用计数为 1。因此 `classA `指向的 `A` 对象并没有析构。`classA`、`classB` 指针已经被析构掉了，智能指针如 `std::shared_ptr` 在离开作用域时会被自动析构。

## 三、 用 unique_ptr 解决循环引用

```cpp
#include <iostream>
#include <memory>
using namespace std;

class B;

class A {
  public:
    shared_ptr<B> b_;
    A() { cout << "A constructed!" << endl; }
    ~A() { cout << "A destructed!" << endl; }
};

class B {
  public:
    weak_ptr<A> a_;
    B() { cout << "B constructed!" << endl; }
    ~B() { cout << "B destructed!" << endl; }
};

int main() {
    auto classA = make_shared<A>();
    auto classB = make_shared<B>();
    classA->b_ = classB;
    classB->a_ = classA;
    cout << "A: " << classA.use_count() << endl;
    cout << "B: " << classB.use_count() << endl;
    return 0;
}
```
运行结果：

```bash
g++ unloopP.cxx -o main -std=c++11
./main
A constructed!
B constructed!
A: 1
B: 2
A destructed!
B destructed!
```
可以看到，A、B对象都被析构掉了。
### 析构过程
这个析构过程就很清晰了。当 `classB`析构的时候，`B` 对象引用计数减一，对象析构失败；当 `classA` 析构的时候，`A` 对象析构成功，然后开始析构成员 `shared_ptr<B> b_`，这导致 `b_`被析构，然后 `b_`指向的 `B` 对象引用计数减一，这时候减到了 0，`B` 对象此时被析构，整个析构过程完毕。

## 四、指针创建
使用 `std::make_shared` 和 `std::make_unique` 被广泛推荐的原因包括性能优势、安全性以及代码的简洁性。这些函数不仅减少了代码冗余，还提供了更好的异常安全性和潜在的性能提升：

### 1. 性能优势

当使用 `std::make_shared` 或 `std::make_unique` 时，对象和它的控制块（对于 `shared_ptr`，控制块包括引用计数和弱引用计数）是在单个内存分配中创建的。这与直接使用 `new` 操作符并将结果赋给一个智能指针不同，后者通常需要两次内存分配（一次为对象，一次为控制块）。

#### 对于 `make_shared`：
- **单次内存分配**：`std::make_shared` 通过单次内存分配来同时创建对象和其元数据（控制块），这减少了内存使用的开销和分配时间。

#### 对于 `make_unique`：
- **直接封装**：虽然 `std::make_unique` 的性能优势不像 `make_shared` 那样显著（因为 `unique_ptr` 不需要控制块），它仍然推荐使用，因为它提供了更好的语法一致性和异常安全性。

### 2. 异常安全性

在复杂表达式中，如果使用 `new` 显式分配内存，且在传递给智能指针之前发生异常，则可能导致内存泄漏。使用 `std::make_shared` 和 `std::make_unique`，分配是封装的，且对象生命周期由智能指针自动管理，**因此即便在构造函数中抛出异常，也不会泄漏资源。**

### 3. 代码简洁性

`std::make_unique` 和 `std::make_shared` 函数模板可以自动推导出对象的类型，从而减少了代码中的类型重复，并使代码更加简洁明了。

### 性能比较

对比：
- 使用 `std::unique_ptr<Widget> upw2(new Widget);` 时，首先分配 Widget 对象，然后创建 `std::unique_ptr` 对象并将其指向新分配的 Widget。
- 使用 `auto upw1(std::make_unique<Widget>());` 时，这个调用直接构造一个 Widget 对象，并封装进 `unique_ptr`，这通常在同一操作中完成，更加高效和安全。

尽管 `std::make_unique` 主要提供的是编码方便性和异常安全，而不是像 `std::make_shared` 那样的性能优势（因为 `make_shared` 减少了内存分配次数），它仍然是创建和使用 `unique_ptr` 的推荐方式，因为它使得代码更简洁、更安全。

## 五、内存分配

`std::make_shared` 通过单次内存分配来同时创建对象和它的控制块，而直接使用 `std::shared_ptr` 构造函数则需要两次内存分配。这不仅提高了内存分配的效率，还增强了程序的性能，特别是在对象频繁创建和销毁的场景中。

- **缓存友好性**：`std::make_shared` 创建的对象和控制块在**内存中是连续存储的，这有助于提高 CPU 缓存的效率**，因为对象数据和其元数据（如引用计数）通常会同时访问。

### 异常安全

- **异常安全保证**：使用 `std::make_shared`，如果对象的构造过程中抛出异常，**已经分配的内存会在同一操作中自动释放，从而避免内存泄漏。** 这是因为整个操作是原子性的——要么成功创建对象并返回智能指针，要么在失败时释放所有资源。

### 对于大型对象或频繁复制的场景

- **内存占用**：对于大型对象，`std::make_shared` 由于使用单次内存分配，可能导致即使 `shared_ptr` 都已销毁，控制块（因为可能存在 `weak_ptr`）仍占用内存。这**可能导致大块内存较晚释放。**
- **内存释放策略**：直接使用 `std::shared_ptr` 构造函数可以使得对象和控制块分别管理，允许对象在不再有 `shared_ptr` 指向它时立即释放，这对于管理大型资源可能更加有效。

这个可以从 `shared_ptr` 的析构函数看到：

```cpp
~shared_ptr() {
        if (ptr && --(*ref_count) == 0) {
            delete ptr;         // 销毁对象
            if (*weak_count == 0) {
                delete ref_count;   // 如果没有weak_ptr观察对象，则删除计数器
                delete weak_count;
            }
        }
    }
```


### 实际应用建议

- **选择 `std::make_shared`**：通常情况下，推荐使用 `std::make_shared`，因为它提供了更好的性能，更高的异常安全性，并减少了代码复杂度。
- **考虑使用直接构造 `std::shared_ptr`**：在涉及非常大的对象或者资源占用敏感的场景中，直接使用 `std::shared_ptr` 构造函数可能更合适，以便能更快地释放资源。

`make_share` 虽然效率高，但是同样不能自定义析构器，同时 `share_ptr` 的对象资源**可能会延迟释放，因为此时对象资源与管理区域在同一块内存中，必须要同时释放。**


## 六、一个场景
从表面上看，调用 `std::unique_ptr<T>(new T(std::forward<Args>(args)...))` 和直接使用 `std::unique_ptr<Widget>(new Widget)` 构造智能指针看起来非常相似。确实，在功能上它们执行相同的基本操作：分配一个新的对象并将其封装在一个 `std::unique_ptr` 中。不过，使用 `std::make_unique` 和直接使用 `new` 操作符有几个关键的区别，主要涉及到代码的安全性和现代 C++ 的最佳实践。

### 1. 异常安全性
使用 `std::make_unique` 提高了代码的异常安全性。考虑下面的例子，在一个复杂的表达式中，例如函数调用：

```cpp
F(std::unique_ptr<string> &u1, std::unique_ptr<string> &u2){}
```
如果有以下调用过程：
```cpp
F(std::unique_ptr<string>(new string("foo")),
std::unique_ptr<string>(new string("bar")));
```
C++未定义求参顺序，但是可能有以下过程：

```cpp
string("foo");
string("bar");
unique_ptr<string>; // 第三步
unique_ptr<string>;
```
如果进行到了第三步抛出异常（比如内存不足），那么一、二步产生的对象就会悬空，没有指针来指向，从而导致内存泄漏。这个问题的核心在于, `unique_ptr` 没有立即获得裸指针。

使用 `std::make_unique`，**每个对象的创建和其封装到智能指针中是一个原子操作，避免了潜在的资源泄漏**：

```cpp
someFunction(std::make_unique<string>("foo"), std::make_unique<string>("bar"));
```


### 2. 代码简洁性和一致性
`std::make_unique` 允许使用一致的语法来创建对象，无论它们的构造函数需要多少参数。它还自动推导出对象的类型，使得代码更简洁：

```cpp
auto upw1 = std::make_unique<Widget>(arg1, arg2);  // 推导类型，传递多个构造函数参数
```

### 3. 为什么提供了异常安全
在 C++ 中，当我们说某个操作是“原子操作”，**我们通常是指在单个不可分割的步骤中完成的操作，这意味着操作要么完全执行，要么完全不执行，中间没有中断的可能**。在 `std::make_unique` 的上下文中，虽然操作在底层实现中不是“原子性的”（atomic in the sense of concurrent programming），但从异常安全的角度看，它被视为一个单一的逻辑单元，这确保了在内存分配和对象构造之间不会出现异常导致的内存泄漏。

### 如何 `std::make_unique` 提供异常安全

`std::make_unique` 被设计用来在**单个表达式中完成内存分配和对象构造，然后立即将这个新构造的对象的地址封装到一个 `std::unique_ptr` 中。这就是所谓的“封装到智能指针中是一个原子操作”的意思**：

1. **内存分配和对象构造**：`std::make_unique` 首先在堆上分配足够的内存来存放特定类型的对象，并在该内存位置上直接构造对象。这个步骤使用了完美转发，将所有给定的参数直接传递给对象的构造函数。

2. **封装进 `std::unique_ptr`**：一旦对象被成功构造，其内存地址立即被封装在一个 `std::unique_ptr` 对象中。这个 `std::unique_ptr` 对象然后作为 `make_unique` 的返回值返回。

### 为什么这提高了异常安全性

在 C++ 中，函数参数的求值顺序是未定义的，这意味着在调用一个函数时，如 `F(a, b)`，`a` 和 `b` 的求值顺序由编译器决定，这可能导致安全问题，尤其是当这些参数表达式有副作用时（如分配内存）。如果在参数求值过程中发生异常，已求值的参数可能无法正确回收，导致资源泄漏。

通过使用 `std::make_unique`，确保每次内存分配和对象构造都立即被 `unique_ptr` 接管。如果构造函数抛出异常，`unique_ptr` 从未被创建，但因为异常是在 `make_unique` 内部抛出的，已分配的内存会在抛出异常前由内部机制（通常是函数的栈展开）释放，这就避免了内存泄漏。







