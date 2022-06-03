---
title: 【Linux系统编程】10 - 信号
date: 2022-05-29
tags: [Linux]
categories: [Coding]
---

## 信号的概念
信号在我们的生活中随处可见，如：古代战争中摔杯为号；现代战争中的信号弹；体育比赛中使用的信号枪……  
他们都有共性：
1. 简单；
2. 不能携带大量信息；
3. 满足某个特设条件才发送。

信号能承载一些消息，它是Linux/Unix环境下，古老、经典的通信方式，现在依然是主要的通信手段。

Unix早期版本就提供了信号机制，但是并不可靠，信号可能丢失。Berkeley和AT&T都对信号模型做了更改，增加了可靠信号机制，但彼此并不兼容。POSIX.1对可靠信号例程进行了标准化。

### 信号的机制
进程在收到信号之前执行自己的代码，收到信号后，无论程序执行到什么位置，都必须立即停止执行，处理信号，处理完毕之后再继续执行。

信号可以看作是软件层面上的“中断”，所有信号的产生和处理都是由**内核**来完成的。

### 信号的相关事件和状态
#### 产生信号
1. 按键产生：如，`Ctrl+c`，`Ctrl+z`，`Ctrl+\`；
2. 系统调用产生：如，`kill`，`raise`，`abort`；
3. 软件条件产生：如，`alarm`；
4. 硬件异常产生：如，非法访问内存（段错误），除0（浮点数例外），内存对齐错误（总线错误）；
5. 命令产生：如，`kill`命令。

#### 递达
信号由内核递送并到达进程。

#### 未决
信号由内核产生后，到递达之前的状态，主要由于阻塞（屏蔽）导致该状态。

#### 信号处理
1. 执行默认动作（每个信号都有自己的默认动作）；
2. 忽略（丢弃）；
3. 捕捉（调用用户处理函数）。

#### 阻塞信号集（信号屏蔽字）
Linux系统的进程控制块PCB中，包含了和信号相关的信息，主要有阻塞信号集和未决信号集。

阻塞信号集，本质是位图，记录信号的屏蔽状态，当屏蔽某信号后，再次收到该信号时，该信号的处理将会被推后（解除屏蔽后）。在解除屏蔽前，该信号一直处于未决状态。

#### 未决信号集
未决信号集，本质是位图，用来记录信号的处理状态。信号产生时，未决信号集中描述该信号的位立即翻转，表示该信号处于未决状态，当信号被处理，对应的位又翻转回去。

信号产生后由于某些原因不能递达（主要是阻塞），这类信号的集合就是未决信号集。

### 常规信号和信号四要素
使用`kill -l`命令可以查看系统支持的所有信号。  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_sigal$ kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX	
```

不存在编号为0的信号，其中1-31号信号称为**常规信号**（普通信号，标准信号），34-64号信号称为实时信号，与底层驱动开发和硬件相关（与常规信号相比没有默认处理动作）。

#### 信号四要素
每个信号都要具备四个要素：
1. 编号；
2. 名称；
3. 事件；
4. 默认处理动作。

使用信号之前要确定信号的四要素，之后再使用。  
使用`man 7 signal`命令查看信号详细信息。

默认处理动作总结：  
- Trem：终止进程；
- Ign：忽略信号（默认动作对其忽略）；
- Core：终止进程，生成Core文件（查验进程死亡原因，用于gdb调试）
- Stop：停止（暂停）进程；
- Cont：继续运行进程。

注意，9号`SIGKILL`和19号`SIGSTOP`信号，不允许忽略和捕捉，只能执行默认动作，甚至不能设置为阻塞。

只有每个信号所对应的事件发生了，该信号才会被递送（但不一定递达），不可以乱发信号！

