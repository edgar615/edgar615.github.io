---
layout: post
title: redis hyperloglog
date: 2019-11-20
categories:
    - redis
comments: true
permalink: redis-hyperloglog.html
---

HyperLogLog是一种基数算法（实际类型为字符串类型）。通过HyperLogLog可以利用极小的内存空间完成独立总数的统计，数据集可以是IP，email，ID等等。

HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

> 什么是基数?
> 比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。 基数估计就是在误差可接受的范围内，快速计算基数。 

HyperLogLog提供了三个命令：pfadd、pfcount、pfmerge


**添加**

```
pfadd key element [element ...]
```

示例

```
127.0.0.1:6379> pfadd 2017_11_09:ids 1 2 3 4 5
(integer) 1
```
**计算独立用户数**

```
pfcount key [key ...]
```

示例：pfcount用于计算一个或多个HyperLogLog的独立总数

```
127.0.0.1:6379> PFCOUNT 2017_11_09:ids
(integer) 5
```
HyperLogLog在数据量大的情况下会对内存的占用量非常小。但是用如此小空间来估算巨大的数据，必然不是100%的正确，其中一定一定存在误差率。Redis官方给出的数字是0.81%的失误率

**合并**

```
pfmerge destkey sourcekey [sourcekey ...]
```

pfmerge可以求出多个HyperLogLog的并集并赋值给destKey

```
127.0.0.1:6379> pfmerge 2017_11_08-09:ids 2017_11_08:ids 2017_11_09:ids
OK
127.0.0.1:6379> pfcount 2017_11_08-09:ids
(integer) 7
```

# 应用场景 
统计网页的UV

UV与PV不同，同一个用户一天之内的多次访问请求只能计数一次，这就要求每一个网页请求都需要带上用户的 ID，无论是登陆用户还是未登陆用户都需要一个唯一 ID 来标识。如果为每一个页面一个独立的 set 集合来存储所有当天访问过此页面的用户 ID。当一个请求过来时，我们使用 sadd 将用户 ID 塞进去就可以了。通过 scard 可以取出这个集合的大小，这个数字就是这个页面的 UV 数据。没错，这是一个非常简单的方案。

但是，如果你的页面访问量非常大，比如一个爆款页面几千万的 UV，你需要一个很大的 set 集合来统计，这就非常浪费空间。如果这样的页面很多，那所需要的存储空间是惊人的。为这样一个去重功能就耗费这样多的存储空间，而实际上UV的数据不需要太精确，105w 和 106w 这两个数字对于并没有多大区别，这时就可以通过HyperLogLog来实现。使用Redis的HyperLogLog最多需要12k就可以统计大量的用户数，尽管它大概有0.81%的错误率，但对于统计UV这种不需要很精确的数据是可以忽略不计的。

> 也可以通过bitmap，但是并没有HyperLogLog方便
