---
layout: post
title: 元空间
date: 2020-04-26
categories:
    - jvm
comments: true
permalink: metaspace.html
---

在JVM运行时数据区域已经了解了[元空间](https://edgar615.github.io/jvm-runtime-data-area.html)，后面看了你假笨的一些文章有了更深的了解，这里抄一下内容做备份

# metaspace的组成

metaspace其实由两大部分组成

- Klass Metaspace
- NoKlass Metaspace

Klass Metaspace就是用来存klass的，klass是我们熟知的class文件在jvm里的运行时数据结构，不过有点要提的是我们看到的类似A.class其实是存在heap里的，是java.lang.Class的一个对象实例。**这块内存是紧接着Heap的**，和我们之前的perm一样，这块内存大小可通过`-XX:CompressedClassSpaceSize`参数(默认1G)来控制。但是这块内存也可以没有，假如没有开启压缩指针就不会有这块内存，这种情况下klass都会存在NoKlass Metaspace里，另外如果我们把-Xmx设置大于32G的话，其实也是没有这块内存的，因为会这么大内存会关闭压缩指针开关。还有就是**这块内存最多只会存在一块。**

**只有开启指针压缩才会有Klass Metaspace**

NoKlass Metaspace专门来存klass相关的其他的内容，比如method，constantPool等，**这块内存是由多块内存组合起来的，所以可以认为是不连续的内存块组成的。**这块内存是必须的，虽然叫做NoKlass Metaspace，但是也其实可以存klass的内容，上面已经提到了对应场景。

Klass Metaspace和NoKlass Mestaspace都是所有classloader共享的，所以类加载器们要分配内存，但是每个类加载器都有一个SpaceManager，来管理属于这个类加载的内存小块。如果Klass Metaspace用完了，那就会OOM了，不过一般情况下不会，NoKlass Mestaspace是由一块块内存慢慢组合起来的，在没有达到限制条件的情况下，会不断加长这条链，让它可以持续工作。

# 参数

## UseLargePagesInMetaspace

默认false，这个参数是说是否在metaspace里使用LargePage，一般情况下我们使用4KB的page size，这个参数依赖于UseLargePages这个参数开启，不过这个参数我们一般不开

## InitialBootClassLoaderMetaspaceSize

64位下默认4M，32位下默认2200K，metasapce前面已经提到主要分了两大块，Klass Metaspace以及NoKlass 
Metaspace，而NoKlass Metaspace是由一块块内存组合起来的，**这个参数决定了NoKlass Metaspace的第一个内存Block的大小，即2*InitialBootClassLoaderMetaspaceSize**，同时为bootstrapClassLoader的第一块内存chunk分配了InitialBootClassLoaderMetaspaceSize的大小

## MetaspaceSize

默认20.8M左右(x86下开启c2模式)，主要是控制metaspaceGC发生的初始阈值，也是最小阈值，但是触发metaspaceGC的阈值是不断变化的，与之对比的主要是指Klass Metaspace与NoKlass Metaspace两块**committed**的内存和。

这个JVM参数是指**Metaspace扩容时触发FGC的初始化阈值**，表示metaspace**首次**使用不够而触发FGC的阈值，只对触发起作用，原因是：垃圾搜集器内部是根据变量`_capacity_until_GC`来判断metaspace区域是否达到阈值的，初始化代码如下所示：

```c
void MetaspaceGC::initialize() {
  // Set the high-water mark to MaxMetapaceSize during VM initializaton since
  // we can't do a GC during initialization.
  _capacity_until_GC = MaxMetaspaceSize;
}
```

GC收集器会在发生对metaspace的回收后，会计算新的_capacity_until_GC值，以后发生FGC就跟MetaspaceSize没有关系了。

1. 如果没有配置参数`-XX:MetaspaceSize`，那么触发FGC的阈值就是21807104（约20.8M）；
2. 如果配置了参数`-XX:MetaspaceSize=256m`，那么触发FGC的阈值就是配置的值256M；
3. Metaspace由于使用不断扩容到`-XX:MetaspaceSize`参数指定的量，就会发生FGC；且之后每次Metaspace扩容可能发生FGC；
4. 如果Old区配置CMS垃圾回收，那么扩容引起的FGC也会使用CMS算法进行回收；
5. 如果MaxMetaspaceSize设置太小，可能会导致频繁FGC，甚至OOM；

Metaspace触发Full GC，是因为Metaspace committed的内存加上这次要分配的内存之和超过了阈值才会触发。

当 Metaspace 内存占用**未**达到 -XX:MetaspaceSize 时，Metaspace 只扩容，不会引起 Full GC。当 Metaspace 内存占用达到 -XX:MetaspaceSize 时，会发生 Full GC。在发生第一次 Full GC 之后，Metaspace 依然会扩容。

第二次触发 Full GC 的条件目前还不明确，还需要多查资料。

## MaxMetaspaceSize

默认基本是无穷大，建议设置这个参数，因为很可能会因为没有限制而导致metaspace被无止境使用(一般是内存泄漏)而被OS Kill。这个参数会限制metaspace(包括了Klass Metaspace以及NoKlass Metaspace)被committed的内存大小，**会保证committed的内存不会超过这个值**，一旦超过就会触发GC，这里要注意和MaxPermSize的区别，**MaxMetaspaceSize并不会在jvm启动的时候分配一块这么大的内存出来**，而MaxPermSize是会分配一块这么大的内存的。

## CompressedClassSpaceSize

默认1G，这个参数主要是设置Klass Metaspace的大小，不过这个参数设置了也不一定起作用，前提是能开启压缩指针，假如-Xmx超过了32G，压缩指针是开启不来的。如果有Klass Metaspace，那这块内存是和Heap连着的。

## MinMetaspaceFreeRatio

MinMetaspaceFreeRatio和下面的MaxMetaspaceFreeRatio，主要是影响触发metaspaceGC的阈值

默认40，表示每次GC完之后，假设我们允许接下来metaspace可以继续被commit的内存占到了被commit之后总共committed的内存量的MinMetaspaceFreeRatio%，如果这个总共被committed的量比当前触发metaspaceGC的阈值要大，那么将尝试做扩容，也就是增大触发metaspaceGC的阈值，不过这个增量至少是MinMetaspaceExpansion才会做，不然不会增加这个阈值

**这个参数主要是为了避免触发metaspaceGC的阈值和gc之后committed的内存的量比较接近，于是将这个阈值进行扩大**

一般情况下在gc完之后，如果被committed的量还是比较大的时候，换个说法就是离触发metaspaceGC的阈值比较接近的时候，这个调整会比较明显

## MaxMetaspaceFreeRatio

默认70，这个参数和上面的参数基本是相反的，是为了避免触发metaspaceGC的阈值过大，而想对这个值进行缩小。这个参数在gc之后committed的内存比较小的时候并且离触发metaspaceGC的阈值比较远的时候，调整会比较明显

## MinMetaspaceExpansion

MinMetaspaceExpansion和MaxMetaspaceExpansion这两个参数和扩容其实并没有直接的关系，也就是并不是为了增大committed的内存，**而是为了增大触发metaspace GC的阈值**

这两个参数主要是在比较特殊的场景下救急使用，比如gcLocker或者`should_concurrent_collect`的一些场景，因为这些场景下接下来会做一次GC，相信在接下来的GC中可能会释放一些metaspace的内存，于是先临时扩大下metaspace触发GC的阈值，而有些内存分配失败其实正好是因为这个阈值触顶导致的，于是可以通过增大阈值暂时绕过去

默认332.8K，增大触发metaspace  GC阈值的最小要求。假如我们要救急分配的内存很小，没有达到MinMetaspaceExpansion，但是我们会将这次触发metaspace  GC的阈值提升MinMetaspaceExpansion，之所以要大于这次要分配的内存大小主要是为了防止别的线程也有类似的请求而频繁触发相关的操作，不过如果要分配的内存超过了MaxMetaspaceExpansion，那MinMetaspaceExpansion将会是要分配的内存大小基础上的一个增量

## MaxMetaspaceExpansion

默认5.2M，增大触发metaspace  GC阈值的最大要求。假如说我们要分配的内存超过了MinMetaspaceExpansion但是低于MaxMetaspaceExpansion，那增量是MaxMetaspaceExpansion，如果超过了MaxMetaspaceExpansion，那增量是MinMetaspaceExpansion加上要分配的内存大小

注：每次分配只会给对应的线程一次扩展触发metaspace GC阈值的机会，如果扩展了，但是还不能分配，那就只能等着做GC了

# 元空间的内存泄漏

> 只摘录了部分内容，完整内容查看你假笨的文章：https://mp.weixin.qq.com/s/3sb_ovHhhTXTid3G5iZUew

当我们多次调用ClassLoader的defineClass方法的时候哪怕是同一个类加载器加载同一个类文件，在JVM里也会在对应的Perm或者Metaspace里创建多份Klass结构，当然一般情况下我们不会直接这么调用。

## 重复类定义带来的影响

正常的类加载都会先走一遍缓存查找，看是否已经有了对应的类，如果有了就直接返回，如果没有就进行定义，如果直接调用类定义的方法，在JVM里会创建多份临时的类结构实例，这些相关的结构是存在Perm或者Metaspace里的，也就是说会消耗Perm或Metaspace的内存，但是这些类在定义出来之后，最终会做一次约束检查，如果发现已经定义了，那就直接抛出LinkageError的异常。

这样这些临时创建的结构，只能等待GC的时候去回收掉了，因为它们不可达，所以在GC的时候会被回收，这些结构在Perm下能正常回收，但是在Metaspace里不能正常回收。

## Perm和Metaspace在类卸载上的差异

在JDK7 CMS下，Perm的结构其实和Old的内存结构是一样的，如果Perm不够的时候我们会做一次Full GC，这个Full GC默认情况下是会对各个分代做压缩的，包括Perm，这样一来根据对象的可达性，任何一个类都只会和一个活着的类加载器绑定，在标记阶段将这些类标记成活的，并将他们进行**新地址的计算及移动压缩**，而之前因为重复定义生成的类结构等，因为没有将它们和任何一个活着的类加载器关联(有个叫做SystemDictionary的Hashtable结构来记录这种关联)，从而在压缩过程中会被回收掉。

在JDK8下，Metaspace是完全独立分散的内存结构，由非连续的内存组合起来，在Metaspace达到了触发GC的阈值的时候(和MaxMetaspaceSize及MetaspaceSize有关)，就会做一次Full GC，但是这次Full GC，并不会对Metaspace做压缩，**唯一卸载类的情况是，对应的类加载器必须是死的**，如果类加载器都是活的，那肯定不会做卸载的事情了

JDK7里会对Perm做压缩，然后JDK8里并不会对Metaspace做压缩，从而只要和那些重复定义的类相关的类加载一直存活，那将一直不会被回收，但是如果类加载死了，那就会被回收，这是因为那些重复类都是在和这个类加载器关联的内存块里分配的，如果这个类加载器死了，那整块内存会被清理并被下次重用。

# 类加载器过多为什么会导致Full GC

> https://mp.weixin.qq.com/s/qgpMMR8-493-Y9uwWiMdRg

类加载器创建过多，带来的一个问题是，**在类加载器第一次加载类的时候，会在Metaspace里会给它分配内存块，为了分配高效，每个类加载器用来存放类信息的内存块都是独立的，所以哪怕你这个类加载器只加载一个类，也会为之分配一块空的内存给这个类加载器**，其实是至少两个内存块，于是你有可能会发现Metaspace的内存使用率非常低，但是committed的内存已经达到了阈值，从而触发了Full GC，如果这种只加载很少类的类加载器非常多，那造成的后果就是很多碎片化的内存

# 参考资料

https://mp.weixin.qq.com/s/SsXbRvtvawKDHstFpU4uog

https://mp.weixin.qq.com/s/jjCObe_WaTg62HSix-hdnw

https://www.jianshu.com/p/5ee71f1724cd

https://mp.weixin.qq.com/s/3sb_ovHhhTXTid3G5iZUew

https://mp.weixin.qq.com/s/qgpMMR8-493-Y9uwWiMdRg

https://www.jianshu.com/p/468fb4c5b28d