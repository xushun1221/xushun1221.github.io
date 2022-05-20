---
title: 【Linux系统编程】07 - 文件系统编程
date: 2022-05-18
tags: [Linux]
categories: [Coding]
---

## Inode和dentry
Inode的本质是一个结构体，存储文件的属性信息。如：权限、类型、大小、时间、用户、盘块位置等等。也叫作文件属性管理结构，大多数的Inode都存储在磁盘上。少量常用、近期使用的Inode会被缓存到内存中。

dentry（directory entry），目录项，本质也是一个结构体，重要的成员变量有两个（文件名，指向Inode的指针，...），目录项所对应的文件存放在磁盘中。

dentry和Inode的关系可以参考下图：  
![](/post_images/posts/Coding/【Linux系统编程】07/dentry和Inode.jpg "dentry和Inode")

- 我们的数据存储在磁盘上的某个区域内，但是我们需要一种结构来帮助我们找到这些数据存放的位置，这个结构就是Inode；
- 同时我们也要通过文件名来找到这个Inode，使用dentry目录项结构来封装文件名和Inode编号，通过Inode编号我们就能找到Inode结构体从而找到文件数据；
- 一个文件的主要就是由目录项dentry和Inode组成；
- 操作系统对文件创建硬链接的原理是，各个硬链接都有相同的Inode号，只是dentry不同，删除一个硬链接实际上就是去掉一个dentry。

## stat / lstat
获取文件的状态。（从Inode中获得）

函数原型：  
```c
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

int stat(const char *pathname, struct stat *statbuf);
```
- 返回值：成功`0`，失败`-1`且设置`errno`；
- `path`：文件路径；
- `statbuf`：返回参数，用于返回文件属性；（stat结构体参考manpages）

### 查询文件大小和类型
demo：查询文件大小和类型  
```c
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
int main(int argc, char* argv[]) {
    struct stat sbuf;
    int ret = stat(argv[1], &sbuf);
    if (ret == -1) {
        perror("stat error");
        exit(1);
    }
    printf("file size : %ld\n", sbuf.st_size);
    // 在 man 7 inode 中查看 st_mode的详细信息
    // 使用宏函数判断文件类型
    printf("file mode : ");
    if (S_ISREG(sbuf.st_mode)) {
        printf("regular file\n");
    } else if (S_ISDIR(sbuf.st_mode)) {
        printf("directory\n");
    } else if (S_ISFIFO(sbuf.st_mode)) {
        printf("FIFO\n");
    } else if (S_ISLNK(sbuf.st_mode)) {
        printf("symbolic link\n");
    }
    // 还有四种不写了
    return 0;
}
```
上述程序可以判断文件类型是否目录或是常规文件，但是`S_ISLNK`的判断会出错，并不能指出是链接类型，而是指出链接文件所指向的文件。这种现象被称为**stat穿透**。

### stat穿透现象
要解决这个问题需要使用`lstat`函数（`stat`直接替换为`lstat`即可），它不会穿透符号链接。

### 掩码方式判断文件类型 
文件的类型和权限`stat.st_mode`是用一个2Bytes（16bits）的bitmap来存储的，如下表所示。  
|文件类型|文件类型|文件类型|文件类型|所有者ID|所属组ID|黏着位|所有者r|所有者w|所有者x|所属组r|所属组w|所属组x|其他用户r|其他用户w|其他用户x|
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
|||||||||||||||||

`S_IFMT`就是用来获得文件类型的掩码，它的值为`0 170000`这是一个8进制数，用上面表格来表示：  
|文件类型|文件类型|文件类型|文件类型|特殊权限位|特殊权限位|特殊权限位|所有者r|所有者w|所有者x|所属组r|所属组w|所属组x|其他用户r|其他用户w|其他用户x|
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
|1|1|1|1|0|0|0|0|0|0|0|0|0|0|0|0|

使用`S_IFMT`和`stat.st_mode`进行`与`运算，就可以得到相应的文件类型，七种文件类型的表示码如下：  
|文件类型宏定义|8进制表示|类型|
|---|---|---|
|S_IFSOCK |0 140000  | socket|
|S_IFLNK  |0 120000  | symbolic link|
|S_IFREG  |0 100000  | regular file|
|S_IFBLK  |0 060000  | block device|
|S_IFDIR  |0 040000  | directory|
|S_IFCHR  |0 020000  | character device|
|S_IFIFO  |0 010000  | FIFO|

