# 【Linux网络编程】07 - UDP-Socket编程


## UDP实现的C/S模型

### recvfrom
从对端接收数据。（UDP）

函数原型：  
```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                struct sockaddr *src_addr, socklen_t *addrlen);
```
- 返回值：
  - 失败，`-1`，`errno`
  - 成功，成功接收的字节数
  - 对端关闭，`0`
- `sockfd`：接收的套接字
- `buf`：缓冲区
- `len`：缓冲区大小
- `flags`：默认`0`
- `src_addr`：发送方的地址结构，传出参数
- `addrlen`：地址结构的大小，传入传出

### sendto
向对端发送数据。（UDP）

函数原型：  
```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
                const struct sockaddr *dest_addr, socklen_t addrlen);
```
- 返回值：
  - 失败，`-1`，`errno`
  - 成功，成功写出的字节数
- `sockfd`：发送的套接字
- `buf`：存储数据的缓冲区
- `len`：缓冲区大小
- `flags`：默认`0`
- `src_addr`：接收方的地址结构，传入参数
- `addrlen`：地址结构的大小

### 逻辑
server：
1. `listen_fd = socket(AF_INET, SOCK_DGRAM, 0)`
2. `bind()`
3. `listen()`可有可无
4. `while(1)`
   1. `recvfrom()`
   2. 处理
   3. `sendto()`
5. `close()`

client:  
1. `client_fd = socket(AF_INET, SOCK_DGRAM, 0)`
2. `sendto(server的地址结构)`
3. `recvfrom()`
4. 输出
5. `close()`

### 实现
server：  
```c
/*
@Filename : server.c
@Description : udp server test
@Datatime : 2022/06/18 14:03:53
@Author : xushun
*/
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <ctype.h>

#define SERV_PORT 8888
#define MAX_BUFSIZE 4096

int main(int argc, char** argv) {
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    int listen_fd = socket(AF_INET, SOCK_DGRAM, 0);
    bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));

    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);

    char buf[MAX_BUFSIZE];
    int recvbytes, sendbytes;
    while (1) {
        recvbytes = recvfrom(listen_fd, buf, MAX_BUFSIZE, 0, (struct sockaddr*)&client_addr, &client_addr_len);
        if (recvbytes == -1) {
            perror("recvfrom error");
            exit(-1);
        }
        write(STDOUT_FILENO, buf, recvbytes);
        for (int i = 0; i < recvbytes; ++ i) {
            buf[i] = toupper(buf[i]);
        }
        sendbytes = sendto(listen_fd, buf, recvbytes, 0, (struct sockaddr*)&client_addr, sizeof(client_addr));
        if (sendbytes == -1) {
            perror("sendto error");
            exit(-1);
        }
    }
    close(listen_fd);
    return 0;
}
```

client：  
```c
/*
@Filename : client.c
@Description : udp client test
@Datatime : 2022/06/18 13:42:37
@Author : xushun
*/
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>

#define SERV_PORT 8888
#define SERV_ADDR "127.0.0.1"
#define MAX_BUFSIZE 4096

int main(int argc, char** argv) {
    
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, SERV_ADDR, &serv_addr.sin_addr.s_addr);

    int client_fd = socket(AF_INET, SOCK_DGRAM, 0);

    char buf[MAX_BUFSIZE];
    int readbytes, sendbytes, recvbytes;
    while ((readbytes = read(STDIN_FILENO, buf, MAX_BUFSIZE)) > 0) {
        sendbytes = sendto(client_fd, buf, readbytes, 0, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
        if (sendbytes == -1) {
            perror("sendto error");
            exit(-1);
        }
        recvbytes = recvfrom(client_fd, buf, MAX_BUFSIZE, 0, NULL, 0); // NULL 不关心对端地址
        if (recvbytes == -1) {
            perror("recvfrom error");
            exit(-1);
        }
        write(STDOUT_FILENO, buf, recvbytes);
    }
    close(client_fd);
    return 0;
}
```
