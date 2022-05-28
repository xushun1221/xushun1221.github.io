---
title: 【Linux系统编程】09 - 进程间通信
date: 2022-05-26
tags: [Linux]
categories: [Coding]
---

## IPC - Inter Process Communiccation
Linux环境下，进程地址空间相互独立，每个进程各自有不同的用户地址空间。任何一个进程的全局变量在另一个进程中都看不到，所以进程和进程之间不能相互访问，要交换数据必须通过内核，在内核中开辟一块缓冲区，进程 1 把数据从用户空间拷到内核缓冲区，进程 2 再从内核缓冲区把数据读走，内核提供的这种机制称为进程间通信（IPC，InterProcess Communication）。

![](/post_images/posts/Coding/【Linux系统编程】09/IPC示意图.jpg "IPC示意图")

在进程之间完成数据传递需要借助操作系统提供特殊的方法，如：文件、管道、信号、共享内存、消息队列、套接字、命名管道等。随着计算机的发展，一些方法由于自身的设计缺陷被淘汰或弃用，如今常用的进程间通信的方法有：
1. 管道（使用最简单）；
2. 信号（开销最小）；
3. 共享映射区（无血缘关系）；
4. 本地套接字（最稳定）。

## 管道
### 管道的概念
管道是一种最基本的IPC机制，作用于**有血缘关系**的进程之间，完成数据传递。调用`pipe`函数即可创建一个管道。特性如下：
1. 管道的本质是一个伪文件（实质为内核缓冲区）；
2. 由两个文件描述符引用，一个表示读端，一个表示写端；
3. 管道的数据只能读一次；
4. 数据只能从写端流入，读端流出。

管道的实现原理：内核使用环形队列机制，借助内核缓冲区（4KiB）实现。

管道的局限性：
1. 数据不能进程自己写，自己读；
2. 管道中的数据不能反复读取，一旦读走，就不再存在；
3. 采用半双工通信，数据只能在单方向上流动。

### pipe
创建（并打开）一个管道。（系统函数）

函数原型：  
```c
#include <unistd.h>

int pipe(int pipefd[2]);
```
- 返回值：成功返回`0`，失败返回`-1`设置`errno`；
- `pipefd[2]`：`pipdfd[0]`读端fd，`pipefd[1]`写端fd；

函数调用成功会返回两个对应`r/w`的文件描述符`pipefd[0] pipefd[1]`。无需`open`，但需要手动`close`。像管道文件读写数据其实是在读写内核缓冲区。

管道创建成功后，创建该管道的进程（父进程）同时掌握着管道的读端和写端。实现父子进程之间通信是靠共享文件描述符实现的。步骤如下：
1. 父进程调用`pipe`函数创建管道，得到两个文件描述符`pipefd[0] pipefd[1]`；
2. 父进程调用`fork`函数创建子进程，子进程共享这两个文件描述符；
3. 父进程关闭管道读端，子进程关闭管道写端。父进程向管道写入数据，子进程将管道数据读出，由于管道是使用环形队列实现的，数据从写端流入，读端流出，这样就实现了父子进程间通信。

图示如下：  
![](/post_images/posts/Coding/【Linux系统编程】09/管道示意图.jpg "管道示意图")

