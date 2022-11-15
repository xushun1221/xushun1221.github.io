---
title: 【MySQL】31 - 读写分离
date: 2022-11-15
tags: [datebase, MySQL]
categories: [DateBase]
---


## 读写分离  原理

读写分离就是在主服务器上修改，数据会同步到从服务器，从服务器只能提供读取数据，不能写入，实现备份的同时也实现了数据库性能的优化，以及提升了服务器安全。单机服务器性能瓶颈后可以采用这种方法。


![](/post_images/posts/Database/MySQL/读写分离.jpg "读写分离")



较为常见的MySQL读写分离的方式有两种：  
1. 在程序代码中实现，这种方式有很明显的缺点，程序代码和服务器环境强相关；
2. 使用中间代理层，如Mycat等中间件，这也是主流做法。


客户端不会直接向主、从服务器请求服务，而是将sql请求发送至代理服务器（Mycat中间件），读写分离的配置在代理服务器中进行，由代理服务器对客户请求进行转发，写操作转发给主服务器，读操作转发给从服务器。主服务器和从服务器之间使用主从复制进行数据同步。


对于开发而言，引入了mycat中间件之后，我们只需要关注mycat，由它作为代理，只需要操作逻辑库表，逻辑库表映射到哪一个主从库上，这些都由mycat来负责。


Mycat中支持多种配置方式，包括：一主一从、一主多从、多主多从等等，支持多种功能，包括容灾等。

Mycat的数据端口为`8066`，客户端向该端口发送请求，代理服务器在转发到主、从服务器的`3306`端口；  
Mycat的管理端口为`9066`，用于管理代理服务器的工作状态。



## 读写分离  实践配置

读写分离显然需要先开启主从复制。

- master主服务器：CentOS7 虚拟机（192.168.131.130）
- slave从服务器： Win10 主机（本机IP：192.168.11.164，虚拟网关：192.168.131.1）
- Mycat代理服务器：CentOS7 虚拟机（192.168.131.130）

### Mycat 读写分离配置

要求：  
1. Mycat需要JDK1.7以上的版本（`java -versjion`检查jdk环境）
2. MySQL的root用户需要远程访问权限（主从都要）


#### 安装Mycat

1. 下载解压`Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz`文件
2. 解压后的文件放在`/usr/local/mycat`目录下
3. `./mycat start`启动服务，通过8066和9066端口使用（先不要急着启动）

```shell
[xushun@localhost mycat]$ pwd
/usr/local/mycat
[xushun@localhost mycat]$ ls
bin  catlet  conf  lib  logs  version.txt
```
`bin`中是可执行文件，`conf`中是配置文件，`logs`中是日志文件。


为了方便使用，在`/usr/bin`中建一个软连接，可以直接`mycat`执行了：  
```shell
[root@localhost bin]# pwd
/usr/bin
[root@localhost bin]# ln -s /usr/local/mycat/bin/mycat .
[root@localhost bin]# 
```


#### 配置文件

`server.xml`：配置登录Mycat的账号信息。

把用户相关的删掉，留这个  
```xml
<user name="root">
	<property name="password">666666</property>
	<property name="schemas">USERDB</property>
</user>
```
这里的`schemas`标签中的`USERDB`是逻辑库，看上去在Mycat一台服务器上，实际上它可能映射到mysql服务器上的某个库，或者进行了分库，映射到不同的机器上，或者库里的表也映射到不同的机器上。

有多个逻辑库的话，就用逗号隔开。


`schema.xml`：配置逻辑库和数据源、读写分离、分库分表信息等。

这个文件改为：  
```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
    <!-- 逻辑数据库 -->
	<schema name="USERDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1"></schema>
	<!-- 存储节点 -->
    <dataNode name="dn1" dataHost="node1" database="mytest" />
	<!-- 数据库主机 -->
    <dataHost name="node1" maxCon="1000" minCon="10" balance="3" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
        <!-- 心跳：检测mysql服务器是否存活，发送select user();查询 -->
		<heartbeat>select user()</heartbeat>
        <!-- 主 -->
		<writeHost host="192.168.131.130" url="192.168.131.130:3306" user="root" password="666666">
            <!-- 从 -->
			<readHost host="192.168.11.164" url="192.168.11.164:3306" user="root" password="666666" />
		</writeHost>
        <!-- 主库宕机后启用备份数据库 -->
		<writeHost host="192.168.11.164" url="192.168.11.164:3306" user="root" password="666666" />
	</dataHost>
</mycat:schema>
```

