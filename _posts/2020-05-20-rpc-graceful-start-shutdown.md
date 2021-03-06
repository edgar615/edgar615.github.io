---
layout: post
title: RPC（7）- 优雅的启动和关闭
date: 2020-05-20
categories:
    - rpc
comments: true
permalink: rpc-graceful-start-shutdown.html
---

微服务开发完成需要上线部署，在整个部署过程中怎么保证业务的连续性，怎么能让服务的客户端无感知，这是一个具有一定挑战性的问题。

一个正在运行的应用如果突然停止，会带来以下情况：

- 数据丢失：内存的中数据尚未持久化至磁盘
- 文件损坏：正在写的文件因没有更新完成，导致文件损坏
- 请求丢失：内存队列中等待处理的请求丢失
- 业务中断：处理一半的业务被强行中断，如支付成功了，却没有更新到数据库中。
- 服务未下线：上游服务依然还会继续往下游服务发送消费请求

所以在关闭服务之前，我们需要先做好善后工作，比如保存数据，清理资源，下线服务，然后才退出应用。这种有计划平滑的关闭应用相对直接停止应用，就显得非常『优雅』。


为了达到不同目的，微服务的部署方式有很多种方式：滚动部署、蓝绿部署、灰度/金丝雀部署。无论是哪一种部署方式，都需要三步操作：停止老版本应用、部署新版本应用、切流量，这三步操作可能是手动也可能是自动，而且它们的顺序也不一定。这其中的两步是非常关键：切流量和停止老版本应用，要想保证业务的连续性和客户端无感知，需要在这两个步骤上下功夫。

# 1. 优雅的关闭

当服务提供方要上线的时候，一般是通过部署系统完成实例重启。在这个过程中，服务提供方的团队并不会事先告诉调用方他们需要操作哪些机器，从而让调用方去事先切走流量。
而对调用方来说，它也无法预测到服务提供方要对哪些机器重启上线，因此负载均衡就有可能把要正在重启的机器选出来，这样就会导致把请求发送到正在重启中的机器里面，从而导致调用方不能拿到正确的响应结果。

在服务重启的时候，对于调用方来说，这时候可能会存在以下几种情况：

- 调用方发请求前，目标服务已经下线。对于调用方来说，跟目标节点的连接会断开，这时候调用方可以立马感知到，并且在其健康列表里面会把这个节点挪掉，自然也就不会被负载均衡选中。
- 调用方发请求的时候，目标服务正在关闭，但调用方并不知道它正在关闭，而且两者之间的连接也没断开，所以这个节点还会存在健康列表里面，因此该节点就有一定概率会被负载均衡选中。

为了避免关闭导致调用方业务受损。我们可以在重启服务机器前，先通过“某种方式”把要下线的机器从调用方维护的“健康列表”里面删除就可以了。

因为服务端都注册到了注册中心，我们可以考虑通过注册中心告诉调用方进行节点摘除。当服务提供方关闭前，是不是可以先通知注册中心进行下线，然后通过注册中心告诉调用方进行节点摘除。整个关闭过程中依赖了两次 RPC 调用:

- 服务提供方通知注册中心下线操作
- 注册中心通知服务调用方下线节点操作

但是我们知道，服务发现只保证最终一致性，并不保证实时性，注册中心在收到服务提供方下线的时候，并不能成功保证把这次要下线的节点推送到所有的调用方。所以通过服务发现并不能做到应用无损关闭。

因为服务提供方已经开始进入关闭流程，那么很多对象就可能已经被销毁了，关闭后再收到的请求按照正常业务请求来处理，肯定是没法保证能处理的。所以我们可以在关闭的时候，设置一个请求“挡板”挡板的作用就是告诉调用方，我已经开始进入关闭流程了，我不能再处理你这个请求了。

然后调用方收到对应的异常响应后，RPC 框架把这个节点从健康列表挪出，并把请求自动重试到其他节点，因为这个请求是没有被服务提供方处理过，所以可以安全地重试到其他节点，这样就可以实现对业务无损。

但如果只是靠等待被动调用，就会让这个关闭过程整体有点漫长。因为有的调用方那个时刻没有业务请求，就不能及时地通知调用方了，所以我们可以加上主动通知流程，这样既可以保证实时性，也可以避免通知失败的情况。

