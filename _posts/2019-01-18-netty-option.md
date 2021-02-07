---
layout: post
title: Netty - 参数设置说明
date: 2019-01-18
categories:
    - netty
comments: true
permalink: netty-option.html
---

> option主要是针对boss线程组ServerSocketChannel，childOption主要是针对worker线程组，SocketChannel

# 1. 系统参数

## 2.1. SocketChannel

- **SO_SNDBUF** TCP数据发送缓冲区大小。该缓冲区即TCP发送滑动窗口，linux操作系统可使用命令：`cat /proc/sys/net/ipv4/tcp_smem`查询其大小。
- **SO_RCVBUF** TCP数据接收缓冲区大小。该缓冲区即TCP接收滑动窗口，linux操作系统可使用命令：`cat /proc/sys/net/ipv4/tcp_rmem`查询其大小。一般情况下，该值可由用户在任意时刻设置，但当设置值超过64KB时，需要在连接到远端之前设置。
- **SO_KEEPALIVE** 连接保活，默认值为False。启用该功能时，TCP会主动探测空闲连接的有效性。可以将此功能视为TCP的心跳机制，需要注意的是：默认的心跳间隔是7200s即2小时。Netty默认关闭该功能。
- **SO_REUSEADDR** 地址复用，默认值False。**不是让TCP绑定到同一个IP+PORT来重复启动。**有四种情况可以使用：
  - (1).当有一个有相同本地地址和端口的socket1处于TIME_WAIT状态时，而你希望启动的程序的socket2要占用该地址和端口，比如重启服务且保持先前端口。
  - (2).有多块网卡或用IP Alias技术的机器在同一端口启动多个进程，但每个进程绑定的本地IP地址不能相同。
  - (3).单个进程绑定相同的端口到多个socket上，但每个socket绑定的ip地址不同。
  - (4).完全相同的地址和端口的重复绑定。但这只用于UDP的多播，不用于TCP。
- **SO_LINGER** 关闭Socket的延迟时间，默认值为-1，表示禁用该功能。-1表示socket.close()方法立即返回，但OS底层会将发送缓冲区全部发送到对端。0表示socket.close()方法立即返回，OS放弃发送缓冲区的数据直接向对端发送RST包，对端收到复位错误。非0整数值表示调用socket.close()方法的线程被阻塞直到延迟时间到或发送缓冲区中的数据发送完毕，若超时，则对端会收到复位错误。
- **IP_TOS** 设置IP头部的Type-of-Service字段，用于描述IP包的优先级和QoS选项。
- **TCP_NODELAY** 而该参数的作用就是禁止使用Nagle算法，使用于小数据即时传输。和TCP_NODELAY相对应的是TCP_CORK，该选项是需要等到发送的数据量最大的时候，一次性发送数据，适用于文件传输。如果需要发送一些较小的报文，需要禁用Nagle算法

## 2.1. ServerSocketChannel

- **SO_RCVBUF** 为accept创建的socket channel设置SO_RCVBUF
- **SO_REUSEADDR** 地址复用，默认值False。
- **SO_BACKLOG** 服务端接受连接的队列长度，如果队列已满，客户端连接将被拒绝。默认值128。

# 2. Netty核心参数

