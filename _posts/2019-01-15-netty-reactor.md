---
layout: post
title: Netty - Reactor实现原理
date: 2019-01-15
categories:
    - netty
comments: true
permalink: netty-reactor.html
---

EventLoopGroup 是 Netty Reactor 线程模型的具体实现方式，Netty 通过创建不同的 EventLoopGroup 参数配置，就可以支持 Reactor 的三种线程模型：

- 单线程模型：EventLoopGroup 只包含一个 EventLoop，Boss 和 Worker 使用同一个EventLoopGroup；
- 多线程模型：EventLoopGroup 包含多个 EventLoop，Boss 和 Worker 使用同一个EventLoopGroup；
- 主从多线程模型：EventLoopGroup 包含多个 EventLoop，Boss 是主 Reactor，Worker 是从 Reactor，它们分别使用不同的 EventLoopGroup，主 Reactor 负责新的网络连接 Channel 创建，然后把 Channel 注册到从 Reactor。

在设置完EventLoopGroup之后，我们就会将它传给引导类

```
ServerBootstrap b = new ServerBootstrap();
// 配置线程池
b.group(bossGroup, workerGroup)
```

进入group方法内部，可以看到ServerBootstrap内部有2个EventLoopGroup

```
    public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
        super.group(parentGroup);
        if (this.childGroup != null) {
            throw new IllegalStateException("childGroup set already");
        }
        this.childGroup = ObjectUtil.checkNotNull(childGroup, "childGroup");
        return this;
    }
```

其中**主Reactor**（parent）是定义在AbstractBootstrap类中，我们看一下那个方法调用了它。通过IDE跟踪，我们可以发现AbstractBootstrap的**initAndRegister**方法调用了AbstractBootstrapConfig中的group方法来获取主Reactor，并与一个Channel绑定。

```
final ChannelFuture initAndRegister() {
	Channel channel = null;
	try {
		channel = channelFactory.newChannel();
		init(channel);
	} catch (Throwable t) {
	...
	}
	// 将EventloopGroup与Channel绑定
	ChannelFuture regFuture = config().group().register(channel);
	...
	return regFuture;
}
```

我们知道，Channel有很多种，那么这里绑定的Channel是哪个？

通过上面的代码可以看到，Channel是通过ChannelFactory来创建的`channelFactory.newChannel()`,而ChannelFactory是通过构造函数传入。

```
AbstractBootstrap(AbstractBootstrap<B, C> bootstrap) {
	...
	// 初始化channelFactory
	channelFactory = bootstrap.channelFactory;
	...
}
```

通过IDE跟踪，我们可以发现channelFactory是通过channelFactory方法来指定

```
public B channelFactory(io.netty.channel.ChannelFactory<? extends C> channelFactory) {
    return channelFactory((ChannelFactory<C>) channelFactory);
}
```

而channelFactory方法又是通过channel方法传入的Channel类找到对应的ChannelFactory

```
public B channel(Class<? extends C> channelClass) {
	// 
    return channelFactory(new ReflectiveChannelFactory<C>(
            ObjectUtil.checkNotNull(channelClass, "channelClass")
    ));
}
```

这个Channel类就是在我们在引导类中设置的NioServerSocketChannel.class

```
ServerBootstrap b = new ServerBootstrap();
// 配置线程池
b.group(bossGroup, workerGroup)
		.channel(NioServerSocketChannel.class)
```

ReflectiveChannelFactory直接通过反射创建一个NioServerSocketChannel对象

```
@Override
public T newChannel() {
    try {
        return constructor.newInstance();
    } catch (Throwable t) {
		...
    }
}
```

我们在回到ServerBootstrap方法中查找从 Reactor（childGroup）被哪个方法调用，结果发现AbstractBootstrapConfig中的childGroup方法并没有被任何地方调用。再在ServerBootstrap的源码中搜索childGroup，发现在**`init(Channel channel)`**方法中childGroup被赋值给了一个新的变量`EventLoopGroup currentChildGroup`，而currentChildGroup最终又被传入了**ServerBootstrapAcceptor**对象

