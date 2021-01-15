---
layout: post
title: Netty - Pipeline
date: 2019-01-05
categories:
    - netty
comments: true
permalink: netty-pipeline.html
---

> 基本复制自《Netty 核心原理剖析与 RPC 实践》

EventLoop 可以说是 Netty 的调度中心，负责监听多种事件类型：I/O 事件、信号事件、定时事件等，然而实际的业务处理逻辑则是由  ChannelPipeline 中所定义的 ChannelHandler 完成的，ChannelPipeline 和  ChannelHandler 也是我们在平时应用开发的过程中打交道最多的组件。Netty 服务编排层的核心组件 ChannelPipeline 和 ChannelHandler 为用户提供了 I/O 事件的全部控制权。

Pipeline 的字面意思是管道、流水线。它在 Netty 中起到的作用，和一个工厂的流水线类似。原始的网络字节流经过 Pipeline ，被一步步加工包装，最后得到加工后的成品。

# 1. 内部结构

ChannelPipeline 作为 Netty 的核心编排组件，负责调度各种类型的 ChannelHandler，实际数据的加工处理操作则是由 ChannelHandler 完成的。

ChannelPipeline 可以看作是 ChannelHandler 的容器载体，它是由一组  ChannelHandler 实例组成的，内部通过双向链表将不同的 ChannelHandler 链接在一起，如下图所示。当有 I/O  读写事件触发时，ChannelPipeline 会依次调用 ChannelHandler 列表对 Channel 的数据进行拦截和处理。

![](/assets/images/posts/netty-pipeline/netty-pipeline-1.png)

每个 Channel 会绑定一个 ChannelPipeline，每一个 ChannelPipeline 都包含多个 ChannelHandlerContext，所有 ChannelHandlerContext 之间组成了双向链表。又因为每个 ChannelHandler 都对应一个 ChannelHandlerContext，所以实际上 ChannelPipeline 维护的是它与 ChannelHandlerContext 的关系。

**为什么这里会多一层 ChannelHandlerContext 的封装呢？**

ChannelHandlerContext 用于保存 ChannelHandler 上下文；ChannelHandlerContext 则包含了 ChannelHandler 生命周期的所有事件，如 connect、bind、read、flush、write、close  等。可以试想一下，如果没有 ChannelHandlerContext 的这层封装，那么我们在做 ChannelHandler  之间传递的时候，前置后置的通用逻辑就要在每个 ChannelHandler 里都实现一份。这样虽然能解决问题，但是代码结构的耦合，会非常不优雅。

根据网络数据的流向，ChannelPipeline 分为入站 ChannelInboundHandler 和出站  ChannelOutboundHandler  两种处理器。在客户端与服务端通信的过程中，数据从客户端发向服务端的过程叫出站，反之称为入站。数据先由一系列 InboundHandler  处理后入站，然后再由相反方向的 OutboundHandler 处理完成后出站，如下图所示。我们经常使用的解码器 Decoder  就是入站操作，编码器 Encoder 就是出站操作。服务端接收到客户端数据需要先经过 Decoder 入站处理后，再通过 Encoder  出站通知客户端。

![](/assets/images/posts/netty-eventlop/netty-eventlop-2.png)

ChannelPipeline 的双向链表分别维护了 HeadContext 和 TailContext 的头尾节点。我们自定义的  ChannelHandler 会插入到 Head 和 Tail 之间，这两个节点在 Netty 中已经默认实现了，它们在  ChannelPipeline 中起到了至关重要的作用。

![](/assets/images/posts/netty-eventlop/netty-eventlop-3.png)

HeadContext 既是 Inbound 处理器，也是 Outbound 处理器。它分别实现了 ChannelInboundHandler 和 ChannelOutboundHandler。网络数据写入操作的入口就是由 HeadContext 节点完成的。HeadContext 作为  Pipeline 的头结点负责读取数据并开始传递 InBound 事件，当数据处理完成后，数据会反方向经过 Outbound 处理器，最终传递到 HeadContext，所以 HeadContext 又是处理 Outbound 事件的最后一站。此外 HeadContext  在传递事件之前，还会执行一些前置操作。

TailContext 只实现了 ChannelInboundHandler 接口。它会在 ChannelInboundHandler  调用链路的最后一步执行，主要用于终止 Inbound 事件传播，例如释放 Message 数据资源等。TailContext 节点作为  OutBound 事件传播的第一站，仅仅是将 OutBound 事件传递给上一个节点。

从整个 ChannelPipeline 调用链路来看，如果由 Channel 直接触发事件传播，那么调用链路将贯穿整个  ChannelPipeline。然而也可以在其中某一个 ChannelHandlerContext 触发同样的方法，这样只会从当前的  ChannelHandler 开始执行事件传播，该过程不会从头贯穿到尾，在一定场景下，可以提高程序性能。

# 2. 接口设计

整个 ChannelHandler 是围绕 I/O 事件的生命周期所设计的，例如建立连接、读数据、写数据、连接销毁等。ChannelHandler 有两个重要的**子接口**：**ChannelInboundHandler**和**ChannelOutboundHandler**，分别拦截**入站和出站的各种 I/O 事件**。

**ChannelInboundHandler 的事件回调方法与触发时机。**

- channelRegistered：Channel 被注册到 EventLoop
- channelUnregistered：Channel 从 EventLoop 中取消注册
- channelActive：Channel 处于就绪状态，可以被读写
- channelInactive：Channel 处于非就绪状态Channel 可以从远端读取到数据
- channelRead：Channel 可以从远端读取到数据
- channelReadComplete：Channel 读取数据完成
- userEventTriggered 	用户事件触发时
- channelWritabilityChanged：Channel 的写状态发生变化

