---
title: "实现简易 ls 命令（一）：dirent.h 与目录读取"
date: 2023-02-15 12:35:16
tags:
  - "C"
  - "ls"
  - "Unix"
categories:
  - "Systems Programming"
---

<!--more-->

## 一、前言
前一段时间，我接到小组的一个小题目，要求实现部分ls 命令，这可把我难住了。于是我想着先实现一个简单的 ls 命令。
## 二、dirent.h
`dirent.h `是 C 标准库中的一个头文件，用于访问目录中的文件和子目录。它包含了一些数据类型和函数原型，用于实现对目录的打开、读取和关闭等操作，例如 `opendir()`、`readdir()`、`closedir() `等函数。在 Unix 和 Linux 系统中，`dirent.h` 是一个常用的头文件。

```c
#include <dirent.h>
DIR *opendir(const char *dirname);
struct dirent *readdir(DIR *dirp);
int closedir(DIR *dirp);
```
具体而言，`opendir`用于打开目录，和流类似，返回一个指向`DIR结构体`的指针，参数`*dirname`是一个字符数组或者字符串常量；`readdir`函数用于读取目录，只有一个参数，就是`opendir`返回的结构体指针，或者叫句柄更容易理解些吧。这个函数也返回一个结构体指针 `dirent *`。
## 三、opendir()
`opendir()`是一个打开目录的函数，它的参数可以是一个目录的名字，比如`.`、`..`、`User`等等。它的返回值是一个 `DIR结构体指针`。
`DIR` 结构体的定义为：

```c
typedef struct {
	int	__dd_fd;	/* file descriptor associated with directory */
	long	__dd_loc;	/* offset in current buffer */
	long	__dd_size;	/* amount of data returned */
	char	*__dd_buf;	/* data buffer */
	int	__dd_len;	/* size of data buffer */
	long	__dd_seek;	/* magic cookie returned */
	__unused long	__padding; /* (__dd_rewind space left for bincompat) */
	int	__dd_flags;	/* flags for readdir */
	__darwin_pthread_mutex_t __dd_lock; /* for thread locking */
	struct _telldir *__dd_td; /* telldir position recording */
} DIR;
```
## 四、readdir()
`readdir`函数用于读取目录，只有一个参数为 `DIR*类型`，它也返回一个结构体指针 `dirent *`。`readdir`在每次使用后，`readdir`会读到下一个文件，依次读出目录中的所有文件，每次只能读一个
`dirent`结构体的定义为：

```c
#define __DARWIN_STRUCT_DIRENTRY { \
	__uint64_t  d_ino;      /* file number of entry */ \
	__uint64_t  d_seekoff;  /* seek offset (optional, used by servers) */ \
	__uint16_t  d_reclen;   /* length of this record */ \
	__uint16_t  d_namlen;   /* length of string in d_name */ \
	__uint8_t   d_type;     /* file type, see below */ \
	char      d_name[__DARWIN_MAXPATHLEN]; /* entry name (up to MAXPATHLEN bytes) */ \
}

#if __DARWIN_64_BIT_INO_T
struct dirent __DARWIN_STRUCT_DIRENTRY;
```
对于本次作业，这里面最关键的信息就是 `d_name`，也就是我们的文件名称了。到这一步，基本就完成这个简单任务了。
## 五、代码实现

```c
#include <dirent.h>
#include <stdio.h>
#include <string.h>
void show_ls(char filename[]);
int main(int argc, char* argv[]) {
    // 如果只有一个参数,说明这个命令后面没有加任何参数,也就是./out
    if (argc == 1) {
        show_ls(".");
    }
    // 如果参数的个数不是 1,就循环打印出所有的目录
    while (--argc) {
        // 打印出文件名称
        printf("%s: \n", *++argv);
        show_ls(*argv);
        printf("\n");
    }
    return 0;
}
void show_ls(char filename[]) {
    DIR* dir_ptr;            // the directory
    struct dirent* direntp;  // each entry
    dir_ptr = opendir(filename);
    // dir_ptr == NULL,说明这个目录名称有误
    if (dir_ptr == NULL) {
        fprintf(stderr, "ls1: cannot open%s \n", filename);
        // 而stderr是无缓冲的，会直接输出。
    }
    while ((direntp = readdir(dir_ptr)) != NULL) {
        // readdir()在每次使用后，readdir会读到下一个文件，readdir是依次读出目录中的所有文件，每次只能读一个
        printf("%-10s", direntp->d_name);
    }
    closedir(dir_ptr);  // 关闭流
}
```
编译一下：

```bash
gcc plan3.c -o ls
```
运行一下：

```bash
./ls . .. ../CPP
```
输出为：

```bash
uliang@shenjian Test % ./ls . .. ../CPP
.: 
.         ..        plan4.c   3.c       father    father.c  ll        plan3.c   child.c   a.out     1.c       5.c       ls        2.c       child     6.c       
..: 
.         ..        .DS_Store Stusys    Test      Plan      Kernal    Small-Git Exercise  CPP       Linux     C_small_projectsDahuaDS   Luogu     .vscode   NetSafe   ACM       .idea     
../CPP: 
.         ..        03        04        05        02        .DS_Store test      DS        11        16        17        10        07        00        README.md 09        08        01        06        .git      15        12        13        14        .idea     
```
完全符合我们的编程目标~

*本文结束，感谢你的阅读。*




