---
layout: post
title: GRPC（1） - 流式调用
date: 2020-06-01
categories:
    - rpc
comments: true
permalink: grpc-stream.html
---

GRPC提供了四种类型的服务方法

- **单项RPC**

这种最简单，就是客户端发一个请求，服务端返回结果。

- **服务端流式RPC**

客户端发一个请求，服务端返回一个流，客户端从流中读取每条信息。

- **客户端流RPC**

客户端向服务端发送一个流，服务端读取完数据返回处理结果。

- **双向流RPC**

客户端和服务端都可以独立的向对方发送、接收流数据。

gRPC 支持同步和异步调用，同步调用只支持单项RPC、服务器端流RPC，异步调用对这4中方式都支持。

通过使用流(streaming)，一端可以多次发送数据给对端，直到没有数据可以发送或者中途出错。 
服务器和客户端在接收这些数据的时候，可以不必等所有的消息全收到后才开始响应，而是接收到第一条消息的时候就可以及时的响应。
下一步怎样发展取决于应用，因为客户端和服务端能在任意顺序上读写 - 这些流的操作是完全独立的。
例如服务端可以一直等直到它接收到所有客户端的消息才写应答，或者服务端得到一个请求就回送一个应答，接着客户端根据应答来发送另一个请求。

下面通过3个例子来演示流的使用

```
syntax = "proto3";
option java_multiple_files = true;
option java_package = "com.github.edgar615.stream";

package com.github.edgar615.stream;

service StreamService {

  // 服务端流，客户端发送一个上限，然后服务端每次都传递一个结果回来，直到超过上限
  rpc facb (FacbRequest) returns (stream FacbResponse) {}
  
  // 客户端流，客户端可以发送不等数量的数字，然后服务端最后一次性返回客户端发送数字的和
  rpc sum (stream SumRequest) returns (SumResponse) {}

  // 客户端和服务器双向流的通信
  rpc chat(stream ChatRequest) returns (stream ChatResponse) {}

}

message SumRequest {
  int64 num = 1;
}

message SumResponse {
  int64 result = 1;
}

message FacbRequest {
  int64 max = 1;
}

message FacbResponse {
  int32 index = 1;
  int64 curr = 2;
}

message ChatRequest {
  string msg = 1;
}

message ChatResponse {
  string reply = 1;
}
```

# 1. 服务端流
服务端实现

```
@Override
public void facb(FacbRequest request, StreamObserver<FacbResponse> responseObserver) {
    LOGGER.log(Level.INFO, "received: {0}", request.getMax());
    for (int i = 0; i < request.getMax(); i ++) {
        // 没有真正实现 斐波那契数列
        FacbResponse facbResponse = FacbResponse.newBuilder().setCurr(i).build();
        responseObserver.onNext(facbResponse);
    }
    responseObserver.onCompleted();
}
```

客户端实现

```
StreamObserver<FacbResponse> responseStreamObserver = new StreamObserver<FacbResponse>() {
    @Override
    public void onNext(FacbResponse facbResponse) {
        logger.log(Level.INFO, "received: {0}", facbResponse.getCurr());
    }

    @Override
    public void onError(Throwable throwable) {

    }

    @Override
    public void onCompleted() {
        logger.log(Level.INFO, "complete");
    }
};

FacbRequest facbRequest = FacbRequest.newBuilder().setMax(num).build();
stub.facb(facbRequest, responseStreamObserver);
```

# 2. 客户端流

服务端

```
@Override
public StreamObserver<SumRequest> sum(StreamObserver<SumResponse> responseObserver) {
    return new StreamObserver<SumRequest>() {
        private int sum;
        @Override
        public void onNext(SumRequest sumRequest) {
            LOGGER.log(Level.INFO, "received: {0}", sumRequest.getNum());
            sum += sumRequest.getNum();
        }

        @Override
        public void onError(Throwable throwable) {
            responseObserver.onError(throwable);
        }

        @Override
        public void onCompleted() {
            SumResponse sumResponse = SumResponse.newBuilder().setResult(sum).build();
            responseObserver.onNext(sumResponse);
            responseObserver.onCompleted();
        }
    };
}
```

客户端实现

```
StreamObserver<SumResponse> responseStreamObserver = new StreamObserver<SumResponse>() {
  @Override
  public void onNext(SumResponse sumResponse) {
      logger.log(Level.INFO, "received: {0}", sumResponse.getResult());
  }

  @Override
  public void onError(Throwable throwable) {

  }

  @Override
  public void onCompleted() {

  }
};

StreamObserver<SumRequest> requestStreamObserver = stub.sum(responseStreamObserver);
for (int i = 1; i <= num; i++) {
  requestStreamObserver.onNext(SumRequest.newBuilder().setNum(i).build());
}
requestStreamObserver.onCompleted();
```

# 3. 双向流

服务端

```
@Override
public StreamObserver<ChatRequest> chat(StreamObserver<ChatResponse> responseObserver) {
    return new StreamObserver<ChatRequest>() {
        boolean exit = false;
        @Override
        public void onNext(ChatRequest chatRequest) {
            LOGGER.log(Level.INFO, "received: {0}", chatRequest.getMsg());
            ChatResponse chatResponse = ChatResponse.newBuilder()
                    .setReply("Rely: " + chatRequest.getMsg())
                    .build();
            responseObserver.onNext(chatResponse);
        }

        @Override
        public void onError(Throwable throwable) {

        }

        @Override
        public void onCompleted() {
            responseObserver.onNext(ChatResponse.newBuilder().setReply("bye").build());
            responseObserver.onCompleted();
        }
    };
}
```

客户端

```
StreamObserver<ChatResponse> responseStreamObserver = new StreamObserver<ChatResponse>() {
  @Override
  public void onNext(ChatResponse sumResponse) {
      logger.log(Level.INFO, "received: {0}", sumResponse.getReply());
  }

  @Override
  public void onError(Throwable throwable) {

  }

  @Override
  public void onCompleted() {

  }
};

StreamObserver<ChatRequest> requestStreamObserver = stub.chat(responseStreamObserver);
requestStreamObserver.onNext(ChatRequest.newBuilder().setMsg("你好").build());
requestStreamObserver.onNext(ChatRequest.newBuilder().setMsg("你好 2").build());
requestStreamObserver.onNext(ChatRequest.newBuilder().setMsg("你好 3").build());
requestStreamObserver.onNext(ChatRequest.newBuilder().setMsg("你好 4").build());
requestStreamObserver.onCompleted();
```
