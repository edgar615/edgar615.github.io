---
layout: post
title: JVM内存与GC（11）- CMS触发时机
date: 2020-01-11
categories:
    - jvm
comments: true
permalink: jvm-cms-trigger.html
---

CMS GC 在实现上分成 foreground collector 和 background collector。

**foreground collector**

foreground collector 触发条件比较简单，一般是遇到对象分配但空间不够，就会直接触发 GC，来立即进行空间回收。采用的算法是标记清理，不压缩。

它发生的场景，比如业务线程请求分配内存，但是内存不够了，于是可能触发一次CMS GC，这个过程就必须要等待内存分配成功后业务线程才能继续往下面走，因此整个过程必须STW，所以这种CMS  GC整个过程都是STW，但是为了提高效率，它并不是每个阶段都会走的，只走其中一些阶段，但不管怎么说如果走了类似foreground这种CMS GC，那么整个过程业务线程都是不可用的，效率会影响挺大。

foreground收集模式事实上就是发生了FullGC，如果触发了FullGC，那就是ParNew+CMS组合最糟糕的情况。因为这个时候并发模式已经搞不定了，而且整个过程单线程，完全STW，甚至可能还会压缩堆，真的不能再糟糕了！想象一下如果这时候业务量比较大，由于FullGC导致服务完全暂停几秒钟，甚至上10秒，对用户体验影响得多大。

**background collector**

配置CMS垃圾回收的话，有两个重要参数：**-XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly**，这两个参数表示只有在Old区占了75%的内存时才**满足触发CMS的条件**。注意这只是满足触发CMS GC的条件。至于什么时候真正触发CMS  GC，由一个后台扫描线程决定。CMSThread默认2秒钟扫描一次，判断是否需要触发CMS，这个参数可以更改这个扫描时间间隔，例如-XX:CMSWaitDuration=5000，此外可以通过jstack日志看到这个线程：

```
"Concurrent Mark-Sweep GC Thread" os_prio=2 tid=0x000000001870f800 nid=0x0f4 waiting on condition
```

CMSWaitDuration 默认时间是 2s（经常会有业务遇到频繁的 CMS GC，注意看每次 CMS GC 之间的时间间隔，如果是 2s，那基本就可以断定是 CMS 的 background collector）

**MSC**

MSC的全称是**Mark Sweep Compact**，即标记-清理-压缩，**MSC是一种算法**，请注意Compact，即它会压缩整理堆，这一点很重要。

这是foreground CMS在特定情况下才会采用的一种垃圾回收算法。foreground收集模式下如果采用MSC算法的压缩模式，那么在`-XX:+UseCMSCompactAtFullCollection`前提下有三种可能：

- 上一次CMS并发GC执行过后，再执行参数`-XX:CMSFullGCsBeforeCompaction=0`指定的Full GC次数，0表示每次FullGC后都会压缩，同时0也是默认值；
- 调用了System.gc()，当然这就要满足`-XX:-DisableExplicitGC`；
- 晋升担保失败，即预计Old区没有足够空间来容纳下次YoungGC晋升的对象； 

**Backgroud CMS采用的标记清理算法会导致内存碎片问题，从而埋下发生FullGC导致长时间STW的隐患**。

**参考资料**

https://mp.weixin.qq.com/s/ezmD1XXgPoVoCo0nkKavdg

https://mp.weixin.qq.com/s/4FznzPL6_OWV3xy7xnRmWg

https://mp.weixin.qq.com/s/zn3-9e7ZZ7skLo1XDL0xww