---
layout: post
title: 架构图这么画
date: 2021-02-02
categories:
    - 架构设计
comments: true
permalink: architecture-diagram.html
---

# 1. 架构图的定义及作用

关于架构，百度百科上是这样定义的：

> 架构，又名软件架构，是有关软件整体结构与组件的抽象描述，于指导型软件系统各个方面的设计。

ISO/IEC 42010:20072 中对架构则有如下定义：

> The fundamental organization of a system, embodied in its components, their relationships to each other and the environment, and the principles  governing its design and  evolution.（系统架构，体现在它的组成部分、它们之间的相互关系和环境中，以及控制其设计和演化的原则。）

也就是说，**架构是由系统组件，以及组件间相互关系共同构成的集合体**。

而架构图，则是用来表达这种集合的载体。

它的作用也很简单，两个：

- **划分目标系统边界**
- **将目标系统的结构可视化**

进而减少沟通障碍，提升协作效率。

# 2. **架构的分类及画法**

架构大致可以分为4类：业务架构、应用架构、数据架构和技术架构，整体逻辑关系如下：

![](/assets/images/posts/architecture-diagram/architecture-diagram-1.png)

## 2.1. **业务架构**

使用一套方法论/逻辑对产品（项目）所涉及到的业务进行边界划分。所以熟悉业务是关键。

比如做一个团购网站，你需要把商品类目、商品、订单、订单服务、支付、退款等进行清晰划分，而业务架构不需要考虑诸如我用什么技术开发、我的并发大怎么办、我选择什么样的硬件等等。

![](/assets/images/posts/architecture-diagram/architecture-diagram-2.png)

## 2.2. **应用架构**

它是对整个系统实现的总体上的架构，需要指出系统的层次、系统开发的原则、系统各个层次的应用服务。

例如，下图就将系统分为数据层、服务层、通讯层、展现层，并细分写明每个层次的应用服务。

![](/assets/images/posts/architecture-diagram/architecture-diagram-3.png)

## 2.3. **数据架构**

是一套对存储数据的架构逻辑，它会根据各个系统应用场景、不同时间段的应用场景 ，对数据进行诸如数据异构、读写分离、缓存使用、分布式数据策略等划分。

数据架构主要解决三个问题：第一，系统需要什么样的数据；第二，如何存储这些数据；第三，如何进行数据架构设计。

![](/assets/images/posts/architecture-diagram/architecture-diagram-4.png)

## 2.4. **技术架构**

应用架构本身只关心需要哪些应用系统，哪些平台来满足业务目标的需求，而不会关心在整个构建过程中你需要使用哪些技术。技术架构则是应接应用架构的技术需求，并根据识别的技术需求，进行技术选型，把各个关键技术和技术之间的关系描述清楚。

技术架构解决的问题包括：纯技术层面的分层、开发框架的选择、开发语言的选择、涉及非功能性需求的技术选择。