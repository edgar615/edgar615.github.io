---
layout: post
title: Spring Cloud CircuitBreaker - Hystrix 
date: 2020-08-15
categories:
    - Spring
comments: true
permalink: spring-cloud-circuit-hystrix.html
---

> Hystrix已停止更新

# 1. Get Started


在 Hystrix 中，最核心的莫过于HystrixCommand 类。HystrixCommand 是一个抽象类，只包含了一个抽象方法，即如下所示的 run 方法：

```
protected abstract R run() throws Exception;
```

在微服务架构中，我们通常在这个 run 方法中添加对远程服务的访问代码。

同时我们在 HystrixCommand 类中还发现了另一个很有用的方法 getFallback。这个方法用于在 HystrixCommand 子类中设置服务回退函数的具体实现

```
protected R getFallback() {
	throw new UnsupportedOperationException("No fallback available.");
}
```

引入依赖

```
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
    </dependencies>
```

基于前面对 HystrixCommand 抽象类的理解，我们就可以提供一个该类的子类来实现服务隔离。针对服务隔离，Hystrix 组件在提供了线程池隔离机制的同时，还实现了信号量隔离。这里，我们基于最常用的线程池隔离来进行介绍。典型的 HystrixCommand 子类代码风格如下所示：

```
public class GetAccountCommand extends HystrixCommand<Account> {

    private final AccountClient accountClient;

    private final String id;

    protected GetAccountCommand(AccountClient accountClient, String id) {
        super(Setter.withGroupKey(
                //设置命令组
                HystrixCommandGroupKey.Factory.asKey("circuitExample"))
                //设置命令键
                .andCommandKey(HystrixCommandKey.Factory.asKey("hystrix"))
                //设置线程池键
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("account"))
                //设置命令属性
                .andCommandPropertiesDefaults(
                        HystrixCommandProperties.Setter()
                                .withExecutionTimeoutInMilliseconds(5000))
                //设置线程池属性
                .andThreadPoolPropertiesDefaults(
                        HystrixThreadPoolProperties.Setter()
                                .withMaxQueueSize(10)
                                .withCoreSize(2))
        );
        this.accountClient = accountClient;
        this.id = id;
    }

    @Override
    protected Account run() throws Exception {
        return accountClient.getAccountInfo(id);
    }

    @Override
    protected Account getFallback() {
        Account account = new Account();
        account.setId("fallback");
        account.setName("fallback");
        return account;
    }
}
```

上述代码中使用了 Hystrix 中的很多常见的配置项，这些配置项大多数也涉及线程池隔离的相关概念。在 Hystrix 中，从控制粒度上讲，开发人员可以从服务分组和服务本身这两个维度出发，对线程隔离机制进行配置。也就是说我们既可以把一批服务都划分到一个线程池中，也可以把单个服务划分到一个线程池中。上述代码中的 HystrixCommandGroupKey 和 HystrixCommandKey 分别用来配置服务分组名称和服务名称，然后 HystrixThreadPoolKey 用来配置线程池的名称。

当我们根据需要设置了分组、服务以及线程池名称后，接下来就需要指定与线程池相关的各个属性。这些属性都包含在 HystrixThreadPoolProperties 中。例如，在上述代码中，我们使用 maxQueueSize 配置线程池队列的最大值，使用 coreSize 配置核心线程池的最大值等。同时，我们也注意到可以使用 withExecutionTimeoutInMilliseconds 配置项来指定请求的超时时间。

测试

```
GetAccountCommand command = new GetAccountCommand(accountClient, "1");
Account account = command.execute();
```

**注意：这里要执行execute方法，而不是run方法**

异步调用

```
Future<Account> future = command.queue();
System.out.println(future.get());
```

响应式

```
Observable<Account> observable = command.observe();
Account account = observable.toBlocking().first();
System.out.println(account);
```

> 也可以直接使用HystrixObservableCommand

# 2. @HystrixCommand

