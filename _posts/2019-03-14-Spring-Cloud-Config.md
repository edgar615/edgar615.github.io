---
layout: post
title: Spring Config
date: 2019-03-04
categories:
    - Spring Cloud
comments: true
permalink: Spring-Cloud-Config.html
---

### 为什么要统一管理微服务配置

对于Spring Boot应用，我们可以将配置内容写入`application.yml`，设置多个profile，也可以用多个`application-{profile}.properties`文件配置，并在启动时指定`spring.profiles.active={profile}`来加载不同环境下的配置。

在Spring Cloud微服务架构中，这种方式未必适用，微服务架构对配置管理有着更高的要求，如：

- 集中管理：成百上千（可能没这么多）个微服务需要集中管理配置，否则维护困难、容易出错；
- 运行期动态调整：某些参数需要在应用运行时动态调整（如连接池大小、熔断阈值等），并且调整时不停止服务；
- 自动更新配置：微服务能够在配置发生变化是自动更新配置。

以上这些要求，传统方式是无法实现的，所以有必要借助一个通用的配置管理机制，通常使用配置服务器来管理配置。

### Sping Cloud Config简介

Spring Cloud Config分为Config Server和Config Client两部分，为分布式系统外部化配置提供了支持。 Spring Cloud Config非常适合Spring应用程序，也能与其他编程语言编写的应用组合使用。

微服务在启动时，通过Config Client请求Config Server以获取配置内容，同时会缓存这些内容。
Spring Cloud Config支持git、svn、consul多种存储，本文主要介绍git的实现
![](/assets/images/posts/spring-cloud-config/spring-cloud_config_01.png)


# get started

## Config Server

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
增加配置文件`bootstrap.yml`
```
spring:
  cloud:
    config:
      server:
        git:
          uri: "https://github.com/edgar615/spring-cloud-consul-config-data/"
          search-paths: /
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

我们在git仓库中提交对应的配置文件
```
config-server-client-development.yml
config-server-client-production.yml
config-server-client.yml
```
文件内容分别如下
config-server-client.yml
```
msg: "Hello world - this is from config server"
```
config-server-client-development.yml
```
msg: "Hello world - this is from config server – Development environment"
```
config-server-client-production.yml
```
msg: "Hello world - this is from config server – Production environment"
```
启动server，观察启动日志可以看到很多与配置中心相关的端点信息

```
 Mapped "{[/encrypt],methods=[POST]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.encrypt(java.lang.String,org.springframework.http.MediaType)
 Mapped "{[/encrypt/{name}/{profiles}],methods=[POST]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.encrypt(java.lang.String,java.lang.String,java.lang.String,org.springframework.http.MediaType)
 Mapped "{[/decrypt/{name}/{profiles}],methods=[POST]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.decrypt(java.lang.String,java.lang.String,java.lang.String,org.springframework.http.MediaType)
 Mapped "{[/decrypt],methods=[POST]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.decrypt(java.lang.String,org.springframework.http.MediaType)
 Mapped "{[/encrypt/status],methods=[GET]}" onto public java.util.Map<java.lang.String, java.lang.Object> org.springframework.cloud.config.server.encryption.EncryptionController.status()
 Mapped "{[/key],methods=[GET]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.getPublicKey()
 Mapped "{[/key/{name}/{profiles}],methods=[GET]}" onto public java.lang.String org.springframework.cloud.config.server.encryption.EncryptionController.getPublicKey(java.lang.String,java.lang.String)
 Mapped "{[/{name}-{profiles}.properties],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.properties(java.lang.String,java.lang.String,boolean) throws java.io.IOException
 Mapped "{[/{name}-{profiles}.yml || /{name}-{profiles}.yaml],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.yaml(java.lang.String,java.lang.String,boolean) throws java.lang.Exception
 Mapped "{[/{name}/{profiles:.*[^-].*}],methods=[GET]}" onto public org.springframework.cloud.config.environment.Environment org.springframework.cloud.config.server.environment.EnvironmentController.defaultLabel(java.lang.String,java.lang.String)
 Mapped "{[/{name}-{profiles}.json],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.jsonProperties(java.lang.String,java.lang.String,boolean) throws java.lang.Exception
 Mapped "{[/{name}/{profiles}/{label:.*}],methods=[GET]}" onto public org.springframework.cloud.config.environment.Environment org.springframework.cloud.config.server.environment.EnvironmentController.labelled(java.lang.String,java.lang.String,java.lang.String)
 Mapped "{[/{label}/{name}-{profiles}.yml || /{label}/{name}-{profiles}.yaml],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.labelledYaml(java.lang.String,java.lang.String,java.lang.String,boolean) throws java.lang.Exception
 Mapped "{[/{label}/{name}-{profiles}.properties],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.labelledProperties(java.lang.String,java.lang.String,java.lang.String,boolean) throws java.io.IOException
 Mapped "{[/{label}/{name}-{profiles}.json],methods=[GET]}" onto public org.springframework.http.ResponseEntity<java.lang.String> org.springframework.cloud.config.server.environment.EnvironmentController.labelledJsonProperties(java.lang.String,java.lang.String,java.lang.String,boolean) throws java.lang.Exception
 Mapped "{[/{name}/{profile}/{label}/**],methods=[GET],produces=[application/octet-stream]}" onto public synchronized byte[] org.springframework.cloud.config.server.resource.ResourceController.binary(java.lang.String,java.lang.String,java.lang.String,javax.servlet.http.HttpServletRequest) throws java.io.IOException
 Mapped "{[/{name}/{profile}/{label}/**],methods=[GET]}" onto public java.lang.String org.springframework.cloud.config.server.resource.ResourceController.retrieve(java.lang.String,java.lang.String,java.lang.String,javax.servlet.http.HttpServletRequest,boolean) throws java.io.IOException
 Mapped "{[/{name}/{profile}/**],methods=[GET],params=[useDefaultLabel]}" onto public java.lang.String org.springframework.cloud.config.server.resource.ResourceController.retrieve(java.lang.String,java.lang.String,javax.servlet.http.HttpServletRequest,boolean) throws java.io.IOException
