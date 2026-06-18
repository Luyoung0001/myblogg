---
title: "C++ 语言笔记：异常处理与对象传递"
date: 2024-04-17 18:01:19
tags:
  - "C++"
  - "exception handling"
categories:
  - "Programming Languages"
---

<!--more-->

## 异常处理中对象参数与引用参数的选择

User：
- `catch()`中的参数如果使用基类的引用，是不是相当于声明了一个引用，它是基类类型的，`catch` 到 `throw`的抛出的对象之后，就相当于这个基类引用指向了抛出的对象？

ChatGPT：
- 是的，你说得很对。当在 `catch` 块中使用基类的引用作为参数时，相当于声明了一个引用，它是基类类型的。当程序抛出一个异常并且在调用栈中找到匹配的 `catch` 块时，**如果异常的类型与 catch 块中指定的引用类型相匹配，那么这个引用会指向被抛出的异常对象**。
这样做的好处是可以使用**基类的引用来捕获派生类对象的异常，从而实现多态性**。因为派生类对象也是基类对象，所以可以使用基类的引用来引用它们，从而实现了异常处理时的多态性。

User：
- 如果 `catch` 中的参数不是引用，而是一个对象呢？情况将会是如何的？

ChatGPT：
- **如果在 catch 块中的参数不是引用，而是一个对象，那么将会发生对象的拷贝**。这意味着异常对象会被复制一份，而不是通过引用来访问原始的异常对象。
具体来说，当抛出异常并且在调用栈中找到匹配的 `catch` 块时，如果异常的类型与 `catch` 块中指定的对象类型相匹配，那么会发生对象的拷贝，即异常对象的副本会被创建并传递给 `catch` 块。
这种情况下，与使用引用相比，会产生额外的开销，因为需要进行对象的拷贝操作。通常情况下，建议在异常处理中使用引用而不是对象，以避免不必要的对象拷贝。

User：
- 如果是这样，那么整个过程将会产生两次复制构造，一次是 `throw`，一次是 `catch`，对吗？

ChatGPT：
- 是的，你说得对。如果在 `catch` 块中的参数是一个对象而不是引用，那么整个过程将会产生两次复制构造。
因此，使用对象作为 `catch` 块的参数会导致两次复制构造，增加了额外的开销。**相比之下，使用引用作为 `catch` 块的参数可以避免这种额外的开销**，因为引用不会引发对象的复制，而是直接引用原始的异常对象。

可以看到 GPT 的回答很好，以下是我写得一个实例，来解释以上的所有信息：
```cpp
#include <iostream>
#include <string>

int a = 0;

class Base {
    int data;

  public:
    Base(int d = 0) : data(d) {}
    virtual ~Base() { std::cout << "Base destructor called.\n" << std::endl; }
    virtual void display() const {
        std::cout << "Base base value:" << data << "\n" << std::endl;
    }
    int value() const { return data; }
    // 复制构造
    Base(const Base &other) : data(other.data) {
        a++;
        std::cout << "Base copy constructor called.\n" << std::endl;
    }
};

class Derived : public Base {
    char data;

  public:
    Derived(char c = 'a') : Base(), data(c) {}
    virtual ~Derived() {
        std::cout << "Derived destructor called.\n" << std::endl;
    }
    virtual void display() const override {
        std::cout << "Derived base value:" << Base::value() << std::endl;
        std::cout << "Derived derived value:" << data << "\n" << std::endl;
    }
    // 复制构造
    Derived(const Derived &other) : Base(other), data(other.data) {
        a++;
        std::cout << "Derived copy constructor called.\n" << std::endl;
    }
};

void throwException(bool useDerived) {
    if (useDerived) {
        Derived d('b');
        throw d;
    } else {
        Base b(100);
        throw b;
    }
}

int main() {
    try {
        // 使用基类的引用捕获异常

        // 调用一次基类复制构造+1
        // 调用一次派生类复制构造+1

        // 析构抛出对象
        try {
            std::cout << "---------------开始构造--------------" << std::endl;
            throwException(true);
        } catch (Base &b) {
            std::cout << "\na: " << a << std::endl;
            b.display(); // 多态调用
            // 生命周期结束,析构指向的对象
            std::cout << "---------------开始析构--------------" << std::endl;
        }

        // 调用一次基类复制构造+1
        // 调用一次派生类复制构造+1
        // 析构抛出对象

        // 由于catch()参数是对象
        // 继续调用基类复制构造+1

        try {
            std::cout << "---------------开始构造--------------" << std::endl;
            throwException(true);

        } catch (Base b) {
            std::cout << "\na: " << a << std::endl;
            b.display(); // 非多态调用,发生切片现象
            // 生命周期结束,析构临时对象,以及 b
            std::cout << "---------------开始析构--------------" << std::endl;
        }

        // 调用一次基类复制构造+1
        // 析构抛出对象
        try {
            std::cout << "---------------开始构造--------------" << std::endl;
            throwException(false);
        } catch (Base &b) {
            std::cout << "\na: " << a << std::endl;
            b.display();
            // 生命周期结束,析构指向的对象
            std::cout << "---------------开始析构--------------" << std::endl;
        }

        // 调用基类复制构造+1
        // 析构抛出对象
        // 由于 catch()参数是对象
        // 调用复制构造+1
        try {
            std::cout << "---------------开始构造--------------" << std::endl;
            throwException(false);
        } catch (Base b) {
            std::cout << "\na: " << a << std::endl;
            b.display();
            // 生命周期结束,析构临时复制对象,以及 b
            std::cout << "---------------开始析构--------------" << std::endl;
        }

    } catch (const std::exception &e) {
        std::cerr << "\nCaught exception: " << e.what() << std::endl;
    }

    return 0;
}


```

运行结果：

```bash
./main
---------------开始构造--------------
Base copy constructor called.

Derived copy constructor called.

Derived destructor called.

Base destructor called.


a: 2
Derived base value:0
Derived derived value:b

---------------开始析构--------------
Derived destructor called.

Base destructor called.

---------------开始构造--------------
Base copy constructor called.

Derived copy constructor called.

Derived destructor called.

Base destructor called.

Base copy constructor called.


a: 5
Base base value:0

---------------开始析构--------------
Base destructor called.

Derived destructor called.

Base destructor called.

---------------开始构造--------------
Base copy constructor called.

Base destructor called.


a: 6
Base base value:100

---------------开始析构--------------
Base destructor called.

---------------开始构造--------------
Base copy constructor called.

Base destructor called.

Base copy constructor called.


a: 8
Base base value:100

---------------开始析构--------------
Base destructor called.

Base destructor called.

```

可以看到`catch()`的参数尽管可以被设为引用，但是其实仍然要发生异常对象的复制（用复制构造函数）。**如果没有被设置为引用，那么在最后，不仅要析构异常对象副本，还要析构异常对象本身。**