Hystrix 为简化开发过程而专门提供的一个注解`@HystrixCommand`

```
public class AccountServiceImpl implements AccountService {

    @Autowired
    private AccountClient accountClient;

    @Override
    @HystrixCommand(
            groupKey = "circuitExample",
            commandKey = "hystrix",
            threadPoolKey = "account",
            commandProperties = {
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000")
            },
            threadPoolProperties =
                    {
                            @HystrixProperty(name="coreSize",value="2"),
                            @HystrixProperty(name="maxQueueSize",value="10")
                    },
            fallbackMethod = "getFallback"
    )
    public Account getAccountInfo(String id) {
        return accountClient.getAccountInfo(id);
    }

    private Account getFallback(String id) {
        Account account = new Account();
        account.setId("fallback");
        account.setName("fallback");
        return account;
    }
}
```

`execution.isolation.thread.timeoutInMilliseconds` 配置项就是用来设置 Hystrix 的超时时间。

## 2.1. 参数

下面是 HystrixCommand 支持的参数，除了 `commandKey/observableExecutionMode/fallbackMethod` 外，都可以使用 `@DefaultProperties` 配置默认值。

- **commandKey**：用来标识一个 Hystrix 命令，默认会取被注解的方法名。需要注意：`Hystrix 里同一个键的唯一标识并不包括 groupKey`，建议取一个独一二无的名字，防止多个方法之间因为键重复而互相影响。
- **groupKey**：一组 Hystrix 命令的集合， 用来统计、报告，默认取类名，可不配置。
- **threadPoolKey**：用来标识一个线程池，`如果没设置的话会取 groupKey`，很多情况下都是同一个类内的方法在共用同一个线程池，如果两个共用同一线程池的方法上配置了同样的属性，在第一个方法被执行后线程池的属性就固定了，所以属性会以第一个被执行的方法上的配置为准。
- **commandProperties**：与此命令相关的属性。
- **threadPoolProperties**：与线程池相关的属性，
- **observableExecutionMode**：当 Hystrix 命令被包装成 RxJava 的 Observer 异步执行时，此配置指定了 Observable 被执行的模式，默认是 `ObservableExecutionMode.EAGER`，Observable 会在被创建后立刻执行，而 `ObservableExecutionMode.EAGER`模式下，则会产生一个 Observable 被 subscribe 后执行。我们常见的命令都是同步执行的，此配置项可以不配置。
- **ignoreExceptions**：默认 Hystrix 在执行方法时捕获到异常时执行回退，并统计失败率以修改熔断器的状态，而被忽略的异常则会直接抛到外层，不会执行回退方法，也不会影响熔断器的状态。
- **raiseHystrixExceptions**：当配置项包括 `HystrixRuntimeException` 时，所有的未被忽略的异常都会被包装成 HystrixRuntimeException，配置其他种类的异常好像并没有什么影响。
- **fallbackMethod**：方法执行时熔断、错误、超时时会执行的回退方法，需要保持此方法与 Hystrix 方法的签名和返回值一致。
- **defaultFallback**：默认回退方法，当配置 fallbackMethod 项时此项没有意义，另外，默认回退方法不能有参数，返回值要与 Hystrix方法的返回值相同。

## 2.2. commandProperties

Hystrix 的命令属性是由 `@HystrixProperty` 注解数组构成的，HystrixProperty 由 name 和 value 两个属性，数据类型都是字符串。

以下将所有的命令属性分组来介绍。

**线程隔离(Isolation)**

