---
layout: post
title: 限流（1）- 介绍
date: 2020-04-15
categories:
    - 分布式
comments: true
permalink: ratelimit.html
---

# 1. 限流

很多做服务接口的人或多或少的遇到这样的场景，由于业务应用系统的负载能力有限，为了防止非预期的请求对系统压力过大而拖垮业务应用系统。

也就是面对大流量时，如何进行流量控制？

服务接口的流量控制策略：分流、降级、限流等。本文讨论下限流策略，虽然降低了服务接口的访问频率和并发量，却换取服务接口和业务应用系统的高可用。

每个API接口都是有访问上限的,当访问频率或者并发量超过其承受范围时候,我们就必须考虑限流来保证接口的可用性或者降级可用性.即接口也需要安装上保险丝,以防止非预期的请求对系统压力过大而引起的系统瘫痪.

通常的策略就是拒绝多余的访问,或者让多余的访问排队等待服务,或者引流.

实际场景中常用的限流策略：

Nginx前端限流

- 按照一定的规则如帐号、IP、系统调用逻辑等在Nginx层面做限流

业务应用系统限流

- 客户端限流
- 服务端限流

数据库限流

- 红线区，力保数据库

**限流总并发/连接/请求数**

对于一个应用系统来说一定会有极限并发/请求数，即总有一个TPS/QPS阀值，如果超了阀值则系统就会不响应用户请求或响应的非常慢，因此我们最好进行过载保护，防止大量请求涌入击垮系统。
例如tomcat的acceptCount、maxConnections、maxThreads，MySQL的max_connections，Redis的tcp-backlog

**限流总资源数**

如果有的资源是稀缺资源（如数据库连接、线程），而且可能有多个系统都会去使用它，那么需要限制应用；可以使用池化技术来限制总资源数：连接池、线程池。比如分配给每个应用的数据库连接是100，那么本应用最多可以使用100个资源，超出了可以等待或者抛异常。

**限流某个接口的总并发/请求数**

如果接口可能会有突发访问情况，但又担心访问量太大造成崩溃，如抢购业务；这个时候就需要限制这个接口的总并发/请求数总请求数了；因为粒度比较细，可以为每个接口都设置相应的阀值。可以使用Java中的AtomicLong进行限流：

```java
try {
    if(atomic.incrementAndGet() > 限流数) {
        //拒绝请求
    }
    //处理请求
} finally {
    atomic.decrementAndGet();
}
```

这种方式适合对业务无损的服务或者需要过载保护的服务进行限流，如抢购业务，超出了大小要么让用户排队，要么告诉用户没货了，对用户来说是可以接受的。而一些开放平台也会限制用户调用某个接口的试用请求量，也可以用这种计数器方式实现。这种方式也是简单粗暴的限流，没有平滑处理，需要根据实际情况选择使用

**限流某个接口的时间窗请求数**

即一个时间窗口内的请求数，如想限制某个接口/服务每秒/每分钟/每天的请求数/调用量。如一些基础服务会被很多其他系统调用，比如商品详情页服务会调用基础商品服务调用，但是怕因为更新量比较大将基础服务打挂，这时我们要对每秒/每分钟的调用量进行限速；简单的做法是维护一个单位时间内的 计数器，每次请求计数器加1，当单位时间内计数器累加到大于设定的阈值，则之后的请求都被拒绝，直到单位时间已经过去，再将  计数器  重置为零。此方式有个弊端：如果在单位时间1s内允许100个请求，在10ms已经通过了100个请求，那后面的990ms，只能眼巴巴的把请求拒绝，我们把这种现象称为“**突刺现象**”。

```java
LoadingCache<Long, AtomicLong> counter =
    CacheBuilder.newBuilder()
    .expireAfterWrite(2, TimeUnit.SECONDS)
    .build(new CacheLoader<Long, AtomicLong>() {
        @Override
        public AtomicLong load(Long seconds) throws Exception {
            return new AtomicLong(0);
        }
    });
long limit = 1000;
while(true) {
    //得到当前秒
    long currentSeconds = System.currentTimeMillis() / 1000;
    if(counter.get(currentSeconds).incrementAndGet() > limit) {
        System.out.println("限流了:" + currentSeconds);
        continue;
    }
    //业务处理
}
```



**平滑限流某个接口的请求数**

之前的限流方式都不能很好地应对突发请求，即瞬间请求可能都被允许从而导致一些问题；因此在一些场景中需要对突发请求进行整形，整形为平均速率请求处理（比如5r/s，则每隔200毫秒处理一个请求，平滑了速率）。这个时候有两种算法满足我们的场景：令牌桶和漏桶算法

