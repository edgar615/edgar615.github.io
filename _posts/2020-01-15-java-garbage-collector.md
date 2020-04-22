---
layout: post
title: JVM垃圾收集器
date: 2020-01-15
categories:
    - jvm
comments: true
permalink: java-garbage-collector.html
---

> 内容基本来自参考资料


HotSpot虚拟机中有7种垃圾收集器：Serial、ParNew、Parallel Scavenge、Serial Old、Parallel Old、CMS、G1

![](/assets/images/posts/garbage-collector/garbage-collector-1.png)

- 在新生代工作的垃圾回收器：Serial, ParNew, ParallelScavenge
- 在老年代工作的垃圾回收器：CMS，Serial Old, Parallel Old
- 同时在新老生代工作的垃圾回收器：G1

图片中的垃圾收集器如果存在连线，则代表它们之间可以配合使用，接下来我们来看看各个垃圾收集器的具体功能。

# Serial垃圾收集器

Serial（英文连续）是最基本垃圾收集器，使用**复制算法**，曾经是 JDK1.3.1 之前新生代唯一的垃圾收集器。

Serial 是一个单线程的收集器，它不但只会使用一个 CPU 或一条线程去完成垃圾收集工作，**并且在进行垃圾收集的同时，必须暂停其他所有的工作线程，直到垃圾收集结束**，也就是说在 GC 期间，此时的应用不可用。Serial 垃圾收集器虽然在收集垃圾过程中需要暂停所有其他的工作线程，但是它简单高效，对于限定单个 CPU 环境来说，没有线程交互的开销，可以获得最高的单线程垃圾收集效率，因此 Serial垃圾收集器依然是 java 虚拟机运行在 **Client** 模式下默认的新生代垃圾收集器。

![](/assets/images/posts/garbage-collector/garbage-collector-2.png)

可以添加参数`-XX:+UseSerialGC`来显式的使用串行垃圾收集器；

# ParNew垃圾收集器

ParNew 垃圾收集器其实是 Serial 收集器的多线程版本，也使用**复制算法**，除了使用多线程进行垃圾收集之外，其余的行为和 Serial 收集器完全一样，ParNew 垃圾收集器在垃圾收集过程中同样也要暂停所有其他的工作线程。

在多 CPU 的情况下，由于 ParNew 的多线程回收特性，毫无疑问垃圾收集会更快，也能有效地减少 STW 的时间，提升应用的响应速度。

ParNew 收集器默认开启和 CPU 数目相同的线程数，可以通过`-X:ParallelGCThreads` 参数来限制垃圾收集器的线程数。ParNew 虽然是除了多线程外和 Serial 收集器几乎完全一样，但是 ParNew 垃圾收集器是很多 java虚拟机运行在 **Server** 模式下新生代的默认垃圾收集器。

![](/assets/images/posts/garbage-collector/garbage-collector-2.1.jpg)

参数

- `-XX:+UseConcMarkSweepGC`：指定使用CMS后，会默认使用ParNew作为新生代收集器；
- `-XX:+UseParNewGC`：强制指定使用ParNew；    
- `-XX:ParallelGCThreads`：指定垃圾收集的线程数量，ParNew默认开启的收集线程与CPU的数量相同

除了 Serial  收集器，**只有它能与 CMS 收集器配合工作**，CMS 是一个划时代的垃圾收集器，是真正意义上的**并发收集器**，它第一次实现了垃圾收集线程与用户线程（基本上）同时工作，它采用的是传统的GC 收集器代码框架，与 Serial,ParNew 共用一套代码框架，所以能与这两者一起配合工作，而后文提到的 Parallel Scavenge 与 G1 收集器没有使用传统的 GC收集器代码框架，而是另起炉灶独立实现的，另外一些收集器则只是共用了部分的框架代码,所以无法与 CMS 收集器一起配合工作。

# Parallel Scavenge收集器

Parallel Scavenge 收集器也是一个**新生代**垃圾收集器，同样使用**复制算法**，也是一个多线程的垃圾收集器，但它与ParNew 收集器的关注点不同。ParNew 等垃圾收集器关注的是尽可能缩短垃圾收集时用户线程的停顿时间，而 Parallel Scavenge它重点关注的是程序达到一个可控制的吞吐量