- **execution.isolation.strategy**： 配置请求隔离的方式，有 threadPool（线程池，默认）和 semaphore（信号量）两种，信号量方式高效但配置不灵活，我们一般采用 Java 里常用的线程池方式。
- **execution.timeout.enabled**：是否给方法执行设置超时，默认为 true。
- **execution.isolation.thread.timeoutInMilliseconds**：方法执行超时时间，默认值是 1000，即 1秒，此值根据业务场景配置。
- **execution.isolation.thread.interruptOnTimeout**： **execution.isolation.thread.interruptOnCancel**：是否在方法执行超时/被取消时中断方法。需要注意在 JVM 中我们无法强制中断一个线程，如果 Hystrix 方法里没有处理中断信号的逻辑，那么中断会被忽略。
- **execution.isolation.semaphore.maxConcurrentRequests**：默认值是 10，此配置项要在 `execution.isolation.strategy` 配置为 `semaphore` 时才会生效，它指定了一个 Hystrix 方法使用信号量隔离时的最大并发数，超过此并发数的请求会被拒绝。信号量隔离的配置就这么一个，也是前文说信号量隔离配置不灵活的原因。

**统计器(Metrics)**

**`滑动窗口`**： Hystrix  的统计器是由滑动窗口来实现的，我们可以这么来理解滑动窗口：一位乘客坐在正在行驶的列车的靠窗座位上，列车行驶的公路两侧种着一排挺拔的白杨树，随着列车的前进，路边的白杨树迅速从窗口滑过，我们用每棵树来代表一个请求，用列车的行驶代表时间的流逝，那么，列车上的这个窗口就是一个典型的滑动窗口，这个乘客能通过窗口看到的白杨树就是 Hystrix 要统计的数据。

**`桶`**： bucket 是 Hystrix  统计滑动窗口数据时的最小单位。同样类比列车窗口，在列车速度非常快时，如果每掠过一棵树就统计一次窗口内树的数据，显然开销非常大，如果乘客将窗口分成十分，列车前进行时每掠过窗口的十分之一就统计一次数据，开销就完全可以接受了。 Hystrix 的 bucket （桶）也就是窗口 N分之一 的概念。

- **metrics.rollingStats.timeInMilliseconds**：此配置项指定了窗口的大小，单位是 ms，默认值是 1000，即一个滑动窗口默认统计的是 1s 内的请求数据。
- **metrics.healthSnapshot.intervalInMilliseconds**：它指定了健康数据统计器（影响 Hystrix 熔断）中每个桶的大小，默认是 500ms，在进行统计时，Hystrix 通过 `metrics.rollingStats.timeInMilliseconds / metrics.healthSnapshot.intervalInMilliseconds` 计算出桶数，在窗口滑动时，每滑过一个桶的时间间隔时就统计一次当前窗口内请求的失败率。
- **metrics.rollingStats.numBuckets**：Hystrix 会将命令执行的结果类型都统计汇总到一块，给上层应用使用或生成统计图表，此配置项即指定了，生成统计数据流时滑动窗口应该拆分的桶数。此配置项最易跟上面的 `metrics.healthSnapshot.intervalInMilliseconds` 搞混，认为此项影响健康数据流的桶数。 此项默认是 10，并且需要保持此值能被 `metrics.rollingStats.timeInMilliseconds` 整除。
- **metrics.rollingPercentile.enabled**：是否统计方法响应时间百分比，默认为 true 时，Hystrix 会统计方法执行的 `1%,10%,50%,90%,99%` 等比例请求的平均耗时用以生成统计图表。
- **metrics.rollingPercentile.timeInMilliseconds**：统计响应时间百分比时的窗口大小，默认为 60000，即一分钟。
- **metrics.rollingPercentile.numBuckets**：统计响应时间百分比时滑动窗口要划分的桶用，默认为6，需要保持能被`metrics.rollingPercentile.timeInMilliseconds` 整除。
- **metrics.rollingPercentile.bucketSize**：统计响应时间百分比时，每个滑动窗口的桶内要保留的请求数，桶内的请求超出这个值后，会覆盖最前面保存的数据。默认值为 100，在统计响应百分比配置全为默认的情况下，每个桶的时间长度为 10s = 60000ms / 6，但这 10s 内只保留最近的 100  条请求的数据。

**熔断器(Circuit Breaker)**

