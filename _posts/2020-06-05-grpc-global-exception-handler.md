---
layout: post
title: GRPC配置Server全局异常处理器
date: 2020-06-05
categories:
    - rpc
comments: true
permalink: grpc-global-exception-handler.html
---

下面的代码将捕获所有运行时异常

```java
public class GlobalGrpcExceptionHandler implements ServerInterceptor {

   @Override
   public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(ServerCall<ReqT, RespT> call,
         Metadata requestHeaders, ServerCallHandler<ReqT, RespT> next) {
      ServerCall.Listener<ReqT> delegate = next.startCall(call, requestHeaders);
      return new SimpleForwardingServerCallListener<ReqT>(delegate) {
         @Override
         public void onHalfClose() {
            try {
               super.onHalfClose();
            } catch (Exception e) {
               call.close(Status.INTERNAL
                .withCause (e)
                .withDescription("error message"), new Metadata());
            }
         }
      };
   }
}
```