下面是一个显示文件所有信息的例程：  
```c
#include <sys/types.h>
#include <sys/stat.h>
#include <time.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/sysmacros.h>

int main(int argc, char *argv[])
{
    struct stat sb;
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <pathname>\n", argv[0]);
        exit(EXIT_FAILURE);
    }
    if (lstat(argv[1], &sb) == -1) {
               perror("lstat");
               exit(EXIT_FAILURE);
           }
    printf("ID of containing device:  [%lx,%lx]\n",
        (long) major(sb.st_dev), (long) minor(sb.st_dev));
    printf("File type:                ");
    switch (sb.st_mode & S_IFMT) {
    case S_IFBLK:  printf("block device\n");            break;
    case S_IFCHR:  printf("character device\n");        break;
    case S_IFDIR:  printf("directory\n");               break;
    case S_IFIFO:  printf("FIFO/pipe\n");               break;
    case S_IFLNK:  printf("symlink\n");                 break;
    case S_IFREG:  printf("regular file\n");            break;
    case S_IFSOCK: printf("socket\n");                  break;
    default:       printf("unknown?\n");                break;
    }
    printf("I-node number:            %ld\n", (long) sb.st_ino);
    printf("Mode:                     %lo (octal)\n",
            (unsigned long) sb.st_mode);
    printf("Link count:               %ld\n", (long) sb.st_nlink);
    printf("Ownership:                UID=%ld   GID=%ld\n",
            (long) sb.st_uid, (long) sb.st_gid);
    printf("Preferred I/O block size: %ld bytes\n",
            (long) sb.st_blksize);
    printf("File size:                %lld bytes\n",
            (long long) sb.st_size);
    printf("Blocks allocated:         %lld\n",
            (long long) sb.st_blocks);
    printf("Last status change:       %s", ctime(&sb.st_ctime));
    printf("Last file access:         %s", ctime(&sb.st_atime));
    printf("Last file modification:   %s", ctime(&sb.st_mtime));
    exit(EXIT_SUCCESS);
}
```

## link / unlink
### link
给文件创建（硬）链接。（给文件创建一个新dentry）

函数原型：  
```c
#include <unistd.h>

int link(const char *oldpath, const char *newpath);
```
- 返回值：成功`0`，失败`-1`且设置`errno`。

### unlink
删除一个文件的目录项（硬链接计数减一）。

函数原型：  
```c
#include <unistd.h>

int unlink(const char *pathname);
```
- 返回值：成功`0`，失败`-1`且设置`errno`。

Linux下删除文件的机制：不断将`st_nlink`减一，直到减到0为止。无目录项dentry对应的文件，系统会择机释放。

我们删除文件，实际上只是让文件具备了被释放的条件。

`unlink`函数的特征，清除文件时，如果文件的硬链接计数为0了，没有dentry对应，但该文件不会被立即释放，要等到所有打开该文件的进程关闭该文件，系统才会挑时间将该文件释放掉。

写一个demo测试一下该特性：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>

