---
title: C++：迭代器
date: 2024-04-23 21:45:57
tags:
    - c++
    - 开发语言
categories: C++
---

<!--more-->

## 〇、前言
在C++中，STL（标准模板库）提供了一组强大的数据结构和算法，其中使用了迭代器来实现数据访问和操作的通用接口。

迭代器是一种行为类似指针的对象，它允许对容器中的元素进行迭代（遍历）。STL定义了几种迭代器类型，包括输入迭代器、输出迭代器、前向迭代器、双向迭代器和随机访问迭代器。这些迭代器类型具有不同的功能和约束条件，使得它们适用于不同类型的数据结构和操作。

关于概念，它在C++中是一种抽象的约束，描述了一组类型所必须满足的要求。例如，输入迭代器概念描述了可以用于读取数据的迭代器的要求，而正向迭代器概念描述了可以向前遍历数据的迭代器的要求。这些概念允许编写通用的算法，这些算法可以适用于满足特定概念的任何类型。

对C++而言，这种双向迭代器是一种内置类型，不能从类派生而来。然而，从概念上看，它确实能够继承。有些STL文献使用术语改进（refinement）来表示这种概念上的继承，因此，双向迭代器是对正向迭代器概念的一种改进。

关于改进，它描述了一个概念可以是另一个概念的特殊情况。例如，双向迭代器是对正向迭代器的改进，因为它具有正向迭代器的所有功能，并且还可以向后遍历数据。

概念的具体实现被称为模型（model）。因此，指向 int 的常规指针是一个随机访问迭代器模型，也是一个正向迭代器模型，因为它满足该概念的所有要求。

**需要注意的是，各种迭代器的类型并不是确定的，而只是一种概念性描述。** 正如前面指出的，每个容器类都定义了一个类级 `typedef 名称——iterator`，因此 `vector<int>` 类的迭代器类型为`vector<int> ::interator`。然而，该类的文档将指出，矢量迭代器是随机访问迭代器，它允许使用基于任何迭代器类型的算法，因为随机访问迭代器具有所有迭代器的功能。

## 一、将指针用作迭代器
迭代器是广义指针，而指针满足所有的迭代器要求。迭代器是STL算法的接口，而指针是迭代器，**因此STL算法可以使用指针来对基于指针的非STL容器进行操作**。

```cpp
#include <algorithm>
#include <iostream>
#include <random>

const int SIZE = 10;
double Receipts[SIZE];

int main() {
    // 为数组添加随机值
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_real_distribution<double> dis(0.0, 1000.0);

    for (int i = 0; i < SIZE; i++) {
        Receipts[i] = dis(gen);
    }

    // 对数组进行排序
    std::sort(Receipts, Receipts + SIZE);

    // 输出排序后的数组
    for (int i = 0; i < SIZE; i++) {
        std::cout << Receipts[i] << " ";
    }

    return 0;
}
```

运行结果：

```bash
./main
63.9171 140.985 420.269 484.506 572.606 612.998 765.69 826.382 834.355 937.741
```


以上例子，`Receipts + n` 表达式是被定义的，只要 `n` 的值确保不超过数组的大小。这样就可以将 `Receipts + n` 作为迭代器传递给 STL 算法，从而对数组进行操作。

这种灵活性使得 STL 算法可以适用于各种不同类型的数据结构，而不仅仅局限于标准库中提供的容器。

有一种算法 `copy()` 可以将数据从一个容器复制到另一个容器中。这种算法是以迭代器方式实现的，所以它可以从一种容器到另一种容器进行复制，甚至可以在数组之间复制，因为可以将指向数组的指针用作迭代器。

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main() {
    int casts[10] = {6, 7, 2, 9, 4, 11, 8, 7, 10, 5};
    std::vector<int> dice(10);
    std::copy(casts, casts + 10, dice.begin());

    // 打印复制后的vector内容
    std::cout << "Copied array to vector: ";
    for (int value : dice) {
        std::cout << value << " ";
    }
    std::cout << std::endl;

    return 0;
}

```
前两个参数必须是（或最好是）输入迭代器，最后一个参数必须是（或最好是）输出迭代器。`copy()` 函数将**覆盖目标容器中已有的数据，同时目标容器必须足够大，以便能够容纳被复制的元素。**

现在要将信息复制到显示器上，**那么显示器就必须被当做一个容器，我们还要保证有一个指向这个容器的迭代器。**

如果有一个表示输出流的迭代器，则可以使用`copy( )`。好消息是，STL为这种迭代器提供了`ostream_iterator` 模板。

```cpp
#include <iterator>
#include <iostream>

