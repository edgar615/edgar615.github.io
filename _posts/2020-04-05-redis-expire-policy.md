---
layout: post
title: redis过期删除策略
date: 2020-04-05
categories:
    - redis
comments: true
permalink: redis-expire-policy.html
---

**[Redis缓存淘汰策略](https://edgar615.github.io/redis-maxmemory-policy.html)与Redis键的过期删除策略并不完全相同，前者是在Redis内存使用超过一定值的时候（一般这个值可以配置）使用的淘汰策略；而后者是通过定期删除+惰性删除两者结合的方式进行内存淘汰的。**

**定时删除**

对于每一个设置了过期时间的 Key 都会创建一个定时器，一旦到达过期时间就立即删除。

该策略可以立即清除过期的数据，对内存较友好，但是缺点是占用了大量的 CPU 资源去处理过期的数据，会影响 Redis 的吞吐量和响应时间。

**惰性删除**

当访问一个 Key 时，才判断该 Key 是否过期，过期则删除。该策略能最大限度地节省 CPU 资源，但是对内存却十分不友好。

有一种极端的情况是可能出现大量的过期 Key 没有被再次访问，因此不会被清除，导致占用了大量的内存。

在计算机科学中，懒惰删除（英文：lazy deletion）指的是从一个散列表（也称哈希表）中删除元素的一种方法。

在这个方法中，删除仅仅是指标记一个元素被删除，而不是整个清除它。被删除的位点在插入时被当作空元素，在搜索之时被当作已占据。

**定期删除**
每隔一段时间，扫描 Redis 中过期 Key 字典，并清除部分过期的 Key。该策略是前两者的一个折中方案，还可以通过调整定时扫描的时间间隔和每次扫描的限定耗时，在不同情况下使得 CPU 和内存资源达到最优的平衡效果。

在 Redis 中，同时使用了定期删除和惰性删除。

## 原理

**RedisDB 结构体定义**

我们知道，Redis 是一个键值对数据库，对于每一个 Redis 数据库，Redis 使用一个 RedisDB 的结构体来保存，它的结构如下：

```c
typedef struct redisDb {
        dict *dict;                 /* 数据库的键空间，保存数据库中的所有键值对 */
        dict *expires;              /* 保存所有过期的键 */
        dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
        dict *ready_keys;           /* Blocked keys that received a PUSH */
        dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
        int id;                     /* 数据库ID字段，代表不同的数据库 */
        long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;
```

> 关于数据结构的介绍可以[参考这篇文章](https://edgar615.github.io/redis-object.html)

从结构定义中我们可以发现，对于每一个 Redis 数据库，都会使用一个字典的数据结构来保存每一个键值对，dict 的结构图如下：

![](/assets/images/posts/redis-expire-policy/redis-expirie-policy-1.png)

以上就是过期策略实现时用到比较核心的数据结构。程序=数据结构+算法，介绍完数据结构以后，接下来继续看看处理的算法是怎样的。

**expires 属性**

RedisDB 定义的第二个属性是 expires，它的类型也是字典，Redis 会把所有过期的键值对加入到 expires，之后再通过定期删除来清理 expires 里面的值。

加入 expires 的场景有：

- **Set 指定过期时间 expire，**如果设置 Key 的时候指定了过期时间，Redis 会将这个 Key 直接加入到 expires 字典中，并将超时时间设置到该字典元素。
- **调用 expire 命令，**显式指定某个 Key 的过期时间。
- **恢复或修改数据，**从 Redis 持久化文件中恢复文件或者修改 Key，如果数据中的 Key 已经设置了过期时间，就将这个 Key 加入到 expires 字典中。

以上这些操作都会将过期的 Key 保存到 expires。Redis 会定期从 expires 字典中清理过期的 Key。

**Redis 清理过期 Key 的时机**

Redis 在启动的时候，会注册两种事件，一种是时间事件，另一种是文件事件。时间事件主要是 Redis 处理后台操作的一类事件，比如客户端超时、删除过期 Key；文件事件是处理请求。

在时间事件中，Redis 注册的回调函数是 serverCron，在定时任务回调函数中，通过调用 databasesCron 清理部分过期 Key。（这是定期删除的实现。） 

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData)
{
    …
    /* Handle background operations on Redis databases. */
    databasesCron();
    ...
}
```

每次访问 Key 的时候，都会调用 expireIfNeeded 函数判断 Key 是否过期，如果是，清理 Key。（这是惰性删除的实现）

```c
robj *lookupKeyRead(redisDb *db, robj *key) {
    robj *val;
    expireIfNeeded(db,key);
    val = lookupKey(db,key);
     ...
    return val;
}
```

每次事件循环执行时，主动清理部分过期 Key。（这也是惰性删除的实现）

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}

void beforeSleep(struct aeEventLoop *eventLoop) {
       ...
       /* Run a fast expire cycle (the called function will return
        - ASAP if a fast cycle is not needed). */
       if (server.active_expire_enabled && server.masterhost == NULL)
           activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);
       ...
   }
```

**过期策略的实现**

我们知道，Redis 是以单线程运行的，在清理 Key 时不能占用过多的时间和 CPU，需要在尽量不影响正常的服务情况下，进行过期 Key 的清理。

过期清理的算法如下：

- server.hz 配置了 serverCron 任务的执行周期，默认是 10，即 CPU 空闲时每秒执行十次。

- 每次清理过期 Key 的时间不能超过 CPU 时间的 25%：timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100。

  比如，如果 hz=1，一次清理的最大时间为 250ms，hz=10，一次清理的最大时间为 25ms。

- 如果是快速清理模式（在 beforeSleep 函数调用），则一次清理的最大时间是 1ms。

- 依次遍历所有的 DB。

- 从 DB 的过期列表中随机取 20 个 Key，判断是否过期，如果过期，则清理。

- 如果有 5 个以上的 Key 过期，则重复步骤 5，否则继续处理下一个 DB。

- 在清理过程中，如果达到 CPU 的 25% 时间，退出清理过程。

从实现的算法中可以看出，这只是基于概率的简单算法，且是随机的抽取，因此是无法删除所有的过期 Key，通过调高 hz 参数可以提升清理的频率，过期 Key 可以更及时的被删除，但 hz 太高会增加 CPU 时间的消耗。

 **删除 Key**

Redis 4.0 以前，删除指令是 del，del 会直接释放对象的内存，大部分情况下，这个指令非常快，没有任何延迟的感觉。

但是，如果删除的 Key 是一个非常大的对象，比如一个包含了千万元素的 Hash，那么删除操作就会导致单线程卡顿，Redis 的响应就慢了。

为了解决这个问题，在 Redis 4.0 版本引入了 unlink 指令，能对删除操作进行“懒”处理，将删除操作丢给后台线程，由后台线程来异步回收内存。

实际上，在判断 Key 需要过期之后，真正删除 Key 的过程是先广播 expire 事件到从库和 AOF 文件中，然后在根据 Redis 的配置决定立即删除还是异步删除。

如果是立即删除，Redis 会立即释放 Key 和 Value 占用的内存空间，否则，Redis 会在另一个 BIO 线程中释放需要延迟删除的空间。

**小结：**总的来说，Redis 的过期删除策略是在启动时注册了 serverCron 函数，每一个时间时钟周期，都会抽取 expires 字典中的部分 Key 进行清理，从而实现定期删除。

另外，Redis 会在访问 Key 时判断 Key 是否过期，如果过期了，就删除，以及每一次 Redis 访问事件到来时，beforeSleep 都会调用 activeExpireCycle 函数，在 1ms 时间内主动清理部分 Key，这是惰性删除的实现。

Redis 结合了定期删除和惰性删除，基本上能很好的处理过期数据的清理，但是实际上还是有点问题的。

如果过期 Key 较多，定期删除漏掉了一部分，而且也没有及时去查，即没有走惰性删除，那么就会有大量的过期 Key 堆积在内存中，导致 Redis 内存耗尽。

# 参考资料

https://mp.weixin.qq.com/s/tAF8j0T6bMLExUQMVri-Tw