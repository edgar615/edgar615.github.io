---
layout: post
title: Spring Cloud Config
date: 2019-08-20
categories:
    - Spring
comments: true
permalink: spring-cloud-config.html
---

Spring Cloud Config分为Config Server和Config Client两部分，为分布式系统外部化配置提供了支持。 Spring Cloud Config非常适合Spring应用程序，也能与其他编程语言编写的应用组合使用。

微服务在启动时，通过Config Client请求Config Server以获取配置内容，同时会缓存这些内容。
Spring Cloud Config支持git、svn、consul多种存储，本文主要介绍git的实现

![](/assets/images/posts/spring-cloud-config/spring-cloud-config-1.png)


# 1. get started

## 1.1. Config Server

引入依赖

```
<dependencies>
	<dependency>
	  <groupId>org.springframework.cloud</groupId>
	  <artifactId>spring-cloud-config-server</artifactId>
	</dependency>
	<dependency>
	  <groupId>org.springframework.boot</groupId>
	  <artifactId>spring-boot-starter-web</artifactId>
	</dependency>
</dependencies>
```
我们在 src/main/resources 目录下创建一个 service-config 文件夹，再在这个文件夹下分别创建  user和order 两个子文件夹，请注意这两个子文件夹的名称必须与各个服务自身的名称完全一致。然后我们可以看到这两个子文件夹下面都放着以服务名称命名的针对不同运行环境的 .yml 配置文件。

```
# user.yml
msg: "Hello user - this is from config server"

# user-dev.yml
msg: "Hello user - this is from config server – Development environment"

# user-prod.yml
msg: "Hello user - this is from config server – Production environment"
```

增加配置文件`application.yml`

```
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          #多个文件,分隔
          searchLocations: classpath:/service-config,classpath:/service-config/user,classpath:/service-config/order
server:
  port: 8080
```

编写启动类：

<pre class="line-numbers "><code class="language-java">
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
      SpringApplication.run(ConfigServerApplication.class, args);
    }
}
</code></pre>

观察启动日志可以看到很多与配置中心相关的端点信息

```
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/encrypt],methods=[POST]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.encrypt(java.lang.String,org.springframework.http.MediaType)
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/encrypt/{name}/{profiles}],methods=[POST]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.encrypt(java.lang.String,java.lang.String,java.lang.String,org.springframework.http.MediaType)
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/decrypt/{name}/{profiles}],methods=[POST]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.decrypt(java.lang.String,java.lang.String,java.lang.String,org.springframework.http.MediaType)
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/decrypt],methods=[POST]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.decrypt(java.lang.String,org.springframework.http.MediaType)
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/encrypt/status],methods=[GET]}" onto public java.util.Map<java.lang.String, java.lang.Object> org.springframework.cloud.config.server.encryption.EncryptionController.status()
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/key],methods=[GET]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.getPublicKey()
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/key/{name}/{profiles}],methods=[GET]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.getPublicKey(java.lang.String,java.lang.String)
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{name}-{profiles}.properties],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.properties(java.lang.String,java.lang.String,boolean) throws java.io.IOException
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{name}-{profiles}.yml || /{name}-{profiles}.yaml],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.yaml(java.lang.String,java.lang.String,boolean) throws java.lang.Exception
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{name}/{profiles:.*[^-].*}],methods=[GET]}" onto public org.springframework.cloud.config.environment.Environment org.springframework.cloud.config.server.environment.EnvironmentController.defaultLabel(java.lang.String,java.lang.String)
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{name}/{profiles}/{label:.*}],methods=[GET]}" onto public org.springframework.cloud.config.environment.Environment org.springframework.cloud.config.server.environment.EnvironmentController.labelled(java.lang.String,java.lang.String,java.lang.String)
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{label}/{name}-{profiles}.properties],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.labelledProperties(java.lang.String,java.lang.String,java.lang.String,boolean) throws java.io.IOException
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{label}/{name}-{profiles}.json],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.labelledJsonProperties(java.lang.String,java.lang.String,java.lang.String,boolean) throws java.lang.Exception
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{label}/{name}-{profiles}.yml || /{label}/{name}-{profiles}.yaml],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.labelledYaml(java.lang.String,java.lang.String,java.lang.String,boolean) throws java.lang.Exception
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{name}-{profiles}.json],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.jsonProperties(java.lang.String,java.lang.String,boolean) throws java.lang.Exception
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{name}/{profile}/{label}/**],methods=[GET],produces=[application/octet-stream]}" onto public synchronized byte[] org.springframework.cloud.config.server.resource.ResourceController.binary(java.lang.String,java.lang.String,java.lang.String,javax.servlet.http.HttpServletRequest) throws java.io.IOException
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{name}/{profile}/{label}/**],methods=[GET]}" onto public java.lang.String org.springframework.cloud.config.server.resource.ResourceController.retrieve(java.lang.String,java.lang.String,java.lang.String,javax.servlet.http.HttpServletRequest,boolean) throws java.io.IOException
12876 --- [           main] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped "{[/{name}/{profile}/**],methods=[GET],params=[useDefaultLabel]}" onto public java.lang.String org.springframework.cloud.config.server.resource.ResourceController.retrieve(java.lang.String,java.lang.String,javax.servlet.http.HttpServletRequest,boolean) throws java.io.IOException
```

