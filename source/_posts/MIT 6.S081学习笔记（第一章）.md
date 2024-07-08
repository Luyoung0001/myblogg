---
title: MIT 6.S081学习笔记（第一章）
date: 2023-09-10 19:30:08
tags:
    - 学习
    - 笔记
    - 操作系统
    - MIT 6.S081
categories:
    - OS
    - 系统编程
    - Unix/Linux
---

<!--more-->

## 〇、前言
本章主要是关于实验环境的搭建和完成 [LAB UTIL](https://pdos.csail.mit.edu/6.828/2023/labs/util.html)。
平台：阿里云 Ubuntu20.04+VScode on macOS（M1 Apple Silicon）。

## 一、环境搭建

### 1、QEMU

> QEMU（quick emulator）是一款由法布里斯·贝拉（Fabrice
> Bellard）等人编写的通用且免费的可执行**硬件虚拟化的（hardware
> virtualization）开源仿真器（Emulator）**。
>
> 其与Bochs，PearPC类似，但拥有高速（配合KVM），跨平台的特性。
>
> QEMU是一个托管的虚拟机，它通过动态的二进制转换，**模拟CPU，并且提供一组设备模型**，使它能够运行多种未修改的客户机OS，可以通过与KVM一起使用进而接近本地速度运行虚拟机（接近真实电脑的速度）。
>
> QEMU还可以为user-level的进程执行CPU仿真，进而允许了为一种架构编译的程序在另外一种架构上面运行（借由VMM的形式）。——wikipedia

**实验中QEMU用来模拟 RISC-V 处理器以及一组必要硬件（内存、硬盘等）。**
### 2、RISC-V

> RISC-V（发音为“risk-five”）是一个基于精简指令集（RISC）原则的开源指令集架构（ISA），简易解释为与开源软体运动相对应的一种“开源硬体”。
>
> 与大多数指令集相比，RISC-V指令集可以自由地用于任何目的，允许任何人设计、制造和销售RISC-V芯片和软件而不必支付给任何公司专利费。虽然这不是第一个开源指令集，但它具有重要意义，因为其设计使其适用于现代计算设备（如仓库规模云计算机、高端移动电话和微小嵌入式系统）。设计者考虑到了这些用途中的性能与功率效率。该指令集还具有众多支持的软件，这解决了新指令集通常的弱点。
>
> RISC-V指令集的设计考虑了小型、快速、低功耗的现实情况来实做，但并没有对特定的微架构做过度的设计。——wikipedia

### 3、xv6

> xv6是以ANSI C重新编写的Unix第六版现代实现版本，适用于多处理器x86或RISC-V系统。xv6于2006年问世，作为麻省理工学院的操作系统工程（6.828）课程的教学使用。
> 与Linux或BSD不同，xv6非常简单，足以在一个学期内讲完，但仍然包含Unix的重要概念和组织。由于PDP-11机器没有被广泛使用，而且最初的操作系统是用过时的pre-ANSI
> C编写的，所以该课程没有学习原始的V6代码，而是使用xv6。——wikipedia

由于 xv6 足够经典也足够简单，因此学习它就可以了解足够的关于 OS 的知识。

### 4、搭建环境
一行命令搞定：

```bash
$ sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```
直接拉取源码：

```bash
$ git clone git://g.csail.mit.edu/xv6-labs-2023
$ cd xv6-labs-2023
```
编译，直接进入 xv6 终端：

```bash
******:~/xv6-labs-2023# make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$
```
玩玩命令啥的：

```bash
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2059
xargstest.sh   2 3 93
cat            2 4 23888
echo           2 5 22720
forktest       2 6 13080
grep           2 7 27248
init           2 8 23824
kill           2 9 22696
ln             2 10 22648
ls             2 11 26120
mkdir          2 12 22792
rm             2 13 22784
sh             2 14 41656
wc             2 18 25032
zombie         2 19 22184
console        3 21 0
$ echo README
README
$ echo hello
hello
$ cat README
xv6 is a re-implementation of Dennis Ritchie's and Ken Thompson's Unix
Version 6 (v6).  xv6 loosely follows the structure and style of v6,
...
$
```

退出：

```bash
ctrl+a // 松开
x
```
## 二、完成第一个任务

### 1、查看所有分支

```bash
******:~/xv6-labs-2023# git branch --all
  master
* util
  remotes/origin/HEAD -> origin/master
  remotes/origin/cow
  remotes/origin/fs
  remotes/origin/lazy
  remotes/origin/lock
  remotes/origin/master
  remotes/origin/mmap
  remotes/origin/net
  remotes/origin/pgtbl
  remotes/origin/riscv
  remotes/origin/syscall
  remotes/origin/thread
  remotes/origin/traps
  remotes/origin/util
```
### 2、切换第一个实验 util 分支

```bash
git checkout util
```
### 3、完成 sleep(easy)
在 `/user`中创建 `sleep.c`，编写程序：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
int main(int argc,char *argv[]){
  if(argc <= 1){
    exit(1);
  }
  int time = atoi(argv[1]);
  sleep(time);
  exit(0);
}
```
这个实验相当简单，就是让你使用一下系统调用 sleep()。
### 4、测试

```c
******:~/xv6-labs-2023# ./grade-lab-util sleep
make: 'kernel/kernel' is up to date.
== Test sleep, no arguments == sleep, no arguments: OK (1.1s)
== Test sleep, returns == sleep, returns: OK (0.8s)
== Test sleep, makes syscall == sleep, makes syscall: OK (1.0s)
```

测试都通过了。

现在就可以顺利地做试验了，把剩下的任务都完成一下。
## 三、完成所有实验

### 1、pingpong(easy)

#### Question requirements

> Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "<pid>: received ping", where <pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "<pid>: received pong", and exit. Your solution should be in the file user/pingpong.c.

#### Some hints

> - Use pipe to create a pipe.
> - Use fork to create a child.
> - Use read to read from the pipe, and write to write to the pipe.
> - Use getpid to find the process ID of the calling process.
> - Add the program to UPROGS in Makefile.
> - User programs on xv6 have a limited set of library functions available to them. You can see - the list in user/user.h;
> - the source  (other than for system calls) is in user/ulib.c, user/printf.c, and
> - user/umalloc.c.

#### Run the program from the xv6 shell and it should produce the following output

```bash
 $ make qemu
 ...
 init: starting sh
 $ pingpong
 4: received ping
 3: received pong
 $
```
#### Solution

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main() {
    char buf[10] = {0};
    // 将用不到的文件描述符尽可能全部关闭
    int p1[2], p2[2];
    pipe(p1); // 父进程发
    pipe(p2); // 子进程发

    int pid = fork();
    if (pid == 0) {
        const char *message = "S";
        // 子进程:在 p2 中发送,p1 中接收
        close(0);
        close(2);

        close(p1[1]);
        close(p2[0]);

        // 接收
        read(p1[0], buf, 10);
        fprintf(1,"%d: received ping\n", getpid());
        close(p1[0]);
        // 发送

        write(p2[1], message, 2);
        // 发送完就关闭
        close(p2[1]);
        exit(0);
    } else if (pid > 0) {
        const char *message = "F";
        // 父进程:在 p1 中 发送,p2 中 接收
        close(0);
        close(2);

        close(p1[0]);
        close(p2[1]);

        // 发送
        write(p1[1], message, 2);
        // 发送完就关闭
        close(p1[1]);
        // 接收
        read(p2[0], buf, 10);
        close(p2[0]);

        int status;
        wait(&status);// 防止抢占1
        fprintf(1,"%d: received pong\n", getpid());

    } else {
        fprintf(1,"err for fork()\n");
        exit(1);
    }
    exit(0);
}
```

#### Output

```bash
******:~/xv6-labs-2023# ./grade-lab-util pingpong
make: 'kernel/kernel' is up to date.
== Test pingpong == pingpong: OK (1.6s)
```

#### Notes
有以下需要注意的地方：
- 如果设置 `CPUS=1`，（建议不要设置，因为后期实验可能需要多线程来处理）那就不用通过 `wait()` 来解决抢占问题。但是那也是多线程编程的好习惯，避免僵尸进程的出现；
- 如果操作系统不同，代码可能略有差异。比如macOS的 stdin、stdout、定义的并不是 0、1。
```c
// macOS
#define	stdin	__stdinp
#define	stdout	__stdoutp
#define	stderr	__stderrp
...
extern FILE *__stdinp;
extern FILE *__stdoutp;
extern FILE *__stderrp;
...
```

### 2、primes(moderate)

#### Question requirements

> Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down this page and the surrounding text explain how to do it. Your solution should be in the file user/primes.c.

#### Some hints

> - Be careful to close file descriptors that a process doesn't need, because otherwise your program will run xv6 out of resources before the first process reaches 35.
> - Once the first process reaches 35, it should wait until the entire pipeline terminates, including all children, grandchildren, &c. Thus the main primes process should only exit after all the output has been printed, and after all the other primes processes have exited.
> - Hint: read returns zero when the write-side of a pipe is closed.
It's simplest to directly write 32-bit (4-byte) ints to the pipes, rather than using formatted ASCII I/O.
> - You should create the processes in the pipeline only as they are needed.
> - Add the program to UPROGS in Makefile.

#### Your solution is correct if it implements a pipe-based sieve and produces the following output:

```bash
  	$ make qemu
    ...
    init: starting sh
    $ primes
    prime 2
    prime 3
    prime 5
    prime 7
    prime 11
    prime 13
    prime 17
    prime 19
    prime 23
    prime 29
    prime 31
    $
```
#### Solution
这就是在父进程中给 pipe 中 写入 2~35，如果不能被 2 整除，就把所有的数移到另一个子进程中的管道；然后循环操作。

```bash
p = get a number from left neighbor print p loop:
    n = get a number from left neighbor
    if (p does not divide n)
        send n to right neighbor
```

图示如下：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/68737d99ab96432998a15f0cce8936e3_1720253724529.png?token=ANB4BCMS2ZGBQQUYQUSBSBTGRD6VY)

