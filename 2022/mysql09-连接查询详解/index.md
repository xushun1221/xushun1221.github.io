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

### 场景一 学生 课程 考试成绩

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
> 搜索结果需要进行过滤，对于INNER JOIN内连接，过滤条件写在WHERE后面和ON连接条件中，效果是一样的


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

