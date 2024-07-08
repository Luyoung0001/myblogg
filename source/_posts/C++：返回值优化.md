---
title: C++：返回值优化
date: 2024-04-17 19:25:57
tags:
    - c++
    - 开发语言
categories: C++
---

<!--more-->

## 正文
对于返回一个对象的函数，它在返回后前，应该在栈外用复制构造函数创建一个临时对象t2，然后返回 t1，随后 t1 在栈内被析构掉，t2 被传回。如下：

```cpp
#include <iostream>
#include <string>

class MyClass {
  private:
    int data;

  public:
    MyClass(int d = 0) : data(d) {
        std::cout << "Constructor called. Data: " << data << std::endl;
    }

    MyClass(const MyClass &other) : data(other.data) {
        std::cout << "Copy constructor called. Data: " << data << std::endl;
    }

    ~MyClass() {
        std::cout << "Destructor called. Data: " << data << std::endl;
    }
    // 赋值构造
    MyClass &operator=(const MyClass &other) {
        std::cout << "Myclass = constructor called.\n" << std::endl;
        if (this != &other) { // 检查自我赋值
            data = other.data;
        }
        return *this;
    }

    int getData() const { return data; }
};

MyClass createObject(int value) {
    MyClass m1(value); // 栈内创建对象
    return m1;         // 触发复制构造函数
}

int main() {
    MyClass obj = createObject(10);
    std::cout << "Returned object data: " << obj.getData() << std::endl;

    return 0;
}


```

这个运行结果应该是：

```bash
Constructor called. Data: 10
Copy constructor called. Data: 10
Destructor called. Data: 10
Returned object data: 10
Destructor called. Data: 10

```

首先，调用构造函数产生t1，然后调用复制构造产生栈外对象t2，然后析构掉t1，然后析构掉obj。
但事实上，运行结果如下：

```bash
./main
Constructor called. Data: 10
Returned object data: 10
Destructor called. Data: 10
```

看起来少生成了一个对象，答案没错，确实是少了一个对象。这个过程被编译器优化掉了，编译器在构建 obj 的时候，往里面传入了obj 的地址，然后直接在函数内部构建对象，整个过程并没有用复制构造函数产生栈外的t2。

**这种优化称为返回值优化（Return Value Optimization，简称 RVO），它能够提高程序的性能并减少不必要的对象拷贝。在这种情况下，编译器将对象直接构造在函数调用方指定的内存空间中，而不是通过复制构造函数创建临时对象。**

这里就清晰了，但是如果是这样呢？

```cpp
#include <iostream>
#include <string>

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
        std::cout << "Base copy constructor called.\n" << std::endl;
    }
    // 赋值构造
    Base &operator=(const Base &other) {
        std::cout << "Base = constructor called.\n" << std::endl;
        if (this != &other) { // 检查自我赋值
            data = other.data;
        }
        return *this;
    }
};

Base r1() {
    Base b(5);
    return b;
}
void r2(Base base) { std::cout << base.value() << std::endl; }
int main() {
    // 这里进行了优化
    // 编译器偷偷地增加了一个参数,传入了b1的地址,直接在函数内部构造了b1对象
    // 因此这里的复制构造不会被调用
    Base b1 = r1();

    // 这里会老实的调用复制构造函数
    r2(b1);

    // 之后析构掉临时对象copied from b1
    std::cout << "Done!\n";
    // 最后析构掉b1
    return 0;
}

```
运行结果：

```bash
./main
Base copy constructor called.

5
Base destructor called.

Done!
Base destructor called.
```

上面的代码中的注释解释得很清楚，如果我们这样写：

```cpp
int main() {
    r2(r1());
    std::cout << "Done!\n";
    return 0;
}
```
运行结果就是：

```bash
./main
5
Base destructor called.

Done!
```

看起来函数参数为对象时，并没有调用复制构造函数产生临时对象，这是因为：
- 在 `r1()` 函数中创建了一个 Base 对象 b，然后通过 return b; 返回该对象。但是在返回之前，编译器可以优化，直接在调用 r2() 函数时在调用栈上构造一个临时对象，而不是在 r1() 函数中创建对象，然后在返回时调用复制构造函数来构造临时对象。

- 在 `r2(Base base)` 函数中，传递的参数 base 是按值传递的，这意味着会调用复制构造函数来创建参数的副本。但是编译器可能会对此进行优化，直接在调用 `r2()` 函数时，通过复制构造函数在调用栈上构造一个临时对象，而不是在调用方和被调用方之间复制对象。

**前者是返回值优化，后者属于对象参数的优化。**对象参数的优化是编译器在调用函数时对传递的对象进行优化的一种方式。在某些情况下，编译器会避免创建临时对象，而是直接在调用栈上构造对象，从而节省了对象的复制操作，提高了程序的性能。这种优化通常是根据函数的调用方式和参数的类型来进行的。