- 吞吐量：CPU用于运行用户代码的时间与CPU消耗的总时间的比值。
- 吞吐量 = （执行用户代码时间）/（执行用户代码时间+垃圾回收占用时间）

也就是说 ParNew 等垃圾收集器更适合用到与用户交互的程序，因为停顿时间越短，用户体验越好，而 Parallel Scavenge 收集器关注的是吞吐量，高吞吐量可以最高效率地利用 CPU 时间，尽快地完成程序的运算任务，所以更适合做后台运算等不需要太多用户交互的任务。

**自适应调节策略也是 ParallelScavenge 收集器与 ParNew 收集器的一个重要区别。**

参数

- `-XX:MaxGCPauseMillis`  控制最大垃圾收集停顿时间，大于0的毫秒数；      MaxGCPauseMillis设置得稍小，停顿时间可能会缩短，但也可能会使得吞吐量下降； 因为可能导致垃圾收集发生得更频繁；
- `-XX:GCTimeRatio`  设置垃圾收集时间占总时间的比率，0<n<100的整数；GCTimeRatio相当于设置吞吐量大小； 垃圾收集执行时间占应用程序执行时间的比例的计算方法是：`1 / (1 + n)`例如选项`-XX:GCTimeRatio=19`，设置了垃圾收集时间占总时间的`5%=1/(1+19)`；默认值是99，`1%=1/(1+99)`
- `-XX:UseAdaptiveSizePolicy` 开启这个参数后，就不需要手工指定新生代大小,Eden 与 Survivor 
  比例（SurvivorRatio）等细节，只需要设置好基本的堆大小（-Xmx 
  设置最大堆）,以及最大垃圾收集时间与吞吐量大小，虚拟机就会根据当前系统运行情况收集监控信息，动态调整这些参数以尽可能地达到我们设定的最大垃圾收集时间或吞吐量大小这两个指标

# Serial Old收集器

Serial Old 是 Serial 垃圾收集器年老代版本，它同样是个单线程的收集器，使用**标记 -整理算法**，这个收集器也主要是运行在 Client 默认的 java 虚拟机默认的年老代垃圾收集器。

在 Server 模式下，主要有两个用途：

- 在 JDK1.5 之前版本中与新生代的 Parallel Scavenge 收集器搭配使用。
- 作为年老代中使用 CMS 收集器的后备垃圾收集方案。在并发收集发生 Concurrent Mode Failure 时使用。

![](/assets/images/posts/garbage-collector/garbage-collector-3.jpg)

# Parallel Old收集器

Parallel Old 收集器是 Parallel Scavenge 的年老代版本，使用多线程的**标记-整理算法**，在 JDK1.6
才开始提供。在 JDK1.6 之前，新生代使用 ParallelScavenge 收集器只能搭配年老代的 Serial Old 收集器，只能保证新生代的吞吐量优先，无法保证整体的吞吐量，Parallel Old 正是为了在年老代同样提供吞吐量优先的垃圾收集器，如果系统对吞吐量要求比较高，可以优先考虑新生代 Parallel Scavenge和年老代 Parallel Old 收集器的搭配策略。

新生代 Parallel Scavenge 和年老代 Parallel Old 收集器搭配运行过程图：

![](/assets/images/posts/garbage-collector/garbage-collector-4.jpg)

参数

- `-XX:+UseParallelOldGC`：指定使用Parallel Old收集器；

# CMS
CMS（Concurrent Mark and Sweep 并发-标记-清除），是一款基于并发、使用**标记清除算法**的垃圾回收算法，只针对老年代进行垃圾回收。CMS 收集器工作时，尽可能让 GC 线程和用户线程并发执行，以达到降低 STW 时间的目的。

通过`-XX:+UseConcMarkSweepGC`参数启用 CMS 垃圾收集器

## 新生代垃圾回收
能与 CMS 搭配使用的新生代垃圾收集器有 Serial 收集器和 ParNew 收集器。

