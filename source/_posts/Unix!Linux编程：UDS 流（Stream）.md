---
title: Unix/Linux编程：UDS 流（Stream）
date: 2023-06-07 21:49:01
tags:
    - linux
    - unix
    - 服务器
    - 网络
categories: socket Web Unix/Linux
---

<!--more-->

## 〇、前言

socket 是一种 IPC （Inter-Process Communication，进程间通信）方法，它允许位于同一主机（计算机）或使用网络连接起来的不同主机上的应用程序之间交换数据。通过使用Socket，开发人员可以创建网络应用程序，使其能够通过网络进行数据交换和通信。

Socket API通常用于基于TCP/IP协议栈的网络通信，但也可以用于其他网络协议。它提供了一组函数和数据结构，允许应用程序创建、连接、发送和接收数据，并管理网络连接。

Socket API的使用通常涉及以下几个步骤：

- 创建Socket：使用socket()函数创建一个Socket对象，该函数指定了协议族（例如，IPv4、IPv6）和Socket类型（例如，流式Socket或数据报Socket）。
- 绑定Socket：对于服务器应用程序，需要使用`bind()`函数将Socket绑定到一个特定的IP地址和端口号。
- 监听连接请求（可选）：对于服务器应用程序，可以使用`listen()`函数开始监听传入的连接请求。
- 接受连接请求（可选）：对于服务器应用程序，使用`accept()`函数接受传入的连接请求，并创建一个新的Socket对象来处理与客户端的通信。

对于客户端：
- 建立连接（可选）：对于客户端应用程序，使用`connect()`函数与服务器建立连接。
- 发送和接收数据：使用`send()`函数将数据发送到远程主机，使用`recv()`函数从远程主机接收数据。
- 关闭Socket：使用`close()`函数关闭Socket连接。

这只是Socket编程的基本概念和步骤，具体的使用方法和函数会根据编程语言和操作系统而有所不同。常见的编程语言，如C、C++、Java和Python，都提供了相应的Socket API库，使开发人员能够使用Socket进行网络编程。

通过Socket编程，应用程序可以实现客户端-服务器模型，建立网络通信、传输数据和实现各种网络应用，如Web服务器、聊天应用、文件传输等。

流程图如下：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/02f24f092c57410994c1ae16f8d84e76_1720253774709.png?token=ANB4BCNYM7LLDGCVJVC4OGTGRD6YY)



本文将会写一个简单的、基本的服务器和客户端，并且成功让两者通信，平台为mac M1。

## 一、地址结构
对于各种 `socket domain` 都需要定义一个不同的结构类型来存储`socket`地址。然而由于诸如`bind()`之类的系统调用适用于所有 `socket domain` ， 因此它们必须要能够接受任意类型的地址结构。为支持这种行为，`socket API` 定义了一个通用 的地址结构 `struct sockaddr`。这个类型的唯一用途是将各种 domain 特定的地址结构转换成**单个类型**以供 socket 系统调用中的各个参数使用。`sockaddr` 结构通常被定义成如下所示的结构：·
```c
struct  sockaddr_un {
	unsigned char   sun_len;        /* sockaddr len including null */
	sa_family_t     sun_family;     /* [XSI] AF_UNIX */
	char            sun_path[104];  /* [XSI] path name (gag) */
};
```

因此，服务器中首先得创建一个 socket，并做好准备工作以及初始化：

```c
#define BACKLOG 5
#define SV_SOCK_PATH "/tmp/us_xfr"
#define BUF_SIZE 100


	struct sockaddr_un addr;
    int sfd, cfd;
    ssize_t numRead;
    char buf[BUF_SIZE];

    // 创建一个 socket
    sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sfd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }
    // 对这个长度是有规定的
    if (strlen(SV_SOCK_PATH) > sizeof(addr.sun_path) - 1) {
        printf("Server socket path too long: %s", SV_SOCK_PATH);
        exit(EXIT_FAILURE);
    }

    // 绑定之前，先将这个文件删除了，因为如果它存在的话，说明已经被绑定到了其它 socket 上
    if (remove(SV_SOCK_PATH) == -1 && errno != ENOENT) {
        printf("remove-%s", SV_SOCK_PATH);
        exit(EXIT_FAILURE);
    }

    // 初始化 socket
    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);
```

