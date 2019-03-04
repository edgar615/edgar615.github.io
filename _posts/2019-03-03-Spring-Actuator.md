---
layout: post
title: Spring 
date: 2019-03-04
categories:
    - Spring Boot
comments: true
permalink: Spring-Actuator.html
---

在Spring Boot的众多Starter POMs中有一个特殊的模块，它不同于其他模块那样大多用于开发业务功能或是连接一些其他外部资源。它完全是一个用于暴露自身信息的模块，所以很明显，它的主要作用是用于监控与管理，它就是：`spring-boot-starter-actuator`。

`spring-boot-starter-actuator`模块的实现对于实施微服务的中小团队来说，可以有效地减少监控系统在采集应用指标时的开发量。当然，它也并不是万能的，有时候我们也需要对其做一些简单的扩展来帮助我们实现自身系统个性化的监控需求。下面，在本文中，我们将详解的介绍一些关于`spring-boot-starter-actuator`模块的内容，包括它的原生提供的端点以及一些常用的扩展和配置方式。

# get started

引入依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```



# 自定义Endpoint 
本章节主要介绍Spring Boot 2.X如何自定义Endpoint，Spring Boot 1.X通过实现`Endpoint`接口实现，不做描述

Spring Boot 2 提供了一个简单的方式实现自定义的Endpoint：`@Endpoint` 。.Spring Boot可以通过`@Endpoint`,`@WebEndpoint`，`@WebEndpointExtension`自动将endpoint通过HTTP协议暴露出来( using Jersey, Spring MVC, or Spring WebFlux)。`@Endpoint`通常和另外三个主键一起组合使用

- `@ReadOperation` GET请求
- `@WriteOperation` POST请求
- `@DeleteOperation` DELETE请求

下面我们通过一个例子来看如何自定义Endpoint

数据模型：

<pre class="line-numbers "><code class="language-java">
@JsonInclude(JsonInclude.Include.NON_EMPTY)
public class CustomHealth {

    private Map<String, Object> healthDetails;
    
    @JsonAnyGetter
    public Map<String, Object> getHealthDetails() {
        return this.healthDetails;
    }
    
    public void setHealthDetails(Map<String, Object> healthDetails) {
        this.healthDetails = healthDetails;
    }
}
</code></pre>

自定义Endpoint
<pre class="line-numbers " data-line="15,20,25,30"><code class="language-java">
@Component
@Endpoint(id = "custom-health")
public class CustomHealthEndPoint {

  private final CustomHealth health = new CustomHealth();

  @PostConstruct
  public void init() {
​    Map<String, Object> details = new LinkedHashMap<>();
​    details.put("CustomHealthStatus", "Everything looks good");
​    health.setHealthDetails(details);
  }

  @ReadOperation
  public CustomHealth health() {
​    return health;
  }

  @ReadOperation
  public Object customEndPointByName(@Selector String arg0) {
​    return health.getHealthDetails().get(arg0);
  }

  @WriteOperation
  public void writeOperation(@Selector String arg0, String value) {
​    health.getHealthDetails().put(arg0, value);
  }

  @DeleteOperation
  public void deleteOperation(@Selector String arg0) {
​    health.getHealthDetails().remove(arg0);
  }
}
</code></pre>

1.对应的API地址为 `GET /actuator/custom-health`

<pre class="line-numbers"><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/custom-health
{"CustomHealthStatus":"Everything looks good"}
</code></pre>

2.对应的API地址为 `GET /actuator/custom-health/{arg0}`

<pre class="line-numbers"><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/custom-health/CustomHealthStatus
Everything looks good
</code></pre>
**注意：这里的参数名一定要写成`arg0`，如`@Selector String arg0`，否则接口会报错，测试版本2.1.3**

3.对应的API地址为 `POST /actuator/custom-health/{arg0}`
<pre class="line-numbers"><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/custom-health/foo

curl -s -X POST -d '{"value": "bar"}' -H "Content-Type: application/json" http://localhost:9000/actuator/custom-health/foo

$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/custom-health/foo
bar
</code></pre>

4.对应的API地址为 `DELETE /actuator/custom-health/{arg0}`

<pre class="line-numbers"><code class="language-shell">
$ curl -s -X DELETE -H "Content-Type: application/json" http://localhost:9000/actuator/custom-health/foo

Administrator@PC-201809260001 MINGW64 /d/dev/workspace/edgar615.github.io (master)
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/custom-health/foo

</code></pre>

## Controller Endpoints
使用`@ControllerEndpoint`和`@RestControllerEndpoint `，spring boot可以仅通过Spring MVC和Spring WebFlux暴露endpoint

<pre class="line-numbers "><code class="language-java">
@Component
@RestControllerEndpoint(id = "rest-end-point")
public class RestCustomEndPoint {

  // GET /actuator/rest-end-point/custom
  @GetMapping("/custom")
  public @ResponseBody
  CustomHealth customEndPoint() {
​    CustomHealth health = new CustomHealth();
​    Map<String, Object> details = new LinkedHashMap<>();
​    details.put("CustomHealthStatus", "Everything looks good");
​    health.setHealthDetails(details);
​    return health;
  }
}
</code></pre>

<pre class="line-numbers"><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/rest-end-point/custom
{"CustomHealthStatus":"Everything looks good"}

</code></pre>

# 参考资料
[https://spring.io/blog/2017/08/22/introducing-actuator-endpoints-in-spring-boot-2-0]

[https://www.javadevjournal.com/spring-boot/spring-boot-actuator-custom-endpoint]

[https://www.javadevjournal.com/spring-boot/spring-boot-actuator-custom-endpoint]: https://www.javadevjournal.com/spring-boot/spring-boot-actuator-custom-endpoint
[https://spring.io/blog/2017/08/22/introducing-actuator-endpoints-in-spring-boot-2-0]: https://spring.io/blog/2017/08/22/introducing-actuator-endpoints-in-spring-boot-2-0