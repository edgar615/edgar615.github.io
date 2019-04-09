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

启动应用后，访问`localhost:8671`可以看到eureka的信息

![](/assets/images/posts/eureka/Eureka1.png)

## Client

增加依赖

```
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
```



增加配置

```
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8671/eureka/
spring:
  application:
    name: eureka-client
```
增加启动类

<pre class="line-numbers " data-line="2"><code class="language-java">
@SpringBootApplication
@EnableEurekaClient
public class EurekaClientApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder(EurekaClientApplication.class).run(args);
    }

}
</code></pre>
启动应用后观察控制台，会发现下面输出
```
c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
com.netflix.discovery.DiscoveryClient    : Disable delta property : false
com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
com.netflix.discovery.DiscoveryClient    : Application is null : false
com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
com.netflix.discovery.DiscoveryClient    : Application version is -1: true
com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
com.netflix.discovery.DiscoveryClient    : The response status is 200
com.netflix.discovery.DiscoveryClient    : Starting heartbeat executor: renew interval is: 30
c.n.discovery.InstanceInfoReplicator     : InstanceInfoReplicator onDemand update allowed rate per min is 4
com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1553992927727 with initial instances count: 0
o.s.c.n.e.s.EurekaServiceRegistry        : Registering application eureka-client with eureka with status UP
com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1553992927732, current=UP, previous=STARTING]
com.netflix.discovery.DiscoveryClient    : DiscoveryClient_EUREKA-CLIENT/DESKTOP-CTAQIMV:eureka-client:9000: registering service...
o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 9000 (http) with context path ''
.s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 9000
c.g.e.e.eureka.EurekaClientApplication   : Started EurekaClientApplication in 4.714 seconds (JVM running for 5.582)
com.netflix.discovery.DiscoveryClient    : DiscoveryClient_EUREKA-CLIENT/DESKTOP-CTAQIMV:eureka-client:9000 - registration status: 204
com.netflix.discovery.DiscoveryClient    : Disable delta property : false
com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
com.netflix.discovery.DiscoveryClient    : Application is null : false
com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
com.netflix.discovery.DiscoveryClient    : Application version is -1: false
com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
com.netflix.discovery.DiscoveryClient    : The response status is 200
```
再次访问`http://localhost:8671`，可以看到`Instances currently registered with Eureka`一栏多了一个实例，状态为`UP`
![](/assets/images/posts/eureka/Eureka2.png)



我们在观察Server


参考资料

https://thepracticaldeveloper.com/2018/03/18/spring-boot-service-discovery-eureka/