---
layout: post
title: GRPC拦截器
date: 2020-06-23
categories:
    - rpc
comments: true
permalink: grpc-interceptor.html
---

> 目前只是放了点文字，后面要在写完整

客户端调用前的拦截

ClientCall该抽象类就是用来调用远程方法的,实现了发送消息和接收消息的功能,该接口由两个泛型ReqT和ReqT,分别对应着请求发送的信息,和请求收到的回复.

ClientCall抽象类主要有两个部分组成,一是public abstract static class Listener<T>用于监听服务端回复的消息,另一部分是针对客户端请求调用的一系列过程,如下代码流程所示:

call = channel.newCall(unaryMethod, callOptions);

call.start(listener, headers);

call.sendMessage(message);//消息发送

call.halfClose();//请求端关闭发送通道

call.request(1);

// wait for listener.onMessage()

客户端收到的回复拦截

通过ClientCall的静态内部类Listener来实现,

public void onHeaders(Metadata headers) {}

public void onMessage(T message) {}//消息正常到达

public void onClose(Status status, Metadata trailers) {}//消息通道关闭

public void onReady() {}

客户端调用前的拦截

ServerCall该抽象类就是用来调用远程方法的,实现了发送消息和接收消息的功能,该接口由两个泛型ReqT和ReqT,分别对应着请求发送的信息,和请求收到的回复.

ServerCall抽象类主要有两个部分组成,一是public abstract static class Listener<T>用于监听客户端发送过来的消息,另一部分是针对客户端回复调用的一系列过程

// wait for listener.onMessage()

call = channel.newCall(call, headers, next);

call.sendMessage(message);

call.sendHeaders(headers);

call.close(status, trailers);}//消息通道关闭

next.start(call, headers)


通过ClientCall的静态内部类Listener来实现,该Listener也是可以嵌套的,其内有如下方法:

 

public void onMessage(T message) {}//请求消息到达

public void onHalfClose() {}//发送端关闭连接的发送通道

public void onCancel() {}

public void onComplete() {}

public void onReady() {}

对于服务端的回复拦截

在ServerCall的子类中有ForwardingServerCall<ReqT, RespT>,该类用于包装ServerCall,然后实现委托嵌套调用,里面方法都如下代码所示:

 

public void sendMessage(RespT message) {}//消息发送

public void request(int numMessages) {}
