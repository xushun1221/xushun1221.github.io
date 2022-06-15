# 【Linux网络编程】05 - epoll - Reactor 模型


## epoll - Reactor 反应堆模型

### 常规模型和Reactor模型的比较
- 常规模型（epoll - ET - 非阻塞IO轮询）
    1. `select() bind() listen()`初始化监听套接字`listen_fd`（监听的套接字要以struct epoll_event的形式进行封装 保存fd和事件类型）
    2. `epfd = epoll_create()`创建监听红黑树
    3. `epoll_ctl(listen_fd)`将监听套接字添加到监听树上
    4. 主循环中调用`epoll_wait()`获取就绪事件
    5. `epoll_wait()`返回就绪的事件数组`events`
    6. 遍历`events`，
       - 如果是`listen_fd`，调用`accept()`接收新客户端（设置非阻塞IO），并将其添加到监听树上
       - 如果是`client_fd`，循环读取`client_fd`直到缓冲区为空，将读到的数据处理并写回`client_fd`，如果客户端断开连接了，就关闭连接，并从监听树上摘下该`client_fd`
- Reactor模型（epoll - ET - 非阻塞IO轮询 + 回调函数）
  1. `epfd = epoll_create()`创建监听红黑树
  2. 自定义`struct myevent_s`结构体类型，用于描述监听的文件描述符信息（处理该fd对应事件的回调函数），该结构的指针存放于`struct epoll_event.data.ptr`
  3. `struct myevent_s g_events[MAX_CLIENTS + 1]`数组用于存储监听的fd信息
  4. `select() bind() listen()`初始化监听套接字`listen_fd`，它位于`g_events[MAX_CLIENTS]`，`epoll_ctl`将其添加到监听树上
  5. 主循环中调用`epoll_wait()`获取就绪事件数组`events`
  6. 遍历`events`，对每个就绪事件调用其指定的处理函数
     - 如果是`listen_fd`，接收新客户端，设置非阻塞IO，添加到监听树上
     - 如果是`client_fd`，满足读事件，从fd读取数据，进行处理，存到该fd对应的`myevent_s`的buf中，（不能立即写回fd，因为有可能不满足写数据的条件），将fd从监听树上摘下，重新设置监听事件为写事件，再重新添加到监听树上（等待内核通知可写）
     - 如果是`listen_fd`，满足写事件，向fd写数据，将fd从监听树上摘下，重新设置监听事件为读事件，再重新添加到监听树上

### 示例 - 基于非阻塞IO事件驱动的反应堆模型（libevent的部分源码实现）
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <fcntl.h>
#include <time.h>

#define MAX_EVENTS 1024     // 监听数量上限
#define MAX_BUFSIZE 4096    // buf大小
#define SERV_PORT 8888      // 监听端口

void recvdata(int fd, int events, void* arg);
void senddata(int fd, int events, void* arg);

// 描述监听文件描述符相关信息  epoll_event.data.ptr 指向该结构体
struct myevent_s {
    int fd;                 // 要监听的文件描述符
    int events;             // 对应的监听事件 EPOLLIN EPOLLOUT
    void* arg;              // 泛型参数
    void (*call_back)(int fd, int events, void* arg);
                            // 回调函数 由fd类型和监听事件决定回调函数的类型
    int status;             // 是否正在被监听 1-在监听树上 0-不在监听树上
    char buf[MAX_BUFSIZE];
    int len;
    long last_active;       // 记录上次活动时间 即加入监听树的时间
};

int g_epfd; // 全局监听树
struct myevent_s g_events[MAX_EVENTS + 1]; // 全局fd描述信息数组 保存需要监听的fd和相关信息

// 初始化struct myevent_s成员变量
void eventset(struct myevent_s* ev, int fd, void (*call_back)(int, int, void*), void* arg) {
    ev -> fd = fd;
    ev -> call_back = call_back;
    ev -> events = 0;
    ev -> arg = arg;
    ev -> status = 0;
    if (call_back != senddata) {
        memset(ev -> buf, 0, sizeof(ev -> buf));
        ev -> len = 0;
    }
    ev -> last_active = time(NULL); // 初始值为当前时间
    return;
}

// 向监听树中添加一个文件描述符
void eventadd(int epfd, int events, struct myevent_s* ev) {
    struct epoll_event epev = {0, {0}}; // 初始化一个epoll_event (events = 0, data = {0})
    // 给这个fd添加信息
    epev.data.ptr = ev;
    epev.events = ev -> events = events;

    int op;
    if (ev -> status == 0) {
        op = EPOLL_CTL_ADD;
        ev -> status = 1; // 修改监听状态
    }

    if (epoll_ctl(epfd, op, ev -> fd, &epev) < 0) {
        printf("event add failed [fd = %d], events[%d]\n", ev -> fd, events);
    } else {
        printf("event add OK [fd = %d], op = %d, events[%0X]\n", ev -> fd, op, events);
    }
    return;
}

