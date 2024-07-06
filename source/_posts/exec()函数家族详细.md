---
title: exec()函数家族详细
date: 2023-03-11 22:35:28
tags: linux
categories: 系统编程
---

<!--more-->

## 〇、前言
`fork` 函数之后，如果想要把子进程换成一个我想要执行的进程，这时，就不得不使用 `exec()`函数了，这也是 `fork()`的意义所在。当然，`exec`系列的函数也可以将当前进程替换掉，不一定非要` fork()`一个子进程。常见的` fork()`调用例子有很多，比如从 `wechat `发起一个语音电话、从 `bash `或者` zsh `执行一个 `a.out` 程序，都是在利用`exec`系统调用将新产生的子进程完全替换成目标进程。
比如，这是一个死循环程序（目的是为了观察，让它活得久一点）：

```c
#include <stdio.h>
int main() {
    int a = 0;
    while (1) {
        a++;
    }
    return 0;
}
```
通过编译，执行：

> gcc fork_example.c -o fork_example
> ./fork_example 

查看进程：`top`
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/1e066854046742f5bc02da847062add1_1720253831934.png?token=ANB4BCJXGNQNJ6IJ4XYAEETGRD64Y)
可以发现，fork_example的进程的 PPID 为 54861，我们看看它是谁：`ps 54861`

```c
  PID   TT  STAT      TIME COMMAND
54861 s018  Ss     0:00.23 /bin/zsh -il
```
很明显，它是 `zsh`，现在可以终止fork_example：` kill 57892`

> zsh: terminated  ./fork_example

程序就会结束！以上例子，可以看到我们的子进程，就是由一个父进程` fork()`之后替换的。
## 一、exec()

```c
#include<unistd.h>
```

原型：

```c
int execl(const char *path, const char *arg, ...)
int execv(const char *path, char *const argv[])
int execle(const char *path, const char *arg, ..., char *const envp[])
int execve(const char *path, char *const argv[], char *const envp[])
int execlp(const char *file, const char *arg, ...)
int execvp(const char *file, char *const argv[])
```
参数：

`path`参数表示你要启动程序的名称,包括路径名;
`arg`参数表示启动程序所带的参数，一般第一个参数为要执行命令名
返回值:成功返回0,失败返回-1
上述`exec系列函数`底层都是通过`execve系统调用`实现：

```c
#include <unistd.h>
int execve(const char *filename, char *const argv[],char *const envp[]);
```
①    查找方式：上表其中前4个函数的查找方式都是完整的文件目录路径，而最后2个函数（也就是以p结尾的两个函数）可以只给出文件名，系统就会自动从环境变量`“$PATH”`所指出的路径中进行查找。

②    参数传递方式：`exec`函数族的参数传递有两种方式，一种是逐个列举的方式，而另一种则是将所有参数整体构造成`指针数组`进行传递。

在这里参数传递方式是以函数名的第5位字母来区分的，字母为“l”（list）的表示逐个列举的方式，字母为“v”（vertor）的表示将所有参数整体构造成指针数组传递，然后将该数组的首地址当做参数传给它，数组中的最后一个指针要求是NULL。读者可以观察execl、execle、execlp的语法与execv、execve、execvp的区别。

③    环境变量：exec函数族使用了`系统默认`的环境变量，也可以传入指定的环境变量。这里以“e”（environment）结尾的两个函数`execle、execve`就可以在`envp[]`中指定当前进程所使用的环境变量替换掉该进程继承的所以环境变量，这极大地提供了灵活度。

## 二、execl()
该函数的定义为：
```c
int execl(const char *path, const char *arg, ...)
```
可以看到，它的参数为一个 `path`，由于不带 `p`，因此，最后一个参数为 `NULL`。
例如：

```c
#include <stdio.h>
#include <unistd.h>
int main() {
    printf("hello!\n");
    // 替换 main 进程
    execl("/bin/ls", "ls", "-a", NULL);
    // good bye! 并不会被打印出来
    printf("good bye!\n");
    return 0;
}
```
执行结果：

```bash
hello!
.               a.out           execlp.c        fork_example    myshell.c
..              execl.c         fork.c          fork_example.c
```

可以看到，它成功地执行了`"ls -a"`命令。

## 三、execlp()
该函数的定义为：

```c
int execlp(const char *file, const char *arg, ...)
```
该函数带` p`,第一个参数是一个 `*file`，说明不需要带完整路径，它会在默认环境变量里面自动查找：

```c
#include <stdio.h>
#include <unistd.h>
int main() {
    printf("hello!\n");
    // 替换 main 进程
    execl("ls","ls", "-a", NULL);
    // good bye! 并不会被打印出来
    printf("good bye!\n");
    return 0;
}
```
运行结果：

```bash
hello!
good bye!
```

说明，并没有成功替换，这是我们带个 p：

```c
#include <stdio.h>
#include <unistd.h>
int main() {
    printf("hello!\n");
    // 替换 main 进程
    execlp("ls","ls", "-a", NULL);
    // good bye! 并不会被打印出来
    printf("good bye!\n");
    return 0;
}
```
运行结果：

```bash
hello!
.               a.out           execlp.c        fork_example    myshell.c
..              execl.c         fork.c          fork_example.c
```
成功替换！其它的函数也是同理，就不再赘述了。

*全文完，感谢阅读！*















