# 【Linux网络编程】04 - epoll进阶


## epoll事件模型
epoll事件有两种模型：
1. Edge Triggered (ET)：边沿触发模型，只有数据到来时才会导致`epoll_wait`返回，无论缓冲区内是否还有数据；（缓冲区还有数据未读，只有新的数据到来时才会触发`epoll_wait`返回）
2. Level Triggered (LT)：水平触发模型，只要缓冲区中有数据就会触发`epoll_wait`返回。

### 实例程序
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

### 
