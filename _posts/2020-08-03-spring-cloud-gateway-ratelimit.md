---
layout: post
title: Spring Cloud Gateway（4）- 限流
date: 2020-08-03
categories:
    - Spring Cloud
comments: true
permalink: spring-cloud-gateway-ratelimit.html
---

Spring Cloud Gateway 已经内置了一个RequestRateLimiterGatewayFilterFactory，我们可以直接使用。

目前RequestRateLimiterGatewayFilterFactory的实现依赖于 Redis，所以我们还要引入spring-boot-starter-data-redis-reactive。否则RequestRateLimiterGatewayFilterFactory不会注入

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

配置限流过滤器

```
spring:
  cloud:
    gateway:
      routes:
        - id: limit_route
          uri: http://httpbin.org:80/get
          predicates:
            - After=2020-04-20T00:00:00+08:00[Asia/Shanghai]
          filters:
            - name: RequestRateLimiter
              args:
                key-resolver: '#{@hostAddrKeyResolver}'
                redis-rate-limiter.replenishRate: 1
                redis-rate-limiter.burstCapacity: 3
```

该过滤器需要配置三个参数：

- burstCapacity：令牌桶总容量。
- replenishRate：令牌桶每秒填充平均速率。
- key-resolver：用于限流的键的解析器的 Bean 对象的名字。它使用 SpEL 表达式根据#{@beanName}从 Spring 容器中获取 Bean 对象。

因此我们需要实现一个key-resolver

```
@Bean
public KeyResolver hostAddrKeyResolver() {
	return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
}
```

上面的KeyResolver是获取请求用户ip作为限流key。我们也可以实现其他的限流方式

**获取请求用户id作为限流key**

```
@Bean
public KeyResolver userKeyResolver() {
	return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("userId"));
}
```

**获取请求地址的uri作为限流key**

```
@Bean
KeyResolver apiKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getPath().value());
}
```

> 自己在测试的时候一直没起作用，后来发现是没有配置redis密码，但是RedisRateLimiter在遇到异常时直接将异常忽略了
>
> ```java
> return flux.onErrorResume(throwable -> {
>     if (log.isDebugEnabled()) {
>         log.debug("Error calling rate limiter lua", throwable);
>     }
>     return Flux.just(Arrays.asList(1L, -1L));
> })
> ```
>



对于被限流的请求，在请求头中会有提示信息

```
$ curl -si http://localhost:8080/hello
HTTP/1.1 429 Too Many Requests
X-RateLimit-Remaining: 0
X-RateLimit-Requested-Tokens: 1
X-RateLimit-Burst-Capacity: 1
X-RateLimit-Replenish-Rate: 1

```



测试过程中可以通过Redis的monito命令观察redis中的key变化

```
127.0.0.1:6379> monitor
OK
1605183850.579635 [0 47.114.93.81:65310] "EVALSHA" "9d491aea731237273f4274f9ed9660b432b23791" "2" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens" "request_rate_limiter.{0:0:0:0:0:0:0:1}.timestamp" "1" "1" "1605183850" "1"
1605183850.599221 [0 47.114.93.81:65310] "EVAL" "local tokens_key = KEYS[1]\nlocal timestamp_key = KEYS[2]\n--redis.log(redis.LOG_WARNING, \"tokens_key \" .. tokens_key)\n\nlocal rate = tonumber(ARGV[1])\nlocal capacity = tonumber(ARGV[2])\nlocal now = tonumber(ARGV[3])\nlocal requested = tonumber(ARGV[4])\n\nlocal fill_time = capacity/rate\nlocal ttl = math.floor(fill_time*2)\n\n--redis.log(redis.LOG_WARNING, \"rate \" .. ARGV[1])\n--redis.log(redis.LOG_WARNING, \"capacity \" .. ARGV[2])\n--redis.log(redis.LOG_WARNING, \"now \" .. ARGV[3])\n--redis.log(redis.LOG_WARNING, \"requested \" .. ARGV[4])\n--redis.log(redis.LOG_WARNING, \"filltime \" .. fill_time)\n--redis.log(redis.LOG_WARNING, \"ttl \" .. ttl)\n\nlocal last_tokens = tonumber(redis.call(\"get\", tokens_key))\nif last_tokens == nil then\n  last_tokens = capacity\nend\n--redis.log(redis.LOG_WARNING, \"last_tokens \" .. last_tokens)\n\nlocal last_refreshed = tonumber(redis.call(\"get\", timestamp_key))\nif last_refreshed == nil then\n  last_refreshed = 0\nend\n--redis.log(redis.LOG_WARNING, \"last_refreshed \" .. last_refreshed)\n\nlocal delta = math.max(0, now-last_refreshed)\nlocal filled_tokens = math.min(capacity, last_tokens+(delta*rate))\nlocal allowed = filled_tokens >= requested\nlocal new_tokens = filled_tokens\nlocal allowed_num = 0\nif allowed then\n  new_tokens = filled_tokens - requested\n  allowed_num = 1\nend\n\n--redis.log(redis.LOG_WARNING, \"delta \" .. delta)\n--redis.log(redis.LOG_WARNING, \"filled_tokens \" .. filled_tokens)\n--redis.log(redis.LOG_WARNING, \"allowed_num \" .. allowed_num)\n--redis.log(redis.LOG_WARNING, \"new_tokens \" .. new_tokens)\n\nif ttl > 0 then\n  redis.call(\"setex\", tokens_key, ttl, new_tokens)\n  redis.call(\"setex\", timestamp_key, ttl, now)\nend\n\n-- return { allowed_num, new_tokens, capacity, filled_tokens, requested, new_tokens }\nreturn { allowed_num, new_tokens }\n" "2" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens" "request_rate_limiter.{0:0:0:0:0:0:0:1}.timestamp" "1" "1" "1605183850" "1"
1605183850.599531 [0 lua] "get" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens"
1605183850.599546 [0 lua] "get" "request_rate_limiter.{0:0:0:0:0:0:0:1}.timestamp"
1605183850.599558 [0 lua] "setex" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens" "2" "0"
1605183850.599572 [0 lua] "setex" "request_rate_limiter.{0:0:0:0:0:0:0:1}.timestamp" "2" "1605183850"
1605183850.777327 [0 47.114.93.81:65310] "EVALSHA" "9d491aea731237273f4274f9ed9660b432b23791" "2" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens" "request_rate_limiter.{0:0:0:0:0:0:0:1}.timestamp" "1" "1" "1605183850" "1"
1605183850.777385 [0 lua] "get" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens"
1605183850.777399 [0 lua] "get" "request_rate_limiter.{0:0:0:0:0:0:0:1}.timestamp"
1605183850.777412 [0 lua] "setex" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens" "2" "0"
1605183850.777424 [0 lua] "setex" "request_rate_limiter.{0:0:0:0:0:0:0:1}.timestamp" "2" "1605183850"
1605183852.087445 [0 47.114.93.81:65310] "EVALSHA" "9d491aea731237273f4274f9ed9660b432b23791" "2" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens" "request_rate_limiter.{0:0:0:0:0:0:0:1}.timestamp" "1" "1" "1605183852" "1"
1605183852.087514 [0 lua] "get" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens"
1605183852.087525 [0 lua] "get" "request_rate_limiter.{0:0:0:0:0:0:0:1}.timestamp"
1605183852.087537 [0 lua] "setex" "request_rate_limiter.{0:0:0:0:0:0:0:1}.tokens" "2" "0"

```

