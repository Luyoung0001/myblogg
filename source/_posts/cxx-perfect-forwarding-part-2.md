---
title: "C++ 语言笔记：完美转发与 std::forward（二）"
date: 2024-05-12 23:30:46
tags:
  - "C++"
  - "perfect forwarding"
  - "std::forward"
categories:
  - "Programming Languages"
---

<!--more-->

## 一、RVO优化和std::move、std::forward
以下是一个综合性的例子：

```cpp
#include <iostream>
#include <memory>
#include <ostream>
using namespace std;

// 1. 针对右值引用实施std::move，针对万能引用实施std::forward
class Data {};

class Widget {
    std::string name;
    std::shared_ptr<Data> ptr;

  public:
    Widget() {cout << "Widget() used for object@" << this
             << " addr_name: " << &(this->name) <<" string buffer addr: "<< static_cast<const void*>(this->name.data())<< endl;};

    //复制构造函数
    Widget(const Widget &w) : name(w.name), ptr(w.ptr) {
        cout << "Widget(const Widget& w) used for object@" << this
             << " addr_name: " << &(this->name) <<" string buffer addr: "<< static_cast<const void*>(this->name.data())<< endl;
    }
    //针对右值引用使用std::move
    Widget(Widget &&rhs) noexcept
        : name(std::move(rhs.name)), ptr(std::move(rhs.ptr)) {
        cout << "Widget(Widget&& rhs) used for object@" << this
             << " addr_name: " << &(this->name) <<" string buffer addr: "<<static_cast<const void*>(this->name.data())<< endl;
    }

    //针对万能引用使用std::forward。
    //注意，这里使用万能引用来替代两个重载版本：void setName(const
    // string&)和void setName(string&&)
    //好处就是当使用字符串字面量时，万能引用版本的效率更高。如w.setName("SantaClaus")，此时字符串会被
    //推导为const
    // char(&)[11]类型，然后直接转给setName函数（可以避免先通过字量面构造临时string对象）。
    //并将该类型直接转给name的构造函数，节省了一个构造和释放临时对象的开销，效率更高。
    template <typename T> void setName(T &&newName) {
        if (newName != name) { //第1次使用newName
            name = std::forward<T>(
                newName); //针对万能引用的最后一次使用实施forward
        }
    }
    ostream &print_addr_of_name(ostream &os) {
        cout << "addr of name: " << static_cast<const void*>(this->name.data()) << endl;
        return os;
    }
};

// 2. 按值返回函数
// 2.1 按值返回的是一个绑定到右值引用的对象
class Complex {
    double x;
    double y;

  public:
    Complex(double x = 0, double y = 0) : x(x), y(y) {}
    Complex &operator+=(const Complex &rhs) {
        x += rhs.x;
        y += rhs.y;
        return *this;
    }
};

Complex operator+(Complex &&lhs, const Complex &rhs) //重载全局operator+
{
    lhs += rhs;
    return std::move(lhs); //由于lhs绑定到一个右值引用，这里可以移动到返回值上。
}

// 2.2 按值返回一个绑定到万能引用的对象
template <typename T> auto test(T &&t) {
    return std::forward<T>(
        t); //由于t是一个万能引用对象。按值返回时实施std::forward
            //如果原对象一是个右值，则被移动到返回值上。如果原对象
            //是个左值，则会被拷贝到返回值上。
}

// 3. RVO优化
// 3.1 返回局部对象
Widget makeWidget() {
    Widget w;
    return w; //返回局部对象，满足RVO优化两个条件。为避免复制，会直接在返回值内存上创建w对象。
              //但如果改成return
              // std::move(w)时，由于返回值类型不同（Widget右值引用，另一个是Widget）
              //会剥夺RVO优化的机会，就会先创建w局部对象，再移动给返回值，无形中增加一个移动操作。
              //对于这种满足RVO条件的，当某些情况下无法避免复制的（如多路返回），编译器仍会默认地对
              //将w转为右值，即return std::move(w)，而无须用户显式std::move!!!
}

// 3.2 按值形参作为返回值
Widget makeWidget(Widget w) //注意，形参w是按值传参的。
{
    return w; //这里虽然不满足RVO条件（w是形参，不是函数内的局部对象），但仍然会被编译器优化。
              //这里会默认地转换为右值，即return std::move(w)
}

int main() {
    cout << "1. 针对右值引用实施std::move,针对万能引用实施std::forward" << endl;
    Widget w;
    w.setName("SantaClaus");
    cout << "w_addr:" << &w << endl;
    w.print_addr_of_name(cout);

    cout << "2. 按值返回时" << endl;
    auto t1 = test(w);
    auto t2 = test(std::move(w));
    cout << "t1_addr:" << &t1 << endl;
    t1.print_addr_of_name(cout);

    cout << "t2_addr:" << &t2 << endl;
    t2.print_addr_of_name(cout);

    cout << "3. RVO优化" << endl;

    Widget w1 = makeWidget(); //按值返回 局部对象（RVO）
    cout << "w1_addr:" << &w1 << endl;
    w1.print_addr_of_name(cout);

    cout << "w2:\n";
    Widget w2 = makeWidget(w1); //按值返回 按值形参对象

    cout << "w2_addr:" << &w2 << endl;
    w2.print_addr_of_name(cout);

    return 0;
}
```
打印结果：

