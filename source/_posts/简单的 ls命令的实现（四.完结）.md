---
title: 简单的 ls命令的实现（四.完结）
date: 2023-03-08 22:08:59
tags: c++ 开发语言
categories: 系统编程
---

<!--more-->

## 〇、思维导图
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/97cc560b7093443892dcef212fe2e119_1720253841019.png?token=ANB4BCP75M5TOOKRQHXIMJDGRD65E)


## 一、准备工作
### （一）对控制参数的处理
一共有 7 个可选参数，分别是`-a、-l、-R、-t、-r、-i、-s`，这些参数可以相互自由组合，因此可以设计一种机制，就是直接把它们全部用循环一次性做`或运算`，得到一个参数标记Vec。

```c
// 标记: -a、-l、-R、-t、-r、-i、-s 参数(向量分量)
#define a 0b1000000
#define l 0b0100000
#define R 0b0010000
#define t 0b0001000
#define r 0b0000100
#define I 0b0000010
#define s 0b0000001
// 向量
int Vec = 0;
```
而 Vec 可以使用全局变量，这样可以避免写函数时不断地给函数参数加入地址参数，使得更加代码整洁，更直观。
### （二）对 dir 参数的处理
同理，依然可以设计一个全局容器，不断地把 `dirname `扔进去：

```c
char* dirname[4096 * 128];
int dirlen = 0;
```
而对于 `filename `也是一样的，但在每次遍历一个`dir `之前，就得`filename `容器做重置处理：

```c
char* filenames[4096 * 128];
int file_cnt = 0;
```
### （三）函数实现

```c
void tags_cal(int argc, char* argv[]) {
    for (int i = 1; i < argc; i++) {
        if (argv[i][0] !=
            '-') {  // 只接受以'-'开头的参数,其它参数要么错误,要么是文件夹名称或文件名
            char* tempdirname = (char*)malloc(sizeof(char) * 4096);
            strcpy(tempdirname, argv[i]);
            dirname[dirlen++] = tempdirname;
        } else {
            int len = strlen(argv[i]);
            for (int j = 1; j < len; j++) {
                switch (argv[i][j]) {
                    case 'a':
                        Vec |= a;
                        break;
                    case 'l':
                        Vec |= l;
                        break;
                    case 'R':
                        Vec |= R;
                        break;
                    case 't':
                        Vec |= t;
                        break;
                    case 'r':
                        Vec |= r;
                        break;
                    case 'i':
                        Vec |= I;
                        break;
                    case 's':
                        Vec |= s;
                        break;
                    default:
                        fprintf(stderr, "%c参数错误!\n", argv[i][j]);
                        break;
                }
            }
        }
    }
    if (dirlen == 0) {
        dirlen = 1;
        char* tempdirname = (char*)malloc(sizeof(char) * 2048);
        strcpy(tempdirname, ".");
        dirname[0] = tempdirname;
    }
}
```
这里需要注意的是，如果`dirlen == 0`，说明我们的命令并没有加参数，默认是对当前文件夹进行操作，因此需要重新对 `dirlen`赋值为 `1`，然后把 `dirname[0]`置为`"."`。

## 二、实现
我们上一步成功得到了，`dirnname`、`dirlen`，这样就可以逐个`dirname[i]`进行处理了！

