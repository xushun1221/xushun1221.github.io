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

### 错误信息输出
之前，我们大多使用`perror`来输出错误信息，这是因为，这些函数会自动设置`errno`，但是`pthread_*`线程控制函数，并不会设置`errno`，而是直接返回错误号，错误信息的正确输出方式是：  
```c
int ret;
if ((ret = pthread_*(...)) != 0) {
    fprintf(stderr, "pthread_* error : %s\n", strerror(ret));
    exit(1);
}
```
后面写的一些代码可能没有使用这样的正确输出方式，请注意！

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
结束当前**线程**。

函数原型：  
```c
#include <pthread.h>

void pthread_exit(void *retval);
```
- 返回值：不返回任何东西；
- `retval`：表示线程退出状态，通常传`NULL`。

### pthread_exit, exit, return 三者的区别
先说结论：
1. `pthread_exit`：退出当前的线程；
2. `exit`：退出当前的进程；
3. `return`：返回到调用者那里去。

所以，在多线程环境中，不要使用`exit`来退出，要使用`pthread_exit`来退出当前线程，在任何线程里的`exit`都会导致整个进程退出。在其他子线程工作没有结束时，主线程不可以使用`return`或`exit`来退出。

还有，`pthread_exit` 或者 `return` 返回的指针所指向的内存单元必须是全局的或者是用 `malloc` 分配的，不能在线程函数的栈上分配，因为当其它线程得到这个返回指针时线程函数已经退出了。

#### 测试
我们使用循环创建子线程的代码，进行测试，创建五个子线程，并在子线程内打印，忽略第三个子线程的打印行为。

```c
void func() {
    pthread_exit(NULL);
}
void* tfunc(void* arg) {
    int i = (int)arg;
    sleep(i);
    if (i == 2) { // 第三个子线程
        // exit(0); // 1
        // return NULL;  // 2
        // pthread_exit(NULL); // 3
        // func(); // 3 这样也可以退出线程
    }
    printf("--child thread : %dth, pid = %d, tid = %lu\n", i + 1, getpid(), pthread_self());
    return NULL;
}
```
测试1 exit 结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ ./more_1 
--child thread : 1th, pid = 4315, tid = 140270440953600
--child thread : 2th, pid = 4315, tid = 140270432560896
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ 
```
很明显，当第二个线程调用`exit(0)`时，整个进程被关闭了。

测试2 return 结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ ./more_2
--child thread : 1th, pid = 4515, tid = 140135603021568
--child thread : 2th, pid = 4515, tid = 140135594628864
--child thread : 4th, pid = 4515, tid = 140135577843456
--child thread : 5th, pid = 4515, tid = 140135569450752
--main thread : pid = 4515, tid = 140135603025728
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ 
```
可以正确输出，只有第三个线程退出了。

测试3结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ ./more_2
--child thread : 1th, pid = 4673, tid = 139833360074496
--child thread : 2th, pid = 4673, tid = 139833351681792
--child thread : 4th, pid = 4673, tid = 139833334896384
--child thread : 5th, pid = 4673, tid = 139833326503680
--main thread : pid = 4673, tid = 139833360078656
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ 
```
可以正常输出，调用`pthread_exit`函数后退出当前线程，在该线程中调用的其他函数中执行了这个函数也可以退出该线程。

#### 主线程退出问题
回到之前的一个问题，在上面的代码里面，我们的主线程在`return 0`之前`sleep`一段时间来等待，其他子线程的打印行为，因为主线程`return 0`之后，该进程就会被销毁（main函数返回到启动例程），其他子线程无法继续执行。现在，我们可以用`pthread_exit((void*)0); // pthread_exit(NULL)`来代替`return 0`，只退出主线程而不退出整个进程。

### pthread_join
阻塞等待线程退出，获取线程退出状态。

它的作用和进程中的`waitpid wait`差不多。

函数原型：  
```c
#include <pthread.h>

int pthread_join(pthread_t thread, void **retval);
```
- 返回值：
  - 成功，返回`0`；
  - 失败，返回一个错误代码。
- `thread`：进程ID；
- `retval`：指向线程返回状态的指针。（线程返回状态是`void*`类型的，指向它的指针就是`void**`类型，进程的返回值是`int`，所以`wait waitpid`获得的返回参数是`int*`）

