---
layout: post
title: Java GC日志
date: 2020-01-16
categories:
    - JVM
comments: true
permalink: java-gc-log.html
---



可以通过在java命令种加入参数来指定对应的gc类型，打印gc日志信息并输出至文件等策略。

- -XX:+PrintGC 输出GC日志
- -XX:+PrintGCDetails 输出GC的详细日志
- -XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）
- -XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
- -XX:+PrintHeapAtGC 在进行GC的前后打印出堆的信息
- -Xloggc:../logs/gc.log 日志文件的输出路径，**目录要在启动前建好**，可以给GC日志的文件后缀加上时间戳`gc-%t.log`，当JVM重启以后会生成新的日志文件，新的日志也不会覆盖老的日志
- -XX:+UseGCLogFileRotation 开启日志循环，默认的文件数目是0（意味着不作任何限制），默认的日志文件大小是0（同样也是不作任何限制）
- -XX:NumberOfGCLogFiles=N 限制日志文件数量
- -XX:GCLogFileSize=20M 现在日志文件大小， JVM的一个日志文件达到了最大值后以后，就会写入另一个新的文件，：gc.log.0、gc.log.1、...、gc.log.N.
- -XX:+PrintReferenceGC 打印各种引用的数量和清理所用时长
- -XX:+PrintGCApplicationStoppedTime 打印垃圾回收期间程序暂停的时间
- -XX:+PrintGCApplicationConcurrentTime 打印每次垃圾回收前,程序未中断的执行时间.
- -XX:+PrintTenuringDistribution 查看每次Young GC后新的存活周期的阈值
- -XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1这两个参数用于记录STW发生的原因、线程情况、STW各个阶段的停顿时间等。
- -XX:+HeapDumpOnOutOfMemoryError 当JVM发生OOM时，自动生成DUMP文件
- -XX:HeapDumpPath:/dumps/ 生成DUMP文件的路径，也可以指定文件名称，如果不指定文件名，默认为：`java_<pid>_<date>_<time>_heapDump.hprof`。



> 通过`-XX:+LogVMOutput -XX:LogFile=vm.log`可以打印所有的虚拟机日志



下面用一段测试代码来测试GC日志，堆内存100m

```java
public class GcTest {

  private static class F {
    // 1M
    private byte[] array = new byte[1024*1024];
  }

  public static void main(String[] args) {
    int i = 0;
    while (true) {
      F f = new F();
      i ++;
      if (i % 1000 == 0) {
        System.gc();
      }
    }
  }
}
```


# -XX:+PrintGC或者-verbose:gc

如果只设置`-XX:+PrintGC`那么打印的日志如下所示：

```
[GC (Allocation Failure)  25175K->1272K(98304K), 0.0014574 secs]
[GC (Allocation Failure)  26699K->2264K(98304K), 0.0022805 secs]
[GC (Allocation Failure)  27368K->2256K(98304K), 0.0020851 secs]
[GC (Allocation Failure)  27382K->2296K(98304K), 0.0019382 secs]
[GC (System.gc())  22939K->3296K(98304K), 0.0020527 secs]
[Full GC (System.gc())  3296K->3204K(98304K), 0.0117657 secs]
[GC (Allocation Failure)  28114K->3340K(97280K), 0.0004554 secs]
[GC (Allocation Failure)  28245K->3284K(93696K), 0.0082109 secs]
[GC (Allocation Failure)  27112K->3324K(96256K), 0.0004757 secs]
[GC (Heap Dump Initiated GC)  7989K->4356K(93696K), 0.0006934 secs]
[Full GC (Heap Dump Initiated GC)  4356K->3209K(53760K), 0.0085079 secs]
```

1. GC 表示是一次YGC（Young GC），Full GC很直观了，就是一次Full GC
2. Allocation Failure 表示是GC的原因，可以看到我们手动触发的显示为`System.gc()`
3. 25175K->1272K 表示年轻代从25175K降为1272K，如果是FullGC，则是整个堆的使用大小（没有验证）
4. 98304K表示整个堆的大小
5. 0.2483711 secs表示这次GC总计所用的时间

执行dump后输入，GC的原因就是`Heap Dump Initiated GC`

```
[GC (Heap Dump Initiated GC)  23473K->4304K(98304K), 0.0007550 secs]
[Full GC (Heap Dump Initiated GC)  4304K->3206K(58880K), 0.0132119 secs]
```

在JDK1.9之后`-XX:+PrintGC`已经废弃，可以使用`-verbose:gc`

# -XX:+PrintGCDetails