直接上代码：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
void prime(int p[]){
  int x,y;
  int child_p[2];
  // 子进程
  // 从 p[0] 接收
  close(p[1]);

  if(read(p[0],&x,sizeof(int))){
    // 打印素数
    fprintf(1,"prime %d\n",x);
    // 处理其它数字
    pipe(child_p);
    int pid = fork();
    if(pid > 0){
      // 父进程
      close(child_p[0]);
      while(read(p[0], &y,sizeof(int))){
        if(y%x != 0){
          // 其它不能被 x 整除的数,全部发往下一个子进程
          write(child_p[1], &y, sizeof(int));
        }
      }
      close(p[0]);
      close(child_p[1]);
      // 返回
      int status;
      wait(&status);
    }else if(pid == 0){
      prime(child_p);
    }
  }
  exit(0);

}

int main(){

  int p[2];
  pipe(p);
  int pid = fork();

  if(pid == 0){
    // 递归实现
    prime(p);

  }else if(pid > 0){
    close(p[0]);
    // 父进程将数据传给子进程
    for(int i = 2; i <= 35; i++){
      write(p[1], &i,sizeof(int));
    }
    close(p[1]);
    int status;
    wait(&status);
  }else{
    fprintf(1, "err for fork()\n");
    exit(1);
  }

  exit(0);
}

