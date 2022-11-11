# 【MySQL】25 - MySQL如何优化？



无论是在实际工程项目中，还是面试中，对MySQL的优化都是一个比较常见的问题。这里分几个方面来谈一下MySQL的优化方法。


## MySQL需要优化的方面有哪些？


1. SQL和索引的优化：[SQL和索引的优化](https://xushun1221.github.io/2022/mysql17-sql%E5%92%8C%E7%B4%A2%E5%BC%95%E4%BC%98%E5%8C%96%E6%85%A2%E6%9F%A5%E8%AF%A2%E6%97%A5%E5%BF%97/)  
2. 应用上的优化
   1. 引入**连接池**
   2. 引入**缓存层**，存储热点数据，redis、memcache等
3. mysql server上的优化：根据服务器性能设置各种参数的配置




## 应用优化

除了优化SQL和索引，很多时候，在实际生产环境中，由于数据库服务器本身的性能局限，就必须要对上层的应用来进行一些优化，使得上层应用访问数据库的压力能够减到最小。

### 连接池

应用上一般访问数据库，都是先和MySQL Server创建连接，然后发送SQL语句，Server处理完成后，再把结果通过网络返回给应用，然后关闭和MySQL Server的连接，因此短时间大量的数据库访问，消耗的TCP三次握手和四次挥手所花费的时间就很大了，稍微大一点的项目，我们都会在应用访问数据库的那一层上，添加连接池模块，相当于应用和MySQL Server事先创建一组连接，当应用需要请求MySQL Server时，不需要再进行TCP连接和释放连接了，一般连接池都会维护以下资源：
1. 连接池里面保持固定数量的活跃TCP连接，供应用使用。
2. 如果应用瞬间访问MySQL的量比较大，那么连接池会实时创建更多的连接给应用使用。
3. 当连接池里面的TCP连接一段时间内没有被用到，连接池会释放多余的连接资源，保留它设置的最大空闲连接量就可以了。
4. 
连接池可以自己实现，也可以用第三方写好的库。

### 增加cache缓存层

业务上增加redis、memcache，一般用缓存把经常访问的数据缓存起来。




## MySQL Server优化

对于MySQL Server端的优化，主要指的就是MySQL Server启动时加载的配置文件的配置项内容的优化（就是那个`my.ini`或者`my.cnf`），下面我们看看它的配置文件中有哪些是我们需要重点关注的优化参数。


### MySQL查询缓存

MySQL的查询缓存是把select查询语句上一次的查询结果记录下来放在缓存当中，下一次再查询相同内容的时候，直接从缓存中取出来就可以了，不用再进行一遍真正的SQL查询。但是当两个select查询中间出现insert，update，delete语句的时候，查询缓存就会被清空。查询缓存适用更新不频繁的表，因为当表更新频繁的话，查询缓存也总是被清空，过多的查询缓存的数据添加和删除，就会影响MySQL的执行效率，还不如每次都从磁盘上查来得快（缓存指的就是一块内存，内存I/O比磁盘I/O快很多）。

可以在MySQL上通过以下命令，来查看查询缓存的设置：  
```sql
mysql> SHOW VARIABLES LIKE '%query_cache%';
+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| have_query_cache             | YES     |
| query_cache_limit            | 1048576 |
| query_cache_min_res_unit     | 4096    |
| query_cache_size             | 1048576 |
| query_cache_type             | OFF     |
| query_cache_wlock_invalidate | OFF     |
+------------------------------+---------+
```

通过show status命令，可以查看MySQL查询缓存的使用状况，如下：  
```sql
mysql> SHOW STATUS LIKE 'Qcache%';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| Qcache_free_blocks      | 1       |
| Qcache_free_memory      | 1031832 |
| Qcache_hits             | 0       |
| Qcache_inserts          | 0       |
| Qcache_lowmem_prunes    | 0       |
| Qcache_not_cached       | 1       |
| Qcache_queries_in_cache | 0       |
| Qcache_total_blocks     | 1       |
+-------------------------+---------+
```

可以通过set命令设置上面的缓存参数开启MySQL查询缓存功能，也可以找到MySQL的配置文件（windows是my.ini，linux是my.cnf），修改query_cache_type参数为1就可以了，然后重启MySQLServer就可以使用了。



### 索引和数据缓存

主要指的就是innodb_buffer_pool_size配置项，从名字上就能看到，该配置项是针对InnoDB存储引擎起作用的，这个参数定义了InnoDB 存储引擎的表数据和索引数据的最大内存缓冲区大小。innodb_buffer_pool_size是同时为数据块和索引块做缓存，这个值设得越高，访问表中数据需要的磁盘 I/O 就越少。



### MySQL线程缓存

主要指配置文件中thread_cache_size配置项。给大家讲过MySQL Server网络模块采用经典的I/O复用+线程池模型，之所以引入线程池，主要就是为了在业务执行的过程中，不会因为临时创建和销毁线程，造成系统性能降低，因为线程的创建和销毁是很耗费性能的，所以线程池就是在业务使用之前，先创建一组固定数量的线程，等待事件发生，当有SQL请求到达MySQL Server的时候，在线程池中取一个线程来执行该SQL请求就可以了，执行完成后，不销毁线程，而是把线程再归还到线程池中，等待下一次任务的处理（MySQL会根据连接量，自动加大线程池的数量）。

可以根据服务器的配置，为其设置相适应的线程缓存数量。



### 并发连接数量和超时时间


MySQL Server作为一个服务器，可以设置客户端的最大连接量和连接超时时间，如果数据库连接统计数量比较大，这两个参数的值需要设置大一些。

```sql
mysql> SHOW VARIABLES LIKE '%connect%';
+-----------------------------------------------+-----------------+
| Variable_name                                 | Value           |
+-----------------------------------------------+-----------------+
| character_set_connection                      | utf8            |
| collation_connection                          | utf8_general_ci |
| connect_timeout                               | 10              |
| disconnect_on_expired_password                | ON              |
| init_connect                                  |                 |
| max_connect_errors                            | 100             |
| max_connections                               | 151             |
| max_user_connections                          | 0               |
| performance_schema_session_connect_attrs_size | 512             |
+-----------------------------------------------+-----------------+
```

在配置文件（my.cnf或my.ini）最下面，添加配置：max_connections=2000，然后重启MySQLServer，设置生效。

MySQL Server对于超时未通信的连接，进行主动关闭操作。设置超时时间，超过设置时间没有请求就主动断开，单位是秒，在配置文件中添加配置：wait_timeout = 600。

```sql
mysql> SHOW GLOBAL VARIABLES LIKE '%timeout%';
+-----------------------------+----------+
| Variable_name               | Value    |
+-----------------------------+----------+
| connect_timeout             | 10       |
| delayed_insert_timeout      | 300      |
| have_statement_timeout      | YES      |
| innodb_flush_log_at_timeout | 1        |
| innodb_lock_wait_timeout    | 50       |
| innodb_rollback_on_timeout  | OFF      |
| interactive_timeout         | 28800    |
| lock_wait_timeout           | 31536000 |
| net_read_timeout            | 30       |
| net_write_timeout           | 60       |
| rpl_stop_slave_timeout      | 31536000 |
| slave_net_timeout           | 60       |
| wait_timeout                | 28800    |
+-----------------------------+----------+
```

