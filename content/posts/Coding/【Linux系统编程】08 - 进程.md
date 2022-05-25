---
title: 【Linux系统编程】08 - 进程
date: 2022-05-21
tags: [Linux]
categories: [Coding]
---

题外话，markdown表格不能光写表头。

## 进程相关概念
### 程序和进程
- 程序：非活动的，只占用磁盘空间；
- 进程：运行起来的程序，占用内存、CPU等系统资源。

### 并发
在操作系统中，一个时间段中有多个进程都处于已启动运行到运行完毕之间的状态。但任一个时刻点上仍只有一个进程在运行（单核CPU）。

单道程序设计，所有进程一个一个排对执行。若A阻塞，B只能等待，即使 CPU 处于空闲状态。而在人机交互时阻塞的出现时必然的。所有这种模型在系统资源利用上及其不合理，在计算机发展历史上存在不久，大部分便被淘汰了。

多道程序设计，在计算机内存中同时存放几道相互独立的程序，它们在管理程序控制之下，相互穿插的运行。多道程序设计必须有硬件基础（时钟中断）作为保证。

实质上，并发是宏观并行，微观串行！

### 虚拟内存和物理内存
![](/post_images/posts/Coding/【Linux系统编程】08/虚拟内存和物理内存.jpg "虚拟内存和物理内存")

### PCB 进程控制块
每个进程在内核中都有一个进程控制块（PCB）来维护进程相关的信息，Linux内核的进程控制块是 `task_struct` 结构体。

`/usr/src/linux-headers-5.13.0-40-generic/include/linux/sched.h` 文件中可以查看 `struct task_struct` 结构体定义。其内部成员有很多，我们重点掌握以下部分即可：
- **进程 id**，系统中每个进程有唯一的 id，在 C 语言中用 pid_t 类型表示，其实就是一个非负整数。（`ps aux`命令查看进程PID）
- **进程的状态**，有就绪、运行、挂起、停止等状态。
- 进程切换时需要保存和恢复的一些 CPU 寄存器。
- 描述虚拟地址空间的信息。
- 描述控制终端的信息。
- **当前工作目录**（Current Working Directory）。
- umask 掩码。
- **文件描述符表**，包含很多指向 file 结构体的指针。
- **和信号相关的信息**。
- **用户 id 和组 id**
- 会话（Session）和进程组。
- 进程可以使用的资源上限（Resource Limit）。

## 环境变量
环境变量，是指在操作系统中用来指定操作系统运行环境的一些参数。通常具备以下特征：
1. 字符串(本质) 
2. 有统一的格式：名=值[:值] 
3. 值用来描述进程环境信息。

存储形式：与命令行参数类似。`char *[]`数组，数组名 `environ`，内部存储字符串，`NULL`作为哨兵结尾。  
使用形式：与命令行参数类似。  
加载位置：与命令行参数类似。位于用户区，高于 `stack` 的起始位置。  
引入环境变量表：须声明环境变量。`extern char ** environ`。

常用环境变量：  
`echo $PATH`   查看环境变量
path环境变量里记录了一系列的值，当运行一个可执行文件时，系统会去环境变量记录的位置里查找这个文件并执行。
`echo $TERM`  查看终端
`echo $LANG`  查看语言
`env`         查看所有环境变量

## 进程控制
### fork
创建子进程。

函数原型：  
```c
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);
```
- 返回值：失败在父进程中返回`-1`，成功，父进程中返回子进程的`PID`，子进程中返回`0`。

![](/post_images/posts/Coding/【Linux系统编程】08/fork函数原理.jpg "fork函数原理")

### fork测试
代码：  
```c
#include <unistd.h>
#include <sys/types.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
int main(int argc, char* argv[]) {
    printf("before fork *\n");
    printf("before fork **\n");
    printf("before fork ***\n");
    printf("before fork ****\n");
    pid_t pid = fork();
    if (pid == -1) {
        perror("fork error");
        exit(1);
    } else if (pid == 0) {
        printf("-- child process : I'm created by %d\n", getppid());
        printf("-- child process : I'm %d\n", getpid());
    } else if (pid > 0) {
        printf("-- parent process : I'm created by %d\n", getppid());
        printf("-- parent process : my child is %d\n", pid);
        printf("-- parent process : I'm %d\n", getpid());
    }
    sleep(1);
    printf("after fork\n");
    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_process$ ./test_fork 
before fork *
before fork **
before fork ***
before fork ****
-- parent process : I'm created by 13830
-- parent process : my child is 14420
-- parent process : I'm 14419
-- child process : I'm created by 14419
-- child process : I'm 14420
after fork
after fork
```
父进程实际上是bash的子进程。

### getpid / getppid
- 获取当前进程PID；
- 获取父进程的PID。

函数原型：  
```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);
```