```
[GC (Allocation Failure) [PSYoungGen: 24611K->872K(29696K)] 24611K->880K(98304K), 0.0008301 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 25944K->792K(29696K)] 25952K->800K(98304K), 0.0006200 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 26373K->776K(29696K)] 26381K->784K(98304K), 0.0005443 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 25844K->776K(29696K)] 25852K->784K(98304K), 0.0005754 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (System.gc()) [PSYoungGen: 8439K->1752K(29696K)] 8447K->1760K(98304K), 0.0008358 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 1752K->0K(29696K)] [ParOldGen: 8K->1715K(68608K)] 1760K->1715K(98304K), [Metaspace: 3470K->3470K(1056768K)], 0.0042107 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```
比`-XX:+PrintGC`多了一下更详细的信息

1. `[PSYoungGen: 24611K->872K(29696K)]`表示年轻代的大小为29696K，YoungGC后从24611K降为29696K，PS是Parallel Scavenge收集器的缩写
2. `4611K->880K(98304K)`表示整个堆从2611K降为880K，堆的大小为98304K
3. `0.0008301 secs` 表示本次GC所花的时间
4. `[Times: user=0.00 sys=0.00, real=0.00 secs] ` 分别表示，用户态占用时长，内核用时，真实用时。
5. `[ParOldGen: 8K->1715K(68608K)]`表示老年代
6. `[Metaspace: 3470K->3470K(1056768K)]`表示元空间

# -XX:+PrintGCTimeStamps
要和`-XX:+PrintGC`或者` -XX:+PrintGCDetails`配合使用，比`-XX:+PrintGC`最前面多了一个时间戳：6.456， 表示从JVM启动到打印GC时刻用了6.456秒。

```
6.456: [GC (Allocation Failure)  24611K->848K(98304K), 0.0007839 secs]
```

