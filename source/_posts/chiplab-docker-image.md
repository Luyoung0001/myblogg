---
title: "ChipLab 环境打包：制作 Docker 镜像"
date: 2025-08-06 13:03:13
tags:
  - "ChipLab"
  - "Docker"
  - "tooling"
categories:
  - "Tooling"
---

## 前言
由于 chiplab 在配置的时候需要将工具一一准备好，还有某些库比如随机测试，还需要自己下载。比较麻烦，于是我将一个配置好的环境打包成 docker 镜像，这样可以很方便得就能立即使用。

## docker 镜像制作

首先在 chiplab 中写一个简单的 Dockerfile:

```Dockerfile
FROM ubuntu:22.04

# 避免交互式配置界面
ENV DEBIAN_FRONTEND=noninteractive

# 安装常见工具 + zsh
RUN apt-get update && apt-get install -y \
    git \
    curl \
    wget \
    vim \
    zsh \
    build-essential \
    sudo \
    ca-certificates \
    locales \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 设置默认 locale 避免终端乱码
RUN locale-gen en_US.UTF-8
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV LC_ALL=en_US.UTF-8

# 设置默认 shell 为 zsh
SHELL ["/bin/zsh", "-c"]
CMD ["/bin/zsh"]

# 将当前目录（chiplab）复制进容器的 /workspace 目录
WORKDIR /workspace
COPY . /workspace
```

然后再 chiplab 中使用命令：`docker build -t my-dev-env .`，然后就可以基于这个镜像运行它了。

为了方便使用 vscode 进行远程开发，可以直接给它开一个端口 22 映射到 host 的 2222 端口上：

```bash
docker run -it --name chiplab_test -p 2222:22 chiplab_remote_ssh /bin/zsh

root at 86072f8b45a9 in /
$ cd ~
root at 86072f8b45a9 in ~
$ ls
chiplab  open-la500  verilator
```

并给它安装好了 verilator 等工具。但是发现它的镜像高达 20 个 GB，可以通过以下命令来缩减：
```bash
docker export -o img.tar 容器ID
docker  import  img.tar   img1:tag
```

之后缩减到了 8 个 GB。然后通过 docker 在本地将镜像上传到 Docker Hub。这里可能由于网络问题总会失败，可以通过配置代理：

## 本地代理

### 方法 1：配置 Docker 用 socks5 代理
编辑 Docker 的 systemd 服务配置：
```bash
Copy
Edit
sudo mkdir -p /etc/systemd/system/docker.service.d

sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
```
写入：
```bash
[Service]
Environment="HTTP_PROXY=socks5h://127.0.0.1:7890"
Environment="HTTPS_PROXY=socks5h://127.0.0.1:7890"

```
然后重启 Docker（假设你的代理运行在本地 127.0.0.1:7890）：

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 打包镜像
Docker Hub 镜像命名格式为：
```bash
<你的 Docker Hub 用户名>/<仓库名>:<tag>
```
必须手动 tag 一下，比如：
```bash
docker tag chiplab_remote_ssh luyoung0001/chiplab_remote_ssh:latest
```
接着登录 docker hub：

```bash
docker login -u luyoung0001

i Info → A Personal Access Token (PAT) can be used instead.
         To create a PAT, visit https://app.docker.com/settings


Password:
Login Succeeded
```

之后就可以 push 上去了：

```bash
docker push luyoung0001/chiplab_remote_ssh:latest
```

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/docker_hub.png)


## 使用方法
运行以下命令直接开箱即用：

```bash
docker pull luyoung0001/chiplab_remote_ssh

docker run -d \
  --name chiplab_env_ssh \
  -p 2222:22 \
  -e TZ=Asia/Shanghai \
  -v ~/.ssh:/root/.ssh \
  luyoung0001/chiplab_remote_ssh:latest \
  /usr/sbin/sshd -D
```

然后就能直接通过 ssh 连接，密码是 123456：
```bash
ssh root@localhost -p 2222
root@localhost's password:
123456
root at c7c2c477e449 in ~
$
```

我已经配置好了 zsh 以及 zsh 主题。

## vscode 连接

假设你的ip 地址为 192.168.10.101，直接编辑 ~/.ssh/config：
```config
Host docker_chiplab
    HostName 192.168.10.101
    User root
    Port 2222
```

然后就能通过vscode 的 remote_ssh 就能连接了：
![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/chiplab_ssh_remote.png)


里面默认使用的是 OpenLA500，运行 func 测试：

```bash
[NEMU] This is syscall 0x11, end
[src/cpu/cpu-exec.c,338,cpu_exec] nemu: HIT GOOD TRAP at pc = 0x1c000230
[src/cpu/cpu-exec.c,78,monitor_statistic] host time spent = 19,815 us
[src/cpu/cpu-exec.c,80,monitor_statistic] total guest instructions = 174,097
[src/cpu/cpu-exec.c,81,monitor_statistic] simulation frequency = 8,786,121 instr/s
END by Syscall
total clock is 609660

==============================================================
total clock             is 609660
total instruction       is 174059
instruction per cycle   is 0.285502
simulation time         is 9.385514 s
difftest time           is 2.853328 s
nemu_step time          is 0.030525 s
verilator eval time     is 5.308443 s
==============================================================

Terminated at 1219334 ns.
Test exit.
Reached test end PC.
[FORK_INFO pid(1162)] clear processes...
total time is 9493128 us
make[2]: Leaving directory '/root/chiplab/sims/verilator/run_prog/tmp'
mv: cannot stat './tmp/logs/*.fst': No such file or directory
make[1]: Leaving directory '/root/chiplab/sims/verilator/run_prog'
```

## 总结

通过将配置好的 chiplab 开发环境打包成 Docker 镜像，不仅极大减少了环境部署的复杂度，也方便了团队协作和迁移开发环境。结合 VSCode 的 Remote-SSH 插件，几乎可以实现本地开发体验，同时保留容器环境的一致性和可重复性。

本次构建的镜像具备以下优点：

* 内置了 chiplab、verilator 等开发所需工具
* 支持 SSH 远程连接，开箱即用
* 已配置 zsh + 主题，提供良好终端体验
* 镜像已上传至 Docker Hub，任意机器均可快速部署
* 支持 VSCode Remote-SSH 一键接入开发

如果你也需要一个干净、统一、可复用的环境，不妨试试这个镜像：

```bash
docker pull luyoung0001/chiplab_remote_ssh
```
