---
layout: post
title: synchronized原理
date: 2019-06-25
categories:
    - 多线程
comments: true
permalink: java-concurrency-synchronized.html
---

synchronized有以下三种使用方式:

- 同步普通方法，锁的是当前对象。
- 同步静态方法，锁的是当前 Class 对象。
- 同步块，锁的是()中的对象。

**实现原理：**

 `JVM` 是通过进入、退出对象监视器( `Monitor` )来实现对方法、同步块的同步的。

具体实现是在编译之后在同步方法调用前加入一个 `monitor.enter` 指令，在退出方法和异常处插入 `monitor.exit` 的指令。

其本质就是对一个对象监视器( `Monitor` )进行获取，而这个获取过程具有排他性从而达到了同一时刻只能一个线程访问的目的。

而对于没有获取到锁的线程将会阻塞到方法入口处，直到获取锁的线程 `monitor.exit` 之后才能尝试继续获取锁。

![](/assets/images/posts/synchronized/synchronized-1.png)

看一段代码

```
public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("synchronized");
        }
    }
}
```
执行`javap -c SynchronizedDemo.class`得到下面信息

```
Compiled from "SynchronizedDemo.java"
public class thread.start.SynchronizedDemo {
  public thread.start.SynchronizedDemo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void method();
    Code:
       0: aload_0
       1: dup
       2: astore_1
       3: monitorenter
       4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       7: ldc           #3                  // String Method 1 start
       9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      12: aload_1
      13: monitorexit
      14: goto          22
      17: astore_2
      18: aload_1
      19: monitorexit
      20: aload_2
      21: athrow
      22: return
    Exception table:
       from    to  target type
           4    14    17   any
          17    20    17   any
}

```
可以看到在同步块的入口和出口分别有`monitorenter`和`monitorexit`

> 字节码我还没学习过

对于下面的代码

```
public class SynchronizedDemo {

  public synchronized void methodB() {
    System.out.println("synchronized");
  }
}
```
执行`javap -c SynchronizedDemo.class`得到下面信息
```
Compiled from "SynchronizedDemo.java"
public class thread.start.SynchronizedDemo {
  public thread.start.SynchronizedDemo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public synchronized void methodB();
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #3                  // String synchronized
       5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```
发现没有`monitorenter`和`monitorexit`，因为这里monitorenter 和 monitorexit 操作所对应的锁对象是隐式的

# 重入机制

**monitorenter**

**每个对象都有一个监视器锁（monitor）,当monitor被占用时就会处于锁定状态。**线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

- 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者；
- 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1；
- 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权；

> monitorenter ：Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:
>
> • If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.
>
> • If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.
> 
> • If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.

**monitorexit**

**执行monitorexit的线程必须是objectref所对应的monitor的所有者。**指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。

> monitorexit：The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.
> 
> The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner. 
> 
> Other threads that are blocking to enter the monitor are allowed to attempt to do so.

synchronized 的语义底层是通过一个 Monitor 的对象来完成，任何对象都有一个monitor与之相关联，当且一个monitor被持有之后，他将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor所有权，即尝试获取对象的锁；

> wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因

## Monitor 的工作机理

![](/assets/images/posts/synchronized/synchronized-7.jpg)

- 线程进入同步方法中。
- 为了继续执行临界区代码，线程必须获取 Monitor 锁。如果获取锁成功，将成为该监视者对象的拥有者。任一时刻内，监视者对象只属于一个活动线程（The Owner）
- 拥有监视者对象的线程可以调用 wait() 进入等待集合（Wait Set），同时释放监视锁，进入等待状态。
- 其他线程调用 notify() / notifyAll() 接口唤醒等待集合中的线程，这些等待的线程需要重新获取监视锁后才能执行 wait() 之后的代码。
- 同步方法执行完毕了，线程退出临界区，并释放监视锁。

## Monitor
什么是Monitor？我们可以把它理解为一个同步工具，也可以描述为一种同步机制，它通常被描述为一个对象。 与一切皆对象一样，所有的Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，因为在Java的设计中 ，每一个Java对象都带了一把看不见的锁，它叫做内部锁或者Monitor锁。

