# 【MySQL】09 - 连接查询详解


## 连接查询

连接查询就是在一次sql查询中查询多张相关联的表。

为什么不使用多个sql进行查询？因为效率低，当mysql client每次发送sql查询时，mysql server都需要执行很多的校验操作，执行一次完整的处理sql查询的流程，而且还有连接建立和销毁的花销。所以多次查询会占用很多额外的开销，连接查询效率更高。

连接查询的分类：
- 内连接查询 inner join
  - 查询表的交集
- 外连接查询
  - 左连接查询 left join
    - 查询表1特有的数据
  - 右连接查询 right join
    - 查询表2特有的数据




用几个场景来练习一下。

## 内连接查询

`SELECT a.属性名1,a.属性名2,...,b.属性名1,b.属性名2,... FROM table_name1 a INNER JOIN table_name2 b ON a.id=b.id WHERE a.属性名 满足xx条件;`

涉及多表连接查询时，应该使用别名。

### 场景： 学生 课程 考试成绩

- 学生表student：`(uid,name,age,sex)`
- 课程表course：`(cid,cname,credit)`
- 考试exame：`(uid,cid,time,score)`

建表：  
```sql
CREATE TABLE student(
    uid INT UNSIGNED PRIMARY KEY NOT NULL AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    age TINYINT UNSIGNED NOT NULL,
    sex ENUM('m', 'w') NOT NULL
);

CREATE TABLE course(
    cid INT UNSIGNED PRIMARY KEY NOT NULL AUTO_INCREMENT,
    cname VARCHAR(50) NOT NULL,
    credit TINYINT UNSIGNED NOT NULL
);

CREATE TABLE exame(
    uid INT UNSIGNED NOT NULL,
    cid INT UNSIGNED NOT NULL,
    time DATE NOT NULL,
    score FLOAT NOT NULL,
    PRIMARY KEY(uid, cid)
);
```

随便弄点数据：  
```sql
INSERT INTO student(name,age,sex) VALUES
('zhangsan',18,'m'),
('gaoyang',20,'w'),
('chenwei',22,'m'),
('linfeng',21,'w'),
('liuxiang',19,'w');

INSERT INTO course(cname,credit) VALUES
('C++基础',5),
('C++高级',10),
('C++项目',8),
('C++算法',12);

INSERT INTO exame(uid,cid,time,score) VALUES
(1,1,'2021-04-09',99.0),
(1,2,'2021-04-09',80.0),
(2,2,'2021-04-09',90.0),
(2,3,'2021-04-09',85.0),
(3,1,'2021-04-09',56.0),
(3,2,'2021-04-09',93.0),
(3,3,'2021-04-09',89.0),
(3,4,'2021-04-09',100.0),
(4,4,'2021-04-09',99.0),
(5,2,'2021-04-09',59.0),
(5,3,'2021-04-09',94.0),
(5,4,'2021-04-09',95.0);
```

#### 问题1

已知张三同学学号`uid=1`，C++高级课程`cid=2`，查询考试成绩。

```sql
mysql> SELECT score FROM exame WHERE uid=1 AND cid=2;
+-------+
| score |
+-------+
|    80 |
+-------+
1 row in set (0.00 sec)
```

#### 问题2

查询该成绩同时返回学生的详细信息。

如果不熟悉内连接查询，应该先分别写出两个sql。  
`SELECT a.uid,a.name,a.age,a.sex FROM student a WHERE a.uid=1;`  
`SELECT c.score FROM exame c WHERE c.uid=1 AND c.cid=2;`

再写成内连接查询的形式：  
```sql
mysql> SELECT a.uid,a.name,a.age,a.sex,c.score FROM student a INNER JOIN exame c ON a.uid=c.uid WHERE c.uid=1 AND c.cid=2;
+-----+----------+-----+-----+-------+
| uid | name     | age | sex | score |
+-----+----------+-----+-----+-------+
|   1 | zhangsan |  18 | m   |    80 |
+-----+----------+-----+-----+-------+
1 row in set (0.01 sec)
```

`ON`连接条件为关联的键

