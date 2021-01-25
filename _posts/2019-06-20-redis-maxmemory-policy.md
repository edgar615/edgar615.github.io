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

> 当某个key被设置了过期时间之后，客户端每次对该key的访问（读写）都会事先检测该key是否过期，如果过期就直接删除；但有一些键只访问一次，因此需要主动删除，默认情况下redis**每秒检测10次**，检测的对象是所有设置了过期时间的键集合，每次从这个集合中**随机检测20个键**查看他们是否过期，如果过期就直接删除，如果删除后还有**超过25%的集合中的键**已经过期，那么继续检测过期集合中的20个随机键进行删除。这样可以保证过期键最大只占所有设置了过期时间键的25%。


# 1. 溢出控制策略

当redis所用的内存达到maxmemory上限时会触发相应的溢出控制策略。具体策略由参数`maxmemory-policy`控制

redis支持下列策略

- noeviction 默认值，不会删除任何数据，拒绝所有写入操作并返回错误信息。**此时redis只响应读操作**
- volatile-lru 首先通过LRU算法从设置了过期时间的键集合中驱逐最久没有使用的键，直到腾出足够的空间为止。如果没有可删除的键对象，回退到noeviction策略
- allkeys-lru  首先通过LRU算法驱逐最久没有使用的键根据，不管数据有没有设置超时属性，直到腾出足够空间为止
- allkeys-random 随机删除所有键，直到腾出足够空间为止
- volatile-random 随机删除过期键，直到腾出足够空间为止
- volatile-ttl 根据键值对象的TTL过期时间，删除最近将要过期的数据，如果没有，回退到noeviction策略
- volatile-lfu  从所有配置了过期时间的键中驱逐使用频率最少的键，如果没有，回退到noeviction策略（4.0及以上版本可用）
- allkeys-lfu  从所有键中驱逐使用频率最少的键（4.0及以上版本可用）

在配置文件中，通过 `maxmemory-policy` 可以配置要使用哪一个淘汰机制。

可以通过`info stats`查看evicted_keys指标找出已删除的键数量

```
127.0.0.1:6379> info stats
# Stats
...
evicted_keys:0
...
```

**什么时候会进行淘汰？**

Redis 会在**每一次处理命令**的时候（processCommand 函数调用 freeMemoryIfNeeded）判断当前 Redis 是否达到了内存的最大限制，如果达到限制，则使用对应的算法去处理需要删除的 Key。

驱逐过程可以这样理解:

- 客户端执行一个命令, 导致 Redis 中的数据增加,占用更多内存。
- Redis 检查内存使用量, 如果超出 maxmemory 限制, 根据策略清除部分 key。
- 继续执行下一条命令, 以此类推。

在这个过程中, 内存使用量会不断地达到 limit 值, 然后超过, 然后删除部分 key, 使用量又下降到 limit 值之下。

如果某个命令导致大量内存占用(比如通过新key保存一个很大的set), 在一段时间内, 可能内存的使用量会明显超过 maxmemory 限制。

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

> noeviction 涉及的返回错误的写命令包含：
> set,setnx,setex,append,incr,decr,rpush,lpush,rpushx,lpushx,linsert,lset,rpoplpush,sadd,sinter,sinterstore,sunion,sunionstore,sdiff,sdiffstore,zadd,zincrby,zunionstore,zinterstore,hset,hsetnx,hmset,hincrby,incrby,decrby,getset,mset,msetnx,exec,sort。

# 2. LRU

LRU（Least Recently Used）最近最少使用。优先淘汰最近未被使用的数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”。

LRU 会把所有的数据组织成一个双向链表，链表的头和尾分别表示 MRU 端和 LRU 端，分别代表最近最常使用的数据和最近最不常用的数据。

![](/assets/images/posts/redis-maxmemory-policy/redis-maxmemory-policy-1.jpg)

们现在有数据 6、3、9、20、5。如果数据 20 和 3 被先后访问，它们都会从现有的链表位置移到 MRU 端，而链表中在它们之前的数据则相应地往后移一位。因为，LRU 算法选择删除数据时，都是从 LRU 端开始，所以把刚刚被访问的数据移到 MRU 端，就可以让它们尽可能地留在缓存中。

如果有一个新数据 15 要被写入缓存，但此时已经没有缓存空间了，也就是链表没有空余位置了，那么，LRU 算法做两件事：

- 数据 15 是刚被访问的，所以它会被放到 MRU 端；
- 算法把 LRU 端的数据 5 从缓存中删除，相应的链表中就没有数据 5 的记录了。

其实，LRU 算法背后的想法非常朴素：它认为刚刚被访问的数据，肯定还会被再次访问，所以就把它放在 MRU 端；长久不访问的数据，肯定就不会再被访问了，所以就让它逐渐后移到 LRU 端，在缓存满时，就优先删除它。

