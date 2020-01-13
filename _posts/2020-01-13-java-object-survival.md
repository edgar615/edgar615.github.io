---
layout: post
title: 如何判断一个对象是否存活
date: 2020-01-13
categories:
    - jvm
comments: true
permalink: java-object-survival.html
---

> 本文文字图片基本来自参考资料

垃圾回收（Garbage Collection，GC），顾名思义就是释放垃圾占用的空间，防止内存泄露。既然我们要做垃圾回收，首先我们得搞清楚垃圾的定义是什么，哪些内存是需要回收的。

判断对象是否存活的算法有两种：

- 引用计数算法
- 可达性分析算法

# 引用计数算法

引用计数算法（Reachability Counting）是通过在对象头中分配一个空间来保存该对象被引用的次数（Reference Count）。如果该对象被其它对象引用，则它的引用计数加 1，如果删除对该对象的引用，那么它的引用计数就减 1，当该对象的引用计数为 0 时，那么该对象就会被回收。

每个计数器只记录了其对应对象的局部信息——被引用的次数，而没有（也不需要）一份全局的对象图的生死信息。由于只维护局部信息，所以不需要扫描全局对象图就可以识别并释放死对象；但也因为缺乏全局对象图信息，所以无法处理循环引用的状况。

先创建一个字符串`String m = new String("jack");`，这时候"jack"有一个引用，就是 m。

![](/assets/images/posts/java-object-survival/java-object-survival-1.jpg)

然后将 m 设置为 null`m = null;`，这时候"jack"的引用次数就等于 0 了，在引用计数算法中，意味着这块内容就需要被回收了。

![](/assets/images/posts/java-object-survival/java-object-survival-2.jpg)

引用计数算法是将垃圾回收分摊到整个应用程序的运行当中了，而不是在进行垃圾收集时，要挂起整个应用的运行，直到对堆中所有对象的处理都结束。因为每个对象在被引用次数为0的时候，是立即就可以知道的，而且对象的回收根本不需要另外的GC线程专门去做，业务线程自己就搞定了。因此，采用引用计数的垃圾收集不属于严格意义上的"Stop-The-World"的垃圾收集机制。

看似很美好，但我们知道 JVM 的垃圾回收就是"Stop-The-World"的，那是什么原因导致我们最终放弃了引用计数算法呢？

```java
public class ReferenceCountingGC {

    public Object instance;

    public ReferenceCountingGC(String name){}
}

public static void testGC(){

    ReferenceCountingGC a = new ReferenceCountingGC("objA");
    ReferenceCountingGC b = new ReferenceCountingGC("objB");

    a.instance = b;
    b.instance = a;

    a = null;
    b = null;
}
```

上面的例子，最后这 2 个对象已经不可能再被访问了，但由于他们相互引用着对方，导致它们的引用计数永远都不会为 0，通过引用计数算法，也就永远无法通知 GC 收集器回收它们。

![](/assets/images/posts/java-object-survival/java-object-survival-3.jpg)

# 可达性分析算法

可达性分析算法的基本思路是通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到GC Root没有任何引用链相连时，则证明此对象是不可用的。

![](/assets/images/posts/java-object-survival/java-object-survival-4.jpg)

通过可达性算法，成功解决了引用计数所无法解决的问题-“循环依赖”，只要你无法与 GC Root 建立直接或间接的连接，系统就会判定你为可回收对象。那这样就引申出了另一个问题，哪些属于 GC Root。

# GC Root
GC Root的对象是可以从堆外部访问的对象

> 摘自https://help.eclipse.org/2019-12/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Fconcepts%2Fgcroots.html
> 
> 是Memory Analyzer对dump分析得出的GC Root

