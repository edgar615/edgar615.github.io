---
layout: post
title: GRPC（3）- 拦截器
date: 2020-06-03
categories:
    - rpc
comments: true
permalink: grpc-interceptor.html
---

在4个地方可以增加拦截器

- 客户端调用前的拦截
- 客户端收到的回复拦截
- 服务端收到的请求拦截
- 服务端回复前的拦截


# 1. 客户端调用前的拦截

ClientCall该抽象类就是用来调用远程方法的,实现了发送消息和接收消息的功能,该接口由两个泛型ReqT和ReqT,分别对应着请求发送的信息,和请求收到的回复.

如下代码流程所示:

```
call = channel.newCall(unaryMethod, callOptions);
call.start(listener, headers);
call.sendMessage(message);//消息发送
call.halfClose();//请求端关闭发送通道
call.request(1);
// wait for listener.onMessage()
```

对于调用前的拦截，我们可以通过实现`ClientInterceptor`接口，然后将`ClientCall`委托给一个新的ClientCall来实现

```java
public class ClientPreInterceptor implements ClientInterceptor {
    private static final Logger LOGGER = Logger.getLogger(ClientPreInterceptor.class.getSimpleName());
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(MethodDescriptor<ReqT, RespT> method, CallOptions callOptions, Channel next) {
        final String methodName = method.getFullMethodName();
        return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                LOGGER.log(Level.INFO, "ClientCall {0} start", methodName);
                super.start(responseListener, headers);
            }

            @Override
            public void sendMessage(ReqT message) {
                LOGGER.log(Level.INFO, "ClientCall {0} sendMessage: {1}", new Object[]{methodName, message});
                super.sendMessage(message);
            }

            @Override
            public void request(int numMessages) {
                LOGGER.log(Level.INFO, "ClientCall {0} request: {1}", new Object[]{methodName, numMessages});
                super.request(numMessages);
            }

            @Override
            public void halfClose() {
                LOGGER.log(Level.INFO, "ClientCall {0} halfClose", methodName);
                super.halfClose();
            }
        };
    }
}
```

Client可以在创建Channel时注册全局的拦截器

```java
ManagedChannel channel = ManagedChannelBuilder.forTarget(target)
        .intercept(new ClientPreInterceptor())
        .usePlaintext()
        .build();
```

也可以在调用方法时指定拦截器

```java
blockingStub.withInterceptors(new ClientPreInterceptor()).sayHello(request);
```

输入日志如下

```
七月 12, 2020 8:56:05 下午 com.github.edgar615.grpc.interceptor.ClientPreInterceptor$1 start
信息: ClientCall helloworld.Greeter/SayHello start
七月 12, 2020 8:56:05 下午 com.github.edgar615.grpc.interceptor.ClientPreInterceptor$1 request
信息: ClientCall helloworld.Greeter/SayHello request: 2
七月 12, 2020 8:56:06 下午 com.github.edgar615.grpc.interceptor.ClientPreInterceptor$1 sendMessage
信息: ClientCall helloworld.Greeter/SayHello sendMessage: name: "world"

七月 12, 2020 8:56:06 下午 com.github.edgar615.grpc.interceptor.ClientPreInterceptor$1 halfClose
信息: ClientCall helloworld.Greeter/SayHello halfClose
```

# 2. 客户端收到的回复拦截

ClientCall抽象类除了请求调用的一系列方法外，还有一个`public abstract static class Listener<T>`用于监听服务端回复的消息

```java
public class ClientPostInterceptor implements ClientInterceptor {
    private static final Logger LOGGER = Logger.getLogger(ClientPostInterceptor.class.getSimpleName());
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(MethodDescriptor<ReqT, RespT> method, CallOptions callOptions, Channel next) {
        final String methodName = method.getFullMethodName();
        return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                ClientCall.Listener listener = new ForwardingClientCallListener.SimpleForwardingClientCallListener(responseListener) {
                    @Override
                    public void onMessage(Object message) {
                        LOGGER.log(Level.INFO, "ClientCall {0} onMessage {1}", new Object[]{methodName, message});
                        super.onMessage(message);
                    }

                    @Override
                    public void onHeaders(Metadata headers) {
                        LOGGER.log(Level.INFO, "ClientCall {0} onHeaders", new Object[]{methodName});
                        super.onHeaders(headers);
                    }

                    @Override
                    public void onClose(Status status, Metadata trailers) {
                        LOGGER.log(Level.INFO, "ClientCall {0} onClose {1}", new Object[]{methodName, status.getCode()});
                        super.onClose(status, trailers);
                    }

                    @Override
                    public void onReady() {
                        LOGGER.log(Level.INFO, "ClientCall {0} onReady", new Object[]{methodName});
                        super.onReady();
                    }
                };

                super.start(listener, headers);
            }
        };
    }
}
```

输出

