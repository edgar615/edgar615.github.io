---
layout: post
title: Java reference
date: 2020-01-13
categories:
    - JVM
comments: true
permalink: java-reference.html
---

Java中一共有四种引用类型, 强引用(StrongReference), 软引用(SoftReference), 弱引用(WeakReference), 虚引用(PhantomReference); 

**强引用(Strong Reference)**

代码中普遍存在的，`Object obj = new Object()` 所创建的引用，只要强引用存在，垃圾收集器就永远不会回收被引用对象。当内存不足的时候，jvm 就会抛出 OutOfMemory 错误，而不会回收强引用的对象。

**软引用(Sofe Reference)**

有用但并非必须的对象，可用`SoftReference`类来实现软引用，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行二次回收。如果这次回收还没有足够的内存，才会抛出内存异常。

在该对象被垃圾回收器回收之前，通过`SoftReference`类的get()方法可以获得该Java对象的强引用，一旦该对象被回收之后，get()就会返回null。

软引用最常用的场景是高速缓存

**弱引用(Weak Reference)**

被弱引用关联的对象只能生存到下一次垃圾收集发生之前，JDK提供了`WeakReference`类来实现弱引用。

弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。 

**虚引用(Phantom Reference)**

也称为幽灵引用或幻影引用，是最弱的一种引用关系，JDK提供了`PhantomReference`类来实现虚引用。一个持有虚引用的对象和几乎没有引用是一样的，随时可能被垃圾回收器所回收。虚引用的get方法总是会返回null。为了追踪垃圾回收过程，虚引用必须和引用队列一起使用。

> 虚引用一般用于追踪垃圾收集器的回收动作。相比对象的finalize方法，虚引用的方式更加灵活和安全。

```java
Object obj = new Object();
ReferenceQueue<Object> refQueue =new ReferenceQueue<>();
PhantomReference<Object> phanRef =new PhantomReference<>(obj, refQueue);

Object objg = phanRef.get();
//这里拿到的是null
System.out.println(objg);
//让obj变成垃圾
obj=null;
System.gc();
Thread.sleep(3000);
//gc后会将phanRef加入到refQueue中
Reference<? extends Object> phanRefP = refQueue.remove();
//这里输出true
System.out.println(phanRefP==phanRef);
```

从以上代码中可以看到，虚引用能够在指向对象不可达时得到一个'通知'（其实所有继承References的类都有这个功能），需要注意的是GC完成后，phanRef.referent依然指向之前创建Object，也就是说Object对象一直没被回收！

**严格的说，虚引用是会影响对象生命周期的，如果不做任何处理，只要虚引用不被回收，那其引用的对象永远不会被回收。所以一般来说，从ReferenceQueue中获得PhantomReference对象后，如果PhantomReference对象不会被回收的话（比如被其他GC ROOT可达的对象引用），需要调用clear方法解除PhantomReference和其引用对象的引用关系，使得对象下次可以GC时可以被回收掉。**

PhantomReference 有两个好处:

- 它可以让我们准确地知道对象何时被从内存中删除， 这个特性可以被用于一些特殊的需求中(例如 Distributed GC， XWork 和 google-guice 中也使用 PhantomReference 做了一些清理性工作).
- 它可以避免 finalization 带来的一些根本性问题, 如果某个对象重载了 finalize 方法并故意在方法内创建本身的强引用, 这将导致这一轮的 GC 无法回收这个对象并有可能引起任意次 GC， 最后的结果就是明明 JVM 内有很多 Garbage 却 OutOfMemory， 使用 PhantomReference 就可以避免这个问题， 因为 PhantomReference 是在 finalize 方法执行后回收的，也就意味着此时已经不可能拿到原来的引用, 也就不会出现上述问题, 当然这是一个很极端的例子, 一般不会出现.


# 可达性判断
Java有5种类型的可达性状态：

