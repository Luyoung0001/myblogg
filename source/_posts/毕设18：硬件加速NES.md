---
title: LA 框架的使用
date: 2026-04-12 10:03:13
tags: LA
categories: loongarch32r
mermaid: true
---

## 准备
登录到服务器后，大概率是这样的一个界面：
```zsh
You can:

(q)  Quit and do nothing.  The function will be run again next time.

(0)  Exit, creating the file ~/.zshrc containing just a comment.
     That will prevent this function being run again.

(1)  Continue to the main menu.

(2)  Populate your ~/.zshrc with the configuration recommended
     by the system administrator and exit (you will need to edit
     the file by hand, if so desired).

--- Type one of the keys in parentheses ---
```

此时输入 `0` ，会创建一个默认的 zsh 用户配置文件 .zshrc。

## 克隆 LA 并运行 demo

在用户目录下输入：
```zsh
git clone https://gitee.com/luyoung0010/LA.git
```

然后回车，就会克隆 LA 框架。然后 cd 到 LA 目录，直接输入并回车：

```zsh
make all EXP=6
```
就进入了仿真，loongarch 工具链，包括 verilator 都已经安装好了。在使用之前，务必详细阅读整个项目，了解 LA 是在做什么，是怎么工作的，不懂的话可以问 AI。





