# 【Linux系统编程】11 - 守护进程、线程


## 进程组和会话
### 进程组 - 概念和特性
**进程组**，也称之为**作业**。BSD 于 1980 年前后向 Unix 中增加的一个新特性。代表一个或多个进程的集合。每个进程都属于一个进程组。在 `waitpid` 函数和 `kill` 函数的参数中都曾使用到。操作系统设计的进程组的概念，是为了简化对多个进程的管理。

- 当父进程，创建子进程的时候，默认子进程与父进程属于同一进程组。进程组 ID==第一个进程 ID(组长进程)。所以，组长进程标识：其进程组 ID==其进程 ID
- 可以使用 kill -SIGKILL -进程组 ID(负的)来将整个进程组内的进程全部杀死。
- 组长进程可以创建一个进程组，创建该进程组中的进程，然后终止。只要进程组中有一个进程存在，进程组就存在，与组长进程是否终止无关。
- 进程组生存期：进程组创建到最后一个进程离开(终止或转移到另一个进程组)。
- 一个进程可以为自己或子进程设置进程组 ID

### 会话
**会话**，是多个进程组的集合。

创建会话的注意事项：  
1. **创建会话的进程不能是进程组组长**，该进程变成新会话首进程（session header）
2. 该进程成为一个新进程组的组长进程
3. 需要root权限（ubuntu不需要）
4. 新会话丢弃原有的控制终端，该会话**没有控制终端**
5. 该调用进程是组长进程，则出错返回
6. 建立新会话时，先调用fork，父进程终止，子进程调用`setsid()`

### getsid
获取进程所属会话id。（系统函数）

函数原型：  
```c
#include <sys/types.h>
#include <unistd.h>

pid_t getsid(pid_t pid);
```
- 返回值：
  - 成功，返回进程的会话id
  - 失败，返回`-1`设置`errno`
- `pid`：为0表示查看当前进程的会话id

注，`ps ajx`命令可以参考系统中的进程信息。

### setsid
创建一个会话，并以自己的pid设置进程组id，同时也是新会话的id。（系统调用）

函数原型：  
```c
#include <sys/types.h>
#include <unistd.h>

pid_t setsid(void);
```
- 返回值：
  - 成功，返回调用进程的会话id
  - 失败，返回`-1`设置`errno`

调用了setsid的进程既是新会话的会长，也是新进程组的组长。  
组长进程不能成为新会话首进程，新会话首进程必定会成为组长进程。（父进程fork，然后结束，子进程setsid）

## 守护进程
**Daemon(精灵)进程**，是 Linux 中的后台服务进程，通常独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。一般采用以 `d` 结尾的名字。

Linux 后台的一些系统服务进程，没有控制终端，不能直接和用户交互。不受用户登录、注销的影响，一直在运行着，他们都是守护进程。如：预读入缓输出机制的实现；ftp 服务器；nfs 服务器等。

创建守护进程，最关键的一步是调用 `setsid` 函数创建一个新的 Session，并成为 Session Leader。

### 创建守护进程模型
1. 创建子进程，父进程退出
   - 所有工作在子进程中进行形式上脱离了控制终端
2. 在子进程中创建新会话
   - `setsid()`函数
   - 使子进程完全独立出来，脱离控制
3. 改变当前目录位置
   - `chdir()`函数
   - 防止占用可卸载的文件系统
   - 也可以换成其它路径
4. 重设文件权限掩码
   - `umask()`函数
   - 防止继承的文件创建屏蔽字拒绝某些权限
   - 增加守护进程灵活性
5. 关闭或重定向文件描述符
   - 继承的打开文件不会用到，浪费系统资源，无法卸载
   - 主要是要关闭`0 1 2`号文件描述符，也可以将它们重定向到`/dev/null`
6. 开始执行守护进程核心工作

### 练习 - 创建守护进程
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>
#include <fcntl.h>

