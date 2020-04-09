---
layout: post
title: 线程池
date: 2020-04-09
categories:
    - 多线程
comments: true
permalink: thread-pool.html
---

> 本文大多数内容来自于美团的技术文章

线程池（Thread Pool）是一种基于池化思想管理线程的工具，经常出现在多线程服务器中，如MySQL。

线程过多会带来额外的开销，其中包括创建销毁线程的开销、调度线程的开销等等，同时也降低了计算机的整体性能。线程池维护多个线程，等待监督管理者分配可并发执行的任务。这种做法，一方面避免了处理任务时创建销毁线程开销的代价，另一方面避免了线程数量膨胀导致的过分调度问题，保证了对内核的充分利用。

当然，使用线程池可以带来一系列好处：

- **降低资源消耗**：通过池化技术重复利用已创建的线程，降低线程创建和销毁造成的损耗。
- **提高响应速度**：任务到达时，无需等待线程创建即可立即执行。
- **提高线程的可管理性**：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
- **提供更多更强大的功能**：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程池ScheduledThreadPoolExecutor，就允许任务延期执行或定期执行。

**线程池解决的问题是什么**

线程池解决的核心问题就是资源管理问题。在并发环境下，系统不能够确定在任意时刻中，有多少任务需要执行，有多少资源需要投入。这种不确定性将带来以下若干问题：

1. 频繁申请/销毁资源和调度资源，将带来额外的消耗，可能会非常巨大。
2. 对资源无限申请缺少抑制手段，易引发系统资源耗尽的风险。
3. 系统无法合理管理内部的资源分布，会降低系统的稳定性。

为解决资源分配这个问题，线程池采用了“池化”（Pooling）思想。池化，顾名思义，是为了最大化收益并最小化风险，而将资源统一在一起管理的一种思想。

> Pooling is the grouping together of resources (assets, equipment, personnel, effort, etc.) for the purposes of maximizing advantage or minimizing risk to the users. The term is used in finance, computing and equipment management.——wikipedia

# 线程池核心设计与实现

JDK8提供了五种创建线程池的方法：

1.创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。

