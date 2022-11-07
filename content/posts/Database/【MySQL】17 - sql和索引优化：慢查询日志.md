---
title: 【MySQL】17 - sql和索引优化：慢查询日志
date: 2022-11-07
tags: [datebase, MySQL]
categories: [DateBase]
---

如何发现和分析sql和索引优化问题呢？

第一，可以用`EXPLAIN`语句分析一下需要优化的sql，能看出一些问题，比如是不是没有加索引、索引有没有被用到、应该加什么索引，等等。

但是，对于实际项目来说`EXPLAIN`分析可能还不够，因为项目中可能涉及非常多的业务，可能用到各种各样的sql，可能有几百上千，甚至上万条sql，很难用`EXPLAIN`一个一个分析。

所以，应该去分析项目工程中，哪些地方运行时间长、耗费大量性能，对于这些sql针对性的使用`EXPLAIN`去分析它们。

## 慢查询日志

问题来了，怎么找出耗时长、占用大量性能的sql呢？答案是，使用**慢查询日志**。


MySQL可以设置慢查询日志，当SQL执行的时间超过我们设定的时间，那么这些SQL就会被记录在慢查询日志当中，然后我们通过查看日志，用`explain`分析这些SQL的执行计划，来判定为什么效率低下，是没有使用到索引？还是索引本身创建的有问题？或者是索引使用到了，但是由于表的数据量太大，花费的时间就是很长，那么此时我们可以考虑把表分成n个小表，比如订单表按年份分成多个小表等。


慢查询日志相关参数如下：  
```sql
mysql> SHOW VARIABLES LIKE '%slow_query%';
+---------------------+-----------------------------------+
| Variable_name       | Value                             |
+---------------------+-----------------------------------+
| slow_query_log      | OFF                               |
| slow_query_log_file | /var/lib/mysql/localhost-slow.log |
+---------------------+-----------------------------------+
2 rows in set (0.00 sec)
```
慢查询日志默认关闭。

慢查询日志记录了包含所有执行时间超过参数 `long_query_time`（单位：秒）所设置值的 SQL语句的日志，在MySQL上用命令可以查看，如下：  
```sql
mysql> SHOW VARIABLES LIKE 'long%';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.00 sec)
```


开启慢查询日志：  
```sql
mysql> SET GLOBAL slow_query_log=ON;
Query OK, 0 rows affected (0.01 sec)
```

修改时间参数：  
```sql
mysql> SET long_query_time=0.1;
Query OK, 0 rows affected (0.00 sec)
```


在一个大表中查询，测试一下：  
```sql
mysql> show create table user_t\G
*************************** 1. row ***************************
       Table: user_t
Create Table: CREATE TABLE `user_t` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `email` varchar(255) DEFAULT NULL,
  `passwor` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3000001 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

mysql> SELECT * FROM user_t WHERE passwor='1500000';
+---------+------------------+---------+
| id      | email            | passwor |
+---------+------------------+---------+
| 2500000 | 1500000@test.com | 1500000 |
+---------+------------------+---------+
1 row in set (0.50 sec)
```

查询时间超过了0.1s，查看慢查询日志：  
```shell
[xushun@localhost ~]$ sudo more /var/lib/mysql/localhost-slow.log
/usr/sbin/mysqld, Version: 5.7.40 (MySQL Community Server (GPL)). started with:
Tcp port: 3306  Unix socket: /var/lib/mysql/mysql.sock
Time                 Id Command    Argument
# Time: 2022-11-07T07:51:59.556402Z
# User@Host: root[root] @ localhost []  Id:     2
# Query_time: 0.506998  Lock_time: 0.000096 Rows_sent: 1  Rows_examined: 2000000
use school;
SET timestamp=1667807519;
SELECT * FROM user_t WHERE passwor='1500000';
[xushun@localhost ~]$ 
```



## SHOW PROFILES

还可以通过`SHOW PROFILES;`查看sql执行的精确时间。

```sql
mysql> SHOW VARIABLES LIKE 'profiling';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| profiling     | OFF   |
+---------------+-------+
1 row in set (0.00 sec)

mysql> SET profiling=ON;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> select * from student where uid=2;
+-----+---------+-----+-----+
| uid | name    | age | sex |
+-----+---------+-----+-----+
|   2 | gaoyang |  20 | w   |
+-----+---------+-----+-----+
1 row in set (0.00 sec)

mysql> SHOW PROFILES;
+----------+------------+-----------------------------------+
| Query_ID | Duration   | Query                             |
+----------+------------+-----------------------------------+
|        1 | 0.00019875 | select * from student where uid=2 |
+----------+------------+-----------------------------------+
1 row in set, 1 warning (0.00 sec)
```