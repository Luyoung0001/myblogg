---
title: "Hexo 博客搭建：从初始化到发布流程"
date: 2024-07-06 21:57:40
tags:
  - "Hexo"
  - "blog"
  - "GitHub Pages"
categories:
  - "Tooling"
---

<!--more-->

## 〇、前言
本文将会讨论，如何将 CSDN 上的博客，拉取到本地，然后PicGo、Hexo、Github 等工具建立个人博客，环境为 Ubuntu 20.04。

## 一、利用 Hexo

### 预备工作
首先安装 Node.js、npm、git工具。

```bash
> node -v
v12.22.9
> npm -v
8.5.1
> git version
git version 2.34.1
```

### 安装 Hexo
```bash
npm install -g hexo-cli

> hexo -v
INFO  Validating config
hexo: 7.3.0
hexo-cli: 4.3.2
os: linux 5.15.0-107-generic Ubuntu 22.04.4 LTS 22.04.4 LTS (Jammy Jellyfish)
node: 12.22.9
v8: 7.8.279.23-node.56
uv: 1.43.0
zlib: 1.2.11
brotli: 1.0.9
ares: 1.18.1
modules: 72
nghttp2: 1.43.0
napi: 8
llhttp: 2.1.6
http_parser: 2.9.4
openssl: 1.1.1m
cldr: 40.0
icu: 70.1
tz: 2024a
unicode: 14.0
```

### 小试牛刀
首先创建一个文件夹，比如 hexotest。这个文件的目的是，我们打算在它里面创建博客，并且它是一个博客网站项目根目录：

```bash
mkdir hexotest
cd hexotest
```

之后，就可以初始化这个文件夹了，初始化的目的是，它会创建一个 helloword.md 文件，然后利用 node.js 等将这个 markdown 文件渲染成一个 html 文件，然后在本地开启一个网络服务器以供前端访问：
```bash
> hexo init
INFO  Cloning hexo-starter https://github.com/hexojs/hexo-starter.git
INFO  Install dependencies
INFO  Start blogging with Hexo!
```

接着启动：
```bash
> hexo s
INFO  Validating config
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/ . Press Ctrl+C to stop.
```