### 练习1 - 回收一个线程并获取退出状态
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>

typedef struct {
    char name[256];
    int age;
} people;

void* tfunc(void* arg) {
    people* p = malloc(sizeof(people));
    strcpy(p -> name, "xushun");
    p -> age = 23;
    pthread_exit((void*)p);
}

int main(int argc, char** argv) {
    pthread_t tid;
    int ret;
    if ((ret = pthread_create(&tid, NULL, tfunc, NULL)) != 0) {
        printf("pthread_create error : %d\n", ret);
        exit(1);
    }
    people* retp;
    if ((ret = pthread_join(tid, (void**)&retp)) != 0) {
        printf("pthread_join error : %d\n", ret);
        exit(1);
    }
    printf("people : name = \"%s\", age = %d\n", retp -> name, retp -> age);
    pthread_exit(NULL);
}
```

### 练习2 - 循环创建多个子线程并回收
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>

void* tfunc(void* arg) {
    int i = (int)arg;
    sleep(i);
    printf("--child thread : %dth, pid = %d, tid = %lu\n", i + 1, getpid(), pthread_self());
    pthread_exit(NULL);
}

int main(int argc, char** argv) {
    pthread_t tid[5];
    int ret;
    for (int i = 0; i < 5; ++ i) {
        if ((ret = pthread_create(tid + i, NULL, tfunc, (void*)i)) != 0) {
            printf("pthread_create error : %d\n", ret);
            exit(1);
        }
    }
    printf("--main thread : pid = %d, tid = %lu\n", getpid(), pthread_self());
    for (int i = 0; i < 5; ++ i) {
        if ((ret = pthread_join(tid[i], NULL)) != 0) {
            printf("pthread_join error : %d\n", ret);
            exit(1);
        }
        printf("--main thread : join tid = %lu\n", tid[i]);
    }
    pthread_exit(NULL);
}
```

### pthread_detach
设置线程分离。

线程分离状态，指定该状态，线程就主动与主线程断开关系。线程结束后，其状态不由其他线程进行回收，而是自己主动释放（自己清理掉PCB的残留资源，不会产生僵尸线程）。多用于网络、多线程服务器。

也可以使用`pthread_create`的参数2，来设置线程分离状态。

在一般情况下，线程终止后，其终止状态会一直保留到其他线程调用`pthread_join`来获取它的状态为止。如果设置了分离状态，线程结束后，就立即回收它占用的所有资源，而不保留终止状态。

注意，不能对一个已经处于detach状态的线程调用`pthread_join`，这样的调用将返回`EINVAL`错误。就是说，如果对一个线程调用了`pthread_detach`就不能再调用`pthread_join`。

函数原型：  
```c
#include <pthread.h>

int pthread_detach(pthread_t thread);
```
- 返回值：
  - 成功：返回`0`；
  - 失败，返回错误号。
- `thread`：待分离的线程ID。

### pthread_cancel
向线程发送取消请求。（杀死线程）

函数原型：  
```c
#include <pthread.h>

int pthread_cancel(pthread_t thread);
```
- 返回值：
  - 成功：返回`0`；
  - 失败，返回错误号。
- `thread`：要杀死的线程ID。

测试：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>

void* tfunc(void* arg) {
    while (1) {
        printf("--child thread : pid = %d, tid = %lu\n", getpid(), pthread_self());
        sleep(1);
    }
    pthread_exit(NULL);
}