## 老年代垃圾回收
CMS GC 以获取最小停顿时间为目的，尽可能减少 STW 时间，可以分为 7 个阶段：

![](/assets/images/posts/garbage-collector/garbage-collector-5.jpg)

> 结合GC日志加深理解

### 阶段 1：初始标记（Initial Mark）
此阶段的目标是标记老年代中所有存活的对象, 包括 GC Root 的直接引用, 以及由新生代中存活对象所引用的对象，**触发第一次 STW 事件**。这个过程是支持多线程的（JDK7 之前单线程，JDK8 之后并行，可通过参数 `CMSParallelInitialMarkEnabled` 调整）。

![](/assets/images/posts/garbage-collector/garbage-collector-6.jpg)

### 阶段 2：并发标记（Concurrent Mark）
此阶段 GC 线程和应用线程并发执行，遍历阶段 1 初始标记出来的存活对象，然后继续递归标记这些对象可达的对象。

![](/assets/images/posts/garbage-collector/garbage-collector-7.jpg)

### 阶段 3：并发预清理（Concurrent Preclean）
此阶段 GC 线程和应用线程也是并发执行用于标记老年代存活的对象。这个阶段目的是为了让重新标记阶段的STW尽可能短。这个阶段的目标是在并发标记阶段被应用线程影响到的老年代对象。因为阶段 2 是与应用线程并发执行，可能有些引用关系已经发生改变：有些对象会从新生代晋升到老年代、有些老年代的对象引用会被改变、有些对象会直接分配到老年代。 

通过卡片标记（Card Marking），提前把老年代空间逻辑划分为相等大小的区域（Card）。

如果引用关系发生改变，JVM 会将发生改变的区域标记为“脏区”（Dirty Card），然后在本阶段，这些脏区会被找出来，刷新引用关系，清除“脏区”标记。

![](/assets/images/posts/garbage-collector/garbage-collector-8.jpg)

### 阶段 4：并发可取消的预清理（Concurrent Abortable Preclean）
此阶段也不停止应用线程。本阶段尝试在 STW 的最终标记阶段（Final Remark）之前尽可能地多做一些工作，以减少应用暂停时间。

在该阶段不断循环处理：标记老年代的可达对象、扫描处理 Dirty Card 区域中的对象，循环的终止条件有：

- 达到循环次数
- 达到循环执行时间阈值
- 新生代内存使用率达到阈值

可中断预清理的价值：在进入重新标记阶段之前尽量等到一个Minor GC，尽量缩短重新标记阶段的停顿时间。另外可中断预清理会在Eden达到50%的时候开始，这时候离下一次minor gc还有半程的时间，这个还有另一个意义，即避免短时间内连着的两个停顿

在预清理步骤后，如果满足下面两个条件，就不会开启可中断的预清理，直接进入重新标记阶段：

- Eden的使用空间大于`CMSScheduleRemarkEdenSizeThreshold`，这个参数的默认值是2M；
- Eden的使用率大于等于`CMSScheduleRemarkEdenPenetration`，这个参数的默认值是50%。

如果不满足上面两个条件，则进入可中断的预清理，可中断预清理可能会执行多次，那么退出这个阶段的出口有两个

- 设置了`CMSMaxAbortablePrecleanLoops`，并且执行的次数超过了这个值，这个参数的默认值是0；
- 执行可中断预清理的时间超过了`CMSMaxAbortablePrecleanTime`，这个参数的默认值是5000毫秒。

有可能可中断预清理过程中一直没等到Minor gc，这时候进入最终标记阶段的话，新生代还有很多活着的对象，就回导致STW变长，因此CMS还提供了`CMSScavengeBeforeRemark`参数，可以在进入最终标记之前强制进行依次Minor gc。

### 阶段 5：最终标记（Final Remark）
**这是 GC 事件中第二次（也是最后一次）STW 阶段**，目标是完成老年代中所有存活对象的标记。
在此阶段执行：

- 遍历新生代对象，重新标记
- 根据 GC Roots，重新标记
- 遍历老年代的 Dirty Card，重新标记

