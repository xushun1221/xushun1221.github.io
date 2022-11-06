# 【MySQL】16 - InnoDB自适应哈希索引


## 自适应哈希索引 - 提高二级索引查询性能

InnoDB中，使用二级索引树进行查询时，如果查询字段在二级索引树上，就直接完成搜索，否则需要根据搜索到的主键，在主键索引树中再次搜索，这就是**回表**。

这种情况下的查询，即搜了二级索引树，也搜了主键索引树，大量这样的查询效率很低，InnoDB存储引擎会自适应地进行优化。

InnoDB存储引擎，如果监测到某个二级索引不断地被使用，那么它就会根据这个二级索引树，在内存上构建一个哈希索引（二级索引key->对应数据），来加速搜索（等值查询），省去了二级索引树和主键索引树的搜索过程。


自适应哈希索引示意图：

![](/post_images/posts/Database/MySQL/自适应哈希索引.jpg "自适应哈希索引")


一些描述信息：  
> In MySQL 5.7, the adaptive hash index search system is partitioned. Each index is bound to a specific partition, and each partition is protected by a separate latch.
Partitioning is controlled by the innodb_adaptive_hash_index_parts configuration option. In earlier releases, the adaptive hash index search system was protected by a single latch which could become a point of contention under heavy workloads. The
innodb_adaptive_hash_index_parts option is set to 8 by default.
The maximum setting is 512.
The hash index is always built based on an existing B-tree index on the table. InnoDB can build a hash index on a prefix of any length of the key defined for the B-tree, depending on the pattern of searches that InnoDB observes for the B-tree index. A hash index can be partial, covering only those pages of the index that are often accessed.
You can monitor the use of the adaptive hash index and the contention for its use in the SEMAPHORES section of the output of the SHOW ENGINE INNODB STATUS command. If you see many threads waiting on an RW-latch created in btr0sea.c, then it might be useful to disable adaptive hash indexing. 



### 自适应哈希使用

> 建立和维护哈希索引需要消耗一些资源，并不是在任何情况下都可以提高二级索引的查询性能，根据性能指标具体分析，我们可以适时地关闭或打开它。

查看自适应哈希状态，默认处于开启状态：  
```sql
ysql> SHOW VARIABLES LIKE 'innodb_adaptive_hash_index';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| innodb_adaptive_hash_index | ON    |
+----------------------------+-------+
1 row in set (0.00 sec)
```


查看分区，默认8个分区，每个分区管理一个或多个桶，每个分区有一把锁，不同分区的内容可以并行处理：  
```sql
mysql> SHOW VARIABLES LIKE 'innodb_adaptive_hash_index_parts';
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| innodb_adaptive_hash_index_parts | 8     |
+----------------------------------+-------+
1 row in set (0.00 sec)
```


查看InnoDB状态，查看自适应哈希索引的两个重要信息  
1. RW-latch等待的线程数量（自适应哈希索引默认分配了8个分区，同一个分区等待的线程数量过多，性能低，关闭）
2. 使用自适应哈希索引查询的占比（占比低，说明不需要，关闭），和使用二级索引查询的占比

```console
mysql> SHOW ENGINE INNODB STATUS\G
*************************** 1. row ***************************
  Type: InnoDB
  Name: 
Status: 
=====================================
2022-11-06 03:59:54 0x7f09a82e2700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 4 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 18 srv_active, 0 srv_shutdown, 65423 srv_idle
srv_master_thread log flush and writes: 65441
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 46
OS WAIT ARRAY INFO: signal count 54
RW-shared spins 0, rounds 112, OS waits 40
RW-excl spins 0, rounds 2, OS waits 0
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 112.00 RW-shared, 2.00 RW-excl, 0.00 RW-sx
------------
TRANSACTIONS
------------
Trx id counter 2008440
Purge done for trx's n:o < 2008438 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421154888624864, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421154888623952, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
15308 OS file reads, 9730 OS file writes, 4711 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
---
LOG
---
Log sequence number 234947964
Log flushed up to   234947964
Pages flushed up to 234947964
Last checkpoint at  234947955
0 pending log flushes, 0 pending chkp writes
174 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 171209
Buffer pool size   8192
Free buffers       1024
Database pages     7168
Old database pages 2626
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 15166, not young 274851
0.00 youngs/s, 0.00 non-youngs/s
Pages read 14960, created 4455, written 4738
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 7168, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=1781, Main thread ID=139679611987712, state: sleeping
Number of rows inserted 0, updated 0, deleted 0, read 16000044
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================

1 row in set (0.00 sec)
```






