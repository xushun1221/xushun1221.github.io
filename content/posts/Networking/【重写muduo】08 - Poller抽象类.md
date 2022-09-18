---
title: 【重写muduo】08 - Poller抽象类
date: 2022-09-18
tags: [network, C++, muduo]
categories: [Networking]
---

muduo中，使用`Poller`抽象类来抽象不同的IO复用方法，muduo支持`poll`和`epoll`两种IO复用。

在`Poller`实现中，我们使用以sockfd和Channel为键值对的哈希表来存储该`Poller`持有的所有`Channel`。

我们使用接口的方式来提供`Poller`派生类的实例化对象，`static Poller* newDefaultPoller(EventLoop* loop);`。注意，不能在`Poller.cc`中实现该接口，因为它会包含`EpollPoller.hh`等派生类的头文件，在基类实现中包含派生类的头文件是不好的。我们使用一个独立的源文件`DefaultPoller.cc`来提供该接口的实现。


## 源码

`Poller.hh`  
```cpp
#ifndef   __POLLER_HH_
#define   __POLLER_HH_

#include "noncopyable.hh"
#include "Timestamp.hh"

#include <vector>
#include <unordered_map>

class Channel;
class EventLoop;

/* Poller抽象类 多路事件分发器的核心IO复用模块 */
class Poller : noncopyable {
public:
    using ChannelList = std::vector<Channel*>;

    Poller(EventLoop* loop) : ownerLoop_(loop) {}
    virtual ~Poller() = default;

    /* 为所有IO复用方法 提供统一的接口 */
    virtual Timestamp poll(int timeoutMs, ChannelList* activeChannels) = 0; /* epoll_wait */
    virtual void updateChannel(Channel* channel) = 0;   /* epoll_ctl */
    virtual void removeChannel(Channel* Channel) = 0;   /* epoll_ctl */
    
    /* channel是否在当前Poller中 */
    virtual bool hasChannel(Channel* channel) const;

    /* EventLoop可以通过该接口获得默认IO复用的具体实现 */
    static Poller* newDefaultPoller(EventLoop* loop); /* 我们并不在Poller.cc中提供该方法的实现 */
protected:
    /* key为sockfd value为sockfd所属的Channel */
    using ChannelMap = std::unordered_map<int, Channel*>;
    ChannelMap channels_;  /* 该Poller持有的所有Channel */
private:
    EventLoop* ownerLoop_; /* 该Poller所属的EventLoop */
};


#endif // __POLLER_HH_
```



`Poller.cc`  
```cpp
#include "Poller.hh"
#include "Channel.hh"

/* channel是否在当前Poller中 */
bool Poller::hasChannel(Channel* channel) const {
    ChannelMap::const_iterator it = channels_.find(channel->fd());
    return it != channels_.end() && it->second == channel;
}



/* 
    如果我们在这里实现newDefaultPoller
    需要返回某个实例化的EpollPoller对象
    那势必需要在这里包含 派生类EpollPoller的头文件
    但是在基类的实现中包含派生类的头文件是不合适的
    所以我们不在这里实现newDeafultPoller
    使用一个独立的源文件DefaultPoller.cc来实现
*/

// #include "EpollPoller.hh"
// Poller* Poller::newDefaultPoller(EventLoop* loop) {
// }

```


`DefaultPoller.cc`  
```cpp
#include "Poller.hh"
// add code 
#include <stdlib.h>

/* EventLoop可以通过该接口获得默认IO复用的具体实现 */
Poller* Poller::newDefaultPoller(EventLoop* loop) {
    /* 默认提供epoll的实例 如果环境变量中有MUDUO_USE_POLL则返回poll的实例 */
    if (::getenv("MUDUO_USE_POLL")) {
        return nullptr; // add code 生成poll的实例
    } else {
        return nullptr; // add code 生成epoll的实例
    }
}

```