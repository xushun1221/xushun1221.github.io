# 【ChatServer】11 - 一对一聊天&离线消息



一对一聊天消息类型为：`PEER_CHAT_MSG`。

发送的消息类似于：`{"msg_type":PEER_CHAT_MSG, "from":myid, "to":peerid, "msg":msgcontent}`。

消息发送到服务器后，如果对方在线，直接找到peerid对应的connection进行转发。如果对方不在线，应将消息存入离线消息表，等待对方上线再通知。

在用户登录时，要查询离线消息表，如果有离线消息，则立即推送给用户。

对离线消息表进行的操作使用`OfflineMessageModel`类来封装。因为比较简单，就不使用ORM类了。

## peerChat

```cpp
void ChatService::peerChat(const TcpConnectionPtr &conn, json &js, Timestamp time)
{
    uint32_t peerid = js["to"].get<uint32_t>();
    // 目标用户是否在线
    std::unordered_map<uint32_t, TcpConnectionPtr>::iterator itr;
    {
        std::lock_guard<std::mutex> lock(connectionMutex);
        itr = userConnectionMap_.find(peerid);
        // 在线消息转发应该在临界区内操作 因为出临界区后连接可能被释放
        if (itr != userConnectionMap_.end())
        {
            // 在线 转发消息
            itr->second->send(js.dump());
            return;
        }
    }
    // 离线 存储离线消息
    offlineMessageModel_.insert(peerid, js.dump());
}
```

我觉得实现的有点点问题，在临界区发送消息是不是会影响并发能力？后面再考虑这个问题。

