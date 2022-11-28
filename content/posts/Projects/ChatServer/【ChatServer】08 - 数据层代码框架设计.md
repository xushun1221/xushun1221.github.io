---
title: 【ChatServer】08 - 数据层代码框架设计
date: 2022-11-28
tags: [Linux, C++, MySQL]
categories: [Projects]
---


使用ORM类来映射数据库中的表。如，`User`类映射数据库中的`user`表，`UserModel`类来封装对user表的数据操作。

这样业务层的代码就不会直接对数据库进行操作，而是通过数据操作类`UserModel`和数据层进行交互，并且业务层操作的都是数据对象而不是数据库字段。

在业务层的`ChatService`类中添加一个`UserModel`对象即可。

```cpp
/*
 *  @Filename : User.hh
 *  @Description : ORM User
 *  @Datatime : 2022/11/28 14:30:40
 *  @Author : xushun
 */
#ifndef __USER_HH_
#define __USER_HH_

#include <string>

// user表的ORM类
class User
{
public:
    User(uint32_t id = -1, std::string name = "", std::string password = "", std::string state = "offline")
        : id_(id), name_(name), password_(password), state_(state)
    {
    }
    void setId(uint32_t id) { id_ = id; }
    void setName(std::string name) { name_ = name; }
    void setPassword(std::string password) { password_ = password; }
    void setState(std::string state) { state_ = state; }
    uint32_t getId() { return id_; }
    std::string getName() { return name_; }
    std::string getPassword() { return password_; }
    std::string getState() { return state_; }

private:
    uint32_t id_;
    std::string name_;
    std::string password_;
    std::string state_;
};

#endif // __USER_HH_
```

```cpp
/*
 *  @Filename : UserModel.hh
 *  @Description : UserModel Definition
 *  @Datatime : 2022/11/28 14:36:46
 *  @Author : xushun
 */
#ifndef __USERMODEL_HH_
#define __USERMODEL_HH_

#include "User.hh"

// user表的数据操作类
class UserModel
{
public:
    bool insert(User &user);
};

#endif // __USERMODEL_HH_
```


```cpp
/*
 *  @Filename : UserModel.cc
 *  @Description : Implementation of UserModel
 *  @Datatime : 2022/11/28 14:38:47
 *  @Author : xushun
 */
#include "UserModel.hh"
#include "db.hh"
#include <iostream>

// user表的插入方法
bool UserModel::insert(User &user)
{
    // 组装sql
    char sql[1024] = {0};
    sprintf(sql, "INSERT INTO user(name, password, state) VALUES('%s', '%s', '%s')",
            user.getName().c_str(), user.getPassword().c_str(), user.getState().c_str());
    // 连接mysqlserver并插入记录
    MySQL mysql;
    if (mysql.connect())
    {
        if (mysql.update(sql))
        {
            // 插入新记录并返回记录的主键ID
            user.setId(mysql_insert_id(mysql.getConnection()));
            return true;
        }
    }
    return false;
}

```