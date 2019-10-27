---
layout: post
title: MySQL日志-redo日志（part2）
date: 2019-06-26
categories:
    - MySQL
comments: true
permalink: innodb-redo-log.html
---

当数据库对数据做修改的时候，流程如下：

1. 需要把数据页从磁盘读到 buffer pool 中
2. 在 buffer pool 中进行修改，那么这个时候 buffer pool 中的数据页就与磁盘上的数据页**内容不一致**，我们称 buffer pool 的数据页为 **dirty page 脏页**。
3. 将dirty page刷新到盘（注意，**同步到磁盘文件是个随机 IO**）

> 注意：InnoDB对脏页的处理不是每次生成脏页就将脏页刷新回磁盘，这样会产生海量的IO操作，严重影响InnoDB的处理性能。

如果脏页刷新到盘期间发生非正常的 DB 服务重启，那么这些数据还在内存，并没有同步到磁盘文件中，就造成了**数据丢失**。

为了避免数据丢失的问题，MySQL提供了一个文件，当 buffer pool 中的 dirty page 变更结束后，**把相应修改记录记录到这个文件**（注意，**记录日志是顺序 IO**），那么当 DB 服务发生 crash 的情况，恢复 DB 的时候，也可以根据这个文件的记录内容，重新应用到磁盘文件，数据保持一致。这个文件就是 redo log ，**用于记录数据修改后的记录**，顺序记录。

> 顺序IO的性能比随机IO要高得多

# Redo log
Redo Log又被称为WAL ( Write Ahead Log)，是InnoDB存储引擎实现事务持久性的关键。在InnoDB存储引擎中，事务执行过程被分割成一个个MTR (Mini TRansaction)，每个MTR在执行过程中对数据页的更改会产生对应的日志，这个日志就是Redo Log。事务在提交时，只要保证Redo Log被持久化，就可以保证事务的持久化。

由于Redo Log在持久化过程中顺序写文件的特性，使得持久化Redo Log的代价要远远小于持久化数据页，因此通常情况下，数据页的持久化要远落后于Redo Log

> redo log 是物理日志而非逻辑日志，记录的是数据页的物理修改，而不是某一行或某几行修改成怎样怎样，它用来恢复提交后的物理数据页（恢复数据页，且只能恢复到最后一次提交的位置）。

![](/assets/images/posts/redo-log/redo-log-1.png)

如图所示，在同一个事务中，每当数据库进行修改数据操作时，将修改结果更新到内存后，会在redo log添加一行记录记录“需要在哪个数据页上做什么修改”，并将该记录状态置为prepare，等到commit提交事务后，会将此次事务中在redo log添加的记录的状态都置为commit状态，之后将修改落盘时，会将redo log中状态为commit的记录的修改都写入磁盘。

# redo log的记录方式

redolog的大小是固定的，在mysql中可以通过修改配置参数`innodb_log_files_in_group`和`innodb_log_file_size`配置日志文件数量和每个日志文件大小，因此总的redo log大小为`innodb_log_files_in_group * innodb_log_file_size`。redolog采用**循环写**的方式记录，当写到结尾时，会回到开头循环写日志。

```
mysql> SHOW VARIABLES LIKE '%innodb_log_file%';
+---------------------------+----------+
| Variable_name             | Value    |
+---------------------------+----------+
| innodb_log_file_size      | 50331648 |
| innodb_log_files_in_group | 2        |
+---------------------------+----------+
2 rows in set (0.10 sec)
```

Redo log文件以`ib_logfile[number]`命名，日志目录可以通过参数`innodb_log_group_home_dir`控制。Redo log 以顺序的方式写入文件文件，写满时则回溯到第一个文件，进行覆盖写。（但在做redo checkpoint时，也会更新第一个日志文件的头部checkpoint标记，所以严格来讲也不算顺序写）。

![](/assets/images/posts/redo-log/redo-log-2.png)

write pos表示日志**当前记录的位置**，当ib_logfile_4写满后，会从ib_logfile_1从头开始记录；check point表示将日志记录的修改写进磁盘，完成数据落盘，数据落盘后checkpoint会将日志上的相关记录擦除掉，

- write pos->checkpoint之间的部分是redo log空着的部分，用于记录新的记录
- checkpoint->write pos之间是redo log待落盘的数据修改记录。
- 当writepos追上checkpoint时，说明redolog已满，不能再执行新的更新操作得先停下记录，先推动checkpoint向前移动，空出位置记录新的日志。
- 只要write pos未赶上checkpoint，就可以执行新的更新操作

有了redo log，当数据库发生宕机重启后，可通过redo log将未落盘的数据恢复，即保证已经提交的事务记录不会丢失。

# log 何时产生 & 释放？

在事务开始之后就产生 redo log，redo log 的落盘并不是随着事务的提交才写入的，而是在**事务的执行过程中**，便开始写入 redo log 文件中。

当对应事务的脏页写入到磁盘之后，redo log 的使命也就完成了，redo log占用的空间就可以重用（被覆盖）。

Redo log文件是循环写入的，在覆盖写之前，总是要保证对应的脏页已经刷到了磁盘。**在非常大的负载下，Redo log可能产生的速度非常快，导致频繁的刷脏操作，进而导致性能下降**，通常在未做checkpoint的日志超过文件总大小的76%之后，InnoDB 认为这可能是个不安全的点，会强制的preflush脏页，导致大量用户线程stall住。如果可预期会有这样的场景，我们建议调大redo log文件的大小。可以做一次干净的shutdown，然后修改Redo log配置，重启实例。

# 缓冲区

InnoDB 修改数据操作写入redo log也不是直接写磁盘，而是先写到redo log缓冲区。

`innodb_flush_log_at_trx_commit`控制如何将redo log缓冲区的内容写入到日志文件。`innodb_flush_log_at_timeout`控制redo log缓存写到redo log文件的频率。

```
mysql> SHOW VARIABLES LIKE '%innodb_flush_log_at_%';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_timeout    | 1     |
| innodb_flush_log_at_trx_commit | 1     |
+--------------------------------+-------+
2 rows in set (21.80 sec)
```

> 更多内容查看 https://edgar615.github.io/innodb-log-buffer.html


# 宕机恢复

DB宕机后重启，InnoDB会首先去查看数据页中的LSN的数值。这个值代表数据页被刷新回磁盘的LSN的大小。然后再去查看redo 
log的LSN的大小。如果数据页中的LSN值大说明数据页领先于redo log刷新回磁盘，不需要进行恢复。反之需要从redo log中恢复数据。

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

> OS 中 write 和 fsync 是不同的操作，我们以为调用了 write 就万事大吉，数据一定到磁盘了，其实不一定，通常情况下 write 只是到了磁盘 IO 缓冲区，何时 fsync 由 OS 控制，这里通过程序强制调用来保证日志一定刷到磁盘

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

https://hoxis.github.io/mysql-zhuanlan-02-redolog-binlog.html

https://zhuanlan.zhihu.com/p/35355751

https://zhuanlan.zhihu.com/p/34650908

http://mysql.taobao.org/monthly/2015/05/01/

https://juejin.im/entry/5ba0a254e51d450e735e4a1f

https://keithlan.github.io/2017/06/12/innodb_locks_redo/

http://www.notedeep.com/note/38/page/222

http://zhongmingmao.me/2019/01/15/mysql-redolog-binlog/

https://chenjiayang.me/2019/04/13/mysql-innodb-redo-undo/
