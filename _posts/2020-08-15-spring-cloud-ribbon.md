---
layout: post
title: Spring Cloud Ribbon
date: 2020-08-15
categories:
    - Spring
comments: true
permalink: spring-cloud-ribbon.html
---

> netflix的文档太少，单独学习Ribbon很困难，直接基于Spring Cloud Netflix Ribbon提供的封装学习

Ribbon是一款用于提供**客户端负载均衡**的工具软件。Ribbon 会自动地基于某种内置的负载均衡算法去连接服务实例，我们也可以设计并实现自定义的负载均衡算法并嵌入 Ribbon 中。同时，Ribbon 客户端组件提供了一系列完善的辅助机制用来确保服务调用过程的可靠性和容错性，包括连接超时和重试等。Ribbon 是客户端负载均衡机制的典型实现方案，所以需要嵌入在服务消费者的内部进行使用。

- **使用 `@LoadBalanced` 注解**

`@LoadBalanced` 注解用于修饰发起 HTTP 请求的 RestTemplate 工具类，并在该工具类中自动嵌入客户端负载均衡功能。开发人员不需要针对负载均衡做任何特殊的开发或配置。

- **使用 `@RibbonClient` 注解**

Ribbon 还允许你使用 `@RibbonClient `注解来完全控制客户端负载均衡行为。这在需要定制化负载均衡算法等某些特定场景下非常有用，我们可以使用这个功能实现更细粒度的负载均衡配置。

# 1. DiscoveryClient 

假如现在没有 Ribbon 这样的负载均衡工具，我们也可以通过`DiscoveryClient`在运行时实时获取注册中心中的服务列表，并通过服务定义并结合各种负载均衡策略动态发起服务调用。

我们获取当前注册到 Eureka 中的服务名称全量列表

```
List<String> serviceNames = discoveryClient.getServices();
```

获取摸一个服务的实例信息

```
List<ServiceInstance> serviceInstances = discoveryClient.getInstances(serviceName);
```

一旦获取了一个 ServiceInstance 列表，我们就可以基于常见的随机、轮询等算法来实现客户端负载均衡，也可以基于服务的 URI 信息等实现各种定制化的路由机制。一旦确定负载均衡的最终目标服务，就可以使用 HTTP 工具类来根据服务的地址信息发起远程调用。

```
List<ServiceInstance> instances = discoveryClient.getInstances("eureka-client");

String userserviceUri = String.format("%s/api/accounts/%s",
		instances.get(0).getUri().toString(), 1);
ResponseEntity<Account> accountResponseEntity =
		restTemplate.exchange(userserviceUri, HttpMethod.GET, null, Account.class);
System.out.println(accountResponseEntity.getBody());
```

# 2. @Loadbalanced

在 Spring Cloud 中基于 Ribbon 来实现负载均衡非常简单，要做的事情就是在声明 `RestTemplate` 上添加一个注解。

```
@Bean
@LoadBalanced
public RestTemplate restTemplate(RestTemplateBuilder builder) {
	return builder.build();
}
```

此时注入 `RestTemplate`就具备了客户端负载均衡功能。

```
ResponseEntity<Account> accountResponseEntity =
                restTemplate.exchange("http://eureka-client/api/accounts/{id}", HttpMethod.GET, null, Account.class, 1);
```

# 3. @RibbonClient

在基于 `@LoadBalanced` 注解执行负载均衡时，采用的是 Ribbon 内置的负载均衡机制，我们完全没有感觉到 Ribbon 组件的存在。默认情况下，Ribbon 使用的是轮询策略，我们无法控制具体生效的是哪种负载均衡算法。但在有些场景下，我们就需要对负载均衡这一过程进行更加精细化的控制，这时候就可以用到 `@RibbonClient` 注解。Spring Cloud Netflix Ribbon 提供 `@RibbonClient `注解的目的在于通过该注解声明自定义配置，从而来完全控制客户端负载均衡行为。

为了使用 @RibbonClient 注解，我们需要创建一个独立的配置类，用来指定具体的负载均衡规则。

```
@Configuration
public class CustomLoadBalanceConfig {

    @Bean
    @ConditionalOnMissingBean
    public IRule accountRule() {
        return new RandomRule();
    }
}
```

该配置类的作用是使用 RandomRule 替换 Ribbon 中的默认负载均衡策略 RoundRobin。此时我们就可以在调用特定服务时使用该配置类，从而对客户端负载均衡实现细粒度的控制。

```
@SpringBootApplication
@EnableDiscoveryClient
@RibbonClient(name = "eureka-client", configuration = CustomLoadBalanceConfig.class)
public class Application {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.build();
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class);
    }
}
```