#### 常规信号一览
|编号|名称|事件|默认动作|
|---|---|---|---|
|1|SIGHUP| 当用户退出 shell 时，由该 shell 启动的所有进程将收到这个信号。|默认动作为终止进程|
|2|SIGINT|当用户按下了`Ctrl+C`组合键时，用户终端向正在运行中的由该终端启动的程序发出此信号。|默认动作为终止进程。|
|3|SIGQUIT|当用户按下`ctrl+\`组合键时产生该信号，用户终端向正在运行中的由该终端启动的程序发出些信号。|默认动作为终止进程。|
|4|SIGILL|CPU 检测到某进程执行了非法指令。|默认动作为终止进程并产生 core 文件|
|5|SIGTRAP|该信号由断点指令或其他 trap 指令产生。|默认动作为终止里程 并产生 core 文件。|
|6|SIGABRT|调用 abort 函数时产生该信号。|默认动作为终止进程并产生 core 文件。|
|7|SIGBUS|非法访问内存地址，包括内存对齐出错，|默认动作为终止进程并产生 core 文件。|
|8|SIGFPE|在发生致命的运算错误时发出。不仅包括浮点运算错误，还包括溢出及除数为 0 等所有的算法错误。|默认动作为终止进程并产生 core 文件。|
|9|SIGKILL|无条件终止进程。本信号不能被忽略，处理和阻塞。|默认动作为终止进程。它向系统管理员提供了可以杀死任何进程的方法。|
|10|SIGUSR1|用户定义 的信号。即程序员可以在程序中定义并使用该信号。|默认动作为终止进程。|
|11|SIGSEGV|指示进程进行了无效内存访问。|默认动作为终止进程并产生 core 文件。|
|12|SIGUSR2|另外一个用户自定义信号，程序员可以在程序中定义并使用该信号。|默认动作为终止进程。|
|13|SIGPIPE|Broken pipe 向一个没有读端的管道写数据。|默认动作为终止进程。|
|14|SIGALRM|定时器超时，超时的时间 由系统调用 alarm 设置。|默认动作为终止进程。|
|15|SIGTERM|程序结束信号，与 SIGKILL 不同的是，该信号可以被阻塞和终止。通常用来要示程序正常退出。执行 shell 命令 Kill 时，缺省产生这个信号。|默认动作为终止进程。|
|16|SIGSTKFLT|Linux 早期版本出现的信号，现仍保留向后兼容。|默认动作为终止进程。|
|17|SIGCHLD|子进程状态发生变化时，父进程会收到这个信号。|默认动作为忽略这个信号。|
|18|SIGCONT|如果进程已停止，则使其继续运行。|默认动作为继续/忽略。|
|19|SIGSTOP|停止进程的执行。信号不能被忽略，处理和阻塞。|默认动作为暂停进程。|
|20|SIGTSTP|停止终端交互进程的运行。按下`ctrl+z`组合键时发出这个信号。|默认动作为暂停进程。|
|21|SIGTTIN|后台进程读终端控制台。|默认动作为暂停进程。|
|22|SIGTTOU|该信号类似于 SIGTTIN，在后台进程要向终端输出数据时发生。|默认动作为暂停进程。|
|23|SIGURG|套接字上有紧急数据时，向当前正在运行的进程发出些信号，报告有紧急数据到达。|如网络带外数据到达，默认动作为忽略该信号。|
|24|SIGXCPU|进程执行时间超过了分配给该进程的 CPU 时间 ，系统产生该信号并发送给该进程。|默认动作为终止进程。|
|25|SIGXFSZ|超过文件的最大长度设置。|默认动作为终止进程。|
|26|SIGVTALRM|虚拟时钟超时时产生该信号。类似于 SIGALRM，但是该信号只计算该进程占用 CPU 的使用时间。|默认动作为终止进程。|
|27|SGIPROF|类似于 SIGVTALRM，它不公包括该进程占用 CPU 时间还包括执行系统调用时间。|默认动作为终止进程。|
|28|SIGWINCH|窗口变化大小时发出。|默认动作为忽略该信号。|
|29|SIGIO|此信号向进程指示发出了一个异步 IO 事件。|默认动作为忽略。|
|30|SIGPWR|关机。|默认动作为终止进程。|
|31|SIGSYS|无效的系统调用。|默认动作为终止进程并产生 core 文件。|
|34-64|SIGRTMIN - SIGRTMAX|LINUX 的实时信号，它们没有固定的含义（可以由用户自定义）。|所有的实时信号的默认动作都为终止进程。|

## 信号的产生
### 终端按键产生信号
- `ctrl + c`， 2) SIGINT（终止/中断） "INT" ----Interrupt
- `Ctrl + z`， 20) SIGTSTP（暂停/停止） "T" ----Terminal 终端。
- `Ctrl + \`， 3) SIGQUIT（退出）

### 硬件异常产生信号
- 除0操作，8) SIGFPE（浮点数例外） "F" ----Float 浮点数
- 非法访问内存，11) SIGSEGV（段错误）
- 总线错误，7) SIGBUS

### kill函数/命令产生信号
注意，kill函数或命令的作用是给某个进程发送指定信号（不一定是杀死该进程）。

#### kill命令
`kill -SIGKILL pid`，发送SIGKILL（啥信号都行）到pid号进程。

#### kill函数
给指定进程发送指定信号。（系统函数）

函数原型：  
```c
#include <sys/types.h>
#include <signal.h>

