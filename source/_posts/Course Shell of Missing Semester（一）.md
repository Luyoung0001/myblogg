---
title: Course Shell of Missing Semester（一）
date: 2023-12-22 21:42:32
tags: Shell 学习 笔记
categories: Missing Semester
---

<!--more-->

## 〇、前言
本文是 **The Missing Semester of Your CS Education** 课程的课后题答案，课程网站点击[这里](https://missing.csail.mit.edu/)，以后系列文章不再描述**前言**。

本文实验环境：**阿里云Ubuntu 22.04**

## Course shell
### 1、本课程需要使用类Unix shell，例如 Bash 或 ZSH。如果您在 Linux 或者 MacOS 上面完成本课程的练习，则不需要做任何特殊的操作。如果您使用的是 Windows，则您不应该使用 cmd 或是 Powershell；您可以使用Windows Subsystem for Linux或者是 Linux 虚拟机。使用echo $SHELL命令可以查看您的 shell 是否满足要求。如果打印结果为/bin/bash或/usr/bin/zsh则是可以的

```bash
echo $SHELL
/bin/bash
```

### 2、在 /tmp 下新建一个名为 missing 的文件夹

```bash
mkdir /tmp/missing
```

### 3、用 man 查看程序 touch 的使用手册

```bash
man touch
```

### 4、用 touch 在 missing 文件夹中新建一个叫 semester 的文件

```bash
touch /tmp/missing/semester
```

### 5、将以下内容一行一行地写入 semester 文件：

```powershell
 #!/bin/sh
 curl --head --silent https://missing.csail.mit.edu
```
直接运行：

```bash
echo '#!/bin/sh
 curl --head --silent https://missing.csail.mit.edu' > /tmp/missing/semester
```
### 6、尝试执行这个文件。例如，将该脚本的路径（./semester）输入到您的shell中并回车。如果程序无法执行，请使用 ls 命令来获取信息并理解其不能执行的原因

```bash
./semester
bash: ./semester: No such file or directory
```
这肯定不能运行的，因为当前目录是 **home** `~`，应该首先切换到目标文件得目录或者输入完整目录：


```bash
/tmp/missing/semester
bash: /tmp/missing/semester: Permission denied
```
没有权限，这很正常。

### 7、查看 chmod 的手册(例如，使用 man chmod 命令)
### 8、使用 chmod 命令改变权限，使 ./semester 能够成功执行，不要使用 sh semester 来执行该程序。您的 shell 是如何知晓这个文件需要使用 sh 来解析呢？
加执行权限执行：
```bash
chmod +x /tmp/missing/semester

/tmp/missing/semester

HTTP/2 200
server: GitHub.com
content-type: text/html; charset=utf-8
last-modified: Wed, 29 Nov 2023 09:35:41 GMT
access-control-allow-origin: *
etag: "656705ed-2015"
...
```

执行成功。

这个脚本：

```powershell
#!/bin/sh
 curl --head --silent https://missing.csail.mit.edu
```
第一行表明脚本解释器为：`/bin/sh`，看看 `bin`：

```bash
ls /bin | grep sh
sh
sha1sum
sha224sum
sha256sum
...
```
其中，**sh** 就是指定的脚本解释器。

这里需要注意的是，有多种方式执行这个脚本，但是归根结底分为两类：

```bash
. /tmp/missing/semester
/tmp/missing/semester
```
前一种，指明在当前 shell 中执行，后一种会打开一个子 shell，然后在子 shell 中执行这个脚本。这意味着，如果我们在当前 shell 中执行脚本，无需用 **shebang**：

```bash
touch /tmp/missing/semester1
echo 'curl --head --silent https://missing.csail.mit.edu' > /tmp/missing/semester1
. /tmp/missing/semester1
HTTP/2 200
server: GitHub.com
content-type: text/html; charset=utf-8
last-modified: Wed, 29 Nov 2023 09:35:41 GMT
access-control-allow-origin: *
etag: "656705ed-2015"
expires: Fri, 22 Dec 2023 13:19:52 GMT
cache-control: max-age=600
...
```
它为什么能运行成功？它没有执行权限：

```bash
ls -l /tmp/missing/
total 8
-rwxr-xr-x 1 root root 62 Dec 22 21:04 semester
-rw-r--r-- 1 root root 53 Dec 22 21:18 semester1
```

如果使用 `.` 命令（或 `source` 命令）来执行脚本，即使脚本没有执行权限，也可以成功执行。

这是因为 `.` 命令是在当前shell中执行脚本，而不是通过创建子shell来执行。当使用 `.` 命令执行脚本时，脚本中的命令将直接在当前shell中运行，而不需要执行权限。

相比之下，如果您直接运行脚本（例如 `./semester1`），则需要确保脚本具有执行权限，否则会出现权限错误。因此，如果您希望在没有执行权限的情况下执行脚本，可以使用 `.` 命令或 `source` 命令来运行它。**换句话说，当用 shebang 指定解释器后，我们希望这个脚本像一个命令那样自然得运行；不用 shebang 指定解释器时，它完全就是一段文本，当然不需要什么执行权限了**。当然，本质依然是，子**shell** 执行需要执行权限。

### 9、使用 | 和 > ，将 semester 文件输出的最后更改日期信息，写入主目录下的 last-modified.txt 的文件中


获取最后修改时间：

```bash
stat -c %y /tmp/missing/semester
2023-12-22 21:04:37.621639025 +0800
```
用重定向符追加到 `/tmp/missing/last-modified.txt`：

```bash
stat -c %y /tmp/missing/semester >> /tmp/missing/last-modified.txt
```

确实增加进去了：

```bash
cat /tmp/missing/last-modified.txt
2023-12-22 21:04:37.621639025 +0800
```


## 总结（大模型总结）
这个实验是一个很好的 **Unix shell** 基础练习，涉及文件操作、权限管理和 **shell**脚本的执行。主要涵盖了以下内容：

1. 创建文件夹及文件，查看文件的使用手册，修改文件权限以及文件执行。
2. 使用**chmod**命令改变文件权限，使其具有执行权限。
3. 使用**shebang**指定脚本解释器，理解不同方式执行脚本的区别。
4. 了解子**shell**执行脚本时的权限要求，以及如何使用`.`命令或`source`命令执行没有执行权限的脚本。
5. 使用`|`和`>`操作符将文件输出结果追加到另一个文件中，实现信息提取和写入文件。

这些操作涵盖了 **Unix shell** 常见的文件操作和权限管理知识点，对于熟悉 **Unix/Linux** 系统的基本操作和 **shell** 脚本的编写都有很好的帮助。



