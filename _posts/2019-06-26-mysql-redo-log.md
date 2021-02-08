---
layout: post
title: MySQL日志（2）- redo日志
date: 2019-06-26
categories:
    - MySQL
comments: true
permalink: mysql-redo-log.html
---

当数据库对数据做修改的时候，流程如下：

1. 需要把数据页从磁盘读到 buffer pool 中
2. 在 buffer pool 中进行修改，那么这个时候 buffer pool 中的数据页就与磁盘上的数据页**内容不一致**，我们称 buffer pool 的数据页为 **dirty page 脏页**。
3. 将dirty page刷新到盘（注意，**同步到磁盘文件是个随机 IO**）

> 注意：InnoDB对脏页的处理不是每次生成脏页就将脏页刷新回磁盘，这样会产生海量的IO操作，严重影响InnoDB的处理性能。

如果脏页刷新到盘期间发生非正常的 DB 服务重启，那么这些数据还在内存，并没有同步到磁盘文件中，就造成了**数据丢失**。

为了避免数据丢失的问题，MySQL提供了一个文件，当 buffer pool 中的 dirty page 变更结束后，**把相应修改记录记录到这个文件**（注意，**记录日志是顺序 IO**），那么当 DB 服务发生 crash 的情况，恢复 DB 的时候，也可以根据这个文件的记录内容，重新应用到磁盘文件，数据保持一致。这个文件就是 redo log ，**用于记录数据修改后的记录**，顺序记录。

> 顺序IO的性能比随机IO要高得多

# 1. Redo log
Redo Log又被称为WAL ( Write Ahead Log)，是InnoDB存储引擎实现事务持久性的关键。在InnoDB存储引擎中，事务执行过程被分割成一个个MTR (Mini TRansaction)，每个MTR在执行过程中对数据页的更改会产生对应的日志，这个日志就是Redo Log。事务在提交时，只要保证Redo Log被持久化，就可以保证事务的持久化。

由于Redo Log在持久化过程中顺序写文件的特性，使得持久化Redo Log的代价要远远小于持久化数据页，因此通常情况下，数据页的持久化要远落后于Redo Log

> **redo log 是物理日志而非逻辑日志，记录的是数据页的物理修改，而不是某一行或某几行修改成怎样怎样，它用来恢复提交后的物理数据页（恢复数据页，且只能恢复到最后一次提交的位置）。**

![](/assets/images/posts/redo-log/redo-log-1.png)

如图所示，在同一个事务中，每当数据库进行修改数据操作时，将修改结果更新到内存后，会在redo log添加一行记录记录“需要在哪个数据页上做什么修改”，并将该记录状态置为prepare，等到commit提交事务后，会将此次事务中在redo log添加的记录的状态都置为commit状态，之后将修改落盘时，会将redo log中状态为commit的记录的修改都写入磁盘。

# 2. 基本概念

redo log包括两部分：一是内存中的**日志缓冲(redo log buffer)**，该部分日志是易失性的；二是磁盘上的重做**日志文件(redo log file)**，该部分日志是持久的。

在概念上，innodb通过***force log at commit\***机制实现事务的持久性，**即在事务提交的时候，必须先将该事务的所有事务日志写入到磁盘上的redo log file和undo log file中进行持久化。**

为了确保每次日志都能写入到事务日志文件中，在每次将log  buffer中的日志写入日志文件的过程中都会调用一次操作系统的fsync操作(即fsync()系统调用)。因为MariaDB/MySQL是工作在用户空间的，MariaDB/MySQL的log buffer处于用户空间的内存中。要写入到磁盘上的log file中(redo:ib_logfileN文件,undo:share  tablespace或.ibd文件)，中间还要经过操作系统内核空间的os buffer，调用fsync()的作用就是将OS  buffer中的日志刷到磁盘上的log file中。

也就是说，从redo log buffer写日志到磁盘的redo log file中，过程如下： 

![](/assets/images/posts/redo-log/redo-log-5.png)

> 在此处需要注意一点，一般所说的log file并不是磁盘上的物理日志文件，而是操作系统缓存中的log file，官方手册上的意思也是如此(例如：With a value of 2, the contents of the **InnoDB log buffer are written to the log file** after each transaction commit and **the log file is flushed to disk approximately once per second**)。
>
> 但说实话，这不太好理解，既然都称为file了，应该已经属于物理文件了。所以在本文后续内容中都以os buffer或者file system buffer来表示官方手册中所说的Log file，然后log  file则表示磁盘上的物理日志文件，即log file on disk。
>
> 另外，之所以要经过一层os buffer，是因为open日志文件的时候，open没有使用O_DIRECT标志位，该标志位意味着绕过操作系统层的os  buffer，IO直写到底层存储设备。不使用该标志位意味着将日志进行缓冲，缓冲到了一定容量，或者显式fsync()才会将缓冲中的刷到存储设备。使用该标志位意味着每次都要发起系统调用。比如写abcde，不使用o_direct将只发起一次系统调用，使用o_object将发起5次系统调用。

