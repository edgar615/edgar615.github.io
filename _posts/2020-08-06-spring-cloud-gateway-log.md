---
layout: post
title: Spring Cloud Gateway - 日志打印
date: 2020-08-06
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-log.html
---

# 1. accessLog

spring cloud gateway底层的Reactor Netty提供了accesslog，在应用启动命名中增加设置`-Dreactor.netty.http.server.accessLogEnabled=true`来开启。因为Reactor Netty不是基于spring boot的，所以它并不会去spring boot的配置中获取上面的配置，所以只能在Java System Property中获取。

可以在常用的日志系统中配置日志的打印文件和格式，如logback的配置：

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="accessLog" class="ch.qos.logback.core.FileAppender">
        <file>access_log.log</file>
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>
    <appender name="async" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="accessLog" />
    </appender>

    <logger name="reactor.netty.http.server.AccessLog" level="INFO" additivity="false">
        <appender-ref ref="async"/>
    </logger>

</configuration>
```

请求日志如下

```
0:0:0:0:0:0:0:1 - - [14/Nov/2020:17:29:58 +0800] "GET / HTTP/1.1" 200 9593 8080 7447 ms
```

查看源码得到具体的日志内容是 `调用方IP - 用户 - 时间 - "请求方法 请求路径 protocol" 响应码 响应体长度 端口 耗时`

```
log.info("{} - {} [{}] \"{} {} {}\" {} {} {} {} ms", new Object[]{this.address, this.user, this.zonedDateTime, this.method, this.uri, this.protocol, this.status, this.contentLength > -1L ? this.contentLength : "-", this.port, this.duration()});
```

# 2. 自己实现日志

我们可以自己实现一个Filter来记录日志，但是需要注意默认请求体通常情况下在被读取一次之后就会失效，这样的话，下游的服务就不能正常获取到请求参数了。所以我们需要重写下请求体。

然而`request.getBody()`方法返回的是一个`Flux<DataBuffer>`，即一个包含 0-N 个`DataBuffer`类型元素的异步序列。

## 2.1. 全局requestId

```
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String correlationId = UUID.randomUUID().toString();
        // 设置X-Request-Id
        ServerHttpRequest request = exchange.getRequest().mutate().headers(httpHeaders -> {
            httpHeaders.set("X-Request-Id", correlationId);
        }).build();

        return chain.filter(exchange.mutate().request(request).build());
    }

}
```

## 2.2. 缓存reqBody

```
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 1)
public class CachedBodyFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 获取用户传来的数据类型
        MediaType mediaType = exchange.getRequest().getHeaders().getContentType();
        // JSON类型
        if (MediaType.APPLICATION_JSON.isCompatibleWith(mediaType)) {
            return DataBufferUtils.join(exchange.getRequest().getBody())
                    .flatMap(dataBuffer -> {
                        DataBufferUtils.retain(dataBuffer);
                        Flux<DataBuffer> cachedFlux = Flux
                                .defer(() -> Flux.just(dataBuffer.slice(0, dataBuffer.readableByteCount())));
                        ServerHttpRequest request = new ServerHttpRequestDecorator(exchange.getRequest()) {
                            @Override
                            public Flux<DataBuffer> getBody() {
                                return cachedFlux;
                            }
                        };
                        return chain.filter(exchange.mutate().request(request).build());
                    });
        }
        // 表单请求
        if (MediaType.APPLICATION_FORM_URLENCODED.isCompatibleWith(mediaType)) {

        }
        return chain.filter(exchange);

    }

}
```

另一种获取请求体的方法

```
DataBufferUtils.retain(dataBuffer);
Flux<DataBuffer> cachedFlux = Flux.defer(() -> Flux.just(dataBuffer.slice(0, dataBuffer.readableByteCount())));

String body = toRaw(cachedFlux);
private static String toRaw(Flux<DataBuffer> body) {
AtomicReference<String> rawRef = new AtomicReference<>();
body.subscribe(buffer -> {
	byte[] bytes = new byte[buffer.readableByteCount()];
	buffer.read(bytes);
	DataBufferUtils.release(buffer);
	rawRef.set(Strings.fromUTF8ByteArray(bytes));
});
return rawRef.get();
}
```



## 2.3. 打印日志

```
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 5)
public class LogFilter implements GlobalFilter {

    private static final Logger LOGGER = LoggerFactory.getLogger(LogFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

        ServerHttpRequest request = exchange.getRequest();
        String method = request.getMethod().name();
        String url = request.getPath().value();
        String requestId = request.getHeaders().getFirst("X-Request-Id");
        Flux<DataBuffer> body = request.getBody();
       return DataBufferUtils.join(body)
                .flatMap(buffer -> {
                    byte[] bytes = new byte[buffer.readableByteCount()];
                    buffer.read(bytes);
                    // 这行代码有篇文章说不需要，保留也没有问题，后面再深入研究
	            	DataBufferUtils.release(buffer);
                    String bodyString = new String(bytes, StandardCharsets.UTF_8);
                    LOGGER.info("{} {} {} {}", method, url, bodyString);
                    return chain.filter(exchange);
                });

    }

}
```

> 我们创建ByteBuf对象后，它的引用计数是1，当你每次调用DataBufferUtils.release之后会释放引用计数对象时，它的引用计数减1，如果引用计数为0，这个引用计数对象会被释放（deallocate）,并返回对象池。当尝试访问引用计数为0的引用计数对象会抛出IllegalReferenceCountException异常 
>
> 因此为了能够在多个自定义过滤器中使用相同的方法来获取body数据，不能调用`DataBufferUtils.release(buffer)`