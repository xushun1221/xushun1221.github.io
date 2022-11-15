# 【MySQL】30 - 主从复制


在实际生产环境中，如果对mysql数据库的读和写都在一台数据库服务器中操作，无论是在安全性、高可用性，还是高并发等各个方面都是不能满足实际需求的，一般要通过**主从复制**的方式来同步数据，再通过**读写分离**来提升数据库的并发负载能力。

1. 主从复制：热备份&容灾&高可用
2. 读写分离：支持更大的并发


## 主从复制  原理


![](/post_images/posts/Database/MySQL/主从复制.jpg "主从复制")

主从复制，主要涉及两个日志（binlog、relaylog）三个线程（master的一个线程，slave的两个线程）。

1. 主库开启**二进制日志**，将更新操作写入binlog中；
2. master主服务器创建一个**binlog转储线程**，将二进制日志发送至slave从服务器；
3. slave从服务器执行`START SLAVE`命令，创建一个IO线程，接收master的binlog并复制到**中继日志relay log**（内存中接收的binlog太多就会存到磁盘中）；  
    slave开启一个工作线程（IO线程），IO线程在master上打开一个普通连接，然后主服务器开始binlog dump process，将master的binlog向从服务器发送，如果slave已经跟上master，该线程会睡眠等待master产生新的事件，该线程将这些事件写入中继日志。
4. slave从服务器的SQL线程处理该过程的最后一步，sql线程从中继日志中读取事件，并重放其中的事件而更新slave机器的数据，使其与master的数据保持一致。  
    只要该线程与IO线程保持一致，中继日志通常会位于OS缓存中，所以中级日志的开销很小。




## 主从复制  实践配置

- master主服务器：CentOS7 虚拟机（192.168.131.130）
- slave从服务器： Win10 主机（192.168.11.164）

注意，win10发送给VMware虚拟机的数据不是从上面的网口发来的，用的是`192.168.131.1`这个虚拟网关转发的，所以从win10发送来的数据，应该是这个IP。

保证master和slave之间网络互通，且保证主库上的3306端口开放。


### 开放centos7防火墙3306端口

在防火墙中开启3306端口  
```shell
# 开启3306
[xushun@localhost ~]$ firewall-cmd --zone=public --add-port=3306/tcp --permanent
success
# 重启firewall
[xushun@localhost ~]$ firewall-cmd --reload
success
# 查看开放的端口
[xushun@localhost ~]$ firewall-cmd --list-ports
3306/tcp
```

或者直接关闭防火墙，这样很危险。  
```shell
systemctl start firewalld.service
systemctl stop firewalld.service
systemctl enable firewalld.service
systemctl disable firewalld.service
systemctl status firewalld.service
```

在Win10的powershell中测试端口是否可以连接，无任何返回值说明连接成功：  
```shell
PS C:\Users\xushun> $tcp = new-object Net.Sockets.TcpClient
PS C:\Users\xushun> $tcp.Connect("192.168.131.130",3306)
PS C:\Users\xushun>
```


### master配置

1，开启二进制日志binlog，在`/etc/my.cnf`中，添加如下内容并重启mysqlserver：  
```
server-id=1
expire_logs_days=7
log-bin=mysql-bin
```
`server-id=1`是全局性的唯一id，需要和从库区别开。


2，创建一个用于主从库通信用的账号，从库连接主库需要登录身份验证：  
```sql
mysql> create user 'mslave'@'192.168.131.1' identified by '111111';
Query OK, 0 rows affected (0.00 sec)

mysql> grant replication slave on *.* to 'mslave'@'192.168.131.1' identified by '111111';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```
第一句，创建账号，用户ip为`192.168.131.1`，密码`111111`，用户名`mslave`。

第二句，授予权限，`*.*`的意思是所有库的所有表都进行同步。

第三句，刷新权限。

注意，win10的ip。


3，查看binlog的文件名和position，往后进行主从复制：  
```sql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000004 |     2798 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```


### slave配置


1，配置全局唯一的server-id，重启mysql57服务。

在`C:\ProgramData\MySQL\MySQL Server 5.7\my.ini`中，将server-id改为`2`。


2，配置主库从库通信的相关配置信息：  
```sql
mysql> change master to
    -> master_host='192.168.131.130',
    -> master_port=3306,
    -> master_user='mslave',
    -> master_password='111111',
    -> master_log_file='mysql-bin.000004',
    -> master_log_pos=2798;
Query OK, 0 rows affected, 2 warnings (0.03 sec)
```

3，启动从库（IO线程、relaylog、sql线程启动）  
```sql
mysql> start slave;
Query OK, 0 rows affected (0.01 sec)
```
如果需要修改配置信息，先`stop slave;`，再修改配置，再`start slave;`

如果出现错误，先`stop slave;`，解决问题（重新设置position），或者跳过错误`set global sql_slave_skip_counter=1;`，再`start slave;`

### 查看slave信息

查看slave信息：  
```sql
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.131.130
                  Master_User: mslave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 2798
               Relay_Log_File: DESKTOP-9QUA3NG-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 2798
              Relay_Log_Space: 537
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: aa621180-49dd-11ed-8cd5-000c29ca8681
             Master_Info_File: C:\ProgramData\MySQL\MySQL Server 5.7\Data\master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```
没什么问题。

查看master和slave线程信息：  
```sql
(slave)
mysql> show processlist\G
*************************** 1. row ***************************
     Id: 3
   User: root
   Host: localhost:64874
     db: NULL
Command: Query
   Time: 0
  State: starting
   Info: show processlist
*************************** 2. row ***************************
     Id: 4
   User: system user
   Host:
     db: NULL
Command: Connect
   Time: 199
  State: Waiting for master to send event
   Info: NULL
*************************** 3. row ***************************
     Id: 5
   User: system user
   Host:
     db: NULL
Command: Connect
   Time: 199
  State: Slave has read all relay log; waiting for more updates
   Info: NULL
3 rows in set (0.00 sec)
```
