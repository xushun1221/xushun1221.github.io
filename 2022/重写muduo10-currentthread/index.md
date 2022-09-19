# 【重写muduo】10 - CurrentThread


## 获得线程的唯一标识

muduo库使用的模型是：**one loop per thread**，每个EventLoop都是在一个单独的线程中运行的，需要控制一些和线程相关的逻辑，所以这里会涉及到一些**获取当前线程ID**的操作。

在Linux中，每个进程都有一个进程id，可以由`getpid()`获得。同样，POSIX线程也有线程id，类型为`pthread_t`，可以由`pthread_self()`获得，但是这个线程id是由pthread线程库维护的，各个进程之间是独立的，用于进程内部区分不同的线程，它只能保证同一个进程中的线程的id不重复。

如果我们需要获得系统中的线程的唯一标识，也就是**LWP(Light Weight Process)**，需要进行系统调用`syscall(SYS_gettid)`。

## __thread 关键字

在多线程环境下，一般的全局共享变量需要进行保护，才能在各个线程间互不干扰。如果我们使用`__thread`关键字来修饰全局共享变量，那么该变量会在每一个线程中存储一份独立的实体，各个线程的该变量互不干扰。

`__thread`是GNUC标准gcc编译器提供的，在其他环境中也可以使用c++11标准的`thread_local`关键字。

注意`__thread`只能修饰编译器内置变量类型，对于类类型而言需要通过指针使用。




## 源码


`CurrentThread.hh`  
```cpp
#ifndef   __CURRENTTHREAD_HH_
#define   __CURRENTTHREAD_HH_

#include <unistd.h>

namespace CurrentThread {
    
    /* 该线程的LWP的缓存 */
    extern __thread int t_cachedTid;

    /* 将该线程的LWP缓存到线程中 */
    void cacheTid();

    /* 获得当前线程的LWP */
    inline int tid() {
        /* 
            __builtin_expect是gcc提供的分支预测优化的宏
            __builtin_expect(exp, c)表示期望exp的值为c
            编译过程中编译器会将可能性更大的代码紧跟前面的代码
            减少指令跳转 优化程序性能

            这里表示t_cachedTid == 0(还没有缓存LWP)的概率很小
            已经缓存的概率很大
            如果没缓存过 调用cacheTid缓存该线程的LWP
        */
        if (__builtin_expect(t_cachedTid == 0, 0)) {
            cacheTid();
        }
        return t_cachedTid;
    }

}


#endif // __CURRENTTHREAD_HH_
```


`CurrentThread.cc`  
```cpp
#include "CurrentThread.hh"

#include <sys/syscall.h>

namespace CurrentThread {

    /* 在.cc文件中定义全局变量 __thread修饰表示每个线程中使用独立实体 */

    /* 线程唯一标识 LWP */
    __thread int t_cachedTid = 0;

    /* 将该线程的LWP缓存到线程中 */
    void cacheTid() {
        if (t_cachedTid == 0) {
            t_cachedTid = static_cast<pid_t>(::syscall(SYS_gettid));
        }
    }

}
```
