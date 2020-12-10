---
layout: post
title: Spring Cloud CircuitBreaker - Resilience4j使用
date: 2020-08-16
categories:
    - Spring
comments: true
permalink: spring-cloud-circuit-resilience4j.html
---

`Resilience4j` 提供了 `RateLimiter` ，`CircuitBreaker` ，`BulkHead` ，`TimeLimiter` ，`Retry` ，`Cache` 等功能。 其中RateLimiter 是一个限速器，CircuitBreaker 实现了断路器模式，BulkHead  限制并发，TimeLimiter实际上是一个超时器， Retry 实现自动重试功能，Cache自动使用缓存。Resilience4j 使用 **装饰器模式** 实现容错功能，使用者只需在业务上层做一下包装即可， 无需侵入业务代码。

> 需要注意的是 `CircuitBreaker` 并不会限制并发数，并发数的限制应该使用 `BulkHead` 。

# 1. CircuitBreaker

**初始化熔断器**

CircuitBreakerRegistry负责创建和管理熔断器实例CircuitBreaker，它是线程安全的，提供原子性操作。

```text
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.ofDefaults();
```

该方式使用默认的全局配置CircuitBreakerConfig创建熔断器实例，你也可以选择使用定制化的配置，可选项有：

1. failureRateThreshold： 触发熔断的失败率阈值，默认50
2. waitDurationInOpenState：熔断器从打开状态到半开状态的等待时间，默认60s
3. ringBufferSizeInHalfOpenState：熔断器在半开状态时环状缓冲区的大小（新版本替换为允许并发数），会限制线程的并发量，例如缓冲区为10则每次只会允许10个请求调用后端服务，默认10
4. ringBufferSizeInClosedState：熔断器在关闭状态时环状缓冲区的大小（新版本替换为滑动窗口），不会限制线程的并发量，在熔断器发生状态转换前所有请求都会调用后端服务，默认100
5. automaticTransitionFromOpenToHalfOpenEnabled：如果置为true，当等待时间结束会自动由打开变为半开，若置为false，则需要一个请求进入来触发熔断器状态转换，默认false
6. recordExceptions：需要记录为失败的异常列表，默认空
7. ignoreExceptions：需要忽略的异常列表，默认空
8. recordException：自定义的谓词逻辑用于判断异常是否需要记录或者需要忽略，默认所有异常都进行记录
9. permittedNumberOfCallsInHalfOpenState：熔断器在半开状态时允许的并发数，默认10
10. maxWaitDurationInHalfOpenState：熔断器进入打开状态时应该在半开状态停留的时间，默认0
11. slidingWindow：滑动窗口由3个参数租车
	- slidingWindowType: 按时间TIME_BASED，按请求数COUNT_BASED，默认COUNT_BASED，如果是TIME_BASED，窗口大小单位是秒
	- slidingWindowSize：滑动窗口大小，默认100
	- minimumNumberOfCalls：计算熔断之前的最小请求数，默认100

```
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
		.failureRateThreshold(20)
		.waitDurationInOpenState(Duration.ofMillis(1000))
		.slidingWindow(5, 5, CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
		.permittedNumberOfCallsInHalfOpenState(2)
		.build();	
```

通过注册中心创建熔断器

```
// 使用定制化配置创建熔断器注册中心
CircuitBreakerRegistry circuitBreakerRegistry = CircuitBreakerRegistry.of(circuitBreakerConfig);

// 从注册中心获取使用默认配置的熔断器
CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("otherName");

// 从注册中心获取使用定制化配置的熔断器
CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("uniqueName", circuitBreakerConfig);
```

不经过注册中心，直接创建熔断器实例

```text
CircuitBreaker defaultCircuitBreaker = CircuitBreaker.ofDefaults("testName");

CircuitBreaker customCircuitBreaker = CircuitBreaker.of("testName", circuitBreakerConfig);
```

熔断器采用装饰器模式

```
CircuitBreaker circuitBreaker = CircuitBreaker.of("backendService", config);
Function<String, Account> docorated = CircuitBreaker.decorateFunction(circuitBreaker, accountRemoteService::getAccountInfo);
for (int i = 0; i < 10; i ++) {
	try {
		Account account = docorated.apply("1");
		System.out.println(account);
	} catch (Exception e) {
		System.out.println(e.getClass());
	}
}
```

