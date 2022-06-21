---
title: 【Linux网络编程】10 - HTTPServer
date: 2022-06-21
tags: [Linux, network, libevent]
categories: [Coding]
---

## 通信流程分析
1. 获取 http协议的第一行。
2. 从首行中拆分  GET、文件名、协议版本。 获取用户请求的文件名。
3. 判断文件是否存在。 stat()
4. 判断是文件还是目录。
5. 是文件-- open -- read -- 写回给浏览器
6. 写 http 应答协议头
7. 写文件数据

## epoll实现
main.c：  
```c
/*
@Filename : main.c
@Description : http server by epoll
@Datatime : 2022/06/21 15:53:08
@Author : xushun
*/
#include "server.h"

int main(int argc, char** argv) {
    // 检查输入的参数
    if (argc < 3) {
        printf("eg: ./server port paht\n");
        exit(1);
    }
    // 采用指定的端口
    int port = atoi(argv[1]);
    // 切换工作目录
    int ret = chdir(argv[2]);
    if (ret == -1) {
        perror("chdir error");
        exit(1);
    }
    // 启动epoll模型
    epoll_run(port);
    return 0;
}
```

server.h：  
```c
/*
@Filename : server.h
@Description : header of server
@Datatime : 2022/06/21 15:33:55
@Author : xushun
*/
#ifndef __SERVER_H_
#define __SERVER_H_

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <dirent.h>
#include <ctype.h>
#include <sys/types.h>
#include <sys/epoll.h>
#include <sys/stat.h>
#include <arpa/inet.h>

#define MAX_CLIENTS 1000

int init_listen_fd(int port, int epfd);
void epoll_run(int port);
void do_accept(int listen_fd, int epfd);
void do_read(int client_fd, int epfd);
int get_line(int sock, char * buf, int size);
void disconnect(int client_fd, int epfd);
void http_request(const char* request, int client_fd);
void send_responsd_head(int client_fd, int no, const char* description, const char* type, long len);
void send_file(int client_fd, const char* filename);
void send_dir(int client_fd, const char* dirname);
void send_error(int client_fd, int status, char* title, char* text);
int hexit(char c);
void encode_str(char* to, int tosize, const char* from);
void decode_str(char* to, char* from);
const char* get_file_type(const char* filename);

#endif
```