```c
void do_myls() {
    for (int i = 0; i < dirlen; i++) {
        if (do_name(dirname[i]) == -1) {
            continue;
        }
        // 且自动字典排序
        if ((Vec & t) == t) {  // 时间排序
            do_t(filenames);
        }
        if ((Vec & r) == r) {  // 逆序
            do_r(filenames, file_cnt);
        }
        printf("当前路径:\"%s\"\n", dirname[i]);
        int tag = 0;  // 换行
        for (int j = 0; j < file_cnt; j++) {
            // 拼凑文件名
            char path[4096] = {0};
            strcpy(path, dirname[i]);
            int len = strlen(dirname[i]);
            strcpy(&path[len], "/");
            strcpy(&path[len + 1], filenames[j]);
            tag++;
            if ((Vec & a) == 0) {
                if ((strcmp(filenames[j], ".") == 0 ||
                     strcmp(filenames[j], "..") == 0) ||
                    filenames[j][0] == '.') {
                    continue;
                }
            }
            struct stat info;
            stat(path, &info);  // 拉进 info
            if (S_ISDIR(info.st_mode) && ((Vec & R) == R)) {
                // 如果是目录,那就直接拉进 dirnames:"dirname/filename"
                char* tempdirname = (char*)malloc(sizeof(char) * 4096);
                strcpy(tempdirname, dirname[i]);
                int len = strlen(tempdirname);
                strcpy(&tempdirname[len], "/");
                strcpy(&tempdirname[len + 1], filenames[j]);
                dirname[dirlen++] = tempdirname;
            }
            if ((Vec & I) == I) {
                do_i(path);
            }
            if ((Vec & s) == s) {
                do_s(path);
            }
            if ((Vec & l) == 0) {
                if (S_ISDIR(info.st_mode))  // 判断是否为目录
                {
                    printf(GREEN "%s\t" NONE, filenames[j]);
                } else {
                    printf(BLUE "%s\t" NONE, filenames[j]);
                }
            }
            if ((Vec & l) == l) {
                void mode_to_letters();
                char modestr[11];
                mode_to_letters(info.st_mode, modestr);
                printf("%s ", modestr);
                printf("%4d ", (int)info.st_nlink);
                printf("%-8s ", uid_to_name(info.st_uid));
                printf("%-8s ", gid_to_name(info.st_gid));
                printf("%8ld ", (long)info.st_size);
                printf("%.12s ", ctime(&info.st_mtime));
                if (S_ISDIR(info.st_mode))  // 判断是否为目录
                {
                    printf(GREEN "%s\t" NONE, filenames[j]);
                } else {
                    printf(BLUE "%s\t" NONE, filenames[j]);
                }
                printf("\n");
            }
            if ((tag % 5 == 0) && ((Vec & l) == 0)) {
                printf("\n");
            }
        }
        // 清空容器
        for (int k = 0; k < file_cnt; k++) {
            memset(filenames[k], 4096, '\0');
        }
        file_cnt = 0;
    }
}
```
这里最关键的就是对`-R`参数的处理，因为我们的整体框架并不适合做函数的递归，因此我们可以在判断某个` filename`是一个 `dir` 之后，就可以把它加入到 `dirname` 中，并且把 `dirlen++`，这样就在逻辑上实现了遍历，这里也充分利用了全局变量的优势：牵一发而动全身。

```c
struct stat info;
            stat(path, &info);  // 拉进 info
            if (S_ISDIR(info.st_mode) && ((Vec & R) == R)) {
                // 如果是目录,那就直接拉进 dirnames:"dirname/filename"
                char* tempdirname = (char*)malloc(sizeof(char) * 4096);
                strcpy(tempdirname, dirname[i]);
                int len = strlen(tempdirname);
                strcpy(&tempdirname[len], "/");
                strcpy(&tempdirname[len + 1], filenames[j]);
                dirname[dirlen++] = tempdirname;
            }
```
而其它的功能性函数实现起来就很简单了，就不累赘了。

## 三、完整代码

