---
layout: post
title: GRPC（11）- 基于Zipkin跟踪调用链
date: 2020-06-11
categories:
    - rpc
comments: true
permalink: grpc-zipkin.html
---

# 1. 基本用法

引入依赖

```
<dependency>
	<groupId>io.zipkin.reporter2</groupId>
	<artifactId>zipkin-sender-okhttp3</artifactId>
	<version>2.16.3</version>
</dependency>
<dependency>
	<groupId>io.zipkin.brave</groupId>
	<artifactId>brave-instrumentation-grpc</artifactId>
	<version>5.13.3</version>
</dependency>
```

创建GrpcTracing

```
Sender sender = OkHttpSender.create("http://192.168.159.131:9411/api/v2/spans");
//   (this dependency is io.zipkin.reporter2:zipkin-reporter-brave)
ZipkinSpanHandler zipkinSpanHandler = AsyncZipkinSpanHandler.create(sender);
// Create a tracing component with the service name you want to see in Zipkin.
Tracing tracing = Tracing.newBuilder()
	   .localServiceName(serviceName)
	   .addSpanHandler(zipkinSpanHandler)
	   .build();
GrpcTracing grpcTracing = GrpcTracing.create(tracing);
```

sesrver

```
Server server = ServerBuilder.forPort(9998)
		.addService(ServerInterceptors.intercept(new GretterGrpcImpl(), grpcTracing.newServerInterceptor()))
		.build()
		.start();
```

client

```
ManagedChannel managedChannel = ManagedChannelBuilder.forAddress("localhost", 9998)
		.intercept(grpcTracing.newClientInterceptor())
		.usePlaintext()
		.build();
```

> 如果和Spring Cloud Zipkin结合的化，直接注入Tracing即可，brave-instrumentation-grpc版本需要与brave一致
>
> 和HTTP一样，trace的相关数据也是通过请求头传递的

# 2. 添加默认tag

```
Tag<GrpcRequest> methodType = new Tag<GrpcRequest>("grpc.method_type") {
   @Override protected String parseValue(GrpcRequest input, TraceContext context) {
	   return input.methodDescriptor().getType().name();
   }
};
RpcRequestParser addMethodType = (req, context, span) -> {
   RpcRequestParser.DEFAULT.parse(req, context, span);
   if (req instanceof GrpcRequest) methodType.tag((GrpcRequest) req, span);
};

GrpcTracing grpcTracing = GrpcTracing.create(RpcTracing.newBuilder(tracing)
	   .clientRequestParser(addMethodType)
	   .serverRequestParser(addMethodType).build());
```

观察zipkin，我们可以看到span中多了一个`grpc.method_type:UNARY`的tag

# 3. 参考资料

https://github.com/openzipkin/brave/tree/master/brave