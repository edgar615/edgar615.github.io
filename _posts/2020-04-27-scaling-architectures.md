---
layout: post
title: 六大架构伸缩性原则
date: 2020-04-27
categories:
    - 架构设计
comments: true
permalink: scaling-architectures.html
---

# 1. 成本和伸缩性之间的关系

对系统进行伸缩的一个核心原则是能够方便地添加新资源来处理增长的负载。对于很多系统来说，一个简单而有效的方法是部署多个无状态服务器实例，并使用负载均衡器在这些实例之间分配请求 。假设这些资源部署在云平台上，比如 Amazon Web Services，那么成本就是：

- 每个服务器实例的虚拟机部署成本。
- 负载均衡器的成本，由新请求和活跃请求的数量以及所处理的数据量决定。

在这个场景中，随着请求负载的增长，已部署的虚拟机需要拥有更大的处理能力，这导致了更高的成本。负载均衡器的开销也会随着请求负载和数据大小成比例增长。

![](/assets/images/posts/scaling-architectures/scaling-architectures-1.png)

因此，成本和规模是相辅相成的。可伸缩性的设计决策不可避免地会影响部署成本。如果忽略了这一点，你可能会在月底收到意想不到的巨额部署账单！

那么该如何降低成本？主要有两种方式：

1. 使用弹性负载均衡器，根据实时请求负载来调整服务器实例的数量。在流量较小的时候，只需要为一部分服务器实例付费。随着请求量的增长，负载均衡器会产生新的实例，容量也会相应地增长。
2. 增加每个服务器实例的容量。这通常是通过调优服务器部署参数 (例如线程数、连接数、堆大小等) 来实现。仔细选择参数设置可以显著提高性能，从而提高容量。你基本上是用相同的资源做了更多的工作——这是实现伸缩性的一个关键原则。

# 2. 注意系统瓶颈

对一个系统进行伸缩本质上就是要增加它的容量。在上面的示例中，我们通过部署更多的服务器实例来提高请求处理能力。但是，软件系统是由多个相互依赖的处理元素或微服务组成的，所以在增加一部分微服务容量的同时，不可避免地会被其他一些微服务拖累。

在我们的负载均衡示例中，假设我们的服务器实例都连接到同一个共享数据库。随着部署服务器数量的增加，数据库的请求负载也随之增加。在某个阶段，数据库将达到饱和，数据库访问将开始出现更大的延迟。现在，数据库成为瓶颈——即使你增加更多的服务器处理能力也无济于事。如果要进一步伸缩，就需要以某种方式增加数据库的容量。你可以尝试优化查询，或者增加更多的 CPU 或内存。你也可以对数据库进行复制或分片。除了这些，还有其他很多解决方案。

![](/assets/images/posts/scaling-architectures/scaling-architectures-2.png)

系统中的共享资源都可能成为瓶颈。当你在架构中的某些部分增加容量时，需要仔细考虑下游的容量，确保不会突然给系统造成冲击。因为这样会迅速导致级联故障 (参见下一条规则)，并导致整个系统崩溃。

**数据库、消息队列、长延迟网络连接、线程和连接池以及共享微服务都是潜在的瓶颈**。可以肯定的是，高流量负载会很快让这些瓶颈暴露出来。关键的是要能够在瓶颈暴露出来的时候能够防止系统突然崩溃，并快速部署更多的能力。

# 3. 慢服务比故障服务更有害

在正常情况下下，系统应该能够为微服务和数据库提供稳定、低延迟的通信。当系统负载保持在正常的配置水平时，性能是可预测、一致和快速的。

![](/assets/images/posts/scaling-architectures/scaling-architectures-3.png)

当客户端负载超过正常水平时，微服务之间的请求延迟将开始增加。首先，这通常是一种缓慢但稳定的增长，不会严重影响整个系统的运行，特别是如果负载激增是短暂的。但是，如果传入的请求负载继续超过容量 (服务 B)，未完成的请求将在微服务 A 中堆积，由于下游延迟变慢，该微服务现在接收的请求比已完成的请求还要多。

