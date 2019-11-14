---
layout: post
title: 缓存-缓存的模式(part2)
date: 2019-11-13
categories:
    - 缓存
comments: true
permalink: cache-pattern.html
---

常用的缓存模式有下面几种：

- Cache Aside
- Read Through
- Write Through
- Write Behind Caching

# Cache Aside

这种模式下应用程序负责对数据库的读写，而缓存不与数据库交互。应用程序在读取数据库中的任何数据之前先检查缓存。同时，应用程序在对数据库进行任何更新后需要更新缓存。通过上述的操作应用程序确保缓存与数据库保持同步。

![](/assets/images/posts/cache/cache-aside-1.png)

![](/assets/images/posts/cache/cache-aside-2.png)

当数据发生变化的时候，对缓存的失效有两种处理策略：

- 更新缓存：数据不但写入数据库，还会写入缓存，缓存不会增加一次miss，命中率高，但处理复杂
- 淘汰缓存：数据只会写入数据库，不会写入缓存，只会把数据淘汰掉，增加了一次cache miss，但处理简单

这两种策略同时又有两种处理方式：

- 先写数据库，再操作缓存
- 先操作缓存，再写数据库

Cache Aside模式建议**先写数据库，再淘汰缓存**。为什么？

我们先假设写数据库和操作缓存都会成功。在两个线程并发写入的时候分别看更新缓存的两种场景

**先更新缓存，再写数据库**

![](/assets/images/posts/cache/cache-aside-8.png)

**先更新缓存，再写数据库**

![](/assets/images/posts/cache/cache-aside-9.png)

对于更新缓存，在线程A和线程B两个并发写发生时，由于无法保证时序，此时不管先操作缓存还是先操作数据库，都会导致缓存和数据库的数据不一致。

在两个线程并发读写的时候分别看淘汰缓存的两种场景

**先淘汰缓存，再写数据库**

![](/assets/images/posts/cache/cache-aside-10.png)

**先写数据库，再淘汰缓存**

![](/assets/images/posts/cache/cache-aside-11.png)

从上面的分析可以得出，先写数据库，再淘汰缓存会导致一次cache miss，其他三种情况都容易出现数据不一致的情况，所以Cache Aside模式建议**先写数据库，再淘汰缓存**

然而这并非说这种模式的缓存处理就一定能做到完美。比如下面的场景

![](/assets/images/posts/cache/cache-aside-3.png)

一个是读操作，但是没有命中缓存，然后就到数据库中取数据，此时来了一个写操作，写完数据库后，让缓存失效，然后，之前的那个读操作再把老的数据放进去，这时缓存和数据库中的数据就不一致。

不过，上述问题实际上出现的概率可能非常低，因为这个条件需要发生在读缓存时缓存失效，而且同时并发着有一个写操作。而实际上数据库的写操作会比读操作慢得多，而且还要锁表，而读操作必需在写操作前进入数据库操作，而又要晚于写操作更新缓存。所以我们一般可以忽略这种数据不一致的场景。

> 为了保证数据一致性，只能通过2PC或者Paxos协议来保证一致性，这会极大增加复杂度。所以我们一般会采取尽量降低并发时脏数据的概率
> 我们也可以将同一个数据的更新、读取操作放到一个队列中排队处理，但这样会带来更大的复杂度。

另外更新缓存的代价有时候是很高的。如果一个缓存涉及的表的字段，在 1 分钟内就修改了 20 次，或者是 100 次，那么缓存更新 20 次、100 次；但是这个缓存在 1 分钟内只被读取了 1 次，有大量的冷数据。实际上，如果你只是删除缓存的话，那么在 1 分钟内，这个缓存不过就重新计算一次而已，开销大幅度降低，用到缓存才去算缓存。

# Read-Through

Read-Through模式是指应用程序始终从缓存中请求数据。如果缓存没有数据，则它负责使用底层提供程序插件从数据库中检索数据。检索数据后，缓存会自行更新并将数据返回给调用应用程序。

![](/assets/images/posts/cache/read-through-1.png)

Read-though模式下应用总是使用key从缓存中请求数据, 调用的应用程序不知道数据库， 由存储方来负责自己的缓存处理，这使代码更具可读性， 代码更清晰。

# Write-Through

Write Through模式和Read Through模式类似，当数据发生更新的时候，首先将数据写入缓存，然后写入数据库。缓存与数据库保持一致，写操作总是通过缓存到达主数据库。如果缓存和数据库都被更新成功，则认为写入操作成功（同步操作）。如果缓存写入成功，数据库写入失败，需要考虑回退的问题。

![](/assets/images/posts/cache/write-through-1.png)

# Write-Behind-Caching 

在Write Through模式下面我们也可以将数据库更新改为延迟更新：只要数据被写入缓存，就认为是成功的，然后再通过异步方式更新数据库。

![](/assets/images/posts/cache/write-behind-caching-1.png)

这个模式的好处就是让数据的I/O操作飞快无比，但是**数据不是强一致性的，可能会丢失**

# 更深入一步

我们在上面对Cache Aside的一致性分析都是基于写数据库和操作缓存都成功的情况下，然而写数据库与操作缓存不能保证原子性，两个操作的操作时序不同页会导致数据不一致的情况发生。

**先更新缓存，再写数据库**：第一步更新缓存成功，第二步写数据库失败，会出现数据库中是旧数据，缓存中是新数据，数据不一致。

![](/assets/images/posts/cache/cache-aside-4.png)

**先写数据库，再更新缓存**：第一步写数据库操作成功，第二步更新缓存失败，则会出现数据库中是新数据，缓存中是旧数据，数据不一致。

![](/assets/images/posts/cache/cache-aside-5.png)

**先淘汰缓存，再写数据库**：第一步淘汰缓存成功，第二步写数据库失败，则只会引发一次Cache miss。

![](/assets/images/posts/cache/cache-aside-6.png)

**先写数据库，再淘汰缓存**：第一步写数据库操作成功，第二步淘汰缓存失败，则会出现数据库中是新数据，缓存中是旧数据，数据不一致。

![](/assets/images/posts/cache/cache-aside-7.png)

根据上面的分析，因为数据库与操作缓存不能保证原子性，**先写数据库，再淘汰缓存**依然无法保证数据的一致性，为了弥补这个缺陷我们可以采用重试机制

![](/assets/images/posts/cache/cache-aside-12.png)

在淘汰缓存失败后，将失败的key发送到消息队列，然后由一个哨兵（也可以是自己）订阅这个消息，然后再次去淘汰缓存。

该方案有一个缺点，对业务线代码造成大量的侵入。我们可以通过两种方式来解耦

1. 通过binlog订阅来发送消息
2. 记录日志，通过日志分析抓取异常发送消息


# 参考资料

http://coolshell.cn/articles/17416.html

https://mp.weixin.qq.com/s/7IgtwzGC0i7Qh9iTk99Bww

https://mp.weixin.qq.com/s/pYVdCqoKauw4K2LgBnXFpw

https://mp.weixin.qq.com/s/CuwTRC8HrMHxWZe3_OX98g