```
我们可以通过下面这些rest接口访问某个服务的配置文件
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

**注意：有的资料写的profile可以在`spring.profiles.active`中配置，但我在测试过程中不起作用，只能使用`spring.cloud.config.profile`，但是在触发/refresh的时候却又自动使用了`spring.profiles.active`中的配置**
示例

<pre class="line-numbers "><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:8080/config-server-client.yml
msg: Hello world - this is from config server
$ curl -s -H "Content-Type: application/json" http://localhost:8080/config-server-client-development.yml
msg: Hello world - this is from config server – Development environment
$ curl -s -H "Content-Type: application/json" http://localhost:8080/config-server-client-production.yml
msg: Hello world - this is from config server – Production environment
</code></pre>

通过`GET /{application}/{profile}[/{label}]`可以看到更完整的信息
<pre class="line-numbers "><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:8080/config-server-client/default
{"name":"config-server-client","profiles":["default"],"label":null,"version":"b9a212dcd67a4bf1d7bfd96a7c814662a0edab6f","state":null,"propertySources":[{"name":"https://github.com/edgar615/spring-cloud-consul-config-data//config-server-client.yml","source":{"msg":"Hello world - this is from config server"}}]}
</code></pre>


## 创建客户端
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
  application:
    name: config-server-client
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
Hello world - this is from config server
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
然后我们可以在通过下面的配置开启某个端点
spring.cloud.config.profile=default
<pre class="line-numbers "><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/msg
Hello world - this is from config server
</code></pre>
spring.cloud.config.profile=development
<pre class="line-numbers "><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/msg
Hello world - this is from config server – Development environment
</code></pre>
# Refresh
我们将`config-server-client.yml`中的文件稍做修改
```
msg: "Hello world - this is from config server, updated"
```
提交git后访问config server

<pre class="line-numbers "><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:8080/config-server-client.yml
msg: Hello world - this is from config server, updated
</code></pre>
我们看到config server中的配置已经更新（通过定时pull）

再次访问config client，发现配置并未更新
<pre class="line-numbers "><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/msg        Hello world - this is from config server
</code></pre>
这时需要我们通过手动方式来触发更新

引入依赖
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
actuator会增加一个`/actuator/refresh`端点用于主动更新配置
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
```
$ curl -s -X POST -H "Content-Type: application/json" http://localhost:9000/actuator/refresh
["msg","config.client.version"]
$ curl -s -H "Content-Type: application/json" http://localhost:9000/msg        Hello world - this is from config server, updated
```

# /bus-refresh
上面的`/refresh`存在一个问题：在分布式环境下，一个服务一般会部署多个示例，如果更新配置需要去每一台实例手动触发`/refresh`将会极大工作量，所以我们可以通过`spring-cloud-bus`提供的`/bus-refresh`来实现

# 参考资料
[https://springbootdev.com/2018/07/14/microservices-introduction-to-spring-cloud-config-server-with-client-examples/]

[https://springbootdev.com/2018/07/14/microservices-introduction-to-spring-cloud-config-server-with-client-examples/]:https://springbootdev.com/2018/07/14/microservices-introduction-to-spring-cloud-config-server-with-client-examples/