int kill(pid_t pid, int sig);
```
- 返回值：
  - 成功，返回`0`；
  - 失败，返回`-1`，并设置`errno`，原因可能是ID非法、信号非法、普通用户杀init进程等权级问题。
- `sig`：要发送的信号，应使用宏名称而非数字；
- `pid`：
  - `pid > 0`：发送信号给pid进程；
  - `pid == 0`：发送信号给与调用kill函数进程属于同一进程组的所有进程；
  - `pid < -1`：取`abs(pid)`发送给对应进程组（因为进程组长和进程组ID相同，以此区分）；
  - `pid == -1`：发送给进程有权发送的系统中的所有进程。

进程组：每个进程都属于一个进程组，进程组是一个或多个进程集合，它们互相关联，共同完成一个实体任务，每个进程组都有一个进程组组长，默认进程组ID与组长ID相同。

权限保护：root用户可以发送信号给任意用户，普通用户是不能向系统用户发送信号的。同样，普通用户也不能向其他普通用户发送信号，终止其进程。只能向自己创建的进程发送信号。普通用户的基本规则是：发送者实际或有效用户ID==接收者实际或有效用户ID。

测试，父进程杀死一个子进程：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

int main(int argc, char** argv) {
    int child_index = 0;
    pid_t pid, kpid;
    for (child_index = 0; child_index < 5; ++ child_index) {
        pid = fork();
        if (pid == 0) {
            break;
        } else if(pid > 0) {
            if (child_index == 2) { // kill 3th child
                kpid = pid;
            }
        } else {
            perror("fork error");
            exit(1);
        }
    }
    if (child_index == 5) { // parent
        sleep(3);
        if (kill(kpid, SIGKILL) == -1) {
            perror("kill error");
            exit(1);
        }
        for (int i = 0; i < 5; ++ i) {
            printf("-- parent : wait %d child\n", wait(NULL));
        }
    } else {
        int sec = 0;
        while (1) {
            if (sec == 6) {
                break;
            }
            printf("-%d- %dth child : %d\n", ++ sec, child_index + 1, getpid());
            sleep(1);
        }
    }
    return 0;
}
```

### 软件条件产生信号
#### alarm函数产生信号
设置一个定时器，指定时间后，内核会向当前进程发送14)SIGALRM信号，默认动作为终止进程。（系统函数）

函数原型：  
```c
#include <unistd.h>

unsigned int alarm(unsigned int seconds);
```
- 返回值：返回上一次设置的定时器还剩下的秒数，如果为0，则表示之前设置的定时器已经到期，或之前没设置定时器；
  - 无错误情况。
- `seconds`：设置定时器的秒数；
- 使用`alarm(0)`可以取消定时器；
- 每一个进程都有且只有唯一一个定时器。

