---
layout: post
title: JAVA内存分配策略
date: 2020-04-24
categories:
    - jvm
comments: true
permalink: java-memory-allocation.html
---

# Java内存分配策略

Java 提供的自动内存管理，可以归结为解决了对象的内存分配和回收的问题。下面介绍几条最普遍的内存分配策略：

- **对象优先在 Eden 区分配**

大多数情况下，对象在先新生代 Eden 区中分配。当 Eden 区没有足够空间进行分配时，虚拟机将发起一次 Young GC。

- **大对象之间进入老年代**

JVM 提供了一个对象大小阈值参数（-XX:PretenureSizeThreshold，默认值为 0，代表不管多大都是先在 Eden 中分配内存）。大于参数设置的阈值值的对象直接在老年代分配，这样可以避免对象在 Eden 及两个 Survivor 直接发生大内存复制。

- **长期存活的对象将进入老年代：**

对象每经历一次垃圾回收，且没被回收掉，它的年龄就增加 1，大于年龄阈值参数（-XX:MaxTenuringThreshold，默认 15）的对象，将晋升到老年代中。

- **空间分配担保：**

当进行 Young GC 之前，JVM 需要预估：老年代是否能够容纳 Young GC 后新生代晋升到老年代的存活对象，以确定是否需要提前触发 GC 回收老年代空间，基于空间分配担保策略来计算。Young GC 之后如果成功（Young GC 后晋升对象能放入老年代），则代表担保成功，不用再进行 Full GC，提高性能。如果失败，则会出现“promotion failed”错误，代表担保失败，需要进行 Full GC。

- **动态年龄判定：**

新生代对象的年龄可能没达到阈值（MaxTenuringThreshold 参数指定）就晋升老年代。

如果 Young GC 之后，新生代存活对象达到相同年龄所有对象大小的总和大于任意 Survivor 空间（S0+S1空间）的一半，此时 S0 或者 S1 区即将容纳不了存活的新生代对象。

年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到 MaxTenuringThreshold 中要求的年龄。

另外，如果 Young GC 后 S0 或 S1 区不足以容纳：未达到晋升老年代条件的新生代存活对象，会导致这些存活对象直接进入老年代，需要尽量避免。

# 参考资料

