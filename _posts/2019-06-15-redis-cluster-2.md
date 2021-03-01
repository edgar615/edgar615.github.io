---
layout: post
title: redis集群（2） - 槽位迁移
date: 2019-06-15
categories:
    - redis
comments: true
permalink: redis-cluster-2.html
---

# 1. **空槽道迁移**

空槽道迁移相对简单一些，因为槽道没有对应数据，因此不需要考虑数据的迁移，接着上面的结果，计划将131节点上1槽道，迁移到133节点上去。

（1）确认1槽道上是否有数据，没有数据才迁移。使用命令cluster getkeysinslot 槽道号 查找范围。

```
127.0.0.1:6379> CLUSTER COUNTKEYSINSLOT 1
(integer) 0
```

（2）在133节点执行命令cluster setslot 槽道号 importing 源节点id，将131节点上1槽道的状态变更为importing。

```
127.0.0.1:6379> cluster nodes
16a3e17c008eacf34eb3ab435b8da0d9197eee2d 192.168.159.132:6379@16379 master - 0 1614333712180 2 connected 5462-10922
07095e16b0f7325e6f3adb32839a07d65053cf0c 192.168.159.131:6379@16379 master - 0 1614333711173 1 connected 0-5461
f87dc3c42a062bdff943c02f4adfab31cf0f708f 192.168.159.133:6379@16379 myself,master - 0 1614333711000 0 connected 10923-16383
# 后面是131节点的ID
127.0.0.1:6379> cluster setslot 1 importing 07095e16b0f7325e6f3adb32839a07d65053cf0c
OK

127.0.0.1:6379> cluster nodes
16a3e17c008eacf34eb3ab435b8da0d9197eee2d 192.168.159.132:6379@16379 master - 0 1614333812000 2 connected 5462-10922
07095e16b0f7325e6f3adb32839a07d65053cf0c 192.168.159.131:6379@16379 master - 0 1614333812898 1 connected 0-5461
f87dc3c42a062bdff943c02f4adfab31cf0f708f 192.168.159.133:6379@16379 myself,master - 0 1614333808000 0 connected 10923-16383 [1-<-07095e16b0f7325e6f3adb32839a07d65053cf0c]

```

（3）登录需要导出的节点131，执行命令cluster setslot 槽道号 migrating 目标节点id，将131节点上的1槽道的状态变更为migrating。

```
127.0.0.1:6379> cluster nodes
16a3e17c008eacf34eb3ab435b8da0d9197eee2d 192.168.159.132:6379@16379 master - 0 1614333869103 2 connected 5462-10922
07095e16b0f7325e6f3adb32839a07d65053cf0c 192.168.159.131:6379@16379 myself,master - 0 1614333867000 1 connected 0-5461
f87dc3c42a062bdff943c02f4adfab31cf0f708f 192.168.159.133:6379@16379 master - 0 1614333868095 0 connected 10923-16383
127.0.0.1:6379> CLUSTER setslot 1 migrating f87dc3c42a062bdff943c02f4adfab31cf0f708f
OK
127.0.0.1:6379> cluster nodes
16a3e17c008eacf34eb3ab435b8da0d9197eee2d 192.168.159.132:6379@16379 master - 0 1614333907391 2 connected 5462-10922
07095e16b0f7325e6f3adb32839a07d65053cf0c 192.168.159.131:6379@16379 myself,master - 0 1614333901000 1 connected 0-5461 [1->-f87dc3c42a062bdff943c02f4adfab31cf0f708f]
f87dc3c42a062bdff943c02f4adfab31cf0f708f 192.168.159.133:6379@16379 master - 0 1614333908398 0 connected 10923-16383
```

（4）通知进行迁移的两个节点槽道迁移了，使用命令cluster setslot 槽道号 node  迁入节点id，需要在两个节点上均进行操作，这个时候两个节点保存槽道所有权的位序列（是一个2048位的byte数组）对应的位才会更改，133节点上1槽道号的二进制位数字由0变成1,131的则由1变成0。

```

# 迁入节点 133 注意：后面跟着迁入节点ID
127.0.0.1:6379> cluster setslot 1 node f87dc3c42a062bdff943c02f4adfab31cf0f708f
OK

# 迁出节点 131 后面跟着迁入节点ID
127.0.0.1:6379> cluster setslot 1 node f87dc3c42a062bdff943c02f4adfab31cf0f708f
OK

127.0.0.1:6379> cluster nodes
16a3e17c008eacf34eb3ab435b8da0d9197eee2d 192.168.159.132:6379@16379 master - 0 1614334091731 2 connected 5462-10922
07095e16b0f7325e6f3adb32839a07d65053cf0c 192.168.159.131:6379@16379 myself,master - 0 1614334091000 1 connected 0 2-5461
f87dc3c42a062bdff943c02f4adfab31cf0f708f 192.168.159.133:6379@16379 master - 0 1614334092737 0 connected 1 10923-16383

```

查看集群节点，发现1成功迁移，后面跟的id为迁入节点的id

# 2. **有数据的槽道迁移**

上面完成了一个空槽道的迁移，如果某个槽道有数据，除了迁移槽道外，还需要将数据一起迁移。

```
# 查看foo6的槽位
127.0.0.1:6379> cluster keyslot foo6
(integer) 1168

127.0.0.1:6379> cluster countkeysinslot 1168
(integer) 1
```

importing和migrating步骤和空槽位一样

在迁出节点迁移key，执行完毕后可以发现key已经被迁移

```
127.0.0.1:6379> migrate 192.168.159.133 6379 foo6 0 1000
OK
```

> http://doc.redisfans.com/key/migrate.html

通知进行迁移的两个节点槽道迁移了，和空槽位一样

**redis-cluster 数据迁移是同步的，在目标节点执行restore 指令到源节点删除key 之间，源节点主线程处于阻塞状态，直达key 被成功删除。**