int main(int argc, char** argv) {
    // 1. fork and shutdown parent
    if (fork() > 0) {
        exit(0);
    }
    // child begin
    // 2. new session
    if (setsid() == -1) { 
        perror("setsid error");
        exit(1);
    }
    // 3. change work dir
    if (chdir("/home/xushun/LinuxSysPrograming/test_session") == -1) {
        perror("chdir error");
        exit(1);
    }
    // 4. reset umask : default mode == 755
    umask(0022);
    // 5.close and dup fd
    close(STDIN_FILENO); // close fd-0
    if (open("/dev/null", O_RDWR) == -1) { // fd(/dec/null)-0
        perror("open error");
        exit(1);
    }
    dup2(0, STDOUT_FILENO); // dup STDOUT to 0
    dup2(0, STDERR_FILENO); // dup STDERR to 0
    // 6. service logic
    while (1) { }
    return 0;
}
```
编译运行，该守护进程成功创建，它没有终端进行控制，用户的登录和注销也不会对它产生影响，必须使用`kill`命令来终止它。

## 线程

### 线程的相关概念
**线程**，也可以称为**LWP**（Light Weight Process）轻量级的进程，在Linux下，它的本质仍然是进程。

- 进程，有独立的地址空间，拥有PCB
- 线程，有独立的PCB，但是没有独立的地址空间（共享的）
- 区别，在于是否共享地址空间
- 进程可以变成线程
- 线程可以看作寄存器和栈的集合
- 线程是**最小的执行单位**，CPU执行时，不是以进程为单位，而是以线程为单位
- 进程是**最小的资源分配单位**，进程也可以看作是只有一个线程的进程

当进程中只有它自己没有其他线程时，我们只会提进程的概念，如果创建了线程，才会提线程的概念。

`ps -Lf pid`，可以查看线程号（LWP）。

- 线程号（LWP），是给CPU用于区分线程，进行执行调度用的；
- 线程ID，是在进程内部用来区分不同线程身份的。

### Linux线程的实现原理
类 Unix 系统中，早期是没有**线程**概念的，80 年代才引入，借助进程机制实现出了线程的概念。因此在这类系统中，进程和线程关系密切。

从内核里看，进程和线程是一样的，都有自己的PCB，但是线程PCB中指向内存资源的三级页表是相同的。

对于进程来说，相同的地址（同一个虚拟地址）在不同的进程中，反复使用而不冲突，原因是它们虽然虚拟地址相同，但是页目录、页表、物理页面都各不相同。即相同的虚拟地址，映射到不同的物理页面内存单元，最终访问不同的物理页面。

但是线程不同，两个线程具有各自独立的PCB，但是共享一个页目录（意味着它们的虚拟地址到物理地址的映射相同），所以两个PCB共享一个地址空间。

实际上，无论是创建进程的`fork()`，还是创建线程的`pthread_create()`，底层的实现都是调用同一个内核函数`clone()`。

如果复制对方的地址空间，那么就产生一个进程，如果共享对方的地址空间，就产生一个线程。

因此，Linux系统内核是不区分进程和线程的，只在用户层面上进行区分。所以，线程的所有操作函数`pthread_*`都是库函数，而非系统调用。

### 线程共享的资源
1. 文件描述符表
2. 每种信号的处理方式
3. 当前工作目录
4. 用户ID和组ID
5. 内存地址空间（.text/.data/heap/共享库）

牢记：线程默认共享数据段、代码段等地址空间，常用的是**全局变量**。而进程不共享全局变量，只能借助 mmap 共享映射区。

### 线程非共享的资源
1. 线程ID（和线程号不同）
2. 处理器现场（寄存器）和栈指针（内核栈）
3. 独立的栈空间（用户空间栈）
4. `errno`变量
5. 信号屏蔽字
6. 调度优先级

### 线程的优、缺点
优点：
1. 提高程序并发性
2. 开销小
3. 数据通信、共享方便

缺点：
1. 库函数，不稳定
2. 调试编写困难、gdb不支持
3. 对信号支持不好

线程的优点比较突出，缺点并不是硬伤，Linux下由于实现方法导致进程、线程的差别不是很大。

## 线程控制原语
编译时，需要加上`-pthread`参数。

### pthread_self
获得自己的线程ID。

函数原型：  
```c
#include <pthread.h>

pthread_t pthread_self(void);
```
- 返回值：
  - 成功，返回该线程的线程ID
  - 不会失败

`pthread_t`是长整型`unsigned long`。

### pthread_create
创建一个线程。

函数原型：  
```c
#include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                    void *(*start_routine) (void *), void *arg);
```
- 返回值：
  - 成功，返回`0`；
  - 失败，返回一个错误号，并不会设置`errno`。
- `thread`：新线程的ID，成功会设置tid，失败则不会设置；
- `attr`：设置新线程属性，如果为`NULL`则使用默认属性；
- `start_routine`：新线程即将进入的执行函数；（我们把该函数称为子线程，而main函数称为主线程）
- `arg`：（`start_routine`的参数，没有传`NULL`）向新线程传递的某些参数。

### 练习1 - 创建一个线程
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>

void* tfunc(void* arg) {
    printf("--thread : pid = %d, tid = %lu\n", getpid(), pthread_self());
    return NULL;
}

int main(int argc, char** argv) {
    pthread_t tid;
    // create a new child thread
    int ret;
    if ((ret = pthread_create(&tid, NULL, tfunc, NULL)) != 0) {
        // perror("pthread_create error");
        printf("pthread_create error : %d\n", ret);
        exit(1);
    }
    printf("--main : pid = %d, tid = %lu\n", getpid(), pthread_self());
    // ensure childe thread complete
    sleep(5);
    return 0;
}
```
注意，主线程最后的`sleep`是为了等待，子线程运行完毕，如果没有这句，主线程结束后，会释放掉该进程的地址空间，子线程就无法执行了。

### 练习2 - 循环创建多个子线程
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>

void* tfunc(void* arg) {
    int i = (int)arg; // 不可以 int i = *((int*)arg);
    sleep(i);
    printf("--child thread : %dth, pid = %d, tid = %lu\n", i + 1, getpid(), pthread_self());
    return NULL;
}

int main(int argc, char** argv) {
    pthread_t tid;
    int ret, i;
    for (i = 0; i < 5; ++ i) {
        if ((ret = pthread_create(&tid, NULL, tfunc, (void*)i)) != 0) { // 不可以 (void*)&i
            printf("pthread_create error : %d\n", ret);
            exit(1);
        }
    }
    sleep(5);
    printf("--main thread : pid = %d, tid = %lu\n", getpid(), pthread_self());
    return 0;
}
```
注意，代码中有两个注释，如果采用注释的方法来转递参数，会出现错误，不会按照预想的结果输出。原因是，如果将主线程中的`i`的地址传给子线程，当子线程被创建完成并能够根据地址去读取主线程里的`i`的时候，主线程的`i`已经改变了，这就导致子线程读取了错误的`i`。所以要使用值传递（如果传递的变量变，那传地址亦可）。

### pthread_exit

### pthread_join

### pthread_detach

### pthread_cancel

## 线程的属性
