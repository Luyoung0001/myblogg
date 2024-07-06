---
title: MIT 6.S081学习笔记（第〇章）
date: 2023-09-07 23:04:29
tags: 学习 笔记 操作系统 MIT 6.S081
categories: OS 系统编程 Unix/Linux
---

<!--more-->

## 〇、前言
- 本文涉及 xv6 《第零章 操作系统接口》相关，主要对涉及的进程、I/O、文件描述符、管道、文件等内容产生个人理解，不具有官方权威解释；
- 文章的目录与书中的目录没有严格的相关性；
- 文中会有问题 (Question) 字段，这来源于对 xv6 book 的扩展；
- 文中涉及的代码均能在macOS 12.5 M1 Apple Silicon 运行，文中涉及的所有代码的运行也在该环境，其它平台未测试未知。
- xv6的很多代码都可以运行在其它类 Unix 上。
## 一、进程与内存
一个进程可以通过系统调用 fork()来创建一个新的进程——子进程。fork()函数会有两个返回值，一个被子进程获取，另一个被父进程获取。对于父进程它返回子进程的 pid，对于子进程它返回 0。考虑下面这段代码：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>
int main() {
    int pid;
    pid = fork();
    if (pid > 0) {
        // 父进程
        printf("这是父进程:子进程的 id 为:%d\n", pid);
        int status; 
        pid = wait(&status);
        if (WIFEXITED(status)) {
            int exit_status = WEXITSTATUS(
                status); 
            printf("子进程 %d 已经退出，退出的状态码为:%d\n", pid, exit_status);
        }
    } else if (pid == 0) {
        // 子进程
        printf("子进程正在运行!\n");
        exit(4); // 设置一个状态码,注意不要溢出~
        // 自定义退出状态码，退出状态码最高是255，一般自定义的代码值为0~255，如果超出255，则返回该数值被256除了之后的余数
    } else {
        printf("fork()出错了~\n");
    }
}
// 可能以任意顺序被打印，这种顺序由父进程或子进程谁先结束 printf
// 决定。当子进程退出时，父进程的 wait 也就返回了.
```
运行结果：

> 这是父进程:子进程的 id 为:43670
子进程正在运行!
子进程 43670 已经退出，退出的状态码为:4


### 问题 1：僵尸进程如何产生？
在 UNIX 系统中，一个进程结束了，但是他的父进程没有等待(调用wait()、waitpid())它, 那么它将变成一个僵尸进程。
在 fork()、execve() 过程中，假设子进程结束时父进程仍存在，而父进程 fork() 之前既没安装SIGCHLD信号处理函数调用 waitpid() 等待子进程结束，又没有显式忽略该信号，则子进程成为僵尸进程。

使用 top 命令查看时有一栏为 S ,如果状态为 Z 说明它就是僵尸进程。
在 macOS 上可以使用 `ps -A -ostat,ppid,pid | grep -e '^[Zz]'`来打印僵尸进程。

以下是一个创造僵尸进程的程序：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>
int main() {
    int pid;
    pid = fork();

    if (pid > 0) {
        printf("父进程正在运行,pid:%d\n", getpid());
        while(1);

    } else if (pid == 0) {
        printf("子进程正在运行,pid:%d\n", getpid());
        exit(4);
    } else {
        printf("fork()出错了~\n");
    }
}
```
运行结果：

> 父进程正在运行,pid:45316
子进程正在运行,pid:45317

我们在终端输入：`ps -A -ostat,ppid,pid | grep -e '^[Zz]'`：打印僵尸进程的父进程、僵尸进程，结果为：

```bash
****** ~ % ps -A -ostat,ppid,pid | grep -e '^[Zz]'
Z+   45316 45317
```
可以看到，确实产生了僵尸进程。

### 问题 2：如何 Kill 僵尸进程？
首先使用 kill 命令试试：

```bash
****** ~ % kill 45317
****** ~ % ps -A -ostat,ppid,pid | grep -e '^[Zz]'
Z+   45316 45317
****** ~ % kill -9 45317                          
****** ~ % ps -A -ostat,ppid,pid | grep -e '^[Zz]'
Z+   45316 45317
```
可见，根本杀不死这个僵尸进程。

