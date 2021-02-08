---
layout: post
title: Netty - 消息发送机制与高低水位
date: 2019-01-19
categories:
    - netty
comments: true
permalink: netty-write.html
---

# 1. OOM

在有些场景下，由于各种原因，会导致客户端消息发送积压，进而导致OOM。

- 1、当netty服务端并发压力过大，超过了服务端的处理能力时，channel中的消息服务端不能及时消费，这时channel堵塞，客户端消息就会堆积在发送队列中
- 2、网络瓶颈，当客户端发送速度超过网络链路处理能力，会导致客户端发送队列积压
- 3、当对端读取速度小于己方发送速度，导致自身TCP发送缓冲区满，频繁发生write 0字节时，待发送消息会在netty发送队列中排队

这三种情况下，如果客户端没有流控保护，这时候就很容易发生内存泄露。

**原因：**

在我们调用channel的write和writeAndFlush时
 io.netty.channel.AbstractChannelHandlerContext#writeAndFlush(java.lang.Object, io.netty.channel.ChannelPromise)，如果发送方为业务线程，则将发送操作封装成WriteTask（继承Runnable），放到Netty的NioEventLoop中执行，当NioEventLoop无法完成如此多的消息的发送的时候，发送任务队列积压，进而导致内存泄漏。

**解决方案：**

为了防止在高并发场景下，由于服务端处理慢导致的客户端消息积压，客户端需要做并发保护，防止自身发生消息积压。Netty提供了一个**高低水位机制，可以实现客户端精准的流控**。

io.netty.channel.ChannelConfig#setWriteBufferHighWaterMark 高水位
 io.netty.channel.ChannelConfig#setWriteBufferLowWaterMark  低水位

当发送队列待发送的字节数组达到高水位时，对应的channel就变为不可写状态，由于高水位并不影响业务线程调用write方法把消息加入到待发送队列，**因此在消息发送时要先对channel的状态进行判断（ctx.channel().isWritable）。**

# 2. **netty的消息发送机制**

业务调用write方法后，经过ChannelPipeline职责链处理，消息被投递到发送缓冲区待发送，调用flush之后会执行真正的发送操作，底层通过调用Java NIO的SocketChannel进行非阻塞write操作，将消息发送到网络上，

![](/assets/images/posts/netty-write/netty-write-1.png)

当用户线程（业务线程）发起write操作时，Netty会进行判断，如果发现不是NioEventLoop（I/O线程），则将发送消息封装成WriteTask，放入NioEventLoop的任务队列，由NioEventLoop线程执行，代码如下

```
private void write(Object msg, boolean flush, ChannelPromise promise) {
    ...
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        final WriteTask task = WriteTask.newInstance(next, m, promise, flush);
        if (!safeExecute(executor, task, promise, m, !flush)) {
            // We failed to submit the WriteTask. We need to cancel it so we decrement the pending bytes
            // and put it back in the Recycler for re-use later.
            //
            // See https://github.com/netty/netty/issues/8343.
            task.cancel();
        }
    }
}
```

Netty的NioEventLoop线程内部维护了一个`Queue<Runnable> taskQuue`，除了处理网络IO读写操作，同时还负责执行网络读写相关的Task，NioEventLoop遍历taskQueue，执行消息发送任务。

```
void invokeWrite(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
    } else {
        write(msg, promise);
    }
}

private void invokeWrite0(Object msg, ChannelPromise promise) {
    try {
        ((ChannelOutboundHandler) handler()).write(this, msg, promise);
    } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
    }
}
```

数据会在 Pipeline 中一直寻找 Outbound 节点并向前传播，直到 Head 节点结束，由 Head 节点完成最后的数据发送。

```
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    unsafe.write(msg, promise);
}

@Override
public void flush(ChannelHandlerContext ctx) {
    unsafe.flush();
}
```

Head 节点是通过调用 unsafe 对象完成数据写入的，unsafe 对应的是 NioSocketChannelUnsafe 对象实例，最终调用到 AbstractChannel 中的 write 方法

```
@Override
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        try {
            ReferenceCountUtil.release(msg);
        } finally {
            safeSetFailure(promise,
                    newClosedChannelException(initialCloseCause, "write(Object, ChannelPromise)"));
        }
        return;
    }

    int size;
    try {
        msg = filterOutboundMessage(msg);
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        try {
            ReferenceCountUtil.release(msg);
        } finally {
            safeSetFailure(promise, t);
        }
        return;
    }

    outboundBuffer.addMessage(msg, size, promise);
}
```

- filterOutboundMessage 方法会对待写入的 msg 进行过滤，如果 msg 使用的不是 DirectByteBuf，那么它会将 msg 转换成 DirectByteBuf。
- ChannelOutboundBuffer 可以理解为一个缓存结构，从源码最后一行 outboundBuffer.addMessage 可以看出是在向这个缓存中添加数据，所以 ChannelOutboundBuffer 才是理解数据发送的关键。

writeAndFlush 主要分为两个步骤，write 和 flush。通过上面的分析可以看出只调用 write 方法，数据并不会被真正发送出去，而是存储在 ChannelOutboundBuffer 的缓存内。

ChannelOutboundBuffer是一个链表结构，它包含三个非常重要的指针：第一个被写到缓冲区的节点 flushedEntry、第一个未被写到缓冲区的节点 unflushedEntry和最后一个节点 tailEntry。

