---
title: Ubuntu 升级 golang 版本完美步骤
date: 2023-06-29 18:35:56
tags:
    - ubuntu
    - golang
    - linux
categories: Some_Tools Golang Unix/Linux
---

<!--more-->

## 一、删除旧的版本（可选）

```bash
sudo rm -rf /usr/local/go
sudo apt-get remove golang
sudo apt-get remove golang-go
sudo apt-get autoremove
```
## 二、下载最新版本

```bash
#wget 后面的下载链接请去golang官网(https://golang.google.cn/dl/)获取你想下载的对应go版本
sudo wget https://golang.google.cn/dl/go1.20.3.linux-amd64.tar.gz
# 解压文件
sudo tar xfz go1.20.3.linux-amd64.tar.gz -C /usr/local
```

## 三、设置环境变量
1、打开profile:
```bash
sudo vim /etc/profile
```
2、添加以下变量：

```bash
export GOROOT=/usr/local/go
export GOPATH=$HOME/gowork
export GOBIN=$GOPATH/bin
export PATH=$GOPATH:$GOBIN:$GOROOT/bin:$PATH
```
3、是环境立即生效

```bash
 source /etc/profile
```
4、将环境立即生效载入脚本

先打开文件这个文件：
```bash
cd ~
sudo vim .bashrc
```
加入这个命令：

```bash
source /etc/profile
```
## 四、查看 go 环境变量

```bash
go env
```

## 五、GO111MOUDLE和更改GOPROXY

```bash
go env -w GOPROXY="https://goproxy.cn"
go env -w GO111MODULE=on
```

## 六、查看 go 版本

```bash
go version
```
看版本是不是你要的最新的。

*参考：[这里](https://zhuanlan.zhihu.com/p/453462046)*
