---
title: 【Linux网络编程】08 - 本地套接字
date: 2022-06-18
tags: [Linux, network]
categories: [Coding]
---

## 本地套接字 Unix Domain Socket
socket API原本是为网络通讯设计的，但后来在socket的框架上发展出一种**IPC机制**，就是**UNIX Domain Socket**。虽然网络socket也可用于同一台主机的进程间通讯（通过loopback地址127.0.0.1），但是UNIX Domain Socket用于IPC更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。这是因为，IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。UNIX Domain Socket也提供面向流和面向数据包两种API接口，类似于TCP和UDP，但是面向消息的UNIX Domain Socket也是可靠的，消息既不会丢失也不会顺序错乱。

### 使用本地套接字
使用UNIX Domain Socket的过程和网络socket十分相似，也要先调用`socket()`创建一个socket文件描述符，address family指定为`AF_UNIX`，type可以选择`SOCK_DGRAM`或`SOCK_STREAM`，protocol参数仍然指定为`0`即可。

UNIX Domain Socket与网络socket编程最明显的不同在于**地址格式**不同，用结构体`sockaddr_un`表示，网络编程的socket地址是IP地址加端口号，而UNIX Domain Socket的地址是一个socket类型的文件在文件系统中的**路径**，这个socket文件由`bind()`调用创建，如果调用`bind()`时该文件已存在，则`bind()`错误返回。

本地套接字的地址结构：  
```c
#include <sys/un.h>

struct sockaddr_un {
    __kernel_sa_family_t sun_family;    /* AF_UNIX */			地址结构类型
    char sun_path[UNIX_PATH_MAX]; 		/* pathname */		socket文件名(含路径)
};
```

`bind()`函数的使用有两点需要注意，`sockaddr_un`结构体前两字节存放`AF_UNIX`类型，后108字节存放socket文件的地址字符串，所以`bind()`函数的第三个参数应该为`2 + 地址长度`，但是这个`2`需要使用一个宏函数`offsetof()`来计算，`int len = offsetof(struct sockaddr_un, sun_path) + strlen(client_addr.sun_path);`（需要头文件`stddef.h`）

`bind()`调用成功会创建一个socket文件，为保证`bind()`成功，之前需要`unlink()`。

### 实现示例
server：  
```c
/*
@Filename : server.c
@Description : Unix Domain Socket IPC test
@Datatime : 2022/06/18 16:29:45
@Author : xushun
*/
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/un.h>
#include <stddef.h>
#include <ctype.h>

#define SERV_ADDR "server.socket"

int main(int argc, char** argv) {
    int listen_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sun_family = AF_UNIX;
    strcpy(serv_addr.sun_path, SERV_ADDR);
    int len = offsetof(struct sockaddr_un, sun_path) + strlen(serv_addr.sun_path);
    unlink(SERV_ADDR);
    bind(listen_fd, (struct sockaddr*)&serv_addr, len);
    listen(listen_fd, 20);

    struct sockaddr_un client_addr;
    int client_addr_len = sizeof(client_addr);
    int client_fd;

    char buf[4096];
    int readbytes;
    while (1) {
        client_addr_len = sizeof(client_addr);
        client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, (socklen_t*)&client_addr_len);
printf("ok %d\n", client_fd);;
        while ((readbytes = read(client_fd, buf, sizeof(buf))) > 0) {
            for (int i = 0; i < readbytes; ++ i) {
                buf[i] = toupper(buf[i]);
            }
            write(client_fd, buf, readbytes);
        }
        close(client_fd);
    }
    close(listen_fd);
    return 0;
}
```

client：  
```c
/*
@Filename : client.c
@Description : Unix domain socket IPC test
@Datatime : 2022/06/18 16:12:24
@Author : xushun
*/
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/un.h>
#include <stddef.h>

#define CLIT_ADDR "client.socket"
#define SERV_ADDR "server.socket"

int main(int argc, char** argv) {
    int client_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un client_addr;
    memset(&client_addr, 0, sizeof(client_addr));
    client_addr.sun_family = AF_UNIX;
    strcpy(client_addr.sun_path, CLIT_ADDR);
    int len = offsetof(struct sockaddr_un, sun_path) + strlen(client_addr.sun_path);
    unlink(CLIT_ADDR);
    bind(client_fd, (struct sockaddr*)&client_addr, len);

    struct sockaddr_un serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sun_family = AF_UNIX;
    strcpy(serv_addr.sun_path, SERV_ADDR);
    len = offsetof(struct sockaddr_un, sun_path) + strlen(serv_addr.sun_path);
    connect(client_fd, (struct sockaddr*)&serv_addr, len);

    char buf[4096];
    int readbytes;
    while ((readbytes = read(STDIN_FILENO, buf, sizeof(buf))) > 0) {
        write(client_fd, buf, readbytes);
        readbytes = read(client_fd, buf, sizeof(buf));
        write(STDOUT_FILENO, buf, readbytes);
    }
    close(client_fd);
    return 0;
}
```