MySQL支持用户自定义在commit时如何将log buffer中的日志刷log file中。这种控制通过变量 innodb_flush_log_at_trx_commit 的值来决定。

# 3. 日志块(log block)

innodb存储引擎中，redo log以块为单位进行存储的，每个块占512字节，这称为redo log block。所以不管是log buffer中还是os buffer中以及redo log file on disk中，都是这样以512字节的块存储的。

每个redo log block由3部分组成：**日志块头、日志块尾和日志主体**。其中日志块头占用12字节，日志块尾占用8字节，所以每个redo log block的日志主体部分只有512-12-8=492字节。

![](/assets/images/posts/redo-log/redo-log-6.png)

因为redo log记录的是数据页的变化，当一个数据页产生的变化需要使用超过492字节()的redo log来记录，那么就会使用多个redo log block来记录该数据页的变化。

日志块头包含4部分：

- log_block_hdr_no：(4字节)该日志块在redo log buffer中的位置ID。
- log_block_hdr_data_len：(2字节)该log block中已记录的log大小。写满该log block时为0x200，表示512字节。
- log_block_first_rec_group：(2字节)该log block中第一个log的开始偏移位置。
- lock_block_checkpoint_no：(4字节)写入检查点信息的位置。

关于log block块头的第三部分 log_block_first_rec_group  ，因为有时候一个数据页产生的日志量超出了一个日志块，这是需要用多个日志块来记录该页的相关日志。例如，某一数据页产生了552字节的日志量，那么需要占用两个日志块，第一个日志块占用492字节，第二个日志块需要占用60个字节，那么对于第二个日志块来说，它的第一个log的开始位置就是73字节(60+12)。如果该部分的值和 log_block_hdr_data_len 相等，则说明该log block中没有新开始的日志块，即表示该日志块用来延续前一个日志块。

日志尾只有一个部分：log_block_trl_no ，该值和块头的 log_block_hdr_no 相等。

上面所说的是一个日志块的内容，在redo log buffer或者redo log file on disk中，由很多log block组成。如下图：

![](/assets/images/posts/redo-log/redo-log-7.png)

# 4. log group和redo log file

log group表示的是redo log group，一个组内由多个大小完全相同的redo log file组成。组内redo log  file的数量由变量 innodb_log_files_group 决定，默认值为2，即两个redo log  file。这个组是一个逻辑的概念，并没有真正的文件来表示这是一个组，但是可以通过变量 **innodb_log_group_home_dir**  来定义组的目录，redo log file都放在这个目录下，默认是在datadir下。

可以看到在默认的数据目录下，有两个ib_logfile开头的文件，它们就是log group中的redo log  file，而且它们的大小完全一致且等于变量 innodb_log_file_size 定义的值。第一个文件ibdata1是在没有开启  innodb_file_per_table 时的共享表空间文件，对应于开启 innodb_file_per_table 时的.ibd文件。

```
# ll /var/lib/mysql/ib*
-rw-r----- 1 mysql mysql 12582912 Feb  8 09:52 /var/lib/mysql/ibdata1
-rw-r----- 1 mysql mysql 50331648 Feb  8 09:52 /var/lib/mysql/ib_logfile0
-rw-r----- 1 mysql mysql 50331648 Jan 19 09:26 /var/lib/mysql/ib_logfile1
```

redolog的大小是固定的，在mysql中可以通过修改配置参数`innodb_log_files_in_group`和`innodb_log_file_size`配置日志文件数量和每个日志文件大小，因此总的redo log大小为`innodb_log_files_in_group * innodb_log_file_size`。

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

在innodb将log buffer中的redo log block刷到这些log  file中时，**会以追加写入的方式循环轮训写入**。即先在第一个log file（即ib_logfile0）的尾部追加写，直到满了之后向第二个log  file（即ib_logfile1）写。当第二个log file满了会清空一部分第一个log file继续写入。

![](/assets/images/posts/redo-log/redo-log-2.png)

write pos表示日志**当前记录的位置**，当ib_logfile_4写满后，会从ib_logfile_1从头开始记录；check point表示将日志记录的修改写进磁盘，完成数据落盘，数据落盘后checkpoint会将日志上的相关记录擦除掉，

- write pos->checkpoint之间的部分是redo log空着的部分，用于记录新的记录
- checkpoint->write pos之间是redo log待落盘的数据修改记录。
- 当writepos追上checkpoint时，说明redolog已满，不能再执行新的更新操作得先停下记录，先推动checkpoint向前移动，空出位置记录新的日志。
- 只要write pos未赶上checkpoint，就可以执行新的更新操作

