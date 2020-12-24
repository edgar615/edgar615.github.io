---
layout: post
title: Spring Cloud - Consul
date: 2020-09-11
categories:
    - Spring
comments: true
permalink: spring-cloud-consul.html
---

# 1. 服务注册与发现

## 1.1. 服务注册

添加依赖

```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

启动类增加`@EnableDiscoveryClient`

```
@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudConsulStudentApplication {

  public static void main(String[] args) {
    SpringApplication.run(SpringCloudConsulStudentApplication.class, args);
  }
}
```

增加consul配置

```
spring:
  application:
    name: student-service
  cloud:
    consul:
      host: localhost
      port: 8500
```

启动服务后可以发现服务已经注册到了Consul中

```
$ curl -s http://localhost:8500/v1/catalog/services
{
    "consul": [],
    "student-service": [
        "secure=false"
    ]
}

$ curl -s http://localhost:8500/v1/catalog/service/student-service
[
    {
        "ID": "bcd2d93d-7894-3014-7374-95467a204bd3",
        "Node": "dev-server",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "lan_ipv4": "127.0.0.1",
            "wan": "127.0.0.1",
            "wan_ipv4": "127.0.0.1"
        },
        "NodeMeta": {
            "consul-network-segment": ""
        },
        "ServiceKind": "",
        "ServiceID": "student-service-9002",
        "ServiceName": "student-service",
        "ServiceTags": [
            "secure=false"
        ],
        "ServiceAddress": "dev-server",
        "ServiceWeights": {
            "Passing": 1,
            "Warning": 1
        },
        "ServiceMeta": {},
        "ServicePort": 9002,
        "ServiceEnableTagOverride": false,
        "ServiceProxy": {
            "MeshGateway": {},
            "Expose": {}
        },
        "ServiceConnect": {},
        "CreateIndex": 5668,
        "ModifyIndex": 5668
    }
]
```

可以看到ServiceID是服务名+端口号组成。可以通过`spring.cloud.consul.discovery.instanceId`修改。可以通过`${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}`指定唯一ID

如果需要关闭服务注册，将`spring.cloud.consul.discovery.register`设置为false。

如果需要关闭服务发现，将`spring.cloud.consul.discovery.enabled`设置为false，它会自动将`spring.cloud.discovery.enabled`设为false。

如果我们将`management.server.port`设置成不同的端口，会将management业务作为服务注册

```
management:
  server:
    port: 4452
```

服务名和示例ID均会在后面加上`-management`

```
 "ServiceID": "student-service-1-management",
 "ServiceName": "student-service-management",
```

## 1.2. 服务发现

使用`@LoadBalanced` 注解可以自动嵌入客户端负载均衡功能。开发人员不需要针对负载均衡做任何特殊的开发或配置。

> https://edgar615.github.io/spring-cloud-ribbon.html

```
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
	return new RestTemplate();
}
```

在RestTemplate请求的地址中之间使用被调用的服务名即可实现服务发现

```
String response = restTemplate
        .exchange("http://student-service/students/{schoolname}", HttpMethod.GET, null,
            new ParameterizedTypeReference<String>() {
            }, schoolname).getBody();
```

也可以通过`DiscoveryClient`自己手动处理

```
List<String> services = discoveryClient.getServices();
System.out.println(services);
List<ServiceInstance> list = discoveryClient.getInstances("student-service");
System.out.println(list);
```

## 1.3. 健康检查

引入依赖，服务会自动像Consul注册HTTP类型的健康检查

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```
$ curl -s http://localhost:8500/v1/agent/checks
{
    "service:student-service-1": {
        "Node": "dev-server",
        "CheckID": "service:student-service-1",
        "Name": "Service 'student-service' check",
        "Status": "passing",
        "Notes": "",
        "Output": "HTTP GET http://dev-server:9002/actuator/health: 200  Output: {\"status\":\"UP\"}",
        "ServiceID": "student-service-1",
        "ServiceName": "student-service",
        "ServiceTags": [
            "secure=false"
        ],
        "Type": "http",
        "Definition": {},
        "CreateIndex": 0,
        "ModifyIndex": 0
    }
}
```

可以看到它是通过`/actuator/health`接口来进行检查，

>  文档上说可以通过`management.health.consul.enabled=false`来关闭健康检查，测试没有效果

可以修改健康检查的地址和检查时间

```
spring:
  cloud:
    consul:
      discovery:
        healthCheckPath: ${management.server.servlet.context-path}/health
        healthCheckInterval: 15s
```

可以为健康检查的请求增加请求头

```
spring:
  cloud:
    consul:
      discovery:
        health-check-headers:
          X-Config-Token: 6442e58b-d1ea-182e-cfa5-cf9cddef0722
```

```
spring:
  cloud:
    consul:
      discovery:
        health-check-headers:
          X-Config-Token:
            - "6442e58b-d1ea-182e-cfa5-cf9cddef0722"
            - "Some other value"
```

## 1.4. 元数据

通过``spring.cloud.consul.discovery.tags`来设置元数据

```
spring:
  cloud:
    consul:
      discovery:
        tags: foo=bar, baz
```

生成的tag如下

```
"ServiceTags": [
	"foo=bar",
	"baz",
	"secure=false"
]
```

## 1.5. Catalog Watch

Spring会自动注册一个Catalog Watch向Consul发送长轮询，当发生变化时会发布一个Heartbeat事件。

# 2. 配置中心

## 2.1. Get Started

添加依赖

```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-consul-config</artifactId>
</dependency>
```

