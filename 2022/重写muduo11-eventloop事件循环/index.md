# 【重写muduo】11 - EventLoop事件循环


`EventLoop`事件循环类，主要是封装`Poller`、回调函数执行逻辑、mainloop和subloop之间的通信方法。


## mainloop 与 subloop 通信方式

muduo库使用**one loop per thread**模型。

- 如果我们的服务器只使用一个线程(mainloop)，那么它不仅需要接收新用户的连接(IO线程)，还要处理已连接用户的读写事件；
- 如果服务器使用了多个线程，那么mainloop负责接收新用户的连接，其他的subloop负责处理已连接用户的读写事件。

通常的做法是，使用一个消息队列，mainloop作为生产者，subloop作为消费者，进行已连接用户读写事件的处理。但是muduo不是这样做的。muduo中，mainloop和subloop直接进行通信。

### wakeupChannel_

`EventLoop`中有一个`wakeupChannel_`成员，每当一个`EventLoop`对象实例化时，都会通过`eventfd()`系统调用获得一个用于main-sub loop之间通信的fd，并将对应的`EPOLLIN`事件注册到Poller监听树上。

`wakeupChannel_`对应的回调，不做任何事情，它的功能仅仅是：在mainloop向该`wakeupFd_`中写入数据时，该loop能立即从`poll`函数的阻塞状态中苏醒。`EventLoop`的共有方法`wakeup`提供了唤醒的功能接口。


### pendingFunctors_

`EventLoop`中的`pendingFunctors_`成员数组，用于保存mainloop注册到该subloop上的回调函数。

当mainloop需要subloop做事情时，就会通过`EventLoop`的共有接口`queueInLoop`将回调函数注册到`pendingFunctors_`中，并视情况唤醒subloop。

subloop进行的每轮`poll`，都会调用`doPendingFunctors`，执行所有mainloop注册到`pendingFunctors_`中的回调。


这就是muduo中，mainloop和subloop之间进行通信的方式。

还有其他的一些问题，比如mainloop如何选择subloop等等问题，在后面的其他组件的实现中继续分析。


## 源码

`EventLoop.hh`  
```cpp
#ifndef   __EVENTLOOP_HH_
#define   __EVENTLOOP_HH_

#include "noncopyable.hh"
#include "Timestamp.hh"
#include "CurrentThread.hh"

#include <functional>
#include <vector>
#include <atomic>
#include <memory>
#include <mutex>

class Channel;
class Poller;

/* EventLoop事件循环类 主要包含Channel和Poller(epoll)两大模块 */
class EventLoop : noncopyable {
public:
    /* 回调的类型 */
    using Functor = std::function<void()>;

    EventLoop();
    ~EventLoop();

    /* 开启事件循环 */
    void loop();
    /* 退出事件循环 */
    void quit();

    Timestamp pollReturnTime() const { return pollReturnTime_; }

    /* 在当前loop中执行cb */
    void runInLoop(Functor cb);
    /* 把cb放入队列中 唤醒loop所在线程执行cb */
    void queueInLoop(Functor cb);

    /* mainloop唤醒subloop 唤醒loop所在线程 */
    void wakeup();

    /* 更新Channel状态 调用Poller的方法*/
    void updateChannel(Channel* channel);
    /* 删除Channel 调用Poller的方法 */
    void removeChannel(Channel* channel);
    /* 判断Channel是否存在 调用Poller的方法 */
    bool hasChannel(Channel* channel);

    /* EventLoop是否在当前线程 */
    bool isInLoopThread() const { return threadId_ == CurrentThread::tid(); }

private:

    /* wakeup相关 */
    void handleRead();
    /* 执行回调 */
    void doPendingFunctors();


    using ChannelList = std::vector<Channel*>;

    std::atomic_bool looping_; /* 标识是否正在looping */
    std::atomic_bool quit_;    /* 控制是否跳出loop */

    const pid_t threadId_; /* 当前loop所在线程的id(LWP) */  

    Timestamp pollReturnTime_; /* Poller返回发生事件的Channel的时间戳 */
    std::unique_ptr<Poller> poller_; /* EventLoop管理的Poller */

    /* 重要!!!!!!!!! */
    int wakeupFd_; 
    std::unique_ptr<Channel> wakeupChannel_; /* 用于封装wakeupFd_ */
        /* 
            当mainloop获取一个新用户的Channel时
            通过轮询算法选择一个subloop
            通过该成员唤醒subloop处理channel
            使用eventfd()系统调用函数来实现该功能
        */
    /* !!!!!!!!!!!!! */

    ChannelList activeChannels_; /* 发生事件的Channel列表 */

    std::atomic_bool callingPendingFunctors_; /* 标识当前loop是否有需要执行的回调 */
    std::vector<Functor> pendingFunctors_; /* 存放loop需要执行的所有的回调操作 */
    std::mutex mutex_; /* 用来保护pendingFunctors_ */
};


#endif // __EVENTLOOP_HH_
```



