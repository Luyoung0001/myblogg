---
title: C++：函数符（一）
date: 2024-04-24 22:00:35
tags: c++ 开发语言
categories: C++
---

<!--more-->

## 正文
**函数对象**也叫函数符，函数符是可以以函数方式与()结合使用的任意对象。这包括**函数名**、**指向函数的指针**和**重载了()运算符的类对象。**

上面这句话的意思是指：函数名、指向函数的指针和重载了括号运算符的类对象与括号结合，从而以函数方式实现某种功能。
对于 `for_each()`，第三个参数我们一般可以写一个函数名，或者类的对象（该类必须对括号运算符进行了重载），但是不可以为一个函数指针，因为函数指针的类型已经确定，要迭代的对象类型并不会提前知道。比如：

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// 定义一个类，其中重载了 () 运算符
class FunctionObject {
public:
    // 重载 () 运算符，用于执行特定操作
    void operator()(int num) const {
        std::cout << num << " squared is: " << num * num << std::endl;
    }
};

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};

    // 使用函数对象执行特定操作，这里是输出数字的平方
    FunctionObject f;
    std::for_each(numbers.begin(), numbers.end(), f);

    return 0;
}

```
当然，上面的代码还可以改成本：

```cpp
...

int main() {
..
    std::for_each(numbers.begin(), numbers.end(), FunctionObject());
...
}

```

这是类 `FunctionObject` 的构造函数构造的一个匿名对象。如果我们用函数指针：

```cpp

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// 定义一个函数，用于执行特定操作
void squareAndPrint(int num) {
    std::cout << num << " squared is: " << num * num << std::endl;
}

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};

    // 使用函数指针执行特定操作，这里是输出数字的平方
    void (*funcPtr)(double) = squareAndPrint;
    std::for_each(numbers.begin(), numbers.end(), funcPtr); // 编译错误

    return 0;
}
```
再来看看 `for_each()` 源码：

```cpp
// for_each

template <class _InputIterator, class _Function>
inline _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR_AFTER_CXX17
_Function
for_each(_InputIterator __first, _InputIterator __last, _Function __f)
{
    for (; __first != __last; ++__first)
        __f(*__first);
    return __f;
}
```
可以看到，只要第三个参数能结合括号运算符，并且接受的参数 `*__first`类型正确，就能编译通过，事实上也确实如此。

```bash
g++ func_ptr.cxx -o main -std=c++11
func_ptr.cxx:14:12: error: cannot initialize a variable of type 'void (*)(double)' with an lvalue of type 'void (int)': type mismatch at 1st parameter ('double' vs 'int')
    void (*funcPtr)(double) = squareAndPrint;
           ^                  ~~~~~~~~~~~~~~
1 error generated.
```

我们修改代码：

```cpp
...
int main() {
...
    void (*funcPtr)(int) = squareAndPrint;
    std::for_each(numbers.begin(), numbers.end(), funcPtr); // 编译正确
...
}
```
运行结果：

```bash
g++ func_ptr.cxx -o main -std=c++11
~/Cpp_Notes/chap16/func ./main
1 squared is: 1
2 squared is: 4
3 squared is: 9
4 squared is: 16
5 squared is: 25
```

你可能会问，为什么？第三个参数明明去要一个对象，我们传入了一个函数指针却能编译通过，难道计算机科学不存在了？

**这是因为 C++ 允许将函数指针隐式转换为函数对象**。在这种情况下，**编译器将自动创建一个临时的函数对象来包装函数指针，并将其传递给 std::for_each 函数。**

因此，虽然 `std::for_each(numbers.begin(), numbers.end(), funcPtr);` 看起来似乎是将一个函数指针传递给 `for_each` 函数，但实际上编译器会将其转换为类似于 `std::for_each(numbers.begin(), numbers.end(), FuncWrapper(funcPtr)); `的形式，其中 `FuncWrapper` 是一个临时的函数对象，用于包装函数指针 **funcPtr**。

使用函数指针作为算法的操作函数参数是合法的，但不建议使用函数指针的主要原因有以下几点：

1. **可读性和可维护性差**：函数指针的语法相对复杂，不够直观，可能会降低代码的可读性和可维护性。相比之下，使用函数对象或 lambda 表达式更加直观和易于理解。

2. **灵活性不足**：函数指针只能指向静态函数或全局函数，无法捕获外部变量。而函数对象或 lambda 表达式可以轻松捕获外部变量，提供更大的灵活性。

3. **类型安全性**：函数指针在类型匹配上需要开发者自行确保，容易出现类型不匹配的问题。而函数对象或 lambda 表达式在编译时会进行类型检查，更加安全。

4. **性能问题**：函数指针的调用可能会引入额外的开销，因为在调用函数指针时需要进行间接跳转。而函数对象或 lambda 表达式可能更加高效。
5. 
相比之下，我们可以使用函数对象或者 lambda 表达式作为 `std::for_each` 的第三个参数，因为它们都是可调用对象。函数对象是一个类，重载了 `operator()` 运算符，可以像函数一样被调用。Lambda 表达式也是一个可调用对象，可以在需要时直接定义并使用。

