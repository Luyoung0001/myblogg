---
title: "C 语言入门路线：从 Linux 基础到项目实践"
date: 1990-10-20 15:03:13
tags:
  - "C"
  - "learning roadmap"
  - "Linux"
categories:
  - "Programming Languages"
---

## 前言
本文提供一条清晰的学习路线，涵盖 Linux 基础、C 语言、Makefile、Git & GitHub、编译与链接等核心内容。目标是帮助读者建立扎实的开发基础，理解系统底层原理，掌握现代开发工具链。

## 学习建议

对于新手，先搭建 Linux 开发环境，然后在 Linux 上学习 C 语言，可以验证书上的内容，学习写 Makefil 来编译自己的代码，同时使用 git 和 github 或者 gitee 来管理自己的代码。这些可以同步进行，当然，同学们可以先学 Linux 好一些，Linux101 里面都会涉及到这些问题。

需要明白的是，C 语言绝对不是一个简单的 C 语言学习，它是一个生态。

## Linux 基础

参考 Linux101，目的是为了在 Linux 环境下开发的时候，从敲一个命令能认识到背后发生了什么。

参考资料是 [Linux101](https://101.lug.ustc.edu.cn/)。

学习完后，应该能回答以下问题：
- 什么是内置命令，什么是外置命令，有什么区别，如何查看一个命令是不是内置命令？

- 什么是 shell，Linux 的 shell 有哪些，如何设置默认的 shell？

- sudo 是什么，为什么有的命令去要 sudo 权限，有的命令不需要？

- shell 的工作原理是什么？和 fork 有什么关系？

- 什么是僵尸晋进程？如何创造一个僵尸进程（用代码示例）？

- tmux 怎么使用？

- ssh 是什么，如何使用 ssh？

### 新增深入问题
1. **进程间通信有哪些方式？**
   - 管道（Pipe）、命名管道（FIFO）、信号（Signal）、消息队列、共享内存、套接字（Socket）
   - 示例：`ls | grep .txt` 使用了匿名管道

2. **文件权限如何管理？**
   - 理解和会使用 `chmod` 命令
   - 权限表示：rwx（读/写/执行）和数字表示（755）


3. **环境变量有什么作用？**
   - `PATH`：决定命令搜索路径
   - `HOME`：用户主目录路径
   - 设置变量：`export VAR=value`，永久生效需写入 `~/.bashrc` 或 `~/.zshrc`

4. **软链接和硬链接有什么区别？**
   - 硬链接：指向文件 inode，删除原文件不影响链接
   - 软链接：指向文件路径，删除原文件链接失效

## C 语言基础

参考 《C primer Plus》，这部分学习在 linux 环境下复现书中的所述的内容以及完成部分你认为值得完成的作业（不一定做完，挑着做）。

完成学习后，应该能回答以下问题：

- 指针是什么？C 语言的指针能强转成任何类型的指针吗？为什么要这样设计？

- 联合体 union 和上面的指针强转有什么关系？

- extern 关键字怎么用？

- static 关键字怎么用？

- inline 关键字怎么用？

- 声明和定义有什么区别？

- 堆栈有什么区别？

- 等等...

### 新增深入问题
1. **函数指针有什么应用场景？**
   ```c
   int add(int a, int b) { return a + b; }
   int sub(int a, int b) { return a - b; }

   int main() {
       int (*operation)(int, int); // 函数指针声明
       operation = add;
       printf("5+3=%d\n", operation(5, 3)); // 8
       operation = sub;
       printf("5-3=%d\n", operation(5, 3)); // 2
       return 0;
   }
   ```

2. **预处理器有哪些高级用法？**
   - 条件编译：`#ifdef`、`#ifndef`、`#endif`
   - 宏定义：带参数的宏 `#define MAX(a,b) ((a)>(b)?(a):(b))`
   - 文件包含：`#include "header.h"` vs `#include <header.h>`

3. **内存对齐有什么作用？**
   - 提高内存访问效率
   - 避免平台相关的访问错误
   - 使用 `#pragma pack(n)` 控制对齐方式


## Makefile

Makefile 的学习参考 [这个](https://liaoxuefeng.com/books/makefile/introduction/index.html)。

学完后，应该能回答一下问题：

- 为什么要有 Makefile？

- 如何解决 .h 文件的依赖问题？-MM 参数是如何解决这一问题的？

- .PHONY 的作用是什么，在哪种情形中有用？

- Makefile 如何实现增量编译的？

### 新增深入问题
1. **Makefile 变量如何正确使用？**
   ```makefile
   CC = gcc
   CFLAGS = -Wall -O2
   TARGET = myapp
   OBJS = main.o utils.o

   $(TARGET): $(OBJS)
       $(CC) $(CFLAGS) -o $@ $^
   ```

2. **如何实现条件编译？**
   ```makefile
   DEBUG = 1

   ifeq ($(DEBUG),1)
       CFLAGS += -g
   else
       CFLAGS += -O3
   endif
   ```

3. **Makefile 函数有哪些实用案例？**
   ```makefile
   SOURCES = $(wildcard src/*.c)
   OBJS = $(patsubst src/%.c, build/%.o, $(SOURCES))

   build/%.o: src/%.c
       $(CC) $(CFLAGS) -c $< -o $@
   ```

4. **如何管理多目录项目？**
   - 使用 `VPATH` 或 `vpath` 指定源文件路径
   - 在子目录中编写 Makefile
   - 顶层 Makefile 调用子目录 Makefile

## Git & GitHub

参考 [这个教程](https://liaoxuefeng.com/books/git/introduction/index.html)。

### 核心问题
1. **Git 工作流程是怎样的？**
   - 工作区 → 暂存区（`git add`）→ 本地仓库（`git commit`）→ 远程仓库（`git push`）

2. **如何解决合并冲突？**

3. **分支管理策略有哪些？**
   - Git Flow：主分支（main）、开发分支（develop）、功能分支（feature）
   - GitHub Flow：基于 Pull Request 的轻量级工作流
   - GitLab Flow：环境分支策略

4. **如何回退错误提交？**
   ```bash
   # 撤销最后一次提交
   git reset --soft HEAD~1

   # 完全丢弃未提交的修改
   git reset --hard HEAD

   # 回退到特定提交
   git revert <commit-hash>
   ```

5. **.gitignore 文件如何使用？**
   - 忽略临时文件：`*.tmp`
   - 忽略目录：`build/`
   - 例外规则：`!important.tmp`

## 编译与链接

### 核心问题
1. **编译过程有哪些阶段？**
   - 预处理 → 编译 → 汇编 → 链接

2. **静态库和动态库有什么区别？**
   | 特性 | 静态库(.a) | 动态库(.so) |
   |------|------------|------------|
   | 链接方式 | 编译时链接 | 运行时链接 |
   | 文件大小 | 较大 | 较小 |
   | 更新 | 需重新编译 | 替换文件即可 |
   | 依赖 | 独立运行 | 需环境支持 |

3. **ELF 文件格式包含哪些部分？**
   - ELF 头：描述文件基本信息
   - 程序头表：描述段信息
   - 节头表：描述节信息
   - .text：代码段
   - .data：已初始化数据
   - .bss：未初始化数据

4. **符号解析如何工作？**
   - 链接器解析未定义符号
   - 在库文件中查找匹配符号
   - 处理重复符号定义

5. **什么是位置无关代码（PIC）？**
   - 动态库必须编译为位置无关代码，为什么？
   - 使用相对地址而非绝对地址
   - 编译选项：`gcc -fPIC -shared -o lib.so source.c`

### 新增深入问题
1. **如何优化编译过程？**
   - 使用预编译头文件
   - 并行编译：`make -j8`

2. **链接脚本有什么作用？**
   - 控制内存布局
   - 指定段地址
   - 嵌入式开发中关键作用

3. **如何实现热更新？**
   - 使用动态库加载机制，思考这是如何实现的？
   - `dlopen()` 和 `dlsym()` 函数
   - 保持接口兼容性

## 探索的问题

1. **glibc 和 系统调用有什么关系？**

2. **GNU C 和 C 语言标准有什么关系？GCC 起到了什么作用？**


