---
layout: post
title: 缓存（8）- 一些实践经验
date: 2019-11-20
categories:
    - 架构设计
comments: true
permalink: cache-principal.html
---

# 1. 分页缓存

以查询博客列表的场景为例。如果我们对对分页内容进行整体缓存。这种方案会按照页码和每页大小组合成一个缓存key，缓存值就是博客信息列表。假如某一个博客内容发生修改, 我们要重新加载缓存，或者删除整页的缓存。这种方案，缓存的颗粒度比较大，如果博客更新较为频繁，则缓存很容易失效。

针对这种场景，我们可以改完N+1查询的方式。

**1. 先从数据库查询当前页的博客id列表**

```
select id from blogs limit 0,10 
```

**2.批量从缓存中获取博客id列表对应的缓存数据 ，并记录没有命中的博客id，若没有命中的id列表大于0，再次从数据库中查询一次，并放入缓**

```
select id from blogs where id in (noHitId1, noHitId2)
```

**3.将没有缓存的博客对象存入缓存中**

**4.返回博客对象列表**

理论上，要是缓存都预热的情况下，一次简单的数据库查询，一次缓存批量获取，即可返回所有的数据。另外，关于缓存批量获取，可以按下面的方式实现

- 本地缓存：性能极高，for 循环即可；
- memcached：使用 mget 命令；
- Redis：若缓存对象结构简单，使用 mget 、hmget命令；若结构复杂，可以考虑使用 pipleline，lua脚本模式。

第 1 种方案适用于数据极少发生变化的场景，比如排行榜，首页新闻资讯等。

第 2 种方案适用于大部分的分页场景，而且能和其他资源整合在一起。举例：在搜索系统里，我们可以通过筛选条件查询出博客 id 列表，然后通过如上的方式，快速获取博客列表。

# 2. **多级缓存**

**缓存离用户越近越高效**

本地缓存速度极快，但是容量有限，而且无法共享内存。分布式缓存容量可扩展，但在高并发场景下，如果所有数据都必须从远程缓存种获取，很容易导致带宽跑满，吞吐量下降。

使用多级缓存的好处在于：高并发场景下, 能提升整个系统的吞吐量，减少分布式缓存的压力。

# 3. 缓存热点

缓存热点，是一个很常见的问题，比如“某某明星宣布结婚”等等，都可能产生大量请求访问的问题，一个最麻烦也是最容易让人忽视的事情就是如何探测到热点 Key。

在缓存系统中，除了一些常用的热点 Key 外，在某些特殊场合下也会出现大量的热点 Key，我们该如何发现呢？

- **数据调研。**可以分析历史数据以及针对不同的场合去预测出热点 Key，这种方式虽然不能百分百使得缓存命中，但是却是一种最简单和节省成本的方案。
- **实时计算。**可以使用现有的实时计算框架，比如 Storm、Spark Streaming、Flink 等框架统计一个时间段内的请求量，从而判断热点 Key。或者也可以自己实现定时任务去统计请求量。

对于热点 Key 问题，当缓存系统中没有发现缓存时，需要去数据库中读取数据。

当大量请求来的时候，一个请求获取锁去请求数据库，其他阻塞，接着全部去访问缓存，这样可能因为一台服务器撑不住从而宕机。

比如正常一台服务器并发量为 5W 左右，产生热点 Key 的时候达到了 10W 甚至 20W，这样服务器肯定会崩。所以我们在发现热点 Key 之后还需要做到如何自动负载均衡。