然后就可以在前端看到博客了：
![博客](https://raw.githubusercontent.com/Luyoung0001/picBed/main/hexotest.png))

## 二、利用 Hexo 将博客部署在 github

这里创建 github 账户、创建博客仓库什么的就不啰嗦了，只提重点。

### 修改配置文件
进入文件夹 hexotest，之后修改配置文件 _config.yml:
```bash
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repo: git@github.com:Luyoung0001/Luyoung0001.github.io.git
  branch: main
```

之后，需要一个小工具将博客部署上去， 原理主要就是 push到远端仓库：
```bash
npm install hexo-deployer-git --save
```

需要注意的是，每一个项目根目录都需要执行一次这个命令，这个命令会在本地生成一些脚本，然后把脚本放在 node_modules 目录中。

之后执行下面的命令就可以了：
```bash
hexo c   #清除缓存文件 db.json 和已生成的静态文件 public
hexo g       #生成网站静态文件到默认设置的 public 文件夹(hexo generate 的缩写)
hexo d       #自动生成网站静态文件，并部署到设定的仓库(hexo deploy 的缩写)
```

### 访问
通过浏览器访问：https://luyoung0001.github.io 就可以了：
![博客](https://raw.githubusercontent.com/Luyoung0001/picBed/main/hexotest.png))

至此，博客系统就初步搭建起来了。

### 更新博客
事实上，可以向 `hexotest/source/_post` 中放置大量的 markdown 文件，这些文件就是你想要写发布的博客。

写好了以后，继续执行：
```bash
hexo c
hexo g
hexo d
```

就好了。

## 三、从 csdn 到 github

经过寻找，终于让我在网上找到了这个项目：https://github.com/flytam/blog-sync-tool?tab=readme-ov-file

这个工具可以将博客从 csdn 等博客网站下载到本地，而且还支持图片下载，这意味着你csdn 中博客的图片都可以下载下来。

### 转移文章

参考 `csdnsynchexo` 的 文档，首先安装：
```bash
npx csdnsynchexo@latest  --help
```
安装好了就可以爬取了，但是别忘了需要阅读、修改等权限，谁的权限最高？当然是 csdn 作者本人了，因此这里需要你登录你自己的 csdn 账号后，获取到 csdn 的 cookie，并将其写入到配置文件。

#### 配置文件
可以将这个配置文件 `config.json` 放到任何地方，但是我还是建议在 ~ 目录下 创建一个 csdn目录，然后将这配置文件放置到 csdn 中：
```bash
{
        "userId": "m0_73651896",
        "type": "csdn",
        "output": "./blogfromcsdn",
        "cookie": ""
}
```

userId 就是 csdn 的 userID,output 就是将文章爬取下来后存入的目录，至于 cookie，这很重要。

获取方式：新开一个页面，F12(mac: cmd+shift+i)打开控制台，点击抓包这个[请求](https://blog-console-api.csdn.net/v1/editor/getArticle?id=104101476)的request headers中的cookie后面那段值。这个值很多，建议全部粘贴。
#### 爬取

然后就可以执行了：
```bash
npx csdnsynchexo@latest --config ./config.json
```

执行完成后，就可以在目标文件夹看到爬取下来的文件了：
```bash
ls
'(1) DNS Protocol Analysis Based on Wireshark at the Application Layer.md'
'2022 Personal Summary.md'
'2023 个人总结.md'
'2024.06.16 刷题日记.md'
'2024.06.17 刷题日记.md'
'2024.06.18 刷题日记.md'
'2024.06.19 刷题日记.md'
...
```
### 将文件拷贝到博客目录
将爬取下来的文章拷贝到 `hexotest/source/_post`，然后继续 `hexo cgd` 就可以了。

## 四、爬取图片

这个很重要，因为这里还牵扯到将以后写的文章的图片存储到哪里的问题。

好消息是 csdnsynchexo 支持爬取图片，并将这些图片上传到图床，然后将图床中图片的链接统一地在 markdown 中置换，这简直太棒了！

支持这一功能的是 PicGo，PicGo 是一个强大的用于快速上传图片并获取图片 URL 链接的工具，PicGo 本体支持如下图床：
- 七牛图床 v1.0
- 腾讯云 COS v4\v5 版本 v1.1 & v1.5.0
- 又拍云 v1.2.0
- GitHub v1.5.0
- SM.MS V2 v2.3.0-beta.0
- 阿里云 OSS v1.6.0
- Imgur v1.6.0

这里支持的很多，但是个人建议选取 github，因为未来我们要用到 CloudFlare+github 的形式来做cdn。

csdnsynchexo 中配置文件只需要这样修改以及添加：
```json
{
        "userId": "m0_73651896",
        "type": "csdn",
        "imgConfig": "./img.json",
        "output": "./blogfromcsdn",
        "cookie": ""
}
```
然后继续添加一个图片配置文件img.json：
```json
{
  "accessKey": "xxxxx",
  "secretKey": "xxxx",
  "bucket": "xxxxx",
  "area": "z2",
  "options": "",
  "path": "./blogfromcsdn",
  "picBed": {
      "uploader": "github",
      "github": {
        "repo": "Luyoung0001/picBed",
        "token": "",
        "branch": "main"
      }
    },
  "picgoPlugins": {}
}
```
具体配置方法，参考 [GitHub 图床](https://picgo.github.io/PicGo-Doc/zh/guide/config.html#github%E5%9B%BE%E5%BA%8A)。需要注意的是，一定要放置图床的仓库设为公开仓库，毕竟要从外面访问，要有访问权限。

之后，重新爬取、转移、生成、布置就好了！


## 五、写附带图片的文章
写文章很简单，就在`hexotest/source/_post`目录下进行创作就行了，但是插入图片怎么办呢？我们继续使用网络图片，放置自己picbed 中的图片。

问题是怎么上床呢？简单方法是直接 push 到这个仓库，然后抛链接，但是这不正规，况且文件的命名都是麻烦事。这里最好使用 PicGo。

## 六、配置使用 PicGo

首先下载 [PicGo](https://github.com/Molunerfinn/PicGo/releases):
选择一个合适的发行版，然后就可以在本地操作了。

具体参考[这里](https://picgo.github.io/PicGo-Doc/zh/guide/config.html#github%E5%9B%BE%E5%BA%8A)

## 总结
以上就是个人博客搭建的全过程，经过一上午的从 0 到 1,博客终于实现了，以后还得学习学习怎么添加一些feature。