// 从监听树中删除一个文件描述符
void eventdel(int epfd, struct myevent_s* ev) {
    struct epoll_event epv = {0, {0}};

    if (ev -> status != 1) {
        return;
    }
    
    // epv.data.ptr = ev;
    epv.data.ptr = NULL;
    ev -> status = 0; // 修改状态
    epoll_ctl(epfd, EPOLL_CTL_DEL, ev -> fd, &epv); // 从监听树上将ev -> fd 摘下

    return;
}

void recvdata(int fd, int events, void* arg) {
    struct myevent_s *ev = (struct myevent_s*)arg; // arg参数是fd的描述信息myevent_s
    
    int len;
    len = recv(fd, ev -> buf, sizeof(ev -> buf), 0); // 读文件描述符 数据存入myevent_s 成员buf中

    eventdel(g_epfd, ev); // 该节点从监听树上摘下

    if (len > 0) {
        ev -> len = len;
        ev -> buf[len] = '\0'; // 添加字符串结束标记
        printf("C[%d] : %s\n", fd, ev -> buf);

        eventset(ev, fd, senddata, ev); // 将该fd的回调函数设置为senddata
        eventadd(g_epfd, EPOLLOUT, ev); // 将fd加入监听树 监听写事件
    } else if (len == 0) {
        close(ev -> fd);
        // ev - g_events 地址相减得到偏移元素位置
        printf("[fd=%d] pos[%ld], closed\n", fd, ev - g_events);
    } else {
        close(ev -> fd);
        printf("recv[fd=%d] error[%d]:%s\n", fd, errno, strerror(errno));
    }
    return;
}

void senddata(int fd, int events, void* arg) {
    struct myevent_s* ev = (struct myevent_s*)arg;
    
    int len;
    len = send(fd, ev -> buf, ev -> len, 0); // 直接将数据写回给客户端 未作处理 

    eventdel(g_epfd, ev); // 从监听树中摘下fd

    if (len > 0) {
        printf("send[fd = %d], [%d]%s\n", fd, len, ev -> buf);
        eventset(ev, fd, recvdata, ev); // 将该fd的回调函数改为 recvdata
        eventadd(g_epfd, EPOLLIN, ev);  // 重新将其添加到监听树上 改为监听读事件
    } else {
        close(ev -> fd); // 关闭连接
        printf("send[fd = %d] error %s\n", fd, strerror(errno));
    }
    return;
}

// listen_fd的回调函数 与客户端建立连接
void acceptconn(int lfd, int events, void* arg) {
    struct sockaddr_in cin;
    socklen_t len = sizeof(cin);
    int cfd, i;

    if ((cfd = accept(lfd, (struct sockaddr*)&cin, &len)) == -1) {
        if (errno != EAGAIN && errno != EINTR) {

        } 
        printf("%s : accept, %s\n", __func__, strerror(errno));
        return;
    }

    do {
        for (i = 0; i < MAX_EVENTS; ++ i) { // 从全局g_events中找一个空闲元素
            if (g_events[i].status == 0) {
                break; // 跳出for
            }
        }
        if (i == MAX_EVENTS) {
            printf("%s : max connect limit[%d]\n", __func__, MAX_EVENTS);
            break; // 跳出do_while
        }

        int flag = 0;
        if ((flag = fcntl(cfd, F_SETFL, O_NONBLOCK)) < 0) { // cfd 设置为nonblocking
            printf("%s : fcntl nonblcoking failed, %s\n", __func__, strerror(errno));
            break;
        }

        // 给cfd设置一个myevent_s结构体 回调函数设置为recvdata
        eventset(&g_events[i], cfd, recvdata, &g_events[i]);
        eventadd(g_epfd, EPOLLIN, &g_events[i]); // cfd添加到监听树中 监听读事件
    } while(0);

    printf("new connect [%s:%d][time:%ld], pos[%d]\n",
        inet_ntoa(cin.sin_addr), ntohs(cin.sin_port), g_events[i].last_active, i
    );
    return;
}

// 初始化listen_fd
void initlistensocket(int epfd, short port) {
    struct sockaddr_in sin;

    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    fcntl(lfd, F_SETFL, O_NONBLOCK); // listen_fd 设为非阻塞

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = htonl(INADDR_ANY);
    sin.sin_port = htons(port);

    bind(lfd, (struct sockaddr*)&sin, sizeof(sin));
    listen(lfd, 20);

    // void eventset(struct myevent_s* ev, int fd, void (*call_back)(int, int, void*), void* arg);
    eventset(&g_events[MAX_EVENTS], lfd, acceptconn, &g_events[MAX_EVENTS]);
    // void eventadd(int epfd, int events, struct myevent_s* ev);
    eventadd(epfd, EPOLLIN, &g_events[MAX_EVENTS]);

    return;
}



