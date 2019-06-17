---
layout: post
title: Spring Cloud Eureka
date: 2019-03-26
categories:
    - Spring
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
​    serviceUrl:
​      defaultZone: http://eureka-peer1:8091/eureka/
spring:
  application:
​    name: eureka-client
</code></pre>
观察`eureka-peer1`和`eureka-peer2`，可以看到client已经注册(换了台电脑测试，所以账有些内容和上面的图有差别)
![](/assets/images/posts/eureka/eureka_peer3.png)

理想的eureka架构

![](/assets/images/posts/eureka/eureka-architecture.png)

> High level architecture by Netflix, licensed under Apache License v2.0

# 自我保护机制
Eureka各个节点都是平等的，没有ZK中角色的概念， 即使N-1个节点挂掉也不会影响其他节点的正常运行。

默认情况下，如果Eureka Server在一定时间内（默认90秒）没有接收到某个微服务实例的心跳，Eureka Server将会移除该实例。但是当网络分区故障发生时，微服务与Eureka Server之间无法正常通信，而微服务本身是正常运行的，此时不应该移除这个微服务，所以引入了自我保护机制。

自我保护模式正是一种针对网络异常波动的安全保护措施，使用自我保护模式能使Eureka集群更加的健壮、稳定的运行。

自我保护机制的工作机制是如果在15分钟内超过85%的客户端节点都没有正常的心跳，那么Eureka就认为客户端与注册中心出现了网络故障，Eureka Server自动进入自我保护机制，此时会出现以下几种情况：

1、Eureka Server不再从注册列表中移除因为长时间没收到心跳而应该过期的服务。
2、Eureka Server仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上，保证当前节点依然可用。
3、当网络稳定时，当前Eureka Server新的注册信息会被同步到其它节点中。

因此Eureka Server可以很好的应对因网络故障导致部分节点失联的情况，而不会像ZK那样如果有一半不可用的情况会导致整个集群不可用而变成瘫痪。

Eureka自我保护机制，通过配置 eureka.server.enable-self-preservation来true打开/false禁用自我保护机制，默认打开状态，建议生产环境打开此配置。