```bash
./main
1. 针对右值引用实施std::move,针对万能引用实施std::forward
Widget() used for object@0x16dd46df0 addr_name: 0x16dd46df0 string buffer addr: 0x16dd46df0
w_addr:0x16dd46df0
addr of name: 0x16dd46df0
2. 按值返回时
Widget(const Widget& w) used for object@0x16dd46db8 addr_name: 0x16dd46db8 string buffer addr: 0x16dd46db8
Widget(Widget&& rhs) used for object@0x16dd46d90 addr_name: 0x16dd46d90 string buffer addr: 0x16dd46d90
t1_addr:0x16dd46db8
addr of name: 0x16dd46db8
t2_addr:0x16dd46d90
addr of name: 0x16dd46d90
3. RVO优化
Widget() used for object@0x16dd46d68 addr_name: 0x16dd46d68 string buffer addr: 0x16dd46d68
w1_addr:0x16dd46d68
addr of name: 0x16dd46d68
w2:
Widget(const Widget& w) used for object@0x16dd46d18 addr_name: 0x16dd46d18 string buffer addr: 0x16dd46d18
Widget(Widget&& rhs) used for object@0x16dd46d40 addr_name: 0x16dd46d40 string buffer addr: 0x16dd46d40
w2_addr:0x16dd46d40
addr of name: 0x16dd46d40
```
需要注意的是，`Widget{}`中的：

```cpp
Widget(Widget&& rhs) noexcept: name(std::move(rhs.name)), ptr(std::move(rhs.ptr))
    {
        cout << "Widget(Widget&& rhs)" << endl;
    }
```
这个函数会利用一个右值对象来构造新的对象，其中`name()`会调用**string** 的移动构造函数来创建新的 **string** 对象，**这两个对象的地址是不同的，但是 string 指向的缓冲区也就是字符串存储地址是相同的，这点需要注意**。
 
string 的移动构造函数：
```cpp
MyString(MyString&& other) noexcept : data(other.data) {
        other.data = nullptr;
    }
```

但是如果仔细观察，会发现 `string.data()` 并不会一样，尽管是通过移动构造的，这是因为**SSO**（小字符串优化）。**SSO** 的具体阈值取决于 `std::string` 的实现，通常在 15 到 24 个字符之间。如果字符串长度低于此阈值，字符串内容将存储在对象本身的内部缓冲区中；超过此阈值，则使用动态内存分配。因为事实是，**移动构造在某些情况下，并不会比复制构造更高效**。

当我们将上述示例的字符变长时：

```cpp
class Widget {
    std::string name = "SantaClausSantaClausSantaClausSantaClausSantaClausSantaClausSantaClausSantaClaus";
    ...
}
int main() {
    ...
    ...
    Widget w;
    w.setName("SantaClausSantaClausSantaClausSantaClaus");
 ...
}
```
编译运行：

