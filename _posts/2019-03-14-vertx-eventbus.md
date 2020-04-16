---
layout: post
title: Vert.x Eventbus
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-eventbus.html
---

# Eventbus
Vert.x提供类三种类型的事件总线
- 发布/订阅
- 点对点
- 请求/响应

Vert.x在创建Vertx对象时会创建一个Eventbus对象，我们先看EventBusImpl(单机版的Eventbus)的实现

	  private void createAndStartEventBus(VertxOptions options, Handler<AsyncResult<Vertx>> resultHandler) {
	    if (options.isClustered()) {
	      eventBus = new ClusteredEventBus(this, options, clusterManager, haManager);
	    } else {
	      eventBus = new EventBusImpl(this);
	    }
	    eventBus.start(ar2 -> {
	      if (ar2.succeeded()) {
		// If the metric provider wants to use the event bus, it cannot use it in its constructor as the event bus
		// may not be initialized yet. We invokes the eventBusInitialized so it can starts using the event bus.
		metrics.eventBusInitialized(eventBus);

		if (resultHandler != null) {
		  resultHandler.handle(Future.succeededFuture(this));
		}
	      } else {
		log.error("Failed to start event bus", ar2.cause());
	      }
	    });
	  }

在创建EventBus之后，会通过EventBus的start方法注册一个回调函数，在EventBus真正启动成功之后会执行这个回调函数．

	  public synchronized void start(Handler<AsyncResult<Void>> completionHandler) {
	    if (started) {
	      throw new IllegalStateException("Already started");
	    }
	    started = true;
	    completionHandler.handle(Future.succeededFuture());
	  }

# Message接口
Message接口用于定义Eventbus中传递的消息.它主要有下列属性：
address：事件发送的地址
replyAddress：事件回应的地址
headers：消息头，通常包含一些公共参数，例如再service-proxy组件中可以将调用对方法放在消息头中
body：消息体，它可以是任意类型的对象.Vertx.x默认仅支持基础类型，JsonObject，JsonArray这几种类型.如果需要支持其他类型的对象需要实现该对象的编解码类并注册到eventbus中．

MessageImpl是Message的实现类，里面主要包括3个参数，sentBody,receivedBody,messageCodec。其中receivedBody是通过messageCodec.transfrom(sentBody)得到
message的reply方法会调用eventbus的sendReply方法发送响应数据

# 发送事件
EventBus的publish(...)方法和send(...)用于发送事件，两种直接对区别是send方法可以注册一个回调函数用于处理事件消费者发回的响应.
这些方法最终会调用createMessage方法来创建一个Message对象.

	  protected MessageImpl createMessage(boolean send, String address, MultiMap headers, Object body, String codecName) {
	    Objects.requireNonNull(address, "no null address accepted");
	    MessageCodec codec = codecManager.lookupCodec(body, codecName);
	    @SuppressWarnings("unchecked")
	    MessageImpl msg = new MessageImpl(address, null, headers, body, codec, send, this);
	    return msg;
	  }
  该方法首先要从CodecManager中找到对应对Codec对象．然后创建一个Message对象.
  
再创建完Message之后， 如果回调函数replyHandler不为null，createReplyHandlerRegistration方法会创建一个HandlerRegistration对象将回调函数与回应地址绑定.回应地址是一个自增长的序列.

# 处理事件
EventBus的consumer方法可以为事件注册一个处理类.它首先会创建一个MessageConsumer对象，然后将处理类注册到该对象
  
# CodecManager
  
# HandlerRegistration

# SendContextImpl
a