一般僵尸进程很难直接 kill 掉，不过您可以kill僵尸进程的父进程。父进程死后，僵尸进程成为”孤儿进程**”，过继给1号进程init，init 始终会负责清理僵尸进程**。它产生的所有僵尸进程也跟着消失。

```bash
****** ~ % kill 45316                             
****** ~ % ps -A -ostat,ppid,pid | grep -e '^[Zz]'

```
成功地间接杀死了僵尸进程！

### 问题 3：如何避免僵尸进程？
这是一个比较臃肿的问题。主要有两种方法来避免僵尸进程：
#### 1、两次 fork() 来避免僵尸进程
很显然，当 fork() 一次时，存在父进程和子进程，这时候，有两种选择来避免僵尸进程：
- 父进程调用 wait()、waitpid()等函数来接收子进程退出状态；
- 父进程结束后，子进程自动托管到 init 进程。

如果父进程没有处理子进程退出的状态，在父进程退出之前，子进程一直处于僵尸状态。
这意味着，即使父进程有调用 wait()等函数，但是子进程退出后，父进程还没运行到相关代码，子进程也会存在僵尸进程的状态。

那么如何应该创建子进程，才能保证子进程不会变成僵尸进程呢？**两次 fork() 就可以做到**。

父进程P fork() 之后产生的一个子进程S，S 立即调用 wait() 函数，接着 fork()，产生孙子进程GS，然后S进程立即执行 exit(0)。这样，进程 S 就会顺利结束。这时，**由于孙子进程没有了父进程（S），就会变成孤儿进程，被 init 托管**。于是父进程 P 和孙子进程GS没有任何继承关系了，它们的父进程都变成了 init 进程。

以下程序就是一个简单的实验：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>
int main() {
    int pid;
    pid = fork();

    if (pid > 0) {
        // 父进程
        printf("父进程正在运行,pid:%d ppid:%d\n", getpid(), getppid());
        int status;
        wait(&status);
        while (1) {
            sleep(1);
            printf("父进程正在运行,pid:%d ppid:%d\n", getpid(), getppid());
        }

    } else if (pid == 0) {
        int pid1 = fork();
        if (pid1 > 0) {
          // 子进程直接结束
            exit(0);
        } else if (pid1 == 0) {
            // 孙子进程
            printf("孙子进程正在运行,pid:%d ppid:%d\n", getpid(), getppid());
            while (1) {
                sleep(1);
                printf("孙子进程正在运行,pid:%d ppid:%d\n", getpid(),
                       getppid());
            }
        } else {
            printf("进程S fork()出错了~\n");
        }

    } else {
        printf("进程P fork()出错了~\n");
    }
}
```
运行结果：

> ****** chap0 % ./main               
父进程正在运行,pid:47198 ppid:18751
孙子进程正在运行,pid:47203 ppid:1
孙子进程正在运行,pid:47203 ppid:1
父进程正在运行,pid:47198 ppid:18751
父进程正在运行,pid:47198 ppid:18751
孙子进程正在运行,pid:47203 ppid:1
...

可以看到，**孙子进程成功地被 init 进程托管**。而且也没有产生僵尸进程：

```bash
****** ~ % ps -A -ostat,ppid,pid | grep -e '^[Zz]'
****** ~ %
```

#### 2、通过信号机制来避免僵尸进程
1）在父进程 fork() 之前安装`SIGCHLD`信号处理函数，并在此handler函数中调用waitpid()等待子进程结束，这样，内核才能获得子进程退出信息从而释放那个进程描述符；

2）设置`SIGCHLD`信号为`SIG_IGN`（即，忽略SIGHLD信号），系统将不产生僵尸进程。通过signal(`SIGCHLD`, `SIG_IGN`)通知内核对子进程的结束不关心，由内核回收。该信号是子进程退出的时候向父进程发送的。常用于并发服务器的性能的一个技巧因为并发服务器常常fork很多子进程，子进程终结之后需要服务器进程去 wait() 清理资源。如果将此信号的处理方式设为忽略，可让内核把僵尸子进程转交给init进程去处理，省去了大量僵尸进程占用系统资源。

比如：对于服务器进程，如果父进程不等待子进程就结束，子进程将成为僵尸进程；若父进程等待子进程结束，就会影响服务器进程的并发性能。所以此时一般就将`SIGCHLD`信号设置为　`SIG_IGN`。

将 `SIGCHLD` 信号设置为 `SIG_IGN` 后，内核会自动处理子进程的退出，包括回收子进程的资源。内核会在子进程退出时，将子进程的退出状态丢弃，不再保存它的信息，**因此不会创建僵尸进程**。

但是大多情况下，我们**仍然希望能收到子进程的退出信**息，这时候可以设置一个信号处理函数 `handler_func()`，里面可以专门为子进程收尸，以下是一个例子：

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

// 信号处理函数
void sigchld_handler(int signo) {
    int status; // 接收子进程的退出状态
    pid_t pid;

    // 等待所有子进程退出，避免成为僵尸进程
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        if (WIFEXITED(status)) {
            printf("子进程 %d 已经退出，退出的状态码为:%d\n", pid,
                   WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("子进程 %d 被信号终止，信号编号为:%d\n", pid,
                   WTERMSIG(status));
        }
    }
}

int main() {
    // 注册信号处理函数
    signal(SIGCHLD, sigchld_handler);

    // 创建子进程
    int pid = fork();

    if (pid == 0) {
        // 子进程
        printf("子进程正在运行pid:%d!\n",getpid());
        sleep(10); // 模拟子进程工作
        exit(42); // 子进程退出
    } else if (pid < 0) {
        perror("fork");
        return 1;
    } else {
        // 父进程
        while (1) {
            // 做其他工作
            printf("父进程在运行!\n");
            sleep(1);
        }
    }
}

```
运行之后，通过 kill 命令杀死子进程，运行结果：