```
#### Output

```bash
$ primes
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
$
```

#### Notes
- 这题目还是用递归比较好，递归的出口就是 `read()` 返回为 0；
- 将管道作为参数传递给另一个函数时，实际上传递的是指向管道的文件描述符数组的指针。


### 3、find (moderate)
#### Question requirements

> Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file user/find.c.

#### Some hints

> - Look at user/ls.c to see how to read directories.
> - Use recursion to allow find to descend into sub-directories.
> - Don't recurse into "." and "..".
> - Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.
> - You'll need to use C strings. Have a look at K&R (the C book), for example Section 5.5.
> - Note that == does not compare strings like in Python. Use strcmp() instead.
> - Add the program to UPROGS in Makefile.


#### Your solution is correct if produces the following output (when the file system contains the files b and a/b):

```bash
    $ make qemu
    ...
    init: starting sh
    $ echo > b
    $ mkdir a
    $ echo > a/b
    $ find . b
    ./b
    ./a/b
    $
```

#### Solution
这个题目比较简单，主要就是想让我了解一下 xv6 的文件系统。系统调用很少，所有的系统调用就这一些：

```c
// system calls
int fork(void);
int exit(int) __attribute__((noreturn));
int wait(int*);
int pipe(int*);
int write(int, const void*, int);
int read(int, void*, int);
int close(int);
int kill(int);
int exec(const char*, char**);
int open(const char*, int);
int mknod(const char*, short, short);
int unlink(const char*);
int fstat(int fd, struct stat*);
int link(const char*, const char*);
int mkdir(const char*);
int chdir(const char*);
int dup(int);
int getpid(void);
char* sbrk(int);
int sleep(int);
int uptime(void);
```
和这个任务有关的，就这两个：

```c
int open(const char*, int);
int fstat(int fd, struct stat*);
```

> *int open(const char *path, int flags);**
> - 作用：open 函数用于打开文件，并返回一个文件描述符（file descriptor），该描述符用于后续对文件的读取和写入操作。 参数：
> - path：一个字符串，表示要打开的文件的路径和名称。
> - flags：一个整数，用于指定打开文件的方式和权限等。这个参数通常包括了标志位，例如 O_RDONLY（只读）、O_WRONLY（只写）、O_RDWR（读写）、O_CREAT（如果文件不存在则创建）等等。
> - 返回值：如果成功打开文件，open 函数返回一个非负整数，表示文件- 描述符。如果失败，它返回-1，并可以通过检查 errno 变量来获取失败的具体原因。
>
> *int fstat(int fd, struct stat *buf);**
> - 作用：fstat 函数用于获取已打开文件的元数据信息，并将这些信息填充到提供的 struct stat 结构体中。 参数：
> - fd：一个整数，表示已打开文件的文件描述符。
> - buf：一个指向 struct stat 结构体的指针，用于存储文件的元数据信息。
> - 返回值：如果成功，fstat 函数返回0，如果失败，返回-1，并可以通过检查 errno 变量来获取失败的具体原因。
> - 填充的 struct stat 结构体包含有关文件的各种信息，如文件类型、大小、权限、所有者、修改时间等等，可以用于进一步的文件操作和分析。

但是，本实验可以不需要用到这两个系统调用，xv6 帮我们实现了几个更好用的用户调用：

```c
// ulib.c
int stat(const char*, struct stat*);
char* strcpy(char*, const char*);
void *memmove(void*, const void*, int);
char* strchr(const char*, char c);
int strcmp(const char*, const char*);
void fprintf(int, const char*, ...);
void printf(const char*, ...);
char* gets(char*, int max);
uint strlen(const char*);
void* memset(void*, int, uint);
void* malloc(uint);
void free(void*);
int atoi(const char*);
int memcmp(const void *, const void *, uint);
void *memcpy(void *, const void *, uint);
```

可能会用到的就几个：

```c
int stat(const char*, struct stat*);
char* strcpy(char*, const char*);
void *memmove(void*, const void*, int);
int strcmp(const char*, const char*);
```

它们被实现在`/user/ulib.c`中：

```c
int strcmp(const char *p, const char *q) {
    while (*p && *p == *q)
        p++, q++;
    return (uchar)*p - (uchar)*q;
}
uint strlen(const char *s) {
    int n;

    for (n = 0; s[n]; n++)
        ;
    return n;
}
int stat(const char *n, struct stat *st) {
    int fd;
    int r;

    fd = open(n, O_RDONLY);
    if (fd < 0)
        return -1;
    r = fstat(fd, st);
    close(fd);
    return r;
}
void *memmove(void *vdst, const void *vsrc, int n) {
    char *dst;
    const char *src;

    dst = vdst;
    src = vsrc;
    if (src > dst) {
        while (n-- > 0)
            *dst++ = *src++;
    } else {
        dst += n;
        src += n;
        while (n-- > 0)
            *--dst = *--src;
    }
    return vdst;
}
```
还有几个结构体需要了解一下：
```c
// Directory is a file containing a sequence of dirent structures.
#define DIRSIZ 14

struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```
目录就是一个文件，包含一系列的 `dirent` 结构，文件长度不超过 13 个字符（最后一个为'\0'）。inum 是这个目录项的所指的文件占用的 inode 个数。

```c
struct stat {
  int dev;     // File system's disk device
  uint ino;    // Inode number
  short type;  // Type of file
  short nlink; // Number of links to file
  uint64 size; // Size of file in bytes
};
```
这个结构体包含了文件的元数据信息，包括文件类型，文件大小等，和 Linux 差别比较大。

代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
char* getfilename(char *path){
  char *p;
  // 从后面向前找
  for(p=path+strlen(path); p >= path && *p != '/'; p--){}
  return ++p;
}
int find(char *path,char *fileName){
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;
  if((fd = open(path, 0)) < 0){
    fprintf(2, "find: cannot open %s\n", path);
    return 1;
  }
  if(fstat(fd, &st) < 0){
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return 1;
  }
  switch(st.type){
    case T_FILE:
      if (strcmp(getfilename(path), fileName) == 0) {
        printf("%s\n", path);
    }
    break;
    case T_DIR:
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("find: path too long\n");
      break;
    }
    strcpy(buf, path);
    // p指向 buf 最后一个字符'\0'
    p = buf+strlen(buf);
    *p++ = '/';
    // 循环读取目录,目录就是一个文件，包含一系列的 `dirent` 结构
    while(read(fd, &de, sizeof(de)) == sizeof(de)){
      // 空的文件
      if(de.inum == 0) {
        continue;
      }
      if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0) {
        continue;
      }
      // 把文件名复制到 buf
      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;
      // 打不开就下一个
      if(stat(buf, &st) < 0){
        printf("find: cannot stat %s\n", buf);
        continue;
      }
      find(buf, fileName);
    }
    break;
  }
  close(fd);
  return 0;
}
int main(int argc,char *argv[]){
  if(argc != 3){
    exit(1);
  }
  // 这个思路就是找出argv[1]的所有文件,然后找到文件名和 argv[2]一样的文件,并打印出目录
  find(argv[1],argv[2]);
  exit(0);
}
```
#### Output

