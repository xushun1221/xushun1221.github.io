---
title: 【Linux网络编程】01 - Socket编程
date: 2022-06-08
tags: [Linux, network]
categories: [Coding]
---

## 网络套接字 socket
Socket本身有插座的意思，在Linux环境下，用于表示**进程间网络通信**的特殊文件类型。本质为内核借助缓冲区形成的伪文件。

既然是文件，那么理所当然的，我们可以使用**文件描述符引用套接字**。与管道类似的，Linux系统将其封装成文件的目的是为了统一接口，使得读写套接字和读写文件的操作一致。区别是管道主要应用于本地进程间通信，而套接字多应用于网络进程间数据的传递。

在TCP/IP协议中，**IP地址+TCP或UDP端口号**唯一标识网络通讯中的一个进程。**IP地址+端口号**就对应一个socket。欲建立连接的两个进程各自有一个socket来标识，那么这两个socket组成的socket pair就唯一标识一个连接。因此可以用Socket来描述网络连接的一对一关系。

- 在网络通信中，套接字一定是成对出现的。
- 一个文件描述符指向一个套接字socket（套接字内部由内核借助两个缓冲区实现），一端的发送缓冲区对应对端的接收缓冲区。我们使用同一个文件描述符索发送缓冲区和接收缓冲区。

## 网络字节序
计算机内存中的**多字节数据**，相对于内存地址来说，有两种存储方式：
- 小端法：用于计算机内存，数据的最低有效位对应低地址，数据的最高有效位对应高地址；
- 大端法：用于网络数据流，数据的最低有效位对应高地址，数据的最高有效位对应低地址。

读数据的顺序通常是从内存的低地址到高地址，本地和网络的字节序不同会造成错误。

例如，主机A有这样一个四字节数据`0x12ef34ab`，它在内存中的起始地址是`0x01`，使用小端法存储：

|地址|数据|
|---|---|
|0x04|0x12|
|0x03|0xef|
|0x02|0x34|
|0x01|0xab|

主机A将该四字节数据通过网络发送给主机B，使用大端字节序发送，发送的字节流为`0x12 0xef 0x34 0xab`，因为主机B使用的也是小段法，如果直接将该字节流装入内存（装入顺序从低地址到高地址），会得到`0xab34ef12`，和原始数据不同，需要进行转换。

收到的大端字节序的字节流：

|地址|数据|
|---|---|
|0x04|0xab|
|0x03|0x34|
|0x02|0xef|
|0x01|0x12|

转换为小端字节序，才可以得到原始数据`0x12ef34ab`：

|地址|数据|
|---|---|
|0x04|0x12|
|0x03|0xef|
|0x02|0x34|
|0x01|0xab|

### 网络字节序和主机字节序的转换
为了让相同的C程序在小端计算机和大端计算机中都可以正常运行，应该使用下面的库函数进行网络字节序和主机字节序的转换，以保证程序的可移植性。

```c
#include <arpa/inet.h>

uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);

uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(uint16_t netshort);
```
- `h`：host，主机；
- `n`：net，网络；
- `l`：32位无符号整型；
- `s`：16位无符号整型；
- `htonl`：主机字节序 -> 网络字节序 32位无符号整型，用于IP地址
- `ntohl`：网络字节序 -> 主机字节序 32位无符号整型，用于IP地址
- `htons`：主机字节序 -> 网络字节序 16位无符号整型，用于port端口
- `ntohs`：网络字节序 -> 主机字节序 16位无符号整型，用于port端口

## IP地址转换
在网络通信中使用的IP地址，是32位的大端字节序，而我们人类阅读的IP地址是点分十进制的字符串类型。需要借助下面两个函数在这两种形式之间进行转化。

```c
#include <arpa/inet.h>

int inet_pton(int af, const char *src, void *dst);

const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```
- `inet_pton`：将点分十进制的IP地址字符串转化为32位二进制大端字节序IP地址（支持IPv4或IPv6地址）
  - `af`：IP协议类型（v4和v6）
    - `AF_INET`：IPv4
    - `AF_INET6`：IPv6
  - `src`：点分十进制的IP地址字符串
  - `dst`：返回转换后的IP
  - 返回值：
    - 成功，返回`1`
    - 当`src`不是一个有效的IP，返回`0`
    - 当`af`不是一个有效的地址族，返回`-1`并设置`errno`
