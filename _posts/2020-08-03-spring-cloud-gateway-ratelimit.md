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