# 配置
## client
<pre class="line-numbers"><code class="language-yml">
eureka:
  client:
  	#关闭eureka client，默认true
    enabled: false 
    # 注册自身到eureka服务器，默认true
    register-with-eureka: true
    # 表示是否从eureka服务器获取注册信息，默认true
    fetch-registry: false
    # 客户端从Eureka Server集群里更新Eureka Server信息的频率，单位秒，默认5分钟
    eureka-service-url-poll-interval-seconds: 60 
    # 从Eureka服务器端获取注册信息的间隔时间，单位：秒,默认值30秒
    registry-fetch-interval-seconds: 5
    # 连接 Eureka Server 的超时时间，单位：秒，默认值5
    eureka-server-connect-timeout-seconds: 5
    # 读取 Eureka Server 信息的超时时间，单位：秒，默认值8
    eureka-server-read-timeout-seconds: 8
    #  获取实例时是否过滤，只保留UP状态的实例，默认值true
    filter-only-up-instances: true
    # Eureka 服务端连接空闲关闭时间，单位：秒，默认值30
    eureka-connection-idle-timeout-seconds：30
    # 从Eureka 客户端到所有Eureka服务端的连接总数
    eureka-server-total-connections: 200
    # 从Eureka客户端到每个Eureka服务主机的连接总数
    eureka-server-total-connections-per-host: 50
    # 复制实例变化信息到eureka服务器所需要的时间间隔（s），默认为30秒
    instance-info-replication-interval-seconds:30
    #   最初复制实例信息到eureka服务器所需的时间（s），默认为40秒
    initial-instance-info-replication-interval-seconds:40
    #  获取eureka服务的代理主机，默认为null
    proxy-host: null
    # 获取eureka服务的代理端口, 默认为null 
    proxy-port: null
    # 获取eureka服务的代理用户名，默认为null
    proxy-user-name
    # 获取eureka服务的代理密码，默认为null 
    proxy-password
    # eureka注册表的内容是否被压缩，默认为true，并且是在最好的网络流量下被压缩
    gZip-content: true
    # 获取实现了eureka客户端在第一次启动时读取注册表的信息作为回退选项的实现名称
    backup-registry-impl: null
    # 实例是否使用同一zone里的eureka服务器，默认为true，理想状态下，eureka客户端与服务端是在同一zone下
    prefer-same-zone-eureka: true
    #服务器是否能够重定向客户端请求到备份服务器。 如果设置为false，服务器将直接处理请求，如果设置为true，它可能发送HTTP重定向到客户端。默认为false
    allow-redirects: false
    # 是否记录eureka服务器和客户端之间在注册表的信息方面的差异，默认为false
    log-delta-diff: false
    # ndicates whether the eureka client should disable fetching of delta and should rather resort to getting the full registry information.
    disable-delta: false
    # eureka服务注册表信息里的以逗号隔开的地区名单，如果不这样返回这些地区名单，则客户端启动将会出错。默认为null
    fetch-remote-regions-registry: null
    # 获取实例所在的地区。默认为us-east-1
    region: us-east-1
    # 获取实例所在的地区下可用性的区域列表
    availability-zones: 
    # 设置eureka服务器所在的地址，查询服务和注册服务都需要依赖这个地址，类型为 HashMap，并设置有一组默认值，默认的Key为 defaultZone；默认的Value为 http://localhost:8761/eureka ，如果服务注册中心为高可用集群时，多个注册中心地址以逗号分隔。如果服务注册中心加入了安全验证，这里配置的地址格式为： http://<username>:<password>@localhost:8761/eureka 其中 <username> 为安全校验的用户名；<password> 为该用户的密码
    serviceUrl:
       # 设置eureka服务器所在的地址，可以同时向多个服务注册服务
      defaultZone: http://127.0.0.1:8000/eureka/
    #  心跳执行程序线程池的大小,默认为2
    heartbeat-executor-thread-pool-size: 2
    # 心跳执行程序回退相关的属性，是重试延迟的最大倍数值，默认为10
    heartbeat-executor-exponential-back-off-bound: 10
    # 执行程序缓存刷新线程池的大小，默认为2
    cache-refresh-executor-thread-pool-size: 2
    # 执行程序指数回退刷新的相关属性，是重试延迟的最大倍数值，默认为10
    cache-refresh-executor-exponential-back-off-bound: 10
    # Eureka服务器序列化/反序列化的信息中获取“$”符号的的替换字符串。默认为“_-”
    dollar-replacement: _-
    # eureka服务器序列化/反序列化的信息中获取“_”符号的的替换字符串。默认为“__”
    escape-char-replacement: __
    #  如果设置为true,客户端的状态更新将会点播更新到远程服务器上，默认为true
    on-demand-update-status-change: true
    # 此客户端只对一个单一的VIP注册表的信息感兴趣。默认为null
    registry-refresh-single-vip-address: null
</code></pre>

