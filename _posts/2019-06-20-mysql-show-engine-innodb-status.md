---
layout: post
title: MySQL查看innodb状态
date: 2019-06-20
categories:
    - MySQL
comments: true
permalink: mysql-show-engine-innodb-status.html
---

首先我们来看一个`show engine innodb status;`返回结果


```
mysql> show engine innodb status;
...
=====================================
2019-06-20 13:58:21 0x7f9650051700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 23 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 1472318 srv_active, 0 srv_shutdown, 21326122 srv_idle
srv_master_thread log flush and writes: 22796915
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 3611483
OS WAIT ARRAY INFO: signal count 4243864
RW-shared spins 0, rounds 7147333, OS waits 3104121
RW-excl spins 0, rounds 66439309, OS waits 174603
RW-sx spins 90448, rounds 1561624, OS waits 16100
Spin rounds per wait: 7147333.00 RW-shared, 66439309.00 RW-excl, 17.27 RW-sx
------------------------
LATEST FOREIGN KEY ERROR
------------------------
2019-04-21 13:30:10 0x7f9652626700 Error in foreign key constraint of table tabao_dev/qrtz_blob_triggers:
 FOREIGN KEY (`SCHED_NAME`, `TRIGGER_NAME`, `TRIGGER_GROUP`) REFERENCES `qrtz_triggers` (`SCHED_NAME`, `TRIGGER_NAME`, `TRIGGER_GROUP`) ON DELETE RESTRICT ON UPDATE RESTRICT
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic:
Cannot find an index in the referenced table where the
referenced columns appear as the first columns, or column types
in the table and the referenced table do not match for constraint.
Note that the internal storage type of ENUM and SET changed in
tables created with >= InnoDB-4.1.12, and such columns in old tables
cannot be referenced by such columns in new tables.
Please refer to http://dev.mysql.com/doc/refman/5.7/en/innodb-foreign-key-constraints.html for correct foreign key definition.
------------
TRANSACTIONS
------------
Trx id counter 6184470
Purge done for trx's n:o < 6184427 undo n:o < 0 state: running but idle
History list length 11
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421759551742464, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421759551741552, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421759551740640, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421759551739728, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
395394 OS file reads, 30394560 OS file writes, 15300961 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 28 merges
merged operations:
 insert 207, delete mark 1, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 18 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 2 buffer(s)
Hash table size 34679, node heap has 2 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
---
LOG
---
Log sequence number 18141044162
Log flushed up to   18141044162
Pages flushed up to 18141044162
Last checkpoint at  18141044153
0 pending log flushes, 0 pending chkp writes
9351897 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 3807133
Buffer pool size   8192
Free buffers       1024
Database pages     7141
Old database pages 2616
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 127464, not young 10579743
0.00 youngs/s, 0.00 non-youngs/s
Pages read 393131, created 205050, written 19241821
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 7141, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=18363, Main thread ID=140284274763520, state: sleeping
Number of rows inserted 19647251, updated 1741697, deleted 208288, read 655383247
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 5.09 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================
...
1 row in set (0.11 sec)

```

依次分析上面的输出

# Header

```
=====================================
2019-06-20 13:58:21 0x7f9650051700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 23 seconds
```
表示至少统计了 23 秒的样本数据。如果平均统计间隔是0或1秒那么结果就没什么意义了

# BACKGROUND THREAD
显示后台线程信息

```
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 1472318 srv_active, 0 srv_shutdown, 21326122 srv_idle
srv_master_thread log flush and writes: 22796915
```

- Srv_master_thread loops Master线程的循环次数，master线程在每次loop过程中都会sleep，sleep的时间为1秒。而在每次loop的过程中会选择active、shutdown、idle中一种状态执行。Master线程在不停循环，所以其值是随时间递增的。
- Srv_active Master线程选择的active状态执行。Active数量增加与数据表、数据库更新操作有关，与查询无关，例如：插入数据、更新数据、修改表等。
- Srv_shutdown 这个参数的值一直为0，因为srv_shutdown只有在mysql服务关闭的时候才会增加。
- Srv_idle 这个参数是在master线程空闲的时候增加，即没有任何数据库改动操作时。
- Log_flush_and_write Master线程在后台会定期刷新日志，日志刷新是由参数`innodb_flush_log_at_timeout`参数控制前后刷新时间差。
- 
Background thread部分信息为统计信息，即mysql服务启动之后该部分值会一直递增，因为它显示的是自mysqld服务启动之后master线程所有的loop和log刷新操作。通过对比active和idle的值，可以获知系统整体负载情况。Active的值越大，证明服务越繁忙。

