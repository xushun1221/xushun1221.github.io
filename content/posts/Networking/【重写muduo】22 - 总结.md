---
title: 【重写muduo】22 - 总结
date: 2022-10-06
tags: [network, C++, muduo]
categories: [Networking]
---

muduo库的核心部分终于重写完了，我们来总结一下各个核心组件的功能和联系。

## Channel

`Channel`是对sockfd和其感兴趣的事件以及对应的回调函数的封装。

核心组件：  
- `EventLoop* loop_`：Channel所属的EventLoop（用于和EventLoop上的Poller通信）；
- `const int fd_`：Channel对应的文件描述符，注册到Poller上的sockfd；
- `int events_`：Channel感兴趣的事件；
- `int revents_`：Poller给Channel返回的发生的具体事件，据此调用下方相应的回调函数；
- 一组回调函数：
  - `ReadEventCallback readCallback_`
  - `EventCallback writeCallback_`
  - `EventCallback closeCallback_`
  - `EventCallback errorCallback_`

> Channel和Poller是互相独立的，它们都从属于EventLoop，所以使用一个EventLoop的指针来进行通信。

核心功能：  
- 向Poller上注册（更新）fd感兴趣的事件，`void update();`；
- 获得Poller返回的fd发生的事件，`void set_revents();`，（由Poller调用）；
- 设置各类事件对应的回调函数，很明显，回调函数来自于Channel的上层Acceptor或TcpConnection；
- 根据返回的发生事件和设置的回调函数对事件进行处理，`void handleEvent(Timestamp receiveTime);`。


> muduo库中有两种不同的sockfd，一种是用于监听新连接的**listenfd**，另一种是用于处理已连接用户IO事件的**connfd**。这两种sockfd和Poller的交互都使用**Channel**进行封装，但由于两种sockfd执行的功能不同，我们使用`Acceptor`和`TcpConnection`两个类来分别封装这两种Channel。  
> 对于socket操作的细节，通过`Socket`类来封装。


## Poller和EPollPoller - Demultiplex

> muduo库支持`epoll`和`poll`两种IO复用模型，所以使用抽象类`Poller`来为所有的IO复用模型提供统一的接口。`EPollPoller`是使用`epoll`方法的事件分发器类。

`EPollPoller`的核心组件：  
- Poller基类的组件
  - `EventLoop* ownerLoop_`，Poller所属的EventLoop，用于和注册到该Poller上的Channel通信；
  - `ChannelMap channels_`，（`using ChannelMap = std::unordered_map<int, Channel*>;`）用一个hashmap来保存所有注册到该Poller上的Channel，key是Channel对应的sockfd；
- `int epollfd_`，epoll的文件描述符；
- `EventList events_`，（`using EventList = std::vector<epoll_event>;`），用于接收发生事件的epoll_event，用作epoll_wait的传出参数。


核心功能：  
- 事件分发器获得发生事件的sockfd（对`epoll_wait`的封装）
  - `Timestamp poll(int timeoutMs, Poller::ChannelList* activeChannels);`（timeoutMs是超时时长，activeChannels是有监听事件发生的Channel列表，交由EventLoop进行处理）
- 注册（更新）Channel在Poller中的状态（对`epoll_ctl`的封装）
  - `void updateChannel(Channel* channel);`
- 将注册在Poller上的Channel删除（对`epoll_ctl`的封装）
  - `void removeChannel(Channel* channel);`


## EventLoop - Reactor

`EventLoop`事件循环类，是Reactor模型的核心，它是对Poller事件分发器的进一步封装，管理属于该EventLoop的所有Channel以及一个Poller事件分发器，为Channel和Poller提供一个通信的途径。它的主要功能是，在有事件到来时，执行Channel对应事件的回调，或者执行其他组件发送来的回调函数。

核心组件：  
- `std::unique_ptr<Poller> poller_`：Poller；
- `int wakeupFd_`和`std::unique_ptr<Channel> wakeupChannel_`：用于唤醒事件循环的eventfd；
- `ChannelList activeChannels_`，（`using ChannelList = std::vector<Channel*>;`），保存事件分发器返回的发送事件的Channel；
- `std::vector<Functor> pendingFunctors_`，（using Functor = std::function<void()>;），用于保存其他组件发送来的需要当前Loop执行的所有回调函数。


