# 【重写muduo】20 - TcpServer终章


写完`TcpConnection`之后，我们就可以来实现`TcpServer`的完整版本了。

`TcpConnection`主要做的事情就是，处理新到来的客户端连接，以及连接关闭时的处理。


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
    void newConnection(int sockfd, const InetAddress& peerAddr);
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
#include "TcpConnection.hh"

#include <string.h>

static EventLoop* CheckLoopNotNull(EventLoop* loop) {
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

/* 析构函数 关闭并释放所有的Tcp连接 */
TcpServer::~TcpServer() {
    for (auto& item : connections_) {
        TcpConnectionPtr conn(item.second); /* TcpConnectionPtr是shared_ptr */
        /* reset()方法会让item.second智能指针不再指向资源(TcpConnection)(引用计数-1) */
        item.second.reset();
        /* 销毁连接 */
        conn->getLoop()->runInLoop(std::bind(&TcpConnection::connectDestroyed, conn));
        /* 离开作用域后conn指向的TcpConnection被析构 */
    }
}


/* 当有新的客户端连接 acceptor对应的channel会执行Acceptor::handleRead回调 过程中会执行该newConnection回调 */
/* 根据轮询算法选择一个subloop 唤醒subloop 把当前connfd封装为相应的channel 分发给subloop */
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr) {
    /* 选择一个subloop来处理io事件 */
    EventLoop* ioLoop = threadPool_->getNextLoop();
    /* 新连接命名 */
    char buf[64] = {0};
    snprintf(buf, sizeof(buf), "-%s#%d", ipPort_.c_str(), nextConnId_);
    ++ nextConnId_;
    std::string connName = name_ + buf;
    LOG_INFO("TcpServer::newConnection [%s] - new connection [%s] from %s \n",
             name_.c_str(), connName.c_str(), peerAddr.toIpPort().c_str());
    /* 通过sockfd获取其绑定的本机ip和端口信息 */
    sockaddr_in local;
    bzero(&local, sizeof(local));
    socklen_t addrlen = sizeof(local);
    if (::getsockname(sockfd, (sockaddr*)&local, &addrlen) < 0) {
        LOG_ERROR("TcpServer::newConnection getLocalAddr error\n");
    }
    InetAddress localAddr(local);
    /* 根据成功连接的sockfd创建TcpConnection连接对象 Socket Channel */
    TcpConnectionPtr conn(new TcpConnection(ioLoop, connName, sockfd, localAddr, peerAddr));
    /* 保存创建的连接 */
    connections_[connName] = conn;
    /* 传入用户设置的回调 */
    /* 用户设置回调=>TcpServer=>TcpConnection=>Channel=>Poller=>notify Channel调用回调 */
    conn->setConnectionCallback(connectionCallback_);
    conn->setMessageCallback(messageCallback_);
    conn->setWriteCompleteCallback(writeCompleteCallback_);
    /* 设置了如何关闭连接的回调 removeConnection */
    conn->setCloseCallback(
        std::bind(&TcpServer::removeConnection, this, std::placeholders::_1)
    );

    /* 有新的TCP连接创建了TcpConnection后 直接调用TcpConnection::connectEstablisted方法 */
    ioLoop->runInLoop(std::bind(&TcpConnection::connectEstablished, conn));
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

/* TcpConnection连接断开时 handleClose执行的回调 */
void TcpServer::removeConnection(const TcpConnectionPtr& conn) {
    loop_->runInLoop(std::bind(
        &TcpServer::removeConnectionInLoop, this, conn
    ));
}
/* removeConnection调用的 在对应subloop中执行 */
void TcpServer::removeConnectionInLoop(const TcpConnectionPtr& conn) {
    LOG_INFO("TcpServer::removeConnectionInLoop [%s] - connection %s\n",
              name_.c_str(), conn->name().c_str());
    /* 在ConnectionMap中删除conn */
    connections_.erase(conn->name());
    /* 销毁TcpConnection::connectDestroyed */
    EventLoop* ioLoop = conn->getLoop();
    ioLoop->queueInLoop(std::bind(&TcpConnection::connectDestroyed, conn));
}
```
