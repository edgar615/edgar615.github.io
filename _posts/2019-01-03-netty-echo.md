---
layout: post
title: Netty Hello World程序
date: 2019-01-03
categories:
    - netty
comments: true
permalink: netty-echo.html
---

# 1. Server

```
public class EchoServer {
    /**
     * 1. 创建ServerBootstrap实例，并制定NioEventLoopGroup来接收和处理新的连接
     * 2. 设置channel的类型为NioServerSocketChannel
     * 3. 设置地址端口，服务器将绑定这个地址以监听新的请求连接
     * 4. ChannelInitializer， 新的连接被接受时，一个新的子Channel被创建，ChannelInitializer会把你的
     *    EchoServerHandler添加到该Channel的ChannelPipeline中
     * 5. EchoServerHandler将会接收到信息
     * 6. 等待绑定完成，sync方法将当前Thread阻塞，一致到绑定操作完成为止
     * 7. 应用程序将会阻塞等待知道服务器的Channel关闭，因为你再Channel.CloseFuture调用了sync方法
     * 8. 关闭EventLoopGroup，释放所有资源，包括被创建的线程
     */
    public static void main(String[] args) {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            // 配置线程池
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(new InetSocketAddress(5000))
                    .childHandler(new ChannelInitializer<Channel>() {
                        protected void initChannel(Channel channel) throws Exception {
                            channel.pipeline()
                                    .addLast("handler", new EchoServerHandler());
                        }
                    }).childOption(ChannelOption.SO_KEEPALIVE, true);
            // start
            // 异步绑定服务器，调用sync方法阻塞等待知道绑定完成
            ChannelFuture future = b.bind().sync();
            // 获取Channel的CloseFuture，并且阻塞当前线程直到它完成
            future.channel().closeFuture().sync();
        } catch (Exception e) {
            // todo
            e.printStackTrace();
        }finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
}
```

```
// 标记一个ChannelHandler被多个Channel安全共享
@ChannelHandler.Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {

    /**
     * 每个传入的消息都要调用
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf in = (ByteBuf) msg;
        System.out.println("server :  "+ in.toString(CharsetUtil.UTF_8));
        // 接收的消息写给发送者，不冲刷出站
        ctx.write(in);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        // 将未决消息冲刷到远程节点，并且关闭该Channel
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        // 打印异常栈跟踪
        cause.printStackTrace();
        // 关闭Channel
        ctx.close();
    }
}
```

# 2. Client

