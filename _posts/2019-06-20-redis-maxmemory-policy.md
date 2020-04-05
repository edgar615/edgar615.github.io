---
layout: post
title: redis内存淘汰策略
date: 2019-06-20
categories:
    - redis
comments: true
permalink: redis-maxmemory-policy.html
---

Redis缓存淘汰策略与Redis键的[过期删除策略](https://edgar615.github.io/redis-expire-policy.html)并不完全相同，前者是在Redis内存使用超过一定值的时候（一般这个值可以配置）使用的淘汰策略；而后者是通过定期删除+惰性删除两者结合的方式进行内存淘汰的。

这里参照官方文档的解释重新叙述一遍过期删除策略：

> 当某个key被设置了过期时间之后，客户端每次对该key的访问（读写）都会事先检测该key是否过期，如果过期就直接删除；但有一些键只访问一次，因此需要主动删除，默认情况下redis每秒检测10次，检测的对象是所有设置了过期时间的键集合，每次从这个集合中随机检测20个键查看他们是否过期，如果过期就直接删除，如果删除后还有超过25%的集合中的键已经过期，那么继续检测过期集合中的20个随机键进行删除。这样可以保证过期键最大只占所有设置了过期时间键的25%。


# 溢出控制策略

当redis所用的内存达到maxmemory上限时会触发相应的溢出控制策略。具体策略由参数`maxmemory-policy`控制

redis支持下列策略

- noeviction 默认值，不会删除任何数据，拒绝所有写入操作并返回错误信息。此时redis只响应读操作
- volatile-lru 首先通过LRU算法从设置了过期时间的键集合中驱逐最久没有使用的键，直到腾出足够的空间为止。如果没有可删除的键对象，回退到noeviction策略
- allkeys-lru  首先通过LRU算法驱逐最久没有使用的键根据，不管数据有没有设置超时属性，直到腾出足够空间为止
- allkeys-random 随机删除所有键，直到腾出足够空间为止
- volatile-random 随机删除过期键，直到腾出足够空间为止
- volatile-ttl 根据键值对象的TTL过期时间，删除最近将要过期的数据，如果没有，回退到noeviction策略
- volatile-lfu  从所有配置了过期时间的键中驱逐使用频率最少的键，如果没有，回退到noeviction策略
- allkeys-lfu  从所有键中驱逐使用频率最少的键

在配置文件中，通过 `maxmemory-policy` 可以配置要使用哪一个淘汰机制。

可以通过`info stats`查看evicted_keys指标找出已删除的键数量

```
127.0.0.1:6379> info stats
# Stats
...
evicted_keys:0
...
```

## 什么时候会进行淘汰？

Redis 会在每一次处理命令的时候（processCommand 函数调用 freeMemoryIfNeeded）判断当前 Redis 是否达到了内存的最大限制，如果达到限制，则使用对应的算法去处理需要删除的 Key。

伪代码如下：

```c
int processCommand(client *c)
{
    ...
    if (server.maxmemory) {
        int retval = freeMemoryIfNeeded();  
    }
    ...
}
```

# LRU

驱逐过程可以这样理解:

    客户端执行一个命令, 导致 Redis 中的数据增加,占用更多内存。
    Redis 检查内存使用量, 如果超出 maxmemory 限制, 根据策略清除部分 key。
    继续执行下一条命令, 以此类推。

在这个过程中, 内存使用量会不断地达到 limit 值, 然后超过, 然后删除部分 key, 使用量又下降到 limit 值之下。

如果某个命令导致大量内存占用(比如通过新key保存一个很大的set), 在一段时间内, 可能内存的使用量会明显超过 maxmemory 限制。
近似LRU算法

Redis 使用的并不是完全LRU算法。自动驱逐的 key , 并不一定是最满足LRU特征的那个. 而是通过近似LRU算法, 抽取少量的 key 样本, 然后删除其中访问时间最古老的那个key。

驱逐算法, 从 Redis 3.0 开始得到了巨大的优化, 使用 pool(池子) 来作为候选. 这大大提升了算法效率, 也更接近于真实的LRU算法。

在 Redis 的 LRU 算法中, 可以通过设置样本(sample)的数量来调优算法精度。 通过以下指令配置:

```
maxmemory-samples 5
```

Redis维护了一个24位时钟，可以简单理解为当前系统的时间戳，每隔一定时间会更新这个时钟。每个key对象内部同样维护了一个24位的时钟，当新增key对象的时候会把系统的时钟赋值到这个内部对象时钟。比如我现在要进行LRU，那么首先拿到当前的全局时钟，然后再找到内部时钟与全局时钟距离时间最久的（差最大）进行淘汰，这里值得注意的是全局时钟只有24位，按秒为单位来表示才能存储194天，所以可能会出现key的时钟大于全局时钟的情况，如果这种情况出现那么就两个相加而不是相减来求最久的key。

在RedisObject结构体内部，有一个lru属性，用来记录对象最后一次被访问的时间
```
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    /* key对象内部时钟 */
    unsigned lru:LRU_BITS;
    int refcount;
    void *ptr;
} robj;
```
Redis中的LRU与常规的LRU实现并不相同，常规LRU会准确的淘汰掉队头的元素，但是Redis的LRU并不维护队列，只是根据配置的策略要么从所有的key中随机选择N个（N可以配置）要么从所有的设置了过期时间的key中选出N个键，然后再从这N个键中选出最久没有使用的一个key进行淘汰。

# LFU
LFU是在Redis4.0后出现的，LRU的最近最少使用实际上并不精确，考虑下面的情况，如果在|处删除，那么A距离的时间最久，但实际上A的使用频率要比B频繁，所以合理的淘汰策略应该是淘汰B。LFU就是为应对这种情况而生的。

LFU把原来的key对象的内部时钟的24位分成两部分，前16位还代表时钟，后8位代表一个计数器。16位的情况下如果还按照秒为单位就会导致不够用，所以一般这里以时钟为单位。而后8位表示当前key对象的访问频率，8位只能代表255，但是redis并没有采用线性上升的方式，而是通过一个复杂的公式，通过配置两个参数来调整数据的递增速度。

- lfu-log-factor 默认10
- lfu-decay-time 默认1

这两个参数的作用我没有仔细研究

在`redis.conf`中可以找到官方描述
```
# +--------+------------+------------+------------+------------+------------+
# | factor | 100 hits   | 1000 hits  | 100K hits  | 1M hits    | 10M hits   |
# +--------+------------+------------+------------+------------+------------+
# | 0      | 104        | 255        | 255        | 255        | 255        |
# +--------+------------+------------+------------+------------+------------+
# | 1      | 18         | 49         | 255        | 255        | 255        |
# +--------+------------+------------+------------+------------+------------+
# | 10     | 10         | 18         | 142        | 255        | 255        |
# +--------+------------+------------+------------+------------+------------+
# | 100    | 8          | 11         | 49         | 143        | 255        |
# +--------+------------+------------+------------+------------+------------+
#
# NOTE: The above table was obtained by running the following commands:
#
#   redis-benchmark -n 1000000 incr foo
#   redis-cli object freq foo
#
# NOTE 2: The counter initial value is 5 in order to give new objects a chance
# to accumulate hits.
#
# The counter decay time is the time, in minutes, that must elapse in order
# for the key counter to be divided by two (or decremented if it has a value
# less <= 10).
#
# The default value for the lfu-decay-time is 1. A Special value of 0 means to
# decay the counter every time it happens to be scanned
```

# 其他模块对过期键的处理
**生成RDB文件时**
执行 SAVE 或 BGSAVE 时 ，数据库键空间中的过期键不会被保存在RDB文件中

**载入RDB文件时**

- Master 载入RDB时，文件中的未过期的键会被正常载入，过期键则会被忽略。
- Slave 载入 RDB 时，文件中的所有键都会被载入，当同步进行时，会和Master 保持一致。

**AOF 文件写入时**

数据库键空间的过期键的过期但并未被删除释放的状态会被正常记录到 AOF 文件中，当过期键发生释放删除时，DEL 也会被同步到 AOF 文件中去。

**重新生成 AOF文件时**

执行 BGREWRITEAOF 时 ，数据库键中过期的键不会被记录到 AOF 文件中

**复制**

- Master 删除 过期 Key 之后，会向所有 Slave 服务器发送一个 DEL命令，从服务器收到之后，会删除这些 Key。
- Slave 在被动的读取过期键时，不会做出操作，而是继续返回该键，只有当Master 发送 DEL 通知来，才会删除过期键，这是统一、中心化的键删除策略，保证主从服务器的数据一致性。


# 参考资料

《Redis开发与运维》

https://www.jianshu.com/p/c8aeb3eee6bc

https://blog.csdn.net/qq_28018283/article/details/80764518