```bash
******:~/xv6-labs-2023# ./grade-lab-util find
make: 'kernel/kernel' is up to date.
== Test find, in current directory == find, in current directory: OK (1.8s)
== Test find, recursive == find, recursive: OK (1.2s)
```
#### Notes
- 能调用 OS 提供的用户调用，就尽量避免直接调用系统调用。


### 4、xargs (moderate)
#### Question requirements

> Write a simple version of the UNIX xargs program for xv6: its arguments describe a command to run, it reads lines from the standard input, and it runs the command for each line, appending the line to the command's arguments. Your solution should be in the file user/xargs.c.

#### Some hints

> - Use fork and exec to invoke the command on each line of input. Use wait in the parent to wait for the child to complete the command.
> - To read individual lines of input, read a character at a time until a newline ('\n') appears.
> - kernel/param.h declares MAXARG, which may be useful if you need to declare an argv array.
> - Add the program to UPROGS in Makefile.
> - Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.

意思就是说，使用 fork、exec 来执行命令；每次读取一个字符，直到读完一行。

#### To test your solution for xargs, run the shell script xargstest.sh. Your solution is correct if it produces the following output

```bash
  $ make qemu
  ...
  init: starting sh
  $ sh < xargstest.sh
  $ $ $ $ $ $ hello
  hello
  hello
  $ $

```

