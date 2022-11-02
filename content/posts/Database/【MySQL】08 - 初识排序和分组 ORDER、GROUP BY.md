---
title: 【MySQL】08 - 初识排序和分组 ORDER、GROUP BY
date: 2022-10-17
tags: [datebase, MySQL]
categories: [DateBase]
---

## ORDER BY 排序

使用方法：  
- `SELECT * FROM user ORDER BY name ASC;`，升序
- `SELECT * FROM user ORDER BY age DESC;`，降序
- `SELECT * FROM user ORDER BY name,age,sex;`，多个排序字段，前面的相同在按后一个字段排序，默认升序排列


### 性能分析

使用`ORDER BY`进行排序其实并不简单，它会涉及到很多问题。例如：  
```sql
mysql> EXPLAIN SELECT * FROM user ORDER BY age;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)
```
可以看到，使用`age`字段进行排序，需要进行整表的遍历，而且还需要使用`Using filesort`文件排序，这就涉及到额外的磁盘IO，效率很低。

出现这种情况，我们肯定需要进行优化。

再来看两个排序语句分析：  
```sql
mysql> EXPLAIN SELECT * FROM user ORDER BY name;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT name FROM user ORDER BY name;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | user  | NULL       | index | NULL          | name | 152     | NULL |    3 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
`name`字段是有索引的，当我们用`name`字段排序时，如果查询的字段为`*`，还需要文件排序，但是如果查询的字段仅为`name`，则不需要进行文件排序。


所以，`ORDER BY`的性能，不仅和排序的字段有关，也和查询的字段有关。这里需要知道聚集索引、非聚集索引、回表等相关知识。


## 面试小技巧

说说你使用MySQL中遇到的需要优化的问题。

> 答：  
> 我的xx项目中用到了`ORDER BY`，我发现数据量一多，它的性能就非常低，然后我就用`EXPLAIN`分析了一下这个SQL语句，发现它会用到`Using filesort`外排序（因为数据量比较大，无法全部放入内存，归并排序），涉及到很多磁盘IO，所以很慢。然后就要针对这个问题进行优化。  
> 首先需要对`ORDER BY`排序的字段添加索引，这样还不够，还需要考虑`SELECT`查询的列，这个涉及到SQL语句回表查询的问题……




## GROUP BY 分组

使用方法：  
- `SELECT sex FROM user GROUP BY sex;`，按`sex`字段分类
- `SELECT sex,count(id) FROM user GROUP BY sex;`，统计两种性别各有多少人
- `SELECT age,count(id) FROM user GROUP BY age HAVING age>25;`，统计大于25岁的，各年龄的人各有多少人
- `SELECT sex,AVG(age) FROM user GROUP BY sex;`，统计各性别的人的平均年龄
- `SELECT age,sex FROM user GROUP BY age,sex;`，按age和sex两个字段分组，age和sex完全相同的是一类
- `SELECT age,sex,COUNT(*) FROM user GROUP BY age,sex ORDER BY age DESC;`，分组完排序，降序，ASC升序

> 注意，`GROUP BY`一般用于统计，像`SELECT name FROM user GROUP BY sex;`这样的SQL没有意义。


### 性能分析

使用`GROUP BY`进行分组时是否存在性能问题呢？分析：  
```sql
mysql> EXPLAIN SELECT age FROM user GROUP BY age;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | Using temporary; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.00 sec)
```
`age`字段没有索引，对其进行分组，可以看到会使用临时表和外排序，`Using temporary`，`Using filesort`。

再用有索引的字段进行分组试一下：  
```sql
mysql> EXPLAIN SELECT name FROM user GROUP BY name;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | user  | NULL       | index | name          | name | 152     | NULL |    3 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
并没有使用临时表和外排序，而是直接使用索引。

使用索引效率会得到很大的提高，所以，使用`GROUP BY`分组的字段，需要加索引。



## 笔试题目

下表`bank_bill`是某银行代缴话费的主流水表结构:

|字段名|描述|
|---|---|
|serno|流水号|
|date|交易日期|
|accno|账号|
|name|姓名|
|amount|金额|
|brno|缴费网点|

1. 统计表中缴费的总笔数和总金额
    `SELECT COUNT(serno),SUM(amount) FROM bank_bill;`
2. 给出一个sql，按网点和日期统计每个网点每天的营业额，并按照营业额进行倒序排序
    `SELECT brno,date,SUM(amount) AS money FROM bank_bill GROUP BY brno,date ORDER BY brno,money DESC;`