注意，VMware的虚拟网关是虚拟机接收win10向虚拟机发送数据时使用的，虚拟机向win10发送数据，直接使用本机ip即可，win10向虚拟机发送数据也应该使用虚拟机ip而不是虚拟网关。



重要参数说明：

maxCon、minCon：最大最小连接数量（mycat使用连接池和mysql服务器通信）


balance：  
- "0"：不开启读写分离
- "1"：全部的readHost和stand by writeHost参与select语句的负载
- "2"：所有读操作随机在readHost和writeHost上分发
- "3"：所有读请求随机分发到writeHost对应的readHost上执行
- 
writeType="0"：所有写操作发送到配置的第一个writeHost，第一个挂掉切换到还生存的第二个writeHost

switchType：  
- "-1"：不自动切换
- "1"：自动切换，根据心跳select user()
- "2"：基于MySQL的主从同步状态决定是否进行切换 show slave status





#### 启动 Mycat

```shell
[xushun@localhost ~]$ mycat start
Starting Mycat-server...
[xushun@localhost ~]$ netstat -tanp | grep 66
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp6       0      0 :::8066                 :::*                    LISTEN      18907/java          
tcp6       0      0 :::9066                 :::*                    LISTEN      18907/java          
[xushun@localhost ~]$ 
```


#### 查看日志

日志目录：`/usr/local/mycat/logs/`

`mycat.log`是运行日志，`wrapper.log`是启动日志。

查看启动日志最后的几十行，可以看到启动成功了：  
```shell
[xushun@localhost logs]$ tail wrapper.log 
INFO   | jvm 1    | 2022/11/15 03:50:36 | 2022-11-15 03:50:36,093 [INFO ][$_NIOREACTOR-2-RW] connected successfuly MySQLConnection [id=10, lastTime=1668513036093, user=root, schema=mytest, old shema=mytest, borrowed=true, fromSlaveDB=false, threadId=69, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.131.130, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.mysql.nio.handler.GetConnectionHandler:GetConnectionHandler.java:67) 
INFO   | jvm 1    | 2022/11/15 03:50:36 | 2022-11-15 03:50:36,187 [INFO ][WrapperSimpleAppMain] init result :finished 10 success 10 target count:10  (io.mycat.backend.datasource.PhysicalDBPool:PhysicalDBPool.java:319) 
INFO   | jvm 1    | 2022/11/15 03:50:36 | 2022-11-15 03:50:36,187 [INFO ][WrapperSimpleAppMain] node1 index:0 init success  (io.mycat.backend.datasource.PhysicalDBPool:PhysicalDBPool.java:265) 
INFO   | jvm 1    | 2022/11/15 03:50:36 | 2022-11-15 03:50:36,189 [INFO ][Timer0] create connections ,because idle connection not enough ,cur is 0, minCon is 10 for 192.168.11.164  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:299) 
INFO   | jvm 1    | 2022/11/15 03:50:36 | 2022-11-15 03:50:36,193 [INFO ][Timer1] no ilde connection in pool,create new connection for 192.168.11.164 of schema mytest  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:413) 
INFO   | jvm 1    | 2022/11/15 03:50:36 | 2022-11-15 03:50:36,193 [INFO ][Timer1] no ilde connection in pool,create new connection for 192.168.11.164 of schema mytest  (io.mycat.backend.datasource.PhysicalDatasource:PhysicalDatasource.java:413) 
INFO   | jvm 1    | 2022/11/15 03:50:36 | 2022-11-15 03:50:36,195 [INFO ][$_NIOREACTOR-4-RW] connectionAcquired MySQLConnection [id=12, lastTime=1668513036187, user=root, schema=mytest, old shema=mytest, borrowed=false, fromSlaveDB=true, threadId=612, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.11.164, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.mysql.nio.handler.NewConnectionRespHandler:NewConnectionRespHandler.java:45) 
INFO   | jvm 1    | 2022/11/15 03:50:36 | 2022-11-15 03:50:36,196 [INFO ][$_NIOREACTOR-5-RW] connectionAcquired MySQLConnection [id=13, lastTime=1668513036187, user=root, schema=mytest, old shema=mytest, borrowed=false, fromSlaveDB=true, threadId=613, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.11.164, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.mysql.nio.handler.NewConnectionRespHandler:NewConnectionRespHandler.java:45) 
INFO   | jvm 1    | 2022/11/15 03:50:36 | 2022-11-15 03:50:36,197 [INFO ][$_NIOREACTOR-3-RW] connectionAcquired MySQLConnection [id=11, lastTime=1668513036187, user=root, schema=mytest, old shema=mytest, borrowed=false, fromSlaveDB=true, threadId=611, charset=utf8, txIsolation=3, autocommit=true, attachment=null, respHandler=null, host=192.168.11.164, port=3306, statusSync=null, writeQueue=0, modifiedSQLExecuted=false]  (io.mycat.backend.mysql.nio.handler.NewConnectionRespHandler:NewConnectionRespHandler.java:45) 
INFO   | jvm 1    | 2022/11/15 03:50:36 | MyCAT Server startup successfully. see logs in logs/mycat.log
[xushun@localhost logs]$ 
```


