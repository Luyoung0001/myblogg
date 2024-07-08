---
title: mmap() 的几种用法
date: 2024-06-12 17:50:07
tags:
    - 系统调用
    - mmap
    - 文件
categories:
    - OS
    - 系统编程
    - Unix/Linux
---

<!--more-->

## 〇、前言
`mmap` 是一个非常强大的系统调用，常用于映射文件到内存中，以实现快速和方便的文件访问，也用于进程间通信等多种场景。下面是一些 `mmap` 的常见用法：

### 1. **文件映射（Memory-mapped files）**
文件映射是 `mmap` 最直接的用途之一，可以将整个文件或文件的一部分映射到进程的地址空间。这允许程序像访问普通内存那样访问文件数据，可以提高文件操作的性能，特别是对于大文件的随机访问。

**示例用途**：
- 数据库系统中的数据文件管理。
- 大型文件的快速读写，如媒体文件处理或科学数据处理。

### 2. **进程间通信（Interprocess Communication, IPC）**
通过映射匿名内存（不与任何文件关联的内存区域），`mmap` 可以用来在父子进程或者任何共享同一个内存映射的不同进程之间进行数据共享。

**示例用途**：
- 多进程应用程序中共享状态或数据。
- 实现复杂的同步机制，如共享内存中的锁或信号量。

### 3. **创建高效的缓冲区（Efficient Buffering）**
使用 `mmap` 创建的内存区域可以用作自定义的缓冲区，例如，为网络数据传输创建大型缓冲区，或者用于音视频数据的快速缓存。

**示例用途**：
- 网络服务器中的数据缓冲，以减少读写文件的系统调用。
- 音视频播放软件中的数据缓冲。

### 4. **动态内存管理**
除了映射文件，`mmap` 还可以请求操作系统分配一块新的内存区域（匿名映射），用于自定义的内存管理策略。

**示例用途**：
- 实现内存池。
- 替代 `malloc` 或 `free` 用于特定场景下的内存分配，以控制内存使用更细致。

### 5. **反射和修改程序的行为（Self-modifying code）**
在一些高级用途中，`mmap` 可以用于实现程序的代码部分的动态修改，比如即时编译（JIT）技术。

**示例用途**：
- 动态编译器和运行时代码生成。
- 修改运行中的程序，以调整其功能或优化性能。

### 6. **大内存对象的管理**
利用 `mmap` 映射的内存不受传统堆大小限制，可以用来管理非常大的内存对象。

**示例用途**：
- 大规模数值计算，需要分配大量内存的应用。
- 处理大型图像或视频帧序列。

通过这些用法，`mmap` 提供了一种灵活而强大的机制，用于处理大量数据、优化性能、实现进程间通信等多种需求。

## 一、文件映射

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(int argc, char* argv[]) {
    int fd, nread;
    struct stat sb;
    char* mapped;

    // 打开文件
    if ((fd = open(argv[1], O_RDWR)) < 0) {
        perror("open");
    }

    // 读取文件的属性
    if ((fstat(fd, &sb)) == -1) {
        perror("fstat");
    }

    // 将文件映射到进程空间
    if ((mapped = (char*)mmap(NULL, sb.st_size, PROT_READ | PROT_WRITE,
                              MAP_SHARED, fd, 0)) == (void*)-1) {
        perror("mmap");
    }

    // 修改一个字，同步到硬盘文件
    mapped[20] = '9';
    if ((msync((void*)mapped, sb.st_size, MS_SYNC)) == -1) {
        perror("msync");
    }
    // 释放存储映射区
    if ((munmap((void*)mapped, sb.st_size)) == -1) {
        perror("munmap");
    }
    return 0;
}

```

这是测试文件，data.txt：
```txt
aaaaaaaaa
bbbbbbbbb
cccccccccccc
dddddd
```

编译运行：

```bash
❯ gcc mmap1.c -o main
❯ ./main data.txt

~/tests/test2                                                                       16:58:25
❯
```
data.txt直接被修改为了：

```txt
aaaaaaaaa
bbbbbbbbb
9ccccccccccc
dddddd
```

通过文件映射，可以先将文件映射到内存中，可以随机访问、修改，然后同步到磁盘文件中，极大的提升了 IO 效率。

## 二、进程间共享文件

A 程序：

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(int argc, char** argv) {
    int fd;
    struct stat sb;
    char* mapped;

    /* 打开文件 */
    if ((fd = open(argv[1], O_RDWR)) < 0) {
        perror("open");
    }

    /* 获取文件的属性 */
    if ((fstat(fd, &sb)) == -1) {
        perror("fstat");
    }

    /* 将文件映射至进程的地址空间 */
    if ((mapped = (char*)mmap(NULL, sb.st_size, PROT_READ | PROT_WRITE,
                              MAP_SHARED, fd, 0)) == (void*)-1) {
        perror("mmap");
    }

    /* 文件已在内存, 关闭文件也可以操纵内存 */
    close(fd);

    /* 每隔两秒查看存储映射区是否被修改 */
    while (1) {
        printf("%s\n", mapped);
        sleep(2);
    }

    return 0;
}

```

