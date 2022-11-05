---
title: 【MySQL】11 - 索引
date: 2022-11-04
tags: [datebase, MySQL]
categories: [DateBase]
---


## MySQL 索引

当表中的数据量到达几十万甚至上百万的时候，SQL查询所花费的时间会很长，导致业务超时出错，此时就需要用索引来加速SQL查询。

索引解决的核心问题就是，**提高查询的速度**！

由于索引也是需要存储成索引文件的，因此对索引的使用也会涉及磁盘I/O操作。如果索引创建过多，使用不当，会造成SQL查询时，进行大量无用的磁盘I/O操作，降低了SQL的查询效率，适得其反，因此，掌握良好的索引创建原则非常重要！


## 索引分类

索引是创建在表上的，是对数据库表中一列或者多列的值进行排序的一种结果。

物理上，可以将索引分类为，**聚集型索引**和**非聚集型索引**。

逻辑上，可以将索引分类为：  
1. 普通索引（二级索引），没有任何限制条件，可以给任何类型的字段创建普通索引（创建新表&已创建的表），数量是不限的，一张表的一次sql查询只能用一个索引，这样的sql只能用一个索引`where a=1 and b='M'`)
2. 唯一性索引，使用`UNIQUE`修饰的字段，值不能够重复，主键索引就属于唯一性索引
3. 主键索引，使用`PRIMARY KEY`修饰的字段会自动创建索引(MyISAM, InnoDB)
4. 单列索引，在一个字段上创建索引
5. 多列索引，在表的多个字段上创建索引 (uid+cid，多列索引必须使用到**第一个列**，才能用到多列索引，否则索引用不上)
6. 全文索引，使用`FULLTEXT`参数可以设置全文索引，只支持`CHAR`，`VARCHAR`和`TEXT`类型的字段上，常用于数据量较大的字符串类型上，可以提高查询速度

实际上，全文索引在真实项目中使用的是较少的，线上项目支持专门的搜索功能，给后台服务器增加专门的搜索引擎，能够支持快速高效的搜索，如elasticsearch，简称es（JAVA），C++开源的搜索引擎，搜狗的workflow。



## 索引的基本使用

使用索引的几个注意点：  
1. 经常作为`WHERE`过滤条件的字段，要考虑添加索引；
2. 字符串列创建索引时，尽量规定索引的长度，不要让索引值的长度`key_len`过长；
3. 索引字段涉及类型强转、mysql函数调用、表达式计算等，就无法使用索引了。

### 索引的创建和删除

创建表时指定索引字段：  
```sql
CREATE TABLE test1(
    id INT,
    name VARCHAR(20),
    sex ENUM('male', 'female'),
    INDEX(id,name)
);
```
使用`PRIMARY KEY`或`UNIQUE`修饰的字段，会自动创建主键索引和唯一键索引。

为已经创建的表添加索引：  
```sql
CREATE [UNIQUE] INDEX 索引名 ON 表名 (属性名(length) [ASC | DESC]);
```

如果给字符串字段添加索引，最好使用`length`表明长度，只要长度可以区分不同字符串即可，使用整个字符串建索引，影响效率（每个索引越长，索引文件就越大，读索引的开销就越大）。

删除索引：  
```sql
DROP INDEX 索引名 ON 表名;
```


### 索引的执行计划

#### EXPLAIN 查看执行计划

我们可以使用`EXPLAIN`来查看sql的执行计划，分析索引的执行过程。

mysql的uer权限表，示例：  
```sql
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> EXPLAIN SELECT Host,User FROM user WHERE Host='%'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user
   partitions: NULL
         type: ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 180
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```
可以看到，使用了主键索引，共扫描了一行，`Using index`表示直接从索引树上查询到结果，无需回表。



#### EXPLAIN 结果字段说明

- `select_type`
  - `simple`：表示不需要`UNION`操作或者不包含子查询的简单`SELECT`语句。有连接查询时，外层的查询为`simple`且只有一个；
  - `primary`：一个需要`UNION`操作或者含有子查询的`SELECT`，位于最外层的单位查询的`select_type`即为`primary`且只有一个；
  - `union`：`UNION`连接的两个`SELECT`查询，出来第一个表外，第二个以后的表的`select_type`都是`union`；
  - `union result`：包含`UNION`的结果集，在`UNION`和`UNION ALL`语句中，因为它不需要参与查询，所以`id`字段为`null`。
- `table`
  - 显示查询的表名；
  - 如果不涉及对数据库的操作，这里显示`null`；
  - 如果显示为尖括号，就表示这是个临时表，后面的N就是执行计划中的id，表示结果来自于这个查询；
  - 如果尖括号括起来`<union M,N>`，也是一个临时表，表示这个结果来自于`union`查询的id为M，N的结果集。