int main() {
    std::ostream_iterator<int, char> out_iter(cout, " ");

    // 对 out_iter 进行操作，类似于 cout << 15 << " ";
    *out_iter++ = 15;
    ...
    return 0;
}

```
第一个模板参数（这里为int）指出了被发送给输出流的数据类型；第二个模板参数（这里为char）指出了输出流使用的字符类型。构造函数的第一个参数（这里为cout）指出了要使用的输出流；**最后一个字符串参数是在发送给输出流的每个数据项后显示的分隔符**。

对于常规指针，这意味着将15赋给指针指向的位置，然后将指针加1。但对于该`ostream_iterator`，这意味着将15和由空格组成的字符串发送到cout管理的输出流中，**并为下一个输出操作做好了准备（op++）**。可以将`copy()`用于迭代器：

```cpp
#include <algorithm>
#include <iostream>
#include <iterator>
#include <vector>

int main() {
    using std::cout;
    int casts[10] = {6, 7, 2, 9, 4, 11, 8, 7, 10, 5};
    std::vector<int> dice(10);
    std::copy(casts, casts + 10, dice.begin());

    // 打印复制后的vector内容
    cout << "Copied array to vector: ";
    for (int value : dice) {
        cout << value << " ";
    }
    cout << std::endl;

    // 声明迭代器
    std::ostream_iterator<int, char> out_iter(cout, " ");
    *out_iter++ = 15; // 对 out_iter 进行操作，类似于 cout << 15 << " ";

 	// 这意味着将dice容器的整个区间复制到输出流中，即显示容器的内容
    std::copy(dice.begin(), dice.end(), out_iter);

    cout << std::endl;

    return 0;
}

```
这意味着将`dice容器`的整个区间复制到输出流中，即显示容器的内容。
输出结构：

```bash
./main
Copied array to vector: 6 7 2 9 4 11 8 7 10 5
15 6 7 2 9 4 11 8 7 10 5
```

## 二、其它迭代器
假设要用反向打印这个容器中的内容，可以考虑反向迭代器：

```cpp
#include <algorithm>
#include <iostream>
#include <iterator>
#include <vector>

int main() {
    using std::cout;
    int casts[10] = {6, 7, 2, 9, 4, 11, 8, 7, 10, 5};
    std::vector<int> dice(10);
    std::copy(casts, casts + 10, dice.begin());

    // 打印复制后的vector内容
    cout << "Copied array to vector: ";
    for (int value : dice) {
        cout << value << " ";
    }
    cout << std::endl;

    // 声明迭代器
    std::ostream_iterator<int, char> out_iter(cout, " ");
    *out_iter++ = 15; // 对 out_iter 进行操作，类似于 cout << 15 << " ";

    std::copy(dice.begin(), dice.end(), out_iter);
    cout << std::endl;

    // 使用反向迭代器逆序输出 dice 中的元素
    std::copy(dice.rbegin(), dice.rend(), out_iter);
    cout << std::endl;

    return 0;
}

```

输出结果：

```bash
./main
Copied array to vector: 6 7 2 9 4 11 8 7 10 5
15 6 7 2 9 4 11 8 7 10 5
5 10 7 8 11 4 9 2 7 6
```
`rbegin()`和`end()`返回相同的值（超尾），但类型不同（`reverse_iterator`和`iterator`）。同样，`rend()`和`begin()`也返回相同的值（指向第一个元素的迭代器），但类型不同。

这里可能就有疑问，假设 rp 指向了 `rbegin()`，对 rp 进行解引用将会发生什么？因为 `rbegin() `返回超尾，因此不能对该地址进行解除引用。同样，如果 `rend()` 是第一个元素的位置，则 `copy()` 必须提早一个位置停止，因为区间的结尾处不包括在区间中。

**反向指针通过先递减，再解除引用解决了这两个问题。即*rp将在*rp的当前值之前对迭代器执行解除引用。** 这是一个例子：

```cpp
#include <iostream>
#include <iterator>
#include <vector>

