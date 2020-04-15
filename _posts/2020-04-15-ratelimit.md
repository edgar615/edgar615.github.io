---
layout: post
title: 限流
date: 2020-04-15
categories:
    - 分布式
comments: true
permalink: ratelimit.html
---

# 限流

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

即一个时间窗口内的请求数，如想限制某个接口/服务每秒/每分钟/每天的请求数/调用量。如一些基础服务会被很多其他系统调用，比如商品详情页服务会调用基础商品服务调用，但是怕因为更新量比较大将基础服务打挂，这时我们要对每秒/每分钟的调用量进行限速；一种实现方式如下所示：

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

# 漏桶算法

![](/assets/images/posts/ratelimit/raftlimit-1.gif)

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

# 令牌桶

![](/assets/images/posts/ratelimit/raftlimit-2.jpg)

在 Wikipedia 上，令牌桶算法是这么描述的：

> 每秒会有 r 个令牌放入桶中，或者说，每过 1/r 秒桶中增加一个令牌
> 桶中最多存放 b 个令牌，如果桶满了，新放入的令牌会被丢弃
> 当一个 n 字节的数据包到达时，消耗 n 个令牌，然后发送该数据包
> 如果桶中可用令牌小于 n，则该数据包将被缓存或丢弃

令牌桶控制的是一个时间窗口内的通过的数据量，在 API 层面我们常说的 QPS、TPS，正好是一个时间窗口内的请求量或者事务量，只不过时间窗口限定在 1s 罢了。

令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。

令牌桶的另外一个好处是可以方便的改变速度. 一旦需要提高速率,则按需提高放入桶中的令牌的速率. 一般会定时(比如100毫秒)往桶中增加一定数量的令牌, 有些变种算法则实时的计算应该增加的令牌的数量.

**“漏桶算法”能够强行限制数据的传输速率，而“令牌桶算法”在能够限制数据的平均传输数据外，还允许某种程度的突发传输。在“令牌桶算法”中，只要令牌桶中存在令牌，那么就允许突发地传输数据直到达到用户配置的上限，因此它适合于具有突发特性的流量。**

## guava的RateLimiter

Google开源工具包Guava提供了限流工具类RateLimiter,该类基于令牌桶算法(Token Bucket)来完成限流,非常易于使用.RateLimiter经常用于限制对一些物理资源或者逻辑资源的访问速率.它支持两种获取permits接口,一种是如果拿不到立刻返回false,一种会阻塞等待一段时间看能不能拿到.

RateLimiter和Java中的信号量(java.util.concurrent.Semaphore)类似,Semaphore通常用于限制并发量.

源码注释中的一个例子,比如我们有很多任务需要执行,但是我们不希望每秒超过两个任务执行,那么我们就可以使用RateLimiter:

```java
final RateLimiter rateLimiter = RateLimiter.create(2.0);
void submitTasks(List<Runnable> tasks, Executor executor) {
    for (Runnable task : tasks) {
        rateLimiter.acquire(); // may wait
        executor.execute(task);
    }
}
```

另外一个例子,假如我们会产生一个数据流,然后我们想以每秒5kb的速度发送出去.我们可以每获取一个令牌(permit)就发送一个byte的数据,这样我们就可以通过一个每秒5000个令牌的RateLimiter来实现:

```java
final RateLimiter rateLimiter = RateLimiter.create(5000.0);
void submitPacket(byte[] packet) {
    rateLimiter.acquire(packet.length);
    networkService.send(packet);
}
```

我们也可以使用非阻塞的形式达到降级运行的目的,即使用非阻塞的tryAcquire()方法:

```java
if(limiter.tryAcquire()) { //未请求到limiter则立即返回false
    doSomething();
}else{
    doSomethingElse();
}
```

## 创建RateLimiter

```java
public static RateLimiter create(double permitsPerSecond);	
```

创建一个稳定输出令牌的RateLimiter，保证了平均每秒不超过permitsPerSecond个请求，当请求到来的速度超过了permitsPerSecond，保证每秒只处理permitsPerSecond个请求，当这个RateLimiter使用不足(即请求到来速度小于permitsPerSecond)，会囤积最多permitsPerSecond个请求.

```java
public static RateLimiter create(double permitsPerSecond, long warmupPeriod, TimeUnit unit)
```

创建一个稳定输出令牌的RateLimiter，保证了平均每秒不超过permitsPerSecond个请求， 还包含一个热身期(warmup period),热身期内，RateLimiter会平滑的将其释放令牌的速率加大，直到起达到最大速率，同样，如果RateLimiter在热身期没有足够的请求(unused),则起速率会逐渐降低到冷却状态。设计这个的意图是为了满足那种资源提供方需要热身时间，而不是每次访问都能提供稳定速率的服务的情况(比如带缓存服务，需要定期刷新缓存的)，参数warmupPeriod和unit决定了其从冷却状态到达最大速率的时间

## 获取令牌

```java
public double acquire();
```

获取一个令牌.如果没有令牌则一直等待,返回等待的时间(单位为秒),没有被限流则直接返回0.0

```java
public double acquire(int permits);
```

一次获取多个令牌.

```java
public boolean tryAcquire();
public boolean tryAcquire(int permits);
```

尝试获取令牌,如果获取到令牌立即返回true，反之返回false

```java
public boolean tryAcquire(long timeout, TimeUnit unit);
public boolean tryAcquire(int permits, long timeout, TimeUnit unit);
```

尝试获取令牌,带超时时间

## 设计

RateLimiter的主要功能就是提供一个稳定的速率,实现方式就是通过限制请求流入的速度,比如计算请求等待合适的时间阈值.

**实现QPS速率的最简单的方式就是记住上一次请求的最后授权时间,然后保证1/QPS秒内不允许请求进入**.比如QPS=5,如果我们保证最后一个被授权请求之后的200ms的时间内没有请求被授权,那么我们就达到了预期的速率.如果一个请求现在过来但是最后一个被授权请求是在100ms之前,那么我们就要求当前这个请求等待100ms.按照这个思路,请求15个新令牌(许可证)就需要3秒.