- 强可到达，如果从GC Root搜索后，发现对象与GC Root之间存在强引用链则为强可到达。强引用链即有强引用对象，引用了该对象
- 软可到达，如果从GC Root搜索后，发现对象与GC Root之间不存在强引用链，但存在软引用链，则为软可到达。软引用链即有软引用对象，引用了该对象
- 弱可到达，如果从GC Root搜索后，发现对象与GC Root之间不存在强引用链与软引用链，但有弱引用链，则为弱可到达。弱引用链即有弱引用对象，引用了该对象
- 虚可到达，如果从GC Root搜索后，发现对象与GC Root之间只存在虚引用链则为虚可到达。虚引用链即有虚引用对象，引用了该对象
- 不可达，如果从GC Root搜索后，找不到对象与GC Root之间的引用链，则为不可到达。

![](/assets/images/posts/java-object-survival/java-reference-1.png)

ObjectA为强可到达，ObjectB也为强可到达，虽然ObjectB对象被SoftReference ObjcetE 引用但由于其还被ObjectA引用所以为强可到达;而ObjectC和ObjectD为弱引用达到，虽然ObjectD对象被PhantomReference ObjcetG引用但由于其还被ObjectC引用，而ObjectC又为弱引用达到，所以ObjectD为弱引用达到;而ObjectH与ObjectI是不可到达。引用链的强弱有关系依次是 强引用 > 软引用 > 弱引用 > 虚引用，如果有更强的引用关系存在，那么引用链到达性，将由更强的引用有关系决定。

## 状态转换

对象可达性状态是随着程序运行而不断变化的

1. 对象创建后一般是强可达的
2. 当GC Roots对象到该对象的强引用被清除后：如果剩余引用链最高为软引用，则状态转换为软可达的；反之如果最高为弱引用，则状态转换为弱可达的，反之则把对象标记为可执行finalize方法状态
3. 当软可达对象重新被强引用连接时，则转换为强可达状态；当软可达对象的软引用被清除后，如果剩余引用链最高为弱引用，则状态转换为弱可达；反之则把对象标记为可执行finalize方法状态
4. 当弱可达对象重新被强引用或者软引用连接时，则可转换为强可达或者软可达；当弱可达对象的弱引用被清除后，则把对象标记为可执行finalize方法状态
5. 可对象被标记为可执行finalize方法状态，如果对象finalize从未被执行，则执行finalize方法，并标记对象的finalize方法已经被执行（在finalize方法可能会重新生成强/软/弱引用等，对象状态会重新转换为强/软/弱可达，不过并不推荐这么做，因为可能会导致对象状态紊乱，无法被正常回收）；反之当对象有虚引用连接时，则转换为虚可达状态，否则转换为不可达状态
6. 虚可达对象在垃圾回收后状态转换为不可达（不能通过虚引用获取对象引用，所以对象状态不会再转换为强/软/弱可达）；

![](/assets/images/posts/java-reference/java-reference-2.jpg)

# Reference
Reference类是所有引用类型的基类，它与垃圾回收是密切配合的，所以该类不能被直接子类化。简单来讲，Reference的继承类都是经过严格设计的，甚至连成员变量的先后顺序都不能改变，所以在代码中直接继承Reference类是没有任何意义的。但是可以继承Reference类的子类。例如：Finalizer 继承自 FinalReference，Cleaner 继承自 PhantomReference

## 私有变量
我们先看看它的私有变量

```java
public abstract class Reference<T> {
	// 引用的对象
    private T referent;
	// 回收队列，由使用者在Reference的构造函数中指定，一旦Reference对象放入了队列里面，那么queue就会被设置为ReferenceQueue.ENQUEUED，来标识当前Reference已经进入到队里里面了
    volatile ReferenceQueue<? super T> queue;
	// 当该引用被加入到queue中的时候，该字段被设置为queue中的下一个元素，以形成链表结构.其取值会根据Reference不同状态发生改变，
    volatile Reference next;
	// 在GC时，JVM底层会维护一个叫DiscoveredList的链表，存放的是Reference对象，discovered字段指向的就是链表中的下一个元素，由JVM设置
    transient private Reference<T> discovered;  /* used by VM */
	// 垃圾收集器线程同步的锁对象，它是一个静态变量，意味着所有Reference对象共用同一个锁。垃圾收集器必须在每次收集开始时获取此锁，因此任何持有此锁的代码都必须尽快完成，尽可能不分配新对象，并避免调用用户代码
    static private class Lock { }
    private static Lock lock = new Lock();
	// 等待加入queue的Reference对象，在GC时由JVM设置，会有一个java层的线程(ReferenceHandler)源源不断的从pending中提取元素加入到queue，它是一个静态对象，意味着所有Reference对象共用同一个pending队列。这个列表由上面的lock对象锁进行保护。列表使用discovered字段来链接它的元素。
    private static Reference<Object> pending = null;
	//
    private static class ReferenceHandler extends Thread {
		//...
    }
}
```