int main(int argc, char** argv) {
    unsigned short port = SERV_PORT;
    
    g_epfd = epoll_create(MAX_EVENTS + 1); // 创建全局监听树
    if (g_epfd <= 0) {
        printf("create epfd in %s err %s\n", __func__, strerror(errno));
    }

    initlistensocket(g_epfd, port); // 初始化监听socket

    struct epoll_event events[MAX_EVENTS + 1];  // 保存已满足就绪事件的文件描述符数组

    printf("server running : port[%d]\n", port);

    int checkpos = 0;
    while (1) {
        // 超时验证 每次测试100个连接，不测试listen_fd 当客户端60秒内没有和服务器通信 则关闭此客户端连接
        long now = time(NULL); // 当前时间
        for (int i = 0; i < 100; ++ i, ++ checkpos) { // 每次循环检测100个 使用checkpos控制检测对象
            if (checkpos == MAX_EVENTS) {
                checkpos = 0;
            }
            if (g_events[checkpos].status != 1) { // 不在监听树上
                continue;
            }

            long duration = now - g_events[checkpos].last_active; // 距离上次活跃过去的时间

            if (duration >= 60) {
                close(g_events[checkpos].fd);   // 关闭与该客户端的连接
                printf("fd[%d] timeout\n", g_events[checkpos].fd);
                eventdel(g_epfd, &g_events[checkpos]);  // 将客户端从监听树上摘下
            }
        }
        //
 
        // 监听树将就绪的事件的文件描述符添加至events数组中返回 1s内没有事件就绪 返回0
        int nfd = epoll_wait(g_epfd, events, MAX_EVENTS + 1, 1000);
        if (nfd < 0) {
            printf("epoll_wait error, exit\n");
            break;
        }

        for (int i = 0; i < nfd; ++ i) {
            // 使用自定义的struct myevent_s类型指针 接收联合体data的void* ptr成员
            struct myevent_s* ev = (struct myevent_s*)events[i].data.ptr;

            if ((events[i].events & EPOLLIN) && (ev -> events & EPOLLIN)) { // 读事件就绪
                ev -> call_back(ev -> fd, events[i].events, ev -> arg);
            }
            if ((events[i].events & EPOLLOUT) && (ev -> events & EPOLLOUT)) { // 写事件就绪
                ev -> call_back(ev -> fd, events[i].events, ev -> arg);
            }
        }
    }
    return 0;
}
```

### 重写上述代码
上面的代码可能是从libevent中截取的部分，有些代码对实现这个简单逻辑没有意义，下面更简洁的代码重写一遍。

```c
/*
@Filename : my_reactor.c
@Description : epoll - Reactor server 不怎么重要的错误检查就不写了
@Datatime : 2022/06/15 16:55:08
@Author : xushun
*/
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <fcntl.h>
#include <time.h>

#define MAX_EVENTS 1000
#define MAX_BUFSIZE 4096
#define SERV_PORT 8888

struct myevent_s {
    int fd;
    int events;
    void* arg;
    void (*call_back)(int fd, int events, void* arg);
    int status;
    char buf[MAX_BUFSIZE];
    int buf_len;
    long last_active;
};

int g_epfd;
struct myevent_s g_events[MAX_EVENTS + 1];

void myeventset(struct myevent_s* ev, int fd, void (*call_back)(int fd, int events, void* arg), void* arg, int keep_buf) {
    ev -> fd = fd;
    ev -> call_back = call_back;
    ev -> events = 0;
    ev -> arg = arg;
    ev -> status = 0;
    ev -> last_active = time(NULL);
    if (keep_buf == 0) {
        memset(ev -> buf, 0, sizeof(ev -> buf));
        ev -> buf_len = 0;
    }
    return;
}

void eventadd(int epfd, int events, struct myevent_s* ev) {
    struct epoll_event epev;
    epev.data.ptr = ev;
    epev.events = events;
    ev -> events = events;
    int op;
    if (ev -> status == 0) {
        ev -> status = 1;
        op = EPOLL_CTL_ADD;
    }
    if (epoll_ctl(epfd, op, ev -> fd, &epev) < 0) {
        printf("event add failed  [fd=%d], [op=%d], [event=%d]\n", ev -> fd, op, events);
    } else {
        printf("event add success [fd=%d], [op=%d], [event=%d]\n", ev -> fd, op, events);
    }
    return;
}