```c
#include <dirent.h>
#include <grp.h>
#include <pwd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <time.h>
#include <unistd.h>
// 标记: -a、-l、-R、-t、-r、-i、-s 参数(向量分量)
#define a 0b1000000
#define l 0b0100000
#define R 0b0010000
#define t 0b0001000
#define r 0b0000100
#define I 0b0000010
#define s 0b0000001
// 颜色宏
#define NONE "\033[m"
#define GREEN "\033[0;32;32m"
#define BLUE "\033[0;32;34m"
// 函数声明
void tags_cal(int argc, char* argv[]);
void restored_ls(struct dirent* cur_item);
void sort(char** filenames, int start, int end);
void do_r(char** filenames, int file_cnt);
int partition(char** filenames, int start, int end);
void swap(char** s1, char** s2);
int compare(char* s1, char* s2);
char* uid_to_name(uid_t);
char* gid_to_name(gid_t);
void mode_to_letters(int, char[]);
char* uid_to_name(uid_t);
// ********函数声明********
void do_i(char filename[]);
void do_s(char filename[]);
int do_name(char dirname[]);
void do_myls();
void do_t(char** filenames);
int Vec = 0;
char* dirname[4096 * 128];
int dirlen = 0;
char* filenames[4096 * 128];
int file_cnt = 0;
int main(int argc, char* argv[]) {
    tags_cal(argc, argv);
    do_myls();
    return 0;
}
void do_myls() {
    for (int i = 0; i < dirlen; i++) {
        if (do_name(dirname[i]) == -1) {
            continue;
        }
        // 且自动字典排序
        if ((Vec & t) == t) {  // 时间排序
            do_t(filenames);
        }
        if ((Vec & r) == r) {  // 逆序
            do_r(filenames, file_cnt);
        }
        printf("当前路径:\"%s\"\n", dirname[i]);
        int tag = 0;  // 换行
        for (int j = 0; j < file_cnt; j++) {
            // 拼凑文件名
            char path[4096] = {0};
            strcpy(path, dirname[i]);
            int len = strlen(dirname[i]);
            strcpy(&path[len], "/");
            strcpy(&path[len + 1], filenames[j]);
            tag++;
            if ((Vec & a) == 0) {
                if ((strcmp(filenames[j], ".") == 0 ||
                     strcmp(filenames[j], "..") == 0) ||
                    filenames[j][0] == '.') {
                    continue;
                }
            }
            struct stat info;
            stat(path, &info);  // 拉进 info
            if (S_ISDIR(info.st_mode) && ((Vec & R) == R)) {
                // 如果是目录,那就直接拉进 dirnames:"dirname/filename"
                char* tempdirname = (char*)malloc(sizeof(char) * 4096);
                strcpy(tempdirname, dirname[i]);
                int len = strlen(tempdirname);
                strcpy(&tempdirname[len], "/");
                strcpy(&tempdirname[len + 1], filenames[j]);
                dirname[dirlen++] = tempdirname;
            }
            if ((Vec & I) == I) {
                do_i(path);
            }
            if ((Vec & s) == s) {
                do_s(path);
            }
            if ((Vec & l) == 0) {
                if (S_ISDIR(info.st_mode))  // 判断是否为目录
                {
                    printf(GREEN "%s\t" NONE, filenames[j]);
                } else {
                    printf(BLUE "%s\t" NONE, filenames[j]);
                }
            }
            if ((Vec & l) == l) {
                void mode_to_letters();
                char modestr[11];
                mode_to_letters(info.st_mode, modestr);
                printf("%s ", modestr);
                printf("%4d ", (int)info.st_nlink);
                printf("%-8s ", uid_to_name(info.st_uid));
                printf("%-8s ", gid_to_name(info.st_gid));
                printf("%8ld ", (long)info.st_size);
                printf("%.12s ", ctime(&info.st_mtime));
                if (S_ISDIR(info.st_mode))  // 判断是否为目录
                {
                    printf(GREEN "%s\t" NONE, filenames[j]);
                } else {
                    printf(BLUE "%s\t" NONE, filenames[j]);
                }
                printf("\n");
            }
            if ((tag % 5 == 0) && ((Vec & l) == 0)) {
                printf("\n");
            }
        }
        // 清空容器
        for (int k = 0; k < file_cnt; k++) {
            memset(filenames[k], 4096, '\0');
        }
        file_cnt = 0;
    }
}
void tags_cal(int argc, char* argv[]) {
    for (int i = 1; i < argc; i++) {
        if (argv[i][0] !=
            '-') {  // 只接受以'-'开头的参数,其它参数要么错误,要么是文件夹名称或文件名
            char* tempdirname = (char*)malloc(sizeof(char) * 4096);
            strcpy(tempdirname, argv[i]);
            dirname[dirlen++] = tempdirname;
        } else {
            int len = strlen(argv[i]);
            for (int j = 1; j < len; j++) {
                switch (argv[i][j]) {
                    case 'a':
                        Vec |= a;
                        break;
                    case 'l':
                        Vec |= l;
                        break;
                    case 'R':
                        Vec |= R;
                        break;
                    case 't':
                        Vec |= t;
                        break;
                    case 'r':
                        Vec |= r;
                        break;
                    case 'i':
                        Vec |= I;
                        break;
                    case 's':
                        Vec |= s;
                        break;
                    default:
                        fprintf(stderr, "%c参数错误!\n", argv[i][j]);
                        break;
                }
            }
        }
    }
    if (dirlen == 0) {
        dirlen = 1;
        char* tempdirname = (char*)malloc(sizeof(char) * 2048);
        strcpy(tempdirname, ".");
        dirname[0] = tempdirname;
    }
}
void do_i(char filename[]) {
    struct stat info;
    if (stat(filename, &info) == -1)
        perror(filename);
    printf("%llu\t", info.st_ino);
}
void do_s(char filename[]) {
    struct stat info;
    if (stat(filename, &info) == -1)
        perror(filename);
    printf("%4llu\t", info.st_size / 4096 * 4 + (info.st_size % 4096 ? 4 : 0));
}
int do_name(char dirname[]) {
    int i = 0;
    int len = 0;
    DIR* dir_ptr;
    struct dirent* direntp;
    if ((dir_ptr = opendir(dirname)) == NULL) {
        fprintf(stderr, "权限不够,cannot open: %s\n", dirname);
        return -1;
    } else {
        while ((direntp = readdir(dir_ptr))) {
            restored_ls(direntp);
        }
        sort(filenames, 0, file_cnt - 1);
    }
    printf("\n");
    closedir(dir_ptr);
    return 1;
}
void sort(char** filenames, int start, int end) {
    if (start < end) {
        int position = partition(filenames, start, end);
        sort(filenames, start, position - 1);
        sort(filenames, position + 1, end);
    }
}
int partition(char** filenames, int start, int end) {
    if (!filenames)
        return -1;
    char* privot = filenames[start];
    while (start < end) {
        while (start < end && compare(privot, filenames[end]) < 0)
            --end;
        swap(&filenames[start], &filenames[end]);
        while (start < end && compare(privot, filenames[start]) >= 0)
            ++start;
        swap(&filenames[start], &filenames[end]);
    }
    return start;
}
void swap(char** s1, char** s2) {
    char* tmp = *s1;
    *s1 = *s2;
    *s2 = tmp;
}
int compare(char* s1, char* s2) {
    if (*s1 == '.')
        s1++;
    if (*s2 == '.')
        s2++;
    while (*s1 && *s2 && *s1 == *s2) {
        ++s1;
        ++s2;
        if (*s1 == '.')
            s1++;
        if (*s2 == '.')
            s2++;
    }
    return *s1 - *s2;
}
void restored_ls(struct dirent* cur_item) {
    char* result = (char*)malloc(sizeof(char) * 4096);
    strcpy(result, cur_item->d_name);
    filenames[file_cnt++] = result;
}
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
char* gid_to_name(gid_t gid) {
    struct group *getgrgid(), *grp_ptr;
    static char numstr[10];
    if ((grp_ptr = getgrgid(gid)) == NULL) {
        sprintf(numstr, "%d", gid);
        return numstr;
    } else {
        return grp_ptr->gr_name;
    }
}
char* uid_to_name(gid_t uid) {
    struct passwd* getpwuid();
    struct passwd* pw_ptr;
    static char numstr[10];
    if ((pw_ptr = getpwuid(uid)) == NULL) {
        sprintf(numstr, "%d", uid);
        return numstr;
    } else {
        return pw_ptr->pw_name;
    }
}
void do_t(char** filenames) {
    char temp[2048] = {0};
    struct stat info1;
    struct stat info2;
    for (int i = 0; i < file_cnt - 1; i++) {
        for (int j = i + 1; j < file_cnt; j++) {
            stat(filenames[i], &info1);
            stat(filenames[j], &info2);
            if (info1.st_mtime < info2.st_mtime) {
                strcpy(temp, filenames[i]);
                strcpy(filenames[i], filenames[j]);
                strcpy(filenames[j], temp);
            }
        }
    }
}
void do_r(char** arr, int file_cnt) {
	// 只需要修改指针
    char left = 0;              
    char right = file_cnt - 1;  
    char temp;
    while (left < right) {
        char* temp = arr[left];
        arr[left] = arr[right];
        arr[right] = temp;
        left++;   
        right--; 
    }
}
```
## 四、总结
这个 `myls`的难点在于整个系统的设计，比如参数怎么处理，怎么根据参数输出相应的信息。而函数的实现就比较简单。