**queue**

ReferenceQueue名义上是一个队列，实际内部是使用单链表来表示的单向队列，**可以理解为queue就是一个链表，其自身仅存储当前的head节点，后面的节点由每个reference节点通过next来保持即可**。所以Reference对象本身就是一个链表的节点，这个链表数据来源是由ReferenceHander线程从pending队列中取的数据构建的。一旦Reference对象放入了队列里面，那么queue就会被设置为ReferenceQueue.ENQUEUED，来标识当前Reference已经进入到队里里面了

这个队列的意义在于增加一种判断机制，可以在外部通过监控这个队列来判断对象是否被回收。如果一个对象即将被回收，那么引用这个对象的reference对象就会被放到这个队列中。通过监控这个队列，就可以取出这个reference后再进行一些善后处理。

如果没有这个队列，就只能通过不断地轮询reference对象，通过get方法是否返回null( phantomReference对象不能这样做，其get方法始终返回null，因此它只有带queue的构造函数 )来判断对象是否被回收。

这两种方法均有相应的使用场景，具体使用需要具体情况具体分析。比如在weakHashMap中，就通过查询queue的数据，来判定是否有对象将被回收。而ThreadLocalMap，则采用判断get()是否为null来进行处理。


**discovered**

表示要处理的对象的下一个对象。取值会根据Reference不同状态发生改变

- 状态为active时，代表由GC维护的discovered-Reference链表的下个节点，如果是尾部则为当前实例本身
- 状态为pending时，代表pending-Reference的下个节点的引用
- 否则为null

queue队列使用next来查找下一个reference，pending队列使用discovered来查找下一个reference。

> 注意要查找的下一个对象实际和reference对象的状态有关

**pending**

pending与discovered一起构成了一个pending单向链表，**注意pending是一个静态对象，所以是全局唯一的**，pending为链表的头节点，discovered为链表当前Reference节点指向下一个节点的引用，这个队列是由jvm的垃圾回收器构建的，当对象除了被reference引用之外没有其它强引用了，jvm的垃圾回收器就会将指向需要回收的对象的Reference都放入到这个队列里面。


##  状态

> JDK11和JDK8的实现不同，网上大多数都是JDK8的实现，
> 11的实现可以参考http://www.throwable.club/2019/02/16/java-reference/


reference对象一共有四种状态，Active（活跃状态）、Pending（半死不活状态）、Enqueued（濒死状态）、Inactive（凉凉状态）。Pending和Enqueued状态是引用实例在创建时注册了引用队列才会有。

Reference源码中并不存在一个成员变量用于描述Reference的状态，它是通过组合判断referent、discovered、queue、next成员的存在性或者顺序”拼凑出”对应的状态javadoc中对状态进行了说明

> **Active:**
> reference如果处于此状态，会受到垃圾处理器的特殊处理。当垃圾回收器检测到referent已经更改为合适的状态后(没有任何强引用和软引用关联)，会在某个时间将实例的状态更改为Pending或者Inactive。具体取决于实例是否在创建时注册到一个引用队列中。
> 在前一种情况下（将状态更改为Pending），他还会将实例添加到pending-Reference列表中。新创建的实例处于活动状态。
>
> **Pending:**
>  实例如果处于此状态，表明它是pending-Reference列表中的一个元素，等待被Reference-handler线程做入队处理。未注册引用队列的实例永远不会处于该状态。
>
> **Enqueued:**
> 实例如果处于此状态，表明它已经是它注册的引用队列中的一个元素，当它被从引用队列中移除时，它的状态将会变为Inactive，未注册引用队列的实例永远不会处于该状态。
>
> **Inactive:**
> 实例如果处于此状态它的状态将永远不会再改变了


一个reference处于Active状态时，表示它是活跃正常的，垃圾回收器会监视这个引用的referent，如果扫描到它没有任何强引用关联时就会进行回收判定了。

