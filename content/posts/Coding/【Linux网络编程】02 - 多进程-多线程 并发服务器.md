---
title: 【Linux网络编程】02 - 多进程-多线程 并发服务器
date: 2022-06-10
tags: [Linux, network]
categories: [Coding]
---

## 多进程并发服务器
父进程创建监听socket监听客户端连接，每次accept一个连接，就fork创建一个子进程来处理这个连接，然后父进程继续监听其他连接请求。

1. `socket()`创建监听套接字 listen_fd
2. `bind()`	绑定地址结构;
3. `listen()`	
4. 循环，accept到一个连接，就fork创建子进程，子进程跳出循环，关闭listen_fd，父进程继续循环，关闭accept到的客户端连接；
5. 子进程：完成业务逻辑
6. 父进程：注册信号捕捉函数：SIGCHLD，在回调函数中，完成子进程回收（阻塞回收的话就不能继续监听连接）

### 实现
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <ctype.h>
#include <signal.h>
#include <sys/wait.h>
#include "wrap.h"

#define SERV_PORT 8888

void catch_child(int signum) {
    while (waitpid(-1, NULL, WNOHANG) > 0) { // 回收到子进程
        
    }
    return;
}

int main(int argc, char** argv) {
    // 监听socket
    int listen_fd = Socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    Listen(listen_fd, 128);
    // 注册捕捉函数
    struct sigaction act, oldact;
    act.sa_handler = catch_child;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    if (sigaction(SIGCHLD, &act, &oldact) != 0) {
        perr_exit("sigaction error");
    }
    // 客户端socket
    int client_fd;
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);

    pid_t pid;

    while (1) {
        client_fd = Accept(listen_fd, (struct sockaddr*)&client_addr, &client_addr_len);
        pid = fork();
        if (pid < 0) {
            perr_exit("fork error");
        } else if (pid == 0) { // child process
            close(listen_fd);
            break; // 子进程不监听 跳出循环 执行业务逻辑
        } else { // parent process
            close(client_fd); // 父进程不和客户端通信 关闭client_fd 继续监听
        }
    }

    if (pid == 0) { // 子进程逻辑
        char client_IP[1024];
        printf("client connected -- ip : %s port : %d\n", 
            inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_IP, sizeof(client_IP)),
            ntohs(client_addr.sin_port)
        );
        char buf[4096];
        int read_bytes;
        while (1) {
            read_bytes = Read(client_fd, buf, sizeof(buf));
            if (read_bytes == 0) { // read_bytes==0 意思是读到末尾（客户端关闭了）
                close(client_fd);
                printf("client shutdown  -- ip : %s port : %d\n", 
                    inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_IP, sizeof(client_IP)),
                    ntohs(client_addr.sin_port)
                );
                exit(1);
            }
            Write(STDOUT_FILENO, buf, read_bytes);
            for (int i = 0; i < read_bytes; ++ i) {
                buf[i] = toupper(buf[i]);
            }
            Write(client_fd, buf, read_bytes);
        }
    }
    return 0;
}
```

## 多线程并发服务器
和多进程并发服务器类似，每accept一个连接，就创建一个线程去处理它，主线程继续监听其他连接请求。

1. `socket()`创建监听套接字 listen_fd
2. `bind()`绑定地址结构
3. `listen()`		
4. 循环，每accept一个连接，就创建一个线程，（设置线程分离，如果要获得线程退出状态，可以创建一个线程专门负责回收其他线程），主线程持续监听
5. 子线程：完成业务逻辑（子线程不能关闭listen_fd，因为主线程要用）


### 实现
注意编译时要加上`-pthread`选项。

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <ctype.h>
#include "wrap.h"

#define SERV_PORT 8888

struct client_info { // 描述客户端连接信息
    struct sockaddr_in client_addr;
    int client_fd;
};

void* thread_work(void* arg) {
    // 接收客户端连接的信息
    struct client_info* client_p = (struct client_info*)arg;
    struct sockaddr_in client_addr = client_p -> client_addr;
    int client_fd = client_p -> client_fd;

    char client_IP[1024];
    printf("client connected -- ip : %s port : %d\n", 
        inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_IP, sizeof(client_IP)),
        ntohs(client_addr.sin_port)
    );
    char buf[4096];
    int read_bytes;
    while (1) {
        read_bytes = Read(client_fd, buf, sizeof(buf));
        if (read_bytes == 0) { // read_bytes==0 意思是读到末尾（客户端关闭了）
            close(client_fd);
            printf("client shutdown  -- ip : %s port : %d\n", 
                inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_IP, sizeof(client_IP)),
                ntohs(client_addr.sin_port)
            );
            pthread_exit(NULL);
        }
        Write(STDOUT_FILENO, buf, read_bytes);
        for (int i = 0; i < read_bytes; ++ i) {
            buf[i] = toupper(buf[i]);
        }
        Write(client_fd, buf, read_bytes);
    }
    return NULL;
}

int main(int argc, char** argv) {
    // 监听socket
    int listen_fd = Socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    Listen(listen_fd, 128);    
    // 客户端socket
    int client_fd;
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    // 客户端连接信息记录
    struct client_info clients[1024];
    int index = 0;

    pthread_t tid;
    while (1) {
        client_fd = Accept(listen_fd, (struct sockaddr*)&client_addr, &client_addr_len);
        // 保存客户端连接信息
        clients[index].client_addr = client_addr;
        clients[index].client_fd = client_fd;
        ++ index;
        // 创建子线程
        int ret = pthread_create(&tid, NULL, thread_work, (void*)&clients[index - 1]);
        if (ret != 0) {
            fprintf(stderr, "pthread_create error : %s\n", strerror(ret));
            exit(-1);
        }
        ret = pthread_detach(tid); // 设置线程分离
        if (ret != 0) {
            fprintf(stderr, "pthread_detach error : %s\n", strerror(ret));
            exit(-1);
        }
    }
    return 0;
}
```
