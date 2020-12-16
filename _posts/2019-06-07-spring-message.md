---
layout: post
title: Spring - message
date: 2019-06-07
categories:
    - Spring
comments: true
permalink: spring-message.html
---

Spring Messaging 是 Spring 框架中的一个底层模块，用于提供统一的消息编程模型。它是spring-kafka，spring-cloud-stream等模块的基础。

**消息**这个数据单元在 Spring Messaging 中统一定义为如下所示的 `Message` 接口，包括一个消息头 Header 和一个消息体 Payload：

```
public interface Message<T> {

	T getPayload();

	MessageHeaders getHeaders();

}
```

而消息通道 `MessageChannel` 的定义也比较简单，我们可以调用 send() 方法将消息发送至该消息通道中

```
@FunctionalInterface
public interface MessageChannel {

	long INDEFINITE_TIMEOUT = -1;

	default boolean send(Message<?> message) {
		return send(message, INDEFINITE_TIMEOUT);
	}

	boolean send(Message<?> message, long timeout);

}
```

消息通道的概念比较抽象，可以简单把它理解为是对队列的一种抽象。我们知道在消息传递系统中，队列的作用就是实现存储转发的媒介，消息发布者所生成的消息都将保存在队列中并由消息消费者进行消费。通道的名称对应的就是队列的名称，但是作为一种抽象和封装，各个消息传递系统所特有的队列概念并不会直接暴露在业务代码中，而是通过通道来对队列进行配置。

Spring Messaging 把通道抽象成如下所示的两种基本表现形式，即支持轮询的 `PollableChannel` 和实现发布-订阅模式的 `SubscribableChannel`，这两个通道都继承自具有消息发送功能的 `MessageChannel`：

```
public interface PollableChannel extends MessageChannel {

	@Nullable
	Message<?> receive();

	@Nullable
	Message<?> receive(long timeout);

}
```

```
public interface SubscribableChannel extends MessageChannel {

	boolean subscribe(MessageHandler handler);

	boolean unsubscribe(MessageHandler handler);

}
```

对于 `PollableChannel` 而言是通过 receive 方法主动获取消息（通过轮询操作）， `SubscribableChannel` 则是通过注册回调函数 MessageHandler 来实现事件响应

```
@FunctionalInterface
public interface MessageHandler {

	void handleMessage(Message<?> message) throws MessagingException;

}
```

Spring Messaging 在基础消息模型之上还提供了很多方便在业务系统中使用消息传递机制的辅助功能，例如各种消息体内容转换器 `MessageConverter` 以及消息通道拦截器 `ChannelInterceptor` 等。

```
public interface MessageConverter {

	@Nullable
	Object fromMessage(Message<?> message, Class<?> targetClass);

	@Nullable
	Message<?> toMessage(Object payload, @Nullable MessageHeaders headers);

}
```

```
public interface ChannelInterceptor {

	@Nullable
	default Message<?> preSend(Message<?> message, MessageChannel channel) {
		return message;
	}

	default void postSend(Message<?> message, MessageChannel channel, boolean sent) {
	}

	default void afterSendCompletion(
			Message<?> message, MessageChannel channel, boolean sent, @Nullable Exception ex) {
	}

	default boolean preReceive(MessageChannel channel) {
		return true;
	}

	@Nullable
	default Message<?> postReceive(Message<?> message, MessageChannel channel) {
		return message;
	}

	default void afterReceiveCompletion(@Nullable Message<?> message, MessageChannel channel,
			@Nullable Exception ex) {
	}

}
```

在 Spring Messaging 的基础上，Spring Integration 还实现了其他几种有用的通道，包括支持阻塞式队列的 RendezvousChannel，该通道与带缓存的 QueueChannel 都属于点对点通道，但只有在前一个消息被消费之后才能发送下一个消息。PriorityChannel 即优先级队列，而 DirectChannel 是 Spring Integration 的默认通道，该通道的消息发送和接收过程处于同一线程中。另外还有 ExecutorChannel，使用基于多线程的 TaskExecutor 来异步消费通道中的消息。

Spring Integration 的设计目的是系统集成，因此内部提供了大量的集成化端点方便应用程序直接使用。当各个异构系统之间进行集成时，如何屏蔽各种技术体系所带来的差异性，Spring Integration 为我们提供了解决方案。通过通道之间的消息传递，在消息的入口和出口我们可以使用通道适配器和消息网关这两种典型的端点对消息进行同构化处理。Spring Integration 提供的常见集成端点包括 File、FTP、TCP/UDP、HTTP、JDBC、JMS、AMQP、JPA、Mail、MongoDB、Redis、RMI、Web Services 等。

# 参考资料

《Spring Cloud 原理与实战 》