如果判定为需要进行回收，则判断其是否注册了引用队列，如果有的话将reference的状态置为pending。当reference处于pending状态时，表明已经准备将它放入引用队列中，在这个状态下要处理的对象将逐个放入queue中。在这个时间窗口期，相应的引用对象为pending状态。

当它进入到Enqueued状态时，表明已经引用实例已经被放到queue当中了，准备由外部线程来轮询获取相应信息。此时引用指向的对即将被垃圾回收器回收掉了。

当它变成Inactive状态时，表明它已经凉透了，它的生命已经到了尽头。不管你用什么方式，也救不了它了。

JVM中并没有显示定义这样的状态，而是通过next和queue来进行判断。

>Active：如果创建Reference对象时，没有传入ReferenceQueue，queue=ReferenceQueue.NULL。如果有传入，则queue指向传入的ReferenceQueue队列对象。next == null；
>
>Pending：queue为初始化时传入ReferenceQueue对象；next == this；
>
>Enqueue：queue == ReferenceQueue.ENQUEUED；next为queue中下一个reference对象，或者若为最后一个了next == this；
>
>Inactive：queue == ReferenceQueue.NULL; next == this.

如下图所示，Reference 的处理流程相当于事件处理

- 如果 new Reference 的时候如果没有传入 ReferenceQueue，相当于使用 JVM 的默认处理流程，达到一定条件的时候由GC回收；
- 如果 new Reference 的时候传入了 ReferenceQueue，相当于使用自定义的事件处理流程，此时的 ReferenceQueue 相当于事件监听器，Reference 则相当于每个事件，GC 标记的时候添加 discovered链表相当于事件发现过程，pending和enqueued则相当于注册事件的过程，最后需要用户自定义事件处理逻辑；

![](/assets/images/posts/java-reference/java-reference-2.png)

在reference引用的对象被回收后，该Reference实例会被添加到ReferenceQueue中，但是这个不是垃圾回收器来做的，这个操作还是一定的复杂度的，如果垃圾回收器还要执行这个操作，就会降低其效率。垃圾回收器做的是一个非常轻量级的操作：把Reference添加到pending-Reference链表中。Reference对象中有一个静态的pending成员变量，它就是这个pending-Reference链表的头结点。而另一个成员变量discovered就是这个链表的指针，指向下一个节点。


## 静态代码块
当 Refrence 类被加载的时候，会执行静态代码块。在静态代码块里面，会启动 ReferenceHandler 线程,并设置线程的级别为最大级别`Thread.MAX_PRIORITY`

```
static {
	// 得到了层级最高的线程组即 System线程组
	ThreadGroup tg = Thread.currentThread().getThreadGroup();
	for (ThreadGroup tgn = tg;
		 tgn != null;
		 tg = tgn, tgn = tg.getParent());
	// 在里面加入了一个名为 “Reference Handler” 的 优先级最高 的 ReferenceHandler 线程；
	Thread handler = new ReferenceHandler(tg, "Reference Handler");
	/* If there were a special system-only priority greater than
	 * MAX_PRIORITY, it would be used here
	 */
	handler.setPriority(Thread.MAX_PRIORITY);
	handler.setDaemon(true);
	handler.start();

	// 用于保证 JVM 在抛出 OOM 之前，原子性的清除非强引用的所有引用，如果空间仍然不足才会抛出 OOM；其中 SharedSecrets用于访问类的私有变量，于反射不同的是，它不会创建新的对象
	SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
		@Override
		public boolean tryHandlePendingReference() {
			return tryHandlePending(false);
		}
	});
}
```

## ReferenceHandler 
`ReferenceHandler`是一个线程，它的任务是死循环执行 `tryHandlePending` 方法，处理 Reference 的 pending 属性，而这个属性其实就是 Reference 自己。该线程的作用就是在GC结束后，将被回收的Reference对象通知其所有者，然后其所有者做相应处理(一般是在下次做查询或者修改的时候清除已经被回收的对象引用)； 

