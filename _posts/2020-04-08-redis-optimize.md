---
layout: post
title: redis性能优化
date: 2020-04-08
categories:
    - redis
comments: true
permalink: redis-optimize.html
---

# 大对象

1. 通过`redis-cli --bigkeys`发现大对象
2. 避免使用算法复杂度高的命令

# CPU过高
1. 通过`redis-cli --stats`分析QPS是否到极限
2. 通过`info commandstats`分析命令不合理开销时间

# 持久化导致阻塞 
1. 使用`info stats`检查`lastest_fork_usec`指标，获取redis最近一次fork操作耗时,如果耗时很大，比如超过1秒，则需要作出优化调整：避免使用过大的内存示例和规避fork缓慢的操作系统
2. 检查日志是否有AOF的阻塞日志，然后通过`info persistence`检查`aof_delay_fsync`指标

  Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting     for fsync to complete, this may slow down Redis.

# 超过最大连接数

连接的客户端数量，可通过命令src/redis-cli info Clients | grep connected_clients得到，这个值跟使用redis的服务的连接池配置关系比较大，所以在监控这个字段的值时需要注意。另外这个值也不能太大，建议不要超过5000，如果太大可能是redis处理太慢，那么需要排除问题找出原因。

另外还有一个拒绝连接数（rejected_connections）也需要关注（使用`info stats`查看），这个值理想状态是0。如果大于0，说明创建的连接数超过了maxclients，需要排查原因。是redis连接池配置不合理还是连接这个redis实例的服务过多等。

根据进程ID检查swap信息

    [root@ihorn-dev redis-3.2.0]# cat /proc/2721/smaps | grep Swap
    Swap:                  0 kB
    Swap:                  0 kB
    Swap:                  0 kB
    Swap:                  0 kB

# 阻塞客户端数量

blocked_clients，一般是执行了list数据类型的BLPOP或者BRPOP命令引起的，可通过命令`src/redis-cli info Clients | grep blocked_clients`得到，很明显，这个值最好应该为0。

# 监控内存峰值

Redis可以通过命令`config set maxmemory 10737418240`设置允许使用的最大内存（强烈建议不要超过20G），为了防止发生swap导致Redis性能骤降，甚至由于使用内存超标导致被系统kill，建议`used_memory_peak`的值与maxmemory的值有个安全区间，例如1G，那么`used_memory_peak`的值不能超过9663676416（9G）

Redis 使用 maxmemory 参数限制最大可用内存。限制内存的目的主要有:

- 用于缓存场景，当超出内存上限 maxmemory 时使用 LRU 等删除策略释放空间。
- 防止所用的内存超过服务器物理内存，导致 OOM 后进程被系统杀死。

maxmemory 限制的是 Redis 实际使用的内存量，也就是 used_memory 统计项对应的内存。实际消耗的内存可能会比 maxmemory 设置的大，要小心因为这部内存导致 OOM。所以，如果你有 10GB 的内存，最好将 maxmemory 设置为 8 或者 9G

# 监控内存碎片率

在理想情况下， used_memory_rss（从操作系统的角度，Redis 已分配的内存总量） 的值应该只比 used_memory（由 Redis 分配器分配的内存总量） 稍微高一点儿。 当 rss > used ，且两者的值相差较大时，表示存在（内部或外部的）内存碎片。 内存碎片的比率可以通过 `mem_fragmentation_ratio` 的值看出。 当 used > rss 时，表示 Redis 的部分内存被操作系统换出到交换空间了，在这种情况下，操作可能会产生明显的延迟。

```plaintext
Because Redis does not have control over how its allocations are mapped to memory pages, high used_memory_rss is often the result of a spike in memory usage.
```

当 Redis 释放内存时，分配器可能会，也可能不会，将内存返还给操作系统。 如果 Redis 释放了内存，却没有将内存返还给操作系统，那么 used_memory 的值可能和操作系统显示的 Redis 内存占用并不一致。 查看 used_memory_peak 的值可以验证这种情况是否发生。

1. 当这个值大于1时，表示分配的内存超过实际使用的内存，数值越大，碎片率越严重。
2. 当这个值小于1时，表示发生了swap，即可用内存不够。由于硬盘速度远远慢于内存，Redis 性能会变得很差，甚至僵死

另外需要注意的是，当内存使用量（used_memory）很小的时候，这个值参考价值不大。所以，建议used_memory至少1G以上才考虑对内存碎片率进行监控。

当 Redis 内存超出可以获得内存时，操作系统会进行 swap，将旧的页写入硬盘。从硬盘读写大概比从内存读写要慢5个数量级。used_memory 指标可以帮助判断 Redis 是否有被swap的风险或者它已经被swap。

> 在 Redis Administration 一文 建议要设置和内存一样大小的交换区，如果没有交换区，一旦 Redis 突然需要的内存大于当前操作系统可用内存时，Redis 会因为 out of memory 而被 Linix Kernel 的 OOM Killer 直接杀死。虽然当 Redis 的数据被换出 (swap out) 时，Redis的性能会变差，但是总比直接被杀死的好。

