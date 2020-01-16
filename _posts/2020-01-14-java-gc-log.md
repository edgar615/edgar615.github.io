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

1. `[PSYoungGen: 24611K->872K(29696K)]`表示年轻代的大小为29696K，YoungGC后从24611K降为29696K
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

打印垃圾回收期间程序暂停的时间

```
Total time for which application threads were stopped: 0.0000555 seconds, Stopping threads took: 0.0000241 seconds
```

# -XX:+PrintGCApplicationConcurrentTime
打印每次垃圾回收前,程序未中断的执行时间.

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
- total ：STW发生时，JVM存在的线程数目。
- initially_running ：STW发生时，仍在运行的线程数，这项是Spin阶段的 时间来源
- wait_to_block ： STW需要阻塞的线程数目，这项是block阶段的时间来源
- spin,block,cleanup,vmop的时间，sync=spin + block + 其他。

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
> - Garbage collection pauses
> - Code deoptimization
> - Flushing code cacheClass redefinition (e.g. hot swap or instrumentation)
> - Biased lock revocation
> - Various debug operation (e.g. deadlock check or stacktrace dump)

# 参考资料

https://www.jianshu.com/p/fd1d4f21733a