int main() {
    int fd = open("test.txt", O_RDWR | O_CREAT | O_TRUNC, 0664);
    if (fd == -1) {
        perror("open test.txt error");
        exit(1);
    }
    int ret;
    // // ******1*******
    // ret = unlink("test.txt");
    // if (ret == -1) {
    //     perror("unlink error");
    //     exit(1);
    // }
    ret = write(fd, "test of unlink\n", strlen("test of unlink\n"));
    if (ret == -1) {
        perror("write error");
        exit(1);
    }
    printf("write success\n");
    ret = write(fd, "after write something\n", strlen("after write something\n"));
    if (ret == -1) {
        perror("write error");
        exit(1);
    }
    printf("enter anykey continue\n");
    getchar();
    // ******2*******
    ret = unlink("test.txt");
    if (ret == -1) {
        perror("unlink error");
        exit(1);
    }
    close(fd);
    return 0;
}
```
编译运行该程序。会发现文件`test.txt`被成功创建并写入，当输入字符结束阻塞后，该文件被关闭，程序运行结束后，文件被释放。

将代码段2注释掉，代码段1取消注释，编译运行。（将`unlink`放在前面的意义是，刚`open`就`unlink`，即使程序后面出错，也会将该文件删除掉）发现文件依然可以被成功创建和写入（这体现了`unlink`的特性，即使文件被`unlink`，它也会在所有打开该文件的进程关闭它之后，才会被删除。），但是在阻塞时，找不到该文件，这是由于内容并没有被写入到磁盘上，而是在内核空间的缓冲区中。

## 隐式回收
当进程结束运行时，所有进程打开的文件会被关闭，申请的内存空间会被释放。系统的这一特性称之为隐式回收系统资源。

比如上面那个程序，要是没有在程序中关闭文件描述符，没有隐式回收的话，这个文件描述符会保留，多次出现这种情况会导致系统文件描述符耗尽。所以隐式回收会在程序结束时收回它打开的文件使用的文件描述符。

但是最好不要依赖这个特性。

## readlink
读取符号链接本身的内容，得到链接所指向的文件名。

函数原型：  
```c
#include <unistd.h>

ssize_t readlink(const char *pathname, char *buf, size_t bufsiz);
```
- 返回值：成功返回`0`，失败返回`-1`并设置`errno`；
- `path`：链接的路径；
- `buf`：保存内容的缓冲区。

## rename
重命名一个文件。（其实和mv一样）

函数原型：  
```c
#include <stdio.h>

int rename(const char *oldpath, const char *newpath);
```
- 返回值：成功返回`0`，失败返回`-1`并设置`errno`。

## getcwd
获得当前进程的工作目录。（标准库函数）

函数原型：  
```c
#include <unistd.h>

char *getcwd(char *buf, size_t size);
```
- 返回值：成功返回当前工作目录字符串指针，失败返回`NULL`；
- `buf size`：存放当前目录字符串，buf的大小。

## chdir
改变当前进程的工作目录。（系统函数）

函数原型：  
```c
#include <unistd.h>

int chdir(const char *path);
```
- 返回值：成功返回`0`，失败返回`-1`并设置`errno`；
- `path`：新的工作目录。

## 文件、目录的权限
在Linux系统中，所见皆文件。目录也是文件，其文件内容是该目录下所有子文件的目录项（dentry），可以尝试用vim打开一个目录。

文件和目录文件的权限含义有所不同：  
|.|r|w|x|
|---------|---------|---------|---------|
|文件|文件的内容可以被查看 cat,more,less|文件可以被修改 vi|可以运行产生一个进程 ./xx|
|目录|目录可以被浏览 ls,tree|创建、删除、修改文件 mv,touch,mkdir|可以被打开、进入 cd|

## opendir
打开一个目录。（标准库函数）

函数原型：  
```c
#include <sys/types.h>
#include <dirent.h> // directory entry

DIR *opendir(const char *name);
```
- 返回值：成功返回指向该目录结构体的指针，失败返回`NULL`设置`errno`；
- `name`：参数支持相对路径和绝对路径两种方式。

## closedir
关闭打开的目录。（标准库函数）

函数原型：  
```c
#include <sys/types.h>
#include <dirent.h>

int closedir(DIR *dirp);
```
- 返回值：成功返回`0`，失败返回`-1`并设置`errno`。

## readdir
读取目录内容。（标准库函数）

函数原型：  
```c
#include <dirent.h>

struct dirent *readdir(DIR *dirp);

struct dirent {
    ino_t          d_ino;       /* Inode number */
    off_t          d_off;       /* Not an offset; see below */
    unsigned short d_reclen;    /* Length of this record */
    unsigned char  d_type;      /* Type of file; not supported
                                              by all filesystem types */
    char           d_name[256]; /* Null-terminated filename */
};

```
- 返回值：成功返回目录项结构体指针，失败返回`NULL`设置`errno`，读到结尾返回`UNLL`不设置`error`；
- `struct dirent`：`ino_t`和`char d_name[256]`重要。

## mkdir
创建一个目录。（系统函数）

函数原型：  
```c
#include <sys/stat.h>
#include <sys/types.h>