- `inet_ntop`：将32位二进制大端字节序IP地址转化为点分十进制的IP地址字符串（支持IPv4和IPv6）
  - `af`：IP协议类型
  - `src`：大端字节序的IP
  - `dst`：返回点分十进制字符串到一个缓冲区
  - `size`：缓冲区`dst`的大小
  - 返回值：
    - 成功，返回`dst`
    - 失败，返回`NULL`并设置`errno`

## sockaddr数据结构
`sockaddr`一系列的数据结构用来描述套接字，示意图如下：

![](/post_images/posts/Coding/【Linux网络编程】01/sockaddr系列示意图.jpg "sockaddr系列示意图")

由于许多网络编程函数诞生早于IPv4协议，那时候使用的都是`sockaddr`结构体，但是现在被淘汰了，为了向前兼容，现在`sockaddr`退化成了`(void*)`的作用，传递一个地址给函数，至于这个函数是`sockaddr_in`（IPv4）还是`sockaddr_in6`（IPv6），由地址族确定，然后函数内部强制类型转换为所需的地址类型。（`sockaddr_un`用于本地套接字）

IPv4和IPv6的地址格式定义在`netinet/in.h`中，IPv4地址用`sockaddr_in`结构体表示，包括16位端口号和32位IP地址，IPv6地址用`sockaddr_in6`结构体表示，包括16位端口号、128位IP地址和一些控制字段。UNIX Domain Socket的地址格式定义在`sys/un.h`中，用`sockaddr_un`结构体表示。

各种socket地址结构体的开头都是相同的，前16位表示整个结构体的长度（并不是所有UNIX的实现都有长度字段，如Linux就没有），后16位表示地址类型。IPv4、IPv6和Unix Domain Socket的地址类型分别定义为常数`AF_INET`、`AF_INET6`、`AF_UNIX`。这样，只要取得某种sockaddr结构体的首地址，不需要知道具体是哪种类型的sockaddr结构体，就可以根据地址类型字段确定结构体中的内容。因此，socket API可以接受各种类型的sockaddr结构体指针做参数，例如`bind`、`accept`、`connect`等函数，这些函数的参数应该设计成`void *`类型以便接受各种类型的指针，但是sock API的实现早于ANSI C标准化，那时还没有void *类型，因此这些函数的参数都用`struct sockaddr *`类型表示，在传递参数之前要强制类型转换一下，

使用命令`man 7 ip`查看结构体详细定义。

### sockaddr_in
定义：  
```c
struct sockaddr_in {
    sa_family_t    sin_family; /* address family: AF_INET */
    in_port_t      sin_port;   /* port in network byte order */
    struct in_addr sin_addr;   /* internet address */
}; // 还包含一些填充字节 这里忽略

/* Internet address. */
struct in_addr {
    uint32_t       s_addr;     /* address in network byte order */
};
```
`sockaddr_in`在头文件`netinet/in.h`或`arpa/inet.h`中定义。

使用示例：  
```c
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8888);
int dst;
inet_pton(AF_INET, "192.168.1.100", (void*)&dst);
addr.sin_addr.s_addr = dst;
// addr.sin_addr.s_addr = htonl(INADDR_ANY); // INADDR_ANY取出系统中有效的一个IP地址 二进制
bind(fd, (struct sockaddr*)&addr, size);
```

## 网络套接字函数

### socket模型流程图

![](/post_images/posts/Coding/【Linux网络编程】01/socket模型流程图.jpg "socket模型流程图")

- 服务器
  1. `socket()`，创建一个监听socket
  2. `bind()`，给监听socket绑定服务器的地址结构（IP+Port）
  3. `listen()`，设置监听数量上限
  4. `accept()`，阻塞监听客户端连接
  5. `read()`，读socket获取客户端发送的数据
  6. 对收到的数据进行处理
  7. `write()`，写socket向客户端发送数据
  8. `close()`
- 客户端
  1. `socket()`，创建一个socket用于和服务器通信
  2. `connet()`，socket和服务器连接
  3. `write()`，写数据到socket发送给服务器
  4. `read()`，读socket接收服务器发来的数据
  5. 处理接收的数据
  6. `close()`


