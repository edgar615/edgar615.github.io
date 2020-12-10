---
layout: post
title: Spring Cloud Eureka - Client原理
date: 2020-07-27
categories:
    - Spring
comments: true
permalink: spring-cloud-eureka-client.html
---

对于 Eureka 而言，微服务的提供者和消费者都是它的客户端，其中服务提供者关注**服务注册、服务续约**和**服务下线**等功能，而服务消费者关注于服务信息的获取。同时，对于服务消费者而言，为了提高服务获取的性能以及在注册中心不可用的情况下继续使用服务，一般都还会具有缓存机制。

在 Netflix Eureka 中，专门提供了一个客户端包，并抽象了一个客户端接口 EurekaClient。EurekaClient 接口继承自 LookupService 接口。

# 1. DiscoveryClient

DiscoveryClient是EurekaClient 接口的实现类 DiscoveryClient，该类包含了服务提供者和服务消费者的核心处理逻辑，同时提供了我们在介绍 Eureka 服务器端基本原理时所介绍的  register、renew 等方法。并且定义了内部类：

- DiscoverClientOptionalArgs，可选参数类，源码里实现为空，是默认实现
- EurekaTransport类，封装了Client请求的类；
- CacheFreshThread，刷新缓存线程，提供定时拉取服务列表等；
- HeartbeatThread，心跳线程，提供定时向服务端续约服务等。

DiscoveryClient的构造方法非常复杂，除了初始化一些参数外，它还会创建几个线程池

```
// default size of 2 - 1 each for heartbeat and cacheRefresh
// 定义了2个线程大小的定时线程池：一个是刷新缓存CacheFreshThread，一个是心跳线程HeartbeatThread
scheduler = Executors.newScheduledThreadPool(2,
		new ThreadFactoryBuilder()
				.setNameFormat("DiscoveryClient-%d")
				.setDaemon(true)
				.build());
// 心跳线程池
heartbeatExecutor = new ThreadPoolExecutor(
		1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
		new SynchronousQueue<Runnable>(),
		new ThreadFactoryBuilder()
				.setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
				.setDaemon(true)
				.build()
);  // use direct handoff
// 刷新缓存线程池
cacheRefreshExecutor = new ThreadPoolExecutor(
		1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
		new SynchronousQueue<Runnable>(),
		new ThreadFactoryBuilder()
				.setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
				.setDaemon(true)
				.build()
);  // use direct handoff
```

在初始化完后，构造方法里调用了`initScheduledTasks`方法来初始化定时任务

```
// finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
initScheduledTasks();
```

Eureka Client启动时会开启**三个定时任务**：

1. 刷新缓存定时服务，即定时拉取服务列表，默认每30秒进行定时拉取服务列表；
2. 开启心跳线程定时服务，即定时向服务端进行服务续约，默认每30秒进行定时续约。
3. 启动实例信息复制器进行刷新服务实例信息或服务注册请求。

在 DiscoveryClient 类中，服务注册操作由register 方法完成，该方法最终会由InstanceInfoReplicator的run方法完成

```
public void run() {
    try {
        discoveryClient.refreshInstanceInfo();

        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```

而DiscoveryClient 通过EurekaTransport发起了远程请求。

```
eurekaTransport.registrationClient.register(instanceInfo);
```

# 2. 客户端缓存

DiscoveryClient配备了缓存机制以加速服务查找

```
@Override
public Application getApplication(String appName) {
    return getApplications().getRegisteredApplications(appName);
}
```

在Applications类中可以看到最终使用了一个Map缓存

```
private final Map<String, Application> appNameApplicationMap;

public Application getRegisteredApplications(String appName) {
    return appNameApplicationMap.get(appName.toUpperCase(Locale.ROOT));
}
```

我们已经在前面的内容中提到 DiscoveryClient 中的 initScheduledTasks 方法会初始化一个刷新缓存定时任务

```
cacheRefreshTask = new TimedSupervisorTask(
		"cacheRefresh",
		scheduler,
		cacheRefreshExecutor,
		registryFetchIntervalSeconds,
		TimeUnit.SECONDS,
		expBackOffBound,
		new CacheRefreshThread()
);
scheduler.schedule(
		cacheRefreshTask,
		registryFetchIntervalSeconds, TimeUnit.SECONDS);
```

可以看到，DiscoveryClient启动了一个CacheRefreshThread线程用于刷新缓存

```
class CacheRefreshThread implements Runnable {
    public void run() {
        refreshRegistry();
    }
}
```

而CacheRefreshThread最终通过fetchRegistry完成注册信息的更新

```
boolean success = fetchRegistry(remoteRegionsModified);

private boolean fetchRegistry(boolean forceFullRegistryFetch) {

	try {
		// 获取应用
		Applications applications = getApplications();
		if (…) //如果满足全量拉取条件
		{
		  // 全量拉取服务实例数据
			getAndStoreFullRegistry();
		} else {
		  // 增量拉取服务实例数据
			getAndUpdateDelta(applications);
		} 
	   // 重新计算和设置一致性hashcode
		applications.setAppsHashCode(applications.getReconcileHashCode());

    } 

	// 刷新本地缓存
	onCacheRefreshed();
	// 更新远程服务实例运行状态
	updateInstanceRemoteStatus();
	return true;

}
```

回顾 Eureka 服务器端基本原理，我们知道 Eureka  服务器端会保存一个服务注册列表的缓存。Eureka 官方文档中提到这个数据保留时间是三分钟，而 Eureka 客户端的定时调度机制会每隔 30  秒刷选本地缓存。原则上，只要 Eureka 客户端不停地获取服务器端的更新数据，就能保证自己的数据和 Eureka  服务器端的保持一致。但如果客户端在 3  分钟之内没有获取更新数据，就会导致自身与服务器端的数据不一致，这是这种更新机制所必须要考虑的问题，也是我们自己在设计类似场景时的一个注意点。

针对上述问题，Eureka 采用了一致性 HashCode 方法来进行解决。Eureka  服务器端每次返回的增量数据中都会带有一个一致性 HashCode，这个 HashCode 会与 Eureka  客户端用本地服务列表数据算出的一致性 HashCode 进行比对，如果两者不一致就证明增量更新出了问题，这时候就需要执行一次全量更新。

计算一致性 HashCode 的方法如下所示

```
public static String getReconcileHashCode(Map<String, AtomicInteger> instanceCountMap) {
    StringBuilder reconcileHashCode = new StringBuilder(75);
    for (Map.Entry<String, AtomicInteger> mapEntry : instanceCountMap.entrySet()) {
        reconcileHashCode.append(mapEntry.getKey()).append(STATUS_DELIMITER).append(mapEntry.getValue().get())
                .append(STATUS_DELIMITER);
    }
    return reconcileHashCode.toString();
}
```