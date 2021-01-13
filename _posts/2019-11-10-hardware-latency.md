---
layout: post
title: 常见硬件组件的延时情况
date: 2019-11-10
categories:
    - 架构设计
comments: true
permalink: heardware-latency.html
---

![](/assets/images/posts/heardware/heardware-latency.jpg)

访问本地内存大概需要 100ns 的时间；使用 SSD 磁盘进行搜索，大概需要 10万ns 时间；数据包在同一个数据中心来回一次的时间，也就是在同一个路由环境里进行一次访问，大概需要 50万ns 时间，也就是 0.5ms；使用非 SSD 磁盘进行一次搜索，大概需要 1000万ns，也就是 10ms 的时间；按顺序从网络中读取 1MB 的数据也是需要 10ms 的时间；按顺序从传统的机械磁盘，即非 SSD 磁盘，读取 1MB 数据，大概需要 30ms 的时间；跨越大西洋进行一次网络数据传输，一个来回大概需要 150ms 的时间。其中，1s 等于 1000ms，等于 10亿ns。