# SEMAPHORES
如果你有一个高并发的系统，你需要关注这一部分的输出。它由两部分组成，event counters, 和可选项输出,即当前等待的事件(current waits)。

```
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 3611483
OS WAIT ARRAY INFO: signal count 4243864
RW-shared spins 0, rounds 7147333, OS waits 3104121
RW-excl spins 0, rounds 66439309, OS waits 174603
RW-sx spins 90448, rounds 1561624, OS waits 16100
Spin rounds per wait: 7147333.00 RW-shared, 66439309.00 RW-excl, 17.27 RW-sx
```

`reservation count`表示Innodb产生了多少次OS WAIT, `signal count`表示，进行OS WAIT的线程，接收到多少次信号（singal）被唤醒。如果你看到signal的数值很大，通常是几十万，上百万。就表明，可能是很多I/O的等待，或是Innodb争用(contention)问题。关于争用问题，可能与OS的进程调度有关，你可尝试减少innodb_thread_concurrency参数。


**什么是OS Wait,什么是spin wait**

要明白这个，首先你要明白Innodb如何处理互斥量(Mutexes)，以及什么是两步获得锁(two-step approach)。首先进程，试图获得一个锁，如果此锁被它人占用。它就会执行所谓的spin wait,即所谓循环的查询”锁被释放了吗？”。如果在循环过程中，一直未得到锁释放的信息，则其转入OS WAIT，即所谓线程进入挂起(suspended)状态。直到锁被释放后，通过信号(singal)唤醒线程。

- Mutex spin     waits 是线程无法获取锁，而进入的spin wait
- rounds 是spin wait进行轮询检查Mutextes的次数
- OS waits 是线程放弃spin-wait进入挂起状态

Spin wait的消耗远小于OS waits。Spin wait利用cpu的空闲时间，检查锁的状态，OS Wait会有所谓上下文切换，从CPU内核中换出当前执行线程以供其它线程使用。你可以通过`innodb_sync_spin_loops`参数来平衡spin wait和os wait。

# TRANSACTIONS

```
------------
TRANSACTIONS
------------
Trx id counter 6184470
Purge done for trx's n:o < 6184427 undo n:o < 0 state: running but idle
History list length 11
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421759551742464, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421759551741552, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421759551740640, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421759551739728, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
```
这一部分包含Innodb 事务（transactions）的统计信息，还有当前活动的事务列表。

`Trx id counter 6184470`显示的是当前的transaction id, 这个ID是一个系统变量随时每次新的transaction产生而增加。

`Purge done for trx's n:o < 6184427 undo n:o < 0 state: running but idle`显示的正在进行清空（purge）操作的transaction ID。你可以通过查看上一行和这一行的区别，明白没有被purge的事务落后的情况。

`History list length 11`记录了undo spaces内unpurged的事务的个数。

下面的内容显示了事务的列表

`---TRANSACTION 421759551742464, not started`显示了事务ID和状态。事务有下列状态`not started`,`active`,`prepared`,`committedin memory`

后面的数据显示显示了thread id,这个值与是在show full processlist 命令显示的进程ID是一样的。当前事务执行的SQL语句，当前事务锁定的数据表和MVCC信息

# FILE I/O
FILE I/O部分显示了I/O Helper thread的状态，包括一些统计信息

```
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
395394 OS file reads, 30394560 OS file writes, 15300961 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
```

`I/O thread 0 state`显示了各个I/O thread的状态

`Pending normal aio reads...`显示各个I/O thread的pending operations, pending的log和buffer pool thread的fsync()调用

`395394 OS file reads...`显示了reads, writes, and fsync()调用次数。

`0.00 reads/s...`显示了每秒的统计信息

# INSERT BUFFER AND ADAPTIVE HASH INDEX

```
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 28 merges
merged operations:
 insert 207, delete mark 1, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 18 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 2 buffer(s)
Hash table size 34679, node heap has 2 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
```
`Ibuf...`显示了insertbuffer的一些信息，包括free list, segment size

