---
title: ngrok内网穿透以及原理
date: 2023-09-29 18:48:28
tags: Web 内网穿透
categories: NetWork Web
---

<!--more-->

## 〇、前言
如果要想在本地部署一个服务，且要向不在本局域网的用户展示我们的服务，此时就要用内网穿透工具，把我们的服务变成公网服务。ngrok就是一个很好的工具，操作简单，服务稳定。

## 一、使用 ngrok
### 1. 下载ngrok
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/4c37e5fe13de4f57a3b704751eabc96a_1720253717123.png?token=ANB4BCP5CBNNCUTIIKN77F3GRD6VQ)
下载下来后，这是一个命令行程序，直接可以运行。
### 2. 配置
运行下命令会将 authtoken 添加到默认的 ngrok.yml 配置文件中。

```bash
./ngrok config add-authtoken ********8HapnsMOQxHkeONvMvV_6GafMD4sWZBpBzxB4fBCC
```
### 3. 运行
比如你的服务运行在 8080 端口，就可以利用下面的命令完成内网穿透：
```bash
./ngrok http 8080
```
## 二、案例
我在本地运行一个简单的网络服务程序：

```bash
[GIN-debug] Listening and serving HTTP on :8080

```
然后直接运行内网穿透程序：

```bash
./ngrok http 8080                                                                                                                                         (Ctrl+C to quit)
                                                                                                                                                              
Introducing Always-On Global Server Load Balancer: https://ngrok.com/r/gslb                                                                                   
                                                                                                                                                              
Session Status                online                                                                                                                          
Account                       ************* (Plan: Free)                                                                                              
Version                       3.3.5                                                                                                                           
Region                        Asia Pacific (ap)                                                                                                               
Latency                       -                                                                                                                               
Web Interface                 http://127.0.0.1:4040                                                                                                           
Forwarding                    https://72b5-124-89-2-78.ngrok-free.app -> http://localhost:8080                                                                
                                                                                                                                                              
Connections                   ttl     opn     rt1     rt5     p50     p90                                                                                     
                              0       0       0.00    0.00    0.00    0.00  
```
这样就完成了本地服务的公网域名映射，在网页上打开`https://72b5-124-89-2-78.ngrok-free.app`，我们可以看到：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/287fd8bed61d4139adf183784d6cd514_1720253724529.png?token=ANB4BCNYSPQSZP4JJBMIS6LGRD6VU)
点击`Visit Site`，就看到了本地的游戏服务：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/1b39877d3492489bb256e1a2759f5e52_1720253724529.png?token=ANB4BCPLIKXEG53GUOJ77M3GRD6VW)

异地用户完成测试之后，我们就可以把内网穿透程、本地服务程序终止了。
## 三、ngrok工作原理：
工作原理很简单，就是咱们本地跑一个 ngrok 服务，然后 ngrok 这个软件公司有自己的服务器，上面跑的是 ngrokd 服务程序。
我们启动ngrok服务后，服务器就产生了一个连接。如果有别的用户访问这个域名，服务器拿到服务请求后，会通过 ngrokd 将这个请求转发给我们本地运行的 ngrok。之后 ngrok 就会把用户请求的数据发送给 ngrokd，然后 ngrokd 拿到数据后再转发给用户，这样就完成一次请求-服务。
总结一下就是：

> 1.当服务端接收到连接,就读取映射表,判断接收的端口对应于哪一个客户端,然后向客户端转发数据.
2.客户端收到数据,读取本地映射表,判断对应哪个内网地址,向内网地址发起连接.
3.客户端和内网的服务器建立连接后,向服务端发起一个连接,作为转发通道.
4.服务端读取请求数据,并通过转发通道转发到客户端,客户端读取响应并通过转发通道返回给请求.

*全文完，感谢阅读。*




