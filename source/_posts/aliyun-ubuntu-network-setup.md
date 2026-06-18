---
title: "阿里云 Ubuntu 网络配置：代理环境与连通性排查"
date: 2023-06-26 22:03:58
tags:
  - "Aliyun"
  - "Ubuntu"
  - "networking"
categories:
  - "Networking"
---

<!--more-->

## 〇、问题描述
在云服务器 Ubuntu 中运行 go 的时候，某些库在加载的时候速度太慢，甚至被限制了以至于没有速度。网上的教程或多或少的有难以捉摸的问题，本文旨在提供一个可靠的方案。
## 一、安装v2ray-core 核心代码

```bash
curl -O https://cdn.jsdelivr.net/gh/v2rayA/v2rayA@master/install/go.sh
sudo bash go.sh
```

按道理，上面两条命令就可以按照脚本 `go.sh`自动化执行，但是由于某些原因，这个脚本是下载不下来的。只需要一些技巧即可解决：

现在本地浏览器中输入上述地址：`https://cdn.jsdelivr.net/gh/v2rayA/v2rayA@master/install/go.sh`，就会在本地下载一个 go.sh 的文件，打开后，里面长这样：

```bash
#!/bin/bash

echo -e "\e[37;44;4;1mThe old script is abandoned, we will redirect you to the new one in 5 seconds.\e[0m"
echo -e "\e[37;44;4;1mIf you want to download the new script and run it manually,
press Ctrl+C and visit https://v2raya.org/en/docs/prologue/installation/\e[0m"

sleep 5s
curl -Ls https://mirrors.v2raya.org/go.sh | sudo bash

```
可以看到这个脚本里面有两条命令，第二条命令依然是去下载一个bash 脚本，然后用 sudo 权限去执行。
继续在浏览器中打开这个地址，这个脚本长这样：

```bash
#!/usr/bin/env bash
# shellcheck disable=SC2268

# The files installed by the script conform to the Filesystem Hierarchy Standard:
# https://wiki.linuxfoundation.org/lsb/fhs

# The URL of the script project is:
# https://github.com/v2fly/fhs-install-v2ray

# The URL of the script is:
# https://hubmirror.v2raya.org/raw/v2fly/fhs-install-v2ray/master/install-release.sh

# If the script executes incorrectly, go to:
# https://github.com/v2fly/fhs-install-v2ray/issues

# You can set this variable whatever you want in shell session right before running this script by issuing:
# export DAT_PATH='/usr/local/share/v2ray'
DAT_PATH=${DAT_PATH:-/usr/local/share/v2ray}

# You can set this variable whatever you want in shell session right before running this script by issuing:
# export JSON_PATH='/usr/local/etc/v2ray'
JSON_PATH=${JSON_PATH:-/usr/local/etc/v2ray}

# Set this variable only if you are starting v2ray with multiple configuration files:
# export JSONS_PATH='/usr/local/etc/v2ray'

# Set this variable only if you want this script to check all the systemd unit file:
# export check_all_service_files='yes'

curl() {
  $(type -P curl) -L -q --retry 5 --retry-delay 10 --retry-max-time 60 "$@"
}

systemd_cat_config() {
  if systemd-analyze --help | grep -qw 'cat-config'; then
    systemd-analyze --no-pager cat-config "$@"
    echo
  else
    echo "${aoi}~~~~~~~~~~~~~~~~"
    cat "$@" "$1".d/*
    echo "${aoi}~~~~~~~~~~~~~~~~"
    echo "${red}warning: ${green}The systemd version on the current operating system is too low."
    echo "${red}warning: ${green}Please consider to upgrade the systemd or the operating system.${reset}"
    echo
  fi
}
......
```
之后就简单了，先复制所有脚本。然后再云 Ubuntu 上新建一个名为 go.sh 的文件：

```bash
vim go.sh
```
打开时候，就可以用 sudo bash 执行这个文件了：

```bash
sudo bash go.sh
```
之后就进入了脚本执行环节（就是一些下载和执行操作）：

```bash
root@**************:~# sudo bash go.sh
info: Installing V2Ray v5.7.0 for x86_64
Downloading V2Ray archive: https://github.com/v2fly/v2ray-core/releases/download/v5.7.0/v2ray-linux-64.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 11.4M  100 11.4M    0     0  18422      0  0:10:49  0:10:49 --:--:-- 11091
Downloading verification file for V2Ray archive: https://github.com/v2fly/v2ray-core/releases/download/v5.7.0/v2ray-linux-64.zip.dgst
Reading package lists... Done
Building dependency tree
Reading state information... Done
......

```
这样就完成了下载，对了再加两条命令，以完成核心服务自启动和现在启动：

```bash
systemctl start v2ray
systemctl enable v2ray
```

## 二、安装v2raya客户端
### （一）方法一（可能会失败）
### 1、添加公钥

```bash
wget -qO - https://apt.v2raya.org/key/public-key.asc | sudo tee /etc/apt/trusted.gpg.d/v2raya.asc
```
### 2、添加软件源

```bash
echo "deb https://apt.v2raya.org/ v2raya main" | sudo tee /etc/apt/sources.list.d/v2raya.list
sudo apt update
```

### 3、安装 v2raya

```bash
sudo apt install v2raya
```
### （二）手动安装 deb 包（100%会成功）

```bash
sudo apt install /path/download/installer_debian_xxx_vxxx.deb
```
自行替换 [deb](https://github.com/v2rayA/v2rayA/releases) 包所在的实际路径。这里的方法是，可以在本地将这个包下载下来，然后通过 ssh 上传到服务器。

## 三、运行服务

```bash
sudo systemctl start v2raya.service
sudo systemctl enable v2raya.service

```
之后就可以登陆 `127.0.0.1:2017` 进行配置了。但是由于云服务器 ubuntu 没有可视化界面和浏览器，因此我们可以通过端口映射， 从本地映射到`127.0.0.1:2017`，然后打开本地浏览器，输入地址即可配置。

```bash
ssh username@remote_address -L 127.0.0.1:8888:127.0.0.1:2017
```
其中8888是本地端口号，2017是服务器端端口号（也就是 v2ray服务器端口号），可根据实际情况进行调整。*如果连接不上，就使用 sudo 权限*。

然后进行配置就好了~

*注：本文章仅仅供个人参考*



