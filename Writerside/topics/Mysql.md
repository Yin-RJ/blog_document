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

默认情况下，InnoDB中有一个重做日志文件组(redo log group)，其中有两个redo log文件组成，在redo log 
group中，两个redo log 文件的大小是固定且一致的，redo log group以循环写的方式工作，从头开始写，写到末尾就又回到开头，相当于一个环形


### 归档日志(binlog)
存储在Server层的日志，主要用于数据备份和主从复制

Mysql在完成一条数据更新操作之后，Server层还会生成一条binlog，等之后事务提交的时候会将该事务执行过程中产生的所有binlog
统一写到binlog文件。

> binlog文件是记录了所有数据库表结构变更和表数据修改的日志，不会记录查询类的操作

binlog有3中格式类型：
1. STATEMENT(默认格式)，每一条修改数据的SQL都会被记录到binlog中，主从复制的时候再根据SQL语句重现。动态函数会导致主从库的数据不一致
2. ROW模式，记录行数据最终被修改成了什么样，这种模式下所有的update语句都会被记录，可能会导致binlog过大
3. MIXED模式，包含了ROW和STATEMENT两种模式，根据不同的情况使用不同的模式

在事务提交的时候，执行器把binlog cache里的完整事务写入到binlog文件中并清空binlog cache
- sync_binlog=0，每次提交事务只写入到page cache，由操作系统决定什么时候刷盘
- sync_binlog=1，每次提交事务后写入到page cache，立即刷盘
- sync_binlog=N，每次提交事务后写入到page cache，积累N个事务之后刷盘

### binlog与redo log的区别
1. 适用对象不同
2. 文件格式不同
3. 写入方式不同
4. 用途不同

### 两阶段提交
事务提交之后，redo log和binlog都要持久化到磁盘，但是这是两个独立的逻辑，可能会发生两份日志不一致的情况。

> 两阶段提交把单个事务的提交拆分成了两个阶段，分别是准备阶段(Prepare)和提交阶段(Commit)，每个阶段都由协调者(Coordinator)
> 和参与者(Participant)共同完成。

binlog作为协调者，存储引擎作为参与者，当客户端执行commit语句或者自动提交的情况下，Mysql内部开启一个XA事务，分两阶段来完成XA事务的提交
![mysql_xa.png](mysql_xa.png)

1. prepare阶段，将XID（内部XA事务的ID）写入到redo log，同时将redo log对应的事务状态设置为prepare，然后将redo 
   log持久化到磁盘
2. commit阶段，把XID写入到binlog，然后将binlog持久化到磁盘，接着调用引擎的提交事务接口，将redo 
   log的状态设置为commit，此时该状态并不需要持久化到磁盘，只需要写到page cache，因为只要binlog写磁盘成功，就算redo 
   log的状态还是prepare也没有关系，一样会被认为事务已经执行成功

> 当Mysql重启之后会按顺序扫描redo log，遇到prepare状态的redo 
> log，就用该XID去binlog中查看，如果binlog中没有，那么就表示binlog没有刷盘，回滚该事务；如果binlog
> 中存在，即以及完成了刷盘，此时可以提交事务

两阶段提交存在的问题：
1. IO次数会很高，每个事务提交过程中，至少会调用两次刷盘操作（binlog和redo log各一次）
2. 锁竞争激烈，通过锁保证事务的顺序性

### 组提交
> Mysql引入了binlog组提交机制，当有多个事务提交的时候，会将多个binlog刷盘操作合并成一个，从而减少IO的次数

prepare阶段不变，将commit阶段进行拆分：
1. flush阶段，多个事务按进入的顺序将binlog从cache写入page cache，不刷盘
2. sync阶段，对binlog文件进行刷盘操作
3. commit阶段，各个事务按顺序做InnoDB commit操作

每个阶段都有一个队列，每个阶段有锁保护，保证事务写入的顺序，第一个进入队列的事务会成为leader，领导队列中的所有事务，锁的粒度减小，多个阶段可以并发执行

Mysql5.7以后增加了redo log组提交，在prepare阶段不再让事务各自执行redo 
log刷盘操作，而是推迟到组提交的flush阶段，也就是prepare阶段融合在了flush阶段，这个优化是将redo 
log的刷盘延迟到了flush阶段中，sync阶段之前。

## 主从复制
1. 写入binlog，主库写binlog日志，提交事务，并更新本地存储数据
2. 同步binlog，把binlog复制到所有的从库上，从库会创建一个专门的IO线程，连接主库的log 
   dump线程来接收主库的binlog，再把binlog信息写入relay log的中继日志中，再返回给主库复制成功的响应
3. 回放binlog，从库会创建一个用于回放binlog的线程，去读relay 
   log中继日志，然后回放binlog更新存储引擎中的数据，最终实现主从的数据一致性

### 复制模型
1. 同步复制
2. 异步复制（默认）
3. 半同步复制