`EventLoop.cc`  
```cpp
#include "EventLoop.hh"
#include "Logger.hh"
#include "Poller.hh"
#include "Channel.hh"

#include <sys/eventfd.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <memory>

/* 防止一个线程创建多个EventLoop 一个线程中只能有一个EventLoop */
__thread EventLoop* t_loopInThisThread = nullptr;

/* 定义默认的Poller的IO复用接口的超时事件 */
const int kPollTimeMs = 10000;

/* 创建wakeupfd 用来唤醒subReactor处理新来的channel */
int createEventfd() {
    int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (evtfd < 0) {
        /* 出错 无法创建evtfd 这是严重错误 */
        LOG_FATAL("eventfd error:%d\n", errno);
    }
    return evtfd;
}

/* 构造 */
EventLoop::EventLoop() 
    : looping_(false)
    , quit_(false)
    , threadId_(CurrentThread::tid())
    , poller_(Poller::newDefaultPoller(this))
    , wakeupFd_(createEventfd())
    , wakeupChannel_(new Channel(this, wakeupFd_))
    , callingPendingFunctors_(false)
    {
        LOG_DEBUG("EventLoop created %p in thread %d \n", this, threadId_);
        if (t_loopInThisThread != nullptr) {
            /* 如果该线程中已经存在了EventLoop无法运行 属于严重错误 */
            LOG_FATAL("another EventLoop %p exists in this thread %d \n", t_loopInThisThread, threadId_);
        } else {
            /* 该线程第一次创建一个EventLoop对象 */
            t_loopInThisThread = this;
        }
        /* 设置wakeupfd的事件类型以及发生事件后的回调 */
        wakeupChannel_->setReadCallback(std::bind(&EventLoop::handleRead, this));
        /* 每个EventLoop都会监听wakeupChannel的EPOLLIN事件 */
        wakeupChannel_->enableReading();
}

/* 析构 */
EventLoop::~EventLoop() {
    wakeupChannel_->disableAll();
    wakeupChannel_->remove();
    ::close(wakeupFd_);
    t_loopInThisThread = nullptr;
}

/* 接收wakeupFd_收到的唤醒消息 */
void EventLoop::handleRead() {
    uint64_t one = 1;
    ssize_t n = read(wakeupFd_, &one, sizeof(one));
    if (n != sizeof(one)) {
        LOG_ERROR("EventLoop::handleRead() reads %zu bytes instead of 8 \n", n);
    }
}


/* 开启事件循环 */
void EventLoop::loop() {
    looping_ = true;
    quit_ = false;

    LOG_INFO("EventLoop %p start looping \n", this);

    while (quit_ == false) {
        activeChannels_.clear();
        /* 获得发生事件的Channel */
        pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
        /* 遍历所有发生事件的Channel 执行对应的回调 */
        for (Channel* channel : activeChannels_) {
            channel->handleEvent(pollReturnTime_);
        }
        /* 执行当前EventLoop事件循环需要处理的回调操作 */
        doPendingFunctors();
        /*
            IO线程 mainloop 主要做accept的工作 fd->Channel => subloop
            1. 如果我们的服务器只使用一个线程 就是mainloop
                那么它不仅要接收新用户的连接 还要处理已连接用户的读写事件
            2. 如果使用了多个线程
                mainloop获得了新用户的Channel后 就会唤醒某一个subloop
                mainloop事先注册一个回调cb 需要subloop来执行
                mainloop wakeup一个subloop时 通过wakeupfd唤醒subloop
                subloop会从poller_->poll的阻塞状态醒来
                然后就会通过doPendingFunctors来执行mainloop之前注册的回调cb
        */
    }
    LOG_INFO("EventLoop %p stop looping \n", this);
    looping_ = false;
}

/* 退出事件循环 */
void EventLoop::quit() {
    quit_ = true;
    /*
        1. 在自己的线程中调用quit()时
            说明没有阻塞在poller_->poll处
        2. 如果在其他线程中调用了该EventLoop的quit()
            需要把该loop唤醒
            再次进入循环时就会发现quit_ == true
    */
    if (isInLoopThread() == false) {
        wakeup();
    }
}


/* 在当前loop中执行cb */
void EventLoop::runInLoop(Functor cb) {
    if (isInLoopThread() == true) {
        cb();
    } else {
        /* 在非loop线程中执行cb 就需要唤醒loop所在线程执行cb */
        queueInLoop(cb);
    }

}

/* 把cb放入队列中 唤醒loop所在线程执行cb */
void EventLoop::queueInLoop(Functor cb) {
    /* cb放入pendingFunctors */
    {
        std::unique_lock<std::mutex> lock(mutex_);
        pendingFunctors_.emplace_back(cb); /* push_back也可 */
    }
    /* 唤醒相应的需要执行cb的loop的线程 */
    if (isInLoopThread() == false || callingPendingFunctors_ == true) { 
        /* 
            callingPendingFunctors_ == true 表示上一轮的doPendingFunctors正在执行
            执行完进入下一轮循环时 仍然有可能再次阻塞
            所以需要进行唤醒操作
        */
        wakeup(); /* 唤醒该loop所在线程 */
    }
}



/* mainloop唤醒该subloop 唤醒该loop所在线程 */
void EventLoop::wakeup() {
    /* 向该EventLoop的wakeupfd_写一个数据即可唤醒 */
    uint64_t one = 1;
    ssize_t n = write(wakeupFd_, &one, sizeof(one));
    if (n != sizeof(one)) {
        /* 无法唤醒该EventLoop */
        LOG_ERROR("EventLoop::wakeup() writes %zu bytes instead of 8 \n", n);
    }
}

/* 更新Channel状态 调用Poller的方法*/
void EventLoop::updateChannel(Channel* channel) {
    poller_->updateChannel(channel);
}

/* 删除Channel 调用Poller的方法 */
void EventLoop::removeChannel(Channel* channel) {
    poller_->removeChannel(channel);
}

/* 判断Channel是否存在 调用Poller的方法 */
bool EventLoop::hasChannel(Channel* channel) {
    return poller_->hasChannel(channel);
}

/* 执行回调 */
void EventLoop::doPendingFunctors() {
    /* 执行mainloop注册到该loop中的回调操作 */

    /*
        注意 这里我们并不能直接对pendingFunctors_进行加锁并遍历执行
        而是需要创建一个局部vector将pendingFunctors_置换出来 再遍历执行
        这是因为
            doPendingFunctors()函数执行可能需要较长时间
            在执行过程中 如果mainloop又注册了新的回调 就会阻塞在互斥锁上
            运行效率很低
        这样做 即使该loop正在执行需要做的回调
        也不妨碍mainloop继续向该loop注册新的回调
    */
    std::vector<Functor> functors;
    callingPendingFunctors_ = true;
    {
        std::unique_lock<std::mutex> lock(mutex_);
        functors.swap(pendingFunctors_);
    }
    for (const Functor& functor : functors) {
        functor();
    }
    callingPendingFunctors_ = false;
}
```