**为什么是  双向链表而不是单链表呢？**单链表可以实现头部插入新节点、尾部删除旧节点的时间复杂度都是O(1)，但是对于中间节点时间复杂度是O(n)，因为对于中间节点c，我们需要将该节点c移动到头部，此时只知道他的下一个节点，要知道其上一个节点需要遍历整个链表，时间复杂度为O(n)。

**近似LRU算法**

LRU 算法在实际实现时，需要用链表管理所有的缓存数据，这会带来额外的空间开销。而且，当有数据被访问时，需要在链表上把该数据移动到 MRU 端，如果有大量数据被访问，就会带来很多链表移动操作，会很耗时，进而会降低 Redis 缓存性能。

Redis 使用的并不是完全LRU算法。自动驱逐的 key , **并不一定是最满足LRU特征的那个**. 而是通过近似LRU算法, 抽取少量的 key 样本, 然后删除其中访问时间最古老的那个key。

Redis维护了一个24位时钟，可以简单理解为当前系统的时间戳，每隔一定时间会更新这个时钟。每个key对象内部同样维护了一个24位的时钟，当新增key对象的时候会把系统的时钟赋值到这个内部对象时钟。

在Redis中，Redis的key的底层结构是 redisObject，redisObject 中 lru:LRU_BITS  字段用于记录该key最近一次被访问时的Redis时钟  server.lruclock（Redis在处理数据时，都会调用lookupKey方法用于更新该key的时钟）。

> server.lruclock 实际是一个 24bit 的整数，默认是 Unix 时间戳对 2^24 取模的结果，其精度是毫秒。

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

**Redis 近似LRU 淘汰策略逻辑：**

- **首次淘汰：随机抽样选出【最多N个数据】放入【待淘汰数据池 evictionPoolEntry】；**

- - 数据量N：由 redis.conf 配置的 `maxmemory-samples` 决定，默认值是5，配置为10将非常接近真实LRU效果，但是更消耗CPU；
  - samples：n.样本；v.抽样；

- **再次淘汰：随机抽样选出【最多N个数据】，只要数据比【待淘汰数据池 evictionPoolEntry】中的【任意一条】数据的 lru 小，则将该数据填充至 【待淘汰数据池】；**

- - evictionPoolEntry 的容容量是 EVPOOL_SIZE = 16；
  - 详见 源码 中 evictionPoolPopulate 方法的注释；

- **执行淘汰：挑选【待淘汰数据池】中 lru 最小的一条数据进行淘汰；**

  Redis为了避免长时间或一直找不到足够的数据填充【待淘汰数据池】，代码里（dictGetSomeKeys 方法）强制写死了单次寻找数据的最大次数是 [maxsteps = count*10; ]，count 的值其实就是  maxmemory-samples。从这里我们也可以获得另一个重要信息：**单次获取的数据可能达不到 maxmemory-samples 个**。此外，如果Redis数据量（所有数据 或 有过期时间 的数据）本身就比 maxmemory-samples 小，那么 count 值等于 Redis 中数据量个数。

Redis中的LRU与常规的LRU实现并不相同，常规LRU会准确的淘汰掉队头的元素，但是Redis的LRU并不维护队列，只是根据配置的策略要么从所有的key中随机选择N个（N可以配置）要么从所有的设置了过期时间的key中选出N个键，然后再从这N个键中选出最久没有使用的一个key进行淘汰。

# 3. LFU

**LFU：Least Frequently Used，使用频率最少的（最不经常使用的）**优先淘汰最近使用的少的数据，其核心思想是“如果一个数据在最近一段时间很少被访问到，那么将来被访问的可能性也很小”。

**LFU与LRU的区别**

如果一条数据仅仅是突然被访问（有可能后续将不再访问），在 LRU 算法下，此数据将被定义为热数据，最晚被淘汰。但实际生产环境下，我们很多时候需要计算的是一段时间下key的访问频率，淘汰此时间段内的冷数据。

**算法原理**

LFU把原来的key对象的内部时钟（ lru:LRU_BITS 字段）的24位分成两部分，

- **Ldt**：last decrement time，16位，精度分钟，存储上一次 LOG_C 更新的时间。
- **LOG_C**：logarithmic counter，8位，最大255，存储key的**访问频率**，随时间衰减。

**为什么 LOG_C 要随时间衰减？**比如在秒杀场景下，热key被访问次数很大，如果不随时间衰减，此部分key将一直存放于内存中。

在实际应用中，一个数据可能会被访问成千上万次。如果每被访问一次，LOG_C 值就加 1 的话，那么，只要访问次数超过了 255，数据的 counter 值就一样了。在进行数据淘汰时，LFU 策略就无法很好地区分并筛选这些数据，反而还可能会把不怎么访问的数据留存在了缓存中。

