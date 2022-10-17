# 【MySQL】07 - LIMIT分页查询


## 分页查询

分页查询的基础语法：  
- `SELECT * FROM user LIMIT 2;`，user表中的前两行
- `SELECT * FROM user LIMIT 1,3;`，从user表中第1行开始取三行
- `SELECT * FROM user LIMIT 3 OFFSET 1;`，同上


## 查询效率相关问题

我们想要知道一些sql语句的详细情况，可以使用`EXPLAIN sql`来查看sql可能的执行信息。

例如，我们想知道按`name`字段查询`user`表中记录的情况：  
```sql
mysql> EXPLAIN SELECT * FROM user WHERE name='zhangsan';
+----+-------------+-------+------------+-------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | user  | NULL       | const | name          | name | 152     | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
可以看到，由于`name`字段有`UNIQUE`约束，所以可能使用了索引，`rows`查询只用了一行。

再试一个：  
```sql
mysql> EXPLAIN SELECT * FROM user WHERE name='wangwu';
+----+-------------+-------+------------+-------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | user  | NULL       | const | name          | name | 152     | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
相同，也是一行。


如果，按没有索引的字段搜索，会怎么样呢？  
```sql
mysql> EXPLAIN SELECT * FROM user WHERE age=24;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
可以看到，`rows`是3，我们的表中就只有三个记录，它搜索了三次，说明是按整表搜索的。如果数据量很大，那么检索的效率非常低，实际使用时应该加索引。


还有一种情况，如果我们只想知道第一个24岁的人，我们可以使用`LIMIT`进行限制，此时查询到第一个24岁的人就不会继续查询了，效率很高。`SELECT * FROM user WHERE age=24 LIMIT 1;`

> 注意，`EXPLAIN`不能看到`LIMIT`的限制，这里仍然会提示使用整表搜索，但其实并没有发生。  
> `EXPLAIN`有可能无法显示出mysql进行的优化。


## 存储过程 - 测试数据

`LIMIT`在一些情况下可以提高效率，但是我们的数据库数据太少了，怎么建一张数据很多的表来测试呢？可以用**存储过程**。

它相当于一段用于处理数据的代码，我们可以调用它来批量处理数据。

先创建一张表：  
```sql
mysql> CREATE TABLE user_t(id INT(11) UNSIGNED PRIMARY KEY NOT NULL AUTO_INCREMENT,
    -> email VARCHAR(255) DEFAULT NULL,
    -> passwor VARCHAR(255) DEFAULT NULL)
    -> ENGINE=INNODB
    -> DEFAULT CHARSET=UTF8
    -> AUTO_INCREMENT=1000001;
Query OK, 0 rows affected (0.01 sec)
```

创建200万测试数据：  
```sql
mysql> delimiter $
mysql> CREATE PROCEDURE add_user_t(IN n INT)
    -> BEGIN
    -> DECLARE i INT;
    -> SET i=0;
    -> WHILE i<n DO
    -> INSERT INTO user_t VALUES(NULL,CONCAT(i+1,'@test.com'), i+1);
    -> set i=i+1;
    -> END WHILE;
    -> END$
Query OK, 0 rows affected (0.35 sec)

mysql> delimiter ;
mysql> call add_user_t(2000000);
Query OK, 1 row affected (6 min 52.12 sec)

mysql> SELECT * FROM user_t LIMIT 10;
+---------+-------------+---------+
| id      | email       | passwor |
+---------+-------------+---------+
| 1000001 | 1@test.com  | 1       |
| 1000002 | 2@test.com  | 2       |
| 1000003 | 3@test.com  | 3       |
| 1000004 | 4@test.com  | 4       |
| 1000005 | 5@test.com  | 5       |
| 1000006 | 6@test.com  | 6       |
| 1000007 | 7@test.com  | 7       |
| 1000008 | 8@test.com  | 8       |
| 1000009 | 9@test.com  | 9       |
| 1000010 | 10@test.com | 10      |
+---------+-------------+---------+
10 rows in set (0.01 sec)
```
查看存储过程：`SHOW CREATE PROCEDURE add_user_t\G`  
删除存储过程：`DROP PROCEDURE add_user_t`


## 测试LIMIT效率问题

如果我们需要查询一个没有索引的字段，那么可以使用`LIMIT`来提高一些查询效率。但是如果该字段有重复值的话，就没办法了，只能整表查询。


### 示例
如果我们在200万数据中按没有索引的字段检索的话，需要进行整表查询，尽管查询的是第一条记录，如下：  
```sql
mysql> SELECT * FROM user_t WHERE email='1@test.com';
+---------+------------+---------+
| id      | email      | passwor |
+---------+------------+---------+
| 1000001 | 1@test.com | 1       |
+---------+------------+---------+
1 row in set (0.48 sec)

mysql> EXPLAIN SELECT * FROM user_t WHERE email='1@test.com';
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | user_t | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1994771 |    10.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

而如果，我们用`LIMIT`限制，只查询第一个email字段为`1@test.com`的记录，就不会进行整表查询，查询用时很少，从下面查询时间可以看出：  
```sql
mysql> SELECT * FROM user_t WHERE email='1@test.com' LIMIT 1;
+---------+------------+---------+
| id      | email      | passwor |
+---------+------------+---------+
| 1000001 | 1@test.com | 1       |
+---------+------------+---------+
1 row in set (0.00 sec)
```

