---
title: "C++ 语言笔记：完美转发与 std::forward（一）"
date: 2024-05-12 20:54:02
tags:
  - "C++"
  - "perfect forwarding"
  - "std::forward"
categories:
  - "Programming Languages"
---

<!--more-->

## 一、理解引用折叠　

### （一）引用折叠

　　1. 在C++中，**“引用的引用”是非法的**。像 `auto& &rx = x;`（注意两个&之间有空格）这种直接定义引用的引用是不合法的，但是编译器在通过类型别名或模板参数推导等语境中，**会间接定义出“引用的引用”，这时引用会形成“折叠”**。

　　2. 引用折叠会发生在**模板实例化**、**auto类型推导**、**创建和运用typedef和别名声明**、以及**decltype语境**中。

### （二）引用折叠规则
#### 　1. 两条规则

　　　　（1）**所有右值引用折叠到右值引用上仍然是一个右值引用**。如X&& &&折叠为X&&。

　　　　（2）**所有的其他引用类型之间的折叠都将变成左值引用**。如X& &, X& &&, X&& &折叠为X&。可见左值引用会传染，沾上一个左值引用就变左值引用了。根本原因：**在一处声明为左值，就说明该对象为持久对象，编译器就必须保证此对象可靠（左值）**。

#### 2. 利用引用折叠进行万能引用初始化类型推导

　　　　（1）当万能引用（T&& param）绑定到左值时，由于万能引用也是一个引用，而左值只能绑定到左值引用。因此，T会被推导为T&类型。从而param的类型为T& &&，引用折叠后的类型为T&。

　　　　（2）当万能引用（T&& param）绑定到右值时，同理，右值只能绑定到右值引用上，故T会被推导为T类型。**从而param的类型就是T&&**（右值引用）。

以下是一个例子：

```cpp
#include <iostream>

using namespace std;
class Widget {};
template <typename T> void func(T &&param) {}

// Widget工厂函数
Widget widgetFactory() { return Widget(); }

//类型别名
template <typename T> class Foo {
  public:
    typedef T &&RvalueRefToT;
};

int main() {
    int x = 0;
    int &rx = x;

    // 1. 引用折叠发生的语境1——模板实例化
    Widget w1;
    func(w1); // w1为左值，T被推导为Widget&。代入得void func(Widget& && param);
              //引用折叠后得void func(Widget& param)

    func(widgetFactory()); //传入右值，T被推导为Widget，代入得void func(Widget&&
                           // param) 注意这里没有发生引用的折叠。

    // 2. 引用折叠发生的语境2——auto类型推导
    auto &&w2 = w1; // w1为左值auto被推导为Widget&，代入得Widget& &&
                    // w2，折叠后为Widget& w2
    auto &&w3 = widgetFactory(); //函数返回Widget，为右值，auto被推导为Widget，代入得Widget
                                 //&&w3

    // 3. 引用折叠发生的语境3——tyedef和using

    Foo<int &> f1; // T被推导为 int&，代入得typedef int& &&
                   // RvalueRefToT;折叠后为typedef int& RvalueRefToT

    // 4. 引用折叠发生的语境3——decltype
    decltype(x) &&var1 = 10; //由于x为int类型，代入得int&& rx。
    decltype(rx) &&var2 =
        x; //由于rx为int&类型，代入得int& && var2，折叠后得int& var2

    return 0;
}

```

## 二、完美转发
### （一）std::forward 原型

```cpp
//左值版本
template<typename T>
T&& forward(typename remove_reference<T>::type& param)
{
    return static_cast<T&&>(param); //可能会发生引用折叠！
}

//右值版本
template<typename T>
T&& forward(typename remove_reference<T>::type&& param)
{
    return static_cast<T&&>(param); 
}
```