- `type`
  - `const`：使用唯一索引或主键，返回记录一定是1行记录的等值`WHERE`条件时，通常`type`为`const`；
  - `ref`：常见于辅助索引的等值查找，或者多列主键、唯一索引中，使用第一个列之外的列作为等值查找会出现，返回数据不唯一的等值查找也会出现；
  - `range`：索引范围扫描，常用于使用`<`、`>`、`IS NULL`、`BETWEEN`、`IN`、`LIKE`等运算符的查询中；
  - `index`：索引全表扫描，把索引从头到尾扫一遍，常见于使用索引列就可以处理不需要读取文件的查询，可以使用索引排序或分组的查询
  - `all`：全表扫描数据文件，然后再server层进行过滤，返回符合要求的记录。
- `ref`
  - 如果使用常数等值查询，这里显示`const`；
  - 如果是连接查询，被驱动表的执行计划这里会显示驱动表的关联字段。
- `Extra`
  - `Using filesort`：排序时无法用到索引，常见于`ORDER BY`和`GROUP BY`语句中；
  - `Using index`：查询时不需要回表查询，直接通过索引就可以获取查询的数据。




### 测试

#### 测试1 - 有无索引进行WHERE过滤的效率

有索引字段搜索和没有索引的字段搜索：  
```sql
mysql> SHOW CREATE TABLE student\G
*************************** 1. row ***************************
       Table: student
Create Table: CREATE TABLE `student` (
  `uid` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL,
  `age` tinyint(3) unsigned NOT NULL,
  `sex` enum('m','w') NOT NULL,
  PRIMARY KEY (`uid`)
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

mysql> EXPLAIN SELECT * FROM student WHERE uid=3;
+----+-------------+---------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | student | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT * FROM student WHERE name='chenwei';
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | student | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    5 |    20.00 | Using where |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.01 sec)
```
有索引字段进行搜索，会直接使用索引，无索引字段搜索会进行整表搜索，效率很低。

给`name`字段添加索引：  
```sql
mysql> EXPLAIN SELECT * FROM student WHERE name='chenwei';
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | student | NULL       | ref  | nameidx       | nameidx | 152     | const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
添加索引后，进行搜索就会使用索引，而不会进行整表搜索。


> 注意，尽管某个字段使用了索引，但是以该字段过滤进行搜索，并不一定会用到索引。  
> 这是由于mysql server进行了优化，如果使用索引和不使用索引的花费大概相当，就不会使用索引，因为使用索引也会引入额外的开销（读索引文件、扫描索引树等等），还不如直接整表扫描。


#### 测试2 - 什么情况下不会使用索引

使用2000000行的`user_t`表进行测试。

```sql
mysql> SHOW CREATE TABLE user_t\G
*************************** 1. row ***************************
       Table: user_t
Create Table: CREATE TABLE `user_t` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `email` varchar(255) DEFAULT NULL,
  `passwor` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3000001 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

mysql> SELECT * FROM user_t WHERE passwor=1000000;
+---------+------------------+---------+
| id      | email            | passwor |
+---------+------------------+---------+
| 2000000 | 1000000@test.com | 1000000 |
+---------+------------------+---------+
1 row in set (0.30 sec)

mysql> EXPLAIN SELECT * FROM user_t WHERE passwor=1000000;
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | user_t | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1801249 |    10.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> CREATE INDEX pwdidx ON user_t(passwor);
Query OK, 0 rows affected (1.59 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> SELECT * FROM user_t WHERE passwor=1000000;
+---------+------------------+---------+
| id      | email            | passwor |
+---------+------------------+---------+
| 2000000 | 1000000@test.com | 1000000 |
+---------+------------------+---------+
1 row in set (0.31 sec)

mysql> EXPLAIN SELECT * FROM user_t WHERE passwor=1000000;
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | user_t | NULL       | ALL  | pwdidx        | NULL | NULL    | NULL | 1801249 |    10.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 3 warnings (0.00 sec)
```
在`user_t`表中使用没有索引的字段`passwor`作为过滤条件查询，用时0.3s，效率很低。  
给该字段加上索引，发现还是0.3s，用`EXPLAIN`查看执行计划，发现，`possible_keys`项中有`pwdidx`，但是`key`项中还是`NULL`，说明根本没用索引。

原因其实是因为，过滤字段`passwor`是字符串，而我们的sql中写成了int类型，mysql server要进行类型转换，这种情况下就不会使用索引了。

```sql
mysql> SELECT * FROM user_t WHERE passwor='1000000';
+---------+------------------+---------+
| id      | email            | passwor |
+---------+------------------+---------+
| 2000000 | 1000000@test.com | 1000000 |
+---------+------------------+---------+
1 row in set (0.00 sec)

mysql> EXPLAIN SELECT * FROM user_t WHERE passwor='1000000';
+----+-------------+--------+------------+------+---------------+--------+---------+-------+------+----------+-------+
| id | select_type | table  | partitions | type | possible_keys | key    | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+--------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | user_t | NULL       | ref  | pwdidx        | pwdidx | 768     | const |    1 |   100.00 | NULL  |
+----+-------------+--------+------------+------+---------------+--------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```