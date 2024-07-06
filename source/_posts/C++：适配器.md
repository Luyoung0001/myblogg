---
title: C++：适配器
date: 2024-04-24 21:30:19
tags: c++ 开发语言
categories: C++
---

<!--more-->

## 正文
当两个类的接口不兼容时，可以使用适配器模式将一个类的接口适配成另一个类的接口，使它们可以协同工作。下面是一个简单的示例，展示**如何使用适配器模式将一个旧的类的接口适配成一个新的类的接口**：

假设有一个旧的类 `OldClass`，它有一个名为 `oldMethod` 的方法，但我们希望将其适配成一个新的类 `NewClass`，并且新类中有一个名为 `newMethod` 的方法。我们可以使用适配器模式来实现这个转换：

```cpp
// 旧的类
class OldClass {
public:
    void oldMethod() {
        std::cout << "Old method is called." << std::endl;
    }
};

// 新的类
class NewClass {
public:
    void newMethod() {
        std::cout << "New method is called." << std::endl;
    }
};

// 适配器类，将 OldClass 的接口适配成 NewClass 的接口
class Adapter {
private:
    OldClass oldObj;

public:
    void newMethod() {
        oldObj.oldMethod(); // 调用旧类的方法
    }
};

int main() {
    Adapter adapter;
    adapter.newMethod(); // 调用适配后的新类方法，实际上会调用旧类的方法
    return 0;
}
```

在上面的示例中，`OldClass` 是旧的类，`NewClass` 是新的类，它们的接口不兼容（假设不兼容，嘻嘻）。通过创建一个适配器类 `Adapter`，在适配器类中调用旧类的方法，从而实现将旧类的接口适配成新类的接口。在 `main` 函数中，我们创建了一个适配器对象，并调用了适配后的新类方法，实际上会调用旧类的方法。

事实上，适配器的概念在 STL 中非常重要，很多的适配器都是基于 STL 提供的容器来实现。换句话说，就是通过 STL 提供的容器来构造要求更高的容器。

在 C++ STL（标准模板库）中，有许多适配器类模板，用于基于不同的容器类型来实现不同的数据结构。这些适配器类模板提供了统一的接口，使得可以通过适配器来构造具有特定功能的容器，从而满足不同的需求。

```cpp
template<typename _Tp, typename _Sequence = deque<_Tp> >
class stack
{

public:
    typedef typename _Sequence::value_type                value_type;
    typedef typename _Sequence::reference                 reference;
    typedef typename _Sequence::const_reference           const_reference;
    typedef typename _Sequence::size_type                 size_type;
    typedef          _Sequence                            container_type;

protected:
    //  See queue::c for notes on this name.
    _Sequence c;

public:
     reference
      top()
      {
        __glibcxx_requires_nonempty();
        return c.back();
      }
    void
      push(const value_type& __x)
      { c.push_back(__x); }
}

```

例如，`std::stack` 是一个栈适配器，它基于其他容器实现的栈数据结构。默认情况下，`std::stack` 使用 `std::deque`（双端队列）作为其底层容器，但也可以通过模板参数指定其他容器类型，如 `std::vector` 或 `std::list`。

类似地，`std::queue` 是一个队列适配器，它基于其他容器实现的队列数据结构。默认情况下，`std::queue` 使用 `std::deque` 作为其底层容器：

```cpp
template<typename _Tp, typename _Sequence = deque<_Tp> >
class queue
{
public:
    typedef typename _Sequence::value_type                value_type;
    typedef typename _Sequence::reference                 reference;
    typedef typename _Sequence::const_reference           const_reference;
    typedef typename _Sequence::size_type                 size_type;
    typedef          _Sequence                            container_type;

protected:

    _Sequence c;

public:

    void push(const value_type& __x)
    { c.push_back(__x); }

    void pop()
    { 
        __glibcxx_requires_nonempty();
      c.pop_front();
    }
}
```


另外，`std::priority_queue` 是一个优先队列适配器，它基于其他容器实现的优先队列数据结构。默认情况下，`std::priority_queue` 使用 `std::vector` 作为其底层容器，但也可以通过模板参数指定其他容器类型：

```cpp
template<typename _Tp, typename _Sequence = vector<_Tp>,
typename _Compare  = less<typename _Sequence::value_type> >
class priority_queue
{
public:
    typedef typename _Sequence::value_type                value_type;
    typedef typename _Sequence::reference                 reference;
    typedef typename _Sequence::const_reference           const_reference;
    typedef typename _Sequence::size_type                 size_type;
    typedef          _Sequence                            container_type;

protected:
    //  See queue::c for notes on these names.
    _Sequence  c;
    _Compare   comp;

public:
    reference
    top() 
    {
	    __glibcxx_requires_nonempty();
	    return c.front();
    }

    void
    push(const value_type& __x)
    {
	    c.push_back(__x);
	    std::push_heap(c.begin(), c.end(), comp);
    }

}
```

这些适配器类模板使得在不同场景下可以方便地使用不同的容器实现，同时提供了统一的接口，使得代码更加灵活和可复用。通过使用 STL 提供的适配器，可以更加方便地构造具有特定功能的容器，满足不同的需求。因此，stack、queue、priority_queue不被称为容器， 把它称为容器配接器。
