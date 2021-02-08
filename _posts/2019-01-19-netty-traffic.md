---
layout: post
title: Netty - 流量整形
date: 2019-01-19
categories:
    - netty
comments: true
permalink: netty-traffic.html
---

Netty的高低水位控制主要是根据发送消息队列积压的大小来控制客户端channel的写状态，然后用户手动根据channel.isWritable（）来控制消息是否发送，用户可以手动处理控制消息不能及时发送后的处理方案（比如，过期、超时）。通常用在客户端比较多。

**流量整形（Traffic Shaping）**是一种主动调整流量输出速率的措施。一个典型应用是基于下游网络结点的TPS指标来控制本地流量的输出。流量整形与流量监管的主要区别在于，流量整形对流量监管中需要丢弃的报文进行缓存——通常是将它们放入缓冲区或队列内，也称流量整形（Traffic Shaping，简称TS）。当令牌桶有足够的令牌时，再均匀的向外发送这些被缓存的报文。流量整形与流量监管的另一区别是，整形可能会增加延迟，而监管几乎不引入额外的延迟。

Netty流量整形的主要作用：

- 防止由于上下游性能不均衡导致下游被冲垮，业务流程中断；
- 防止由于通信模块接收消息过快，后端业务线程处理不及时，导致出现“撑死”问题。

Netty中通过实现抽象类AbstractTrafficShapingHandler，提供了三个流量整形的类；

- 单个链路的流量整形：ChannelTrafficShapingHandler，可以对某个链路的消息发送和读取速度进行控制
- 全局流量整形：GlobalTrafficShapingHandler，针对某个进程所有链路的消息发送和读取速度进行控制
- 全局和单个链路综合型流量整形：GlobalChannelTrafficShapingHandler，同时对全局和单个链路的消息发送和读取速度进行控制。相比于GlobalTrafficShapingHandler增加了一个误差概念，以平衡各个Channel间的读/写操作。也就是说，使得各个Channel间的读/写操作尽量均衡。比如，尽量避免不同Channel的大数据包都延迟近乎一样的是时间再操作，以及如果小数据包在一个大数据包后才发送，则减少该小数据包的延迟发送时间等。

**使用**

```
// 全局
GlobalTrafficShapingHandler globalTrafficShapingHandler = new GlobalTrafficShapingHandler(new NioEventLoopGroup(), 10 * 1024 * 1024, 50 * 1024 * 1024);

channel.pipeline().addLast(globalTrafficShapingHandler)
```

**注意事项：**

- 全局流量整形实例只需要创建一次
   GlobalChannelTrafficShapingHandler 和 GlobalTrafficShapingHandler 是全局共享的，因此实例只需要创建一次，添加到不同的ChannelPipeline即可，不要创建多个实例，否则流量整形将失效。
- 流量整形参数调整不要过于频繁
- 流量整形只对ByteBuf、ByteBufHolder、FileRegion有效，handler要放对位置
- **消息发送保护机制**
   通过流量整形可以控制发送速度，但是它的控制原理是将待发送的消息封装成Task放入消息队列，等待执行时间到达后继续发送，所以如果业务发送线程不判断channle的可以状态，就可能会导致OOM问题。