> ****** chap0 % ./main               
父进程在运行!
子进程正在运行pid:48472!
父进程在运行!
父进程在运行!
子进程 48472 被信号终止，信号编号为:15
父进程在运行!
父进程在运行!
...

kill 命令是通过**向进程发送指定的信号**来结束相应进程的。 在默认情况下，采用编号为15的TERM信号。 

### 问题 4： 为什么 kill 不能终止僵尸进程？
kill -9 发送SIGKILL信号将其终止，但是以下两种情况不起作用：
- 1、该进程处于"Zombie"状态（使用ps命令返回defunct的进程）。此时进程已经释放所有资源，但还未得到其父进程的确认。"zombie"进程要等到下次重启时才会消失，但它的存在不会影响系统性能。
- 2、该进程处于"kernel mode"（核心态）且在等待不可获得的资源。处于核心态的进程忽略所有信号处理，因此对于这些一直处于核心态的进程只能通过重启系统实现。进程会处于两种状态，即用户态和核心态。只有处于用户态的进程才可以用“kill”命令将其终止。

系统调用 exec 将从某个文件（通常是可执行文件）里读取内存镜像，并将其替换到调用它的进程的内存空间,这份文件必须符合特定的格式。xv6 使用 ELF 文件格式，当exec执行成功后，它并不返回到原来的调用进程，而是从ELF头中声明的入口开始，执行从文件中加载的指令。exec 接受两个参数：可执行文件名和一个字符串参数数组。以下是一个案例：

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    char *argv[3];
    argv[0] = "/bin/echo"; // 第一个参数是可执行文件的路径
    argv[1] = "hello";     // 第二个参数是命令的参数
    argv[2] = NULL;        // 参数数组的最后一个元素必须是 NULL

    execv(argv[0], argv); // 执行 /bin/echo 命令并传递参数

    // 如果execv失败，才会执行下面的代码
    printf("exec error\n");
    return 1;
}
```
运行结果：

> ****** chap0 % ./main            
hello

这段代码将调用程序替换为“/bin/echo”这个程序，这个程序的参数列表为“hello”。

xv6 shell 用以上调用为用户执行程序。shell 的主要结构很简单，主循环通过 getcmd 读取命令行的输入，然后它调用 fork 生成一个 shell 进程的副本。父 shell 调用 wait，而子进程执行用户命令。
举例来说，用户在命令行输入“echo hello”，getcmd 会以 echo hello 为参数调用 runcmd(), 由 runcmd 执行实际的命令。对于 “echo hello“， runcmd 将调用 exec 。如果 exec 成功被调用，子进程就会转而去执行 echo 程序里的指令。在某个时刻 echo 会调用 exit，这会使得其父进程从 wait 返回。源代码如下：
```c
int main(void) {
    static char buf[100];
    int fd;
    // Ensure that three file descriptors are open.
    while ((fd = open("console", O_RDWR)) >= 0) {
        if (fd >= 3) {
            close(fd);
            break;
        }
    }

    // Read and run input commands.
    while (getcmd(buf, sizeof(buf)) >= 0) {
        if (buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' ') {
            // Chdir must be called by the parent, not the child.
            buf[strlen(buf) - 1] = 0; // chop \n
            if (chdir(buf + 3) < 0)
                fprintf(2, "cannot cd %s\n", buf + 3);
            continue;
        }
        if (fork1() == 0)
            runcmd(parsecmd(buf));
        wait(0);
    }
    exit(0);
}
```
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/33901d1345134a7791b4c470b9de4491_1720253724529.png?token=ANB4BCJCA3VQHPIBA5GCCTLGRD6V2)

xv6 通常隐式地分配用户的内存空间。**fork 在子进程需要装入父进程的内存拷贝时分配空间**，**exec 在需要装入可执行文件时分配空间**。

xv6 没有用户这个概念当然更没有不同用户间的保护隔离措施。按照 Unix 的术语来说，所有的 xv6 进程都以 **root 用户**执行
## 二、I/O 和文件描述符
文件描述符是一个高级抽象，它代表了一个进程可以读写的被内核管理的对象，被设计成一个**整数**。
进程可以通过**多种方式**获得一个文件描述符。打开文件、目录、设备，或者创建一个管道（pipe），或者复制已经存在的文件描述符，都可以得到一个文件描述符。简单起见，我们常常把文件描述符指向的对象称为“文件”。文件描述符的接口是对文件、管道、设备等的抽象，这种抽象使得它们看上去**都是同一个东西**。

每个进程都一张表，而 xv6 内核就以文件描述符作为这张表的索引，所以每个进程都有一个**从0开始的文件描述符空间**。进程从文件描述符0读入（**标准输入**），从文件描述符1输出（**标准输出**），从文件描述符2输出错误（**标准错误输出**）。

xv6 的 shell 利用了这个习惯来实现 I/O 重定向，shell 保证任何时候都有 3 个打开的文件描述符：

```c
// Ensure that three file descriptors are open.
  while((fd = open("console", O_RDWR)) >= 0){
    if(fd >= 3){
      close(fd);
      break;
    }
  }
