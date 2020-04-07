---
layout: post
title: redis性能优化
date: 2020-04-08
categories:
    - redis
comments: true
permalink: redis-optimize.html
---

## 慢查询

1. 通过`redis-cli --bigkeys`发现大对象
2. 避免使用算法复杂度高的命令

## CPU过高
1. 通过`redis-cli --stats`分析QPS是否到极限
2. 通过`info commandstats`分析命令不合理开销时间

## 持久化导致阻塞 
1. 使用`info stats`检查`lastest_fork_usec`指标，获取redis最近一次fork操作耗时,如果耗时很大，比如超过1秒，则需要作出优化调整：避免使用过大的内存示例和规避fork缓慢的操作系统
2. 检查日志是否有AOF的阻塞日志，然后通过`info persistence`检查`aof_delay_fsync`指标

  Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting     for fsync to complete, this may slow down Redis.

## 超过最大连接数
使用`info stats`查看rejected_connections指标，通过maxclients参数控制客户端最大连接数

根据进程ID检查swap信息

    [root@ihorn-dev redis-3.2.0]# cat /proc/2721/smaps | grep Swap
    Swap:                  0 kB
    Swap:                  0 kB
    Swap:                  0 kB
    Swap:                  0 kB

## 控制键的数量
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