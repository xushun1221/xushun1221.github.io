---
title: 【Linux系统编程】06 - 文件IO
date: 2022-05-14
tags: [Linux]
categories: [Coding]
---

## 系统调用和系统函数
系统调用，是由操作系统实现并提供给外部应用程序的编程接口，是应用程序同操作系统之间数据交互的桥梁。

系统函数又是什么？例如，我们打开`open()`函数的帮助文档`man 2 open`，可以看到它是属于第二章的内容，也就是属于系统调用，而如果去阅读Linux内核的源码就会发现，在源码中没有`open()`函数，而它对应的真正的系统调用函数应该是`sys_open()`。`open()`只是`sys_open()`的一个简单封装，原因在于，系统需要和应用进行交互，但是又不希望被应用直接窥探到，所以进行了浅封装。严谨地来说`open()`这类函数，应该被称为**系统函数**而非系统调用。

方便起见，将`open()`这类系统函数不加区分得称为系统调用。

看下面这张图可以简单地了解一下，c标准库函数和系统调用之间的关系，如何将`Hello World!`打印到屏幕上？  
![](/post_images/posts/Coding/【Linux系统编程】06/系统调用示意图.jpg "系统调用示意图")

## open
打开或创建一个文件。

函数原型：  
```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```
- `open`函数所需的头文件：`sys/types.h`和`sys/stat.h`，可以用`unistd.h`代替（unix standard）； 
- 返回值：打开文件获得的文件描述符descriptor（整型），出错返回-1（`errno`会被设置为相应的值）；
- `pathname`：文件路径；
- `flag`：该参数用来表明以何种模式访问文件；
  - （宏定义在`fcntl.h`中）
  - 必须包含下列三种访问模式之一，`O_RDONLY只读`、`O_WRONLY只写`、`O_RDWR读写`；
  - 其他有关文件创建和文件状态的flag可以以或的形式组合，如`O_RDONLY | O_XXXXX`;
- `mode`：文件访问的权限，在创建文件的时候需要指定它的权限（`flag`参数指定了`O_CREAT`），打开已有文件时不需要这个参数，`mode_t`是八进制的整型，应将`0777, 0751`这样的数赋值给`mode`(第一个0表示八进制)。

常用`flag`参数：
- `O_CREAT`：创建文件；
- `O_APPEND`：在文件末尾追加；
- `O_EXCL`：确定文件是否存在，如果文件已存在，并使用了`O_CREAT`则返回-1，文件不存在返回文件描述符；
- `O_TRUNC`：将文件大小截断为0；
- `O_NONBLOCK`：非阻塞

`mode`计算方式：`mode & (~umask)`。  
终端命令`umask`查看，`umask`的值为`0002`，`~umask`就是`0775`。

常见的错误：
- 打开的文件不存在；
- 以写方式打开只读文件（权限不够）；
- 以只写方式打开目录文件；
- 当`open`出错时，程序会自动设置`errno`，可以通过`strerror(errno)`来查看`errno`代表的含义：以写方式打开只读文件为例；
    ```c
    #include <unistd.h>
    #include <fcntl.h>
    #include <stdio.h>
    #include <errno.h> // 支持errno
    #include <string.h>// 支持errno代表的含义
    int main(int argc, char *argv[]) {
        int fd = open("./testopen.txt", O_RDWR); // 是个只读文件
        printf("fd = %d, errno = %d : %s\n", fd, errno, strerror(errno));
        close(fd);
        return 0;
    }
    ```
    输出：
    ```console
    fd = -1, errno = 13 : Permission denied
    ```

## close
关闭一个文件。

函数原型：  
```c
#include <unistd.h>

int close(int fd);
```
- 返回值：成功返回`0`，失败返回`-1`；
- `fd`：文件描述符。

## read
从一个文件中读取数据。

函数原型：  
```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
```
- 返回值：有符号整数；
  - 成功读取会返回读到的字节数；
  - 如果返回`0`，表示读到文件结尾；
  - 如果失败，返回`-1`并设置`errno`。
  - 如果返回`-1`且`errorno == EAGAIN 或 EWOULDBLOCK`,说明正在读取非阻塞文件且无数据（设备文件或网络文件）。
- `fd`：文件描述符；
- `buf`：缓冲区；
- `count`：缓冲区的大小。

## write
向一个文件中写数据。

函数原型：  
```c
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t count);
```
- 返回值：有符号整数；
  - 成功写出，会返回写出的字节数；
  - 如果返回`0`，表示没有数据可以写出；
  - 如果失败，返回`-1`并设置`errno`；
