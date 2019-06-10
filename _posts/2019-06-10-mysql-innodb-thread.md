---
layout: post
title: InnoDB线程
date: 2019-06-10
categories:
    - MySQL
comments: true
permalink: innodb-thread.html
---

# 后台线程

Innodb是多线程模型，后台有多个不同的后台线程，负责处理不同的任务

## Master Thread

master thread是MySQL中一个非常核心，非常重要的一个线程，主要负责将缓冲池中的数据异步的刷新到磁盘，保证数据的一致性，包括刷新脏页到磁盘，刷新redo日志，合并插入缓冲，回收undo空间等

## IO Thread

在InnoDB存储引擎中，使用了大量的AIO来处理IO请求，这样可以极大的提高IO的性能，而IO
 thread的工作就是主要负责这些IO请求的回调。一共有四种类型的IO Thread，分别是write，read，insert 
buffer和log io thread,在Linux环境下，Io thread的数量不能调节，在Windows环境下可以通过innodb_read_io_threads和innodb_write_io_threads这两个参数分别调整write和read iothread。通过命令`show variables like 'innodb_%io_version'\G`可以看见这些io thread。

```
mysql> show variables like 'innodb_%io_threads'\G
*************************** 1. row ***************************
Variable_name: innodb_read_io_threads
        Value: 4
*************************** 2. row ***************************
Variable_name: innodb_write_io_threads
        Value: 4
2 rows in set (0.00 sec)
```

InnoDB 1.0.x版本开始，read thread和write thread分别增大到4个

也可以用`show engine innodb status\G`命令来观察IO Thread

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
504 OS file reads, 18466 OS file writes, 8930 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
```

可以看到I/O thread 0为insert buffer thread，I/O thread 1为log thread，之后就是根据参数innodb_read_io_threads和innodb_write_io_threads来设置的读写线程，读线程总是小于写线程

## purge thread 

事物被提交后，其所使用的undolog可能不在使用，因此需要purge thread来回收已经使用并分配的undo 
page，在之前的版本中purge thread的工作是master thread完成的，为了减轻master 
thread的工作，提高cpu的使用率以及提升存储引擎的性能，用户可以将参数`innodb_purge_threads=1`来启动单独的purge thread，最多可以启动4个。

## page cleaner thread    

 page cleaner thread的作用是将脏页刷新到磁盘，其目的是为了减轻原Master Thread的工作以及对用户查询线程的阻塞，进一步提高InnoDb存储引擎的性能

# 内存

## 缓冲池

InnoDB是基于磁盘存储的，并将其中的记录按照页的方式进行管理。

在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常是由缓冲池技术来提高数据库的整体性能。缓冲池简单来说就是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。

在数据库中进行读取页的操作，首先将从磁盘读到的页存放在缓冲池中，这个过程称为将页“FIX”在缓冲池中。下次再读取相同的页时，受限判断该页是否在缓冲池中，若在缓冲池中，称该页在缓冲池中被命中，直接读取该页。否则读取磁盘上的页。

对数据库中页的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上，这里需要注意的是，页从缓冲池刷新回磁盘的操作并不是在每次页发生变更时触发，而是通过一种称为CheckPoint的机制刷新回磁盘，同样这也是为了提高数据库的整体性能。

综上所述，缓冲池的大小直接影响着数据库的整体性能。InnoDB的缓冲池配置可以通过参数`innodb_buffer_pool_size来设置`

```
mysql> show variables like 'innodb_buffer_pool_size'\G
*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_size
        Value: 134217728
1 row in set (0.01 sec)
```

 缓冲池中缓存的数据页类型有：索引页，数据页，undo页，插入缓冲(insert buffer)、自适应哈希索引(adaptive hash index)、InnoDB存储的锁信息(lock info)，数据字典信息(data dirctionary)等。索引页和数据页只是占缓冲池很大的一部分而已

InnoDB允许有多个缓冲池实例，每个页根据哈希评价分片到不同的缓冲池实例中，这样做的好处是减少数据库内部的资源竞争，增加数据库的并发处理能力，可以通过`innodb_buffer_pool_instances`来进行配置

```
mysql> show variables like 'innodb_buffer_pool_instances'\G
*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_instances
        Value: 1
1 row in set (0.00 sec)
```

也可以用`show engine innodb status\G`命令来观察缓冲池

# LRU List、Free List和Flush List

通常来说数据库中的缓冲池是通过LRU算法进行管理的。

在InnoDB中，缓冲池中页的大小默认为16KB，使用LRU算法进行管理。但是与传统的LRU算法不同，LRU列表中加入了一个midpoint位置。新读取到的页，虽然是最新访问的页，单并不是直接放入到LUR列表的首部，而是放入到LRU列表的midpoint位置。可以通过`innodb_old_blocks_pct`控制

```
mysql> show variables like 'innodb_old_blocks_pct'\G
*************************** 1. row ***************************
Variable_name: innodb_old_blocks_pct
        Value: 37
1 row in set (0.01 sec)
```

在innodb把midpoint之后的列表称为old列表，之前的列表称为new列表，可以简单的理解为new列表中的页都是最为活跃的热点数据。

原因：某些SQL操作可能会使缓冲池中的页被刷新出，从而影响缓冲池的效率。例如作为索引或者数据的扫描操作，这类操作需要访问表中的许多页，甚至是全部的页，而这些页通常来说进击在这次查询操作中需要，并不是活跃的热点数据。如果页被放入LRU列表的首部，那么非常可能需要的热点数据页从LRU列表中移除，而在下一次需要读取该页时，InnoDB需要再次访问磁盘。

InnoDB通过`innodb_old_blocks_time`参数来控制页读取到mid位置后需要等待多久才会被加入到LRU列表的热端。

```
mysql> show variables like 'innodb_old_blocks_time'\G
*************************** 1. row ***************************
Variable_name: innodb_old_blocks_time
        Value: 1000
1 row in set (0.00 sec)
```

LRU列表用了管理已经读取的页，但当数据库刚启动时，LRU列表是空的，即没有任何页，这时页都存放在Free列表中。当需要从缓冲池中分页时，首先从free列表中查找是否有可用的空闲页，若有则将该页从free列表中移除放入到LRU列表中，否则根据LRU算法，淘汰LRU列表尾部的页，将该内存空间分片给新的页。当页从LRU的old部分加入到new部分时，称此时发生的操作为page made young，而因为`innodb_old_blocks_time`的设置导致页每页从old部分移动到new部分的操作称为page not made young。可以用`show engine innodb status\G`命令来观察LRU列表和free列表的使用情况和运行状态

```
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 308201
Buffer pool size   8191
Free buffers       6999
Database pages     1189
Old database pages 418
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 457, created 732, written 11706
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 1189, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
```

Buffer pool size 共有 8191个页，共8191*16K的缓冲池空间，Free buffers表示当前Free列表的页的数量。Database pages 表示LRU列表中页的数量。可能Free buffers+Database pages 可能并不等于Buffer pool size ，因为缓冲池中的页还可能会被分配给insert buffer、自适应哈希索引、锁信息(lock info)，数据字典信息等

# 参考资料

