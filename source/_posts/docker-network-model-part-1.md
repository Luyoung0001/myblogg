---
title: "Docker 网络模型初探（一）：docker0 与容器互联"
date: 2023-07-04 15:51:59
tags:
  - "Docker"
  - "networking"
  - "container"
categories:
  - "Networking"
---

<!--more-->

## 〇、前言
安装Docker时，它会自动创建三个网络，默认`bridge网桥`（创建容器默认连接到此网络）、 `none` 、`host`。各个方式有各自的特点，它们有着特定的差距，比如网络性能等，一般按照实际应用方式手动指定（或者选用默认）一个网络，或者使用命令自己添加一个自己定义的网络。
## 一、关于Docker0

Docker0 是 Docker 引擎创建的默认网络接口。当 Docker 引擎安装和启动时，它会自动创建一个名为 docker0 的**虚拟网络接口**，本质上是一个虚拟网桥。

Docker0 接口用于连接 **Docker 容器**和**宿主机**之间的通信。它是一个虚拟以太网桥接口，用于在 Docker 主机和容器之间进行数据传输。

Docker0 接口有以下特点：

- IP 地址：默认情况下，Docker0 接口会**自动**分配一个 IP 地址，通常是 172.17.0.1。这个 IP 地址用作 Docker 主机上运行的容器的**网关地址**。

- 网络连接：Docker0 接口会与宿主机的物理网卡（通常是 eth0）进行连接，通过桥接的方式实现容器和宿主机之间的通信。

- 网络转发：Docker0 接口负责将从容器发出的网络流量转发到宿主机的物理网卡，以及将从宿主机发出的网络流量转发给容器。

- 网络配置：Docker0 接口的网络配置可以通过 Docker 引擎的配置文件进行调整，如修改 IP 地址范围、子网掩码等。

需要注意的是，Docker0 接口是与宿主机网络环境紧密相关的，每个 Docker 主机只有一个 Docker0 接口。在 Docker 容器之间通信时，可以通过 Docker0 接口进行转发，但如果需要与外部网络通信，**还需要进行端口映射或使用其他网络模式**。

***罗素说过，我们不管听到多么复杂的概念和事物，我们应该及时的问自己，这个概念或事物真正说了些什么，或者事实究竟是什么。***

那么 Docker0 到底是什么呢？

启动两个容器看看。

## 二、容器网络分析

### 1、获取两个镜像

```bash
docker pull archlinux
docker pull ubuntu
```

### 2、以默认方式启动镜像
以下命令以**默认方式**（bridge网络）创建并启动以及运行了容器里面的bash 程序（最好打开多个终端窗口，每个窗口运行不同的容器，不用来回切换），这样能直接进入容器里面：

```bash
docker run -it --name=arch2 archlinux /bin/bash
docker run -it --name=ubuntu2 ubuntu /bin/bash
```
在另一个终端（宿主机）上看看：

```bash
root@**********:~# docker ps
CONTAINER ID   IMAGE       COMMAND       CREATED        STATUS          PORTS     NAMES
30f913d8e3ff   ubuntu      "/bin/bash"   14 hours ago   Up 35 minutes             ubuntu2
ec0ca9aedc9f   archlinux   "/bin/bash"   14 hours ago   Up 36 minutes             arch2
```
可以看到确实有两个容器正在运行。

### 3、查看容器网络

```bash
root@********:~# docker network ls
NETWORK ID     NAME           DRIVER    SCOPE
da149ecc5bf3   bridge         bridge    local
5c60928659e8   host           host      local
1c8eeab72508   none           null      local
```
可以看到本地确实有 3 个网络，因为这是以默认方式启动的，直接查看 **bridge** 网络信息：