以上是 C++ 中 `std::forward` 函数模板的实现，这是完美转发的关键机制。完美转发是指在模板函数中将参数维持原样（保持其值类别——左值或右值）传递给另一个函数的技术。`std::forward` 通常在实现需要将参数转发到其他函数的模板中使用，尤其是在构造函数、函数模板和其他接受任意参数的场景中。

这段代码定义了两个重载版本的 `forward` 函数模板，一个用于左值，另一个用于右值。

#### 左值版本

```cpp
template<typename T>
T&& forward(typename remove_reference<T>::type& param)
{
    return static_cast<T&&>(param); //可能会发生引用折叠！
}
```

这里，`T` 是模板参数，而 `typename remove_reference<T>::type&` 表示去除 `T` 的引用部分后再加上左值引用。这确保了 `param` 是一个左值引用。

- **作用**：这个版本的 `forward` 用于将一个左值以保持其原始类型（左值或右值）的方式传递。当你传递一个左值给 `forward` 时，`param` 会**匹配到这个重载版本**。
- **引用折叠**：在 `return static_cast<T&&>(param);` 中使用 `T&&` 可以通过引用折叠规则处理左值和右值。如果 `T` 是左值引用类型，比如 `int&`，那么 `T&&` 折叠为 `int&`；如果 `T` 是非引用类型或右值引用类型，比如 `int` 或 `int&&`，那么 `T&&` 折叠为 `int&&`。

#### 右值版本

```cpp
template<typename T>
T&& forward(typename remove_reference<T>::type&& param)
{
    return static_cast<T&&>(param);
}
```

这里，`typename remove_reference<T>::type&&` 也去除了 `T` 的引用部分，但这次它是被右值引用修饰。由于这是一个右值引用到右值引用的匹配，所以它只会被右值触发。

- **作用**：这个版本的 `forward` 用于将一个右值以保持其类型（即右值）的方式传递。这是真正实现完美转发的关键部分，因为它确保了只有真正的右值才会被识别和处理为右值。
- **使用场景**：当 `forward` 被用于一个右值时，**它会触发这个重载**，并且 `param` 将传递为一个右值。

这两个版本的 `forward` 函数共同支持在模板中进行完美转发，**即不改变传入参数的值类别（左值或右值）。**这对于编写通用代码库、实现委托构造函数或任何需要保持参数值类别不变的场合至关重要。

### （二）完美转发的必要性
完美转发的主要优势在于**它允许我们编写可接收任意参数类型（包括其值类别）的函数模板，并且能够将这些参数原封不动地转发到其他函数。**这种技术特别适用于那些函数行为依赖于参数的值类别（左值或右值）的场景。如果没有完美转发，我们可能需要为不同类型的参数（如左值和右值）编写多个函数重载，这会使得代码更加冗长和复杂。

以下是一个使用完美转发的实例，展示它在实际编程中的应用和优势：

### 示例：泛型包装器
假设我们正在编写一个泛型包装器类，它可以封装任意类型的对象，并提供一个通用的接口来访问这些对象。我们希望能够直接在包装器内部构造这些对象，而不是先构造一个对象再将其复制到包装器中。

```cpp
template<typename T>
class Wrapper {
public:
    T value;

    template<typename... Args>
    Wrapper(Args&&... args) : value(std::forward<Args>(args)...) {}
};

struct ExpensiveToCopy {
    ExpensiveToCopy() {}
    ExpensiveToCopy(const ExpensiveToCopy&) {
        std::cout << "Copy constructor called!" << std::endl;
    }
    ExpensiveToCopy(ExpensiveToCopy&&) noexcept {
        std::cout << "Move constructor called!" << std::endl;
    }
};

int main() {
    ExpensiveToCopy etc;
    Wrapper<ExpensiveToCopy> w1(etc);  // 应调用复制构造
    Wrapper<ExpensiveToCopy> w2(std::move(etc));  // 应调用移动构造
}
```