现在每次访问 eureka-client 时将使用 RandomRule 这一随机负载均衡策略。

我们也可以通过 `@RibbonClients` 为所有的RibbonClient提供默认的配置

```
@RibbonClients(defaultConfiguration = DefaultRibbonConfig.class)
public class RibbonClientDefaultConfigurationTestsConfig {

    public static class BazServiceList extends ConfigurationBasedServerList {

        public BazServiceList(IClientConfig config) {
            super.initWithNiwsConfig(config);
        }

    }

}

@Configuration(proxyBeanMethods = false)
class DefaultRibbonConfig {

    @Bean
    public IRule ribbonRule() {
        return new BestAvailableRule();
    }

    @Bean
    public IPing ribbonPing() {
        return new PingUrl();
    }

    @Bean
    public ServerList<Server> ribbonServerList(IClientConfig config) {
        return new RibbonClientDefaultConfigurationTestsConfig.BazServiceList(config);
    }

    @Bean
    public ServerListSubsetFilter serverListFilter() {
        ServerListSubsetFilter filter = new ServerListSubsetFilter();
        return filter;
    }

}
```



下表是Spring Cloud Netflix Ribbon默认提供的实现

| Bean Type                  | Bean Name                 | Class Name                       |
| -------------------------- | ------------------------- | -------------------------------- |
| `IClientConfig`            | `ribbonClientConfig`      | `DefaultClientConfigImpl`        |
| `IRule`                    | `ribbonRule`              | `ZoneAvoidanceRule`              |
| `IPing`                    | `ribbonPing`              | `DummyPing`                      |
| `ServerList<Server>`       | `ribbonServerList`        | `ConfigurationBasedServerList`   |
| `ServerListFilter<Server>` | `ribbonServerListFilter`  | `ZonePreferenceServerListFilter` |
| `ILoadBalancer`            | `ribbonLoadBalancer`      | `ZoneAwareLoadBalancer`          |
| `ServerListUpdater`        | `ribbonServerListUpdater` | `PollingServerListUpdater`       |

所以我们也可以直接替换默认实现

```
@Configuration(proxyBeanMethods = false)
public class FooConfiguration {

    @Bean
    public ZonePreferenceServerListFilter serverListFilter() {
        ZonePreferenceServerListFilter filter = new ZonePreferenceServerListFilter();
        filter.setZone("myTestZone");
        return filter;
    }

    @Bean
    public IPing ribbonPing() {
        return new PingUrl();
    }

    @Bean
    public IRule ribbonRule() {
        return new RandomRule();
    }
}
```



# 4. Netflix Ribbon 基本架构

作为一款客户端负载均衡工具，要做的事情无非就是两件：第一件事情是获取注册中心中的服务器列表；第二件事情是在这个服务列表中选择一个服务进行调用。针对这两个问题，Netflix Ribbon 提供了自身的一套基本架构，并抽象了一批核心类。

Netflix Ribbon 的核心接口 `ILoadBalancer` 就是围绕着上述两个问题来设计的

```
public interface ILoadBalancer {

	//添加后端服务
	public void addServers(List<Server> newServers);

	//选择一个后端服务
	public Server chooseServer(Object key); 

	//标记一个服务不可用
	public void markServerDown(Server server);

	//获取当前可用的服务列表
	public List<Server> getReachableServers();

	//获取所有后端服务列表
   	public List<Server> getAllServers();

}
```

接下来我们看一下具体实现类`BaseLoadBalancer`它包含的作为一个负载均衡器应该具备的一些核心组件

- **IRule** IRule 接口是对负载均衡策略的一种抽象，可以通过实现这个接口来提供各种适用的负载均衡算法
- **IPing** IPing 接口判断目标服务是否存活
- **LoadBalancerStats** LoadBalancerStats 类记录负载均衡的实时运行信息，用来作为负载均衡策略的运行时输入。

在 BaseLoadBalancer 中的 chooseServer 方法如下所示：

```
public Server chooseServer(Object key) {
	if (counter == null) {
		counter = createCounter();
	}
	counter.increment();
	if (rule == null) {
		return null;
	} else {
		try {
			return rule.choose(key);
		} catch (Exception e) {
			logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
			return null;
		}
	}
}
```

可以看到使用了 IRule 接口的 choose 方法。接下来就让我们看看 Ribbon 中的 IRule 接口为我们提供了具体哪些负载均衡算法。