void eventdel(int epfd, struct myevent_s* ev) {
    int op;
    if (ev -> status == 1) {
        ev -> status = 0;
        op = EPOLL_CTL_DEL;    
    }
    if (epoll_ctl(epfd, op, ev -> fd, NULL) < 0) {
        printf("event del failed  [fd=%d], [op=%d]\n", ev -> fd, op);
    } else {
        printf("event del success [fd=%d], [op=%d]\n", ev -> fd, op);
    }
    return;
}

void senddata(int fd, int events, void* arg);
void recvdata(int fd, int events, void* arg) {
    struct myevent_s* ev = (struct myevent_s*)arg;
    int recv_bytes = recv(fd, ev -> buf, sizeof(ev -> buf), 0);
    eventdel(g_epfd, ev);
    if (recv_bytes > 0) {
        ev -> buf_len = recv_bytes;
        ev -> buf[recv_bytes] = '\0';
        printf("recv[fd=%d], [%d]%s\n", fd, recv_bytes, ev -> buf);
        myeventset(ev, fd, senddata, ev, 1);
        eventadd(g_epfd, EPOLLOUT, ev);
    } else if (recv_bytes == 0) {
        printf("[fd=%d], [pos=%d] closed\n", fd, (int)(ev - g_events));
        close(fd);
    } else {
        printf("recv[fd=%d] error %s\n", fd, strerror(errno));
    }
    return;
}

void senddata(int fd, int events, void* arg) {
    struct myevent_s* ev = (struct myevent_s*)arg;
    int send_bytes = send(fd, ev -> buf, ev -> buf_len, 0);
    eventdel(g_epfd, ev);
    if (send_bytes > 0) {
        printf("send[fd=%d], [%d]%s\n", fd, send_bytes, ev -> buf);
        myeventset(ev, fd, recvdata, ev, 0);
        eventadd(g_epfd, EPOLLIN, ev);
    } else {
        printf("send[fd=%d] error %s\n", fd, strerror(errno));
        close(fd);
    }
}

void acceptconn(int listen_fd, int events, void* arg) {
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr_len);
    int client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client_addr_len);
    int i;
    for (i = 0; i < MAX_EVENTS; ++ i) {
        if (g_events[i].status == 0) {
            break;
        }
    }
    if (i == MAX_EVENTS) {
        printf("cannot accept more clients [fd=%d]\n", client_fd);
        return;
    }
    int flag = fcntl(client_fd, F_GETFL);
    flag |= O_NONBLOCK;
    fcntl(client_fd, F_SETFL, flag);
    myeventset(&g_events[i], client_fd, recvdata, &g_events[i], 0);
    eventadd(g_epfd, EPOLLIN, &g_events[i]);
    printf("new connect [%s:%d], [time=%ld], [pos=%d]\n",
        inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port), g_events[i].last_active, i
    );
    return;
}

void initlistensocket(int epfd, unsigned short nport) {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = nport;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    listen(listen_fd, 20);

    myeventset(&g_events[MAX_EVENTS], listen_fd, acceptconn, &g_events[MAX_EVENTS], 0);
    eventadd(epfd, EPOLLIN, &g_events[MAX_EVENTS]);
    return;
}


int main(int argc, char** argv) {
    unsigned short nport = htons(SERV_PORT);
    if (argc == 2) {
        nport = htons(atoi(argv[1]));
    }
    g_epfd = epoll_create(MAX_EVENTS + 1);
    initlistensocket(g_epfd, nport);
    struct epoll_event events[MAX_EVENTS + 1];
    printf("server running : [port=%d]\n", ntohs(nport));

    int checkpos = 0;
    while (1) {
        long now = time(NULL);
        for (int i = 0; i < 100; ++ i, ++ checkpos) {
            if (checkpos == MAX_EVENTS) {
                checkpos = 0;
            }
            if (g_events[checkpos].status != 1) {
                continue;
            }
            long duration = now - g_events[checkpos].last_active;
            if (duration >= 60) {
                eventdel(g_epfd, &g_events[checkpos]);
                close(g_events[checkpos].fd);
                printf("[fd=%d] timeout\n", g_events[checkpos].fd);
            }
        }

        int nready = epoll_wait(g_epfd, events, MAX_EVENTS + 1, 1000);
        if (nready < 0) {
            printf("epoll_wait error\n");
            return -1;
        }

        for (int i = 0; i < nready; ++ i) {
            struct myevent_s* ev = (struct myevent_s*)events[i].data.ptr;
            if ((events[i].events & EPOLLIN) && (ev -> events & EPOLLIN)) {
                ev -> call_back(ev -> fd, events[i].events, ev -> arg);
            }
            if ((events[i].events & EPOLLOUT) && (ev -> events & EPOLLOUT)) {
                ev -> call_back(ev -> fd, events[i].events, ev -> arg);
            }
        }
    }
    return 0;
}
```
