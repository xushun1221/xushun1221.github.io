# 【ChatServer】13 - 添加好友业务


业务逻辑是这样滴，系统中的所有人都可以互发消息，无需好友身份。添加好友也无需对方同意，好友的功能类似于通讯录，客户端登录时，服务器会将该用户的好友列表发送给该客户，方便客户获取对方id。（这里为了方便起见这样设计XD）

注意，`friend`表中的`userid`和`friendid`是联合主键，只要一个用户添加了另一个用户，他们就互为好友了。

添加好友的消息类型为：`ADD_FRIEND_MSG`

对`friend`表进行数据操作，当然需要一个数据操作类`FriendModel`。

```cpp
bool FriendModel::insert(uint32_t userid, uint32_t friendid)
{
    // 组装sql
    char sql[1024] = {0};
    sprintf(sql, "INSERT INTO friend(userid,friendid) VALUES('%ld','%ld')", userid, friendid);
    // 连接mysqlserver并插入记录
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

std::vector<User> FriendModel::queryById(uint32_t userid)
{
    // 用于返回好友列表 如果没有返回空vector
    std::vector<User> vec;
    // 组装sql user和friend表的联合查询
    char sql[1024] = {0};
    sprintf(sql, "SELECT u.id,u.name,u.state FROM user u INNER JOIN friend f ON f.friendid=u.id WHERE f.userid=%ld", userid);
    // 连接mysqlserver并查询
    MySQL mysql;
    if (mysql.connect())
    {
        MYSQL_RES *res = mysql.query(sql);
        if (res)
        {
            MYSQL_ROW row;
            while (row = mysql_fetch_row(res))
            {
                User user;
                user.setId(atoi(row[0]));
                user.setName(row[1]);
                user.setState(row[2]);
                vec.push_back(user);
            }
            mysql_free_result(res); // 别忘了释放资源
        }
    }
    return vec;
}
```

```cpp
void ChatService::addFriend(const TcpConnectionPtr &conn, json &js, Timestamp time)
{
    uint32_t userid = js["userid"].get<uint32_t>();
    uint32_t friendid = js["friendid"].get<uint32_t>();
    // 存储好友信息
    if (userid != friendid)
    {
        friendModel_.insert(userid, friendid);
    }
}

```

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

            // 1. 登录成功需要记录该用户的连接信息
            {
                std::lock_guard<std::mutex> lock(connectionMutex);
                userConnectionMap_.insert({id, conn});
            }

            // 2. 登录成功需要更新用户状态信息
            user.setState("online");
            userModel_.updateState(user);

            // 3. 返回响应信息
            json response;
            response["msg_type"] = LOGIN_ACK;
            response["err_no"] = 0;
            response["id"] = user.getId();
            response["name"] = user.getName(); // 返回用户的信息

            // 4. 如有离线消息 在响应消息中捎带 立即推送
            std::vector<std::string> vecOffMsg = offlineMessageModel_.queryById(id);
            if (!vecOffMsg.empty())
            {
                response["offline_msg"] = vecOffMsg;
                // 读取完离线消息后 应该将离线消息表中的数据删除 否则下次登录还会推送
                offlineMessageModel_.remove(id);
            }

            // 5. 如果该用户好友列表不空 推送给他
            std::vector<User> vecFrd = friendModel_.queryById(id);
            if (!vecFrd.empty())
            {
                std::vector<json> frdList;
                frdList.reserve(vecFrd.size());
                for (auto frd : vecFrd)
                {
                    json fjs;
                    fjs["id"] = frd.getId();
                    fjs["name"] = frd.getName();
                    fjs["state"] = frd.getState();
                    frdList.push_back(fjs);
                }
                response["friend_list"] = frdList;
            }

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
