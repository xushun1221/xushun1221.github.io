# 【Linux网络编程】03 - 多路IO转接服务器


## 多路IO转接服务器
我们之前写的服务器程序的逻辑是，主线程循环调用`accept()`，进行阻塞监听客户端的连接，当有连接到来时，主线程获得`accept()`返回的`clinet_fd`，并创建一个子线程来处理这个连接，子线程循环调用`read()`，阻塞读取`clinet_fd`的内容并进行处理。

多路IO转接服务器与之不同，该类服务器的主旨思想是，不再由服务器程序自己监听客户端连接和套接字收到的数据，取而代之的是由内核接替服务器程序监听连接，必要时通知服务器程序处理，相当于server程序同时监听多个文件描述符。主要有select、poll、epoll三种方法。

## select

### select方法的逻辑
server程序类似于老板的角色，而`select`过程类似于秘书的角色。server不再使用`accept`阻塞监听或非阻塞轮询，而是采取**响应式**的处理方法，server将listen_fs和client_fd都交给`select`，当有客户端要进行连接或有数据需要读取，`select`就会通知server进行处理，此时server才需要调用`accept read`等函数。（server还是要循环调用select）

### select函数
函数原型：  
```c
/* According to POSIX.1-2001, POSIX.1-2008 */
#include <sys/select.h>

/* According to earlier standards */
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int nfds, fd_set *readfds, fd_set *writefds,
            fd_set *exceptfds, struct timeval *timeout);

void FD_CLR(int fd, fd_set *set);
int  FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);
```
- `nfds`：监听的所有文件描述符中，最大的文件描述符+1（除了默认的012号fd，其他的从3号开始，listen_fd = 3，假如监听3个客户端，cfd1 = 4 cfd2 = 5 cfd3 = 6，`nfds` = 7）
- `readfds`：读监听 文件描述符集合
- `writefds`：写监听 文件描述符集合
- `exceptfds`：异常监听 文件描述符集合，上述三个集合都是传入传出参数，传入的是需要监听的文件描述符集合，传出的是实际进行了读写异常事件的文件描述符集合
- `timeout`
  - `>0`：设置监听超时时长
  - `NULL`：阻塞监听
  - `0`：非阻塞监听，轮询
- 返回值：
  - `>0`：所有监听集合中（3个），满足对应时间的总数
  - `0`：没有满足监听条件的事件发生
  - `-1`：错误，`errno`
- `FD_CLR`：将一个文件描述符从监听集合中移除
- `FD_ISSET`：判断一个文件描述符是否在监听集合中
- `FD_SET`：将待监听的文件描述符加入到监听集合中
- `FD_ZERO`：清空一个文件描述符集合

### select服务器实现
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

int main(int argc, char** argv) {
    // listen socket
    int listen_fd = Socket(AF_INET, SOCK_STREAM, 0);  // listen_fd == 3
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)); // **
    struct sockaddr_in serv_addr;
    bzero(&serv_addr, sizeof(serv_addr)); // **
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    Listen(listen_fd, 128);
    // client socket
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    int client_fd;
    char client_ip[1024];
    // read buffer
    char buf[4096];
    int read_bytes;

    fd_set all_rset, ret_rset; // all_rset 保存所有需要读监听的fd ret_rset 保存select返回的读fd集合
    FD_ZERO(&all_rset);
    FD_SET(listen_fd, &all_rset); // listen_fd 监听新的客户端连接 添加到读监听集合中
    
    int ret_nready; // 保存select返回的满足监听条件的fd数
    int max_fd = listen_fd; // 当前监听的最大的fd
    while (1) {
        ret_rset = all_rset;
        ret_nready = select(max_fd + 1, &ret_rset, NULL, NULL, 0); // 只监听读事件 非阻塞轮询
        if (ret_nready < 0) {
            perr_exit("select error");
        } else if (ret_nready > 0) { // 有监听事件发生
            if (FD_ISSET(listen_fd, &ret_rset)) { // listen_fd 有读事件发生 即有客户端连接请求
                client_fd = Accept(listen_fd, (struct sockaddr*)&client_addr, &client_addr_len);
                printf("client connected -- ip : %s port : %d\n", 
                    inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_ip, sizeof(client_ip)),
                    ntohs(client_addr.sin_port)
                );
                FD_SET(client_fd, &all_rset); // 新的客户端fd需要监听
                if (client_fd > max_fd) { // 更新当前监听的最大的fd
                    max_fd = client_fd;
                }
                // 如果只有一个事件 且为listen_fd的 则不需要遍历其他监听的client_fd
                if (-- ret_nready == 0) { 
                    continue;
                }
            }
            // 除了listen_fd之外 还有读事件发生 处理其他在ret_rset中的client_fd
            for (int fd = listen_fd + 1; fd <= max_fd; ++ fd) {
                if (FD_ISSET(fd, &ret_rset)) { // 该client_fd有读事件
                    read_bytes = Read(fd, (void*)&buf, sizeof(buf));
                    if (read_bytes == 0) { // client关闭了
                        Close(fd);
                        printf("client closed\n");
                        FD_CLR(fd, &all_rset); // 不需要再监听了
                    } else if (read_bytes > 0) { // 读到数据 处理
                        Write(STDOUT_FILENO, buf, read_bytes); // print
                        for (int i = 0; i < read_bytes; ++ i) {
                            buf[i] = toupper(buf[i]);
                        }
                        Write(fd, buf, read_bytes); // upper chars send to client
                    }
                    if (-- ret_nready == 0) { // 处理完了
                        break;
                    }
                }
            }
        }
    }
    Close(listen_fd);
    return 0;
}
```

### select的优缺点
缺点：
- 能够监听的客户端数量上限受文件描述符上限的限制（默认最大1024）；
- 检测满足条件（有事件发生）的fd不方便，需要自己添加逻辑进行控制。
优点：
- 可以跨平台。（Win，Linux，Unix等）；
- 不需要建立多个线程或进程就可以实现一对多的通信；
- 可以同时等待多个fd，效率比多线程多进程要高。

注，select相比于poll和epoll，性能并不弱，只是设计上有些缺陷，使用不那么方便。

### 优化检测fd的逻辑
优化了，也没完全优化。。。。
```c
// -----------------
int clients[FD_SETSIZE]; // 自定义数组clients 防止遍历1024个fd FD_SETSIZE 默认为1024
int max_idx = -1; // clients 顶端元素的下标
for (int i = 0; i < FD_SETSIZE; ++ i) {
    clients[i] = -1;
}
// ------------------