测试：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char** argv) {
    alarm(5);
    int sec = 0;
    while (1) {
        printf("sec : %d\n", ++ sec);
        sleep(1);
        if (sec == 3) {
            alarm(10);
            sec = 10;
        }
    }
    return 0;
}
```
测试表明，如果上一个alarm还没结束就调用下一个alarm，会重置定时器。

#### setitimer函数产生信号 / getitimer
设定和获取内置定时器。（系统函数）

可以替代`alarm`，精度为微秒级us，可以实现周期定时（需要捕捉信号）。

函数原型：  
```c
#include <sys/time.h>

int getitimer(int which, struct itimerval *curr_value);
int setitimer(int which, const struct itimerval *new_value,
                struct itimerval *old_value);

struct itimerval {
    struct timeval it_interval; /* Interval for periodic timer 两次定时任务的间隔 */
    struct timeval it_value;    /* Time until next expiration 定时时长*/
};
struct timeval {
    time_t      tv_sec;         /* seconds 秒*/
    suseconds_t tv_usec;        /* microseconds 微秒*/
};
```
- 返回值：
  - 成功：返回`0`；
  - 失败：返回`-1`。
- `which`：指定定时方式，
  - `ITIMER_REAL`，自然定时，14)SIGLARM，计算自然时间；
  - `ITIMER_VIRTUAL`，虚拟空间定时（用户空间），26)SIGVTALRM，只计算进程占用CPU的时间；
  - `ITIMER_PROF`，运行时定时（用户+内核），27)SIGPROF，计算占用CPU及执行系统调用的时间。
- `new_value`：定时的参数（周期定时时长，第一次定时时长，第一次定时后，每过一个周期发送一次定时信号）（两个参数都是0，清零）；
- `old_value`：上次定时剩余的时间。

测试：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/time.h>
#include <signal.h>

void myfunc(int signo) {
    printf("alarm\n");
}

int main(int argc, char** argv) {
    struct itimerval it, oldit;

    signal(SIGALRM, myfunc);

    // 定时时长 2s0us
    it.it_value.tv_sec = 2;
    it.it_value.tv_usec = 0;
    // 定时间隔 5s0us
    it.it_interval.tv_sec = 5;
    it.it_interval.tv_usec = 0;

    if (setitimer(ITIMER_REAL, &it, &oldit) == -1) {
        perror("setitimer error");
        exit(1);
    }

    int sec = 0;
    while (1) {
        printf("-%3d-\n", ++ sec);
        sleep(1);
    }

    return 0;
}
```
测试结果：  
```console
xushun@xushun-virtual-machine:~/LinuxSysPrograming/test_sigal$ ./setitimer 
-  1-
-  2-
alarm
-  3-
-  4-
-  5-
-  6-
-  7-
alarm
-  8-
-  9-
- 10-
- 11-
- 12-
alarm
```

## 信号集操作函数
内核通过读取未决信号集爱判断信号是否应该被处理，但如果该信号未决且在屏蔽信号集里面就会被延后处理，信号屏蔽字mask可以影响未决信号集，我们可以在应用程序中自定义一个`set`和mask进行位运算来改变它，以达到屏蔽指定信号的目的。

阻塞信号集（信号屏蔽字mask）和未决信号集（pending）示意图如下：  
![](/post_images/posts/Coding/【Linux系统编程】10/信号集示意图.jpg "信号集示意图")

### 信号集设定函数
（自定义的）信号集设定的函数有以下几个（库函数）：  
```c
#include <signal.h>

// 清空集合
int sigemptyset(sigset_t *set);
// 置1
int sigfillset(sigset_t *set);
// 加一个信号
int sigaddset(sigset_t *set, int signum);
// 删一个信号
int sigdelset(sigset_t *set, int signum);
// 查看某个信号是否在集合中
int sigismember(const sigset_t *set, int signum);
```

### sigprocmask
用来屏蔽信号和解除信号屏蔽。（系统函数）

设定了自定义的信号集set后，需要使用`sigprocmask`来修改进程（PCB）中的信号屏蔽字。

注意：屏蔽信号只是将信号延后处理（解除屏蔽后处理），而忽略表示直接将信号丢弃处理。