Spring Cloud Config 为我们提供了强大的集成入口，配置服务器可以将存放在本地文件系统中的配置文件信息自动转化为 RESTful 风格的接口数据。我们可以通过下面这些rest接口访问某个服务的配置文件
```
GET /{application}/{profile}[/{label}]
GET /{application}-{profile}.yml
GET /{label}/{application}-{profile}.yml
GET /{application}-{profile}.properties
GET /{label}/{application}-{profile}.properties
```

- {application}映射客户端的`spring.application.name`
- {profile}映射客户端的`spring.profiles.active`（逗号分隔列表）
- {label}它是服务端的特性，标记版本的一组配置文件



当我们启动配置服务器，并访问 http://localhost:8080/user/default 端点时，可以得到如下信息：

```
$ curl -s http://localhost:8080/user/default
{"name":"user","profiles":["default"],"label":null,"version":null,"state":null,"propertySources":[{"name":"classpath:/service-config/user/user.yml","source":{"msg":"Hello user - this is from config server"}}]}
```

因为我们访问的是http://localhost:8080/user/default 端点，相当于获取的是 user.yml 文件中的配置信息，所以这里的"profiles"值为"default"，意味着我们的配置文件的 Profile  是默认环境。而"label"的值是"master"，实际上也是代表着一种默认版本信息。最后的"propertySources"段展示了配置文件的路径以及具体内容。

如果我们想访问其他环境，修改端点即可

```
$ curl -s http://localhost:8080/user/dev
{"name":"user","profiles":["dev"],"label":null,"version":null,"state":null,"propertySources":[{"name":"classpath:/service-config/user/user-dev.yml","source":{"msg":"Hello user - this is from config server - Development environment"}},{"name":"classpath:/service-config/user/user.yml","source":{"msg":"Hello user - this is from config server"}}]}
```

## 1.2. Config Client
添加依赖
```
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
  </dependencies>
```
增加配置文件`bootstrap.yml`
```
spring:
  cloud:
    config:
      uri: "http://localhost:8080/"
```
编写启动类：
<pre class="line-numbers "><code class="language-java">
@SpringBootApplication
public class ConfigClientApplication {
    public static void main(String[] args) {
      ApplicationContext context = SpringApplication.run(ConfigClientApplication.class, args);
      Environment environment = context.getEnvironment();
      System.out.println(environment.getProperty("msg"));
    }
}
</code></pre>

启动应用，我们可以看到如下输出
```
Fetching config from server at : http://localhost:8080/
...
Hello user - this is from config server
```
在客户端上我们只需要向使用普通配置文件一样使用配置文件即可
<pre class="line-numbers "><code class="language-java">
@RestController
public class MessageRestController {

    @Value("${msg:Hello world - Config Server is not working..pelase check}")
    private String msg;
    
    @RequestMapping("/msg")
    String getMsg() {
        return this.msg;
    }
}
</code></pre>

```
$ curl -s http://localhost:9000/msg
Hello user - this is from config server
```



然后我们可以在通过下面的配置开启某个端点

