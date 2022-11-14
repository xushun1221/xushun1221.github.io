---
title: 【MySQL】28 - mysqldump数据备份
date: 2022-11-14
tags: [datebase, MySQL]
categories: [DateBase]
---

`mysqldump`也是mysql自带的一个数据备份工具。

使用方法：  
```sql
[root@localhost mysql]# mysql dump
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
[root@localhost mysql]# mysqldump
Usage: mysqldump [OPTIONS] database [tables]
OR     mysqldump [OPTIONS] --databases [OPTIONS] DB1 [DB2 DB3...]
OR     mysqldump [OPTIONS] --all-databases [OPTIONS]
For more options, use mysqldump --help
```

## 备份、恢复  测试

导出mytest中的uesr表：  
```shell
[root@localhost home]# mysqldump -u root -p mytest user > /home/user.sql
Enter password: 
[root@localhost home]#
```

把user表删除，再恢复`user.sql`中的数据，文件中所有的sql都重新执行了一遍，数据被恢复了：  
```sql
mysql> drop table user;
Query OK, 0 rows affected (0.00 sec)

mysql> source /home/user.sql
Query OK, 0 rows affected (0.01 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected, 1 warning (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 3 rows affected (0.00 sec)
Records: 3  Duplicates: 0  Warnings: 0

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected, 1 warning (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

```


## 格式化方式导出表

```shell
[xushun@localhost ~]$ mysql -u root -p -D school -e 'select * from user where age>18' > user_data.txt
Enter password: 
[xushun@localhost ~]$ more user_data.txt 
id	name	age	sex
7	zhangsan	22	m
8	gaoyang	22	w
9	chenwei	20	m
10	zhangfan	21	w
11	zhanglan	22	w
12	aaa	20	m
22	bbb	25	m
24	eee	25	w
[xushun@localhost ~]$ 
```
