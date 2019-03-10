---
layout: post
title: Spring Actuator
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

重新启动应用。此时，我们可以在控制台中看到如下所示的输出：
```
2019-03-06 23:00:18.774  INFO 5004 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
```
spring boot默认启动了两个Endpoint：`/health` and `/info`，访问`/actuator`我们可以看到返回如下内容
```
{
    "_links":{
        "self":{
            "href":"http://localhost:9000/actuator",
            "templated":false
        },
        "health-component":{
            "href":"http://localhost:9000/actuator/health/{component}",
            "templated":true
        },
        "health-component-instance":{
            "href":"http://localhost:9000/actuator/health/{component}/{instance}",
            "templated":true
        },
        "health":{
            "href":"http://localhost:9000/actuator/health",
            "templated":false
        },
        "info":{
            "href":"http://localhost:9000/actuator/info",
            "templated":false
        }
    }
}
```
`/health`用来获取应用的各类健康指标信息，`/info`用来返回一些应用自定义的信息。后面详细介绍

# 开启端点（endpoint）
默认情况下，除`shutdown`外，其他的端点都是开启的。我们可以通过配置开启
```
management:
  endpoint:
    shutdown:
      enabled: true
```
我们也可以通过如下配置将所有端口关闭
```
management:
  endpoints:
    enabled-by-default: false
```
重新启动应用，我们可以看到如下输出
```
2019-03-06 23:16:40.495  INFO 7656 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 0 endpoint(s) beneath base path '/actuator'
```
再次访问`/actuator`，我们看到返回了如下内容
```
{
    "_links":{
        "self":{
            "href":"http://localhost:9000/actuator",
            "templated":false
        }
    }
}
```
然后我们可以在通过下面的配置开启某个端点
```
management:
  endpoint:
    health:
      enabled: true
  endpoints:
    enabled-by-default: false
```
# 暴露端点
在上一章节我们提到，spring boot除`shutdown`外，其他的端点都是开启的。但是我们在启动日志中只看到了两个端点
```
Exposing 2 endpoint(s) beneath base path '/actuator'
```
是因为为了安全，Spring boot仅对外暴露了两个端口`/health` and `/info`，可以通过下面的配置暴露
```
management:
  endpoints:
    jmx:
      exposure:
      #        exclude:
        include: "*"
    web:
      exposure:
        exclude: env, beans #默认值null
        include: "*" #默认值info, health
```
重启应用后看到启动日志
```
Exposing 13 endpoint(s) beneath base path '/actuator'
```
# 安全

可以通过 spring-security保护我们的http端点，**因为我并没有用spring-security，所以这部分并未测试**

<pre class="line-numbers "><code class="language-java">
@Configuration
public class ActuatorSecurity extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.requestMatcher(EndpointRequest.toAnyEndpoint()).authorizeRequests()
				.anyRequest().hasRole("ENDPOINT_ADMIN")
				.and()
			.httpBasic();
	}

}
</code></pre>

<pre class="line-numbers "><code class="language-java">
@Configuration
public class ActuatorSecurity extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.requestMatcher(EndpointRequest.toAnyEndpoint()).authorizeRequests()
			.anyRequest().permitAll();
	}

}
</code></pre>

# 缓存
可以用下面的配置设置端点的缓存**未测试**
```
management:
  endpoint:
    beans:
      cache:
        time-to-live: 10s
```

# 跨域
可以用下面的配置设置端点的缓存**未测试**
```
management:
  endpoints:
    web:
      cors:
        allowed-origins: example.com
        allowed-methods: GET, POST
        allowed-headers: "*"
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

# 原生端点
`spring-boot-starter-actuator`模块中已经实现了一些原生端点。如果根据端点的作用来说，我们可以原生端点分为三大类：

- 应用配置类：获取应用程序中加载的应用配置、环境变量、自动化配置报告等与Spring Boot应用密切相关的配置类信息。
- 度量指标类：获取应用程序运行过程中用于监控的度量指标，比如：内存信息、线程池信息、HTTP请求统计等。
- 操作控制类：提供了对应用的关闭等操作类功能。

## 健康检查 /health
健康检查端点用来获取应用的各类健康指标信息。在`spring-boot-starter-actuator`模块中自带实现了一些常用资源的健康指标检测器。这些检测器都通过`HealthIndicator`接口实现，并且会根据依赖关系的引入实现自动化装配，可以通过配置是否显示健康检查的细节

```
management:
  endpoint:
    health:
      show-details: "ALWAYS" 