```

系统调用 read() 和 write()  从文件描述符所指的文件中读或者写 n 个字节。read(fd, buf, n) 从 fd 读最多 n 个字节（fd 可能没有 n 个字节），将它们拷贝到 buf 中，然后返回读出的字节数。

**每一个指向文件的文件描述符都和一个偏移关联**。read() 从当前文件偏移处读取数据，然后把**偏移**增加读出字节数。紧随其后的 read() 会从新的起点开始读数据。当没有数据可读时，read ()就会返回0，这就表示文件结束了。

write(fd, buf, n) 写 buf 中的 n 个字节到 fd 并且返回实际写出的字节数。如果返回值小于 n 那么只可能是发生了错误。就像 read() 一样，write() 也从当前文件的**偏移**处开始写，在写的过程中增加这个偏移。

以下是一个案例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int main() {
    char buf[512];
    int n;
    for (;;) {
        n = read(0, buf, sizeof buf);
        if (n == 0) {
            break;
        } else if (n < 0) {
            fprintf(2, "read error\n");
            exit(1);
        } else if (write(1, buf, n) != n) {
            fprintf(2, "write error\n");
            exit(1);
        }
    }
}
```
运行结果：

> ****** chap0 % ./main
xsaxas
xsaxas
dvf
dvf
dewdwed
dewdwed
...