有一点很重要:上面这个设计思路的RateLimiter记忆非常的浅,它的脑容量非常的小,只记得上一次被授权的请求的时间.如果RateLimiter的一个被授权请求q之前很长一段时间没有被使用会怎么样?这个RateLimiter会立马忘记过去这一段时间的利用不足,而只记得刚刚的请求q.

过去一段时间的利用不足意味着有过剩的资源是可以利用的.这种情况下,RateLimiter应该加把劲(speed up for a while)将这些过剩的资源利用起来.比如在向网络中发生数据的场景(限流),过去一段时间的利用不足可能意味着网卡缓冲区是空的,这种场景下,我们是可以加速发送来将这些过程的资源利用起来.

另一方面,过去一段时间的利用不足可能意味着处理请求的服务器对即将到来的请求是准备不足的(less ready for future requests),比如因为很长一段时间没有请求当前服务器的cache是陈旧的,进而导致即将到来的请求会触发一个昂贵的操作(比如重新刷新全量的缓存).

为了处理这种情况,RateLimiter中增加了一个维度的信息,就是过去一段时间的利用不足(past underutilization),代码中使用storedPermits变量表示.当没有利用不足这个变量为0,最大能达到maxStoredPermits(maxStoredPermits表示完全没有利用).因此,请求的令牌可能从两个地方来:

```
1.过去剩余的令牌(stored permits, 可能没有)
2.现有的令牌(fresh permits,当前这段时间还没用完的令牌)
```

我们将通过一个例子来解释它是如何工作的:

对一个每秒产生一个令牌的RateLimiter,每有一个没有使用令牌的一秒,我们就将storedPermits加1,如果RateLimiter在10秒都没有使用,则storedPermits变成10.0.这个时候,一个请求到来并请求三个令牌(acquire(3)),我们将从storedPermits中的令牌为其服务,storedPermits变为7.0.这个请求之后立马又有一个请求到来并请求10个令牌,我们将从storedPermits剩余的7个令牌给这个请求,剩下还需要三个令牌,我们将从RateLimiter新产生的令牌中获取.我们已经知道,RateLimiter每秒新产生1个令牌,就是说上面这个请求还需要的3个请求就要求其等待3秒.

想象一个RateLimiter每秒产生一个令牌,现在完全没有使用(处于初始状态),现在一个昂贵的请求acquire(100)过来.如果我们选择让这个请求等待100秒再允许其执行,这显然很荒谬.我们为什么什么也不做而只是傻傻的等待100秒,一个更好的做法是允许这个请求立即执行(和acquire(1)没有区别),然后将随后到来的请求推迟到正确的时间点.这种策略,我们允许这个昂贵的任务立即执行,并将随后到来的请求推迟100秒.这种策略就是让任务的执行和等待同时进行.

一个重要的结论:RateLimiter不会记最后一个请求,而是即下一个请求允许执行的时间.这也可以很直白的告诉我们到达下一个调度时间点的时间间隔.然后定一个一段时间未使用的Ratelimiter也很简单:下一个调度时间点已经过去,这个时间点和现在时间的差就是Ratelimiter多久没有被使用,我们会将这一段时间翻译成storedPermits.所有,如果每秒钟产生一个令牌(rate==1),并且正好每秒来一个请求,那么storedPermits就不会增长.

一个重要的结论:**RateLimiter不会记最后一个请求,而是即下一个请求允许执行的时间**.这也可以很直白的告诉我们到达下一个调度时间点的时间间隔.然后定一个一段时间未使用的Ratelimiter也很简单:下一个调度时间点已经过去,这个时间点和现在时间的差就是Ratelimiter多久没有被使用,我们会将这一段时间翻译成storedPermits.所有,如果每秒钟产生一个令牌(rate==1),并且正好每秒来一个请求,那么storedPermits就不会增长.

也就是说 **RateLimiter 允许某次请求拿走超出剩余令牌数的令牌，但是下一次请求将为此付出代价，一直等到令牌亏空补上，并且桶中有足够本次请求使用的令牌为止**

## 平滑突发限流 SmoothBursty

### 示例1
```java
//桶容量为5，且每秒新增5个令牌，即每隔200毫秒新增一个令牌
RateLimiter limiter = RateLimiter.create(5);
//limiter.acquire表示消费一个令牌，如果当前桶中有足够令牌则成功(返回值为0)
//如果动作没有令牌则暂停一段时间，比如发令牌的间隔是200毫秒，则等待200毫秒后再去消费令牌
//这种实现将突发请求速率平均为了固定请求速率
System.out.println(System.currentTimeMillis());
double waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);
```

输出

```
1484193737102
1484193737102:acquire ticket, waitTime:0.0
1484193737102
1484193737302:acquire ticket, waitTime:0.19948
1484193737303
1484193737502:acquire ticket, waitTime:0.198852
1484193737502
1484193737702:acquire ticket, waitTime:0.199784
1484193737702
1484193737902:acquire ticket, waitTime:0.199809
1484193737902
1484193738102:acquire ticket, waitTime:0.199774
1484193738102
1484193738302:acquire ticket, waitTime:0.199799
```

通过输出可以看到，除第一个令牌，其他的令牌都等待了大约200毫秒的时间

### 示例2

```java
RateLimiter limiter = RateLimiter.create(5);
System.out.println(System.currentTimeMillis());
double waitTime = limiter.acquire(5);
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

//下面的方法将等待差不多1秒左右桶中才能有令牌
System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

//下面的方法将等待差不多200毫秒左右桶中才能有令牌
System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);
```

输出

```
1484193813941
1484193813941:acquire ticket, waitTime:0.0
1484193813941
1484193814941:acquire ticket, waitTime:0.999481
1484193814943
1484193815141:acquire ticket, waitTime:0.197737
```

通过输出可以看到，第二个令牌等待了大约1秒的时间，第三个令牌等待了200毫秒的时间

### 示例3

```java
RateLimiter limiter = RateLimiter.create(2);
System.out.println(System.currentTimeMillis());
double waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);
TimeUnit.SECONDS.sleep(5);

//下面的三个方法都能获取到令牌
System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);

//下面的方法将等待差不多200毫秒左右桶中才能有令牌
System.out.println(System.currentTimeMillis());
waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);
```

输出