如果进程结束过快会造成正在处理的请求还没有来得及应答，同时调用方会也会抛出异常。为了尽可能地完成正在处理的请求，服务方在关闭过程中，在拒绝新请求的同时，需要根据引用计数器等待正在处理的请求全部结束之后才会真正关闭。

但考虑到有些业务请求可能处理时间长，或者存在被挂住的情况，为了避免一直等待造成应用无法正常退出，我们可以在整个 ShutdownHook 里面，加上超时时间控制，当超过了指定时间没有结束，则强制退出应用。

# 2. 优雅的启动

启动预热：就是让刚启动的服务提供方应用不承担全部的流量，而是让它被调用的次数随着时间的移动慢慢增加，最终让流量缓和地增加到跟已经运行一段时间后的水平一样。

对于刚启动的应用，我们可以让它被选择到的概率特别低，但这个概率会随着时间的推移慢慢变大，从而实现一个动态增加流量的过程。

如何知道服务提供方的启动时间？有两种时间

- 服务提供方在启动的时候，把自己启动的时间告诉注册中心；
- 注册中心收到的服务提供方的请求注册时间。

因为整个预热过程的时间是一个粗略值，即使机器之间的日期时间存在 1 分钟的误差也不影响，并且在真实环境中机器都会默认开启 NTP 时间同步功能，来保证所有机器时间的一致性。

为了实现预热，我们需要将服务提供方的权重变成动态的，并且是随着时间的推移慢慢增加到服务提供方设定的固定值，通过这个小逻辑的改动，我们就可以保证当服务提供方运行时长小于预热时间时，对服务提供方进行降权，减少被负载均衡选择的概率，避免让应用在启动之初就处于高负载状态，从而实现服务提供方在启动后有一个预热的过程。

启动预热更多是从调用方的角度出发，去解决服务提供方应用冷启动的问题，让调用方的请求量通过一个时间窗口过渡，慢慢达到一个正常水平，从而实现平滑上线。但对于服务提供方本身来说，可以通过延迟暴露来实现。

我们应用启动的时候都是通过 main 入口，然后顺序加载各种相关依赖的类。以 Spring 应用启动为例，在加载的过程中，Spring 容器会顺序加载 Spring Bean，如果某个 Bean 是 RPC 服务的话，我们不光要把它注册到 Spring-BeanFactory 里面去，还要把这个 Bean 对应的接口注册到注册中心。当调用方向这个服务提供方建立连接发起请求时，可能存在服务提供方可能并没有启动完成的情况。因为服务提供方应用可能还在加载其它的 Bean，例如加载外部资源。

对于调用方来说，只要获取到了服务提供方的 IP，就有可能发起 RPC 调用，但如果这时候服务提供方没有启动完成的话，就会导致调用失败，从而使业务受损。

我们就可以把接口注册到注册中心的时间挪到应用启动完成后。具体的做法就是在应用启动加载、解析 Bean 的时候，如果遇到了 RPC 服务的 Bean，只先把这个 Bean 注册到 Spring-BeanFactory 里面去，而并不把这个 Bean 对应的接口注册到注册中心，只有等应用启动完成后，才把接口注册到注册中心用于服务发现，从而实现让服务调用方延迟获取到服务提供方地址。

这样是可以保证应用在启动完后才开始接入流量的，但其实这样做，我们还是没有实现最开始的目标。因为这时候应用虽然启动完成了，但并没有执行相关的业务代码，所以 JVM 内存里面还是冷的。如果这时候大量请求过来，还是会导致整个应用在高负载模式下运行，从而导致不能及时地返回请求结果。而且在实际业务中，一个服务的内部业务逻辑一般会依赖其它资源的，比如缓存数据。如果我们能在服务正式提供服务前，先完成缓存的初始化操作，而不是等请求来了之后才去加载，我们就可以降低重启后第一次请求出错的概率

们还是需要利用服务提供方把接口注册到注册中心的那段时间。我们可以在服务提供方应用启动后，接口注册到注册中心前，预留一个 Hook 过程，让用户可以实现可扩展的 Hook 逻辑。用户可以在 Hook 里面模拟调用逻辑，从而使 JVM 指令能够预热起来，并且用户也可以在 Hook 里面事先预加载一些资源，只有等所有的资源都加载完成后，最后才把接口注册到注册中心。整个应用启动过程如下图所示：



# 3. 参考资料

《RPC实战与核心原理》