函数原型：  
```c
#include <signal.h>

/* Prototype for the glibc wrapper function */
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);

/* Prototype for the underlying system call */
int rt_sigprocmask(int how, const kernel_sigset_t *set,
                    kernel_sigset_t *oldset, size_t sigsetsize);

/* Prototype for the legacy system call (deprecated) */
int sigprocmask(int how, const old_kernel_sigset_t *set,
                old_kernel_sigset_t *oldset);
```
- 返回值：
  - 成功，返回`0`；
  - 失败，返回`-1`并设置`errno`。
- `set`：自定义的信号集，`set`中为`1`的位置就代表要屏蔽的信号；
- `oldset`：保存旧的信号屏蔽字；
- `how`：执行的操作，
  - `SIG_BLOCK`：`set`表示需要屏蔽的信号，相当于`mask |= set`；
  - `SIG_UNBLOCK`：`set`表示需要解除屏蔽的信号，相当于`mask &= ~set`；
  - `SIG_SETMASK`：`set`用于替换原始的信号屏蔽字，相当于`mask = set`。
- 如果调用`sigprocmask`解除了对当前若干信号的阻塞，则在`sigprocmask`返回之前，至少将会一个信号递达。

### sigpending
读取当前的未决信号集。（系统函数）

虽然我们不能直接操作未决信号集，但是内核也提供了函数让我们查看当前未处理的信号。

函数原型：  
```c
#include <signal.h>

int sigpending(sigset_t *set);
```
- 返回值：
  - 成功，返回`0`；
  - 失败，返回`-1`设置`errno`。
- `set`：返回的未决信号集。

### 练习 - 屏蔽SIGINT信号
代码：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

void print_set(sigset_t set) {
    for (int i = 1; i <= 32; ++ i) {
        if (sigismember(&set, i)) {
            putchar('1');
        } else {
            putchar('0');
        }
    }
    printf("\n");
}

int main(int argc, char** argv) {
    sigset_t set, oldset, pedset;
    sigemptyset(&set); // 创建空的信号集
    sigaddset(&set, SIGINT); // 添加SIGINT信号 (Ctrl+c)
    if (sigprocmask(SIG_BLOCK, &set, &oldset) == -1) { // 将SIGINT信号阻塞
        perror("sigprocmask error");
        exit(1);
    }
    while (1) {
        if (sigpending(&pedset) == -1) { // 读取pending未决信号集
            perror("sigpending error");
            exit(1);
        }
        print_set(pedset); // 打印pending
        sleep(1);
    }
    if (sigprocmask(SIG_SETMASK, &oldset, &set)) { // 恢复原来的mask
        perror("sigprocmask error");
        exit(1);
    }
    return 0;
}
```

## 信号捕捉
### signal
**注册**一个信号捕捉函数（系统函数）。

`signal`函数是由ANSI定义的，由于历史原因，在不同版本的Linux和Unix中可能由不同的行为，因此应该尽量避免使用它，而应该使用`sigaction`函数替代之。

函数原型：  
```c
#include <signal.h>

typedef void (*sighandler_t)(int); // 定义sighandler 一个返回值为void 参数为int 的 函数指针

sighandler_t signal(int signum, sighandler_t handler);
```
- 返回值：
  - 成功，返回函数指针；
  - 失败，返回`SIG_ERR`，设置`errno`。
- `signum`：要捕捉的信号编号；
- `handler`：处理函数。

测试：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

void catch(int signo) {
    printf("catch sig : %d\n", signo);
    return;
}

int main(int argc, char** argv) {
    if (signal(SIGTSTP, catch) == SIG_ERR) { // 捕捉20 SIGTSTP Ctrl+z
        perror("signal error");
        exit(1);
    }
    while (1) { }
    return 0;
}
```

### sigaction
修改信号处理动作（Linux中**注册**一个信号的捕捉函数）（系统函数）。