如果预清理的工作没做好，这一步扫描新生代的时候就会花很多时间，导致这个阶段的停顿时间过长。这个过程是多线程的

### 阶段 6：并发清除（Concurrent Sweep）
此阶段与应用程序并发执行，不需要 STW 停顿，根据标记结果清除垃圾对象。

![](/assets/images/posts/garbage-collector/garbage-collector-9.jpg)

### 阶段 7：并发重置（Concurrent Reset）
此阶段与应用程序并发执行，重置 CMS 算法相关的内部数据, 为下一次 GC 循环做准备。

从上面的描述中可以看到初始标记和最终标记两个阶段会发生 STW，造成用户线程挂起，不过初始标记仅标记 GC Roots 能关联的对象，速度很快，并发标记是进行 GC Roots  Tracing  的过程，重新标记是为了修正并发标记期间因用户线程继续运行而导致标记产生变动的那一部分对象的标记记录，这一阶段停顿时间一般比初始标记阶段稍长，但**远比并发标记时间短**。

整个过程中耗时最长的是并发标记和标记清理，不过这两个阶段用户线程都可工作，所以不影响应用的正常使用。

## CMS常见问题
### CPU敏感
CMS 收集器对 CPU 资源非常敏感  原因也可以理解，比如本来我本来可以有 10 个用户线程处理请求，现在却要分出 3 个作为回收线程，吞吐量下降了30%，CMS 默认启动的回收线程数是 `（CPU数量+3）/ 4`, 如果 CPU 数量只有一两个，那吞吐量就直接下降 50%,显然是不可接受的

### 最终标记阶段停顿时间过长问题

CMS 的 GC 停顿时间约 80% 都在最终标记阶段（Final Remark），若该阶段停顿时间过长，常见原因是新生代对老年代的无效引用，在上一阶段的并发可取消预清理阶段中，执行阈值时间内未完成循环，来不及触发 Young GC，清理这些无效引用

通过添加参数：`-XX:+CMSScavengeBeforeRemark`在执行最终操作之前先触发 Young GC，从而减少新生代对老年代的无效引用，降低最终标记阶段的停顿。但如果在上个阶段（并发可取消的预清理）已触发 Young GC，也会重复触发 Young GC。

## 浮动垃圾

CMS 无法处理浮动垃圾（Floating Garbage）,可能出现 「Concurrent Mode Failure」而导致另一次 Full GC 的产生，由于在并发清理阶段用户线程还在运行，所以清理的同时新的垃圾也在不断出现，这部分垃圾只能在下一次 GC 时再清理掉（即浮动垃圾）。同时在垃圾收集阶段用户线程也要继续运行，就需要预留足够多的空间要确保用户线程正常执行，这就意味着 CMS 收集器不能像其他收集器一样等老年代满了再使用，JDK 1.5 默认当老年代使用了68%空间后就会被激活，jdk1.6之后是92%，当然这个比例可以通过` -XX:CMSInitiatingOccupancyFraction` 来设置，但是如果设置地太高很容易导致在 CMS 运行期间预留的内存无法满足程序要求，会导致 **Concurrent Mode Failure** 失败，这时会启用 Serial Old 收集器来重新进行老年代的收集，而我们知道 Serial Old 收集器是单线程收集器，这样就会导致 STW 更长了。

### 并发模式失败（concurrent mode failure）&晋升失败（promotion failed）问题。
**并发模式失败**：当 CMS 在执行回收时，新生代发生垃圾回收，同时老年代又没有足够的空间容纳晋升的对象时，CMS 垃圾回收就会退化成单线程的 Full GC。所有的应用线程都会被暂停，老年代中所有的无效对象都被回收。

![](/assets/images/posts/garbage-collector/garbage-collector-10.jpg)

**晋升失败**：当新生代发生垃圾回收，老年代有足够的空间可以容纳晋升的对象，但是由于空闲空间的碎片化，导致晋升失败，此时会触发单线程且带压缩动作的 Full GC。

![](/assets/images/posts/garbage-collector/garbage-collector-11.jpg)

