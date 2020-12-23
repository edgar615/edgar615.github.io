---
layout: post
title: GRPC（6）- 健康检查
date: 2020-06-06
categories:
    - rpc
comments: true
permalink: grpc-healthcheck.html
---

GRPC提供了健康检查机制

> https://github.com/grpc/grpc/blob/master/doc/health-checking.md

```
syntax = "proto3";

package grpc.health.v1;

message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;  // Used only by the Watch method.
  }
  ServingStatus status = 1;
}

service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);

  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

一个服务有4中状态

- UNKNOWN = 0;
- SERVING = 1;
- NOT_SERVING = 2;
- SERVICE_UNKNOWN = 3

健康检查机制提供了两个方法

- Check方法 

查询服务的运行状况，服务名建议为`package_names.ServiceName`格式

服务端实现

```
HealthStatusManager healthStatusManager = new HealthStatusManager();
server = ServerBuilder.forPort(port)
        .addService(new GreeterImpl())
        .addService(healthStatusManager.getHealthService())
        .build()
        .start();
LOGGER.info("Server started, listening on " + port);
healthStatusManager.setStatus("com.github.edgar615.grpc.health.Greeter", HealthCheckResponse.ServingStatus.SERVING);
```

**服务端应该使用空字符串作为服务器整体运行状况的键**

客户端实现

```
HealthGrpc.HealthBlockingStub blockingStub = HealthGrpc.newBlockingStub(channel);
HealthCheckRequest request = HealthCheckRequest.newBuilder()
      .setService("com.github.edgar615.grpc.health.GreeterGrpc").build();
HealthCheckResponse response = blockingStub.check(request);
logger.info("HealthCheck: " + response.getStatus());
```

如果客户端传入了服务端没有定义的服务会抛出StatusRuntimeException

```
new StatusException(Status.NOT_FOUND.withDescription("unknown service " + request.getService()))
```

- Watch方法

客户端可以调用Watch方法来执行流式运行状况检查。服务器将立即发回指示当前服务状态的消息。随后，每当服务的服务状态发生变化时，它将发送一条新消息。

```
HealthCheckRequest request = HealthCheckRequest.newBuilder()
        .setService("com.github.edgar615.grpc.health.Greeter").build();
StreamObserver<HealthCheckResponse> responseStreamObserver = new StreamObserver<HealthCheckResponse>() {
    @Override
    public void onNext(HealthCheckResponse healthCheckResponse) {
        logger.info("HealthCheck: " + healthCheckResponse.getStatus());
    }

    @Override
    public void onError(Throwable throwable) {

    }

    @Override
    public void onCompleted() {

    }
};
stub.watch(request, responseStreamObserver);
```
