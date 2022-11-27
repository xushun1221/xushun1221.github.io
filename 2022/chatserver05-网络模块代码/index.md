# 【ChatServer】05 - 网络模块代码


## 网络模块代码

使用muduo库，构建基于Reactor模型的服务器框架。


```cpp
/*
 *  @Filename : ChatServer.hh
 *  @Description : ChatServer Definition
 *  @Datatime : 2022/11/27 15:29:18
 *  @Author : xushun
 */
#ifndef __CHATSERVER_HH_
#define __CHATSERVER_HH_

#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>

using namespace muduo;
using namespace muduo::net;

class ChatServer
{
public:
    // init chatserver
    ChatServer(EventLoop *loop, const InetAddress &listenAddr, const string &nameArg, uint32_t threadNum);
    // start server
    void start();

private:
    // server class in muduo lib
    TcpServer server_;
    // mainloop
    EventLoop *loop_;

private:
    // callback: connection create and shutdown
    void onConnection(const TcpConnectionPtr &conn);
    // callback: user io event
    void onMessage(const TcpConnectionPtr &conn, Buffer *buffer, Timestamp time);
};

#endif // __CHATSERVER_HH_
```

```cpp
/*
 *  @Filename : ChatServer.cc
 *  @Description : ChatServer Implementation
 *  @Datatime : 2022/11/27 15:38:53
 *  @Author : xushun
 */
#include "ChatServer.hh"
#include <functional>

using namespace std::placeholders;

ChatServer::ChatServer(EventLoop *loop, const InetAddress &listenAddr, const string &nameArg, uint32_t threadNum)
    : server_(loop, listenAddr, nameArg), loop_(loop)
{
    // register ConnectionCallback
    server_.setConnectionCallback(std::bind(&ChatServer::onConnection, this, _1));
    // register MessageCallback
    server_.setMessageCallback(std::bind(&ChatServer::onMessage, this, _1, _2, _3));
    // set thread number: 1 acceptor loop, threadNum - 1 event loop
    server_.setThreadNum(threadNum);
}

void ChatServer::start()
{
    // start server
    server_.start();
}

void ChatServer::onConnection(const TcpConnectionPtr &conn)
{
}

void ChatServer::onMessage(const TcpConnectionPtr &conn, Buffer *buffer, Timestamp time)
{
}
```

```cpp
/*
*  @Filename : main.cc
*  @Description : ChatServer Entry
*  @Datatime : 2022/11/27 15:53:16
*  @Author : xushun
*/
#include "ChatServer.hh"

int main(int argc, char** argv) {

    EventLoop loop;
    InetAddress addr("127.0.0.1", 6000);
    ChatServer server(&loop, addr, "ChatServer", 8);

    server.start();
    loop.loop();

    return 0;
}
```
