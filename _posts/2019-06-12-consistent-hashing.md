---
layout: post
title: 一致性hash算法
date: 2019-06-12
categories:
    - redis
comments: true
permalink: consistent-hashing.html
---

学习redis cluster需要先对一致性hash算法有一定的了解。参考资料的几篇文章写的挺清楚的，这篇文章只是简单的整理了下

# 分布式缓存的问题

随着业务的扩展，流量的剧增，单体项目逐渐划分为分布式系统。对于经常使用的数据，我们可以使用Redis作为缓存机制，减少数据层的压力。下图是一个单体应用的架构（画的很LOW，^_^）

![](/assets/images/posts/consistent-hashing/consistent-hashing-1.png)

假设我们有3个redis实例，那么对于redis的访问有两种策略

## 随机访问
将每一次请求随机发送到一台redis服务器，但是这种策略可能会带来两个问题：

- 同一份数据可能被存在不同的机器上而造成数据冗余
- 无法保证对相同key的所有访问都被发送到相同的服务器，有可能某数据已经被缓存但是访问却没有命中

## HASH访问
![](/assets/images/posts/consistent-hashing/consistent-hashing-2.png)

如上图所示，我们把Redis编号设置成0,1,2，然后对请求的key求hash值
```
h = Hash(key) % 3
```
这个算式计算每个key的请求应该被发送到哪台服务器，其中N为服务器的台数，并且服务器按照0 – (N-1)编号。

但是hash算法也会面临容错性和扩展性的问题。容错性是指当系统中的某个服务出现问题时，不能影响其他系统。扩展性是指当加入新的服务器后，整个系统能正确高效运行。

如果有一台Redis服务器宕机了，那么为了填补空缺，要将宕机的服务器从编号列表中移除，后面的服务器按顺序前移一位并将其编号值减一，此时每个key就要按h = Hash(key) % 2重新计算。

同样，如果新增一台服务器，规则也同样需要重新计算，h = Hash(key) % 4。

因此，系统中如果有服务器更变，会直接影响到Hash值，大量的key会重定向到其他服务器中，造成缓存命中率降低，而这种情况在分布式系统中是十分糟糕的。

一个设计良好的分布式哈希方案应该具有良好的单调性，即服务节点的变更不会造成大量的哈希重定位。

# 一致性hash算法
> 一致哈希 是一种特殊的哈希算法。在使用一致哈希算法后，哈希表槽位数（大小）的改变平均只需要对 K/n 个关键字重新映射，其中K是关键字的数量， n是槽位数量。然而在传统的哈希表中，添加或删除一个槽位的几乎需要对所有关键字进行重新映射。

1. 一致性哈希是将整个哈希值空间组织成一个虚拟的圆环，假设哈希函数H的值空间为0-2^32-1（哈希值是32位无符号整形），整个空间按顺时针方向组织，0和2^32-1在零点中方向重合。整个哈希空间环如下：

![](/assets/images/posts/consistent-hashing/consistent-hashing-3.png)

2. 下一步将各个服务器使用H进行一个哈希，具体可以选择服务器的ip或主机名作为关键字进行哈希，这样每台机器就能确定其在哈希环上的位置，

![](/assets/images/posts/consistent-hashing/consistent-hashing-4.png)

3. 例如我们有A、B、C、D四个数据对象，经过哈希计算出在环空间上的位置：数据A会被定为到Server 1上，数据B被定为到Server 2上，而C、D被定为到Server 3上（顺时针查找）

![](/assets/images/posts/consistent-hashing/consistent-hashing-5.png)

## 容错性
假设redis-2宕机，A、C、D没有没有影响 数据B迁移到Redis-3中（顺时针）

![](/assets/images/posts/consistent-hashing/consistent-hashing-6.png)

在一致性哈希算法中，如果一台服务器不可用，则受影响的数据仅仅是此服务器到其环空间中前一台服务器（即顺着逆时针方向行走遇到的第一台服务器）之间数据，其它不会受到影响

## 扩展性

假设增加redis-4，A、B、D没有没有影响 数据C迁移到Redis-4中

![](/assets/images/posts/consistent-hashing/consistent-hashing-7.png)

在一致性哈希算法中，如果增加一台服务器，则受影响的数据仅仅是新服务器到其环空间中前一台服务器（即顺着逆时针方向行走遇到的第一台服务器）之间数据，其它不会受到影响。

综上所述，一致性哈希算法对于节点的增减都只需重定位环空间中的一小部分数据，具有较好的容错性和可扩展性。

# 虚拟节点
一致性哈希算法在服务节点太少时，容易因为节点分部不均匀而造成数据倾斜问题。例如我们的系统中有两台服务器，其环分布如下：

![](/assets/images/posts/consistent-hashing/consistent-hashing-8.png)

此时必然造成大量数据集中到redis-2上，而只有极少量会定位到redis-1上。为了解决这种数据倾斜问题，一致性哈希算法引入了虚拟节点机制，即对每一个服务节点计算多个哈希，每个计算结果位置都放置一个此服务节点，称为虚拟节点。

具体做法可以在服务器IP或主机名的后面增加编号来实现，例如上面的情况，可以为每个服务节点增加三个虚拟节点，于是可以分为 redis-1#1、 redis-1#2、 redis-1#3、 redis-2#1、 redis-2#2、 redis-2#3

![](/assets/images/posts/consistent-hashing/consistent-hashing-9.png)

对于数据定位的hash算法仍然不变，只是增加了虚拟节点到实际节点的映射。例如，数据C保存到虚拟节点Redis-1#2，实际上数据保存到Redis-1中。这样，就能解决服务节点少时数据不平均的问题。在实际应用中，通常将虚拟节点数设置为32甚至更大，因此即使很少的服务节点也能做到相对均匀的数据分布。

# 参考资料

http://blog.codinglabs.org/articles/consistent-hashing.html

https://segmentfault.com/a/1190000017847097