## 二、绑定地址
在Socket编程中，`bind()`函数用于将一个特定的网络地址或者本地地址绑定到一个Socket对象上。这个绑定操作指定了Socket要使用的本地网络地址和端口，使得其他进程可以通过这个地址和端口与该Socket进行通信。

`bind()` 函数的原型如下：

```c
#include <sys/types.h>
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

```
- sockfd：Socket文件描述符，表示要进行绑定的Socket对象。
- addr：指向struct sockaddr类型的**指针**，包含要绑定的本地地址信息。
- addrlen：addr结构体的大小。

以下是一个绑定网络地址的例子：

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>

int main() {
	// ipv4
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr;
    // 初始化
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);
    addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(sockfd, (struct sockaddr*)&addr, sizeof(addr)) == 0) {
        printf("Socket绑定成功\n");
    } else {
        perror("Socket绑定失败\n");
    }

    return 0;
}

```
## 三、监听：listen()

在Socket编程中，`listen()`函数用于将一个已绑定的Socket对象标记为被动状态，以便它可以开始监听传入的连接请求。`listen()`函数告知操作系统，该Socket将用于接受传入的连接，从而创建一个服务器端的Socket。
`listen()` 函数定义如下：

```c
#include <sys/types.h>
#include <sys/socket.h>

int listen(int sockfd, int backlog);

```
- sockfd：Socket文件描述符，表示要监听的Socket对象。
- backlog：等待连接队列的最大长度。

在调用`listen()` 函数时，需要提供一个已绑定的Socket对象，并指定等待连接队列的最大长度。等待连接队列是一个存储传入连接请求的缓冲区，如果该队列已满，新的连接请求将被拒绝。


## 四、请求链接：connect()
当服务器在 listen()的时候，这时候就可以由客户端发起链接请求。在Socket编程中，`connect()`函数用于建立客户端与服务器之间的连接。通过`connect()`函数，客户端可以向特定的服务器地址和端口发起连接请求，以便进行数据交换和通信。
以下是`connect()` 的原型：

```c
#include <sys/types.h>
#include <sys/socket.h>

int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

```
- sockfd：Socket文件描述符，表示要进行连接的Socket对象。
- addr：指向struct sockaddr类型的指针，包含要连接的服务器地址信息。
- addrlen：addr结构体的大小。

在调用`connect()` 函数时，需要传递一个struct sockaddr结构体指针作为参数，其中包含要连接的服务器地址信息。具体的地址信息结构体取决于使用的网络协议族，例如struct sockaddr_in用于IPv4地址。
以下是一个连接网络地址的示例：

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>

int main() {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8080);
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

    if (connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) == 0) {
        printf("成功与服务器建立连接\n");
    } else {
        perror("连接失败");
    }

    return 0;
}

```
需要注意的是，`connect()` 函数是一个阻塞调用，它会阻塞当前进程，直到连接建立成功或失败。如果连接成功建立，客户端就可以通过Socket与服务器进行数据交换。


## 五、接受链接请求：accept()


在Socket编程中，`accept()`函数用于服务器端接受客户端的连接请求，并创建一个新的Socket对象来与客户端进行通信。`accept()`函数在服务器端被调用，用于接受传入的连接请求，并返回一个新的Socket文件描述符，以便与客户端进行数据交换。
这里需要格外理解的是，如果在 `accept()`之前，没有客户端请求 `connect()`（未决的请求），那么服务端就会阻塞，直到有请求链接 `connect()`。`accept()`会创建一个新的 socket，并将这个新的 socket 替换为原来的 socket，这样原来的 socket 就可以一直处在监听状态！

accept()函数的原型如下：

```c
#include <sys/types.h>
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

- sockfd：Socket文件描述符，表示要接受连接请求的Socket对象；
- addr：指向struct sockaddr类型的指针，用于存储客户端的地址信息；
- addrlen：指向socklen_t类型的指针，用于存储addr结构体的大小。

## 六、关闭：close()

在Socket编程中，`close()`函数用于关闭一个打开的Socket连接或文件描述符。在网络编程中，`close()`函数用于关闭与对方主机的连接或者释放已创建的Socket对象。

`close()`函数的原型如下：

```c
#include <unistd.h>
int close(int sockfd);
```

- sockfd：Socket文件描述符或者文件描述符，表示要关闭的连接或文件。

在调用`close()`函数时，需要提供要关闭的Socket文件描述符或文件的文件描述符。`close()`函数将关闭指定的连接或文件，并释放相关的资源。

## 七、一个服务端的例子

这里会展示一个服务端的例子，这个例子给出了基本的创建方式和功能：

```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <zconf.h>


