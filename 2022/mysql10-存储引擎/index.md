# 【MySQL】10 - 存储引擎


## MySQL 存储引擎

表的结构、数据、索引的存储方式，由**存储引擎**直接决定。

MySQL的一个特点就是，支持**插件式的存储引擎**，可以更换不同的存储引擎。

查看mysql支持的存储引擎：  
```sql
mysql> shwo engines;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'shwo engines' at line 1
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.00 sec)
```
mysql5.7之前默认使用MyISAM，而5.7之后默认使用InnoDB引擎。



mysql默认数据库文件位置在`/var/lib/mysql`，某个数据库对应一个和数据库名称相同的目录：  
```shell
[root@localhost mysql]# pwd
/var/lib/mysql
[root@localhost mysql]# ls
auto.cnf    client-cert.pem  ibdata1      ibtmp1      mysql.sock.lock     public_key.pem   server-key.pem
ca-key.pem  client-key.pem   ib_logfile0  mysql       performance_schema  school           sys
ca.pem      ib_buffer_pool   ib_logfile1  mysql.sock  private_key.pem     server-cert.pem
[root@localhost mysql]# cd mysql/
[root@localhost mysql]# ls
columns_priv.frm  general_log.CSM         innodb_table_stats.frm  proxies_priv.MYI          tables_priv.MYD
columns_priv.MYD  general_log.CSV         innodb_table_stats.ibd  server_cost.frm           tables_priv.MYI
columns_priv.MYI  general_log.frm         ndb_binlog_index.frm    server_cost.ibd           time_zone.frm
db.frm            gtid_executed.frm       ndb_binlog_index.MYD    servers.frm               time_zone.ibd
db.MYD            gtid_executed.ibd       ndb_binlog_index.MYI    servers.ibd               time_zone_leap_second.frm
db.MYI            help_category.frm       plugin.frm              slave_master_info.frm     time_zone_leap_second.ibd
db.opt            help_category.ibd       plugin.ibd              slave_master_info.ibd     time_zone_name.frm
engine_cost.frm   help_keyword.frm        proc.frm                slave_relay_log_info.frm  time_zone_name.ibd
engine_cost.ibd   help_keyword.ibd        proc.MYD                slave_relay_log_info.ibd  time_zone_transition.frm
event.frm         help_relation.frm       proc.MYI                slave_worker_info.frm     time_zone_transition.ibd
event.MYD         help_relation.ibd       procs_priv.frm          slave_worker_info.ibd     time_zone_transition_type.frm
event.MYI         help_topic.frm          procs_priv.MYD          slow_log.CSM              time_zone_transition_type.ibd
func.frm          help_topic.ibd          procs_priv.MYI          slow_log.CSV              user.frm
func.MYD          innodb_index_stats.frm  proxies_priv.frm        slow_log.frm              user.MYD
func.MYI          innodb_index_stats.ibd  proxies_priv.MYD        tables_priv.frm           user.MYI
```

每张表对应的文件名都和表名相同，后缀名表明了该文件是该表的**结构定义、数据、索引**

### MyISAM

例如，这三个文件`columns_priv.frm  columns_priv.MYD  columns_priv.MYI`就对应了`columns_priv`表的结构定义、数据、索引，该表使用的存储引擎是**`MyISAM`**。

`MyISAM`存储引擎对应的磁盘文件：  
- `frm`：表结构
- `MYD`：表数据
- `MYI`：表索引

> 重要！MyISAM引擎，数据文件和索引文件是分开单独存放的。

`MyISAM`不支持事务、也不支持外键，索引采用非聚集索引，其优势是访问的速度快，对事务完整性没
有要求，以 `SELECT`、`INSERT` 为主的应用基本上都可以使用这个存储引擎来创建表。


### InnoDB

再看，这两个文件`servers.frm  servers.ibd`，对应了`servers`表的结构定义、数据和索引，该表使用的存储引擎是**`InnoDB`**。

`InnoDB`存储引擎对应的磁盘文件：  
- `frm`：表结构
- `ibd`：表数据和表索引

> 重要！InnoDB引擎，数据文件和索引文件存放在同一个文件中！

`InnoDB` 存储引擎提供了具有提交、回滚和崩溃恢复能力的事务安全，支持自动增长列，外键等功能，
索引采用聚集索引。


### MEMORY

`MEMORY` 存储引擎使用存在内存中的内容来创建表。每个`MEMORY` 表实际只对应一个磁盘文件，格式
是`.frm`（表结构定义）。`MEMORY` 类型的表访问非常快，因为它的数据是放在内存中的，并且默认使用 `HASH` 索引（不适合做范围查询），但是一旦服务关闭，表中的数据就会丢失掉。






## 各引擎的区别

|存储引擎|锁机制|B-树索引|哈希索引|外键|事务|索引缓存|数据缓存|
|---|---|---|---|---|---|---|---|
|MyISAM|表锁|支持|不支持|不支持|不支持|支持|不支持|
|InnoDB|行锁|支持|不支持|支持|支持|支持|支持|
|MEMORY|表锁|支持|支持|不支持|不支持|支持|支持|

- 锁机制：表示数据库在并发请求访问的时候，多个事务在操作时，并发操作的粒度。
- B-树索引和哈希索引：主要是加速SQL的查询速度。
- 外键：子表的字段依赖父表的主键，设置两张表的依赖关系。
- 事务：多个SQL语句，保证它们共同执行的原子操作，要么成功，要么失败，不能只成功一部分，失败
需要回滚事务。
- 索引缓存和数据缓存：和MySQL Server的查询缓存相关，在没有对数据和索引做修改之前，重复查询
可以不用进行磁盘I/O（数据库的性能提升，目的是为了减少磁盘I/O操作来提升数据库访问效率），读
取上一次内存中查询的缓存就可以了。