### 使用 mycat

在win10上登陆需要和之前一样开放虚拟机centos的8066和9066防火墙。


#### 管理端口

使用9066端口登录mycat，进入一个mysql shell，但是它是用来管理mycat的，不进行sql操作：  
```shell
PS C:\Users\xushun> mysql -u root -p -h 192.168.131.130 -P 9066
Enter password: ******
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 5.6.29-mycat-1.6-RELEASE-20161028204710 MyCat Server (monitor)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
注意，`(monitor)`表示这是一个管理端口，进行监控操作。


查看支持的命令：  
```shell
mysql> show @@help;
+------------------------------------------+--------------------------------------------+
| STATEMENT                                | DESCRIPTION                                |
+------------------------------------------+--------------------------------------------+
| show @@time.current                      | Report current timestamp                   |
| show @@time.startup                      | Report startup timestamp                   |
| show @@version                           | Report Mycat Server version                |
| show @@server                            | Report server status                       |
| show @@threadpool                        | Report threadPool status                   |
| show @@database                          | Report databases                           |
| show @@datanode                          | Report dataNodes                           |
| show @@datanode where schema = ?         | Report dataNodes                           |
| show @@datasource                        | Report dataSources                         |
| show @@datasource where dataNode = ?     | Report dataSources                         |
| show @@datasource.synstatus              | Report datasource data synchronous         |
| show @@datasource.syndetail where name=? | Report datasource data synchronous detail  |
| show @@datasource.cluster                | Report datasource galary cluster variables |
| show @@processor                         | Report processor status                    |
| show @@command                           | Report commands status                     |
| show @@connection                        | Report connection status                   |
| show @@cache                             | Report system cache usage                  |
... 还有很多
```


#### 数据端口

使用8066端口登录mycat，进入一个mysql shell，可以进行所有的sql操作，就和直接操作mysql数据库一样。（当然使用的是逻辑库表）  
```shell
PS C:\Users\xushun> mysql -u root -p -h 192.168.131.130 -P 8066
Enter password: ******
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 5.6.29-mycat-1.6-RELEASE-20161028204710 MyCat Server (OpenCloundDB)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
注意，`(OpenCloundDB)`表示这是数据库，mycat像一朵云一样把底层的mysql数据库给隐藏了起来。



#### 验证mycat的使用

对于不同的操作，写操作应该在centos7虚拟机上的mysql服务器执行，而读操作应该在win10上的mysql服务器执行，如何验证呢？

答：使用查询日志。


