---
layout: post
title: Netty - Java堆外内存
date: 2019-01-07
categories:
    - netty
	- java
comments: true
permalink: netty-directbuffer.html
---

> 基本复制自《Netty 核心原理剖析与 RPC 实践》

# 1. 什么是堆外内存

在 Java 中对象都是在堆内分配的，通常我们说的**JVM 内存**也就指的**堆内内存**，**堆内内存**完全被**JVM 虚拟机**所管理，JVM 有自己的垃圾回收算法，对于使用者来说不必关心对象的内存如何回收。

**堆外内存**与堆内内存相对应，对于整个机器内存而言，除**堆内内存以外部分即为堆外内存**。堆外内存不受 JVM 虚拟机管理，直接由操作系统管理。

堆外内存和堆内内存各有利弊。

1. 堆内内存由 JVM GC 自动回收内存，降低了 Java 用户的使用心智，但是 GC 是需要时间开销成本的，堆外内存由于不受 JVM 管理，所以在一定程度上可以降低 GC 对应用运行时带来的影响。
2. 堆外内存需要手动释放，这一点跟 C/C++ 很像，稍有不慎就会造成应用程序内存泄漏，当出现内存泄漏问题时排查起来会相对困难。
3. 当进行网络 I/O 操作、文件读写时，堆内内存都需要转换为堆外内存，然后再与底层设备进行交互，所以直接使用堆外内存可以减少一次内存拷贝。
4. 堆外内存可以实现进程之间、JVM 多实例之间的数据共享。

由此可以看出，如果你想实现高效的 I/O 操作、缓存常用的对象、降低 JVM GC 压力，堆外内存是一个非常不错的选择。

# 2. 堆外内存的分配

Java 中堆外内存的分配方式有两种：**ByteBuffer#allocateDirect**和**Unsafe#allocateMemory**。

## 2.1. **ByteBuffer#allocateDirect**

分片10M堆外内存

```
ByteBuffer buffer = ByteBuffer.allocateDirect(10 * 1024 * 1024); 
```

上面的方法实际创建的是一个DirectByteBuffer对象。如下图所示，描述了 DirectByteBuffer 的内存引用情况。在堆内存放的  DirectByteBuffer 对象并不大，仅仅包含堆外内存的地址、大小等属性，同时还会创建对应的 Cleaner 对象，通过  ByteBuffer 分配的堆外内存不需要手动回收，它可以被 JVM 自动回收。当堆内的 DirectByteBuffer 对象被 GC  回收时，Cleaner 就会用于回收对应的堆外内存。

![](/assets/images/posts/directbuffer/directbuffer-1.png)

从 DirectByteBuffer 的构造函数中可以看出，真正分配堆外内存的逻辑还是通过 unsafe.allocateMemory(size)来处理

```
DirectByteBuffer(int cap) {                   // package-private
	...
	try {
		base = unsafe.allocateMemory(size);
	} catch (OutOfMemoryError x) {
		Bits.unreserveMemory(size, cap);
		throw x;
	}
	unsafe.setMemory(base, size, (byte) 0);
	...
	cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
	...


}
```

## 2.2. **Unsafe#allocateMemory**

Unsafe 是一个非常不安全的类，它用于执行内存访问、分配、修改等**敏感操作**，可以越过 JVM 限制的枷锁。Unsafe 最初并不是为开发者设计的，使用它时虽然可以获取对底层资源的控制权，但也失去了安全性的保证，所以使用  Unsafe 一定要慎重。

在 Java 中是不能直接使用 Unsafe 的，但是我们可以通过反射获取 Unsafe 实例，使用方式如下所示。

```
private static Unsafe unsafe = null;
static {
    try {
        Field getUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
        getUnsafe.setAccessible(true);
        unsafe = (Unsafe) getUnsafe.get(null);
    } catch (NoSuchFieldException | IllegalAccessException e) {
        e.printStackTrace();
    }
}
```

获得 Unsafe 实例后，我们可以通过 allocateMemory 方法分配堆外内存，allocateMemory 方法返回的是内存地址，使用方法如下所示：

```
// 分配 10M 堆外内存
long address = unsafe.allocateMemory(10 * 1024 * 1024);
```

与 DirectByteBuffer 不同的是，Unsafe#allocateMemory 所分配的内存必须自己手动释放，否则会造成内存泄漏，这也是 Unsafe 不安全的体现。Unsafe 同样提供了内存释放的操作：

```
unsafe.freeMemory(address);
```

到目前为止，我们了解了堆外内存分配的两种方式，对于 Java 开发者而言，常用的是 ByteBuffer.allocateDirect 分配方式，我们平时常说的堆外内存泄漏都与该分配方式有关

# 3. 堆外内存的回收

因为 DirectByteBuffer 对象有可能长时间存在于堆内内存，所以它很可能晋升到 JVM 的老年代，所以这时候  DirectByteBuffer 对象的回收需要依赖 Old GC 或者 Full GC 才能触发清理。如果长时间没有 Old GC 或者  Full GC 执行，那么堆外内存即使不再使用，也会一直在占用内存不释放，很容易将机器的物理内存耗尽，这是相当危险的。

那么在使用 DirectByteBuffer 时我们如何避免物理内存被耗尽呢？因为 JVM 并不知道堆外内存是不是已经不足了，所以我们最好通过  JVM 参数 **-XX:MaxDirectMemorySize** 指定堆外内存的上限大小，当堆外内存的大小超过该阈值时，就会触发一次 Full GC 进行清理回收，如果在 Full GC 之后还是无法满足堆外内存的分配，那么程序将会抛出 OOM 异常。

此外在 ByteBuffer.allocateDirect  分配的过程中，如果没有足够的空间分配堆外内存，在 Bits.reserveMemory 方法中也会主动调用 System.gc() 强制执行  Full GC，但是在生产环境一般都是设置了 **-XX:+DisableExplicitGC**，System.gc() 是不起作用的，所以依赖  System.gc() 并不是一个好办法。

通过前面堆外内存分配方式的介绍，我们知道 DirectByteBuffer 在初始化时会创建一个 Cleaner 对象，它会负责堆外内存的回收工作，那么 Cleaner 是如何与 GC 关联起来的呢？

Java 对象有四种引用方式：强引用 StrongReference、软引用 SoftReference、弱引用 WeakReference  和虚引用 PhantomReference。其中 PhantomReference 是最不常用的一种引用方式，Cleaner 就属于  PhantomReference 的子类，PhantomReference 不能被单独使用，需要与引用队列  ReferenceQueue 联合使用。

当初始化堆外内存时，内存中的对象引用情况如下图所示，first 是 Cleaner 类中的静态变量，Cleaner 对象在初始化时会加入  Cleaner 链表中。DirectByteBuffer 对象包含堆外内存的地址、大小以及 Cleaner  对象的引用，ReferenceQueue 用于保存需要回收的 Cleaner 对象。

![](/assets/images/posts/directbuffer/directbuffer-2.png)

当发生 GC 时，DirectByteBuffer 对象被回收，内存中的对象引用情况发生了如下变化：

![](/assets/images/posts/directbuffer/directbuffer-3.png)

此时 Cleaner 对象不再有任何引用关系，在下一次 GC 时，该 Cleaner 对象将被添加到 ReferenceQueue 中，并执行 clean() 方法。clean() 方法主要做两件事情：

- 将 Cleaner 对象从 Cleaner 链表中移除；
- 调用 unsafe.freeMemory 方法清理堆外内存。