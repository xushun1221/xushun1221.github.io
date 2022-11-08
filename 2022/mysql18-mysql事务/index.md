# 【MySQL】18 - MySQL事务


## 事务的核心概念

一个事务是由一条或者多条对数据库操作的SQL语句所组成的一个不可分割的单元（原子性），只有当事务中的所有操作都正常执行完了，整个事务才会被提交给数据库；如果有部分事务处理失败，那么事务就要回退到最初的状态，因此，事务要么全部执行成功，要么全部失败。

记住事务的几个基本概念，如下：  
1. 事务是一组SQL语句的执行，要么全部成功，要么全部失败，不能出现部分成功，部分失败的结
果。保证事务执行的原子操作。
2. 事务的所有SQL语句全部执行成功，才能提交（commit）事务，把结果写回磁盘上。
3. 事务执行过程中，有的SQL出现错误，那么事务必须要回滚（rollback）到最初的状态。（回滚操作由 **undo log 回滚日志** 支持）


执行过程示意：  
```
try{
    begin;  开启事务
    多条sql
    commit; 提交事务
} catch (... err) {
    rollback;   回滚事务
}
```

> MyISAM 不支持事务  
> InnoDB 支持事务、支持行锁（最大的特点）


### 示例

例如，这样一张表：`account(name,balance)`，存储用户账户余额，如果zhangsan给liuchao转账50元，那么会涉及下面两条sql：  
```sql
UPDATE account SET balance=balance-50.0 WHERE name='zhangsan';
UPDATE account SET balance=balance+50.0 WHERE name='liuchao';
```
这一转账操作，必须由这两条sql共同构成。如果在第一条sql和第二条sql之间，业务代码返回或者发生异常，又或者mysql server本身出现问题，导致第一条sql执行成功，而第二条sql未执行。转账操作就失败了。

这两条sql应该组成一个完整的事务，必须全部执行成功或者全部失败。


### autocommit 自动提交事务

查看事务提交状态：  
```sql
mysql> SELECT @@autocommit;
+--------------+
| @@autocommit |
+--------------+
|            1 |
+--------------+
1 row in set (0.00 sec)
```
默认情况下，事务自动提交。


如果业务中涉及了事务，需要把事务自动提交改为手动提交，在业务层代码上进行控制。  
```sql
mysql> SET autocommit=0;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT @@autocommit;
+--------------+
| @@autocommit |
+--------------+
|            0 |
+--------------+
1 row in set (0.00 sec)
```





## ACID 特性

每一个事务必须满足下面的4个特性：  
1. 事务的原子性（**A**tomic）：
    事务是一个不可分割的整体，事务必须具有原子特性，及当数据修改时，要么全执行，要么全不执行，即不允许事务部分的完成。
2. 事务的一致性（**C**onsistency）：
    一个事务执行之前和执行之后，数据库数据必须保持一致性状态。数据库的一致性状态必须由用户来负责，由并发控制机制实现。就拿网上购物来说，你只有让商品出库，又让商品进入顾客的购物车才能构成一个完整的事务。
3. 事务的隔离性（**I**solation）：
    当两个或者多个事务并发执行时，为了保证数据的安全性，将一个事物内部的操作与其它事务的操作隔离起来，不被其它正在执行的事务所看到，使得并发执行的各个事务之间不能互相影响。
4. 事务的持久性（**D**urability）：
    事务完成(commit)以后，DBMS保证它对数据库中的数据的修改是永久性的，即使数据库因为故障出错，也应该能够恢复数据！

> 热知识，向DB中写数据时（commit提交事务时），数据先被写入内存的cache缓存中，然后再通过磁盘IO适时地写入磁盘中。从cache写数据到磁盘中，需要耗时，如果在这个过程中，出现内部或外部错误，没有成功地将数据写入磁盘，但commit提交已经返回，这样就会出错。  
> MySQL 可以通过 **redo log 重做日志**，保证对数据库的修改是永久性的！  
> 只要commit返回了，就表示对数据库的修改成功了，DBMS保证这一点。

MySQL中最重要的是：**日志**！而不是数据。

- MySQL的ACD特性，是由redo log和undo log机制来保证的；
- MySQL的I特性，是由锁机制来保证的。




## MySQL事务处理命令

1. `SELECT @@autocommit;`：查看自动提交事务
2. `SET autocommit=0;`：0，手动提交，1，自动提交
3. `BEGIN;`：开启一个事务
4. `COMMIT;`：提交一个事务
5. `ROLLBACK;`：回滚一个事务到初始位置
6. `SAVEPOINT point1;`：设置一个名为point1的保存点
7. `ROLLBACK TO point1;`：事务回滚到保存点point1，而不是初始状态
8. `SET TX_ISOLATION='REPEATABLE-READ'`：设置事务的隔离级别
9. `SELECT @@tx_isolation;`查询事务的隔离级别


