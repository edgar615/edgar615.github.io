---
layout: post
title: GRPC异常重试
date: 2020-05-16
categories:
    - rpc
comments: true
permalink: grpc-retry.html
---

# 1. 异常重试
之前我们已经介绍了RPC的[异常重试](https://edgar615.github.io/rpc-retry.html)，下面我们看看GRPC内置的重试模块。

RPC 调用失败可以分为三种情况：

- RPC 请求还没有离开客户端
- RPC 请求到达服务器，但是服务器的应用逻辑还没有处理该请求
- 服务器应用逻辑开始处理请求，并且处理失败

对于前两种情况，gRPC 客户端会自动重试，因为这两种情况，服务端的逻辑并没有开始处理请求，所以始终可以重试，也被称为透明重试(transparent retries)。
对于第三种情况是GRPC是通过server config配置的重试策略来处理的

在客户端创建gRPC连接时可以通过`defaultServiceConfig(Map<String, ?> serviceConfig)`方法设置重试策略，我们一般将配置写入JSON文件读取

```
{
  "methodConfig": [
    {
      "name": [
        {
          "service": "helloworld.Greeter",
          "method": "SayHello"
        }
      ],

      "retryPolicy": {
        "maxAttempts": 5,
        "initialBackoff": "0.5s",
        "maxBackoff": "30s",
        "backoffMultiplier": 2,
        "retryableStatusCodes": [
          "UNAVAILABLE"
        ]
      }
    }
  ]
}
```
> 详细描述可以查看 https://github.com/grpc/grpc-proto/blob/master/grpc/service_config/service_config.proto


- **name** 指定需要配置异常重试的RPC方法，service是必填项，method是可选项
- **retryPolicy** 指定重试策略
    - **maxAttempts** 最大重试次数，指定一次RPC 调用中最多的请求次数，包括第一次请求。必须是大于 1 的整数，对于大于5的值会被视为5。如果设置了调用的过期时间，那么到了过期时间，无论重试情况如果都会返回超时错误DeadlineExceeded
    - **retryableStatusCode** 重试状态码，当 RPC 调用返回非 OK 响应，会根据 retryableStatusCode 来判断是否进行重试，GRPC并没有提供自定义CODE的功能，所以只能用内置的CODE
    - **initialBackoff，maxBackoff，backoffMultiplier** 指数退避参数，在进行下一次重试请求前，会计算需要等待的时间。必须指定，并且必须具有大于0。第一次重试间隔是`random(0, initialBackoff)`, 第 n 次的重试间隔为`random(0, min( initialBackoff*backoffMultiplier**(n-1) , maxBackoff))`
    
示例：

```
ManagedChannel channel = ManagedChannelBuilder.forTarget(target)
        .defaultServiceConfig(getRetryingServiceConfig())
        .enableRetry() // 重要
        .usePlaintext()
        .build();
        
protected static Map<String, ?> getRetryingServiceConfig() {
    return new Gson()
            .fromJson(
                    new JsonReader(
                            new InputStreamReader(
                                    HelloWorldClient.class.getResourceAsStream(
                                            "retrying_service_config.json"),
                                    UTF_8)),
                    Map.class);
}
```

# 2. 对冲策略
对冲是指在不等待响应的情况主动发送单次调用的多个请求。

对冲策略里面，请求是按照如下逻辑发出的：

1. 第一次正常的请求正常发出
2. 在等待固定时间间隔后，没有收到正确的响应，第二个对冲请求会被发出
3. 再等待固定时间间隔后，没有收到任何前面两个请求的正确响应，第三个会被发出
4. 一直重复以上流程直到发出的对冲请求数量达到配置的最大次数
5. 一旦收到正确响应，所有对冲请求都会被取消，响应会被返回给应用层

注意： 使用对冲的时候，请求可能会访问到不同的后端(如果设置了负载均衡)，那么就要求方法在多次执行下是安全，并且符合预期的；对冲策略应该只用于幂等的操作

与retryPolicy 一样，对冲也会受到调用过期时间的影响，过期时间到了那么直接返回DeadlineExceeded

配置文件

```
{
  "methodConfig": [
    {
      "name": [
        {
          "service": "helloworld.Greeter",
          "method": "SayHello"
        }
      ],
      "hedgingPolicy": {
        "maxAttempts": 3,
        "hedgingDelay": "1s"
      }
    }
  ]
}
```

- **name** 指定需要配置对冲策略的RPC方法，service是必填项，method是可选项
- **hedgingPolicy** 指定对冲策略
    - **maxAttempts** 最大请求次数，指定一次RPC 调用中最多的请求次数，包括第一次请求。 必须是大于 1 的整数，对于大于5的值会被视为5。如果设置了调用的过期时间，那么到了过期时间，无论重试情况如果都会返回超时错误DeadlineExceeded
    - **hedgingDelay** 等待时间，如果hedgingDelay时间内没有响应，那么直接发送第二次请求，如果指定0S，会立即将maxAttempts个请求发出
    - **nonFatalStatusCodes** 当对冲请求接收到 nonFatalStatusCodes后，会立即发送下一个对冲请求，不管 hedgingDelay。如果受到其他的状态码，则所有未完成的对冲请求都将被取消，并且将状态码返回给调用者
本质上，对冲可以看做是受到 FatalStatusCodes 前对 RPC 调用的重试。**可选的字段，因为在上一个请求没有响应的时候也会发送对冲请求**

**测试没有验证成功，后面有机会再检查问题**
   