断路器也支持重置，重置之后数据清空，恢复到初始状态

```java
circuitBreaker.reset();
```

服务请求降级

```
Supplier<String> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, backendService::doSomething);

String result = Try.ofSupplier(decoratedSupplier)
    .recover(throwable -> "Hello from Recovery").get();
```

```
CheckedFunction0<Account> function0 = CircuitBreaker.decorateCheckedSupplier(circuitBreaker, () -> accountRemoteService.getAccountInfo("1"));
Try<Account> result = Try.of(function0).recover(throwable -> {
	Account account = new Account();
	account.setName("fallback");
	account.setId("fallback");
	return account;
});
```

熔断器事件CircuitBreakerEvent包含状态转换、重置、成功、失败异常、忽略异常事件，所有事件包含发生时间、处理时长的信息。

```
circuitBreaker.getEventPublisher()
		.onEvent(event -> System.out.println(event));
```

```
circuitBreaker.getEventPublisher()
    .onSuccess(event -> logger.info(...))
    .onError(event -> logger.info(...))
    .onIgnoredError(event -> logger.info(...))
    .onReset(event -> logger.info(...))
    .onStateTransition(event -> logger.info(...));
```

也可以使用CircularEventConsumer将事件存储在缓存中

```
CircularEventConsumer<CircuitBreakerEvent> ringBuffer = 
  new CircularEventConsumer<>(10);
circuitBreaker.getEventPublisher().onEvent(ringBuffer);
List<CircuitBreakerEvent> bufferedEvents = ringBuffer.getBufferedEvents()
```

还可以将EventPublisher转换为RxJava/Reactor的事件流，该方法的优势在于，可以进一步指定调度器进行异步化处理：

```
RxJava2Adapter.toFlowable(circuitBreaker.getEventPublisher())
    .filter(event -> event.getEventType() == Type.ERROR)
    .cast(CircuitBreakerOnErrorEvent.class)
    .subscribe(event -> logger.info(...))
```

**环形缓冲区**

Resilience4j的早期版本记录请求状态的数据结构采用Ring Bit Buffer(环形缓冲区)进行存储，新版本使用滑动窗口来进行存储的，Ring Bit Buffer在内部使用BitSet这样的数据结构来进行存储，BitSet的结构如下图所示：

![](/assets/images/posts/resilience4j/resilience4j-1.jpg)

每一次请求的成功或失败状态只占用一个bit位，与boolean数组相比更节省内存。BitSet使用long[]数组来存储这些数据，意味着16个值(64bit)的数组可以存储1024个调用状态。

计算失败率需要填满环形缓冲区。例如，如果环形缓冲区的大小为10，则必须至少请求满10次，才会进行故障率的计算，如果仅仅请求了9次，即使9个请求都失败，熔断器也不会打开。但是CLOSE状态下的缓冲区大小设置为10并不意味着只会进入10个 请求，在熔断器打开之前的所有请求都会被放入。

当故障率高于设定的阈值时，熔断器状态会从由CLOSE变为OPEN。这时所有的请求都会抛出CallNotPermittedException异常。当经过一段时间后，熔断器的状态会从OPEN变为HALF_OPEN，HALF_OPEN状态下同样会有一个Ring Bit Buffer，用来计算HALF_OPEN状态下的故障率，如果高于配置的阈值，会转换为OPEN，低于阈值则装换为CLOSE。与CLOSE状态下的缓冲区不同的地方在于，HALF_OPEN状态下的缓冲区大小会限制请求数，只有缓冲区大小的请求数会被放入。

除此以外，熔断器还会有两种特殊状态：DISABLED（始终允许访问）和FORCED_OPEN（始终拒绝访问）。这两个状态不会生成熔断器事件（除状态装换外），并且不会记录事件的成功或者失败。退出这两个状态的唯一方法是触发状态转换或者重置熔断器。

**滑动窗口**

我们可以这么来理解滑动窗口：一位乘客坐在正在行驶的列车的靠窗座位上，列车行驶的公路两侧种着一排挺拔的白杨树，随着列车的前进，路边的白杨树迅速从窗口滑过，我们用每棵树来代表一个请求，用列车的行驶代表时间的流逝，那么，列车上的这个窗口就是一个典型的滑动窗口，这个乘客能通过窗口看到的白杨树就是滑动窗口要统计的数据。