系统调用 close()  会释放一个文件描述符，它未来可以被 open, pipe, dup 等调用重用。一个新分配的文件描述符**永远都是当前进程的最小的未被使用的文件描述符**。

fork() 会复制父进程的文件描述符（复制了文件描述符表）和内存内容，所以子进程和父进程的文件描述符一模一样。换句话说，它们在一定意义上共享了**文件和内存代码**。

exec() 会替换调用它的进程的内存内容，但是**依然会保留它的文件描述符表**。这种行为使得 shell 可以这样实现重定向：fork 一个进程，重新打开指定文件的文件描述符，然后执行新的程序。以下是一个简单示例：

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    char *argv[3];
    argv[0] = "/bin/cat";
    argv[1] = NULL;
    int pid = fork();

    if (pid == 0) {
        // 子进程
        close(0); // 关闭标准输入
        int fd = open("input.txt",
                      O_RDONLY); // 文件 0 此时标准输入指向了 "input.txt"
        if (fd < 0) {
            perror("open");
            exit(1);
        }
        execv(argv[0], argv); // 只会修改内存内容,但不会修改文件描述符表,cat
                              // 将会从文件 0 获取数据流
        perror("exec");
        exit(1);
    } else if (pid > 0) {
        // 父进程
        execv(argv[0], argv); // 子进程对 0 进程重定向不会影响父进程的文件描述符表,因此不会影响父进程从终端标准输入获取输入流
        perror("exec");
        exit(1);
    }

    return 0;
}

```
运行结果：

> ****** chap0 % ./main            
hello world1                                                                                                                        
hello world2
hello world3
this is fproc
this is fproc

丛运行结果可以看出，子进程对 0 进程重定向不会影响父进程的文件描述符表,因此不会影响父进程从终端标准输入获取输入流。

子进程关闭文件描述符0后，我们可以保证open() 会使用0作为新打开的文件 input.txt的文件描述符（因为0是 open() 执行时的最小可用文件描述符）。之后 cat 就会在标准输入指向 input.txt 的情况下运行。对于 cat 而言，它根本无法分辨输入流来源于终端还是某个文件。

**xv6 的 shell 正是这样实现 I/O 重定向的**。虽然 fork 复制了文件描述符，**但每一个文件当前的偏移仍然是在父子进程之间共享的**。这一点很重要，考虑下面这个例子：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>
int main() {
    int pid = fork();
    if (pid == 0) {
        write(1, "hello ", 6);
        exit(0);
    } else {
        int status;
        wait(&status);
        write(1, "world\n", 6);
    }
    return 0;
}
```
运行结果：

> hello world

在这段代码的结尾，绑定在文件描述符1上的文件有数据"hello world"，父进程的 write 会从子进程 write 结束的地方继续写 (因为 wait ,父进程只在子进程结束之后才运行 write)。这种行为有利于顺序执行的 shell 命令的顺序输出。再考虑这个例子：

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    char *argv[3];
    argv[0] = "/bin/cat";
    argv[1] = NULL;
    close(0);
    int fd = open("input.txt", O_RDONLY);
    if (fd < 0) {
        perror("open");
        exit(1);
    }
    int pid = fork();

    if (pid == 0) {
        printf("子进程运行!\n");
        execv(argv[0], argv);
        perror("exec");
        exit(1);
    } else if (pid > 0) {
        printf("父进程运行!\n");
        execv(argv[0], argv);
        perror("exec");
        exit(1);
    } else {
        perror("fork()");
        return 1;
    }
}