- **circuitBreaker.enabled**：是否启用熔断器，默认为 true;
- **circuitBreaker.forceOpen**： **circuitBreaker.forceClosed**：是否强制启用/关闭熔断器，强制启用关闭都想不到什么应用的场景，保持默认值，不配置即可。
- **circuitBreaker.requestVolumeThreshold**：启用熔断器功能窗口时间内的最小请求数。试想如果没有这么一个限制，我们配置了 50% 的请求失败会打开熔断器，窗口时间内只有 3 条请求，恰巧两条都失败了，那么熔断器就被打开了，5s  内的请求都被快速失败。此配置项的值需要根据接口的 QPS  进行计算，值太小会有误打开熔断器的可能，值太大超出了时间窗口内的总请求数，则熔断永远也不会被触发。建议设置为 `QPS * 窗口秒数 * 60%`。
- **circuitBreaker.errorThresholdPercentage**：在通过滑动窗口获取到当前时间段内 Hystrix 方法执行的失败率后，就需要根据此配置来判断是否要将熔断器打开了。 此配置项默认值是 50，即窗口时间内超过 50% 的请求失败后会打开熔断器将后续请求快速失败。
- **circuitBreaker.sleepWindowInMilliseconds**：熔断器打开后，所有的请求都会快速失败，但何时服务恢复正常就是下一个要面对的问题。熔断器打开时，Hystrix  会在经过一段时间后就放行一条请求，如果这条请求执行成功了，说明此时服务很可能已经恢复了正常，那么会将熔断器关闭，如果此请求执行失败，则认为服务依然不可用，熔断器继续保持打开状态。此配置项指定了熔断器打开后经过多长时间允许一次请求尝试执行，默认值是 5000。

**其他(Context/Fallback)**

- **requestCache.enabled**：是否启用请求结果缓存。默认是 true，但它并不意味着我们的每个请求都会被缓存。缓存请求结果和从缓存中获取结果都需要我们配置 `cacheKey`，并且在方法上使用 `@CacheResult` 注解声明一个缓存上下文。
- **requestLog.enabled**：是否启用请求日志，默认为 true。
- **fallback.enabled**：是否启用方法回退，默认为 true 即可。
- **fallback.isolation.semaphore.maxConcurrentRequests**：回退方法执行时的最大并发数，默认是10，如果大量请求的回退方法被执行时，超出此并发数的请求会抛出 `REJECTED_SEMAPHORE_FALLBACK` 异常。

## 2.3. threadPoolProperties

线程池的配置也是由 HystrixProperty 数组构成，配置方式与命令属性一致。

**配置项**

- **coreSize**：核心线程池的大小，默认值是 10，一般根据 `QPS * 99% cost + redundancy count` 计算得出。
- **allowMaximumSizeToDivergeFromCoreSize**：是否允许线程池扩展到最大线程池数量，默认为 false;
- **maximumSize**：线程池中线程的最大数量，默认值是 10，此配置项单独配置时并不会生效，需要启用 `allowMaximumSizeToDivergeFromCoreSize` 项。
- **maxQueueSize**：作业队列的最大值，默认值为 -1，设置为此值时，队列会使用 `SynchronousQueue`，此时其 size 为0，Hystrix 不会向队列内存放作业。如果此值设置为一个正的 int 型，队列会使用一个固定 size 的 `LinkedBlockingQueue`，此时在核心线程池内的线程都在忙碌时，会将作业暂时存放在此队列内，但超出此队列的请求依然会被拒绝。
- **queueSizeRejectionThreshold**：由于 `maxQueueSize` 值在线程池被创建后就固定了大小，如果需要动态修改队列长度的话可以设置此值，即使队列未满，队列内作业达到此值时同样会拒绝请求。此值默认是 5，所以有时候只设置了 `maxQueueSize` 也不会起作用。
- **keepAliveTimeMinutes**：由上面的 `maximumSize`，我们知道，线程池内核心线程数目都在忙碌，再有新的请求到达时，线程池容量可以被扩充为到最大数量，等到线程池空闲后，多于核心数量的线程还会被回收，此值指定了线程被回收前的存活时间，默认为 2，即两分钟。