`BaseLoadBalancer`中定义了一个`IPingStrategy`，用来描述服务检查策略，`IPingStrategy`默认实现采用了`SerialPingStrategy`实现，`SerialPingStrategy`中的`pingServers`方法就是遍历所有的服务实例，一个一个发送请求，查看这些服务实例是否还有效，如果网络环境不好的话，这种检查策略效率会很低，如果我们想自定义检查策略的话，可以重写`SerialPingStrategy`的`pingServers`方法。

## 4.1. DynamicServerListLoadBalancer

`DynamicServerListLoadBalancer`是`BaseLoadBalancer`的一个子类，在`DynamicServerListLoadBalancer`中对基础负载均衡器的功能做了进一步的扩展。

1. 首先`DynamicServerListLoadBalancer`类一开始就声明了一个变量serverListImpl，serverListImpl变量的类型是一个`ServerList<T extends Server>`，这里的泛型得是Server的子类，`ServerList`是一个接口，里边定义了两个方法：一个getInitialListOfServers用来获取初始化的服务实例清单；另一个getUpdatedListOfServers用于获取更新的服务实例清单。

2. `DynamicServerListLoadBalancer`中还定义了一个ServerListUpdater.UpdateAction类型的服务更新器，Spring  Cloud提供了两种服务更新策略：一种是`PollingServerListUpdater`，表示定时更新；另一种是`EurekaNotificationServerListUpdater`表示由Eureka的事件监听来驱动服务列表的更新操作，默认的实现策略是第一种，即定时更新，定时的方式很简单，创建Runnable，调用`DynamicServerListLoadBalancer`中updateAction对象的doUpdate方法，Runnable延迟启动时间为1秒，重复周期为30秒。

3. 在更新服务清单的时候，调用了我们在第一点提到的getUpdatedListOfServers方法，拿到实例清单之后，又调用了一个过滤器中的方法进行过滤。过滤器的类型有好几种，默认是`DefaultNIWSServerListFilter`，这是一个继承自`ZoneAffinityServerListFilter`的过滤器，具有区域感知功能。即它会对服务提供者所处的Zone和服务消费者所处的Zone进行比较，过滤掉哪些不是同一个区域的实例。

综上，`DynamicServerListLoadBalancer`主要是实现了服务实例清单在运行期间的动态更新能力，同时提供了对服务实例清单的过滤功能。

## 4.2. ZoneAwareLoadBalancer

`ZoneAwareLoadBalancer`是`DynamicServerListLoadBalancer`的子类，`ZoneAwareLoadBalancer`的出现主要是为了弥补`DynamicServerListLoadBalancer`的不足。由于`DynamicServerListLoadBalancer`中并没有重写chooseServer方法，所以`DynamicServerListLoadBalancer`中负责均衡的策略依然是我们在`BaseLoadBalancer`中分析出来的线性轮询策略，这种策略不具备区域感知功能，这样当需要跨区域调用时，可能会产生高延迟。`ZoneAwareLoadBalancer`重写了setServerListForZones方法，该方法在其父类中的功能主要是根据区域Zone分组的实例列表，为负载均衡器中的`LoadBalancerStats`对象创建ZoneStats并存入集合中，ZoneStats是一个用来存储每个Zone的状态和统计信息。重写之后的setServerListForZones方法主要做了两件事：一件是调用getLoadBalancer方法来创建负载均衡器，同时创建服务选择策略；另一件是对Zone区域中的实例清单进行检查，如果对应的Zone下已经没有实例了，则将Zone区域的实例列表清空，防止节点选择时出现异常。

# 5. Netflix Ribbon 中的负载均衡策略

一般而言，负载均衡算法可以分成两大类，即**静态负载均衡算法**和**动态负载均衡算法**。静态负载均衡算法比较容易理解和实现，典型的包括随机（Random）、轮询（Round Robin）和加权轮询（Weighted Round  Robin）算法等。所有涉及权重的静态算法都可以转变为动态算法，因为权重可以在运行过程中动态更新。例如动态轮询算法中权重值基于对各个服务器的持续监控并不断更新。另外，基于服务器的实时性能分析分配连接是常见的动态策略。典型动态算法包括源 IP 哈希算法、最少连接数算法、服务调用时延算法等。

Netflix Ribbon 中的负载均衡实现策略非常丰富，既提供了 RandomRule、RoundRobinRule  等无状态的静态策略，又实现了 AvailabilityFilteringRule、WeightedResponseTimeRule  等多种基于服务器运行状况进行实时路由的动态策略

- **BestAvailableRule**

选择一个并发请求量最小的服务器，逐个考察服务器然后选择其中活跃请求数最小的服务器。

- **WeightedResponseTimeRule** 