> 连接过程：  
> 连接中会区分**大表**和**小表**，按照数据量来区分，**小表永远是整表扫描，然后去大表搜索**  
> 这个sql查询，首先从student（小表）中取出所有的uid，然后用这些uid在exame（大表）中搜索  
> 搜索结果需要进行过滤，对于INNER JOIN内连接，过滤条件写在WHERE后面和ON连接条件中，效果是一样的，（例如`SELECT a.*,b.* FROM student a INNER JOIN exame b on a.uid=b.uid AND b.cid=3;`和`SELECT a.*,b.* FROM student a INNER JOIN exame b on a.uid=b.uid WHERE b.cid=3;`，结果是相同的，是mysql server进行了优化）  
> 如果大表在WHERE过滤后，记录数量小于小表了，那么大表就成为小表，也就是说，WHERE过滤是在搜索之前的。


#### 问题3

现在不仅要知道学生的详细信息和考试成绩，还要知道课程的详细信息。

先写查询课程信息的sql：  
`SELECT b.cid,b.cname,b.credit FROM course b WHERE b.cid=2;`

把它加到上面写的内连接查询中：  
```sql
mysql> SELECT a.uid,a.name,a.age,a.sex,b.cid,b.cname,b.credit,c.score FROM exame c
    -> INNER JOIN student a ON a.uid=c.uid
    -> INNER JOIN course b ON b.cid=c.cid
    -> WHERE c.uid=1 AND c.cid=2;
+-----+----------+-----+-----+-----+-----------+--------+-------+
| uid | name     | age | sex | cid | cname     | credit | score |
+-----+----------+-----+-----+-----+-----------+--------+-------+
|   1 | zhangsan |  18 | m   |   2 | C++高级   |     10 |    80 |
+-----+----------+-----+-----+-----+-----------+--------+-------+
1 row in set (0.00 sec)
```
连接三张表时，要注意顺序，在这里`exame c`要和两张表进行连接，所以在最前。

#### 问题4

统计每个课程有多少人考试，同时输出课程详细信息。

`SELECT b.cid,b.cname,b.credit FROM course b;`  
`SELECT c.cid,count(*) FROM exame c GROUP BY c.cid;`

```sql
mysql> SELECT b.cid,b.cname,b.credit,count(*) 
    -> FROM exame c
    -> INNER JOIN course b ON c.cid=b.cid
    -> GROUP BY c.cid
    -> ;
+-----+-----------+--------+----------+
| cid | cname     | credit | count(*) |
+-----+-----------+--------+----------+
|   1 | C++基础   |      5 |        2 |
|   2 | C++高级   |     10 |        4 |
|   3 | C++项目   |      8 |        3 |
|   4 | C++算法   |     12 |        3 |
+-----+-----------+--------+----------+
4 rows in set (0.00 sec)
```

还可以添加一些过滤条件：(各科大于90分的各有多少人)  
```sql
mysql> SELECT b.cid,b.cname,b.credit,count(*)
    -> FROM exame c
    -> INNER JOIN course b ON c.cid=b.cid
    -> WHERE c.score>=90.0
    -> GROUP BY c.cid;
+-----+-----------+--------+----------+
| cid | cname     | credit | count(*) |
+-----+-----------+--------+----------+
|   1 | C++基础   |      5 |        1 |
|   2 | C++高级   |     10 |        2 |
|   3 | C++项目   |      8 |        1 |
|   4 | C++算法   |     12 |        3 |
+-----+-----------+--------+----------+
4 rows in set (0.00 sec)
```

`GROUP BY`分组后使用`HAVING`过滤：  
```sql
mysql> SELECT b.cid,b.cname,b.credit,count(*)
    -> FROM exame c
    -> INNER JOIN course b ON c.cid=b.cid
    -> WHERE c.score>=90.0
    -> GROUP BY c.cid
    -> HAVING c.cid=2 OR c.cid=3;
+-----+-----------+--------+----------+
| cid | cname     | credit | count(*) |
+-----+-----------+--------+----------+
|   2 | C++高级   |     10 |        2 |
|   3 | C++项目   |      8 |        1 |
+-----+-----------+--------+----------+
2 rows in set (0.00 sec)
```

按参加考试的人数对课程进行降序排序：  
```sql
mysql> SELECT b.cid,b.cname,b.credit,count(*) cnt
    -> FROM exame c
    -> INNER JOIN course b ON c.cid=b.cid
    -> GROUP BY c.cid
    -> ORDER BY cnt DESC;
+-----+-----------+--------+-----+
| cid | cname     | credit | cnt |
+-----+-----------+--------+-----+
|   2 | C++高级   |     10 |   4 |
|   3 | C++项目   |      8 |   3 |
|   4 | C++算法   |     12 |   3 |
|   1 | C++基础   |      5 |   2 |
+-----+-----------+--------+-----+
4 rows in set (0.00 sec)
```

