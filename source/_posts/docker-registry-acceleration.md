---
title: "Docker 镜像加速方案：代理、镜像源与配置取舍"
date: 2025-1-16 17:57:27
tags:
  - "Docker"
  - "registry mirror"
  - "tooling"
categories:
  - "Tooling"
---

<!--more-->

## 前言

docker 在 pull 镜像的时候，速度要么很慢，要么直接卡住报错，这是因为网络不通的原因。主要有两个思路，方案一就是换源，将 docker 的仓库换到镜像源上，因为镜像源在国内，因此这种方式便宜。但缺点是镜像可能会在某个时间节点停止服务，不够稳定。方案二就是不用管源的事情，直接在本地架设代理，缺点是技术门槛高，需要架设代理，还需要支付额外的流量费。优点是源不会挂掉，很稳定。

## 方案一

创建或修改 /etc/docker/daemon.json 文件，修改：

```bash
cd /etc/docker
vim daemon.json
```

加入如下配置：

```bash
{
    "registry-mirrors" : [
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://cr.console.aliyun.com",
    "https://mirror.ccs.tencentyun.com"
  ]
}
```

重启docker服务使配置生效：

```bash
systemctl daemon-reload
systemctl restart docker.service
```
这个方案可能还有一些问题，使得不能正常工作。

## 方案二
编辑 /etc/systemd/system/docker.service.d/http-proxy.conf 文件（如果文件不存在，则创建路径以及文件）：

```bash
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
```

添加或修改以下内容，将 替换为你的代理地址：

```bash
[Service]
Environment="HTTP_PROXY=http://<proxy_url>:<port>"
Environment="HTTPS_PROXY=https://<proxy_url>:<port>"
```

修改完成后，运行以下命令以重启 Docker 服务并应用配置：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Docker 客户端在拉取镜像时，会按照以下顺序尝试：

- 检查本地是否存在镜像： 如果本地存在所需镜像，则直接使用本地镜像。
- 检查配置的镜像加速器： 如果配置了镜像加速器，则依次尝试从加速器拉取镜像。
- 连接 Docker Hub： 如果所有加速器都无法拉取镜像，则尝试直接连接 Docker Hub。