### socket函数
创建一个套接字（通信端点）。

函数原型：  
```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```
- `domain`：使用的IP协议
  - `AF_INET`
  - `AF_INET6`
  - `AF_UNIX`
- `type`：数据传输协议
  - `SOCK_STREAM`：流式
  - `SOCK_DGRAM`：报式
- `protocol`：默认传`0`，根据`type`自动选择
  - `SOCK_STREAM`：TCP
  - `SOCK_DGRAM`：UDP
- 返回值：
  - 成功，返回新套接字对应的文件描述符
  - 失败，返回`-1`并`errno`

### bind
给socket绑定一个地址结构（IP+Port）。

函数原型：  
```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
- `sockfd`：套接字对应的文件描述符，`socket()`的返回值
- `addr`：要绑定的地址结构`(struct sockaddr*)&addr`（服务器自己的地址结构）
- `addrlen`：`sizeof(addr)`，地址结构的大小
- 返回值：
  - 成功，返回`0`
  - 失败，返回`-1`并`errno`

主要用于服务器，给监听socket绑定一个端口号，而客户端不需要绑定，系统会进行隐式绑定，自动分配一个端口号。

### listen
设置同时与服务器建立连接的上限数。（同时和服务器握手的客户端数量）

函数原型：  
```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int listen(int sockfd, int backlog);
```
- `sockfd`：套接字对应的文件描述符，`socket()`的返回值
- `backlog`：最大连接数（最大值为128）
- 返回值：
  - 成功，返回`0`
  - 失败，返回`-1`并`errno`

### accept
阻塞等待客户端连接。

函数原型：  
```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```
- `sockfd`：监听socket的描述符，`socket()`返回值
- `addr`：传出参数，成功与服务器建立连接的客户端的地址结构（IP+Port）
- `addrlen`：传入传出参数，传入时值应为`sizeof(addr)`，返回客户端`addr`的大小
- 返回值：
  - 成功，与客户端成功连接的socket文件描述符（非负）
  - 失败，返回`-1`并`errno`

### connect
使用现有的socket与服务器建立连接。

函数原型：  
```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
- `sockfd`：之前创建的socket的文件描述符
- `addr`：服务器的地址结构（IP+Port）
- `addrlen`：服务器地址结构的大小
- 返回值：
  - 成功，返回`0`
  - 失败，返回`-1`并`errno`

## socket编程示例
客户端发送小写字符给服务器，服务器返回对应的大写字符。

### 服务器实现
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <ctype.h>

#define SERV_PORT 8888

int main(int argc, char** argv) {
    int ret;
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0); // sys/scoket.h
    if (listen_fd == -1) {
        perror("socket error"); exit(1);
    }
    struct sockaddr_in serv_addr; // arpa/inet.h
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT); // arpa/inet.h
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    ret = bind(listen_fd, (struct scokaddr*)&serv_addr, sizeof(serv_addr));
    if (ret == -1) {
        perror("bind error"); exit(1);
    }
    ret = listen(listen_fd, 128);
    if (ret == -1) {
        perror("listen error"); exit(1);
    }
    struct sockaddr_in clit_addr;
    socklen_t clit_addr_len = sizeof(clit_addr);
    int clit_fd = accept(listen_fd, (struct sockaddr*)&clit_addr, &clit_addr_len);
    if (clit_fd == -1) {
        perror("accept error"); exit(1);
    }
    char clit_IP[1024];
    printf("client ip : %s port : %d\n", 
        inet_ntop(AF_INET, &clit_addr.sin_addr.s_addr, clit_IP, sizeof(clit_IP)),
        ntohs(clit_addr.sin_port)
    );
    // char read_buf[BUFSIZ]; // BUFSIZ stdio.h 中定义的默认缓存区大小 8192
    char buf[4096];
    int read_bytes;
    while (1) {
        read_bytes = read(clit_fd, buf, sizeof(buf));
        write(STDOUT_FILENO, buf, read_bytes);
        for (int i = 0; i < read_bytes; ++ i) {
            buf[i] = toupper(buf[i]); // ctype.h
        }
        write(clit_fd, buf, read_bytes);
    }
    close(listen_fd);
    close(clit_fd);
    return 0;
}
```

还没写客户端，可以用`nc 127.0.0.1 8888`命令测试：  
```console
xushun@xushun-virtual-machine:~$ nc 127.0.0.1 8888
hello
HELLO