```
1484194198975
1484194198975:acquire ticket, waitTime:0.0
1484194203976
1484194203976:acquire ticket, waitTime:0.0
1484194203976
1484194203976:acquire ticket, waitTime:0.0
1484194203976
1484194203976:acquire ticket, waitTime:0.0
1484194203976
1484194204478:acquire ticket, waitTime:0.49965
```

通过输出可以看到，中间的3个令牌都没有等待，第4个令牌等待了500毫秒的时间。

**令牌桶算法允许将一段时间内没有消费的令牌暂存到令牌桶中，留待未来使用，应对未来请求的突发**


平滑突发限流（SmoothBursty）的构造方法中有个参数：最大突发秒数maxBurstSeconds，默认值1秒

```java
public static RateLimiter create(double permitsPerSecond) {
    /*
	 * The default RateLimiter configuration can save the unused permits of up to one second.
	 * This is to avoid unnecessary stalls in situations like this: A RateLimiter of 1qps,
	 * and 4 threads, all calling acquire() at these moments:
	 *
	 * T0 at 0 seconds
	 * T1 at 1.05 seconds
	 * T2 at 2 seconds
	 * T3 at 3 seconds
	 *
	 * Due to the slight delay of T1, T2 would have to sleep till 2.05 seconds,
	 * and T3 would also have to sleep till 3.05 seconds.
	 */
    return create(SleepingStopwatch.createFromSystemTimer(), permitsPerSecond);
}

@VisibleForTesting
static RateLimiter create(SleepingStopwatch stopwatch, double permitsPerSecond) {
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
    rateLimiter.setRate(permitsPerSecond);
    return rateLimiter;
}

SmoothBursty(SleepingStopwatch stopwatch, double maxBurstSeconds) {
    super(stopwatch);
    this.maxBurstSeconds = maxBurstSeconds;
}
```

**最大突发秒数maxBurstSeconds的计算公式**

	突发量/桶容量=速率*maxBurstSeconds

## 类似漏桶的算法 SmoothWarmingUp

SmoothBursty允许一定程度的突发，但假设突然连了很大的流量，那么系统很可能抗不住这种突发。
因此需要一个**平滑速率的限流工具，从而使系统冷启动后慢慢的趋于平均固定速率**（即刚开始速率小一些，然后慢慢趋于我们设置的固定速率）

漏桶算法强制一个常量的输出速率而不管输入数据流的突发性,当输入空闲时，该算法不执行任何动作.就像用一个底部开了个洞的漏桶接水一样,水进入到漏桶里,
桶里的水通过下面的孔以固定的速率流出,当水流入速度过大会直接溢出,可以看出漏桶算法能强行限制数据的传输速率.

类似漏桶的实现SmoothWarmingUp

```java
RateLimiter limiter = RateLimiter.create(5, 1000, TimeUnit.MILLISECONDS);
```

后面两个参数表示从冷启动速率过渡到平均速率的时间间隔，可以通过调节warmupPeriod参数实现一开始就是平滑固定速率
### 示例1

```
RateLimiter limiter = RateLimiter.create(5, 1000, TimeUnit.MILLISECONDS);
for (int i = 0; i < 6; i ++) {
System.out.println(System.currentTimeMillis());
double waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);
}
TimeUnit.SECONDS.sleep(1);
System.out.println("--------------------------------------------");
for (int i = 0; i < 10; i ++) {
System.out.println(System.currentTimeMillis());
double waitTime = limiter.acquire();
System.out.println(System.currentTimeMillis() + ":acquire ticket, waitTime:" + waitTime);
}
```

输出

```
1484195488592
1484195488593:acquire ticket, waitTime:0.0
1484195488593
1484195489113:acquire ticket, waitTime:0.519855
1484195489113
1484195489473:acquire ticket, waitTime:0.359091
1484195489473
1484195489693:acquire ticket, waitTime:0.219833
1484195489693
1484195489893:acquire ticket, waitTime:0.199852
1484195489893
1484195490093:acquire ticket, waitTime:0.199837
--------------------------------------------
1484195491093
1484195491093:acquire ticket, waitTime:0.0
1484195491093
1484195491453:acquire ticket, waitTime:0.360015
1484195491453
1484195491673:acquire ticket, waitTime:0.220133
1484195491673
1484195491873:acquire ticket, waitTime:0.200117
1484195491873
1484195492073:acquire ticket, waitTime:0.200096
1484195492073
1484195492273:acquire ticket, waitTime:0.200064
1484195492273
1484195492473:acquire ticket, waitTime:0.200112
1484195492473
1484195492673:acquire ticket, waitTime:0.20012
1484195492673
1484195492873:acquire ticket, waitTime:0.200085
1484195492873
1484195493073:acquire ticket, waitTime:0.199973
```

通过输出可以看到,速率是梯形上升速率，也就是说冷启动会以一个比较大的速率慢慢到平均速率；然后趋于平均速率

## 内部实现
SmoothWarmingUp的acquire方法最终会调用reserveEarliestAvailable方法

```java
public double acquire(int permits) {
    long microsToWait = reserve(permits);
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return 1.0 * microsToWait / SECONDS.toMicros(1L);
}

final long reserve(int permits) {
    checkPermits(permits);
    synchronized (mutex()) {
        return reserveAndGetWaitLength(permits, stopwatch.readMicros());
    }
}  

final long reserveAndGetWaitLength(int permits, long nowMicros) {
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    return max(momentAvailable - nowMicros, 0);
}

final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    double freshPermits = requiredPermits - storedPermitsToSpend;

    long waitMicros = storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
        + (long) (freshPermits * stableIntervalMicros);

    this.nextFreeTicketMicros = nextFreeTicketMicros + waitMicros;
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
}

private void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
        storedPermits = min(maxPermits,
                            storedPermits + (nowMicros - nextFreeTicketMicros) / stableIntervalMicros);
        nextFreeTicketMicros = nowMicros;
    }
}
```

仔细阅读代码可以发现，token数量的计算是通过下面的方法来实现的：