由于是将log buffer中的日志刷到log file，所以在log file中记录日志的方式也是log block的方式。

在每个组的第一个redo log file中，前2KB记录4个特定的部分，从2KB之后才开始记录log block。除了第一个redo log file中会记录，log group中的其他log file不会记录这2KB，但是却会腾出这2KB的空间。如下：

![](/assets/images/posts/redo-log/redo-log-8.png)

**redo log file的大小对innodb的性能影响非常大，设置的太大，恢复的时候就会时间较长，设置的太小，就会导致在写redo log的时候循环切换redo log file。**

有了redo log，当数据库发生宕机重启后，可通过redo log将未落盘的数据恢复，即保证已经提交的事务记录不会丢失。

# 5. redo log的格式

因为innodb存储引擎存储数据的单元是页(和SQL Server中一样)，所以redo log也是基于页的格式来记录的。默认情况下，innodb的页大小是16KB(由  innodb_page_size 变量控制)，一个页内可以存放非常多的log block(每个512字节)，而log  block中记录的又是数据页的变化。

其中log block中492字节的部分是log body，该log body的格式分为4部分：

- redo_log_type：占用1个字节，表示redo log的日志类型。
- space：表示表空间的ID，采用压缩的方式后，占用的空间可能小于4字节。
- page_no：表示页的偏移量，同样是压缩过的。
- redo_log_body表示每个重做日志的数据部分，恢复时会调用相应的函数进行解析。例如insert语句和delete语句写入redo log的内容是不一样的。

如下图，分别是insert和delete大致的记录方式。

![](/assets/images/posts/redo-log/redo-log-9.png)

# 6.  日志刷盘规则

log buffer中未刷到磁盘的日志称为脏日志(dirty log)。MySQL支持用户自定义在commit时如何将log buffer中的日志刷log file中。这种控制通过变量 innodb_flush_log_at_trx_commit  的值来决定。

`innodb_flush_log_at_trx_commit`参数可以控制将 redo buffer 中的更新记录写入到日志文件以及将日志文件数据刷新到磁盘的操作时机。通过调整这个参数，可以在性能和数据安全之间做取舍。

- 0：在事务提交时，innodb 不会立即触发将缓存日志写到磁盘文件的操作，而是每秒触发一次缓存日志回写磁盘操作，并调用系统函数 fsync 刷新 IO 缓存。这种方式效率最高，也最不安全。
- 1：在每个事务提交时，innodb 立即将缓存中的 redo 日志回写到日志文件，并调用 fsync 刷新 IO 缓存。
- 2：在每个事务提交时，innodb 立即将缓存中的 redo 日志回写到日志文件，但并不马上调用 fsync 来刷新 IO 缓存，而是每秒只做一次磁盘IO 缓存刷新操作。只要操作系统不发生崩溃，数据就不会丢失，这种方式是对性能和数据安全的折中，其性能和数据安全性介于其他两种方式之间。

innodb_flush_log_at_trx_commit 参数的默认值是 1，即每个事务提交时都会从 log buffer 写更新记录到日志文件，而且会实际刷新磁盘缓存，显然，这完全能满足事务的持久化要求，是最安全的，但这样会有较大的性能损失。

在某些需要尽量提高性能，并且可以容忍在数据库崩溃时丢失小部分数据，那么通过将参数 `innodb_flush_log_at_trx_commit`设置成 0 或 2 都能明显减少日志同步 IO，加快事务提交，从而改善性能。

> OS 中 write 和 fsync 是不同的操作，我们以为调用了 write 就万事大吉，数据一定到磁盘了，其实不一定，通常情况下 write 只是到了磁盘 IO 缓冲区，何时 fsync 由 OS 控制，这里通过程序强制调用来保证日志一定刷到磁盘

![](/assets/images/posts/redo-log/redo-log-3.png)

刷日志到磁盘有以下几种规则：

**1.发出commit动作时。已经说明过，commit发出后是否刷日志由变量 innodb_flush_log_at_trx_commit 控制。**

**2.每秒刷一次。这个刷日志的频率由变量 innodb_flush_log_at_timeout 值决定，默认是1秒。要注意，这个刷日志频率和commit动作无关。**

**3.当log buffer中已经使用的内存超过一半时。**

**4.当有checkpoint时，checkpoint在一定程度上代表了刷到磁盘时日志所处的LSN位置。**

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

# 7. 数据页刷盘的规则及checkpoint

内存中(buffer pool)未刷到磁盘的数据称为脏数据(dirty data)。由于数据和日志都以页的形式存在，所以脏页表示脏数据和脏日志。