- `fd`：文件描述符；
- `buf`：从该缓冲区写出；
- `count`：实际要写出的数据大小。

## 实现cp程序
使用`open`、`close`、`read`、`write`实现`cp`程序，`my_cp filename1 filename2`，将文件1复制到文件2，如果文件2存在则覆盖它，不存在则创建。  
代码：  
```c
#include <unistd.h>
#include <fcntl.h>
int main(int argc, char *argv[]) {
    int fd_in = open(argv[1], O_RDONLY);
    int fd_out = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0664);
    char buf[1024];
    int read_bytes;
    while ((read_bytes = read(fd_in, buf, 1024)) != 0) {
        write(fd_out, buf, read_bytes);
    }
    close(fd_in);
    close(fd_out);
    return 0;
}
```
注意：对于所有的系统调用，都需要去判断和处理返回值。我们可以用上面提到的`errno`和`printf`来打印错误提示，更推荐使用`perror`。

## 错误处理函数
### errno 和 strerror
当出错时，程序会自动设置`errno`，可以通过`strerror(errno)`来查看`errno`代表的含义。

### perror 打印错误提示
`perror`在标准错误中生成一条信息，描述在调用系统函数或库函数遇到的最后一个错误。

函数原型：  
```c
#include <stdio.h>
void perror(const char *s);
```
- `s`：打印`s`，后跟一个冒号或空格，然后描述`errno`表示的错误；
- `s`应设置为简单的错误提示语句。

带错误处理的cp程序：  
```c
#include <unistd.h>
#include <fcntl.h>
#include <stdlib.h>
#include <stdio.h>
int main(int argc, char *argv[]) {
    int fd_in = open(argv[1], O_RDONLY);
    if (fd_in == -1) {
        perror("argv1 open failed"); // stdio.h
        exit(1); // stdlib.h
    }
    int fd_out = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0664);
    if (fd_out == -1) {
        perror("argv2 open failed");
        close(fd_in);
        exit(1);
    }
    char buf[1024];
    int read_bytes;
    while ((read_bytes = read(fd_in, buf, 1024)) != 0) {
        if (read_bytes == -1) {
            perror("read error");
            break;
        }
        if (write(fd_out, buf, read_bytes) == -1) {
            perror("write error");
            break;
        }
    }
    close(fd_in);
    close(fd_out);
    return 0;
}
```

## 系统调用与库函数 - 预读入和缓输出
### 问题
考虑这样一个问题：使用C标准库中的`fgetc fputc`函数每次读取、写出一个字符，使用系统调用函数`read write`每次读取、写出一个字符，拷贝相同的文件，谁的效率更高呢？

### 朴素的猜想
一个朴素的想法是：如果`fgetc fputc`是库函数，无法直接进入内核，如果要对磁盘进行读写，还是需要进行系统调用，使用`read write`函数。那如果直接使用系统调用，应该会更快？  
![](/post_images/posts/Coding/【Linux系统编程】06/朴素想法.jpg "朴素想法")

### 实验
做个实验，用标准C库和系统调用来拷贝同一个大文件（大概4MiB），看看运行时间。  
标准C库程序：  
```c
#include <stdio.h>
#include <stdlib.h>
int main() {
    FILE *fp_in, *fp_out;
    int ch;
    fp_in = fopen("bigfile.txt", "r"); // 做实验就不写的很严谨了 其实需要判断一下错误
    fp_out = fopen("bigfile.cp", "w");
    while ((ch = fgetc(fp_in)) != EOF) {
        fputc(ch, fp_out);
    }
    fclose(fp_in);
    fclose(fp_out);
    return 0;
}
```
系统调用：  
```c
#include <unistd.h>
#include <fcntl.h>
#include <stdlib.h>
#include <stdio.h>
int main() {
    int fd_in = open("./bigfile.txt", O_RDONLY);
    int fd_out = open("./bigfile.cp1", O_WRONLY | O_CREAT | O_TRUNC, 0664);
    char buf[1];
    int read_bytes;
    while ((read_bytes = read(fd_in, buf, 1)) != 0) {
        write(fd_out, buf, read_bytes);
    }
    close(fd_in);
    close(fd_out);
    return 0;
}
```
看一下运行时间：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_fileIO$ time ./fgetcfputc 

real    0m0.046s
user    0m0.029s
sys     0m0.016s
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_fileIO$ time ./readwrite 