# 3. 原理

@EnableCircuitBreaker 注解的作用就是告诉 Spring Cloud 在该服务中启用 Hystrix，其效果就相当于在应用程序中自动注入了熔断器。

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableCircuitBreakerImportSelector.class)
public @interface EnableCircuitBreaker {

}
```

这里通过 @Import 注解引入了 `EnableCircuitBreakerImportSelector` 类，该类定义如下

```
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableCircuitBreakerImportSelector
		extends SpringFactoryImportSelector<EnableCircuitBreaker> {

	@Override
	protected boolean isEnabled() {
		return getEnvironment().getProperty("spring.cloud.circuit.breaker.enabled",
				Boolean.class, Boolean.TRUE);
	}

}
```

它会加载 spring.factories 中标明为 EnableCircuitBreaker 的配置类

```
org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker=\
org.springframework.cloud.netflix.hystrix.HystrixCircuitBreakerConfiguration
```

HystrixCircuitBreakerConfiguration 配置类内部构造了一个切面，即 HystrixCommandAspect，用于拦截 HystrixCommand 的切面。这个切面会根据注解创建一个`HystrixInvokable`对象

```
MetaHolderFactory metaHolderFactory = META_HOLDER_FACTORY_MAP.get(HystrixPointcutType.of(method));
MetaHolder metaHolder = metaHolderFactory.create(joinPoint);
HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder);
```

`HystrixInvokable`实际上只是一个空接口。在 Hystrix中，存在一批以“-able”结尾的接口定义。例如，HystrixExecutable 和 HystrixObservable 接口就继承了 HystrixInvokable 接口，而这些接口最终都由各种以“-Command”结尾的类来负责实现。

在 `AbstractCommand` 类中初始化了 `HystrixCircuitBreaker` 接口的变量

```
private static HystrixCircuitBreaker initCircuitBreaker(boolean enabled, HystrixCircuitBreaker fromConstructor,
														HystrixCommandGroupKey groupKey, HystrixCommandKey commandKey,
														HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
	if (enabled) {
		if (fromConstructor == null) {
			// get the default implementation of HystrixCircuitBreaker
			return HystrixCircuitBreaker.Factory.getInstance(commandKey, groupKey, properties, metrics);
		} else {
			return fromConstructor;
		}
	} else {
		return new NoOpCircuitBreaker();
	}
}
```

这里的 enabled 标志位就是前面介绍的 @EnableCircuitBreaker 注解中，通过 EnableCircuitBreakerImportSelector 的 isEnabled() 方法获取的配置，相当于是一个控制是否启用熔断器的开关。如果该标志位为 true，则会创建一个 HystrixCircuitBreaker 实例，反之则返回一个什么都不做的 NoOpCircuitBreaker。

 HystrixCircuitBreaker 接口只有三个方法，它的实现类为 HystrixCircuitBreakerImpl，该实现类通过一个 Factory 工厂类进行创建。

```
public interface HystrixCircuitBreaker {

	//请求是否可被执行
	public boolean allowRequest();

	//返回当前熔断器是否打开
	public boolean isOpen();