函数原型：  
```c
#include <signal.h>

int sigaction(int signum, const struct sigaction *act,
                struct sigaction *oldact);

struct sigaction {
    void     (*sa_handler)(int); // 处理动作的函数指针
    void     (*sa_sigaction)(int, siginfo_t *, void *); // 一般不用 可以用来传递数据
    sigset_t   sa_mask; // 屏蔽字 只工作于信号捕捉函数运行期间
    int        sa_flags; // 一些参数 一般情况为零使用默认属性（屏蔽本信号）
    void     (*sa_restorer)(void); // 废弃
};
```
- 返回值：
  - 成功，返回`0`；
  - 失败，`-1`，设置`errno`。
- `act`：要设置的动作；
- `oldact`：保存原来的动作；
- `sa_handler`：信号捕捉后的处理函数名，可赋值为`SIG_IGN`表示忽略或者`SIG_DFL`表示执行默认动作；

测试：  
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

void catch(int signo) {
    printf("\ncatch sig : %d\n", signo);
    sleep(3);
    return;
}

int main(int argc, char** argv) {

    struct sigaction act, oldact;
    act.sa_handler = catch;        // 设置回调函数
    sigemptyset(&act.sa_mask);      // 设置清空屏蔽字
    act.sa_flags = 0;               // 设置参数 0 默认屏蔽该函数捕捉的信号
    if (sigaction(SIGTSTP, &act, &oldact) == -1) { // 注册 20 Ctrl+z
        perror("sigaction error");
        exit(1);
    }
    if (sigaction(SIGQUIT, &act, &oldact) == -1) { // 注册 3 Ctrl+\
        perror("sigaction error");
        exit(1);
    }
    while (1) { }
    return 0;
}
```
注意，在捕捉函数处理过程中如果和捕捉函数当前捕捉的信号**相同**的信号到达，不会立即进行处理（临时屏蔽），在捕捉函数结束后处理。和当前捕捉信号不同的信号会被及时处理。

假如我们在Ctrl+z信号捕捉处理期间，快速按多次Ctrl+z产生多个SIGTSTP信号，也只会在当前处理函数结束后，再执行一次处理函数。（不支持排队）

### 信号捕捉的特性
1. 进程正常运行时，默认PCB中有一个信号屏蔽字，mask，它决定了进程自动屏蔽哪些信号。当注册了某个信号捕捉函数，捕捉到该信号以后，要调用该函数。而该函数有可能执行很长时间，在这期间所屏蔽的信号不由mask来指定。而是用 sa_mask 来指定。调用完信号处理函数，再恢复为mask。
2. XXX 信号捕捉函数执行期间，XXX 信号自动被屏蔽。（`sa_flag = 0`）
3. 阻塞的常规信号不支持排队，产生多次只记录一次。（后 32 个实时信号支持排队）

### 内核信号捕捉过程
![](/post_images/posts/Coding/【Linux系统编程】10/内核信号捕捉过程.jpg "内核信号捕捉过程")

## 借助信号捕捉回收子进程
### SIGCHLD
SIGCHLD产生的条件：
1. 子进程终止；
2. 子进程收到SIGSTOP信号停止；
3. 子进程停止，收到SIGCONT信号后唤醒。

### 借助SIGCHLD回收子进程
之前介绍`execlp`的时候提到过使用信号来回收子进程。

方法是，在子进程结束后，父进程会收到`SIGCHLD`信号，该信号的默认处理动作是忽略，我们可以捕捉该信号，在捕捉函数中来完成子进程的回收。

#### 实现方法 - 基础
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <sys/wait.h>

void catch_child(int signo) {
    pid_t wpid = wait(NULL);
    if (wpid == -1) {
        perror("wait error");
        exit(1);
    }
    printf("--catch child : %d\n", wpid);
    return;
}

int main(int argc, char** argv) {
    int i;
    const int n = 20;
    for (i = 0; i < n; ++ i) { // fork 20 child
        if (fork() == 0) { // parent
            break;
        }
    }
    if (i == n) { // parent
        printf("--parent process : %d\n", getpid());
        struct sigaction act, oldact;
        act.sa_handler = catch_child;
        sigemptyset(&act.sa_mask);
        act.sa_flags = 0;
        if (sigaction(SIGCHLD, &act, &oldact) == -1) {
            perror("sigaction error");
            exit(1);
        }
        while (1) { }
    } else { // child
        printf("--child process : %d\n", getpid());
    }
    return 0;
}
```
通过测试，我们发现，捕捉SIGCHLD信号可以用来回收子进程。但是，我们创建了20个子进程，确只回收了6个（测试截图就不放了），剩下的都变成僵尸进程了。这是为什么呢？

