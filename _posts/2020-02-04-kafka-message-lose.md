---
layout: post
title: kafka在什么情况下才能保证消息不丢失
date: 2020-02-04
categories:
    - kafka
comments: true
permalink: kafka-message-lose.html
---

**Kafka 只对“已提交”的消息（committed message）做有限度的持久化保证。**

- **已提交的消息** 当 Kafka 的若干个 Broker 成功地接收到一条消息并写入到日志文件后，它们会告诉生产者程序这条消息已成功提交。此时，这条消息在 Kafka 看来就正式变为“已提交”消息了。你可以选择只要有一个 Broker 成功保存该消息就算是已提交，也可以是令所有 Broker 都成功保存该消息才算是已提交。
- **有限度的持久化保证**  Kafka 不可能保证在任何情况下都做到不丢失消息。 Kafka 不丢消息是有前提条件的。假如你的消息保存在 N 个 Kafka Broker 上，那么这个前提条件就是这 N 个 Broker 中至少有 1 个存活。只要这个条件成立，Kafka 就能保证你的这条消息永远不会丢失。

# 1. Broker丢失消息

Broker丢失消息是由于Kafka本身的原因造成的，kafka为了得到更高的性能和吞吐量，将数据异步批量的存储在磁盘中。消息的刷盘过程，为了提高性能，减少刷盘次数，kafka采用了**批量刷盘**的做法。即，按照一定的消息量，和时间间隔进行刷盘。这种机制也是由于linux操作系统决定的。将数据存储到linux操作系统种，会先存储到页缓存（Page cache）中，按照时间或者其他条件进行刷盘（从page cache到file），或者通过fsync命令强制刷盘。数据在page  cache中时，如果系统挂掉，数据会丢失。

![](/assets/images/posts/kafka-message-lose/kafka-message-lose-2.png)

上图简述了broker写数据以及同步的一个过程。broker写数据只写到PageCache中，而pageCache位于内存。这部分数据在断电后是会丢失的。pageCache的数据通过linux的flusher程序进行刷盘。刷盘触发条件有三：

- 主动调用sync或fsync函数
- 可用内存低于阀值
- dirty data时间达到阀值。dirty是pagecache的一个标识位，当有数据写入到pageCache时，pagecache被标注为dirty，数据刷盘以后，dirty标志清除。

Broker配置刷盘机制，是通过调用fsync函数接管了刷盘动作。从单个Broker来看，pageCache的数据会丢失。

也就是说，理论上，要完全让kafka保证单个broker不丢失消息是做不到的，只能通过调整刷盘机制的参数缓解该情况。比如，减少刷盘间隔，减少刷盘数据量大小。时间越短，性能越差，可靠性越好（尽可能可靠）。

配置项

```
log.flush.interval.messages 消息达到多少条时将数据写入到日志文件
log.flush.interval.ms 当达到该时间时，强制执行一次flush  null
log.flush.scheduler.interval.ms	周期性检查，是否需要将信息flush
```

为了解决该问题，kafka通过producer和broker协同处理单个broker丢失参数的情况。一旦producer发现broker消息丢失，即可自动进行retry。除非retry次数超过阀值（可配置），消息才会丢失。此时需要生产者客户端手动处理该情况。那么producer是如何检测到数据丢失的呢？是通过ack机制，

在生产者客户端KafkaProducer，可以设置acks参数。然后这个参数实际上有三种常见的值可以设置，分别是：0、1 和 all。

- **acks=0**，生产者发送消息后不需要等待任何服务端的响应。即生产者只要把消息发送出去，不管数据有没有在Partition Leader上落到磁盘，就认为这个消息发送成功了。设置为0可以达到最大吞吐量，但是如果在消息写入kafka的过程中出现某些异常，导致kafka没有收到这条消息，消息就丢失了。
- **acks=1**，默认值为1，生产者发送消息之后，只要分区的leader副本成功写入消息，那么它就会收到来自服务端的成功响应。如果消息无法写入leader副本，必然在leader副本崩溃、重新选举的过程中，那么生产者会收到一个错误响应，为了避免消息丢失，生产者可以选择重发消息。如果消息写入leader并返回成功响应给生产者，但在被其他follower副本拉取之前leader崩溃，那么此时消息还是会丢失，因为新选举的leader没有这条消息。acks设置为1，是消息可靠性和吞吐量之间的折中方案
- **acks=-1或者acks=all** 生产者在消息发送后，需要等待ISR中的所有副本都成功写入消息后才能收到来自服务端的成功响应。

在其他配置环境相同的情况下，acks设置为-1(all)可以达到最强的可靠性。但这并不意味这消息就一定可靠，因为ISR中可能只有一个leader副本，这样就退化成了acks=1的情况，要获得更高的消息可靠性，需要配合`min.insync.replicas`的联动。该参数表示ISR中最少的副本数。如果不设置该值，ISR中的follower列表可能为空。此时相当于acks=1。