```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

- 它是一种固定大小的线程池；
- corePoolSize和maximunPoolSize都为用户设定的线程数量nThreads；
- keepAliveTime为0，意味着一旦有多余的空闲线程，就会被立即停止掉；但这里keepAliveTime无效；
- 阻塞队列采用了LinkedBlockingQueue，它是一个无界队列；
- 由于阻塞队列是一个无界队列，因此永远不可能拒绝任务；
- 由于采用了无界队列，实际线程数量将永远维持在nThreads，因此maximumPoolSize和keepAliveTime将无效。


2.(JDK8新增)会根据所需的并发数来动态创建和关闭线程。能够合理的使用CPU进行对任务进行并发操作，所以适合使用在很耗时的任务。

> 注意返回的是ForkJoinPool对象。

```
public static ExecutorService newWorkStealingPool(int parallelism) {
    return new ForkJoinPool
        (parallelism,
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}
```

3.创建一个可缓存的线程池,可灵活回收空闲线程，若无可回收，则新建线程。

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

- 它是一个可以无限扩大的线程池；
- 它比较适合处理执行时间比较小的任务；
- corePoolSize为0，maximumPoolSize为无限大，意味着线程数量可以无限大；
- keepAliveTime为60S，意味着线程空闲时间超过60S就会被杀死；
- 采用SynchronousQueue装等待的任务，这个阻塞队列没有存储空间，这意味着只要有请求到来，就必须要找到一条工作线程处理他，如果当前没有空闲的线程，那么就会再创建一条新的线程。

4.创建一个单线程的线程池。

```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

- 它只会创建一条工作线程处理任务；
- 采用的阻塞队列为LinkedBlockingQueue；

5.创建一个定长线程池，支持定时及周期性任务执行。

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

- 它采用DelayQueue存储等待的任务，DelayQueue内部封装了一个PriorityQueue，它会根据time的先后时间排序，若time相同则根据sequenceNumber排序；
- DelayQueue也是一个无界队列；

工作线程的执行过程：

- 工作线程会从DelayQueue取已经到期的任务去执行；
- 执行结束后重新设置任务的到期时间，再次放回DelayQueue

# 实现

## UML

![](/assets/images/posts/thread-pool/thread-pool-1.png)

ThreadPoolExecutor实现的顶层接口是Executor，顶层接口Executor提供了一种思想：将任务提交和任务执行进行解耦。用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供Runnable对象，将任务的运行逻辑提交到执行器(Executor)中，由Executor框架完成线程的调配和任务的执行部分。ExecutorService接口增加了一些能力：（1）扩充执行任务的能力，补充可以为一个或一批异步任务生成Future的方法；（2）提供了管控线程池的方法，比如停止线程池的运行。

AbstractExecutorService则是上层的抽象类，将执行任务的流程串联了起来，保证下层的实现只需关注一个执行任务的方法即可。最下层的实现类ThreadPoolExecutor实现最复杂的运行部分，ThreadPoolExecutor将会一方面维护自身的生命周期，另一方面同时管理线程和任务，使两者良好的结合从而执行并行任务。

ThreadPoolExecutor的运行机制如下图所示：

![](/assets/images/posts/thread-pool/thread-pool-2.jpg)

线程池在内部实际上构建了一个生产者消费者模型，将线程和任务两者解耦，并不直接关联，从而良好的缓冲任务，复用线程。线程池的运行主要分成两部分：任务管理、线程管理。任务管理部分充当生产者的角色，当任务提交后，线程池会判断该任务后续的流转：（1）直接申请线程执行该任务；（2）缓冲到队列中等待线程执行；（3）拒绝该任务。线程管理部分是消费者，它们被统一维护在线程池内，根据任务请求进行线程的分配，当线程执行完任务后则会继续获取新的任务去执行，最终当线程获取不到任务的时候，线程就会被回收。

## 构造函数

```
public ThreadPoolExecutor(int corePoolSize,
						  int maximumPoolSize,
						  long keepAliveTime,
						  TimeUnit unit,
						  BlockingQueue<Runnable> workQueue,
						  ThreadFactory threadFactory,
						  RejectedExecutionHandler handler)
```

7个参数

- corePoolSize 线程池中核心线程数量
- maximumPoolSize 最大线程数量
- keepAliveTime 空闲时间（当线程池数量超过核心数量时，多余的空闲时间的存活时间，即超过核心线程数量的空闲线程，在多长时间内，会被销毁）
- unit 时间单位
- workQueue 当核心线程工作已满，需要存储任务的队列
- threadFactory 创建线程的工厂
- handler 当队列满了之后的拒绝策略

corePoolSize，maximumPoolSize，workQueue之间关系，会在后面任务调度部分介绍。

## 线程池的状态（生命周期）

线程池运行的状态，并不是用户显式设置的，而是伴随着线程池的运行，由内部来维护。线程池内部使用一个变量维护两个值：运行状态(runState)和线程数量 (workerCount)。在具体实现中，线程池将运行状态(runState)、线程数量 (workerCount)两个关键参数的维护放在了一起，如下代码所示：

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    
    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

ctl这个AtomicInteger类型，是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段， 它同时包含两部分的信息：线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)，高3位保存runState，低29位保存workerCount，两个变量之间互不干扰。用一个变量去存储两个值，可避免在做相关决策时，出现不一致的情况，不必为了维护两者的一致，而占用锁资源。

线程池共有五种状态

```java
    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```

- RUNNING 运行状态，可以接受新提交的任务，也能处理阻塞队列中的任务
- SHUTDOWN 关闭状态，指调用了 shutdown() 方法，不再接受新提交的任务了，但是可以继续处理阻塞队列中的任务
- STOP 指调用了 shutdownNow() 方法，不再接受新任务，同时抛弃阻塞队列里的所有任务并中断所有正在执行任务的线程。
- TIDYING 所有任务都执行完毕，workerCount(有效线程数)为0，在调用 shutdown()/shutdownNow() 中都会尝试更新为这个状态。
- TERMINATED 终止状态，当执行 terminated() 后会更新为这个状态。

![](/assets/images/posts/thread-pool/thread-pool-3.jpg)

## 任务调度

任务调度是线程池的主要入口，当用户提交了一个任务，接下来这个任务将如何执行都是由这个阶段决定的。了解这部分就相当于了解了线程池的核心运行机制。

