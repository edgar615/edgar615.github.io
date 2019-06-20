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



# 参考资料

