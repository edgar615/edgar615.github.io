---
layout: post
title: GRPC配置Server全局异常处理器
date: 2020-06-05
categories:
    - rpc
comments: true
permalink: grpc-global-exception-handler.html
---

先看一段代码

```
public class GreeterImpl extends GreeterGrpc.GreeterImplBase {

  @Override
  public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
    if (Strings.isNullOrEmpty(req.getName())) {
      throw new IllegalArgumentException("Missing name");
    }
    HelloReply reply = HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
    responseObserver.onNext(reply);
    responseObserver.onCompleted();
  }

}
```

客户端调用的时候会抛出异常

```
HelloRequest request = HelloRequest.newBuilder().build();
HelloReply response = blockingStub.sayHello(request);
```

```
Exception in thread "main" io.grpc.StatusRuntimeException: UNKNOWN
```
 
我们可以看到异常并没有提供任何有用的信息。

我们可以在server端通过拦截器，捕获所有运行时异常，并返回更多的异常信息给调用方

```java
public class GlobalGrpcExceptionHandler implements ServerInterceptor {

   @Override
   public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(ServerCall<ReqT, RespT> call,
                                                                Metadata requestHeaders, ServerCallHandler<ReqT, RespT> next) {
      ServerCall.Listener<ReqT> delegate = next.startCall(call, requestHeaders);
      return new ExceptionHandlingServerCallListener<>(delegate, call, requestHeaders);
   }

   private class ExceptionHandlingServerCallListener<ReqT, RespT>
           extends ForwardingServerCallListener.SimpleForwardingServerCallListener<ReqT> {
      private ServerCall<ReqT, RespT> serverCall;
      private Metadata metadata;

      ExceptionHandlingServerCallListener(ServerCall.Listener<ReqT> listener, ServerCall<ReqT, RespT> serverCall,
                                          Metadata metadata) {
         super(listener);
         this.serverCall = serverCall;
         this.metadata = metadata;
      }

      @Override
      public void onHalfClose() {
         try {
            super.onHalfClose();
         } catch (RuntimeException ex) {
            handleException(ex, serverCall, metadata);
            throw ex;
         }
      }

      @Override
      public void onReady() {
         try {
            super.onReady();
         } catch (RuntimeException ex) {
            handleException(ex, serverCall, metadata);
            throw ex;
         }
      }

      private void handleException(RuntimeException exception, ServerCall<ReqT, RespT> serverCall, Metadata metadata) {
         if (exception instanceof IllegalArgumentException) {
            serverCall.close(Status.INVALID_ARGUMENT.withDescription(exception.getMessage()), metadata);
         } else {
            serverCall.close(Status.UNKNOWN, metadata);
         }
      }
   }
}
```

再次运行client，可以看到抛出的异常内容已经变了

```
Exception in thread "main" io.grpc.StatusRuntimeException: INVALID_ARGUMENT: Missing name
```

这里只拦截了`onReady`和`onHalfClose`方法，因为拦截`onCancel`和`onComplete`方法对应返回内容已经没有任何作用