redis4.0有一个主要特性就是优化内存碎片率问题（Memory de-fragmentation）。在redis.conf配置文件中有介绍即ACTIVE DEFRAGMENTATION：碎片整理允许Redis压缩内存空间，从而回收内存。这个特性默认是关闭的，可以通过命令`CONFIG SET activedefrag yes`热启动这个特性。

Redis 默认的内存分配器采用 jemalloc，可选的分配器还有：glibc、tcmalloc。内存分配器为了更好地管理和重复利用内存，分配内存策略一般采用固定范围的内存块进行分配。具体的分配策略后续会具体讲解，但是 Redis 正常碎片率一般在 1.03 左右(为什么是这个值)。但是当存储的数据长度长度差异较大时，以下场景容易出现高内存碎片问题：

- 频繁做更新操作，例如频繁对已经存在的键执行 append、setrange 等更新操作。
- 大量过期键删除，键对象过期删除后，释放的空间无法得到重复利用，导致碎片率上升。

# 缓存命中率

`keyspace_misses/keyspace_hits`这两个指标用来统计缓存的命令率，`keyspace_misses`指未命中次数，`keyspace_hits`表示命中次数。`keyspace_hits/(keyspace_hits+keyspace_misses)`就是缓存命中率。视情况而定，建议0.9以上，即缓存命中率要超过90%。如果缓存命中率过低，那么要排查对缓存的用法是否有问题！

# 失效KEY

如果把Redis当缓存使用，那么建议所有的key都设置了expire属性，通过命令src/redis-cli info Keyspace得到每个db中key的数量和设置了expire属性的key的属性，且expires需要等于keys

# 慢日志

通过命令`slowlog get`得到Redis执行的slowlog集合，理想情况下，slowlog集合应该为空，即没有任何慢日志，不过，有时候由于网络波动等原因造成`set key value`这种命令执行也需要几毫秒，在监控的时候我们需要注意，而不能看到slowlog就想着去优化，简单的set/get可能也会出现在slowlog中。

```
# The following time is expressed in microseconds, so 1000000 is equivalent to one second. Note that a negative number disables the slow log, while a value of zero forces the logging of every command.
# 阈值，单位微秒，默认值1秒，为0时会记录所有命令，为-1时不会记录所有命令
slowlog-log-slower-than 10000 

# 最多存储多少条慢查询日志，默认值128，redis使用一个list类型存储日志
# There is no limit to this length. Just be aware that it will consume memory. You can reclaim memory used by the slow log with SLOWLOG RESET.
slowlog-max-len 128 
```



#  控制键的数量

当使用redis存储大量数据的时候，通常会存在大量键，过多的键会销毁大量内存。对应存储相同的数据内容可以利用redis的数据结构降低外层键的数量，可以节约大量内存。例如可以把大量键分组映射到多个hash结构中降低键的数量。（注意hash中value不要超过hash-max-ziplist-value的设置，如果使用hashtable反而会增加内存消耗）
注意：

- hash类型节省内存的原理是使用ziplist编码，如果使用hashtable编码方式反而会增加内存消耗。
- ziplist长度需要控制在1000以内，否则由于存取操作时间复杂度在O(n)到O(n2)之间，长列表会导致CPU消耗严重，得不偿失。
- ziplist适合存储的小对象，对于大对象不但内存优化效果不明显还会增加命令操作耗时。
- 需要预估键的规模，从而确定每个hash结构需要存储的元素数量。
- 根据hash长度和元素大小，调整hash-max-ziplist-entries和hash-max-ziplist-value参数，确保hash类型使用ziplist编码。

关于hash键和field键的设计：

- 当键离散度较高时，可以按字符串位截取，把后三位作为哈希的field，之前部分作为哈希的键。如：key=1948480 哈希key=group:hash:1948，哈希field=480。
- 当键离散度较低时，可以使用哈希算法打散键，如:使用crc32(key)&10000函数把所有的键映射到“0-9999”整数范围内，哈希field存储键的原始值。
- 尽量减少hash键和field的长度，如使用部分键内容。

使用hash结构控制键的规模虽然可以大幅降低内存，但同样会带来问题，需要提前做好规避处理。如下:

- 客户端需要预估键的规模并设计hash分组规则，加重客户端开发成本。
- hash重构后所有的键无法再使用超时(expire)和LRU淘汰机制自动删除，需要手动维护删除。
- 对于大对象，如1KB以上的对象。使用hash-ziplist结构控制键数量。
- 不过瑕不掩瑜，对于大量小对象的存储场景，非常适合使用ziplist编码的hash类型控制键的规模来降低内存。
	
使用ziplist+hash优化keys后，如果想使用超时删除功能，开发人员可以存储每个对象写入的时间，再通过定时任