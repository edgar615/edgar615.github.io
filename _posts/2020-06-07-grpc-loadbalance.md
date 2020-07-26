---
layout: post
title: GRPC负载均衡
date: 2020-06-07
categories:
    - rpc
comments: true
permalink: grpc-loadbalance.html
---

GRPC 中负载均衡的主要机制是外部负载均衡，其中外部负载均衡器为简单客户端提供最新的服务器列表。

GRPC 客户端确实支持内置负载均衡策略的 API。 但是，只有少数这些（其中一个是实现外部负载均衡的 grpclb 策略），并且不鼓励用户尝试通过添加更多来扩展 gRPC。 相反，应在外部负载均衡器中实施新的负载均衡策略

工作流
负载均衡策略适用于命名解析和与服务器的连接之间的 gRPC 客户端工作流。

gRPC 开源组件官方并未直接提供服务注册与发现的功能实现，但其设计文档已提供实现的思路，并在不同语言的 gRPC 代码 API 中已提供了命名解析和负载均衡接口供扩展。

以下是它的工作原理：

![](/assets/images/posts/grpc-loadbalance/grpc-loadbalance-1.png)

1. 启动时，gRPC 客户端会为服务器名称发出 名称解析 请求。该名称将解析为一个或多个 IP 地址，每个 IP 地址将指示它是服务器地址还是负载均衡器地址，以及指示要使用哪个客户端负载均衡策略的 服务配置 （例如，round_robin 或 grpclb）。
2. 客户端实例化负载均衡策略。
  - 注意：如果解析程序返回的任何一个地址是均衡器地址，则无论服务配置请求了什么负载均衡策略，客户端都将使用 grpclb 策略。否则，客户端将使用服务配置请求的负载均衡策略。如果服务配置未请求负载均衡策略，则客户端将默认使用选择第一个可用服务器地址的策略。
3. 负载均衡策略为每个服务器地址创建一个子通道。
  - 对于除 grpclb 之外的所有策略，这意味着解析器返回的每个地址都有一个子通道。请注意，这些策略会忽略解析程序返回的任何均衡器地址。
  - 对于grpclb策略，工作流程如下： 
    - a. 该策略打开一个流到解析器返回的平衡器地址之一。它要求平衡器将服务器地址用于客户端最初请求的服务器名称（即，最初传递给名称解析器的服务器名称）。 注意：在 grpclb 策略中，解析器返回的非平衡器地址用作后备，以防在启动 LB 策略时无法联系到平衡器。 
    - b. 如果负载均衡器的配置需要该信息，则负载均衡器指向客户端的 gRPC 服务器可以向负载均衡器报告负载。 
    - c. 负载均衡器将服务器列表返回给 gRPC 客户端的 grpclb 策略。然后，grpclb 策略将为列表中的每个服务器创建一个子通道。
4. 对于发送的每个 RPC ，负载平衡策略决定应将 RPC 发送到哪个子通道（即哪个服务器）。
    - 对于 grpclb 策略，客户端将按负载均衡器返回的顺序向服务器发送请求。如果服务器列表为空，则调用将阻塞，直到收到非空的调用。

创建3个服务端

```java
public class HelloWorldServer {
    private static final Logger LOGGER = Logger.getLogger(HelloWorldServer.class.getSimpleName());
    private static void startServer(String name, int port) throws IOException, InterruptedException {
        Server server = ServerBuilder
                .forPort(port)
                .addService(new GreeterImpl(name))
                .build();

        server.start();
        System.out.println(name + " server started, listening on port: " + server.getPort());
        server.awaitTermination();
    }

    public static void main(String[] args) throws Exception {
        final int nServers = 3;
        ExecutorService executorService = Executors.newFixedThreadPool(nServers);
        for (int i = 0; i < nServers; i++) {
            String name = "Server_" + i;
            int port = 50050 + i;
            executorService.submit(() -> {
                try {
                    startServer(name, port);
                } catch (IOException | InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

    }
}
```

客户端

首先我们需要实现名称解析，获取所有的IP地址，GRPC默认使用DNS名称解析，但如果我们想使用服务注册组件，如Eureka、Consul，我们需要实现自己的名称解析。

实现自己的名称解析只需要集成`NameResolver`即可

```java
public class LocalNameResolver extends NameResolver {
    private final List<EquivalentAddressGroup> equivalentAddressGroups;

    public LocalNameResolver(List<EquivalentAddressGroup> equivalentAddressGroups) {
        this.equivalentAddressGroups = equivalentAddressGroups;
    }


    @Override
    public String getServiceAuthority() {
        return "fakeAuthority";
    }


    @Override
    public void start(Listener2 listener) {
        listener.onResult(ResolutionResult.newBuilder()
                .setAddresses(equivalentAddressGroups)
                .setAttributes(Attributes.EMPTY)
                .build());
    }

    @Override
    public void shutdown() {

    }
}
```
主要逻辑就是`List<EquivalentAddressGroup> equivalentAddressGroups`的维护，这个变量可以也可以防止start方法中再创建.