在这个例子中，`Wrapper` 的构造函数使用完美转发来接收任意数量和类型的参数，并将它们转发给它封装的值的构造函数。这保证了当我们传入一个右值时，将使用移动构造函数而不是复制构造函数，从而提高效率。

如果没有完美转发，是达不到我们预期的效果的：

在不使用 `std::forward` 的情况下，尽管 `Args&&... args` 使用的是通用引用（也称作转发引用），它可以绑定到左值和右值。但在没有显式地指定 `std::forward` 来保持参数的左右值属性的情况下，**所有通过 `args...` 传递的参数在构造函数体内都会被当作左值处理。这是因为 `args` 是具名的变量，而具名的变量都是左值。** 理解这一点很重要：

```cpp
template<typename T>
class Wrapper {
public:
    T value;

    template<typename... Args>
    Wrapper(Args&&... args) : value(args...) {}  // 没有完美转发
};

struct ExpensiveToCopy {
    ExpensiveToCopy() {}
    ExpensiveToCopy(const ExpensiveToCopy&) {
        std::cout << "Copy constructor called!" << std::endl;
    }
    ExpensiveToCopy(ExpensiveToCopy&&) noexcept {
        std::cout << "Move constructor called!" << std::endl;
    }
};

int main() {
    ExpensiveToCopy etc;
    Wrapper<ExpensiveToCopy> w1(etc);  // 会调用复制构造
    Wrapper<ExpensiveToCopy> w2(std::move(etc));  // 也会调用复制构造，而不是移动构造
}
```

在这个修改后的示例中：

- `Wrapper<ExpensiveToCopy> w1(etc);` 显然调用复制构造函数，因为 `etc` 是一个左值。
- `Wrapper<ExpensiveToCopy> w2(std::move(etc));` 即使原始参数 `etc` 被转化为右值，但在 `Wrapper` 构造函数中 `args` 仍然是左值。**因此，尽管 `etc` 最初被转为右值，它在传递到 `ExpensiveToCopy` 的构造函数时又被当作左值处理，结果依然调用复制构造函数而不是移动构造函数**。

没有使用 `std::forward`，即使参数原本是右值，一旦传入 `Wrapper` 的构造函数，就丢失了其右值性质，从而导致不必要的复制。**这说明了完美转发的重要性，特别是在需要保留参数原始属性（如移动语义）的场合。**完美转发确保参数的值类别被保留和正确处理，从而**可以有效利用 C++ 的移动语义，减少不必要的性能开销**。

## 三、std::move和std::forward

### （一）两者比较

　　1. **move**和**forward**都是仅仅执行强制类型转换的函数。`std::move`无条件地将实参强制转换成右值。而`std::forward`则仅在某个特定条件满足时（传入func的实参是右值时）才执行强制转换（**本来都是具名参数，都是左值**）。

　　2. `std::move`并不进行任何移动，`std::forward` 也不进行任何转发。这两者在运行期都无所作为。它们不会生成任何可执行代码，连一个字节都不会生成。

### （二）使用时机

　　1. 针对右值引用的最后一次使用实施`std::move`，针对万能引用的最后一次使用实施`std::forward`。最后一次使用的意思是，在一个对象的生命周期中，你确定之后不再需要读取或修改这个对象的状态时。在这个时间点之后，对象的任何资源（如动态内存）都可以安全地转让给另一个对象。

　　2. 在按值返回的函数中，如果返回的是一个绑定到右值引用或万能引用的对象时，可以实施`std::move`或`std::forward`。因为如果原始对象是一个右值，它的值就应当被移动到返回值上，而如果是左值，就必须通过复制构造出副本作为返回值。这种情况可以用这个例子来进行解释：
　　