### 循环创建n个子进程
代码：  
```c
#include <unistd.h>
#include <sys/types.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
int main(int argc, char* argv[]) {
    int i;
    for (i = 0; i < 5; ++ i) {
        if (fork() == 0) {// 子进程
            break; // 如果不break 子进程还会创建它的子进程
        }
    }
    if (i == 5) {
        sleep(1); // 保证父进程最后结束
        printf("I'm parent\n");
    } else {
        printf("I'm %dth child\n", i + 1); // 标识自己的身份
    }
    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_process$ ./test_fork1 
I'm 1th child
I'm 2th child
I'm 4th child
I'm 3th child
I'm 5th child
I'm parent
```

### getuid / geteuid / getgid / getegid
- 获取当前进程实际用户ID；
- 获取当前进程有效用户ID；
- 获取当前进程使用用户组ID；
- 获取当前进程有效用户组ID；

### 父子进程共享的内容
父子进程共享的内容：
- 文件描述符；
- mmap映射区。

父子进程不同的内容：
- 进程PID；
- fork的返回值；
- 父进程的PID；
- 进程运行时间；
- 定时器；
- 未决信号集。

父子进程相同的内容：（刚fork之后相同的内容）
- 全局变量；
- data段；
- text段；
- 栈；
- 堆；
- 环境变量；
- 用户ID；
- 宿主目录；
- 进程工作目录；
- 信号处理方式。

似乎，子进程复制了父进程在0-3GiB用户空间上的内容以及父进程的PCB（除了PID），但是，子进程并不是将父进程0-3GiB用户空间完全拷贝一份然后映射到物理内存，而是遵循**读时共享，写时复制**的原则，以节省内存开销。

注意：父子进程的全局变量不是共享的！我们可以进行测试：  
```c
#include <unistd.h>
#include <sys/types.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
int global_var = 100;
int main(int argc, char* argv[]) {
    pid_t pid = fork();
    if (pid == -1) {
        perror("fork error\n");
        exit(1);
    } else if (pid == 0) {
        global_var = 200;
        printf("-- child process : pid = %d, ppid = %d\n", getpid(), getppid());
        printf("-- child process : global_var = %d\n", global_var);
    } else {
        global_var = 300;
        printf("-- parent process : pid = %d, ppid = %d\n", getpid(), getppid());
        printf("-- parent process : global_var = %d\n", global_var);
    }
    printf("--finish--\n");
    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_process$ ./fork_shared 
-- parent process : pid = 15297, ppid = 13830
-- parent process : global_var = 300
--finish--
-- child process : pid = 15298, ppid = 15297
-- child process : global_var = 200
--finish--
```

注，fork之后先执行父进程还是子进程是不确定的，取决于内核使用的调度算法。

### fork - gdb调试
设置父进程调试路径：`set follow-fork-mode parent` (默认)

设置子进程调试路径：`set follow-fork-mode child`
注意，一定要在fork函数调用之前设置才有效。

## exec函数族
`fork`创建子进程后执行的是和父进程相同的程序（但有可能执行不同的代码分支），子进程往往要调用一种 `exec` 函数以执行另一个程序。

当进程调用一种 `exec` 函数时，该进程的用户空间代码和数据完全被新程序替换，从新程序的启动例程开始执行。

调用 `exec` 并不创建新进程，所以调用 `exec` 前后该进程的 PID 并未改变。

将当前进程的`.text`、`.data` 替换为所要加载的程序的`.text`、`.data`，然后让进程从新的`.text`第一条指令开始执行，但进程 PID 不变，换核不换壳。

### execlp
加载一个进程，借助`PATH`环境变量。（库函数）

函数原型：  
```c
#include <unistd.h>

int execlp(const char *file, const char *arg, ...
                /* (char  *) NULL */);
```
- 返回值：成功无返回值，失败返回`-1`；
- `file`：要加载的程序名字，在`PATH`的目录中进行查找，找不到就出错返回；
- `arg`：程序的参数，需要以`NULL`结尾（作为哨兵），例如`execlp("ls", "ls", "-l", "-h", NULL);`，注意，`arg`从`argv[0]`开始算起；
- 该函数常用来调用系统程序，如`ls date cp cat`等命令。

### execl
加载一个进程，通过路径和程序名来加载。

函数原型：  
```c
#include <unistd.h>

int execl(const char *pathname, const char *arg, ...
                /* (char  *) NULL */);
```
- 返回值：成功无返回值，失败返回`-1`；
- `pathname`：程序的路径名，绝对路径和相对路径都行；
- `arg`：同样是从`argv[0]`开始算起，以哨兵`NULL`结尾；
- 例如，`execl("/bin/ls", "ls", "-l", NULL);`。

### 练习 - 将系统进程信息保存到文件中
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
int main(int argc, char* argv[]) {
    int fd = open("ps.out", O_WRONLY | O_CREAT | O_TRUNC, 0664);
    if (fd == -1) {
        perror("open ps.out error");
        exit(1);
    }
    dup2(fd, STDOUT_FILENO);
    execlp("ps", "ps", "aux", NULL);
    // close(fd); // 这里close没用 因为execlp会从之前的进程切换 之后可以使用信号知识处理 暂时先不管
    return 0;
}
```

### execvp
加载一个进程，通过`PATH`，和参数数组。和`execlp`基本相同。

函数原型：  
```c
#include <unistd.h>

