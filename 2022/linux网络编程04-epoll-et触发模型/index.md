# 【Linux网络编程】04 - epoll - ET触发模型


## epoll事件触发模型 - LT & ET
epoll事件有两种模型：
1. Edge Triggered (ET)：边沿触发模型，只有数据到来时才会导致`epoll_wait`返回，无论缓冲区内是否还有数据；（缓冲区还有数据未读，只有新的数据到来时才会触发`epoll_wait`返回）
2. Level Triggered (LT)：水平触发模型，只要缓冲区中有数据就会触发`epoll_wait`返回。

### 实例程序 - 基于管道的epoll-ET模型
看一个实例程序，程序中子进程每隔五秒通过pipe给主进程发送10个字节的数据（`aaaa\\nbbbb\\n`，五秒后发送`cccc\\ndddd\\n`）。主进程使用epoll监听pipe_fd，每次从管道中读取5个字节的数据。

程序代码：  
```c
// 头文件略
#define MAX_BUFSIZE 10

int main(int argc, char** argv) {
    int pipe_fd[2];
    pipe(pipe_fd);
    char buf[MAX_BUFSIZE];
    pid_t pid = fork();
    if (pid == 0) { // child
        Close(pipe_fd[0]); // close read end
        char ch = 'a';
        while (1) { // loop  write 10 chars to parent per 5 secs
            int i;
            for (i = 0; i < MAX_BUFSIZE / 2; ++ i) {
                buf[i] = ch;
            }
            buf[i - 1] = '\n';
            ++ ch;
            for (; i < MAX_BUFSIZE; ++ i) {
                buf[i] = ch;
            }
            buf[i - 1] = '\n';
            ++ ch;
            Write(pipe_fd[1], buf, MAX_BUFSIZE);
            sleep(5);
        }
        Close(pipe_fd[1]);
    } else if (pid > 0) { // parent
        Close(pipe_fd[1]); // close write end
        struct epoll_event event;
        struct epoll_event events[10];
        int epfd = epoll_create(10);
        event.data.fd = pipe_fd[0]; // add read end to RBTree
        // -------------------------------------
        // LT
        // event.events = EPOLLIN;
        // ET
        event.events = EPOLLIN | EPOLLET;
        // -----------------------------------------
        epoll_ctl(epfd, EPOLL_CTL_ADD, pipe_fd[0], &event);

        while (1) { // read 5 chars from child
            int ret_nready = epoll_wait(epfd, events, 10, -1);
            if (events[0].data.fd == pipe_fd[0]) {
                int read_bytes =  Read(pipe_fd[0], buf, MAX_BUFSIZE / 2);
                Write(STDOUT_FILENO, buf, read_bytes);
            }
        }
        Close(pipe_fd[0]);
        Close(epfd);
    }
    return 0;
}
```
epoll在默认情况下使用LT水平触发模型，使用ET触发模型需要手动设置，例如`event.events = EPOLLIN | EPOLLET;`我们将管道读端设置为ET触发。 

开始时，子进程写了10个字节，主进程读了5个`aaaa\\n`，然后就阻塞在这了，因为在这5秒中，尽管缓冲区中还有5个字节，但是也要到5秒后，子进程发送下一批数据时，才会触发`epoll_wait`返回。

如果使用LT触发模式，主进程会瞬间读完这10个字节`aaaa\\nbbbb\\n`，因为5个字节读完后，缓冲区还有数据，此时`epoll_wait`会持续触发。

### 基于网络socket的epoll-ET模型（blocking IO）
把上面的例子用C/S模型写一下。（真正使用epoll-ET模型时不可以使用阻塞IO）

client：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include "wrap.h"

#define SERV_PORT 8888
#define SERV_IP "127.0.0.1"
#define MAX_BUFSIZE 10