#### 问题5

cid=2这门课考试成绩最高的学生信息和这门课的信息。

```sql
mysql> SELECT a.uid,a.name,b.cid,b.cname,b.credit,c.score 
    -> FROM exame c
    -> INNER JOIN student a ON c.uid=a.uid
    -> INNER JOIN course b ON c.cid=b.cid
    -> WHERE c.cid=2
    -> ORDER BY c.score DESC
    -> LIMIT 1;
+-----+---------+-----+-----------+--------+-------+
| uid | name    | cid | cname     | credit | score |
+-----+---------+-----+-----------+--------+-------+
|   3 | chenwei |   2 | C++高级   |     10 |    93 |
+-----+---------+-----+-----------+--------+-------+
1 row in set (0.00 sec)
```
自己写的可能效率不高。



#### 问题6

每门课的考试平均成绩和课程信息。

```sql
mysql> SELECT b.cid,b.cname,b.credit,avg(c.score)
    -> FROM exame c
    -> INNER JOIN course b ON b.cid=c.cid
    -> GROUP BY c.cid;
+-----+-----------+--------+-------------------+
| cid | cname     | credit | avg(c.score)      |
+-----+-----------+--------+-------------------+
|   1 | C++基础   |      5 |              77.5 |
|   2 | C++高级   |     10 |              80.5 |
|   3 | C++项目   |      8 | 89.33333333333333 |
|   4 | C++算法   |     12 |                98 |
+-----+-----------+--------+-------------------+
4 rows in set (0.01 sec)
```



### 内连接对LIMIT的优化

带有偏移的LIMIT的查询效率和查询的字段是有关系的，例如：  
```sql
mysql> SELECT COUNT(*) FROM user_t;
+----------+
| COUNT(*) |
+----------+
|  2000000 |
+----------+
1 row in set (0.16 sec)

mysql> SELECT * FROM user_t LIMIT 1500000,10;
+---------+------------------+---------+
| id      | email            | passwor |
+---------+------------------+---------+
| 2500001 | 1500001@test.com | 1500001 |
| 2500002 | 1500002@test.com | 1500002 |
| 2500003 | 1500003@test.com | 1500003 |
| 2500004 | 1500004@test.com | 1500004 |
| 2500005 | 1500005@test.com | 1500005 |
| 2500006 | 1500006@test.com | 1500006 |
| 2500007 | 1500007@test.com | 1500007 |
| 2500008 | 1500008@test.com | 1500008 |
| 2500009 | 1500009@test.com | 1500009 |
| 2500010 | 1500010@test.com | 1500010 |
+---------+------------------+---------+
10 rows in set (0.18 sec)

mysql> SELECT id FROM user_t LIMIT 1500000,10;
+---------+
| id      |
+---------+
| 2500001 |
| 2500002 |
| 2500003 |
| 2500004 |
| 2500005 |
| 2500006 |
| 2500007 |
| 2500008 |
| 2500009 |
| 2500010 |
+---------+
10 rows in set (0.12 sec)
```
查询所有字段用时0.18s，而仅查询id字段则用时0.12s。

我们可以使用`WHERE`对id字段（有索引）进行过滤从而避免`LIMIT`偏移产生的花费。

如果我们不能使用`WHERE`进行过滤，并且要查询所有的字段，而且要求查询效率和仅查询id字段相同，如何做到？

使用内连接进行优化：  
```sql
mysql> SELECT a.id,a.email,a.passwor 
    -> FROM user_t a
    -> INNER JOIN (
    -> SELECT id FROM user_t
    -> LIMIT 1500000,10
    -> ) b ON a.id=b.id;
+---------+------------------+---------+
| id      | email            | passwor |
+---------+------------------+---------+
| 2500001 | 1500001@test.com | 1500001 |
| 2500002 | 1500002@test.com | 1500002 |
| 2500003 | 1500003@test.com | 1500003 |
| 2500004 | 1500004@test.com | 1500004 |
| 2500005 | 1500005@test.com | 1500005 |
| 2500006 | 1500006@test.com | 1500006 |
| 2500007 | 1500007@test.com | 1500007 |
| 2500008 | 1500008@test.com | 1500008 |
| 2500009 | 1500009@test.com | 1500009 |
| 2500010 | 1500010@test.com | 1500010 |
+---------+------------------+---------+
10 rows in set (0.12 sec)
```
结果仅使用0.12s。

