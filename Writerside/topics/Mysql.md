# Mysql

## Buffer Pool
InnoDB通过Buffer Pool缓冲池来提高读写性能。
- 在读取数据的时候，如果Buffer Pool中存在，则直接将Buffer Pool中的数据返回
- 在更新数据的时候，如果Buffer Pool中存在，则直接修改Buffer 
  Pool中数据所在页，然后将其所在页设置为脏页，为了减少磁盘IO，不会直接将脏页写会磁盘，后台会选择一个时间使用新的线程来把脏页的数据写入磁盘

![buffer_pool.png](mysql_buffer_pool.png)

InnoDB会将数据划分成若干个页，以页作为磁盘和内存交互的最小单位，一个页的默认大小是16KB，Buffer Pool同样按照页来进行划分。

Mysql启动的时候，InnoDB会为Buffer 
Pool申请一块连续的内存空间，然后按照默认16KB
把内存空间分成一个个的页，也就是缓存页，一开始缓存页是空的，随着数据库的运行，才会有磁盘上的数据被写入到缓存页中

## 日志系统

> undo log记录的是一次事务开始前的数据状态，记录的是更新之前的值
> redo log记录的是一次事务结束后的数据状态，记录的是更新之后的值

### 回滚日志(undo log)
存储在InnoDB引擎层的日志，实现了事务中的原子性，主要用于事务回滚和MVCC

> undo log是一种用于撤销回退的日志，在事务没有提交之前，Mysql会记录更新前的数据到undo 
> log中，当事务发生回滚的时候，可以利用undo log来进行回滚

![undo_log.png](mysql_undo_log.png)

- 插入一条数据的时候，记录这条数据的主键，如果需要回滚，则将这条主键数据删除
- 删除一条数据的时候，记录这条数据的所有内容，如果需要回滚，将这条数据插入即可
- 更新一条数据的时候，记录这条数据的旧值，如果需要回滚，更新回旧值即可

一条记录的每一次更新操作产生的undo log格式都会有一个roll_pointer指针和一个trx_id事务id
- 通过trx_id事务id可以知道这次的修改操作是哪个事务做的
- 通过roll_pointer指针可以将整个undo log进行串联，形成一个版本链

作用：
1. 实现事务回滚，保证事务的原子性
2. 实现MVCC的关键因素之一。在执行快照读的时候可以基于undo log形成的版本链知道满足其可见性的版本数据

> undo log的刷盘和数据页的刷盘机制一致，依靠redo log保证持久性。
> buffer pool 中有 undo 页，对 undo 页的修改也都会记录到 redo log。
> redo log 会每秒刷盘，提交事务时也会刷盘，数据页和 undo 页都是靠这个机制保证持久化的。


### 重做日志(redo log)
存储在InnoDB引擎层的日志，实现了事务中的持久性，主要用于掉电故障恢复

> redo log是物理日志，记录了某个数据页做了哪些修改

为了方式内存缓存数据丢失的问题，当有一条数据需要更新的时候，InnoDB就会先更新内存并将该缓存页标记为脏页，然后将本次对这页的操作记录在redo 
log中，此时算是更新完成。后续再由后台线程将脏页的数据写回到磁盘

> WAL(Write-Ahead Logging)：写操作不是直接写到磁盘，而是先写到日志中，然后再在合适的时间写入到磁盘中

![redo_log.png](mysql_redo_log.png)

作用：
1. 实现事务的持久性，让Mysql具有crash-safe的能力，能够保证Mysql奔溃之后，重启之后数据不丢失
2. 将随机写变成顺序写，提高写磁盘IO的效率（redo 
   log是追加写，所以每次是顺序写入磁盘，而脏页数据写回磁盘需要先找到数据的位置，然后再进行写入，所以是随机写，顺序写的效率大于随机写）

redo log也存在一个redo log buffer，而redo log 
buffer（默认16MB）也是在内存中的，所以也是先写入到内存，在以下情况下会将buffer中的内容刷新到磁盘中：
- Mysql正常关闭时
- 当redo log buffer中的使用量大于其一半，会触发落盘操作
- InnoDB的后台线程，每隔1秒将redo log buffer中的内容写入磁盘
- 每次事务提交的时候都将redo log buffer中的数据写入到磁盘（由`innodb_flush_log_at_trx_commit`参数控制）

> `innodb_flush_log_at_trx_commit`参数：
> 
> 1. 取值为0，每次事务提交后，不会主动触发刷盘操作
> 2. 取值为1，每次事务提交后，将buffer中的内容直接写入磁盘
> 3. 取值为2，每次事务提交后，将buffer中的内容写入到Page Cache，然后依靠InnoDB每隔一秒执行的写入磁盘线程将数据从Page 
     > Cache写入到磁盘中


### 归档日志(binlog)
存储在Server层的日志，主要用于数据备份和主从复制