int main(int argc, char** argv) {
    int client_fd = Socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in serv_addr;
    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, SERV_IP, &serv_addr.sin_addr.s_addr);
    Connect(client_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    int read_bytes;
    char buf[MAX_BUFSIZE];
    char ch = 'a';
    while (1) {
        int i;
        for (i = 0; i < MAX_BUFSIZE / 2; ++ i) {
            buf[i] = ch;
        }
        buf[i - 1] = '\n';
        ++ ch;
        for (; i < MAX_BUFSIZE; ++ i) {
            buf[i] = ch;
        }
        buf[i - 1] = '\n';
        ++ch;
        Write(client_fd, buf, MAX_BUFSIZE);
        sleep(5);
    }
    Close(client_fd);
    return 0;
}
```

server：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <sys/epoll.h>
#include <ctype.h>
#include "wrap.h"

#define SERV_PORT 8888
#define MAX_BUFSIZE 10
#define MAX_CLIENTS 10

int main(int argc, char** argv) {
    // listen socket
    int listen_fd = Socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    struct sockaddr_in serv_addr;
    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    Listen(listen_fd, 128);
    // client socket
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr_len);
    int client_fd;
    char client_ip[INET_ADDRSTRLEN];
    // read buffer
    char buf[MAX_BUFSIZE];
    int read_bytes;

    struct epoll_event event;
    struct epoll_event events[MAX_CLIENTS];
    int epfd = epoll_create(10);
    // just one client for test
    client_fd = Accept(listen_fd, (struct sockaddr*)&client_addr, &client_addr_len);
    printf("client connected\n");
    event.data.fd = client_fd;
    event.events = EPOLLIN | EPOLLET;
    epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &event);

    int ret_nready;
    while (1) {
        ret_nready = epoll_wait(epfd, events, MAX_CLIENTS, -1);
        if (events[0].data.fd == client_fd) {
            read_bytes = Readn(client_fd, buf, MAX_BUFSIZE / 2);
            Write(STDOUT_FILENO, buf, read_bytes);
        }
    }
    Close(listen_fd);
    return 0;
}
```

### 基于网络socket的epoll-ET模型（non-blocking IO）
上面的代码在对socket进行IO操作时，使用的是阻塞IO（blocking IO），在epoll-ET模型下，不能使用阻塞IO（原因后面再分析），下面把阻塞改为非阻塞。

改为非阻塞IO，并用循环来读取缓冲区中的数据，可以一次将缓冲区读空。

server：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <sys/epoll.h>
#include <ctype.h>
#include <fcntl.h>
#include "wrap.h"

#define SERV_PORT 8888
#define MAX_BUFSIZE 10
#define MAX_CLIENTS 10

int main(int argc, char** argv) {
    // listen socket
    int listen_fd = Socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    struct sockaddr_in serv_addr;
    bzero(&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    Listen(listen_fd, 128);
    // client socket
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr_len);
    int client_fd;
    char client_ip[INET_ADDRSTRLEN];
    // read buffer
    char buf[MAX_BUFSIZE];
    int read_bytes;

    struct epoll_event event;
    struct epoll_event events[MAX_CLIENTS];
    int epfd = epoll_create(10);
    // just one client for test
    client_fd = Accept(listen_fd, (struct sockaddr*)&client_addr, &client_addr_len);
    printf("client connected\n");
    // ------------------------- // non-blocking
    int flag = fcntl(listen_fd, F_GETFL);
    flag |= O_NONBLOCK;
    fcntl(listen_fd, F_SETFL, flag);
    // -------------------------
    event.data.fd = client_fd;
    event.events = EPOLLIN | EPOLLET;
    epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &event);

    int ret_nready;
    while (1) {
        ret_nready = epoll_wait(epfd, events, MAX_CLIENTS, -1);
        if (events[0].data.fd == client_fd) {
            // ---------------------------
            while ((read_bytes = Readn(client_fd, buf, MAX_BUFSIZE / 2)) > 0) {
                Write(STDOUT_FILENO, buf, read_bytes);
            }
            // ---------------------------
        }
    }
    Close(listen_fd);
    return 0;
}
```

### epoll-ET模型为何一定要使用非阻塞IO？
LT模式是epoll的缺省工作方式，并且同时支持阻塞和非阻塞IO，在这种做法中，内核告诉你一个fd是否就绪，然后你可以对这个fd进行IO操作。如果你不进行任何操作（或者没完全处理完），内核还是会继续通知你，所以这种模式出错的概率要小一些，传统的select、poll都是这种模型的代表。

ET模式是epoll的高速工作方式，它**只支持非阻塞IO**，在这种模式下，当fd从未就绪变为就绪时，内核通过epoll告诉你，然后内核会假设你已经知道fd已就绪，无论你是否对fd进行了处理，内核都不会再对该事件进行通知，知道该fd的下一个事件到来。

为何epoll-ET模型一定要使用非阻塞IO呢？

假设，使用epoll-ET-blockingIO，某个客户端向server发送数据，epoll通知server进行处理，server循环读取该client_fd，直到没有数据可读，此时server会阻塞在最后一次的读操作上，那么server就无法调用epoll_wait来获得其他fd的事件通知。

假设使用epoll-ET-nonblockingIO，server循环读取client_fd，如果没有数据可读，该读操作会返回`EAGAIN`错误，此时中止循环读取即可。server会在主循环中调用epoll_wait处理其他fd的事件通知。