- **System Class**   Class loaded by bootstrap/system class loader. For example, everything from the rt.jar like`java.util.*`、. 由系统类加载器(bootstrap/system class loader)加载的对象，这些类是不能够被回收的，他们可以以静态字段的方式保存持有其它对象。我们需要注意的一点就是，通过用户自定义的类加载器加载的类，除非相应的Java.lang.Class实例以其它的某种（或多种）方式成为roots，否则它们并不是roots
- **JNI Local**  Local variable in native code, such as user defined JNI code or JVM internal code.Native代码中的本地变量, 例如用户定义的JNI代码或JVM内部代码.
- **JNI Global**  Global variable in native code, such as user defined JNI code or JVM internal code.Native代码中的全局变量, 例如用户定义的JNI代码或JVM内部代码. 
- ** Thread Block**   Object referred to from a currently active thread block.被当前活的线程块引用的对象.
- **Thread**   A started, but not stopped, thread.一个启动了, 未停止的线程.
- **Busy Monitor**  Everything that has called wait() or notify() or that is synchronized. For example, by calling synchronized(Object) or by entering a synchronized method. Static method means class, non-static method means object. 一切已经调用了wait()或notify()或是同步的(synchronized)的X. 例如, 调用synchronized(Object)或是进入一个同步的方法, 如果是静态的, 这个X就是X类, 非静态的, 这个X就是这个X的对象.(用于同步的监控对象)
- **Java Local**  Local variable. For example, input parameters or locally created objects of methods that are still in the stack of a thread.本地变量. 例如, 运行中的Thread的方法栈中的传入参数, 局部变量等.
- **Native Stack** In or out parameters in native code, such as user defined JNI code or JVM internal code. This is often the case as many methods have native parts and the objects handled as method parameters become GC roots. For example, parameters used for file/network I/O methods or reflection.Native代码中的传入/输出参数, 例如用户定义的JNI代码或JVM内部代码. 例如, 方法有部分native调用和方法的参数被当成GC Roots, 比如参数被 file/network I/O方法使用, 或者是使用了反射.
- **Finalizable** An object which is in a queue awaiting its finalizer to be run. 在队列中等待finalizer的对象.
- **Unfinalized**  An object which has a finalize method, but has not been finalized and is not yet on the finalizer queue. 对象有finalize方法, 但并没有被finalized且不在finalizer队列中.
- **Unreachable**  An object which is unreachable from any other root, but has been marked as a root by MAT to retain objects which otherwise would not be included in the analysis. 不能被任何Roots支配, 但是被MAT标记为Root而不被分析的对象.
- **Java Stack Frame**  A Java stack frame, holding local variables. Only generated when the dump is parsed with the preference set to treat Java stack frames as objects. Java栈的一帧, 持有本地变量. 仅当heap dump被设置成作为Java stack frames时才会产生.
- **Unknown**   An object of unknown root type. Some dumps, such as IBM Portable Heap Dump files, do not have root information. For these dumps the MAT parser marks objects which are have no inbound references or are unreachable from any other root as roots of this type. This ensures that MAT retains all the objects in the dump. 其他未知的root

# 生存还是死亡

即使在可达性分析算法中不可达的对象，也并非是“非死不可”的，这时候它们暂时处于“缓刑”阶段，要真正宣告一个对象死亡，**至少要经历两次标记过程：**

1. 如果对象在进行可达性分析后发现没有与GC Roots相连接的引用，那它将会被第一次标记并且进行一次筛选，筛选的条件是此对象是否是否有必要执行finalize()方法。对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。也就是说，finalize 方法只会被执行一次。
2. 如果这个对象被判定为有必要执行finalize()方法，那么这个对象将会放置在一个叫做F-Queue的队列之中。并在稍后由一个虚拟机自动建立的，低优先级的Finalizer线程去执行它。这里所谓“执行”是指虚拟机会触发这个方法，但并不承诺会等待它运行结束，这样做的原因是，如果有一个对象在finalize()方法中执行缓慢，或者发生死循环，将可能会导致F-Queue队列中其他对象永久处于等待，甚至导致整个内存回收系统崩溃。

**finalize()方法是对象逃脱死亡命运的最后一次机会，稍后GC将对F-Queue中的对象进行第二次小规模的标记，如果对象要在finalize（）中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己（this关键字）赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移除出“即将回收”的集合；如果对象这时候还没有逃脱，那基本上它就真的被回收了**

一段测试代码

