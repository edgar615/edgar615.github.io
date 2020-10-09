---
layout: post
title: InnoDB内存（2）- 设置Buffer Pool大小
date: 2019-06-22
categories:
    - MySQL
comments: true
permalink: innodb-buffer-pool-size.html
---

# 1. Buffer Pool Size

理想情况下，在给服务器的其他进程留下足够的内存空间的情况下，Buffer Pool Size应该设置的尽可能大。当Buffer Pool Size设置的足够大时，整个数据库就相当于存储在内存当中，当读取一次数据到Buffer Pool Size以后，后续的读操作就不用再访问磁盘。

当数据库已经启动的情况下，我们可以通过在线调整的方式修改Buffer Pool Size的大小。通过以下语句：

```
SET GLOBAL innodb_buffer_pool_size=402653184;
```

当执行这个语句以后，并不会立即生效，而是要等所有的事务全部执行成功以后才会生效；新的连接和事务必须等其他事务完全执行成功以后，Buffer Pool Size设置生效以后才能够连接成功，不然会一直处于等待状态。

期间，Buffer Pool Size要完成碎片整理，去除缓存page等等操作。在执行增加或者减少Buffer Pool Size的操作时，操作会作为一个执行块执行，`innodb_buffer_pool_chunk_size`的大小会定义一个执行块的大小，默认的情况下，这个值是128M。

Buffer Pool Size的大小最好设置为`innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances`的**整数倍**，而且是大于等于1。

如果你的机器配置的大小**不是整数倍**的话，Buffer Pool Size的大小是会自适应修改为`innodb_buffer_pool_chunk_size*innodb_buffer_pool_instances`的整数倍，会略小于你配置的Buffer Pool Size的大小。

**比如以8G为例**：

`mysqld --innodb_buffer_pool_size=8G  --innodb_buffer_pool_instances=16`，然后`innodb_buffer_pool_instances=16`的大小刚好设置为16，是一个整数倍的关系。而且`innodb_buffer_pool_chunk_size`的大小也是可以在my.cnf里面指定的。

**还有一种情况是**`innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances`大于buffer pool size的情况下，`innodb_buffer_pool_chunk_size `也会自适应为`Buffer Pool size/innodb_buffer_pool_instances`。

 如果我们要查看Buffer Pool的状态的话：

```
SHOW STATUS WHERE Variable_name='InnoDB_buffer_pool_resize_status';
```

可以帮我们查看到状态。我们可以看一下增加Buffer Pool的时候的一个过程，再看一下减少的时候的日志，其实还是很好理解的，我们可以看成每次增大或者减少Buffer Pool的时候就是进行innodb_buffer_pool_chunk的增加或者释放，按照innodb_buffer_pool_chunk_size 设定值的大小增加或者释放执行块。

**增加的过程：**增加执行块，指定新地址，将新加入的执行块加入到free list（控制执行块的一个列表，可以这么理解）。

**减少的过程：**重新整理Buffer Pool和空闲页，将数据从块中移除，指定新地址。

# 2. Buffer Pool Instances

在64位操作系统的情况下，可以拆分缓冲池成多个部分，这样可以在高并发的情况下最大可能的减少争用。

配置多个Buffer Pool Instances能在很大程度上能够提高MySQL在高并发的情况下处理事物的性能，优化不同连接读取缓冲页的争用。

**我们可以通过设置 innodb_buffer_pool_instances来设置Buffer Pool Instances**。当InnoDB Buffer Pool 足够大的时候，你能够从内存中读取时候能有一个较好的性能，但是也有可能碰到多个线程同时请求缓冲池的瓶颈。这个时候设置多个Buffer Pool Instances能够尽量减少连接的争用。

这能够保证每次从内存读取的页都对应一个Buffer Pool Instances，而且这种对应关系是一个随机的关系。并不是热数据存放在一个Buffer Pool  Instances下，内部也是通过hash算法来实现这个随机数的。每一个Buffer Pool Instances都有自己的free  lists，LRU和其他的一些Buffer Poll的数据结构，各个Buffer Pool Instances是相对独立的。

`innodb_buffer_pool_instances`的设置必须大于1才算得上是多配置，**但是这个功能起作用的前提**是`innodb_buffer_pool_size`的大小必须大于1G，理想情况下`innodb_buffer_pool_instances`的每一个instance都保证在1G以上。

有文章推荐将`innodb_buffer_pool_instances `设为8，实际还是要真的不同环境做压测。

> 实际上 innodb_buffer_pool_instances = 64 显示出最佳的吞吐量和较小的可变性。从可变性的角度来看，建议的 innodb_buffer_pool_instances = 8 似乎比 1-4 的值更好，但不会产生最佳的吞吐量。

> [MySQL 8 需要多大的 innodb_buffer_pool_instances 值（上）](https://mp.weixin.qq.com/s?__biz=MzU2NzgwMTg0MA==&mid=2247489115&idx=1&sn=33e88824346ce39fc95354b994ce889a&chksm=fc96f4c4cbe17dd2e5aa25a9f3e4f6cc1a27ace44b05b618a12f4d4e5d88a0417f0c49e241bb&scene=21#wechat_redirect)
>
> [MySQL 8 需要多大的 innodb_buffer_pool_instances 值（下）](https://mp.weixin.qq.com/s/2zc_KPfEpetPc181Iwx86A)

# 3. 参考资料

https://mp.weixin.qq.com/s/ZPXcsogmO9BkKMKNLQxNLA