```

这个配置有三个选项

- `never` 不显示细节
- `when-authorized` 仅对特定的角色显示，可以通过`management.endpoint.health.roles`设置
- `always` 永远显示细节



我们可以通过` HealthIndicatorRegistry`在运行时动态添加和删除健康检查

## 编写自定义健康检查
**方式1：实现HealthIndicator接口**

对于响应式实现`ReactiveHealthIndicator `接口

<pre class="line-numbers"><code class="language-java">
@Component
public class HealthCheck implements HealthIndicator {

    @Override
    public Health health() {
        int errorCode = check(); // perform some specific health check
        if (errorCode != 0) {
            return Health.down()
              .withDetail("Error Code", errorCode).build();
        }
        return Health.up().build();
    }
    
    public int check() {
        // Our logic to check health
        return 1;
    }
}

</code></pre>

<pre class="line-numbers"><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/health
{"status":"DOWN","details":{"healthCheck":{"status":"DOWN","details":{"Error Code":1}},"diskSpace":{"status":"UP","details":{"total":253519544320,"free":161544491008,"threshold":10485760}}}}
</code></pre>
**方式2**
<pre class="line-numbers"><code class="language-java">
@Component
public class DiskSpaceHealthIndicator extends AbstractHealthIndicator {

    private final FileStore fileStore;
    private final long thresholdBytes;
     
    @Autowired
    public DiskSpaceHealthIndicator(@Value("${health.filestore.path:${user.dir}}") String path,
                                    @Value("${health.filestore.threshold.bytes:10485760}") long thresholdBytes) throws IOException {
        fileStore = Files.getFileStore(Paths.get(path));
        this.thresholdBytes = thresholdBytes;
    }
     
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        long diskFreeInBytes = fileStore.getUnallocatedSpace();
        if (diskFreeInBytes >= thresholdBytes) {
            builder.up();
        } else {
            builder.down();
        }
    }
}
</code></pre>

Spring Boot 提供了4个状态

- `DOWN` HTTP返回503
- `OUT_OF_SERVICE` HTTP返回503
- `UP`  HTTP返回200
- `UNKNOWN` HTTP返回200

我们也可以通过`Health.status(String)`方法指定自定义的状态，如`FATAL`然后通过`management.health.status.order=FATAL, DOWN, OUT_OF_SERVICE, UNKNOWN, UP`来指定状态的顺序

同样，我们也可以通过`management.health.status.http-mapping.FATAL=503`来自定义状态的编码

**配置说明**
```
management.health.db.enabled=true # Whether to enable database health check.
management.health.cassandra.enabled=true # Whether to enable Cassandra health check.
management.health.couchbase.enabled=true # Whether to enable Couchbase health check.
management.health.defaults.enabled=true # Whether to enable default health indicators.
management.health.diskspace.enabled=true # Whether to enable disk space health check.
management.health.diskspace.path= # Path used to compute the available disk space.
management.health.diskspace.threshold=0 # Minimum disk space, in bytes, that should be available.
management.health.elasticsearch.enabled=true # Whether to enable Elasticsearch health check.
management.health.elasticsearch.indices= # Comma-separated index names.
management.health.elasticsearch.response-timeout=100ms # Time to wait for a response from the cluster.
management.health.influxdb.enabled=true # Whether to enable InfluxDB health check.
management.health.jms.enabled=true # Whether to enable JMS health check.
management.health.ldap.enabled=true # Whether to enable LDAP health check.
management.health.mail.enabled=true # Whether to enable Mail health check.
management.health.mongo.enabled=true # Whether to enable MongoDB health check.
management.health.neo4j.enabled=true # Whether to enable Neo4j health check.
management.health.rabbit.enabled=true # Whether to enable RabbitMQ health check.
management.health.redis.enabled=true # Whether to enable Redis health check.
management.health.solr.enabled=true # Whether to enable Solr health check.
management.health.status.http-mapping= # Mapping of health statuses to HTTP status codes. By default, registered health statuses map to sensible defaults (for example, UP maps to 200).
management.health.status.order=DOWN,OUT_OF_SERVICE,UP,UNKNOWN # Comma-separated list of health statuses in order of severity.
```
## /info
该端点用来返回一些应用自定义的信息，通过`InfoContributor `收集。默认情况下，该端点只会返回一个空的json内容。我们可以在application.properties配置文件中通过info前缀来设置一些属性，比如下面这样
```
info:
  name: @project.artifactId@
  build:
    artifact:   @project.artifactId@
    name: @project.artifactId@
    version: @project.version@
    description: 这是一个测试用例
