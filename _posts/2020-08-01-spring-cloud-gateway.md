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

## 3.1. After

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

## 3.3. Between

Between ·�ɶ��� Factory������������datetime1��datetime2����datetime1��datetime2֮������󽫱�ƥ�䡣datetime2������ʵ��ʱ�������datetime1֮��

```
- id: between_route
  uri: http://localhost:9001
  predicates:
  - Between=2020-08-22T17:30:00.843+08:00[Asia/Shanghai],2020-08-30T17:30:00.843+08:00[Asia/Shanghai]
```

## 3.4. Cookie

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

## 3.5. Header

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

## 3.6. Host

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

## 3.7. Method

Method ·�ɶ��� Factoryֻ����һ������: ��Ҫƥ���HTTP����ʽ

```
  - id: method_route
	uri: http://localhost:9001
	predicates:
	- Method=GET
```

## 3.8. Path

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

## 3.9. Query

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

## 3.10. RemoteAddr

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

## 4.4. DedupeResponseHeader

�����ظ�����Ӧͷ

```
- id: dedupe_response_header_route
  uri: http://localhost:9001
  filters:
	- DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
	# �������ͷ�ÿո�ָ�
```

> δ����ͨ��

���������ò���strategy��RETAIN_FIRST` (default), `RETAIN_LAST��RETAIN_UNIQUE

## 4.5. Hystrix

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

## 4.6. CircuitBreaker

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

����ͨ��`statusCodes`ֻ���ض��Ĵ���Ŵ�����·��

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

`FallbackHeaders`������ת�����ⲿӦ�ó����е�`FallbackUri`�������header�����Hystrix�쳣��ϸ��Ϣ��

�쳣���͡���Ϣ��������ã�cause exception���ͺ���Ϣ��ͷ������`FallbackHeaders` filter��ӵ��������С�

- `executionExceptionTypeHeaderName` (`"Execution-Exception-Type"`)
- `executionExceptionMessageHeaderName` (`"Execution-Exception-Message"`)
- `rootCauseExceptionTypeHeaderName` (`"Root-Cause-Exception-Type"`)
- `rootCauseExceptionMessageHeaderName` (`"Root-Cause-Exception-Message"`)

> δ����ͨ��

## 4.8. MapRequestHeader

�����������ͷӳ��Ϊ����������ͷ��ֻ�������д���Ҫӳ�������ͷ�Ż���Ч

> ����ɾ��ԭ��������ͷ

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

�⽫������ƥ�������·����ǰ׺`/mypath`����ˣ���`/hello`���͵����󽫷��͵�`/mypath/hello`��

```
- id: prefixpath_route
  uri: http://localhost:9001
  filters:
	- PrefixPath=/mypath
```

## 4.10. PreserveHostHeader

GatewayFilter����ʹ����HTTP�ͻ���ȷ����host header �����Ƿ���ԭʼhost header 

```
- id: preserve_host_route
  uri: ttp://localhost:9001
  filters:
	- PreserveHostHeader
```

## 4.11. RedirectTo

�ù�������һ�� `status` ��һ�� `url`������status��300���ض���HTTP���룬��301����URLӦΪ��Ч��URL���⽫�� `Location` header��ֵ��

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

���󱻷�������ʱɾ������ͷ����һ��`name`����. ����Ҫɾ����header������

```
  - id: removerequestheader_route
	uri: http://localhost:9001
	predicates:
	  - Method=GET
	filters:
	  - RemoveRequestHeader=X-Request-Foo
```

## 4.13. RemoveResponseHeader

�ڷ��ص�����client֮ǰ����Ӧ��ɾ����Ӧͷ

```
   - id: removeresponseheader_route
	  uri: http://localhost:9001
	  predicates:
		- Method=GET
	  filters:
		- RemoveResponseHeader=X-Response-Foo
```

## 4.14. RemoveRequestParameter

ɾ����ѯ�ַ���

```
  - id: removerequestparameter_route
	uri: https://example.org
	filters:
	- RemoveRequestParameter=red
```

## 4.15. RewritePath

����һ�� `regexp`������ʽ������һ�� `replacement` ����. ͨ��ʹ��Java������ʽ������д����·����

��������·��`/red/hello`�����ڷ�����������֮ǰ��·������Ϊ`/hello��ע��,����YAML�淶����ʹ�� `$\`�滻 `$`��

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

���� `name`, `regexp`�� `replacement` ����.��ͨ��ʹ��Java������ʽ������д��Ӧͷ��ֵ

```
  - id: rewriteresponseheader_route
	uri: http://localhost:9001
	filters:
	- RewriteResponseHeader=X-Response-Red, , password=[^&]+, password=***
