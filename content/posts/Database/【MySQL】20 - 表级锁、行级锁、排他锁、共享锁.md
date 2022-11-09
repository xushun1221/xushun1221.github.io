---
title: 【MySQL】20 - 表级锁、行级锁、排他锁、共享锁
date: 2022-11-08
tags: [datebase, MySQL]
categories: [DateBase]
---


## 表级锁、行级锁（粒度）


- 表级锁：对整张表加锁。开销小，加锁快，不会出现死锁；锁粒度大，发生锁冲突的概率高，并发度低；
- 行级锁：对某行记录加锁。开销大，加锁慢，会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度高。

行级锁开销大、加锁慢的原因在于，表级锁只需直接锁住一张表即可，而行级锁需要找到对应的记录，需要搜索索引。


> MyISAM支持表级锁，InnoDB支持行级锁。

## 排他锁、共享锁

- 排它锁（Exclusive），又称为X锁，写锁；
- 共享锁（Shared），又称为S锁，读锁。

X和S锁之间有以下的关系： SS可以兼容的，XS、SX、XX之间是互斥的。

- 一个事务对数据对象 O 加了 S 锁，可以对 O 进行读取操作但不能进行更新操作。加锁期间其它事务能对O 加 S 锁但不能加 X 锁。
- 一个事务对数据对象 O 加了 X 锁，就可以对 O 进行读取和更新。加锁期间其它事务不能对 O 加任何锁。


显式加锁：`select ... lock in share mode` 强制获取共享锁，`select ... for update` 获取排它锁。



## InnoDB 行级锁

InnoDB存储引擎支持行级锁，并发能力很好。

InnoDB行级锁的特点：  
1. InnoDB行锁是通过给索引上的索引项加锁来实现的，而不是给表的行记录加锁实现的，这就意味着只有通过索引条件检索数据，InnoDB才使用行级锁，否则InnoDB将使用表锁。（过滤条件为二级索引的，同样会映射到主键索引树）
2. 由于InnoDB的行锁实现是针对索引字段添加的锁，不是针对行记录加的锁，因此虽然访问的是InnoDB引擎下表的不同字段，但是如果使用相同的索引字段作为过滤条件，依然会发生锁冲突，只能串行进行，不能并发进行。
3. 即使SQL中使用了索引，但是经过MySQL的优化器后，如果认为全表扫描比使用索引效率更高，此时会放弃使用索引，因此也不会使用行锁，而是使用表锁，比如对一些很小的表，MySQL就不会去使用索引。




## 测试示例

测试用的表：  
```sql
mysql> show create table user\G
*************************** 1. row ***************************
       Table: user
Create Table: CREATE TABLE `user` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL,
  `age` tinyint(3) unsigned NOT NULL,
  `sex` enum('m','w') NOT NULL,
  PRIMARY KEY (`id`),
  KEY `ageidx` (`age`)
) ENGINE=InnoDB AUTO_INCREMENT=13 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

mysql> select * from user;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  22 | m   |
|  8 | gaoyang  |  22 | w   |
|  9 | chenwei  |  20 | m   |
| 10 | zhangfan |  21 | w   |
| 11 | zhanglan |  22 | w   |
| 12 | aaa      |  20 | m   |
+----+----------+-----+-----+
6 rows in set (0.00 sec)
```


### 测试1  共享锁、排他锁、行锁

<html>
    <table style="margin: auto">
        <tr>
            <td>A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> set autocommit=0;
Query OK, 0 rows affected (0.01 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id=7 for update;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  22 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)

mysql> select * from user where id=8 for update;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
                </code></pre>
            </td>
            <td>B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> set autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id=7 for update;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> select * from user where id=7 lock in share mode;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> select * from user where id=8 lock in share mode;
+----+---------+-----+-----+
| id | name    | age | sex |
+----+---------+-----+-----+
|  8 | gaoyang |  22 | w   |
+----+---------+-----+-----+
1 row in set (0.00 sec)

mysql> select * from user where id=8 for update;
+----+---------+-----+-----+
| id | name    | age | sex |
+----+---------+-----+-----+
|  8 | gaoyang |  22 | w   |
+----+---------+-----+-----+
1 row in set (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

事务A给id=7的行显式加上了排他锁，事务B再对id=7这行加排他锁或共享锁都会阻塞，而事务B可以对id=8的行加共享锁或排他锁，这说明，InnoDB支持行级锁。



### 测试2  索引和行锁


<html>
    <table style="margin: auto">
        <tr>
            <td>A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where name='zhangsan' for update;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  22 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)
                </code></pre>
            </td>
            <td>B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where name='zhangsan' for update;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> select * from user where name='chenwei' for update;
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
                </code></pre>
            </td>
        </tr>
    </table>
</html>

事务A给`name=zhangsan`的行加上了排他锁，注意，`name`字段没有索引，此时，事务B既不能给`name=zhangsan`的行加锁，也不能给其他的行加锁，此时使用的是表锁。

> InnoDB的行锁是加在索引项上的，是给索引在枷锁，并不是给单纯的行记录枷锁。如果过滤条件没有用到索引的话，使用的就是表锁，而不是行锁。


### 测试3  串行化隔离级别


<html>
    <table style="margin: auto">
        <tr>
            <td><br>
                <!--左侧内容-->
                <pre><code>
mysql> set tx_isolation='SERIALIZABLE';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> select * from user;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  22 | m   |
|  8 | gaoyang  |  22 | w   |
|  9 | chenwei  |  20 | m   |
| 10 | zhangfan |  21 | w   |
| 11 | zhanglan |  22 | w   |
| 12 | aaa      |  20 | m   |
+----+----------+-----+-----+
6 rows in set (0.00 sec)
                </code></pre>
            </td>
            <td><br>
                <!--右侧内容-->
                <pre><code>
mysql> set tx_isolation='SERIALIZABLE';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> select * from user;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  22 | m   |
|  8 | gaoyang  |  22 | w   |
|  9 | chenwei  |  20 | m   |
| 10 | zhangfan |  21 | w   |
| 11 | zhanglan |  22 | w   |
| 12 | aaa      |  20 | m   |
+----+----------+-----+-----+
6 rows in set (0.00 sec)

mysql> insert into user(name,age,sex) values('bbb',25,'w');
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
                </code></pre>
            </td>
        </tr>
    </table>
</html>

在串行化的隔离级别下，mysql server完全使用共享锁和排他锁来管理并行操作。可以看到，左边终端查询了user表（加共享锁），右边的终端也可以查询user表（加共享锁），而右边的终端无法插入一行数据（加排他锁）。