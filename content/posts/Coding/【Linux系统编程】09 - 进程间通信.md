---
title: 【Linux系统编程】09 - 进程间通信
date: 2022-05-26
tags: [Linux]
categories: [Coding]
---

## IPC - Inter Process Communication
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
使用文件也可以进行进程间通信。（当然不建议使用这种方式）

- 有血缘关系的进程：fork后，父子进程共享文件描述符，它们的同一个文件描述符对应同一个文件，父进程向文件中写数据，子进程从文件中读数据，同样也是借助内核的缓冲区实现的；
- 无血缘关系的进程：也可以使用文件进行进程间通信，只要对同一个文件进行读写即可，和父子进程不同的是，它们打开的文件描述符可能不一样，但只要对应的是同一个文件就行。

## 存储映射 I/O
存储映射I/O（Memory-mapped I/O）,使一个磁盘文件与内存空间中的一个缓冲区映射。从缓冲区中取数据，就相当于读文件中的对应字节；将数据存入缓冲区，就相当于将相应的字节写入文件。这样就可以在不使用`read write`函数的情况下，使用地址（指针）来完成IO操作。

使用这种方法，需要通知内核，将一个指定文件映射到存储区域中，这个映射工作通过`mmap`函数来实现。图示如下：

![](/post_images/posts/Coding/【Linux系统编程】09/存储映射IO.jpg "存储映射IO")

### mmap 函数
将文件映射到内存。（系统函数）

函数原型：  
```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags,
            int fd, off_t offset);
```
- 返回值：
  - 成功，返回创建的映射区的首地址；
  - 失败：返回`MAP_FAILED`，实际上是一个`(void*) -1`的宏，并设置`errno`。
- `addr`：指定映射区的首地址（通常传入`NULL`，表示让系统自动分配）；
- `length`：指定映射区的大小；
- `prot`：映射区的读写属性，
  - `PROT_READ`，读；
  - `PROT_WRITE`，写；
  - `PROT_READ | PROT_WRITE`，读写。
- `flags`：共享内存的共享属性，
  - `MAP_SHARED`，共享的（对内存修改会同步修改磁盘上的文件）；
  - `MAP_PRIVATE`，私有的（不会同步修改磁盘）。
- `fd`，用于创建映射区的文件的文件描述符；
- `offset`，偏移位置（4K的整数倍）。

### munmap 函数
释放映射区。（系统函数）

使用`mmap`建立的映射区，在结束使用后也应该调用`munmap`来释放映射区。

函数原型：  
```c
#include <sys/mman.h>

int munmap(void *addr, size_t length);
```
- 返回值：
  - 成功，返回`0`；
  - 失败，返回`-1`，并设置`errno`。
- `addr`：`mmap`的返回值；
- `length`：映射区大小。

