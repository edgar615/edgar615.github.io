---
layout: post
title: Spring Cloud Eureka
date: 2019-03-26
categories:
    - Spring Boot
    - Spring Cloud
comments: true
permalink: Spring-Cloud-Eureka.html
---

我一般是用consul做服务发现，不过大多数使用spring cloud都会用eureka做服务发现，所以抽时间简单用了一下


# get started
## Server
增加依赖
```
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
```

增加application.yml配置

```
server:
  port: 8671
```

增加启动类

<pre class="line-numbers " data-line="2"><code class="language-java">
@SpringBootApplication
@EnableEurekaServer
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).run(args);
    }
}
</code></pre>

启动应用后，



参考资料

https://thepracticaldeveloper.com/2018/03/18/spring-boot-service-discovery-eureka/