# 【MySQL】27 - binlog数据恢复


## 使用binlog

默认binlog文件位置  
```shell
[xushun@localhost ~]$ sudo ls /var/lib/mysql
auto.cnf	 ib_logfile0	     mysql-bin.000003	 school
ca-key.pem	 ib_logfile1	     mysql-bin.index	 server-cert.pem
ca.pem		 ibtmp1		     mysql.sock		 server-key.pem
client-cert.pem  localhost-slow.log  mysql.sock.lock	 sys
client-key.pem	 mysql		     performance_schema
ib_buffer_pool	 mysql-bin.000001    private_key.pem
ibdata1		 mysql-bin.000002    public_key.pem
```

查看和刷新binlog，flush命令会生成新binlog  
```sql
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       154 |
+------------------+-----------+
2 rows in set (0.00 sec)

mysql> flush logs;
Query OK, 0 rows affected (0.00 sec)

mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       201 |
| mysql-bin.000003 |       154 |
+------------------+-----------+
3 rows in set (0.00 sec)
```


测试日志是否生效，修改数据后，新生成的binlog大小变大了：  
```sql
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       201 |
| mysql-bin.000003 |       154 |
+------------------+-----------+
3 rows in set (0.00 sec)

mysql> use school;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
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

mysql> insert into user(name,age,sex) values('eee',25,'w');
Query OK, 1 row affected (0.00 sec)

mysql> update user set age=18 where name='ccc';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       201 |
| mysql-bin.000003 |       710 |
+------------------+-----------+
3 rows in set (0.00 sec)
```

## mysqlbinlog工具 恢复数据

使用`mysqlbinlog`查看binlog的内容。

```sql
[root@localhost mysql]# mysqlbinlog --no-defaults --database=school --base64-output=decode-rows -v --start-datetime='2022-11-14 00:00:00' --stop-datetime='2022-11-14 17:00:00' mysql-bin.000003 | more
WARNING: The option --database has been used. It may filter parts of transactions, but will include the GTIDs in any case. If you want to exclude or include transactions, you should use the options --exclude-gtids or --include-gtids, respectively, instead.
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#221114  0:15:11 server id 1  end_log_pos 123 CRC32 0xc5fd429d 	Start: binlog v 
4, server v 5.7.40-log created 221114  0:15:11
# Warning: this binlog is either in use or was not closed properly.
# at 123
#221114  0:15:11 server id 1  end_log_pos 154 CRC32 0x170d6bda 	Previous-GTIDs
# [empty]
# at 154
#221114  0:18:26 server id 1  end_log_pos 219 CRC32 0x4b0ed958 	Anonymous_GTID	
last_committed=0	sequence_number=1	rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#221114  0:18:26 server id 1  end_log_pos 293 CRC32 0xbf79bbe5 	Query	thread_i
d=2	exec_time=0	error_code=0
SET TIMESTAMP=1668413906/*!*/;
SET @@session.pseudo_thread_id=2/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.uniq
ue_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
--More--
```
往下翻，可以找到刚刚进行的insert和update操作。



## binlog数据恢复 示例


创建测试用的数据库表：  
```sql
mysql> create database mytest;
Query OK, 1 row affected (0.00 sec)

mysql> use mytest;
Database changed
mysql> create table user(id int, name varchar(20));
Query OK, 0 rows affected (0.01 sec)

mysql> insert into user values(1,'aaa'),(2,'bbb'),(3,'ccc');
Query OK, 3 rows affected (0.00 sec)
Records: 3  Duplicates: 0  Warnings: 0

mysql> select * from user;
+------+------+
| id   | name |
+------+------+
|    1 | aaa  |
|    2 | bbb  |
|    3 | ccc  |
+------+------+
3 rows in set (0.00 sec)
```

假设这个库被误删了：  
```sql
mysql> drop database mytest;
Query OK, 1 row affected (0.01 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| school             |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```

刷新一下binlog，恢复数据的操作存在04文件中  
```sql
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       201 |
| mysql-bin.000003 |      1508 |
+------------------+-----------+
3 rows in set (0.00 sec)

mysql> flush logs;
Query OK, 0 rows affected (0.01 sec)

mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       201 |
| mysql-bin.000003 |      1555 |
| mysql-bin.000004 |       154 |
+------------------+-----------+
4 rows in set (0.00 sec)
```