#### Solution

这个也不难，关键步骤就是将标准输入**0** 中的字符一个一个读取，直到遇见换行符或者读取结束。然后将这个读取的结果稍作处理，加到 argv 字符串数组里面，然后fork() 一个子线程，用 exec() 去执行它。

代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"

int main(int argc, char *argv[]) {
    int bufP , i, readLen;
    char buf[512];
    char* new_argv[MAXARG];
    // 提取命令以及参数
    for (i = 1; i < argc; i++) {
        new_argv[i - 1] = argv[i];
    }
    while (1) {
      bufP = -1;
        do {
            bufP++;
            readLen = read(0, &buf[bufP], sizeof(char));
            // 一次从标准输入里面读一个字符,合成参数
        } while (readLen > 0 && buf[bufP] != '\n');

        if (readLen == 0 && bufP == 0) { // 如果没有读到任何标准输入,就退出
            break;
        }

        buf[bufP] = '\0';
        // 把参数放到 exe_argv 数组
        new_argv[argc - 1] = buf;
        // 0标志命令行参数列表的结束
        new_argv[argc] = 0;

        if (fork() == 0) {
            exec(new_argv[0], new_argv);
            exit(0);
        } else {
            wait(0);
        }
    }
    exit(0);
}
```

#### Output

```bash
******:~/xv6-labs-2023# ./grade-lab-util xargs
make: 'kernel/kernel' is up to date.
== Test xargs == xargs: OK (2.1s)
```

#### Notes
- exec()和 Linux 中的系统调用还是有一些差别，书中说的是：

> This fragment replaces the calling program with an instance of the program /bin/echo running
with the argument list echo hello. Most programs ignore the first element of the argument array,
which is conventionally the name of the program.

- 因此会忽略第一个参数，因为它的第一个参数已经传进去了；
- 不同的操作系统是对于 NULL 空指针的定义不同:

```c
#ifndef NULL
	#include <sys/_types.h> /* __DARWIN_NULL */
	#define NULL  __DARWIN_NULL
#endif  /* NULL */

#ifdef __cplusplus
	#ifdef __GNUG__
		#define __DARWIN_NULL __null
	#else /* ! __GNUG__ */
		#ifdef __LP64__
			#define __DARWIN_NULL (0L)
		#else /* !__LP64__ */
			#define __DARWIN_NULL 0
		#endif /* __LP64__ */
	#endif /* __GNUG__ */
#else /* ! __cplusplus */
	#define __DARWIN_NULL ((void *)0)
#endif /* __cplusplus */
```
- 也就是说，如果在 C++ 环境下，就是用`0`、`0L`、`__null` 来表示`NULL`；C 环境下，就用`(void*)0`来表示空指针。因此为了统一，但是不要用`0`、`1` 等字符来表示空指针，以防止跨平台什么的出错。

## 四、总结
键入 `make grade` 就会打分：
```bash
== Test sleep, no arguments ==
$ make qemu-gdb
sleep, no arguments: OK (3.3s)
== Test sleep, returns ==
$ make qemu-gdb
sleep, returns: OK (1.1s)
== Test sleep, makes syscall ==
$ make qemu-gdb
sleep, makes syscall: OK (1.0s)
== Test pingpong ==
$ make qemu-gdb
pingpong: OK (1.1s)
== Test primes ==
$ make qemu-gdb
primes: OK (1.0s)
== Test find, in current directory ==
$ make qemu-gdb
find, in current directory: OK (1.1s)
== Test find, recursive ==
$ make qemu-gdb
find, recursive: OK (1.4s)
== Test xargs ==
$ make qemu-gdb
xargs: OK (1.9s)
== Test time ==
time: OK
Score: 100/100
```

大概肝了周末两天，总算完成了实验，总算是对 xv6 有了大致的了解。

*全文完，感谢阅读。*


















