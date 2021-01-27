---
layout: post
title: Netty - 服务端启动流程
date: 2019-01-11
categories:
    - netty
comments: true
permalink: netty-server-bind.html
---

> 未完成，肝不动了

在服务端启动之前，需要配置 ServerBootstrap 的相关参数，这一步大致可以分为以下几个步骤：

- 配置 EventLoopGroup 线程组；
- 配置 Channel 的类型；
- 设置 ServerSocketChannel 对应的 Handler；

- 设置网络监听的端口；

- 设置 SocketChannel 对应的 Handler；

- 配置 Channel 参数。

```
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();

ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
		.channel(NioServerSocketChannel.class)
		.localAddress(new InetSocketAddress(5000))
		.childHandler(new ChannelInitializer<Channel>() {
			protected void initChannel(Channel channel) throws Exception {
				channel.pipeline()
						.addLast("handler", new EchoServerHandler());
			}
		}).childOption(ChannelOption.SO_KEEPALIVE, true);
```

在配置完成后，我们通过`b.bind()`进行服务器端口绑定和启动，通过sync() 阻塞等待服务器启动完成.

```
// 异步绑定服务器，调用sync方法阻塞等待知道绑定完成
ChannelFuture future = b.bind().sync();
// 获取Channel的CloseFuture，并且阻塞当前线程直到它完成
future.channel().closeFuture().sync();
```

接下来我们对 bind() 方法进行展开分析。进入bind方法，可以看到最终是通过doBind方法实现绑定逻辑。

```
private ChannelFuture doBind(final SocketAddress localAddress) {
	// 调用 initAndRegister() 初始化并注册 Channel，同时返回一个 ChannelFuture 实例 regFuture
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    // 如果发生异常则直接返回。
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        // 如果执行完毕则调用 doBind0() 进行 Socket 绑定
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // 如果 initAndRegister() 还没有执行结束，增加一个监听器，在initAndRegister完成后调用doBind0() 进行 Socket 绑定
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
            	...
                doBind0(regFuture, channel, localAddress, promise);
                ...
            }
        });
        return promise;
    }
}
```

我们着重看两个方法 **initAndRegister**和**doBind0**

initAndRegister主要做了3件事

- 创建 Channel
- 初始化 Channel 
- 注册 Channel

```
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
    	// 创建一个Channel
        channel = channelFactory.newChannel();
        // 初始化Channel
        init(channel);
    } catch (Throwable t) {
        ...
    }
	// 注册Channel
    ChannelFuture regFuture = config().group().register(channel);
	...
    return regFuture;
}
```

**创建 Channel**是通过ChannelFactory创建，我们在ServerBootstrap上通过channel方法设置的是NioServerSocketChannel。

NioServerSocketChannel的构造方法使用Java的`SelectorProvider.openServerSocketChannel`方法创建了服务端的 ServerSocketChannel

```
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();

private static ServerSocketChannel newSocket(SelectorProvider provider) {
	try {
		return provider.openServerSocketChannel();
	} catch (IOException e) {
		throw new ChannelException(
				"Failed to open a server socket.", e);
	}
}

public NioServerSocketChannel() {
	this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}

public NioServerSocketChannel(ServerSocketChannel channel) {
	// SelectionKey
	super(null, channel, SelectionKey.OP_ACCEPT);
	config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

回到 NioServerSocketChannel 的构造函数，接着它会通过 super() 依次调用到父类的构造进行初始化工作，

```
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
	super(parent);
	...
	// 将channel标记为非阻塞
	ch.configureBlocking(false);
	...
}
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    // 全局唯一 id 
    id = newId();
    // unsafe 操作底层读写对象
    unsafe = newUnsafe();
    // pipeline 负责业务处理器编排
    pipeline = newChannelPipeline();
}
```

AbstractChannel 的构造函数创建了三个重要的成员变量，分别为 id、unsafe、pipeline。id 表示全局唯一的 Channel，unsafe 用于操作底层数据的读写操作，pipeline 负责业务处理器的编排。

Channel创建后，接着就需要**初始化**，init方法是一个抽象方法。

```
void init(Channel channel) {
	// 设置属性
    setChannelOptions(channel, newOptionsArray(), logger);
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

    ChannelPipeline p = channel.pipeline();
	// 获取 ServerBootstrapAcceptor 的构造参数
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);
	// 添加特殊的 Handler 处理器
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }
			// 添加一个ServerBootstrapAcceptor处理器，用来接受客户端连接
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

**注册服务端 Channel**

回到 initAndRegister() 的主流程，创建完服务端 Channel 之后，会选择一个Eventlop注册Channel。

```
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
	...
    AbstractChannel.this.eventLoop = eventLoop;
	// eventLoop 线程内部调用
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
       ...
		//外部线程调用
		eventLoop.execute(new Runnable() {
			@Override
			public void run() {
				register0(promise);
			}
		});
        ...
    }
}
```

Netty 会在线程池 EventLoopGroup 中选择一个 EventLoop 与当前 Channel 进行绑定，之后 Channel 生命周期内的所有 I/O 事件都由这个 EventLoop 负责处理，如 accept、connect、read、write 等 I/O 事件。可以看出，不管是 EventLoop 线程本身调用，还是外部线程用，最终都会通过 register0() 方法进行注册：

```
private void register0(ChannelPromise promise) {
	...
	// 注册
	doRegister();
	...

	// 触发 handlerAdded 事件
	pipeline.invokeHandlerAddedIfNeeded();

	safeSetSuccess(promise);
	// 触发 channelRegistered 事件
	pipeline.fireChannelRegistered();
	// 此时 Channel 还未注册绑定地址，所以处于非活跃状态
	if (isActive()) {
		if (firstRegistration) {
			// Channel 当前状态为活跃时，触发 channelActive 事件
			pipeline.fireChannelActive();
		} else if (config().isAutoRead()) {
			// This channel was registered before and autoRead() is set. This means we need to begin read
			// again so that we process inbound data.
			//
			// See https://github.com/netty/netty/issues/4805
			beginRead();
		}
	}
	...
}
```

register0() 主要做了四件事：调用 JDK 底层进行 Channel 注册、触发 handlerAdded 事件、触发 channelRegistered 事件、Channel 当前状态为活跃时，触发 channelActive 事件。