```

### 客户端
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define SERV_PORT 8888

int main(int argc, char** argv) {
    int ret;
    int client_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (client_fd == -1) {
        perror("socket error"); exit(1);
    }
    struct sockaddr_in serv_addr;
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr.s_addr);
    ret = connect(client_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if (ret == -1) {
        perror("connect error"); exit(1);
    }
    int read_bytes;
    char buf[4096];
    while (1) {
        read_bytes = read(STDIN_FILENO, buf, sizeof(buf));
        write(client_fd, buf, read_bytes);
        read_bytes = read(client_fd, buf, sizeof(buf));
        write(STDOUT_FILENO, buf, read_bytes);
    }
    close(client_fd);
    return 0;
}
```

## 错误处理封装
上面的例子不仅功能简单，而且简单到几乎没有什么错误处理，我们知道，系统调用不能保证每次都成功，必须进行出错处理，这样一方面可以保证程序逻辑正常，另一方面可以迅速得到故障信息。

为使错误处理的代码不影响主程序的可读性，我们把与socket相关的一些系统函数加上错误处理代码包装成新的函数，做成一个模块。

`wrap.h`：  
```c
#ifndef __WRAP_H_
#define __WRAP_H_

#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <sys/socket.h>
#include <arpa/inet.h>

void perr_exit(const char* s);
int Accept(int fd, struct sockaddr* sa, socklen_t* salen); // 和accept相比只有一个字母变大写 可以在vim中跳转manpages
int Bind(int fd, const struct sockaddr* sa, socklen_t salen);
int Connect(int fd, const struct sockaddr* sa, socklen_t salen);
int Listen(int fd, int backlog);
int Socket(int family, int type, int protocol);
ssize_t Read(int fd, void* buf, size_t nbytes);
ssize_t Write(int fd, const void* buf, size_t nbytes);
int Close(int fd);
ssize_t Readn(int fd, void* buf, size_t n);
ssize_t Writen(int fd, const void* buf, size_t n);
ssize_t my_read(int fd, char* ptr);
ssize_t Readline(int fd, void* buf, size_t maxlen);

#endif
```

`wrap.c`：  
```c
#include "wrap.h"

void perr_exit(const char *s) {
    perror(s);
    exit(-1);
}

int Accept(int fd, struct sockaddr* sa, socklen_t* salen) {
    int ret;
again:
    if ((ret = accept(fd, sa, salen)) < 0) {
        if ((errno == ECONNABORTED) || (errno == EINTR)) { // accept 被终止
            goto again;
        } else {
            perr_exit("accept error");
        }
    }
    return ret;
}

int Bind(int fd, const struct sockaddr* sa, socklen_t salen) {
    int ret;
    if ((ret = bind(fd, sa, salen)) < 0) {
        perr_exit("bind error");
    }
    return ret;
}

int Connect(int fd, const struct sockaddr* sa, socklen_t salen) {
    int ret;
    if ((ret = connect(fd, sa, salen)) < 0) {
        perr_exit("connect error");
    }
    return ret;
}

int Listen(int fd, int backlog) {
    int ret;
    if ((ret = listen(fd, backlog)) < 0) {
        perr_exit("listen error");
    }
    return ret;
}

int Socket(int family, int type, int protocol) {
    int ret;
    if ((ret = socket(family, type, protocol)) < 0) {
        perr_exit("socket error");
    }
    return ret;
}

ssize_t Read(int fd, void* buf, size_t nbytes) {
    ssize_t ret;
again:
    if ((ret = read(fd, buf, nbytes)) == -1) {
        if (errno == EINTR) {
            goto again;
        }
    }
    return ret;
}

ssize_t Write(int fd, const void* buf, size_t nbytes) {
    ssize_t ret;
again:
    if ((ret = write(fd, buf, nbytes)) == -1) {
        if (errno == EINTR) {
            goto again;
        }
    }
    return ret;
}

int Close(int fd) {
    int ret;
    if ((ret = close(fd)) == -1) {
        perr_exit("close error");
    }
    return ret;
}

ssize_t Readn(int fd, void* buf, size_t n) {
	size_t nleft;
	ssize_t nread;
	char *ptr;

	ptr = buf;
	nleft = n;

	while (nleft > 0) {
		if ( (nread = read(fd, ptr, nleft)) < 0) {
			if (errno == EINTR)
				nread = 0;
			else
				return -1;
		} else if (nread == 0)
			break;
		nleft -= nread;
		ptr += nread;
	}
	return n - nleft;

}

ssize_t Writen(int fd, const void* buf, size_t n) {
	size_t nleft;
	ssize_t nwritten;
	const char *ptr;

	ptr = buf;
	nleft = n;

	while (nleft > 0) {
		if ( (nwritten = write(fd, ptr, nleft)) <= 0) {
			if (nwritten < 0 && errno == EINTR)
				nwritten = 0;
			else
				return -1;
		}
		nleft -= nwritten;
		ptr += nwritten;
	}
	return n;

}

ssize_t my_read(int fd, char* ptr) {
	static int read_cnt;
	static char *read_ptr;
	static char read_buf[100];

	if (read_cnt <= 0) {
again:
		if ((read_cnt = read(fd, read_buf, sizeof(read_buf))) < 0) {
			if (errno == EINTR)
				goto again;
			return -1;	
		} else if (read_cnt == 0)
			return 0;
		read_ptr = read_buf;
	}
	read_cnt--;
	*ptr = *read_ptr++;
	return 1;

}

ssize_t Readline(int fd, void* buf, size_t maxlen) {
	ssize_t n, rc;
	char c, *ptr;
	ptr = buf;

	for (n = 1; n < maxlen; n++) {
		if ( (rc = my_read(fd, &c)) == 1) {
			*ptr++ = c;
			if (c == '\n')
				break;
		} else if (rc == 0) {
			*ptr = 0;
			return n - 1;
		} else
			return -1;
	}
	*ptr = 0;
	return n;
}

```

