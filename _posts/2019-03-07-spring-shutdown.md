---
layout: post
title: Spring Boot - 优雅停机
date: 2019-03-08
categories:
    - Spring
comments: true
permalink: spring-shutdown.html
---

在最新的 spring boot 2.3 版本，内置了优雅停机功能

```
server:
  shutdown: graceful # 开启优雅停机，默认使用IMMEDIATE立即关机
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s # 设置缓冲区，最大等待时间,超时候无论线程任务是否执行完毕都会停机处理,默认30s

```

模拟一个耗时请求

```
  @GetMapping("/")
  public String hello() throws InterruptedException {
    Thread.sleep(20 * 1000L);
    return "hello";
  }
```

在执行优雅关键后，日志如下，耗时请求也收到了返回值

```
[extShutdownHook] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
[tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
[extShutdownHook] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'applicationTaskExecutor'

```

我们可以通过Spring Boot提供了shutdown的actuator来触发关闭操作，默认是关闭的

```
management:
  endpoint:
    shutdown:
      enabled: true
  endpoints:
    web:
      exposure:
        include: shutdown
```

