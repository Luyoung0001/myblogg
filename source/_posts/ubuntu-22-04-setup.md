---
title: "Ubuntu 22.04 开发环境配置：SSH、Git 与常用工具"
date: 2024-09-10 14:20:21
tags:
  - "Ubuntu"
  - "development environment"
  - "SSH"
categories:
  - "Tooling"
---

<!--more-->

## 前言
这里记录一下配置Ubuntu22.04的过程，主要实现了：

- ssh 远程免密访问
- 配置 git & github
- 内网穿透远程 ssh 访问
- v2ray、v2raya(虽然最后采用了其它的网络加速方案，但也是可行的)
- zsh 美化
- 远程桌面访问（目前只能在同一局域网下，内网穿透失效）
- 换软件源
- 开机启动脚本
- hexo & npm

## ssh 远程免密访问

因为这台电脑的角色是一台服务器，但是我安装的是桌面版，主要考虑到的可能会偶尔用到桌面，比如做仿真实验的时候，虽然 99% 的时间都不到桌面。ssh 远程访问就很重要。
在 server 上 安装 openssh-server：
```bash
sudo apt-get install openssh-server
```

之后就可以登录了，详见：[buntu开启SSH服务远程登录](https://blog.csdn.net/jackghq/article/details/54974141)。

首先在 local 机器上生成密钥对：
```bash
ssh-keygen -t rsa -C "xxx@xxx.com"
//执行后一直回车即可
```
然后将秘钥传送到 server:
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub username@ip地址
```

详见：[SSH 三步解决免密登录](https://blog.csdn.net/jeikerxiao/article/details/84105529)

## 配置 git、github

初始化：
```bash
$ git config --global user.name "John Doe"
$ git config --global user.email johndoe@example.com
```

在 server 上生成密钥对：

```bash
ssh-keygen -t rsa -C "xxx@xxx.com"
//执行后一直回车即可
```

将 publickey 复制粘贴到 github ssh keys，具体参考：[Github配置ssh key的步骤](https://blog.csdn.net/weixin_42310154/article/details/118340458)。

## 内网穿透远程 ssh 访问

这个需要注册一下，还需要实名认证（很简单，几分钟就搞定了）。

详见：[内网穿透实现ssh远程连接Ubuntu（Sakura frp实现方法)](https://blog.csdn.net/LJH021028/article/details/129307892)。

这里需要注意的是，我们最后需要执行的是：
```bash
nohup frpc -f <复制的密钥> &
```

这个命令的意思是放在后台的进程忽略 HUP信号，可以在 shell 进程结束时，它也不会终止。这个进程应该开机自动运行，可以尝试把它放到某个开机自动执行的脚本中。


## v2ray、v2raya

添加公钥，验证软件安全性：
```bash
wget -qO - https://apt.v2raya.org/key/public-key.asc | sudo tee /etc/apt/keyrings/v2raya.asc
```
添加软件源、更新 list：

```bash
echo "deb [signed-by=/etc/apt/keyrings/v2raya.asc] https://apt.v2raya.org/ v2raya main" | sudo tee /etc/apt/sources.list.d/v2raya.list
sudo apt update

```
安装核心和基于核心的软件：
```bash
sudo apt install v2raya v2ray
```

然后配置就行了。

详见：[用户文档](https://v2raya.org/docs/prologue/installation/debian/)。

也可以使用同一个局域网下的代理模式，这样加速更好一些。

## zsh 美化

简单至极：直接看[Zsh 安装与配置，使用 Oh-My-Zsh 美化终端](https://www.haoyep.com/posts/zsh-config-oh-my-zsh/)。

这里需要阅读上面的 proxy() 那里，对于 git、curl、wget 等服务的加速效果非常好。


## 换软件源

换 alibaba 的源，其它的源现在很慢。直接在图形化界面进行操作。

## 远程桌面访问

这个没啥意思，挺无聊的，没啥用。

## 开机启动脚本

由于内网穿透等工具需要在开机的自动执行，因此需要将我们的某些操作加在开机自动启动的脚本中。

这里以 fprc 服务为例，来创建 systemd 控制的程序。

对于包括 `nohup` 和 `&` 的命令，可以稍微调整 `systemd` 服务文件的写法，因为 `systemd` 默认就是在后台运行服务，并且会处理程序的守护化，使之不受终端关闭的影响。因此，通常不需要使用 `nohup` 和 `&`。

如果你的命令是 `nohup frpc XXXX &`，在 `systemd` 服务文件中，你只需要直接指定 `ExecStart` 即可，如下所示：

#### 修改后的 `systemd` 服务文件

1. **打开或创建服务文件**：
   ```bash
   sudo vim /etc/systemd/system/frpc.service
   ```

2. **服务文件内容**：
   ```ini
   [Unit]
   Description=Frpc Client Service
   After=network.target

   [Service]
   Type=simple
   ExecStart=/usr/local/bin/frpc XXXX
   Restart=always
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

   - 这里 `Type=simple` 默认适用，`systemd` 认为该服务在启动后将持续运行。
   - `Restart=always` 和 `RestartSec=5` 选项意味着如果服务失败，`systemd` 将在 5 秒后尝试重启服务。

3. **启用并启动服务**：
   ```bash
   sudo systemctl enable frpc.service
   sudo systemctl start frpc.service
   ```

4. **检查服务状态**：
   ```bash
   sudo systemctl status frpc.service
   ```

通过这种方式，`systemd` 将负责启动和守护 `frpc` 进程，不需要 `nohup` 或 `&`。这也使得服务管理（如启动、停止、重启和查看状态）更为一致和标准化。

直接 reboot，就可以了。

## npm
npm 由于某些问题下载很慢，这里可以这样做(一般安装软件都需要 sudo 权限，因为要写入系统目录)：

下载 cnpm:
```bash
sudo npm install -g cnpm --registry=https://registry.npmmirror.com

```

```bash
sudo cnpm install [name]
```

然后利用 cnpm 来安装 hexo:
```bash
sudo cnpm install -g hexo-cli
```