如果我们查询第一个email字段为`1000000@test.com`的记录，它在第100万行，还是需要从第0行到第100万行过一边（因为email字段没有索引），需要用多一点时间，但也不需要整表查询：  
```sql
mysql> SELECT * FROM user_t WHERE email='1000000@test.com' LIMIT 1;
+---------+------------------+---------+
| id      | email            | passwor |
+---------+------------------+---------+
| 2000000 | 1000000@test.com | 1000000 |
+---------+------------------+---------+
1 row in set (0.25 sec)
```


## 分页业务 - LIMIT的正确使用方法

考虑这样的常见功能，我们需要根据页数来查询一些信息，假设每页20条记录，`pagenum=20`，我们应该怎么写查询语句？

最朴素的想法是，根据页数来计算偏移，再找到偏移后的20条记录即可：`SELECT * FROM user_t LIMIT (page-1)*pagenum,pagenum`。

但是，这样的方法存在一个问题，例如查询第10页和第5万页：  
```sql
mysql> SELECT * FROM user_t LIMIT 180,20;
+---------+--------------+---------+
| id      | email        | passwor |
+---------+--------------+---------+
| 1000181 | 181@test.com | 181     |
| 1000182 | 182@test.com | 182     |
| 1000183 | 183@test.com | 183     |
| 1000184 | 184@test.com | 184     |
| 1000185 | 185@test.com | 185     |
| 1000186 | 186@test.com | 186     |
| 1000187 | 187@test.com | 187     |
| 1000188 | 188@test.com | 188     |
| 1000189 | 189@test.com | 189     |
| 1000190 | 190@test.com | 190     |
| 1000191 | 191@test.com | 191     |
| 1000192 | 192@test.com | 192     |
| 1000193 | 193@test.com | 193     |
| 1000194 | 194@test.com | 194     |
| 1000195 | 195@test.com | 195     |
| 1000196 | 196@test.com | 196     |
| 1000197 | 197@test.com | 197     |
| 1000198 | 198@test.com | 198     |
| 1000199 | 199@test.com | 199     |
| 1000200 | 200@test.com | 200     |
+---------+--------------+---------+
20 rows in set (0.00 sec)

mysql> SELECT * FROM user_t LIMIT 999980,20;
+---------+------------------+---------+
| id      | email            | passwor |
+---------+------------------+---------+
| 1999981 | 999981@test.com  | 999981  |
| 1999982 | 999982@test.com  | 999982  |
| 1999983 | 999983@test.com  | 999983  |
| 1999984 | 999984@test.com  | 999984  |
| 1999985 | 999985@test.com  | 999985  |
| 1999986 | 999986@test.com  | 999986  |
| 1999987 | 999987@test.com  | 999987  |
| 1999988 | 999988@test.com  | 999988  |
| 1999989 | 999989@test.com  | 999989  |
| 1999990 | 999990@test.com  | 999990  |
| 1999991 | 999991@test.com  | 999991  |
| 1999992 | 999992@test.com  | 999992  |
| 1999993 | 999993@test.com  | 999993  |
| 1999994 | 999994@test.com  | 999994  |
| 1999995 | 999995@test.com  | 999995  |
| 1999996 | 999996@test.com  | 999996  |
| 1999997 | 999997@test.com  | 999997  |
| 1999998 | 999998@test.com  | 999998  |
| 1999999 | 999999@test.com  | 999999  |
| 2000000 | 1000000@test.com | 1000000 |
+---------+------------------+---------+
20 rows in set (0.22 sec)
```

可以看到，虽然同样是查询20条记录，第10页用时0s，第5万页用时0.22s，相差很多，这是因为，没有使用索引，在LIMIT偏移时，还是遍历了第5万页之前的所有记录。

优化方法，使用`id`字段的索引，以常数时间跳过5万页之前的记录：  
```sql
mysql> SELECT * FROM user_t WHERE id>1999980 LIMIT 20;
+---------+------------------+---------+
| id      | email            | passwor |
+---------+------------------+---------+
| 1999981 | 999981@test.com  | 999981  |
| 1999982 | 999982@test.com  | 999982  |
| 1999983 | 999983@test.com  | 999983  |
| 1999984 | 999984@test.com  | 999984  |
| 1999985 | 999985@test.com  | 999985  |
| 1999986 | 999986@test.com  | 999986  |
| 1999987 | 999987@test.com  | 999987  |
| 1999988 | 999988@test.com  | 999988  |
| 1999989 | 999989@test.com  | 999989  |
| 1999990 | 999990@test.com  | 999990  |
| 1999991 | 999991@test.com  | 999991  |
| 1999992 | 999992@test.com  | 999992  |
| 1999993 | 999993@test.com  | 999993  |
| 1999994 | 999994@test.com  | 999994  |
| 1999995 | 999995@test.com  | 999995  |
| 1999996 | 999996@test.com  | 999996  |
| 1999997 | 999997@test.com  | 999997  |
| 1999998 | 999998@test.com  | 999998  |
| 1999999 | 999999@test.com  | 999999  |
| 2000000 | 1000000@test.com | 1000000 |
+---------+------------------+---------+
20 rows in set (0.00 sec)
```

那么，这个业务的逻辑就是：`SELECT * FROM user_t WHERE id>上一页最后一条记录的id LIMIT 20;`