```

����һ��`/42?user=ford&password=omg!what&flag=true`��headerֵ��������������ʱ��������Ϊ`/42?user=ford&password=***&flag=true`������YAML�淶����ʹ�� `$\`�滻 `$`

> δ����ͨ��

## 4.18. SaveSession

SaveSession GatewayFilter Factory������ת��������֮ǰǿ��ִ��WebSession::save ����������ʹ�� Spring Session ֮��ʱ�ر����ã���Ҫȷ���Ự״̬�ڽ���ת������֮ǰ�ѱ��档

```
  - id: save_session
	uri: http://localhost:9001
	predicates:
	- Path=/foo/**
	filters:
	- SaveSession
```

> δ����

## 4.19. SecureHeaders

SecureHeaders GatewayFilter Factory �����headers��ӵ�reccomedation������Ӧ��

## 4.20. SetPath

SetPath GatewayFilter Factory ���� `template`·�����������ṩ��һ��ͨ������·����ģ�廯segments����������·���ļ򵥷�����ʹ��Spring Framework�е�URIģ�壬������ƥ��segments��

```
  - id: setpath_route
	uri: https://example.org
	predicates:
	- Path=/red/{segment}
	filters:
	- SetPath=/{segment}
```

����һ�� `/foo/bar`����������������ǰ��·����������Ϊ`/bar`

## 4.21. SetRequestHeader

��������ͷ

```
- id: setrequestheader_route
  uri: http://localhost:9001
  predicates:
	- Method=GET
  filters:
	- SetRequestHeader=X-Request-Red, Blue
```

�����η�������ʱ��������ͷ`X-Request-Red:Blue`,��ʹ��������ʱ������`X-Request-Red`Ҳ�ᱻ�滻

## 4.22. SetResponseHeader

������Ӧͷ��

```
  - id: setresponseheader_route
	uri: http://localhost:9001
	predicates:
	  - Method=GET
	filters:
	  - SetResponseHeader=X-Request-Red, Blue
```

## 4.23. SetStatus

ָ�����ص���Ӧ��

```
  - id: setstatusint_route
	uri: http://localhost:9001
	predicates:
	  - Method=GET
	filters:
	  - SetStatus=401
```

## 4.24. StripPrefix

����һ��`parts`������ `parts`����ָʾ�ڽ������͵�����֮ǰ��Ҫ��������ȥ����·���еĽ�����

```
  - id: nameRoot
	uri: http://localhost:9001
	predicates:
	  - Path=/name/**
	filters:
	  - StripPrefix=2
```

��ͨ�����ط���`/name/bar/foo`����ʱ�������η�����������`http://localhost:9001/foo`

## 4.25. Retry GatewayFilter

����

`retries`: Ӧ���Ե����Դ���

`statuses`: Ӧ�����Ե�HTTP״̬���룬��`org.springframework.http.HttpStatus`��ʶ

`methods`: Ӧ�����Ե�HTTP�������� `org.springframework.http.HttpMethod`��ʶ

`series`: Ҫ���Ե�һϵ��״̬�룬�� `org.springframework.http.HttpStatus.Series`��ʶ

`exceptions`: Ӧ�����Ե��쳣.

`backoff`: The configured exponential backoff for the retries. Retries are performed after a backoff interval of `firstBackoff * (factor ^ n)`, where `n` is the iteration. If `maxBackoff` is configured, the maximum backoff applied is limited to `maxBackoff`. If `basedOnPreviousValue` is true, the backoff is calculated byusing `prevBackoff * factor`.

Ĭ��ֵ

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

�������С�������������ʱ��RequestSize GatewayFilter Factory�����������󲻵������η��񡣹�������`RequestSize`��Ϊ���������Ƕ�������������С����(���ֽ�Ϊ��λ)

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

���������С�����ܾ�ʱ�� RequestSize GatewayFilter Factory ����Ӧ״̬����Ϊ`413 Payload Too Large`�������ж����header `errorMessage` ��������һ�� `errorMessage`�����ӡ�

```
errorMessage` : `Request size is larger than permissible limit. Request size is 6.0 MB where permissible limit is 5.0 MB
```

## 4.27. SetRequestHost

���������host����ͷ

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

���������屻���ط��͵�����֮ǰ��������޸ġ�

> ֻ��ʹ��Java DSL���ô˹�����

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

�������ڽ���Ӧ���ķ��ͻؿͻ���֮ǰ��������޸�

> ֻ��ʹ��Java DSL���ô˹�����

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

����ͨ��`spring.cloud.gateway.default-filters`ָ��Ĭ��filter���������е�·�ɶ���Ч

```
spring:
  cloud:
    gateway:
      default-filters:
      - AddResponseHeader=X-Response-Default-Red, Default-Blue
      - PrefixPath=/httpbin
```

# 5. Global Filters

`GlobalFilter`�ӿ���`GatewayFilter`������ͬ��ǩ��������������Ӧ��������·�ɵ������������

## 5.1. ȫ��Filter��GatewayFilter�������

��������루����·��ƥ�䣩ʱ��ɸѡWeb Handler �Ὣ`GlobalFilter`������ʵ�������е�`GatewayFilter`·���ض�ʵ����ӵ� filter chain��filter��ϵ�������`org.springframework.core.Ordered`�ӿھ���������ͨ��ʵ��`getOrder()`������ʹ��`@Order`ע�������á�

����Spring Cloud Gateway������ִ�й������߼�����Ϊ��ǰ�á��͡����á��׶Σ�����������ȼ��Ĺ��������ǡ�ǰ�á��׶εĵ�һ�����������á��׶ε����һ����



# �ο�����

https://www.jianshu.com/p/6ff196940b67

https://docs.spring.io/spring-cloud-gateway/docs/2.2.5.RELEASE/reference/html