int mkdir(const char *pathname, mode_t mode);
```
- 返回值：成功返回`0`，失败`-1`设置`errno`；
- `pathname`：目录名；
- `mode`：访问模式，权限。

`mode`计算方式：`mode & (~umask)`。  
终端命令`umask`查看，`umask`的值为`0002`，`~umask`就是`0775`。

## 练习 - 实现ls
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <dirent.h>
int main(int argc, char* argv[]) {
    DIR* dp = opendir(argv[1]);
    if (dp == NULL) {
        perror("opendir error");
        exit(1);
    }
    struct dirent* sdp;
    while ((sdp = readdir(dp)) != NULL) {
        printf("%s\n", sdp-> d_name);
    }
    closedir(dp);
    return 0;
}
```

## 练习 - 实现递归遍历目录
思路：
1. 判断命令行参数`argv[1]`，获取用户查询的目录名；
2. 判断用户指定的是否为目录，`stat S_ISDIR()`，封装一个函数；
3. 读目录`opendir readdir closedir`
   1. 普通文件，直接打印；
   2. 目录文件，拼接访问的绝对路径，递归调用；

代码：  
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <dirent.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>

void fetchdir(char* dirpath);
void printFile(char* path);

void fetchdir(char* dirpath) {
    DIR* dp = opendir(dirpath);
    if (dp == NULL) {
        fprintf(stderr, "fetchdir: can't open %s\n", dirpath);
        exit(1);
    }
    struct dirent* sdp;
    char subpath[257];
    while ((sdp = readdir(dp)) != NULL) {
        if (strcmp(sdp -> d_name, ".") == 0 || strcmp(sdp -> d_name, "..") == 0)
            continue; // ；处理"."".."目录 防止无限递归
        if (strlen(dirpath) + strlen(sdp -> d_name) + 2 > 256) { // 路径长度太长
            fprintf(stderr, "fetchdir: name %s %s too long\n", dirpath, sdp -> d_name);
        } else {
            sprintf(subpath, "%s/%s", dirpath, sdp -> d_name); // 拼接路径
            printFile(subpath); // 递归
        }
    }
    closedir(dp);
}

void printFile(char* path) {
    struct stat sbuf;
    if (lstat(path, &sbuf) == -1) {
        fprintf(stderr, "isFile: can't access %s\n", path);
        exit(1);
    }
    if ((sbuf.st_mode & S_IFMT) == S_IFDIR) { // 如果是目录 就展开
        fetchdir(path);
    } else { // 普通文件 直接打印 文件大小 和 路径
        printf("%8ld %s\n", sbuf.st_size, path);
    }
}

int main(int argc, char* argv[]) {
    if (argc == 1) { // 没有命令行参数默认当前目录
        printFile(".");
    } else {
        while (-- argc > 0) { // 处理多个命令行参数
            printFile(*++ argv);
        }
    }
    return 0;
}
```

## 重定向 dup / dup2
复制一个文件描述符。（系统函数）

函数原型：  
```c
#include <unistd.h>

int dup(int oldfd);
int dup2(int oldfd, int newfd); // dup to
```
- 返回值：成功，返回新的文件描述符，失败返回`-1`设置`errno`；
- `oldfd`：已有的文件描述符；
- `newfd`：新的文件描述符（旧的复制给新的）。

- `dup`返回的文件描述符是，未使用的最小的文件描述符，它是`oldfd`的一个复制，和`oldfd`指向相同的文件。`dup`基本就起一个保存文件描述符的作用；
- `dup2`返回的文件描述符是，`newfd`，也是`oldfd`的复制，和`oldfd`指向相同的文件；
- 区别在于，`dup`自动分配，`dup2`可以指定。

一般用该函数来进行重定向的功能，demo，将写入屏幕的内容重定向到文件：  
```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
int main(int argc, char* argv[]) {
    int fd = open("test.txt", O_RDWR);
    if (fd == -1) {
        perror("open error");
        exit(1);
    }
    // 把标准输出重定向到test.txt
    if (dup2(fd, STDOUT_FILENO) == -1) {
        perror("dup2 error");
        exit(1);
    }
    printf("dup2 test write to STDOUT\n");
    close(fd);
    return 0;
}
```
编译运行，本来要输入到`STDOUT`的内容，被重定向到`test.txt`文件中。

注：`fcntl`函数也可以实现dup操作。