```java
public class FinalizeEscapeGC {

  public static FinalizeEscapeGC SAVE_HOOK = null;

  public void isAlive() {
    System.out.println("yes, i am still alive :)");
  }

  @Override
  protected void finalize() throws Throwable {
    super.finalize();
    System.out.println("finalize mehtod executed!");
    FinalizeEscapeGC.SAVE_HOOK = this;
  }

  public static void main(String[] args) throws Throwable {
    SAVE_HOOK = new FinalizeEscapeGC();

    //对象第一次成功拯救自己
    SAVE_HOOK = null;
    System.gc();
    // 因为Finalizer方法优先级很低，暂停0.5秒，以等待它
    Thread.sleep(500);
    if (SAVE_HOOK != null) {
      SAVE_HOOK.isAlive();
    } else {
      System.out.println("no, i am dead :(");
    }

    // 下面这段代码与上面的完全相同，但是这次自救却失败了
    SAVE_HOOK = null;
    System.gc();
    // 因为Finalizer方法优先级很低，暂停0.5秒，以等待它
    Thread.sleep(500);
    if (SAVE_HOOK != null) {
      SAVE_HOOK.isAlive();
    } else {
      System.out.println("no, i am dead :(");
    }
  }
}
```
输出

```
finalize mehtod executed!
yes, i am still alive :)
no, i am dead :(
```
运行结果可以看到，SAVE_HOOK对象的finalize()方法确实被GC收集器触发过，并且在被收集前成功逃脱了。

另外一个值得注意的地方就是，代码中有两段完全一样的代码片段，执行结果却是一次逃脱成功，一次失败，**这是因为任何一个对象的finalize()方法都只会被系统自动调用一次**，如果对象面临下一次回收，它的finalize()方法不会被再次执行，因此第二段代码的自救行动失败了。

**finalize方法的特点**

1. 不要主动调用某个对象的finalize方法，该方法应该交给垃圾回收机制调用。
2. finalize方法何时被调用，是否被调用具有不确定性，不要把finalize方法当做一定会执行的方法
3. 当JVM执行可恢复对象的finalize方法时，可能使该对象或系统中其他对象重新变成可达状态
4. 当JVM调用finalize方法出现异常时，垃圾回收机制不会报告异常，程序继续执行。

> 由于finalize方法不一定被执行，那么我们想清理某各类里打开的资源时，则不要方法finalize方法中。

大部分场景finalizer线程清理finalizer队列是比较快的,但是一旦你在finalize方法里执行一些耗时的操作,可能导致内存无法及时释放进而导致内存溢出的错误,在实际场景还是推荐尽量少用finalize方法.

## 分析
如果我们在finalize方法上打上断点，可以看到，一个 FinalizerThread 的线程执行了我们的 finalize 方法。

![](/assets/images/posts/java-object-survival/java-object-survival-5.png)

这个 FinalizerThread 的初始化和启动在 Finalizer 的 static 块中，由 JVM 主动访问其外部类 Finalizer 初始化这个静态块。静态块会启动这个线程，这个线程的优先级是 8 ，比普通的线程要高一点，但是是 demon 线程。

```java
static {
	ThreadGroup tg = Thread.currentThread().getThreadGroup();
	for (ThreadGroup tgn = tg;
		 tgn != null;
		 tg = tgn, tgn = tg.getParent());
	Thread finalizer = new FinalizerThread(tg);
	finalizer.setPriority(Thread.MAX_PRIORITY - 2);
	finalizer.setDaemon(true);
	finalizer.start();
}
```

这个线程的任务则是死循环从 Finalizer 的队列`private static ReferenceQueue<Object> queue = new ReferenceQueue<>();`中，取出 Finalizer 对象，然后调用这些对象的 runFinalizer 方法。

```
for (;;) {
	try {
		Finalizer f = (Finalizer)queue.remove();
		f.runFinalizer(jla);
	} catch (InterruptedException x) {
		// ignore and continue
	}
}
```

当一个对象需要执行 finalize 方法（未执行过且重写了该方法）的时候， JVM 会将这个对象包装成 Finalizer 实例，然后，链接到 Finalizer 链表中，并放入这个队列。而这个 runFinalizer 方法的具体逻辑则是获取 Finalizer 对象包装的引用，即实际对象（是枚举则跳过），执行这个对象的 finalize 方法。执行完毕后，清空 Finalizer。

