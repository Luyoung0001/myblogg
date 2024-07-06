---
title: macOS DOSBox 汇编环境搭建
date: 2024-05-05 22:35:14
tags: macos 汇编 DOSBox 8086
categories: 经验方法 8086
---

<!--more-->

## 正文
### 一、安装DOSBox
首先前往[DOSBox](https://www.dosbox.com/download.php?main=1)的官网下载并安装最新版本的DOSBox。

### 二、下载必备的工具包
在用户目录下新建一个文件夹，比如 **dosbox**:

```bash
mkdir dosbox
```
然后下载一些[常用的工具](https://cdn.oxdl.cn/files/%E5%BD%92%E6%A1%A3.zip)。下载好了后，将这些工具解压，重新放在 **dosbox** 这个文件夹中：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/1ebdb91472984c688955a71222bf1e51_1720253679644.png?token=ANB4BCMKEFJADVHJ3GSX533GRD6S6)
 ### 三、配置自动挂载
 
由于**DOSBox**启动之后所有的设置都会复原，因此要实现自动挂载需要配置自动运行命令。

打开**Terminal**，进入到 `~/Library/Preferences` 目录下，运行：

```bash
echo "mount c ~/dosbox\nC:" >> "DOSBox 0.74-3-3 Preferences"
```

其中`~/dosbox` 请替换为你刚才新建的文件夹目录，`0.74-3-3` 替换为你的**DOSBox**的版本。

运行一下**cat**命令可以看到这两行命令已经被加入到`[autoexec]`块内：

```bash
cat DOSBox\ 0.74-3-3\ Preferences

...
xms=true
ems=true
umb=true
keyboardlayout=auto

[ipx]
# ipx: Enable ipx over UDP/IP emulation.

ipx=false

[autoexec]
# Lines in this section will be run at startup.
# You can put your MOUNT lines here.


mount c ~/dosbox
C:
```
这个配置文件在启动 **DOSBox** 时会被加载并生效。其中的 `[autoexec]` 部分包含了一些命令，这些命令会在 **DOSBox** 启动时自动执行。这些命令通常用于配置 **DOSBox** 的一些初始设置，例如挂载游戏的磁盘镜像、设置运行环境等。

`[autoexec]` 部分包含了以下注释：

```
[autoexec]
# Lines in this section will be run at startup.
# You can put your MOUNT lines here.
```

这意味着任何在 `[autoexec]` 部分中的命令都会在 **DOSBox** 启动时执行。通常，用户会在这个部分中添加一些命令，用于设置 **DOSBox** 的环境，以便在启动时自动执行所需的操作：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/97dacac2ec2f4996933ecd190a2e6db2_1720253679644.png?token=ANB4BCKBOMIL2AMNU4KZGO3GRD6TA)
 这样，就搭建好了汇编环境。