int main(int argc, char** argv) {
    int ret;
    pthread_t tid;
    ret = pthread_create(&tid, NULL, tfunc, NULL);
    if (ret != 0) {
        fprintf(stderr, "pthread_create error : %s\n", strerror(ret));
        exit(1);
    }
    sleep(3);
    printf("--main thread : pid = %d, tid = %lu, cancel thread tid(%lu)\n", getpid(), pthread_self(), tid);
    ret = pthread_cancel(tid);
    if (ret != 0) {
        fprintf(stderr, "pthread_cancel error : %s\n", strerror(ret));
        exit(1);
    }
    while(1);
    return 0;
}
```

### pthread_cancel如何终止线程
#### 测试1
看一段测试代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <pthread.h>

void* tfunc_1(void* arg) {
    printf("thread 1 : returning\n");
    return (void*)1;
}

void* tfunc_2(void* arg) {
    printf("thread 2 : returning\n");
    pthread_exit((void*)2);
}

void* tfunc_3(void* arg) {
    while (1) {
        printf("thread 3 : going to die in 5 secs ...\n");
        sleep(1);
    }
    return (void*)3;
}

int main(int argc, char** argv) {
    int ret;
    pthread_t tid;
    void* tret = NULL;
    // thread 1
    pthread_create(&tid, NULL, tfunc_1, NULL);
    pthread_join(tid, &tret);
    printf("thread 1 exit code = %d\n", (int)tret);
    // thread 2
    pthread_create(&tid, NULL, tfunc_2, NULL);
    pthread_join(tid, &tret);
    printf("thread 2 exit code = %d\n", (int)tret);
    // thread 3
    pthread_create(&tid, NULL, tfunc_3, NULL);
    sleep(5);
    pthread_cancel(tid);
    pthread_join(tid, &tret);
    printf("thread 3 exit code = %d\n", (int)tret);
    return 0;
}
```

测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ ./testcancel 
thread 1 : returning
thread 1 exit code = 1
thread 2 : returning
thread 2 exit code = 2
thread 3 : going to die in 5 secs ...
thread 3 : going to die in 5 secs ...
thread 3 : going to die in 5 secs ...
thread 3 : going to die in 5 secs ...
thread 3 : going to die in 5 secs ...
thread 3 exit code = -1
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ 
```
可以看到，如果线程是异常退出的话，那么它的退出状态码为`-1`。

#### 测试2
修改一点代码，将`tfunc_3`中打印和睡眠的代码注释掉：  
```c
void* tfunc_3(void* arg) {
    while (1) {
        //printf("thread 3 : going to die in 5 secs ...\n");
        //sleep(1);
    }
    return (void*)3;
}
```

测试结果：  
```c
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ ./testcancel 
thread 1 : returning
thread 1 exit code = 1
thread 2 : returning
thread 2 exit code = 2

```
可以看到，thread 3 线程一直运行，并没有被`pthread_cancel`掉，主线程阻塞在`pthread_join`处，显然`pthread_cancel`没有能将线程杀死。

#### 测试3
再修改一些代码：  
```c
void* tfunc_3(void* arg) {
    while (1) {
        //printf("thread 3 : going to die in 5 secs ...\n");
        //sleep(1);
        pthread_testcancel();
    }
    return (void*)3;
}
```

测试结果：  
```c
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ ./testcancel 
thread 1 : returning
thread 1 exit code = 1
thread 2 : returning
thread 2 exit code = 2
thread 3 exit code = -1
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_thread$ 
```
可以看到，添加了一条`pthread_testcancel`后，子线程就可以被正常`pthread_cancel`

#### 结论
线程的取消（`pthread_cancel`）并不是实时的，而是有一定的延迟，需要等待线程到达某个取消点（检查点）才会被终止。

取消点（检查点），是线程检查是否被取消，并按请求进行动作的一个位置。这些位置，通常是，一些系统调用如`create, open, pause, close, read, write...`，执行命令`man 7 pthreads`可以查看具备这些取消点的系统调用列表，也可参阅APUE.12.7取消选项小节。

可以粗略地认为，一个系统调用（进入内核）就是一个取消点。如果线程执行中没有取消点，可以通过调用`pthread_testcancel`来自定一个取消点。

被取消的线程，退出值定义在Linux的pthread库中。常数`PTHREAD_CANCELED`的值是`-1`。可在头文件`pthread.h`中找到它的定义：`#define PTHREAD_CANCELED ((void *) -1)`。因此当我们对一个已经被取消的线程使用 `pthread_join`回收时，得到的返回值为`-1`。

### 终止线程的方式
终止某个线程而不是整个进程，有三种方法：
1. 从线程主函数`return`；（这种方法对主线程不适用，在主线程中`return`相当于`exit`）
2. 一个线程可以调用`pthread_cancel`来终止同一个进程中的另一个线程；
3. 线程可以`pthread_exit`终止自己。

### 控制原语对比
|线程控制原语|进程控制原语|
|---|---|
|pthread_create()|fork()|
|pthread_self()|getpid()|
|pthread_exit()|exit()|
|pthread_join()|wait() waitpid()|
|pthread_cancel()|kill()|
|pthread_detach()|*|