#define BACKLOG 5
#define SV_SOCK_PATH "/tmp/us_xfr"
#define BUF_SIZE 100
int main(int argc, char *argv[]) {
    struct sockaddr_un addr;

    int sfd, cfd;
    ssize_t numRead;
    char buf[BUF_SIZE];

    // 创建一个 socket
    sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sfd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }
    // 对这个长度是有规定的
    if (strlen(SV_SOCK_PATH) > sizeof(addr.sun_path) - 1) {
        printf("Server socket path too long: %s", SV_SOCK_PATH);
        exit(EXIT_FAILURE);
    }

    // 删除所有与路径名一致的既有文件，这样才能将 socket 绑定到这个路径名上
    if (remove(SV_SOCK_PATH) == -1 && errno != ENOENT) {
        printf("remove-%s", SV_SOCK_PATH);
        exit(EXIT_FAILURE);
    }

    // 初始化 socket
    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);

    //将 socket 绑定到该地址上
    if (bind(sfd, (struct sockaddr *)&addr, sizeof(struct sockaddr_un)) == -1) {
        printf("bind");
        exit(EXIT_FAILURE);
    }

    // 将这个 socket 标记为监听 socket。
    if (listen(sfd, BACKLOG) == -1) {
        printf("listen");
        exit(EXIT_FAILURE);
    }

    // 执行一个无限循环来处理进入的客户端请求。每次循环迭代执行下列任务
    for (;;) {
        // 接受一个连接，为该连接获取一个新 socket cfd
        cfd = accept(sfd, NULL, NULL);
        if (cfd == -1) {
            printf("accept");
            exit(EXIT_FAILURE);
        }

        //从已连接的 socket 中读取所有数据并将这些数据写入到标准输出中
        while ((numRead = read(cfd, buf, BUF_SIZE)) > 0)

            if (write(STDOUT_FILENO, buf, numRead) != numRead) {
                printf("partial/failed write");
                exit(EXIT_FAILURE);
            }

        if (numRead == -1) {
            printf("read");
            exit(EXIT_FAILURE);
        }

        // 关闭已连接的 socket cfd
        if (close(cfd) == -1) {
            printf("close");
            exit(EXIT_FAILURE);
        }
    }
}

```
这里需要注意的是：服务端在处理字符的时候，用了系统调用 `read()`和 `write()`，其中 `read()`从 `cfd` 中读取了某些字符存储到 `buf` 中；而 `write()`则是将 `buff` 中的数据写入到`STDOUT_FILENO`中，`STDOUT_FILENO`为标准输出，通常用文件描述符 1 表示，标准错误通常用文件描述符 2 表示，标准输入通常用文件描述符 0 表示。


## 八、一个客户端的例子

```c
// us_xfr.h
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <zconf.h>

#define BACKLOG 5
#define SV_SOCK_PATH "/tmp/us_xfr"
#define BUF_SIZE 100
int main(int argc, char *argv[]) {
    struct sockaddr_un addr;
    int sfd;
    ssize_t numRead;
    char buf[BUF_SIZE];

    sfd = socket(AF_UNIX, SOCK_STREAM, 0); /* Create client socket */
    if (sfd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }
    // 初始化
    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);

    if (connect(sfd, (struct sockaddr *)&addr, sizeof(struct sockaddr_un)) ==
        -1) {
        perror("connect");
        exit(EXIT_FAILURE);
    }

    /* Copy stdin to socket */
    while ((numRead = read(STDIN_FILENO, buf, BUF_SIZE)) > 0)
        if (write(sfd, buf, numRead) != numRead) {
            printf("partial/failed write");
            exit(EXIT_FAILURE);
        }

    if (numRead == -1) {
        printf("read");
        exit(EXIT_FAILURE);
    }

    exit(EXIT_SUCCESS); /* Closes our socket; server sees EOF */
}

```
## 九、启动客户端和服务端，并进行通信
编译，运行服务端：

```bash
(base) ***@shenjian Test % gcc 1.c -o server
(base) ***@shenjian Test % gcc 2.c -o client
./server


```
在另一个窗口运行客户端：

```bash
./client
hello

```
结果在服务端成功地接收到了客户端发来的消息`hello`:

```bash
(base) ***@shenjian Test % ./server
hello
```
*这就是一个简单的UDS 例子了，全文完，感谢阅读。*



