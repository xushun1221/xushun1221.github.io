---
title: 【MySQL】01 - 安装配置
date: 2022-10-11
tags: [datebase]
categories: [DateBase]
---

使用MySQL5.7

## 安装CentOS7

之前一直用Ubuntu，这次用CentOS试试。

下载好镜像文件`CentOS-7-x86_64-DVD-2009.iso`。

VMwareWorkstation一路安装就完事了。


## CentOS7安装MySQL

下载安装包：`[xushun@localhost Downloads]$ wget http://repo.mysql.com/mysql57-community-release-el7.rpm`

检查是否已安装：`[xushun@localhost Downloads]$ ll /ect/yum.repos.d/mysql*`

安装：`[xushun@localhost Downloads]$ sudo rpm -ivh mysql57-community-release-el7.rpm`

检查：  
```shell
[xushun@localhost Downloads]$ ll /etc/yum.repos.d/mysql*
-rw-r--r--. 1 root root 1838 Apr 27  2017 /etc/yum.repos.d/mysql-community.repo
-rw-r--r--. 1 root root 1885 Apr 27  2017 /etc/yum.repos.d/mysql-community-source.repo
```

安装MySQL Server：`sudo yum install mysql-server`

有可能会出现密钥错误，运行：`rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022`，再次安装即可。

启动服务：`sudo service mysqld start`

查看是否启动：`netstat -tanp`，看到3306端口的mysqld，成功启动。

如果需要重启：`service mysqld restart`

### 修改密码登录

安装mysql默认生成的密码在log日志文件中：  
```shell
[root@localhost Downloads]# grep "password" /var/log/mysqld.log
2022-10-12T03:26:31.666545Z 1 [Note] A temporary password is generated for root@localhost: -usC;&6qWQj:
```

通过上面的密码可以首次登陆到mysql server：  
```shell
[root@localhost Downloads]# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.40

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
mysql> 
```

必须按照提示修改密码并重新登录：  
```shell
mysql> set global validate_password_policy=0;
Query OK, 0 rows affected (0.00 sec)

mysql> set global validate_password_length=1;
Query OK, 0 rows affected (0.00 sec)

mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '666666';
Query OK, 0 rows affected (0.00 sec)

mysql> quit
Bye
[root@localhost Downloads]# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.40 MySQL Community Server (GPL)

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> 
```

### 配置文件

mysql的数据文件默认目录在：`/var/lib/mysql`  
```shell
[root@localhost mysql]# ls
auto.cnf    client-cert.pem  ibdata1      ibtmp1      mysql.sock.lock     public_key.pem   sys
ca-key.pem  client-key.pem   ib_logfile0  mysql       performance_schema  server-cert.pem
ca.pem      ib_buffer_pool   ib_logfile1  mysql.sock  private_key.pem     server-key.pem
```
数据库文件都在`/var/lib/mysql/mysql`目录下。


mysql的配置文件默认在：`/etc/my.cnf`。我们可以对其进行修改，更改配置信息，主要修改`[mysqld]`这一项。修改后需要重启服务。

我们添加一些常用配置：  
```
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

character-set-server=utf8

default-storage-engine=INNODB
```


### 设置ip远程访问

mysql默认只能通过localhost访问，不能通过ip远程访问，主要考虑安全问题。我们设置一下远程访问。

在配置文件中修改`bind-address=0.0.0.0`，重启mysqld服务。

可以看到，mysql默认只支持localhost访问：  
```
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select Host,User,authentication_string from user;
+-----------+---------------+-------------------------------------------+
| Host      | User          | authentication_string                     |
+-----------+---------------+-------------------------------------------+
| localhost | root          | *B2B366CA5C4697F31D4C55D61F0B17E70E5664EC |
| localhost | mysql.session | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE |
| localhost | mysql.sys     | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE |
+-----------+---------------+-------------------------------------------+
3 rows in set (0.00 sec)
```

设置支持远程访问：  
```
mysql> set global validate_password_policy=0;
Query OK, 0 rows affected (0.00 sec)

mysql> set global validate_password_length=1;
Query OK, 0 rows affected (0.00 sec)

mysql> grant all privileges on *.* to 'root'@'%' identified by '666666' with grant option;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> quit
Bye
```

通过ip远程登录，测试：  
```
[root@localhost ~]# mysql -h 192.168.20.131 -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.40 MySQL Community Server (GPL)

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> quit
Bye
```


## WIN10安装MySQL

这里安装MySQL5.7.17，打开`mysql-installer-community-5.7.17.0.msi`安装程序，一步一步走就完事了。

打开任务管理器->服务选项卡->看到MySQL57正在运行。

我们可以使用`MySQL 5.7 Command Line Client - Unicode`来使用MySQL。

### 配置文件

MySQL Server 5.7 默认的安装目录是`C:/Program Files/MySQL/MySQL Server 5.7`。

配置文件目录是`C:/ProgramData/MySQL/MySQL Server 5.7`。

在该目录下，有一个`my.ini`配置文件，在MySQL Server启动时需要加载该配置文件。我们可以修改该文件以更改MySQL Server的配置信息。主要看`[mysqld]`的内容。修改配置文件后需要重启服务（任务管理器）。

配置文件中有一行`datadir=C:/ProgramData/MySQL/MySQL Server 5.7\Data`，也就是该目录下的`Data`文件夹，我们的数据库信息都存放在这里。

在99行添加上utf8编码：  
```
# The default character set that will be used when a new schema or table is
# created and no character set is defined
character-set-server=utf8
```