**ChannelOutboundHandler 的事件回调方法与触发时机。**

直接通过 ChannelOutboundHandler  的接口列表可以看到每种操作所对应的回调方法，如下图所示。这里每个回调方法都是在相应操作执行之前触发。此外  ChannelOutboundHandler 中绝大部分接口都包含ChannelPromise 参数，以便于在操作完成时能够及时获得通知。

![](/assets/images/posts/netty-eventlop/netty-eventlop-4.png)

# 3. 事件传播机制

ChannelPipeline 可分为入站 ChannelInboundHandler 和出站 ChannelOutboundHandler 两种处理器，与此对应传输的事件类型可以分为**Inbound 事件**和**Outbound 事件**。

通过下面的代码

```
serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) {
        ch.pipeline()
                .addLast(new SampleInBoundHandler("SampleInBoundHandlerA", false))
                .addLast(new SampleInBoundHandler("SampleInBoundHandlerB", false))
                .addLast(new SampleInBoundHandler("SampleInBoundHandlerC", true));
        ch.pipeline()
                .addLast(new SampleOutBoundHandler("SampleOutBoundHandlerA"))
                .addLast(new SampleOutBoundHandler("SampleOutBoundHandlerB"))
                .addLast(new SampleOutBoundHandler("SampleOutBoundHandlerC"));

    }
}
public class SampleInBoundHandler extends ChannelInboundHandlerAdapter {
    private final String name;
    private final boolean flush;
    public SampleInBoundHandler(String name, boolean flush) {
        this.name = name;
        this.flush = flush;
    }
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("InBoundHandler: " + name);
        if (flush) {
            ctx.channel().writeAndFlush(msg);
        } else {
            super.channelRead(ctx, msg);
        }
    }
}

public class SampleOutBoundHandler extends ChannelOutboundHandlerAdapter {
    private final String name;
    public SampleOutBoundHandler(String name) {
        this.name = name;
    }
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        System.out.println("OutBoundHandler: " + name);
        super.write(ctx, msg, promise);
    }
}
```

通过 Pipeline 的 addLast 方法分别添加了三个 InboundHandler 和 OutboundHandler，添加顺序都是 A -> B -> C，下图可以表示初始化后 ChannelPipeline 的内部结构。

![](/assets/images/posts/netty-eventlop/netty-eventlop-5.png)

当客户端向服务端发送请求时，会触发 SampleInBoundHandler 调用链的 channelRead 事件。经过 SampleInBoundHandler 调用链处理完成后，在 SampleInBoundHandlerC 中会调用 writeAndFlush 方法向客户端写回数据，此时会触发 SampleOutBoundHandler 调用链的 write 事件。

```
InBoundHandler: SampleInBoundHandlerA
InBoundHandler: SampleInBoundHandlerB
InBoundHandler: SampleInBoundHandlerC
OutBoundHandler: SampleOutBoundHandlerC
OutBoundHandler: SampleOutBoundHandlerB
OutBoundHandler: SampleOutBoundHandlerA
```

可以看到Inbound 事件和 Outbound 事件的传播方向相反，Inbound 事件的传播方向为 Head -> Tail，而 Outbound 事件传播方向是 Tail -> Head。

如果我们在后面增加一个SampleInBoundHandlerD

```
.addLast(new SampleInBoundHandler("SampleInBoundHandlerD", false));
```

再次测试，我们会发现SampleInBoundHandlerD并没有执行。因为SampleInBoundHandlerC，并没有将事件传递给它。

# 4. 异常传播机制

我们通过修改 SampleInBoundHandler 的实现来模拟业务逻辑异常：

```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
	System.out.println("InBoundHandler: " + name);
	if (flush) {
		ctx.channel().writeAndFlush(msg);
	} else {
		throw new RuntimeException("InBoundHandler: " + name);
	}
}

@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
	System.out.println("InBoundHandlerException: " + name);
	ctx.fireExceptionCaught(cause);
}
```

由输出结果可以看出 ctx.fireExceptionCaugh 会将异常按顺序从 Head 节点传播到 Tail 节点。如果用户没有对异常进行拦截处理，最后将由 Tail 节点统一处理。

```
InBoundHandler: SampleInBoundHandlerA
InBoundHandlerException: SampleInBoundHandlerA
InBoundHandlerException: SampleInBoundHandlerB
InBoundHandlerException: SampleInBoundHandlerC
一月 15, 2021 2:06:11 下午 io.netty.channel.DefaultChannelPipeline onUnhandledInboundException
```

虽然 Netty 提供了兜底的异常处理逻辑，但是在很多场景下，并不能满足我们的需求。通过异常传播机制的学习，我们应该可以想到最好的方法是在 ChannelPipeline 自定义处理器的末端添加统一的异常处理器，此时 ChannelPipeline 的内部结构如下图所示。

![](/assets/images/posts/netty-eventlop/netty-eventlop-6.png)

加入统一的异常处理器后，可以看到异常已经被优雅地拦截并处理掉了。这也是 Netty 推荐的最佳异常处理实践。

```
InBoundHandler: SampleInBoundHandlerA
InBoundHandlerException: SampleInBoundHandlerA
InBoundHandlerException: SampleInBoundHandlerB
InBoundHandlerException: SampleInBoundHandlerC
Handle Business Exception Success.
```

异常事件的处理顺序与 ChannelHandler 的添加顺序相同，会依次向后传播，与 Inbound 事件和 Outbound 事件无关。