---
title: 简单的 ls 命令的实现（二）
date: 2023-02-17 17:08:47
tags:
    - 开发语言
    - C
categories: 系统编程
---

<!--more-->

 ## 一、前言
 题目要求，实现 ls 的 -a、-l、-R、-t、-r、-i、-s 参数，并允许这些参数任意组合。
 - -a：--all的缩写，显示所有的文件，包括隐藏文件(以.开头的文件)
 - l：列出长数据串，显示出文件的属性与权限等数据信息(常用)
 - -t：以修改时间排序
 - -r：--reverse，将排序结果以倒序方式显示
 - -i：结合-l参数，列出每个文件的inode
 - -s, –size 以块大小为单位列出所有文件的大小
 - -R, –recursive 同时列出所有子目录层

### （一）显示所有的文件，包括隐藏文件
```c
luliang@shenjian linux-6.1.1 % ls -a
.                       .mailmap                LICENSES                crypto                  kernel                  security
..                      .rustfmt.toml           MAINTAINERS             drivers                 lib                     sound
.clang-format           COPYING                 Makefile                fs                      mm                      tools
.cocciconfig            CREDITS                 README                  include                 net                     usr
.get_maintainer.ignore  Documentation           arch                    init                    rust                    virt
.gitattributes          Kbuild                  block                   io_uring                samples
.gitignore              Kconfig                 certs                   ipc                     scripts
```
### （二）列出长数据串，显示出文件的属性与权限等数据信息
```c
luliang@shenjian linux-6.1.1 % ls -l
total 1728
-rw-rw-r--@   1 luliang  staff     496 12 22 00:48 COPYING
-rw-rw-r--@   1 luliang  staff  101639 12 22 00:48 CREDITS
drwxrwxr-x@ 101 luliang  staff    3232 12 22 00:48 Documentation
-rw-rw-r--@   1 luliang  staff    2573 12 22 00:48 Kbuild
-rw-rw-r--@   1 luliang  staff     555 12 22 00:48 Kconfig
drwxrwxr-x@   6 luliang  staff     192 12 22 00:48 LICENSES
-rw-rw-r--@   1 luliang  staff  688447 12 22 00:48 MAINTAINERS
-rw-rw-r--@   1 luliang  staff   70608 12 22 00:48 Makefile
-rw-rw-r--@   1 luliang  staff     727 12 22 00:48 README
drwxrwxr-x@  26 luliang  staff     832 12 22 00:48 arch
drwxrwxr-x@  81 luliang  staff    2592 12 22 00:48 block
drwxrwxr-x@  14 luliang  staff     448 12 22 00:48 certs
drwxrwxr-x@ 152 luliang  staff    4864 12 22 00:48 crypto
drwxrwxr-x@ 141 luliang  staff    4512 12 22 00:48 drivers
drwxrwxr-x@ 157 luliang  staff    5024 12 22 00:48 fs
drwxrwxr-x@  31 luliang  staff     992 12 22 00:48 include
drwxrwxr-x@  17 luliang  staff     544 12 22 00:48 init
drwxrwxr-x@  58 luliang  staff    1856 12 22 00:48 io_uring
drwxrwxr-x@  15 luliang  staff     480 12 22 00:48 ipc
drwxrwxr-x@ 133 luliang  staff    4256 12 22 00:48 kernel
drwxrwxr-x@ 287 luliang  staff    9184 12 22 00:48 lib
drwxrwxr-x@ 135 luliang  staff    4320 12 22 00:48 mm
drwxrwxr-x@  78 luliang  staff    2496 12 22 00:48 net
drwxrwxr-x@  12 luliang  staff     384 12 22 00:48 rust
drwxrwxr-x@  41 luliang  staff    1312 12 22 00:48 samples
drwxrwxr-x@ 158 luliang  staff    5056 12 22 00:48 scripts
drwxrwxr-x@  23 luliang  staff     736 12 22 00:48 security
drwxrwxr-x@  32 luliang  staff    1024 12 22 00:48 sound
drwxrwxr-x@  42 luliang  staff    1344 12 22 00:48 tools
drwxrwxr-x@  11 luliang  staff     352 12 22 00:48 usr
drwxrwxr-x@   5 luliang  staff     160 12 22 00:48 virt
```
### （三）以修改时间排序
```c
luliang@shenjian linux-6.1.1 % ls -t
COPYING         Kconfig         README          crypto          init            lib             samples         tools
CREDITS         LICENSES        arch            drivers         io_uring        mm              scripts         usr
Documentation   MAINTAINERS     block           fs              ipc             net             security        virt
Kbuild          Makefile        certs           include         kernel          rust            sound
```
### （四）将排序结果以倒序方式显示
```c
luliang@shenjian linux-6.1.1 % ls -r
virt            security        net             ipc             fs              block           MAINTAINERS     Documentation
usr             scripts         mm              io_uring        drivers         arch            LICENSES        CREDITS
tools           samples         lib             init            crypto          README          Kconfig         COPYING
sound           rust            kernel          include         certs           Makefile        Kbuild
```
### （五）列出每个文件的inode
```c
luliang@shenjian linux-6.1.1 % ls -i
60508833 COPYING        60508494 MAINTAINERS    60449690 crypto         60527795 ipc            60508518 samples        60508496 virt
60532891 CREDITS        60502255 Makefile       60468171 drivers        60532892 kernel         60526544 scripts
60449877 Documentation  60508495 README         60527868 fs             60527056 lib            60501981 security
60508493 Kbuild         60508848 arch           60502256 include        60527614 mm             60530087 sound
60508492 Kconfig        60533478 block          60449674 init           60466188 net            60459396 tools
60501955 LICENSES       60508834 certs          60527810 io_uring       60508453 rust           60459381 usr
```
### （六）以块大小为单位列出所有文件的大小

