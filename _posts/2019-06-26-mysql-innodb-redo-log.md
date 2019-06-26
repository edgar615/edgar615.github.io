---
layout: post
title: InnoDB redo日志
date: 2019-06-26
categories:
    - MySQL
comments: true
permalink: innodb-redo-log.html
---

Redo Log又被称为WAL ( Write Ahead Log)，是InnoDB存储引擎实现事务持久性的关键

在InnoDB存储引擎中，事务执行过程被分割成一个个MTR (Mini TRansaction)，每个MTR在执行过程中对数据页的更改会产生对应的日志，这个日志就是Redo Log。事务在提交时，只要保证Redo Log被持久化，就可以保证事务的持久化。

由于Redo Log在持久化过程中顺序写文件的特性，使得持久化Redo Log的代价要远远小于持久化数据页，因此通常情况下，数据页的持久化要远落后于Redo Log

**注意：mysql不是每次数据更改都立刻写到磁盘，而是会先将修改后的结果暂存在内存中（change buffer）,当一段时间后，再一次性将多个修改写到磁盘上，减少磁盘io成本，同时提高操作速度。**

![](/assets/images/posts/redo-log/redo-log-1.png)

如图所示，在同一个事务中，每当数据库进行修改数据操作时，将修改结果更新到内存后，会在redo log添加一行记录记录“需要在哪个数据页上做什么修改”，并将该记录状态置为prepare，等到commit提交事务后，会将此次事务中在redo log添加的记录的状态都置为commit状态，之后将修改落盘时，会将redo log中状态为commit的记录的修改都写入磁盘。


**redo log记录方式**
redolog的大小是固定的，在mysql中可以通过修改配置参数`innodb_log_files_in_group`和`innodb_log_file_size`配置日志文件数量和每个日志文件大小，redolog采用循环写的方式记录，当写到结尾时，会回到开头循环写日志。

![](/assets/images/posts/redo-log/redo-log-2.png)

write pos表示日志当前记录的位置，当ib_logfile_4写满后，会从ib_logfile_1从头开始记录；check point表示将日志记录的修改写进磁盘，完成数据落盘，数据落盘后checkpoint会将日志上的相关记录擦除掉，即write pos->checkpoint之间的部分是redo log空着的部分，用于记录新的记录，checkpoint->write pos之间是redo log待落盘的数据修改记录。当writepos追上checkpoint时，得先停下记录，先推动checkpoint向前移动，空出位置记录新的日志。
有了redo log，当数据库发生宕机重启后，可通过redo log将未落盘的数据恢复，即保证已经提交的事务记录不会丢失。

> 上述内容抄自 https://www.jianshu.com/p/4bcfffb27ed5

# redo log和binlog

- redo log是在MySQL的InnoDB引擎层产生，而binlog则是在MySQL的上层产生，它不仅针对InnoDB引擎，其他任何引擎对于数据库的更改都会产生binlog。
- 两种日志记录的内容形式不同，binlog是一种逻辑日志，其记录的是对应的SQL语句。而redo log则是记录的物理格式日志，其记录的是对于每个页的修改。
- 两种日志记录写入磁盘的时间点不同，binlog只在事务提交完成后一次性写入，而redo log在上面也说了是在事务进行中不断被写入，这表现为日志并不是随事务提交的顺序进行写入的。

# 原理

每个Redo Log都有一个对应的序号LSN (Log Sequence Number)，同时数据页上也会记录修改了该数据页的Redo Log的LSN，当数据页持久化到磁盘上时，就不再需要这个数据页记录的LSN之前的Redo日志，这个LSN被称作**Checkpoint**。

当做故障恢复的时候，只需要将Checkpoint之后的Redo Log重新应用一遍，便可得到实例Crash之前未持久化的全部数据页。

InnoDB存储引擎在内存中维护了一个全局的Redo Log Buffer用以缓存对Redo Log的修改，mtr在提交的时候，会将mtr执行过程中产生的本地日志copy到全局Redo Log Buffer中，并将mtr执行过程中修改的数据页（被称做脏页dirty page）加入到一个全局的队列中flush list。

InnoDB存储引擎会根据不同的策略将Redo Log Buffer中的日志落盘，或将flush list中的脏页刷盘并推进Checkpoint。

在脏页落盘以及Checkpoint推进的过程中，需要严格保证Redo日志先落盘再刷脏页的顺序，在MySQL 8之前，InnoDB存储引擎严格的保证MTR写入Redo Log Buffer的顺序是按照LSN递增的顺序，以及flush list中的脏页按LSN递增顺序排序。