```
// Entry(flushedEntry) --> ... Entry(unflushedEntry) --> ... Entry(tailEntry)
//
// The Entry that is the first in the linked-list structure that was flushed
private Entry flushedEntry;
// The Entry which is the first unflushed in the linked-list structure
private Entry unflushedEntry;
// The Entry which represents the tail of the buffer
private Entry tailEntry;
// The number of flushed entries that are not written yet
```

```
public void addMessage(Object msg, int size, ChannelPromise promise) {
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
    }
    tailEntry = entry;
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }

    incrementPendingOutboundBytes(entry.pendingSize, false);
}
```

在初始状态下这三个指针都指向 NULL，当我们每次调用 write 方法是，都会调用 addMessage 方法改变这三个指针的指向，可以参考下图理解指针的移动过程会更加形象。

![](/assets/images/posts/netty-write/netty-write-2.png)

第一次调用 write，因为链表里只有一个数据，所以 unflushedEntry 和 tailEntry 指针都指向第一个添加的数据 msg1。flushedEntry 指针在没有触发 flush 动作时会一直指向 NULL。

第二次调用 write，tailEntry 指针会指向新加入的 msg2，unflushedEntry 保持不变。

第 N 次调用 write，tailEntry 指针会不断指向新加入的 msgN，unflushedEntry 依然保持不变，unflushedEntry 和 tailEntry 指针之间的数据都是未写入 Socket 缓冲区的。

以上便是写 Buffer 队列写入数据的实现原理，但是我们不可能一直向缓存中写入数据，**所以 addMessage 方法中每次写入数据后都会调用 incrementPendingOutboundBytes 方法判断缓存的水位线**

```
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
    if (size == 0) {
        return;
    }

    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
    // 判断缓存大小是否超过高水位线
    if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
        setUnwritable(invokeLater);
    }
}
```

incrementPendingOutboundBytes 的逻辑非常简单，每次添加数据时都会累加数据的字节数，然后判断缓存大小是否超过所设置的高水位线 64KB，如果超过了高水位，那么 Channel 会被设置为不可写状态。直到缓存的数据大小低于低水位线 32KB 以后，Channel 才恢复成可写状态。

在看一下flush方法

```
public void addFlush() {
    // There is no need to process all entries if there was already a flush before and no new messages
    // where added in the meantime.
    //
    // See https://github.com/netty/netty/issues/2577
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            // there is no flushedEntry yet, so start with the entry
            flushedEntry = entry;
        }
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) {
                // Was cancelled so make sure we free up memory and notify about the freed bytes
                int pending = entry.cancel();
                // 减去待发送的数据，如果总字节数低于低水位，那么 Channel 将变为可写状态
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);

        // All flushed so reset unflushedEntry
        unflushedEntry = null;
    }
}
```

addFlush 方法同样也会操作 ChannelOutboundBuffer 缓存数据。在执行 addFlush 方法时，缓存中的指针变化又是如何呢？如下图所示，我们在写入流程的基础上继续进行分析。

![](/assets/images/posts/netty-write/netty-write-3.png)

此时 flushedEntry 指针有所改变，变更为 unflushedEntry 指针所指向的数据，然后 unflushedEntry 指针指向 NULL，flushedEntry 指针指向的数据才会被真正发送到 Socket 缓冲区。

在 addFlush 源码中 decrementPendingOutboundBytes 与之前 addMessage 源码中的 incrementPendingOutboundBytes 是相对应的。decrementPendingOutboundBytes 主要作用是减去待发送的数据字节，如果缓存的大小已经小于低水位，那么 Channel 会恢复为可写状态。

```
private void decrementPendingOutboundBytes(long size, boolean invokeLater, boolean notifyWritability) {
    if (size == 0) {
        return;
    }

    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, -size);
    if (notifyWritability && newWriteBufferSize < channel.config().getWriteBufferLowWaterMark()) {
        setWritable(invokeLater);
    }
}
```

# 3. 水位用法

在实际项目中，根据业务QPS规划，客户端处理性能、网络带宽、链路数、消息平均码流大小等综合因数，设置Netty高水位（setWriteBufferHighWaterMark）值，可以防止在发送队列处于高水位时继续发送消息，导致积压更严重，甚至发生内存泄漏。在系统中合理利用Netty的高低水位机制做消息发送的流控，既可以保护自身，同时又能减轻服务端的压力，可以提升系统的可靠性。

> Netty中提供 了writeBufferLowWaterMark和writeBufferHighWaterMark选项用来控制高低水位。可以通过监控当前写缓冲区的水位状况，来避免占用大量的内存，因为ChannelOutboundBuffer本身是无界的，

ChannelConfig默认的水位配置为低水位32K，高水位64K，如果用户没有配置就会使用默认配置。

设置水位

```
childOption(ChannelOption.WRITE_BUFFER_WATER_MARK, new WriteBufferWaterMark(32 * 1024, 64 * 1024));
```

同时在业务发送消息时，添加socketChannel.isWritable()是否可以发送判断

```
if (ctx.channel().isWritable()) {
    ctx.write(in);
} else {
    // TODO,记录日志，消息可能会被丢弃
}
```

# 4. 参考资料

《Netty 核心原理剖析与 RPC 实践》

