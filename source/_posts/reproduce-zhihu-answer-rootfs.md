---
title: "复现一篇知乎答案：构建 Ubuntu 文件系统实验"
date: 2024-03-31 17:20:23
tags:
  - "rootfs"
  - "Ubuntu"
  - "QEMU"
categories:
  - "Operating Systems"
---

<!--more-->

## 〇、前言
早上在逛知乎的时候，瞥见了一篇答案：[如何通俗解释Docker是什么？](https://www.zhihu.com/question/28300645/answer/2488146755)感觉很不错，然后就耐着性子看了下，并重现了作者的整个过程。但是并不顺利，记载一下这些坑。嫌麻烦的话可以直接clone 研究，[git仓库](https://github.com/Luyoung0001/mocker/tree/main)。

## 一、构建 ubuntu 文件系统
具体可以看这篇文章：**[Ubuntu Base构建根文件系统](https://doc.embedfire.com/lubancat/build_and_deploy/zh/latest/building_image/ubuntu_rootfs/ubuntu_rootfs.html#ubuntu-base)**。

主要步骤就是：
### 下载镜像
- ubuntu-base-20.04.1-base-armhf.tar.gz
### 安装依赖

```bash
sudo apt-get install tar qemu-user-static vim -y
```

### 配置根文件系统
这是必要的，因为 `execvp()`执行的时候（作者给的例子），它会在这个文件系统中查找你要运行的命令。

```bash
mkdir ubuntu_rootfs && cd ubuntu_rootfs
tar -vxf ubuntu-base-20.04.1-base-armhf.tar.gz
sudo cp /usr/bin/qemu-arm-static ./usr/bin/
sudo cp ./etc/apt/sources.list ./etc/apt/sources.list.back
sudo echo "nameserver 8.8.8.8"  > ./etc/resolv.conf
sudo vim ./etc/apt/sources.list
```
### 更新源
如果你不需要在这个文件系统中做很多的事，就不需要。

### 添加应用

如果你不需要在这个文件系统中做很多的事，就不需要。

```bash
sudo chroot ./
apt install vim sudo kmod net-tools ethtool ifupdown language-pack-en-base rsyslog htop iputils-ping -y //添加一些需要的应用
passwd root //设置root的密码
exit
```
## 二、mocker

```cpp
#include <cstring>
#include <iostream>
#include <sys/wait.h>
#include <unistd.h>

#include <fcntl.h>
#include <sys/mount.h>
#include <sys/stat.h>
#include <sys/types.h>

static void run(int argc, char *argv[]);
static std::string cmd(int argc, char *argv[]);
static void run_child(int argc, char *argv[]);

int main(int argc, char *argv[]) {

    if (argc < 3) {
        std::cerr << "Too few arguments" << std::endl;
        exit(-1);
    }
    if (!strcmp(argv[1], "run")) {
        run(argc - 2, &argv[2]);
    }
    return 0;
}
static void run(int argc, char *argv[]) {
    // 合成命令
    std::cout << "parent running " << cmd(argc, argv) << " as " << getpid()
              << std::endl;

    if (unshare(CLONE_NEWPID) < 0) {
        std::cerr << "failed to unshare in child: PID" << std::endl;
        exit(-1);
    }

    // 用子进程运行命令,同时父进程保持

    pid_t child_pid = fork();
    if (child_pid < 0) {
        std::cerr << "failed to fork" << std::endl;
        return;
    }
    if (child_pid)

    {
        if (waitpid(child_pid, NULL, 0) < 0) {
            std::cerr << "failed to wait for child" << std::endl;
        } else {
            std::cout << "child terminited" << std::endl;
        }
    } else {
        run_child(argc, argv);
    }
}

const char *child_hostname = "container";

static void run_child(int argc, char *argv[]) {
    std::cout << "child running " << cmd(argc, argv) << " as " << getpid()
              << std::endl;
    int flags = CLONE_NEWUTS | CLONE_NEWNS;
    if (unshare(flags) < 0) {
        std::cerr << "failed to share in child: UTS" << std::endl;
        exit(-1);
    }
    if (mount(NULL, "/", NULL, MS_SLAVE | MS_REC, NULL) < 0) {
        std::cerr << "failed to mount /" << std::endl;
    }

    // 修改 filesystem 的 view,在制定目录下
    if (chroot("./ubuntu_rootfs") < 0) {
        std::cerr << "failed to chroot" << std::endl;
        exit(-1);
    }
    if (chdir("/") < 0) {
        std::cerr << "failed to chdir to /" << std::endl;
        exit(-1);
    }

    // 挂载
    if (mount("proc", "proc", "proc", 0, NULL) < 0) {
        std::cerr << "failed to mount /proc" << std::endl;
    }

    // 修改 host name
    if (sethostname(child_hostname, strlen(child_hostname)) < 0) {
        std::cerr << "failed to changge hostname" << std::endl;
    }

    if (execvp(argv[0], argv)) {
        perror("execvp failed");
    }
}

static std::string cmd(int argc, char *argv[]) {
    std::string cmd = "";
    for (int i = 0; i < argc; i++) {
        cmd.append(argv[i] + std::string(" "));
    }
    return cmd;
}
```


总结：这段代码实现了一个简单的容器化环境的初始化过程，包括命名空间的隔离、文件系统的挂载与切换、进程环境的修改等操作，从而创建了一个隔离的运行环境。