	//关闭熔断器
	void markSuccess();

}
```

```
public static HystrixCircuitBreaker getInstance(HystrixCommandKey key, HystrixCommandGroupKey group, HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
	// this should find it for all but the first time
	HystrixCircuitBreaker previouslyCached = circuitBreakersByCommand.get(key.name());
	if (previouslyCached != null) {
		return previouslyCached;
	}

	// if we get here this is the first time so we need to initialize

	// Create and add to the map ... use putIfAbsent to atomically handle the possible race-condition of
	// 2 threads hitting this point at the same time and let ConcurrentHashMap provide us our thread-safety
	// If 2 threads hit here only one will get added and the other will get a non-null response instead.
	HystrixCircuitBreaker cbForCommand = circuitBreakersByCommand.putIfAbsent(key.name(), new HystrixCircuitBreakerImpl(key, group, properties, metrics));
	if (cbForCommand == null) {
		// this means the putIfAbsent step just created a new one so let's retrieve and return it
		return circuitBreakersByCommand.get(key.name());
	} else {
		// this means a race occurred and while attempting to 'put' another one got there before
		// and we instead retrieved it and will now return it
		return cbForCommand;
	}
}
```

这段代码很明显采用了基于 Key-Value 对的缓存设计思想，其中 Key 为 HystrixCommandKey 的 name，Value  为一个 HystrixCircuitBreakerImpl 实例。如果缓存中能够获取已有 Key 对应的  HystrixCircuitBreakerImpl 实例则直接返回，如果没有则创建一个新的实例并放入缓存。

- allowRequest()

是否已经熔断了，而该判断取决于服务访问的失败率。allowRequest 方法首先会判断熔断器是否被强制打开或关闭。如果是强制打开，则直接拒绝请求；如果为强制关闭，则会调用 isOpen() 方法来判断当前熔断器是否需要打开。

```
@Override
public boolean allowRequest() {
	if (properties.circuitBreakerForceOpen().get()) {
		// properties have asked us to force the circuit open so we will allow NO requests
		return false;
	}
	if (properties.circuitBreakerForceClosed().get()) {
		// we still want to allow isOpen() to perform it's calculations so we simulate normal behavior
		isOpen();
		// properties have asked us to ignore errors so we will ignore the results of isOpen and just allow all traffic through
		return true;
	}
	return !isOpen() || allowSingleTest();
}
```

- isOpen()

该方法返回当前熔断器是否打开的状态。如果熔断器为 Open 状态，则直接返回 true。如果不是，则会从度量指标中获取请求健康信息并根据熔断阈值判断熔断结果，

```
    public boolean isOpen() {
                if (circuitOpen.get()) {
                    return true;
                }
     
                HealthCounts health = metrics.getHealthCounts();
     
    	        // 检查是否达到最小请求数,如果未达到的话即使请求全部失败也不会熔断
                if (health.getTotalRequests() < properties.circuitBreakerRequestVolumeThreshold().get()) {
                    return false;
                }
     
    	        // 检查错误百分比是否达到设定的阈值，如果未达到的话也不会熔断
                if (health.getErrorPercentage() < properties.circuitBreakerErrorThresholdPercentage().get()) {
                    return false;
                } else {
                    // 如果错误率过高，则进行熔断，并记录下熔断时间
                    if (circuitOpen.compareAndSet(false, true)) {
                        circuitOpenedOrLastTestedTime.set(System.currentTimeMillis());
                        return true;
                    } else {
                        return true;
                    }
                }
    }

```

- markSuccess()

HystrixCircuitBreaker 中的最后一个 markSuccess 方法用于关闭熔断器。在 HystrixCmmand 执行成功的情况下，通过调用该方法可以将打开的熔断器关闭，并重置度量指标对象

```
public void markSuccess() {
	if (circuitOpen.get()) {
		if (circuitOpen.compareAndSet(true, false)) {
			//win the thread race to reset metrics
			//Unsubscribe from the current stream to reset the health counts stream.  This only affects the health counts view,
			//and all other metric consumers are unaffected by the reset
			metrics.resetStream();
		}
	}
}
```

HystrixCircuitBreaker 通过一个 circuitOpen 状态位控制着整个熔断判断流程，而这个状态位本身的状态值则取决于系统目前的执行数据和健康指标。。我们在这三个方法中看到的 HealthCounts 和 HystrixCommandMetrics  都是这些指标的具体体现。而针对这些指标的采集和处理过程，Hystrix  提供了一套值得我们学习和借鉴的设计思想和实现机制，这就是滑动窗口（Rolling Window）机制。

# 4. 参考资料

《Spring Cloud 原理与实战 》