Monitor 是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联（对象头的[MarkWord](https://edgar615.github.io/java-object-memory.html)中的LockWord指向monitor的起始地址），同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。其结构如下：

![](/assets/images/posts/synchronized/synchronized-4.jpg)

- Owner：初始时为NULL表示当前没有任何线程拥有该monitor record，当线程成功拥有该锁后保存线程唯一标识，当锁被释放时又设置为NULL；
- EntryQ: 关联一个系统互斥锁（semaphore），阻塞所有试图锁住monitor record失败的线程
- RcThis: 表示blocked或waiting在该monitor record上的所有线程的个数。
- Nest: 用来实现重入锁的计数。
- HashCode:保存从对象头拷贝过来的HashCode值（可能还包含GC age）。
- Candidate:用来避免不必要的阻塞或等待线程唤醒，因为每一次只有一个线程能够成功拥有锁，如果每次前一个释放锁的线程唤醒所有正在阻塞或等待的线程，会引起不必要的上下文切换（从阻塞到就绪然后因为竞争锁失败又被阻塞）从而导致性能严重下降。Candidate只有两种可能的值0表示没有需要唤醒的线程1表示要唤醒一个继任线程来竞争锁。

# 锁存放的位置

锁标记存放在Java对象头的Mark Word中。

> https://edgar615.github.io/java-object-memory.html

Mark Word在不同的锁状态下存储的内容不同，在32位JVM中是这么存的：

![](/assets/images/posts/object-memory/object-memory-4.png)

# Java 内部锁优化
JDK6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”。

锁主要存在四种状态，依次是：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁。但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级。

> 在 JDK 1.6 中默认是开启偏向锁和轻量级锁的，可以通过-XX:-UseBiasedLocking来禁用偏向锁。

## 自旋锁
线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁的阻塞和唤醒对CPU来说是一件负担很重的工作，势必会给系统的并发性能带来很大的压力。同时我们发现在许多应用上面，对象锁的锁状态只会持续很短一段时间，为了这一段很短的时间频繁地阻塞和唤醒线程是非常不值得的。所以引入自旋锁。

**自旋锁，就是指当一个线程尝试获取某个锁时，如果该锁已被其他线程占用，就一直循环检测锁是否被释放，而不是进入线程挂起或睡眠状态。**

自旋锁适用于锁保护的临界区很小的情况，临界区很小的话，锁占用的时间就很短。自旋等待不能替代阻塞，虽然它可以避免线程切换带来的开销，但是它占用了CPU处理器的时间。如果持有锁的线程很快就释放了锁，那么自旋的效率就非常好，反之，自旋的线程就会白白消耗掉处理的资源，它不会做任何有意义的工作所以说，自旋等待的时间（自旋的次数）必须要有一个限度，如果自旋超过了定义的时间仍然没有获取到锁，则应该被挂起。

自旋锁在JDK 1.4.2中引入，默认关闭，但是可以使用`-XX:+UseSpinning`开开启，在JDK1.6中默认开启。同时自旋的默认次数为10次，可以通过参数`-XX:PreBlockSpin`来调整。

如果通过参数`-XX:PreBlockSpin`来调整自旋锁的自旋次数，会带来诸多不便。假如将参数调整为10，但是系统很多线程都是等你刚刚退出的时候就释放了锁（假如多自旋一两次就可以获取锁），是不是很尴尬。于是JDK1.6引入自适应的自旋锁，让虚拟机会变得越来越聪明。

## 适应性自旋锁
JDK 1.6引入了更加聪明的自旋锁，即自适应自旋锁。所谓自适应就意味着自旋的次数不再是固定的，它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。

线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。反之，如果对于某个锁，很少有自旋能够成功，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。

## 重量级锁

重量级锁是 Java 虚拟机中最为基础的锁实现。在这种状态下，Java 虚拟机会阻塞加锁失败的线程，并且在目标锁被释放的时候，唤醒这些线程。在Linux中，这是通过pthread库的互斥锁来实现的。此外，这些操作将涉及系统调用，需要从操作系统的用户态切换至内核态，其开销非常之大。

为了尽量避免昂贵的线程阻塞、唤醒操作，Java虚拟机会在线程进入阻塞状态之前，以及被唤醒后竞争不到锁的情况下，进入自旋状态，在处理器上空跑并且轮询锁是否被释放。如果此时锁恰好被释放了，那么当前线程便无须进入阻塞状态，而是直接获得这把锁。

## 轻量锁

轻量锁是为了解决减少无实际竞争情况下，使用重量级锁产生的性能消耗

当代码进入同步块时，如果同步对象为无锁状态时，当前线程会在栈帧中创建一个锁记录(Lock Record)区域，同时将锁对象的对象头中 Mark Word 拷贝到锁记录中，再尝试使用 CAS 将 Mark Word 更新为指向锁记录的指针。

- 如果更新成功，当前线程就获得了锁。
- 如果更新失败 JVM 会先检查锁对象的 Mark Word 是否指向当前线程的锁记录。
- 如果是则说明当前线程拥有锁对象的锁，可以直接进入同步块。
- 不是则说明有其他线程抢占了锁，如果存在多个线程同时竞争一把锁，轻量锁就会膨胀为重量锁。

**解锁**

轻量锁的解锁过程也是利用 CAS 来实现的，会尝试锁记录替换回锁对象的 Mark Word 。如果替换成功则说明整个同步操作完成，失败则说明有其他线程尝试获取锁，这时就会唤醒被挂起的线程(此时已经膨胀为重量锁)
轻量锁能提升性能的原因是：认为大多数锁在整个同步周期都不存在竞争，所以使用 CAS 比使用互斥开销更少。但如果锁竞争激烈，轻量锁就不但有互斥的开销，还有 CAS 的开销，甚至比重量锁更慢。

## 偏向锁
为了进一步的降低获取锁的代价，JDK1.6 之后还引入了偏向锁。

偏向锁的特征是:锁不存在多线程竞争，并且应由一个线程多次获得锁。

“偏向”的意思是，偏向锁假定将来只有第一个申请锁的线程会使用锁（不会有任何线程再来申请锁）

当线程访问同步块时，会使用 CAS 将线程 ID 更新到锁对象的 Mark Word 中，如果更新成功则获得偏向锁，并且之后每次进入这个对象锁相关的同步块时都不需要再次获取锁了.否则，说明有其他线程竞争，膨胀为轻量级锁。

**释放锁**
当有另外一个线程获取这个锁时，持有偏向锁的线程就会释放锁，释放时会等待全局安全点(这一时刻没有字节码运行)，接着会暂停拥有偏向锁的线程，根据锁对象目前是否被锁来判定将对象头中的 Mark Word 设置为无锁或者是轻量锁状态。
轻量锁可以提高带有同步却没有竞争的程序性能，但如果程序中大多数锁都存在竞争时，那偏向锁就起不到太大作用。可以使用` -XX:-userBiasedLocking=false` 来关闭偏向锁，并默认进入轻量锁。

**偏向锁针对的是从始至终只有一个线程请求某一把锁。是轻量级锁的更进一步的乐观情况。**


JVM一般是这样使用锁和Mark Word的：

1，当没有被当成锁时，这就是一个普通的对象，Mark Word记录对象的HashCode，锁标志位是01，是否偏向锁那一位是0。

2，当对象被当做同步锁并有一个线程A抢到了锁时，锁标志位还是01，但是否偏向锁那一位改成1，前23bit记录抢到锁的线程id，表示进入偏向锁状态。

3，当线程A再次试图来获得锁时，JVM发现同步锁对象的标志位是01，是否偏向锁是1，也就是偏向状态，Mark Word中记录的线程id就是线程A自己的id，表示线程A已经获得了这个偏向锁，可以执行同步锁的代码。

4，当线程B试图获得这个锁时，JVM发现同步锁处于偏向状态，但是Mark Word中的线程id记录的不是B，那么线程B会先用CAS操作试图获得锁，这里的获得锁操作是有可能成功的，因为线程A一般不会自动释放偏向锁。如果抢锁成功，就把Mark Word里的线程id改为线程B的id，代表线程B获得了这个偏向锁，可以执行同步锁代码。如果抢锁失败，则继续执行步骤5。

5，偏向锁状态抢锁失败，代表当前锁有一定的竞争，偏向锁将升级为轻量级锁。JVM会在当前线程的线程栈中开辟一块单独的空间，里面保存指向对象锁Mark Word的指针，同时在对象锁Mark Word中保存指向这片空间的指针。上述两个保存操作都是CAS操作，如果保存成功，代表线程抢到了同步锁，就把Mark Word中的锁标志位改成00，可以执行同步锁代码。如果保存失败，表示抢锁失败，竞争太激烈，继续执行步骤6。

6，轻量级锁抢锁失败，JVM会使用自旋锁，自旋锁不是一个锁状态，只是代表不断的重试，尝试抢锁。从JDK1.7开始，自旋锁默认启用，自旋次数由JVM决定。如果抢锁成功则执行同步锁代码，如果失败则继续执行步骤7。

7，自旋锁重试之后如果抢锁依然失败，同步锁会升级至重量级锁，锁标志位改为10。在这个状态下，未抢到锁的线程都会被阻塞。

网上找到的从偏向锁膨胀至重量锁的完全流程图

![](/assets/images/posts/synchronized/synchronized-2.png)

另一个图

![](/assets/images/posts/synchronized/synchronized-3.jpg)

## 锁消除（Lock Elimination）
锁削除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行削除。

Java JIT 会通过逃逸分析的方式，去分析加锁的代码段/共享资源，他们是否被一个或者多个线程使用，或者等待被使用。如果通过分析证实，只被一个线程访问，在编译这个代码段的时候就不生成 Synchronized 关键字，仅仅生成代码对应的机器码。

换句话说，即便开发人员对代码段/共享资源加上了 Synchronized（锁），只要 JIT 发现这个代码段/共享资源只被一个线程访问，也会把这个 Synchronized（锁）去掉。从而避免竞态，提高访问资源的效率。

![](/assets/images/posts/synchronized/synchronized-8.jpg)

## 锁粗化（Lock Coarsening）
锁粗化是指减少不必要的紧连在一起的unlock，lock操作，将多个连续的锁扩展成一个范围更大的锁。

假设有几个在程序上相邻的同步块（代码段/共享资源）上，每个同步块使用的是同一个锁实例。那么 JIT 会在编译的时候将这些同步块合并成一个大同步块，并且使用同一个锁实例。这样避免一个线程反复申请/释放锁。

![](/assets/images/posts/synchronized/synchronized-9.jpg)

锁粗化默认是开启的。如果要关闭这个特性可以在 Java 程序的启动命令行中添加虚拟机参数`-XX:-EliminateLocks`。

# Java 代码中进行锁优化

锁的开销主要是在争用锁上，当多线程对共享资源进行访问时，会出现线程等待。即便是使用内存屏障，也会导致冲刷写缓冲器，清空无效化队列等开销。

为了降低这种开销，通常可以从几个方面入手，例如：减少线程申请锁的频率（减少临界区）和减少线程持有锁的时间长度（减小锁颗粒）以及多线程的设计模式。

## 减少临界区的范围

当共享资源需要被多线程访问时，会将共享资源或者代码段放到临界区中。

如果在代码书写中减少临界区的长度，就可以减少锁被持有的时间，从而降低锁被征用的概率，达到减少锁开销的目的。

![](/assets/images/posts/synchronized/synchronized-10.jpg)

如上图，尽量避免对一个方法进行加锁同步，可以只针对方法中的需要同步资源/变量进行同步。其他的代码段不放到 Synchronzied 中，减少临界区的范围。

## 减小锁的颗粒度
减小锁的颗粒度可以降低锁的申请频率，从而减小锁被争用的概率。其中一种常见的方法就是将一个颗粒度较粗的锁拆分成颗粒度较细的锁。

![](/assets/images/posts/synchronized/synchronized-11.jpg)

JDK 内置的 ConcurrentHashMap 与 SynchronizedMap 就使用了类似的设计。

## 读写锁
也叫做线程的读写模式（Read-Write Lock），其本质是一种多线程设计模式。

将读取操作和写入操作分开考虑，在执行读取操作之前，线程必须获取读取的锁。在执行写操作之前，必须获取写锁。当线程执行读取操作时，共享资源的状态不会发生变化，其他的线程也可以读取。但是在读取时，不可以写入。‘

其实，读写模式就是将原来共享资源的锁，转化成为读和写两把锁，将其分两种情况考虑。

如果都是读操作可以支持多线程同时进行，只有在写时其他线程才会进入等待。

![](/assets/images/posts/synchronized/synchronized-12.jpg)

Reader 线程正在读取，Writer 线程正在等待

![](/assets/images/posts/synchronized/synchronized-13.jpg)

Writer 线程正在写入，Reader 线程正在等待

# 参考资料

https://www.jianshu.com/p/2ba154f275ea

https://www.jianshu.com/p/e62fa839aa41

https://mp.weixin.qq.com/s/tV48ZCwZUAUO-1xL7uESUA