`merged operations:...`显示了Innodb进行了多少次buffer操作。通过比较 inserts和 merges，可以看出insert buffer的效率

`Hash table size 34679...`显示了hash table的一些信息

`0.00 hash searches/s, 0.00 non-hash searches/s`显示了每秒进行了多少次hash搜索，以及非hash搜索

# LOG
这里记录了tansaction log子系统的信
```
---
LOG
---
Log sequence number 18141044162
Log flushed up to   18141044162
Pages flushed up to 18141044162
Last checkpoint at  18141044153
0 pending log flushes, 0 pending chkp writes
9351897 log i/o's done, 0.00 log i/o's/second
```

`Log sequence number 18141044162`显示了当前的LSN。

`Log flushed up to   18141044162`显示了已经被flushed(写入磁盘)的logs

`Pages flushed up to 18141044162`显示了最后一个checkpoint的logs

`0 pending log flushes...`显示了pending log 的统计信息

# BUFFER POOL AND MEMORY
显示buffer pool的使用情况
```
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 3807133
Buffer pool size   8192
Free buffers       1024
Database pages     7141
Old database pages 2616
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 127464, not young 10579743
0.00 youngs/s, 0.00 non-youngs/s
Pages read 393131, created 205050, written 19241821
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 7141, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
```

`Total large memory allocated 137428992`分配给Innodb的内存大小

`Dictionary memory allocated 3807133`分配给Innodb数据字典的内存大小

`Buffer pool size   8192` innodb_buffer_pool的大小(page)

`Free buffers       1024` innodb_buffer_pool lru列表中的空闲页面数量

`Database pages     7141` innodb_buffer_pool lru列表中的非空闲页面数

`Old database pages 2616` innodb_buffer_pool old子列表的页面数量

`Modified db pages  0` innodb_buffer_pool 中脏页的数量

`Pending reads`和`Pending writes`显示了pending的reads 和writes

`Buffer pool hit rate 1000 / 1000...`显示了Innodb的buffer pool命中率，通常要保证在998/1000以上。如果没有，可考虑增大buffer pool size，以及优化你的查询

`Pages read ahead 0.00/s`显示了每秒线性预读的次数（读入）。

`evicted without access`显示了每秒读出的pages

`Random read ahead 0.00/s`显示了每秒随机预读的次数。

可能Free buffers+Database pages 可能并不等于Buffer pool size ，因为缓冲池中的页还可能会被分配给insert buffer、自适应哈希索引、锁信息(lock info)，数据字典信息等

**需要关注的值：**

- `youngs/s`：该指标表示的是每秒访问Old 链表中页面，使其移动到Young链表的次数。如果MySQL实例都是一些小事务，没有大表全扫描，且该指标很小，就需要调大innodb_old_blocks_pct 或者减小innodb_old_blocks_time，这样会使得Old List 的长度更长，Old页面被移动到Old List 的尾部消耗的时间会更久，那么就提升了下一次访问到Old List里面的页面的可能性。如果该指标很大，可以调小innodb_old_blocks_pct，同时调大innodb_old_blocks_time，保护热数据。
- `non-youngs/s`：该指标表示的是每秒访问Old 链表中页面，没有移动到Young链表的次数，因为其不符合innodb_old_blocks_time。如果该指标很大，一般情况下是MySQL存在大量的全表扫描。如果MySQL存在大量全表扫描，且这个指标又不大的时候，需要调大innodb_old_blocks_time，因为这个指标不大意味着全表扫描的页面被移动到Young 链表了，调大innodb_old_blocks_time时间会使得这些短时间频繁访问的页面保留在Old 链表里面。

每隔1秒钟，Page Cleaner线程执行LRU List Flush的操作，来释放足够的Free Page。`innodb_lru_scan_depth` 变量控制每个Buffer Pool实例每次扫描LRU List的长度，来寻找对应的脏页，执行Flush操作。

# ROW OPERATIONS
这一部分显示了rowoperation及其它的一些统计信息

```
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=18363, Main thread ID=140284274763520, state: sleeping
Number of rows inserted 19647251, updated 1741697, deleted 208288, read 655383247
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 5.09 reads/s
----------------------------
```

`0 queries inside InnoDB...`显示了有多少线程在Innodb内核

`0 read views open inside InnoDB`显示了有多少read view被打开了

# 参考资料

https://blog.csdn.net/qq_38125183/article/details/80658822