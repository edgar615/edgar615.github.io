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



