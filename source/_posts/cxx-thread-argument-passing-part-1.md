---
title: "C++ 并发编程：线程函数传参陷阱（一）"
date: 2024-05-09 23:39:44
tags:
  - "C++"
  - "thread"
  - "argument passing"
categories:
  - "Programming Languages"
---

<!--more-->

## 一、问题
当创建 `std::thread` 对象时，传递给线程的函数的所有参数都会**被复制或移动**到新创建的线程的内存空间中。这是为了确保线程的执行不会依赖于父线程可能销毁的栈上变量，从这个机制上看，这是很合理的。

在新的线程的栈上，这些变量都会以右值的方式传递给线程函数，这主要是为了提高传参的性能。但是在某些情况下，**这些右值并不能满足线程函数的参数**。如果线程函数需要引用参数，直接传递普通变量会因为**无法从临时对象（复制或移动产生的）绑定到非 const 引用而失败**。比如以下的例子：
```cpp
#include <iostream>
#include <thread>

class BigObject {
    std::string s = "hello";

  public:
    const std::string &getData() const { return s; }
    void upDateData(const std::string &str) { s = str; }
    void showInfo() const {
        std::cout << "addr: " << this << " value: " << s << std::endl;
    }
    ~BigObject(){};
    BigObject(){};
};
void update_data_for_BigOb(std::string newString, BigObject &data);
void printInfo(BigObject &ob);

void oops_again(std::string w) {
    BigObject data;
    printInfo(data);
    std::thread t(update_data_for_BigOb, w, data);
    t.join();
}

int main() {
    oops_again("hello_new!"); // 函数调用
    return 0;
}

void update_data_for_BigOb(std::string newString, BigObject &data) {
    // 修改
    data.upDateData(newString);
    printInfo(data);
}
void printInfo(BigObject &ob) { ob.showInfo(); }



```
这样会编译失败：
```bash
g++ parameter1.cxx -o main -std=c++11
In file included from parameter1.cxx:2:
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/thread:280:5: error: attempt to use a deleted function
    __invoke(_VSTD::move(_VSTD::get<1>(__t)), _VSTD::move(_VSTD::get<_Indices>(__t))...);
    ^
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/thread:291:5: note: in instantiation of function template specialization 'std::__1::__thread_execute<std::__1::unique_ptr<std::__1::__thread_struct>, void (*)(std::__1::basic_string<char>, BigObject &), int, BigObject, 2, 3>' requested here
    __thread_execute(*__p, _Index());
...
```

遇到的错误与尝试在 `std::thread` 构造器中使用不匹配的参数类型有关。错误的根本原因是在创建 `std::thread` 实例时**传递的参数类型与线程函数所期望的参数类型不兼容**。因此我们要考虑，**传递的参数如何才能正确兼容线程函数期望的参数类型**。这正是上面讨论到的一种情况，接着讨论如何优雅的解决这种问题。
## 二、自动类型转换

在C++中，可以通过**构造函数**和**类型转换操作符**来实现类对象之间的隐式类型转换：

```cpp
#include <iostream>

class Number {
private:
    int value;

public:
    // 构造函数
    Number(int val) : value(val) {}

    // 类型转换操作符
    operator int() const {
        return value;
    }
};

int main() {
    Number num1 = 5;
    int result = num1 + 10;  // 隐式类型转换发生在这里
    std::cout << result << std::endl;  // 输出 15
    return 0;
}

```
上文示例中，可以看到 **num1** 发生了隐式转换（我们暂不考虑强制显式转换），**Number** 对象转换成了 **int** 基本类型。这是因为我们定义了**类型转换操作符**。当我们在 **main** 函数中执行 `num1 + 10` 时，由于 10 是一个整数，C++ 编译器将自动调用 `operator int()` 来将 **num1** 隐式转换为 **int** 类型，然后执行加法操作。

## 三、包装器

以下是 std::ref 的简化版本源码示例：

```cpp
namespace std {
    // 定义 ref 类模板
    template<class T>
    class reference_wrapper {
    public:
        // 构造函数，接受一个对象的引用
        reference_wrapper(T& ref) : _ref(ref) {}

        // 拷贝构造函数和赋值运算符被删除，禁止拷贝和赋值
        reference_wrapper(const reference_wrapper&) = delete;
        reference_wrapper& operator=(const reference_wrapper&) = delete;

        // 重载解引用运算符，返回引用对象
        // 也可以隐式转换为引用对象
        operator T&() const { return _ref; }

    private:
        T& _ref; // 存储引用对象的引用
    };

    // ref 函数模板，接受一个对象，并返回一个 reference_wrapper 包装后的对象
    template<class T>
    reference_wrapper<T> ref(T& t) {
        return reference_wrapper<T>(t);
    }
}

```

在使用 `std::ref` 的时候，实际上是将传递的对象包装成了一个	`reference_wrapper<T>(t)` 对象，如果我们将 **问题** 中的代码这样改：

```cpp
...
void oops_again(int w) {
...
    std::thread t(update_data_for_BigOb, w, std::ref(data));

}
...
```
这将会发生什么？
这个包装器对象（很轻量）将会被**移动到**新线程的栈中（类中禁止了复制构造函数），然后这个包装器对象被以右值的方式绑定到线程函数的参数。别忘了，这个包装器类内部定义了**隐式转换函数**：

```cpp
operator T&() const { return _ref; }
```

这个函数会将**包装器对象隐式转化为被包装的对象的引用**，而此时，这个被包装的对象正在另一个栈空间中呢，所以它非常适合被绑定到左值引用。当然，这发生在包装器对象尝试**以右值的方式被绑定到线程函数的引用类型参数上时**。

至此，我们可以运行一下修改后的代码，以作验证：

```bash
g++ parameter1.cxx -o main -std=c++11
./main
addr: 0x16af47078 value: hello
addr: 0x16af47078 value: hello_new!
```

可以看到，尽管线程函数传参的路途再曲折，也会顺利将 **data** 传进去。