我们已经知道了，要恢复的数据操作，在000003文件中：  
```shell
[root@localhost mysql]# mysqlbinlog mysql-bin.000003
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#221114  0:15:11 server id 1  end_log_pos 123 CRC32 0xc5fd429d 	Start: binlog v 4, server v 5.7.40-log created 221114  0:15:11
BINLOG '
D/lxYw8BAAAAdwAAAHsAAAAAAAQANS43LjQwLWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AZ1C/cU=
'/*!*/;
# at 123
#221114  0:15:11 server id 1  end_log_pos 154 CRC32 0x170d6bda 	Previous-GTIDs
# [empty]
# at 154
#221114  0:18:26 server id 1  end_log_pos 219 CRC32 0x4b0ed958 	Anonymous_GTID	last_committed=0	sequence_number=1	rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#221114  0:18:26 server id 1  end_log_pos 293 CRC32 0xbf79bbe5 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1668413906/*!*/;
SET @@session.pseudo_thread_id=2/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 293
#221114  0:18:26 server id 1  end_log_pos 349 CRC32 0x0e309230 	Table_map: `school`.`user` mapped to number 112
# at 349
#221114  0:18:26 server id 1  end_log_pos 395 CRC32 0x23ba6fff 	Write_rows: table id 112 flags: STMT_END_F

BINLOG '
0vlxYxMBAAAAOAAAAF0BAAAAAHAAAAAAAAEABnNjaG9vbAAEdXNlcgAEAw8B/gSWAPcBADCSMA4=
0vlxYx4BAAAALgAAAIsBAAAAAHAAAAAAAAEAAgAE//AYAAAAA2VlZRkC/2+6Iw==
'/*!*/;
# at 395
#221114  0:18:26 server id 1  end_log_pos 426 CRC32 0xed858b13 	Xid = 17
COMMIT/*!*/;
# at 426
#221114  0:18:48 server id 1  end_log_pos 491 CRC32 0x1fc7a1d0 	Anonymous_GTID	last_committed=1	sequence_number=2	rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 491
#221114  0:18:48 server id 1  end_log_pos 565 CRC32 0xbd58af08 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1668413928/*!*/;
BEGIN
/*!*/;
# at 565
#221114  0:18:48 server id 1  end_log_pos 621 CRC32 0x42901722 	Table_map: `school`.`user` mapped to number 112
# at 621
#221114  0:18:48 server id 1  end_log_pos 679 CRC32 0xab2df0ff 	Update_rows: table id 112 flags: STMT_END_F

BINLOG '
6PlxYxMBAAAAOAAAAG0CAAAAAHAAAAAAAAEABnNjaG9vbAAEdXNlcgAEAw8B/gSWAPcBACIXkEI=
6PlxYx8BAAAAOgAAAKcCAAAAAHAAAAAAAAEAAgAE///wFwAAAANjY2MPAvAXAAAAA2NjYxIC//At
qw==
'/*!*/;
# at 679
#221114  0:18:48 server id 1  end_log_pos 710 CRC32 0x85efe3cd 	Xid = 18
COMMIT/*!*/;
# at 710
#221114  0:27:19 server id 1  end_log_pos 775 CRC32 0x9b6548cf 	Anonymous_GTID	last_committed=2	sequence_number=3	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 775
#221114  0:27:19 server id 1  end_log_pos 875 CRC32 0xdaccd1f1 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1668414439/*!*/;
create database mytest
/*!*/;
# at 875
#221114  0:28:19 server id 1  end_log_pos 940 CRC32 0x59f5f533 	Anonymous_GTID	last_committed=3	sequence_number=4	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 940
#221114  0:28:19 server id 1  end_log_pos 1061 CRC32 0xaa6fe273 	Query	thread_id=2	exec_time=0	error_code=0
use `mytest`/*!*/;
SET TIMESTAMP=1668414499/*!*/;
create table user(id int, name varchar(20))
/*!*/;
# at 1061
#221114  0:28:52 server id 1  end_log_pos 1126 CRC32 0x4e0954c8 	Anonymous_GTID	last_committed=4	sequence_number=5	rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1126
#221114  0:28:52 server id 1  end_log_pos 1200 CRC32 0xa59fd4b4 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1668414532/*!*/;
BEGIN
/*!*/;
# at 1200
#221114  0:28:52 server id 1  end_log_pos 1252 CRC32 0x8862e282 	Table_map: `mytest`.`user` mapped to number 114
# at 1252
#221114  0:28:52 server id 1  end_log_pos 1314 CRC32 0xb9a20212 	Write_rows: table id 114 flags: STMT_END_F

BINLOG '
RPxxYxMBAAAANAAAAOQEAAAAAHIAAAAAAAEABm15dGVzdAAEdXNlcgACAw8CPAADguJiiA==
RPxxYx4BAAAAPgAAACIFAAAAAHIAAAAAAAEAAgAC//wBAAAAA2FhYfwCAAAAA2JiYvwDAAAAA2Nj
YxICork=
'/*!*/;
# at 1314
#221114  0:28:52 server id 1  end_log_pos 1345 CRC32 0x4df830db 	Xid = 27
COMMIT/*!*/;
# at 1345
#221114  0:30:26 server id 1  end_log_pos 1410 CRC32 0x71992700 	Anonymous_GTID	last_committed=5	sequence_number=6	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1410
#221114  0:30:26 server id 1  end_log_pos 1508 CRC32 0xb3c31c9b 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1668414626/*!*/;
drop database mytest
/*!*/;
# at 1508
#221114  0:37:00 server id 1  end_log_pos 1555 CRC32 0x7c5eeaf5 	Rotate to mysql-bin.000004  pos: 4
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
[root@localhost mysql]# 
```
在这个文件中，找到创建库和删除库的位置，775，1410。




恢复，把创建库表到删除库的操作拿出来再执行一遍：  
```shell
[root@localhost mysql]# mysqlbinlog --start-position=775 --stop-position=1410 mysql-bin.000003 | mysql -u root -p
Enter password: 
[root@localhost mysql]# 
```




查看恢复的数据，成功恢复：  
```sql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| mytest             |
| performance_schema |
| school             |
| sys                |
+--------------------+
6 rows in set (0.00 sec)

mysql> use mytest;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from user;
+------+------+
| id   | name |
+------+------+
|    1 | aaa  |
|    2 | bbb  |
|    3 | ccc  |
+------+------+
3 rows in set (0.00 sec)
```


当然，不通过position来恢复，可以通过datetime来恢复。
