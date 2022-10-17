# 【MySQL】05 - 基础CRUD操作


## 结构化查询语句 SQL

**SQL**是结构化查询语言（Structure Query Language），它是关系型数据库的通用语言。

SQL主要可以划分为以下3个类别：  
- DDL（Data Definition Languages）语句
    数据定义语言，这些语句定义了不同的数据库、表、列、索引等数据库对象的定义。常用的语句关键字主要包括 create、drop、alter等。
- DML（Data Manipulation Language）语句
    数据操纵语句，用于添加、删除、更新和查询数据库记录，并检查数据完整性，常用的语句关键字主要包括 insert、delete、update 和select 等。
- DCL（Data Control Language）语句
    数据控制语句，用于控制不同的许可和访问级别的语句。这些语句定义了数据库、表、字段、用户的访问权限和安全级别。主要的语句关键字包括 grant、revoke 等。


## 库操作

- 查询数据库信息
  - `show databases;`
- 创建数据库
  - `create database testDB;`
- 删除数据库
  - `drop database testDB;`
- 选择数据库
  - `use testDB;`

对表进行操作前，需要选择一个数据库。


## 表操作

创建一个`school`数据库，测试用。

`use database school;`

- 查看表
  - `show tables;`
- 创建表（默认存储引擎和字符集可以在配置文件中定义）
  ```sql
  create table user(id int unsigned primary key not null auto_increment,
                    name varchar(50) not null unique,
                    age tinyint unsigned not null,
                    sex enum('m','w') not null) engine=INNODB default charset=utf8;
  ```
- 查看表结构
  - `desc user;`
- 查看建表SQL
  - `show create table user\G`
  - `show create table user;`
- 删除表
  - `drop table user;`

注意操作命令最好都大写，自己定义的都小写。



## 基础 CRUD 操作

> Create Retrieve Update Delete

- `INSERT`插入
  - 插入时应该在表名后注明添加的字段，否则需要添加全部字段
  - `INSERT INTO user(name, age, sex) values('zhangsan', 20, 'm');`
  - `INSERT INTO user(name, age, sex) values('lisi', 28, 'w');`
  - `INSERT INTO user(name, age, sex) values('wangwu', 30, 'm');`
  - 一次可以添加多行记录
  - `INSERT INTO user(name, age, sex) values('zhangsan', 20, 'm'),('lisi', 28, 'w'),('wangwu', 30, 'm');`
- `DELETE`删除数据
  - `DELETE FROM user`，删除所有数据
  - `DELETE FROM user WHERE name='lisi';`
  - `DELETE FROM user WHERE age BETWEEN 25 AND 40;`
- `UPDATE`更新数据
  - `UPDATE user SET age=23 WHERE name='zhangsan';`
  - `UPDATE user SET age=age+1;`，更新所有人的年龄
- `SELECT`查询数据
  - `SELECT * FROM user;`，最好不要用`*`，需要什么字段就写什么字段
  - `SELECT name FROM user;`
  - `SELECT name,sex FROM user;`
  - `SELECT id,name,age,sex FROM user WHERE sex='w' OR age<=30;`，使用`WHERE`进行过滤
  - `SELECT name,age,sex FROM user WHERE name LIKE 'zhang%';`，使用`LIKE`部分匹配