## instance
<pre class="line-numbers"><code class="language-yml">
eureka:
  instance:
  	#此实例注册到eureka服务端的唯一的实例ID,其组成为{spring.application.instance_id:${random.value}}
  	instance-id: 
  	# 不使用主机名来定义注册中心的地址，而使用IP地址的形式，如果设置eureka.instance.ip-address 属性，则使用该属性配置的IP，否则自动获取除环路IP外的第一个IP地址
  	prefer-ip-address: false
  	# IP地址
  	ip-address: 127.0.0.1
  	# 当前实例的主机名
  	hostname: eureka-client
  	# 服务名，默认取 spring.application.name 配置值，如果没有则为 unknown
  	appname: eureka-client
  	#  获得在eureka服务上注册的应用程序组的名字，默认为unknown
  	app-group-name: unkown
	# 指定服务实例所属数据中心
	data-center-info
  	# 实例注册到eureka服务器时，是否开启通讯，默认为false
  	instance-enabled-onit: false
	# http通信端口 默认值80
	non-secure-port: 80
	eureka.instance.
	# 是否启用HTTP通信端口 	默认为true
	non-secure-port-enabled: true
	# HTTPS通信端口 默认为443
	secure-port: 443
	# 是否启用HTTPS通信端口 	默认为false
	secure-port-enabled: false
	# 服务实例安全主机名称（HTTPS） 	unknown
	secure-virtual-host-name: unknown
	# 该服务实例环境配置
 	environment:
	# 默认地址解析顺序
	default-address-resolution-order:
	# 该服务实例注册到Eureka Server 的初始状态 	up
	initial-status: up
	# 【Eureka Server 端属性】默认开启通信的数量 	
	registry.default-open-for-traffic-count: 1
	# 【Eureka Server 端属性】每分钟续约次数 	
	expected-number-of-renews-per-min: 1
  	# 获取该实例应该接收通信的非安全端口。默认为80
  	# 定义服务续约任务（心跳）的调用间隔，单位：秒，默认值30
  	lease-renewal-interval-in-seconds: 30
  	# 定义服务失效的时间，单位：秒，默认值90
  	lease-expiration-duration-in-seconds: 90
  	# 状态页面的URL，相对路径，默认使用 HTTP 访问，如果需要使用 HTTPS则需要使用绝对路径配置
  	status-page-url-path: /info
  	# 状态页面的URL，绝对路径
  	status-page-url: 
  	# 健康检查页面的URL，相对路径，默认使用 HTTP 访问，如果需要使用 HTTPS则需要使用绝对路径配置
  	health-check-url-path: /health
  	# 健康检查页面的URL，绝对路径
  	health-check-url:
	# 该服务实例安全健康检查地址（URL），绝对地址
	secure-health-check-url:
	# 该服务实例的主页地址（url），绝对地址 
	home-page-url:
	# 该服务实例的主页地址，相对地址 	
	home-page-url-path:	/

</code></pre>

## server
<pre class="line-numbers"><code class="language-yml">
eureka:
	server:
	# 启用自我保护机制，默认为true 
	enable-self-preservation: true
	# 清除无效服务实例的时间间隔（ms），默认1分钟
	eviction-interval-timer-in-ms: 60000
	# 清理无效增量信息的时间间隔（ms），默认30秒 
	delta-retention-timer-interval-in-ms: 30000
	# 禁用增量获取服务实例信息 	
	disable-delta: false
	# 是否记录登录日志 
	log-identity-headers: true
	# 限流大小
	rate-limiter-burst-size: 10
	# 是否启用限流
	rate-limiter-enabled: false
	# 平均请求速率
	rate-limiter-full-fetch-average-rate: 100
	# 是否对标准客户端进行限流
	rate-limiter-throttle-standard-clients: false
	# 服务注册与拉取的平均速率
	rate-limiter-registry-fetch-average-rate: 500
	# 信任的客户端列表
	rate-limiter-privileged-clients: 
	# 15分钟内续约服务的比例小于0.85，则开启自我保护机制，再此期间不会清除已注册的任何服务（即便是无效服务）
	renewal-percent-threshold: 0.85
	# 更新续约阈值的间隔（分钟），默认15分钟 	
	renewal-threshold-update-interval-ms: 15
	# 注册信息缓存有效时长（s），默认180秒
	response-cache-auto-expiration-in-seconds: 180
	# 注册信息缓存更新间隔（s），默认30秒 
	response-cache-update-interval-ms: 30
	# 保留增量信息时长（分钟），默认3分钟 
	retention-time-in-m-s-in-delta-queue: 3
	# 当时间戳不一致时，是否进行同步
	sync-when-timestamp-differs: true
	# 是否使用只读缓存策略
	use-read-only-response-cache: true
</code></pre>

还有些集群相关的配置就不写了，o(╥﹏╥)o
参考资料

https://thepracticaldeveloper.com/2018/03/18/spring-boot-service-discovery-eureka/

https://blog.asarkar.org/technical/netflix-eureka/

https://github.com/Netflix/eureka/wiki/Understanding-Eureka-Peer-to-Peer-Communication

https://mp.weixin.qq.com/s/vwPstQ0R0s_PsEhZnALP9Q