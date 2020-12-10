---
layout: post
title: Spring Cloud Eureka - Server原理
date: 2020-07-26
categories:
    - Spring
comments: true
permalink: spring-cloud-eureka-server.html
---

# 1. 核心概念

![](/assets/images/posts/eureka/eureka-3.png)

在上图中，Eureka 有以下几个概念与服务治理直接相关，首当其冲的是服务注册。服务注册（Register）是服务治理的最基本概念，内嵌了 Eureka 客户端的各个微服务通过向 Eureka 服务器提供 IP 地址、端点等各项与服务发现相关的基本信息完成服务注册操作。

因为 Eureka 客户端与服务器端通过短连接完成交互，所以在服务续约（Renew）中，Eureka 客户端需要每隔一定时间主动上报自己的运行时状态，从而进行服务续约。

服务取消（Cancel）的意思就是 Eureka 客户端主动告知 Eureka 服务器自己不想再注册到 Eureka 中。当Eureka客户端连续一段时间没有向 Eureka 服务器发送服务续约信息时，Eureka 服务器就会认为该服务实例已经不再运行，从而将其从服务列表中进行剔除（Evict）。

> 现在我们假设一套大型的分布式系统，一共100个服务，每个服务部署在20台机器上，也就是说，相当于你一共部署了100 * 20 = 2000个服务实例，有2000台机器。
>
> 每台机器上的服务实例内部都有一个Eureka Client组件，它会每隔30秒请求一次Eureka Server，拉取变化的注册表。此外，每个服务实例上的Eureka Client都会每隔30秒发送一次心跳请求给Eureka Server。
>
> 那么Eureka Server作为一个微服务注册中心，**每秒钟要被请求多少次？一天要被请求多少次？**
>
> - 按标准的算法，每个服务实例每分钟请求2次拉取注册表，每分钟请求2次发送心跳
> - 这样一个服务实例每分钟会请求4次，2000个服务实例每分钟请求8000次
> - 换算到每秒，则是8000 / 60 = 133次左右，我们就大概估算为Eureka Server每秒会被请求150次
> - 那一天的话，就是8000 * 60 * 24 = 1152万，也就是**每天千万级访问量**
>
> 所以通过设置一个适当的拉取注册表以及发送心跳的频率，可以保证大规模系统里对Eureka Server的请求压力不会太大。

# 2. 服务存储

对于一个注册中心而言，我们首先需要关注它的数据存储方法。在 Eureka 中是由 `InstanceRegistry` 接口及其实现承接了这部分职能

在`AbstractInstanceRegistry`有一个**CocurrentHashMap**，就是注册表的核心结构

```
public abstract class AbstractInstanceRegistry implements InstanceRegistry {
	...
    private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
            = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
	...
}
```

可以看到`registry`是一个双层Map，其中第一层的 ConcurrentHashMap 的 Key 为 `spring.application.name，`也就是服务名，Value 为一个 ConcurrentHashMap；而第二层的 ConcurrentHashMap 的 Key 为 instanceId，也就是服务的唯一实例 ID，Value 为 Lease 对象。Eureka 采用 Lease（租约）这个词来表示对服务注册信息的抽象，Lease 对象保存了服务实例信息以及一些实例服务注册相关的时间，如注册时间 registrationTimestamp、最新的续约时间 lastUpdateTimestamp 等。

![](/assets/images/posts/eureka/eureka-4.png)

而对于 InstanceRegistry 本身，它也继承了 Eureka 中两个非常重要的接口，即LeaseManager 接口和 LookupService 接口。

```
public interface LeaseManager<T> {

    void register(T r, int leaseDuration, boolean isReplication);

    boolean cancel(String appName, String id, boolean isReplication);

    boolean renew(String appName, String id, boolean isReplication);

    void evict();
}
```

显然 LeaseManager 做的事情就是 Eureka 注册中心模型中的服务注册、服务续约、服务取消和服务剔除等核心操作，关注对服务注册过程的管理。

而 LookupService 接口关注对应用程序与服务实例的管理：

```
public interface LookupService<T> {

    Application getApplication(String appName);

    Applications getApplications();

    List<InstanceInfo> getInstancesById(String id);

    InstanceInfo getNextServerFromEureka(String virtualHostname, boolean secure);

}

```

# 3. 服务缓存

Eureka Server为了避免同时读写内存数据结构造成的并发冲突问题，还采用了**多级缓存机制**来进一步提升服务请求的响应速度。

我们知道为了获取注册到 Eureka 服务器上具体某一个服务实例的详细信息，可以访问如下地址

```
http://<eureka-server-ip>:8761/eureka/apps/<APPID>
```