首先，所有任务的调度都是由execute方法完成的，这部分完成的工作是：检查现在线程池的运行状态、运行线程数、运行策略，决定接下来执行的流程，是直接申请线程执行，或是缓冲到队列中执行，亦或是直接拒绝该任务。其执行过程如下：

1. 首先检测线程池运行状态，如果不是RUNNING，则直接拒绝，线程池要保证在RUNNING的状态下执行任务。
2. 当线程池中线程数(workerCount)小于corePoolSize时，新提交任务将创建一个新线程执行任务，**即使此时线程池中存在空闲线程。**
3. 当线程池中线程数达到corePoolSize且线程池内的阻塞队列(workQueue)未满，新提交任务将被放入workQueue中，等待线程池中任务调度执行 。
4. 当workQueue已满，且maximumPoolSize > corePoolSize时，新提交任务会创建新线程执行任务。
5. 当workQueue已满，且提交任务数超过maximumPoolSize，任务由RejectedExecutionHandler处理，, 默认的处理方式是直接抛异常。
6. 当线程池中线程数超过corePoolSize，且超过这部分的空闲时间达到keepAliveTime时，回收这些线程。
7. 当设置allowCoreThreadTimeOut(true)时，线程池中corePoolSize范围内的线程空闲时间达到keepAliveTime也将回收。

流程如下图

![](/assets/images/posts/thread-pool/thread-pool-4.jpg)

```java
public void execute(Runnable command) {
	if (command == null)
		throw new NullPointerException();
	/*
	 * Proceed in 3 steps:
	 *
	 * 1. If fewer than corePoolSize threads are running, try to
	 * start a new thread with the given command as its first
	 * task.  The call to addWorker atomically checks runState and
	 * workerCount, and so prevents false alarms that would add
	 * threads when it shouldn't, by returning false.
	 *
	 * 2. If a task can be successfully queued, then we still need
	 * to double-check whether we should have added a thread
	 * (because existing ones died since last checking) or that
	 * the pool shut down since entry into this method. So we
	 * recheck state and if necessary roll back the enqueuing if
	 * stopped, or start a new thread if there are none.
	 *
	 * 3. If we cannot queue task, then we try to add a new
	 * thread.  If it fails, we know we are shut down or saturated
	 * and so reject the task.
	 */
	 // clt记录着runState和workerCount
	int c = ctl.get();
	// 表示当前活动的线程数和核心线程数做比较
	if (workerCountOf(c) < corePoolSize) {
		// 如果活动线程数<核心线程数，添加一个新线程
        //addWorker中的第二个参数表示限制添加线程的数量是根据corePoolSize来判断还是maximumPoolSize来判断
		if (addWorker(command, true))
			// 如果添加线程成功则返回
			return;
		// 如果失败则重新获取 runState和 workerCount	
		c = ctl.get();
	}
	// 如果当前线程池是运行状态并且任务添加到队列成功
	if (isRunning(c) && workQueue.offer(command)) {
		// 重新获取 runState和 workerCount，双重检查
		int recheck = ctl.get();
		// 如果线程状态变了（非运行状态）就需要从阻塞队列移除任务
		if (! isRunning(recheck) && remove(command))
			// 拒绝任务
			reject(command);
		// 如果线程数等于0
		else if (workerCountOf(recheck) == 0)
			// 第一个参数为null，表示在线程池中创建一个线程，但不去启动
            // 第二个参数为false，将线程池的有限线程数量的上限设置为maximumPoolSize
			addWorker(null, false);
	}
	// 队列已满，再次调用addWorker方法，但第二个参数传入为false，将线程池的有限线程数量的上限设置为maximumPoolSize
	else if (!addWorker(command, false))
		reject(command);
}

```
通过上面的execute方法可以看到，最主要的逻辑还是在addWorker方法中实现的，addWorker接受两个参数

- `firstTask` the task the new thread should run first (or null if none). (指定新增线程执行的第一个任务或者不执行任务)
- `core` if  true use corePoolSize as bound, else  maximumPoolSize.(core如果为true则使用corePoolSize绑定，否则为maximumPoolSize。  （此处使用布尔指示符而不是值，以确保在检查其他状态后读取新值）。)