```c
luliang@shenjian linux-6.1.1 % ls -s
total 1728
   8 COPYING            1352 MAINTAINERS           0 crypto                0 ipc                   0 samples               0 virt
 200 CREDITS             144 Makefile              0 drivers               0 kernel                0 scripts
   0 Documentation         8 README                0 fs                    0 lib                   0 security
   8 Kbuild                0 arch                  0 include               0 mm                    0 sound
   8 Kconfig               0 block                 0 init                  0 net                   0 tools
   0 LICENSES              0 certs                 0 io_uring              0 rust                  0 usr
```
### （七）同时列出所有子目录层

```c
luliang@shenjian Stusys % ls -aR
.               ..              .DS_Store       stuinfo         stusts

./stuinfo:
.                       .DS_Store               class2的副本.txt        class4的副本.txt
..                      class1.txt              class3的副本.txt        class5的副本.txt

./stusts:
.               ..              1.txt           a.out           haha            stu_man_sys.c   test.c
```
以上就是 7 种参数的组合
## 二、ls -l
`ls -l`需要列出文件类型和许可权限、文件链接数、用户ID、所属组ID、所占空间大小、文件修改时间和文件名称。
而我们可以通过`readdir函数`读取到的文件名，再通过调用函数
`int stat(const char *file_name, struct stat *buf)`获取文件名为`d_name`的文件的详细信息，存储在`stat结构体中`。
以下为`stat结构体`的定义：

```c
struct stat {
	dev_t           st_dev;         /* [XSI] ID of device containing file */
	ino_t           st_ino;         /* [XSI] File serial number */
	mode_t          st_mode;        /* [XSI] Mode of file (see below) */
	nlink_t         st_nlink;       /* [XSI] Number of hard links */
	uid_t           st_uid;         /* [XSI] User ID of the file */
	gid_t           st_gid;         /* [XSI] Group ID of the file */
	dev_t           st_rdev;        /* [XSI] Device ID */
#if !defined(_POSIX_C_SOURCE) || defined(_DARWIN_C_SOURCE)
	struct  timespec st_atimespec;  /* time of last access */
	struct  timespec st_mtimespec;  /* time of last data modification */
	struct  timespec st_ctimespec;  /* time of last status change */
#else
	time_t          st_atime;       /* [XSI] Time of last access */
	long            st_atimensec;   /* nsec of last access */
	time_t          st_mtime;       /* [XSI] Last data modification time */
	long            st_mtimensec;   /* last data modification nsec */
	time_t          st_ctime;       /* [XSI] Time of last status change */
	long            st_ctimensec;   /* nsec of last status change */
#endif
	off_t           st_size;        /* [XSI] file size, in bytes */
	blkcnt_t        st_blocks;      /* [XSI] blocks allocated for file */
	blksize_t       st_blksize;     /* [XSI] optimal blocksize for I/O */
	__uint32_t      st_flags;       /* user defined flags for file */
	__uint32_t      st_gen;         /* file generation number */
	__int32_t       st_lspare;      /* RESERVED: DO NOT USE! */
	__int64_t       st_qspare[2];   /* RESERVED: DO NOT USE! */
};
```
先小试牛刀一下：

```c
#include <stdio.h>
#include <sys/stat.h>
int main(int argc, char* argv[]) {
    struct stat info;
    int ans = stat(argv[1], &info);
    printf("    mode : %o\n", info.st_mode);
    printf("    links: %d\n", info.st_nlink);
    printf("    user : %d\n", info.st_uid);
    printf("    group: %d\n", info.st_gid);
    printf("    size : %d\n", info.st_size);
    printf("    blocks: %d\n", info.st_blocks);
    return 0;
}

```
编译并传入一个测试的参数：

