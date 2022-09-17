# 【重写muduo】07 - Channel通道


muduo库里面，`TcpServer`的事件循环`EventLoop`中，主要包含两大模块，一个是`Channel`列表，另一个是`Poller`。

`Channel`，通道类，实际上包含了，sockfd，及其感兴趣的事件（如`EPOLLIN EPOLLOUT`等），以及poller返回的就绪的具体事件。


源码中有两处由`EventLoop`调用函数的语句没写，其他都完成了。

值得一提的是，`Channel`类中使用了一个`weak_ptr<void> tie_`弱智能指针成员变量来观察对象是否被析构。在调用事件回调时会尝试提升该弱智能指针，如果提升失败则表明该对象被析构，不会继续调用。

当然，我们现在还没有使用到`tie_`和`tie()`函数，后面用到时再详细分析。


## 源码

`Channel.hh`  
```cpp
#ifndef   __CHANNEL_HH_
#define   __CHANNEL_HH_

#include "noncopyable.hh"
#include "Timestamp.hh"

#include <functional>
#include <memory>

/* 前置声明类类型 而不包含头文件 在.cc文件中包含头文件 可以暴漏更少信息 */
class EventLoop;

/* Channel通道类 封装sockfd及其感兴趣的EPOLL事件以及poller返回的具体事件 */
class Channel : noncopyable {
public:
    using EventCallback = std::function<void()>;
    using ReadEventCallback = std::function<void(Timestamp)>;

    /* 构造和析构 使用fd和EventLoop指针构造 */
    explicit Channel(EventLoop* loop, int fd);  /* EventLoop不用包含头文件是因为只需要指针而非具体信息 指针大小都一样 */
    ~Channel();
    
    /* 防止当Channel被Poller手动remove Channel还在执行回调 */
    void tie(const std::shared_ptr<void>& obj);
    
    /* 处理事件 调用相应的回调 */
    void handleEvent(Timestamp receiveTime);    /* Timestamp不用前置类型声明而是包含头文件 因为这里需要知道它的具体大小信息 */
    /* 设置回调函数对象 */
    void setReadCallback(ReadEventCallback cb) { readCallback_ = std::move(cb); } /* cb是局部对象 出作用域就析构 所以使用move右值赋值 转移资源所属权 */
    void setWriteCallback(EventCallback cb) { writeCallback_ = std::move(cb); }
    void setCloseCallback(EventCallback cb) { closeCallback_ = std::move(cb); }
    void setErrorCallback(EventCallback cb) { errorCallback_ = std::move(cb); }

    /* 返回fd */
    int fd() const { return fd_; }
    /* 返回fd感兴趣的事件 */
    int events() const { return events_; }
    /* 设置poller返回的发生的事件 提供给Poller使用 */
    void set_revents(int revt) { revents_ = revt; }
    
    /* 更新事件 */
    void enableReading() { events_ |= kReadEvent; update(); } /* update epoll_ctl */
    void disableReading() { events_ &= ~kReadEvent; update(); }
    void enableWriting() { events_ |= kWriteEvent; update(); }
    void disableWriting() { events_ &= ~kWriteEvent; update(); }
    void disableAll() { events_ = kNoneEvent; update();}

    /* 当前的事件状态 */
    bool isReading () const { return events_ & kReadEvent; }
    bool isWriting () const { return events_ & kWriteEvent; }
    bool isNoneEvent() const { return events_ == kNoneEvent; }

    /* 索引号 */
    int index() { return index_; }
    void set_index(int idx) { index_ = idx; }
    /* 该Channel所属的EventLoop  one loop per thread */
    EventLoop* ownerLoop() { return loop_; }

    /* 在Channel所属的EventLoop中 删除当前的Channel */
    void remove();

private:
    /* 更新在poller上注册的事件 */
    void update();

    /* 受保护的handleEvent  由handleEvent调用 */
    void handleEventWithGuard(Timestamp receiveTime);

private:
    /* 事件标识 */
    static const int kNoneEvent;  /* 没有事件 */
    static const int kReadEvent;  /* 读事件 */
    static const int kWriteEvent; /* 写事件 */

    EventLoop* loop_;   /* 事件循环 */
    const int fd_;      /* fd poller监听的对象 */
    int events_;        /* 注册fd感兴趣的事件 */
    int revents_;       /* poller返回的具体发生的事件 */
    int index_; /* 索引号 */

    std::weak_ptr<void> tie_;
    bool tied_;

    /* 
        不同事件的回调 
        Channel通道能够获知fd发生的具体的事件revents 所以它负责调用具体事件的回调
    */
    ReadEventCallback readCallback_;
    EventCallback writeCallback_;
    EventCallback closeCallback_;
    EventCallback errorCallback_;
};


#endif // __CHANNEL_HH_
```


`Channel.cc`  
```cpp
#include "Channel.hh"
#include "EventLoop.hh"
#include "Logger.hh"

#include <sys/epoll.h>

const int Channel::kNoneEvent = 0;                   /* 没有事件 */
const int Channel::kReadEvent = EPOLLIN | EPOLLPRI;  /* 读事件 */
const int Channel::kWriteEvent = EPOLLOUT;           /* 写事件 */

/* 构造和析构 */
Channel::Channel(EventLoop* loop, int fd) 
    : loop_(loop), fd_(fd), events_(0), revents_(0), index_(-1), tied_(false) {}
Channel::~Channel() {

}

/* 防止当Channel被Poller手动remove Channel还在执行回调 */
void Channel::tie(const std::shared_ptr<void>& obj) {
    tie_ = obj; /* 弱智能指针绑定强智能指针 */
    tied_ = true;
}

/* 更新在poller上注册的事件 */
void Channel::update() {
    /* 通过Channel所属的EventLoop 调用Poller的相应方法注册fd的events事件 */
    // add code
    // loop_->updateChannel(this);
}

/* 在Channel所属的EventLoop中 删除当前的Channel */
void Channel::remove() {
    // add code
    // loop_->removeChannel(this);
}

/* 处理事件 调用相应的回调 */
void Channel::handleEvent(Timestamp receiveTime) {
    if (tied_) {
        std::shared_ptr<void> guard = tie_.lock(); /* 提升 */
        if (guard) { /* 提升成功 */
            handleEventWithGuard(receiveTime);
        }
    } else {
        handleEventWithGuard(receiveTime);
    }
}

/* 受保护的handleEvent  由handleEvent调用 */
void Channel::handleEventWithGuard(Timestamp receiveTime) {

    LOG_INFO("Channel handleEvent revents: %d", revents_);

    /* 执行相应的回调 */
    if ((revents_ & EPOLLHUP) && !(revents_ & EPOLLIN)) {
        /* 关闭 如果有closeCallback_就执行之*/
        if (closeCallback_) {
            closeCallback_();
        }
    }
    if (revents_ & EPOLLERR) {
        /* 出错 如果有errorCallback_就执行之 */
        if (errorCallback_) {
            errorCallback_();
        }
    }
    if (revents_ & (EPOLLIN | EPOLLPRI)) {
        /* 可读事件 */
        if (readCallback_) {
            readCallback_(receiveTime);
        }
    }
    if (revents_ & EPOLLOUT) {
        /* 可写事件 */
        if (writeCallback_) {
            writeCallback_();
        }
    }
}
```
