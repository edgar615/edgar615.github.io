---
layout: post
title: 限流（2）- Guava
date: 2020-04-16
categories:
    - 分布式
comments: true
permalink: ratelimit-guava.html
---

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

# 1. 创建RateLimiter

```java
public static RateLimiter create(double permitsPerSecond);	
```

创建一个稳定输出令牌的RateLimiter，保证了平均每秒不超过permitsPerSecond个请求，当请求到来的速度超过了permitsPerSecond，保证每秒只处理permitsPerSecond个请求，当这个RateLimiter使用不足(即请求到来速度小于permitsPerSecond)，会囤积最多permitsPerSecond个请求.

```java
public static RateLimiter create(double permitsPerSecond, long warmupPeriod, TimeUnit unit)
```

创建一个稳定输出令牌的RateLimiter，保证了平均每秒不超过permitsPerSecond个请求， 还包含一个热身期(warmup period),热身期内，RateLimiter会平滑的将其释放令牌的速率加大，直到起达到最大速率，同样，如果RateLimiter在热身期没有足够的请求(unused),则起速率会逐渐降低到冷却状态。设计这个的意图是为了满足那种资源提供方需要热身时间，而不是每次访问都能提供稳定速率的服务的情况(比如带缓存服务，需要定期刷新缓存的)，参数warmupPeriod和unit决定了其从冷却状态到达最大速率的时间

# 2. 获取令牌

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

# 3. 设计

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

# 4. 平滑突发限流 SmoothBursty

## 4.1. 示例1
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

## 4.2. 示例2

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

## 4.3. 示例3

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

# 5. 类似漏桶的算法 SmoothWarmingUp

SmoothBursty允许一定程度的突发，但假设突然连了很大的流量，那么系统很可能抗不住这种突发。
因此需要一个**平滑速率的限流工具，从而使系统冷启动后慢慢的趋于平均固定速率**（即刚开始速率小一些，然后慢慢趋于我们设置的固定速率）

漏桶算法强制一个常量的输出速率而不管输入数据流的突发性,当输入空闲时，该算法不执行任何动作.就像用一个底部开了个洞的漏桶接水一样,水进入到漏桶里,
桶里的水通过下面的孔以固定的速率流出,当水流入速度过大会直接溢出,可以看出漏桶算法能强行限制数据的传输速率.

类似漏桶的实现SmoothWarmingUp

```java
RateLimiter limiter = RateLimiter.create(5, 1000, TimeUnit.MILLISECONDS);
```

后面两个参数表示从冷启动速率过渡到平均速率的时间间隔，可以通过调节warmupPeriod参数实现一开始就是平滑固定速率
## 5.1. 示例1

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

## 5.2 内部实现
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

# 6. 参考资料

http://xiaobaoqiu.github.io/blog/2015/07/02/ratelimiter/

http://www.cnblogs.com/exceptioneye/p/4783904.html

https://blog.jamespan.me/2015/10/19/traffic-shaping-with-token-bucket/

http://jinnianshilongnian.iteye.com/blog/2305117

https://redis.io/commands/INCR#pattern-rate-limiter

https://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis/155

http://www.binpress.com/tutorial/introduction-to-rate-limiting-with-redis-part-2/166

http://vinoyang.com/2015/08/23/redis-incr-implement-rate-limit/

https://blog.figma.com/an-alternative-approach-to-rate-limiting-f8a06cf7c94c