到这里，一个对象的 finalize 方法就执行结束了

** 如何放入队列？**

Jvm会给每个实现了finalize方法的实例创建一个监听,这个称为Finalizer,每次调用对象的finalize方法时,JVM会创建一个`java.lang.ref.Finalizer`对象,这个Finalizer对象会持有这个对象的引用,由于这些对象被Finilizer对象引用了,当对象数量较多时,就会导致Eden区空间满了,经历多次youngGC后可能对象就进入到老年代了.
`java.lang.ref.Finalizer`类继承自`java.lang.ref.FinalReference`,也是Reference的一种,因此Finalizer类里也有一个引用队列,这个引用队列是JVM和垃圾回收器打交道的唯一途径,当垃圾回收器需要回收该对象时,会把该对象放到引用队列中,这样java.lang.ref.Finalizer类就可以从队列中取出该对象,执行对象的finalize方法,并清除和该对象的引用关系.需要注意的是只有finalize方法实现不为空时JVM才会执行上述操作,JVM在类的加载过程中会标记该类是否为finalize类.

Reference类有一个高优先级的线程`ReferenceHandler`他的任务则是死循环执行 `tryHandlePending` 方法。处理 Reference 的 pending 属性，而这个属性其实就是 Reference 自己。GC 的时候，会设置这个地址 pending 地址。当这个线程发现 pending 地址不是空，就会尝试将自身放到自己的 queue 属性队列中。

```
ReferenceQueue<? super Object> q = r.queue;
if (q != ReferenceQueue.NULL) q.enqueue(r);
```
因此，当我们构造了一个 Finalizer 对象，这个对象会被 GC 设置到自该对象的 pending 属性中，然后 ReferenceHandler 线程会处理这个 pending 属性，具体处理则是将自己添加到构造函数设置的队列中。这个时候，Finalizer 中的线程就可以从队列中取出这个 Finalizer 对象了。

所以，finalize 方法需要两个线程来处理它，一个是 ReferenceHandler ，一个是 FinalizerThread。前者负责将 Finalizer 对象放入到 Reference 队列中，后者负责从队列中取出 Finalizer 对象并调用实际对象的 finalize 方法。

## 小结
finalize对象至少经历两次GC才能被回收,因为只有在FinalizerThread执行完了finalize对象的finalize方法的情况下才有可能被下次GC回收，而有可能期间已经经历过多次GC了，但是一直还没执行finalize对象的finalize方法;

CPU资源不足的场景FinalizerThread线程可能因为优先级较低而一直没有执行对象的finalize方法,可能导致大部分对象进入到老年代,进而触发老年代GC,设置触发Full GC.

# 回收方法区

Java虚拟机规范中确实说过可以不要求虚拟机在方法区中实现垃圾回收，而且在方法区中进行垃圾回收的“性价比”一般比较低，方法区的垃圾收集主要回收两部分内容：废弃的常量和无用的类。

废弃的常量，以常量池中字面量的回收为例，假如一个字符串“abc”已经进入常量池中，但是当前系统已经没有任何一个String对象叫做“abc”的，也没有任何其他地方引用这个字面量，这个“abc”常量就会被清理出常量池。

判断一个无用的类需要同时满足下面3个条件才能算是“无用的类”

1. 该类的所有实例都已经被回收
2. 加载该类的ClassLoader已经被回收
3. 该类对应的java.lang.Class对象已经没有任何地方被引用，无法在任何地方通过反射访问该类的方法。

# 参考资料

https://www.jianshu.com/p/09a574dcd5df

https://mp.weixin.qq.com/s/y6HE6WlDb-SuuF7g5qlKyg

https://zhuanlan.zhihu.com/p/27939756

https://www.zhihu.com/question/21539353

https://help.eclipse.org/2019-12/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Fconcepts%2Fgcroots.html

https://www.yourkit.com/docs/java/help/gc_roots.jsp

https://www.ezlippi.com/blog/2018/04/final-reference.html