原因是，信号是不排队的，在很短的时间内，如果有多个子进程终止并发出了SIGCHLD信号，此时捕捉函数只能处理一个。解决方法：使用循环来回收子进程。  
```c
void catch_child(int signo) {
    int wpid;
    while ((wpid = wait(NULL)) != -1) {
        printf("--catch child : %d\n", wpid);
    }
    return;
}
```

还有一些问题：  
1. 如果父进程注册捕捉函数之前，子进程就结束了怎么办？简单的方法是在子进程中`sleep(1)`等待，更好的方法是信号屏蔽；
2. 可以结合`wstatus`和`WIFEXITED`判断SITCHLD的原因。
这些问题下面解决。

#### 实现方法 - 完整
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <sys/wait.h>

void catch_child(int signo) {
    int wpid, wstatus;
    while ((wpid = waitpid(-1, &wstatus, WNOHANG)) > 0) {
        printf("--catch child : %d : ", wpid);
        if (WIFEXITED(wstatus)) { // 正常退出 退出代码
            printf("exit %d\n", WEXITSTATUS(wstatus));
        } else if (WIFSIGNALED(wstatus)) { // 异常终止 终止信号
            printf("cancel signal %d\n", WTERMSIG(wstatus));
        }
    }
    return;
}

int main(int argc, char** argv) {
    // 屏蔽SIGCHLD 防止未注册捕捉函数时 子进程结束
    sigset_t set, oldset, pedset;
    sigemptyset(&set);
    sigaddset(&set, SIGCHLD);
    if (sigprocmask(SIG_BLOCK, &set, &oldset) == -1) {
        perror("sigprocmask error");
        exit(1);
    }
    //
    const int n = 20;
    int i;
    pid_t pid;
    for (i = 0; i < n; ++ i) { // fork 20 child
        if ((pid = fork()) == 0) { // parent
            break;
        } else if (pid < 0) {
            perror("fork error");
            exit(1);
        }
    }
    if (i == n) { // parent
        printf("--parent process : %d\n", getpid());
        struct sigaction act, oldact;
        act.sa_handler = catch_child;
        sigemptyset(&act.sa_mask);
        act.sa_flags = 0;
        if (sigaction(SIGCHLD, &act, &oldact) == -1) {
            perror("sigaction error");
            exit(1);
        }
        // 捕捉函数注册完成 解除对SIGCHLD的阻塞
        if (sigprocmask(SIG_SETMASK, &oldset, &set)) { // 恢复原来的mask
            perror("sigprocmask error");
            exit(1);
        }
        //
        while (1) { }
    } else { // child
        printf("--child process : %d\n", getpid());
        return i + 1; // 退出代码为child创建顺序
    }
    return 0;
}
```

## 中断系统调用
系统调用可分为两类：慢速系统调用和其他系统调用。

1. 慢速系统调用：可能会使进程永远阻塞的一类。如果在阻塞期间收到一个信号，该系统调用就被中断,不再继续执行(早期)；也可以设定系统调用是否重启。如，read、write、pause、wait...
2. 其他系统调用：getpid、getppid、fork...

可修改 sa_flags 参数来设置被信号中断后系统调用是否重启。SA_INTERRURT 不重启。 SA_RESTART 重启。

sa_flags 还有很多可选参数，适用于不同情况。如：捕捉到信号后，在执行捕捉函数期间，不希望自动阻塞该信号，可将 sa_flags 设置为 SA_NODEFER，除非 sa_mask 中包含该信号。