### 建立映射区 - 测试
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>
int main(int argc, char** argv) {
    int fd = open("test.txt", O_CREAT | O_TRUNC | O_RDWR, 0664);
    if (fd == -1) {
        perror("open error");
        exit(1);
    }
    lseek(fd, 99, SEEK_END);
    if (write(fd, "\0", 1) == -1) { // 扩展文件
        perror("write error");
        exit(1);
    }
    int len = lseek(fd, 0, SEEK_END); // 文件长度
    void* p = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0); // 创建映射区
    if (p == MAP_FAILED) {
        perror("mmap error");
        exit(1);
    }
    // 对内存中的映射区进行读写
    strcpy(p, "hello mmap\n");
    printf("read from mmap : %s", p);

    if (munmap(p, len) == -1) {
        perror("munmap error");
        exit(1);
    }
    close(fd);
    return 0;
}
```
对内存中映射区的读写可以反应到磁盘的文件中。

### mmap - 注意事项
1. 可以`open O_CREAT`一个新文件并创建它的映射区吗？
   - 可以，但是要注意拓展文件长度，并且避免以下错误：
   - 文件大小为0，创建大小大于0的映射区，**总线错误**；
   - 文件大小为0，创建大小为0的映射区，**无效参数**，创建出错；
2. 文件读写方式为**只读**，而创建**读写**方式的映射区，**无效参数**，创建出错；
3. 创建映射区时，文件必须要有read权限（只写是不行的）；
4. `MAP_SHARED`时，mmap的权限`<=`文件的权限；`MAP_PRIVATE`时则无所谓，因为mmap的权限是对内存的限制；
5. 文件描述符要在映射区创建之后关闭，不一定在映射区关闭之后关闭；
6. `offset`必须是4K（4096）的整数倍，否则**无效参数**，创建出错（原因是MMU映射的最小单位是4096）；
7. 不能对映射区进行越界访问；
8. 对`mem`指针（`mmap`返回的指针）进行改变，如自增运算等，`munmap`就无法释放映射区，`munmap`的只接收`mmap`返回的指针；
9. 使用`mmap`时，必须检查其返回值；
10. 当`MAP_PRIVATE`时，对映射区的修改不会反映到磁盘上，文件的读写方式只需只读`RDONLY`即可，而映射区的权限可以为读写。

创建读写映射区的最佳方式：  
```c
fd = open("filename", O_RDWR);
mmap(NULL, 有效文件大小, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
```

### mmap - 父子进程间通信
父子进程可以通过共享一块内存映射区来实现进程间的通信。  
1. 父进程创建映射区，`open(RDWR) mmap(MAP_SHARED)`（注意，要想共享一块内存区域，必须使用`MAP_SHARED`参数，如果使用`MAP_PRIVATE`，父子进程将各自独享一块内存映射区）；
2. fork；
3. 对映射区进行读写。

示例，父子进程共享一个`int`变量：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>

int global_int = 100;

int main(int argc, char** argv) {
    int fd = open("temp", O_CREAT | O_TRUNC | O_RDWR, 0664);
    if (fd == -1) {
        perror("open error");
        exit(1);
    }
    if (ftruncate(fd, sizeof(int)) == -1) {
        perror("ftruncate error");
        exit(1);
    }
    int* shared_int = (int*)mmap(NULL, sizeof(int), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    // int* shared_int = (int*)mmap(NULL, sizeof(int), PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0);
    if (shared_int == MAP_FAILED) {
        perror("mmap error");
        exit(1);
    }
    close(fd);
    
    pid_t pid = fork();
    if (pid > 0) {
        sleep(2); // 等待子进程修改变量
        // 读共享内存
        printf("parent process : global_int = %d, shared_int = %d\n", global_int, *shared_int);
        wait(NULL);
    } else if (pid == 0) {
        *shared_int = 666; // 写共享内存
        global_int = 777;
        printf(" child process : global_int = %d, shared_int = %d\n", global_int, *shared_int);
    } else {
        perror("fork error");
        exit(1);
    }
    if (munmap(shared_int, sizeof(int)) == -1) {
        perror("munmap error");
        exit(1);
    }
    return 0;
}
```

测试结果：使用共享映射区的内存可以共享  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_mmap$ ./test_mmap_family 
 child process : global_int = 777, shared_int = 666
parent process : global_int = 100, shared_int = 666
```

### mmap - 无血缘关系进程间通信
为同一个文件创建共享映射区即可。

示例，两个进程读写结构体：打开两个终端分别执行读写  
`mmap_w.c`：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>

typedef struct {
    int id;
    char name[256];
    int age;
} student;

int main(int argc, char** argv) {
    // write a struct student to mmap
    int fd = open("temp", O_RDWR | O_CREAT | O_TRUNC, 0664);
    if (fd == -1) {
        perror("open error");
        exit(1);
    }
    if (ftruncate(fd, sizeof(student)) == -1) {
        perror("ftruncate error");
        exit(1);
    }
    student* mem = (student*)mmap(NULL, sizeof(student), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (mem == MAP_FAILED) {
        perror("mmap error");
        exit(1);
    }
    close(fd);
    sleep(5);
    student stu = {1, "Alice", 18};
    while (1) {
        // update mem every 2 secs
        memcpy(mem, &stu, sizeof(student)); // string.h
        stu.id ++;
        sleep(2);
    }
    if (munmap(mem, sizeof(student)) == -1) {
        perror("munmap error");
        exit(1);
    }
    return 0;
}
```
`mmap_r.c`：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>

typedef struct {
    int id;
    char name[256];
    int age;
} student;

int main(int argc, char** argv) {
    // read a struct student from mmap
    int fd = open("temp", O_RDONLY, 0664);
    if (fd == -1) {
        perror("open error");
        exit(1);
    }
    student* mem = (student*)mmap(NULL, sizeof(student), PROT_READ, MAP_SHARED, fd, 0);
    if (mem == MAP_FAILED) {
        perror("mmap error");
        exit(1);
    }
    close(fd);

    while (1) {
        // read mem every sec
        printf("id = %d, name = %s, age = %d\n", mem -> id, mem -> name, mem -> age);
        sleep(1);
    }
    if (munmap(mem, sizeof(student)) == -1) {
        perror("munmap error");
        exit(1);
    }
    return 0;
}
```

### mmap - 多个读写端
可以使用多个读写端进行读写，因为mmap是缓冲区的模式。与pipe和FIFO不同，共享映射区可以重复读取。

### (匿名)mmap - 父子进程间通信
使用共享映射区来进行文件读写和父子进程间通信比较方便，但缺陷是，每次创建映射区都需要依赖于一个文件才行。通常为了建立映射区，需要`open`一个临时文件，创建完成后再`unlink close`，比较麻烦。

这种**父子进程间的通信**可以使用**匿名映射区**来实现。Linux系统为我们提供了这样的方法，无需依赖文件即可创建映射区，需要借助标志位参数`flags`来指定。

使用方法：`void* mem = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_ANONYMOUS, -1, 0);`

注：`MAP_ANON`和`MAP_ANONYMOUS`同义，但不推荐使用。

上述两个宏是Linux系统特有的宏，在类Unix系统中如果无法使用该宏定义，可以使用如下两步来完成匿名映射区的建立：  
```c
int fd = open("dev/zero", O_RDWR);
void* mem = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
```