> ### 空间分配担保：
>
> 在发生Minor  GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那么Minor  GC可以确保是安全的，如果不成立，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。如果允许，则继续检查老年代最大可用连续空间是否大于历次晋升到老年代对象的平均大小，如果是，就尝试进行一次Minor  GC(失败进行Full GC),如果不是，或者不允许担保失败，改为进行一次Full GC。

并发模式失败和晋升失败都会导致长时间的停顿，常见解决思路如下：

- 降低触发 CMS GC 的阈值。即参数` -XX:CMSInitiatingOccupancyFraction` 的值，让 CMS GC 尽早执行，以保证有足够的空间。
- 增加 CMS 线程数，即参数`-XX:ConcGCThreads`。
- 增大老年代空间。让对象尽量在新生代回收，避免进入老年代。

### 内存碎片问题
通常 CMS 的 GC 过程基于标记清除算法，不带压缩动作，导致越来越多的内存碎片需要压缩。常见以下场景会触发内存碎片压缩：

- 新生代 Young GC 出现新生代晋升担保失败（promotion failed)）
- 程序主动执行System.gc()

可通过参数 `CMSFullGCsBeforeCompaction`的值，设置多少次 Full GC 触发一次压缩。默认值为 0，代表每次进入 Full GC 都会触发压缩，带压缩动作的算法为上面提到的单线程 Serial Old 算法，暂停时间（STW）时间非常长，需要尽可能减少压缩时间。也可以通过`-XX:+UseCMSCompactAtFullCollection`在 CMS 收集器顶不住要进行 FullGC 时开启内存碎片的合并整理过程

### 永久代空间（或Java8的元空间）耗尽
默认情况下,CMS不会对永久代进行收集，一旦永久代空间耗尽，就回触发Full GC。

## **CMS的调优**

### **针对停顿时间过长的调优**
   首先需要判断是哪个阶段的停顿导致的，然后再针对具体的原因进行调优。使用CMS收集器的JVM可能引发停顿的情况有：

1. Minor gc的停顿；
2. 并发周期里初始标记的停顿；
3. 并发周期里重新标记的停顿；
4. Serial-Old收集老年代的停顿；
5. Full GC的停顿。

其中并发模式失败会导致第4种情况，晋升失败和永久代空间耗尽会导致第5种情况。

## 参数
1. `UseConcMarkSweepGC`	启用CMS收集器
2. `UseCMSInitiatingOccupancyOnly` 关闭CMS的动态检查机制，只通过预设的阈值来判断是否启动并发收集周期
3. `CMSInitiatingOccupancyFraction` 老年代空间占用到多少的时候启动并发收集周期，跟UseCMSInitiatingOccupancyOnly一起使用
4. `ExplicitGCInvokesConcurrentAndUnloadsClasses` 将System.gc()触发的Full GC转换为一次CMS并发收集，并且在这个收集周期中卸载     Perm（Metaspace）区域中不需要的类
5. `CMSClassUnloadingEnabled` 在CMS收集周期中，是否卸载类
6. `ParallelRefProcEnabled` 是否开启并发引用处理
7. `CMSScavengeBeforeRemark` 如果开启这个参数，会在进入重新标记阶段之前强制触发一次minor gc

# G1
G1（Garbage-First）是一款面向服务器的垃圾收集器，支持新生代和老年代空间的垃圾收集，主要针对配备多核处理器及大容量内存的机器。G1 最主要的设计目标是：实现可预期及可配置的 STW 停顿时间。

## G1 堆空间划分

![](/assets/images/posts/garbage-collector/garbage-collector-12.jpg)

**Region**

为实现大内存空间的低停顿时间的回收，将划分为多个大小相等的 Region。每个小堆区都可能是 Eden 区，Survivor 区或者 Old 区，但是在同一时刻只能属于某个代。在逻辑上, 所有的 Eden 区和 Survivor 区合起来就是新生代，所有的 Old 区合起来就是老年代，且新生代和老年代各自的内存 Region 区域由 G1 自动控制，不断变动。

**巨型对象**

