# 【ChatServer】09 - 用户注册和登录业务




## 用户注册业务

用户注册业务在`ChatService`中完成`registr`方法即可。

客户端发送`REG_MSG`类型的消息，并附上`name, password`，服务器返回`REG_ACK`消息，并附上`id`。

```cpp
void ChatService::registr(const TcpConnectionPtr &conn, json &js, Timestamp time)
{
    std::string name = js["name"];
    std::string password = js["password"];

    User user;
    user.setName(name);
    user.setPassword(password);

    if (userModel_.insert(user)) {
        // 注册成功 发送响应消息给用户并且告知用户id
        json response;
        response["msg_type"] = REG_ACK;
        response["errno"] = 0;
        response["id"] = user.getId();
        conn->send(response.dump());
    } else {
        // 注册失败
        json response;
        response["msg_type"] = REG_ACK;
        response["errno"] = 1;
        conn->send(response.dump());
    }

}
```


## 用户登录业务

用户登录业务在`ChatService`中完成`login`方法即可。

客户端发送`LOGIN_MSG`类型的消息，并附上`id, password`，服务器返回`LOGIN_ACK`消息，并附上`id, name`。

注意，用户登录成功要在数据库中更改用户当前状态。

如果用户已登录，则不允许再次登录。如果用户id或密码错误，同样不允许登录。

> 用户登录成功后，我们需要记录用户id和用于通信的连接(`TcpConnectionPtr`)，因为如果有其他用户要发送消息给已登录用户，我们需要根据id找到通信连接发送消息。这里使用哈希表实现。  
> 注意，使用`unordered_map`在多线程环境下一定要注意线程安全问题，我们需要使用互斥锁。

```cpp
void ChatService::login(const TcpConnectionPtr &conn, json &js, Timestamp time)
{
    uint32_t id = js["id"].get<uint32_t>();
    std::string password = js["password"];

    User user = userModel_.queryById(id);
    if (user.getId() != 0 && user.getPassword() == password)
    {
        // id存在并且密码正确
        if (user.getState() == "online")
        {
            // 该用户已登录 不允许重复登录
            json response;
            response["msg_type"] = LOGIN_ACK;
            response["err_no"] = 2;
            response["err_msg"] = "already logged in, donot login again";
            conn->send(response.dump());
        }
        else
        {
            // 该用户未登录 登录成功
            // 登录成功需要记录该用户的连接信息
            {
                std::lock_guard<std::mutex> lock(connectionMutex);
                userConnectionMap_.insert({id, conn});
            }

            // 登录成功需要更新用户状态信息
            user.setState("online");
            userModel_.updateState(user);

            // 返回响应信息
            json response;
            response["msg_type"] = LOGIN_ACK;
            response["err_no"] = 0;
            response["id"] = user.getId();
            response["name"] = user.getName(); // 返回用户的信息
            conn->send(response.dump());
        }
    }
    else
    {
        // 登录失败
        json response;
        response["msg_type"] = LOGIN_ACK;
        response["err_no"] = 1;
        response["err_msg"] = "wrong id or password";
        conn->send(response.dump());
    }
}
```