增加consul配置

```
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
```

增加2个KV

```
$ consul kv put config/config-client/foo bar
$ consul kv put config/applicaiton/minConns 1
```

读取配置

```
Environment environment = context.getEnvironment();
// 对应consul的KV config/config-client/foo
System.out.println(environment.getProperty("foo"));
// 对应consul的KV config/application/minConns
System.out.println(environment.getProperty("minConns"));
```

`config/config-client/foo`是服务的私有配置，而config/applicaiton/minConns是全局配置

我们可以通过下面的配置修改默认配置

```
spring:
  cloud:
    consul:
      config:
#        #enabled setting this value to "false" disables Consul Config
#        enabled: true
#        #prefix sets the base folder for configuration values，default:config
        prefix: config
#        #defaultContext sets the folder name used by all applications, default:application
        defaultContext: global
```

## 2.2. Config Watch

Spring会自动注册一个Config Watch向Consul发送长轮询，当配置发生变化时发生一个Refresh事件

通过`@RefreshScope`可以自动更新配置参数

```
@RefreshScope
@RestController
public class MessageRestController {

    @Value("${foo:Hello world - Config Server is not working..pelase check}")
    private String msg;

    @RequestMapping("/msg")
    String getMsg() {
        return this.msg;
    }
}
```

配置项

```
spring:
  cloud:
    consul:
      config:
        watch:
          enabled: true
          delay: 1000
```

## 2.3. 直接使用yml格式

```
spring:
  cloud:
    consul:
      config:
        format: YAML
```

> 没有测试过

```
config/testApp,dev/data
config/testApp/data
config/application,dev/data
config/application/data
```

## 2.4. 使用git2consul

git2consul接受一个或者多个git存储库，并将它的镜像到consul KVs 。 使consul KVs 支持git管理。

安装git2consul

```
npm install -g git2consul
```

创建配置文件git2consul.json

```
{
  "version": "1.0",
  "local_store": "本地仓库备份地址", // 填写本机路径，会把git clone 下来
  "logger": { // 配置日志信息
    "name": "git2consul",
    "streams": [
      {
        "level": "trace",
        "type": "rotating-file",
        "path": "./git2consul.log"
      }
    ],
  "repos": [ // 远程仓库 可以有多个
    {
      "name": "根目录名称",
      "url": "git 地址",
      "include_branch_name": true, //分支信息是否包含到请求中，建议使用
      "branches": [
        "master" // 分支名称
      ],
      "hooks": [
        {
          "type": "polling",
          "interval": "1" // 1分钟，分钟级别，必须为整数
        }
      ]
    }
  ]
}
```

启动git2consul

```
git2consul --config-file git2consul.json -e localhost -p 8500
```

修改Spring配置

```
spring:
  cloud:
    consul:
      config:
        format: FILES
```

> 环境关系暂时没有测试

# 3. 配置项

> https://docs.spring.io/spring-cloud-consul/docs/2.2.4.RELEASE/reference/html/appendix.html#common-application-properties

```
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        #启用服务发现?
        enabled: true
        #注册服务时使用的标签
        tags: foo=bar, baz
        #注册管理服务时使用的标签
        managementTags: foo=bar, baz
        #服务名称
        serviceName: ${spring.application.name}
        #唯一的服务实例ID
        instanceId: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
        #服务实例区
        instanceZone: test
        #调用健康检查的备用服务器路径
        healthCheckPath: ${management.contextPath}/health
        #自定义运行状况检查网址以覆盖默认值
        healthCheckUrl: /check
        #执行健康检查的频率(e.g.10s)
        healthCheckInterval: 15s
        #健康检查超时时间(e.g.10s)
        healthCheckTimeout: 10s
        #访问服务时使用的IP地址（必须设置preferIpAddress使用
        ipAddress: 127.0.0.1
        #访问服务器时使用的主机名
        #在注册时使用ip地址而不是主机名
        preferIpAddress: true
        hostname: edgar-pc
        #端口注册服务（默认为侦听端口（8500））
        port: 9000
        #端口注册管理服务（默认为管理端口）
        managementPort: 9000
        #是使用http还是https注册服务
        scheme: http
        managementSuffix: .do
        #我们将如何确定使用地址的来源
        preferAgentAddress: false
        catalogServicesWatchDelay: 10
        catalogServicesWatchTimeout: 10
        #服务实例区域来自元数据。这允许更改元数据标签名称。
        defaultZoneMetadataName: zone
        # 服务地图->在服务器列表中查询的标签。
        # 这允许通过单个标签过滤服务。
        serverListQueryTags:
          foo: bar
        #如果没有在serverListQueryTags中列出，请在服务列表中查询标签。
        defaultQueryTag: test=test
        #将“passing”参数添加到v1healthserviceserviceName。这会将健康检查推送到服务器
        queryPassing: true
        #是否在consul中注册
        register: true
        #是否进行健康检查（一般在开发中使用）
        registerHealthCheck: true
        #在服务注册期间抛出异常，如果为true，否则为log
        failFast: true
      config:
        #enabled setting this value to "false" disables Consul Config
        enabled: true
        #prefix sets the base folder for configuration values，default:config
        prefix: config
        #defaultContext sets the folder name used by all applications, default:application
        defaultContext: global
        #profileSeparator sets the value of the separator used to separate the profile name in property sources with profiles,default:,
        profileSeparator: '::'
        #cause the configuration module to log a warning rather than throw an exception
        failFast: false
        watch:
          enabled: true
          delay: 1000
```