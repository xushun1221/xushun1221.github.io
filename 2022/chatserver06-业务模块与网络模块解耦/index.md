# 【ChatServer】06 - 业务模块与网络模块解耦


## 业务与网络模块解耦

我们使用了muduo网络库搭建服务器框架，可以直接注册回调函数对客户端的消息进行处理，但是这样业务模块就和网络模块耦合了。

我们需要将业务模块和网络模块解耦，使用另一个单例类`ChatService`，包装业务处理回调，根据消息类型获取对应的业务处理回调。这样在网络模块中，就完全没有业务代码了。

```cpp
/*
 *  @Filename : ChatService.hh
 *  @Description : Service Business of ChatServer
 *  @Datatime : 2022/11/27 16:16:46
 *  @Author : xushun
 */
#ifndef __CHATSERVICE_HH_
#define __CHATSERVICE_HH_

#include "json.hpp"
#include <muduo/net/TcpConnection.h>
#include <unordered_map>
#include <functional>

using namespace muduo;
using namespace muduo::net;
using json = nlohmann::json;

// 处理消息的回调类型
using MsgHandler = std::function<void(const TcpConnectionPtr &conn, json &js, Timestamp time)>;

// 业务类  单例
class ChatService
{
public:
    // 获取单例对象
    static ChatService *instance();
    // 获取消息对应的处理器
    MsgHandler getHandler(int msgType);

    // 登录业务
    void login(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 注册业务
    void registr(const TcpConnectionPtr &conn, json &js, Timestamp time);

private:
    // 消息类型对应的业务处理方法
    std::unordered_map<int, MsgHandler> msgHandlerMap_;

private:
    // 在构造中注册消息类型和其对应的回调操作
    ChatService();
};

#endif // __CHATSERVICE_HH_
```

```cpp
/*
 *  @Filename : ChatService.cc
 *  @Description : Implementation of ChatService
 *  @Datatime : 2022/11/27 16:50:04
 *  @Author : xushun
 */

#include "ChatService.hh"
#include "public.hh"
#include <muduo/base/Logging.h>
#include <string>

using namespace std::placeholders;
using namespace muduo;

ChatService::ChatService()
{
    msgHandlerMap_[LOGIN_MSG] = std::bind(&ChatService::login, this, _1, _2, _3);
    msgHandlerMap_[REG_MSG] = std::bind(&ChatService::registr, this, _1, _2, _3);
}

ChatService *ChatService::instance()
{
    static ChatService service;
    return &service;
}

MsgHandler ChatService::getHandler(int msgType)
{
    auto itr = msgHandlerMap_.find(msgType);
    // 错误的msgType 不存在对应的消息处理回调
    if (itr == msgHandlerMap_.end())
    {
        // 返回一个空的处理器 不做任何处理  打印错误日志
        return [=](const TcpConnectionPtr &conn, json &js, Timestamp time)
        {
            LOG_ERROR << "MSG_TYPE: " << msgType << " cannot find a handler!";
        };
    }
    return itr->second;
}

void ChatService::login(const TcpConnectionPtr &conn, json &js, Timestamp time)
{
}

void ChatService::registr(const TcpConnectionPtr &conn, json &js, Timestamp time)
{
}
```

```cpp
/*
 *  @Filename : ChatServer.cc
 *  @Description : ChatServer Implementation
 *  @Datatime : 2022/11/27 15:38:53
 *  @Author : xushun
 */


void ChatServer::onMessage(const TcpConnectionPtr &conn, Buffer *buffer, Timestamp time)
{
    // get message
    std::string buf = buffer->retrieveAllAsString();
    // parse message data
    json js = json::parse(buf);
    // 获取消息对应的事件处理器
    auto msgHandler = ChatService::instance()->getHandler(js["msg_type"].get<int>());
    // 处理
    msgHandler(conn, js, time);
    // 解耦 业务代码 和 网络代码
}
```
