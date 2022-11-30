---
title: 【ChatServer】12 - 服务器异常中止处理
date: 2022-11-30
tags: [Linux, C++, MySQL]
categories: [Projects]
---

之前测试功能时，如果服务器异常中止，那么当前的在线用户的在线状态并不会被更改为离线（没有机会更改数据库状态），如果下次服务器启动，用户就无法再次登录了。

处理方法也很简单，截获中止信号，在信号处理函数中重置用户的在线状态。

```cpp
// 服务器异常中止 重置用户在线状态
void resetHandler(int)
{
    ChatService::instance()->reset();
    exit(0);
}

int main(int argc, char **argv)
{

    // 捕获程序中止信号 重置用户在线状态信息
    signal(SIGINT, resetHandler);

    EventLoop loop;
    InetAddress addr("127.0.0.1", 6000);
    ChatServer server(&loop, addr, "ChatServer", 8);

    server.start();
    loop.loop();

    return 0;
}
```

```cpp
void ChatService::reset()
{
    // 重置所有在线用户的在线状态
    userModel_.resetStateAll();
}
```

```cpp
bool UserModel::resetStateAll()
{
    // 组装sql
    char sql[1024] = {0};
    sprintf(sql, "UPDATE user SET state='offline' WHERE state='online'");
    // 连接mysqlserver并修改记录
    MySQL mysql;
    if (mysql.connect())
    {
        if (mysql.update(sql))
        {
            return true;
        }
    }
    return false;
}
```