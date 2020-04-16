---
layout: post
title: zookeeper
date: 2020-04-14
categories:
    - zookeeper
comments: true
permalink: zookeeper.html
---

ZooKeeper是一个分布式协调服务，可用于服务发现、分布式锁、分布式领导选举、配置管理等。有很多开源组件，尤其是中间件领域，使用 Zookeeper 作为配置中心或者注册中心。例如，它是 Hadoop 和 HBase 的重要组件，是 Kafka 的管理和协调服务，是 Dubbo 等服务框架的注册中心等。

众所周知，协调服务很难做到。他们特别容易出现诸如竞争条件和死锁等错误。ZooKeeper 背后的动机是减轻分布式应用程序从头开始实施协调服务的责任。

# 原理
## 架构

下图是 Zookeeper 的架构图，ZooKeeper 集群中包含 Leader、Follower 以及 Observer 三个角色：

![](/assets/images/posts/zookeeper/zk-1.png)

- **Leader**
- 
Leader 服务器是整个 ZooKeeper 集群工作机制中的核心，负责进行投票的发起和决议，更新系统状态。Leader 是由选举产生，其主要工作有以下两个：

    - 事务请求的唯一调度和处理者，保证集群事务处理的顺序性。
    - 集群内部各服务器的调度者。

- **Follower**

用于接受客户端请求并向客户端返回结果，在选主过程中参与投票。其主要工作有以下三个：

    - 处理客户端非事务请求，转发事务请求给 Leader 服务器。
    - 参与事务请求 Proposal 的投票。
    - 参与 Leader 选举投票。

- **Observer**

可以接受客户端连接，接受读写请求，写请求转发给 Leader，但 Observer 不参加投票过程，只同步 Leader 的状态，Observer 的目的是为了扩展系统，提高读取速度。

Observer与Follower唯一的区别在于，它不参与任何形式的投票。

- **Client**

Client 是 Zookeeper 的客户端，请求发起方。

## zk写入流程

![](/assets/images/posts/zookeeper/zk-2.png)

- Follower/Observer均可接受写请求，但不能直接处理，而需要将写请求转发给Leader处理
- Leader/Follower/Observer都可直接处理读请求，从本地内存中读取数据并返回给客户端即可。

> - Leader并不需要得到Observer的ACK，即Observer无投票权
> - Leader不需要得到所有Follower的ACK，只要收到过半的ACK即可，同时Leader本身对自己有一个ACK。
> - Observer虽然无投票权，但仍须同步Leader的数据从而在处理读请求时可以返回尽可能新的数据
> - 由于处理读请求不需要服务器之间的交互，Follower/Observer越多，整体可处理的读请求量越大，也即读性能越好。

zookeeper的各个复制集节点（follower，leader，observer）都包含了集群所有的数据且存在内存中，像个内存数据库。更新操作会以日志的形式记录到磁盘以保证可恢复性，并且写入操作会在写入内存数据库之前序列化到磁盘。