```cpp
#include <iostream>
#include <string>
#include <utility>
#include <vector>

// 返回局部右值引用对象，使用 std::move
std::vector<int> createVector() {
    std::vector<int> localVec = {1, 2, 3, 4, 5};
    return std::move(localVec); // 移动 localVec 到返回值
}

// 接收万能引用，返回相同类型
template <typename T>
T relay(T &&obj) {
    // std::forward 确保 obj 的值类别保持不变
    return std::forward<T>(obj);
}

int main() {
    auto vec = createVector(); // vec 通过移动构造器获取 localVec 的资源
    for (auto v : vec) {
        std::cout << v << " ";
    }
    std::cout << "\n";

    std::string str = "Hello, World!";
    auto result = relay(std::move(str)); // str 被视为右值，使用移动语义

    std::cout << "Result: " << result << " addr:" << &str << "\n";
    std::cout << "Original string: " << str << " addr:" << &str << " (moved)\n";

    std::string anotherStr = "Another test";
    auto anotherResult = relay(anotherStr); // anotherStr 仍为左值，使用复制语义

    std::cout << "AnotherStr " << anotherResult <<" addr:" << &anotherStr << "\n";
    std::cout << "Another result: " << anotherResult <<" addr:" << &anotherResult << "\n";

}

```
运行结果：

```bash
./main
1 2 3 4 5 
Result: Hello, World! addr:0x16d592db8
Original string:  addr:0x16d592db8 (moved)
AnotherStr Another test addr:0x16d592d88
Another result: Another test addr:0x16d592d70
```
从打印结果可以看到，**result** 的内存地址（资源地址）和 **str** 的相同，而**anotherstr** 与 **another** 的资源地址不同，**这说明前者使用的是移动语义，后者使用的是赋值语义**。

至于 **vec**，情况有点复杂，在不考虑返回值优化的情况时（后面后提到，事实上 **move** 操作会抑制返回值优化，是否会 **ROV**，取决于编译器），它会将 `createVector()` 的局部变量移动到临时变量返回值，然后利用移动语义构造 **vec**。总之，移动语义和 **ROV** 都很重要。

### （三）返回值优化（RVO）

　　1.两个前提条件

　　　　（1）**局部对象类型和函数返回值类型相同；**

　　　　（2）**返回的就是局部对象本身**（含局部对象或作为**return** 语句中的临时对象等）

　　2. 注意事项

　　　　（1）在**RVO**的前提条件被满足时，要么**避免复制，要么会自动地用std::move隐式实施于返回值**。

　　　　（2）按值传递的函数形参，把它们作为函数返回值时，情况与返回值优化类似。编译器这里会选择第2种处理方案，即返回时将形参转为右值处理。

　　　　（3）如果局部变量有资格进行**RVO**优化，就不要把`std::move`或`std::forward`用在这些局部变量中。**因为这可能会让返回值丧失优化的机会**。

用下面的例子来解释这些。

#### 版本一

```cpp
#include <iostream>
using namespace std;
class A {
    int data;

  public:
    A(int d = 0) : data(d) {}
    ~A() { cout << "destructor called for object " << this << endl; }
};
A creat() {
    A a;
    cout << "a_addr " << &a << endl;
    return move(a);
}
int main() {
    A aa = creat();
    cout << "a_addr " << &aa << endl;
    return 0;
}
```
运行结果：

```bash
g++  3.cxx -o main -std=c++11
./main
a_addr 0x16bdc6de4
destructor called for object 0x16bdc6de4
a_addr 0x16bdc6e28
destructor called for object 0x16bdc6e28
```
可以看到，这里析构了两次，第一次是 a，然后由于 ROV，直接在 creat() 调用位置得到了 aa，也就是通过 a 移动后得到的临时返回返回对象。
#### 版本二
仅仅去掉 move()：
```cpp
A creat() {
    A a;
    cout << "a_addr " << &a << endl;
    return a;
}
```
运行结果：

```bash
g++  3.cxx -o main -std=c++11
./main
a_addr 0x16d652e28
a_addr 0x16d652e28
destructor called for object 0x16d652e28
```
 这次ROV 直接放开了，直接在 creat()调用处构建对象，从这两个例子可以看到，如果可以 ROV，尽量不要使用 move()，这会降低性能。

