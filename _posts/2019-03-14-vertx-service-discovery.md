---
layout: post
title: Vert.x的服务发现模块
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-service-discovery.html
---
**前部分内容基本来自于官方文档**

Vert.x提供了服务发现组件，每个服务都被封装为一个Record对象

服务提供者：

- 发布一个服务
- 注销一个服务
- 更新已发布的服务

服务消费者：

- 查询服务
- 使用选定的服务
- 一旦用户用完服务，释放服务
- 监听服务的发布、注销和修改

# 核心概念
**Service records 服务记录**

用来描述服务提供者发布的服务的对象。它包含名称，元数据，地址（描述服务在哪里）等属性

**Service Provider and publisher 服务提供者和发布者**

用来发布或注销一个服务

**Service Consumer 服务消费者**

服务消费者在服务发现中查找服务（可能返回0个或多个服务记录）。从这些记录中，消费者可以获得ServiceReference对象，用来表示消费者和提供者之间的绑定。这个对象允许消费者使用该服务，并释放服务。**需要注意释放服务引用来清理对象和更新服务**

**Service object 服务对象**

服务对象是提供访问服务的对象。它可以以各种形式出现，如代理、客户端，甚至有些不存在的服务类型。服务对象的性质取决于服务类型。

**Service types 服务类型**

服务可以是各种类型的服务，比如功能性服务，数据库，REST API等。每个服务类型定义了：
​	
- 这个服务的地址（URL，事件总线地址，IP/DNS）
- 服务对象的性质（服务代理，HTTP客户端，消息消费者）

某些服务类型由服务发现组件实现并提供，我们也可以添加自己的服务类型。

**Service events 服务事件**

每次发布或注销服务时，总会在事件总线上触发一个announce事件。这个事件包括已经修改的服务记录。此外为了跟踪谁在使用服务，每次ServiceReference的绑定和释放都会触发一个usage事件


**Backend**

服务发现使用分布式结构存储服务记录，因此群集的所有成员都可以访问所有记录。也可以实现你自己的servicediscoverybackend SPI来存储服务记录.

# 使用
## 创建一个服务发现实例
<pre class="line-numbers "><code class="language-java">
	ServiceDiscovery discovery = ServiceDiscovery.create(vertx);
</code></pre>
使用默认配置创建了一个服务发现实例，发布和注销服务时触发的事件地址为`vertx.discovery.announce`，绑定和释放ServiceReference时触发的事件地址为`vertx.discovery.usage`

我们可以使用自定义的配置
<pre class="line-numbers "><code class="language-java">
    ServiceDiscovery discovery = ServiceDiscovery.create(vertx,
                new ServiceDiscoveryOptions()
                        .setAnnounceAddress("service-announce")
                        .setName("my-name"));
</code></pre>

Vertx.官方目前提供了下面几种类型的服务发现（ServiceType）

- HttpEndpoint restAPI
- EventBusService  服务代理
- MessageSource 消息
- JDBCDataSource JDBC

## 发布服务
发布服务的过程如下：

- 为服务提供者创建服务记录(Record)
- 发布这个服务记录
- 保存已发布的服务记录，用来注销或者修改服务
<pre class="line-numbers "><code class="language-java">
    Record record = new Record()
            .setType("eventbus-service-proxy")
            .setLocation(new JsonObject().put("endpoint", "the-service-address"))
            .setName("my-service")
            .setMetadata(new JsonObject().put("some-label", "some-value"));
    discovery.publish(record, ar -> {
      if (ar.succeeded()) {
        Record publishedRecord = ar.result();
        System.out.println(record.getRegistration());
        System.out.println(publishedRecord.getLocation());
        System.out.println(publishedRecord.getType());
        System.out.println(publishedRecord.getMetadata());
        System.out.println(publishedRecord.getName());
      } else {

      }
    });
</code></pre>
服务发布之后，会给服务记录生成一个唯一的ID(registration)

Vert.x也提供了一下基本服务类型用于创建服务记录（后面再描述）

## 注销服务
在服务发布之后，可以使用registration将服务注销
<pre class="line-numbers "><code class="language-java">
	discovery.unpublish(record.getRegistration(), ar -> {
	  if (ar.succeeded()) {
	    // Ok
	  } else {
	    // cannot un-publish the service, may have already been removed, or the record is not published
	  }
	});
</code></pre>
## 查找服务
服务消费者可以通过getRecord搜索单个服务记录,或者通过getRecords搜索所有匹配的服务记录。

服务消费者可以通过一个过滤器来匹配服务，过滤器有两种：

- Function<Record, Boolean> filter，以Record为参数，返回一个bool的函数
- JsonObject，对JsonObject中的每个属性检查，必须所有的属性都匹配的记录才是要搜索的服务记录。

- { "name" = "a" } => 匹配所有name属性为a的记录
- { "color" = "*" } => 匹配所有设置了color属性的记录
- { "color" = "red" } => 匹配所有color属性为red的记录

**搜索全部记录**

<pre class="line-numbers "><code class="language-java">
    discovery.getRecords(r -> true, ar -> {
        ListRecord records = ar.result();
        for (Record record : records) {
        System.out.println(record.getName() + ":" + record.getMetadata());
        }
    });
</code></pre>
aaa