```
<pre class="line-numbers"><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/info
{"name":"spring-actuator-info","build":{"artifact":"spring-actuator-info","name":"spring-actuator-info","version":"1.0-SNAPSHOT","description":"这是一个测试用例"}}
</code></pre>

spring boot内置了三个info

- `EnvironmentInfoContributor` 环境变量
- `GitInfoContributor` 如果存在`git.properties`显示git信息
- `BuildInfoContributor`如果存在`META-INF/build-info.properties`显示build信息

可以通过配置关闭它们
```
management:info.defaults.enabled
```

生成git信息
```
<build>
	<plugins>
		<plugin>
			<groupId>pl.project13.maven</groupId>
			<artifactId>git-commit-id-plugin</artifactId>
		</plugin>
	</plugins>
</build>
```
<pre class="line-numbers"><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/info
{"name":"spring-actuator-info","build":{"artifact":"spring-actuator-info","name":"spring-actuator-info","version":"1.0-SNAPSHOT","description":"这是一个测试用例"},"git":{"commit":{"time":"2019-03-10T03:24:35Z","id":"70b9f29"},"branch":"master"},"example":{"key":"value"}}
</code></pre>
我们可以看到git信息只显示了一部分，如果我们需要显示全部信息，可以使用下面配置
```
management:
  info:
    git:
      mode: full
```
<pre class="line-numbers"><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/info
{"name":"spring-actuator-info","build":{"artifact":"spring-actuator-info","name":"spring-actuator-info","version":"1.0-SNAPSHOT","description":"这是一个测试用例"},"git":{"build":{"host":"DESKTOP-CTAQIMV","version":"1.0-SNAPSHOT","time":"2019-03-10T13:23:02Z","user":{"name":"edgar615","email":"edgar615@gmail.com"}},"branch":"master","commit":{"message":{"short":"info","full":"info"},"id":{"describe":"70b9f29-dirty","abbrev":"70b9f29","describe-short":"70b9f29-dirty","full":"70b9f29fa4388f238fca5fc7b68e3663db79d7f5"},"time":"2019-03-10T03:24:35Z","user":{"email":"edgar615@gmail.com","name":"edgar615"}},"closest":{"tag":{"name":"","commit":{"count":""}}},"dirty":"true","remote":{"origin":{"url":"https://github.com/edgar615/spring-actuator-example.git"}},"tags":"","total":{"commit":{"count":"4"}}},"example":{"key":"value"}}
</code></pre>


生成build-info信息

```
<build>
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
			<version>2.1.3.RELEASE</version>
			<executions>
				<execution>
					<goals>
						<goal>build-info</goal>
					</goals>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```

### 自定义端点
Spring Boot提供了下列方法来实现自定义端点

- `static properties` – we can add static information using key value pairs.
- `maven build properties` – spring boot automatically exposes maven build properties. We can access them using the `@property.name@` syntax.
- `environment variables` – we can expose environment variables.
- `loading properties from java class` – we can load dynamic properties from java classes by implementing the `InfoContributor` interface.

示例

```
info:

  # static properties
  app:
    name: Spring Boot Actuator Info Endpoint Configuration Example
    description: This tutorial demonstrates how to customize the Spring Boot Actuator Info Endpoint.

  # build properties from maven
  build:
    groupId: @project.groupId@
    artifact: @project.artifactId@
    name: @project.name@
    version: @project.version@

  # environment variables
  env:
    java:
      vendor: ${java.specification.vendor}
      vm-name: ${java.vm.name}
      runtime-version: ${java.runtime.version}
