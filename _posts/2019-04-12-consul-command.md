---
layout: post
title: Consul - 其他命令行
date: 2019-04-12
categories:
    - Consul
comments: true
permalink: consul-command.html
---

除了agent、watch外，Consul还提供了很多命令行工具，目前没有怎么用到，先记录

# 1. force-leave

`force-leave`可以强制consul集群中的成员进入left状态(空闲状态)，记住，即使一个成员处于活跃状态，它仍旧可以再次加入集群中，这个方法的真实目的是强制移除failed的节点。如果failed的节点还是网络的一部分，则consul会周期性的重新链接failed的节点，如果经过一段时间后(默认是72小时)，consul则会宣布停止尝试链接failed的节点。force-leave指令可以快速的把failed节点转换到left状态。

```
-rpc-addr:一个rpc地址，agent可以链接上来发送命令，如果没有指定，默认是127.0.0.1:8400。
```

# 2. info

`info`提供了各种操作时可以用到的debug信息，对于client和server，info有返回不同的子系统信息，目前有以下几个KV信息：agent(提供agent信息)，consul(提供consul库的信息)，raft(提供raft库的信息)，serf_lan(提供LAN gossip pool),serf_wan(提供WAN gossip pool)

```
-rpc-addr：一个rpc地址，agent可以链接上来发送命令，如果没有指定，默认是127.0.0.1:8400
1
```

# 3. join

`join`告诉consul agent加入一个已经存在的集群中，一个新的consul  agent必须加入一个已经有至少一个成员的集群中，这样它才能加入已经存在的集群中，如果你不加入一个已经存在的集群，则agent是它自身集群的一部分，其他agent则可以加入进来。agents可以加入其他agent多次。consul join [options] address。如果你想加入多个集群，则可以写多个地址，consul会加入所有的地址。

```
-wan：agent运行在server模式，xxxxxxx
-rpc-addr：一个rpc地址，agent可以链接上来发送命令，如果没有指定，默认是127.0.0.1:8400。12
```

# 4. keygen

`keygen`生成加密的密钥，可以用在consul agent通讯加密

生成一个key

# 5. leave

`leave`触发一个优雅的离开动作并关闭agent，节点离开后不会尝试重新加入集群中。运行在server状态的节点，节点会被优雅的删除，这是很严重的，在某些情况下一个不优雅的离开会影响到集群的可用性。

```
-rpc-addr:一个rpc地址，agent可以链接上来发送命令，如果没有指定，默认是127.0.0.1:8400。1
```

# 6. members

`members`输出consul agent目前所知道的所有的成员以及它们的状态，节点的状态只有alive、left、failed三种状态。

```
-detailed：输出每个节点更详细的信息。
-rpc-addr：一个rpc地址，agent可以链接上来发送命令，如果没有指定，默认是127.0.0.1:8400。
-status：过滤出符合正则规则的节点
-wan：xxxxxx1234
```

# 7. monitor

`monitor`用来链接运行的agent，并显示日志。monitor会显示最近的日志，并持续的显示日志流，不会自动退出，除非你手动或者远程agent自己退出。

```
-log-level：显示哪个级别的日志，默认是info
-rpc-addr：一个rpc地址，agent可以链接上来发送命令，如果没有指定，默认是127.0.0.1:840012
```

# 8. reload

`reload`可以重新加载agent的配置文件。SIGHUP指令在重新加载配置文件时使用，任何重新加载的错误都会写在agent的log文件中，并不会打印到屏幕。

```
-rpc-addr：一个rpc地址，agent可以链接上来发送命令，如果没有指定，默认是127.0.0.1:84001
```

# 9. version

打印consul的版本

