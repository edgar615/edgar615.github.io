---
layout: post
title: Spring Cloud Gateway
date: 2020-08-01
categories:
    - Spring Cloud
comments: true
permalink: spring-cloud-gateway.html
---

# 1. 快速入门

## 1.1. 添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
        <exclusions>
            <exclusion>
                <artifactId>spring-boot-starter-web</artifactId>
                <groupId>org.springframework.boot</groupId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 1.2. 添加一个Route

```java
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class);
    }

    @Bean
    public RouteLocator myRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
                .route(p -> p
                        .path("/hello")
                        .filters(f -> f.addRequestHeader("Hello", "World"))
                        .uri("http://localhost:9001"))
                .build();
    }
}
```

当请求网关的`/hello`时，网关会将请求转发到`http://localhost:9001/hello`，并添加请求头`Hello:World`。

```
$ curl -s http://localhost:8080/hello
{"content-length":"0","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","hello":"World","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:60324\"","accept":"*/*","user-agent":"curl/7.59.0"}
```

## 1. 3. 使用Hystrix

添加依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

增加route

```java
.route(p -> p.path("/hystrix")
		.filters(f -> f.hystrix(config -> config.setName("hystrix-test")))
		.uri("http://localhost:9001"))
```

增加一个测试`hystric`api

```java
@GetMapping("/hystrix")
public Map<String, String> hystrix() throws InterruptedException {
    TimeUnit.SECONDS.sleep(5);
    return Collections.singletonMap("path", "hystrix");
}
```

```shell
$ curl  --dump-header -  -s http://localhost:8080/hystrix
HTTP/1.1 504 Gateway Timeout
Content-Type: application/json;charset=UTF-8
Content-Length: 158

{"timestamp":"2020-08-22T08:00:07.882+0000","path":"/hystrix","status":504,"error":"Gateway Timeout","message":"Response took longer than configured timeout"}
```

可以看到当Hystrix在超时后返回了504的错误

**降级**

在gateway上增加一个降级的接口

```java
@RequestMapping("/fallback")
public Mono<Map<String, String>> fallback() {
    return Mono.just(Collections.singletonMap("path", "fallback"));
}
```

在路由`/hystrix`上增加一个降级配置

```java
.route(p -> p.path("/hystrix")
       .filters(f -> f.hystrix(config -> config.setName("hystrix-test").
                               setFallbackUri("forward:/fallback")))
       .uri("http://localhost:9001"))
```

可以看，返回的不再是504

```shell
$ curl  --dump-header -  -s http://localhost:8080/hystrix
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Content-Length: 19

{"path":"fallback"}
```

## 1.4. 单元测试

为了实现单元测试，我们首先要对Gateway的路由做改造，首先定义一个`UriConfiguration`用于动态修改下游服务的地址

```java
@ConfigurationProperties
class UriConfiguration {

    private String httpbin = "http://httpbin.org:80";

    public String getHttpbin() {
        return httpbin;
    }

    public void setHttpbin(String httpbin) {
        this.httpbin = httpbin;
    }
}
```

`GatewayApplication`增加注解`@EnableConfigurationProperties(UriConfiguration.class)`用来激活`ConfigurationProperties`

对Route进行改造

```java
@Bean
public RouteLocator myRoutes(RouteLocatorBuilder builder, UriConfiguration uriConfiguration) {
	String httpUri = uriConfiguration.getHttpbin();
	return builder.routes()
			.route(p -> p.path("/hello")
					.filters(f -> f.addRequestHeader("Hello", "World"))
					.uri(httpUri))
			.route(p -> p
					.host("*.hystrix.com")
					.filters(f -> f
							.hystrix(config -> config
									.setName("mycmd")
									.setFallbackUri("forward:/fallback")))
					.uri(httpUri))
			.build();
}
```

编写单元测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
        properties = {"httpbin=http://localhost:${wiremock.server.port}"})
@AutoConfigureWireMock(port = 0)
public class GatewayTest {
    @Autowired
    private WebTestClient webClient;