#### 版本三
这次不修改代码，仅仅关闭返回值优化。

```cpp
A creat() {
    A a;
    cout << "a_addr " << &a << endl;
    return move(a);
}
```
编译运行：

```bash
g++ -fno-elide-constructors 3.cxx -o main -std=c++11
./main
a_addr 0x16db0edd4
destructor called for object 0x16db0edd4
destructor called for object 0x16db0ee24
a_addr 0x16db0ee28
destructor called for object 0x16db0ee28
```
可以看到这次析构了三次。第一次是临时变量 a（0x16db0edd4），然后**通过移动语义创建的返回值临时变量**（0x16db0ee24），接着是利用复制构造函数构造的对象 aa（0x16db0ee28）。可以看到这里的 move 仅仅增加了一点点性能（移动语义创建临时变量上）。

#### 版本四
这次不修改代码，仅仅关闭返回值优化。
```cpp
A creat() {
    A a;
    cout << "a_addr " << &a << endl;
    return a;
}
```

编译运行：

```bash
g++ -fno-elide-constructors 3.cxx -o main -std=c++11
./main
a_addr 0x16d542dd4
destructor called for object 0x16d542dd4
destructor called for object 0x16d542e24
a_addr 0x16d542e28
destructor called for object 0x16d542e28
```
这个和上面的区别就是，**没有使用移动语义创建临时返回对象**，其它都一样。

### （四）总结以上四个版本
这四个例子非常好地说明了 C++ 中关于返回值优化（RVO）、移动语义以及它们对程序性能的影响。下面是每个例子的详细分析和它们所揭示的关键概念：

#### 版本一：使用 `std::move` 返回局部对象
- **编译和运行结果：** 对象 `a` 的地址在 `creat()` 和 `main()` 函数中不同，说明发生了一次移动操作。
- **性能影响：** 显式使用 `std::move` 禁止了 RVO 的应用。尽管利用了移动语义，但仍然有额外的移动构造调用，导致两次析构：一次是 `a` 的析构，一次是 `aa` 的析构。

#### 版本二：正常返回局部对象
- **编译和运行结果：** `a` 的地址在 `creat()` 和 `main()` 中相同，表明直接在 `aa` 的存储位置构造了 `a`，没有发生复制或移动。
- **性能影响：** RVO 完全生效，避免了任何复制或移动操作，只有一次析构，即 `aa` 的析构。

#### 版本三：使用 `std::move` 且禁用 RVO
- **编译和运行结果：** 出现三次析构，首先是局部变量 `a`，然后是由 `std::move(a)` 生成的临时对象，最后是 `aa`。
- **性能影响：** 禁用 RVO 后，必须通过移动构造函数生成返回值的临时对象和最终的 `aa` 对象。这增加了构造和析构的调用次数，降低了性能。

#### 版本四：正常返回局部对象且禁用 RVO
- **编译和运行结果：** 与版本三类似，有三次析构，但所有对象都是通过复制构造函数创建，没有使用移动构造函数。
- **性能影响：** 禁用 RVO 后，每次返回都会创建一个新的对象实例。由于没有使用 `std::move`，所有对象都是通过复制构造，而不是移动构造，这通常更加耗费资源。

这些例子强调了几个重要点：
1. **返回值优化的重要性：** RVO 可以显著提高性能，通过避免不必要的复制和移动操作。
2. **`std::move` 的谨慎使用：** 在返回局部对象时，通常应避免使用 `std::move`，以允许编译器执行 RVO。使用 `std::move` 可能会阻止这种优化，除非确实需要（如返回类成员或函数参数）。
3. **编译器优化的智能性：** 现代编译器非常擅长优化，通常最好的做法是写出清晰直接的代码，让编译器为我们优化。
