// --------------
int i;
for (i = 0; i < FD_SETSIZE; ++ i) { // 找clients里第一个未使用的位置（-1）
    if (clients[i] == -1) {
        clients[i] = client_fd;
        break;
    }
}
if (i == FD_SETSIZE) { // 到达select能监听的fd的上限
    fputs("too many clients\n", stderr);
    exit(-1);
}
if (i > max_idx) { // 保证max_idx是clients最后一个元素的下标
    max_idx = i;
}
// --------------

// -------------------------------------
for (int i = 0; i <= max_idx; ++ i) { // 检测哪个fd监听到事件
    int fd;
    if ((fd = clients[i]) < 0) {
        continue;
    }
    if (FD_ISSET(fd, &ret_rset)) {
        read_bytes = Read(fd, buf, sizeof(buf));
        if (read_bytes == 0) {
            Close(fd);
            FD_CLR(fd, &all_rset);
            clients[i] = -1;
            printf("client closed\n");
        } else if (read_bytes > 0) {
            Write(STDOUT_FILENO, buf, read_bytes); 
            for (int j = 0; j < read_bytes; ++ j) {
                buf[j] = toupper(buf[j]);
            }
            Write(fd, buf, read_bytes); 
        }
        if (-- ret_nready == 0) {
            break;
        }
    }
}
// ---------------------------------
```

## poll
`poll`和`select`逻辑基本相同，只是使用方法略有区别，`poll`更方便些。

### poll函数
函数原型：  
```c
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};
```
- `fds`：监听的文件描述符数组（`pollfd`数组），将需要监听的fd放入该数组，默认`fds[index].fd = -1`，表示空位，放入监听fd时，将`events`设为要监听的事件
- `nfds`：`fds`数组中实际监听的fd的最大下标，需要一个`max_idx`变量来保存最大下标
- `timeout`：超时时长
  - `0`：非阻塞，忙轮询
  - `-1`：阻塞
  - `>0`：超时时长
- 返回值：
- `pollfd`结构体：
  - `fd`：文件描述符
  - `events`：表示该fd需要监听的事件（取值POLLIN,POLLOUT,POLLERR对应读写异常）（位图）
  - `revents`：传出参数，调用`poll`时，如果该fd有需要处理的事件，就返回对应事件（如需要读，返回`POLLIN`）

### poll服务器实现
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <ctype.h>
#include <poll.h>
#include "wrap.h"

#define SERV_PORT 8888
#define MAX_CLIENTS 1024

int main(int argc, char** argv) {
    // listen socket
    int listen_fd = Socket(AF_INET, SOCK_STREAM, 0);  // listen_fd == 3
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)); // **
    struct sockaddr_in serv_addr;
    bzero(&serv_addr, sizeof(serv_addr)); // **
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    Listen(listen_fd, 128);
    // client socket
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    int client_fd;
    char client_ip[INET_ADDRSTRLEN];
    // read buffer
    char buf[4096];
    int read_bytes;
    // 
    struct pollfd clients[MAX_CLIENTS]; // 监听的client数组
    for (int i = 0; i < MAX_CLIENTS; ++ i) {
        clients[i].fd = -1; // 初始化 fd = -1 表示空
    }
    clients[0].fd = listen_fd; // 第一个要监听的是listen_fd
    clients[0].events = POLLIN; // 监听读事件
    int max_idx = 0; // 有效元素最大下标

    int ret_nready; // 接收poll返回值 记录满足监听事件的fd个数 

    while (1) {
        ret_nready = poll(clients, max_idx + 1, - 1); // -1 表示阻塞监听
        if (ret_nready < 0) {
            perr_exit("poll error");
        } else if (ret_nready > 0) {
            if (clients[0].revents & POLLIN) { // listen_fd有读事件 处理新连接 
                client_fd = Accept(listen_fd, (struct sockaddr*)&client_addr, &client_addr_len);
                printf("client connected -- ip : %s port : %d\n", 
                    inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_ip, sizeof(client_ip)),
                    ntohs(client_addr.sin_port)
                );
                int i;
                for (i = 1; i < MAX_CLIENTS; ++ i) { // clients[0] == listen_fd
                    if (clients[i].fd == -1) {
                        clients[i].fd = client_fd; // 找到clients中的空闲位置放新的client_fd
                        clients[i].events = POLLIN;
                        break;
                    }
                }
                if (i == MAX_CLIENTS) { // 没找到空位 说明客户端太多
                    fputs("too many clients\n", stderr);
                    exit(-1);
                }
                if (i > max_idx) { // 更新最大有效下标
                    max_idx = i;
                }
                if (-- ret_nready == 0) { // 没有更多事件 返回poll阻塞
                    continue;
                }
            }
            // 还有事件要处理
            for (int i = 1; i <= max_idx; ++ i) { // 处理需要处理的fd
                if (clients[i].fd == -1) {
                    continue;
                }
                if (clients[i].revents & POLLIN){ // 有读事件
                    read_bytes = Read(clients[i].fd, buf, sizeof(buf));
                    if (read_bytes == 0) { // 客户端关闭
                        Close(clients[i].fd);
                        clients[i].fd = -1; // 在poll中取消监听某个fd 直接在监听数组中设为-1即可
                        printf("client closed\n");
                    } else if (read_bytes > 0) {
                        Write(STDOUT_FILENO, buf, read_bytes);
                        for (int j = 0; j < read_bytes; ++ j) {
                            buf[j] = toupper(buf[j]);
                        }
                        Write(clients[i].fd, buf, read_bytes);
                    } else { // read_bytes < 0 出错
                        if (errno == ECONNRESET) { // 收到RST 连接被重置
                            Close(clients[i].fd);
                            clients[i].fd = -1;
                            printf("client reset     -- ip : %s port : %d\n", 
                                inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_ip, sizeof(client_ip)),
                                ntohs(client_addr.sin_port)
                            );
                        } else {
                            perr_exit("read error");
                        }
                    }
                }
                if (-- ret_nready == 0) {
                    break; // 不存在其他事件需要处理 退出该循环
                }
            }
        }
    }
    Close(listen_fd);
    return 0;
}
```

