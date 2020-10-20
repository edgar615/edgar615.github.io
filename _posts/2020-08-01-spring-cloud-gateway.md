---
layout: post
title: Spring Cloud Gateway
date: 2020-08-01
categories:
    - Spring Cloud
comments: true
permalink: spring-cloud-gateway.html
---

# 1. ��������

## 1.1. �������

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

## 1.2. ���һ��Route

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

���������ص�`/hello`ʱ�����ػὫ����ת����`http://localhost:9001/hello`�����������ͷ`Hello:World`��

```
$ curl -s http://localhost:8080/hello
{"content-length":"0","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","hello":"World","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:60324\"","accept":"*/*","user-agent":"curl/7.59.0"}
```

## 1. 3. ʹ��Hystrix

�������

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

����route

```java
.route(p -> p.path("/hystrix")
		.filters(f -> f.hystrix(config -> config.setName("hystrix-test")))
		.uri("http://localhost:9001"))
```

����һ������`hystric`api

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

���Կ�����Hystrix�ڳ�ʱ�󷵻���504�Ĵ���

**����**

��gateway������һ�������Ľӿ�

```java
@RequestMapping("/fallback")
public Mono<Map<String, String>> fallback() {
    return Mono.just(Collections.singletonMap("path", "fallback"));
}
```

��·��`/hystrix`������һ����������

```java
.route(p -> p.path("/hystrix")
       .filters(f -> f.hystrix(config -> config.setName("hystrix-test").
                               setFallbackUri("forward:/fallback")))
       .uri("http://localhost:9001"))
```

���Կ������صĲ�����504

```shell
$ curl  --dump-header -  -s http://localhost:8080/hystrix
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Content-Length: 19

{"path":"fallback"}
```

## 1.4. ��Ԫ����

Ϊ��ʵ�ֵ�Ԫ���ԣ���������Ҫ��Gateway��·�������죬���ȶ���һ��`UriConfiguration`���ڶ�̬�޸����η���ĵ�ַ

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

`GatewayApplication`����ע��`@EnableConfigurationProperties(UriConfiguration.class)`��������`ConfigurationProperties`

��Route���и���

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

��д��Ԫ����

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

`@AutoConfigureWireMock(port = 0)`ע������ʹ��һ������˿�����WireMock ��



# 2. ����

## 2.1. �ʻ��

- Route ·�ɣ�gateway�Ļ�������ģ�顣����ID��Ŀ��URI�����Լ��Ϻ͹�����������ɡ�����ۺ϶��Խ��Ϊ�棬��ƥ�䵽��·�ɡ�
- Predicate ���ԣ�����һ��Java 8 Function Predicate������������ Spring Framework ServerWebExchange������������Ա����ƥ������HTTP������κ����ݣ�����Header�������
- Filter ����������Щ��ʹ���ض����������� Spring FrameworkGatewayFilterʵ�������Կ����ڷ�������֮ǰ��֮���޸��������Ӧ�����ݡ�

## 2.2. ��������

![](/assets/images/posts/spring-cloud-gateway/spring-cloud-gateway-1.png)

�ͻ�����Spring Cloud Gateway�����������Gateway Handler  Mappingȷ��������·��ƥ�䣬���䷢�͵�Gateway Web  Handler����handlerͨ���ض��ڸ�����Ĺ���������������

ͼ��filters�����߻��ֵ�ԭ����filters�����ڷ��ʹ�������֮ǰ��֮ ��ִ���߼�����ִ�����С�pre filter���߼���Ȼ���������������������ִ�����ִ�С�post filter���߼���

# 3. ·�ɶ���

Spring Cloud Gateway��·����ΪSpring WebFlux `HandlerMapping`�����ṹ��һ���ֽ���ƥ�䡣Spring Cloud Gateway����������õ�·�ɶ���Factories����Щ���Զ�ƥ��HTTP����Ĳ�ͬ���ԡ����·�ɶ���Factories����ͨ�� `and` ���ʹ�á�

## 3.1. After·�ɶ���

After Route Predicate Factory����һ��������������ʱ�䡣�ڸ�����ʱ��֮���������󶼽���ƥ�䡣

ʱ��`ISO-8601`��ʽ������`2007-12-03T10:15:30+01:00 Europe/Paris`, ����`ZonedDateTime`ʵ��

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

���û��ƥ�䵽After������404

## 3.3. Between ·�ɶ���

Between ·�ɶ��� Factory������������datetime1��datetime2����datetime1��datetime2֮������󽫱�ƥ�䡣datetime2������ʵ��ʱ�������datetime1֮��

```
- id: between_route
  uri: http://localhost:9001
  predicates:
  - Between=2020-08-22T17:30:00.843+08:00[Asia/Shanghai],2020-08-30T17:30:00.843+08:00[Asia/Shanghai]
```

## 3.4. Cookie ·�ɶ���

Cookie ·�ɶ��� Factory������������cookie���ƺ�������ʽ�����������cookie������������ʽΪ��Ľ��ᱻƥ�䡣

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

ʹ��������ʽ

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

## 3.5. Header ·�ɶ���

Header ·�ɶ��� Factory������������header���ƺ�������ʽ�����������header������������ʽΪ��Ľ��ᱻƥ�䡣

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

## 3.6. Host ·�ɶ���

Host ·�ɶ��� Factory����һ��������host name�б�ʹ��Ant·��ƥ�����`.`��Ϊ�ָ�����

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

## 3.7. Method ·�ɶ���

Method ·�ɶ��� Factoryֻ����һ������: ��Ҫƥ���HTTP����ʽ

```
  - id: method_route
	uri: http://localhost:9001
	predicates:
	- Method=GET
```

## 3.8. Path ·�ɶ���

Path ·�ɶ��� Factory ��2������: һ��Spring `PathMatcher`���ʽ�б�Ϳ�ѡ`matchOptionalTrailingSeparator`��ʶ .

```
- id: path_route
  uri: http://localhost:9001
  predicates:
  - Path=/foo/{segment},/bar/{segment}
```

����: `/foo/1` or `/foo/bar` or `/bar/baz`�����󶼽���ƥ��

URI ģ����� (�������е� `segment` ) ����Map�ķ�ʽ������`ServerWebExchange.getAttributes()` keyΪ`ServerWebExchangeUtils.URI_TEMPLATE_VARIABLES_ATTRIBUTE`.

����ʹ�����·�����������ط�����Щ������

```dart
Map<String, String> uriVariables = ServerWebExchangeUtils.getPathPredicateVariables(exchange);

String segment = uriVariables.get("segment");
```

## 3.9. Query ·�ɶ���

Query ·�ɶ��� Factory ��2������: ��ѡ�� `param` �Ϳ�ѡ�� `regexp`.

```
- id: query_route
  uri: http://localhost:9001
  predicates:
  - Query=foo,bar
```

������������� foo���Ҳ���ֵΪbar�Ľ���ƥ�䡣

```shell
$ curl -s http://localhost:8080/hello?foo=bar
{"content-length":"0","x-forwarded-proto":"http","x-forwarded-host":"localhost:8080","host":"localhost:9001","x-forwarded-port":"8080","x-forwarded-for":"0:0:0:0:0:0:0:1","forwarded":"proto=http;host=\"localhost:8080\";for=\"0:0:0:0:0:0:0:1:50260\"","accept":"*/*","user-agent":"curl/7.59.0"}

$ curl -s http://localhost:8080/hello?foo=bar1
{"timestamp":"2020-08-23T10:05:20.480+0000","path":"/hello","status":404,"error":"Not Found","message":null}
```

��������ã�ֻ��Ҫ�������������foo�ͽ���ƥ��

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

## 3.10. RemoteAddr ·�ɶ���

RemoteAddr ·�ɶ��� Factory�Ĳ���Ϊ һ��CIDR���ţ�IPv4��IPv6���ַ������б���СֵΪ1������192.168.0.1/16������192.168.0.1��IP��ַ����16���������룩��

```
  - id: remote_addr_route
	uri: http://localhost:9001
	predicates:
	- RemoteAddr=192.168.1.1/24
```

��������remote address Ϊ `192.168.1.10`�򽫱�·��

�����ַ��`,`�ָ�

```
  - id: remote_addr_route
	uri: http://localhost:9001
	predicates:
	- RemoteAddr=192.168.1.1/24,172.16.1.1/24
```

Ĭ������£�RemoteAddr ·�ɶ��� Factoryʹ�ô��������е�remote address�����Spring Cloud Gatewayλ�ڴ������棬�������ʵ�ʿͻ���IP��ַ��ƥ�䡣

����ͨ�������Զ���`RemoteAddressResolver`���Զ������Զ�̵�ַ�ķ�ʽ��Spring Cloud Gateway���ظ���һ����Ĭ��Զ�̵�ַ��������������`X-Forwarded-For header`, `XForwardedRemoteAddressResolver`.