```java
storedPermits = min(maxPermits,
                    storedPermits + (nowMicros - nextFreeTicketMicros) / stableIntervalMicros);
```
我们可以在桶中存放剩余的令牌数量和上一次访问的时间戳，当用户请求获取一个令牌的时候，根据当前时间戳，计算从上一次访问的时间戳开始，到现在的这个时间点应该向桶中添加的令牌数量，从而获得当前令牌桶中剩余的令牌数**(如果桶满了，直接取桶的最大容量)**。

# Redis实现

redis官网介绍了用incr命令实现RateLimit的两种方法

## 模式1

```
FUNCTION LIMIT_API_CALL(ip)
ts = CURRENT_UNIX_TIME()
keyname = ip+":"+ts
current = GET(keyname)
IF current != NULL AND current > 10 THEN
	ERROR "too many requests per second"
ELSE
	MULTI
		INCR(keyname,1)
		EXPIRE(keyname,10)
	EXEC
	PERFORM_API_CALL()
END
```

简单来说，我们对每个IP的每一秒都有一个计数器，但每个计数器都有一个额外的设置：它们都将被设置一个10秒的过期时间。这可以使得当时间已经不是当前秒时（此时该计数器也无效了），能够让redis自动移除它。

需要注意的是，这里我们使用multi和exec命令来确保对每个API调用既执行了incr也同时能够执行expire命令。

	multi命令用于标识一个命令集被包含在一个事务块中，exec保证该事务块命令集执行的原子性。
## 模式2

另外的一种实现是采用单一的计数器，但是为了避免race condition（竞态条件），它也更复杂。我们来看几种不同的变体：

### 单一计数器

```
FUNCTION LIMIT_API_CALL(ip):
current = GET(ip)
IF current != NULL AND current > 10 THEN
	ERROR "too many requests per second"
ELSE
	value = INCR(ip)
	IF value == 1 THEN
		EXPIRE(value,1)
	END
	PERFORM_API_CALL()
END
```

该计数器在当前秒内第一次请求被执行时创建，但它只能存活一秒。如果在当前秒内，发送超过10次请求，那么该计数器将超过10。否则它将失效并从0开始重新计数。

在上面的代码中，存在一个race condition。如果因为某个原因，上面的代码只执行了incr命令，却没有执行expire命令，那么这个key将会被泄漏，直到我们再次遇到相同的ip

这种问题也不难处理，可以将incr命令以及另外的expire命令打包到一个lua脚本里，该脚本可以用eval命令提交给redis执行（该方式只在redis版本大于等于2.6之后才能支持）

```
local current
current = redis.call("incr",KEYS[1])
if tonumber(current) == 1 then
	redis.call("expire",KEYS[1],1)
end
```

### list

当然，也有另一种方式来解决这个问题而不需要动用lua脚本，但需要用redis的list数据结构来替代计数器。这种实现方式将会更复杂，并使用更高级的特性。但它有一个好处是记住调用当前API的每个客户端的IP。这种方式可能很有用也可能没用，这取决于应用需求。

```
FUNCTION LIMIT_API_CALL(ip)
current = LLEN(ip)
IF current > 10 THEN
	ERROR "too many requests per second"
ELSE
	IF EXISTS(ip) == FALSE
		MULTI
			RPUSH(ip,ip)
			EXPIRE(ip,1)
		EXEC
	ELSE
		RPUSHX(ip,ip)
	END
	PERFORM_API_CALL()
END
```

**rpushx命令只在key存在时才会将值加入list**

仍然需要注意的是，这里也存在一个race condition（但这却不会产生太大的影响）。问题是：exists可能返回false，但在我们执行multi/exec块内的创建list的代码之前，该list可能已被其他客户端创建。然而，在这个race condition发生时，将仅仅只是丢失一个API调用，所以rate limiting仍然工作得很好。

    这里产生race condition不会有大问题的原因在于，else分支使用的rpushx，它不会导致if not than init的问题，并且expire命令将在创建list的时候以原子的形式捆绑执行。不会产生key泄漏，导致永不失效的情况产生。

# 固定窗口算法Fixed window counters

在固定窗口（Fixed Window）算法中，使用n秒的窗口大小（通常使用对人类友好的值，如60秒或3600秒）来跟踪速率。每个传入的请求都会增加窗口的计数器。如果计数器超过阈值，则丢弃请求。窗口通常由当前时间戳的层定义，因此12:00:03的窗口长度为60秒，应该在12:00:00的窗口中。

## 示例

lua脚本

```
	local key = "rate.limit:" .. KEYS[1] --限流KEY
	local limit = tonumber(ARGV[1]) --限流大小
	local expire_time = ARGV[2] --过期时间
	local current = tonumber(redis.call('get', key) or "0")
	if current + 1 > limit then --超出限流大小
	  return 0
	else --请求数+1，并设置expire_time秒之后过期
	  redis.call("INCRBY", key, "1")
	  redis.call("EXPIRE", key, expire_time)
	  return 1
	end
```

java代码

```java
	private boolean accessLimit(String ip, int limit, int timeout,
	                          Jedis connection) throws IOException {
	List<String> keys = Collections.singletonList(ip);
	List<String> argv = Arrays.asList(String.valueOf(limit), String.valueOf(timeout));
	
	return 1 == (Long) connection.eval(loadScriptString("limit.lua"), keys, argv);
	}
	
	// 加载Lua代码
	private String loadScriptString(String fileName) throws IOException {
	Reader reader =
	        new InputStreamReader(LimitTest.class.getClassLoader().getResourceAsStream(fileName));
	return CharStreams.toString(reader);
	}
```

测试

```
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
TimeUnit.SECONDS.sleep(3);
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
TimeUnit.SECONDS.sleep(2);
System.out.println(new LimitTest().accessLimit("192.168.1.100", 2, 3, jedis));
```

输入如下：

```
true
true
false
true
true
false
```

## 完整代码1：单个限流规则

```
local key = "rate.limit." .. ARGV[1] --限流KEY
local limit = tonumber(ARGV[2]) --限流窗口允许的最大请求数
local interval = ARGV[3] --限流间隔,秒
local now = ARGV[4] --当前Unix时间戳
--计算key
local subject = math.floor(now / interval)
key = key .. '.' .. subject .. '.' .. limit .. '.' .. interval

local current = tonumber(redis.call("incr", key))
if current == 1 then
	redis.call("expire", key, interval)
end

local ttl = redis.call("ttl", key)
local remaining = math.max(limit - current, 0);
--返回值为：是否通过0或1，最大请求数，剩余令牌,限流窗口重置时间
return {limit - current >= 0,  remaining, limit, ttl}
```

## 完整代码2：多个限流规则组合

```
local rate_limits = cjson.decode(ARGV[1])
local now = tonumber(ARGV[2])

local prefix_key = "rate.limit." --限流KEY
local result = {}
local passed = true;
--计算请求是否满足每个限流规则
for i, rate_limit in ipairs(rate_limits) do
	local limit = rate_limit[2]
	local interval = rate_limit[3]
	local subject = math.floor(now / interval)
	local limit_key =  prefix_key .. rate_limit[1] .. '.' .. subject .. '.' .. limit .. '.' .. interval
	local requested_num = tonumber(redis.call('GET', limit_key)) or 0 --请求数，默认为0
	if requested_num >= limit then
		passed = false
		--依次为　限流key,限流大小,限流间隔,剩余请求数,是否通过
		table.insert(result, { limit_key, limit, interval,  math.max(limit - requested_num, 0), 0})
	else
		table.insert(result, { limit_key, limit, interval,  math.max(limit - requested_num, 0), 1 })
	end
end

local summary = { }
for key,value in ipairs(result) do
	local limit_key = value[1]
	local limit = value[2]
	local interval = value[3]
	if passed then
		--如果通过, 将所有的限流请求数加1
		local current = tonumber(redis.call("incr", limit_key))
		if current == 1 then
			redis.call("expire", limit_key, interval)
		end
		--添加两个值　是否通过0或1　剩余请求数
		table.insert(summary, 1)
		table.insert(summary, math.max(limit - current, 0))
	else
		local pass = value[5]
		if not pass or pass == 0 then
			table.insert(summary, 0)
			table.insert(summary, 0)
		else
			table.insert(summary, 1)
			table.insert(summary, value[4])
		end
	end
	local ttl = redis.call("ttl", limit_key)
	--添加两个值　最大请求数　重置时间
	table.insert(summary, limit)
	table.insert(summary, ttl)
end

--返回值为: 是否通过0或1, 剩余令牌, 最大请求数, 限流窗口重置时间 ,...
return summary
```

这种算法的优点是，它可以确保处理更多最近的请求，而不会被旧的请求饿死。然而，发生在窗口边界附近的单个流量突发会导致处理请求的速度增加一倍，因为它允许在短时间内同时处理当前窗口和下一个窗口的请求。另外，如果许多消费者等待一个重置窗口，例如在一个小时的顶部，那么他们可能同时扰乱你的API。

**时间边界问题**

上述Redis的实现在实际应用中存在本篇最开始时候描述的问题**“没有很好的处理单位时间的边界”**。假如一个接口每小时值允许240次调用。如果调用方在6:59PM调用200次请求之后，在7:00PM可以再次调用240次请求，再两分钟之内这个接口就被调用了440次。
针对上述的情况，只有一个计数器是不够的。我们需要将整个流量控制（1小时240次调用）看做一个大的计数桶，然后将这个大的桶拆分成一堆小桶，在每个小桶里都有自己的个性计数。我们可以使用1分钟、5分钟或者15分钟的小桶来拆分1小时的桶（这取决于系统需求，更小的桶意味着更多的内存和清理工作）。
例如我们将1小时的水桶拆分为了1分钟的小桶，那么我们会记录6:00PM,6:01PM,6:02PM的调用次数。但当时间变为7:00PM时，我们需要将6:00PM的桶重置为0，并重新标记桶7:00PM。在7:01PM时会对6:01PM和7:01PM的桶做同样的操作。但是这种实现太复杂，很难做到复杂的限制规则。

# **滑动日志算法Sliding window log**

滑动日志（Sliding Log）速率限制涉及到跟踪每个使用者请求的时间戳日志。这些日志通常存储在按时间排序的散列集或表中。时间戳超过阈值的日志将被丢弃。当新请求出现时，我们计算日志的总和来确定请求率。如果请求将超过阈值速率，则保留该请求。

<table>
	<thead>
		<tr>
			<th>索引</th>
			<td>0</td>
			<td>1</td>
			<td>2</td>
			<td>3</td>
			<td>4</td>
		</tr>
	</thead>
	<tbody>
		<tr>
			<th>时间戳</th>
			<td>12:34:28</td>
			<td>12:34:26</td>
			<td>12:34:14</td>
			<td>12:33:37</td>
			<td>12:33:35</td>
		</tr>
	</tbody>
</table>

假如我们有两个限流规则：每秒1次请求和每分钟5次请求。那么只需要关注最近的第一次请求和第五次请求即可。如果第一个请求距离现在还未过去1秒，那么请求就无法通过每秒1次请求的规则。同理，如果第五个请求距离现在还未过去1分钟，那么请求就无法通过每分钟5次请求的规则。

在12:34:31的时候第一个请求已经过去了3秒钟，但是第5个请求才过去了56秒钟，所以请求依然无法通过每分钟5次请求的规则。我们必须等到4秒钟之后重试。
在12:34:40的时候第一个请求已经过去了12秒钟，第5个请求已经过去了65秒钟，这是我们就可以允许请求通过，并将请求追加到list中。

如果没有其他的限流规则，那么我们永远不会检查第5个请求后的请求，这时我们可以将整个列表的长度控制在5个元素。同时如果我们最大的限流长度是1分钟，我们还可以将list设置为60秒后过期。

在追加一个新的请求之后，list会变为

<table>
	<thead>
		<tr>
			<th>索引</th>
			<td>0</td>
			<td>1</td>
			<td>2</td>
			<td>3</td>
			<td>4</td>
		</tr>
	</thead>
	<tbody>
		<tr>
			<th>时间戳</th>
			<td>12:34:40</td>
			<td>12:34:28</td>
			<td>12:34:26</td>
			<td>12:34:14</td>
			<td>12:33:37</td>
		</tr>
	</tbody>
</table>
lua代码