### poll的优缺点
优点：
- 使用自带类型的数组来保存和返回fd和事件数据，可以将监听事件集合和返回事件集合分离；
- 可以拓展监听数量上限，超过1024的限制。

缺点：
- 不能跨平台，poll属于Linux独有的特性；
- 无法直接定位监听事件的文件描述符，编码难度较大。

## 拓展监听上限
`cat /proc/sys/fs/file-max`命令，查看计算机所能打开的最大文件个数，受硬件影响。

`ulimit -a`命令，查看当前用户下的进程，默认能打开的文件描述符个数（默认为1024）。

修改方法：`sudo vi /etc/security/limits.conf`，打开该配置文件，在最后写入这两行：
- `*    soft    nofile  10000`，设置默认值，可以直接借助命令修改`ulimits -n xxx`
- `*    hard    nofile  20000`，设置上限
注销重新登录生效。


## epoll
`epoll`是Linux下多路复用IO接口`select/poll`的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率，因为它会复用文件描述符集合来传递结果而不用迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合，另一点原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。

目前`epoll`是Linux大规模并发网络程序中的热门首选模型。

`epoll`除了提供`select/poll`那种IO事件的电平触发（Level Triggered）外，还提供了边沿触发（Edge Triggered），这就使得用户空间程序有可能缓存IO状态，减少`epoll_wait`的调用，提高应用程序效率。

### epoll相关函数

#### epoll_create
epoll使用红黑树来管理监听fd，该函数创建一个监听红黑树。