这个程序将 data.txt 映射到内存中之后将文件关闭，接着每隔 2 s 打印一次映射到内存中的文件（此时就算 data.txt被删除，都不受影响）。

B 程序：

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(int argc, char** argv) {
    int fd;
    struct stat sb;
    char* mapped;

    /* 打开文件 */
    if ((fd = open(argv[1], O_RDWR)) < 0) {
        perror("open");
    }

    /* 获取文件的属性 */
    if ((fstat(fd, &sb)) == -1) {
        perror("fstat");
    }
    /* 私有文件映射将无法修改文件 */
    if ((mapped = (char*)mmap(NULL, sb.st_size, PROT_READ | PROT_WRITE,
                              MAP_SHARED, fd, 0)) == (void*)-1) {
        perror("mmap");
    }

    /* 映射完后, 关闭文件也可以操纵内存 */
    close(fd);

    /* 修改一个字符 */
    mapped[20] = '9';

    return 0;
}

```

B 程序修改了文件，因为是共享的，所以会被同步到磁盘中，又因为 A 也是共享的映射，所以内存文件就会被影响到：

```bash
./a data.txt
aaaaaaaaa
bbbbbbbbb
cccccccccccc
dddddd
aaaaaaaaa		<--- B程序启动
bbbbbbbbb
hccccccccccc
...
```

## 三、进程间通信
还可以映射一个匿名文件，它的文件描述符为-1，通过这样的方法来通信：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>

#define BUF_SIZE 100

int main() {
    char* p_map;
    /* 匿名映射,创建一块内存供父子进程通信 */
    p_map = (char*)mmap(NULL, BUF_SIZE, PROT_READ | PROT_WRITE,
                        MAP_SHARED | MAP_ANONYMOUS, -1, 0);

    if (fork() == 0) {
        sleep(1);
        printf("child got a message: %s\n", p_map);
        sprintf(p_map, "%s", "hi, dad, this is son");
        munmap(p_map, BUF_SIZE);  // 实际上，进程终止时，会自动解除映射。
        exit(0);
    }

    sprintf(p_map, "%s", "hi, this is father");
    sleep(2);
    printf("parent got a message: %s\n", p_map);

    return 0;
}
```

运行结果：

```bash
gcc mmapname.c -o main
❯ ./main
child got a message: hi, this is father
parent got a message: hi, dad, this is son
```

在 `fork()` 之前，匿名映射的变量等，父子进程共享。

## 四、mmap() 访问范围
对文件进行映射以后，并不是说映射了多少内存，就能随意读写多少内存。能否访问，主要看文件文件权限&文件大小：

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

int main(int argc, char** argv) {
    int fd, i;
    int pagesize, offset;
    char* p_map;
    struct stat sb;

    /* 取得page size */
    pagesize = sysconf(_SC_PAGESIZE);
    printf("pagesize is %d\n", pagesize);

    /* 打开文件 */
    fd = open(argv[1], O_RDWR, 00777);
    fstat(fd, &sb);
    printf("file size is %zd\n", (size_t)sb.st_size);

    offset = 0;
    p_map = (char*)mmap(NULL, pagesize * 2, PROT_READ | PROT_WRITE, MAP_SHARED,
                        fd, offset);
    close(fd);

    p_map[sb.st_size-1] = '9';
    msync((void*)p_map, sb.st_size, MS_SYNC);

    munmap(p_map, pagesize * 2);
    return 0;
}
```
运行结果：

```bash
aaaaaaaaaaaaaaa
aaaaaaaaaaaaaaa
aaaaaaaaaaaaaaa
aaaaaaaaaaaaaa9
```

可以看到成功得将最后一个字符修改为了 '9'。但是如果我们修改超过文件范围的：

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>

int main(int argc, char** argv) {
    int fd, i;
    int pagesize, offset;
    char* p_map;
    struct stat sb;

    /* 取得page size */
    pagesize = sysconf(_SC_PAGESIZE);
    printf("pagesize is %d\n", pagesize);

    /* 打开文件 */
    fd = open(argv[1], O_RDWR, 00777);
    fstat(fd, &sb);
    printf("file size is %zd\n", (size_t)sb.st_size);

    offset = 0;
    p_map = (char*)mmap(NULL, pagesize * 2, PROT_READ | PROT_WRITE, MAP_SHARED,
                        fd, offset);
    close(fd);

    p_map[sb.st_size] = '9';  // 没有意义，覆盖不上
    p_map[pagesize] = '9';    // bus error

    msync((void*)p_map, sb.st_size, MS_SYNC);
    munmap(p_map, pagesize * 2);
    return 0;
}

```
运行结果：

```bash
❯ gcc mmap.c -o main
❯ ./main data.txt
pagesize is 4096
file size is 63
zsh: bus error  ./main data.txt
```




## 五、参考

参考[这里](https://blog.csdn.net/whatday/article/details/89239331)。