```
七月 12, 2020 9:15:13 下午 com.github.edgar615.grpc.interceptor.ClientPostInterceptor$1$1 onHeaders
信息: ClientCall helloworld.Greeter/SayHello onHeaders
七月 12, 2020 9:15:13 下午 com.github.edgar615.grpc.interceptor.ClientPostInterceptor$1$1 onMessage
信息: ClientCall helloworld.Greeter/SayHello onMessage message: "Hello world"

七月 12, 2020 9:15:13 下午 com.github.edgar615.grpc.interceptor.ClientPostInterceptor$1$1 onClose
信息: ClientCall helloworld.Greeter/SayHello onClose OK
```

# 3. 服务端收到的请求拦截

`ServerCall`该抽象类就是用来调用远程方法的,和`ClientCall`类似

收到请求的拦截也是通过监听器实现

```java
public class ServerPreInterceptor implements ServerInterceptor {

    private static final Logger LOGGER = Logger.getLogger(ServerPreInterceptor.class.getSimpleName());

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(ServerCall<ReqT, RespT> call, Metadata headers, ServerCallHandler<ReqT, RespT> next) {
        final String methodName = call.getMethodDescriptor().getFullMethodName();
        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<ReqT>(next.startCall(call, headers)) {
            @Override
            public void onMessage(ReqT message) {
                LOGGER.log(Level.INFO, "ServerCall {0} onMessage {1}", new Object[]{methodName, message});
                super.onMessage(message);
            }

            @Override
            public void onHalfClose() {
                LOGGER.log(Level.INFO, "ServerCall {0} onHalfClose", new Object[]{methodName});
                super.onHalfClose();
            }

            @Override
            public void onCancel() {
                LOGGER.log(Level.INFO, "ServerCall {0} onCancel", new Object[]{methodName});
                super.onCancel();
            }

            @Override
            public void onComplete() {
                LOGGER.log(Level.INFO, "ServerCall {0} onComplete", new Object[]{methodName});
                super.onComplete();
            }

            @Override
            public void onReady() {
                LOGGER.log(Level.INFO, "ServerCall {0} onReady", new Object[]{methodName});
                super.onReady();
            }
        };
    }
}
```

可以在创建server时注册全局拦截器

```
server = ServerBuilder.forPort(port)
        .addService(new GreeterImpl())
        .intercept(new ServerPreInterceptor())
        .build()
        .start();
```

也可以为每个Service单独注册拦截器

```
server = ServerBuilder.forPort(port)
        .addService(ServerInterceptors.intercept(new GreeterImpl(), new ServerPreInterceptor()))
        .build()
        .start();
```

输出

```
七月 12, 2020 9:27:13 下午 com.github.edgar615.grpc.interceptor.ServerPreInterceptor$1 onReady
信息: ServerCall helloworld.Greeter/SayHello onReady
七月 12, 2020 9:27:13 下午 com.github.edgar615.grpc.interceptor.ServerPreInterceptor$1 onMessage
信息: ServerCall helloworld.Greeter/SayHello onMessage name: "world"

七月 12, 2020 9:27:13 下午 com.github.edgar615.grpc.interceptor.ServerPreInterceptor$1 onHalfClose
信息: ServerCall helloworld.Greeter/SayHello onHalfClose
七月 12, 2020 9:27:13 下午 com.github.edgar615.grpc.interceptor.ServerPreInterceptor$1 onComplete
信息: ServerCall helloworld.Greeter/SayHello onComplete

```
# 4. 服务端回复前的拦截

```java
public class ServerPostInterceptor implements ServerInterceptor {
    private static final Logger LOGGER = Logger.getLogger(ServerPostInterceptor.class.getSimpleName());

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(ServerCall<ReqT, RespT> call, Metadata headers, ServerCallHandler<ReqT, RespT> next) {
        final String methodName = call.getMethodDescriptor().getFullMethodName();
        ServerCall<ReqT, RespT> newCall = new ForwardingServerCall.SimpleForwardingServerCall<ReqT, RespT>(call) {
            @Override
            public void sendMessage(RespT message) {
                LOGGER.log(Level.INFO, "ServerCall {0} sendMessage {1}", new Object[]{methodName, message});
                super.sendMessage(message);
            }

            @Override
            public void sendHeaders(Metadata headers) {
                LOGGER.log(Level.INFO, "ServerCall {0} sendHeaders", new Object[]{methodName});
                super.sendHeaders(headers);
            }

            @Override
            public void close(Status status, Metadata trailers) {
                LOGGER.log(Level.INFO, "ServerCall {0} close {1}", new Object[]{methodName, status.getCode()});
                super.close(status, trailers);
            }
        };
        return next.startCall(newCall, headers);
    }
}
```

输出

```
七月 12, 2020 9:37:21 下午 com.github.edgar615.grpc.interceptor.ServerPostInterceptor$1 sendHeaders
信息: ServerCall helloworld.Greeter/SayHello sendHeaders
七月 12, 2020 9:37:21 下午 com.github.edgar615.grpc.interceptor.ServerPostInterceptor$1 sendMessage
信息: ServerCall helloworld.Greeter/SayHello sendMessage message: "Hello world"

七月 12, 2020 9:37:21 下午 com.github.edgar615.grpc.interceptor.ServerPostInterceptor$1 close
信息: ServerCall helloworld.Greeter/SayHello close OK
```

还有些方法没有写出来