```bash
root@*******:~# docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "da149ecc5bf3de9eef56945b6061cc512e78e909cde4c53b8a80cc6fb0ad3015",
        "Created": "2023-07-03T23:29:11.078138805+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "30f913d8e3fffbc5e03df7be0e99c7d43dc52c5f761638e8deae6da30933ab06": {
                "Name": "ubuntu2",
                "EndpointID": "b51799e6235505d0c571595cc210a004933822ab7d0c164353b6ceab562e1eb4",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "ec0ca9aedc9fff82688a5590d8b3243cb60b33bd745962cb94ea5848bc170525": {
                "Name": "arch2",
                "EndpointID": "e4bdf03a6363a2b7aa80af73aba71f6f5c775025d61164dccc4474376520663f",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

只需要把目光放到关于 `bridge`、`containers` 的信息上来：

```bash
"IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
```
可以看到 bridge 网关为 `172.18.0.1`，子网掩码为`172.18.0.0/16`。
```bash
 "Containers": {
            "30f913d8e3fffbc5e03df7be0e99c7d43dc52c5f761638e8deae6da30933ab06": {
                "Name": "ubuntu2",
                "EndpointID": "b51799e6235505d0c571595cc210a004933822ab7d0c164353b6ceab562e1eb4",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "ec0ca9aedc9fff82688a5590d8b3243cb60b33bd745962cb94ea5848bc170525": {
                "Name": "arch2",
                "EndpointID": "e4bdf03a6363a2b7aa80af73aba71f6f5c775025d61164dccc4474376520663f",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
```
对于 `ubuntu2`，它的 Mac 地址为：`02:42:ac:12:00:03`，ipv4为：`172.18.0.3/16`；
对于 `arch2`，它的 Mac 地址为：`02:42:ac:12:00:02`，ipv4 为：`172.18.0.2/16`。


***这两项信息非常重要，因为一会儿要验证。***

### 4、宿主机网络配置
我们通过以下命令查看本机网络(这里夹杂了一个自定义的网络，可以忽略)：

```bash
root@********:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:16:3e:00:7c:2d brd ff:ff:ff:ff:ff:ff
    inet 172.17.9.165/18 brd 172.17.63.255 scope global dynamic eth0
       valid_lft 314697479sec preferred_lft 314697479sec
    inet6 fe80::216:3eff:fe00:7c2d/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:2c:62:19:d9 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:2cff:fe62:19d9/64 scope link
       valid_lft forever preferred_lft forever
4: br-b1e26274afd7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:8e:59:43:24 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.1/16 brd 172.19.255.255 scope global br-b1e26274afd7
       valid_lft forever preferred_lft forever
    inet6 fe80::42:8eff:fe59:4324/64 scope link
       valid_lft forever preferred_lft forever
14: veth7958290@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 5e:67:bd:18:47:4d brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::5c67:bdff:fe18:474d/64 scope link
       valid_lft forever preferred_lft forever
16: vetha2319a1@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-b1e26274afd7 state UP group default
    link/ether 0e:28:d3:40:1e:97 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::c28:d3ff:fe40:1e97/64 scope link
       valid_lft forever preferred_lft forever
18: veth23c3fd5@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 56:a1:a5:02:37:0b brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::54a1:a5ff:fe02:370b/64 scope link
       valid_lft forever preferred_lft forever
20: veth952254a@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-b1e26274afd7 state UP group default
    link/ether 42:83:4d:c3:7d:f8 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::4083:4dff:fec3:7df8/64 scope link
       valid_lft forever preferred_lft forever
```

重点关注 `docker0` 上的信息：

```bash
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:2c:62:19:d9 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:2cff:fe62:19d9/64 scope link
       valid_lft forever preferred_lft forever
     valid_lft forever preferred_lft forever

14: veth7958290@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 5e:67:bd:18:47:4d brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::5c67:bdff:fe18:474d/64 scope link
       valid_lft forever preferred_lft forever

18: veth23c3fd5@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 56:a1:a5:02:37:0b brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::54a1:a5ff:fe02:370b/64 scope link
       valid_lft forever preferred_lft forever

```

解释一下就是：

 - ` docker0`：这是 Docker 默认创建的虚拟网桥接口，用于连接 Docker 主机和容器。它的状态为 UP，表示已启用。它的 IP 地址是 **172.18.0.1**，子网掩码为 16。它的 MAC 地址是 02:42:2c:62:19:d9。这个接口的主要作用是实现 Docker 主机和容器之间的网络通信。
-  `veth7958290@if13`：这是一个虚拟以太网设备，与 **docker0 网桥**相连。它的状态为 UP，表示已启用。它的 MAC 地址是 5e:67:bd:18:47:4d。它的 IPv6 地址是 fe80::5c67:bdff:fe18:474d。这个接口是连接到 docker0 网桥的容器的一部分。
-  `veth23c3fd5@if17`：这是另一个虚拟以太网设备，与 **docker0 网桥**相连。它的状态为 UP，表示已启用。它的 MAC 地址是 56:a1:a5:02:37:0b。它的 IPv6 地址是 fe80::54a1:a5ff:fe02:370b。这个接口也是连接到 docker0 网桥的容器的一部分。

这些接口是 Docker 在创建和管理容器时自动创建的虚拟网络设备，用于容器之间和容器与主机之间的通信。每个容器都有自己的虚拟网络接口，并且通过网桥与主机和其他容器连接起来。这样，容器之间可以进行网络通信。

现在一个清晰的图景被描述出来了：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/3ef0f50bbeff41279852706101fa273a_1720253746882.png?token=ANB4BCKDN6AXTOJXMEV2JF3GRD6XC)

为了获得一个完整的图景，进入容器再研究研究。

### 5、ubuntu2、arch2网络配置
`ubuntu2` 网络信息：
```bash
root@*******:/# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
17: eth0@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever

```
解释一下就是：
- 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
这是回环接口（loopback interface），用于本地主机内部通信。
<LOOPBACK,UP,LOWER_UP> 表示接口是一个回环接口，并且处于启用状态。
mtu 65536 表示最大传输单元的大小。
qdisc noqueue 表示没有队列机制，即无需排队即可传输数据。
eth0@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default

- 这是第一个以太网接口（Ethernet interface），标记为 eth0。
<BROADCAST,MULTICAST,UP,LOWER_UP> 表示接口支持广播和多播，并且处于启用状态。
mtu 1500 表示最大传输单元的大小为1500字节。
qdisc noqueue 表示没有队列机制，即无需排队即可传输数据。
state UP 表示接口处于启用状态。
link/ether 02:42:ac:12:00:03 表示接口的物理地址（MAC 地址）为 02:42:ac:12:00:03。
inet 172.18.0.3/16 表示接口的 IPv4 地址为 172.18.0.3，子网掩码为 /16。
brd 172.18.255.255 表示广播地址。
scope global 表示接口的地址在全局范围内可达。


`arch2` 网络信息：

```bash
[root@******* /]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
13: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

### 6、完整的网络拓扑图景

通过以上的分析，就能轻而易举地得到这个图：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/8bb4cafcf70b41018793d0fc80f3fa11_1720253746882.png?token=ANB4BCKER4WYBYWSZETJLZ3GRD6XG)
## 三、结论
1、ubuntu2、arch2是共用的一个路由器（网桥）：`docker0`；

2、所有的容器启动时，如果不指定网络的情况下，都是`docker0`路由的；

3、`docker0` 通过 NAT 穿透连到 host 主机的物理网卡上（补充）。

## 四、测试

通过 arch2 去 ping ubuntu1:

```bash
root@******** /]# ping 172.18.0.3
PING 172.18.0.3 (172.18.0.3) 56(84) bytes of data.
64 bytes from 172.18.0.3: icmp_seq=1 ttl=64 time=0.095 ms
64 bytes from 172.18.0.3: icmp_seq=2 ttl=64 time=0.123 ms
64 bytes from 172.18.0.3: icmp_seq=3 ttl=64 time=0.160 ms
......
```
这是没问题的，因为它们位于同一个内网，当然 ping 的通。

```bash
[root@******** /]# ping baidu.com
PING baidu.com (110.242.68.66) 56(84) bytes of data.
64 bytes from 110.242.68.66 (110.242.68.66): icmp_seq=1 ttl=48 time=28.2 ms
64 bytes from 110.242.68.66 (110.242.68.66): icmp_seq=2 ttl=48 time=28.2 ms
64 bytes from 110.242.68.66 (110.242.68.66): icmp_seq=3 ttl=48 time=28.2 ms
......
```
这是 ping 互联网的服务器，当然也 ping 的通，就像你手机连接 wifi，肯定可以上网。

*本节完，感谢你的阅读。*