假设第一个数据 A 的累计访问次数是 256，访问时间戳是 202010010909，所以它的 counter 值为 255，而第二个数据 B 的累计访问次数是 1024，访问时间戳是 202010010810。如果 counter 值只能记录到 255，那么数据 B 的 counter 值也是 255。此时，缓存写满了，Redis 使用 LFU 策略进行淘汰。数据 A 和 B 的 counter 值都是 255，LFU 策略再比较 A 和 B 的访问时间戳，发现数据 B 的上一次访问时间早于 A，就会把 B 淘汰掉。但其实数据 B 的访问次数远大于数据 A，很可能会被再次访问。这样一来，使用 LFU 策略来淘汰数据就不合适了。

因此，在实现 LFU 策略时，Redis 并没有采用数据每被访问一次，就给对应的 counter 值加 1 的计数规则（线性上升），而是采用了一个更优化的计数规则。而是通过一个复杂的公式，通过配置两个参数来调整数据的递增速度。

- lfu-log-factor 默认10
- lfu-decay-time 默认1

简单来说，LFU 策略实现的计数规则是：每当数据被访问一次时，首先，用计数器当前的值乘以配置项 lfu_log_factor 再加 1，再取其倒数，得到一个 p 值；然后，把这个 p 值和一个取值范围在（0，1）间的随机数 r 值比大小，只有 p 值大于 r 值时，计数器才加 1。

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

可以看到，当 **lfu_log_factor** 取值为 1 时，实际访问次数为 100K 后，LOG_C  值就达到 255 了，无法再区分实际访问次数更多的数据了。而当 lfu_log_factor 取值为 100 时，当实际访问次数为 10M 时，LOG_C  值才达到 255，此时，实际访问次数小于 10M 的不同数据都可以通过 LOG_C  值区分出来。

正是因为使用了非线性递增的计数器方法，即使缓存数据的访问次数成千上万，LFU 策略也可以有效地区分不同的访问次数，从而进行合理的数据筛选。从刚才的表中，我们可以看到，当 lfu_log_factor 取值为 10 时，百、千、十万级别的访问次数对应的 LOG_C  值已经有明显的区分了，**所以，我们在应用 LFU 策略时，一般可以将 lfu_log_factor 取值为 10。**

在一些场景下，有些数据在短时间内被大量访问后就不会再被访问了。那么再按照访问次数来筛选的话，这些数据会被留存在缓存中，但不会提升缓存命中率。为此，Redis 在实现 LFU 策略时，还设计了一个 LOG_C  值的衰减机制。

LFU 策略使用衰减因子配置项 **lfu_decay_time** 来控制访问次数的衰减。LFU 策略会计算当前时间和数据最近一次访问时间的差值，并把这个差值换算成以**分钟**为单位。然后，LFU 策略再把这个差值除以 **lfu_decay_time** 值，所得的结果就是数据 LOG_C  要衰减的值。

假设 lfu_decay_time 取值为 1，如果数据在 N 分钟内没有被访问，那么它的访问次数就要减 N。**如果 lfu_decay_time 取值更大，那么相应的衰减值会变小，衰减效果也会减弱**。所以，如果业务应用中有短时高频访问的数据的话，建议把 lfu_decay_time 值设置为 1，这样一来，LFU 策略在它们不再被访问后，会较快地衰减它们的访问次数，尽早把它们从缓存中淘汰出去，避免缓存污染。

# 4.  使用建议

- 优先使用 allkeys-lru 策略。这样，可以充分利用 LRU 这一经典缓存算法的优势，把最近最常访问的数据留在缓存中，提升应用的访问性能。如果你的业务数据中有明显的冷热数据区分，我建议你使用 allkeys-lru 策略。
- 如果业务应用中的数据访问频率相差不大，没有明显的冷热数据区分，建议使用 allkeys-random 策略，随机选择淘汰的数据就行。
- 如果你的业务中有置顶的需求，比如置顶新闻、置顶视频，那么，可以使用 volatile-lru 策略，同时不给这些置顶数据设置过期时间。这样一来，这些需要置顶的数据一直不会被删除，而其他数据会在过期时根据 LRU 规则进行筛选。
- 在实际业务应用中，LRU 和 LFU 两个策略都有应用。LRU 和 LFU 两个策略关注的数据访问特征各有侧重，L**RU 策略更加关注数据的时效性，而 LFU 策略更加关注数据的访问频次。**通常情况下，实际应用的负载具有较好的时间局部性，所以 LRU 策略的应用会更加广泛。但是，在扫描式查询的应用场景中，LFU 策略就可以很好地应对**缓存污染**问题了，建议你优先使用。

# 5. 其他模块对过期键的处理

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


# 6. 参考资料

《Redis开发与运维》

《Redis核心技术与实战》

https://www.jianshu.com/p/c8aeb3eee6bc

https://blog.csdn.net/qq_28018283/article/details/80764518