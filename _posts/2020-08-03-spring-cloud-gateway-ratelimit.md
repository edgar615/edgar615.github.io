---
layout: post
title: Spring Cloud Gateway��4��- ����
date: 2020-08-03
categories:
    - Spring Cloud
comments: true
permalink: spring-cloud-gateway-ratelimit.html
---

Spring Cloud Gateway �Ѿ�������һ��RequestRateLimiterGatewayFilterFactory�����ǿ���ֱ��ʹ�á�

ĿǰRequestRateLimiterGatewayFilterFactory��ʵ�������� Redis���������ǻ�Ҫ����spring-boot-starter-data-redis-reactive������RequestRateLimiterGatewayFilterFactory����ע��

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

��������������

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

�ù�������Ҫ��������������

- burstCapacity������Ͱ��������
- replenishRate������Ͱÿ�����ƽ�����ʡ�
- key-resolver�����������ļ��Ľ������� Bean ��������֡���ʹ�� SpEL ���ʽ����#{@beanName}�� Spring �����л�ȡ Bean ����

���������Ҫʵ��һ��key-resolver

```
@Bean
public KeyResolver hostAddrKeyResolver() {
	return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
}
```

�����KeyResolver�ǻ�ȡ�����û�ip��Ϊ����key������Ҳ����ʵ��������������ʽ

**��ȡ�����û�id��Ϊ����key**

```
@Bean
public KeyResolver userKeyResolver() {
	return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("userId"));
}
```

**��ȡ�����ַ��uri��Ϊ����key**

```
@Bean
KeyResolver apiKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getPath().value());
}
```

> �Լ��ڲ��Ե�ʱ��һֱû�����ã�����������û������redis���룬����RedisRateLimiter�������쳣ʱֱ�ӽ��쳣������
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



���ڱ�����������������ͷ�л�����ʾ��Ϣ

```
$ curl -si http://localhost:8080/hello
HTTP/1.1 429 Too Many Requests
X-RateLimit-Remaining: 0
X-RateLimit-Requested-Tokens: 1
X-RateLimit-Burst-Capacity: 1
X-RateLimit-Replenish-Rate: 1

```



���Թ����п���ͨ��Redis��monito����۲�redis�е�key�仯

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