当对象大小超过 Region 的一半，则认为是巨型对象（Humongous Object），直接被分配到老年代的巨型对象区（Humongous Regions）。这些巨型区域是一个连续的区域集，每一个 Region 中最多有一个巨型对象，巨型对象可以占多个 Region。

G1 把堆内存划分成一个个 Region 的意义在于：

- 每次 GC 不必都去处理整个堆空间，而是每次只处理一部分 Region，实现大容量内存的 GC。
- 通过计算每个 Region 的回收价值，包括回收所需时间、可回收空间，在有限时间内尽可能回收更多的垃圾对象，把垃圾回收造成的停顿时间控制在预期配置的时间范围内，这也是 G1 名称的由来：Garbage-First。

## G1工作模式

针对新生代和老年代，G1 提供 2 种 GC 模式，Young GC 和 Mixed GC，两种会导致 Stop The World。

**Young GC**：当新生代的空间不足时，G1 触发 Young GC 回收新生代空间。

Young GC 主要是对 Eden 区进行 GC，它在 Eden 空间耗尽时触发，基于分代回收思想和复制算法，每次 Young GC 都会选定所有新生代的 Region。

同时计算下次 Young GC 所需的 Eden 区和 Survivor 区的空间，动态调整新生代所占 Region 个数来控制 Young GC 开销。

**Mixed GC**：当老年代空间达到阈值会触发 Mixed GC，选定所有新生代里的 Region，根据全局并发标记阶段(下面介绍到)统计得出收集收益高的若干老年代 Region。

在用户指定的开销目标范围内，尽可能选择收益高的老年代 Region 进行 GC，通过选择哪些老年代 Region 和选择多少 Region 来控制 Mixed GC 开销。

### 全局并发标记

![](/assets/images/posts/garbage-collector/garbage-collector-13.jpg)

全局并发标记主要是为 Mixed GC 计算找出回收收益较高的 Region 区域，具体分为 5 个阶段：

#### 阶段 1：初始标记（Initial Mark）

暂停所有应用线程（STW），并发地进行标记从 GC Root 开始直接可达的对象（原生栈对象、全局对象、JNI 对象）。当达到触发条件时，G1 并不会立即发起并发标记周期，而是等待下一次新生代收集，利用新生代收集的 STW 时间段，完成初始标记，这种方式称为借道（Piggybacking）。

#### 阶段 2：根区域扫描（Root Region Scan）

在初始标记暂停结束后，新生代收集也完成的对象复制到 Survivor 的工作，应用线程开始活跃起来。此时为了保证标记算法的正确性，所有新复制到 Survivor 分区的对象，需要找出哪些对象存在对老年代对象的引用，把这些对象标记成根（Root）。这个过程称为根分区扫描（Root Region Scanning），同时扫描的 Suvivor 分区也被称为根分区（Root Region）。

根分区扫描必须在下一次新生代垃圾收集启动前完成（接下来并发标记的过程中，可能会被若干次新生代垃圾收集打断），因为每次 GC 会产生新的存活对象集合。

#### 阶段 3：并发标记（Concurrent Marking）

标记线程与应用程序线程并行执行，标记各个堆中 Region 的存活对象信息，这个步骤可能被新的 Young GC 打断。所有的标记任务必须在堆满前就完成扫描，如果并发标记耗时很长，那么有可能在并发标记过程中，又经历了几次新生代收集。

#### 阶段 4：再次标记（Remark）

和 CMS 类似暂停所有应用线程（STW），以完成标记过程短暂地停止应用线程, 标记在并发标记阶段发生变化的对象，和所有未被标记的存活对象，同时完成存活数据计算。

#### 阶段 5：清理（Cleanup）

为即将到来的转移阶段做准备, 此阶段也为下一次标记执行所有必需的整理计算工作：

- 整理更新每个 Region 各自的 RSet（Remember Set，HashMap 结构，记录有哪些老年代对象指向本 Region，key 为指向本 Region 的对象的引用，value 为指向本 Region 的具体 Card 区域，通过 RSet 可以确定 Region 中对象存活信息，避免全堆扫描）。
- 回收不包含存活对象的 Region。
- 统计计算回收收益高（基于释放空间和暂停目标）的老年代分区集合。