测试demo：父进程向子进程传送字符串  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char** argv) {
    int pipefd[2];
    if (pipe(pipefd) == -1) {
        perror("pipe error");
        exit(1);
    }
    pid_t pid = fork();
    if (pid > 0) {
        close(pipefd[0]); // 父进程 关闭读端
        write(pipefd[1], "hello pipe\n", strlen("hellp pipe\n"));
        close(pipefd[1]);
        sleep(1); // 保证在子进程后结束
    } else if (pid == 0) {
        close(pipefd[1]); // 子进程 关闭写端
        char buf[1024];
        int readbytes = read(pipefd[0], buf, sizeof(buf));
        close(pipefd[0]);
        int writebytes = write(STDOUT_FILENO, buf, readbytes);
        if (writebytes == -1) {
            perror("write error");
            exit(1);
        }
    } else {
        perror("fork error");
        exit(1);
    }
    return 0;
}
```
有个有意思的现象，子进程读取数据之前没有sleep，但是依然可以正确读取管道中的数据，我猜，是因为子进程的`read`被阻塞。（猜对了^^，不过实际情况更复杂一些，详解请看管道的读写行为）

### 管道的读写行为
（假设对管道的读写操作都是阻塞IO操作，并没有设置`O_NONBLOCK`标志）
#### 读管道
1. 管道中还有数据，`read`返回实际读到的字节数；
2. 管道中无数据，
   1. 管道写端被全部关闭，`read`返回`0`，类似读到文件结尾；
   2. 管道写端没有被全部关闭，`read`阻塞等待（不久的将来可能会有数据到达，此时会让出CPU）

#### 写管道
1. 管道读端全部被关闭，进程异常终止（`SIGPIPE`信号导致的）；
2. 管道读端没有被全部关闭，
   1. 管道已经满了，`write`阻塞；（在较新的Linux内核版本中，管道缓冲区会自动扩充，可能不会出现这种情况）
   2. 管道未满，`write`将数据写入，返回实际写入的字节数。

### 练习 - 父子进程实现 ls | wc -l
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
int main(int argc, char** argv) {
    int pipefd[2];
    if (pipe(pipefd) == -1) {
        perror("pipe error");
        exit(1);
    }
    pid_t pid = fork();
    if (pid > 0) { // 父进程 ls
        close(pipefd[0]);
        // ls 默认写入STDOUT 需要重定向
        if (dup2(pipefd[1], STDOUT_FILENO) == -1) {
            perror("dup2 error");
            exit(1);
        }
        execlp("ls", "ls", NULL);
        perror("execlp error");
        // close(pipfd[1]); 关闭不了 需要依赖隐式回收
    } else if (pid == 0) { // 子进程 wc -l
        close(pipefd[1]);
        // wc 默认从STDIN输入 需要重定向
        if (dup2(pipefd[0], STDIN_FILENO) == -1) {
            perror("dup2 error");
            exit(1);
        }
        execlp("wc", "wc", "-l", NULL);
        perror("execlp error");
        // close(pipfd[0]);
    } else {
        perror("fork error");
        exit(1);
    }
    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_IPC$ ./ls-wc-l 
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_IPC$ 5
```
测试结果中，有一点小问题：子进程的`wc -l`还没有输出结果时，`xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_IPC$`命令提示符就被输出了，这是因为，父进程实现`ls`，而子进程实现`wc -l`，子进程需要（阻塞）等待父进程的管道输出，才能继续，这导致父进程有可能会先于子进程结束，父进程结束后，会被`shell`回收，然后就先输出了命令提示符，而不是子进程的输出。  
解决这个问题可以用父进程实现`wc -l`，而子进程实现`ls`，这样父进程就一定会比子进程后结束。  
代码：  
```c
if (pid > 0) { // 父进程 wc -l
    close(pipefd[1]);
    // wc 默认从STDIN输入 需要重定向
    if (dup2(pipefd[0], STDIN_FILENO) == -1) {
        perror("dup2 error");
        exit(1);
    }
    execlp("wc", "wc", "-l", NULL);
    perror("execlp error");
    // close(pipfd[0]);
} else if (pid == 0) { // 子进程 ls
    close(pipefd[0]);
    // ls 默认写入STDOUT 需要重定向
    if (dup2(pipefd[1], STDOUT_FILENO) == -1) {
        perror("dup2 error");
        exit(1);
    }
    execlp("ls", "ls", NULL);
    perror("execlp error");
    // close(pipfd[1]); 关闭不了 需要依赖隐式回收
} else {
    perror("fork error");
    exit(1);
}
```

### 练习 - 兄弟进程实现 ls | wc -l
要求，兄弟分别实现ls和wc，父进程回收，使用循环创建子进程的方法实现。

