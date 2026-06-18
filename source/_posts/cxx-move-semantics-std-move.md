---
title: "C++ 语言笔记：移动语义与 std::move"
date: 2024-05-11 23:43:16
tags:
  - "C++"
  - "move semantics"
  - "std::move"
categories:
  - "Programming Languages"
---

<!--more-->

## 一、std::move
### （一）std::move 的原型

```cpp
 template<typename T>
 decltype(auto) move(T&& param)  //注意，形参是个引用（万能引用）
{
        using ReturnType = typename remove_reference<T>::type&&; 
        return static_cast<ReturnType>(param); //强制转换为右值引用类型
}
```
这里涉及到了萃取类型，将 `T` 的引用去除，然后转为 `type&&`，变成了一个右值引用，也就是说`ReturnType` 的类型为 `T&&`（**干净的 T**）。下面顺便讨论下萃取类型。
#### 萃取类型
类型萃取是 C++ 模板元编程中的一种技术，用于在编译时查询或修改类型的属性。类型萃取可以解决复杂模板编程中的类型相关问题，如条件编译、类型转换、类型检查等。这些技术主要通过标准库中的 `type_traits` 头文件提供的工具实现。

##### 基本概念

类型萃取使用模板结构体来封装编译时的类型信息。这些模板结构体通常具有一个或多个静态成员，可以是一个类型（通过 `typedef` 或 `using` 定义的别名），也可以是一个常量值。

##### 常用的类型萃取

1. **类型修改器**：
   - `std::remove_reference<T>`：去除类型 `T` 的引用部分。
   - `std::add_const<T>`：给类型 `T` 添加 `const` 修饰符。
   - `std::remove_const<T>`：去除类型 `T` 的 `const` 修饰符。
   - `std::make_signed<T>`：将整型类型 `T` 转换为相应的有符号类型。
   - `std::make_unsigned<T>`：将整型类型 `T` 转换为相应的无符号类型。

2. **类型属性检查器**：
   - `std::is_integral<T>`：检查 `T` 是否是整数类型。
   - `std::is_floating_point<T>`：检查 `T` 是否是浮点数类型。
   - `std::is_array<T>`：检查 `T` 是否是数组类型。
   - `std::is_pointer<T>`：检查 `T` 是否是指针类型。
   - `std::is_const<T>`：检查 `T` 是否有 `const` 修饰。

3. **类型关系检查器**：
   - `std::is_same<T, U>`：检查两个类型 `T` 和 `U` 是否完全相同。
   - `std::is_base_of<T, U>`：检查 `T` 是否是 `U` 的基类。
   - `std::is_convertible<T, U>`：检查类型 `T` 是否可以被隐式转换为类型 `U`。

##### 示例

以下是一些使用类型萃取的简单示例：

```cpp
#include <type_traits>
#include <iostream>

int main() {
    std::cout << std::boolalpha; // 使得布尔值输出为 true/false 而不是 1/0

    // 检查是否为整数类型
    std::cout << "is_integral<int>: " << std::is_integral<int>::value << '\n';

    // 检查是否为浮点数类型
    std::cout << "is_floating_point<float>: " << std::is_floating_point<float>::value << '\n';

    // 移除引用
    using no_ref = std::remove_reference<int&>::type;
    std::cout << "is_same<int, no_ref>: " << std::is_same<int, no_ref>::value << '\n';

    // 添加 const 修饰符
    using add_const_to_int = std::add_const<int>::type;
    std::cout << "is_const<add_const_to_int>: " << std::is_const<add_const_to_int>::value << '\n';
}
```

运行结果：

```bash
./main
is_integral<int>: true
is_floating_point<float>: true
is_same<int, no_ref>: true
is_const<add_const_to_int>: true
```
### （二）注意事项

   　1. `std::move` 的本质就强制类型转换，它无条件地将**实参转为右值引用类型**（匿名对象，是个右值），继而用于移动语义。

   　2. 该函数只是将实参转为右值，除此之外并没有真正的 move 任何东西。**实际上，它在运行期没任何作为，编译器也不会为它生成任何的可执行代码，连一个字节都没有**。

   　3. 如果要对某个对象执行移动操作时，则不要将其声明为常量。**因为针对常量对象执行移动操作将变成复制操作**。
