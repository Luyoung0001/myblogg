---
title: "C++ 语言笔记：智能指针（一）unique_ptr 的移动语义"
date: 2024-04-21 23:18:59
tags:
  - "C++"
  - "smart pointer"
  - "unique_ptr"
categories:
  - "Programming Languages"
---

<!--more-->

## 正文
关于这个例子：

```cpp
#include <iostream>
#include <memory>
#include <string>

std::unique_ptr<std::string> demo(const char *s) {
    std::unique_ptr<std::string> temp(new std::string(s));
    return temp;
}

int main() {
    // 调用demo函数，返回一个unique_ptr<string>，并将其赋值给ps
    std::unique_ptr<std::string> ps;
    ps = demo("Uniquely special");

    // 输出ps所指向的字符串内容
    std::cout << *ps << std::endl;

    return 0;
}

```

我们很容易理解，智能指针来源于模板类，对于上面的 `demo()`，它返回了一个对象 `temp`。按照一般认知，在不考虑前面讨论的返回值优化的前提下，`demo()` 在返回前会移动构造创建一个临时栈外的对象，然后通过这个对象再移动赋值给 ps，然后析构掉 `temp`，这个过程依赖于移动赋值函数。智能指针之所以智能，就是做了一些特定的设计：

```cpp
#ifdef _LIBCPP_CXX03_LANG
  unique_ptr(unique_ptr const&) = delete;
  unique_ptr& operator=(unique_ptr const&) = delete;
#endif
...

```

在头文件 `memory` 中，可以看到上面的代码禁止了复制构造函数和复制赋值构造函数，因此上面的提到的过程就会发生。但是这会导致空指针异常吗？按照之前的经验，返回的对象 `temp` 会被销毁，从而导致 `ps` 对象是空的。

这时候，就要了解智能指针设计的精妙了：

```cpp
template <class _Up, class _Ep,
      class = _EnableIfMoveConvertible<unique_ptr<_Up, _Ep>, _Up>,
      class = _EnableIfDeleterAssignable<_Ep>
  >
  _LIBCPP_INLINE_VISIBILITY
  unique_ptr& operator=(unique_ptr<_Up, _Ep>&& __u) _NOEXCEPT {
    reset(__u.release());
    __ptr_.second() = _VSTD::forward<_Ep>(__u.get_deleter());
    return *this;
  }
...
```

**尽管智能指针禁止了复制构造函数和复制赋值构造函数，但是提供了移动赋值运算符重载函数。**这将会在赋值的时候，将右值的资源转让给左值。因此上面的程序是没有任何错误与风险的。

再来看看关于智能指针的一个程序：

```cpp
#include <iostream>
#include <memory>
#include <vector>

std::unique_ptr<int> make_int(int n) {
    return std::unique_ptr<int>(new int(n));
}

void show(std::unique_ptr<int> &pi) {
    std::cout << *pi << ' ';
}

int main() {
    // 创建一个包含动态分配整数的 unique_ptr 向量
    std::vector<std::unique_ptr<int>> vp(10);
    for (int i = 0; i < vp.size(); ++i) {
        // 通过 make_int 函数创建动态分配的整数，并将其存储在向量中
        vp[i] = make_int(rand() % 1000);
    }

    // 使用 for_each 遍历向量，并调用 show 函数显示每个整数
    for (auto& ptr : vp) {
        show(ptr);
    }

    return 0;
}

```

在给`show()`函数传递参数时，如果按值而不是按引用传递`unique_ptr`对象，会导致编译器尝试复制`unique_ptr`对象，这是不允许的，因为`unique_ptr`禁止了复制构造函数。因此，如果按值传递`unique_ptr`对象给`show()`函数，编译器将发出错误提示。

另外，`for_each()`函数中的lambda表达式会将`vp`中的每个元素传递给`show()`函数，而lambda表达式默认情况下是按值捕获参数的。因此，如果`show()`函数接受的参数是按值而不是按引用传递的，那么`for_each()`语句将会产生编译错误，因为它会尝试复制构造`unique_ptr`对象。

为了解决这个问题，可以将`show()`函数的参数声明为`const std::unique_ptr<int>&`，以便按引用传递参数，或者使用`std::move()`将`unique_ptr`对象作为右值传递给`show()`函数。





