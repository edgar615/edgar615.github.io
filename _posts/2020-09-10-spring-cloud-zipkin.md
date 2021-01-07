---
layout: post
title: Spring Cloud Zipkin
date: 2020-09-10
categories:
    - Spring
comments: true
permalink: spring-cloud-zipkin.html
---

# 1. Get Started

我们之间使用在Consul部分school和student代码来做测试。

```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

增加zipkin配置

```
spring:
  zipkin:
    baseUrl: http://192.168.159.131:9411
```

向schools发送请求后我们可以看到school的控制台打印了如下日志

```
2021-01-07 13:22:34.294  INFO [school-service,54e4379600572c35,54e4379600572c35,true] 14524 --- [nio-9001-exec-3] c.g.e.s.c.s.SchoolServiceController      : Going to call student service to get data!
```

上面的日志括号内的内容包括了四段内容，即服务名称、TraceId、SpanId 和 Zipkin 标志位，它是格式如下所示：

```
[服务名称, TraceId, SpanId, Zipkin 标志位]
```

第一段中的 school-service代表着该服务的名称，使用的就是在 bootstrap.yml 中 spring.application.name 指定的服务名称。考虑到服务跟踪的需求，为服务指定一个统一而友好的名称是一项最佳实践。

第二段中的 TraceId 代表一次完整请求的唯一编号，上例中的 54e4379600572c35就是该次请求的唯一编号。可以通过 TraceId 查看完整的服务调用链路。

第三段中的SpanId代表一个服务之间的调用过程的唯一标识，TraceId 和 SpanId 是一对多的关系，即一个  TraceId 一般都会包含多个 SpanId，每一个 SpanId 都从属于特定的 TraceId。当然，也可以通过 SpanId  查看某一个服务调用过程的详细信息。

最后的第四段代表 Zipkin 标志位，该标志位用于识别是否将服务跟踪信息同步到 Zipkin。

我们可以在zipkin中找到对应的调用信息

![](/assets/images/posts/zipkin/zipkin-1.png)

根据这个traceId`54e4379600572c35`，我们在zipkin中看到调用时序

![](/assets/images/posts/zipkin/zipkin-2.png)

上图中最重要的就是各个 Span 信息。一个服务调用链路被分解成若干个 Span，每个 Span 代表完整调用链路中的一个可以衡量的部分。我们通过可视化的界面，可以看到整个访问链路的整体时长以及各个 Span 所花费的时间。每个 Span 的时延都已经被量化，并通过背景颜色的深浅来表示时延的大小。

在上图中，我们点击任何一个感兴趣的 Span 就可以获取该 Span 对应的各项服务调用数据明细。

![](/assets/images/posts/zipkin/zipkin-3.png)

![](/assets/images/posts/zipkin/zipkin-4.png)

上图展示了针对该 Span 的 cs、sr、ss 和 cr 这四个关键事件数据，也可以看到这些数据包括 HTTP 请求相关的方法、路径等各项基础数据，也包括使用 Spring 构建 RESTful 风格调用时的 Controller 类以及端点信息。

最后，作为数据管理和展示的统一平台，Zipkin 还实现了更为低层的数据表现形式，也就是通过 JSON 数据提供对调用过程的详细描述。我们可以通过`/zipkin/api/v2/trace/<traceId>`来获取

# 7. 参考资料

《Spring Cloud 原理与实战 》