---
layout: post
title: redis有那些阻塞操作
date: 2020-04-10
categories:
    - redis
comments: true
permalink: redis-block.html
---

**集合全量查询和聚合操作**

Redis 中涉及集合的操作复杂度通常为 O(N)，我们要在使用时重视起来。例如集合元素全量查询操作 HGETALL、SMEMBERS，以及集合的聚合统计操作，例如求交、并和差集。可以使用集合提供的 **SCAN** 命令，分批读取数据，再在客户端进行聚合计算

**bigkey 删除操作**

删除操作的本质是要释放键值对占用的内存空间。释放内存只是第一步，为了更加高效地管理内存空间，在应用程序释放内存时，操作系统需要把释放掉的内存块插入一个空闲内存块的链表，以便后续进行管理和再分配。这个过程本身需要一定时间，而且会阻塞当前释放内存的应用程序，所以，如果一下子释放了大量内存，空闲内存块链表操作时间就会增加，相应地就会造成 Redis 主线程的阻塞。**可以使用UNLINK 操作**，UNLINK是DEL的异步删除版本，UNLINK命令与DEL阻塞删除不同，UNLINK在删除集合类键时，如果集合键的元素个数大于64个，会把真正的内存释放操作，给单独的BackgroundIO线程来操作

我们也可以直接开启惰性删除策略

```
# 针对redis内存使用达到maxmeory，并设置有淘汰策略时；在被动淘汰键时，是否采用lazy free机制；
lazyfree-lazy-eviction no 
# 针对设置有TTL的键，达到过期后，被redis清理删除时是否采用lazy free机制
lazyfree-lazy-expire no
# 针对有些指令在处理已存在的键时，会带有一个隐式的DEL键的操作。如rename命令，当目标键已存在,redis会先删除目标键
lazyfree-lazy-server-del no
# 针对slave进行全量数据同步，slave在加载master的RDB文件前，会运行flushall来清理自己的数据场景，
replica-lazy-flush no

# It is also possible, for the case when to replace the user code DEL calls
# with UNLINK calls is not easy, to modify the default behavior of the DEL
# command to act exactly like UNLINK, using the following configuration
# directive:

lazyfree-lazy-user-del no
```



**清空数据库**

清空数据库（例如 FLUSHDB 和 FLUSHALL 操作也涉及到删除和释放所有的键值对，**可以使用`FLUSHDB ASYNC`异步执行**

**AOF 日志同步写**

Redis 直接记录 AOF 日志时，会根据不同的写回策略对数据做落盘保存。一个同步写磁盘的操作的耗时大约是 1～2ms，如果有大量的写操作需要记录在 AOF 日志中，并同步写回的话，就会阻塞主线程。**应该使用异步写**

**从库加载 RDB **

从库在清空当前数据库后，还需要把 RDB 文件加载到内存，这个过程的快慢和 RDB 文件的大小密切相关，RDB 文件越大，加载过程越慢。把主库的数据量大小控制在 2~4GB 左右，以保证 RDB 文件能以较快的速度加载。