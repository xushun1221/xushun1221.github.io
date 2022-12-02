---
title: 【ChatServer】14 - 群组业务
date: 2022-11-30
tags: [Linux, C++, MySQL]
categories: [Projects]
---

涉及群组的功能有三个，创建群、加入群、群聊。分别对应`CREATE_GROUP_MSG, JOIN_GROUP_MSG, GROUP_CHAT_MSG`三类消息。

涉及到两张表`allgroup, groupuser`，分别对应两个ORM类`Group, GroupUser`。

还有一个`GroupModel`数据操作类。

```cpp
/*
 *  @Filename : GroupModel.cc
 *  @Description : Implementation of GroupModel
 *  @Datatime : 2022/12/02 15:09:21
 *  @Author : xushun
 */

#include "GroupModel.hh"
#include "db.hh"

bool GroupModel::createGroup(Group &group)
{
    char sql[1024] = {0};
    sprintf(sql, "INSERT INTO allgroup(groupname, groupdesc) VALUES('%s', '%s')", group.getName().c_str(), group.getDesc().c_str());
    MySQL mysql;
    if (mysql.connect())
    {
        if (mysql.update(sql))
        {
            // 获取新群组的主键id
            group.setId(mysql_insert_id(mysql.getConnection()));
            return true;
        }
    }
    // 有可能创建失败 因为已经有相同的名字的群组了
    return false;
}

bool GroupModel::joinGroup(uint32_t userid, uint32_t groupid, std::string role)
{
    char sql[1024] = {0};
    sprintf(sql, "INSERT INTO groupuser VALUES(%ld, %ld, '%s')", groupid, userid, role.c_str());
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

std::vector<Group> GroupModel::queryGroupsById(uint32_t userid)
{
    // 查询用户所在的所有群组的详细信息
    std::vector<Group> vecGrp;
    char sql[1024] = {0};
    sprintf(sql, "SELECT g.id,g.groupname,g.groupdesc FROM allgroup g INNER JOIN groupuser u ON g.id=u.groupid WHERE u.userid=%ld", userid);
    MySQL mysql;
    if (mysql.connect())
    {
        MYSQL_RES *res = mysql.query(sql);
        if (res)
        {
            MYSQL_ROW row;
            while (row = mysql_fetch_row(res))
            {
                Group group;
                group.setId(atoi(row[0]));
                group.setName(row[1]);
                group.setDesc(row[2]);
                vecGrp.push_back(group);
            }
            mysql_free_result(res);
        }
    }
    // 查询群组中的用户信息 保存到Group对象中
    for (Group &group : vecGrp)
    {
        sprintf(sql, "SELECT a.id,a.name,a.state,b.grouprole From user a INNER JOIN groupuser b ON b.userid=a.id WHERE b.groupid=%ld", group.getId());
        MYSQL_RES *res = mysql.query(sql);
        if (res)
        {
            MYSQL_ROW row;
            while (row = mysql_fetch_row(res))
            {
                GroupUser user;
                user.setId(atoi(row[0]));
                user.setName(row[1]);
                user.setState(row[2]);
                user.setRole(row[3]);
                group.getUsers().push_back(user); // 群组用户信息保存在这
            }
            mysql_free_result(res);
        }
    }
    return vecGrp;
}

std::vector<uint32_t> GroupModel::queryGroupUsers(uint32_t userid, uint32_t groupid)
{
    std::vector<uint32_t> vecId;
    char sql[1024] = {0};
    sprintf(sql, "SELECT userid FROM groupuser WHERE groupid=%ld AND userid!=%ld", groupid, userid);
    MySQL mysql;
    if (mysql.connect())
    {
        MYSQL_RES *res = mysql.query(sql);
        if (res)
        {
            MYSQL_ROW row;
            while (row = mysql_fetch_row(res))
            {
                vecId.push_back(atoi(row[0]));
            }
            mysql_free_result(res);
        }
    }
    return vecId;
}
```


业务层代码。

```cpp
void ChatService::createGroup(const TcpConnectionPtr &conn, json &js, Timestamp time)
{
    uint32_t userid = js["userid"].get<uint32_t>();
    std::string groupname = js["groupname"];
    std::string groupdesc = js["groupdesc"];
    // 新的群组
    Group group(0, groupname, groupdesc);
    if (groupModel_.createGroup(group))
    {
        // 创建新的群组 首先要把创建人添加进群组
        groupModel_.joinGroup(userid, group.getId(), "creator");
    }
    else
    {
        // 如果群组名已经存在了 就无法创建
    }
}

void ChatService::joinGroup(const TcpConnectionPtr &conn, json &js, Timestamp time)
{
    uint32_t userid = js["userid"].get<uint32_t>();
    uint32_t groupid = js["groupid"].get<uint32_t>();
    groupModel_.joinGroup(userid, groupid, "normal");
}

void ChatService::groupChat(const TcpConnectionPtr &conn, json &js, Timestamp time)
{
    uint32_t userid = js["userid"].get<uint32_t>();
    uint32_t groupid = js["groupid"].get<uint32_t>();
    // 获得该群组中除自己的所有用户id
    std::vector<uint32_t> vecUsrId = groupModel_.queryGroupUsers(userid, groupid);
    std::lock_guard<std::mutex> lock(connectionMutex);
    {
        for (uint32_t id : vecUsrId)
        {
            auto itr = userConnectionMap_.find(id);
            if (itr != userConnectionMap_.end())
            {
                // 如果用户在线 直接发送
                itr->second->send(js.dump());
            }
            else
            {
                // 不在线 发送离线消息
                offlineMessageModel_.insert(id, js.dump());
            }
        }
    }
}
```