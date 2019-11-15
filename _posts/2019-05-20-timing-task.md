---
layout: post
title: 定时任务的简易实现
date: 2019-05-20
categories:
    - 设计
comments: true
permalink: timing-task.html
---

最近做物联网类的项目，有个需求是要求实现设备的心跳监测**设备每15秒发送一次心跳，服务端如果30秒未收到设备的心跳则需要将设备标记为离线**

# 轮询法
最早想到的方案是使用timer或者quartz定时扫描设备表
1. 每次收到心跳后更新设备的上次心跳时间
2. timer每隔30秒搜索一次数据库，找出上次心跳时间在30之前的任务，标记为下线，并通知用户

但这个方案在数据量较大的时候效率比较低

# Redis Keyspace Notifications
在 Redis 里面有一些事件，比如键到期、键被删除等。然后我们可以通过配置一些东西来让 Redis 一旦触发这些事件的时候就往特定的channel推一条消息。

Redis默认是关闭这个功能的，我们可以在redis.conf文件中修改 `notify-keyspace-events` 配置，其值可以是 `Ex`、`Klg` 等等。这些字母的具体含义如下所示：
- **K**，表示 `keyspace` 事件，有这个字母表示会往 `__keyspace@<db>__` 频道推消息。
- **E**，表示 `keyevent` 事件，有这个字母表示会往 `__keyevent@<db>__` 频道推消息。
- **g**，表示一些通用指令事件支持，如 `DEL`、`EXPIRE`、`RENAME` 等等。
- **$**，表示字符串（String）相关指令的事件支持。
- **l**，表示列表（List）相关指令事件支持。
- **s**，表示集合（Set）相关指令事件支持。
- **h**，哈希（Hash）相关指令事件支持。
- **z**，有序集（Sorted Set）相关指令事件支持。
- **x**，过期事件，与 **g** 中的 `EXPIRE` 不同的是，**g** 的 `EXPIRE` 是指执行 `EXPIRE key ttl` 这条指令的时候顺便触发的事件，而这里是指那个 `key` 刚好过期的这个时间点触发的事件。
- **e**，驱逐事件，一个 `key` 由于内存上限而被驱逐的时候会触发的事件。
- **A**，`g$lshzxe` 的别名。也就是说 `AKE` 的意思就代表了所有的事件

如果我们删除了在 db 0 中一个叫 `foo` 的键，那么系统会往两个频道推消息，一个是 `del` 事件频道推 `foo` 消息，另一个是 `foo` 频道推 `del` 消息，它们小俩口被系统推送的指令分别等价于：

```
PUBLISH __keyspace@0__:foo del
PUBLISH __keyevent@0__:del foo
```

其中往 `foo` 推送 `del` 的频道名为 `__keyspace@0__:foo`，即是 `"__keyspace@" + DB_NUMBER + "__:" + KEY_NAME`；而 `del` 的频道名为 `"__keyevent@" + DB_NUMBER + "__:" + EVENT_NAME`



针对我们的需求，只需要开启字符串删除、过期的`keyevent`事件

```
notify-keyspace-events "Eg$x
```

我们为每个设备维护两个key

- device:connected:<设备ID> 记录设备的心跳信息，如果存在这个键，说明设备的心跳正常，设置过期时间为30秒
- device:disconnected:<设备ID> 记录设备的离线信息，如果存在这个键，说明设备已经离线


1. 当收到设备的心跳时，设置设备的连接key

```
setex device:connected:<设备ID> 30 <设备ID>
del device:disconnected:<设备ID>
```

2. 订阅`io.vertx.redis.__keyevent@0__:expired`通道，当收到`device:connected:*`事件时，说明有设备超过30秒没有心跳，通过解析key，算出离线的设备ID（消息中不会有数据的value）

3. 订阅`io.vertx.redis.__keyevent@0__:del`通道，当收到`device:disconnected:*`事件时，说明有设备从离线中恢复

这个实现有个较大的问题是如果订阅方漏掉了对应的消息，那么设备可能永远不会上线或掉线

# 时间轮算法

首先我们维护一个31个槽(与心跳检测事件有关)的环形队列，每个槽内存放任务的set集合，定时器从0开始，每秒扫描一个槽

![](/assets/images/posts/timing-wheel/timingwheel_1.png)

在第一秒游标指向第0个槽，收到设备A和设备B的心跳，将设备A，设备B存入第31个槽（30秒后的第1个槽）

![](/assets/images/posts/timing-wheel/timingwheel_2.png)

在第4秒收到设备A的心跳，将设备A从第31个槽移动到当前游标的上一个槽(即第3个槽)

*PS：因为图的原因这里按第4秒算的，严格来说应该是第15秒*

![](/assets/images/posts/timing-wheel/timingwheel_3.png)

在第31秒指针走到第31个槽，目前还存在这个槽内的所有设备表明在之前的30秒没有收到过任何心跳

![](/assets/images/posts/timing-wheel/timingwheel_4.png)

与前面的方案相比，时间轮的优势在于只需要一个定时器，而且每秒只会触发一次，效率很高。

上述方案的实现都比较简单，这里就不上源码了

# 深入一步

时间轮除了可以用来检测心跳外，还可以用来实现定时任务，只需要在时间轮中的set集合中存放的值增加一个属性：循环次数round

![](/assets/images/posts/timing-wheel/timingwheel_5.png)

假设我们是时间轮是1分钟，当前游标在0，我们要在5分20秒后执行某个定时任务，通过计算可以得出任务需要保存在第20个槽内，循环次数为5。游标每走完一圈就需要将循环次数-1，如果此时槽内集合的任务如果循环次数小于0，就说明任务是到期的任务。

如果任务的时间跨度很大，数量也多，传统的时间轮会造成单个槽的任务集合很长，并会维持很长一段时间。这时可将时间轮按时间粒度分级。

![](/assets/images/posts/timing-wheel/timingwheel_6.png)

每个任务除了要维护在当前轮子的round，还要计算在所有下级轮子的round。当本层的round为0时，任务按下级round值被下放到下级轮子，最终在最底层的轮子得到执行。

这种方式的优点在于能够保证任务链表的长度一直在比较短的状态，但缺点是需要更多的空间

# 适用场景

1. 下单之后如果三十分钟之内或12小时没有付款就自动取消订单
2. 下单成功后60s之后给用户发送短信通知
3. 用户希望通过手机远程遥控家里的智能设备在指定的时间进行工作。这时候就可以将用户指令发送到延时队列，当指令设定的时间到了再将指令推送到只能设备。
4. 七天自动收货
5. 一定时间后自动评价
6. 业务执行失败之后隔10分钟重试一次


# 参考资料

https://mp.weixin.qq.com/s/sVzs8vDlTH9xySXwmeb63w