核心功能：  
- 进行事件循环，执行发生事件的Channel对应的回调，以及其他应该执行的回调
  - `void loop();`
  - `void doPendingFunctors();`
- 在当前Loop中执行指定回调，如果当前在该Loop对应的Thread中，直接执行，否则放入队列等待执行
  - `void runInLoop(Functor cb);`
  - `void queueInLoop(Functor cb);`
- 唤醒功能
  - `void wakeup();`


> 在Poller上没有事件发生的情况下，事件循环会阻塞在`poll`处，如果此时有其他需要处理的回调（如Acceptor接收了新的连接，并将其注册到Poller上），那么就无法被及时处理  
> 我们使用eventfd（`wakeupFd_`）来解决这个问题，在创建EventLoop时我们使用`eventfd()`系统调用创建一个用于消息通知的fd，封装为Channel，并将其读事件注册到Poller上，当我们有回调操作需要Loop立即处理时，只需向`pendingFunctors_`添加回调操作，并向该Loop对应的wakeupFd_写入8字节数据，就可以让当前Loop从阻塞状态唤醒，立即处理回调。



## Thread、EventLoopThread、EventLoopThreadPool

muduo网络库中的一个最重要的概念就是：**One Loop per Thread**，每个事件循环都在一个单独的线程中执行。

`Thread`线程类，封装了对底层线程的操作；  
`EventLoopThread`事件循环线程类，实现了启动事件循环线程的操作；  
`EventLoopThreadPool`事件循环线程池类，封装了多个事件循环线程。


每一个TcpServer中都有一个EventLoopThreadPool，当TcpServer启动时，EventLoopThreadPool也被启动，EventLoopThreadPool会根据用户设定的线程数量创建多个子线程(EventLoopThread)，用于处理已连接fd的IO事件，这样的事件循环线程称为**SubLoop**，而用于接收新连接的事件循环称为**BaseLoop**，它作为参数传入TcpServer。

每个线程创建完成后，就立即执行事件循环了(`loop()`)。


值得一提的是，每当BaseLoop接收到一个新连接时，会按**轮询**的方式选择一个SubLoop并注册，由该Loop处理该连接的IO事件。线程池会使用`EventLoop* getNextLoop();`方法给出下一个接收连接的LoopThread，如果用户没有设置额外的线程用于IO操作，那么所有的事件都由MAINLOOP来处理。




## Acceptor

`Acceptor`实际上就是对listenfd的封装，将listenfd封装为一个acceptChannel，并交由BaseLoop监听新用户连接，当有新用户连接时，调用`NewConnectionCallback newConnectionCallback_;`回调函数进行处理。



## Buffer

muduo库使用`Buffer`缓冲区来实现相关的IO操作。

当某个连接要发送数据时，它不会直接写入内核的TCP发送缓冲区，而是，写入该连接自带的发送缓冲区，并注册写事件，当可以发送时，写事件回调会将连接的发送缓冲区中的数据写入TCP发送缓冲区，进行发送。

当某个连接接收到数据时，它会调用写事件回调，将接收的数据写入该连接自己的接收缓冲区，并调用`messageCallback_`，进行处理。

实现细节和原理在源码中有非常详细的注解。



## TcpConnection

`TcpConnection`类，是对一个成功连接的客户Channel的封装，它管理对这个Channel的所有操作。

核心组件：  
- `EventLoop* loop_`
- `std::unique_ptr<Channel> channel_`
- `Buffer inputBuffer_`
- `Buffer outputBuffer_`
- 连接的各种回调（和Channel的回调不同）
    - ConnectionCallback connectionCallback_;
    - MessageCallback messageCallback_;
    - WriteCompleteCallback writeCompleteCallback_;
    - HighWaterMarkCallback highWaterMarkCallback_;
    - CloseCallback closeCallback_;

核心功能：  
- 建立连接，`void connectEstablished();`
- 销毁连接，`void connectDestroyed();`
- 发送数据，`void send(const std::string& buf);`
- 关闭连接，`void shutdown();`
- Channel的各类事件回调
  - `void handleRead(Timestamp receiveTime);`
  - `void handleWrite();`
  - `void handleClose();`
  - `void handleError();`


## TcpServer

`TcpServer`是muduo库服务器编程的入口类，它管理所有的EventLoop、EventLoopThreadPool、Acceptor、所有的TcpConnection连接，以及连接事件的回调。



