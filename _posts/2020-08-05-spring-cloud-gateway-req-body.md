---
layout: post
title: Spring Cloud Gateway - 读取Request Body
date: 2020-08-05
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-req-body.html
---

我们使用SpringCloud Gateway做微服务网关的时候，经常需要在过滤器Filter中读取到Post请求中的Body内容进行日志记录、参数教育、签名验证、权限验证等操作。但是Request的Body是只能读取一次的，如果直接通过在Filter中读取，而不封装回去回导致后面的服务无法读取数据。
SpringCloud Gateway 内部提供了一个断言工厂类ReadBodyPredicateFactory，这个类实现了读取Request的Body内容并放入缓存，我们可以通过从缓存中获取body内容来实现我们的目的。

查看ReadBodyPredicateFactory的实现，该工厂类将request body内容读取后存放在 exchange的cachedRequestBodyObject中。那么我们可以通过代码：`exchange.getAttribute("cachedRequestBodyObject");`将body内容取出来。


```java
return ServerWebExchangeUtils.cacheRequestBodyAndRequest(exchange,
		(serverHttpRequest) -> ServerRequest
				.create(exchange.mutate().request(serverHttpRequest)
						.build(), messageReaders)
				.bodyToMono(inClass)
				.doOnNext(objectValue -> exchange.getAttributes().put(
						CACHE_REQUEST_BODY_OBJECT_KEY, objectValue))
				.map(objectValue -> config.getPredicate()
						.test(objectValue)));
```

查看ReadBodyPredicateFactory的配置

```
private Class inClass;

private Predicate predicate;

private Map<String, Object> hints;
```

配置该工厂类需要两个参数:

- inClass：接收body内容的对象Class，我们用字符串接收，配置String即可。
- Predicate：Predicate的接口实现类，我们自定义一个Predicate的实现类即可。

**自定义Predicate**

```
@Component
public class ReadBodyPredicate implements Predicate {
    @Override
    public boolean test(Object o) {
        return true;
    }
}
```

在Route上增加断言

```
- id: body_read
  uri: http://httpbin.org:80/post
  predicates:
	- After=2020-04-20T00:00:00+08:00[Asia/Shanghai]
	- name: ReadBodyPredicateFactory #使用ReadBodyPredicateFactory断言，将body读入缓存
	  args:
		inClass: '#{T(String)}'
		predicate: '#{@reqBodyPredicate}' #注入实现predicate接口类
  filters:
	- BodyLog
```

在过滤器中读取请求体

```
@Override
public GatewayFilter apply(Object config) {

	return (exchange, chain) -> {
		Object requestBody = exchange.getAttribute("cachedRequestBodyObject");
		LOGGER.info("request body is:{}", requestBody);

		return chain.filter(exchange);
	};
}
```