## G1调优注意点

### Full GC 问题

G1 的正常处理流程中没有 Full GC，只有在垃圾回收处理不过来（或者主动触发）时才会出现，G1 的 Full GC 就是单线程执行的 Serial old gc，会导致非常长的 STW，是调优的重点，需要尽量避免 Full GC。

常见原因如下： 

- 程序主动执行 System.gc()
- 全局并发标记期间老年代空间被填满（并发模式失败）
- Mixed GC 期间老年代空间被填满（晋升失败）
- Young GC 时 Survivor 空间和老年代没有足够空间容纳存活对象

类似 CMS，常见的解决是：

- 增大` -XX:ConcGCThreads=n` 选项增加并发标记线程的数量，或者 STW 期间并行线程的数量：`-XX:ParallelGCThreads=n`。
- 减小` -XX:InitiatingHeapOccupancyPercent` 提前启动标记周期。
- 增大预留内存` -XX:G1ReservePercent=n`，默认值是 10，代表使用 10% 的堆内存为预留内存，当 Survivor 区域没有足够空间容纳新晋升对象时会尝试使用预留内存。

### 巨型对象分配
巨型对象区中的每个 Region 中包含一个巨型对象，剩余空间不再利用，导致空间碎片化，当 G1 没有合适空间分配巨型对象时，G1 会启动串行 Full GC 来释放空间。

可以通过增加 `-XX:G1HeapRegionSize` 来增大 Region 大小，这样一来，相当一部分的巨型对象就不再是巨型对象了，而是采用普通的分配方式。

### 不要设置 Young 区的大小

原因是为了尽量满足目标停顿时间，逻辑上的 Young 区会进行动态调整。如果设置了大小，则会覆盖掉并且会禁用掉对停顿时间的控制

#### 平均响应时间设置

使用应用的平均响应时间作为参考来设置 `MaxGCPauseMillis`，JVM 会尽量去满足该条件，可能是 90% 的请求或者更多的响应时间在这之内， 但是并不代表是所有的请求都能满足，平均响应时间设置过小会导致频繁 GC。

## 参数列表

- -XX:+UseG1GC   使用垃圾优先(G1,Garbage First)收集器 
- -XX:MaxGCPauseMillis=n  设置垃圾收集暂停时间最大值指标。这是一个软目标，Java虚拟机将尽最大努力实现它 
- -XX:InitiatingHeapOccupancyPercent=n  触发并发垃圾收集周期的整个堆空间的占用比例。它被垃圾收集使用，用来触发并发垃圾收集周期，基于整个堆的占用情况，不只是一个代上(比如：G1)。0值 表示’do constant GC cycles’。默认是45 
- -XX:NewRatio=n 年轻代与年老代的大小比例，默认值是2 
- -XX:SurvivorRatio=n eden与survivor空间的大小比例，默认值8 
- -XX:MaxTenuringThreshold=n  最大晋升阈值，默认值15
- -XX:ParallerGCThreads=n  设置垃圾收集器并行阶段的线程数量。默认值根据Java虚拟机运行的平台有所变化
- -XX:ConcGCThreads=n  并发垃圾收集器使用的线程数量，默认值根据Java虚拟机运行的平台有所变化
- -XX:G1ReservePercent=n  为了降低晋升失败机率设置一个假的堆的储备空间的上限大小，默认值是10
- -XX:G1HeapRegionSize=n  使用G1收集器，Java堆被细分成一致大小的区域。这设置个体的细分的大小。这个参数的默认值由工学意义上的基于堆的大小决定


# 参考资料

https://www.cnblogs.com/cxxjohnson/p/8625713.html

https://mp.weixin.qq.com/s/gddff77gPdi5s2Hc9HBtcg

https://mp.weixin.qq.com/s/ezmD1XXgPoVoCo0nkKavdg

https://plumbr.io/handbook/garbage-collection-algorithms-implementations

https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html

https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector.htm

https://mp.weixin.qq.com/s/Q2FMEf7gQysGaqNHYvMJ-A