## 端口复用
考虑这样的情况，我启动了server主线程监听客户端连接请求，子线程们和客户端们进行通信，此时，主动关闭server进程，（然后关闭客户端），如果想要立即重启server进行监听，会被系统拒绝，原因是server的TCP连接没有完全断开之前不允许重新监听（主动和客户端断开连接，需要等待2MSL时长来完全关闭TCP连接）。

这是不合理的。因为，TCP连接没有完全断开，是因为`client_fd`（127.0.0.1:8888（本机测试的客户端））没有完全断开，而我们需要重新监听的是`listen_fd`（0.0.0.0:8888），虽然占用同一个端口，但是IP不同，`client_fd`对应的是与某个客户端通信的一个具体的IP，而listen_fd对应的是wildcard.address任意IP。

解决这个问题的方法是，使用`setsockopt()`设置socket描述符的选项`SO_REUSEADDR`为1，表示允许创建端口号相同但是IP地址不同的多个socket描述符。

函数原型：  
```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int getsockopt(int sockfd, int level, int optname,
                void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname,
                const void *optval, socklen_t optlen);
```

使用方法：  
```c
// socket();
int opt = 1;
setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, (void*)&opt, sizeof(opt))
// bind();
```
这样设置后，如果主动关闭server（然后关闭客户端），server可以立即重新启动监听，但之前的`client_fd`仍然需要等待2MSL时长。

## 半关闭 close / shutdown
TCP通信中一端关闭通信，进入FIN-WAIT-2半关闭状态，可以通过`close(client_fd)`或`shutdown(sockfd, how)`来实现。

shutdown函数原型：  
```c
 #include <sys/socket.h>

int shutdown(int sockfd, int how);
```
- 返回值：
  - 成功，返回`0`
  - 失败，返回`-1`并设置`errno`
- `sockfd`：要关闭的socket描述符；
- `how`：如何关闭
  - `SHUT_RD`，关闭读端
  - `SHUT_WR`，关闭写端
  - `SHUT_RDWR`，关闭读写

`shutdown`和`close`的区别
- `close`中止一个连接，只是减少描述符的引用计数，并不直接关闭连接，只有当描述符的引用计数为0时，才关闭；（其他进程不会被影响）
- `shutdown`不考虑描述符的引用计数，直接关闭。（其他进程会被影响）