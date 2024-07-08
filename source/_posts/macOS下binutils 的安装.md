---
title: macOS下binutils 的安装
date: 2023-01-10 15:14:04
tags:
    - macos
    - linux
    - 服务器
categories: Some_Tools
---

<!--more-->

## 安装
输入：
```bash
brew update && install binutils
```
之后弹出：
```bash
binutils is keg-only, which means it was not symlinked into /usr/local,
because because Apple provides the same tools and binutils is poorly supported on macOS.

If you need to have binutils first in your PATH run:
  echo 'export PATH="/usr/local/opt/binutils/bin:$PATH"' >> ~/.zshrc

For compilers to find binutils you may need to set:
  export LDFLAGS="-L/usr/local/opt/binutils/lib"
  export CPPFLAGS="-I/usr/local/opt/binutils/include"
```

这里已经给了很详细的提示了，我们只需要键入：

```bash
echo 'export PATH="/usr/local/opt/binutils/bin:$PATH"' >> ~/.zshrc
```
然后执行：

```bash
source ~/.zshrc
```
这个命令的作用是：
- 刷新当前的shell环境
- 在当前环境使用source执行Shell脚本
- 从脚本中导入环境中一个Shell函数
- 从另一个Shell脚本中读取变量

换句话说，可以让环境变量立即生效。然后依次执行：
```bash
export LDFLAGS="-L/usr/local/opt/binutils/lib"
```

```bash
export CPPFLAGS="-I/usr/local/opt/binutils/include"
```
之后，就可以使用这个工具包中的命令了，比如：

```bash
gobjdump -v
```
弹出：

```bash
GNU objdump (GNU Binutils) 2.39
Copyright (C) 2022 Free Software Foundation, Inc.
This program is free software; you may redistribute it under the terms of
the GNU General Public License version 3 or (at your option) any later version.
This program has absolutely no warranty.
```
这样就说明软件包安装好了。