int execvp(const char *file, char *const argv[]);
```
- 返回值：成功无返回值，失败返回`-1`；
- `file`：程序名；
- `argv`：参数列表；
- 例如，`char* argv[] = {"ls", "-l", NULL}; execvp("ls", argv);`。

### exec函数族总结
`exec` 函数一旦调用成功即执行新的程序，不返回。只有失败才返回，错误值`-1`。所以通常我们直接在 `exec` 函数调用后直接调用 `perror()`和 `exit()`，无需 `if` 判断。

- `l (list)` 命令行参数列表
- `p (path)` 搜素 file 时使用 path 变量
- `v (vector)` 使用命令行参数数组
- `e (environment)` 使用环境变量数组,不使用进程原有的环境变量，设置新加载程序运行的环境变量

事实上，只有 `execve` 是真正的系统调用，其它五个函数最终都调用 `execve`，所以 `execve`在 man 手册第 2 节，其它函数在 man 手册第 3 节。这些函数之间的关系如下图所示。  
![](/post_images/posts/Coding/【Linux系统编程】08/exec函数族.jpg "exec函数族")

## 回收子进程
### 孤儿进程
通常来说，进程都是从某个进程`fork`而来，而当父进程先于子进程结束，则子进程成为孤儿进程，子进程的父进程变为`init`进程（进程孤儿院），这个事件称为*`init`进程领养孤儿进程*。

### 僵尸进程
子进程的残留资源应被父进程回收，当子进程终止，而父进程尚未回收时，子进程残留资源（PCB）存放于内核中，变成僵尸进程。

当父进程尚未结束，而子进程已经结束，但是父进程没有处理子进程的退出状态（获取子进程的退出码，退出码写在PCB中），子进程的PCB会保留一段时间等待父进程获取退出码，直到父进程获取了子进程的退出状态，子进程的PCB才会被移除。

不能用`kill`命令来解决僵尸进程，因为它已经死了。可以杀掉父进程，`init`进程领养后会释放资源。

### wait
一个进程在终止时会关闭所有文件描述符，释放在用户空间分配的内存，但它的 `PCB`还保留着，内核在其中保存了一些信息：如果是正常终止则保存着退出状态，如果是异常终止则保存着导致该进程终止的信号是哪个。

这个进程的父进程可以调用 `wait` 或 `waitpid` 获取这些信息，然后彻底清除掉这个进程。

我们知道一个进程的退出状态可以在 Shell 中用特殊变量`$?`查看，因为 Shell 是它的父进程，当它终止时 Shell 调用 `wait` 或 `waitpid` 得到它的退出状态同时彻底清除掉这个进程。

`wait`函数有以下三个功能：
1. 阻塞，等待子进程退出；
2. 回收子进程残留资源；
3. 获取子进程结束状态。

函数原型：  
```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *wstatus);
```
- 返回值：成功，清理掉的子进程PID，失败，`-1`（没有子进程）；
- `wstatus`：用来保存进程的退出状态，借助宏函数进行判断；（也可传入`NULL`，表示不关心进程如何终止）

#### 获取子进程退出值和异常终止信号
用到的宏函数：
- `WIFEXITED(wstatus)`，非0，子进程正常终止；
  - `WEXITSTATUS(wstatus)`，子进程正常退出情况下的退出状态（`exit()`的参数）。
- `WIFSIGNALED(wstatus)`，非0，子进程被异常终止；
  - `WTERMSIG(wstatus)`，使得子进程异常终止的**信号**。
- `WIFSTOPPED(wstatus)`，非0，进程处于暂停状态（阻塞）；（非重点）
  - `WSTOPSIG(wstatus)`，使得子进程暂停的**信号**；
  - `WIFCONTINUED(wstatus)`，进程暂停后已经继续运行。

测试代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(int argc, char* argv[]) {
    pid_t pid = fork();
    if (pid == 0) { // 子进程
        printf("-- child process : my pid = %d, sleep 10s\n", getpid());
        sleep(10);
        printf("-- child process : die\n");
    } else if (pid > 0) { // 父进程
        int wstatus;
        pid_t wpid = wait(&wstatus);
        if (wpid == -1) {
            perror("-- parent process : wait error");
            exit(1);
        }
        if (WIFEXITED(wstatus)) {
            printf("-- parent process : child exit with %d\n", WEXITSTATUS(wstatus));
        } else if (WIFSIGNALED(wstatus)) {
            printf("-- parent process : child killed by signal %d\n", WTERMSIG(wstatus));
        }
    }
    return 0;
}
```

### waitpid
作用和`wait`相同，但可指定PID，可选非阻塞模式。

函数原型：  
```c
#include <sys/types.h>
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int *wstatus, int options);
```