## 二、移动语义
   1. 复制/移动操作的函数声明

```cpp
①Object(T&);       //复制构造，仅接受左值

②Object(const T&); //复制构造，即可以接受左值又可接收右值

③Object(T&&) noexcept; //移动构造，仅接受右值

④T& operator=(const T&);//复制赋值函数，即可以接受左值又可接收右值

⑤T& operator=(T&&); //移动赋值函数，仅接受右值
```

   2. 注意事项

　　①移动语义一定是要修改临时对象的值，**所以声明移动构造时应该形如Test(Test&&)，而不能声明为Test(const Test&&)**。

　　②默认的移动构造函数实际上跟默认的拷贝构造函数一样，都是“浅拷贝”。**通常情况下，必须自定义移动构造函数**。

　　③对于移动构造函数来说，抛出异常是很危险的。因为移动语义还没完成，一个异常就抛出来，可能会造成悬挂指针。因此，**应尽量通过noexcept声明不抛出异常，而一旦出现异常就可以直接调用std::terminate终止程序**。

　　④特殊成员函数之间存在相互抑制的生成机制，可能会影响到默认拷贝构造和默认移动构造函数的自动生成。

以下例子展示了 `std::move` 的移动语义：

```cpp
#include <iostream>
#include <vector>

using namespace std;

// 1. 移动语义
class HugeMem {
  public:
    int *buff;
    int size;

    HugeMem(int size) : size(size > 0 ? size : 1) { buff = new int[size]; }

    //移动构造函数
    HugeMem(HugeMem &&hm) noexcept : buff(hm.buff), size(hm.size) {
        hm.buff = nullptr;
    }

    ~HugeMem() { delete[] buff; }
};

class Moveable {
  public:
    HugeMem h;
    int *i;

  public:
    Moveable() : h(1024), i(new int(3)) {}
    // 移动构造函数（强制转为右值，以调用h的移动构造函数。注意m虽然是右值
    // 引用，但形参是具名变量，m是个左值。因此m.h也是左值，需转为右值。
    Moveable(Moveable &&m) noexcept : h(std::move(m).h), i(m.i) {
        m.i = nullptr;
    }
    ~Moveable() { delete i; }
};

Moveable GetTemp() {
    Moveable tmp = Moveable();
    cout << hex << "Huge mem from " << __func__ << " @" << tmp.h.buff
         << " addr:" << &tmp << endl;
    return tmp;
}

// 2. 对常量对象实施移动将变成复制操作
class Annotation {
    std::string value;

  public:
    // 注意：对常量的text对象实施移动操作时，由于std::move(text)返回的结果是个
    // const std::string对象，由于带const，不能匹配string(&& rhs)移动构造函数，
    // 但匹配 string(const string& rhs)
    // 复制构造函数，因此当执行value(std::move(text))
    // 时，实际上是将text复制给value。对于非string类型的情况也一样，因此对常量对象的
    // 移动操作实际上会变成复制操作！
    explicit Annotation(const std::string text) : value(std::move(text)) {}
};

// 3. 利用移动语义实现高性能的swap函数
template <typename T>
void Swap(T &a, T &b) noexcept //声明为noexcept以便在交换失败时，终止程序
{
    //如果a、b是可移动的，则直接转移资源的所有权
    //如果是不可移动的，则通过复制来交换两个对象。
    T tmp(std::move(a)); //先把a的资源转交给tmp
    a = std::move(b);
    b = std::move(tmp);
}

int main() {
    // 1. 移动语义
    Moveable a(GetTemp()); //移动构造,涉及到了返回值优化
    cout << hex << "Huge mem from " << __func__ << " @" << a.h.buff
         << " addr:" << &a << endl;

    Moveable b(std::move(a));
    cout << hex << "Huge mem from " << __func__ << " @" << b.h.buff
         << " addr:" << &b << endl;

    return 0;
}

```