```




### 编写自定义info
实现`InfoContributor`接口
<pre class="line-numbers"><code class="language-java">
@Component
public class MetaDataContributor implements InfoContributor {

    @Autowired
    private ApplicationContext ctx;

    @Override
    public void contribute(Info.Builder builder) {
        Map<String, Object> details = new HashMap<>();
        details.put("bean-definition-count", ctx.getBeanDefinitionCount());
        details.put("startup-date", ctx.getStartupDate());

        builder.withDetail("context", details);
    }
}
</code></pre>
<pre class="line-numbers"><code class="language-shell">
$ curl -s -H "Content-Type: application/json" http://localhost:9000/actuator/info
{"app":{"name":"Spring Boot Actuator Info Endpoint Configuration Example","description":"This tutorial demonstrates how to customize the Spring Boot Actuator Info Endpoint."},"build":{"groupId":"com.github.edgar615","artifact":"spring-actuator-info","name":"spring-actuator-info","version":"1.0-SNAPSHOT"},"env":{"java":{"vendor":"Oracle Corporation","vm-name":"Java HotSpot(TM) 64-Bit Server VM","runtime-version":"1.8.0_181-b13"}},"git":{"build":{"host":"DESKTOP-CTAQIMV","version":"1.0-SNAPSHOT","time":"2019-03-10T13:23:02Z","user":{"name":"edgar615","email":"edgar615@gmail.com"}},"branch":"master","commit":{"message":{"short":"info","full":"info"},"id":{"describe":"70b9f29-dirty","abbrev":"70b9f29","describe-short":"70b9f29-dirty","full":"70b9f29fa4388f238fca5fc7b68e3663db79d7f5"},"time":"2019-03-10T03:24:35Z","user":{"email":"edgar615@gmail.com","name":"edgar615"}},"closest":{"tag":{"name":"","commit":{"count":""}}},"dirty":"true","remote":{"origin":{"url":"https://github.com/edgar615/spring-actuator-example.git"}},"tags":"","total":{"commit":{"count":"4"}}},"context":{"bean-definition-count":264,"startup-date":1552225060528}}
</code></pre>


# 参考资料
[https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#production-ready-endpoints-cors]

[https://spring.io/blog/2017/08/22/introducing-actuator-endpoints-in-spring-boot-2-0]

[https://www.javadevjournal.com/spring-boot/spring-boot-actuator-custom-endpoint]

[https://memorynotfound.com/spring-boot-customize-actuator-info-endpoint-example-configuration]

[https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#production-ready-endpoints-cors]:https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#production-ready-endpoints-cors
[https://www.javadevjournal.com/spring-boot/spring-boot-actuator-custom-endpoint]: https://www.javadevjournal.com/spring-boot/spring-boot-actuator-custom-endpoint
[https://spring.io/blog/2017/08/22/introducing-actuator-endpoints-in-spring-boot-2-0]: https://spring.io/blog/2017/08/22/introducing-actuator-endpoints-in-spring-boot-2-0
[https://memorynotfound.com/spring-boot-customize-actuator-info-endpoint-example-configuration/]:https://memorynotfound.com/spring-boot-customize-actuator-info-endpoint-example-configuration/