```java
private boolean addWorker(Runnable firstTask, boolean core) {
	retry:
	for (;;) {
		//  获取运行状态
		int c = ctl.get();
		int rs = runStateOf(c);

		// Check if queue empty only if necessary.
        // 如果状态值 >= SHUTDOWN (不接新任务&不处理队列任务)
        // 并且 如果 ！(rs为SHUTDOWN 且 firsTask为空 且 阻塞队列不为空，说明任务已经处理完)		
		if (rs >= SHUTDOWN &&
			! (rs == SHUTDOWN &&
			   firstTask == null &&
			   ! workQueue.isEmpty()))
			// 返回false，拒绝添加任务
			return false;

		for (;;) {
			// 获取线程数wc
			int wc = workerCountOf(c);
			// 如果wc大与容量 || core如果为true表示根据corePoolSize来比较,否则为maximumPoolSize
			if (wc >= CAPACITY ||
				wc >= (core ? corePoolSize : maximumPoolSize))
				// 返回false，拒绝添加任务
				return false;
			// 增加workerCount（原子操作）
			if (compareAndIncrementWorkerCount(c))
				// 如果增加成功，则跳出
				break retry;
			// 如果增加失败，则再次获取runState
			c = ctl.get();  // Re-read ctl
			// 如果当前的运行状态不等于rs，说明状态已被改变，返回重新执行
			if (runStateOf(c) != rs)
				continue retry;
			// else CAS failed due to workerCount change; retry inner loop
		}
	}

	boolean workerStarted = false;
	boolean workerAdded = false;
	Worker w = null;
	try {
		// 根据firstTask来创建Worker对象
		w = new Worker(firstTask);
		final Thread t = w.thread;
		if (t != null) {
			// 加锁
			final ReentrantLock mainLock = this.mainLock;
			mainLock.lock();
			try {
				// Recheck while holding lock.
				// Back out on ThreadFactory failure or if
				// shut down before lock acquired.
				// 再次获得线程池状态状态
				int rs = runStateOf(ctl.get());
				// 如果rs小于SHUTDOWN(处于运行)或者(rs=SHUTDOWN && firstTask == null)
                // firstTask == null证明只新建线程而不执行任务
				if (rs < SHUTDOWN ||
					(rs == SHUTDOWN && firstTask == null)) {
					// 如果线程活着就抛异常
					if (t.isAlive()) // precheck that t is startable
						throw new IllegalThreadStateException();
					// 将worker加入到workers(hashset)，workers包含池中的所有工作线程
					workers.add(w);
					// 获取工作线程数量
					int s = workers.size();
					//  //largestPoolSize记录着线程池中出现过的最大线程数量，初始值为0
					if (s > largestPoolSize)
						// 如果 s比它还要大，则将s赋值给它
						largestPoolSize = s;
					// worker的添加工作状态改为true    
					workerAdded = true;
				}
			} finally {
				mainLock.unlock();
			}
			// 如果worker的添加工作完成，启动线程，修改线程启动状态
			if (workerAdded) {
				t.start();
				workerStarted = true;
			}
		}
	} finally {
		// 如果worker未启动，说明worker没有被添加
		if (! workerStarted)
			// addWorkerFailed
			addWorkerFailed(w);
	}
	// 返回启动状态
	return workerStarted;
}
```

根据上面的源码，addWorker的执行流程如下图所示：

![](/assets/images/posts/thread-pool/thread-pool-5.png)

## Worker线程

线程池为了掌握线程的状态并维护线程的生命周期，设计了线程池内的工作线程Worker

```java
private final class Worker
	extends AbstractQueuedSynchronizer
	implements Runnable
{

	/** Thread this worker is running in.  Null if factory fails. */
	final Thread thread;
	/** Initial task to run.  Possibly null. */
	Runnable firstTask;
	/** Per-thread task counter */
	volatile long completedTasks;
}
```

Worker这个工作线程，实现了Runnable接口，并持有一个线程thread，一个初始化的任务firstTask。thread是在调用构造方法时通过ThreadFactory来创建的线程，可以用来执行任务；firstTask用它来保存传入的第一个任务，这个任务可以有也可以为null。如果这个值是非空的，那么线程就会在启动初期立即执行这个任务，也就对应核心线程创建时的情况；如果这个值是null，那么就需要创建一个线程去执行任务列表（workQueue）中的任务，也就是非核心线程的创建。