real    0m11.713s
user    0m3.023s
sys     0m8.688s
```
可以看到，使用`fgetc fputc`的方法用时远远低于系统调用方法。  
使用`strace`命令来跟踪程序执行过程中的系统调用。  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_fileIO$ strace ./fgetcfputc 
...
read(3, "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"..., 4096) = 4096
write(4, "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"..., 4096) = 4096
read(3, "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"..., 4096) = 4096
write(4, "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"..., 4096) = 4096
...

xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_fileIO$ strace ./readwrite 
...
read(3, "a", 1)                         = 1
write(4, "a", 1)                        = 1
read(3, "a", 1)                         = 1
write(4, "a", 1)                        = 1
...
```
可以看到，`fgetc fputc`函数每次调用`read write`函数时都向内核读取、写入了4096个字节，而不是一个字节。而使用系统调用方法的程序，每次都真的向内核读取、写入了1个字节。

### 结果分析
原因是什么？可以参考这张图。  
![](/post_images/posts/Coding/【Linux系统编程】06/预读入和缓输出.jpg "预读入和缓输出")

对于写文件的操作，在内核中存在一个缓冲区（一般大小为4096字节），进行一次`write`，并不会直接将数据写入硬盘中，而是先将数据存入缓冲区，当缓冲区满之后一次性写入硬盘。（称为**缓输出**）  
- 对于`fputc`函数而言，当调用`fputc`写一个字节时，并不会直接调用`write`将其写入内核，而是在用户空间也进行了一次缓冲，只有当该缓冲区中的字节数达到4096，才会写入内核；  
- 对于直接使用系统调用而言，每写一个字节就进入内核一次。

对于读文件的操作，也会在内核中使用缓冲区，如果向内核请求读取一个字节的数据，内核不会从硬盘中只读一个字节，而是一次读取4096字节，如果用户再次申请后续的数据，就无需再次和硬盘进行交互。
- 从strace的结果可以看出，对于`fgetc`函数而言，读取数据不会只读一个字节，而是从内核一次读取4096个字节存入缓冲区，然后一个一个交付给用户；
- 对于直接使用系统调用而言，每读一个字节就进入内核一次。

从用户空间进入内核空间的操作是非常耗时的，`fgetc fputc`会使用缓冲区来减少用户空间到内核空间的切换次数；而直接使用系统调用，每读写一个字节就切换一次，时间消耗非常大。

### 总结
`read`和`write`函数常常被称为**Unbuffered IO**，也就是无缓冲IO，这里的缓冲是指用户级缓冲，不保证不使用内核缓冲。

系统调用和库函数各有优劣，即使我们学习了系统调用，在多数时候也要优先使用库函数。

在有些时候，我们希望尽快地将数据写到硬盘或其他终端设备中，可以考虑使用系统调用。

## 文件描述符
文件描述符的图示如下：  
![](/post_images/posts/Coding/【Linux系统编程】06/文件描述符.jpg "文件描述符")  
- 文件描述符这个概念是属于进程的，在PCB进程控制块中，有一个指针指向一个文件描述符表，每一个表项就是一个该进程打开的文件；
- 文件描述符（整数）可以理解为一个指针数组的索引（key），数组内容是指向文件结构体（`struct file`，文件的描述信息）的指针（value）；
- 操作系统不想让用户知道该表的具体细节，所以只将索引暴露给用户；
- 文件描述符表的前三项（0、1、2），分别对应标准输入、标准输出、标准错误，推荐使用对应的宏（`STDIN_FILENO`，`STDOUT_FILENO`，`STDERR_FILENO`）而非数字012；
- 进程打开的其他文件从`3`号开始，最大为`1023`号，所以一个进程最多打开1024个文件；
- 新打开的文件使用的文件描述符，是表中可用的最小的序号，例如已打开3、4、5、6号，此时关闭3号，下次打开文件的描述符就为3。

## 阻塞和非阻塞
产生阻塞的场景：读设备文件、读网络。（阻塞是设备文件、网络文件的属性，常规文件无阻塞概念）  

终端对应的文件为：`/dev/tty`，尝试使用这个文件。  
从标准输入（键盘）读数据，写到标准输出：  
```c
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
int main() {
    char buf[10];
    int read_bytes = read(STDIN_FILENO, buf, 10); // 阻塞
    if (n < 0) {
        perror("read STDIO_FILENO");
        exit();
    }
    write(STDOUT_FILENO, buf, read_bytes);
    return 0;
}
```
编译运行该程序，发现终端的光标在闪烁，该程序阻塞在了`read`函数处，随便输入一些字符，回车，阻塞状态结束，`read`读到了字符，并由`write`函数回显到终端。这说明`/dev/tty`这个文件默认就是阻塞的。

