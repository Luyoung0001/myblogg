---
title: C语言实现删除匹配的某一行文本
date: 2023-02-24 11:48:44
tags:
    - c语言
    - 开发语言
categories: Things about C
---

<!--more-->

## 一、前言
在处理文本文件需要对某行文本进行查询、修改、删除操作，本文采用了创建中间缓冲文件`buff.txt`的思想对这一删除操作进行实现。
## 二、代码

```c
#include <stdio.h>
#include <string.h>
void ChangeFile( char *FileName, char *KeyStr) {
    FILE* fp1 = fopen(FileName, "r");
    if (fp1 == NULL) {
        printf("打开文件失败!\n");
        return;
    }
    // 创建一个临时文件
    FILE* fp2 = fopen("buff.txt", "a");
    // 给 buff 里面写;
    char strLine[1024] = {0};
    while (!feof(fp1)) {
        fgets(strLine, 1024, fp1);
        char* ans = strstr(strLine, KeyStr);
        if (ans != NULL) {
            continue;
        } else {
            fprintf(fp2, "%s", strLine);
        }
    }
    fclose(fp1);
    fclose(fp2);
    // 删除原来的文件,再将 buff 文件的名称修改为 FileName
    remove(FileName);
    rename("buff.txt", FileName);
}
int main() {
    char *FileName = "/Users/CProjects/Stusys/stusts/1.txt";
    char KeyStr = "2222";
    ChangeFile(FileName, KeyStr);
    return 0;
}
```
## 三、效果
比如我们的 `1.txt` 文本文件以下内容:

> fqeasdqecwqe
dfsdfdsfsd
sdfsdfsdfsd
2222fefwefwe
fwefwesdw2222
fewsdcedsf
redscxwescr3334234
3sdfsa
dfsdfdsfsdewfd
c
zs
cwe
fewsdcedsfwe
fewsdcedsfewf
w
xasxcascs

编译运行程序，运行结果如下：

> fqeasdqecwqe
dfsdfdsfsd
sdfsdfsdfsd
fewsdcedsf
redscxwescr3334234
3sdfsa
dfsdfdsfsdewfd
c
zs
cwe
fewsdcedsfwe
fewsdcedsfewf
w
xasxcascs

成功地删除了两行：

> 2222fefwefwe
fwefwesdw2222

## 四、总结
我们用`FILE* fp2 = fopen("buff.txt", "a");`这一行代码，创建和打开一个中间缓冲文件，它默认创建在和运行程序同一个文件夹下面。我们自然希望它和原来被修改的文件等价，因此就把它放在和我们要修改的 `1.txt`同一个目录下。

之后利用：

```c
remove(FileName);
rename("buff.txt", FileName);
```
两行代码将原来的文件删除以及对缓冲文件进行改名，就达到了修改的预期效果。

*全文完，感谢阅读。*

