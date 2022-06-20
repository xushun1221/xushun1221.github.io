# 【Linux网络编程】09 - libevent


libevent库是开源的、精简的、跨平台的、专注于网络通信的库。

## 安装libevent
这里使用libevent-2.1.8版本。

libevent官方网站 [libevent.org](https://libevent.org)

解压安装过程：  
```console
# 解压
xushun@xushun-virtual-machine:~$ tar zxvf Downloads/libevent-2.1.8-stable.tar.gz
xushun@xushun-virtual-machine:~$ cd libevent-2.1.8-stable/

# 检查安装环境 生成makefile
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable$ ./configure 
# 生成.o和可执行文件
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable$ make
# 安装 将必要的资源拷贝置系统指定目录
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable$ sudo make install

# sample目录下有编译好的程序可以用于测试
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable$ cd sample/
# 启动hello-world服务器
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable/sample$ ./hello-world

# 使用另一个终端测试
xushun@xushun-virtual-machine:~$ nc 127.1 9995
Hello, World!

# 安装完成
```

`libevent.so libevent.a`在`/usr/local/lib`目录下  
相关头文件在`/usr/local/include`目录下

## libevent的特点
- 事件驱动，高性能
- 轻量级，专注于网络
- 跨平台，支持Windows，Linux，MacOS等
- 支持多种IO多路复用技术，epoll，pool，dev/poll，select，kqueue等
- 支持IO，定时器，信号等事件

libevent是基于**事件**的**异步**通信模型。（异步通信主要依赖于**注册-回调**机制）

## Libevent 框架
使用libevent需要搭建起Libevent的框架。

1. 创建`event_base`（libevent中`event_base`是万物起源）
   `struct event_base* event_base_new(void);` 
2. 创建事件`event`或`bufferevent`
   1. 常规事件 `struct event* event_new(...);`
   2. 带缓冲区的事件 `struct bufferevent* bufferevent_socket_new(...);`
3. 将事件添加到`event_base`
   `int event_add(struct event* ev, const struct timeval* tv);`
4. 启动循环监听
   `int event_base_dispatch(struct event_base* base);`
5. 释放`event_base`
    `void event_base_free(struct event_base* base);`

## 查看系统支持哪些多路IO方法
```c
#include <stdio.h>
#include <event2/event.h>

int main(int argc, char** argv) {
    //struct event_base* base = event_new();
    const char** buf = event_get_supported_methods();
    for (int i = 0; i < 5; ++ i) {
        printf("buf[i] = %s\n", buf[i]);
    }
    return 0;
}
```

编译运行，遇到这个问题：  
```console
xushun@xushun-virtual-machine:~/LinuxNetProgramming/test_libevent$ ./test_io_multiplexing 
./test_io_multiplexing: error while loading shared libraries: libevent-2.1.so.6: cannot open shared object file: No such file or directory
```

解决方法：我们需要为动态链接器指定动态库的位置，在终端配置`~/.bashrc`中添加一行`export LD_LIBRARY_PATH=/usr/local/lib`，因为libevent存放在`/usr/local/lib`，重启终端即可。

输出：  
```console
xushun@xushun-virtual-machine:~/LinuxNetProgramming/test_libevent$ ./test_io_multiplexing 
buf[i] = epoll
buf[i] = poll
buf[i] = select
buf[i] = (null)
buf[i] = (null)
```

## 常规事件 event
```c
#include <event2/event>
```

### event_new
创建一个事件。

函数原型：  
```c
struct event* event_new(struct event_base* base, evutil_t fd, short what, event_callback_fn cb, void* arg);
typedef void(*event_callback_fn)(evutil_socket_t fd, short what, void* arg);
```
- 返回值：成功创建的`event`
- `base`：event_base
- `fd`：绑定到`event`的文件描述符
- `what`：事件类型
  - `EV_READ`，一次读事件
  - `EV_WRITE`，一次写事件
  - `EV_PERSIST`，连续触发，结合`event_base_dispatch`使用
- `cb`：监听事件就绪时调用的回调函数
- `arg`：回调函数的参数

### event_add
将事件添加到`event_base`。

函数原型：  
```c
int event_add(struct event* ev, const struct timeval* tv);
```
- 返回值： 
  - 成功，`0`
  - 失败，`-1`
- `ev`：event
- `tv`：超时时长
  - `NULL`：不会超时，一直等到事件被触发，回调函数才会被调用
  - `非0`：等待期间，检查事件是否触发，时间到，即使没有触发回调函数依旧会被调用

### evnet_del
将事件从`event_base`上摘下。

函数原型：  
```c
int event_del(struct event* ev);
```
- 返回值： 
  - 成功，`0`
  - 失败，`-1`

### event_free
释放一个event

函数原型：  
```c
int event_free(struct event* ev);
```
- 返回值： 
  - 成功，`0`
  - 失败，`-1`

## 未决和非未决状态
事件有**未决**和**非未决**两种状态。  
- 未决：有资格被处理，但尚未被处理
- 非未决：没有资格被处理

1. `ev = event_new()`，新创建的事件属于非未决
2. `event_add(ev)`，将事件添加到`event_base`上后变为未决
3. 调用`event_base_dispatch()`并且监听事件被触发，变为激活态
4. 执行回调函数（处理态），执行完成后变为非未决态
5. 如果创建新事件时，设置了`EV_PERSIST`，那么回调函数执行完后，依然为未决态
6. 使用`event_del()`可以转为非未决态

## 示例 - fifo读写
read_fifo.c：  
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <event2/event.h>
#include <sys/stat.h>
#include <fcntl.h>

void read_cb(evutil_socket_t fd, short what, void* arg) {
    char buf[4096] = {0};
    int read_bytes = read(fd, buf, sizeof(buf));
    printf("read event: %s, ", what & EV_READ ? "Yes" : "No");
    printf("data len = %d, buf = %s\n", read_bytes, buf);
    sleep(1);
    return;
}

int main(int argc, char** argv) {
    unlink("myfifo");
    mkfifo("myfifo", 0644);
    int fd = open("myfifo", O_RDONLY | O_NONBLOCK);
    struct event_base* base = event_base_new();
    struct event* ev = event_new(base, fd, EV_READ | EV_PERSIST, read_cb, NULL);
    // struct event* ev = event_new(base, fd, EV_READ, read_cb, NULL); // 使用这一行 只会读一次就退出循环
    event_add(ev, NULL);
    event_base_dispatch(base); // while (1) { epoll_wait(); ...}
    event_free(ev);
    event_base_free(base);
    close(fd);
    unlink("myfifo");
    return 0;
}
```

write_fifo.c：  
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <event2/event.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>

void write_cb(evutil_socket_t fd, short what, void* arg) {
    char buf[4096];
    static int num = 0;
    sprintf(buf, "hello %d", num ++);
    write(fd, buf, strlen(buf) + 1); // sprintf 自动加 \0
    sleep(1);
    return;
}

int main(int argc, char** argv) {
    int fd = open("myfifo", O_WRONLY);
    struct event_base* base = event_base_new();
    struct event* ev = event_new(base, fd, EV_WRITE | EV_PERSIST, write_cb, NULL);
    // struct event* ev = event_new(base, fd, EV_WRITE, write_cb, NULL);
    event_add(ev, NULL);
    event_base_dispatch(base);
    event_free(ev);
    event_base_free(base);
    close(fd);
    return 0;
}
```

测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxNetProgramming/test_libevent$ ./read_fifo 
read event: Yes, data len = 8, buf = hello 0
read event: Yes, data len = 8, buf = hello 1
read event: Yes, data len = 8, buf = hello 2
read event: Yes, data len = 8, buf = hello 3
read event: Yes, data len = 8, buf = hello 4
read event: Yes, data len = 0, buf = 
read event: Yes, data len = 0, buf = 
read event: Yes, data len = 0, buf = 
read event: Yes, data len = 0, buf = 
read event: Yes, data len = 0, buf = 
^C
```
第五次写数据后关闭写端，依然会触发读事件。

## 带缓冲区的事件 bufferevent
```c
#include <event2/bufferevent>
```

`bufferevent`有读写两个缓冲区，是借助队列实现的，先进先出。

当读缓冲区有数据时，读回调函数就会被调用，使用`bufferevent_read()`读数据。

使用`bufferevent_read()`向写缓冲区写数据，该缓冲区中存在数据时，就会自动写出到对端，然后写回调函数会被调用。

### bufferevent_socket_new
创建带缓冲区的`bufferevent`。

函数原型：  
```c
struct bufferevent* bufferevent_socket_new(struct event_base *base, evutil_socket_t fd, enum bufferevent_options options);
```
- 返回值：成功创建的`bufferevent`
- `base`：event_base
- `fd`：封装的文件描述符
- `options`：
  - `BEV_OPT_CLOSE_ON_FREE`：指定该选项释放时会关闭fd

### bufferevent_free
释放`bufferevent`。

函数原型：  
```c
void buffereventt_free(struct bufferevent* ev);
```

### bufferevent_setcb
为`bufferevent`设置回调函数。

函数原型：  
```c
void bufferevent_setcb(struct bufferevent * bufev,
                        bufferevent_data_cb readcb,
                        bufferevent_data_cb writecb,
                        bufferevent_event_cb eventcb,
                        void *cbarg );

typedef void(*bufferevent_data_cb)(struct bufferevent* bev, void* ctx); // ctx -- cbarg
typedef void(*bufferevent_event_cb)(struct bufferevent* bev, short events, void* ctx);

// 用于读写缓冲区 代替read和write
size_t bufferevent_read(struct bufferevent* bufev, void* data, size_t bufsize);
int bufferevent_write(struct bufferevent* bufev, const void* data, size_t size);
```
- `bufev`：bufferevent
- `readcb`：读缓冲回调
  - `void read_cb(struct bufferevent* bev, void* arg) { ... bufferevent_read(); ... }`
- `writecb`：写缓冲回调
- `eventcb`：其他情况，可以NULL
- `cbarg`：回调参数

- `events`：
  - `EV_EVENT_READING`：读数据时触发
  - `BEV_EVENT_WRITING`：写数据时触发
  - `BEV_EVENT_ERROR`：发生错误，调用`EVUTIL_SOCKET_ERROR()`查看错误信息
  - `BEV_EVENT_TIMEOUT`：超时
  - `BEV_EVENT_EOF`：文件结束
  - `BEV_EVENT_CONNECTED`：（重点）请求的连接过程已完成，实现客户端时可用

### bufferevent_enable (disable)
启用、禁用缓冲区。

函数原型：  
```c
void bufferevent_enable(struct bufferevent* bufev, short events); // 启用
void bufferevent_disable(struct bufferevent* bufev, short events); // 禁用
short bufferevent_get_enabled(struct bufferevent* bufev); // 获取禁用状态 &
```
- `events`：
  - `EV_READ`：读
  - `EV_WRITE`：写
  - `EV_READ | EV_WRITE`：读写

默认情况下，写缓冲开启，读缓冲关闭，`bufferevent_enable(bev, EV_READ)`开启读缓冲。

### 客户端连接 bufferevent_socket_connect
客户端连接服务器。

函数原型：  
```c
int bufferevent_socket_connect(struct bufferevent* bev, struct sockaddr* address, int addrlen);
```
- `bev`：事件对象（封装了fd）
- `address`：服务器端的地址结构
- `addrlen`：地址结构的大小

### 监听器 evconnlistener_new_bind
创建监听器。相当于`socket() bind() listen() accept()`的作用。

函数原型：  
```c
#include <event2/listener.h>

struct evconnlistener* evconnlistener_new_bind (	
        struct event_base* base,
        evconnlistener_cb cb, 
        void* ptr, 
        unsigned flags,
        int backlog,
        const struct sockaddr* sa,
        int socklen);
```
- 返回值：成功创建的监听器
- `base`：event_base
- `cb`：监听回调，接收连接后，要做的操作
- `ptr`：回调的参数
- `flags`：
  - `LEV_OPT_CLOSE_ON_FREE`：释放时，关闭底层的套接字
  - `LEV_OPT_REUSEABLE`：端口复用
  - `LEV_OPT_CLOSE_ON_FREE | LEV_OPT_REUSEABLE`
- `backlog`：`listen`的参2，`-1`表示最大值
- `sa`：服务器自己的地址结构
- `socklen`：地址结构大小

### evconnlistener_free
释放监听器。

函数原型：  
```c
#include <event2/listener.h>

void evconnlistener_free(struct evconnlistener* lev);
```

### 监听器的回调函数
客户端连接后使用的回调。

定义：  
```c
typedef void(*evconnlistener cb)(
    struct evconnlistener* listener,
    evutil_socket_t sock,
    struct sockaddr* addr,
    int len,
    void* ptr
);
```
- `listener`：`evconnlistener_new_bind()`返回值
- `sock`：用于通信的fd
- `addr`：客户端地址结构
- `len`：addr的大小
- `ptr`：外部`ptr`的值

## TCP服务器的实现流程
1. 创建`event_base`
2. 创建服务器连接监听器`evconnlistener_new_bind()`
3. 在`evconnlistener_new_bind()`的回调函数中，处理接收连接后的操作
4. 回调函数被调用说明有一个新客户端连接，会得到一个新的fd，用于和客户端通信
5. 使用`bufferevent_socket_new()`创建一个新`bufferevent`事件，将fd封装到这个事件对象中
6. 使用`bufferevent_setcb()`给这个`bufferevent`设置回调
7. 设置`bufferevent`的读写缓冲区`enable/disable`
8. 接收、发送数据`bufferevent_read() bufferevent_write()`
9. 启动监听循环
10. 释放资源

## TCP客户端的实现流程
1. 创建`event_base`
2. 使用`bufferevent_socket_new()`创建一个和服务器通信的`bufferevet`
3. 使用`bufferevent_socket_connect()`连接服务器
4. 使用`bufferevent_setcb()`设置回调函数
5. 设置`bufferevent`的读写缓冲区`enable/disable`
6. 接收、发送数据
7. 启动监听循环
8. 释放资源

## C/S模型TCP通信实现
server.c：  
```c
/*
@Filename : server.c
@Description : libevent TCP server
@Datatime : 2022/06/20 20:17:12
@Author : xushun
*/
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <event2/bufferevent.h>
#include <event2/listener.h>

#define SERV_PORT 8888
#define MAX_BUFSIZE 1024

void read_cb(struct bufferevent* bev, void* ctx) {
    char buf[MAX_BUFSIZE] = {0};
    bufferevent_read(bev, buf, sizeof(buf));
    printf("client : %s\n", buf);
    sprintf(buf, "server : got your message\n");
    bufferevent_write(bev, buf, strlen(buf) + 1);
    return;
}

void write_cb(struct bufferevent* bev, void* ctx) {
    printf("server : reply client success, callback called\n");
    return;
}

void event_cb(struct bufferevent* bev, short events, void* ctx) {
    if (events & BEV_EVENT_EOF) {
        printf("client closed\n");
    } else if (events & BEV_EVENT_ERROR) {
        printf("bufferevent error\n");
    } else {
        return;
    }
    bufferevent_free(bev);
    printf("bufferevent free\n");
    return;
}

// listener callback
void listener_cb(
    struct evconnlistener* listener, evutil_socket_t sock, 
    struct sockaddr* addr, int len, void* ptr
) {
    printf("new client connected\n");
    // get event_base from arg
    struct event_base* base = (struct event_base*)ptr;
    // new bufferevent
    struct bufferevent* bev = bufferevent_socket_new(base, sock, BEV_OPT_CLOSE_ON_FREE);
    // set callback function for buffer
    bufferevent_setcb(bev, read_cb, write_cb, event_cb, NULL);
    // enable read_buffer
    bufferevent_enable(bev, EV_READ);
    return;
}

int main(int argc, char** argv) {
    // server socket
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htons(INADDR_ANY);
    // base
    struct event_base* base = event_base_new();
    // socket bind listen accept
    struct evconnlistener* listener = evconnlistener_new_bind(
        base, listener_cb, base,
        LEV_OPT_CLOSE_ON_FREE | LEV_OPT_REUSEABLE,
        -1, (struct sockaddr*)&serv_addr, sizeof(serv_addr)
    );
    // loop
    event_base_dispatch(base);
    // free
    evconnlistener_free(listener);
    event_base_free(base);
    return 0;
}
```

client.c：  
```c
/*
@Filename : client.c
@Description : libevent TCP client
@Datatime : 2022/06/20 21:12:43
@Author : xushun
*/
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <event2/bufferevent.h>
#include <event2/listener.h>

#define SERV_PORT 8888
#define SERV_ADDR "127.0.0.1"
#define MAX_BUFSIZE 1024

void read_cb(struct bufferevent* bev, void* ctx) {
    char buf[MAX_BUFSIZE] = {0};
    bufferevent_read(bev, buf, sizeof(buf));
    // 从终端写的数据发送给server
    // 从server读取的数据显示到终端
    if (buf[0] == '#') {
        bufferevent_write(bev, buf + 1, strlen(buf));
    } else {
        printf("%s\n", buf);
    }
    return;
}

void write_cb(struct bufferevent* bev, void* ctx) {
    printf("client : send to server success, callback called\n");
    return;
}

void event_cb(struct bufferevent* bev, short events, void* ctx) {
    if (events & BEV_EVENT_EOF) {
        printf("client closed\n");
    } else if (events & BEV_EVENT_ERROR) {
        printf("bufferevent error\n");
    } else if (events & BEV_EVENT_CONNECTED){
        printf("connected to server\n");
        return;
    } else {
        return;
    }
    bufferevent_free(bev);
    printf("bufferevent free\n");
    return;
}

// read bytes from terminal
void read_terminal(evutil_socket_t fd, short what, void* arg) {
    char buf[MAX_BUFSIZE] = {0};
    int read_bytes = read(fd, buf + 1, sizeof(buf) - 1);
    buf[0] = '#';
    struct bufferevent* bev = (struct bufferevent*)arg;
    bufferevent_write(bev, buf, read_bytes + 2);
}

int main(int argc, char** argv) {
    // base
    struct event_base* base = event_base_new();
    // client fd -> bufferevent
    int client_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct bufferevent* bev = bufferevent_socket_new(base, client_fd, BEV_OPT_CLOSE_ON_FREE);
    // server info
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, SERV_ADDR, &serv_addr.sin_addr.s_addr);
    // connect to server
    bufferevent_socket_connect(bev, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    // set callback
    bufferevent_setcb(bev, read_cb, write_cb, event_cb, NULL);
    // enable read
    bufferevent_enable(bev, EV_READ);

    // event : listen bytes from terminal
    struct event* ev = event_new(base, STDOUT_FILENO, EV_READ | EV_PERSIST, read_terminal, bev);
    event_add(ev, NULL);

    // loop
    event_base_dispatch(base);

    event_base_free(base);
    event_free(ev);
    bufferevent_free(bev);
    return 0;
}
```
