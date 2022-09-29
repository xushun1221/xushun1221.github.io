# 【重写muduo】19 - TcpConnection


`TcpConnection`管理一个socket及其对应的channel、tcp连接状态、一对输入输出缓冲区、设置channel的各种回调函数、发送数据方法、关闭连接方法。




## 源码

`TcpConnection.hh`  
```cpp
#ifndef   __TCPCONNECTION_HH_
#define   __TCPCONNECTION_HH_

#include "noncopyable.hh"
#include "InetAddress.hh"
#include "Callbacks.hh"
#include "Buffer.hh"
#include "Timestamp.hh"

#include <memory>
#include <string>
#include <atomic>

class Channel;
class EventLoop;
class Socket;


/* TcpConnection代表一条已经建立的客户端连接 */
class TcpConnection : noncopyable, public std::enable_shared_from_this<TcpConnection> {
public:
    TcpConnection(EventLoop* loop,
                  const std::string& nameArg,
                  int sockfd,
                  const InetAddress& localAddr,
                  const InetAddress& peerAddr);
    ~TcpConnection();

    EventLoop* getLoop() const { return loop_; }
    const std::string& name() const { return name_; }
    const InetAddress& localAddress() const { return localAddr_; }
    const InetAddress& peerAddress() const { return peerAddr_; }
    Buffer* inputBuffer() { return &inputBuffer_; }
    Buffer* outputBuffer() { return &outputBuffer_; }

    bool connected() const { return state_ == kConnected; }
    bool disconnected() const { return state_ == kDisconnected; }

    /* 发送数据 调用sendInLoop */
    void send(const std::string& buf);
    /* 关闭连接 调用shutdownInLoop */
    void shutdown();

    /* 设置回调 */
    void setConnectionCallback(const ConnectionCallback& cb) { connectionCallback_ = cb; }
    void setMessageCallback(const MessageCallback& cb) { messageCallback_ = cb; }
    void setWriteCompleteCallback(const WriteCompleteCallback& cb) { writeCompleteCallback_ = cb; }
    void setHighWaterMarkCallback(const HighWaterMarkCallback& cb, size_t highWaterMark) { 
        highWaterMarkCallback_ = cb;
        highWaterMark_ = highWaterMark;
    }
    void setCloseCallback(const CloseCallback& cb) { closeCallback_ = cb; }

    /* 连接建立 */
    void connectEstablished();
    /* 连接销毁 */
    void connectDestroyed();
    
private:
    /* TCP状态 */
    enum StateE { kDisconnected, kConnecting, kConnected, kDisconnecting };
    void setState(StateE state) { state_ = state; }

    /* 事件处理回调 */
    void handleRead(Timestamp receiveTime);
    void handleWrite();
    void handleClose();
    void handleError();

    /* 由当前loop发送数据 */
    void sendInLoop(const void* message, size_t len);
    /* 在当前loop中删除掉对应的channel */
    void shutdownInLoop();

    EventLoop* loop_;        /* 从属的subloop */
    const std::string name_;
    std::atomic_int state_;  /* TCP状态 */
    bool reading_;

    std::unique_ptr<Socket> socket_;
    std::unique_ptr<Channel> channel_;

    const InetAddress localAddr_;
    const InetAddress peerAddr_;

    ConnectionCallback connectionCallback_;
    MessageCallback messageCallback_;
    WriteCompleteCallback writeCompleteCallback_;
    HighWaterMarkCallback highWaterMarkCallback_; /* 高水位标记回调 */
    CloseCallback closeCallback_;
    size_t highWaterMark_; /* 高水位标记 */

    Buffer inputBuffer_;    /* 接收数据缓冲区 */
    Buffer outputBuffer_;   /* 发送数据缓冲区 */
};


#endif // __TCPCONNECTION_HH_
```



