# 【MySQL】21 - InnoDB间隙锁：解决幻读


串行化隔离级别下，如何解决虚读（幻读）问题？

答：间隙锁（gap lock）！


> 本章内容是，串行化隔离级别的实现原理，即间隙锁解决幻读问题。

## 幻读问题

幻读问题在下面这种情况下，可能会发生。

<html>
    <table style="margin: auto">
        <tr>
            <td valign="top">A事务<br>
                <!--左侧内容-->
                <pre><code>
begin;
<br>
<br>
insert into xxx_table values(满足指定条件);
commit;
                </code></pre>
            </td>
            <td valign="top">B事务<br>
                <!--右侧内容-->
                <pre><code>
begin;
select * from xxx_table where 指定条件;
(原始数据)
<br>
<br>
select * from xxx_table where 指定条件;
(出现一行新数据)
                </code></pre>
            </td>
        </tr>
    </table>
</html>


## 使用的测试数据

```sql
mysql> set tx_isolation='SERIALIZABLE', autocommit=0;
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
| 22 | bbb      |  25 | m   |
| 23 | ccc      |  15 | w   |
+----+----------+-----+-----+
8 rows in set (0.00 sec)
```



## 范围查询 示例

<html>
    <table style="margin: auto">
        <tr>
            <td valign="top">A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> 
mysql> 
mysql> 
mysql> 
mysql> 
mysql> 
mysql> 
mysql> 
mysql> 
mysql> insert into user(name,age,sex) values('ddd',25,'w');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
                </code></pre>
            </td>
            <td valign="top">B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id>11;
+----+------+-----+-----+
| id | name | age | sex |
+----+------+-----+-----+
| 12 | aaa  |  20 | m   |
| 22 | bbb  |  25 | m   |
| 23 | ccc  |  15 | w   |
+----+------+-----+-----+
3 rows in set (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

很神奇（其实也不神奇），事务A对user表的插入被阻塞了！（在可重复读级别下，不会阻塞）

事务B查询`id>11`的记录时，很明显给`id=12,22,23`这些记录加上了共享锁（你先别急），但是事务B插入了一条数据，其`id`应为24，为什么不可以对`id=24`加排他锁呢？这就涉及到**间隙锁**了，事务B并不仅仅给`id=12,22,23`这三条记录加锁了。




## InnoDB 间隙锁

当我们用范围条件而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的**已有数据记录**的索引项加锁；对于键值在条件范围内但并不存在的记录，叫做“**间隙（GAP)**” ，InnoDB 也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁。

了解几个概念，`record lock`：记录锁（行锁），`gap lock`：间隙锁，`next-key lock`：`record lock + gap lock`，有些地方将间隙锁称为`next-key lock`。

分析上节的例子：

事务B，查询`id>11`的记录时，给`id=12,22,23`这3条记录加上了共享锁（`record lock`）。还需要对**间隙**加锁，间隙由左开右闭区间定义，在这个例子中间隙有：`(11,12], (12,22], (22,23], (23,∞]`。

所以，事务B不仅对已有的记录加锁，也对满足过滤条件的不存在的记录（间隙）加了锁。这样事务A就无法添加`id>11`的记录了。否则就会出现幻读的现象。




## 间隙锁的控制范围（B+树叶子节点顺序、索引）

间隙锁了解的差不多了，但是！还存在一些小问题，做两个测试看一下。

测试用的表：  
```sql
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
| 22 | bbb      |  25 | m   |
| 23 | ccc      |  15 | w   |
+----+----------+-----+-----+
8 rows in set (0.01 sec)
```

### 测试1  间隙锁的边界在哪里？


<html>
    <table style="margin: auto">
        <tr>
            <td valign="top">A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into user(name,age,sex) values('ddd',22,'m');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> insert into user(name,age,sex) values('ddd',21,'m');
Query OK, 1 row affected (0.00 sec)
                </code></pre>
            </td>
            <td valign="top">B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where age>22;
