---
title: "C 项目实践：实现个人日志命令行工具"
date: 2023-12-19 16:43:26
tags:
  - "C"
  - "CLI"
  - "project"
categories:
  - "Programming Projects"
---

<!--more-->

## 〇、前言
中午上课的时候，打开 github 看了一下个人主页，虽然最近很忙，但是这个活动记录有点过于冷清：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/783d3101c6ca4eb7b170c1ea7ea11ad5_1720253679644.png?token=ANB4BCJSYJLQJ6I54RSWTN3GRD6TC)
于是我就想着写一个日志命令行工具，输入以下命令就能将我的日志立即同步到 github 上：

```bash
mylog today is really a long day!
```
之后 **today is really a long day!**  这句话就会被同步到 github 上的今天的文件中了。

## 一、思路以及实现
思路很简单：
- 1、获取当前日期；
- 2、利用当前日期作为日志文件名，后缀为 `_log.txt`；
- 3、从 `argv` 中获取具体日志，写入到日志文件中；
- 4、**git** 相关操作，并上传到远端仓库。

以下就是实现的代码：

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>
// 当前日志名
char filename[100] = {0};
// 文件夹
const char *dir = 0;
// 文件名,用于 commit信息
char real_file_name[100] = {0};
// 获取当前日期

char *getTime() {
    struct tm *timeinfo;
    time_t rawtime;
    char *buffer =
        malloc(sizeof(char) * 80); // 防止返回后,栈空间被冲掉,推荐用堆

    time(&rawtime);
    timeinfo = localtime(&rawtime);

    strftime(buffer, 80, "%Y-%m-%d", timeinfo);
    return buffer;
}

// 生成带有日期文件的文件

FILE *creatFile() {
    char *currentTime = getTime();
    char *lastName = "_log.txt";
    dir = "/Users/******/Daily_Life_Log/";

    snprintf(real_file_name, 100, "%s%s", currentTime, lastName);

    snprintf(filename, sizeof(filename), "%s%s%s", dir, currentTime, lastName);
    FILE *file = fopen(filename, "a");
    if (file != NULL) {
        printf("文件已创建成功!\n");
    } else {
        printf("无法创建文件!\n");
    }
    return file;
}

int main(int argc, char *argv[]) {
    // 安全检查
    if (argc < 2) {
        printf("请输入日志!\n");
        return -1;
    }

    // 往文件中写入文件
    FILE *file = creatFile();
    for (int i = 1; i < argc; i++) {
        fprintf(file, "%s", argv[i]);
        fprintf(file, "%s", " ");
    }
    fprintf(file, "%s", "\n"); // 加入换行符
    fclose(file);

    // git
    if (chdir(dir) == 0) {
        printf("目录切换成功！\n");
    } else {
        perror("chdir");
    }
    // 合成git add 命令

    char git_add[100] = {0};
    snprintf(git_add, sizeof(git_add), "%s%s", "git add ", real_file_name);
    system(git_add);

    // 合成 git commit
    char *command0 = "git commit -m \"";
    char *tag1 = "\"";
    char command1[100] = {0};
    snprintf(command1, sizeof(command1), "%s%s%s", command0, real_file_name, tag1);
    system(command1);

    system("git push");
    return 0;
}
```

## 二、一些细节
上面的代码为了能让编译好的二进制文件在 `~`（用户目录）直接运行（直接输入命令名字）并且 **git** 操作不发生错误，额外多了一些东西，比如**切换目录**、**合成 git 相关命令**。

编译好后，直接在当前目录终端将二进制文件移到 `/usr/local/bin`中，就好了。
```bash
sudo mv ./mylog /usr/local/bin/
```
运行一下：

```bash
~ mylog today I created a interesting mylog tool!
文件已创建成功!
目录切换成功！
[main 511bdd4] 2023-12-19_log.txt
 1 file changed, 1 insertion(+), 1 deletion(-)
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 306 bytes | 306.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To github.com:******  main -> main
```
结果符合预期，成功写入了一条日志，并同步到了远端：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/dd115a76918446aab331df8e660d8339_1720253679644.png?token=ANB4BCNY5XBTEQQR4TSAHQTGRD6TG)
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/57c9d91af5b64b3b8d4c260c2ec43756_1720253687327.png?token=ANB4BCK2LDAUGD3QN4VUQRDGRD6TI)

我想，这可能就是编程的趣味性~

*全文完，感谢阅读！*






