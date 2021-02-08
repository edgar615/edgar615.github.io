---
layout: post
title: Netty - flush优化
date: 2019-01-19
categories:
    - netty
comments: true
permalink: netty-flush.html
---

最早的时候我们想客户端写入数据的方式如下：

```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ctx.writeAndFlush(...);
}
```

使用**ctx.writeAndFlush()**这种方式为相当于加急式快递，每次write后就调用Flush，这样会对吞吐量有影响。

**改进方式1：**我们可以在read的时候write，在readComplete时flush，这样就减少了flush次数。

```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ctx.write(...);
}

@Override
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    ctx.writeAndFlush(Unpooled.EMPTY_BUFFER);
}
```

但这样还是存在2个缺点

1. 不适合异步业务线程。因为channelRead中的业务处理结果的write可能发生在channelReadComplete之后（我们一般会将业务处理单独放在一个业务线程池里异步实现）
2. 不适合精细控制，例如连续16次读时，flush一次，不能保持连续的次数不变

**改进方式2：**利用**FlushConsolidationHandler**减少flush使用   

```
channel.pipeline().addLast(new FlushConsolidationHandler(5, true))
```

**使用FlushConsolidationHandler提升吞吐量会增加一部分延迟，需要根据实际情况抉择**

如果异步处理的时候，即channelReadComplete比channelRead结束的要早，所以在flush调用的时候，readInProgress已经是false了，然后根据用户决定是否开启对于异步的强化来决定是直接flush还是走consolidateWhenNoReadInProgress，如果consolidateWhenNoReadInProgress为true（开启增强），那么就用次数是否到达阈值来决定立即刷新还是延时刷新。

```
@Override
public void flush(ChannelHandlerContext ctx) throws Exception {
    if (readInProgress) {
        // If there is still a read in progress we are sure we will see a channelReadComplete(...) call. Thus
        // we only need to flush if we reach the explicitFlushAfterFlushes limit.
        if (++flushPendingCount == explicitFlushAfterFlushes) {
            flushNow(ctx);
        }
    } else if (consolidateWhenNoReadInProgress) {
        // Flush immediately if we reach the threshold, otherwise schedule
        if (++flushPendingCount == explicitFlushAfterFlushes) {
            flushNow(ctx);
        } else {
            scheduleFlush(ctx);
        }
    } else {
        // Always flush directly
        flushNow(ctx);
    }
}
```

FlushConsolidationHandler在channelRead的时候将readInProgress设置为true，代表正在读，还没有执行channelReadComplete方法，此时flush是同步，因为还没到channelReadComplete的时候。

```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    readInProgress = true;
    ctx.fireChannelRead(msg);
}
```

然后channelReadComplete方法中把它又置为false，说明channelReadComplete方法执行完了，此后的flush都是异步了，因为业务执行慢，错过了channelReadComplete。

```
@Override
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    // This may be the last event in the read loop, so flush now!
    resetReadAndFlushIfNeeded(ctx);
    ctx.fireChannelReadComplete();
}

private void resetReadAndFlushIfNeeded(ChannelHandlerContext ctx) {
	readInProgress = false;
	flushIfNeeded(ctx);
}
```