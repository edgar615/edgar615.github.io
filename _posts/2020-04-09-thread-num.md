---
layout: post
title: 到底该设置多少个线程
date: 2020-04-09
categories:
    - 多线程
comments: true
permalink: thread-num.html
---

线程的执行，是由CPU进行调度的，一个CPU在同一时刻只会执行一个线程，我们看上去的线程A 和 线程B并发执行。

为了让用户感觉这些任务正在同时进行，操作系统利用了时间片轮转的方式，CPU给每个任务都服务一定的时间，然后把当前任务的状态保存下来，在加载下一任务的状态后，继续服务下一任务。任务的状态保存及再加载，这段过程就叫做上下文切换。

**上下文切换是需要时间的**

我们的真实业务一般如下所示

![](/assets/images/posts/thread-number/thread-num-1.jpg)

整个过程涉及到下列计算机处理流程

1. 网络请求----->网络IO
2. 解析请求----->CPU
3. 请求数据库----->网络IO
4. MySQL查询数据----->磁盘IO
5. MySQL返回数据----->网络IO
6. 数据处理----->CPU
7. 返回数据给用户----->网络IO

在真实业务中我们不单单会涉及CPU计算，还有网络IO和磁盘IO处理，这些处理是非常耗时的。如果一个线程整个流程是上图的流程，真正涉及到CPU的只有2个节点，其他的节点都是IO处理，那么线程在做IO处理的时候，CPU就空闲出来了，CPU的利用率就不高。

所以多线程的作用就是提升CPU利用率

# 系统性能

衡量系统性能如何，主要指标系统的（QPS/TPS）

> QPS/TPS：每秒能够处理请求/事务的数量
>
> 并发数：系统同时处理的请求/事务的数量
>
> 响应时间：就是平均处理一个请求/事务需要时长

QPS/TPS = 并发数/响应时间

上面公式代表并发数越大，QPS就越大；所以很多人就会以为调大线程池，并发数就会大，也会提升QPS。但其实QPS还跟响应时间成反比，响应时间越大，QPS就会越小。虽然并发数调大了，就会提升QPS，但线程数也会影响响应时间，因为上面我们也提到了上下文切换的问题。

# 基础常规标准

别人总结的一个基础值（还是要根据实际情况调整）

- CPU密集型：操作内存处理的业务，一般线程数设置为：`CPU核数 + 1 或者 CPU核数*2`。核数为4的话，一般设置 5 或 8
- IO密集型：文件操作，网络操作，数据库操作，一般线程设置为：`cpu核数 / (1-0.9)`，核数为4的话，一般设置 40

# 线程数计算方式

```
最佳线程数目 = （（线程等待时间+线程CPU时间）/线程CPU时间 ）* CPU数目
```

比如平均每个线程CPU运行时间为0.5s，而线程等待时间（非CPU运行时间，比如IO）为1.5s，CPU核心数为8，那么根据上面这个公式估算得到：`((0.5+1.5)/0.5)*8=32`。

这个公式可以进一步转化为：

```
最佳线程数目 = （线程等待时间与线程CPU时间之比 + 1）* CPU数目
```

可以得出一个结论：**线程等待时间所占比例越高，需要越多线程。线程CPU时间所占比例越高，需要越少线程。**

一个系统最快的部分是CPU，所以决定一个系统吞吐量上限的是CPU。增强CPU处理能力，可以提高系统吞吐量上限。但根据短板效应，真实的系统吞吐量并不能单纯根据CPU来计算。那要提高系统吞吐量，就需要从“系统短板”（比如网络延迟、IO）着手：

- 尽量提高短板操作的并行化比率，比如多线程下载技术
- 增强短板能力，比如用NIO替代IO

# dark magic估算法

参考下面的代码，这里就拷贝了

https://github.com/sunshanpeng/dark_magic

# 参考资料

https://mp.weixin.qq.com/s/-9jFy3Z0JsKihGoG1nr2rw

http://ifeve.com/how-to-calculate-threadpool-size/