```
public class EchoClient {
    public static void main(String[] args) {
        EventLoopGroup group = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<Channel>() {
                        protected void initChannel(Channel channel) throws Exception {
                            channel.pipeline().addLast(new EchoClientHandler());
                        }
                    });
            // Start the client.
            ChannelFuture f = bootstrap.connect("localhost", 5000).sync();
            f.channel().closeFuture().sync();
        } catch (Exception e) {
            // ignore
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

```
@ChannelHandler.Sharable
public class EchoClientHandler extends SimpleChannelInboundHandler<ByteBuf> {

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        // 当被通知Channel是活跃的时候，发送一条消息
        ctx.writeAndFlush(Unpooled.copiedBuffer("Netty socks", CharsetUtil.UTF_8));
    }

    /**
     * 1. 每次接收数据，都会调用这个方法
     * 2. 服务器发送的数据可能被分块接收，意味着这个方法会被调用多次
     */
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) throws Exception {
        // 接收消息
        System.out.println("Client : "+ byteBuf.toString(CharsetUtil.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        // 异常，记录错误，并关闭Channel
        cause.printStackTrace();
        ctx.close();
    }
}
```

# 3. 引导器

Netty 服务端的启动过程大致分为三个步骤：

- 配置线程池；
- Channel 初始化；
- 端口绑定。

## 3.1. 配置线程池

Netty 是采用 Reactor 模型进行开发的，可以非常容易切换三种 Reactor 模式：单线程模式、多线程模式、主从多线程模式。

**单线程模式**

Reactor 单线程模型所有 I/O 操作都由一个线程完成，所以只需要启动一个 EventLoopGroup 即可

```
EventLoopGroup group = new NioEventLoopGroup(1);
ServerBootstrap b = new ServerBootstrap();
b.group(group);
```

**多线程模式**

在 Netty 中使用 Reactor 多线程模型与单线程模型非常相似，区别是 NioEventLoopGroup 可以不需要任何参数，它默认会启动 2 倍 CPU 核数的线程。当然，你也可以自己手动设置固定的线程数。

```
EventLoopGroup group = new NioEventLoopGroup();
ServerBootstrap b = new ServerBootstrap();
b.group(group);
```

**主从多线程模式**

在大多数场景下，我们采用的都是**主从多线程 Reactor 模型**。Boss 是主 Reactor，Worker 是从 Reactor。它们分别使用不同的 NioEventLoopGroup，主 Reactor 负责处理 Accept，然后把 Channel 注册到从 Reactor 上，从 Reactor 主要负责 Channel 生命周期内的所有 I/O  事件。

```
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup);
```

## 3.2. Channel 初始化

**设置 Channel 类型**

NIO 模型是 Netty 中最成熟且被广泛使用的模型。因此，推荐 Netty 服务端采用 NioServerSocketChannel 作为 Channel 的类型，客户端采用 NioSocketChannel。设置方式如下：

```
b.channel(NioServerSocketChannel.class);
```

**注册 ChannelHandler**

在 Netty 中可以通过 ChannelPipeline 去注册多个 ChannelHandler，每个 ChannelHandler 各司其职，这样就可以实现最大化的代码复用，充分体现了 Netty 设计的优雅之处。

```
b.childHandler(new ChannelInitializer<Channel>() {
	protected void initChannel(Channel channel) throws Exception {
		channel.pipeline()
				.addLast("handler", new EchoServerHandler());
	}
}).
```

ServerBootstrap 的 childHandler() 方法需要注册一个 ChannelHandler。**ChannelInitializer**是实现了 ChannelHandler**接口的匿名类**，通过实例化 ChannelInitializer 作为 ServerBootstrap 的参数。

Channel 初始化时都会绑定一个 Pipeline，它主要用于服务编排。Pipeline 管理了多个 ChannelHandler。I/O 事件依次在 ChannelHandler 中传播，ChannelHandler 负责业务逻辑处理。例如一个HTTP Server使用链式的方式加载了多个 ChannelHandler，包含**HTTP 编解码处理器、HTTPContent 压缩处理器、HTTP 消息聚合处理器、自定义业务逻辑处理器**。

```
b.childHandler(new ChannelInitializer<SocketChannel>() {
	@Override
	public void initChannel(SocketChannel ch) {
		ch.pipeline()
				.addLast("codec", new HttpServerCodec())
				.addLast("compressor", new HttpContentCompressor())
				.addLast("aggregator", new HttpObjectAggregator(65536)) 
				.addLast("handler", new HttpServerHandler());
	}
})
```

当服务端收到 HTTP 请求后，会依次经过 HTTP 编解码处理器、HTTPContent 压缩处理器、HTTP 消息聚合处理器、自定义业务逻辑处理器分别处理后，再将最终结果通过 HTTPContent 压缩处理器、HTTP 编解码处理器写回客户端。

**设置 Channel 参数**

```
b.option(ChannelOption.SO_KEEPALIVE, true);
```

- SO_KEEPALIVE：设置为 true 代表启用了 TCP SO_KEEPALIVE 属性，TCP 会主动探测连接状态，即连接保活
- SO_BACKLOG：已完成三次握手的请求队列最大长度，同一时刻服务端可能会处理多个连接，在高并发海量连接的场景下，该参数应适当调大
- TCP_NODELAY：Netty 默认是 true，表示立即发送数据。如果设置为 false 表示启用 Nagle 算法，该算法会将 TCP 网络数据包累积到一定量才会发送，虽然可以减少报文发送的数量，但是会造成一定的数据延迟。Netty 为了最小化数据传输的延迟，默认禁用了 Nagle 算法
- SO_SNDBUF：TCP 数据发送缓冲区大小
- SO_RCVBUF：TCP数据接收缓冲区大小
- SO_LINGER：设置延迟关闭的时间，等待缓冲区中的数据发送完成
- CONNECT_TIMEOUT_MILLIS ：建立连接的超时时间

**端口绑定**

在完成上述 Netty 的配置之后，bind() 方法会真正触发启动，sync() 方法则会阻塞，直至整个启动过程完成，

```
ChannelFuture f = b.bind().sync();
```