```bash
./main
1. 针对右值引用实施std::move,针对万能引用实施std::forward
Widget() used for object@0x16f89adf0 addr_name: 0x16f89adf0 string buffer addr: 0x1286066c0
w_addr:0x16f89adf0
addr of name: 0x1286066c0
2. 按值返回时
Widget(const Widget& w) used for object@0x16f89adb8 addr_name: 0x16f89adb8 string buffer addr: 0x128606720
Widget(Widget&& rhs) used for object@0x16f89ad90 addr_name: 0x16f89ad90 string buffer addr: 0x1286066c0
t1_addr:0x16f89adb8
addr of name: 0x128606720
t2_addr:0x16f89ad90
addr of name: 0x1286066c0
3. RVO优化
Widget() used for object@0x16f89ad68 addr_name: 0x16f89ad68 string buffer addr: 0x128606750
w1_addr:0x16f89ad68
addr of name: 0x128606750
w2:
Widget(const Widget& w) used for object@0x16f89ad18 addr_name: 0x16f89ad18 string buffer addr: 0x1286067b0
Widget(Widget&& rhs) used for object@0x16f89ad40 addr_name: 0x16f89ad40 string buffer addr: 0x1286067b0
w2_addr:0x16f89ad40
addr of name: 0x1286067b0
```
这里的运行结果，符合所有的预期。

## 二、完美转发失败的情形

### （一）完美转发失败

　　1. 完美转发不仅转发对象，**还转发其类型、左右值特征以及是否带有const或volation等修饰词。**而完美转发的失败，主要**源于模板类型推导失败或推导的结果是错误的类型**。

　　2. 实例说明：假设转发的目标函数`f`，而转发函数为`fwd`（**天然就应该是泛型**）。函数如下：

```cpp
template<typename… Ts>
void fwd(Ts&&… params)
{
     f(std::forward<Ts>(params)…);
}

f(expression);    //如果本语句执行了某操作
fwd(expression);  //而用同一实参调用fwd则会执行不同操作，则称完美转发失败。
```

### （二）五种完美转发失败的情形
#### 1. 使用大括号初始化列表时

　　（1）失败原因分析：由于转发函数是个模板函数，而在模板类型推导中，大括号初始**不能自动被推导为std::initializer_list<T>**。

　　（2）解决方案：**先用auto声明一个局部变量，再将该局部变量传递给转发函数**。

#### 2. 0和NULL用作空指针时

　　（1）失败原因分析：**0或NULL**以空指针之名传递给模板时，类型推导的结果是整型，而不是所希望的指针类型。

　　（2）解决方案：传递**nullptr**，而非**0或NULL**。

#### 3. 仅声明static const 整型成员变量，而无其定义时。

　　（1）失败原因分析：**C++中常量一般是进入符号表的，只有对其取地址时才会实际分配内存。** 调用 `f` 函数时，**其实参是直接从符号表中取值，此时不会发生问题**。但当调用 `fwd` 时由于其形参是万能引用，而引用本质上是一个可解引用的指针。 **因此当传入 `fwd` 时会要求准备某块内存以供解引用出该变量出来。但因其未定义，也就没有实际的内存空间， 编译时可能失败（取决于编译器和链接器的实现）。**

　　（2）解决方案：**在类外定义该成员变量**。注意这声变量在声明时一般会先给初始值。因此定义时无需也不能再重复指定初始值。

#### 4. 使用重载函数名或模板函数名时

　　（1）失败原因分析：由于 `fwd` 是个模板函数，其形参没有任何关于类型的信息。当传入重载函数名或模板函数（代表许许多多的函数）时，就会导致 `fwd` 的形参不知绑定到哪个函数上。

　　（2）解决方案：**在调用fwd调用时手动为形参指定类型信息**。

#### 5. 转发位域时

　　（1）失败原因分析：位域是由机器字的若干任意部分组成的（如32位int的第3至5个比特），但这样的实体是无法直接取地址的。**而fwd的形参是个引用，本质上就是指针，所以也没有办法创建指向任意比特的指针。**

　　（2）解决方案：制作位域值的副本，并以该副本来调用转发函数。
　　
以下例子分别解释了这些情况：

