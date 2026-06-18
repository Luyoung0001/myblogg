---
title: "C 结构体赋值问题：浅拷贝语义分析"
date: 2022-11-29 21:26:49
tags:
  - "C"
  - "struct"
  - "memory model"
categories:
  - "Programming Languages"
---

<!--more-->


## About the problem of struct structure assignment
### 1: Shallow copy
The abstract data structure instance of the struct structure can be directly assigned, see this example:

```c
#include <stdio.h>
#include <string.h>
typedef struct con{
    int a;
    int b;
    char c;
    double d;
}Con;
int main(){
    Con a, b;
    a.a = 1;
    a.b = 2;
    a.c = '3';
    a.d = 5.0;

    // assignment
    b = a;
    printf("%d %d %c %f\n", b.a, b.b, b.c, b.d);
    return 0;
}
```

> Operation result:
> 1 2 3 5.000000

Amazing right? Here is  an even more amazing thing:

```c
#include <stdio.h>
#include <string.h>
typedef struct con{
    int a;
    char string[100];
}Con;
int main(){
    Con a, b;
    a.a = 1;
    scanf("%s", a.string);
    // assignment
    b = a;
    printf("%d %s\n", b.a, b.string);
    return 0;
}
```
> Operation result:
> hellll
> 1 hellllo

Even structures nested with structures can also be assigned directly:

```c
#include <stdio.h>
#include <string.h>
typedef struct man{
    int age;
    char name[100];
}Man;
typedef struct con{
    int a;
    char string[100];
    Man man;
}Con;
int main(){
    Con a, b;
    a.man.age = 100;
    scanf("%s", a.man.name);
    a.a = 1;
    scanf("%s", a.string);
    b = a;

    // assignment
    b = a;
    a.man.age = 1000000;
    printf("%d %s %d %s\n", b.a, b.string, b.man.age, b.man.name);
    return 0;
}
```
> Operation result:
> Tony
helllooooooooooooo
1 helllooooooooooooo 100 Tony

I suspect that this kind of copy is just a shallow copy, then I did this experiment:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
struct MyStruct {
    int a;
    int b;
    char* c;
};
int main() {
    struct MyStruct t1;
    t1.a = 1;
    t1.b = 2;
    char* p = (char*)malloc(10 * sizeof(char));
    strcpy(p, "hello");
    t1.c = p;
    struct MyStruct t2;
    t2 = t1;
    printf("MyStruct t1: %d, %d, %s\n", t1.a, t1.b, t1.c);
    // Change the value of t1 at this time to see if t2 is affected
    // *(t1.c) = 'p';
    printf("MyStruct t2: %d, %d, %s\n", t2.a, t2.b, t2.c);
    printf("t1 pointer addr: %p\n", t1.c);
    printf("t2 pointer addr: %p\n", t2.c);
    return 0;
}
```
> Operation result:
> MyStruct t1: 1, 2, hello
MyStruct t2: 1, 2, hello
t1 pointer addr: 0x125e06760
t2 pointer addr: 0x125e06760

If `*(t1.c) = 'p';` is not commented, it will be like this:
> MyStruct t1: 1, 2, hello
MyStruct t2: 1, 2, pello
t1 pointer addr: 0x126606760
t2 pointer addr: 0x126606760

Obviously, t1 and t2 point to the same memory unit. At this time, if the content pointed to by t1 is changed, t2 will change accordingly. Therefore, this copy is just a simple assignment of the literal value, so this copy is a shallow copy .

### 2: The benefits of shallow copy
I came across this topic on PTA today:
**7-1 通讯录排序:**
>输入格式:
输入第一行给出正整数n（<10）。随后n行，每行按照“姓名 生日 电话号码”的格式给出一位朋友的信息，其中“姓名”是长度不超过10的英文字母组成的字符串，“生日”是yyyymmdd格式的日期，“电话号码”是不超过17位的数字及+、-组成的字符串。

>输出格式:
按照年龄从大到小输出朋友的信息，格式同输出。

> - 输入样例:
3
zhang 19850403 13912345678
wang 19821020 +86-0571-88018448
qian 19840619 13609876543
> - 输出样例:
wang 19821020 +86-0571-88018448
qian 19840619 13609876543
zhang 19850403 13912345678

Code size would be a disaster if there were no shallow copies of C structs in the world, like this:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
typedef struct info{
    char name[20];
    int birth;
    char tel[100];
}Info;
int main(){
    int n = 0;
    scanf("%d", &n);
    Info *list = (Info*)malloc(sizeof(Info)*n);
    for(int i = 0; i < n; i++){
        scanf("%s %d %s", list[i].name, &list[i].birth, list[i].tel);
    }
    // 排序
    for(int i = 0; i < n-1; i++){
        for(int j = i+1; j < n; j++){
            if(list[i].birth > list[j].birth){
                Info temp;
               strcpy(temp.name, list[i].name);
                temp.birth = list[i].birth;
                strcpy(temp.tel, list[i].tel);

                strcpy(list[i].name, list[j].name);
                list[i].birth = list[j].birth;
                strcpy(list[i].tel , list[j].tel);

                strcpy(list[j].name, temp.name);
                list[j].birth = temp.birth;
                strcpy(list[j].tel, temp.tel);
            }
        }
    }
    for(int i = 0; i < n; i++){
        printf("%s %d %s\n", list[i].name, list[i].birth, list[i].tel);
    }

    return 0;
}
```
And the shallow copy of the structure can simply let our code like this:

```c
#include <stdio.h>
#include <stdlib.h>
typedef struct info {
    char name[20];
    int birth;
    char tel[100];
} Info;
int main() {
    int n = 0;
    scanf("%d", &n);
    Info* list = (Info*)malloc(sizeof(Info) * n);
    for (int i = 0; i < n; i++) {
        scanf("%s %d %s", list[i].name, &list[i].birth, list[i].tel);
    }
    // Sorting
    for (int i = 0; i < n - 1; i++) {
        for (int j = i + 1; j < n; j++) {
            if (list[i].birth > list[j].birth) {
                Info temp = list[i];
                list[i] = list[j];
                list[j] = temp;
            }
        }
    }
    for (int i = 0; i < n; i++) {
        printf("%s %d %s\n", list[i].name, list[i].birth, list[i].tel);
    }
    return 0;
}
```