int main() {
    using namespace std;

    int casts[10] = {6, 7, 2, 9, 4, 11, 8, 7, 10, 5};
    vector<int> dice(10);

    // 从数组复制到向量
    copy(casts, casts + 10, dice.begin());
    cout << "Let the dice be cast!\n";

    // 创建一个 ostream 迭代器
    ostream_iterator<int, char> out_iter(cout, " ");

    // 从 vector 复制到输出
    copy(dice.begin(), dice.end(), out_iter);
    cout << endl;

    // 隐式使用反向迭代器
    cout << "Implicit use of reverse iterator.\n";
    copy(dice.rbegin(), dice.rend(), out_iter);
    cout << endl;

    // 显式使用反向迭代器
    cout << "Explicit use of reverse iterator.\n";
    vector<int>::reverse_iterator ri;
    for (ri = dice.rbegin(); ri != dice.rend(); ++ri) {
        cout << *ri << ' ';
    }
    cout << endl;

    return 0;
}

```

运行结果：

```bash
./main
Let the dice be cast!
6 7 2 9 4 11 8 7 10 5
Implicit use of reverse iterator.
5 10 7 8 11 4 9 2 7 6
Explicit use of reverse iterator.
5 10 7 8 11 4 9 2 7 6
```

另外三种迭代器（`back_insert_iterator`、`front_insert_iterator` 和 `insert_iterator`）也将提高 STL 算法的通用性。很多STL函数都与 `copy()` 相似，将结果发送到输出迭代器指示的位置：

```cpp
copy(casts, casts + 10, dice.begin());
```
这些值将覆盖 `dice` 中以前的内容，且该函数假设 `dice` 有足够的空间，能够容纳这些值，即`copy()``不能自动根据发送值调整目标容器的长度。

上面的例子考虑到了这种情况，将 `dice` 声明为包含10个元素。然而，如果**预先并不知道dice的长度**，该如何办呢？**或者要将元素添加到dice中，而不是覆盖已有的内容**，又该如何办呢？

三种插入迭代器通过**将复制转换为插入**解决了这些问题。插入将添加新的元素，而不会覆盖已有的数据，并**使用自动内存分配来确保能够容纳新的信息**。`back_insert_iterator` 将元素插入到容器尾部，而 `front_insert_iterator` 将元素插入到容器的前端。最后，`insert_iterator`将元素插入到``insert_iterator` 构造函数的参数指定的位置前面。**这三个插入迭代器都是输出容器概念的模型。**

这里存在一些限制：
- `back_insert_iterator` 只能用于允许在尾部快速插入的容器，vector类符合这种要求。
- `front_insert_iterator` 只能用于允许在起始位置做时间固定插入的容器类型，vector类不能满足这种要求，但queue满足。
- `insert_iterator` 没有这些限制，因此可以用它把信息插入到 vector 的前端。然而，front_insert_iterator对于那些支持它的容器来说，完成任务的速度更快。

以上这些要求很重要，总之，迭代器对象要适配容器类型，就像要为不同的设备提供不同的螺丝刀以拆解它们。以下是一个例子：
```cpp
#include <algorithm>
#include <iostream>
#include <iterator>
#include <string>
#include <vector>

void output(const std::string &s) { std::cout << s << " "; }

int main() {
    using namespace std;

    string s1[4] = {"fine", "fish", "fashion", "fate"};
    string s2[2] = {"busy", "bats"};
    string s3[2] = {"silly", "singers"};
    vector<string> words(4);

    // 复制 s1 到 words
    copy(s1, s1 + 4, words.begin());
    for_each(words.begin(), words.end(), output);
    cout << endl;

    // 构造匿名的 back_insert_iterator 对象
    copy(s2, s2 + 2, back_insert_iterator<vector<string> >(words));
    for_each(words.begin(), words.end(), output);
    cout << endl;

    // 构造匿名的 insert_iterator 对象
    copy(s3, s3 + 2, insert_iterator<vector<string> >(words, words.begin()));
    for_each(words.begin(), words.end(), output);
    cout << endl;

    return 0;
}

```
运行结果：

```bash
./main
fine fish fashion fate
fine fish fashion fate busy bats
silly singers fine fish fashion fate busy bats
```

如果程序试图使用`words.end( )`（尾插）和 `words.begin( )`（头插）作为迭代器，将 s2 和 s3 复制到**words** 中，**words** 将没有空间来存储新数据，程序可能会由于内存违规而异常终止。

