**acks=all，必须跟ISR列表里至少有2个以上的副本配合使用，起码是有一个Leader和一个Follower才可以。这样才能保证说写一条数据过去，一定是2个以上的副本都收到了才算是成功，此时任何一个副本宕机，不会导致数据丢失。**

# 1. 生产者程序丢失数据

目前 Kafka Producer 是异步发送消息的，也就是说如果你调用的是 producer.send(msg) 这个 API，那么它通常会立即返回，但此时你不能认为消息发送已成功完成。如果用这个方式，可能会导致消息没有发送成功：

- 网络抖动，导致消息压根就没有发送到 Broker 端；
- 消息本身不合格导致 Broker 拒绝接收（比如消息太大了，超过了 Broker 的承受能力）

**Producer 永远要使用带有回调通知的发送 API，也就是说不要使用 `producer.send(msg)`，而要使用 `producer.send(msg, callback)`。**callback（回调），它能准确地告诉你消息是否真的提交成功了。一旦出现消息提交失败的情况，你就可以有针对性地进行处理。

如果是因为那些瞬时错误，那么仅仅让 Producer 重试就可以了；如果是消息不合格造成的，那么可以调整消息格式后再次发送。总之，处理发送失败的责任在 Producer 端而非 Broker 端。

# 2. 消费者程序丢失数据

Consumer消费消息有下面几个步骤：

- 接收消息
- 处理消息
- 反馈“处理完毕”（commited）

Consumer的消费方式主要分为两种：

- 自动提交offset，Automatic Offset Committing
- 手动提交offset，Manual Offset Control

Consumer自动提交的机制是根据一定的时间间隔，将收到的消息进行commit。commit过程和消费消息的过程是异步的。

Consumer 端丢失数据主要体现在 Consumer 端要消费的消息不见了。Consumer 程序有个“位移”的概念，表示的是这个 Consumer 当前消费到的 Topic 分区的位置。下面这张图来自于官网，它清晰地展示了 Consumer 端的位移数据。

![](/assets/images/posts/kafka-message-lose/kafka-message-lose-1.png)

比如对于 Consumer A 而言，它当前的位移值就是 9；Consumer B 的位移值是 11。

这里的“位移”类似于我们看书时使用的书签，它会标记我们当前阅读了多少页，下次翻书的时候我们能直接跳到书签页继续阅读。正确使用书签有两个步骤：第一步是读书，第二步是更新书签页。如果这两步的顺序颠倒了，就可能出现这样的场景：当前的书签页是第 90 页，我先将书签放到第 100 页上，之后开始读书。当阅读到第 95 页时，我临时有事中止了阅读。那么问题来了，当我下次直接跳到书签页阅读时，我就丢失了第 96～99 页的内容，即这些消息就丢失了。

同理，Kafka 中 Consumer 端的消息丢失就是这么一回事。要对抗这种消息丢失，办法很简单：**维持先消费消息（阅读），再更新位移（书签）的顺序即可**。这样就能最大限度地保证消息不丢失。当然，这种处理方式可能带来的问题是**消息的重复处理**

除了上面所说的场景，其实还存在一种比较隐蔽的消息丢失场景。Consumer程序从 Kafka 获取到消息后开启了多个线程异步处理消息，而 Consumer 程序自动地向前更新位移。假如其中某个线程运行失败了，它负责的消息没有被成功处理，但位移已经被更新了，因此这条消息对于 Consumer 而言实际上是丢失了。

这个问题的解决方案也很简单：**如果是多线程异步处理消费消息，Consumer 程序不要开启自动提交位移，而是要应用程序手动提交位移。**

> 代码实现要注意，我之前这样处理经常导致OOM

# 4. 最佳实践

1. 不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。
2. 设置 acks = all。
3. 设置 retries 为一个较大的值。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了 retries > 0 的 Producer 能够自动重试消息发送，避免消息丢失。重发会有两个问题：1）消息顺序可能不会一致 2）由于网络问题，本来消息已经成功写入了但是没有成功响应给 producer，进行重试时就可能会出现 `消息重复`
4. 设置 unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么它一旦成为新的 Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false，即不允许这种情况的发生。
5. 设置 replication.factor >= 3。这也是 Broker 端的参数。其实这里想表述的是，最好将消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余。
6. 设置 min.insync.replicas > 1。这依然是 Broker 端参数，控制的是消息至少要被写入到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。
7. 确保 replication.factor > min.insync.replicas。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas + 1。
8. 确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设置成 false，并采用手动提交位移的方式。



# 参考资料

《Kafka核心技术与实战》