Eureka 中所有对服务器端的访问都是通过**RESTful 风格**的**资源（Resource）** 进行获取，ApplicationResource 类提供了根据应用获取注册信息的入口。查看getApplication方法，可以看到Eureka是从一个缓存中读取的服务信息

```
Key cacheKey = new Key(
		Key.EntityType.Application,
		appName,
		keyType,
		CurrentRequestVersion.get(),
		EurekaAccept.fromString(eurekaAccept)
);

String payLoad = responseCache.get(cacheKey);
```

在来看 `ResponseCache` 的定义，在它的内部又有两个Map

```
private final ConcurrentMap<Key, Value> readOnlyCacheMap = new ConcurrentHashMap<Key, Value>();

private final LoadingCache<Key, Value> readWriteCacheMap;

Value getValue(final Key key, boolean useReadOnlyCache) {
	Value payload = null;
	try {
		if (useReadOnlyCache) {
			final Value currentPayload = readOnlyCacheMap.get(key);
			if (currentPayload != null) {
				payload = currentPayload;
			} else {
				payload = readWriteCacheMap.get(key);
				readOnlyCacheMap.put(key, payload);
			}
		} else {
			payload = readWriteCacheMap.get(key);
		}
	} catch (Throwable t) {
		logger.error("Cannot get value for key : {}", key, t);
	}
	return payload;
}
```

其中 readOnlyCacheMap 就是一个 JDK 中的 ConcurrentMap，而 readWriteCacheMap 使用的则是  Google Guava Cache 库中的 LoadingCache 类型。在创建 LoadingCache过程中，缓存数据的来源是调用  generatePayload 方法来生成。而在这个 generatePayload 方法中，就会调用 AbstractInstanceRegistry 中的 getApplications  方法获取应用信息并放到缓存中。这样我们就实现了把注册信息与缓存信息进行关联。

这里有一个设计和实现上的技巧。把缓存设计为一个只读的 readOnlyCacheMap 以及一个可读写的  readWriteCacheMap，可以更好地分离职责。但因为两个缓存中保存的实际上是同一份数据，所以，我们在不断更新  readWriteCacheMap 时，也需要确保 readOnlyCacheMap 中的数据得到同步。为此 ResponseCacheImpl 提供了一个定时任务 CacheUpdateTask从 readWriteCacheMap 更新数据到 readOnlyCacheMap。

而在通过`invalidate`方法清除缓存时，仅清除readWriteCacheMap 中的数据，CacheUpdateTask会定时清除readOnlyCacheMap中的数据

```
Value cacheValue = readWriteCacheMap.get(key);
Value currentCacheValue = readOnlyCacheMap.get(key);
if (cacheValue != currentCacheValue) {
	readOnlyCacheMap.put(key, cacheValue);
}
```

简单总结一下多级缓存的流程

- 在拉取注册表的时候：

- - 首先从**ReadOnlyCacheMap**里查缓存的注册表。

- - 若没有，就找**ReadWriteCacheMap**里缓存的注册表。

- - 如果还没有，就从**内存中获取实际的注册表数据。**

- 在注册表发生变更的时候：

- - 会在内存中更新变更的注册表数据，同时**过期掉ReadWriteCacheMap**，此过程不会影响ReadOnlyCacheMap提供的查询注册表。

- - Eureka Server的后台线程发现ReadWriteCacheMap已经清空了，也会清空ReadOnlyCacheMap中的缓存

- - 下次有服务拉取注册表，又会从内存中获取最新的数据了，同时填充各个缓存。

# 4. Peer Awareness 模式

Eureka 的高可用部署方式由PeerAwareInstanceRegistry实现，在它的register方法中我们可以看到调用了一个replicateToPeers方法

```
public void register(final InstanceInfo info, final boolean isReplication) {
	int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
	if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
		leaseDuration = info.getLeaseInfo().getDurationInSecs();
	}
	super.register(info, leaseDuration, isReplication);
	replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
```

replicateToPeers 方法就是用来实现服务器节点之间的状态同步，它会遍历Eureka节点（排除掉自己）进行复制

```
for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
	// If the url represents this host, do not replicate to yourself.
	if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
		continue;
	}
	replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
}
```

replicateInstanceActionsToPeers方法根据不同的action 来调用 PeerEurekaNode 的不同方法

```
public void register(final InstanceInfo info) throws Exception {
	long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
	batchingDispatcher.process(
			taskId("register", info),
			new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
				public EurekaHttpResponse<Void> execute() {
					return replicationClient.register(info);
				}
			},
			expiryTime
	);
}
```

而PeerEurekaNode最终又是通过HttpReplicationClient来完成节点之间的通信。

# 5. 参考资料

《Spring Cloud 原理与实战》