![](/assets/images/posts/scaling-architectures/scaling-architectures-4.png)

在这种情况下，事情可能很快就会恶化。当一个服务由于抖动或资源耗尽而不堪重负，服务就会无法响应客户端，客户端也将陷入停滞。直接导致的结果就是级联故障——慢服务会导致请求沿着请求路径不断累积，直到整个系统崩溃。

这就是为什么说慢服务比不可用的服务更有害。如果你调用了一个失败的服务或一个由于临时网络问题而出现分区的服务，你会立即收到一个异常，并且可以决定该怎么处理 (例如回退和重试、报告错误)。渐进式超负荷的服务能够正常运行，只是延迟变长了，而这暴露了所有依赖服务的潜在瓶颈，最终会导致严重的错误。

一些架构模式（如回路断路器和隔板）可用于防止级联故障。如果服务的延迟超过指定值，断路器就会调节请求负载，甚至是将其断开。当只有一个下游依赖项发生故障时，隔板可以保护上游的微服务不发生故障。这些措施可用来构建弹性和高度可伸缩的架构。

# 4. 数据层最难伸缩

数据库实际上是每个系统的核心。它们通常包括“事实来源”事务数据库，包含了业务正常运行所需的核心数据和向数据仓库提供临时数据项的运营数据源。

核心业务数据包括客户概况、交易和帐户余额，它们必须是正确、一致且可用的。

运营数据包括用户会话长短、每小时的访问者和页面浏览计数。这些数据通常有一个保质期，可以基于时间段进行聚合和汇总。如果不是百分百完整问题也并不大。因此，我们可以更容易地捕获和存储运营数据，例如将其写入到日志文件或消息队列。然后，使用者定期检索数据并将其写入到数据存储。

随着系统请求处理层的伸缩，共享事务数据库的负载会逐渐增加。随着查询负载的增长，它们迅速成为瓶颈。查询优化变得非常有用，同样，也需要添加更多的内存，让数据库引擎能够缓存索引和表数据。但最终数据库引擎都会耗尽资源，需要进行更彻底的改变。

首先要注意的是，在数据层做出数据结构变更是件痛苦的事情。如果修改了关系型数据库的模式，可能需要运行脚本重新加载数据来匹配新模式。在脚本运行期间 (对于大型数据库来说，这可能是很长的一段时间)，系统就不能执行写操作。这可能会让客户不高兴。

NoSQL、无模式的数据库降低了对重新加载数据库的需求，但仍然需要修改查询代码来匹配修改后的数据结构。如果你的业务数据中有一些数据项的格式已被修改，而有些保留原始格式，那么你可能还需要进行数据对象版本控制。

进一步伸缩可能需要使用分布式数据库，或许包含只读副本的首领和追随者模型就足够了。大多数数据库都很容易配置这种模式，但需要进行密切的监控。当首领发生宕机，故障转移需要花费一些时间，有时候还需要手动干预。这些问题都非常依赖数据库引擎。

如果采用无首领模式，则必须做好跨节点的数据分布和分区。在大多数数据库中，一旦选择了分区键，要做出变更就必须进行数据库重建。所以，分区键的选择要十分明智！**分区键的选择决定了数据是如何跨节点分布的。在添加节点时需要进行再均衡，以便在新节点之间传播数据和请求。**一般的数据库文档都描述了其工作原理，但有时这并不像想象得那么容易。

由于管理分布式数据库存在潜在的问题，基于云的托管替代方案 (例如 AWS Dynamodb、Google Firestore) 往往是更受欢迎的选择。当然，这也是要权衡利弊的。

通过修改逻辑和物理数据模型来扩展查询处理能力通常不是一个平稳而简单的过程，我们应该尽可能减少面对这样的问题。

# 5. 缓存、缓存、再缓存

降低数据库负载的一种方法是尽可能避免经常访问数据库，而这就是缓存的用武之地。好的数据库引擎应该能够尽可能多地利用节点缓存。这是一个简单而有用的解决方案，但成本可能很高。