如上所示，我们先查询我们想要的记录的id字段（带索引），然后用内连接INNER JOIN，连接这张临时表和user_t原表，很明显临时表是小表，所以会使用id字段在大表（原表）中进行搜索，因为id字段带索引，所以仅用常数时间，查询到其他的字段。

这样，使用INNER JOIN可以对使用LIMIT的查询进行优化，查询字段更多的同时，耗时也没有增加。







## 外连接查询

我们还是用学生考试的场景来测试。

在`student`表中添加一个学生`mysql> INSERT INTO student(name,age,sex) VALUES ('weiwei',20,'m');`。


### 左连接 LEFT JOIN

`SELECT a.属性名列表,b.属性名列表 FROM table_name1 a LEFT [OUTER] JOIN table_name2 b ON a.id=b.id;`  
整表扫描左表`a`中数据，在右表中搜索，在右表中不存在相应的数据则显示`NULL`。

```sql
mysql> SELECT a.*,b.* FROM student a LEFT JOIN exame b ON a.uid=b.uid;
+-----+----------+-----+-----+------+------+------------+-------+
| uid | name     | age | sex | uid  | cid  | time       | score |
+-----+----------+-----+-----+------+------+------------+-------+
|   1 | zhangsan |  18 | m   |    1 |    1 | 2021-04-09 |    99 |
|   1 | zhangsan |  18 | m   |    1 |    2 | 2021-04-09 |    80 |
|   2 | gaoyang  |  20 | w   |    2 |    2 | 2021-04-09 |    90 |
|   2 | gaoyang  |  20 | w   |    2 |    3 | 2021-04-09 |    85 |
|   3 | chenwei  |  22 | m   |    3 |    1 | 2021-04-09 |    56 |
|   3 | chenwei  |  22 | m   |    3 |    2 | 2021-04-09 |    93 |
|   3 | chenwei  |  22 | m   |    3 |    3 | 2021-04-09 |    89 |
|   3 | chenwei  |  22 | m   |    3 |    4 | 2021-04-09 |   100 |
|   4 | linfeng  |  21 | w   |    4 |    4 | 2021-04-09 |    99 |
|   5 | liuxiang |  19 | w   |    5 |    2 | 2021-04-09 |    59 |
|   5 | liuxiang |  19 | w   |    5 |    3 | 2021-04-09 |    94 |
|   5 | liuxiang |  19 | w   |    5 |    4 | 2021-04-09 |    95 |
|   6 | weiwei   |  20 | m   | NULL | NULL | NULL       |  NULL |
+-----+----------+-----+-----+------+------+------------+-------+
13 rows in set (0.00 sec)
```
新添加的`weiwei`没有考试记录，在`exame`表中查询不到，所以显示`NULL`。


当然这种场景使用内连接也可以，它们没有什么区别：  
```sql
mysql> SELECT a.*,b.* FROM student a INNER JOIN exame b ON a.uid=b.uid;
+-----+----------+-----+-----+-----+-----+------------+-------+
| uid | name     | age | sex | uid | cid | time       | score |
+-----+----------+-----+-----+-----+-----+------------+-------+
|   1 | zhangsan |  18 | m   |   1 |   1 | 2021-04-09 |    99 |
|   1 | zhangsan |  18 | m   |   1 |   2 | 2021-04-09 |    80 |
|   2 | gaoyang  |  20 | w   |   2 |   2 | 2021-04-09 |    90 |
|   2 | gaoyang  |  20 | w   |   2 |   3 | 2021-04-09 |    85 |
|   3 | chenwei  |  22 | m   |   3 |   1 | 2021-04-09 |    56 |
|   3 | chenwei  |  22 | m   |   3 |   2 | 2021-04-09 |    93 |
|   3 | chenwei  |  22 | m   |   3 |   3 | 2021-04-09 |    89 |
|   3 | chenwei  |  22 | m   |   3 |   4 | 2021-04-09 |   100 |
|   4 | linfeng  |  21 | w   |   4 |   4 | 2021-04-09 |    99 |
|   5 | liuxiang |  19 | w   |   5 |   2 | 2021-04-09 |    59 |
|   5 | liuxiang |  19 | w   |   5 |   3 | 2021-04-09 |    94 |
|   5 | liuxiang |  19 | w   |   5 |   4 | 2021-04-09 |    95 |
+-----+----------+-----+-----+-----+-----+------------+-------+
12 rows in set (0.00 sec)

mysql> EXPLAIN SELECT a.*,b.* FROM student a INNER JOIN exame b ON a.uid=b.uid;
+----+-------------+-------+------------+------+---------------+---------+---------+--------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key     | key_len | ref          | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+---------+---------+--------------+------+----------+-------+
|  1 | SIMPLE      | a     | NULL       | ALL  | PRIMARY       | NULL    | NULL    | NULL         |    6 |   100.00 | NULL  |
|  1 | SIMPLE      | b     | NULL       | ref  | PRIMARY       | PRIMARY | 4       | school.a.uid |    2 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+---------+---------+--------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT a.*,b.* FROM student a LEFT JOIN exame b ON a.uid=b.uid;
+----+-------------+-------+------------+------+---------------+---------+---------+--------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key     | key_len | ref          | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+---------+---------+--------------+------+----------+-------+
|  1 | SIMPLE      | a     | NULL       | ALL  | NULL          | NULL    | NULL    | NULL         |    6 |   100.00 | NULL  |
|  1 | SIMPLE      | b     | NULL       | ref  | PRIMARY       | PRIMARY | 4       | school.a.uid |    2 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+---------+---------+--------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
```


