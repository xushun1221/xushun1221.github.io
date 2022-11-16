---
title: 【MySQL】32 - 分库分表
date: 2022-11-15
tags: [datebase, MySQL]
categories: [DateBase]
---


## 数据库架构演变

刚开始多数项目用单机数据库就够了，随着服务器流量越来越大，面对的请求也越来越多，我们做了数据库读写分离， 使用多个从库副本（Slave）负责读，使用主库（Master）负责写，master和slave通过主从复制实现数据同步更新，保持数据一致。slave 从库可以水平扩展，所以更多的读请求不成问题。

但是当用户量级上升，写请求越来越多，怎么保证数据库的负载足够？增加一个Master是不能解决问题的， 因为数据要保存一致性，写操作需要2个或多个master之间同步，相当于是重复了，而且架构设计更加复杂。

这时需要用到**分库分表（sharding）**，对写操作进行切分。



## 库表问题

单库太大，单库处理能力有限、所在服务器上的磁盘空间不足、遇到IO瓶颈，需要把单库切分成更多更小的库。

单表太大，CURD效率都很低、数据量太大导致索引膨胀、查询超时，需要把单表切分成多个数据集更小的表。


## 拆分策略

单个库太大，先考虑是表多还是数据多：  
- 如果因为表多而造成数据过多，则使用垂直拆分，即根据业务拆分成不同的库
- 如果因为单张表的数据量太大，则使用水平拆分，即把表的数据按照某种规则拆分成多张表

分库分表的原则应该是先考虑垂直拆分，再考虑水平拆分。


## 垂直拆分

### 垂直分表

也就是“大表拆小表”，基于列字段进行。

一般是表中的字段较多，将不常用的， 数据较大，长度较长（比如text类型字段）的拆分到“扩展表”。

一般是针对几百列的这种大表，也避免查询时，数据量太大造成的“跨页”问题。


### 垂直分库
垂直分库针对的是一个系统中的不同业务进行拆分。

比如用户User一个库，商品Product一个库，订单Order一个库， 切分后，要放在多个服务器上，而不是一个服务器上。想象一下，一个购物网站对外提供服务，会有用户，商品，订单等的CRUD。没拆分之前， 全部都是落到单一的库上的，这会让数据库的单库处理能力成为瓶颈。

按垂直分库后，如果还是放在一个数据库服务器上， 随着用户量增大，这会让单个数据库的处理能力成为瓶颈，还有单个服务器的磁盘空间，内存，tps等非常吃紧。 所以我们要拆分到多个服务器上，这样上面的问题都解决了，以后也不会面对单机资源问题。

数据库业务层面的拆分，和服务的“治理”，“降级”机制类似，也能对不同业务的数据分别的进行管理，维护，监控，扩展等。 数据库往往最容易成为应用系统的瓶颈，而数据库本身属于“有状态”的，相对于Web和应用服务器来讲，是比较难实现“横向扩展”的。 数据库的连接资源比较宝贵且单机处理能力也有限，在高并发场景下，垂直分库一定程度上能够突破IO、连接数及单机硬件资源的瓶颈。



### mycat 配置示例

`server.xml`  
```xml
<user name="root">
    <property name="password">666666</property>
    <property name="schemas">USERDB1,USERDB2</property>
</user>
```


`schema.xml`  
```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
    <!-- 逻辑数据库 -->
    <schema name="USERDB1" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1" />
    <schema name="USERDB2" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn2" />
    <!-- 存储节点 -->
    <dataNode name="dn1" dataHost="node1" database="mytest1" />
    <dataNode name="dn2" dataHost="node2" database="mytest2" />
    <!-- 数据库主机 -->
    <dataHost name="node1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="192.168.131.130" url="192.168.131.130:3306" user="root" password="666666">
        </writeHost>
    </dataHost>
    <dataHost name="node2" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="192.168.11.164" url="192.168.11.164:3306" user="root" password="666666" />
    </dataHost>
</mycat:schema>
```




## 水平拆分


### 水平分表

针对数据量巨大的单张表（比如订单表），按照某种规则（RANGE,HASH取模等），切分到多张表里面去。 但是这些表还是在同一个库中，所以库级别的数据库操作还是有IO瓶颈，不建议采用。

### 水平分库分表

将单张表的数据切分到多个服务器上去，每个服务器具有相应的库与表，只是表中数据集合不同。 水平分库分表能够有效的缓解单机和单库的性能瓶颈和压力，突破IO、连接数、硬件资源等的瓶颈。


### mycat 配置示例

`server.xml`  
```xml
<user name="root">
    <property name="password">666666</property>
    <property name="schemas">USERDB</property>
</user>
```

`schema.xml`  
```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
    <!-- 逻辑数据库 -->
    <schema name="USERDB" checkSQLschema="false" sqlMaxLimit="100">
        <table name="user" dataNode="dn1" />
        <table name="student" primaryKey="id" autoIncrement="true" dataNode="dn1,dn2" rule="mod-long"/>
    </schema>
    <!-- 存储节点 -->
    <dataNode name="dn1" dataHost="node1" database="mytest1" />
    <dataNode name="dn2" dataHost="node2" database="mytest2" />
    <!-- 数据库主机 -->
    <dataHost name="node1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="192.168.131.130" url="192.168.131.130:3306" user="root" password="123456">
        </writeHost>
    </dataHost>
    <dataHost name="node2" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="192.168.11.164" url="192.168.11.164:3306" user="root" password="123456" />
    </dataHost>
</mycat:schema>
```

`"mod-long"`在`rule.xml`中进行配置。