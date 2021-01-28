---
layout: post
title: kafka生产者消息分区原理
date: 2020-02-03
categories:
    - kafka
comments: true
permalink: kafka-partition.html
---

# 1. 分区

Kafka中主题（Topic）是承载真实数据的逻辑容器，而在主题之下还分为若干个分区，也就是说 Kafka 的消息组织方式实际上是三级结构：主题 - 分区 - 消息。主题下的每条消息只会保存在某一个分区中，而不会在多个分区中被保存多份。官网上的这张图非常清晰地展示了 Kafka 的三级结构，如下所示：

![](/assets/images/posts/kafka-partition/kafka-partition-1.png)

分区的作用就是提供负载均衡的能力，实现系统的高伸缩性（Scalability）。不同的分区能够被放置到不同节点的机器上，而数据的读写操作也都是针对分区这个粒度而进行的，这样每个节点的机器都能独立地执行各自分区的读写请求处理。并且，我们还可以通过添加新的节点机器来增加整体系统的吞吐量。

在通过 API 方式发布消息时，生产者是以 Record 为消息进行发布的。Record 中包含 Key 与 Value，Value 才是我们真正的消息本身，而 Key 用于路由消息所要存放的 Partition。

消息要写入到哪个 Partition 并不是随机的，而是有路由策略的：

- 若指定了 Partition，则直接写入到指定的 Partition。
- 若未指定 Partition 但指定了 Key，则通过对 Key 的 Hash 值与 Partition 数量取模，结果就是要选出的 Partition 索引。
- 若 Partition 和 Key 都未指定，则使用轮询算法选出一个 Partition。

如果要自定义分区策略，你需要显式地配置生产者端的参数partitioner.class。方法很简单，在编写生产者程序时，你可以编写一个具体的类实现org.apache.kafka.clients.producer.Partitioner接口。

# 2. 轮询策略

也称 Round-robin 策略，即顺序分配。比如一个主题下有 3 个分区，那么第一条消息被发送到分区 0，第二条被发送到分区 1，第三条被发送到分区 2，以此类推。当生产第 4 条消息时又会重新开始，即将其分配到分区 0，就像下面这张图展示的那样。

![](/assets/images/posts/kafka/kafka-4.png)

这就是所谓的轮询策略。轮询策略是 Kafka Java 生产者 API 默认提供的分区策略。如果你未指定partitioner.class参数，那么你的生产者程序会按照轮询的方式在主题的所有分区间均匀地“码放”消息。

轮询策略有非常优秀的负载均衡表现，它总是能保证消息最大限度地被平均分配到所有分区上，故默认情况下它是最合理的分区策略，也是我们最常用的分区策略之一。

# 3. 随机策略

也称 Randomness 策略。所谓随机就是我们随意地将消息放置到任意一个分区上，如下面这张图所示。

![](/assets/images/posts/kafka/kafka-5.png)

如果要实现随机策略版的 partition 方法

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return ThreadLocalRandom.current().nextInt(partitions.size());
```

本质上看随机策略也是力求将数据均匀地打散到各个分区，但从实际表现来看，它要逊于轮询策略，所以如果追求数据的均匀分布，还是使用轮询策略比较好。事实上，随机策略是老版本生产者使用的分区策略，在新版本中已经改为轮询了。

# 4. 按消息键保序策略

Kafka 允许为每条消息定义消息键，简称为 Key。这个 Key 的作用非常大，它可以是一个有着明确业务含义的字符串，比如客户代码、部门编号或是业务 ID 等；也可以用来表征消息元数据。特别是在 Kafka 不支持时间戳的年代，在一些场景中，工程师们都是直接将消息创建时间封装进 Key 里面的。

一旦消息被定义了 Key，那么你就可以保证同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略，如下图所示。

![](/assets/images/posts/kafka/kafka-6.png)

实现这个策略的 partition 方法同样简单，只需要下面两行代码即可：

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return Math.abs(key.hashCode()) % partitions.size();
```

# 5. 其他分区策略

我们就可以根据 Broker 所在的 IP 地址实现定制化的分区策略。

```

List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return partitions.stream().filter(p -> isSouth(p.leader().host())).map(PartitionInfo::partition).findAny().get();
```

# 6. 参考资料

《Kafka核心技术与实战》