```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```

再前面介绍的addWorker方法中，如果worker添加成功，会启动线程执行任务

```java
// 如果worker的添加工作完成，启动线程，修改线程启动状态
if (workerAdded) {
    t.start();
    workerStarted = true;
}
```

下面我们就来看看Worker线程的run方法，run方法实际上调用的`ThreadPoolExecutor`的`runWorker(this)`方法

```java
final void runWorker(Worker w) {
	// 拿到当前线程
	Thread wt = Thread.currentThread();
	// 拿到当前任务
	Runnable task = w.firstTask;
	// 将Worker.firstTask置空 并且释放锁
	w.firstTask = null;
	w.unlock(); // allow interrupts
	boolean completedAbruptly = true;
	try {
		 // 如果task或者getTask不为空，则一直循环，getTask是从任务队列中取出任务
		while (task != null || (task = getTask()) != null) {
			w.lock();
			// If pool is stopping, ensure thread is interrupted;
			// if not, ensure thread is not interrupted.  This
			// requires a recheck in second case to deal with
			// shutdownNow race while clearing interrupt
			// 如果线程池状态>=STOP 或者 (线程中断且线程池状态>=STOP)且当前线程没有中断
            // 其实就是保证两点：
            // 1. 线程池没有停止
            // 2. 保证线程没有中断
			if ((runStateAtLeast(ctl.get(), STOP) ||
				 (Thread.interrupted() &&
				  runStateAtLeast(ctl.get(), STOP))) &&
				!wt.isInterrupted())
				// 中断当前线程
				wt.interrupt();
			try {
				// 空方法，用于扩展
				beforeExecute(wt, task);
				Throwable thrown = null;
				try {
					// 执行run方法(Runable对象)
					task.run();
				} catch (RuntimeException x) {
					thrown = x; throw x;
				} catch (Error x) {
					thrown = x; throw x;
				} catch (Throwable x) {
					thrown = x; throw new Error(x);
				} finally {
					// 空方法，用于扩展
					afterExecute(task, thrown);
				}
			} finally {
				// 执行完后， 将task置空， 完成任务++， 释放锁，会继续从队列中获取任务
				task = null;
				w.completedTasks++;
				w.unlock();
			}
		}
		completedAbruptly = false;
	} finally {
		 // 退出工作，回收线程
		processWorkerExit(w, completedAbruptly);
	}
}
```

通过源码，我们可以知道runWorker方法的执行过程：

1. while循环中，不断地通过getTask()方法从workerQueue中获取任务
2. 如果线程池正在停止，则中断线程。否则调用3.
3. 调用task.run()执行任务；
4. 如果task为null则跳出循环，执行processWorkerExit()方法，销毁线程`workers.remove(w);`

![](/assets/images/posts/thread-pool/thread-pool-6.jpg)

### getTask

```java
private Runnable getTask() {
	boolean timedOut = false; // Did the last poll() time out?

	for (;;) {
		// 获取线程池状态
		int c = ctl.get();
		int rs = runStateOf(c);

		// Check if queue empty only if necessary.
		// 线程池关闭、任务返回null,worker线程需要回收
		if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
			decrementWorkerCount();
			return null;
		}
		// 当前工作线程数
		int wc = workerCountOf(c);

		// Are workers subject to culling?
		// 如果allowCoreThreadTimeOut为true，核心线程也可以被回收
		// 如果工作线程数大于核心线程数，说明有线程需要被回收
		boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

		// 当前线程数量过多，需要回收线程
		if ((wc > maximumPoolSize || (timed && timedOut))
			&& (wc > 1 || workQueue.isEmpty())) {
			if (compareAndDecrementWorkerCount(c))
				return null;
			continue;
		}
		// 限时获取任务
		try {
			Runnable r = timed ?
				workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
				workQueue.take();
			if (r != null)
				return r;
			timedOut = true;
		} catch (InterruptedException retry) {
			timedOut = false;
		}
	}
}
```

getTask这部分进行了多次判断，为的是控制线程的数量，使其符合线程池的状态。如果线程池现在不应该持有那么多线程，则会返回null值。工作线程Worker会不断接收新任务去执行，而当工作线程Worker接收不到任务的时候，就会开始被回收。

