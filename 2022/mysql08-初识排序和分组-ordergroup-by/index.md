# 【MySQL】08 - 初识排序和分组 ORDER、GROUP BY


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

> 注意，`GROUP BY`一般用于统计，像`SELECT name FROM user GROUP BY sex;`这样的SQL没有意义。