在多线程并发写入Redo Log Buffer及flush list时，这一约束是通过两个全局锁log_sys_t::mutex和log_sys_t::flush_order_mutex实现的。这引入了一个严重的锁竞争点，为了优化这个问题MySQL8.0对日志系统进行了重新设计，将整个模块变成了lock-free的模式

- 拷贝到buffer: 每个mini transaction将自己的本地日志拷贝到全局Buffer中
- 写磁盘：包括写磁盘和调用fsync进行持久化
- 事务提交：当事务undo被标记为prepare(如果binlog打开) 或者commit时，需要确保日志被刷到磁盘，以确保事务的持久性
- Checkpoint: 定期对日志做checkpoint，减少崩溃恢复时日志的应用量

# 参数
## innodb_flush_log_at_trx_commit
`innodb_flush_log_at_trx_commit`参数可以控制将 redo buffer 中的更新记录写入到日志文件以及将日志文件数据刷新到磁盘的操作时机。通过调整这个参数，可以在性能和数据安全之间做取舍。

- 0：在事务提交时，innodb 不会立即触发将缓存日志写到磁盘文件的操作，而是每秒触发一次缓存日志回写磁盘操作，并调用系统函数 fsync 刷新 IO 缓存。这种方式效率最高，也最不安全。
- 1：在每个事务提交时，innodb 立即将缓存中的 redo 日志回写到日志文件，并调用 fsync 刷新 IO 缓存。
- 2：在每个事务提交时，innodb 立即将缓存中的 redo 日志回写到日志文件，但并不马上调用 fsync 来刷新 IO 缓存，而是每秒只做一次磁盘IO 缓存刷新操作。只要操作系统不发生崩溃，数据就不会丢失，这种方式是对性能和数据安全的折中，其性能和数据安全性介于其他两种方式之间。

innodb_flush_log_at_trx_commit 参数的默认值是 1，即每个事务提交时都会从 log buffer 写更新记录到日志文件，而且会实际刷新磁盘缓存，显然，这完全能满足事务的持久化要求，是最安全的，但这样会有较大的性能损失。

在某些需要尽量提高性能，并且可以容忍在数据库崩溃时丢失小部分数据，那么通过将参数 `innodb_flush_log_at_trx_commit`设置成 0 或 2 都能明显减少日志同步 IO，加快事务提交，从而改善性能。

## innodb_log_file_size
可以通过下面的方式来计算 innodb 每小时产生的日志量并估算合适的 innodb_log_file_size 的值：

> 在Linux的MySQL Client下执行，pager的用法可以查询相应文档

```
mysql> pager grep -i "log sequence number"
PAGER set to 'grep -i "log sequence number"'
mysql> show engine innodb status\G select sleep(60);show engine innodb status\G
Log sequence number 20312242784
1 row in set (0.00 sec)

1 row in set (1 min 0.00 sec)

Log sequence number 20312460608
1 row in set (0.00 sec)
```

一分钟日志大小 = 20312460608 - 20312242784 =217824 byte = 0.2077 Mb

也就是一分钟时间产生了 0.2077 Mb 日志，那么推算一小时的日常量 = 0.2077 Mb * 60 =12.464Mb。

而日志的切换频率20 - 30分钟一次最好，那么单个日志文件大小 = 12.464Mb/2/日志文件个数 = 3.116Mb，也就是一个日志文件大小为3.116Mb。
```
innodb_log_file_size=3.116 M
```

> 建议在业务高分期做计算、或者采集一个比较长时间的日常量，计算的结果会更加准确。 
> 如果你计算得到了一个GB大小的值，那么可能是当时正在插入大量数据的。

# innodb_log_buffer_size
Innodb_log_buffer_size决定InnoDB重做日志缓存池的大小，默认值是8M。对于可能产生大量更新记录的大事务，增加innodb_log_buffer_size的大小，可以避免InnoDB在事务提交前就执行不必要的日志写入磁盘操作。因此，对于会在一个事务中更新、插入、或删除大量记录的应用，我们可以通过增大innodb_log_buffer_size来减少日志写磁盘操作，从而提高事务处理的性能。

# 参考资料

https://segmentfault.com/a/1190000014884228

https://www.jianshu.com/p/d829df873332

https://segmentfault.com/a/1190000011322006

http://www.sohu.com/a/290417712_411876

https://www.jianshu.com/p/4bcfffb27ed5
