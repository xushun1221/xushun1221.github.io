---
title: 【重写muduo】13 - EventLoopThread
date: 2022-09-21
tags: [network, C++, muduo]
categories: [Networking]
---

上一篇我们封装了`Thread`类，`EventLoopThread`类将一个`EventLoop`和一个`Thread`封装在一起，实现一个线程运行一个事件循环的功能。

one loop per thread!


## 源码

`EventLoopThread.hh`
```cpp
#ifndef   __EVENTLOOPTHREAD_HH_
#define   __EVENTLOOPTHREAD_HH_

#include "noncopyable.hh"
#include "Thread.hh"

#include <functional>
#include <mutex>
#include <condition_variable>

class EventLoop;

/* one loop per thread! */
class EventLoopThread : noncopyable {
public:
    using ThreadInitCallback = std::function<void(EventLoop*)>;

    EventLoopThread(const ThreadInitCallback& cb = ThreadInitCallback()
                    , const std::string& name = std::string());
    ~EventLoopThread();

    EventLoop* startLoop();
private:
    void threadFunc();

    EventLoop* loop_;
    bool exiting_;
    Thread thread_;
    std::mutex mutex_;
    std::condition_variable cond_;
    ThreadInitCallback callback_; /* 在线程中创建新EventLoop后调用该函数 */
};


#endif // __EVENTLOOPTHREAD_HH_
```


`EventLoopThread.cc`
```cpp
#include "EventLoopThread.hh"
#include "EventLoop.hh"


EventLoopThread::EventLoopThread(const ThreadInitCallback& cb, const std::string& name) 
    : loop_(nullptr)
    , exiting_(false)
    , thread_(std::bind(&EventLoopThread::threadFunc, this), name) /* 注意这里线程还没创建 线程调用start()后才创建 */
    , mutex_()
    , cond_()
    , callback_(cb)
    {   
}

EventLoopThread::~EventLoopThread() {
    exiting_ = true;
    if (loop_ != nullptr) {
        loop_->quit();
        thread_.join();
    }
}

EventLoop* EventLoopThread::startLoop() {
    /* 启动新线程 */
    thread_.start(); /* 线程执行的函数就是下面的EventLoopThread::threadFunc() */

    /* 注意 这里是多线程环境了 */

    EventLoop* loop = nullptr;
    {
        std::unique_lock<std::mutex> lock(mutex_);
        while (loop_ == nullptr) {
            cond_.wait(lock); /* 等待新线程初始化loop_ */
        }
        /* 初始化loop_完成 */
        loop = loop_;
    }
    return loop; /* 返回新线程中的loop对象 */
}

/* 该方法是在新线程中运行的 */
void EventLoopThread::threadFunc() {
    /* 创建一个独立的EventLoop 和新线程是一一对应的 */
    EventLoop loop; /* one loop per thread */

    if (callback_) {
        callback_(&loop);
    }

    {
        std::unique_lock<std::mutex> lock(mutex_);
        loop_ = &loop;
        cond_.notify_one();
    }

    loop.loop(); /* 开启poller.poll */

    /* 事件循环结束 */
    std::unique_lock<std::mutex> lock(mutex_);
    loop_ = nullptr;
}
```