在 `main` 中，`Moveable a(GetTemp());` 利用 `GetTemp()` 生成了一个右值作为 `a` 的构造函数参数（暂时可以这么认为，先不考虑返回值优化）。这会匹配移动构造函数进行构造，打印结果也说明了这一点。其次，`b` 也是通过移动构造函数创建，依次转移了对象的资源。程序运行结果：

```bash
g++ -fno-elide-constructors 2.cxx -o main -std=c++11
./main
Huge mem from GetTemp @0x12d008e00 addr:0x16b722d20
Huge mem from main @0x12d008e00 addr:0x16b722e10
Huge mem from main @0x12d008e00 addr:0x16b722dd0
```
可以看到 `buff` 的地址是相同的。如果仔细观察，**可以看到上面的编译指令禁止了编译优化**，这意味着`Moveable a(GetTemp())`不涉及返回值优化。`tmp` 对象在`GetTemp()` 中构建，然后通过移动语义构建临时变量，然后通过临时变量再构建对象 `a`。

再来梳理一下：当关闭返回值优化（RVO）时，移动操作发生两次，**这主要是由于编译器必须创建一个临时对象来持有从函数返回的值**。这个过程可以细分为以下步骤：

 - **第一步：从 `tmp` 移动到临时对象**
- - 在函数 `GetTemp()` 结束时，`tmp` 对象需要被返回。由于关闭了 `RVO`，编译器不能直接在目标位置构造 `tmp`，因此它构造了一个临时对象。
这个临时对象是通过调用 `Moveable` 的移动构造函数从 `tmp` 中创建的。这是第一次应用移动构造，它将 `tmp` 中的资源（例如指向动态内存的指针）转移到了这个临时对象中，并将 `tmp` 中相应的指针设为 `nullptr` 或其他无效状态。
- **第二步：从临时对象移动到 a**
- - 这个临时对象接着用来初始化 `main` 函数中的 `Moveable a`。
再次调用 `Moveable` 的移动构造函数，将临时对象中的资源转移到 `a`。这是第二次移动操作，再次将资源所有权转移，而且临时对象被置于无效状态。

我们可以通过在析构函数中打印一些信息验证下：

```cpp
~HugeMem() {
        cout << hex << "destructor called in " << __func__ << " for object@"
             << this << endl;
        delete[] buff;
    }
```
运行结果：

```bash
g++ -fno-elide-constructors 2.cxx -o main -std=c++11
./main
destructor called in ~HugeMem for object@0x16d7bed08
Huge mem from GetTemp @0x158008e00 addr:0x16d7bed20
destructor called in ~HugeMem for object@0x16d7bed20
destructor called in ~HugeMem for object@0x16d7bedf8
Huge mem from main @0x158008e00 addr:0x16d7bee10
Huge mem from main @0x158008e00 addr:0x16d7bedd0
destructor called in ~HugeMem for object@0x16d7bedd0
destructor called in ~HugeMem for object@0x16d7bee10
```
这是关闭返回值优化后的析构信息以及打印信息。

首先在`GetTemp()`中：

```cpp
Moveable GetTemp() {
    Moveable tmp = Moveable();
    cout << hex << "Huge mem from " << __func__ << " @" << tmp.h.buff
         << " addr:" << &tmp << endl;
    return tmp;
}
```
`Moveable()` 通过**默认构造函数**创建了右值对象`object@0x16d7bed08`，然后通过移动语义初始化 `tmpobject@0x16d7bed20`，这时候右值对象生命周期结束，打印出了**第一个析构**信息：

```bash
destructor called in ~HugeMem for object@0x16d7bed08
```