    @Test
    public void contextLoads() throws Exception {
        //Stubs
        stubFor(get(urlEqualTo("/hello"))
                .willReturn(aResponse()
                        .withBody("{\"headers\":{\"Hello\":\"World\"}}")
                        .withHeader("Content-Type", "application/json")));
        ObjectMapper objectMapper = new ObjectMapper();
        byte[] body = objectMapper.writeValueAsBytes(Collections.singletonMap("path", "fallback"));
        stubFor(get(urlEqualTo("/delay/3"))
                .willReturn(aResponse()
                        .withBody(body)
                        .withFixedDelay(3000)));

        webClient
                .get().uri("/hello")
                .exchange()
                .expectStatus().isOk()
                .expectBody()
                .jsonPath("$.headers.Hello").isEqualTo("World");

        webClient
                .get().uri("/delay/3")
                .header("Host", "www.hystrix.com")
                .exchange()
                .expectStatus().isOk()
                .expectBody(Map.class)
                .consumeWith(
                        response -> assertThat(response.getResponseBody()).containsEntry("path", "fallback"));
    }

}
```

`@AutoConfigureWireMock(port = 0)`注解用来使用一个随机端口启动WireMock 。



# 2. 进阶

## 2.1. 词汇表

- Route 路由：gateway的基本构建模块。它由ID、目标URI、断言集合和过滤器集合组成。如果聚合断言结果为真，则匹配到该路由。
- Predicate 断言：这是一个Java 8 Function Predicate。输入类型是 Spring Framework ServerWebExchange。这允许开发人员可以匹配来自HTTP请求的任何内容，例如Header或参数。
- Filter 过滤器：这些是使用特定工厂构建的 Spring FrameworkGatewayFilter实例。所以可以在返回请求之前或之后修改请求和响应的内容。

## 2.2. 工作流程

![](/assets/images/posts/spring-cloud-gateway/spring-cloud-gateway-1.png)

客户端向Spring Cloud Gateway发出请求，如果Gateway Handler  Mapping确定请求与路由匹配，则将其发送到Gateway Web  Handler，此handler通过特定于该请求的过滤器链处理请求。

图中filters被虚线划分的原因是filters可以在发送代理请求之前或之 后执行逻辑。先执行所有“pre filter”逻辑，然后进行请求代理。在请求代理执行完后，执行“post filter”逻辑。

# 3. 路由断言

Spring Cloud Gateway将路由作为Spring WebFlux `HandlerMapping`基础结构的一部分进行匹配。Spring Cloud Gateway包含许多内置的路由断言Factories。这些断言都匹配HTTP请求的不同属性。多个路由断言Factories可以通过 `and` 组合使用。

## 3.1. After

After Route Predicate Factory采用一个参数——日期时间。在该日期时间之后发生的请求都将被匹配。

时间`ISO-8601`格式，例如`2007-12-03T10:15:30+01:00 Europe/Paris`, 基于`ZonedDateTime`实现

```
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: http://localhost:9001
        predicates:
        - After=2020-08-23T17:30:00.843+08:00[Asia/Shanghai]
```

```shell
$ curl -s http://localhost:8080/hello
{"content-length":"0","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:49822\"","accept":"*/*","user-agent":"curl/7.59.0"}
```

如果没有匹配到After将返回404

## 3.3. Between

Between 路由断言 Factory有两个参数，datetime1和datetime2。在datetime1和datetime2之间的请求将被匹配。datetime2参数的实际时间必须在datetime1之后

```
- id: between_route
  uri: http://localhost:9001
  predicates:
  - Between=2020-08-22T17:30:00.843+08:00[Asia/Shanghai],2020-08-30T17:30:00.843+08:00[Asia/Shanghai]
```

## 3.4. Cookie

Cookie 路由断言 Factory有两个参数，cookie名称和正则表达式。请求包含次cookie名称且正则表达式为真的将会被匹配。

```
- id: cookie_route
  uri: http://localhost:9001
  predicates:
  - Cookie=user,edgar
