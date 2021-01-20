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

| Work                               |         Latency |
| :--------------------------------- | --------------: |
| L1 cache reference                 |          0.5 ns |
| Branch mispredict                  |           5  ns |
| L2 cache reference                 |           7  ns |
| Mutex lock/unlock                  |          25  ns |
| Main memory reference              |         100  ns |
| Compress 1K bytes with Zippy       |       3,000  ns |
| Send 1K bytes over 1 Gbps network  |      10,000  ns |
| Read 4K randomly from SSD*         |     150,000  ns |
| Read 1 MB sequentially from memory |     250,000  ns |
| Round trip within same datacenter  |     500,000  ns |
| Read 1 MB sequentially from SSD*   |   1,000,000  ns |
| Disk seek                          |  10,000,000  ns |
| Read 1 MB sequentially from disk   |  20,000,000  ns |
| Send packet CA->Netherlands->CA    | 150,000,000  ns |

访问本地内存大概需要 100ns 的时间；使用 SSD 磁盘进行搜索，大概需要 10万ns 时间；数据包在同一个数据中心来回一次的时间，也就是在同一个路由环境里进行一次访问，大概需要 50万ns 时间，也就是 0.5ms；使用非 SSD 磁盘进行一次搜索，大概需要 1000万ns，也就是 10ms 的时间；按顺序从网络中读取 1MB 的数据也是需要 10ms 的时间；按顺序从传统的机械磁盘，即非 SSD 磁盘，读取 1MB 数据，大概需要 30ms 的时间；跨越大西洋进行一次网络数据传输，一个来回大概需要 150ms 的时间。其中，1s 等于 1000ms，等于 10亿ns。

虽然磁盘的寻道时间只需要 10ms，但是在 CPU 看来已经是非常长的时间了，当我们将上述的时间等比例放大后，就能直观地感受到它们的性能差异。如果 CPU 访问 L1 缓存需要 1 秒，那么访问主存需要 3 分钟、从 SSD 中随机读取数据需要 3.4 天、磁盘寻道需要 2 个月，网络传输可能需要 1 年多的时间。

在计算机体系结构中，硬盘属于一种常见的输入输出设备，操作系统在启动时不一定需要硬盘，它既可以通过硬盘启动，也可以通过网络设备或者外部设备启动，所以硬盘不是计算机运行的必要条件。