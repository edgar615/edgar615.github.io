---
layout: post
title: 分离（2）- 读写分离
date: 2020-05-11
categories:
    - 架构设计
comments: true
permalink: read-write-separation.html
---

# 1. 什么是读写分离

读写分离是互联网应用系统中提升数据访问性能最常见的一种技术。

在早期的架构中为了保证可靠性，我们对数据库采用了主从结构部署。主库负责了所有的读写操作，而从库只对主库进行了备份，如果只实现了一个备份，不能读写分离和故障转移，不能降低Master节点的IO压力。这样的主从架构看起来性价比似乎不是很高。

![](/assets/images/posts/read-write-separation/read-write-separation-1.png)

我们所希望的主从架构是，当我们在写数据时，请求全部发到Master节点上，当我们需要读数据时，请求全部发到Slave节点上。并且多个Slave节点最好可以存在负载均衡，让集群的效率最大化。

读写分离基本原理就是将数据库的读写操作分配到不同的节点上。让主数据库处理事务性查询，而从数据库处理SELECT查询。 当然，主服务器也可以提供查询服务。使用读写分离最大的作用无非是缓解服务器压力。

![](/assets/images/posts/read-write-separation/read-write-separation-2.png)

**读写分离的基本实现是：**

- 数据库服务器搭建主从集群，一主一从、一主多从都可以
- 数据库主机负责读写操作，从机只负责读操作
- 数据库主机通过复制将数据同步到从机，每台数据库服务器都存储了所有的业务数据
- 业务服务器将写操作发给数据库主机，将读操作发给数据库从机

**何种场景下使用读写分离？**

当你在实际业务中遇到以下情形，则可以考虑使用读写分离解决方案。

- 数据量大
- 所有写数据的请求效率尚可
- 查询数据的请求效率很低
- 所有的数据任何时候都可能被修改
- 业务希望我们优化查询数据的功能。

> 读写分离提升了数据库的读性能，但是并没有提升写性能

# 2. 读写分离的实现

## 2.1. 代码层

我们在代码层实现逻辑，对到达的读/写请求进行解析，针对性地分发到不同的数据库中，从而实现读写分离；

1. 注入数据源，包括Master-Slave；
2. 通过拦截器或者AOP，判断一个SQL语句是读还是写；
3. 选择对应的数据源进行执行

这种方法的优势就是比较灵活，我们可以按照自己的逻辑来决定读写分离的规则。但弊端也显而易见，这样对代码的侵入性太强，当需要对数据源进行增删时，一定会对代码造成影响。这不但会给开发人员造成很大的困扰，而且不符合软件设计的开闭原则。而且故障情况下，如果主从发生切换，则可能需要所有系统都修改配置并重启。

## 2.2. 中间层

这种方式最主要的特点就是我们在整体架构中新建了一个虚拟节点，所有的请求先到达这个虚拟节点上，由这个虚拟节点来转发读写请求到相应的数据库。

虚拟节点对业务服务器提供 SQL 兼容的协议，业务服务器无须自己进行读写分离。对于业务服务器来说，虚拟节点和访问数据库没有区别，事实上在业务服务器看来，虚拟节点就是一个数据库服务器。其基本架构是：

![](/assets/images/posts/read-write-separation/read-write-separation-3.png)

相对于代码层实现，使用中间层的优点十分明显，我们只需要进行配置就可以享受读写分离带来的效率的提升，不用写一行代码。

其缺点当然是需要我们维护这样一个虚拟节点，增加了运维成本。

由于数据库中间件的复杂度要比程序代码封装高出一个数量级，一般情况下建议采用程序语言封装的方式，或者使用成熟的开源数据库中间件。如果是大公司，可以投入人力去实现数据库中间件，因为这个系统一旦做好，接入的业务系统越多，节省的程序开发投入就越多，价值也越大。

# 3. 读写分离带来的问题

读写分离会带来**数据一致性问题**。以 MySQL 为例，主从复制延迟可能达到 1 秒，如果有大量数据同步，延迟 1 分钟也是有可能的。主从复制延迟会带来一个问题：如果业务服务器将数据写入到数据库主服务器后立刻（1 秒内）进行读取，此时读操作访问的是从机，主机还没有将数据复制过来，到从机读取数据是读不到最新数据的，业务上就可能出现问题。例如，用户刚注册完后立刻登录，业务服务器会提示他“你还没有注册”，而用户明明刚才已经注册成功了。

> **数据冗余一定会引发一致性问题。（数据库主从、数据库与缓存）**

## 3.1. 忽略不计

如果业务对于数据一致性要求不高，我们就可以采用这种方案。

## 3.2. 强制读主

对于需要强一致的场景，我们可以将其的读请求都操作主库，这样**读写都在主库**，就没有不一致的情况。

此时为了提升读性能，我们可能会引入缓存机制，此时又会引起数据不一致问题。

![](/assets/images/posts/read-write-separation/read-write-separation-1.png)

## 3.3. 选择性读组

例如，注册账号完成后，登录时读取账号的读操作也发给数据库主服务器。这种方式和业务强绑定，对业务的侵入和影响较大，如果哪个新来的程序员不知道这样写代码，就会导致一个 bug。

写请求发往主库，同时缓存记录操作的 key，缓存的失效时间设置为主从的延时；

![](/assets/images/posts/read-write-separation/read-write-separation-4.png)

读请求首先判断缓存是否存在：

- 若存在，代表刚发生过写操作，读请求操作主库；
- 若不存在，代表近期没发生写操作，读请求操作从库。

![](/assets/images/posts/read-write-separation/read-write-separation-5.png)

## 3.4. 读从机失败后再读一次主机

这就是通常所说的“二次读取”，二次读取和业务无绑定，只需要对底层数据库访问的 API 进行封装即可，实现代价较小，不足之处在于如果有很多二次读取，将大大增加主机的读操作压力。例如，黑客暴力破解账号，会导致大量的二次读取操作，主机可能顶不住读操作的压力从而崩溃。

## 3.5. 关键业务读写操作全部指向主机，非关键业务采用读写分离

例如，对于一个用户管理系统来说，注册 + 登录的业务读写操作全部访问主机，用户的介绍、爱好、等级等业务，可以采用读写分离，因为即使用户改了自己的自我介绍，在查询时却看到了自我介绍还是旧的，业务影响与不能登录相比就小很多，还可以忍受。