bucket 是统计滑动窗口数据时的最小单位。同样类比列车窗口，在列车速度非常快时，如果每掠过一棵树就统计一次窗口内树的数据，显然开销非常大，如果乘客将窗口分成十分，列车前进行时每掠过窗口的十分之一就统计一次数据，开销就完全可以接受了。 Hystrix 的 bucket （桶）也就是窗口 N分之一 的概念。

![](/assets/images/posts/resilience4j/resilience4j-2.png)

上图的每个小矩形代表一个桶，可以看到，每个桶都记录着1秒内的四个指标数据：成功量、失败量、超时量和拒绝量。10个桶合起来是一个完整的滑动窗口，所以计算一个滑动窗口的总数据需要将10个桶的数据加起来。

# 2. RateLimiter

RateLimiter的配置方式与熔断类似，有对应的RateLimiterRegistry 和 RateLimiterConfig，自定义配置的可选项有：

1. timeoutDuration：一个线程等待令牌的时间，默认5秒
2. limitRefreshPeriod：刷新令牌的时间，默认500ns
3. limitForPeriod: 一个周期内允许的请求，默认50

```
RateLimiterConfig config = RateLimiterConfig.custom()
		.limitRefreshPeriod(Duration.ofMillis(3000))
		.limitForPeriod(5)
		.timeoutDuration(Duration.ofMillis(1000))
		.build();
RateLimiter rateLimiter = RateLimiter.of("ratelimiter", config);
Function<String, Account> accountFunction = RateLimiter.decorateFunction(rateLimiter, accountRemoteService::getAccountInfo);
```

可以在运行时动态修改RateLimiter配置

```
rateLimiter.changeLimitForPeriod(5);
rateLimiter.changeTimeoutDuration(Duration.ofMillis(1000));
```

监听事件

```
rateLimiter.getEventPublisher()
		.onSuccess(event -> logger.info(...))
.onFailure(event -> logger.info(...));

rateLimiter.getEventPublisher()
		.onEvent(event -> logger.info(...));
```

# 3. Bulkhead

Bulkhead通过信号量的方式隔离不同种类的调用，并进行流控，这样可以避免某类调用异常危及系统整体。

自定义配置的可选项有：

1. maxConcurrentCalls：最大并行数，默认25
2. maxWaitDuration：尝试进入饱和态的Bulkhead时，线程的最大阻塞时间，默认0

```

BulkheadConfig config = BulkheadConfig.custom()
		.maxConcurrentCalls(5)
		.maxWaitDuration(Duration.ofSeconds(1))
		.build();
Bulkhead bulkhead = Bulkhead.of("bulkhead", config);
Function<String, Account> accountFunction = Bulkhead.decorateFunction(bulkhead, accountRemoteService::getAccountInfo);
```

# 4. Retry

可配置选项有：

1. maxAttempts：最大重试次数，默认3
2. waitDuration：重试间隔，默认500ms
3. intervalFunction：修改重试间隔
4. retryOnResultPredicate、retryOnExceptionPredicate： 评估是否重试的Predicate
5. retryExceptions
6. ignoreExceptions

```
RetryConfig config = RetryConfig.custom()
		.retryOnResult(o -> {
			Account account = (Account) o;
			return "retry".equals(account.getName());
		})
		.build();
Retry retry = Retry.of("retry", config);

Function<String, Account> accountFunction = Retry.decorateFunction(retry, accountRemoteService::getAccountInfo);
```

# 5. TimeLimiter

限时器与Future配合使用，限时器将Future Supplier转换为Callable，它将尝试在给定时间内获取Future的值，如果超时未获取到，Future将会被取消。

