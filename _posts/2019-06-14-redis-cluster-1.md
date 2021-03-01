---
layout: post
title: redis集群（1） - 部署
date: 2019-06-14
categories:
    - redis
comments: true
permalink: redis-cluster-1.html
---

使用redis集群时首先应该对[一致性hash算法](https://edgar615.github.io/consistent-hashing.html)有一定了解

Redis Cluser采用虚拟槽分区， 所有的键根据哈希函数映射到0~16383整数槽内， 计算公式： `lot=CRC16（key） &16383`。 每一个节点负责维护一部分槽以及槽所映射的键值数据。

> 为什么是16384（2^14）个？
在redis节点发送心跳包时需要把所有的槽放到这个心跳包里，以便让节点知道当前集群信息，16384=16k，在发送心跳包时使用bitmap压缩后是2k（2 * 8 (8 bit) * 1024(1k) = 2K），也就是说使用2k的空间创建了16k的槽数。
虽然使用CRC16算法最多可以分配65535（2^16-1）个槽位，65535=65k，压缩后就是8k（8 * 8 (8 bit) * 1024(1k) = 8K），也就是说需要需要8k的心跳包，作者认为这样做不太值得；并且一般情况下一个redis集群不会有超过1000个master节点，所以16k的槽位是个比较合适的选择。

# 启动节点
Redis集群一般由多个节点组成， 节点数量至少为6个(3个主，3个从)才能保证组成完整高可用的集群。 每个节点需要开启配置cluster-enabled yes， 让Redis运行在集群模式下。 建议为集群内所有节点统一目录， 一般划分三个目录：`conf`、`data`、`log`， 分别存放配置、 数据和日志相关文件
将`redis.conf`复制一个`redis-6379.conf到redis-6384.conf`，修改配置文件，开启集群

```
port 6379
logfile "/data/log/redis-6379.log"
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000
```
按照上面的配置准备6个配置文件redis-6379.conf到redis-6384.conf

启动6379
```
src/redis-server conf/redis-6379.conf
```
可以看到`/data/log/redis-6379.log`输出
```
No cluster configuration found, I'm 106be52ad427071bc37f9a7dfa610a1781e1ee89
```
可以看到redis-data目录下多了一个`nodes-6379.conf`文件(仅在没有集群配置文件的时候生成)

集群模式的Redis除了原有的配置文件之外又加了一份集群配置文件。当集群内节点信息发生变化， 如添加节点、 节点下线、 故障转移等。 节点会自动保存集群状态到配置文件中。 需要注意的是， Redis自动维护集群配置文件， 不要手动修改， 防止节点重启时产生集群信息错乱

`nodes-6379.conf`文件内容记录了集群初始状态， 这里最重要的是节点ID， 它是一个40位
16进制字符串， 用于唯一标识集群内一个节点， 之后很多集群操作都要借助于节点ID来完成。 需要注意是， 节点ID不同于运行ID。 节点ID在集群初始化时只创建一次， 节点重启时会加载集群配置文件进行重用， 而Redis的运行ID每次重启都会变化

```
106be52ad427071bc37f9a7dfa610a1781e1ee89 :0@0 myself,master - 0 0 0 connected
vars currentEpoch 0 lastVoteEpoch 0
```

可以通过`cluster nodes`命令获取集群状态，在启动6个节点后，运行`cluster nodes`

```
127.0.0.1:6379> cluster nodes
106be52ad427071bc37f9a7dfa610a1781e1ee89 :6379@16379 myself,master - 0 0 0 connected
```
可以看到每个节点只能识别出自己的信息

# 节点握手
节点握手是指一批运行在集群模式下的节点通过Gossip协议彼此通信，达到感知对方的过程。 节点握手是集群彼此通信的第一步， 由客户端发起命令： `cluster meet {ip} {port}`

```
127.0.0.1:6379> cluster meet 127.0.0.1 6380
OK
127.0.0.1:6379> cluster nodes
c0ade44291d1fe4c6136bd3790bc5ac6ea5e6c3c 127.0.0.1:6380@16380 master - 0 1560419951859 0 connected
106be52ad427071bc37f9a7dfa610a1781e1ee89 127.0.0.1:6379@16379 myself,master - 0 0 1 connected
127.0.0.1:6379> cluster meet 127.0.0.1 6381
OK
127.0.0.1:6379> cluster meet 127.0.0.1 6382
OK
127.0.0.1:6379> cluster meet 127.0.0.1 6383
OK
127.0.0.1:6379> cluster meet 127.0.0.1 6384
OK
```

再次运行`cluster nodes`，可以识别出其他节点的信息。
```
127.0.0.1:6379> cluster nodes
c0ade44291d1fe4c6136bd3790bc5ac6ea5e6c3c 127.0.0.1:6380@16380 master - 0 1560419964928 0 connected
7753ef9a24fc71db76c9f5649e6da744a0e24752 127.0.0.1:6383@16383 master - 0 1560419964000 4 connected
1ddfc7f8fcd78899b143aeea49edce8a508c4c17 127.0.0.1:6384@16384 master - 0 1560419962413 0 connected
216eddd754dd1ccf5f0c33e39a76fd33a9b16729 127.0.0.1:6382@16382 master - 0 1560419963922 3 connected
106be52ad427071bc37f9a7dfa610a1781e1ee89 127.0.0.1:6379@16379 myself,master - 0 1560419963000 1 connected
43edd3a40be29f836156700fba883d049c9c8396 127.0.0.1:6381@16381 master - 0 1560419962000 2 connected
```

通过集群内消息的传播，其他节点也识别到了集群内其他节点的信息

```
127.0.0.1:6380> cluster nodes
106be52ad427071bc37f9a7dfa610a1781e1ee89 127.0.0.1:6379@16379 master - 0 1560420050179 1 connected
43edd3a40be29f836156700fba883d049c9c8396 127.0.0.1:6381@16381 master - 0 1560420053193 2 connected
216eddd754dd1ccf5f0c33e39a76fd33a9b16729 127.0.0.1:6382@16382 master - 0 1560420052188 3 connected
c0ade44291d1fe4c6136bd3790bc5ac6ea5e6c3c 127.0.0.1:6380@16380 myself,master - 0 1560420050000 0 connected
1ddfc7f8fcd78899b143aeea49edce8a508c4c17 127.0.0.1:6384@16384 master - 0 1560420051184 5 connected
7753ef9a24fc71db76c9f5649e6da744a0e24752 127.0.0.1:6383@16383 master - 0 1560420051000 4 connected
```

通过`cluster info`命令我们可以发现集群仍然不能正常工作`cluster_state:fail`
```
127.0.0.1:6379> cluster info
cluster_state:fail
cluster_slots_assigned:0
cluster_slots_ok:0
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:0
cluster_current_epoch:5
cluster_my_epoch:1
cluster_stats_messages_ping_sent:207
cluster_stats_messages_pong_sent:210
cluster_stats_messages_meet_sent:5
cluster_stats_messages_sent:422
cluster_stats_messages_ping_received:210
cluster_stats_messages_pong_received:212
cluster_stats_messages_received:422
```
从输入内容可以看到，分配的槽是0，`cluster_slots_assigned:0`。由于目前所有的槽没有分配到节点， 因此集群无法完成槽到节点的映射。 只有当16384个槽全部分配给节点后， 集群才进入在线状态。

# 分配槽

```
src/redis-cli -p 6371 cluster addslots {0..5461}
src/redis-cli -p 6380 cluster addslots {5462..10922}
src/redis-cli -p 6381 cluster addslots {10923..16383}
```

再次查看集群状态，可以看到状态变为`ok`

```
src/redis-cli -p 6379 cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:5
cluster_my_epoch:1
cluster_stats_messages_ping_sent:333092
cluster_stats_messages_pong_sent:315657
cluster_stats_messages_meet_sent:5
cluster_stats_messages_sent:648754
cluster_stats_messages_ping_received:315657
cluster_stats_messages_pong_received:333097
cluster_stats_messages_received:648754

```

通过 `cluster nodes`可以看到槽的分配关系

```
1ddfc7f8fcd78899b143aeea49edce8a508c4c17 127.0.0.1:6384@16384 master - 0 1560754671157 5 connected
7753ef9a24fc71db76c9f5649e6da744a0e24752 127.0.0.1:6383@16383 master - 0 1560754674172 4 connected
43edd3a40be29f836156700fba883d049c9c8396 127.0.0.1:6381@16381 myself,master - 0 1560754671000 2 connected 10923-16383
c0ade44291d1fe4c6136bd3790bc5ac6ea5e6c3c 127.0.0.1:6380@16380 master - 0 1560754673000 0 connected 5462-10922
216eddd754dd1ccf5f0c33e39a76fd33a9b16729 127.0.0.1:6382@16382 master - 0 1560754673167 3 connected
106be52ad427071bc37f9a7dfa610a1781e1ee89 127.0.0.1:6379@16379 master - 0 1560754673000 1 connected 0-5461

```

作为一个完整的集群， 每个负责处理槽的节点应该具有从节点， 保证当它出现故障时可以自动进行故障转移。 集群模式下， Reids节点角色分为主节点和从节点。 首次启动的节点和被分配槽的节点都是主节点， 从节点负责复制主节点槽信息和相关的数据。 使用`cluster replicate {nodeId}`命令让一个节点成为从节点。 其中命令执行必须在对应的从节点上执行，**nodeId是要复制主节点的节点ID**

```
src/redis-cli -a tabao -p 6382 cluster replicate 106be52ad427071bc37f9a7dfa610a1781e1ee89
src/redis-cli -a tabao -p 6383 cluster replicate c0ade44291d1fe4c6136bd3790bc5ac6ea5e6c3c
src/redis-cli -a tabao -p 6384 cluster replicate 43edd3a40be29f836156700fba883d049c9c8396
```

再次查看集群状态

```
1ddfc7f8fcd78899b143aeea49edce8a508c4c17 127.0.0.1:6384@16384 slave 43edd3a40be29f836156700fba883d049c9c8396 0 1560755043086 5 connected
7753ef9a24fc71db76c9f5649e6da744a0e24752 127.0.0.1:6383@16383 slave c0ade44291d1fe4c6136bd3790bc5ac6ea5e6c3c 0 1560755044091 4 connected
43edd3a40be29f836156700fba883d049c9c8396 127.0.0.1:6381@16381 myself,master - 0 1560755041000 2 connected 10923-16383
c0ade44291d1fe4c6136bd3790bc5ac6ea5e6c3c 127.0.0.1:6380@16380 master - 0 1560755045095 0 connected 5462-10922
216eddd754dd1ccf5f0c33e39a76fd33a9b16729 127.0.0.1:6382@16382 slave 106be52ad427071bc37f9a7dfa610a1781e1ee89 0 1560755044000 3 connected
106be52ad427071bc37f9a7dfa610a1781e1ee89 127.0.0.1:6379@16379 master - 0 1560755042079 1 connected 0-5461

```

自此我们手动建立一个集群。 它由6个节点构成，3个主节点负责处理槽和相关数据， 3个从节点负责故障转移 

我们有可以通过`redis-trib`安装集群，这里就写了


# 参考资料

《Redis开发与运维》

