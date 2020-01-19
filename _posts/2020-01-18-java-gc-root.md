---
layout: post
title: JVM虚拟机如何枚举根节点
date: 2020-01-18
categories:
    - jvm
comments: true
permalink: java-gc-root.html
---

在[如何判断一个对象是否存活](https://edgar615.github.io/java-object-survival.html)的文章中，我们介绍了GC Root：**HotSpot 虚拟机采取的是可达性分析算法。即通过 GC Roots 枚举判定待回收的对象**

那么，首先就要要找到哪些是 GC Roots。~~有两种查找 GC Roots 的方法：~~

- ~~一种是遍历方法区和栈区查找（保守式GC）。~~
- ~~一种是通过 OopMap 数据结构来记录 GC Roots 的位置（准确式 GC）。~~

# GC Roots 枚举

可作为GC Roots的节点主要在全局性的引用（例如常量或类静态属性）与执行上下文（例如栈帧中的本地变量表）。在进行 **GC Roots 枚举**时必须在一个能确保**一致性的快照**中进行——在整个分析期间整个执行系统看起来就像被冻结在某个时间点上，不可以出现分析过程中对象引用关系还在不断变化的情况，该点不满足的话分析结果准确性就无法得到保证。即所有正在运行的程序必须停顿，不能出现正在枚举 GC Roots，而程序还在跑的情况。

**GC Roots 枚举为了确保一致性的快照就必然会导致Stop The World**。即使是在号称（几乎）不会发生停顿的CMS收集器中，枚举根节点时也是必须要停顿的。

# 准确式 GC

为解决上述问题，HotSpot 采用了一种 “准确式 GC”  的技术；该技术主要功能就是让虚拟机可以准确的知道内存中某个位置的数据类型是什么；比如某个内存位置到底是一个整型的变量，还是对某个对象的  reference；这样在进行 GC Roots 枚举时，只需要枚举 reference 类型的即可。

在能够准确地确定 Java 堆和方法区等 reference 准确位置之后，HotSpot 就能极大地缩短 GC Roots 枚举时间。

## OopMap
在HotSpot中，使用一组OopMap的数据结构来标记对象引用的位置。当类加载完成后，HotSpot 就将对象内存布局之中什么偏移量上数值是一个什么样的类型的数据这些信息存放到 OopMap  中；在 HotSpot 的 JIT 编译过程中，同样会插入相关指令来标明哪些位置存放的是对象引用等。

这样在 GC 发生时，HotSpot  就可以直接扫描 OopMap 来获取这些信息，从而进行 GC Roots 枚举，

# 安全点
OopMap内容变化的指令非常多，如果为每一条指令都生成对应的OopMap，那将会需要大量的额外空间，这样GC的空间成本将会变得很高。

实际上，HotSpot也的确没有为每条指令都生成OopMap，只是在“特定的位置”记录了这些信息，这些位置称为**安全点（Safepoint）**，即程序执行时并非在所有地方都能停顿下来开始GC，只有在到达安全点时才能暂停。Safepoint的选定既不能太少以致于让GC等待时间太长，也不能过于频繁以致于过分增大运行时的负荷。

安全点意味着在这个点时，所有工作线程的状态是确定的，JVM 就可以安全地执行 GC 。

## **如何选定安全点？** 

安全点的选定是以**“是否具有让程序长时间执行的特征”**为标准进行选定的——因为每条指令执行的时间都非常短暂，程序不太可能因为指令流长度太长这个原因而过长时间运行，“长时间执行”的最明显特征就是指令序列复用，例如方法调用、循环跳转、异常跳转等，所以具有这些功能的指令才会产生Safepoint。

一般会在如下几个位置选择安全点：

1. 循环的末尾
2. 方法临返回前
3. 调用方法之后
4. 抛异常的位置

> 选择这些位置主要的目的就是避免程序长时间无法进入 Safe Point。比如 JVM 在做 GC 之前要等所有的应用线程进入安全点，如果有一个线程一直没有进入安全点，就会导致 GC 时 JVM 停顿时间延长。比如R大之前回复的这个[例子](http://hllvm.group.iteye.com/group/topic/38232),这里面就是写了一个超大的循环导致线程一直没有进入到安全点,GC前停顿了8秒.

## 安全点的使用场景

1. 垃圾回收(这是最常见的场景)
2. 取消偏向锁(JVM会使用偏向锁来优化锁的获取过程)
3. Class重定义(比如常见的hotswap和instrumentation)
4. Code Cache Flushing(JDK1.8在CodeCache满的情况下就可能出现)
5. 线程堆栈转储(jstack命令)

可以通过下面的代码测试偏向锁引起的STW

```
public class BiasedLocks {

    private static synchronized void contend() {
        LockSupport.parkNanos(100_000);
    }

    public static void main(String[] args) throws InterruptedException {

        Thread.sleep(5_000); // Because of BiasedLockingStartupDelay

        Stream.generate(() -> new Thread(BiasedLocks::contend))
                .limit(10)
                .forEach(Thread::start);
    }

}
```

在有些高并发的应用可以在启动参数里加一句`-XX:-UseBiasedLocking`取消掉偏向锁。

## 安全点上停止线程方式

由于在 GC 过程中必须保证程序已执行，那么也就是说必须等待所有线程都到达安全点上方可进行 GC。一般会有两种解决方案：

- **抢先式中断：**不需要线程的执行代码去主动配合，当发生 GC 时，先强制中断所有线程，然后如果发现某些线程未处于安全点，那么将其唤醒，直至其到达安全点再次将其中断；这样一直等待所有线程都在安全点后开始 GC。
- **主动式中断：**不强制中断线程，只是简单地设置一个中断标记，各个线程在执行时轮询这个标记，一旦发现标记被改变(出现中断标记)时，那么将运行到安全点后自己中断挂起；目前所有商用虚拟机全部采用主动式中断。

HotSpot 选定的标记位置与安全点位置是重合的

> 下面部分完全摘自 https://blog.csdn.net/iter_zc/article/details/41847887

对一个Java线程来说，它要么处在safepoint,要么不在safepoint。 

GC的标记阶段需要stop the world，让所有Java线程挂起，这样JVM才可以安全地来标记对象。safepoint可以用来实现让所有Java线程挂起的需求。这是一种"主动式"(Voluntary Suspension)的实现。JVM有两种执行方式：解释型和编译型(JIT)，JVM要保证这两种执行方式下safepoint都能工作。

在JIT执行方式下，JIT编译的时候直接把safepoint的检查代码加入了生成的本地代码，当JVM需要让Java线程进入safepoint的时候，只需要设置一个标志位，让Java线程运行到safepoint的时候主动检查这个标志位，如果标志被设置，那么线程停顿，如果没有被设置，那么继续执行。

例如hotspot在x86中为轮询safepoint会生成一条类似于“test %eax,0x160100”的指令，JVM需要进入gc前，先把0x160100设置为不可读，那所有线程执行到检查0x160100的test指令后都会停顿下来

```
0x01b6d627: call   0x01b2b210         ; OopMap{[60]=Oop off=460}    
                                       ;*invokeinterface size    
                                       ; - Client1::main@113 (line 23)    
                                       ;   {virtual_call}    
 0x01b6d62c: nop                       ; OopMap{[60]=Oop off=461}    
                                       ;*if_icmplt    
                                       ; - Client1::main@118 (line 23)    
 0x01b6d62d: test   %eax,0x160100      ;   {poll}    
 0x01b6d633: mov    0x50(%esp),%esi    
 0x01b6d637: cmp    %eax,%esi 
```

> **在解释器执行方式下**，JVM会设置一个2字节的dispatch tables,解释器执行的时候会经常去检查这个dispatch tables，当有safepoint请求的时候，就会让线程去进行safepoint检查。

VMThread会一直等待直到VMOperationQueue中有操作请求出现，比如GC请求。而VMThread要开始工作必须要等到所有的Java线程进入到safepoint。JVM维护了一个数据结构，记录了所有的线程，所以它可以快速检查所有线程的状态。当有GC请求时，所有进入到safepoint的Java线程会在一个Thread_Lock锁阻塞，直到当JVM操作完成后，VM释放Thread_Lock，阻塞的Java线程才能继续运行。

GC stop the world的时候，所有运行Java code的线程被阻塞，如果运行native code线程不去和Java代码交互，那么这些线程不需要阻塞。VM操作相关的线程也不会被阻塞。

## 安全点的执行阶段

safepoint的执行一共可以分为四个阶段：

- Spin阶段。因为jvm在决定进入全局safepoint的时候，有的线程在安全点上，而有的线程不在安全点上，这个阶段是等待未在安全点上的用户线程进入安全点。

- Block阶段。即使进入safepoint，用户线程这时候仍然是running状态，保证用户不在继续执行，需要将用户线程阻塞。

- Cleanup。这个阶段是JVM做的一些内部的清理工作。

- VM Operation. JVM执行的一些全局性工作，例如GC,代码反优化。

  只要分析这个四个阶段，就能知道什么原因导致的STW时间过长。

# 安全区域

安全点的机制似乎已经完美的解决了 “什么时候以及何时开始 GC” 的问题，但是实际情况并非如此；安全点机制仅仅是保证了程序执行时不需要太长时间就可以进入一个安全点进行 GC 动作，但是当特殊情况时，比如线程休眠、线程阻塞等状态的情况下，显然 JVM 不可能一直等待被阻塞或休眠的线程正常唤醒执行；对于这种情况，就需要安全区域（Safe Region）来解决。

**安全区(Saferegion)：**安全区域是指在一段区域内，对象引用关系等不会发生变化，在此区域内任意位置开始 GC 都是安全的；线程运行时，首先标记自己进入了安全区，然后在这段区域内，如果线程发生了阻塞、休眠等操作，JVM 发起 GC 时将忽略这些处于安全区的线程。当线程再次被唤醒时，首先他会检查是否完成了 GC Roots枚举(或这个GC过程)，然后选择是否继续执行，否则将继续等待 GC 的完成。

> safepoint只能处理正在运行的线程，它们可以主动运行到safepoint。而一些Sleep或者被blocked的线程不能主动运行到safepoint。这些线程也需要在GC的时候被标记检查，JVM引入了safe region的概念。safe region是指一块区域，这块区域中的引用都不会被修改，比如线程被阻塞了，那么它的线程堆栈中的引用是不会被修改的，JVM可以安全地进行标记。线程进入到safe region的时候先标识自己进入了safe region，等它被唤醒准备离开safe region的时候，先检查能否离开，如果GC已经完成，那么可以离开，否则就在safe region呆在。这可以理解，因为如果GC还没完成，那么这些在safe region中的线程也是被stop the world所影响的线程的一部分，如果让他们可以正常执行了，可能会影响标记的结果
> 
> https://blog.csdn.net/iter_zc/article/details/41847887

对于安全点更深入的源码分析可以参考下面的文章，因为我还没有深入到源码，所以后面也就没有仔细研究了

> https://www.jianshu.com/p/1f897ab1eed0
>
> https://www.jianshu.com/p/43bc97230299
>
> https://www.jianshu.com/p/c79c5e02ebe6
>
> https://blog.csdn.net/iter_zc/article/details/41892567

# 参考资料

https://mritd.me/2016/03/24/HotSpot-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%AE%97%E6%B3%95%E5%AE%9E%E7%8E%B0/

https://m.aliyun.com/yunqi/articles/87842

https://www.ezlippi.com/blog/2018/01/safepoint.html

https://www.zhihu.com/question/29268019/answer/43762165