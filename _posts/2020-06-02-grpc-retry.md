---
layout: post
title: GRPC（2）- 异常重试
date: 2020-06-02
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
        .enableRetry() // 重要，客户端是默认关闭了重试的
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

**在本地测试时第一次不会使用对冲策略，第二次才会使用，后面有机会再检查问题，DEBUG后发现，第一次调用时认为GRC未初始化完成，不会使用我们自定义的对冲策略。所以这个特性暂时还用不了，等后面搞清楚再补充**

# 3. 重试限流

当客户端的失败和成功比超过某个阈值时，gRPC 会通过禁用这些重试策略来防止由于重试导致服务器过载

```
"retryThrottling": {
    "maxTokens": 10,
    "tokenRatio": 0.1
}
```

重试限流是根据服务器来设置的，而不是针对方法或者服务。对于每个 server，gRPC 的客户端都维护了一个 token_count 变量，变量初始值为配置的 maxTokens 值，值的范围是 0 - maxToken，每次 RPC 请求都会影响这个 token_count 变量值：

- 每次失败的 RPC 请求都会对 token_count 减 1
- 每次成功的 RPC 请求都会对 token_count 增加 tokenRatio 值

如果 `token_count <= (maxTokens / 2)`，那么后续发出的请求即使失败也不会进行重试了，但是正常的请求还是会发出去，直到这个 `token_count > (maxTokens / 2)` 才又恢复对失败请求的重试。这种策略可以有效的处理长时间故障。

tokenRatio介于0~1之间，支持3位小数

> 网上有篇文章说可以通过服务端返回头`grpc-retry-pushback-ms`让客户端发起重试，后续测试
> 也说可以在请求头中获取到`grpc-previous-rpc-attempts`，未测试成功
