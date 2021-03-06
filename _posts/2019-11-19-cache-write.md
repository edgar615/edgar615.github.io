---
layout: post
title: 缓存（7）-写缓存提高数据库写操作资源
date: 2019-11-19
categories:
    - 架构设计
comments: true
permalink: cache-write.html
---

**写缓存**的思路是后台服务接收到用户请求时，如果请求校验没问题，数据并不会直接落库，而是先存储在缓冲层中，待缓冲层中写请求达到一定数量再进行批量落库。这里所说的缓冲地带，实际上指的就是写缓存。它的意义在于利用写缓存比数据库高几个量级的吞吐能力来承受洪峰流量，再平速搬运数据到数据库。

> 就是我们说的Write-Behind-Caching模式

从以上设计方案中，不难看出写缓存可以大幅减少数据库写操作的频率，从而减少数据库压力。

但该方案在具体实施过程中，往往需要考虑下列问题。

# 1.  如何触发批量落库

关于批量落库触发逻辑，目前市面上共分为 2 种触发方式。

- 写请求满足特定次数后就落库1次，比如 10个请求落1次。

按照次数批量落库的优点是访问数据库的次数变为 1/N，从数据库压力上来说会小很多。不过也存在不足，如果访问数据库的次数未凑齐 N 次，用户的预约就一直无法落库。

- 每隔一个时间窗口落库1次，比如每隔 1s 落库1 次。

按照时间窗口落库的优点是能保证用户等待的时间不会太久，其缺点是如果某个瞬时流量太大，在那个时间窗口落库的数据就会很多，多到在 1 次数据库访问中没法完成所有数据的插入（比如 1s 内堆积了 5 千条数据），它们只好通过分批次实现插入，这就又变回第 1 种逻辑了。

我们可以参考Redis的持久化机制，将这两种方式同时使用，具体实现逻辑如下：

- 每收集 1 次写请求，插入预约的数据到缓存中，再判断缓存中预约的总数是否达到一定数量，达到后直接触发批量落库；
- 开 1 个定时器，每隔 1s 触发 1 次批量落库。

通过以上操作，我们既避免了触发方案 1 提到的数量不足无法落库的情况，也避免了方案2因为瞬时流量大，待插入数据堆积太多的情况。

# 2. 缓存高可用

我们是先把用户提交的数据保存到缓存中，因此必须保证缓存中的数据不丢失，这就要求我们实现缓存的高可用。

# 3. 参考资料

《软件架构场景实战 22 讲 》