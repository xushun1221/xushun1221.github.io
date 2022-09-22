---
title: 【重写muduo】17 - TcpServer服务器类
date: 2022-09-22
tags: [network, C++, muduo]
categories: [Networking]
---

到这里，muduo中和`TcpServer`相关的组件基本都写好了，我们先把`TcpServer`写个大概。

因为`TcpConnection`的内容还没写，和它相关的就先空着。



## 源码

`TcpServer.hh`  
```cpp
#ifndef   __TCPSERVER_HH_
#define   __TCPSERVER_HH_

#include "noncopyable.hh"
#include "EventLoop.hh"
#include "EventLoopThread.hh"
#include "EventLoopThreadPool.hh"
#include "Acceptor.hh"
#include "InetAddress.hh"
#include "Callbacks.hh"

#include <functional>
#include <string>
#include <memory>
#include <atomic>
#include <unordered_map>


/* TcpServer服务器类 服务器编程的入口类 */
class TcpServer : noncopyable {
public:
    using ThreadInitCallback = std::function<void(EventLoop*)>; /* EventLoopThread创建时对loop进行操作的回调类型 */
    enum Option { kNoReusePort, kReusePort };

    TcpServer(EventLoop* loop, 
              const InetAddress& listenAddr, 
              const std::string& nameArg, 
              Option option = kNoReusePort);
    ~TcpServer();

    /* 设置各种回调 */
    void setThreadInitCallback(const ThreadInitCallback& cb) { threadInitCallback_ = cb; }
    void setConnectionCallback(const ConnectionCallback& cb) { connectionCallback_ = cb; }
    void setMessageCallback(const MessageCallback& cb) { messageCallback_ = cb; }
    void setWriteCompleteCallback(const WriteCompleteCallback& cb) { writeCompleteCallback_ = cb; }
    /* 设置subloop的数量 */
    void setThreadNum(int numThreads);
    /* 启动服务器 */
    void start();
private:
    using ConnectionMap = std::unordered_map<std::string, TcpConnectionPtr>;

    /* 连接相关 */
    void newConnection(int sockfd, const InetAddress& peerAddr); /* 新连接分配给subloop的回调 */
    void removeConnection(const TcpConnectionPtr& conn);
    void removeConnectionInLoop(const TcpConnectionPtr& conn);

/* 组件 */
    EventLoop* loop_; /* baseloop是由用户传入的 */
    const std::string ipPort_;
    const std::string name_;
    std::unique_ptr<Acceptor> acceptor_; /* 运行在mainloop监听新连接 */
    std::shared_ptr<EventLoopThreadPool> threadPool_; /* one loop per thread */
/* 回调 */
    ConnectionCallback connectionCallback_; /* 处理新连接用户的回调 */
    MessageCallback messageCallback_;       /* 处理已连接用户的读写事件的回调 */
    WriteCompleteCallback writeCompleteCallback_; /* 消息发送完成后的回调 */
    ThreadInitCallback threadInitCallback_; /* 线程创建时对loop进行处理的回调 */

    std::atomic_int started_;

    int nextConnId_;
    ConnectionMap connections_; /* 保存所有的连接 */
};


#endif // __TCPSERVER_HH_
```


`TcpServer.cc`  
```cpp
#include "TcpServer.hh"
#include "Logger.hh"



EventLoop* CheckLoopNotNull(EventLoop* loop) {
    if (loop == nullptr) {
        /* mainloop为空不能初始化TcpServer 严重错误 */
        LOG_FATAL("CheckLoopNotNull mainloop is null \n");
    }
    return loop;
}


TcpServer::TcpServer(EventLoop* loop, 
const InetAddress& listenAddr, 
const std::string& nameArg, 
Option option) 
    : loop_(CheckLoopNotNull(loop)) /* loop_初始化时必须不为空 */
    , ipPort_(listenAddr.toIpPort())
    , name_(nameArg)
    , acceptor_(new Acceptor(loop, listenAddr, option == kReusePort))
    , threadPool_(new EventLoopThreadPool(loop, name_))
    , connectionCallback_()
    , messageCallback_()
    , nextConnId_(1)
    {
        /* 新用户连接时执行TcpServer::newConnection 分配subloop */
        /* 运行在mainloop中 Acceptor::handleRead */
        acceptor_->setNewConnectionCallback(
            std::bind(&TcpServer::newConnection, 
            this, std::placeholders::_1, std::placeholders::_2));
}


TcpServer::~TcpServer() {

}


/* 根据轮询算法选择一个subloop 唤醒subloop 把当前connfd封装为相应的channel 分发给subloop */
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr) {

}

/* 设置subloop的数量 */
void TcpServer::setThreadNum(int numThreads) {
    threadPool_->setThreadNum(numThreads);
}

/* 启动服务器 */
void TcpServer::start() {
    if (started_ ++ == 0) {
        /* 防止TcpServer对象被start多次 */
        threadPool_->start(threadInitCallback_); /* subloop全部启动 */
        loop_->runInLoop(std::bind(&Acceptor::listen, acceptor_.get())); /* 在mainloop上注册listenfd */
    }
    /* 调用完该方法 马上就会调用loop.loop()方法开启mainloop */
}
```