```
	--两种规则，1秒钟1次请求，每分钟5次请求，为简单起见，规则未使用参数传入
	local current_timestamp = tonumber(ARGV[1]) -- 当前请求的时间
	local key = "rate.limit2:" .. KEYS[1] --限流KEY
	
	--删除列表中超过1分钟的请求
	redis.pcall('ZREMRANGEBYSCORE', key, 0, current_timestamp - 60)
	
	local passed1 = true
	local passed2 = true
	--获取第一个请求
	local last_drip = redis.pcall('ZREVRANGEBYSCORE', key, '+inf', '-inf', 'LIMIT', 0, 1)
	
	--比较上次的时间戳和当前请求的时间戳
	if last_drip[1] then
	    --上次请求的时间
	    local last_time = tonumber(last_drip[1])
	    passed1 = current_timestamp >=  (last_time + 1)
	end
	
	--获取前5个请求
	local last_drip = redis.pcall('ZREVRANGEBYSCORE', key, '+inf', '-inf', 'LIMIT', 0, 5)
	
	--比较上次的时间戳和当前请求的时间戳
	if last_drip[5] then
	    --第5次请求的时间
	    local last_time = tonumber(last_drip[5])
	    passed2  = current_timestamp >=  (last_time + 60)
	end
	
	if passed1 and passed2 then
	  redis.pcall('ZADD', key, current_timestamp, current_timestamp)
	end
	--设置1分钟过期
	redis.call('EXPIRE', key, 60)
	return {current_timestamp,last_drip, passed1, passed2}
```

测试代码

```
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(1);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
TimeUnit.SECONDS.sleep(61);
System.out.println(new LimitTest3().accessLimit("192.168.1.100", jedis));
```

输出

```
	[1484551710, [], 1, 1]
	[1484551710, [1484551710], null, 1]
	[1484551711, [1484551710], 1, 1]
	[1484551712, [1484551711, 1484551710], 1, 1]
	[1484551713, [1484551712, 1484551711, 1484551710], 1, 1]
	[1484551714, [1484551713, 1484551712, 1484551711, 1484551710], 1, 1]
	[1484551715, [1484551714, 1484551713, 1484551712, 1484551711, 1484551710], 1, null]
	[1484551776, [], 1, 1]
```

通过输出我们可以看到:

- 第1个请求通过
- 第2个请求没有通过每秒1次请求的规则
- 等待1秒钟后请求通过
- 等待1秒钟后请求通过
- 等待1秒钟后请求通过
- 等待1秒钟后请求通过
- 等待1秒钟后没有通过每分钟5次请求的规则
- 等待一分钟后请求通过

**但是这个方法在并发大的情况下会对内存有较高的要求，例如每个用户一个队列，每个队列1000个字节，如果有100万个用户的限流规则，那么就会占用1000*100W≈1G的内存**。

参考 https://blog.domaintools.com/2013/04/rate-limiting-with-redis/
https://engineering.classdojo.com/blog/2015/02/06/rolling-rate-limiter/

# 滑动窗口计数Sliding window counters

我们也可以通过将大桶拆分为小桶的方式来处理时间边界问题

```
local subject = "rate.limit." .. ARGV[1] --限流KEY
local limit = tonumber(ARGV[2]) --限流窗口允许的最大请求数
local interval = tonumber(ARGV[3]) --限流间隔,秒
local precision = tonumber(ARGV[4])
local now = tonumber(ARGV[5]) --当前Unix时间戳
--桶的大小不能超过限流的间隔
precision = math.min(precision, interval)

--重新计算限流的KEY，避免传入相同的key，不同的间隔导致冲突
subject = subject .. '.' .. limit .. '.' .. interval .. '.' .. precision

--桶的数量，与位置
local bucket_num = math.ceil(interval / precision)
local oldest_req_key = subject .. ':o'
local bucket_key = math.floor(now / precision)
local trim_before = bucket_key - bucket_num + 1 --需要删除的key
local oldest_req = tonumber(redis.call('GET', oldest_req_key)) or trim_before --最早请求时间，默认为0
trim_before = math.min(oldest_req, trim_before)
--判断当前桶是否存在
local current_bucket = redis.call("hexists", subject, bucket_key)
if current_bucket == 0 then
	redis.call("hset", subject, bucket_key, 0)
end


	
	--请求总数
	local max_req = 0;
	local subject_hash = redis.call("hgetall", subject) or {}
	local last_req = now; --最近的访问时间，计算重置窗口
	for i = 1, #subject_hash, 2 do
	    local ts_key = tonumber(subject_hash[i])
	    if ts_key < trim_before and #subject_hash > 2  then
	        redis.call("hdel", subject, ts_key)
	--        table.insert(dele, ts_key)
	    else
	        local req_num =tonumber(subject_hash[i + 1])
	        max_req = max_req +  req_num
	        if req_num ~= 0 then
	            last_req = math.min(last_req, math.floor(ts_key * precision))
	        end
	    end
	end
	local reset_time =interval - now+  last_req;
	
	if max_req >= limit then
	    --返回值为：是否通过0或1，最大请求数，剩余令牌,限流窗口重置时间
	    return {0, 0, limit, reset_time}
	end
	
-- 当前请求+1
local current = tonumber(redis.call("hincrby", subject, bucket_key, 1))
-- interval+precision之后过期
redis.call("expire", subject, interval+precision)
redis.call("setex", oldest_req_key,interval+precision , now )

--返回值为：是否通过0或1，最大请求数，剩余令牌,限流窗口重置时间
return {1,  limit - max_req - 1, limit, reset_time}

```
**多个滑动窗口组合**