+----+------+-----+-----+
| id | name | age | sex |
+----+------+-----+-----+
| 22 | bbb  |  25 | m   |
+----+------+-----+-----+
1 row in set (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>


事务B，查询`age>22`的记录，根据上面的分析，这里对于`age>22`的范围加上了间隙锁。但是，事务A添加`age=22`的记录时，阻塞了，添加`age=21`的记录能够成功。

这是为什么呢？`age>22`的间隙锁，为什么能阻止对`age=22`的记录的添加？

原因在于，`age`的二级索引树中，叶子节点的排列顺序。

在`age`的二级索引树中，叶子节点中存储`age,id`，按照`age`从小到大的顺序排列，`age`相同时，按照`id`从小到大的顺序排列，间隙锁的示意图如下：

|age|15|20|20|21|22|22|22|间隙锁|25(行锁)|间隙锁|
|---|---|---|---|---|---|---|---|---|---|---|
|id|23|9|12|10|7|8|11|间隙锁|22(行锁)|间隙锁|

当事务A，插入一个`age=22`的记录，该条记录的`id`应为`24`，按照叶子节点的排列顺序，它应该排在`(age=22,id=11)`后面，但是间隙锁把从`(age=22,id=11)`往后的位置全部锁住了，所以加排他锁失败。



### 测试2  索引何时使用

现在我们知道了，在串行化级别下，尽管事务A查润的是`age>22`的记录，事务B也不能对`age=22`的记录加排他锁，这是由于B+树叶子节点排序顺序导致的。

我们再来看下面这种情况。

<html>
    <table style="margin: auto">
        <tr>
            <td valign="top">A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into user(name,age,sex) values('ddd',10,'w');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
                </code></pre>
            </td>
            <td valign="top">B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where age>20;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  22 | m   |
|  8 | gaoyang  |  22 | w   |
| 10 | zhangfan |  21 | w   |
| 11 | zhanglan |  22 | w   |
| 22 | bbb      |  25 | m   |
+----+----------+-----+-----+
5 rows in set (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

这就奇怪了，事务B加锁的明明是`age>20`的记录，为什么事务A插入的`age=10`的记录也被阻塞了呢？

其实这里和间隙锁无关，事务B查询`age>20`的记录时，mysql server认为，不使用索引，直接扫描整表效率更高，所以，事务B给这张表加了共享锁。

遇到这种搞不明白的情况，不要慌，`EXPLAIN`分析一下先。

```sql
mysql> explain select * from user where age>22\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user
   partitions: NULL
         type: range
possible_keys: ageidx
          key: ageidx
      key_len: 1
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from user where age>20\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user
   partitions: NULL
         type: ALL
possible_keys: ageidx
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 9
     filtered: 55.56
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

看出来了，`age>22`时，搜索的记录很少，使用了索引，而`age>20`时，搜索的记录和整表的数量差不多，所以mysql server没有使用索引。




## 等值查询

等值查询的情况下，避免幻读现象，有两种情况。

1. 使用主键索引（唯一键）作为过滤条件，使用`record lock`行锁（共享锁）；
2. 使用辅助索引作为过滤条件，使用`record lock + gap lock`行锁和间隙锁。

这两种情况表现完全不同。


### 测试1  唯一键


<html>
    <table style="margin: auto">
        <tr>
            <td valign="top">A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into user(name,age,sex) values('ddd',16,'w');
Query OK, 1 row affected (0.00 sec)

mysql> insert into user(id,name,age,sex) values(7,'eee',17,'w');
ERROR 1062 (23000): Duplicate entry '7' for key 'PRIMARY'
                </code></pre>
            </td>
            <td valign="top">B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id=7;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  22 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

事务B，以主键（唯一键）为过滤条件进行查询时，为该行数据加上了行锁（共享锁），事务A，仍然可以插入记录行到该表中，原因是唯一键是过滤条件，不可能插入一行记录和它的主键相同，所以不可能产生幻读。


### 测试2  非唯一键  next-key lock ?!

<html>
    <table style="margin: auto">
        <tr>
            <td valign="top">A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into user(name,age,sex) values('ddd',21,'w');
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into user(name,age,sex) values('ddd',20,'w');
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into user(name,age,sex) values('ddd',19,'w');
Query OK, 1 row affected (0.00 sec)

mysql> insert into user(name,age,sex) values('ddd',18,'w');
Query OK, 1 row affected (0.00 sec)

mysql> insert into user(name,age,sex) values('ddd',22,'w');
Query OK, 1 row affected (0.00 sec)
                </code></pre>
            </td>
            <td valign="top">B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where age=21;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
| 10 | zhangfan |  21 | w   |
+----+----------+-----+-----+
1 row in set (0.01 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

事务B查询`age=21`的记录，事务A插入一行`age=21`的记录，被阻塞了，这很合理。但是，为什么插入`age=20`会被阻塞呢？又为什么插入`age=19,18`就能成功呢？又又为什么`age=22`也能成功呢？

还是和`age`二级索引树的叶子节点顺序有关：

|age|15|20|20|gap lock|21(record lock)|gap lock|22|22|22|25|
|---|---|---|---|---|---|---|---|---|---|---|
|id|23|9|12|gap lock|10(record lock)|gap lock|7|8|11|22|

如图，`next-key lock = gap lock + record lock`。

如果，只使用一个行锁（共享锁），锁住`(age=21,id=10)`，无法阻止其他的事务再插入`age=21`的记录，所以需要在`(age=21,id=10)`的左右两侧使用间隙锁（gap lock），这样一来`(20,21], (21,22]`这两个范围就被间隙锁锁住了。

又为什么`age=20`的记录不能插入呢？因为`age`相同的情况下`id`升序排列，新的`age=20`的记录`id`肯定大于`10`所以无法插入到`(20,21]`这个范围中。

为什么`age=22`的记录可以插入呢？因为新的`age=22`的记录的`id`肯定大于已经插入的`age=22`的记录，所以新的记录应该插在`(22,25]`中，这个范围并没有被锁住。

从上面这个例子也可以看出，在串行化的隔离级别下，锁冲突的情况下，串行执行，锁不冲突的情况下，并行执行。