代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
int main(int argc, char** argv) {
    int pipefd[2];
    if(pipe(pipefd) == -1) {
        perror("pipe error");
        exit(1);
    }
    int index = 0, n = 2;
    pid_t pid;
    for (index = 0; index < n; ++ index) {
        pid = fork();
        if (pid == -1) {
            perror("fork error");
            exit(1);
        } else if (pid == 0) {
            break;
        }
    }
    if (index == 2) { // 父
    // *************
        close(pipefd[0]);
        close(pipefd[1]);
    // *************
        wait(NULL); // 父进程一定最后结束
        wait(NULL);
    } else if (index == 1) { // 1 号进程 wc -l
        close(pipefd[1]);
        if (dup2(pipefd[0], STDIN_FILENO) == -1) {
            perror("dup2 error");
            exit(1);
        }
        execlp("wc", "wc", "-l", NULL);
        perror("execlp error");
        // close(pipefd[0]);
    } else if (index == 0) { // 0 号进程 ls
        close(pipefd[0]);
        if (dup2(pipefd[1], STDOUT_FILENO) == -1) {
            perror("dup2 error");
            exit(1);
        }
        execlp("ls", "ls", NULL);
        perror("execlp error");
        // close(pipefd[1]);
    }
    return 0;
}
```
注意，在父进程中，我们需要把管道的读写端都关掉（准确来说是，需要关掉父进程写端），如果不关，那么`wc`时，需要读管道时，父进程的写端没有关闭（另一个兄弟写完后关闭了），`wc`就会阻塞在这里。

总的来说，使用管道时，需要及时关闭不需要的读写端，尽量保证数据的单向流动（一个读端一个写端），要尽量避免使用多个读端和写端进行操作。

### 管道的缓冲区大小
- 使用终端命令`ulimit -a`查看，默认的大小是`4KiB`；
- 使用`fpathconf`函数查看。

### 管道的优劣
- 优点：简单，相比信号、套接字等实现进程间通信，简单得多；
- 缺点
  1. 只能单向通信，双向通信需要建立两个管道；
  2. 只能用于父子、兄弟进程之间。（该问题由fifo有名管道解决）

## FIFO
FIFO常被称为**命名管道**，以区分管道（pipe），pipe只能用于有血缘关系的进程之间通信，但通过FIFO，不相关的进程之间也可以交换数据。

FIFO是Linux基础文件类型的一种，但是，FIFO文件在磁盘上没有数据块，仅仅用来标识内核中的一条通道。各个进程可以打开这个文件进行读写，实际上是在读写内核通道，这样就实现了进程间通信。

### 创建FIFO
终端命令：`mkfifo fifoname`。

库函数：`mkfifo`  
函数原型：  
```c
#include <sys/types.h>
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);
```
- 返回值：成功返回`0`，失败返回`-1`；
- `pathname`：管道名；
- `mode`：访问权限，计算公式为`mode & (~umask)`

使用`mkfifo`创建FIFO后，就可以使用`open`打开它，常见的IO函数都可以用于FIFO，如`close read write unlink`等。

### FIFO - 实现非血缘关系进程间通信
`fifo_w.c`：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <string.h>
#include <fcntl.h>
#include <time.h>

int main (int argc, char** argv) {
    if (argc < 2) {
        printf("enter like : fifo_c fifoname\n");
        return -1;
    }
    int fd = open(argv[1], O_WRONLY);
    if (fd == -1) {
        perror("open fifo error");
        exit(1);
    }
    srand(time(NULL));
    char buf[1024];
    int i = 0;
    while (1) {
        sprintf(buf, "pid %d : write to %s : %d\n", getpid(), argv[1], ++ i);
        write(fd, buf, strlen(buf));
        sleep(rand() % 3 + 1);
    }
    close(fd);
    return 0;
}
```

`fifo_r.c`：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <string.h>
#include <fcntl.h>
#include <time.h>

int main (int argc, char** argv) {
    if (argc < 2) {
        printf("enter like : fifo_c fifoname\n");
        return -1;
    }
    int fd = open(argv[1], O_RDONLY);
    if (fd == -1) {
        perror("open fifo error");
        exit(1);
    }
    char buf[1024];
    while (1) {
        int read_bytes = read(fd, buf, sizeof(buf));
        write(STDOUT_FILENO, buf, read_bytes);
    }
    close(fd);
    return 0;
}
```

测试1，打开多个终端执行`./fifo_w testfile`，打开一个终端执行`./fifo_r testfifo`，结果显示，读端可以读取多个写端写入的数据。

测试2，打开一个终端执行`./fifo_w testfile`，打开多个终端执行`./fifo_r testfifo`，结果显示，多个读端竞争FIFO内的数据，FIFO内的数据只能被读一次。

## 文件
使用文件也可以进行进程间通信。

- 有血缘关系的进程：fork后，父子进程共享文件描述符，它们的同一个文件描述符对应同一个文件，父进程向文件中写数据，子进程从文件中读数据，同样也是借助内核的缓冲区实现的；
- 无血缘关系的进程：也可以使用文件进行进程间通信，只要对同一个文件进行读写即可，和父子进程不同的是，它们打开的文件描述符可能不一样，但只要对应的是同一个文件就行。

## 存储映射 I/O
存储映射I/O（Memory-mapped I/O）,使一个磁盘文件与内存空间中的一个缓冲区映射。从缓冲区中取数据，就相当于读文件中的对应字节；将数据存入缓冲区，就相当于将相应的字节写入文件。这样就可以在不使用`read write`函数的情况下，使用地址（指针）来完成IO操作。

使用这种方法，需要通知内核，将一个指定文件映射到存储区域中，这个映射工作通过`mmap`函数来实现。图示如下：

![](/post_images/posts/Coding/【Linux系统编程】09/存储映射IO.jpg "存储映射IO")

### mmap 函数