```
TimeLimiterConfig config = TimeLimiterConfig.custom()
		.timeoutDuration(Duration.ofMillis(50))
		.cancelRunningFuture(true)
		.build();
TimeLimiter timeLimiter = TimeLimiter.of("TimeLimiter", config);

ExecutorService executorService = Executors.newSingleThreadExecutor();

// 将待执行任务提交到线程池，获取Future Supplier
Supplier<Future<Account>> futureSupplier = () -> executorService.submit(() -> accountRemoteService.getAccountInfo("1"));
// 使用限时器包装Future Supplier
Callable restrictedCall = TimeLimiter
		.decorateFutureSupplier(timeLimiter, futureSupplier);
// 若任务执行超时，onFailure会被触发
Try.of(restrictedCall::call)
		.onFailure(throwable -> throwable.printStackTrace());
```

可以将限时器与熔断器配合使用，在超时次数过多时触发熔断

```
Callable restrictedCall = TimeLimiter
    .decorateFutureSupplier(timeLimiter, futureSupplier);

Callable chainedCallable = CircuitBreaker.decorateCallable(circuitBreaker, restrictedCall);

Try.of(chainedCallable::call)
    .onFailure(throwable -> LOG.info("We might have timed out or the circuit breaker has opened."));
```

# 6. Feign集成

```
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("backendName");
RateLimiter rateLimiter = RateLimiter.ofDefaults("backendName");
FeignDecorators decorators = FeignDecorators.builder()
        .withRateLimiter(rateLimiter)
        .withCircuitBreaker(circuitBreaker)
        .build();
AccountClient accountClient = Resilience4jFeign
        .builder(decorators)
        .decoder(new JacksonDecoder())
        .target(AccountClient.class, "http://localhost:8080/");
```

CircuitBreaker和RateLimiter是由他们声明的顺序决定的。

# 7. Spring Boot集成

官方文档很详细了，这里仅做个简单记录

>  https://resilience4j.readme.io/docs/getting-started-3

引入依赖

```
<dependency>
	<groupId>io.github.resilience4j</groupId>
	<artifactId>resilience4j-spring-boot2</artifactId>
	<version>1.6.1</version>
</dependency>
```

> 不要写成了resilience4j-spring

配置信息见官方文档

```
@FeignClient(value = "account", url = "http://localhost:8080")
public interface AccountClient {

    @CircuitBreaker(name = "account", fallbackMethod = "getFallback")
    @GetMapping("/api/accounts/{id}")
    Account getAccountInfo(@PathVariable("id") String id);

	// 可以根据不同的异常定义重载方法
    default Account getFallback(String id, Exception e) {
        Account account = new Account();
        account.setId("fallback");
        account.setName("fallback");
        return account;
    }
}
```

消息监听

```
@Bean
public RegistryEventConsumer<CircuitBreaker> myRetryRegistryEventConsumer() {

	return new RegistryEventConsumer<CircuitBreaker>() {
		@Override
		public void onEntryAddedEvent(EntryAddedEvent<CircuitBreaker> entryAddedEvent) {
			entryAddedEvent.getAddedEntry().getEventPublisher().onEvent(event -> System.out.println(event));
		}

		@Override
		public void onEntryRemovedEvent(EntryRemovedEvent<CircuitBreaker> entryRemoveEvent) {

		}

		@Override
		public void onEntryReplacedEvent(EntryReplacedEvent<CircuitBreaker> entryReplacedEvent) {

		}
	};
}
```

定义各个组件的顺序

```
resilience4j:
  circuitbreaker:
    circuitBreakerAspectOrder: 1
  retry:
    retryAspectOrder: 2
```

```
@CircuitBreaker(name = BACKEND, fallbackMethod = "fallback")
@RateLimiter(name = BACKEND)
@Bulkhead(name = BACKEND)
@Retry(name = BACKEND, fallbackMethod = "fallback")
@TimeLimiter(name = BACKEND)
public Mono<String> method(String param1) {
    return Mono.error(new NumberFormatException());
}

private Mono<String> fallback(String param1, IllegalArgumentException e) {
    return Mono.just("test");
}

private Mono<String> fallback(String param1, RuntimeException e) {
    return Mono.just("test");
}
```

# 8. Spring Cloud集成

Spring官方的依赖没有定义`EnableCircuitBreaker`的实现，等后续版本尝试

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

Resilience4j提供的`resilience4j-spring-cloud2`比`resilience4j-spring-boot2`多了一个从配置中心刷新配置的功能。因为在配置中心模块已经有过例子，就不过多描述了。

# 9. 参考资料

https://zhuanlan.zhihu.com/p/51008825

