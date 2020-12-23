---
layout: post
title: GRPC（4）- 通过Metadata传递RequestId
date: 2020-06-04
categories:
    - rpc
comments: true
permalink: grpc-request-id.html
---

在分布式系统中，一次请求的日志信息分布在不同的机器上或目录，如果不能很清晰的描述一个请求的生命周期，报错时也无法快速的定位问题。

一个简单的做法是为每次请求分配一个流水号traceId，在日志打印处加上这个traceId，模块调用时亦将traceId往下传，直至整条消息处理完成，返回时清除traceId

# 1. Metadata和Interceptor

Metadata 可以理解为一个 HTTP 请求的 Header（它的底层实现就是 HTTP/2 的 Header），用户可以通过访问和修改每个 gRPC Call 的 Metadata 来传递额外的信息：比如认证信息，比如本文中提到的 Request ID。

Metadata 是以key-value的形式存储数据的，其中key是string类型，而value是[]string，即一个字符串数组类型。

Metadata 的生命周期则是一次 RPC 调用。

Interceptor 有点类似于我们平时常用的 HTTP Middleware，不同的是它可以用在 Client 端和 Server 端。比如在收到请求之后输出日志，在请求出现错误的时候输出错误信息，比如获取请求中设置的 Request ID。

# 2. 实现
我们通过传递RequestId的例子学习一下

```java
public class Constant {

  public static final Context.Key<String> TRACE_ID_CTX_KEY = Context.key("traceId");

  public static final Metadata.Key<String> TRACE_ID_METADATA_KEY = Metadata.Key.of("traceId", ASCII_STRING_MARSHALLER);
}
```

server的拦截器

```java
public class TraceIdServerInterceptor implements ServerInterceptor {
  @Override
  public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(ServerCall<ReqT, RespT> serverCall, Metadata metadata, ServerCallHandler<ReqT, RespT> serverCallHandler) {
    //从metadata获取traceId,在放入context中
    String traceId = metadata.get(Constant.TRACE_ID_METADATA_KEY);
    Context ctx = Context.current().withValue(Constant.TRACE_ID_CTX_KEY, traceId);

    return Contexts.interceptCall(ctx, serverCall, metadata, serverCallHandler);
  }
}
```

server的实现类

```java
public class GreeterImpl extends GreeterGrpc.GreeterImplBase {

  @Override
  public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
    // 都能获取到traceId
    System.out.println(Constant.TRACE_ID_CTX_KEY.get());
    System.out.println(Constant.TRACE_ID_CTX_KEY.get(Context.current()));
    HelloReply reply = HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
    responseObserver.onNext(reply);
    responseObserver.onCompleted();
  }
}
```

Client的拦截器

```java
public class TraceIdClientInterceptor implements ClientInterceptor {
  @Override
  public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(MethodDescriptor<ReqT, RespT> methodDescriptor, CallOptions callOptions, Channel channel) {
    return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(channel.newCall(methodDescriptor, callOptions)) {
      @Override
      public void start(Listener<RespT> responseListener, Metadata headers) {
        if (Constant.TRACE_ID_CTX_KEY.get() != null) {
          headers.put(Constant.TRACE_ID_METADATA_KEY, Constant.TRACE_ID_CTX_KEY.get());
        }
        super.start(responseListener, headers);
      }
    };
  }
}
```

Client的实现

```java
ManagedChannel greetingChannel = ManagedChannelBuilder.forAddress("localhost", 9099)
    .usePlaintext(true)
    .intercept(new TraceIdClientInterceptor())
    .build();
    
Context.current().withValue(Constant.TRACE_ID_CTX_KEY, "1").run(() -> {
  GreetingServiceGrpc.GreetingServiceBlockingStub greetingStub = GreetingServiceGrpc.newBlockingStub(greetingChannel).withCallCredentials(callCredential);
  HelloReply helloReply = greetingStub.greeting(HelloRequest.newBuilder().setName("Edgar").build());
  System.out.println(helloReply);
});
```