```cpp
#include <iostream>
#include <vector>

using namespace std;

// 1. 大括号初始化列表
void f(const std::vector<int> &v) {
    cout << "void f(const std::vector<int> & v)" << endl;
}

// 2. 0或NULL用作空指针时
void f(int x) { cout << "void f(int x)" << endl; }

// 3. 仅声明static const的整型成员变量而无定义
class Widget {
  public:
    static const std::size_t MinVals;//仅声明，无定义（因为静态变量需在类外定义！）
};

const std::size_t Widget::MinVals = 10;

// 4. 使用重载函数名或模板函数名
int f(int (*pf)(int)) {
    cout << "int f(int(*pf)(int))" << endl;
    return 0;
}

int processVal(int value) { return 0; }
int processVal(int value, int priority) { return 0; }

// 5.位域
struct IPv4Header {
    std::uint32_t version : 4, IHL : 4, DSCP : 6, ECN : 2, totalLength : 16;
    //...
};

template <typename T>
T workOnVal(T param) //函数模板，代表许许多多的函数。
{
    return param;
}

//用于测试的转发函数
template <typename... Ts>
void fwd(Ts &&...param) //转发函数
{
    f(std::forward<Ts>(param)...); //目标函数
}

int main() {
    cout << "-------------------1. 大括号初始化列表---------------------"
         << endl;
    // 1.1 用同一实参分别调用f和fwd函数
    f({1, 2, 3}); //{1, 2, 3}会被隐式转换为std::vector<int>
    // fwd({ 1, 2, 3 });
    // //编译失败。由于fwd是个函数模板，而模板推导时{}不能自动被推导为std:;initializer_list<T>
    // 1.2 解决方案
    auto il = {1, 2, 3};
    fwd(il);

    cout << "-------------------2. 0或NULL用作空指针-------------------"
         << endl;
    // 2.1 用同一实参分别调用f和fwd函数
    // f(NULL);   //调用void f(int)函数，
    fwd(NULL); // NULL被推导为int，仍调用void f(int)函数
    // 2.2 解决方案：使用nullptr
    f(nullptr); //匹配int f(int(*pf)(int))
    fwd(nullptr);

    cout << "-------3. 仅声明static const的整型成员变量而无定义--------"
         << endl;
    // 3.1 用同一实参分别调用f和fwd函数
    f(Widget::MinVals); //调用void f(int)函数。实参从符号表中取得，编译成功！
    fwd(Widget::
            MinVals); // fwd的形参是引用，而引用的本质是指针，但fwd使用到该实参时需要解引用
                      //这里会因没有为MinVals分配内存而出现编译失败（取决于编译器和链接器）
    // 3.2 解决方案：在类外定义该变量

    cout << "-------------4. 使用重载函数名或模板函数名---------------" << endl;
    // 4.1 用同一实参分别调用f和fwd函数
    f(processVal); // ok，由于f形参为int(*pf)(int)，带有类型信息，会匹配int
                   // processVal(int value)
    // fwd(processVal);
    // //error,fwd的形参不带任何类型信息，不知该匹配哪个processVals重载函数。
    // fwd(workOnVal);
    // //error,workOnVal是个函数模板，代表许许多多的函数。这里不知绑定到哪个函数
    // 4.2 解决方案：手动指定类型信息
    using ProcessFuncType = int (*)(int);
    ProcessFuncType processValPtr = processVal;
    fwd(processValPtr);
    fwd(static_cast<ProcessFuncType>(workOnVal)); //调用int f(int(*pf)(int))

    cout << "----------------------5. 转发位域时---------------------" << endl;
    // 5.1 用同一实参分别调用f和fwd函数
    IPv4Header ip = {};
    f(ip.totalLength); //调用void f(int)
    // fwd(ip.totalLength);
    // //error，fwd形参是引用，由于位域是比特位组成。无法创建比特位的引用！
    //解决方案：创建位域的副本，并传给fwd
    auto length = static_cast<std::uint16_t>(ip.totalLength);
    fwd(length);

    return 0;
}

```

运行结果：

```bash
./main
-------------------1. 大括号初始化列表---------------------
void f(const std::vector<int> & v)
void f(const std::vector<int> & v)
-------------------2. 0或NULL用作空指针-------------------
void f(int x)
int f(int(*pf)(int))
int f(int(*pf)(int))
-------3. 仅声明static const的整型成员变量而无定义--------
void f(int x)
void f(int x)
-------------4. 使用重载函数名或模板函数名---------------
int f(int(*pf)(int))
int f(int(*pf)(int))
int f(int(*pf)(int))
----------------------5. 转发位域时---------------------
void f(int x)
void f(int x)
```

这样修改后，就可以实现完美转发了。

## 三、参考
[这里](https://www.cnblogs.com/5iedu/p/11324772.html)。