```
spring:
  profiles:
    active: dev
```

在测试过程中，`spring.profiles.active`需要放置bootstrap.yml中，在applicaiton.yml中不起作用

```
$ curl -s http://localhost:9000/msg
Hello user - this is from config server - Development environment
```

也可以通过`spring.cloud.config.profile`来指定

```
spring:
#  profiles:
#    active: dev
  cloud:
    config:
      label: master
      profile: prod
```

```
$ curl -s http://localhost:9000/msg
Hello user - this is from config server - Production environment
```

# 2. Git

对于 Spring Cloud Config 而言，更加推荐将配置信息存放在 Git 等具有版本控制机制的远程仓库中。假如我们把配置信息放在 Git 仓库中，通常的做法是把所有的配置文件放到自建或公共的 Git 系统中。

因为改变了配置仓库的实现方式，我们同样需要修改 application.yml 中关于配置仓库的配置信息，调整后的配置内容示例如下所示：

```
spring:
  cloud:
    config:
      server:
        git:
          uri: "https://github.com/edgar615/spring-cloud-consul-config-data/"
          search-paths: /service-config, /service-config/user, /service-config/order
          username: xxx
          password: xxx
          #Skipping SSL Certificate Validation
          #skipSslValidation: true
          #Setting HTTP Connection Timeout
```

事实上，基于 Git 的配置方案的最终结果也是将位于 Git 仓库中的远程配置文件加载到本地。一旦配置文件已经加载到本地，那么对这些配置文件的处理方式以及处理效果与前面介绍的本地文件系统是完全一样的。

# 3. Refresh
我们将user.yml`中的文件稍做修改
```
msg: "Hello world - this is from config server, updated"
```
提交git后访问config server

$ curl -s http://localhost:8080/user/default
{"name":"user","profiles":["default"],"label":null,"version":"634dcd3183a7c7d04c5561b880c9615dc16409d9","state":null,"propertySources":[{"name":"https://github.com/edgar615/spring-cloud-consul-config-data//service-config/user/user.yml","source":{"msg":"Hello user - this is from config server, updated"}}]}

我们看到config server中的配置已经更新（通过定时pull）

再次访问config client，发现配置并未更新



```
$ curl -s http://localhost:9000/msg
Hello user - this is from config server
```

这时需要我们通过手动方式来触发更新

引入依赖
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
actuator会增加一个`/actuator/refresh`端点用于主动更新配置

```
management:
  info:
    defaults:
      enabled: true
  endpoints:
    web:
      exposure:
        include: refresh
```

修改MessageRestController，增加注解

<pre class="line-numbers "><code class="language-java">
@RefreshScope
@RestController
public class MessageRestController {

    @Value("${msg:Hello world - Config Server is not working..pelase check}")
    private String msg;
    
    @RequestMapping("/msg")
    String getMsg() {
        return this.msg;
    }
}
</code></pre>
一旦` /actuator/refresh` 被触发, 所有`@RefreshScope`标记的spring bean都会刷新

$ curl -s -X POST http://localhost:9000/actuator/refresh
[]
$ curl -s http://localhost:9000/msg
Hello user - this is from config server, updated


# 5. 集成 Spring Cloud Bus
上面的`/refresh`存在一个问题：在分布式环境下，一个服务一般会部署多个示例，如果更新配置需要去每一台实例手动触发`/refresh`将会极大工作量，所以我们可以通过`spring-cloud-bus`提供的`/bus-refresh`来实现。

Spring Cloud Bus 是 Spring Cloud 中用于实现消息总线的专用组件，集成了 RabbitMQ、Kafka 等主流消息中间件。当我们在 Spring Cloud Config 服务器端工程的类路径中添加 Spring Cloud Bus  的引用并启动应用程序之后，Spring Boot Actuator 就为我们提供了 /actuator/bus-refresh  端点，通过访问该端点就可以达到对客户端所有服务实例的配置进行自动更新的效果。在这种方案中，服务端会主动通知所有客户端进行配置信息的更新，这样我们就不需要关注各个客户端，而只对服务端进行操作即可。

增加依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

增加配置

```
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
          minPartitionCount: 1
          autoCreateTopics: true
          autoAddPartitions: true
