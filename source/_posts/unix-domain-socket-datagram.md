---
title: "Unix/Linux 系统编程：UDS 数据报通信"
date: 2023-06-08 20:51:32
tags:
  - "Unix domain socket"
  - "datagram"
  - "IPC"
categories:
  - "Systems Programming"
---

<!--more-->

## 〇、前言
对于`recvfrom()`来讲，src_addr 和 addrlen 参数会返回用来发送数据报的远程 socket 的地址。
（这些参数类似于 `accept()`中的 addr 和 addrlen 参数，它们返回已连接的对等 socket 的地址。) src_addr 参数是一个指针，它指向了一个与通信 domain 匹配的地址结构。与 `accept()`一样， addrlen 是一个值-结果参数。在调用之前应该将 addrlen 初始化为 src_addr 指向的结构的大小；在返回之后，它包含了实际写入这个结构的字节数。

## 一、接收：recvfrom()
在Socket编程中，`recvfrom()`函数用于从套接字接收数据，并获取发送方的地址信息。它适用于基于数据报的套接字，如UDP套接字。
`recvfrom()`函数的原型如下：

```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);

```

- sockfd：Socket文件描述符，表示要接收数据的套接字。
- buf：指向接收数据的缓冲区的指针。
- len：缓冲区的大小，表示最多接收多少字节的数据。
- flags：标志参数，用于控制接收操作的行为，通常设置为0。
- src_addr：指向struct sockaddr类型的指针，用于存储发送方的地址信息。
- addrlen：指向socklen_t类型的指针，用于存储src_addr结构体的大小。

在调用`recvfrom()` 函数时，需要提供一个已经创建和绑定的套接字，并传递一个缓冲区以存储接收到的数据。函数将阻塞，直到有数据到达套接字。

当接收到数据时，`recvfrom()` 函数将数据存储在提供的缓冲区中，并将发送方的地址信息存储在src_addr中。实际接收的字节数将作为函数的返回值返回，如果返回值为-1，则表示发生了错误。

下面是一个简单的示例，演示如何使用`recvfrom()`函数接收UDP套接字的数据：

```c
// us_xfr.h
#include <ctype.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <zconf.h>

#define BUF_SIZE 10

#define SV_SOCK_PATH "/tmp/ud_ucase"

int main(int argc, char *argv[]) {
    struct sockaddr_un svaddr, claddr;
    int sfd, j;
    ssize_t numBytes;
    socklen_t len;
    char buf[BUF_SIZE];

    sfd = socket(AF_UNIX, SOCK_DGRAM, 0); /* Create server socket */
    if (sfd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    if (strlen(SV_SOCK_PATH) > sizeof(svaddr.sun_path) - 1) {
        printf("Server socket path too long: %s", SV_SOCK_PATH);
        exit(EXIT_FAILURE);
    }

    if (remove(SV_SOCK_PATH) == -1 && errno != ENOENT) {
        printf("remove-%s", SV_SOCK_PATH);
        exit(EXIT_FAILURE);
    }

    memset(&svaddr, 0, sizeof(struct sockaddr_un));
    svaddr.sun_family = AF_UNIX;
    strncpy(svaddr.sun_path, SV_SOCK_PATH, sizeof(svaddr.sun_path) - 1);

    if (bind(sfd, (struct sockaddr *)&svaddr, sizeof(struct sockaddr_un)) ==
        -1) {
        printf("bind");
        exit(EXIT_FAILURE);
    }

    /* Receive messages, convert to uppercase, and return to client */
    for (;;) {
        len = sizeof(struct sockaddr_un);
        numBytes =
            recvfrom(sfd, buf, BUF_SIZE, 0, (struct sockaddr *)&claddr, &len);
        if (numBytes == -1) {
            printf("recvfrom");
            exit(EXIT_FAILURE);
        }

        printf("Server received %zd bytes from %s\n", numBytes,
               claddr.sun_path);

        for (j = 0; j < numBytes; j++)
            buf[j] = toupper((unsigned char)buf[j]);

        if (sendto(sfd, buf, numBytes, 0, (struct sockaddr *)&claddr, len) !=
            numBytes) {
            printf("sendto");
            exit(EXIT_FAILURE);
        }
    }
}

```

## 二、发送：sendto()
在Socket编程中，`sendto()` 函数用于通过套接字发送数据到指定的目标地址。它适用于基于数据报的套接字，如UDP套接字。
`sendto()` 函数的原型如下：

```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);

```
- sockfd：Socket文件描述符，表示要发送数据的套接字。
- buf：指向要发送的数据的缓冲区的指针。
- len：要发送的数据的字节数。
- flags：标志参数，用于控制发送操作的行为，通常设置为0。
- dest_addr：指向struct sockaddr类型的目标地址结构体的指针，用于指定发送数据的目标地址。
- addrlen：dest_addr结构体的大小。

下面是一个简单的示例，演示如何使用`sendto()` 函数发送UDP套接字的数据：

```c
#include <ctype.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <zconf.h>

#define BUF_SIZE                                                               \
    10 /* Maximum size of messages exchanged between client and server */
#define SV_SOCK_PATH "/tmp/ud_ucase"
int main(int argc, char *argv[]) {
    struct sockaddr_un svaddr, claddr;
    int sfd, j;
    size_t msgLen;
    ssize_t numBytes;
    char resp[BUF_SIZE];

    if (argc < 2 || strcmp(argv[1], "--help") == 0) {
        printf("%s msg...\n", argv[0]);
        exit(1);
    }

    /* Create client socket; bind to unique pathname (based on PID) */

    sfd = socket(AF_UNIX, SOCK_DGRAM, 0);
    if (sfd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    memset(&claddr, 0, sizeof(struct sockaddr_un));
    claddr.sun_family = AF_UNIX;
    snprintf(claddr.sun_path, sizeof(claddr.sun_path), "/tmp/ud_ucase_cl.%ld",
             (long)getpid());

    if (bind(sfd, (struct sockaddr *)&claddr, sizeof(struct sockaddr_un)) ==
        -1) {
        perror("bind");
        exit(EXIT_FAILURE);
    }

    /* Construct address of server */

    memset(&svaddr, 0, sizeof(struct sockaddr_un));
    svaddr.sun_family = AF_UNIX;
    strncpy(svaddr.sun_path, SV_SOCK_PATH, sizeof(svaddr.sun_path) - 1);


    for (j = 1; j < argc; j++) {
        msgLen = strlen(argv[j]);
        if (sendto(sfd, argv[j], msgLen, 0, (struct sockaddr *)&svaddr,
                   sizeof(struct sockaddr_un)) != msgLen) {
            perror("sendto");
            exit(EXIT_FAILURE);
        }

        numBytes = recvfrom(sfd, resp, BUF_SIZE, 0, NULL, NULL);

        if (numBytes == -1) {
            perror("recvfrom");
            exit(EXIT_FAILURE);
        }
        printf("Response %d: %.*s\n", j, (int)numBytes, resp);
    }
    remove(claddr.sun_path);
    exit(EXIT_SUCCESS);
}

```
## 三、运行
编译运行之后，启动客户端和服务端，键入：

```bash
(base) ***@shenjian Test % ./client_ud 你好
hello nihao
Response 1: 你好
Response 2: HELLO
Response 3: NIHAO
```
可以在服务端看到：

```bash
Server received 6 bytes from /tmp/ud_ucase_cl.16568
Server received 5 bytes from /tmp/ud_ucase_cl.16568
Server received 5 bytes from /tmp/ud_ucase_cl.16568
```
可以看到运行结果符合预期。

*全文完，感谢阅读。*
