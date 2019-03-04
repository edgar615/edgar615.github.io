---
layout: post
title: Spring Actuator
categories:
    - Spring Boot
comments: true
permalink: welcome-to-jekyll.html
---

# 自定义Endpoint 
本章节主要介绍Spring Boot 2.X如何自定义Endpoint，Spring Boot 1.X通过实现`Endpoint`接口实现，不做描述

Spring Boot 2 提供了一个简单的方式实现自定义的Endpoint：`@Endpoint` 。.Spring Boot可以通过`@Endpoint`,`@WebEndpoint`，`@WebEndpointExtension`自动将endpoint通过HTTP协议暴露出来。`@Endpoint`通常和另外三个主键一起组合使用

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
<pre class="line-numbers " data-line="15,20,25,30" data-line-offset><code class="language-java">
@Component
@Endpoint(id = "custom-health")
public class CustomHealthEndPoint {

  private final CustomHealth health = new CustomHealth();

  @PostConstruct
  public void init() {
    Map<String, Object> details = new LinkedHashMap<>();
    details.put("CustomHealthStatus", "Everything looks good");
    health.setHealthDetails(details);
  }

  @ReadOperation
  public CustomHealth health() {
    return health;
  }

  @ReadOperation
  public Object customEndPointByName(@Selector String arg0) {
    return health.getHealthDetails().get(arg0);
  }

  @WriteOperation
  public void writeOperation(@Selector String arg0, String value) {
    health.getHealthDetails().put(arg0, value);
  }

  @DeleteOperation
  public void deleteOperation(@Selector String arg0) {
    health.getHealthDetails().remove(arg0);
  }
}
</code></pre>

1.对应的API地址为 `GET /actuator/custom-health`
<pre class="line-numbers"><code class="language-shell">
$ curl -s -H “Content-Type: application/json” http://localhost:9000/actuator/custom-health
{"CustomHealthStatus":"Everything looks good"}
</code></pre>

2.对应的API地址为 `GET /actuator/custom-health/{arg0}`
3.对应的API地址为 `POST /actuator/custom-health/{arg0}`
4.对应的API地址为 `DELETE /actuator/custom-health/{arg0}`