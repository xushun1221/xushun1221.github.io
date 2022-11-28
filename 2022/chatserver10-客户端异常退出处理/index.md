# 【ChatServer】10 - 客户端异常退出处理


客户端异常退出，客户端没有向服务器发送下线消息，而是直接断开连接。这种情况下，我们需要进行在数据库中更改用户的登录状态。

用户连接异常断开，我们并不能得到用户的id，只能得到用户的连接，需要在在线用户连接哈希表中遍历得到用户的id。

```cpp
void ChatServer::onConnection(const TcpConnectionPtr &conn)
{
    // client close connection
    if (!conn->connected())
    {
        // 连接异常断开
        ChatService::instance()->clientCloseException(conn);
        conn->shutdown();
    }
}
```


```cpp
void ChatService::clientCloseException(const TcpConnectionPtr &conn)
{
    // 某个连接异常下线 找到该用户id删除连接信息 并更改在线状态
    User user;
    {
        std::lock_guard<std::mutex> lock(connectionMutex);
        for (auto itr = userConnectionMap_.begin(); itr != userConnectionMap_.end(); ++ itr) {
            if (itr->second == conn) {
                user.setId(itr->first);
                // 移除连接信息
                userConnectionMap_.erase(itr);
                break;
            }
        }
    }
    if (user.getId() != 0) {
        // 找到了连接对应的id 更改在线状态
        user.setState("offline");
        userModel_.updateState(user);
    }
}

```
