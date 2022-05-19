# 【Linux系统编程】07 - 文件系统


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

要解决这个问题需要使用`lstat`函数（`stat`直接替换为`lstat`即可），它不会穿透符号链接。

掩码方式判断文件类型：