```
	local rate_limits = cjson.decode(ARGV[1])
	local now = tonumber(ARGV[2])
	
	local prefix_key = "rate.limit." --限流KEY
	local result = {}
	local passed = true;
	--计算请求是否满足每个限流规则
	for i, rate_limit in ipairs(rate_limits) do
	    local limit = rate_limit[2]
	    local interval = rate_limit[3]
	    local precision = rate_limit[4]
	    --桶的大小不能超过限流的间隔
	    precision = math.min(precision, interval)
	
	    --重新计算限流的KEY，避免传入相同的key，不同的间隔导致冲突
	    local subject = prefix_key .. rate_limit[1] .. '.' .. limit .. '.' .. interval .. '.' .. precision
	
	    --桶的数量，与位置
	    local bucket_num = math.ceil(interval / precision)
	    local oldest_req_key = subject .. '.o'
	    local bucket_key = math.floor(now / precision)
	    local trim_before = bucket_key - bucket_num + 1 --需要删除的key
	    local oldest_req = tonumber(redis.call('GET', oldest_req_key)) or trim_before --最早请求时间，默认为0
	    trim_before = math.min(oldest_req, trim_before)
	    --判断当前桶是否存在
	    local current_bucket = redis.call("hexists", subject, bucket_key)
	    if current_bucket == 0 then
	        redis.call("hset", subject, bucket_key, 0)
	    end
	
	    --请求总数
	    local max_req = 0;
	    local subject_hash = redis.call("hgetall", subject) or {}
	    local last_req = now; --最近的访问时间，计算重置窗口
	    for i = 1, #subject_hash, 2 do
	        local ts_key = tonumber(subject_hash[i])
	        if ts_key < trim_before and #subject_hash > 2  then
	            redis.call("hdel", subject, ts_key)
	            --        table.insert(dele, ts_key)
	        else
	            local req_num =tonumber(subject_hash[i + 1])
	            max_req = max_req +  req_num
	            if req_num ~= 0 then
	                last_req = math.min(last_req, math.floor(ts_key * precision))
	            end
	        end
	    end
	    local reset_time =interval - now+  last_req;
	    if max_req >= limit then
	        passed = false
	        --依次为　限流key,限流大小,限流间隔,对应的桶,最大请求数,是否成功,重置时间
	        table.insert(result, { subject, limit, interval, bucket_key, max_req, 0, reset_time, precision,oldest_req_key})
	    else
	        table.insert(result, { subject, limit, interval, bucket_key, max_req, 1, reset_time, precision,oldest_req_key})
	    end
	
	end
	
	-- 如果通过，增加每个限流规则
	if passed == true then
	    for key,value in ipairs(result) do
	        -- 当前请求+1
	        local subject = value[1]
	        local interval = value[3]
	        local bucket_key = value[4]
	        local precision = value[8]
	        local oldest_req_key = value[9]
	        local current = tonumber(redis.call("hincrby", subject, bucket_key, 1))
	        -- interval+precision之后过期
	        redis.call("expire", subject, interval+precision)
	        redis.call("setex", oldest_req_key,interval+precision , now )
	    end
	end
	
	-- 返回结果
	local summary = {}
	for key,value in ipairs(result) do
	    local pass = value[6]
	    if not pass or pass == 0 then
	        --添加四个值　是否通过0或1　剩余请求数 最大请求数　重置时间
	        table.insert(summary, 0)
	        table.insert(summary, 0)
	        table.insert(summary, value[2])
	        table.insert(summary, value[7])
	    else
	        table.insert(summary, 1)
	        if passed == true then
	            table.insert(summary, value[2] - value[5] - 1)
	        else
	            table.insert(summary, value[2] - value[5])
	        end
	
	        table.insert(summary, value[2])
	        table.insert(summary, value[7])
	    end
	end
	
	return summary
```

# 令牌桶 Token bucket
可以参考Guava的实现，基于redis实现自己的令牌桶，这个令牌桶有下列属性:

- key: 桶的标识符
- maxAmount: 桶中最大令牌数
- refillTime: 向桶中添加令牌的周期，单位毫秒
- refillAmount: 每次refillTime向桶中添加的令牌数量，默认为1
- value: 桶中可用的令牌数
- lastUpdate: 桶最近一次更新时间（refillTime）


```
local subject = ARGV[1]
local key = "token.bucket." .. subject --桶的标识符
local burst = math.max(tonumber(ARGV[2]), 1) --桶中最大令牌数，最小值1，
local refillTime = tonumber(ARGV[3]) or 1000-- 向桶中添加令牌的周期，单位毫秒
local refillAmount = math.max(tonumber(ARGV[4]), 1) or 1 -- 每次refillTime向桶中添加的令牌数量，默认为1
local tokens_to_take       = tonumber(ARGV[5]) or 1 --当前请求需要的令牌数量
local now = tonumber(ARGV[6]) --当前时间，单位毫秒

local available_tokens = burst --可用的令牌数默认等于桶的容量
local last_update = now --第一次的last_update等于当前时间
local current = redis.call('HMGET', key, 'last_update', 'available_tokens')
if current.err ~= nil then
	redis.call('DEL', key)
	current = {}
	redis.log(redis.LOG_NOTICE, 'Cannot get ratelimit ' .. key)
	return redis.error_reply('Cannot get ratelimit ' .. key)
end


		
		--计算从上次的时间戳与当前时间戳计算应该添加的令牌数
		if current[1] then
		    --上次请求的时间
		    last_update = current[1]
		    local content = current[2]
		    --计算应该生成的令牌数
		    local delta_ms = math.max(now - last_update, 0)
		    local refillCount  = math.floor(delta_ms / refillTime) * refillAmount
		    --如果桶满，直接使用桶的容量
		    available_tokens = math.min(content + refillCount, burst)
		end
		
		-- 计算是否有足够的令牌给调用方
		local enough_tokens = available_tokens >= tokens_to_take
		--之后的请求要偿还之前发生的突发
		
-- 将令牌给调用方之后，桶中剩余的令牌数
if enough_tokens then
	available_tokens = math.min(available_tokens - tokens_to_take, burst)
	last_update = last_update + math.floor(tokens_to_take / refillAmount * refillTime)
else
	redis.log(redis.LOG_NOTICE, 'Cannot get enough tokens, tokens_to_take:' .. tokens_to_take .. ',available_tokens:' .. available_tokens)
	--计算恢复的时间
	local reset_time = last_update - now
	if reset_time < 0 then
		reset_time = now - last_update
	end
	return { 0, math.max(available_tokens, 0), burst, reset_time}
end
--重新设置令牌桶
redis.call('HMSET', key,
	'last_update', last_update,
	'available_tokens', available_tokens)

--如果没有新的请求过来，在桶满之后可以直接将该令牌删除。
redis.call('PEXPIRE', key, math.ceil((burst /  refillAmount) * refillTime))

return {1,  math.max(available_tokens, 0), burst,last_update - now }
```

