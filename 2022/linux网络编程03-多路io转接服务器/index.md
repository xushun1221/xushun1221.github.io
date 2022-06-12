# 【Linux网络编程】03 - 多路IO转接服务器


## 多路IO转接服务器
我们之前写的服务器程序的逻辑是，主线程循环调用`accept()`，进行阻塞监听客户端的连接，当有连接到来时，主线程获得`accept()`返回的`clinet_fd`，并创建一个子线程来处理这个连接，子线程循环调用`read()`，阻塞读取`clinet_fd`的内容并进行处理。

多路IO转接服务器与之不同，该类服务器的主旨思想是，不再由服务器程序自己监听客户端连接和套接字收到的数据，取而代之的是由内核接替服务器程序监听连接，必要时通知服务器程序处理。主要有select、poll、epoll三种方法。

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
                        printf("client closed    -- ip : %s port : %d\n", 
                            inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_ip, sizeof(client_ip)),
                            ntohs(client_addr.sin_port)
                        );
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
            printf("client closed    -- ip : %s port : %d\n", 
                inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_ip, sizeof(client_ip)),
                ntohs(client_addr.sin_port)
            );
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