```

运行结果：

> ****** chap0 % ./main            
父进程运行!
子进程运行!
hello world1
hello world2

父子进程通过共享文件描述符偏移共同完成了对文件的读写，并发执行。

dup 复制一个已有的文件描述符，返回一个指向同一个输入/输出对象的新描述符。**这两个描述符共享一个文件偏移**，**正如被 fork 复制的文件描述符一样**。这里有另一种打印 “hello world” 的办法：

```c
#include <unistd.h>
int main() {
    int fd = dup(1);
    write(1, "hello ", 6);
    write(fd, "world\n", 6);
    return 0;
}
```
从同一个原初文件描述符通过一系列 **fork 和 dup 调用产生的文件描述符都共享同一个文件偏移，而其他情况下产生的文件描述符就不是这样**，即使它们打开的都是同一份文件。

文件描述符是一个**强大的抽象**，因为它们将它们所连接的细节隐藏起来了：一个进程向描述符1写出，它有可能是写到一份文件，一个设备（如控制台），或一个管道。**文件描述符连接的细节对读取文件描述符的进程是透明的**。

## 三、管道

管道是一个小的**内核缓冲区**，它以**文件描述符对**的形式提供给进程，一个用于**写操作**，一个用于**读操作**。从管道的一端写的数据可以从管道的另一端读取。**管道提供了一种进程间交互的方式**。
下面的示例代码运行了程序 wc，它的标准输出绑定到了一个管道的读端口：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int p[2];
    char *argv[2];
    argv[0] = "wc";
    argv[1] = NULL; // 注意修改为 NULL
    pipe(p);
    if (fork() == 0) {
        close(0);
        dup(p[0]); // 重定向到 0
        close(p[0]);
        // 如果没有关闭,程序会一直等,不会出现 eof
        // 对管道执行的read会一直等待，直到有数据了或者其他绑定在这个管道写端口的描述符都已经关闭了
        close(p[1]); // 不需要写，因此关闭
        execl("/usr/bin/wc", "wc", (char *)0); // 使用完整路径和命令名称
    } else {
        write(p[1], "hello world\n", 12);
        close(p[0]); // 不需要读
        close(p[1]); // 不需要继续写
    }
    return 0;
}

```
这段程序调用 pipe，创建一个管道并且将读写描述符记录在数组 p 中。在 fork 之后，父进程和子进程**都有了指向管道的文件描述符**。子进程将管道的读端口拷贝在描述符0上，关闭 p 中的描述符，然后执行 wc。当 wc 从标准输入读取时，它实际上是从管道读取的。父进程向管道的写端口写入然后关闭它的两个文件描述符。

如果数据没有准备好，那么对管道执行的read会一直等待，直到有数据了或者其他绑定在这个管道写端口的描述符都已经关闭了。读操作会一直阻塞直到不可能再有新数据到来了，这就是为什么我们在执行 wc 之前要关闭子进程管道的写端口。如果 wc 指向了一个管道的写端口，那么 wc 就永远看不到 eof 了。**这意味我们必须把读的标准输入一定要重定向到管道上，而不是直接操作管道符**。以下是一个案例：

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int p[2];
    char *argv[2];
    argv[0] = "/usr/bin/wc";
    argv[1] = NULL;
    pipe(p);
    int pid = fork();
    if (pid == 0) {
        close(0);
        dup(p[0]); // 重定向到标准输入
        close(p[1]); // 关闭这个输入
        execv(argv[0], argv);
    } else if (pid > 0) {
        write(p[1], "hello world\n", 12);
    } else {
        perror("fork()");
        return 1;
    }
    return 0;
}

