---
layout: post
title: Spring Cloud Gateway - 服务发现
date: 2020-08-09
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-discovery.html
---

Spring Cloud Gateway支持两种方式的服务发现

- 直接在路由的url中指定对应的微服务

```
routes:
  - id: body_read
    uri: lb://eureka-client
    predicates:
      - After=2020-04-20T00:00:00+08:00[Asia/Shanghai]
```

需要配置相应的服务发现客户端`@EnableEurekaClient`或者`@EnableDiscoveryClient`

- 由`DiscoveryClientRouteDefinitionLocator`自动生成

Spring Cloud Gateway可以使用服务发现客户端接口DiscoveryClient，从服务注意中心获取服务注册信息，然后配置相应的路由。注意，需要在配置中添加如下配置开启这个功能：

```
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
```

默认情况下，通过服务发现客户端DiscoveryClient自动配置的路由信息中，只包括一个默认的Predicate和Filter。默认的Predicate是一个Path Predicate，模式是 `/serviceId/**`，serviceId就是从服务发现客户端中获取的服务ID。默认的Filter是一个重写路径过滤器，它的正则表达式为：`/serviceId/(?<remaining>.*)`，它的作用是将请求路径中的serviceId去掉，变为`/(?<remaining>.*)`

> 我在测试的时候serviceId必须是大写，还未深入了解原因
>
> ```
> $ curl -si http://localhost:8080/EUREKA-CLIENT/str
> HTTP/1.1 200 OK
> Content-Type: text/plain;charset=UTF-8
> Content-Length: 13
> Date: Tue, 24 Nov 2020 12:38:46 GMT
> 
> Hello Gateway
> 
> $ curl -si http://localhost:8080/eureka-client/str
> HTTP/1.1 404 Not Found
> transfer-encoding: chunked
> Vary: Origin
> Vary: Access-Control-Request-Method
> Vary: Access-Control-Request-Headers
> Content-Type: application/json
> Date: Tue, 24 Nov 2020 12:42:56 GMT
> 
> {"timestamp":"2020-11-24T12:42:56.031+00:00","status":404,"error":"Not Found","message":"","path":"/eureka-client/str"}
> 
> ```
>
> 

如果想要添加自定义的Predicate和Filters，可以这样配置：`spring.cloud.gateway.discovery.locator.predicates[x]`和`spring.cloud.gateway.discovery.locator.filters[y]`,当使用这种配置方式时，不会再保留原来默认的Predicate和Filter了，如果你还需要原来的配置，需要手动添加到配置中，如下面所示

```
spring.cloud.gateway.discovery.locator.predicates[0].name: Path
spring.cloud.gateway.discovery.locator.predicates[0].args[pattern]: "'/'+serviceId+'/**'"
spring.cloud.gateway.discovery.locator.predicates[1].name: Host
spring.cloud.gateway.discovery.locator.predicates[1].args[pattern]: "'**.foo.com'"
spring.cloud.gateway.discovery.locator.filters[0].name: Hystrix
spring.cloud.gateway.discovery.locator.filters[0].args[name]: serviceId
spring.cloud.gateway.discovery.locator.filters[1].name: RewritePath
spring.cloud.gateway.discovery.locator.filters[1].args[regexp]: "'/' + serviceId + '/(?<remaining>.*)'"
spring.cloud.gateway.discovery.locator.filters[1].args[replacement]: "'/${remaining}'"
```

> 暂时不会用到，后面有机会在深入了解