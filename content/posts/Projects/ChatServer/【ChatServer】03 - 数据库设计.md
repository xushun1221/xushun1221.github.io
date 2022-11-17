---
title: 【ChatServer】03 - 数据库设计
date: 2022-11-17
tags: [MySQL]
categories: [Projects]
---

mysql-server的主机：CentOS7虚拟机（192.168.131.130）

库名：`chat`

## 表设计

### user表

|字段|类型|说明|约束|
|---|---|---|---|
|id|INT UNSIGNED|用户id|PRIMARY KEY, NOT NULL, AUTO_INCREMENT|
|name|VARCHAR(50)|用户名|NOT NULL, UNIQUE|
|password|VARCHAR(50)|用户密码|NOT NULL|
|state|ENUM('online','offline')|当前登录状态|DEFAULT 'offline'|

### friend表

|字段|类型|说明|约束|
|---|---|---|---|
|userid|INT UNSIGNED|用户id|NOT NULL, 联合主键|
|friendid|INT UNSIGNED|好友id|NOT NULL, 联合主键|

### allgroup表

|字段|类型|说明|约束|
|---|---|---|---|
|id|INT UNSIGNED|组id|PRIMARY KEY, NOT NULL, AUTO_INCREMENT|
|groupname|VARCHAR(50)|组名称|NOT NULL, UNIQUE|
|groupdesc|VARCHAR(200)|组描述|DEFAULT ''|

### groupuser表

|字段|类型|说明|约束|
|---|---|---|---|
|groupid|INT UNSIGNED|组id|NOT NULL, 联合主键|
|userid|INT UNSIGNED|组员id|NOT NULL, 联合主键|
|grouprole|ENUM('creator','normal')|组内角色|DEFAULT 'normal'|


### offlinemessage表

|字段|类型|说明|约束|
|---|---|---|---|
|userid|INT UNSIGNED|用户id|NOT NULL, PRIMARY KEY|
|message|VARCHAR(500)|离线消息（JSON）|NOT NULL|


### 脚本

```sql
CREATE TABLE user(
id INT UNSIGNED PRIMARY KEY NOT NULL AUTO_INCREMENT,
name VARCHAR(50) NOT NULL UNIQUE,
password VARCHAR(50) NOT NULL,
state ENUM('online','offline') DEFAULT 'offline')
ENGINE=INNODB,
DEFAULT CHARSET=UTF8;

CREATE TABLE friend(
userid INT UNSIGNED NOT NULL,
friendid INT UNSIGNED NOT NULL,
PRIMARY KEY(userid,friendid))
ENGINE=INNODB,
DEFAULT CHARSET=UTF8;

CREATE TABLE allgroup(
id INT UNSIGNED PRIMARY KEY NOT NULL AUTO_INCREMENT,
groupname VARCHAR(50) NOT NULL UNIQUE,
groupdesc VARCHAR(200) DEFAULT '')
ENGINE=INNODB,
DEFAULT CHARSET=UTF8;

CREATE TABLE groupuser(
groupid INT UNSIGNED NOT NULL,
userid INT UNSIGNED NOT NULL,
grouprole ENUM('creator','normal') DEFAULT 'normal',
PRIMARY KEY(groupid,userid))
ENGINE=INNODB,
DEFAULT CHARSET=UTF8;

CREATE TABLE offlinemessage(
userid INT UNSIGNED PRIMARY KEY NOT NULL,
message VARCHAR(500) NOT NULL)
ENGINE=INNODB,
DEFAULT CHARSET=UTF8;
```
