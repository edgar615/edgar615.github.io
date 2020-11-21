---
layout: post
title: Spring Cloud Gateway - 重写请求头和响应体
date: 2020-08-09
categories:
    - Spring
comments: true
permalink: spring-cloud-gateway-rewrite-body.html
---

假设我们要这样实现一个功能：调用方请求经过网关，原始报文是密文，我们需要在网关实现密文解密，然后把解密后的明文路由到下游服务，下游服务处理成功响应明文，需要在网关把明文加密成密文再返回到调用方

# 1. 重写请求头

在日志章节，我们已经了解了如何读取请求头

```
public class DecryptRequestFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return DataBufferUtils.join(exchange.getRequest().getBody())
                .flatMap(dataBuffer -> {
                    ServerHttpRequest request = decryptRequest(exchange.getRequest(), dataBuffer);
                    return chain.filter(exchange.mutate().request(request).build());
                });

    }

    private ServerHttpRequest decryptRequest(ServerHttpRequest request, DataBuffer dataBuffer) {
        DataBufferUtils.retain(dataBuffer);
        Flux<DataBuffer> cachedFlux = Flux
                .defer(() -> Flux.just(dataBuffer.slice(0, dataBuffer.readableByteCount())));
        String body = toRaw(cachedFlux);
        byte[] decryptedBodyBytes = Base64.getDecoder().decode(body.getBytes(StandardCharsets.UTF_8));
        return new ServerHttpRequestDecorator(request) {

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders httpHeaders = new HttpHeaders();
                httpHeaders.putAll(request.getHeaders());
                if (decryptedBodyBytes.length > 0) {
                    httpHeaders.setContentLength(decryptedBodyBytes.length);
                }
//                httpHeaders.set(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON.toString());
                return httpHeaders;
            }

            @Override
            public Flux<DataBuffer> getBody() {
                return Flux.just(body).
                        map(s -> new DefaultDataBufferFactory().wrap(decryptedBodyBytes));
            }
        };
    }

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


    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 1;
    }
}
```

# 2. 重写响应头

```
public class EncryptResponseFilter implements GlobalFilter, Ordered {

    // 通过@Order注解不起作用
    @Override
    public int getOrder() {
        // 控制在NettyWriteResponseFilter后执行
        return NettyWriteResponseFilter.WRITE_RESPONSE_FILTER_ORDER - 1;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange.mutate()
                .response(encrypteResponse(exchange)).build());
    }

    private ServerHttpResponse encrypteResponse(ServerWebExchange exchange) {
        ServerHttpResponse response = exchange.getResponse();
        return new ServerHttpResponseDecorator(response) {

            @Override
            public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {

                HttpHeaders httpHeaders = new HttpHeaders();
                httpHeaders.set(HttpHeaders.CONTENT_TYPE, MediaType.TEXT_PLAIN_VALUE);
                httpHeaders.set(HttpHeaders.CONTENT_ENCODING, "application/octet-stream");
                ClientResponse clientResponse = prepareClientResponse(body, httpHeaders);
                Mono<String> modifiedBody = clientResponse.bodyToMono(String.class)
                        .flatMap( originalBody ->
                                Mono.just(Objects.requireNonNull(Base64.getEncoder().encodeToString(originalBody.getBytes()))))
                        .switchIfEmpty(Mono.empty());
                BodyInserter<Mono<String>, ReactiveHttpOutputMessage> bodyInserter = BodyInserters.fromPublisher(modifiedBody, String.class);

                CachedBodyOutputMessage outputMessage = new CachedBodyOutputMessage(exchange, response.getHeaders());
                return bodyInserter.insert(outputMessage, new BodyInserterContext())
                        .then(Mono.defer(() -> {
                            Mono<DataBuffer> messageBody = updateBody(getDelegate(), outputMessage);
                            HttpHeaders headers = getDelegate().getHeaders();
                            headers.setContentType(MediaType.TEXT_PLAIN);
                            if (headers.containsKey(HttpHeaders.CONTENT_LENGTH)) {
                                messageBody = messageBody.doOnNext(data -> {
                                    headers.setContentLength(data.readableByteCount());
                                });
                            }

                            return getDelegate().writeWith(messageBody);
                        }));

            }

            private Mono<DataBuffer> updateBody(ServerHttpResponse httpResponse,
                                                CachedBodyOutputMessage message) {

                Mono<DataBuffer> response = DataBufferUtils.join(message.getBody());
                // 参考ModifyResponseBodyGatewayFilterFactory
//                List<String> encodingHeaders = httpResponse.getHeaders()
//                        .getOrEmpty(HttpHeaders.CONTENT_ENCODING);
//                for (String encoding : encodingHeaders) {
//                    MessageBodyEncoder encoder = messageBodyEncoders.get(encoding);
//                    if (encoder != null) {
//                        DataBufferFactory dataBufferFactory = httpResponse.bufferFactory();
//                        response = response.publishOn(Schedulers.parallel())
//                                .map(encoder::encode).map(dataBufferFactory::wrap);
//                        break;
//                    }
//                }

                return response;

            }

            private Mono<String> extractBody(ServerHttpResponse response, ClientResponse clientResponse) {

//                List<String> encodingHeaders = response.getHeaders()
//                        .getOrEmpty(HttpHeaders.CONTENT_ENCODING);
//                for (String encoding : encodingHeaders) {
//                    MessageBodyDecoder decoder = messageBodyDecoders.get(encoding);
//                    if (decoder != null) {
//                        return clientResponse.bodyToMono(byte[].class)
//                                .publishOn(Schedulers.parallel()).map(decoder::decode)
//                                .map(bytes -> response.bufferFactory()
//                                        .wrap(bytes))
//                                .map(buffer -> prepareClientResponse(Mono.just(buffer),
//                                        response.getHeaders()))
//                                .flatMap(resp -> resp.bodyToMono(String.class));
//                    }
//                }


                return clientResponse.bodyToMono(String.class);

            }

            private ClientResponse prepareClientResponse(Publisher<? extends DataBuffer> body, HttpHeaders httpHeaders) {
                ClientResponse.Builder builder = ClientResponse.create(response.getStatusCode(),
                        HandlerStrategies.withDefaults().messageReaders());
                return builder.headers(headers -> headers.putAll(httpHeaders)).body(Flux.from(body)).build();
            }
        };
    }

}
```