接着通过 `tmpobject@0x16d7bed20` 通过移动语义创建右值临时返回对象 `object@0x16d7bedf8`，接着`tmpobject@0x16d7bed20` 被析构掉。然后通过临时对象构建 `a`，临时对象`object@0x16d7bedf8`被析构，打印出了：

```bash
destructor called in ~HugeMem for object@0x16d7bed20
destructor called in ~HugeMem for object@0x16d7bedf8
```
接着逆序析构 `b`,`a`：

```bash
destructor called in ~HugeMem for object@0x16d7bedd0
destructor called in ~HugeMem for object@0x16d7bee10
```

这就是上面的整个过程，可见编译优化大大提升了性能。



如果我们不禁止编译优化：

```bash
g++ 2.cxx -o main -std=c++11
./main
Huge mem from GetTemp @0x12f008e00 addr:0x16dc3ee10
Huge mem from main @0x12f008e00 addr:0x16dc3ee10
Huge mem from main @0x12f008e00 addr:0x16dc3ede8
destructor called in ~HugeMem for object@0x16dc3ede8
destructor called in ~HugeMem for object@0x16dc3ee10
```

从打印的输出信息来看：

```
Huge mem from GetTemp @0x12f008e00 addr:0x16dc3ee10
Huge mem from main @0x12f008e00 addr:0x16dc3ee10
```

上述显示了 `GetTemp()` 函数中的 `Moveable` 对象 `tmp` 和在 `main` 函数中通过 `Moveable a(GetTemp());` 创建的对象 `a` 具有相同的内存地址。这表明 `a` 是直接在 `GetTemp()` 的返回位置上构建的，而没有发生额外的移动或复制操作。

编译器可以直接在调用函数的返回位置构造 `Moveable` 对象。即在 `GetTemp()` 调用的栈帧上直接构建 `a`，**而不是首先在 `GetTemp()` 的本地构建一个临时对象然后再将其复制或移动到 `a`（禁掉编译器优化后）。**

以上信息非常关键，这为返回值优化结合移动构造提供了大量的细节。

最后，需要注意的是，**“移动” 操作实际上是一种请求，因为有些类型不存在移动操作，对于这些对象会通过其复制操作来实现“移动”**  ，比如 `const std::string text`。

## 三、总结（gpt4.0）
本文详细探讨了C++中`std::move`和移动语义的概念、用法和注意事项。下面是对文章的总结：

### std::move
- `std::move`是一个模板函数，它的作用是将其参数无条件地转换为右值引用，从而使得移动语义可以被应用。这一转换仅在类型级别发生，不涉及实际的数据移动。
- `std::move`实现涉及到类型萃取，使用`type_traits`中的`std::remove_reference`来移除引用，然后将结果强制转换为右值引用。
- 使用`std::move`时，应避免对常量对象使用，因为这会降级为复制操作。

### 移动语义
- 移动语义允许资源（如动态内存）在对象间转移，而非复制，这可以显著提高性能，尤其是对于大型对象。
- 移动构造函数和移动赋值操作通常应声明为`noexcept`以确保在资源转移过程中不抛出异常，这是因为异常可能导致资源泄露或其他问题。
- 实现移动操作时，应确保正确处理自赋值的情况和资源的安全释放。

### 示例与应用
- 文章通过一个具体的例子展示了如何使用移动语义来处理大型数据对象的移动，而非复制。这包括了自定义的移动构造函数和移动赋值操作的实现。
- 另外，还展示了如何使用`std::move`来实现高效的资源交换（swap）操作，这在很多算法实现中非常有用。
- 文章还讨论了编译器优化（如返回值优化RVO）如何影响移动操作的行为，以及如何通过禁用优化来观察这些底层行为。

整体上，本文是对C++中移动语义和`std::move`的一个全面而深入的介绍，适合那些希望更好理解现代C++资源管理技术的开发者。

## 四、参考
参考[这里](https://www.cnblogs.com/5iedu/p/11318729.html)。