### 右连接 RIGHT JOIN

`SELECT a.属性名列表,b.属性名列表 FROM table_name1 a RIGHT [OUTER] JOIN table_name2 b ON a.id=b.id;`  
整表扫描右表`a`中数据，在左表中搜索，在左表中不存在相应的数据则显示`NULL`。

```sql
mysql> SELECT a.*,b.* FROM student a RIGHT JOIN exame b ON a.uid=b.uid;
+------+----------+------+------+-----+-----+------------+-------+
| uid  | name     | age  | sex  | uid | cid | time       | score |
+------+----------+------+------+-----+-----+------------+-------+
|    1 | zhangsan |   18 | m    |   1 |   1 | 2021-04-09 |    99 |
|    1 | zhangsan |   18 | m    |   1 |   2 | 2021-04-09 |    80 |
|    2 | gaoyang  |   20 | w    |   2 |   2 | 2021-04-09 |    90 |
|    2 | gaoyang  |   20 | w    |   2 |   3 | 2021-04-09 |    85 |
|    3 | chenwei  |   22 | m    |   3 |   1 | 2021-04-09 |    56 |
|    3 | chenwei  |   22 | m    |   3 |   2 | 2021-04-09 |    93 |
|    3 | chenwei  |   22 | m    |   3 |   3 | 2021-04-09 |    89 |
|    3 | chenwei  |   22 | m    |   3 |   4 | 2021-04-09 |   100 |
|    4 | linfeng  |   21 | w    |   4 |   4 | 2021-04-09 |    99 |
|    5 | liuxiang |   19 | w    |   5 |   2 | 2021-04-09 |    59 |
|    5 | liuxiang |   19 | w    |   5 |   3 | 2021-04-09 |    94 |
|    5 | liuxiang |   19 | w    |   5 |   4 | 2021-04-09 |    95 |
+------+----------+------+------+-----+-----+------------+-------+
12 rows in set (0.00 sec)
```
查询右表，`weiwei`没有考试记录，所以查不到，也不会在左表中查到。

使用右连接查询和内连接在这里是不同的，会先对右表`exame`进行整表扫描：  
```sql
mysql> EXPLAIN SELECT a.*,b.* FROM student a RIGHT JOIN exame b ON a.uid=b.uid;
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref          | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------+------+----------+-------+
|  1 | SIMPLE      | b     | NULL       | ALL    | NULL          | NULL    | NULL    | NULL         |   12 |   100.00 | NULL  |
|  1 | SIMPLE      | a     | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | school.b.uid |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+--------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
```


### 场景： 学生 课程 考试

#### 问题1

查询没参加过考试的学生的详细信息。

可以使用带`IN`的子查询实现：  
```sql
mysql> SELECT * FROM student WHERE uid NOT IN (SELECT DISTINCT uid FROM exame);
+-----+--------+-----+-----+
| uid | name   | age | sex |
+-----+--------+-----+-----+
|   6 | weiwei |  20 | m   |
+-----+--------+-----+-----+
1 row in set (0.00 sec)
```
这种方式*可能*会对`SELECT DISTINCT uid FROM exame`的结果产生一张中间表，以便外部的sql查询，查完释放。  
而且`NOT IN`*可能*不会用到索引，效率可能较低。  
所以，一般不使用带`IN`的子查询来做这类的查询。


