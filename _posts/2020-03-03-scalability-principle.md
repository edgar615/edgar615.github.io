---
layout: post
title: 高可扩展性（4）- 可伸缩系统设计原则 
date: 2020-03-03
categories:
    - 架构设计
comments: true
permalink: scalability-principle.html
---

# 1. 简单


隐藏复杂与构建抽象
随着系统的发展，会发现越来越复杂，可能没法了解整个系统的全部，每个人的大脑处理能力有限，不可能了解系统的每个细节。

所以，保持软件简单可以帮助你更好的了解系统。随着系统的逐渐壮大，我们只能做到的是保持局部简单，无法保持整体简单。

开发系统服务时，要创建暴露更高层次的抽象，实现抽象允诺的功能，从而隐藏其复杂性。

# 参考资料

https://mp.weixin.qq.com/s/jhruPpGCdT5VYx-ZHCPrpg