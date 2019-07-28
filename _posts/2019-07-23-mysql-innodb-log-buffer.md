---
layout: post
title: InnoDB Log Buffer
date: 2019-07-28
categories:
    - MySQL
comments: true
permalink: innodb-log-buffer.html
---

Log Buffer(日志缓冲区)是一块内存区域用来保存要写入磁盘上的日子文件的数据。 Log Buffer的大小由innodb_log_buffer_size变量定义。默认大小为16MB。Log Buffer的内容会定期刷到磁盘上。大的Log Buffer让较大事务能够运行，而无需在事务提交之前将redo log中的数据写入磁盘。

因此，如果有DML操作并且会影响很多行这样是事务，增加日志缓冲区的大小可以节省磁盘IO

它会存储InnoDB存储引擎层日志：redo日志和undo日志

`innodb_flush_log_at_trx_commi`控制如何将缓冲区的内容写入到日志文件。`innodb_flush_log_at_timeout`控制缓存写到redo log文件的频率。

后面有时间在整理