如果一个阻塞文件被读取，没有数据它就会等待数据；而一个非阻塞文件被读取，但是它没有数据，`read`会直接返回`-1`，并且`errno`会被设置为`EAGAIN`或`EWOULDBLOCK`，但这种情况不是读取错误，表示正在读取非阻塞文件而没有数据。

`/dev/tty`可以直接读，说明它已经是打开的状态，如果想要使用非阻塞方式，我们需要重新打开这个文件，并添加`O_NONBLOCK`flag，设备文件不需要关闭。  
```c
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <errno.h>
int main() {
    char buf[10];
    int fd = open("/dev/tty", O_RDONLY | O_NONBLOCK); // 非阻塞
    if (fd < 0) {
        perror("open /dev/tty");
        exit(1);
    }
    int read_bytes;
tryagain:
    read_bytes = read(fd, buf, 10);
    if (read_bytes < 0) {
        if (errno != EAGAIN) {
            perror("read /dev/tty");
            exit(1);
        } else {
            write(STDOUT_FILENO, "try again\n", strlen("try again\n"));
            sleep(2);
            goto tryagain;
        }
    }
    write(STDOUT_FILENO, buf, read_bytes);
    close(fd);
    return 0;
}
```
编译运行，可以看到每隔2秒，终端就会输出`try again`提示进行输入，输入一些字符后回车，终端就会回显这些字符并退出。

## fcntl
获取和设置文件访问模式属性。

上一部分我们使用重新打开文件的方法来设置文件的访问模式，其实可以不重新打开文件，直接使用`fcntl`函数进行设置即可。（file control）

函数原型：  
```c
#include <unistd.h>
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* arg */ );
```
- 返回值：
  - `-1`，出错，`errno`设置；
  - 成功情况的返回值和操作相关。
- `fd`：文件描述符；
- `cmd`：命令；
- `/*arg*/`：根据`cmd`决定后续的参数。

重点掌握的命令：
- `F_GETFL`：获取文件的访问模式（get file flags），不需要参数；
- `F_SETFL`：设置文件的访问模式（set file flags），需要一个`int flags`参数；
- 要给文件添加一个访问模式，可以使用`F_GETFL`获取文件的`flags`然后或等上需要添加的访问模式flag(bitmap)，再使用`F_SETFL`进行设置即可。

示例，非阻塞终端回显：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
int main() {
    char buf[10];
    int flags = fcntl(STDIN_FILENO, F_GETFL);
    if (flags == 1) {
        perror("fcntl error");
        exit(1);
    }
    flags |= O_NONBLOCK; // 设置为非阻塞
    int ret = fcntl(STDIN_FILENO, F_SETFL, flags);
    if (ret == -1) {
        perror("fcntl error");
        exit(1);
    }
    int read_bytes;
tryagain:
    read_bytes = read(STDIN_FILENO, buf, 10);
    if (read_bytes < 0) {
        if (errno != EAGAIN) {
            perror("read /dev/tty");
            exit(1);
        } else {
            write(STDOUT_FILENO, "try again\n", strlen("try again\n"));
            sleep(2);
            goto tryagain;
        }
    }
    write(STDOUT_FILENO, buf, read_bytes);
    return 0;
}
```

## lseek
定位读写文件的偏移值。

函数原型：
```c
#include <sys/types.h>
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);
```
- 返回值：从文件起始位置开始的偏移长度，出错返回`-1`，且设置`errno`；
- `fd`：文件描述符；
- `offset`：偏移量（可正可负）；
- `whence`：偏移的起始位置
  - `SEEK_SET`：起始字节；
  - `SEEK_CUR`：当前字节，假如已经读了10个字节，`SEEK_CUR == 11`;
  - `SEEK_END`：末尾后一个字节。

`lseek`应用场景：
- 文件的读和写使用的是同一个偏移量；
- 使用lseek获取文件大小（并不是非常正规的方式）`lseek(fd, 0, SEEK_END);`；
- 使用lseek拓展文件大小，`lseek(fd, 100, SEEK_END);`，在文件后追加100个字节，这里还没有真正拓展，如果要使文件正在拓展，必须在lseek之后引起IO操作（在结尾随便写一个字符`'\0'`即可，这样就追加了101个字节）。

补充：
- `od -tcx filename`：查看文件的 16 进制表示形式；
- `od -tcd filename`：查看文件的 10 进制表示形式；
- 使用`truncate`函数，直接拓展文件。`int ret = truncate("test.txt", 250)`，成功返回`0`，失败返回`-1`并设置`errno`;（它不能创建文件，只能拓展现有文件）