# 【MySQL】26 - MySQL 日志系统



## MySQL 日志系统

本章介绍MySQL的日志系统，主要介绍mysql server层的日志系统，之前讲的undo log和redo log是在存储引擎层的事务日志。如图。

![](/post_images/posts/Database/MySQL/MySQL日志系统.jpg "MySQL日志系统")


查看日志相关全局变量：  
```sql
+----------------------------------------+---------------------+
| Variable_name                          | Value               |
+----------------------------------------+---------------------+
| log_bin                                | OFF                 |
| log_bin_basename                       |                     |
| log_bin_index                          |                     |
| log_bin_trust_function_creators        | OFF                 |
| log_bin_use_v1_row_events              | OFF                 |
| log_builtin_as_identified_by_password  | OFF                 |
| log_error                              | /var/log/mysqld.log |
| log_error_verbosity                    | 3                   |
| log_output                             | FILE                |
| log_queries_not_using_indexes          | OFF                 |
| log_slave_updates                      | OFF                 |
| log_slow_admin_statements              | OFF                 |
| log_slow_slave_statements              | OFF                 |
| log_statements_unsafe_for_binlog       | ON                  |
| log_syslog                             | OFF                 |
| log_syslog_facility                    | daemon              |
| log_syslog_include_pid                 | ON                  |
| log_syslog_tag                         |                     |
| log_throttle_queries_not_using_indexes | 0                   |
| log_timestamps                         | UTC                 |
| log_warnings                           | 2                   |
+----------------------------------------+---------------------+
```

设置配置，可以在终端中set，但是这种方法只能针对当前的session，在配置文件`my.ini`中添加并重启(`service mysqld restart`)才能持久生效。  
```
#Enter a name for the error log file. Otherwise a default name will be used.
log-error=err.log
#Enter a name for the query log file. Otherwise a default name will be used.
#log=
#Enter a name for the slow query log file. Otherwise a default name will be
used.
#log-slow-queries=
#Enter a name for the update log file. Otherwise a default name will be used.
#log-update=
#Enter a name for the binary log. Otherwise a default name will be used.
#log-bin=
```

## 错误日志

错误日志是 MySQL 中最重要的日志之一，它记录了当 mysqld 启动和停止时，以及服务器在运行过程中发生任何严重错误时的相关信息。当数据库出现任何故障导致无法正常使用时，可以首先查看此日志。

mysqld 使用错误日志名 host_name.err(host_name 为主机名) 并默认在参数 DATADIR(数据目录)指定的目录中写入日志文件。


## 查询日志

查询日志记录了客户端的所有语句。由于上线项目sql特别多，开启查询日志IO太多导致MySQL效率低，只有在调试时才开启，比如通过查看sql发现热点数据进行缓存。

```sql
mysql> show global variables like "%genera%";
+----------------------------------------+------------------------------+
| Variable_name                          | Value                        |
+----------------------------------------+------------------------------+
| auto_generate_certs                    | ON                           |
| general_log                            | OFF                          |
| general_log_file                       | /var/lib/mysql/localhost.log |
| sha256_password_auto_generate_rsa_keys | ON                           |
+----------------------------------------+------------------------------+
```

## 慢查询日志

[慢查询日志](https://xushun1221.github.io/2022/mysql17-sql%E5%92%8C%E7%B4%A2%E5%BC%95%E4%BC%98%E5%8C%96%E6%85%A2%E6%9F%A5%E8%AF%A2%E6%97%A5%E5%BF%97/)


## redo log & undo log


[redo log](https://xushun1221.github.io/2022/mysql24-redo-log-%E9%87%8D%E5%81%9A%E6%97%A5%E5%BF%97/)

[undo log](https://xushun1221.github.io/2022/mysql22-mvcc%E5%92%8Cundo-log/)