# 2. 漏桶算法

![](/assets/images/posts/ratelimit/ratelimit-1.gif)

漏桶的主要目的是控制数据注入到网络的速率，平滑网络上的突发流量。漏桶算法提供了一种机制，通过它，突发流量可以被整形以便为网络提供一个稳定的流量。 漏桶可以看作是一个带有常量服务时间的单服务器队列，如果漏桶（包缓存）溢出，那么数据包会被丢弃。 

漏桶(Leaky Bucket)算法思路很简单,水(请求)先进入到漏桶里,漏桶以一定的速度出水(接口有响应速率),当水流入速度过大会直接溢出(访问频率超过接口响应速率),然后就拒绝请求,可以看出漏桶算法能强行限制数据的传输速率。

```java
double rate;               // leak rate in calls/s
double burst;              // bucket size in calls

long refreshTime;          // time for last water refresh
double water;              // water count at refreshTime

refreshWater() {
  long  now = getTimeOfDay();
  water = max(0, water- (now - refreshTime)*rate); // 水随着时间流逝，不断流走，最多就流干到0.
  refreshTime = now;
}

bool permissionGranted() {
  refreshWater();
  if (water < burst) { // 水桶还没满，继续加1
    water ++;
    return true;
  } else {
    return false;
  }
}
```

可见这里有两个变量,一个是桶的大小,支持流量突发增多时可以存多少的水(burst),另一个是水桶漏洞的大小(rate)

在某些情况下，漏桶算法不能够有效地使用网络资源。因为漏桶的漏出速率是固定的参数，所以，即使网络中不存在资源冲突（没有发生拥塞），漏桶算法也不能使某一个单独的流突发到端口速率。因此，漏桶算法对于存在突发特性的流量来说缺乏效率。而令牌桶算法则能够满足这些具有突发特性的流量。通常，漏桶算法与令牌桶算法可以结合起来为网络流量提供更大的控制。

# 3. 令牌桶

令牌桶算法 和漏桶算法 效果一样但方向相反的算法，更加容易理解。随着时间流逝，系统会按恒定 1/QPS 时间间隔（如果 QPS=100，则间隔是 10ms）往桶里加入 Token（想象和漏洞漏水相反，有个水龙头在不断的加水），如果桶已经满了就不再加了。新请求来临时，会各自拿走一个  Token，如果没有 Token 可拿了就阻塞或者拒绝服务。 

![](/assets/images/posts/ratelimit/ratelimit-2.jpg)

在 Wikipedia 上，令牌桶算法是这么描述的：

> 每秒会有 r 个令牌放入桶中，或者说，每过 1/r 秒桶中增加一个令牌
> 桶中最多存放 b 个令牌，如果桶满了，新放入的令牌会被丢弃
> 当一个 n 字节的数据包到达时，消耗 n 个令牌，然后发送该数据包
> 如果桶中可用令牌小于 n，则该数据包将被缓存或丢弃

令牌桶控制的是一个时间窗口内的通过的数据量，在 API 层面我们常说的 QPS、TPS，正好是一个时间窗口内的请求量或者事务量，只不过时间窗口限定在 1s 罢了。

令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。

令牌桶的另外一个好处是可以方便的改变速度. 一旦需要提高速率,则按需提高放入桶中的令牌的速率. 一般会定时(比如100毫秒)往桶中增加一定数量的令牌, 有些变种算法则实时的计算应该增加的令牌的数量.

**“漏桶算法”能够强行限制数据的传输速率，而“令牌桶算法”在能够限制数据的平均传输数据外，还允许某种程度的突发传输。在“令牌桶算法”中，只要令牌桶中存在令牌，那么就允许突发地传输数据直到达到用户配置的上限，因此它适合于具有突发特性的流量。**


# 4. 参考资料

http://xiaobaoqiu.github.io/blog/2015/07/02/ratelimiter/

http://www.cnblogs.com/exceptioneye/p/4783904.html

https://blog.jamespan.me/2015/10/19/traffic-shaping-with-token-bucket/

http://jinnianshilongnian.iteye.com/blog/2305117

https://redis.io/commands/INCR#pattern-rate-limiter

https://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis/155

http://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis-part-2/166

http://vinoyang.com/2015/08/23/redis-incr-implement-rate-limit/

https://blog.figma.com/an-alternative-approach-to-rate-limiting-f8a06cf7c94c