```
6.443: [GC (Allocation Failure) [PSYoungGen: 24611K->776K(29696K)] 24611K->784K(98304K), 0.0010357 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

# -XX:+PrintGCDateStamps
要和`-XX:+PrintGC`或者` -XX:+PrintGCDetails`配合使用，比`-XX:+PrintGC`最前面最前面多了一个日期时间，表示打印GC的时刻的时间

```
2020-01-16T12:25:39.611+0800: [GC (Allocation Failure)  24611K->848K(98304K), 0.0007788 secs]
```

```
2020-01-16T12:27:51.966+0800: [GC (Allocation Failure) [PSYoungGen: 24611K->808K(29696K)] 24611K->816K(98304K), 0.0008092 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

也可以和`-XX:+PrintGCTimeStamps`一起使用

```
2020-01-16T12:27:13.229+0800: 6.446: [GC (Allocation Failure)  24611K->848K(98304K), 0.0009138 secs]
```

# -XX:+PrintHeapAtGC

```
{Heap before GC invocations=1 (full 0):
 PSYoungGen      total 29696K, used 24611K [0x00000000fdf00000, 0x0000000100000000, 0x0000000100000000)
  eden space 25600K, 96% used [0x00000000fdf00000,0x00000000ff708ef0,0x00000000ff800000)
  from space 4096K, 0% used [0x00000000ffc00000,0x00000000ffc00000,0x0000000100000000)
  to   space 4096K, 0% used [0x00000000ff800000,0x00000000ff800000,0x00000000ffc00000)
 ParOldGen       total 68608K, used 0K [0x00000000f9c00000, 0x00000000fdf00000, 0x00000000fdf00000)
  object space 68608K, 0% used [0x00000000f9c00000,0x00000000f9c00000,0x00000000fdf00000)
 Metaspace       used 3470K, capacity 4500K, committed 4864K, reserved 1056768K
  class space    used 381K, capacity 388K, committed 512K, reserved 1048576K
Heap after GC invocations=1 (full 0):
 PSYoungGen      total 29696K, used 840K [0x00000000fdf00000, 0x0000000100000000, 0x0000000100000000)
  eden space 25600K, 0% used [0x00000000fdf00000,0x00000000fdf00000,0x00000000ff800000)
  from space 4096K, 20% used [0x00000000ff800000,0x00000000ff8d2020,0x00000000ffc00000)
  to   space 4096K, 0% used [0x00000000ffc00000,0x00000000ffc00000,0x0000000100000000)
 ParOldGen       total 68608K, used 8K [0x00000000f9c00000, 0x00000000fdf00000, 0x00000000fdf00000)
  object space 68608K, 0% used [0x00000000f9c00000,0x00000000f9c02000,0x00000000fdf00000)
 Metaspace       used 3470K, capacity 4500K, committed 4864K, reserved 1056768K
  class space    used 381K, capacity 388K, committed 512K, reserved 1048576K
}
```
invocations 表示GC的次数，每次GC增加一次，每次GC前后的invocations相等

1. Heap before GC invocations=1 表示是第1次GC调用之前的堆内存状况

2.  (full 0)标识full gc的次数

#  -Xloggc:gc.log
使用整个参数默认会开启`-XX:+PrintGC -XX:+PrintGCTimeStamps `

在gc.log最前面还会默认打印系统的JDK版本与启动的参数

```
Java HotSpot(TM) 64-Bit Server VM (25.181-b13) for windows-amd64 JRE (1.8.0_181-b13), built on Jul  7 2018 04:01:33 by "java_re" with MS VC++ 10.0 (VS2010)
Memory: 4k page, physical 16646108k(8532336k free), swap 33290316k(24222972k free)
CommandLine flags: -XX:InitialHeapSize=104857600 -XX:MaxHeapSize=104857600 -XX:+PrintGC -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC 
```

# -XX:+PrintReferenceGC

打印出SoftReference，WeakReference，FinalReference，PhantomReference，JNI Weak Reference几种引用的数量，以及清理所用的时长，该参数在进行YGC调优时可以排上用场。

```
[GC (Allocation Failure) [SoftReference, 0 refs, 0.0000109 secs][WeakReference, 12 refs, 0.0000045 secs][FinalReference, 108 refs, 0.0000359 secs][PhantomReference, 0 refs, 0 refs, 0.0000048 secs][JNI Weak Reference, 0.0000038 secs][PSYoungGen: 24611K->840K(29696K)] 24611K->848K(98304K), 0.0008692 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

# -XX:+PrintGCApplicationStoppedTime

打印程序暂停的时间，下面的例子标识程序暂停了0.0000555秒，其中有0.0000241秒在等待所有的应用线程都到达安全点：不止GC引起的

```
Total time for which application threads were stopped: 0.0000555 seconds, Stopping threads took: 0.0000241 seconds
```

# -XX:+PrintGCApplicationConcurrentTime
打印程序未中断的执行时间，即持续运行时间.

```
Application time: 0.8746348 seconds
```

# -XX:+PrintTenuringDistribution
查看每次Young GC后新的存活周期的阈值

```
[GC (Allocation Failure) 
Desired survivor size 4194304 bytes, new threshold 7 (max 15)
[PSYoungGen: 24611K->840K(29696K)] 24611K->848K(98304K), 0.0008099 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

new threshold 7即标识新的存活周期的阈值为7。

# -XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1
这两个参数用于记录STW发生的原因、线程情况、STW各个阶段的停顿时间等。

```
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
1.025: no vm operation                  [      10          0              0    ]      [     0     0     0     0     0    ]  0   
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
4.258: EnableBiasedLocking              [      10          0              0    ]      [     0     0     0     0     0    ]  0   
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
6.455: ParallelGCFailedAllocation       [      10          0              0    ]      [     0     0     0     0     0    ]  0   
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
13.656: ParallelGCFailedAllocation       [      10          0              0    ]      [     0     0     0     0     0    ]  0   
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
20.858: ParallelGCFailedAllocation       [      10          0              0    ]      [     0     0     0     0     0    ]  0   
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
28.058: ParallelGCFailedAllocation       [      10          0              0    ]      [     0     0     0     0     0    ]  0   
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
30.158: ParallelGCSystemGC               [      10          0              0    ]      [     0     0     0     0     4    ]  0   
```

- vmop：引发STW的原因，以及触发时间
  - ”no vm operation”，就说明这是一个”保证安全点”。JVM默认每秒会触发一次安全点来处理那些非紧急的排队的操作，GuaranteedSafepointInterval选项可以用来调整这一行为（设置为0的话就会禁用该功能）
- total ：安全点里的总线程数。
- initially_running ：安全点时开始时正在运行状态的线程数，这项是Spin阶段的 时间来源
- wait_to_block ： 在VM Operation开始前需要等待其暂停的线程数，这项是block阶段的时间来源
- spin 等待线程响应safepoint号召的时间
- block 暂停所有线程所用的时间
- sync 等于 spin+block，这是从开始到进入安全点所耗的时间，可用于判断进入安全点耗时
- cleanup 清理所用时间
- vmop  真正执行VM Operation的时间

> safepoint的执行一共可以分为四个阶段：
>
> - Spin阶段。因为jvm在决定进入全局safepoint的时候，有的线程在安全点上，而有的线程不在安全点上，这个阶段是等待未在安全点上的用户线程进入安全点。
> - Block阶段。即使进入safepoint，用户线程这时候仍然是running状态，保证用户不在继续执行，需要将用户线程阻塞。
> - Cleanup。这个阶段是JVM做的一些内部的清理工作。
> - VM Operation. JVM执行的一些全局性工作，例如GC,代码反优化。
>
>   只要分析这个四个阶段，就能知道什么原因导致的STW时间过长。

> JVM STW 很多情况都是因为要GC，但在JVM里需要STW的情况有下面几种
>
> - 垃圾回收(这是最常见的场景)
> - 取消偏向锁(JVM会使用偏向锁来优化锁的获取过程)
> - Class重定义(比如常见的hotswap和instrumentation)
> - Code Cache Flushing(JDK1.8在CodeCache满的情况下就可能出现)
> - 线程堆栈转储(jstack命令)
>
> 安全点的相关知识查看 https://edgar615.github.io/java-gc-root.html



不同的垃圾收集器打印的日志会有些不同

## Serial垃圾收集器

`-XX:+UseSerialGC -XX:+PrintGCDetails`输入如下

```
[GC (Allocation Failure) [DefNew: 26834K->743K(30720K), 0.0016810 secs] 26834K->743K(99008K), 0.0017227 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [DefNew: 27898K->693K(30720K), 0.0014417 secs] 27898K->693K(99008K), 0.0014732 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [DefNew: 27366K->692K(30720K), 0.0008160 secs] 27366K->692K(99008K), 0.0008400 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [Tenured: 0K->1715K(68288K), 0.0025560 secs] 27842K->1715K(99008K), [Metaspace: 3470K->3470K(1056768K)], 0.0025858 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

- DefNew 表示这次GC发生在年轻代
- Tenured 表示这次GC发生在老年代

## ParNew垃圾收集器

`-XX:+UseParNewGC -XX:+PrintGCDetails`输入如下

```
[GC (Allocation Failure) [ParNew: 26834K->760K(30720K), 0.0072134 secs] 26834K->760K(99008K), 0.0072548 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
[GC (Allocation Failure) [ParNew: 27914K->965K(30720K), 0.0005350 secs] 27914K->965K(99008K), 0.0005594 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [ParNew: 27638K->986K(30720K), 0.0005401 secs] 27638K->986K(99008K), 0.0005635 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [Tenured: 0K->1715K(68288K), 0.0029441 secs] 28136K->1715K(99008K), [Metaspace: 3471K->3471K(1056768K)], 0.0029822 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
```

- ParNew 表示这次GC发生在年轻代
- Tenured 表示这次GC发生在老年代

## Parallel Old收集器

`-XX:+UseParallelOldGC -XX:+PrintGCDetails`输出如下

```
[GC (Allocation Failure) [PSYoungGen: 25796K->776K(29696K)] 25804K->784K(98304K), 0.0005773 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (System.gc()) [PSYoungGen: 8439K->1784K(29696K)] 8447K->1792K(98304K), 0.0009103 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 1784K->0K(29696K)] [ParOldGen: 8K->1715K(68608K)] 1792K->1715K(98304K), [Metaspace: 3471K->3471K(1056768K)], 0.0042623 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

- PSYoungGen表示这次GC发生在年轻代 PS是Parallel Scavenge收集器的缩写
- ParOldGen 表示这次GC发生在老年代

## CMS
`-XX:+UseConcMarkSweepGC -XX:+PrintGCDetails`输出如下

```
2020-01-16T15:51:35.957+0800: 1.945: [GC (Allocation Failure) 2020-01-16T15:51:35.957+0800: 1.945: [ParNew: 272640K->29126K(306688K), 0.0420362 secs] 272640K->29126K(1014528K), 0.0421878 secs] [Times: user=0.12 sys=0.01, real=0.04 secs] 
2020-01-16T15:51:36.009+0800: 1.997: [GC (CMS Initial Mark) [1 CMS-initial-mark: 0K(707840K)] 33761K(1014528K), 0.0061043 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
2020-01-16T15:51:36.015+0800: 2.003: [CMS-concurrent-mark-start]
2020-01-16T15:51:36.020+0800: 2.009: [CMS-concurrent-mark: 0.005/0.005 secs] [Times: user=0.03 sys=0.00, real=0.00 secs] 
2020-01-16T15:51:36.020+0800: 2.009: [CMS-concurrent-preclean-start]
2020-01-16T15:51:36.022+0800: 2.011: [CMS-concurrent-preclean: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-01-16T15:51:36.022+0800: 2.011: [CMS-concurrent-abortable-preclean-start]
2020-01-16T15:51:36.860+0800: 2.848: [CMS-concurrent-abortable-preclean: 0.537/0.838 secs] [Times: user=3.01 sys=0.05, real=0.84 secs] 
2020-01-16T15:51:36.860+0800: 2.849: [GC (CMS Final Remark) [YG occupancy: 181798 K (306688 K)]2020-01-16T15:51:36.860+0800: 2.849: [Rescan (parallel) , 0.0533260 secs]2020-01-16T15:51:36.914+0800: 2.902: [weak refs processing, 0.0000396 secs]2020-01-16T15:51:36.914+0800: 2.902: [class unloading, 0.0073531 secs]2020-01-16T15:51:36.921+0800: 2.909: [scrub symbol table, 0.0061381 secs]2020-01-16T15:51:36.927+0800: 2.916: [scrub string table, 0.0008029 secs][1 CMS-remark: 0K(707840K)] 181798K(1014528K), 0.0693454 secs] [Times: user=0.25 sys=0.00, real=0.07 secs] 
2020-01-16T15:51:36.930+0800: 2.918: [CMS-concurrent-sweep-start]
2020-01-16T15:51:36.930+0800: 2.918: [CMS-concurrent-sweep: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-01-16T15:51:36.930+0800: 2.918: [CMS-concurrent-reset-start]
2020-01-16T15:51:36.941+0800: 2.929: [CMS-concurrent-reset: 0.011/0.011 secs] [Times: user=0.04 sys=0.01, real=0.01 secs] 
```

### 阶段 1：初始标记（Initial Mark）

```
2020-01-16T15:51:36.009+0800: 1.997: [GC (CMS Initial Mark) [1 CMS-initial-mark: 0K(707840K)] 33761K(1014528K), 0.0061043 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
```

- `CMS Initial Mark` 初始标记阶段
- `0K`：当前老年代使用的容量，这里是 0G；
- `(707840K)`：老年代可用的最大容量
- `33761K`：整个堆目前使用的容量；
- `(1014528K)`：堆可用的容量；
- `0061043 secs`：这个阶段的持续时间；
- `[Times: user=0.02 sys=0.00, real=0.01 secs]`：相应 user、system and real 的时间统计

> **这个阶段会触发STW 事件**

### 阶段 2：并发标记（Concurrent Mark）

```
2020-01-16T15:51:36.015+0800: 2.003: [CMS-concurrent-mark-start]
2020-01-16T15:51:36.020+0800: 2.009: [CMS-concurrent-mark: 0.005/0.005 secs] [Times: user=0.03 sys=0.00, real=0.00 secs]
```

- `CMS-concurrent-mark-start`：并发收集开始
- `CMS-concurrent-mark`：并发收集阶段，这个阶段会遍历老年代，并标记所有存活的对象；
0.005/0.005 secs：这个阶段的持续时间与时钟时间；
- `[Times: user=1.01 sys=0.21, real=0.14 secs]`：如前面所示，但是这部的时间，其实意义不大，因为它是从并发标记的开始时间开始计算，这期间因为是并发进行，不仅仅包含 GC 线程的工作

### 阶段 3：并发预清理（Concurrent Preclean）

```
2020-01-16T15:51:36.020+0800: 2.009: [CMS-concurrent-preclean-start]
2020-01-16T15:51:36.022+0800: 2.011: [CMS-concurrent-preclean: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

- `CMS-concurrent-preclean-start`：并发预清理开始
- `CMS-concurrent-mark`：并发预清理阶段，这个阶段会将发生改变的区域标记为“脏区”（Dirty Card）；
0.002/0.002 secs：这个阶段的持续时间与时钟时间；

### 阶段 4：并发可取消的预清理（Concurrent Abortable Preclean）

```
2020-01-16T15:51:36.022+0800: 2.011: [CMS-concurrent-abortable-preclean-start]
2020-01-16T15:51:36.860+0800: 2.848: [CMS-concurrent-abortable-preclean: 0.537/0.838 secs] [Times: user=3.01 sys=0.05, real=0.84 secs] 
```

### 阶段 5：最终标记（Final Remark）

```
2020-01-16T15:51:36.860+0800: 2.849: [GC (CMS Final Remark) [YG occupancy: 181798 K (306688 K)]2020-01-16T15:51:36.860+0800: 2.849: [Rescan (parallel) , 0.0533260 secs]2020-01-16T15:51:36.914+0800: 2.902: [weak refs processing, 0.0000396 secs]2020-01-16T15:51:36.914+0800: 2.902: [class unloading, 0.0073531 secs]2020-01-16T15:51:36.921+0800: 2.909: [scrub symbol table, 0.0061381 secs]2020-01-16T15:51:36.927+0800: 2.916: [scrub string table, 0.0008029 secs][1 CMS-remark: 0K(707840K)] 181798K(1014528K), 0.0693454 secs] [Times: user=0.25 sys=0.00, real=0.07 secs] 
```

- `YG occupancy: 181798 K (306688 K)]`：年轻代当前占用量及容量
- `[Rescan (parallel) , 0.0533260 secs]`：这个 Rescan 是当应用暂停的情况下完成对所有存活对象的标记，这个阶段是并行处理的，这里花费了 0.0533260s；
- `[weak refs processing, 0.0000396 secs]`：第一个子阶段，它的工作是处理弱引用；
- `[class unloading, 0.0073531 secs]`：第二个子阶段，它的工作是：unloading the unused classes；
- `[scrub symbol table, 0.0061381 secs] ... [scrub string table, 0.0008029 secs]` 最后一个子阶段，它的目的是：cleaning up symbol and string tables which hold class-level metadata and internalized string respectively，时钟的暂停也包含在这里；
- `[1 CMS-remark: 0K(707840K)] 181798K(1014528K), 0.0693454 secs]`：
`0K(707840K)`：这个阶段之后，老年代的使用量与总量；`181798K(1014528K)`：这个阶段之后，堆的使用量与总量；0.0693454 secs：这个阶段的持续时间；

经历过这五个阶段之后，老年代所有存活的对象都被标记过了，现在可以通过清除算法去清理那些老年代不再使用的对象。

> **这个阶段会触发STW 事件**

### 阶段 6：并发清除（Concurrent Sweep）

```
2020-01-16T15:51:36.930+0800: 2.918: [CMS-concurrent-sweep-start]
2020-01-16T15:51:36.930+0800: 2.918: [CMS-concurrent-sweep: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

### 阶段 7：并发重置（Concurrent Reset）


根据标记结果清除垃圾对象。

```
2020-01-16T15:51:36.930+0800: 2.918: [CMS-concurrent-reset-start]
2020-01-16T15:51:36.941+0800: 2.929: [CMS-concurrent-reset: 0.011/0.011 secs] [Times: user=0.04 sys=0.01, real=0.01 secs] 
```

重置 CMS 算法相关的内部数据, 为下一次 GC 循环做准备

# G1

> 核心内容来自https://mp.weixin.qq.com/s/-9Phd8PPAUKCFe2HBVknUg

## YoungGC阶段
```
2020-01-16T16:37:26.617+0800: 0.568: [GC pause (G1 Evacuation Pause) (young), 0.0059354 secs]
   [Parallel Time: 4.0 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 568.3, Avg: 568.5, Max: 569.1, Diff: 0.8]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.4, Max: 0.8, Diff: 0.8, Sum: 1.7]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.9, Max: 3.2, Diff: 3.2, Sum: 3.7]
      [Object Copy (ms): Min: 0.2, Avg: 2.2, Max: 3.1, Diff: 2.8, Sum: 8.7]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.6]
         [Termination Attempts: Min: 1, Avg: 2.0, Max: 4, Diff: 3, Sum: 8]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 3.1, Avg: 3.7, Max: 3.9, Diff: 0.8, Sum: 14.8]
      [GC Worker End (ms): Min: 572.2, Avg: 572.2, Max: 572.2, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 1.8 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 1.4 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.1 ms]
   [Eden: 51.0M(51.0M)->0.0B(52.0M) Survivors: 0.0B->5120.0K Heap: 51.0M(1024.0M)->4489.7K(1024.0M)]
 [Times: user=0.01 sys=0.00, real=0.00 secs] 
```

这段日志是典型的Evacuation阶段日志（GC pause (G1 Evacuation Pause) (young)）。Evacution这个词在G1中出现的频率非常高，中文意思是疏散，在G1中可以理解为拷贝&清理，就是把存活的对象拷贝到1个或者多个Region中，目标Region可能只是Young区，也可能是Young区+Old区，取决于这次YGC是否有符合晋升到Old区的对象。

由这段GC日志我们可知，整个YGC由多个子任务以及嵌套子任务组成，且一些核心任务为：Root Scanning，Update/Scan RS，Object Copy，CleanCT，Choose CSet，Ref Proc，Humongous Reclaim，Free CSet。

```
2020-01-16T16:37:26.617+0800: 0.568: [GC pause (G1 Evacuation Pause) (young), 0.0059354 secs]
```
这次YGC是在JVM启动后0.568秒的时候发生的，并且整个过程耗时0.0059354秒

### Parallel Time

```
[Parallel Time: 0.7 ms, GC Workers: 4]
```

本次YGC总计由4个GC线程并行收集，并总计消耗时间0.7毫秒：

```
[GC Worker Start (ms): Min: 568.3, Avg: 568.5, Max: 569.1, Diff: 0.8]
```
几个GC线程开始的最小时间、平均时间和最大时间（这个时间是相对于JVM启动后到现在的毫秒数），最后的Diff:0.5表示几个GC线程最大启动时间差有0.8毫秒：

```
[Ext Root Scanning (ms): Min: 0.0, Avg: 0.4, Max: 0.8, Diff: 0.8, Sum: 1.7]
```

几个GC线程ROOT扫描阶段消耗的时间统计信息，从这行日志可知，ROOT扫描平均耗时0.4毫秒：

```
[Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0]
```

更新RSet消耗的时间统计信息，RSet即Remember Sets，它是用来记录引用指向一个Region的跟踪信息的数据结构。我们看到后面还有一段Processed Buffers的日志，Mutator线程会记录对象图的变化，JVM将这些变化的track信息记录在被称为Update Buffers中。这个Update RS的子任务Processed Buffers就是负责处理这个Update Buffers的，从而将所有Region的RSets更新到一致的状态

如下图所示，我们假设O1，即紫色块就是某一个Old类型的Region，而绿色块就是某一个Eden类型的Region：

![](/assets/images/posts/gc-log/gc-log-1.png)

如果程序中执行了O1.attr=E1，即O1有对E1的引用，那么Card Table就会记录下来。而Remember Sets包含了对Card Table中这个Card的引用：

![](/assets/images/posts/gc-log/gc-log-2.png)


```
[Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
```

Scan RS阶段：这个阶段会扫描遍历Remember Sets。由上图可知，一个Region的RSet包含了指向这个Region的引用的Cards，这个阶段就是扫描RSet中这些Cards，从而找出任何有指向CSet中所有Region的引用。通过这一步就能知道，Eden区哪些对象被老年代引用，从而不会在GC时回收掉

```
[Object Copy (ms): Min: 0.2, Avg: 2.2, Max: 3.1, Diff: 2.8, Sum: 8.7]
```

对象拷贝阶段：这个阶段会将前面扫描到的存活的对象拷贝到目标Region中，可能是Survivor类型Region，也可能是Old类型Region（如果达到晋升条件的话），并记录拷贝过程消耗的时间统计信息：

![](/assets/images/posts/gc-log/gc-log-3.png)

![](/assets/images/posts/gc-log/gc-log-4.png)

```
[GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
[GC Worker Total (ms): Min: 3.1, Avg: 3.7, Max: 3.9, Diff: 0.8, Sum: 14.8]
[GC Worker End (ms): Min: 572.2, Avg: 572.2, Max: 572.2, Diff: 0.0]
```

Parallel阶段最后几个子任务，GC Worker Total表示GC线程整个生命周期总计消耗的时间，而GC Worker End表示GC线程完成任务停止的时间（这个时间是相对于JVM进程启动后的毫秒数）

### Clear CT

CT就是Card Table，即清理Card Table消耗的时间。这个阶段对应的GC日志如下所示：

```
[Clear CT: 0.1 ms]
```

###     Other

最后一个阶段Other，这个阶段也包含了比较多的子任务

```
[Other: 1.8 ms]
```

Choose Cset：就是选择哪些Region作为CSet一部分需要的时间，G1会根据用户设置的停顿时间来决定Region的数量：

```
[Choose CSet: 0.0 ms]
```

Ref Proc：即处理引用对象消耗的时间，可能是Weak、Soft、Phantom引用，强烈建议配置参数：`-XX:+ParallelRefProcEnabled`（这个参数默认是关闭的），即让这个阶段并行处理：

```
[Ref Proc: 1.4 ms]
```

超大（humongous）对象的处理：YGC阶段如果发现某些H区域的对象都不是存活对象，就会回收掉这些对象占用的空间：

```
[Humongous Register: 0.0 ms]
[Humongous Reclaim: 0.0 ms]
```

Free CSet：即释放掉CSet中Region占用的内存空间所消耗的时间（前面的Object Copy已经将CSet中存活的对象拷贝到了目标Region中，所以这里需要回收Region占用的内存空间）：

```
[Free CSet: 0.1 ms]
```

### 内存情况

```
[Eden: 51.0M(51.0M)->0.0B(52.0M) Survivors: 0.0B->5120.0K Heap: 51.0M(1024.0M)->4489.7K(1024.0M)]
```
`Eden: 51.0M(51.0M)->0.0B(52.0M)` 表示整个Eden区从占用51.0M到回收后的0.0B，并且GC前后 整个Eden区大小增大了1M

`Survivors: 0.0B->5120.0K` 标识回收期0.0B，回收后5120.0K

`Heap: 51.0M(1024.0M)->4489.7K(1024.0M)`表示整个堆回收前占用51.0M，回收后占用4489.7K，并且整个堆大小是1024.0M

## 并发标记周期

> 下面都是抄的

上面提分析的日志全部是YGC阶段的日志，G1模式下还有一个重要的周期，即全局并发标记周期，其GC日志如下所示：

```
# 这一行日志是全局并发标记的第一个阶段，即初始化标记，是伴随YGC一起发生的，后面的857M->617M表示YGC发生前后堆内存变化，0.0112237表示YGC的耗时
[GC pause (G1 Evacuation Pause) (young) (initial-mark) 857M->617M(1024M), 0.0112237 secs]
# 开始并发ROOT区域扫描
[GC concurrent-root-region-scan-start]
# 结束并发ROOT区域扫描，并统计这个阶段的耗时
[GC concurrent-root-region-scan-end, 0.0000525 secs]
[GC concurrent-mark-start]
[GC concurrent-mark-end, 0.0083864 secs]
# 最终标记阶段完成并发标记阶段后遗留的工作，即SATB buffer处理，并统计这个阶段耗时
[GC remark, 0.0038066 secs]
# 清理阶段会根据所有Region标记信息，计算出每个Region存活对象信息，并且把Region根据GC回收效率排序
[GC cleanup 680M->680M(1024M), 0.0006165 secs]
```

## Mixed GC

Evacuation Pause除了是YGC触发之外，还可能是Mixed GC（[GC pause (G1 Evacuation Pause) (mixed)），日志如下所示，Mixed GC的整个子任务和YGC完全一样，只是回收的范围（CSet）不一样而已，YGC只回收Young区，而Mixed GC回收Young区+部分Old区：

```
29.268: [GC pause (G1 Evacuation Pause) (mixed), 0.0059011 secs]
   [Parallel Time: 5.6 ms, GC Workers: 4]
      ... ...
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.3 ms]
      ... ...
   [Eden: 14.0M(14.0M)->0.0B(156.0M) Survivors: 10.0M->4096.0K Heap: 165.9M(512.0M)->148.7M(512.0M)]
 [Times: user=0.02 sys=0.01, real=0.00 secs] 
```

## Full GC

最后一种垃圾收集方式即FullGC，事实上，在G1的正常工作流程中没有Full GC的概念，只有在G1搞不定的情况下（或者主动触发），才会发生的GC方式。日志如下，也非常容易理解，由第一行日志可知，这次的FullGC是由System.gc()触发的：

```
4.358: [Full GC (System.gc())  298M->509K(512M), 0.0101774 secs]
   [Eden: 122.0M(154.0M)->0.0B(230.0M) Survivors: 4096.0K->0.0B Heap: 298.8M(512.0M)->509.4K(512.0M)], [Metaspace: 3308K->3308K(1056768K)]
 [Times: user=0.01 sys=0.00, real=0.01 secs] 
```

下面这段日志是G1搞不定的情况下发触发的FullGC，从第二行日志可以看出，Mixed GC前后，堆大小都是99G，说明没有任何回收效果，而堆由已经满了，所以触发了Full GC：

```
805.815: [GC pause (G1 Evacuation Pause) (young) 96G->74G(100G), 2.4778659 secs] 813.964: [GC pause (G1 Evacuation Pause) (mixed)-- 97G->99G(100G), 23.7970094 secs] 
837.762: [GC pause (G1 Evacuation Pause) (mixed)-- 99G->99G(100G), 32.0781615 secs] 
869.842: [Full GC (Allocation Failure) 99G->62G(100G), 169.3897706 secs]
```

# 参考资料

https://www.jianshu.com/p/fd1d4f21733a

https://mp.weixin.qq.com/s/-9Phd8PPAUKCFe2HBVknUg