![](/assets/images/posts/thread-pool/thread-pool-7.png)

### 回收线程

线程池中线程的销毁依赖JVM自动的回收，线程池做的工作是根据当前线程池的状态维护一定数量的线程引用，防止这部分线程被JVM回收，当线程池决定哪些线程需要回收时，只需要将其引用消除即可。Worker被创建出来后，就会不断地进行轮询，然后获取任务去执行，核心线程可以无限等待获取任务，非核心线程要限时获取任务。当Worker无法获取到任务，也就是获取的任务为空时，循环会结束，Worker会主动消除自身在线程池内的引用。线程回收的工作是在processWorkerExit方法完成的。

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
	if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
		decrementWorkerCount();

	final ReentrantLock mainLock = this.mainLock;
	mainLock.lock();
	try {
		completedTaskCount += w.completedTasks;
		workers.remove(w);
	} finally {
		mainLock.unlock();
	}

	tryTerminate();

	int c = ctl.get();
	if (runStateLessThan(c, STOP)) {
		if (!completedAbruptly) {
			int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
			if (min == 0 && ! workQueue.isEmpty())
				min = 1;
			if (workerCountOf(c) >= min)
				return; // replacement not needed
		}
		addWorker(null, false);
	}
}
```

![](/assets/images/posts/thread-pool/thread-pool-8.jpg)

事实上，在这个方法中，将线程引用移出线程池就已经结束了线程销毁的部分。但由于引起线程销毁的可能性有很多，线程池还要判断是什么引发了这次销毁，是否要改变线程池的现阶段状态，是否要根据新状态，重新分配线程。

Worker是通过继承AQS，使用AQS来实现独占锁这个功能。没有使用可重入锁ReentrantLock，而是使用AQS，为的就是实现不可重入的特性去反应线程现在的执行状态。

- lock方法一旦获取了独占锁，表示当前线程正在执行任务中。
- 如果正在执行任务，则不应该中断线程。
- 如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断。
- 线程池在执行shutdown方法或tryTerminate方法时会调用interruptIdleWorkers方法来中断空闲的线程，interruptIdleWorkers方法会使用tryLock方法来判断线程池中的线程是否是空闲状态；如果线程是空闲状态则可以安全回收。

> AQS就是一个并发包的基础组件，用来实现各种锁，各种同步组件的。它包含了state变量、加锁线程、等待队列等并发中的核心组件。

在线程回收过程中就使用到了这种特性，回收过程如下图所示：

![](/assets/images/posts/thread-pool/thread-pool-11.jpg)

# 任务缓冲

任务缓冲模块是线程池能够管理任务的核心部分。线程池的本质是对任务和线程的管理，而做到这一点最关键的思想就是将任务和线程两者解耦，不让两者直接关联，才可以做后续的分配工作。线程池中是以生产者消费者模式，通过一个阻塞队列来实现的。阻塞队列缓存任务，工作线程从阻塞队列中获取任务。

阻塞队列(BlockingQueue)是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

下图中展示了线程1往阻塞队列中添加元素，而线程2从阻塞队列中移除元素：

![](/assets/images/posts/thread-pool/thread-pool-9.jpg)

使用不同的队列可以实现不一样的任务存取策略

![](/assets/images/posts/thread-pool/thread-pool-10.jpg)

# 任务拒绝

任务拒绝模块是线程池的保护部分，线程池有一个最大的容量，当线程池的任务缓存队列已满，并且线程池中的线程数目达到maximumPoolSize时，就需要拒绝掉该任务，采取任务拒绝策略，保护线程池。

```java
public interface RejectedExecutionHandler {

    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

用户可以通过实现这个接口去定制拒绝策略，也可以选择JDK提供的四种已有拒绝策略

- DiscardOldestPolicy: 该策略将丢弃最老的一个请求，也就是即将被执行的一个任务，并尝试再次提交当前任务
- DiscardPolicy: 该策略默默地丢弃无法处理的任务，不予任何处理，如果允许任务丢失，我觉得这是最好的方案..

**AbortPolicy（中止策略）**

```java
public static class AbortPolicy implements RejectedExecutionHandler {

        public AbortPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```

**功能：**当触发拒绝策略时，直接抛出拒绝执行的异常，中止策略的意思也就是打断当前执行流程

**使用场景：**这个就没有特殊的场景了，但是一点要正确处理抛出的异常。

ThreadPoolExecutor中默认的策略就是AbortPolicy，ExecutorService接口的系列ThreadPoolExecutor因为都没有显示的设置拒绝策略，所以默认的都是这个。但是请注意，ExecutorService中的线程池实例队列都是无界的，也就是说把内存撑爆了都不会触发拒绝策略。当自己自定义线程池实例时，使用这个策略一定要处理好触发策略时抛的异常，因为他会打断当前的执行流程。

**DiscardPolicy（丢弃策略）**

```java
public static class DiscardPolicy implements RejectedExecutionHandler {

        public DiscardPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```

**功能：**直接静悄悄的丢弃这个任务，不触发任何动作

**使用场景：**如果你提交的任务无关紧要，你就可以使用它 。因为它就是个空实现，会悄无声息的吞噬你的的任务。所以这个策略基本上不用了

**DiscardOldestPolicy（弃老策略）**

```java
public static class DiscardOldestPolicy implements RejectedExecutionHandler {

        public DiscardOldestPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

**功能：**如果线程池未关闭，就弹出队列头部的元素，然后尝试执行

**使用场景：**这个策略还是会丢弃任务，丢弃时也是毫无声息，但是特点是丢弃的是老的未执行的任务，而且是待执行优先级较高的任务。基于这个特性，我能想到的场景就是，发布消息，和修改消息，当消息发布出去后，还未执行，此时更新的消息又来了，这个时候未执行的消息的版本比现在提交的消息版本要低就可以被丢弃了。因为队列中还有可能存在消息版本更低的消息会排队执行，所以在真正处理消息的时候一定要做好消息的版本比较。

**CallerRunsPolicy（调用者运行策略）**

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {

        public CallerRunsPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```

**功能**：当触发拒绝策略时，只要线程池没有关闭，就由提交任务的当前线程处理。

**使用场景**：一般在不允许失败的、对性能要求不高、并发量较小的场景下使用，因为线程池一般情况下不会关闭，也就是提交的任务一定会被运行，但是由于是调用者线程自己执行的，当多次提交任务时，就会阻塞后续任务执行，性能和效率自然就慢了。

# 常见问题
## Worker为什么不使用ReentrantLock来实现呢？

tryAcquire方法它是不允许重入的，而ReentrantLock是允许重入的。对于线程来说，如果线程正在执行是不允许其它锁重入进来的。

线程只需要两个状态，一个是独占锁，表明正在执行任务；一个是不加锁，表明是空闲状态。
在runWorker方法中，为什么要在执行任务的时候对每个工作线程都加锁呢？

## afterExecute

```java

class ExtendedExecutor extends ThreadPoolExecutor {
  // ...
  protected void afterExecute(Runnable r, Throwable t) {
    super.afterExecute(r, t);
    if (t == null && r instanceof Future<?>) {
      try {
        Object result = ((Future<?>) r).get();
      } catch (CancellationException ce) {
          t = ce;
      } catch (ExecutionException ee) {
          t = ee.getCause();
      } catch (InterruptedException ie) {
          Thread.currentThread().interrupt(); // ignore/reset
      }
    }
    if (t != null)
      System.out.println(t);
  }
}
```

## 如何正确关闭一个线程池

为了实现优雅停机的目标，我们应当先调用shutdown方法，调用这个方法也就意味着，这个线程池不会再接收任何新的任务，但是已经提交的任务还会继续执行，包括队列中的。所以，之后你还应当调用awaitTermination方法，这个方法可以设定线程池在关闭之前的最大超时时间，如果在超时时间结束之前线程池能够正常关闭，这个方法会返回true，否则，一旦超时，就会返回false。通常来说我们不可能无限制地等待下去，因此需要我们事先预估一个合理的超时时间，然后去使用这个方法。

如果awaitTermination方法返回false，你又希望尽可能在线程池关闭之后再做其他资源回收工作，可以考虑再调用一下shutdownNow方法，此时队列中所有尚未被处理的任务都会被丢弃，同时会设置线程池中每个线程的中断标志位。shutdownNow并不保证一定可以让正在运行的线程停止工作，除非提交给线程的任务能够正确响应中断。到了这一步，可以考虑继续调用awaitTermination方法，或者直接放弃，去做接下来要做的事情。

```java
executorService.shutdown();

while (!executorService.awaitTermination(100, TimeUnit.MILLISECONDS)) {

	logger.info("thread running");

}
```

## prestartAllCoreThreads方法一次性创建所有核心线程
我们知道一个线程池创建出来之后，在没有给它提交任何任务之前，这个线程池中的线程数为0。有时候我们事先知道会有很多任务会提交给这个线程池，但是等它一个个去创建新线程开销太大，影响系统性能，因此可以考虑在创建线程池的时候就将所有的核心线程全部一次性创建完毕，这样系统起来之后就可以直接使用了。

## 线程池创建完毕之后也是可以更改其线程数的

因为线程池提供了设置核心线程数和最大线程数的方法，它们分别是setCorePoolSize方法和setMaximumPoolSize方法。

面对线程池高负荷运行的情况，我们可以这么处理：

- 起一个定时轮询线程（守护类型），定时检测线程池中的线程数，具体来说就是调用getActiveCount方法。
- 当发现线程数超过了核心线程数大小时，可以考虑将CorePoolSize和MaximumPoolSize的数值同时乘以2，当然这里不建议设置很大的线程数，因为并不是线程越多越好的，可以考虑设置一个上限值，比如50、100之类的。
- 同时，去获取队列中的任务数，具体来说是调用getQueue方法再调用size方法。当队列中的任务数少于队列大小的二分之一时，我们可以认为现在线程池的负载没有那么高了，因此可以考虑在线程池先前有扩容过的情况下，将CorePoolSize和MaximumPoolSize还原回去，也就是除以2。

## 简化线程池配置

线程池构造参数有8个，但是最核心的是3个：corePoolSize、maximumPoolSize，workQueue，它们最大程度地决定了线程池的任务分配和线程分配策略。考虑到在实际应用中我们获取并发性的场景主要是两种：

1. 并行执行子任务，提高响应速度。这种情况下，应该使用同步队列，没有什么任务应该被缓存下来，而是应该立即执行。
2. 并行执行大批次任务，提升吞吐量。这种情况下，应该使用有界队列，使用队列去缓冲大批量的任务，队列容量必须声明，防止任务无限制堆积。

所以线程池只需要提供这三个关键参数的配置，并且提供两种队列的选择，就可以满足绝大多数的业务需求，

## `Executors`创建的线程池存在OOM的风险

真正的导致OOM的其实是`LinkedBlockingQueue.offer`方法。

Java中的`BlockingQueue`主要有两种实现，分别是`ArrayBlockingQueue` 和 `LinkedBlockingQueue`。

`ArrayBlockingQueue`是一个用数组实现的有界阻塞队列，必须设置容量。

`LinkedBlockingQueue`是一个用链表实现的有界阻塞队列，容量可以选择进行设置，不设置的话，将是一个无边界的阻塞队列，最大长度为`Integer.MAX_VALUE`。

这里的问题就出在：**不设置的话，将是一个无边界的阻塞队列，最大长度为Integer.MAX_VALUE。**也就是说，如果我们不设置`LinkedBlockingQueue`的容量的话，其默认容量将会是`Integer.MAX_VALUE`。

而`newFixedThreadPool`中创建`LinkedBlockingQueue`时，并未指定容量。此时，`LinkedBlockingQueue`就是一个无边界队列，对于一个无边界队列来说，是可以不断的向队列中加入任务的，这种情况下就有可能因为任务过多而导致内存溢出问题。

上面提到的问题主要体现在`newFixedThreadPool`和`newSingleThreadExecutor`两个工厂方法上，并不是说`newCachedThreadPool`和`newScheduledThreadPool`这两个方法就安全了，这两种方式创建的最大线程数可能是`Integer.MAX_VALUE`，而创建这么多线程，必然就有可能导致OOM。

# 参考资料

https://mp.weixin.qq.com/s/baYuX8aCwQ9PP6k7TDl2Ww

https://mp.weixin.qq.com/s/FVfuwIQ08mRrQy_PAd6WLw

https://mp.weixin.qq.com/s/owrAeEDf4wpgEWjKPueFaA