```java
private static class ReferenceHandler extends Thread {

	// 确保类已经被初始化
	private static void ensureClassInitialized(Class<?> clazz) {
		try {
			Class.forName(clazz.getName(), true, clazz.getClassLoader());
		} catch (ClassNotFoundException e) {
			throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
		}
	}

	// 预加载并初始化 InterruptedException 和 Cleaner 类，来避免出现在循环运行过程中时由于内存不足而无法加载它们 
	static {
		ensureClassInitialized(InterruptedException.class);
		ensureClassInitialized(Cleaner.class);
	}

	ReferenceHandler(ThreadGroup g, String name) {
		super(g, name);
	}

	public void run() {
		while (true) {
			tryHandlePending(true);
		}
	}
}
```

我们在看一下`tryHandlePending`方法，这个方法主要完成了`discovered -> pending -> enqueued`的整个入队注册流程；值得注意的是虽然Cleaner是虚引用，但是它并不会入队，而是直接执行clean操作，也就意味着在使用Cleaner的时候不需要在起一个线程监听ReferenceQueue了；

```java
static boolean tryHandlePending(boolean waitForNotify) {
	Reference<Object> r;
	Cleaner c;
	try {
		// 全局同步锁
		synchronized (lock) {
			// 如果pending链表不为null，则开始进行处理	
			if (pending != null) {
				r = pending;
				// 使用 'instanceof' 有时会导致OutOfMemoryError，所以在将r从链表中摘除时先进行这个操作（why?）
				c = r instanceof Cleaner ? (Cleaner) r : null;
				// 移除头结点，将pending指向其后一个节点，即discovered
				pending = r.discovered;
				// 将当前节点的discovered设置为null；当前节点已经出队，不需要组成链表了
				r.discovered = null;
			} else {
				// 如果pending队列为空，则等待
				// 在锁上等待可能会造成OutOfMemoryError，因为它会试图分配exception对象
				// 垃圾收集器在完成收集时会通过notify方法唤醒ReferenceHandler
				if (waitForNotify) {
					lock.wait();
				}
				// retry if waited
				return waitForNotify;
			}
		}
	} catch (OutOfMemoryError x) {
		// Give other threads CPU time so they hopefully drop some live references
		// and GC reclaims some space.
		// Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
		// persistently throws OOME for some time...
		Thread.yield();
		// retry
		return true;
	} catch (InterruptedException x) {
		// retry
		return true;
	}

	// 如果摘除的元素是Cleaner类型，则执行其clean方法
	if (c != null) {
		c.clean();
		// 注意这里，这里已经不往下执行了，所以Cleaner对象是不会进入到队列里面的，给它设置ReferenceQueue的作用是为了让它能进入Pending队列后被ReferenceHander线程处理；
		return true;
	}

	// 最后，如果其引用队列不为空，则将对象放入到它自己的ReferenceQueue队列里
	ReferenceQueue<? super Object> q = r.queue;
	if (q != ReferenceQueue.NULL) q.enqueue(r);
	return true;
}
```

> 垃圾回收器会把 Reference对象添加进入discovered和 pending，ReferenceHandler 会移除它

# ReferenceQueue
ReferenceQueue队列是一个单向链表，ReferenceQueue里面只有一个header成员变量持有队列的队头，Reference对象是从队头做出队入队操作，所以它是一个后进先出的队列

## 私有变量

```java
public class ReferenceQueue<T> {

	// 内部类，它是用来做状态识别的，重写了enqueue入队方法，永远返回false，所以它不会存储任何数据
    private static class Null<S> extends ReferenceQueue<S> {
        boolean enqueue(Reference<? extends S> r) {
            return false;
        }
    }
	// 当Reference对象创建时没有指定queue或Reference对象已经处于inactive状态
    static ReferenceQueue<Object> NULL = new Null<>();
    // 当Reference已经被ReferenceHander线程从pending队列移到queue里面时
    static ReferenceQueue<Object> ENQUEUED = new Null<>();

    static private class Lock { };
    // 同步对象
    private Lock lock = new Lock();
    // 头节点
    private volatile Reference<? extends T> head = null;
    private long queueLength = 0;
}
```

入队方法

