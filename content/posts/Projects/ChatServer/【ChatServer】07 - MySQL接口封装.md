---
title: 【ChatServer】07 - MySQL接口封装
date: 2022-11-28
tags: [Linux, C++, MySQL]
categories: [Projects]
---

先检查一下mysql编程库有没有：  
```bash
[xushun@localhost ~]$ sudo find /usr -name libmysqlclient*
[sudo] password for xushun: 
/usr/lib64/mysql/libmysqlclient.so.20
/usr/lib64/mysql/libmysqlclient.so.20.3.27
/usr/lib64/mysql/libmysqlclient.a
/usr/lib64/mysql/libmysqlclient.so
```

没有的话安装一个mysql开发包。


## 封装mysql接口

mysql开发包提供的都是c的接口，我们要把它封装一下。

```cpp
/*
 *  @Filename : db.hh
 *  @Description : mysql interface
 *  @Datatime : 2022/11/28 13:19:27
 *  @Author : xushun
 */
#ifndef __DB_HH_
#define __DB_HH_

#include <mysql/mysql.h>
#include <string>

// 数据库操作类
class MySQL
{
public:
    // 初始化资源
    MySQL();
    // 释放资源
    ~MySQL();
    // 连接mysqlserver
    bool connect();
    // 更新操作
    bool update(std::string sql);
    // 查询操作
    MYSQL_RES *query(std::string sql);

private:
    MYSQL *conn_;
};

#endif // __DB_HH_
```


```cpp
/*
 *  @Filename : db.cc
 *  @Description : Implement of MySQL class
 *  @Datatime : 2022/11/28 13:37:11
 *  @Author : xushun
 */
#include "db.hh"
#include <muduo/base/Logging.h>

// 数据库配置信息
static std::string server = "127.0.0.1";
static std::string user = "root";
static std::string password = "666666";
static std::string dbname = "chat";

MySQL::MySQL()
{
    conn_ = mysql_init(nullptr);
}

MySQL::~MySQL()
{
    if (conn_ != nullptr)
    {
        mysql_close(conn_);
    }
}

bool MySQL::connect()
{
    MYSQL *p = mysql_real_connect(conn_, server.c_str(), user.c_str(), password.c_str(), dbname.c_str(), 3306, nullptr, 0);
    if (p != nullptr)
    {
        mysql_query(conn_, "set names gbk");
    }
    return p;
}

bool MySQL::update(std::string sql)
{
    if (mysql_query(conn_, sql.c_str()))
    {
        LOG_INFO << __FILE__ << ":" << __LINE__ << ":" << sql << " updata fail!";
        return false;
    }
    return true;
}

MYSQL_RES *MySQL::query(std::string sql)
{
    if (mysql_query(conn_, sql.c_str()))
    {
        LOG_INFO << __FILE__ << ":" << __LINE__ << ":" << sql << " query fail!";
        return nullptr;
    }
    return mysql_use_result(conn_);
}
```