```



```shell
$ curl -s --cookie "user=edgar" http://localhost:8080/hello
{"content-length":"0","cookie":"user=edgar","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:50014\"","accept":"*/*","user-agent":"curl/7.59.0"

$ curl -s --cookie "user=anon" http://localhost:8080/hello
{"timestamp":"2020-08-23T09:48:20.870+0000","path":"/hello","status":404,"error":"Not Found","message":null}
```

使用正则表达式

```
- id: cookie_route
  uri: http://localhost:9001
  predicates:
  - Cookie=user,edgar
```

```shell
$ curl -s --cookie "user=edgar" http://localhost:8080/hello
{"timestamp":"2020-08-23T09:49:07.320+0000","path":"/hello","status":404,"error":"Not Found","message":null}

$ curl -s --cookie "user=edgar5" http://localhost:8080/hello
{"content-length":"0","cookie":"user=edgar5","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:50043\"","accept":"*/*","user-agent":"curl/7.59.0"}
```

## 3.5. Header

Header 路由断言 Factory有两个参数，header名称和正则表达式。请求包含次header名称且正则表达式为真的将会被匹配。

```
- id: header_route
  uri: http://localhost:9001
  predicates:
  - Header=X-Request-Id,\d+
```

```shell
$ curl -s -H "X-Request-Id:123" http://localhost:8080/hello
{"x-request-id":"123","content-length":"0","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:50109\"","accept":"*/*","user-agent":"curl/7.59.0"}
```

## 3.6. Host

Host 路由断言 Factory包括一个参数：host name列表。使用Ant路径匹配规则，`.`作为分隔符。

```
  - id: host_route
	uri: http://localhost:9001
	predicates:
	- Host=**.somehost.org,**.anotherhost.org
```

```shell
$ curl -s -H "Host:www.somehost.org" http://localhost:8080/hello
{"content-length":"0","x-forwarded-proto":"http","x-forwarded-host":"www.somehost.org","host":"localhost:9001","x-forwarded-port":"80","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=www.somehost.org;for=\"0:0:0:0:0:0:0:1:50143\"","accept":"*/*","user-agent":"curl/7.59.0"}

$ curl -s -H "Host:www.somehost.io" http://localhost:8080/hello
{"timestamp":"2020-08-23T09:56:42.115+0000","path":"/hello","status":404,"error":"Not Found","message":null}

```

## 3.7. Method

Method 路由断言 Factory只包含一个参数: 需要匹配的HTTP请求方式

```
  - id: method_route
	uri: http://localhost:9001
	predicates:
	- Method=GET
```

## 3.8. Path

Path 路由断言 Factory 有2个参数: 一个Spring `PathMatcher`表达式列表和可选`matchOptionalTrailingSeparator`标识 .

```
- id: path_route
  uri: http://localhost:9001
  predicates:
  - Path=/foo/{segment},/bar/{segment}
```

例如: `/foo/1` or `/foo/bar` or `/bar/baz`的请求都将被匹配

URI 模板变量 (如上例中的 `segment` ) 将以Map的方式保存于`ServerWebExchange.getAttributes()` key为`ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE`.

可以使用以下方法来更方便地访问这些变量。

```dart
Map<String, String> uriVariables = ServerWebExchangeUtils.getPathPredicateVariables(exchange);

String segment = uriVariables.get("segment");
```

## 3.9. Query

Query 路由断言 Factory 有2个参数: 必选项 `param` 和可选项 `regexp`.

```
- id: query_route
  uri: http://localhost:9001
  predicates:
  - Query=foo,bar
```

包含了请求参数 foo，且参数值为bar的将被匹配。

```shell
$ curl -s http://localhost:8080/hello?foo=bar
{"content-length":"0","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:50260\"","accept":"*/*","user-agent":"curl/7.59.0"}

$ curl -s http://localhost:8080/hello?foo=bar1
{"timestamp":"2020-08-23T10:05:20.480+0000","path":"/hello","status":404,"error":"Not Found","message":null}
```

下面的配置，只需要包含了请求参数foo就将被匹配

```
  - id: query_route2
	uri: http://localhost:9001
	predicates:
	- Query=foo
```

```shell
$ curl -s http://localhost:8080/hello?foo=bar1
{"content-length":"0","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:50301\"","accept":"*/*","user-agent":"curl/7.59.0"}
```

## 3.10. RemoteAddr

RemoteAddr 路由断言 Factory的参数为 一个CIDR符号（IPv4或IPv6）字符串的列表，最小值为1，例如192.168.0.1/16（其中192.168.0.1是IP地址并且16是子网掩码）。

```
  - id: remote_addr_route
	uri: http://localhost:9001
	predicates:
	- RemoteAddr=192.168.1.1/24
```

如果请求的remote address 为 `192.168.1.10`则将被路由

多个地址用`,`分隔

```
  - id: remote_addr_route
	uri: http://localhost:9001
	predicates:
	- RemoteAddr=192.168.1.1/24,172.16.1.1/24
```

默认情况下，RemoteAddr 路由断言 Factory使用传入请求中的remote address。如果Spring Cloud Gateway位于代理层后面，则可能与实际客户端IP地址不匹配。

可以通过设置自定义`RemoteAddressResolver`来自定义解析远程地址的方式。Spring Cloud Gateway网关附带一个非默认远程地址解析程序，它基于`X-Forwarded-For header`, `XForwardedRemoteAddressResolver`.

`XForwardedRemoteAddressResolver` 有两个静态构造函数方法，采用不同的安全方法：

1. `XForwardedRemoteAddressResolver:：TrustAll`返回一个`RemoteAddressResolver`，它始终采用`X-Forwarded-for`头中找到的第一个IP地址。这种方法容易受到欺骗，因为恶意客户端可能会为解析程序接受的“x-forwarded-for”设置初始值。
2. `XForwardedRemoteAddressResolver:：MaxTrustedIndex`获取一个 索引，该索引与在Spring  Cloud网关前运行的受信任基础设施数量相关。例如，如果SpringCloudGateway只能通过haproxy访问，则应使用值1。如果在访问 Spring Cloud Gateway之前需要两个受信任的基础架构跃点，那么应该使用2。

给定以下的header值：

```css
X-Forwarded-For: 0.0.0.1, 0.0.0.2, 0.0.0.3
```

下面的` maxTrustedIndex值将生成以下远程地址:

```
[MIN_VALUE,0] -> IllegalArgumentException
1 -> 0.0.0.3
2 -> 0.0.0.2
3 -> 0.0.0.1
[4, MAX_VALUE] -> 0.0.0.1
```

java的配置方式

```rust
RemoteAddressResolver resolver = XForwardedRemoteAddressResolver
    .maxTrustedIndex(1);

...

.route("direct-route",
    r -> r.remoteAddr("10.1.1.1", "10.10.1.1/24")
        .uri("https://downstream1")
.route("proxied-route",
    r -> r.remoteAddr(resolver,  "10.10.1.1", "10.10.1.1/24")
        .uri("https://downstream2")
)
```

> 在application.yml里无法配置RemoteAddressResolver，因为RemoteAddrRoutePredicateFactory的shortcutFieldOrder会将所有值都变成sources

```java
	@Override
	public List<String> shortcutFieldOrder() {
		return Collections.singletonList("sources");
	}
```

# 4. GatewayFilter 

过滤器允许以某种方式修改传入的HTTP请求或返回的HTTP响应。过滤器的作用域是某些特定路由。Spring Cloud Gateway包括许多内置的 Filter工厂

## 4.1. AddRequestHeader 

```
  - id: add_request_header_route
	uri: http://localhost:9001
	filters:
	- AddRequestHeader=X-Request-Foo, Bar
```

对于所有匹配的请求，这将向下游请求的头中添加 `x-request-foo:bar` header

```shell
$ curl -s http://localhost:8080/hello
{"content-length":"0","x-request-foo":"Bar","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:55740\"","accept":"*/*","user-agent":"curl/7.59.0"}

```

## 4.2. AddRequestParameter

```
  - id: add_request_param_route
	uri: http://localhost:9001
	predicates:
	- Method=GET
	filters:
	- AddRequestParameter=foo,bar
```

对于所有匹配的请求，这将向下游请求添加`foo=bar`查询字符串

## 4.3. AddResponseHeader

```
- id: add_response_param_route
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
	- AddResponseHeader=X-Response-Foo, Bar
```

```shell
$ curl -s -i http://localhost:8080/hello
HTTP/1.1 200 OK
transfer-encoding: chunked
X-Response-Foo: Bar
Content-Type: application/json;charset=UTF-8
Date: Mon, 24 Aug 2020 10:14:37 GMT
```

## 4.4. DedupeResponseHeader

消除重复的响应头

```
- id: dedupe_response_header_route
  uri: http://localhost:9001
  filters:
	- DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
	# 多个请求头用空格分隔
```

> 未测试通过

还可以配置策略strategy：RETAIN_FIRST` (default), `RETAIN_LAST，RETAIN_UNIQUE

## 4.5. Hystrix

启用Hystrix网关过滤器，需要添加对 `spring-cloud-starter-netflix-hystrix`的依赖

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

```
- id: hytrix_route
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
	- Hystrix=myCommandName
```

```
$ curl -s -i http://localhost:8080/hystrix
HTTP/1.1 504 Gateway Timeout
Content-Type: application/json;charset=UTF-8
Content-Length: 158

{"timestamp":"2020-10-17T05:59:42.110+0000","path":"/hystrix","status":504,"error":"Gateway Timeout","message":"Response took longer than configured timeout"}
```

> 通过下面的配置控制超时
>
> ```
> hystrix:
>   command:
>     fallbackcmd:
>       execution:
>         isolation:
>           thread:
>             timeoutInMilliseconds: 10000
> ```

也可以增加一个fallback

```
- id: hytrix_route
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
  - name: Hystrix
	args:
	  name: fallbackcmd
	  fallbackUri: forward:/fallback
```

当调用hystrix fallback时，这将转发到`/fallback` uri。主要场景是使用`fallbackUri` 到网关应用程序中的内部控制器或处理程序。

```
$ curl -s -i http://localhost:8080/hystrix
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Content-Length: 27

{"path":"gateway-fallback"}
```

但是，也可以将请求重新路由到外部应用程序中的控制器或处理程序，如：

```
- id: hytrix_route
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
  - name: Hystrix
	args:
	  name: fallbackcmd
	  fallbackUri: forward:/fallback
- id: fallback_route
  uri: http://localhost:9001
  predicates:
	- Path=/fallback
```

```
$ curl -s -i http://localhost:8080/hystrix
HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/json
Date: Sat, 17 Oct 2020 07:27:49 GMT

{"path":"fallback"}

```

> 需要用高版本的Spring Cloud才能转发到下游的fallback，如果遇到下面的问题，需要升级版本
>
> ```
> $ curl -s -i http://localhost:8080/hystrix
> HTTP/1.1 200 OK
> content-length: 0
> 
> ```

在将请求转发到fallback的情况下，Hystrix Gateway过滤还支持直接抛出`Throwable` 。它被作为`ServerWebExchangeUtils.HYSTRIX_EXECUTION_EXCEPTION_ATTR`属性添加到`ServerWebExchange`中，可以在处理网关应用程序中的fallback时使用。

对于外部控制器/处理程序方案，可以添加带有异常详细信息的header。可以在 `FallbackHeaders GatewayFilter Factory section`中找到有关它的更多信息。

## 4.6. CircuitBreaker

Spring Cloud的断路器支持两种 Hystrix、Resilience4J.

> Hystrix已经闭源

```
- id: circuit_breaker_route
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
	- CircuitBreaker=myCircuitBreaker
```

> 用法、参数都和Hystrix很相似

可以通过`statusCodes`只让特定的错误才触发断路器

```
- id: circuit_breaker_route
  uri: http://localhost:9001
  predicates:
    - Method=GET
  filters:
    - name: CircuitBreaker
      args:
        name: myCircuitBreaker
        fallbackUri: forward:/fallback
        statusCodes:
          - 500
          - "NOT_FOUND"
```

## 4.7. FallbackHeaders

`FallbackHeaders`允许在转发到外部应用程序中的`FallbackUri`的请求的header中添加Hystrix异常详细信息。

异常类型、消息（如果可用）cause exception类型和消息的头，将由`FallbackHeaders` filter添加到该请求中。

- `executionExceptionTypeHeaderName` (`"Execution-Exception-Type"`)
- `executionExceptionMessageHeaderName` (`"Execution-Exception-Message"`)
- `rootCauseExceptionTypeHeaderName` (`"Root-Cause-Exception-Type"`)
- `rootCauseExceptionMessageHeaderName` (`"Root-Cause-Exception-Message"`)

> 未测试通过

## 4.8. MapRequestHeader

将请求的请求头映射为其他的请求头，只有请求中存在要映射的请求头才会生效

> 不会删除原来的请求头

```
- id: map_request_header_route
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
	- MapRequestHeader=Blue, X-Request-Red
```



````
$  curl -s -i http://localhost:8080/hello
HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/json
Date: Tue, 20 Oct 2020 11:02:00 GMT

{"content-length":"0","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:58181\"","user-agent":"curl/7.71.1","accept":"*/*"}

$  curl -H 'Blue:xxx' -s -i http://localhost:8080/hello
HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/json
Date: Tue, 20 Oct 2020 11:02:19 GMT

{"content-length":"0","blue":"xxx","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","x-request-red":"xxx","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:58188\"","user-agent":"curl/7.71.1","accept":"*/*"}
````

## 4.9. PrefixPath

这将给所有匹配请求的路径加前缀`/mypath`。因此，向`/hello`发送的请求将发送到`/mypath/hello`。

```
- id: prefixpath_route
  uri: http://localhost:9001
  filters:
	- PrefixPath=/mypath
```

## 4.10. PreserveHostHeader

GatewayFilter将不使用由HTTP客户端确定的host header ，而是发送原始host header 

```
- id: preserve_host_route
  uri: ttp://localhost:9001
  filters:
	- PreserveHostHeader
```

## 4.11. RedirectTo

该过滤器有一个 `status` 和一个 `url`参数。status是300类重定向HTTP代码，如301。该URL应为有效的URL，这将是 `Location` header的值。

```
  - id: redirect_route
	uri: http://localhost:9001
	predicates:
	  - Method=GET
	filters:
	  - RedirectTo=302, http://localhost:9002
```

```
$  curl -s -i http://localhost:8080/hello
HTTP/1.1 302 Found
Location: http://localhost:9002
content-length: 0
```

## 4.12. RemoveRequestHeader

请求被发到下游时删除请求头，有一个`name`参数. 这是要删除的header的名称

```
  - id: removerequestheader_route
	uri: http://localhost:9001
	predicates:
	  - Method=GET
	filters:
	  - RemoveRequestHeader=X-Request-Foo
```

## 4.13. RemoveResponseHeader

在返回到网关client之前从响应中删除响应头

```
   - id: removeresponseheader_route
	  uri: http://localhost:9001
	  predicates:
		- Method=GET
	  filters:
		- RemoveResponseHeader=X-Response-Foo
```

## 4.14. RemoveRequestParameter

删除查询字符串

```
  - id: removerequestparameter_route
	uri: https://example.org
	filters:
	- RemoveRequestParameter=red
```

## 4.15. RewritePath

包含一个 `regexp`正则表达式参数和一个 `replacement` 参数. 通过使用Java正则表达式灵活地重写请求路径。

对于请求路径`/red/hello`，将在发出下游请求之前将路径设置为`/hello。注意,由于YAML规范，请使用 `$\`替换 `$`。

```
- id: rewritepath_route
  uri: http://localhost:9001
  predicates:
	- Path=/red/**
  filters:
	- RewritePath=/red(?<segment>/?.*), $\{segment}
```

## 4.16. RewriteLocationResponseHeader

## 4.17. RewriteResponseHeader

包含 `name`, `regexp`和 `replacement` 参数.。通过使用Java正则表达式灵活地重写响应头的值

```
  - id: rewriteresponseheader_route
	uri: http://localhost:9001
	filters:
	- RewriteResponseHeader=X-Response-Red, , password=[^&]+, password=***
```

对于一个`/42?user=ford&password=omg!what&flag=true`的header值，在做下游请求时将被设置为`/42?user=ford&password=***&flag=true`，由于YAML规范，请使用 `$\`替换 `$`

> 未测试通过

## 4.18. SaveSession

SaveSession GatewayFilter Factory将调用转发到下游之前强制执行WebSession::save 操作。这在使用 Spring Session 之类时特别有用，需要确保会话状态在进行转发调用之前已保存。

```
  - id: save_session
	uri: http://localhost:9001
	predicates:
	- Path=/foo/**
	filters:
	- SaveSession
```

> 未测试

## 4.19. SecureHeaders

SecureHeaders GatewayFilter Factory 将许多headers添加到reccomedation处的响应中

## 4.20. SetPath

SetPath GatewayFilter Factory 采用 `template`路径参数。它提供了一种通过允许路径的模板化segments来操作请求路径的简单方法。使用Spring Framework中的URI模板，允许多个匹配segments。

```
  - id: setpath_route
	uri: https://example.org
	predicates:
	- Path=/red/{segment}
	filters:
	- SetPath=/{segment}
```

对于一个 `/foo/bar`请求，在做下游请求前，路径将被设置为`/bar`

## 4.21. SetRequestHeader

设置请求头

```
- id: setrequestheader_route
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
	- SetRequestHeader=X-Request-Red, Blue
```

想下游发送请求时增加请求头`X-Request-Red:Blue`,即使请求网关时传入了`X-Request-Red`也会被替换

## 4.22. SetResponseHeader

设置响应头，

```
  - id: setresponseheader_route
	uri: http://localhost:9001
	predicates:
	  - Method=GET
	filters:
	  - SetResponseHeader=X-Request-Red, Blue
```

## 4.23. SetStatus

指定返回的响应码

```
  - id: setstatusint_route
	uri: http://localhost:9001
	predicates:
	  - Method=GET
	filters:
	  - SetStatus=401
```

## 4.24. StripPrefix

包括一个`parts`参数。 `parts`参数指示在将请求发送到下游之前，要从请求中去除的路径中的节数。

```
  - id: nameRoot
	uri: http://localhost:9001
	predicates:
	  - Path=/name/**
	filters:
	  - StripPrefix=2
```

当通过网关发出`/name/bar/foo`请求时，向下游发出的请求将是`http://localhost:9001/foo`

## 4.25. Retry GatewayFilter

参数

`retries`: 应尝试的重试次数

`statuses`: 应该重试的HTTP状态代码，用`org.springframework.http.HttpStatus`标识

`methods`: 应该重试的HTTP方法，用 `org.springframework.http.HttpMethod`标识

`series`: 要重试的一系列状态码，用 `org.springframework.http.HttpStatus.Series`标识

`exceptions`: 应该重试的异常.

`backoff`: The configured exponential backoff for the retries. Retries are performed after a backoff interval of `firstBackoff * (factor ^ n)`, where `n` is the iteration. If `maxBackoff` is configured, the maximum backoff applied is limited to `maxBackoff`. If `basedOnPreviousValue` is true, the backoff is calculated byusing `prevBackoff * factor`.

默认值

- `retries`: Three times
- `series`: 5XX series
- `methods`: GET method
- `exceptions`: `IOException` and `TimeoutException`
- `backoff`: disabled

```
- id: retry_test
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
	- name: Retry
	  args:
		retries: 3
		statuses: INTERNAL_SERVER_ERROR
		methods: GET,POST
		backoff:
		  firstBackoff: 10ms
		  maxBackoff: 50ms
		  factor: 2
		  basedOnPreviousValue: false
```

## 4.26. RequestSize

当请求大小大于允许的限制时，RequestSize GatewayFilter Factory可以限制请求不到达下游服务。过滤器以`RequestSize`作为参数，这是定义请求的允许大小限制(以字节为单位)

```
  - id: request_size_route
	uri: http://localhost:8080/upload
	predicates:
	- Path=/upload
	filters:
	- name: RequestSize
	  args:
		maxSize: 5000000
```

当请求因大小而被拒绝时， RequestSize GatewayFilter Factory 将响应状态设置为`413 Payload Too Large`，并带有额外的header `errorMessage` 。下面是一个 `errorMessage`的例子。

```
errorMessage` : `Request size is larger than permissible limit. Request size is 6.0 MB where permissible limit is 5.0 MB
```

## 4.27. SetRequestHost

设置请求的host请求头

```
  - id: set_request_host_header_route
	uri: http://localhost:8080
	predicates:
	  - Method=GET
	filters:
	  - name: SetRequestHostHeader
		args:
		  host: example.org
```

## 4.28. ModifyRequestBody

在请求主体被网关发送到下游之前对其进行修改。

> 只能使用Java DSL配置此过滤器

```
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
	return builder.routes()
			.route("rewrite_request_obj", r -> r.method("POST")
					.filters(f -> f.modifyRequestBody(String.class, Hello.class, MediaType.APPLICATION_JSON_VALUE,
							(serverWebExchange, s) -> {
								return Mono.just(new Hello(s.toUpperCase()));
							})).uri("http://localhost:9001"))
			.build();
}

static class Hello {
	String message;

	public Hello() {
	}

	public Hello(String message) {
		this.message = message;
	}

	public String getMessage() {
		return message;
	}

	public void setMessage(String message) {
		this.message = message;
	}
}
```

## 4.29. ModifyResponseBody

可用于在将响应正文发送回客户端之前对其进行修改

> 只能使用Java DSL配置此过滤器

```
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
	return builder.routes()
			.route("rewrite_request_obj", r -> r.method("GET")
					.filters(f -> f.modifyResponseBody(String.class, String.class, MediaType.APPLICATION_JSON_VALUE,
							(serverWebExchange, s) -> {
								return Mono.just(s.toUpperCase());
							})).uri("http://localhost:9001"))
			.build();
}
```

## 4.30. RequestRateLimiter

## 4.31 Default Filters

可以通过`spring.cloud.gateway.default-filters`指定默认filter，对于所有的路由都生效

```
spring:
  cloud:
    gateway:
      default-filters:
      - AddResponseHeader=X-Response-Default-Red, Default-Blue
      - PrefixPath=/httpbin
```

# 5. Global Filters

`GlobalFilter`接口与`GatewayFilter`具有相同的签名。是有条件地应用于所有路由的特殊过滤器。

## 5.1. 全局Filter和GatewayFilter组合排序

当请求进入（并与路由匹配）时，筛选Web Handler 会将`GlobalFilter`的所有实例和所有的`GatewayFilter`路由特定实例添加到 filter chain。filter组合的排序由`org.springframework.core.Ordered`接口决定，可以通过实现`getOrder()`方法或使用`@Order`注释来设置。

由于Spring Cloud Gateway将用于执行过滤器逻辑区分为“前置”和“后置”阶段，具有最高优先级的过滤器将是“前置”阶段的第一个，而“后置”阶段的最后一个。



# 参考资料

https://www.jianshu.com/p/6ff196940b67

https://docs.spring.io/spring-cloud-gateway/docs/2.2.5.RELEASE/reference/html