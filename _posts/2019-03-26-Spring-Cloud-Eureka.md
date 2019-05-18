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

<pre class="line-numbers"><code class="language-java">
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

<pre class="line-numbers"><code class="language-java">
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



# 集群

server1的配置

<pre class="line-numbers"><code class="language-yml">
eureka:
  instance:
    #集群这个名字必须相同，如果没有填写，默认为unkown
    appname: eureka-peer
  client:
    registerWithEureka: true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://localhost:8092/eureka/
</code></pre>

server2的配置

<pre class="line-numbers"><code class="language-yml">
eureka:
  instance:
    #集群这个名字必须相同，如果没有填写，默认为unkown
    appname: eureka-peer
  client:
    registerWithEureka: true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://localhost:8091/eureka/
</code></pre>

启动server1，我们可以看到日志中有一条错误信息
```
Getting all instance registry info from the eureka server
Request execution error

com.sun.jersey.api.client.ClientHandlerException: java.net.ConnectException: Connection refused: connect

...

DiscoveryClient_EUREKA-PEER/PC-201809260001:8091 - was unable to send heartbeat!

com.netflix.discovery.shared.transport.TransportException: Cannot execute request on any known server

```
启动server2，我们可以看到日志中没有错误信息
```
DiscoveryClient_EUREKA-PEER/PC-201809260001:8092 - registration status: 204
Initializing Spring FrameworkServlet 'dispatcherServlet'
FrameworkServlet 'dispatcherServlet': initialization started
FrameworkServlet 'dispatcherServlet': initialization completed in 19 ms
Registered instance EUREKA-PEER/PC-201809260001:8092 with status UP (replication=true)
```
server1也不再报错
```
Registered instance EUREKA-PEER/PC-201809260001:8092 with status UP (replication=false)
Got 1 instances from neighboring DS node
Renew threshold is: 1
Changing status to UP
Started Eureka Server
Disable delta property : false
Single vip registry refresh property : null
Force full registry fetch : false
Application is null : false
Registered Applications size is zero : true
Application version is -1: true
Getting all instance registry info from the eureka server
DiscoveryClient_EUREKA-PEER/PC-201809260001:8091 - Re-registering apps/EUREKA-PEER
DiscoveryClient_EUREKA-PEER/PC-201809260001:8091: registering service...
DiscoveryClient_EUREKA-PEER/PC-201809260001:8091 - registration status: 204
The response status is 200
Registered instance EUREKA-PEER/PC-201809260001:8091 with status UP (replication=true)
```
访问`http://localhost:8091`，可以看到`instances`处有了eureka-peer的信息
![](/assets/images/posts/eureka/eureka_peer1.png)

但是我我们发现`8092`在unavailable-replicas中，这是因为eureka集群不能工作在同一个hostname中，我们做如下修改
server1
<pre class="line-numbers"><code class="language-yml">
eureka:
  instance:
    appname: eureka-peer
    hostname: eureka-peer1
  client:
    registerWithEureka: true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://eureka-peer2:8092/eureka/
</code></pre>
server2
<pre class="line-numbers"><code class="language-yml">
eureka:
  instance:
    appname: eureka-peer
    hostname: eureka-peer2
  client:
    registerWithEureka: true
    fetchRegistry: true
    serviceUrl:
      defaultZone: http://eureka-peer1:8091/eureka/
</code></pre>
在hosts文件中设置好`eureka-peer1`和`eureka-peer2`，重新启动server后发现`8092`出现在了`available-replicas`
![](/assets/images/posts/eureka/eureka_peer2.png)

修改并启动client
<pre class="line-numbers"><code class="language-yml">
server:
  port: 9000

eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka-peer1:8091/eureka/
spring:
  application:
    name: eureka-client
</code></pre>
观察`eureka-peer1`和`eureka-peer2`，可以看到client已经注册
![](/assets/images/posts/eureka/eureka_peer3.png)

理想的eureka架构
![](/assets/images/posts/eureka/eureka-architecture.png)

参考资料

https://thepracticaldeveloper.com/2018/03/18/spring-boot-service-discovery-eureka/

https://blog.asarkar.org/technical/netflix-eureka/