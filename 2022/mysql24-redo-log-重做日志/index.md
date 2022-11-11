# 【MySQL】24 - redo log 重做日志



## redo log 重做日志

> redo log 重做日志，解决的核心问题是：数据持久性！  
> 在COMMIT提交事务成功后，无论系统崩溃、掉电，还是发生其他不可预期的问题，下次mysql server正常运行时，都能保证之前已COMMIT提交的数据能够根据redo log的记录恢复到正常状态。


redo log 示意图

![](/post_images/posts/Database/MySQL/redolog.jpg "redo log")


redo log 在事务BEGIN之后，就开始记录了，无论事务是否提交。在异常发生时（如数据持久化过程中掉电），InnoDB会使用redo log恢复到掉电前的时刻，保证数据的完整性。

InnoDB引擎修改操作数据，不会直接修改磁盘上的数据，而只是修改BufferPool中的数据。InnoDB总是先把BufferPool中的数据改变记录到redo log中，用来进行数据恢复。优先记录redo log，然后再慢慢地把BufferPool中的脏数据刷到磁盘上。

redo log也不会直接进行磁盘IO，而是将日志记录在Log Buffer中。在事务COMMIT提交时，InnoDB会将LogBuffer中redo log重做日志写入磁盘，写入成功的话，就会记录COMMITTED已提交状态。如果没写完，状态就是PREPARE准备状态。在这个过程中，也有可能发生异常，导致redo log没有刷到磁盘上，这种情况视作事务没有提交成功，在下一次启动mysql server时，就不会保证这个事务的恢复。

undo log回滚日志的内容也要记录在redo log中，如果事务回滚过程中系统异常，下一次启动时，就可以根据redo log完成回滚操作。

注意，BufferPool里面的脏数据，不会在事务COMMIT提交的时候刷入磁盘，有专门的线程在合适的时机将其刷入磁盘。

> innodb_log_buffer_size默认大小为16M，就是redo log的缓冲区大小。  
> innodb_log_group_home_dir指定的目录下的两文件`ib_logfile0  ib_logfile1`就是重做日志文件。
