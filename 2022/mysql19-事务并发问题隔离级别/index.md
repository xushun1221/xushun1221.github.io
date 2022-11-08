# 【MySQL】19 - 事务并发问题&隔离级别


## 事务并发存在的问题

事务处理不经隔离，并发执行事务时通常会发生以下的问题：

- **脏读（Dirty Read）**：一个事务读取了另一个事务未提交的数据。例如当事务A和事务B并发执行时，当事务A更新后，事务B查询读取到A尚未提交的数据，此时事务A回滚，则事务B读到的数据就是无效的脏数据。（事务B读取了事务A尚**未提交**的数据）（绝对无法接受的）
- **不可重复读（NonRepeatable Read）**：一个事务的操作导致另一个事务前后两次读取到不同的数据。例如当事务A和事务B并发执行时，当事务B查询读取数据后，事务A更新操作更改事务B查询到的数据，此时事务B再次去读该数据，发现前后两次读的数据不一样。（事务B读取了事务A**已提交**的数据）（业务上不一定无法接受）
- **虚读（Phantom Read）幻读**：一个事务的操作导致另一个事务前后两次查询的结果数据量不同。例如当事务A和事务B并发执行时，当事务B查询读取数据后，事务A新增或者删除了一条满足事务B查询条件的记录，此时事务B再去查询，发现查询到前一次不存在的记录，或者前一次查询的一些记录不见了。（事务B读取了事务A新增加的数据或者读不到事务A删除的数据）（业务上不一定无法接受）

> 脏读是错误的，不可接受的；  
> 不可重复读和虚读，不是一种错误，是否允许要看业务的需求，取决于对事务并发程度高低的需要。

为了数据的安全性，提出了数据的隔离级别，在不同的程度上，允许或不允许脏读、不可重复读、虚读的出现。




## 事务的隔离级别

MySQL支持的四种隔离级别是：

1. `TRANSACTION_READ_UNCOMMITTED`，**未提交读**。说明在提交前一个事务可以看到另一个事务的变化。这样读脏数据，不可重复读和虚读都是被允许的；
2. `TRANSACTION_READ_COMMITTED`，**已提交读**。说明读取未提交的数据是不允许的。这个级别仍然允许不可重复读和虚读产生；
3. `TRANSACTION_REPEATABLE_READ`，**可重复读**。说明事务保证能够再次读取相同的数据而不会失败，但虚读仍然会出现（不是一定出现，一定程度上能防止虚读）；（MySQL的默认隔离级别）
4. `TRANSACTION_SERIALIZABLE`，**串行化**。不能并行，是最高的事务级别，它防止读脏数据，不可重复读和虚读。


|隔离级别|脏读|不可重复读|虚读|
|---|---|---|---|
|未提交读|Y|Y|Y|
|已提交读|N|Y|Y|
|可重复读|N|N|Y|
|串行化|N|N|N|


> 事务隔离级别越高，为避免冲突所花费的性能也就越多。  
> 在可重复读级别，实际上可以解决部分的虚读问题，但是不能防止update更新产生的虚读问题，要禁止虚读产生，还是需要设置串行化隔离级别。

MySQL的默认隔离级别是：可重复读。





## 测试示例

用两个终端各自连接mysql server进行测试，记住要关闭自动提交事务（`SET autocommit=0;`）。

用这个表：  
```sql
mysql> select * from user;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  20 | m   |
|  8 | gaoyang  |  22 | w   |
|  9 | chenwei  |  20 | m   |
| 10 | zhangfan |  21 | w   |
| 11 | zhanglan |  22 | w   |
+----+----------+-----+-----+
5 rows in set (0.00 sec)
```

### 未提交读

#### 脏读测试

<html>
    <table style="margin: auto">
        <tr>
            <td>A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> select * from user;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  20 | m   |
|  8 | gaoyang  |  22 | w   |
|  9 | chenwei  |  20 | m   |
| 10 | zhangfan |  21 | w   |
| 11 | zhanglan |  22 | w   |
+----+----------+-----+-----+
5 rows in set (0.00 sec)

mysql> SET tx_isolation='READ-UNCOMMITTED';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> UPDATE user SET age=21 WHERE name='zhangsan';
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> ROLLBACK;
Query OK, 0 rows affected (0.00 sec)
                </code></pre>
            </td>
            <td>B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> select * from user;
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  20 | m   |
|  8 | gaoyang  |  22 | w   |
|  9 | chenwei  |  20 | m   |
| 10 | zhangfan |  21 | w   |
| 11 | zhanglan |  22 | w   |
+----+----------+-----+-----+
5 rows in set (0.00 sec)

mysql> SET tx_isolation='READ-UNCOMMITTED';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM user WHERE name='zhangsan';
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  20 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)

mysql> SELECT * FROM user WHERE name='zhangsan';
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  21 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)

mysql> ROLLBACK;
Query OK, 0 rows affected (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

可以看到，初始时，`zhangsan`的年龄为20，事务A将他的年龄改为21，但是并没有COMMIT的情况下，事务B，可以读取到他的年龄为21，事务B可能用21这个年龄进行一些操作，最终事务A回滚，导致事务B的执行是错误的。


### 已提交读

已提交读级别下，脏读不会发生了，但是不可重复读和虚读仍会发生。


#### 脏读、不可重复读测试

<html>
    <table style="margin: auto">
        <tr>
            <td>A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> SET tx_isolation='READ-COMMITTED';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> UPDATE user SET age=21 WHERE name='zhangsan';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> COMMIT;
Query OK, 0 rows affected (0.00 sec)
                </code></pre>
            </td>
            <td>B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> SET tx_isolation='READ-COMMITTED';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * from user WHERE name='zhangsan';
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  20 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)

