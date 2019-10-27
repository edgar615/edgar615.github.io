---
layout: post
title: InnoDB架构-Log Buffer（part4）
date: 2019-10-26
categories:
    - MySQL
comments: true
permalink: innodb-log-buffer.html
---

Log Buffer(日志缓冲区)是一块内存区域用来保存要写入磁盘上的日子文件的数据。 Log Buffer的大小由`innodb_log_buffer_size`变量定义。默认大小为16MB。Log Buffer的内容会定期刷到磁盘上。大的Log Buffer让较大事务能够运行，而无需在事务提交之前将redo log中的数据写入磁盘。因此，如果有DML操作并且会影响很多行这样是事务，增加日志缓冲区的大小可以节省磁盘IO

它会存储InnoDB存储引擎层日志：redo日志和undo日志

InnoDB 在写事务日志的时候，为了提高性能，先将信息写入 Log Buffer 中，当满足 `innodb_flush_log_trx_commit`参数所设置的相应条件（或者日志缓冲区写满）之后，才会将日志写到文件（或者同步到磁盘）中

- `innodb_flush_log_at_trx_commit`控制如何将缓冲区的内容写入到日志文件。
- `innodb_flush_log_at_timeout`控制缓存写到redo log文件的频率。

# 参数
## innodb_log_buffer_size
该参数就是用来设置 InnoDB 的 Log Buffer 大小，系统默认值为 16MB，主要作用就是缓冲 redo log 数据，增加缓存可以使大事务在提交前不用写入磁盘，从而提高写 IO 性能。因此对于会在一个事务中更新、插入、或删除大量记录的应用，我们可以通过增大`innodb_log_buffer_size`来减少日志写磁盘操作，从而提高事务处理的性能

可以通过系统状态参数，查看性能统计数据来分析 Log 的使用情况：

```
mysql> SHOW STATUS LIKE 'innodb_log%';
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| Innodb_log_waits          | 0     |  由于缓存过小，导致事务必须等待的次数
| Innodb_log_write_requests | 30    |  日志写请求数
| Innodb_log_writes         | 21    |  向日志文件的物理写次数
+---------------------------+-------+
3 rows in set (0.03 sec)
```

## innodb_flush_log_at_trx_commit

`innodb_flush_log_at_trx_commit`参数可以控制将 log buffer 中的更新记录写入到日志文件以及将日志文件数据刷新到磁盘的操作时机。通过调整这个参数，可以在性能和数据安全之间做取舍。

![](/assets/images/posts/log-buffer/log-buffer-1.png)

- 0：在事务提交时，innodb 不会立即触发将缓存日志写到磁盘文件的操作，而是每秒触发一次缓存日志回写磁盘操作，并调用系统函数 fsync 刷新 IO 缓存。这种方式效率最高，也最不安全。
- 1：在每个事务提交时，innodb 立即将缓存中的 redo 日志回写到日志文件，并调用 fsync 刷新 IO 缓存。
- 2：在每个事务提交时，innodb 立即将缓存中的 redo 日志回写到日志文件，但并不马上调用 fsync 来刷新 IO 缓存，而是每秒只做一次磁盘IO 缓存刷新操作。只要操作系统不发生崩溃，数据就不会丢失，这种方式是对性能和数据安全的折中，其性能和数据安全性介于其他两种方式之间。

innodb_flush_log_at_trx_commit 参数的默认值是 1，即每个事务提交时都会从 log buffer 写更新记录到日志文件，而且会实际刷新磁盘缓存，显然，这完全能满足事务的持久化要求，是最安全的，但这样会有较大的性能损失。

在某些需要尽量提高性能，并且可以容忍在数据库崩溃时丢失小部分数据，那么通过将参数 `innodb_flush_log_at_trx_commit`设置成 0 或 2 都能明显减少日志同步 IO，加快事务提交，从而改善性能。

> 宏观上写进logfile就是写进磁盘了。但是微观上写进logfile是先写进了os cahce，然后再刷新到raid cache(前提是做了raid)最后到磁盘。 
> OS 中 write 和 fsync 是不同的操作，我们以为调用了 write 就万事大吉，数据一定到磁盘了，其实不一定，通常情况下 write 只是到了磁盘 IO 缓冲区，何时 fsync 由 OS 控制，这里通过程序强制调用来保证日志一定刷到磁盘

http://blog.itpub.net/29654823/viewspace-2143511/

http://www.mysqlab.net/knowledge/kb/detail/topic/innodb/id/6553

https://zhuanlan.zhihu.com/p/66041606

https://segmentfault.com/a/1190000011322006