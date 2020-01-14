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

而对象的可达性与[引用类型](https://edgar615.github.io/java-reference.html)密切相关

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

这个 FinalizerThread 的初始化和启动在 Finalizer 的 static 块中，由 JVM 主动访问其外部类 Finalizer 初始化这个静态块。静态块会启动这个线程，这个线程的优先级是 8 ，比普通的线程要高一点，但是是 demon 线程。在cpu很紧张的情况下其被调度的优先级可能会受到影响

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

这个线程的任务则是死循环从 Finalizer 的队列`private static ReferenceQueue<Object> queue = new ReferenceQueue<>();`中，取出 Finalizer 对象(Finalizer对象链中剥离)，然后调用这些对象的 runFinalizer 方法。

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

runFinalizer方法将这个Finalizer对象关联的f对象传给了一个native方法invokeFinalizeMethod

```java
private void runFinalizer(JavaLangAccess jla) {
	synchronized (this) {
		if (hasBeenFinalized()) return;
		remove();
	}
	try {
		Object finalizee = this.get();
		if (finalizee != null && !(finalizee instanceof java.lang.Enum)) {
			jla.invokeFinalize(finalizee);

			/* Clear stack slot containing this variable, to decrease
			   the chances of false retention with a conservative GC */
			finalizee = null;
		}
	} catch (Throwable x) { }
	super.clear();
}
```
> runFinalizer方法里对Throwable的异常进行了捕获，所以finalize方法抛出的异常不会导致FinalizeThread退出

当一个对象需要执行 finalize 方法（未执行过且重写了该方法）的时候， JVM 会将这个对象包装成 Finalizer 实例，然后，链接到 Finalizer 链表中，并放入这个队列。而这个 runFinalizer 方法的具体逻辑则是获取 Finalizer 对象包装的引用，即实际对象（是枚举则跳过），执行这个对象的 finalize 方法。执行完毕后，清空 Finalizer。

到这里，一个对象的 finalize 方法就执行结束了

## 如何放入队列？

> 后面的分析的核心内容来自你假笨的文章https://mp.weixin.qq.com/s/fftHK8gZXHCXWpHxhPQpBg

观察Finalizer类，它扩展自FinalReference，而且并定义为final，构造函数是私有方法，意味着Finalizer类不能再被扩展，也不能被外部创建

```java
final class Finalizer extends FinalReference<Object> {
	// finalizee参数：FinalReference指向的对象引用；
    private Finalizer(Object finalizee) {
        super(finalizee, queue);
        // 通过add方法将当前对象插入到Finalizer对象链里，链里的对象和Finalizer类静态关联
        add();
    }
}
```

仔细阅读源码，我们发现在下面有一段交由JVM调用的静态方法

```java
/* Invoked by VM */
static void register(Object finalizee) {
	new Finalizer(finalizee);
}
```

那么VM什么时候调用这个方法将Finalizer 对象注册到 Finalizer 对象链里呢？

类的修饰有很多，比如 final，abstract，public 等，如果某个类用 final 修饰，我们就说这个类是 final 类，上面列的都是语法层面我们可以显式指定的，在 JVM 里其实还会给类标记一些其他符号，比如`finalizer`，表示这个类是一个`finalizer`类（为了和`java.lang.ref.Fianlizer`类区分，下文在提到的`finalizer`类时会简称为 f 类），GC 在处理这种类的对象时要做一些特殊的处理，如在这个对象被回收之前会调用它的`finalize`方法。

## 如何判断一个类是不是一个f类
在`java.lang.Object`里的有一个finalize的空方法

```java
protected void finalize() throws Throwable { }
```
这意味着 Java 里的所有类都会继承这个方法，甚至可以覆写该方法，并且根据方法覆写原则，如果子类覆盖此方法，方法访问权限至少 protected 级别的，这样其子类就算没有覆写此方法也会继承此方法。

而判断当前类是否是 f 类的标准并不仅仅是当前类是否含有一个参数为空，返回值为 void 的finalize方法，还要求finalize 方法必须非空，因此 Object 类虽然含有一个finalize方法，但它并不是 f 类，Object 的对象在被 GC 回收时其实并不会调用它的finalize方法。

**需要注意的是，类在加载过程中其实就已经被标记为是否为 f 类了。（JVM 在类加载的时候会遍历当前类的所有方法，包括父类的方法，只要有一个参数为空且返回 void 的非空finalize方法就认为这个类是 f 类。）**

**f类的对象何时传到Finalizer.register方法？**

对象的创建是被拆分为几个步骤的，对于`FinalizeTest finalizeTest = new FinalizeTest();`语句，对应的字节码

```
0: new           #5                  // class com/xxxx/FinalizeTest
3: dup
4: invokespecial #6                  // Method "<init>":()V
7: astore_1
```

先执行new分配好对象空间，然后再执行invokespecial调用构造函数，jvm里其实可以让用户选择在这两个时机中的任意一个将当前对象传递给Finalizer.register方法来注册到Finalizer对象链里，这个选择依赖于`RegisterFinalizersAtInit`这个vm参数是否被设置，默认值为true，也就是在调用构造函数返回之前调用`Finalizer.register`方法，如果通过`-XX:-RegisterFinalizersAtInit`关闭了该参数，那将在对象空间分配好之后就将这个对象注册进去。

另外需要提一点的是当我们通过clone的方式复制一个对象的时候，如果当前类是一个f类，那么在clone完成的时候将调用Finalizer.register方法进行注册。

**hotspot如何实现f类对象在构造函数执行完毕后调用Finalizer.register**

我们知道一个构造函数执行的时候，会去调用父类的构造函数，主要是为了能对继承自父类的属性也能做初始化，那么任何一个对象的初始化最终都会调用到Object的空构造函数里（任何空的构造函数其实并不空，会含有三条字节码指令，如下代码所示），为了不对所有的类的构造函数都做埋点调用`Finalizer.register`方法，hotspot的实现是在Object这个类在做初始化的时候将构造函数里的return指令替换为`_return_register_finalizer`指令，该指令并不是标准的字节码指令，是hotspot扩展的指令，这样在处理该指令的时候调用`Finalizer.register`方法，这样就在侵入性很小的情况下完美地解决了这个问题。

**f对象的finalize方法会执行多次吗**

如果我们在f对象的finalize方法里重新将当前对象赋值出去，变成可达对象，当这个f对象再次变成不可达的时候还会被执行finalize方法吗？答案是否定的，因为在执行完第一次finalize方法之后，这个f对象已经和之前的Finalizer对象关系剥离了，也就是下次gc的时候不会再发现Finalizer对象指向该f对象了，自然也就不会调用这个f对象的finalize方法了。

**Finalizer对象何时被放到ReferenceQueue里**

当gc发生的时候，gc算法会判断f类对象是不是只被Finalizer类引用（f类对象被Finalizer对象引用，然后放到Finalizer对象链里），如果这个类仅仅被Finalizer对象引用的时候，说明这个对象在不久的将来会被回收了现在可以执行它的finalize方法了，于是会将这个Finalizer对象放到Finalizer类的ReferenceQueue里，但是这个f类对象其实并没有被回收，因为Finalizer这个类还对他们持有引用，在gc完成之前，jvm会调用ReferenceQueue里的lock对象的notify方法（当ReferenceQueue为空的时候，FinalizerThread线程会调用ReferenceQueue的lock对象的wait方法直到被jvm唤醒），此时就会执行上面FinalizeThread线程里看到的其他逻辑了。

所以，finalize 方法需要两个线程来处理它，一个是 ReferenceHandler（更高优先级的线程） ，一个是 FinalizerThread。前者负责将 Finalizer 对象放入到 Reference 队列中，后者负责从队列中取出 Finalizer 对象并调用实际对象的 finalize 方法。

> 这部分原理可以查看https://edgar615.github.io/java-reference.html

## finalize导致的内存溢出
通过上面的分析，我们知道Jvm会给每个实现了finalize方法的实例创建一个监听Finalizer并加入到全局的ReferenceQueue，只要finalize没有执行，那么这些对象就会一直存在堆区，当对象数量较多时,就会导致Eden区空间满了,经历多次youngGC后可能对象就进入到老年代了.甚至导致内存溢出

测试代码

```java
public class FinalizeTest {

  private static class F {
    // 1M
    private byte[] array = new byte[1024*1024];
    @Override
    protected void finalize() throws Throwable {
//      System.out.println("finalize");
      TimeUnit.SECONDS.sleep(1);
      return;
    }
  }

  public static void main(String[] args) {
    for (int i =0; i < 1000; i ++) {
      F f = new F();
    }
  }
}
```

`-Xmx100m -XX:+PrintGCDetails`输出

```
Heap
 PSYoungGen      total 29696K, used 25303K [0x00000000fdf00000, 0x0000000100000000, 0x0000000100000000)
  eden space 25600K, 98% used [0x00000000fdf00000,0x00000000ff7b5ce0,0x00000000ff800000)
  from space 4096K, 0% used [0x00000000ffc00000,0x00000000ffc00000,0x0000000100000000)
  to   space 4096K, 0% used [0x00000000ff800000,0x00000000ff800000,0x00000000ffc00000)
 ParOldGen       total 68608K, used 68310K [0x00000000f9c00000, 0x00000000fdf00000, 0x00000000fdf00000)
  object space 68608K, 99% used [0x00000000f9c00000,0x00000000fdeb5ba8,0x00000000fdf00000)
 Metaspace       used 3504K, capacity 4500K, committed 4864K, reserved 1056768K
  class space    used 385K, capacity 388K, committed 512K, reserved 1048576K
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at com.xxxx.FinalizeTest$F.<init>(FinalizeTest.java:9)
	at com.xxxx.FinalizeTest$F.<init>(FinalizeTest.java:7)
	at com.xxxx.FinalizeTest.main(FinalizeTest.java:20)
```

## 小结
f 对象因为Finalizer的引用而变成了一个临时的强引用，即使没有其他的强引用，还是无法立即被回收

finalize对象至少经历两次GC才能被回收,因为只有在FinalizerThread执行完了finalize对象的finalize方法的情况下才有可能被下次GC回收，而有可能期间已经经历过多次GC了，但是一直还没执行finalize对象的finalize方法;

CPU资源不足的场景FinalizerThread线程可能因为优先级较低而一直没有执行对象的finalize方法,可能导致大部分对象进入到老年代,进而触发老年代GC,设置触发Full GC.

f 对象的finalize方法被调用后，这个对象其实还并没有被回收，虽然可能在不久的将来会被回收。

**最后贴一段大神RednaxelaFX的文字**

> https://www.zhihu.com/question/62953438
> **Java为什么有finalizer（尽管不可靠）？**
>
> 只是作为last line of defense，是一种保底的措施。由于GC只能管理自动内存资源而无法管理程序所需要的各种其它资源（例如GC堆外的native memory、file handle、socket、draw device handle啥的），程序员必须要自己手动来管理这些资源。只是纯粹为了避免在对象死了之后它原本持有的这样的资源泄漏，Java才提供了finalizer机制到让用户注册 finalize() 这么个回调方法来定制GC清理对象的行为。
>
> 确实Java语言规范里是说底层实现可以自由选择当对象可以被回收之后何时调用 finalize() ；换句话说在无限远的未来才调用 finalize() 也是符合规范的，现实来说就是等价于永远不调用 finalize() 也是符合规范的。但靠谱的JVM实现都还是会在维持GC效率的前提下尽快对可以回收的对象去调用 finalize() 方法。
>
> **有没有任何办法保证finalizer（或类似机制）在一定条件满足时（如GC被触发时）必会执行？**
>
> 现实上说是会被执行的，但没办法抽象地、严格地有什么机制去保证 finalize() 一定被调用。
>
> 作者：RednaxelaFX
>
> 链接：https://www.zhihu.com/question/62953438/answer/203759926
>
> 来源：知乎
>
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
>
> HotSpot VM实现finalizer的办法其实很直观：系统通过特殊的FinalReference（一种介于WeakReference和PhantomReference之间的内部实现的弱引用类型）来引用带有非空 finalize() 方法的对象——这些对象在创建的时候就会伴随创建出一个引用它的FinalReference出来。然后在GC的时候，FinalReference也跟其它弱引用类型一样由ReferenceProcessor发现并处理：GC的marking分为strong marking和weak marking两个阶段，在strong marking过程中如果mark到弱引用的话，并不是立即把其referent也mark上，而是会把弱引用记录在ReferenceProcessor里对应的队列里。Strong marking之后会做reference processing，扫描前面记录下来的弱引用看它们的referent是否已经被strong marking标记为活，如果是的话说明对象还活得好好的，就不管它了让它继续活下去，反之则意味着referent指向的对象只是weak-reachable，就要做相应的弱引用处理。对FinalReference来说，弱引用处理就是这次把referent还是标记为活的，并把它加入到finalize queue里去等着被FinalizeThread去调用其 finalize() 方法。等它的 finalize() 方法被调用过之后，下次再GC的时候这个对象就没有FinalReference引用了，所以就不会再经历一次弱引用处理，就可以好好长眠了。
>
> 那么finalizer会有啥问题呢？首先是会慢——GC在做reference processing的时候常常是在stop-the-world pause里做的，会增加GC暂停时间，而且后面跑finalizer也要占CPU；其次是finalizer还是有可能没能被调用上，会泄漏资源。
>
> 举个简单的例子，HotSpot VM默认是在一个JVM实例内开**一个** FinalizeThread 来专门去调用 finalize() 方法的。所有GC发现可以被回收但有finalizer的对象都会被挂在一个全局的finalize queue上，然后这个FinalizeThread从queue里逐个取出对象来调用其 finalize() 方法。那么假如有个对象在自己的 finalize() 写了个无限循环会怎样？在默认配置下，finalize queue里挂在这个对象之后的对象的 finalize() 方法就要在无限远的未来才能被调用到了——悲剧。
>
> JDK有提供一个标准API，System.runFinalization() ，来让当前Java线程在处理完finalize queue之前block住。这个API的描述也就是“尽力而为”，并不保证任何行为，但现实来说它是会等到finalize queue真的空了才返回的。JDK8u里HotSpot VM的实现方法是调用 [java.lang.ref.Finalizer.runFinalization()](https://link.zhihu.com/?target=http%3A//hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/e74259b3eadc/src/share/classes/java/lang/ref/Finalizer.java%23l111) ，其中会在 FinalizeThread 之外再开**一个** Secondary Finalize Thread 来尝试加速处理finalize queue。那么如果我们的finalize queue上挂着两个调皮的对象都有无限循环，这就还是能把两个处理线程都卡住，于是queue里剩下的对象还是得到无限远的未来才能得到处理


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

https://blog.csdn.net/qq_27639777/article/details/90143738

https://mp.weixin.qq.com/s/fftHK8gZXHCXWpHxhPQpBg

https://www.zhihu.com/question/62953438