- **WRITE_BUFFER_HIGH_WATER_MARK** 写高水位标记，默认值64KB。如果Netty的写缓冲区中的字节超过该值，Channel的isWritable()返回False。每个连接一个，所以不能设太大。
- **WRITE_BUFFER_LOW_WATER_MARK **写低水位标记，默认值32KB。当Netty的写缓冲区中的字节超过高水位之后若下降到低水位，则Channel的isWritable()返回True。写高低水位标记使用户可以控制写入数据速度，从而实现流量控制。推荐做法是：每次调用channl.write(msg)方法首先调用channel.isWritable()判断是否可写。
- **CONNECT_TIMEOUT_MILLIS** 连接超时毫秒数，默认值30000毫秒即30秒。
- **MAX_MESSAGES_PER_READ** 一次Loop读取的最大消息数，对于ServerChannel或者NioByteChannel，默认值为16，其他Channel默认值为1。默认值这样设置，是因为：ServerChannel需要接受足够多的连接，保证大吞吐量，NioByteChannel可以减少不必要的系统调用select。**控制连读功能：最大运行“连续”读次数**
- **WRITE_SPIN_COUNT** 一个Loop写操作执行的最大次数，默认值为16。也就是说，对于大数据量的写操作至多进行16次，如果16次仍没有全部写完数据，此时会提交一个新的写任务给EventLoop，任务将在下次调度继续执行。这样，其他的写请求才能被响应不会因为单个大数据量写请求而耽误。**控制连写功能：最大允许“连续”写次数**
- **ALLOCATOR** ByteBuf的分配器，默认值为ByteBufAllocator.DEFAULT，4.0版本为UnpooledByteBufAllocator，4.1版本为PooledByteBufAllocator。该值也可以使用系统参数io.netty.allocator.type配置，使用字符串值："unpooled"，"pooled"。
- **RCVBUF_ALLOCATOR** 用于Channel分配接受Buffer的分配器，默认值为AdaptiveRecvByteBufAllocator.DEFAULT，是一个自适应的接受缓冲区分配器，能根据接受到的数据自动调节大小。可选值为FixedRecvByteBufAllocator，固定大小的接受缓冲区分配器。ALLOCATOR负责ByteBuf怎么分配（从哪里分配），RCVBUF_ALLOCATOR负责计算为接收数据分配多少ByteBuf。
- **AUTO_READ** 是否监听读事件。默认值为True。Netty只在必要的时候才设置关心相应的I/O事件。对于读操作，需要调用channel.read()设置关心的I/O事件为OP_READ，这样若有数据到达才能读取以供用户处理。该值为True时，每次读操作完毕后会自动调用channel.read()，从而有数据到达便能读取；否则，需要用户手动调用channel.read()。需要注意的是：当调用config.setAutoRead(boolean)方法时，如果状态由false变为true，将会调用channel.read()方法读取数据；由true变为false，将调用config.autoReadCleared()方法终止数据读取。
- **AUTO_CLOSE** “写数据”失败，是否关闭连接。默认true，即关闭。
- **MESSAGE_SIZE_ESTIMATOR** 消息大小估算器，默认为DefaultMessageSizeEstimator.DEFAULT。估算ByteBuf、ByteBufHolder和FileRegion的大小，其中ByteBuf和ByteBufHolder为实际大小，FileRegion估算值为0。该值估算的字节数在计算水位时使用，FileRegion为0可知FileRegion不影响高低水位。
- **SINGLE_EVENTEXECUTOR_PER_GROUP** 单线程执行ChannelPipeline中的事件，默认值为True。该值控制执行ChannelPipeline中执行ChannelHandler的线程。如果为True，整个pipeline由一个线程执行，这样不需要进行线程切换以及线程同步，是Netty4的推荐做法；如果为False，ChannelHandler中的处理过程会由Group中的不同线程执行。增加一个handler且指定EventExecutorGroup时，决定这个handler是否只使用EventExecutorGroup中的一个固定的EventExecutor。
- **ALLOW_HALF_CLOSURE** 一个连接的远端关闭时本地端是否关闭，默认值为False。值为False时，连接自动关闭；为True时，触发ChannelInboundHandler的userEventTriggered()方法，事件为ChannelInputShutdownEvent。

# 3. system property

- **io.netty.allocator.numHeapArenas**	内存池堆内存内存区域的个数。默认值:`Math.min(runtime.availableProcessors(),Runtime.getRuntime().maxMemory()/defaultChunkSize/2/3)`
- io.netty.allocator.numDirectArenas	内存池直接内存内存区域的个数。默认值:`Math.min(runtime.availableProcessors(),Runtime.getRuntime().maxMemory()/defaultChunkSize /2/3)`
- io.netty.allocator.pageSize	一个page的内存大小,默认值8192
- io.netty.allocator.maxOrder	用于计算内存池中一个 Chunk内存的大小:默认值11,计算公式如下:`1 Chunk = 8192<<11 = 16MB`
- io.netty.allocator.chunkSize	一个 Chunk内存的大小,如果没有设置,默认值为 `pageSize<< maxOrder=16M`
- io.netty.noKeySetOptimization	Netty的 JDK SelectionKey优化开关,默认关闭,**设置true开启,性能优化开关**,对上层用户不感知
- io.netty.selectorAutoRebuildThreshold	重建 selector的阈值,修复 JDK NIO多路复用器死循环问题。默认值为512
- io.netty.threadLocalDirectBufferSize	线程本地变量直接内存缓冲区大小,默认64KB
- io.netty.machineId	用户设置的机器id,默认会根据MAC地址自动生成
- io.netty.processId	用户设置的流程ID,默认会使用随机数生成
- io.netty.eventLoopThreads	Reactor线程 NioEventLoop的个数,默认值CPU个数×2
- io.netty.noJdkZlibDecoder	是否使用 JDK Zlib压缩解码器,默认不使用
- io.netty.noPreferDirect	是否允许通过底层AP直接访问直接内存。默认值:允许
- io.netty.noUnsafe	是否允许使用 sun.misc.Unsafe,默认允许。注意:使用sun的私有类库存在平台可移植问题:另外, sun.miscUnsafe类是不安全的,如果操作失败,不是抛出异常,而是虚拟机 core dump。不建议使用 Unsafe
- io.netty.noJavassist	是否允许使用 Javassist类库,默认允许
- io.netty.initialSeedUniquifier	本地线程相关的随机种子初始值,默认值为0