mysql> SELECT * from user WHERE name='zhangsan';
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  20 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)

mysql> SELECT * from user WHERE name='zhangsan';
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  21 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)

mysql> COMMIT;
Query OK, 0 rows affected (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

可以看到，初始时事务B读`zhangsan`的年龄为20，事务A将其年龄改为21，未提交时，事务B再次读取，显示的年龄仍然为20，说明脏读未发生。事务A提交之后，事务B再次读取，显示年龄为21，对于事务B而言，两次读取数据不同，不可重复读发生了。


### 可重复读

可重复读级别下（MySQL默认工作级别），脏读、不可重复读被限制了，但是虚读仍然会发生（不是一定发生，一定程度上能防止虚读出现）。

#### 脏读、不可重复读测试

<html>
    <table style="margin: auto">
        <tr>
            <td>A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> SET tx_isolation='REPEATABLE-READ';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> UPDATE user SET age=22 WHERE name='zhangsan';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> COMMIT;
Query OK, 0 rows affected (0.00 sec)
                </code></pre>
            </td>
            <td>B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> SET tx_isolation='REPEATABLE-READ';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM user WHERE name='zhangsan';
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  21 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)

mysql> SELECT * FROM user WHERE name='zhangsan';
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  21 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)

mysql> SELECT * FROM user WHERE name='zhangsan';
+----+----------+-----+-----+
| id | name     | age | sex |
+----+----------+-----+-----+
|  7 | zhangsan |  21 | m   |
+----+----------+-----+-----+
1 row in set (0.00 sec)

mysql> ROLLBACK;
Query OK, 0 rows affected (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

可以看出，初始时事务B查询`zhangsan`的年龄为21，事务A修改其为22后，未提交时，事务B查询，仍为21，说明脏读未发生，事务A提交后，事务B再次查询，发现其年龄仍为21，说明不可重复读未发生。


#### 虚读测试 - 事务A INSERT

<html>
    <table style="margin: auto">
        <tr>
            <td>A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO user(name,age,sex) VALUES('aaa',20,'m');
Query OK, 1 row affected (0.00 sec)

mysql> COMMIT;
Query OK, 0 rows affected (0.00 sec)
                </code></pre>
            </td>
            <td>B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM user WHERE age=20;
+----+---------+-----+-----+
| id | name    | age | sex |
+----+---------+-----+-----+
|  9 | chenwei |  20 | m   |
+----+---------+-----+-----+
1 row in set (0.00 sec)

mysql> SELECT * FROM user WHERE age=20;
+----+---------+-----+-----+
| id | name    | age | sex |
+----+---------+-----+-----+
|  9 | chenwei |  20 | m   |
+----+---------+-----+-----+
1 row in set (0.00 sec)

mysql> SELECT * FROM user WHERE age=20;
+----+---------+-----+-----+
| id | name    | age | sex |
+----+---------+-----+-----+
|  9 | chenwei |  20 | m   |
+----+---------+-----+-----+
1 row in set (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

可以看出，初始时事务B查询年龄为20的记录，只有一条，事务A在表中添加了一条年龄为20的数据，未提交时和提交后，事务B都不能查询到新添加的记录，说明虚读没有发生。

这说明了，在可重复读级别下，虚读不是一定会发生的。


#### 虚读测试 - 事务B UPDATE

在上一节的事务B中继续执行

```
mysql> UPDATE user SET age=20 WHERE name='aaa';
Query OK, 0 rows affected (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 0

mysql> SELECT * FROM user WHERE age=20;
+----+---------+-----+-----+
| id | name    | age | sex |
+----+---------+-----+-----+
|  9 | chenwei |  20 | m   |
| 12 | aaa     |  20 | m   |
+----+---------+-----+-----+
2 rows in set (0.00 sec)
```

可以看到，在事务B中，对新添加进来的记录进行UPDATE操作后，再次查询，就能查到该记录了，这说明，虚读还是发生了。

所以，在可重复读级别下，对于UPDATE操作仍然没有限制虚读的发生。（这是由于并发控制决定的）


### 串行化


#### 虚读测试

<html>
    <table style="margin: auto">
        <tr>
            <td>A事务<br>
                <!--左侧内容-->
                <pre><code>
mysql> SET tx_isolation='SERIALIZABLE';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO user(name,age,sex) VALUES('bbb',20,'w');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
                </code></pre>
            </td>
            <td>B事务<br>
                <!--右侧内容-->
                <pre><code>
mysql> SET tx_isolation='SERIALIZABLE';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM user WHERE age=20;
+----+---------+-----+-----+
| id | name    | age | sex |
+----+---------+-----+-----+
|  9 | chenwei |  20 | m   |
| 12 | aaa     |  20 | m   |
+----+---------+-----+-----+
2 rows in set (0.00 sec)
                </code></pre>
            </td>
        </tr>
    </table>
</html>

可以看到，事务B初始时，查询了user表中的数据，事务B未提交时，事务A对user表进行了插入，阻塞了，很明显这里有锁进行了控制，最终事务A插入的sql超时返回（mysql server为了防止线程死锁，设置了超时时间），虚读显然不能发生。

关于锁相关的内容后面再介绍。
