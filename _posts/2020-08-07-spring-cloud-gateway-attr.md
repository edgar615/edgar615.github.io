---
layout: post
title: Spring Cloud Gateway - ServerWebExchangeUtils的上下文熟悉
date: 2020-08-07
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-attr.html
---

`ServerWebExchangeUtils`里面存放了很多静态公有的字符串KEY值(**这些字符串KEY的实际值是`org.springframework.cloud.gateway.support.ServerWebExchangeUtils.` + 下面任意的静态公有KEY**)，这些字符串KEY值一般是用于`ServerWebExchange`的属性(`Attribute`，见上文的`ServerWebExchange#getAttributes()`方法)的KEY，这些属性值都是有特殊的含义，在使用过滤器的时候如果时机适当可以直接取出来使用。

- `PRESERVE_HOST_HEADER_ATTRIBUTE`：是否保存Host属性，值是布尔值类型，写入位置是`PreserveHostHeaderGatewayFilterFactory`，使用的位置是`NettyRoutingFilter`，作用是如果设置为true，HTTP请求头中的Host属性会写到底层Reactor-Netty的请求Header属性中。
- `CLIENT_RESPONSE_ATTR`：保存底层Reactor-Netty的响应对象，类型是`reactor.netty.http.client.HttpClientResponse`。
- `CLIENT_RESPONSE_CONN_ATTR`：保存底层Reactor-Netty的连接对象，类型是`reactor.netty.Connection`。
- `URI_TEMPLATE_VARIABLES_ATTRIBUTE`：`PathRoutePredicateFactory`解析路径参数完成之后，把解析完成后的占位符KEY-路径Path映射存放在`ServerWebExchange`的属性中，KEY就是`URI_TEMPLATE_VARIABLES_ATTRIBUTE`。
- `CLIENT_RESPONSE_HEADER_NAMES`：保存底层Reactor-Netty的响应Header的名称集合。
- `GATEWAY_ROUTE_ATTR`：用于存放`RoutePredicateHandlerMapping`中匹配出来的具体的路由(`org.springframework.cloud.gateway.route.Route`)实例，通过这个路由实例可以得知当前请求会路由到下游哪个服务。
- `GATEWAY_REQUEST_URL_ATTR`：`java.net.URI`类型的实例，这个实例代表直接请求或者负载均衡处理之后需要请求到下游服务的真实URI。
- `GATEWAY_ORIGINAL_REQUEST_URL_ATTR`：`java.net.URI`类型的实例，需要重写请求URI的时候，保存原始的请求URI。
- `GATEWAY_HANDLER_MAPPER_ATTR`：保存当前使用的`HandlerMapping`具体实例的类型简称(一般是字符串"RoutePredicateHandlerMapping")。
- `GATEWAY_SCHEME_PREFIX_ATTR`：确定目标路由URI中如果存在schemeSpecificPart属性，则保存该URI的scheme在此属性中，路由URI会被重新构造，见`RouteToRequestUrlFilter`。
- `GATEWAY_PREDICATE_ROUTE_ATTR`：用于存放`RoutePredicateHandlerMapping`中匹配出来的具体的路由(`org.springframework.cloud.gateway.route.Route`)实例的ID。
- `WEIGHT_ATTR`：实验性功能(此版本还不建议在正式版本使用)存放分组权重相关属性，见`WeightCalculatorWebFilter`。
- `ORIGINAL_RESPONSE_CONTENT_TYPE_ATTR`：存放响应Header中的ContentType的值。
- `HYSTRIX_EXECUTION_EXCEPTION_ATTR`：`Throwable`的实例，存放的是Hystrix执行异常时候的异常实例，见`HystrixGatewayFilterFactory`。
- `GATEWAY_ALREADY_ROUTED_ATTR`：布尔值，用于判断是否已经进行了路由，见`NettyRoutingFilter`。
- `GATEWAY_ALREADY_PREFIXED_ATTR`：布尔值，用于判断请求路径是否被添加了前置部分，见`PrefixPathGatewayFilterFactory`。

```
@Slf4j
@Component
public class AccessLogFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getPath().pathWithinApplication().value();
        HttpMethod method = request.getMethod();
        // 获取路由的目标URI
        URI targetUri = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        InetSocketAddress remoteAddress = request.getRemoteAddress();
        return chain.filter(exchange.mutate().build()).then(Mono.fromRunnable(() -> {
            ServerHttpResponse response = exchange.getResponse();
            HttpStatus statusCode = response.getStatusCode();
            log.info("请求路径:{},客户端远程IP地址:{},请求方法:{},目标URI:{},响应码:{}",
                    path, remoteAddress, method, targetUri, statusCode);
        }));
    }
}

```