使用左连接实现：  
```sql
mysql> SELECT a.* 
    -> FROM student a
    -> LEFT JOIN exame b ON a.uid=b.uid
    -> WHERE b.uid IS NULL;
+-----+--------+-----+-----+
| uid | name   | age | sex |
+-----+--------+-----+-----+
|   6 | weiwei |  20 | m   |
+-----+--------+-----+-----+
1 row in set (0.00 sec)
```
`weiwei`没参加过考试，所以用`weiwei`的`uid`在`exame`表中查不到记录，用`WHERE b.uid IS NULL`过滤，即可得到。



#### 问题2

查询所有没参加课程cid=3的考试的学生的详细信息。

正确写法：  
```sql
mysql> SELECT a.* FROM student a
    -> LEFT JOIN exame b 
    -> ON a.uid=b.uid AND b.cid=3
    -> WHERE b.cid IS NULL;
+-----+----------+-----+-----+
| uid | name     | age | sex |
+-----+----------+-----+-----+
|   1 | zhangsan |  18 | m   |
|   4 | linfeng  |  21 | w   |
|   6 | weiwei   |  20 | m   |
+-----+----------+-----+-----+
3 rows in set (0.00 sec)
```

注意！如果要查询cid=3的记录，左连接条件，**必须写在`ON`后面**，不可以在`WHERE`后面写。如果写在`WHERE`后，就和内连接相同了（右表`b`会先被`WHERE b.cid=3`过滤，然后用过滤后的`b`右表在左表`a`中进行搜索，这样就不能找到没有参加cid=3考试的学生了）。分析：  
```sql
mysql> SELECT a.*,b.* FROM student a INNER JOIN exame b ON a.uid=b.uid WHERE b.cid=3;
+-----+----------+-----+-----+-----+-----+------------+-------+
| uid | name     | age | sex | uid | cid | time       | score |
+-----+----------+-----+-----+-----+-----+------------+-------+
|   2 | gaoyang  |  20 | w   |   2 |   3 | 2021-04-09 |    85 |
|   3 | chenwei  |  22 | m   |   3 |   3 | 2021-04-09 |    89 |
|   5 | liuxiang |  19 | w   |   5 |   3 | 2021-04-09 |    94 |
+-----+----------+-----+-----+-----+-----+------------+-------+
3 rows in set (0.00 sec)

mysql> SELECT a.*,b.* FROM student a LEFT JOIN exame b ON a.uid=b.uid WHERE b.cid=3;
+-----+----------+-----+-----+------+------+------------+-------+
| uid | name     | age | sex | uid  | cid  | time       | score |
+-----+----------+-----+-----+------+------+------------+-------+
|   2 | gaoyang  |  20 | w   |    2 |    3 | 2021-04-09 |    85 |
|   3 | chenwei  |  22 | m   |    3 |    3 | 2021-04-09 |    89 |
|   5 | liuxiang |  19 | w   |    5 |    3 | 2021-04-09 |    94 |
+-----+----------+-----+-----+------+------+------------+-------+
3 rows in set (0.00 sec)

mysql> SELECT a.*,b.* FROM student a LEFT JOIN exame b ON a.uid=b.uid AND b.cid=3;
+-----+----------+-----+-----+------+------+------------+-------+
| uid | name     | age | sex | uid  | cid  | time       | score |
+-----+----------+-----+-----+------+------+------------+-------+
|   1 | zhangsan |  18 | m   | NULL | NULL | NULL       |  NULL |
|   2 | gaoyang  |  20 | w   |    2 |    3 | 2021-04-09 |    85 |
|   3 | chenwei  |  22 | m   |    3 |    3 | 2021-04-09 |    89 |
|   4 | linfeng  |  21 | w   | NULL | NULL | NULL       |  NULL |
|   5 | liuxiang |  19 | w   |    5 |    3 | 2021-04-09 |    94 |
|   6 | weiwei   |  20 | m   | NULL | NULL | NULL       |  NULL |
+-----+----------+-----+-----+------+------+------------+-------+
6 rows in set (0.00 sec)
```

> 外连接的连接条件，必须在`ON`后面写，不可以在`WHERE`后写。