两个start方法都可以，推荐使用`start(Listener2 listener)`方法，因为`start(final Listener listener)`实际上也是调用的这个方法

```java
  public void start(final Listener listener) {
    if (listener instanceof Listener2) {
      start((Listener2) listener);
    } else {
      start(new Listener2() {
          @Override
          public void onError(Status error) {
            listener.onError(error);
          }

          @Override
          public void onResult(ResolutionResult resolutionResult) {
            listener.onAddresses(resolutionResult.getAddresses(), resolutionResult.getAttributes());
          }
      });
    }
  }
```

`getServiceAuthority()`方法返回用于验证与服务器的连接的权限(authority)。 必须来自受信任的来源，因为如果权限被篡改，RPC可能被发送到攻击者，泄露敏感用户数据。
实现必须以不阻塞的方式生成它，通常在一行中(in line)，必须保持不变。使用同样的参数从同一个的 factory 中创建出来的 NameResolver 必须返回相同的 authority 。

继承`NameResolver.Factory`抽象类，实现`newNameResolver`方法

```java
public class LocalNameResolverFactory extends NameResolver.Factory {

    private final List<EquivalentAddressGroup> equivalentAddressGroups;

    public LocalNameResolverFactory(SocketAddress... addresses) {
        this.equivalentAddressGroups = Arrays.stream(addresses)
                .map(EquivalentAddressGroup::new)
                .collect(Collectors.toList());
    }

    /**
     * 服务协议
     * @return
     */
    @Override
    public String getDefaultScheme() {
        return "local";
    }

    @Override
    public NameResolver newNameResolver(URI targetUri, NameResolver.Args args) {
        return new LocalNameResolver(equivalentAddressGroups);
    }
}
```

`newNameResolver(URI targetUri, NameResolver.Args args)`方法创建 NameResolver 用于给定的目标URI，或者在给定URI无法被这个 factory 解析时返回 null。决定应该仅仅基于 URI 的 scheme。
参数 targetUri 表示要解析的目标 URI，而这个 URI 的 scheme 必须不能为 null。它是在初始化Channel的时候传入，`ManagedChannelBuilder.forTarget(target)`

`getDefaultScheme()` 返回默认的scheme， 当 ManagedChannelBuilder.forTarget(String) 方法传入的字符串而不是合法的URI时，会用这个返回值构建 URI 。

```
targetUri = new URI(nameResolverFactory.getDefaultScheme(), "", "/" + target, null);
```

也可以继承`NameResolverProvider`，它也继承自`NameResolver.Factory`，多了几个辅助方法

客户端实现

```
List<SocketAddress> socketAddresses = new ArrayList<>();
socketAddresses.add(new InetSocketAddress("localhost", 50052));
socketAddresses.add(new InetSocketAddress("localhost", 50051));
socketAddresses.add(new InetSocketAddress("localhost", 50050));

NameResolver.Factory factory = new LocalNameResolverFactory(socketAddresses.toArray(new SocketAddress[0]));

ManagedChannel channel = ManagedChannelBuilder.forTarget(target)
		.nameResolverFactory(factory)
		.defaultLoadBalancingPolicy("round_robin")
		.usePlaintext()
		.build();
GreeterGrpc.GreeterBlockingStub stub = GreeterGrpc.newBlockingStub(channel);
for (int i = 0; i < 10; i++) {
	HelloRequest request = HelloRequest.newBuilder().setName("Edgar").build();
	HelloReply response;
	try {
		response = stub.sayHello(request);
	} catch (StatusRuntimeException e) {
		logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
		return;
	}
	logger.info("Greeting: " + response.getMessage());
}
```

输出如下

```
信息: Greeting: Server_0 say Hello Edgar
信息: Greeting: Server_0 say Hello Edgar
信息: Greeting: Server_1 say Hello Edgar
信息: Greeting: Server_2 say Hello Edgar
信息: Greeting: Server_0 say Hello Edgar
信息: Greeting: Server_1 say Hello Edgar
信息: Greeting: Server_2 say Hello Edgar
信息: Greeting: Server_0 say Hello Edgar
信息: Greeting: Server_1 say Hello Edgar
信息: Greeting: Server_2 say Hello Edgar
```

# 参考资料

https://github.com/grpc/grpc/blob/master/doc/load-balancing.md