```c
#include <sys/epoll.h>

int epoll_create(int size);
```
- `size`：创建的红黑树的监听节点个数（不是固定的，会动态变化，仅供内核参考）
- 返回值：
  - 成功，返回指向新创建的红黑树的根节点的fd
  - 失败，返回`-1`并设置`errno`

#### epoll_ctl
对监听红黑树进行操作。

```c
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;
struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};
```
- `epfd`：监听红黑树的fd，`epoll_create`的返回值
- `op`：对该监听红黑树进行的操作
  - `EPOLL_CTL_ADD`：添加fd到监听红黑树
  - `EPOLL_CTL_MOD`：修改fd在监听红黑树上的监听事件
  - `EPOLL_CTL_DEL`：将一个fd从监听红黑树上取下（取消监听），如果要取消监听某个fd，必须在`epoll_ctl`之后关闭fd
- `fd`：用于待监听的fd
- `event`：（`struct epoll_event`类型）事件类型
  - `events`成员：`EPOLLIN/EPOLLOUT/EPOLLERR`对应读写异常
  - `data`成员：（`epoll_data_t`联合体）
    - `fd`对应监听事件的fd
    - `void* ptr`
    - `uint32_t u32`
    - `uint64_t u64`
- 返回值：
  - 成功，返回`0`
  - 失败，返回`-1`，并设置`errno`

#### epoll_wait
阻塞等待epoll监听的IO事件。

```c
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```
- `epfd`：监听红黑树的fd，`epoll_create`的返回值
- `events`：传出参数，是一个数组，保存的是发生监听事件的fd数组（`struct epoll_event`结构体数组）
- `maxevents`：`events`数组的总数，`events`数组是自己定义的；（一半1024）
- `timeout`：超时时长
  - `0`：非阻塞
  - `-1`：阻塞
  - `>0`：超时时长
- 返回值：
  - `>0`：发生监听事件的fd的个数，用作循环处理的上限
  - `0`：没有监听事件发生
  - `-1`：出错，设置`errno`

### epoll服务器实现
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <ctype.h>
#include <sys/epoll.h>
#include "wrap.h"

#define SERV_PORT 8888
#define MAX_BUFSIZE 1024
#define MAX_CLIENTS 1024

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
    socklen_t client_addr_len = sizeof(client_addr);
    int client_fd;
    char client_ip[INET_ADDRSTRLEN];
    // read buffer
    char buf[MAX_BUFSIZE];
    int read_bytes;

    // create a RBTree for epoll
    int epfd = epoll_create(128);
    if (epfd == -1) {
        perr_exit("epoll_create error");
    }

    // epoll_event
    struct epoll_event clients_events[MAX_CLIENTS]; // returned events set from epoll_wait
    struct epoll_event ep_fdevt; // set info for new fd
    // add listen_fd to RBTree
    ep_fdevt.events = EPOLLIN;
    ep_fdevt.data.fd = listen_fd;
    int ret = epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ep_fdevt);
    if (ret == -1) {
        perr_exit("epoll_ctl error");
    }

    int ret_nready;
    while (1) { // start listen
        ret_nready = epoll_wait(epfd, clients_events, MAX_CLIENTS, -1); // epoll blocking
        if (ret_nready < 0) {
            perr_exit("epoll error");
        } else if (ret_nready > 0) {
            for (int i = 0; i < ret_nready; ++ i) { // clients_events has ret_nready elems
                if (clients_events[i].data.fd == listen_fd) { // new client
                    client_fd = Accept(listen_fd, (struct sockaddr*)&client_addr, &client_addr_len);
                    printf("client connected -- ip : %s port : %d\n", 
                        inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_ip, sizeof(client_ip)),
                        ntohs(client_addr.sin_port)
                    );
                    // add new client to RBTree
                    ep_fdevt.data.fd = client_fd;
                    ep_fdevt.events = EPOLLIN;
                    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ep_fdevt);
                    if (ret == -1) {
                        perr_exit("epoll_ctl error");
                    }
                } else { // client IO
                    read_bytes = Read(clients_events[i].data.fd, buf, sizeof(buf));
                    if (read_bytes == 0) { // client closed
                        ret = epoll_ctl(epfd, EPOLL_CTL_DEL, clients_events[i].data.fd, NULL);
                        if (ret == -1) {
                            perr_exit("epoll_ctl error");
                        }
                        Close(clients_events[i].data.fd); // have to after epoll_ctl
                        printf("client closed\n");
                    } else if (read_bytes > 0) { // read data
                        Write(STDOUT_FILENO, buf, read_bytes);
                        for (int j = 0; j < read_bytes; ++ j) {
                            buf[j] = toupper(buf[j]);
                        }
                        Write(clients_events[i].data.fd, buf, read_bytes);
                    } else {
                        // other error
                    }
                }
            }
        }
    }
    Close(epfd);
    Close(listen_fd);
    return 0;
}
```