`XForwardedRemoteAddressResolver` ��������̬���캯�����������ò�ͬ�İ�ȫ������

1. `XForwardedRemoteAddressResolver:��TrustAll`����һ��`RemoteAddressResolver`����ʼ�ղ���`X-Forwarded-for`ͷ���ҵ��ĵ�һ��IP��ַ�����ַ��������ܵ���ƭ����Ϊ����ͻ��˿��ܻ�Ϊ����������ܵġ�x-forwarded-for�����ó�ʼֵ��
2. `XForwardedRemoteAddressResolver:��MaxTrustedIndex`��ȡһ�� ����������������Spring  Cloud����ǰ���е������λ�����ʩ������ء����磬���SpringCloudGatewayֻ��ͨ��haproxy���ʣ���Ӧʹ��ֵ1������ڷ��� Spring Cloud Gateway֮ǰ��Ҫ���������εĻ����ܹ�Ծ�㣬��ôӦ��ʹ��2��

�������µ�headerֵ��

```css
X-Forwarded-For: 0.0.0.1, 0.0.0.2, 0.0.0.3
```

�����` maxTrustedIndexֵ����������Զ�̵�ַ:

```
[MIN_VALUE,0] -> IllegalArgumentException
1 -> 0.0.0.3
2 -> 0.0.0.2
3 -> 0.0.0.1
[4, MAX_VALUE] -> 0.0.0.1
```

java�����÷�ʽ

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

> ��application.yml���޷�����RemoteAddressResolver����ΪRemoteAddrRoutePredicateFactory��shortcutFieldOrder�Ὣ����ֵ�����sources

```java
	@Override
	public List<String> shortcutFieldOrder() {
		return Collections.singletonList("sources");
	}
```

# 4. GatewayFilter 

������������ĳ�ַ�ʽ�޸Ĵ����HTTP����򷵻ص�HTTP��Ӧ������������������ĳЩ�ض�·�ɡ�Spring Cloud Gateway����������õ� Filter����

## 4.1. AddRequestHeader 

```
  - id: add_request_header_route
	uri: http://localhost:9001
	filters:
	- AddRequestHeader=X-Request-Foo, Bar
```

��������ƥ��������⽫�����������ͷ����� `x-request-foo:bar` header

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

��������ƥ��������⽫�������������`foo=bar`��ѯ�ַ���

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

## 4.4. Hystrix GatewayFilter

����Hystrix���ع���������Ҫ��Ӷ� `spring-cloud-starter-netflix-hystrix`������

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

> ͨ����������ÿ��Ƴ�ʱ
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

Ҳ��������һ��fallback

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

������hystrix fallbackʱ���⽫ת����`/fallback` uri����Ҫ������ʹ��`fallbackUri` ������Ӧ�ó����е��ڲ��������������

```
$ curl -s -i http://localhost:8080/hystrix
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Content-Length: 27

{"path":"gateway-fallback"}
```

���ǣ�Ҳ���Խ���������·�ɵ��ⲿӦ�ó����еĿ�������������磺

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

> ��Ҫ�ø߰汾��Spring Cloud����ת�������ε�fallback�����������������⣬��Ҫ�����汾
>
> ```
> $ curl -s -i http://localhost:8080/hystrix
> HTTP/1.1 200 OK
> content-length: 0
> 
> ```



�ڽ�����ת����fallback������£�Hystrix Gateway���˻�֧��ֱ���׳�`Throwable` ��������Ϊ`ServerWebExchangeUtils.HYSTRIX_EXECUTION_EXCEPTION_ATTR`������ӵ�`ServerWebExchange`�У������ڴ�������Ӧ�ó����е�fallbackʱʹ�á�

�����ⲿ������/������򷽰���������Ӵ����쳣��ϸ��Ϣ��header�������� `FallbackHeaders GatewayFilter Factory section`���ҵ��й����ĸ�����Ϣ��

## 4.6. CircuitBreaker GatewayFilter

Spring Cloud�Ķ�·��֧������ Hystrix��Resilience4J.

> Hystrix�Ѿ���Դ

```
- id: circuit_breaker_route
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
	- CircuitBreaker=myCircuitBreaker
```

> �÷�����������Hystrix������



## 4.5. FallbackHeaders GatewayFilter

`FallbackHeaders`������ת�����ⲿӦ�ó����е�`FallbackUri`�������header�����Hystrix�쳣��ϸ��Ϣ



# �ο�����

