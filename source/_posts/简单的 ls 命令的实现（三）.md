---
title: 简单的 ls 命令的实现（三）
date: 2023-02-17 18:22:21
tags: c++ 开发语言
categories: 系统编程
---

<!--more-->

（接上）
## 思考：如何处理命令的参数?
ls 命令后面可以跟进一些参数，比如一个命令可以是这样的：`ls -a -l -rit User . .. `，这时候，就一定要对输入的参数进行一定的处理。
观察容易想到，可选的命令参数，不管是`-a -l -i`，还是`-rit`，这些参数前面都有一个`“-”`符号。而文件夹的名字或者文件的名字，前面都没有这些，这就简单很多了。
首先，一共有 7 个可选参数：

```c
// 标记: -a、-l、-R、-t、-r、-i、-s 参数(向量分量)
int tag_a = 0b1000000;
int tag_l = 0b0100000;
int tag_R = 0b0010000;
int tag_t = 0b0001000;
int tag_r = 0b0000100;
int tag_i = 0b0000010;
int tag_s = 0b0000001;
```
然后设置一个向量，用它来存储所有的参数，形成一个位图：

```c
// 设计可选参数向量
int Vec = 0;
```
紧接着用`dirname[]`来存储从参数处接受的文件夹。



```c
// 存储dirname
char* dirname[2018];
// 存放文件夹参数的数量
int dirlen = 0;
```
然后用以下程序生成向量 `Vec`，以及 `dirname 数组`。

```c
void tags_cal(int argc, char* argv[]) {
    for (int i = 1; i < argc; i++) {
        // 只接受以'-'开头的参数,其它参数要么错误,要么是文件夹名称
        if (argv[i][0] != '-') {
            char* tempdirname = (char*)malloc(sizeof(char) * 1024);
            strcpy(tempdirname, argv[i]);
            dirname[dirlen++] = tempdirname;
        } else {
            int len = strlen(argv[i]);
            for (int j = 1; j < len; j++) {
                switch (argv[i][j]) {
                    case 'a':
                        Vec |= tag_a;
                        break;
                    case 'l':
                        Vec |= tag_l;
                        break;
                    case 'R':
                        Vec |= tag_R;
                        break;
                    case 't':
                        Vec |= tag_t;
                        break;
                    case 'r':
                        Vec |= tag_r;
                        break;
                    case 'i':
                        Vec |= tag_i;
                        break;
                    case 's':
                        Vec |= tag_s;
                        break;
                    default:
                        break;
                }
            }
        }
        if (dirlen == 0) {
            dirlen = 1;
            dirname[0] = ".";
        }
    }
}
```
这里需要注意的是，如果只输入参数，比如` ls -a -l `，这样就得在程序的末尾加上：

```c
if (dirlen == 0) {
            dirlen = 1;
            dirname[0] = ".";
        }
```
虽然现在还不能解决文件名输入的问题，但是也很简单。
以下是实现`ls -l`的程序：

```c
void do_ls_l(char dirname[]) {
    DIR* dir_ptr;
    struct dirent* direntp;
    if ((dir_ptr = opendir(dirname)) == NULL)
        fprintf(stderr, "lsl:cannot open %s\n", dirname);
    else {
        while ((direntp = readdir(dir_ptr))) {
            restored_ls(direntp);
        }
        sort(filenames, 0, file_cnt - 1);
        int j = 0;
        for (j = 0; j < file_cnt; ++j) {
            if (filenames[j][0] == '.')
                continue;
            char temp1[PATH_MAX];
            sprintf(temp1, "%s/%s", dirname, filenames[j]);
            dostat(temp1, filenames[j]);
        }

        closedir(dir_ptr);
    }
}
void dostat(char* path, char* filename) {
    struct stat info;
    if (stat(path, &info) == -1)
        perror(path);
    else
        show_file_info(path, filename, &info);
}
void show_file_info(char* path, char* filename, struct stat* info_p) {
    char *uid_to_name(), *ctime(), *git_to_name(), *filemode();
    void mode_to_letters();
    char modestr[11];
    struct stat info;
    if (stat(path, &info) == -1)
        perror(path);
    int color = get_color(info);
    if ((Vec & 0b0000001) == 1) {
        long long size = info_p->st_size / 1024;
        if (size <= 4)
            printf("4   ");
        else
            printf("%-4lld", size);
    }
    mode_to_letters(info_p->st_mode, modestr);
    if ((Vec & 0b1100010) == 0b1100010 || (Vec & 0b0100010) == 0b0100010)
        printf("%ul ", info_p->st_ino);
    printf("%s ", modestr);
    printf("%4d ", (int)info_p->st_nlink);
    printf("%-8s ", uid_to_name(info_p->st_uid));
    printf("%-8s ", gid_to_name(info_p->st_gid));
    printf("%8ld ", (long)info_p->st_size);
    printf("%.12s ", 4 + ctime(&info_p->st_atimespec));
    printf_name1(filename, color);
    printf("\n");
}
```
## 二、ls -a
这一步骤实现起来非常简单，此步骤就是为了不显示隐藏文件，我们可以在执行到这一步时。对参数进行判断，查看是否有参数` -a `若有该参数，直接打印就好了。

## 三、ls -s
此步骤也极为简单，在`结构体stat`中存有该文件的所占字节的大小，`st_size`。我们只需要对该文件字节大小除以1024就可以得到所需显示的文件大小。需要注意的是，除以1024后得到的结果小于等于4且不等于0时，也需要显示为4。
实现起来也很简单，当检测到` -s `参数时执行下面代码即可：

```c
struct stat info;
    if (stat(filename, &info) == -1)
        perror(filename);
    long long size = info.st_size >> 10; // 左移更快
    if (size <= 4) {
        printf("4   ");
    } else {
        printf("%-4lld", size);
    }
```
## 四、ls -i
文件的索引信息也是保存在`结构体stat`中的`st_ino`中的,当检测到参数` -i `执行下面即可。

```c
struct stat info;
    if(stat(filenames[j],&info)==-1)
       perror(filenames[j]);
     printf("%d  ",info.st_ino); 

```