```
启动后观察日志我们可以看到暴露了一个`/actuator/bus-refresh`端点
```
Mapped "{[/actuator/bus-refresh/{destination}],methods=[POST]}" onto public java.lang.Object org.springframework.boot.actuate.endpoint.web.servlet.AbstractWebMvcEndpointHandlerMapping$OperationHandler.handle(javax.servlet.http.HttpServletRequest,java.util.Map<java.lang.String, java.lang.String>)
Mapped "{[/actuator/bus-refresh],methods=[POST]}" onto public java.lang.Object org.springframework.boot.actuate.endpoint.web.servlet.AbstractWebMvcEndpointHandlerMapping$OperationHandler.handle(javax.servlet.http.HttpServletRequest,java.util.Map<java.lang.String, java.lang.String>)

```
启动成功后，我们可以发现kafka中多了一个topic
<pre class="line-numbers "><code class="language-shell">
kafka-topics.sh --list --zookeeper localhost:2181
__consumer_offsets
springCloudBus
</code></pre>

请求一下`/actuator/bus-refresh`我们可以看到springCloudBus的主题中多了一条消息
<pre class="line-numbers "><code class="language-shell">
$ curl -s -X POST -H "Content-Type: application/json" http://localhost:9001/actuator/bus-refresh

$ ./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic springCloudBus --from-beginning
{"type":"RefreshRemoteApplicationEvent","timestamp":1552546588187,"originService":"config-bus-refresh:9001:0e7842bbedb5cb2ff9ffdce822756310","destinationService":"**","id":"d1061741-1d49-45f2-aefc-3a2359ba6600"}
{"type":"AckRemoteApplicationEvent","timestamp":1552546588346,"originService":"config-bus-refresh:9001:0e7842bbedb5cb2ff9ffdce822756310","destinationService":"**","id":"45d31331-2fda-49de-893e-25f38f5fa8c0","ackId":"d1061741-1d49-45f2-aefc-3a2359ba6600","ackDestinationService":"**","event":"org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent"}

</code></pre>

**我们可以在构建另外一个config client来测试广播的效果，这里就不多余描述了**，`/bus-refresh`触发后，所有的服务都会刷新配置

![](/assets/images/posts/spring-cloud-config/spring-cloud-config-2.png)

# 6. Git Webhook
上一章的`/bus-refresh`依然依赖于手动触发，我们可以通过`spring-cloud-config-monitor`来实现Git Webhook自动触发

给config server增加依赖
```
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-config-monitor</artifactId>
    </dependency>
```

增加配置
```
management:
  endpoints:
    web:
      exposure:
        include: monitor
spring:
  applicaiton:
    name: config-server
  cloud:
    bus:
      enabled: true
    stream:
      kafka:
        binder:
          brokers: 192.168.1.212:9092
          minPartitionCount: 1
          autoCreateTopics: true
          autoAddPartitions: true

```
启动server，观察输出发现暴露了`/monitor`端点
```
 Mapped "{[/error],produces=[text/html]}" onto public org.springframework.web.servlet.ModelAndView org.springframework.boot.autoconfigure.web.servlet.error.BasicErrorController.errorHtml(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse)
 Mapped "{[/monitor],methods=[POST]}" onto public java.util.Set<java.lang.String> org.springframework.cloud.config.monitor.PropertyPathEndpoint.notifyByPath(org.springframework.http.HttpHeaders,java.util.Map<java.lang.String, java.lang.Object>)
 Mapped "{[/monitor],methods=[POST],consumes=[application/x-www-form-urlencoded]}"
```
**注意/monitor端点只有`spring.cloud.bus.enabled=true`才起作用**

当webhook被触发,  Config Server会发送一个refresh事件给配置被修改了的应用 ( RefreshRemoteApplicationEvent)

模拟webhook的触发
<pre class="line-numbers "><code class="language-shell">
curl -v -X POST "http://localhost:8080/monitor" \
-H "Content-Type: application/json" \
-H "X-Event-Key: repo:push" \
-H "X-Hook-UUID: webhook-uuid" \
-d '{"push": {"changes": []} }'
</code></pre>
![](/assets/images/posts/spring-cloud-config/spring-cloud-config-3.png)