```bash
luliang@shenjian Test % gcc plan4.c -o ls
luliang@shenjian Test % ./ls plan3.c
    mode : 100644
    links: 1
    user : 501
    group: 20
    size : 1136
    blocks: 8
```
可以看到达到了预期效果。
但是，刚获得到的一些信息，比如 `mode`、`user` 、`group`，为了方便阅读，还需要将它们转化成字符。
### （一）对 st_mode进行转化
我们想要的信息需要从成员`st_mode`中获取，其类型`mode_t`在我的系统中定义就是`unsigned int`。但在命令`ls -l`的输出应该是类似`-rw-rw-rw-`的10个字符的字符串。其实这些信息是以位图的方式放进了这个16位（2B）整数类型中。
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/ce573cf49dce4edd92a9506cd303b07a_1720253841019.png?token=ANB4BCKZRHG6E3ADNZTS53TGRD65I)
- 其中前4位用作文件类型。
- 接下来3位是文件特殊属性，分别是`set-user-ID`，`set-group-ID`和`sticky`位，1表示具有该属性，0表示没有。
- 最后9位就是我们想找的许可权限了，分为3组，分别对应文件所有者、同组用户、其他用户的读、写、执行权限。同样有的话为1，没有的话为0。

既然是位图，那么必然可以通过掩码来取得其信息。如关于文件类型，在`<sys/stat.h>`中定义了`掩码S_IFMT`来取得文件类型信息，用法如下:

```c
switch (sb.st_mode & S_IFMT) {
	case S_IFBLK:  printf("block device\n");            break;
	case S_IFCHR:  printf("character device\n");        break;
	case S_IFDIR:  printf("directory\n");               break;
	case S_IFIFO:  printf("FIFO/pipe\n");               break;
	case S_IFLNK:  printf("symlink\n");                 break;
	case S_IFREG:  printf("regular file\n");            break;
	case S_IFSOCK: printf("socket\n");                  break;
	default:       printf("unknown?\n");                break;
}
```
而掩码`S_IFMT`以及表示相应文件类型的`S_IFBLK`等实际上是一些**八进制数**的宏。
因此，可以自然而然地用一个字符数组，将**翻译**得到的信息储存起来：

```c
void mode_to_letters(int mode, char str[]) {
    strcpy(str, "----------");
    if (S_ISDIR(mode))
        str[0] = 'd';
    if (S_ISCHR(mode))
        str[0] = 'c';
    if (S_ISBLK(mode))
        str[0] = 'b';
    if (mode & S_IRUSR)
        str[1] = 'r';
    if (mode & S_IWUSR)
        str[2] = 'w';
    if (mode & S_IXUSR)
        str[3] = 'x';
    if (mode & S_IRGRP)
        str[4] = 'r';
    if (mode & S_IWGRP)
        str[5] = 'w';
    if (mode & S_IXGRP)
        str[6] = 'x';
    if (mode & S_IROTH)
        str[7] = 'r';
    if (mode & S_IWOTH)
        str[8] = 'w';
    if (mode & S_IXOTH)
        str[9] = 'x';
}
```
### （二）获取链接数、文件大小和时间信息
1、链接信息即保存在成员`st_nlinks`里，直接打印即可。

2、文件大小信息即保存在成员`st_size`里，同样直接打印即可。

3、时间信息即保存在成员`st_mtime`里，需要注意的是这里需要使用`ctime()`函数来将其转换成要求的格式来打印。

```c
#include <time.h>
// 返回的是类似"Wed Jun 30 21:49:08 1993\n"一个字符串
char *ctime(const time_t *timep);
```
### （三）获取所有者和组名信息
首先需要知道Linux系统的用户信息保存在`/etc/passwd`中，但这个文件又没有包括所有的用户，在一些网络计算机系统中，所有主机通过`NIS`来进行身份验证，本地只保存所有用户的一个自己以备离线操作。

这里可以通过`getwuid()`来获取完整的用户列表，该调用会自动选择从`/etc/passwd`还是`NIS`中获取信息，提高了程序的可移植性。

```c
#include <pwd.h>
// 接受用户的uid作为参数，返回指针指向的结构体中包含了用户信息
struct passwd *getpwuid(uid_t uid);

struct passwd {
	char   *pw_name;       /* username */
	char   *pw_passwd;     /* user password */
	uid_t   pw_uid;        /* user ID */
	gid_t   pw_gid;        /* group ID */
	char   *pw_gecos;      /* user information */
	char   *pw_dir;        /* home directory */
	char   *pw_shell;      /* shell program */
};
```
所以可以调用`getpwuid()`来获取用户信息，打印用户名称。另外还有一种可能是，文件所有者已经搬走了，账号被删除，但这个文件还在，此时该函数将返回NULL。所以这里还需要检查返回值，而不能直接读取，否则可能出现段错误。

类似的使用`getgrgid()`来获取组列表，用法大同小异:

```c
#include <grp.h>
struct group *getgrgid(gid_t gid);

struct group {
	char   *gr_name;        /* group name */
	char   *gr_passwd;      /* group password */
	gid_t   gr_gid;         /* group ID */
	char  **gr_mem;         /* NULL-terminated array of pointers
                                to names of group members */
};
```
需要注意的是，该函数也是有可能返回NULL的，就不细说了。