`TcpConnection.cc`  
```cpp
#include "TcpConnection.hh"
#include "Logger.hh"
#include "Socket.hh"
#include "Channel.hh"
#include "EventLoop.hh"

#include <functional>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <string.h>
#include <netinet/tcp.h>

static EventLoop* CheckLoopNotNull(EventLoop* loop) {
    if (loop == nullptr) {
        /* mainloop为空不能初始化TcpServer 严重错误 */
        LOG_FATAL("CheckLoopNotNull TcpConnection loop is null \n");
    }
    return loop;
}

TcpConnection::TcpConnection(EventLoop* loop,
                             const std::string& nameArg,
                             int sockfd,
                             const InetAddress& localAddr,
                             const InetAddress& peerAddr) 
    : loop_(CheckLoopNotNull(loop))
    , name_(nameArg)
    , state_(kConnecting) /* 初始时正在连接 */
    , reading_(true)
    , socket_(new Socket(sockfd))
    , channel_(new Channel(loop, sockfd))
    , localAddr_(localAddr)
    , peerAddr_(peerAddr)
    , highWaterMark_(64 * 1024 * 1024) /* 64M */
    {
        /* 为Channel设置回调 */
        channel_->setReadCallback(
            std::bind(&TcpConnection::handleRead, this, std::placeholders::_1)
        );
        channel_->setWriteCallback(
            std::bind(&TcpConnection::handleWrite, this)
        );
        channel_->setCloseCallback(
            std::bind(&TcpConnection::handleClose, this)
        );
        channel_->setErrorCallback(
            std::bind(&TcpConnection::handleError, this)
        );
        LOG_INFO("TcpConnection create[%s] at fd=%d \n", name_.c_str(), sockfd);
        socket_->setKeepAlive(true);
}

TcpConnection::~TcpConnection() {
    LOG_INFO("TcpConnection destroyed[%s] at fd=%d state=%d \n", 
                name_.c_str(), channel_->fd(), (int)state_);
}


void TcpConnection::handleRead(Timestamp receiveTime) {
    int savedErrno = 0;
    /* fd数据写入缓冲区 */
    ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno);
    if (n > 0) {
        /* 已连接的客户 有可读事件发生 调用用户传入的回调onMessage */
        messageCallback_(shared_from_this(), &inputBuffer_, receiveTime);
        /* 注意 shared_from_this不属于std:: */
    } else if (n == 0) {
        /* 客户端断开连接 */
        handleClose();
    } else {
        /* 出错 */
        errno = savedErrno;
        LOG_ERROR("TcpConnection::handleRead error \n");
        handleError(); /* 当错误发生时 发生错误的sockfd会被标记为可读写 */
    }
}

void TcpConnection::handleWrite() {
    /* 判断channel是否注册了写事件 */
    if (channel_->isWriting()) {
        int saveErrno = 0;
        /* 将缓冲区中的数据写入fd */
        ssize_t n = outputBuffer_.writeFd(channel_->fd(), &saveErrno);
        if (n > 0) {
            /* 成功写出数据 从Buffer中删掉写出的n个字符 */
            outputBuffer_.retrieve(n);
            if (outputBuffer_.readableBytes() == 0) {
                /* 发送完成了 设置channel不可写 */
                channel_->disableWriting();
                if (writeCompleteCallback_) {
                    /* 如果注册过写完的回调 调用它 */
                    loop_->queueInLoop(
                        std::bind(writeCompleteCallback_, shared_from_this())
                    );
                }
                if (state_ == kDisconnecting) {
                    /* 连接正在断开 */
                    shutdownInLoop();
                }
            }
        } else { /* n <= 0 */
            /* 出错了 */
            LOG_ERROR("TcpConnection::handleWrite error \n");
        }
    } else {
        LOG_ERROR("TcpConnection fd=%d is down, no more writing \n", channel_->fd());
    }
}

void TcpConnection::handleClose() {
    LOG_INFO("fd=%d state=%d \n", channel_->fd(), (int)state_);
    setState(kDisconnected); /* 设置已断开状态 */
    channel_->disableAll();  /* 摘下所有监听事件 */

    TcpConnectionPtr guardThis(shared_from_this());
    connectionCallback_(guardThis); /* 执行连接关闭的回调 */
    closeCallback_(guardThis);      /* 执行关闭连接的回调 */
}

void TcpConnection::handleError() {
    int optval;
    socklen_t optlen = static_cast<socklen_t>(sizeof(optval));
    int err = 0;
    /* 获取socket的错误 */
    if (::getsockopt(channel_->fd(), SOL_SOCKET, SO_ERROR, &optval, &optlen) < 0) {
        err = errno;
    } else {
        err = optval;
    }
    LOG_ERROR("TcpConnection::handleError name:%s - SO_ERROR:%d \n", name_.c_str(), err);
}



/* 发送数据 调用sendInLoop */
void TcpConnection::send(const std::string& str) {
    if (state_ == kConnected) {
        /* 由loop线程发送数据 */
        if (loop_->isInLoopThread()) {
            sendInLoop(str.c_str(), str.size());
        } else {
            loop_->runInLoop(std::bind(&TcpConnection::sendInLoop, this, str.c_str(), str.size()));
        }
    }
}

/* 由当前loop线程发送数据 */
void TcpConnection::sendInLoop(const void* data, size_t len) {
    /*
        发送数据时 应用写的快 但是内核发送数据慢
        需要将待发送数据写入发送缓冲区 而且有高水位回调 防止发送太快
    */
    ssize_t nwrote = 0;      /* 已写入缓冲区的数据 */
    size_t remaining = len;  /* 待写的数据 */
    bool faultError = false; /* 是否产生错误 */
    if (state_ == kDisconnected) {
        /* 连接断开 无法发送数据 */
        LOG_ERROR("TcpConnection::sendInLoop disconnected give up writing\n");
        return;
    }
    /* 该channel第一次开始发送数据 缓冲区中无数据待发 */
    if (!channel_->isWriting() && outputBuffer_.readableBytes() == 0) {
        /* 尝试直接发送 */
        nwrote = ::write(channel_->fd(), data, len);
        if (nwrote >= 0) {
            /* 发送成功了 */
            remaining = len - nwrote; /* 剩下多少 */
            if (remaining == 0) {
                /* 全部发送成功 无需缓冲  也无需给channel注册EPOLLOUT事件了 也就不会执行handleWrite方法了 */
                loop_->queueInLoop(std::bind(writeCompleteCallback_, shared_from_this())); /* 执行发送完成回调 */
            }
        } else { /* nwrote < 0  出错 */
            nwrote = 0;
            if (errno != EWOULDBLOCK) { /* EWOULDBLOCK表示TCP发送缓冲区满 */
                LOG_ERROR("TcpConnection::sendInLoop write error \n");
                if (errno == EPIPE || errno == ECONNRESET) {
                    faultError = true;
                }
            }
        }
    }
    /* 
        没有发生错误 并且有剩余数据待发  
        剩余数据需要保存到缓冲区 然后给channel注册EPOLLOUT事件 
        当poller发现发送缓冲区有空间 就会通知相应的channel调用handleWrite回调
        最终就是调用TcpConnection::handleWrite方法 将发送缓冲区中待发数据全部发送完成
        (只有发送缓冲区中的数据全部发送完毕 才会将channel的EPOLLOUT事件关闭)
    */
    if (!faultError && remaining > 0) {
        size_t oldLen = outputBuffer_.readableBytes(); /* 原待发数据量 */
        if (oldLen + remaining >= highWaterMark_ /* 现在的待发数据 超过了高水位标记 */
                    && oldLen < highWaterMark_   /* 原待发数据不会超过高水位标记 如果超过了 肯定已经调用过高水位回调 */
                    && highWaterMarkCallback_){  /* 得设置过高水位回调 */
            /* 执行高水位回调 */
            loop_->queueInLoop(std::bind(highWaterMarkCallback_, shared_from_this(), oldLen + remaining));
        }
        /* 剩余的数据写入输出缓冲区 */
        outputBuffer_.append(static_cast<const char*>(data) + nwrote, remaining);
        if (!channel_->isWriting()) {
            /* 给channel设置EPOLLOUT事件 */
            channel_->enableWriting();
        }
    }
}

/* 连接建立 */
void TcpConnection::connectEstablished() {
    setState(kConnected);
    /* 将TcpConnection管理的channel绑定到TcpConnection上 */
    channel_->tie(shared_from_this());
    /* 注册读事件 */
    channel_->enableReading();
    /* 新连接建立 执行回调 */
    connectionCallback_(shared_from_this());
}

/* 连接销毁 */
void TcpConnection::connectDestroyed() {
    if (state_ == kConnected) {
        setState(kDisconnected);
        /* 注销所有监听事件 */
        channel_->disableAll();
        /* 连接销毁 执行回调 */
        connectionCallback_(shared_from_this());
    }
    /* 连接关闭了 就从EventLoop上取下 */
    channel_->remove();
}

/* 关闭连接 调用shutdownInLoop */
void TcpConnection::shutdown() {
    if (state_ == kConnected) {
        setState(kDisconnecting);
        loop_->runInLoop(std::bind(&TcpConnection::shutdownInLoop, this));
    }
}

/* 在当前loop中删除掉对应的channel */
void TcpConnection::shutdownInLoop() {
    if (!channel_->isWriting()) { /* 发送缓冲区中无待发数据了 */
        /* 关闭TCP的写端 会调用handleClose方法 */
        socket_->shutdownWrite();
        /* 关闭TCP写端 会触发socket的EPOLLHUP事件(EPOLLHUP不需要注册) */
    }
}
```