上一节介绍了日志是何时刷到磁盘的，不仅仅是日志需要刷盘，脏数据页也一样需要刷盘。

**在innodb中，数据刷盘的规则只有一个：checkpoint。**但是触发checkpoint的情况却有几种。**不管怎样，checkpoint触发后，会将buffer中脏数据页和脏日志页都刷到磁盘。**

innodb存储引擎中checkpoint分为两种：

- sharp checkpoint：在重用redo log文件(例如切换日志文件)的时候，将所有已记录到redo log中对应的脏数据刷到磁盘。

- fuzzy checkpoint：一次只刷一小部分的日志到磁盘，而非将所有脏日志刷盘。有以下几种情况会触发该检查点：

- - master thread checkpoint：由master线程控制，**每秒或每10秒**刷入一定比例的脏页到磁盘。
  - flush_lru_list checkpoint：从MySQL5.6开始可通过 innodb_page_cleaners 变量指定专门负责脏页刷盘的page cleaner线程的个数，该线程的目的是为了保证lru列表有可用的空闲页。
  - async/sync flush checkpoint：同步刷盘还是异步刷盘。例如还有非常多的脏页没刷到磁盘(非常多是多少，有比例控制)，这时候会选择同步刷到磁盘，但这很少出现；如果脏页不是很多，可以选择异步刷到磁盘，如果脏页很少，可以暂时不刷脏页到磁盘
  - dirty page too much checkpoint：脏页太多时强制触发检查点，目的是为了保证缓存有足够的空闲空间。too much的比例由变量  innodb_max_dirty_pages_pct 控制，MySQL  5.6默认的值为75，即当脏页占缓冲池的百分之75后，就强制刷一部分脏页到磁盘。

由于刷脏页需要一定的时间来完成，所以记录检查点的位置是在每次刷盘结束之后才在redo log中标记的。

> MySQL停止时是否将脏数据和脏日志刷入磁盘，由变量innodb_fast_shutdown={ 0|1|2  }控制，默认值为1，即停止时只做一部分purge，忽略大多数flush操作(但至少会刷日志)，在下次启动的时候再flush剩余的内容，实现fast shutdown。

# 8. log 何时产生 & 释放？

在事务开始之后就产生 redo log，redo log 的落盘并不是随着事务的提交才写入的，而是在**事务的执行过程中**，便开始写入 redo log 文件中。

当对应事务的脏页写入到磁盘之后，redo log 的使命也就完成了，redo log占用的空间就可以重用（被覆盖）。

Redo log文件是循环写入的，在覆盖写之前，总是要保证对应的脏页已经刷到了磁盘。**在非常大的负载下，Redo log可能产生的速度非常快，导致频繁的刷脏操作，进而导致性能下降**，通常在未做checkpoint的日志超过文件总大小的76%之后，InnoDB 认为这可能是个不安全的点，会强制的preflush脏页，导致大量用户线程stall住。如果可预期会有这样的场景，我们建议调大redo log文件的大小。可以做一次干净的shutdown，然后修改Redo log配置，重启实例。

# 宕机恢复

在启动innodb的时候，不管上次是正常关闭还是异常关闭，总是会进行恢复操作。

因为redo log记录的是数据页的物理变化，因此恢复的时候速度比逻辑日志(如二进制日志)要快很多。而且，innodb自身也做了一定程度的优化，让恢复速度变得更快。

重启innodb时，checkpoint表示已经完整刷到磁盘上data  page上的LSN（这个值代表数据页被刷新回磁盘的LSN的大小），因此恢复时仅需要恢复从checkpoint开始的日志部分。例如，当数据库在上一次checkpoint的LSN为10000时宕机，且事务是已经提交过的状态。启动数据库时会检查磁盘中数据页的LSN，如果数据页的LSN小于日志中的LSN，则会从检查点开始恢复。

还有一种情况，在宕机前正处于checkpoint的刷盘过程，且数据页的刷盘进度超过了日志页的刷盘进度。这时候一宕机，数据页中记录的LSN就会大于日志页中的LSN，在重启的恢复过程中会检查到这一情况，这时超出日志进度的部分将不会重做，因为这本身就表示已经做过的事情，无需再重做。

另外，事务日志具有幂等性，所以多次操作得到同一结果的行为在日志中只记录一次。而二进制日志不具有幂等性，多次操作会全部记录下来，在恢复的时候会多次执行二进制日志中的记录，速度就慢得多。例如，某记录中id初始值为2，通过update将值设置为了3，后来又设置成了2，在事务日志中记录的将是无变化的页，根本无需恢复；而二进制会记录下两次update操作，恢复时也将执行这两次update操作，速度比事务日志恢复更慢。

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

https://mp.weixin.qq.com/s/rNFy_qwnNWUvzjYznOXKJw