```
@Override
void init(Channel channel) {
	...
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
			...
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                	// 传入childGroup
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

>  这里的childHandler就是引导类传入的childHandler
>
> ServerBootstrap b = new ServerBootstrap();
> b.group(bossGroup, workerGroup)
> 		.childHandler(new ChannelInitializer<Channel>() {
> 			protected void initChannel(Channel channel) throws Exception {
> 				channel.pipeline()
> 						.addLast("handler", new EchoServerHandler());
> 			}
> 		})

到ServerBootstrapAcceptor中看一下childGroup被谁调用了。

```
@Override
@SuppressWarnings("unchecked")
public void channelRead(ChannelHandlerContext ctx, Object msg) {
	final Channel child = (Channel) msg;
    child.pipeline().addLast(childHandler);
	...
    try {
    	// 绑定从Reactor和SocketChannel
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

可以看到在**channelRead**方法中，接收到传入的msg（其实就是SocketChannel），然后就把这个Channel和从Reactor绑定了。

我们知道从 Reactor可能会有多个（netty默认是cpu核数*2），那么**netty是如何选择一个Eventloop的呢**？

继续看`childGroup.register`方法的实现，它有好几个实现类

- SingleThreadEventLoop
- MultithreadEventLoopGroup
- ThreadPerChannelEventLoopGroup，用于OIO实现，已废弃

SingleThreadEventLoop的实现很简单，就是讲Channel和自己绑定

```
@Override
public ChannelFuture register(final ChannelPromise promise) {
	ObjectUtil.checkNotNull(promise, "promise");
	// 注意这里的this
	promise.channel().unsafe().register(this, promise);
	return promise;
}
```

我们重点看一下**MultithreadEventLoopGroup**的实现，可以发现它是通过**next**方法选择一个EventLoop

```
@Override
public ChannelFuture register(Channel channel) {
	// 选择一个EventLoop
	return next().register(channel);
}
```

next方法由父类**MultithreadEventExecutorGroup**实现

```
@Override
public EventLoop next() {
	// 调用父类的next方法
    return (EventLoop) super.next();
}
```

```
@Override
public EventExecutor next() {
	// 调用EventExecutorChooser的next方法
    return chooser.next();
}
```

而MultithreadEventExecutorGroup又是通过`EventExecutorChooser.next()`方法来实现的选择。EventExecutorChooser是由构造函数传入的**EventExecutorChooserFactory**创建

```
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
										EventExecutorChooserFactory chooserFactory, Object... args) {
	...
	chooser = chooserFactory.newChooser(children);
	..
}
```

进入EventExecutorChooserFactory的实现类**DefaultEventExecutorChooserFactory**可以看到这样一段逻辑

```
@Override
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}
```

Netty实现了两种选择器，我们先看**GenericEventExecutorChooser**，它就是通过递增取模的方式计算选择的Chooser

```
@Override
public EventExecutor next() {
	// 递增取模
    return executors[(int) Math.abs(idx.getAndIncrement() % executors.length)];
}
```

在看一下**PowerOfTwoEventExecutorChooser**，它也是取模计算，只不过Netty为了追求高性能，采用&的方式来进去取模。但这有一个前提就是executors的数量必须是2的N次方才行。

```
@Override
public EventExecutor next() {
    return executors[idx.getAndIncrement() & executors.length - 1];
}
```

所以在DefaultEventExecutorChooserFactory的newChooser方法中只有通过isPowerOfTwo方法判断executors的数量必须是2的N次方才会使用PowerOfTwoEventExecutorChooser。

了解了Reactor的线程模型后，主Reactor还需要通过 select 监控连接事件才能完成。下面我们看看一下这部分的实现。回到**NioEventLoopGroup**，在它的构造函数里可以发现这样一段

```
public NioEventLoopGroup(int nThreads, Executor executor) {
	// 多路选择器
    this(nThreads, executor, SelectorProvider.provider());
}
```

`SelectorProvider.provider()`就是用来指定当前使用的多路选择器是哪一个。SelectorProvider是JDK自带的SPI实现

进去看一下具体实现。

```
public static SelectorProvider provider() {
    synchronized (lock) {
        if (provider != null)
            return provider;
        return AccessController.doPrivileged(
            new PrivilegedAction<SelectorProvider>() {
                public SelectorProvider run() {
                        if (loadProviderFromProperty())
                            return provider;
                        if (loadProviderAsService())
                            return provider;
                        // 默认实现
                        provider = sun.nio.ch.DefaultSelectorProvider.create();
                        return provider;
                    }
                });
    }
}
```

发现是通过**sun.nio.ch.DefaultSelectorProvider**来创建的，在我的开发环境上是Windows实现

```
public class DefaultSelectorProvider {
    private DefaultSelectorProvider() {
    }

    public static SelectorProvider create() {
        return new WindowsSelectorProvider();
    }
}
```