如果不是必需的，为什么一定要查询数据库呢？对于经常读取和很少发生变化的数据，可以先尝试从分布式缓存中获取，比如 memcached。这需要进行远程调用，但如果你需要的数据刚好存在于高速网络的缓存中，这也比查询数据库实例快得多。

在引入缓存层后，我们需要修改处理逻辑，先从缓存中获取数据。如果你想要的内容不在缓存中，就要查询数据库，然后将结果放到缓存中，并将其返回给调用者。你还需要决定何时删除或让缓存失效——这取决于应用程序对陈旧数据的容错程度。

在伸缩系统时，设计良好的缓存方案绝对是无价的。如果你可以通过缓存处理很大比例的读取请求，那么就可以购买额外的数据库容量，因为它们不需要处理大多数请求。这意味着在为越来越多的请求增加容量时，可以避免复杂而痛苦的数据层修改。这是一个让每个人都开心的秘诀，即使是会计师。

# 6. 监控是可伸缩系统的基础

所有的团队在面对大工作负载时都需要解决的一个问题是进行大规模测试。真实的负载测试很难进行。假设你想测试一个已有的部署，看看如果数据库大小增加 10 倍之后是否仍然能够提供快速的响应。你首先需要生成大量的数据，这些数据最好与实际的数据集和数据关系特征相呼应。你还需要生成一个真实的工作负载。是用于读取，还是用于读和写？然后你再加载和部署数据集，并进行负载测试，这可能需要使用负载测试工具。

这里有很多工作要做。想要让每一件事都接近真实是很难的，所以很少会有人这样做。

另一种选择是进行监控。简单的系统监控包括监控基础设施。如果资源耗尽，例如内存或磁盘空间不足或者远程调用失败，你都应该收到报警，以便在糟糕的事情发生之前采取补救措施。

监控是必要的，但还不够。随着系统的伸缩，你需要了解应用程序行为之间的关系。例如，当并发写请求量增长时，数据库写操作是如何执行的。你还需要知道什么时候回路断路器会由于下游延迟增加而断开微服务连接，什么时候负载均衡器开始生成新实例，或者消息在队列中停留的时间是否超过了指定的阈值。

监控解决方案有很多。Splunk 是一个全面而强大的日志聚合框架。云平台都提供了自己的监控框架，比如 AWS Cloudwatch。用户可以捕获有关系统行为的指标，并在仪表盘中显示这些指标，以便对性能进行监控和分析。“可观察性”通常是指性能监控和分析的整个过程。

有两个方面需要考虑到。

首先，为了深入了解性能，你需要生成与应用程序行为细节相关的自定义指标。仔细设计这些指标，并在微服务中加入代码，将它们注入到监控框架中，以便在系统仪表盘中观察和分析它们。

其次，监控是系统的必要功能 (和成本)。当你需要调优性能和伸缩系统时，你所捕获的数据将会为你的实验和工作提供指导。在系统演进过程中，基于数据驱动的方法有助于确保你的时间被用在修改和改进有用的事情上，而这些是系统性能和伸缩需求的基础。

# 7. 结论

对于大多数系统来说，高性能和可伸缩性通常不是优先考虑的需求。理解、实现和演进功能需求通常是很有问题的，会消耗掉所有可用的时间和预算。但是，有时候由于外部事件或意外事件的驱动，系统需要具备可伸缩性，否则系统就变得不可用，因为它可能在高负载下发生崩溃。不可用的系统 (或由于性能差导致可用性很差的系统) 对任何人来说都是没有用处的。

就像任何一种复杂的软件架构一样，解决系统伸缩性问题也并不存在什么银弹。以精确的系统需求作为指引，做出权衡和妥协对于实现可伸缩性来说至关重要。记住上述的这些规则，通向可伸缩性的道路上就会少踩一些意想不到的坑！

# 8. 参考资料

https://mp.weixin.qq.com/s/77-owRKydWDQ14Lu4mmpiw