参考:https://github.com/auth0/limitd/blob/master/lib/db/redis/drip_and_take.lua

# nginx接入层限流
nginx自带两个限流模块

- 连接数限流模块`ngx_http_limit_conn_module`
- 请求限流模块`ngx_http_limit_req_module` 漏桶实现

OpenResty提供了`lua-resty-limit-traffic`可以实现更复杂的限流

## ngx_http_limit_conn_module

我们经常会遇到这种情况，服务器流量异常，负载过大等等。对于大流量恶意的攻击访问，会带来带宽的浪费，服务器压力，影响业务，往往考虑对同一个ip的连接数，并发数进行限制。

ngx_http_limit_conn_module 模块来实现该需求。该模块可以根据定义的键来限制每个键值的连接数，如同一个IP来源的连接数。并不是所有的连接都会被该模块计数，只有那些正在被处理的请求（这些请求的头信息已被完全读入）所在的连接才会被计数。

**配置示例**

可以在nginx_conf的http{}中加上如下配置实现限制：

```
#限制每个用户的并发连接数，取名one
limit_conn_zone $binary_remote_addr zone=one:10m;

#配置记录被限流后的日志级别，默认error级别
limit_conn_log_level error;
#配置被限流后返回的状态码，默认返回503
limit_conn_status 503;
```

然后在server{}里加上如下代码：(也可以设置在单独的location)

```
#限制用户并发连接数为1
limit_conn one 1;
```

- limit_conn: 配置存放key和计数器的共享区域和指定key的最大连接数，此处指定最大连接数是1，表述nginx最多同时并发处理一个请求
- limit_conn_zone 配置限流key及存放key对应信息的共享内存区域大小，此处的key是IP地址，也可以使用`$server_name`做完key来现在域名级别的最大连接数
- limit_conn_status 配置被限流后返回的状态码，默认503
-  limit_conn_log_level 配置记录被限流后的日志级别，默认error

然后我们是使用ab测试来模拟并发请求：

```
ab -n 5 -c 5 http://localhost/
```

另外刚才是配置针对单个IP的并发限制，还是可以针对域名进行并发限制，配置和客户端IP类似。

```
#http{}段配置
limit_conn_zone $ server_name zone=perserver:10m;
#server{}段配置
limit_conn perserver 1;
```

## ngx_http_limit_req_module

ngx_http_limit_req_module模块可以通过定义的键值来限制请求处理的频率。特别的，可以限制来自单个IP地址的请求处理频率。限制的方法是使用了漏斗算法，每秒固定处理请求数，推迟过多请求。如果请求的频率超过了限制域配置的值，请求处理会被延迟或被丢弃，所以所有的请求都是以定义的频率被处理的。

在http{}中配置

```
#区域名称为one，大小为10m，平均处理的请求频率不能超过每秒一次。
limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
```

在server{}中配置（也可以设置在单独的location）

```
#设置每个IP桶的数量为5
limit_req zone=one burst=5 nodelay;
#limit_req zone=one burst=5;
```

上面设置定义了每个IP的请求处理只能限制在每秒1个。并且服务端可以为每个IP缓存5个请求，如果操作了5个请求，请求就会被丢弃。

- limit_req 配置限流区域、桶容量（突发容量，默认为0）、是否延迟模式（默认延迟）
- limit_req_zone 配置限流key，存放key对应信息的共享内存区域大小、固定请求速率，请求速率使用rate参数设置，支持`10r/s`和`60r/m`最终都会转换为每秒的固定请求
- limit_conn_status 配置被限流后返回的状态码，默认503
- limit_conn_log_level 配置记录被限流后的日志级别，默认error

延迟模式下，时间窗口内的请求会被暂存到桶中，并以固定平均速率处理请求

limit_req_zone/limit_conn_zone定义的内存不足，后续的请求将一直被限流，所以要根据需求设置好相应的内存大小

 limit_req_zone/limit_conn_zone的限流都是基于单nginx，假设我们接入层有多个nginx，这时我们就需要使用lua-resty-limit-traffic

## lua-resty-limit-traffic

OpenResty的lua-resty-limit-traffic也提供了limit.conn和limit.req实现

配置用来存放限流用的共享字典：

```
lua_shared_dict limit_req_store 100m;
```

限流的lua代码

```
local limit_req = require "resty.limit.req"
local rate = 2 --固定平均速率 2r/s
local burst = 3 --桶容量
local error_status = 503
local nodelay = false --是否需要不延迟处理
local lim, err = limit_req.new("limit_req_store", rate, burst)
if not lim then --没定义共享字典
    ngx.exit(error_status)
end
local key = ngx.var.binary_remote_addr --IP维度的限流
--流入请求，如果请求需要被延迟则delay > 0
local delay, err = lim:incoming(key, true)
if not delay and err == "rejected" then --超出桶大小了
    ngx.exit(error_status)
end
if delay > 0 then --根据需要决定是延迟或者不延迟处理
    if nodelay then
        --直接突发处理了
    else
        ngx.sleep(delay) --延迟处理
    end
end
```

即限流逻辑再nginx access阶段被访问，如果不被限流继续后续流程；如果需要被限流要么sleep一段时间继续后续流程，要么返回相应的状态码拒绝请求。

另外在使用Nginx+Lua时也可以获取ngx.var.connections_active进行过载保护，即如果当前活跃连接数超过阈值进行限流保护。

```
if tonumber(ngx.var.connections_active) >= tonumber(limit) then
    //限流
end
```

nginx也提供了limit_rate用来对流量限速，如limit_rate 50k，表示限制下载速度为50k。

# 参考资料

http://xiaobaoqiu.github.io/blog/2015/07/02/ratelimiter/

http://www.cnblogs.com/exceptioneye/p/4783904.html

https://blog.jamespan.me/2015/10/19/traffic-shaping-with-token-bucket/

http://jinnianshilongnian.iteye.com/blog/2305117

https://redis.io/commands/INCR#pattern-rate-limiter

https://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis/155

http://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis-part-2/166

http://vinoyang.com/2015/08/23/redis-incr-implement-rate-limit/

https://blog.figma.com/an-alternative-approach-to-rate-limiting-f8a06cf7c94c