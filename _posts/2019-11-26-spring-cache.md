---
layout: post
title: spring - cache原理
date: 2019-11-26
categories:
    - Spring
comments: true
permalink: spring-cache.html
---

最近为了在spring cache的基础上扩展更多的功能，简单看了一下spring的cache实现，这里记录一下

首先我们通过`@EnableCaching`注解会引入一个Selector：`CachingConfigurationSelector`，通过ImportSelector引入了两个配置类

- AutoProxyRegistrar
- ProxyCachingConfiguration

`AutoProxyRegistrar`又通过`registerBeanDefinitions`方法注册了一个`InfrastructureAdvisorAutoProxyCreator`对象

```
if (mode == AdviceMode.PROXY) {
	AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
	if ((Boolean) proxyTargetClass) {
		AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
		return;
	}
}
```

`InfrastructureAdvisorAutoProxyCreator`继承自`AbstractAdvisorAutoProxyCreator`，`AbstractAdvisorAutoProxyCreator`用来启动了spring的AOP机制（具体没深入）

`ProxyCachingConfiguration`用来初始化cache相关类

- BeanFactoryCacheOperationSourceAdvisor：一个PointcutAdvisor，该advisor会织入到需要执行缓存操作的bean的增强代理中形成一个切面。并在方法调用时在该切面上执行拦截器CacheInterceptor的业务逻辑。
- AnnotationCacheOperationSource：获取定义在类和方法上的SpringCache相关的注解并将其转换为对应的CacheOperation属性 
- CacheInterceptor：一个拦截器，当方法调用时碰到了BeanFactoryCacheOperationSourceAdvisor定义的切面，就会执行CacheInterceptor的业务逻辑，该业务逻辑就是缓存的核心业务逻辑。



`AnnotationCacheOperationSource`内部初始化了一个`SpringCacheAnnotationParser`用于找出使用了cache相关主键的类和方法（通过`parseCacheAnnotations`方法）然后初始化为不同的`CacheOperation`



`CacheInterceptor`继承了`CacheAspectSupport`并实现了`MethodInterceptor`接口，因此它本质上是一个Advice也就是可在切面上执行的增强逻辑

`invoke`里面调用的`return execute(aopAllianceInvoker, invocation.getThis(), method, invocation.getArguments());`封装了缓存的核心逻辑，它又通过调用下面的方法来最终执行缓存的逻辑

```
execute(invoker, method,
  new CacheOperationContexts(operations, method, args, target, targetClass))
```

首先判断是否需要在方法调用前执行缓存清除

```
processCacheEvicts(contexts.get(CacheEvictOperation.class), true,
  CacheOperationExpressionEvaluator.NO_RESULT);
```

然后检查是否能得到一个符合条件的缓存值

```
Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));
```

如果Cacheable miss，就会创建一个对应的CachePutRequest并收集起来

```
List<CachePutRequest> cachePutRequests = new LinkedList<>();
if (cacheHit == null) {
 collectPutRequests(contexts.get(CacheableOperation.class),
   CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
}
```

如果得到了缓存，`cachePutRequests`为空并且不含有符合条件（condition match）的`@CachePut`注释，那么就将returnValue赋值为缓存值；否则实际执行方法，并将returnValue赋值为方法返回值

```
Object cacheValue;
Object returnValue;

if (cacheHit != null && cachePutRequests.isEmpty() && !hasCachePut(contexts)) {
 // If there are no put requests, just use the cache hit
 cacheValue = cacheHit.get();
 returnValue = wrapCacheValue(method, cacheValue);
}
else {
 // Invoke the method if we don't have a cache hit
 returnValue = invokeOperation(invoker);
 cacheValue = unwrapReturnValue(returnValue);
}
```

收集符合条件的`@CachePut`定义的`CachePutRequest`，并添加到上面的`cachePutRequests`中：

```
collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);
```

执行CachePutRequest将符合条件的数据写入缓存

```
for (CachePutRequest cachePutRequest : cachePutRequests) {
 cachePutRequest.apply(cacheValue);
}
```

最后判断是否需要在方法调用后执行缓存清除

```
processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);
```