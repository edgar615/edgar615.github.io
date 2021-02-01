---
layout: post
title: Netty - keepalive
date: 2019-01-16
categories:
    - netty
comments: true
permalink: netty-keepalive.html
---

# 1.TCP保活（keepalive）

https://edgar615.github.io/tcp-keepalive.html

# 2. Netty开启TCP keepalive

在Server端开启TCP keepalive:　两种方式

```
serverBootstrap.childOption(ChannelOption.SO_KEEPALIVE, true);
serverBootstrap.childOption(NioChannelOption.SO_KEEPALIVE,true)
```

**提示：“.option(ChannelOption.SO_KEEPALIVE,true)”存在，但是无效。**

**这两种实现有什么区别呢？**

我们在ServerBootstrap引导类中找到对childOption的使用

```
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ...
    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);
	...
}
```

逐步分析，发现它最终在NioServerSocketChannel的setOption进行了处理

```
public <T> boolean setOption(ChannelOption<T> option, T value) {
	// NioChannelOption.SO_KEEPALIVE
    if (PlatformDependent.javaVersion() >= 7 && option instanceof NioChannelOption) {
        return NioChannelOption.setOption(jdkChannel(), (NioChannelOption<T>) option, value);
    }
    // ChannelOption.SO_KEEPALIVE
    return super.setOption(option, value);
}
```

NioChannelOption.SO_KEEPALIVE处理如下

```
@SuppressJava6Requirement(reason = "Usage guarded by java version check")
static <T> boolean setOption(Channel jdkChannel, NioChannelOption<T> option, T value) {
	...
    try {
        channel.setOption(option.option, value);
        return true;
    } catch (IOException e) {
        throw new ChannelException(e);
    }
}
```

ChannelOption.SO_KEEPALIVE的实现如下

```
@Override
public <T> boolean setOption(ChannelOption<T> option, T value) {
    validate(option, value);

    if (option == SO_RCVBUF) {
        setReceiveBufferSize((Integer) value);
    } else if (option == SO_REUSEADDR) {
        setReuseAddress((Boolean) value);
    } else if (option == SO_BACKLOG) {
        setBacklog((Integer) value);
    } else {
        return super.setOption(option, value);
    }

    return true;
}
```

可以看到ChannelOption.SO_KEEPALIVE是写一堆if...else来确定的，不太优雅。

# 3. IdleStateHandler

Netty提供了对心跳机制的天然支持，心跳可以检测远程端是否存活，或者活跃

IdleStateHandler通过构造参数，提供了三种不同风格的Idle监测。

```
public IdleStateHandler(
        long readerIdleTime, long writerIdleTime, long allIdleTime,
        TimeUnit unit) {
    this(false, readerIdleTime, writerIdleTime, allIdleTime, unit);
}
```

- readerIdleTimeSeconds: 读超时. 即当在指定的时间间隔内没有从 `Channel` 读取到数据时, 会触发一个 `READER_IDLE` 的 `IdleStateEvent` 事件.
- writerIdleTimeSeconds: 写超时. 即当在指定的时间间隔内没有数据写入到 `Channel` 时, 会触发一个 `WRITER_IDLE` 的 `IdleStateEvent` 事件.
- allIdleTimeSeconds: 读/写超时. 即当在指定的时间间隔内没有读或写操作时, 会触发一个 `ALL_IDLE` 的 `IdleStateEvent` 事件.

**Server示例**

```
// 设定IdleStateHandler心跳检测每五秒进行一次读检测，如果五秒内ChannelRead()方法未被调用则触发一次userEventTrigger()方法
channel.pipeline()
        .addLast(new IdleStateHandler(5, 0, 0, TimeUnit.SECONDS))
        .addLast(new ServerIdleStateEventHandler());
```

```
public class ServerIdleStateEventHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleState state = ((IdleStateEvent) evt).state();
            System.out.println(state);
            if (state == IdleState.READER_IDLE) {
                // 在规定时间内没有收到客户端的上行数据, 主动断开连接
                ctx.disconnect();
//                ctx.channel().close();
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
}
```

启动EchoClient后，发现过了5秒就自动关闭了，因为server触发了READER_IDLE 。

**Client示例**

```
channel.pipeline()
        .addLast(new IdleStateHandler(0, 2, 0))
        .addLast(new ClientIdleStateEventHandler());
```

```
public class ClientIdleStateEventHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleState state = ((IdleStateEvent) evt).state();
            System.out.println(state);
            if (state == IdleState.WRITER_IDLE) {
                // 写超时，发送一个心跳
                ctx.writeAndFlush(Unpooled.copiedBuffer("heart beat", CharsetUtil.UTF_8));
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
}
```

再次启动EchoClient后，EchoClient不会再被关闭，Server会一直打印心跳信息。

# 4. 断线重连

客户端在监测到与服务器端的连接断开后，或者一开始就无法连接的情况下，使用指定的重连策略进行重连操作，直到重新建立连接或重试次数耗尽。

**启动时连接重试**