server.c：  
```c
/*
@Filename : server.c
@Description : server functions
@Datatime : 2022/06/21 15:51:10
@Author : xushun
*/

#include "server.h"

// 创建监听套接字 并将其添加到监听树上 监听其读事件
int init_listen_fd(int port, int epfd) {
    // 创建用于监听连接的套接字
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd == -1) {
        perror("socket error (listen_fd init fail)");
        exit(1);
    }
    // 端口复用
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    // 绑定本地IP和端口
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(port);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    int ret = bind(listen_fd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if (ret == -1) {
        perror("bind error");
        exit(1);
    }
    // 设置监听
    ret = listen(listen_fd, 64);
    if (ret == -1) {
        perror("listen error");
        exit(1);
    }
    // listen_fd 添加到监听树
    struct epoll_event listen_ev;
    listen_ev.data.fd = listen_fd;
    listen_ev.events = EPOLLIN;
    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &listen_ev);
    if (ret == -1) {
        perror("epoll_ctl add listen_fd error");
        exit(1);
    }
    return listen_fd;
}

// 创建epoll监听树 并启动监听循环
void epoll_run(int port) {
    // 创建epoll监听树
    int epfd = epoll_create(MAX_CLIENTS);
    if (epfd == -1) {
        perror("epoll_create error");
        exit(1);
    }
    // 将监听套接字添加到监听树上
    int listen_fd = init_listen_fd(port, epfd);
    // 启动监听循环 监听树上节点的事件
    struct epoll_event ret_events[MAX_CLIENTS];
    int ret_nready;
    while (1) {
        // 获得满足条件的监听事件
        ret_nready = epoll_wait(epfd, ret_events, MAX_CLIENTS, 0);
        if (ret_nready == -1) {
            perror("epoll_wait error");
            exit(1);
        } else if (ret_nready > 0) {
            // 遍历返回的事件
            struct epoll_event* epev;
            for (int i = 0; i < ret_nready; ++ i) {
                epev = &ret_events[i];
                // 只处理读事件
                if (!(epev -> events & EPOLLIN)) {
                    continue;
                }
                // 处理新连接请求 或http请求
                if (epev -> data.fd == listen_fd) {
                    do_accept(listen_fd, epfd);
                } else {
                    do_read(epev -> data.fd, epfd);
                }
            }
        }
    }
    return;
}

// 接收新的连接请求 并将其读事件添加到监听树
void do_accept(int listen_fd, int epfd) {
    // 接受连接请求
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    int client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client_addr_len);
    if (client_fd == -1) {
        perror("accept error");
        exit(1);
        // close(listen_fd); return;
    }
    // 打印客户端信息
    char client_ip[100] = {0};
    printf("[new client] ip : %s, port : %d, fd : %d\n",
        inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, client_ip, sizeof(client_ip)),
        ntohs(client_addr.sin_port),
        client_fd
    );
    // 设置为非阻塞
    int flag = fcntl(client_fd, F_GETFL);
    flag |= O_NONBLOCK;
    fcntl(client_fd, F_SETFL, flag);
    // 新客户端添加到监听树上 使用ET模式
    struct epoll_event client_ev;
    client_ev.data.fd = client_fd;
    client_ev.events = EPOLLIN | EPOLLET;
    int ret = epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &client_ev);
    if (ret == -1) {
        perror("epoll_ctl add client_fd fail");
        exit(1);
        // close(listen_fd); return;
    }
    return;
}

// 处理客户端连接的读事件
void do_read(int client_fd, int epfd) {
    // 读取浏览器发来的请求行
    char line[1024] = {0};
    int recv_bytes = get_line(client_fd, line, sizeof(line));
    // 客户端关闭 or 正常读请求行
    if (recv_bytes == 0) {
        printf("client closed\n");
        disconnect(client_fd, epfd);
    } else {
        // 如果有数据没读完 继续读
        while (1) {
            char buf[1024] = {0};
            recv_bytes = get_line(client_fd, buf, sizeof(buf));
            // 读到空行 读完了
            if (buf[0] == '\n') {
                break;
            } else if(recv_bytes == -1) {
                break;
            }
        }
    }
    // 判断是否GET请求 strncasecmp比较字符串前n字符
    if (strncasecmp("get", line, 3) == 0) {
        // 处理http请求
        http_request(line, client_fd);
        // 关闭连接
        disconnect(client_fd, epfd);
    }
    return;
}

// 接收请求消息的每一行内容
int get_line(int sock, char * buf, int size) {
    int i = 0;
    char ch = '\0';
    int recv_byte;
    while ((i < size - 1) && (ch != '\n')) {
        // 在读到`\n`之前 每次读一个字符
        recv_byte = recv(sock, &ch, 1, 0);
        if (recv_byte > 0) {
            // "\r\n"表示行尾
            if (ch == '\r') {
                // 参看后一个字符是否为`\n` 不会读出
                recv_byte = recv(sock, &ch, 1, MSG_PEEK);
                // 读到行尾
                if ((recv_byte > 0) && (ch == '\n')) {
                    recv(sock, &ch, 1, 0);
                } else {
                    ch = '\n';
                }
            }
            buf[i] = ch;
            ++ i;
        } else {
            ch = '\n';
        }
    }
    buf[i] = '\0';
    return i;
}

// 断开和客户端的连接 并将其从监听树上取下 关闭套接字
void disconnect(int client_fd, int epfd) {
    int ret = epoll_ctl(epfd, EPOLL_CTL_DEL, client_fd, NULL);
    if (ret == -1) {
        perror("epoll_ctl del client_fd fail");
        exit(1);
    }
    close(client_fd);
}

// 处理http请求行
void http_request(const char* request, int client_fd) {
    // 拆分请求行 正则表达式拆分以空格间隔的字段
    char method[12], path[1024], protocol[12];
    sscanf(request, "%[^ ] %[^ ] %[^ ]", method, path, protocol);
    printf("method = %s, path = %s, protocol = %s\n", method, path, protocol);
    // 解码
    decode_str(path, path);
    // 去掉paht中的'/' 获取文件名
    char* filename = path + 1;
    // 如果没有指定访问的资源 默认显示资源目录中的内容
    if (strcmp(path, "/") == 0) {
        filename = "./";
    }
    // 获取文件属性
    struct stat st;
    int ret = stat(filename, &st);
    // 没有这个文件
    if (ret == -1) {
        send_error(client_fd, 404, "Not Found", "No such file or direntry");
        return;
    }
    // 判断文件时普通文件还是目录文件
    if (S_ISDIR(st.st_mode)) {
        // 发送报头
        send_responsd_head(client_fd, 200, "OK", get_file_type(".html"), -1);
        // 发送目录信息
        send_dir(client_fd, filename);
    } else if (S_ISREG(st.st_mode)) {
        // 发送报头
        send_responsd_head(client_fd, 200, "OK", get_file_type(filename), st.st_size);
        // 发送文件内容
        send_file(client_fd, filename);
    }
    return;
}

// 发送报文头
void send_responsd_head(int client_fd, int no, const char* description, const char* type, long len) {
    char buf[4096] = {0};
    // 状态行
    sprintf(buf, "http/1.1 %d %s\r\n", no, description);
    send(client_fd, buf, strlen(buf), 0);
    // 报头
    sprintf(buf, "Content-Type:%s\r\n", type);
    sprintf(buf + strlen(buf), "Content-Length:%ld\r\n", len);
    send(client_fd, buf, strlen(buf), 0);
    // 空行
    send(client_fd, "\r\n", 2, 0);
}

// 发送文件内容
void send_file(int client_fd, const char* filename) {
    // 打开文件
    int fd = open(filename, O_RDONLY);
    if (fd == -1) {
        send_error(client_fd, 404, "Not Found", "No such file or direntry");
        // 处理请求行的时候检查过该文件存在 此时没有说明出错
        perror("open error");
        exit(1);
    }
    // 循环读文件
    char buf[4096] = {0};
    int read_bytes = 0, send_bytes = 0;
    while ((read_bytes = read(fd, buf, sizeof(buf))) > 0) {
        // 发送读到的数据
        send_bytes = send(client_fd, buf, read_bytes, 0);
        if (send_bytes == -1) {
            if (errno == EAGAIN) {
                perror("send error EAGAIN");
                continue;
            } else if (errno == EINTR) {
                perror("send error EINTR");
                continue;
            } else {
                perror("send error");
                exit(1);
            }
        }
    }
    if (read_bytes == -1) {
        perror("read error");
        exit(1);
    }
    printf("regular file send OK\n");
    close(fd);
    return;
}

// 发送目录文件内容
void send_dir(int client_fd, const char* dirname) {
    // 作为html页面发送
    char buf[4096] = {0};
    sprintf(buf, "<html><head><title>目录名: %s</title></head>", dirname);
    sprintf(buf + strlen(buf), "<body><h1>当前目录: %s</h1><table>", dirname);
    // 目录项数组
    struct dirent** dptr;
    int dnum = scandir(dirname, &dptr, NULL, alphasort);
    // 遍历目录项
    char en_str[1024] = {0};
    char path[1024] = {0};
    for (int i = 0; i < dnum; ++ i) {
        char* name = dptr[i] -> d_name;
        // 拼接文件的完整路径
        sprintf(path, "%s/%s", dirname, name);
        // 获得文件信息
        struct stat st;
        stat(path, &st);
        // 编码文件名
        encode_str(en_str, sizeof(en_str), name);
        // 普通文件 or 目录文件
        if (S_ISREG(st.st_mode)) {
            sprintf(buf + strlen(buf),
                "<tr><td><a href=\"%s\">%s</a></td><td>%ld</td></tr>",
                en_str, name, (long)st.st_size
            );
        } else if (S_ISDIR(st.st_mode)) {
            sprintf(buf + strlen(buf),
                "<tr><td><a href=\"%s/\">%s/</a></td><td>%ld</td></tr>",
                en_str, name, (long)st.st_size
            );
        }
        // 发送
        int send_bytes = send(client_fd, buf, strlen(buf), 0);
        if (send_bytes == -1) {
            if (errno == EAGAIN) {
                perror("send error EAGAIN");
                continue;
            } else if (errno == EINTR) {
                perror("send error EINTR");
                continue;
            } else {
                perror("send error");
                exit(1);
            }
        }
        memset(buf, 0, sizeof(buf));
    }
    sprintf(buf + strlen(buf), "</table></body></html>");
    send(client_fd, buf, strlen(buf), 0);
    printf("dir message send OK\n");
    return;
}

// 发送404给浏览器
void send_error(int client_fd, int status, char* title, char* text) {
    char buf[4096] = {0};
    // 报文头
	sprintf(buf, "%s %d %s\r\n", "HTTP/1.1", status, title);
	sprintf(buf+strlen(buf), "Content-Type:%s\r\n", "text/html");
	sprintf(buf+strlen(buf), "Content-Length:%d\r\n", -1);
	sprintf(buf+strlen(buf), "Connection: close\r\n");
	send(client_fd, buf, strlen(buf), 0);
	send(client_fd, "\r\n", 2, 0);
    // 404 页面
	memset(buf, 0, sizeof(buf));
	sprintf(buf, "<html><head><title>%d %s</title></head>\n", status, title);
	sprintf(buf+strlen(buf), "<body bgcolor=\"#cc99cc\"><h2 align=\"center\">%d %s</h4>\n", status, title);
	sprintf(buf+strlen(buf), "%s\n", text);
	sprintf(buf+strlen(buf), "<hr>\n</body>\n</html>\n");
	send(client_fd, buf, strlen(buf), 0);
	return ;
}

// 字符串编码 向浏览器写数据时 将数字字母以及/_.-~以外的字符编码
void encode_str(char* to, int tosize, const char* from) {
    int tolen;
    for (tolen = 0; *from != '\0' && tolen + 4 < tosize; ++from) {    
        if (isalnum(*from) || strchr("/_.-~", *from) != (char*)0) {      
            *to = *from;
            ++to;
            ++tolen;
        } else {
            sprintf(to, "%%%02x", (int) *from & 0xff);
            to += 3;
            tolen += 3;
        }
    }
    *to = '\0';
}
// 16进制数转化为10进制
int hexit(char c) {
    if (c >= '0' && c <= '9')
        return c - '0';
    if (c >= 'a' && c <= 'f')
        return c - 'a' + 10;
    if (c >= 'A' && c <= 'F')
        return c - 'A' + 10;
    return 0;
}
// 字符串解码
void decode_str(char* to, char* from) {
    for ( ; *from != '\0'; ++to, ++from  ) {     
        if (from[0] == '%' && isxdigit(from[1]) && isxdigit(from[2])) {       
            *to = hexit(from[1])*16 + hexit(from[2]);
            from += 2;                      
        } else {
            *to = *from;
        }
    }
    *to = '\0';
}

// 通过文件名获得文件的类型
const char* get_file_type(const char* filename) {
    // 自右向左查找`.`字符的位置 不存在返回NULL
    char* dot = strrchr(filename, '.');
    if (dot == NULL)
        return "text/plain; charset=utf-8";
    if (strcmp(dot, ".html") == 0 || strcmp(dot, ".htm") == 0)
        return "text/html; charset=utf-8";
    if (strcmp(dot, ".jpg") == 0 || strcmp(dot, ".jpeg") == 0)
        return "image/jpeg";
    if (strcmp(dot, ".gif") == 0)
        return "image/gif";
    if (strcmp(dot, ".png") == 0)
        return "image/png";
    if (strcmp(dot, ".css") == 0)
        return "text/css";
    if (strcmp(dot, ".au") == 0)
        return "audio/basic";
    if (strcmp(dot, ".wav" ) == 0)
        return "audio/wav";
    if (strcmp(dot, ".avi") == 0)
        return "video/x-msvideo";
    if (strcmp(dot, ".mov") == 0 || strcmp(dot, ".qt") == 0)
        return "video/quicktime";
    if (strcmp(dot, ".mpeg") == 0 || strcmp(dot, ".mpe") == 0)
        return "video/mpeg";
    if (strcmp(dot, ".vrml") == 0 || strcmp(dot, ".wrl") == 0)
        return "model/vrml";
    if (strcmp(dot, ".midi") == 0 || strcmp(dot, ".mid") == 0)
        return "audio/midi";
    if (strcmp(dot, ".mp3") == 0)
        return "audio/mpeg";
    if (strcmp(dot, ".ogg") == 0)
        return "application/ogg";
    if (strcmp(dot, ".pac") == 0)
        return "application/x-ns-proxy-autoconfig";
    return "text/plain; charset=utf-8";
}
```