每个ZooKeeper服务器都为客户端服务。客户端只连接到一台服务器以提交请求。读取请求由每个服务器数据库的本地副本提供服务。更改服务状态，写请求的请求由[zab协议](https://edgar615.github.io/zab.html)处理。

作为协议协议的一部分，**来自客户端的所有写入请求都被转发到称为leader的单个服务器。其余的ZooKeeper服务器（称为followers）接收来自领导者leader的消息提议并同意消息传递**。消息传递层负责替换失败的leader并将followers与leader同步。

ZooKeeper使用自定义原子消息传递协议[zab](https://edgar615.github.io/zab.html)。由于消息传递层是原子的，当领导者收到写入请求时，它会计算应用写入时系统的状态，并将其转换为捕获此新状态的事务。

## 数据结构

在 ZooKeeper 中，每一个节点都被称为一个 ZNde，所有 ZNode 按层次化机构进行组织，形成一棵树。ZNode 节点路径标识方式和 Unix 文件系统路径非常相似，都是由一系列使用斜杠（/）进行分割的路径表示，开发人员可以向这个节点中写入数据，也可以在节点下面创建子节点。

![](/assets/images/posts/zookeeper/zk-3.png)

作为存储媒介来说，ZNode分为持久节点和临时节点：

- **持久节点（PERSISTENT），**该数据节点被创建后，就一直存在于 ZooKeeper 服务器上，除非删除操作（delete）清除该节点。
- **临时节点（EPHEMERAL），**该数据节点的生命周期会和客户端（Client）会话（Session）绑定在一起。如果客户端（Client）会话丢失了，那么节点就自动清除掉。

如果把临时节点看成资源的话，当客户端和服务端产生会话并生成临时节点，一旦客户端与服务器中断联系，节点资源会被从 ZNode 中删除。

**顺序节点（SEQUENTIAL），**ZNode 节点被分配唯一个单调递增的整数。例如多个客户端在服务器 /tasks 上申请节点时，根据客户端申请的先后顺序，将数字追加到 /tasks/task 后面。如果有三个客户端申请节点资源，那么在 /tasks 下面建立三个顺序节点，分别是 /tasks/task1，/tasks/task2，/tasks/task3。

顺序节点，在处理分布式事务的时候非常有帮助，当多个客户端（Client）协作工作的时候，会按照一定的顺序执行。

> - Non-sequence节点，多个客户端同时创建同一 Non-sequence 节点时，只有一个可创建成功，其它匀失败。并且创建出的节点名称与创建时指定的节点名完全一样
> - Sequence节点，创建出的节点名在指定的名称之后带有10位10进制数的序号。多个客户端创建同一名称的节点时，都能创建成功，只是序号不同

如果将上面的两类节点和顺序节点进行组合的话，就有四种节点类型，分别是

- 持久节点
- 持久顺序节点
- 临时节点
- 临时顺序节点

### znode的结构
ZooKeeper数据模型中的每个Znode都维护着一个 stat 结构。一个stat仅提供一个Znode的元数据。它由版本号，操作控制列表(ACL)，时间戳和数据长度组成

- **版本号** - 每个Znode都有版本号，这意味着每当与Znode相关联的数据发生变化时，其对应的版本号也会增加。当多个ZooKeeper客户端尝试在同一Znode上执行操作时，版本号的使用就很重要。
  - version : 当前节点内容（数据）的版本号
  - cversion : 当前节点子节点的版本号
  - aversion : 当前节点ACL的版本号

- **操作控制列表(ACL)** - ACL基本上是访问Znode的认证机制。它管理所有Znode读取和写入操作。每一个节点都拥有自己的ACL，这个列表规定了用户的权限，即限定了特定用户对目标节点可以执行的操作。权限的种类有：
  - CREATE : 创建子节点的权限
  - READ : 获取节点数据和节点列表的权限
  - WRITE : 更新节点数据的权限
  - DELETE : 删除子节点的权限
  - ADMIN : 设置节点ACL的权限

- **时间戳** -  致使ZooKeeper节点状态改变的每一个操作都将使节点接收到一个Zxid格式的时间戳，并且这个时间戳全局有序。也就是说，每个对节点的改变都将产生一个唯一的Zxid。如果Zxid1的值小于Zxid2的值，那么Zxid1所对应的事件发生在Zxid2所对应的事件之前。实际上，ZooKeeper的每个节点维护者三个Zxid值，为别为：cZxid、mZxid、pZxid。
  - cZxid： 是节点的创建时间所对应的Zxid格式时间戳。
  - mZxid：是节点的修改时间所对应的Zxid格式时间戳。

- **数据长度** - 存储在znode中的数据总量是数据长度。最多可以存储1MB的数据。

每个Znode由3部分组成:

- stat：此为状态信息, 描述该Znode的版本, 权限等信息
- data：与该Znode关联的数据
- children：该Znode下的子节点

## 节点监听(Wacher)

znode watch机制也是zk的核心功能之一，是配置管理功能的基石。client端通过注册watch对象后，只要相应的znode触发更改，watch管理器就会向客户端发起回调，可借此机制实现配置管理的分布式更新和同步队列等场景

![](/assets/images/posts/zookeeper/zk-4.png)

Watch 有如下特点：

- 主动推送：Watch被触发时，由 ZooKeeper 服务器主动将更新推送给客户端，而不需要客户端轮询。
- 一次性：数据变化时，Watch 只会被触发一次。如果客户端想得到后续更新的通知，必须要在 Watch 被触发后重新注册一个 Watch。
- 可见性：如果一个客户端在读请求中附带 Watch，Watch 被触发的同时再次读取数据，客户端在得到 Watch 消息之前肯定不可能看到更新后的数据。换句话说，更新通知先于更新结果。
- 顺序性：如果多个更新触发了多个 Watch ，那 Watch 被触发的顺序与更新顺序一致。

## 版本(version)

如果有一个客户端（ClientD），它尝试修改 C 的值，此时其他两个客户端会收到通知，并且进行后续的业务处理了。

![](/assets/images/posts/zookeeper/zk-5.png)

那么在分布式系统中，会出现这么一种情况：在 ClientD 对 C 进行写入操作的时候，又有一个 ClientE 也对 C 进行写入操作。这两个 Client 会去竞争 C 资源，通常这种情况需要对 C 进行加锁操作。

![](/assets/images/posts/zookeeper/zk-6.png)

因此引入 ZNode 版本（Version）概念。版本是用来保证分布式数据原子性操作的。ZNode 的版本（Version）信息保存在 ZNode 的 Stat 对象中。有如下三种：

- version : 当前节点内容（数据）的版本号
- cversion : 当前节点子节点的版本号
- aversion : 当前节点ACL的版本号

本例只关注“数据节点内容的版本号”，也就是 Version。

如果说 ClientD 和 ClientE 对 C 进行写入操作视作是一个事务的话。在执行写入操作之前，两个事务分别会获取节点上的值，即节点保存的数据和节点的版本号（Version）。

以乐观锁为例，对数据的写入会分成以下三个阶段：数据读取，写入校验和数据写入。例如 C 上的数据是 1， Version 是 0。

此时 ClientD 和 ClientE 都获取了这些信息。假设 ClientD 先做写入操作，在做写入校验的时候，发现之前获得的 Version 和节点上的 Version 是相同的，都是 1，因此直接执行数据写入。

写入以后，Version 由原来的 1 变成了 2。当 ClientE 做写入校验时，发现自己持有的 Version=1 和节点当前的 Version=2，不一样。于是，写入失败，重新获取 Version 和节点数据，再次尝试写入。

除了上述方案以外，还可以利用 ZNode 的有序性。在 C 下面建立多个有序的子节点。每当一个 Client 准备写入数据的时候，创建一个临时有序的节点。节点的顺序根据 FIFO 算法，保证先申请写入的 Client 排在其前面。每个节点都有一个序号，后面申请的节点按照序号依次递增。

![](/assets/images/posts/zookeeper/zk-8.png)

每个 Client 在执行修改 C 操作的时候，都要检查有没有比自己序号小的节点，如果存在那么就进入等待。直到比自己序号小的节点进行完毕以后，才轮到自己执行修改操作。从而保证了事物处理的顺序性。

## 会话（session）
客户端都会和 ZooKeeper 的服务端进行通信，或读取数据或修改数据。我们将客户端与服务端完成的这种连接称为会话。ZooKeeper 的会话有 Connecting，Connected，Reconnecting，Reconnected 和 Close 这几种状态。并且在服务端由专门的进程来管理他们，客户端初始化的时候就会根据配置自动连接服务器，从而建立会话，客户端连接服务器时会话处于 Connecting 状态。

![](/assets/images/posts/zookeeper/zk-7.png)

一旦连接完成，就会进入 Connected 状态。如果出现延迟或者短暂失联，客户端会自动重连，Reconnecting 和 Reconnected 状态也就应运而生。

如果长时间超时，或者客户端断开服务器，ZooKeeper 会清理掉会话，以及该会话创建的临时数据节点，并且关闭和客户端的连接。

如果Client因为Timeout和Zookeeper Server失去连接，client处在CONNECTING状态，会自动尝试再去连接Server，如果在session有效期内再次成功连接到某个Server，则回到CONNECTED状态。

注意：如果因为网络状态不好，client和Server失去联系，client会停留在当前状态，会尝试主动再次连接Zookeeper Server。client不能宣称自己的session expired，session expired是由Zookeeper Server来决定的，client可以选择自己主动关闭session。


Session 作为会话实体，用来代表客户端会话，其包括 4 个属性：

- **SessionID，**用来全局唯一识别会话。
- **TimeOut，**会话超时事件。客户端在创造 Session 实例的时候，会设置一个会话超时的时间。
- **TickTime，**下次会话超时时间点。后面“分桶策略”会用到。
- **isClosing，**当服务端如果检测到会话超时失效了，会通过设置这个属性将会话关闭。

SessionTracker 有一个工作就是，将超时的会话清除掉。于是“分桶策略”就登场了。

由于每个会话在生成的时候都会定义超时时间，通过当前时间+超时时间可以算出会话的过期时间。 SessionTracker 不是实时监听会话超时，它是按照一定时间周期来监听的。也就是说，如果没有到达 SessionTracker 的检查时间周期，即使有会话过期，SessionTracker 也不会去清除。由此，就引入会话超时计算公式，也就是 TickTime 的计算公式。

> **TickTime=（（当前时间+会话过期时间）/检查时间间隔+1）*检查时间间隔**

将这个值计算出来以后，SessionTracker 会把对应的会话按照这个时间放在对应的时间轴上面。SessionTracker 在对应的 TickTime 检查会话是否过期。

![](/assets/images/posts/zookeeper/zk-9.png)

每当客户端连接上服务器都会做激活操作，同时每隔一段时间客户端会向服务器发送心跳检测。服务器收到激活或者心跳检测以后，会重新计算会话过期时间，根据“分桶策略”进行重新调整。把会话从“老的区块“放到”新的区块“中去。

![](/assets/images/posts/zookeeper/zk-10.png)

对于超时的会话，SessionTracker 也会做如下清理工作：

- 标记会话状态为“已关闭”，也就是设置 isClosing 为 True。
- 发起“会话关闭”的请求，让关闭操作在整个集群生效。
- 收集需要清理的临时节点。
- 添加“节点删除”的事务变更。
- 删除临时节点
- 移除会话
- 关闭客户端与服务端的连接

会话关闭以后客户端就无法从服务端获取/写入数据了。

# 典型应用

## 分布式锁

分布式锁可以分为两类，一个是保持独占，另一个是控制时序。

对于第一类，我们将zookeeper上的一个znode看作是一把锁，通过create znode的方式来实现。所有客户端都去创建  /distribute_lock  节点，最终成功创建的那个客户端也即拥有了这把锁。用完删除掉自己创建的distribute_lock  节点就释放出锁。

对于第二类， /distribute_lock 已经预先存在，所有客户端在它下面创建临时顺序编号目录节点，和leader选举一样，编号最小的获得锁，用完删除，依次获得锁。

### 排他锁

排他锁，又称写锁或独占锁。如果事务T1对数据对象O1加上了排他锁，那么在整个加锁期间，只允许事务T1对O1进行读取或更新操作，其他任务事务都不能对这个数据对象进行任何操作，直到T1释放了排他锁。

排他锁核心是保证当前有且仅有一个事务获得锁，并且锁释放之后，所有正在等待获取锁的事务都能够被通知到。

Zookeeper 的强一致性特性，能够很好地保证在分布式高并发情况下节点的创建一定能够保证全局唯一性，即Zookeeper将会保证客户端无法重复创建一个已经存在的数据节点。可以利用Zookeeper这个特性，实现排他锁。

1. 定义锁：通过Zookeeper上的数据节点来表示一个锁
2. 获取锁：客户端通过调用 create 方法创建表示锁的临时节点，可以认为创建成功的客户端获得了锁，同时可以让没有获得锁的节点在该节点上注册Watcher监听，以便实时监听到lock节点的变更情况
3. 释放锁：以下两种情况都可以让锁释放
    ​    - 当前获得锁的客户端发生宕机或异常，那么Zookeeper上这个临时节点就会被删除
    ​    - 正常执行完业务逻辑，客户端主动删除自己创建的临时节点

![](/assets/images/posts/zookeeper/zk-11.png)

### 共享锁
共享锁，又称读锁。如果事务T1对数据对象O1加上了共享锁，那么当前事务只能对O1进行读取操作，其他事务也只能对这个数据对象加共享锁，直到该数据对象上的所有共享锁都被释放。

共享锁与排他锁的区别在于，加了排他锁之后，数据对象只对当前事务可见，而加了共享锁之后，数据对象对所有事务都可见。

**定义锁**：通过Zookeeper上的数据节点来表示一个锁，是一个类似于 `/lockpath/[hostname]-请求类型-序号` 的临时顺序节点

 **获取锁**：客户端通过调用 `create` 方法创建表示锁的临时顺序节点，如果是读请求，则创建 `/lockpath/[hostname]-R-序号` 节点，如果是写请求则创建 `/lockpath/[hostname]-W-序号` 节点

判断读写顺序

大概分为4个步骤 

1. 创建完节点后，获取 `/lockpath` 节点下的所有子节点，并对该节点注册子节点变更的Watcher监听

2. 确定自己的节点序号在所有子节点中的顺序

3. 对于读请求：1. 如果没有比自己序号更小的子节点，或者比自己序号小的子节点都是读请求，那么表明自己已经成功获取到了共享锁，同时开始执行读取逻辑 2. 如果有比自己序号小的子节点有写请求，那么等待 
4. 对于写请求，如果自己不是序号最小的节点，那么等待
5. 接收到Watcher通知后，重复步骤1

**释放锁**：与排他锁逻辑一致

Zookeeper实现共享锁节点树如下

![](/assets/images/posts/zookeeper/zk-12.png)

基于Zookeeper实现共享锁流程：

![](/assets/images/posts/zookeeper/zk-13.png)

### 羊群效应

在实现共享锁的 "判断读写顺序" 的第1个步骤是：创建完节点后，获取 `/lockpath` 节点下的所有子节点，并对该节点注册子节点变更的Watcher监听。这样的话，任何一次客户端移除共享锁之后，Zookeeper将会发送子节点变更的Watcher通知给所有机器，系统中将有大量的 "Watcher通知" 和 "子节点列表获取" 这个操作重复执行，然后所有节点再判断自己是否是序号最小的节点(写请求)或者判断比自己序号小的子节点是否都是读请求(读请求)，从而继续等待下一次通知。

然而，这些重复操作很多都是 "无用的"，实际上**每个锁竞争者只需要关注序号比自己小的那个节点是否存在即可**

当集群规模比较大时，这些 "无用的" 操作不仅会对Zookeeper造成巨大的性能影响和网络冲击，更为严重的是，如果同一时间有多个客户端释放了共享锁，Zookeeper服务器就会在短时间内向其余客户端发送大量的事件通知--这就是所谓的 "**羊群效应**"。

具体实现如下：

1. 客户端调用 `create` 方法创建一个类似于 `/lockpath/[hostname]-请求类型-序号` 的临时顺序节点
2. 客户端调用 `getChildren` 方法获取所有已经创建的子节点列表(这里不注册任何Watcher)
3. 如果无法获取任何共享锁，那么调用 `exist` 来对比自己小的那个节点注册Watcher
  - 读请求：向比自己序号小的最后一个**写请求节点**注册Watcher监听
  - 写请求：向比自己序号小的最后一个**节点**注册Watcher监听
4. 等待Watcher监听，继续进入步骤2

## 服务注册与发现

![](/assets/images/posts/zookeeper/zk-14.png)

在微服务中，服务提供方把服务注册到Zookeeper中心去如图中的Member服务，但是每个应用可能拆分成多个服务对应不同的Ip地址，Zookeeper注册中心可以动态感知到服务节点的变化。
服务消费方（Order 服务）需要调用提供方（Member 服务）提供的服务时，从Zookeeper中获取提供方的调用地址列表，然后进行调用。这个过程称为服务的订阅。

### 服务注册原理

![](/assets/images/posts/zookeeper/zk-15.png)

在Zookeeper的注册目录下，为每个应用创建一个持久节点，如order应用创建order持久节点，member应用创建member持久节点。

然后在对应的持久节点下，为每个微服务创建一个临时节点，记录每个服务的URL等信息。

### 服务动态发现原理

![](/assets/images/posts/zookeeper/zk-16.png)

由于服务消费方向Zookeeper订阅了（监听）服务提供方，一旦服务提供方有变动的时候（增加服务或者减少服务），Zookeeper就会把最新的服务提供方列表（member list）推送给服务消费方，这就是服务动态发现的原理。

临时节点的优势是当服务节点下线或者服务节点不可用，Zookeeper 集群会自动将节点地址信息从注册中心删除。这样保证了注册中心不会残留脏数据。缺点是当存在网络抖动，会导致服务节点自动被移除，导致调用方找不到可用的服务节点信息。

为了解决服务节点摘除问题，需要引入第三方探测节点，来探测当前服务节点是否可用，如果不可用可以直接修改注册中心服务节点状态信息，或者直接删除服务节点的注册中心，另外在服务节点重新可用时，还需要重新将服务节点状态信息更新，或者重新写入服务节点信息

## 元数据/配置信息管理

zookeeper 可以用作很多系统的配置信息的管理，比如 kafka、storm 等等很多分布式系统都会选用 zookeeper 来做一些元数据、配置信息的管理。

把相关配置全部放到zookeeper上去，保存在 Zookeeper 的某个目录节点中，然后所有相关应用程序对这个目录节点进行监听，一旦配置信息发生变化，每个应用程序就会收到 Zookeeper 的通知，然后从Zookeeper 获取新的配置信息应用到系统中。

![](/assets/images/posts/zookeeper/zk-17.png)

## 分布式队列
在日常使用中，特别是像生产者消费者模式中，经常会使用BlockingQueue来充当缓冲区的角色。但是在分布式系统中这种方式就不能使用BlockingQueue来实现了，但是Zookeeper可以实现。

1. 首先利用Zookeeper中临时顺序节点的特点
2. 当生产者创建节点生产时，需要判断父节点下临时顺序子节点的个数，如果达到了上限，则阻塞等待；如果没有达到，就创建节点。
3. 当消费者获取节点时，如果父节点中不存在临时顺序子节点，则阻塞等待；如果有子节点，则获取执行自己的业务，执行完毕后删除该节点即可。
4. 获取时获取最小值，保证FIFO特性。

## 集群管理

随着分布式系统规模的日益扩大，集群中的机器规模也随之变大，因此，如何更好的进行集群管理也显得越来越重要了。

所谓集群管理，包括集群监控与集群控制两大块，前者侧重对集群运行时状态的收集，后者则是对集群进行操作与控制。在日常开发和运维过程中，我们经常会有类似于如下的需求。

- 希望知道当前集群中究竟有多少机器在工作。
- 对集群中每台机器的运行时状态进行数据收集。
- 对集群中机器进行上下线操作。

使用Zookeeper可以方便的实现集群管理的功能。思路如下，每个服务器启动时都向zk服务器提出创建临时节点的请求，并且使用getChildren设置父节点的观察，当该服务器挂掉之后，它创建的临时节点也被Zookeeper服务器删除，然后会触发监视器，其他服务器便得到通知。创建新节点也是同理。

所谓集群管理无在乎两点：集群监控（是否有机器退出和加入）、选举master。

### 集群监控

所有机器约定在父目录GroupMembers下创建临时目录节点，然后监听父目录节点的子节点变化消息。一旦有机器挂掉，该机器与 zookeeper的连接断开，其所创建的临时目录节点被删除，所有其他机器都收到通知：某个兄弟目录被删除，于是，所有人都知道。新机器加入 也是类似。

### master选举

Master选举则是zookeeper中最为经典的使用场景了，在分布式环境中，相同的业务应用分布在不同的机器上，有些业务逻辑，例如一些耗时的计算，网络I/O处，往往只需要让整个集群中的某一台机器进行执行，其余机器可以共享这个结果，这样可以大大减少重复劳动，提高性能，于是这个master选举便是这种场景下的碰到的主要问题。

利用ZooKeeper中两个特性，就可以实施另一种集群中Master选举：

1. 利用ZooKeeper的强一致性，能够保证在分布式高并发情况下节点创建的全局唯一性，即：同时有多个客户端请求创建 /Master 节点，最终一定只有一个客户端请求能够创建成功。利用这个特性，就能很轻易的在分布式环境中进行集群选举了。

2. 另外，这种场景演化一下，就是动态Master选举。这就要用到 EPHEMERAL_SEQUENTIAL类型节点的特性了，这样每个节点会自动被编号。允许所有请求都能够创建成功，但是得有个创建顺序，每次选取序列号最小的那个机器作为Master 。 

算法

1. 每个服务创建一个`/election/candidate-sessionID_`的节点，节点的类型是SEQUENCE| EPHEMERAL类型。 在 SEQUENCE 标志下， ZooKeeper 将自动地为每一个 ZooKeeper 服务器分配一个比前一个分配的序号要大的序号。 序列号最小的节点将成为Leader
2. 每个服务读取`/election/`节点下的子节点,`L = getChildren("/election", false)`,这里并不监听子结点的变化,是为了避免羊群效应
3. 每个服务监听`/election/candidate-sessionID_M`节点的编号，M是比自己节点的序列号小一号的节点,`exists("/election/candidate-sessionID_M", true)`。
4. 当服务监听到`/election/candidate-sessionID_M`的删除事件之后，读取`/election/`节点下的子节点`getChildren(("/election", false)`
5. 在读取到`/_election_`节点下的子节点挂掉后，按照下面的算法选举出一个leader:
    - 如果服务的`/election/candidate-sessionID_N`,是最小的节点，把它选举为leader
    - 监听比自己节点的序列号小一号的节点
6. 如果leader挂了，监听了这个leader的服务会选举为新的leader。

## 栅栏Barrier
barrier的作用是所有的线程等待，知道某一时刻，锁释放，所有的线程同时执行。

某个node路径为"/queue_barrier",为该根节点赋值为某个默认值,假设为10,当根路径"/queue_barrier"下的子节点个数为10时,则所有子进程都完成了任务,主进程开始执行。

基于zookeeper的节点类型,创建临时连续的节点会在创建的节点后给节点名加上一个数字后缀,基于这个顺序，我们可以有如下的思路：

1. 通过调用getData()来获取某个节点的值,假设为10。
2. 调用getChildren()来获取所有的子节点,同时注册watcher监听。
3. 统计子节点的个数。
4. 将统计的个数和getData()获取的值比较,如果还不足10,就需要等待。
5. 接收watcher通知

## HA高可用性
比如 hadoop、hdfs、yarn 等很多大数据系统，都选择基于 zookeeper 来开发 HA 高可用机制。

具体来说就是一个重要进程一般会做主备两个，主进程挂了立马通过 zookeeper 感知到切换到备用进程。

![](/assets/images/posts/zookeeper/zk-19.png)

# 参考资料

https://mp.weixin.qq.com/s/2Sz2XLRJmZq54SpTrf3JNA

https://www.jianshu.com/p/a974eec257e6

https://mp.weixin.qq.com/s/pCWwIgnUnQUGlKZOrZ4RYA

https://blog.csdn.net/lingbo229/article/details/81052078

http://blog.itpub.net/31509949/viewspace-2218265/

http://www.imooc.com/article/80689