该策略与请求的响应时间有关，显然，如果响应时间越长，就代表这个服务的响应能力越有限，那么分配给该服务的权重就应该越小。WeightedResponseTimeRule 会定时从  LoadBalancerStats  读取平均响应时间，为每个服务更新权重。权重的计算也比较简单，即每次请求的平均响应时间减去每个服务自己平均的响应时间就是该服务的权重。

- **AvailabilityFilteringRule** 

通过检查 LoadBalancerStats 中记录的各个服务器的运行状态，过滤掉那些处于一直连接失败或处于高并发状态的后端服务器。

# 6. @LoadBalanced原理

为什么通过` @LoadBalanced `注解创建的 RestTemplate 就能自动具备客户端负载均衡的能力？

在 Spring Cloud Netflix Ribbon 中存在一个自动配置类`LoadBalancerAutoConfiguration`  类。而在该类中，维护着一个被 `@LoadBalanced` 修饰的 RestTemplate 对象的列表。在初始化的过程中，对于所有被  `@LoadBalanced` 注解修饰的 RestTemplate，调用 `RestTemplateCustomizer` 的 `customize`  方法进行定制化，该定制化的过程就是对目标 RestTemplate 增加拦截器 `LoadBalancerInterceptor`

```
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
static class LoadBalancerInterceptorConfig {

	@Bean
	public LoadBalancerInterceptor ribbonInterceptor(
			LoadBalancerClient loadBalancerClient,
			LoadBalancerRequestFactory requestFactory) {
		return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
	}

	@Bean
	@ConditionalOnMissingBean
	public RestTemplateCustomizer restTemplateCustomizer(
			final LoadBalancerInterceptor loadBalancerInterceptor) {
		return restTemplate -> {
			List<ClientHttpRequestInterceptor> list = new ArrayList<>(
					restTemplate.getInterceptors());
			list.add(loadBalancerInterceptor);
			restTemplate.setInterceptors(list);
		};
	}

}
```

这个 LoadBalancerInterceptor 用于实时拦截，而在它的拦截方法本质上就是使用 LoadBalanceClient  来执行真正的负载均衡。LoadBalancerInterceptor 类代码如下所示：

```
@Override
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
		final ClientHttpRequestExecution execution) throws IOException {
	final URI originalUri = request.getURI();
	String serviceName = originalUri.getHost();
	Assert.state(serviceName != null,
			"Request URI does not contain a valid hostname: " + originalUri);
	return this.loadBalancer.execute(serviceName,
			this.requestFactory.createRequest(request, body, execution));
}
```

**LoadBalanceClient 接口**

LoadBalancerClient 是一个非常重要的接口

```
public interface LoadBalancerClient extends ServiceInstanceChooser {

    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

    <T> T execute(String serviceId, ServiceInstance serviceInstance,
           LoadBalancerRequest<T> request) throws IOException;

    URI reconstructURI(ServiceInstance instance, URI original);

}

```

这里有两个 execute 重载方法，用于根据负载均衡器所确定的服务实例来执行服务调用。而 reconstructURI 方法则用于构建服务  URI，使用负载均衡所选择的 ServiceInstance 信息重新构造访问 URI，也就是用服务实例的 host 和 port  再加上服务的端点路径来构造一个真正可供访问的服务。

`LoadBalancerClient`继承了`ServiceInstanceChooser`，它有一个`choose`方法用来实现负载均衡

```
public interface ServiceInstanceChooser {

	ServiceInstance choose(String serviceId);

}
```

在 LoadBalancerClient 接口的实现类 RibbonLoadBalancerClient 中，choose 方法最终调用了Netflix Ribbon 中的 ILoadBalancer 接口的实现类。

# 7. 配置

```
# 每台服务器最多重试次数，但是首次调用不包括在内 Max number of retries on the same server (excluding the first try)
hello-client.ribbon.MaxAutoRetries=1
# 最多重试多少台服务器 Max number of next servers to retry (excluding the first server)
hello-client.ribbon.MaxAutoRetriesNextServer=1
# 无论是请求超时或者socket read timeout都进行重试 Whether all operations can be retried for this client
hello-client.ribbon.OkToRetryOnAllOperations=true
# Interval to refresh the server list from the source
hello-client.ribbon.ServerListRefreshInterval=2000
# Connect timeout used by Apache HttpClient
hello-client.ribbon.ConnectTimeout=3000
# Readtimeout used by Apache HttpClient
hello-client.ribbon.ReadTimeout=3000
# Initial list of servers, can be changed via Archaius dynamic property at runtime
#ribbon.listOfServers=localhost:8765
hello-client.ribbon.listOfServers=localhost:8765,localhost:8766
hello-client.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RoundRobinRule
```



# 8. 参考资料

《Spring Cloud 原理与实战 》