```
如果不执行`close(p[1]);`子进程将会一直会等待管道的输入或者接收到了空字符串。

xv6 shell 对管道的实现（比如 fork sh.c | wc -l）和上面的描述是类似的。子进程创建一个管道连接管道的左右两端。然后它为管道左右两端都调用 runcmd，然后通过两次 wait 等待左右两端结束。因此，shell 可能创建出一颗进程树。树的叶子节点是命令，中间节点是进程，它们会等待左子和右子执行结束。

pipe 可能看上去和临时文件没有什么两样：命令

```bash
echo hello world | wc
```

可以用无管道的方式实现：

```bash
echo hello world > /tmp/xyz; wc < /tmp/xyz
```

但管道和临时文件起码有三个关键的不同点。首先，**管道会进行自我清扫**，如果是 shell 重定向的话，我们必须要在任务完成后删除 /tmp/xyz。第二，管道可以传输任意长度的数据。第三，**管道允许同步：两个进程可以使用一对管道来进行二者之间的信息传递**，**每一个读操作都阻塞调用进程**，直到另一个进程用 write 完成数据的发送。

## 四、文件系统
xv6 文件系统提供文件和目录，文件就是一个简单的字节数组，而目录包含指向文件和其他目录的引用。xv6 把目录实现为一种特殊的文件。目录是一棵树，它的根节点是一个特殊的目录 root。不从 / 开始的目录表示的是相对调用进程当前目录的目录，调用进程的当前目录可以通过 chdir 这个系统调用进行改变。下面的这些代码都打开同一个文件（假设所有涉及到的目录都是存在的）：

```c
chdir("/a");
chdir("b");
open("c", O_RDONLY);

open("/a/b/c", O_RDONLY);
```
fstat 可以获取一个文件描述符指向的文件的信息。它填充一个名为 stat 的结构体，它在 stat.h 中定义为：

```c
#define T_DIR  1
#define T_FILE 2
#define T_DEV  3
// Directory
// File
// Device
     struct stat {
       short type;  // Type of file
       int dev;     // File system’s disk device
       uint ino;    // Inode number
       short nlink; // Number of links to file
       uint size;   // Size of file in bytes
};
```

文件名和这个文件本身是有很大的区别。同一个文件（称为 inode）可能有多个名字，称为**连接 (links)**。系统调用 link 创建另一个文件系统的名称，它指向同一个 inode。下面的代码创建了一个既叫做 a 又叫做 b 的新文件：

```c
open("a", O_CREATE|O_WRONGLY);
link("a", "b");
```
读写 a 就相当于读写 b。每一个 inode 都由一个唯一的 inode 号 直接确定。在上面这段代码中，**我们可以通过 fstat 知道 a 和 b 都指向同样的内容**：a 和 b 都会返回同样的 inode 号（ino），并且 nlink 数会设置为2。

系统调用 unlink 从文件系统移除一个文件名。**一个文件的 inode 和磁盘空间只有当它的链接数变为 0 的时候才会被清空**，也就是没有一个文件再指向它。

我们同样可以通过 b 访问到它。另外，

```c
fd = open("/tmp/xyz", O_CREATE|O_RDWR);
unlink("/tmp/xyz");
```

以上是创建一个临时 inode 的最佳方式，这个 inode 会在进程关闭 fd 或者退出的时候被清空。

xv6 关于文件系统的操作都被实现为**用户程序**，诸如 mkdir，ln，rm 等等。这种设计允许任何人都可以通过用户命令拓展 shell 。

有一个例外，那就是 cd，它是在 shell 中实现的。**cd 必须改变 shell 自身的当前工作目录。**如果 cd 作为一个普通命令执行，那么 shell 就会 fork 一个子进程，而子进程会运行 cd，cd 只会改变子进程的当前工作目录。**父进程的工作目录保持原样，这意味着这个命令没有起到预期的作用**。以下是 sv6 中 cd 处理的源代码：

```c
// Read and run input commands.
while (getcmd(buf, sizeof(buf)) >= 0) {
    if (buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' ') {
        // Chdir must be called by the parent, not the child.
        buf[strlen(buf) - 1] = 0; // chop \n
        if (chdir(buf + 3) < 0)
            fprintf(2, "cannot cd %s\n", buf + 3);
        continue;
    }
    if (fork1() == 0)
        runcmd(parsecmd(buf));
    wait(0);
}
```
可以看到，**它把 cd 命令作为 shell 的一个特例对待**，它并不会为 cd fork()一个新的进程，它会只会改变shell 的工作目录。

*全文完，感谢阅读。*