```java
boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
	synchronized (lock) {
		// 获取reference对象关联的ReferenceQueue，如果创建r时未注册ReferenceQueue则为NULL，同样如果r已从ReferenceQueue中移除其也为null
		ReferenceQueue<?> queue = r.queue;
		if ((queue == NULL) || (queue == ENQUEUED)) {
			return false;
		}
		// 只有r的队列是当前队列才允许入队
		assert queue == this;
		// 将 refrence 的状态置为 Enqueued，表示已经被回收
		r.queue = ENQUEUED;
		// 更新头节点
		r.next = (head == null) ? r : head;
		head = r;
		// 队列长度加1
		queueLength++;
		// 为FinalReference类型引用增加FinalRefCount数量
		if (r instanceof FinalReference) {
			sun.misc.VM.addFinalRefCount(1);
		}
		// 唤醒其他线程
		lock.notifyAll();
		return true;
	}
}
```

出队

```java
public Reference<? extends T> poll() {
	// 头结点为null直接返回，代表Reference还没有加入ReferenceQueue中
	if (head == null)
		return null;
	synchronized (lock) {
		return reallyPoll();
	}
}
// 真正的出队方法,将队列头部第一个对象从队列中移除出来，如果队列为空则直接返回null（此方法不会被阻塞
private Reference<? extends T> reallyPoll() {       /* Must hold lock */
	Reference<? extends T> r = head;
	if (r != null) {
		@SuppressWarnings("unchecked")
		// 保存下一个节点
		Reference<? extends T> rn = r.next;
		// 更新头节点，如果rn==r为null，否则就是下一个节点
		head = (rn == r) ? null : rn;
		// 更新Reference的queue值，代表r已从队列中移除
		r.queue = NULL;
		// 更新Reference的next为其本身
		r.next = r;
		// 队列长度减1
		queueLength--;
		// 为FinalReference类型引用减少FinalRefCount数量
		if (r instanceof FinalReference) {
			sun.misc.VM.addFinalRefCount(-1);
		}
		return r;
	}
	return null;
}
```

将头部第一个对象移出队列并返回，如果队列为空，则等待timeout时间后，返回null，这个方法会阻塞线程

```java
public Reference<? extends T> remove(long timeout)
	throws IllegalArgumentException, InterruptedException
{
	if (timeout < 0) {
		throw new IllegalArgumentException("Negative timeout value");
	}
	synchronized (lock) {
		// 获取队列头节点指向的Reference
		Reference<? extends T> r = reallyPoll();
		if (r != null) return r;
		long start = (timeout == 0) ? 0 : System.nanoTime();
		// 在timeout时间内尝试重试获取
		for (;;) {
			// 等待队列上有结点通知
			lock.wait(timeout);
			// 获取队列中的头节点指向的Reference
			r = reallyPoll();
			if (r != null) return r;
			if (timeout != 0) {
				long end = System.nanoTime();
				timeout -= (end - start) / 1000_000;
				// 已超时但还没有获取到队列中的头节点指向的Reference返回null
				if (timeout <= 0) return null;
				start = end;
			}
		}
	}
}
```

再来整体回顾一下Reference的核心处理流程。JVM在GC时如果当前对象只被Reference对象引用，JVM会根据Reference具体类型与堆内存的使用情况决定是否把对应的Reference对象加入到一个由Reference构成的pending链表上，如果能加入pending链表JVM同时会通知ReferenceHandler线程进行处理。ReferenceHandler线程收到通知后会调用`Cleaner#clean`或`ReferenceQueue#enqueue`方法进行处理。如果引用当前对象的Reference类型为WeakReference且堆内存不足，那么JMV就会把WeakReference加入到pending-Reference链表上，然后ReferenceHandler线程收到通知后会异步地做入队列操作。而我们的应用程序中的线程便可以不断地去拉取ReferenceQueue中的元素来感知JMV的堆内存是否出现了不足的情况，最终达到根据堆内存的情况来做一些处理的操作。实际上WeakHashMap低层便是过通上述过程实现的，只不过实现细节上有所偏差。

这一块看了很久还是有些没有完全搞清楚，网上找了个图片加深理解

![](/assets/images/posts/java-reference/java-reference-2.png)

![](/assets/images/posts/java-reference/java-reference-3.jpg)

# 参考资料

https://zhuanlan.zhihu.com/p/77357666

https://cloud.tencent.com/developer/article/1366147

http://www.throwable.club/2019/02/16/java-reference

https://juejin.im/post/5d9c61d0e51d45780c34a83a