```
public class ConnectionListener implements ChannelFutureListener {
    private Client client;

    public ConnectionListener(Client client) {
        this.client = client;
    }

    @Override
    public void operationComplete(ChannelFuture channelFuture) throws Exception {
        //通过channelFuture.isSuccess()可以知道在连接的时候是成功了还是失败了，
        //如果失败了我们就启动一个单独的线程来执行重新连接的操作。
        if (!channelFuture.isSuccess()) {
            System.out.println("Reconnect");
            final EventLoop loop = channelFuture.channel().eventLoop();
            loop.schedule(new Runnable() {
                @Override
                public void run() {
                    client.createBootstrap(new Bootstrap(), loop);
                }
            }, 1L, TimeUnit.SECONDS);
        }
    }
}
```

```
public class Client {
    private EventLoopGroup loop = new NioEventLoopGroup();

    public static void main(String[] args) {
        new Client().run();
    }

    public Bootstrap createBootstrap(Bootstrap bootstrap, EventLoopGroup eventLoop) {
        if (bootstrap != null) {
            final MyInboundHandler handler = new MyInboundHandler(this);
            bootstrap.group(eventLoop);
            bootstrap.channel(NioSocketChannel.class);
            bootstrap.option(ChannelOption.SO_KEEPALIVE, true);
            bootstrap.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel socketChannel) throws Exception {
                    socketChannel.pipeline().addLast(handler);
                }
            });
            bootstrap.remoteAddress("localhost", 8888);
            bootstrap.connect().addListener(new ConnectionListener(this));
        }
        return bootstrap;
    }

    public void run() {
        createBootstrap(new Bootstrap(), loop);
    }
}
```

**运行中断线重连**

直接参考官方例子

https://github.com/netty/netty/tree/4.1/example/src/main/java/io/netty/example/uptime

# 5. 原理

查看IdleStateHandler的源码，我们可以在handlerAdded、channelRegistered和channelActive中找到初始化方法

```
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    // This method will be invoked only if this handler was added
    // before channelActive() event is fired.  If a user adds this handler
    // after the channelActive() event, initialize() will be called by beforeAdd().
    initialize(ctx);
    super.channelActive(ctx);
}
```

initialize方法的作用是根据配置的readerIdleTime，WriteIdleTIme等超时事件参数往任务队列taskQueue中添加定时任务task ；

```
private void initialize(ChannelHandlerContext ctx) {
	...
    lastReadTime = lastWriteTime = ticksInNanos();
    if (readerIdleTimeNanos > 0) {
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                writerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                allIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
}
```

我们在看一下**ReaderIdleTimeoutTask**类

```
    @Override
    protected void run(ChannelHandlerContext ctx) {
        long nextDelay = readerIdleTimeNanos;
        if (!reading) {
        	// 计算idle的关键
        	// readerIdleTimeNanos - （当前时间 - 上次读取时间）
            nextDelay -= ticksInNanos() - lastReadTime;
        }
		// 发生了READER_IDLE
        if (nextDelay <= 0) {
            // Reader is idle - set a new timeout and notify the callback.
            // 重新加入定时任务，任务时间用默认的超时时间readerIdleTimeNanos
            readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);

            boolean first = firstReaderIdleEvent;
            firstReaderIdleEvent = false;

            try {
            	// 封装一个事件
                IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
                // 实际上是调用的ctx.fireUserEventTriggered(evt);
                channelIdle(ctx, event);
            } catch (Throwable t) {
                ctx.fireExceptionCaught(t);
            }
        } else {
            // Read occurred before the timeout - set a new timeout with shorter delay.
            // 如果没有超时，重新加入定时任务，任务时间用距离超时的时间间隔
            readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
        }
    }
}
```

WriterIdleTimeoutTask和AllIdleTimeoutTask基本类似

Netty还直接提供了ReadTimeoutHandler，它继承自IdleStateHandler，在收到READER_IDLE后直接抛出异常

```
@Override
protected final void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
    assert evt.state() == IdleState.READER_IDLE;
    readTimedOut(ctx);
}

/**
 * Is called when a read timeout was detected.
 */
protected void readTimedOut(ChannelHandlerContext ctx) throws Exception {
    if (!closed) {
        ctx.fireExceptionCaught(ReadTimeoutException.INSTANCE);
        ctx.close();
        closed = true;
    }
}
```

**WriteTimeoutHandler**的作用与ReadTimeoutHandler不同，它是用来检测向channe中写入数据时是否超时的。

```
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    if (timeoutNanos > 0) {
        promise = promise.unvoid();
        // 写入前启动一个定时任务
        scheduleTimeout(ctx, promise);
    }
    ctx.write(msg, promise);
}
```

```
private final class WriteTimeoutTask implements Runnable, ChannelFutureListener {
	...
	@Override
	public void run() {
		// 如果写任务没完成，抛出WriteTimeoutException异常
		if (!promise.isDone()) {
			try {
				writeTimedOut(ctx);
			} catch (Throwable t) {
				ctx.fireExceptionCaught(t);
			}
		}
		removeWriteTimeoutTask(this);
	}
	...

}
```