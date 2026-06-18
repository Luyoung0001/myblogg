---
title: "Unix 系统编程：理解 fork 函数"
date: 2023-03-11 21:23:10
tags:
  - "Unix"
  - "fork"
  - "process"
categories:
  - "Systems Programming"
---

<!--more-->

## fork()的基本概况
fork() 函数调用成功之后，会有两个返回值。当前进程，也就是父进程返回子进程的 pid，子进程返回 0。如果函数调用错误，返回为-1。

```c
#include <stdio.h>
#include <unistd.h>
int main(void) {
    int i = 0;
    printf("i\tson/pa\tppid\tpid\tfpid\n");
    for (i = 0; i < 2; i++) {
        pid_t fpid = fork();
        if (fpid == 0)
            printf("%d\tchild\t%4d\t%4d\t%4d\n", i, getppid(), getpid(), fpid);
        else
            printf("%d\tparent\t%4d\t%4d\t%4d\n", i, getppid(), getpid(), fpid);
    }
    return 0;
}
```

运行结果：

>i       son/pa  ppid    pid     fpid
0       parent  54861   57344   57345
0       child   57344   57345      0
1       parent  54861   57344   57346
1       parent  57344   57345   57347
1       child   57344   57346      0
1       child   57345   57347      0


这里做一下简单分析：
1、pid 为 57344 的进程 fork()之后，返回了 57345，这是一个子进程的 pid。
2、子进程的返回值为0，显然它的父进程 pid 为 57344。
3、for 循环继续执行；
4、此时 pid 为 56344 的进程又创建了一个子进程，子进程 pid 为 56346。
5、上一个 pid 为 56345 的子进程此时充当的是父进程，它创建了一个子进程，pid 为 56347。
6、然后，56346、56347 的进程继续执行，程序结束。

## fork()做的工作
```c
#include<unistd.h>
pid_t fork(void)

```
返回值：`pid_t` 是进程描述符，实质就是一个`int`，如果fork函数调用失败，返回一个负数，调用成功则返回两个值：0和子进程ID。
函数功能：以当前进程作为父进程创建出一个新的子进程，并且将父进程的所有资源拷贝给子进程，这样子进程作为父进程的一个副本存在。父子进程几乎时完全相同的，但也有不同的如父子进程ID不同。

## fork()之后
如果程序只是简单的新建一个几乎一摸一样的进程，那么这样的进程是没什么作用的。因此，如果能把新的子进程用其它程序替换掉，我们就成功地利用一个进程，创建了一个完全不同的子进程。关于进程替换，这里不再赘述，后续会再次提及。

