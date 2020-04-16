---
layout: post
title: Vert.x Context
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-context.html
---

Context

http://vertx.io/blog/an-introduction-to-the-vert-x-context-object/

当执行一个vert.x的handler，或者verticle的start、stop方法时，方法的执行会与一个特定的context关联。一般这个context是一个event-loop类型的context，它会与一个event-loop的线程关联。context会被传播，当一个handler被一个特定的context创建之后，这个handler的执行会被同一个context执行。例如：如果verticle的start方法设置了一些eventbus的handler，那么这些handler会在同一个context里执行，这个context与verticle的start方法也是同一个context.

> runOnContext is typically used when you've received some result from a 3rd party asynchronous API and you wish to process it on the correct context for the verticle.


    Context context = Vertx.currentContext();
    System.out.println("Running with context : " + Vertx.currentContext());
// Our blocking action
    System.out.println(Thread.currentThread());
    Thread thread = new Thread() {
      public void run() {
        // No context here!
        System.out.println(Thread.currentThread());
        System.out.println("Current context : " + Vertx.currentContext());
        context.runOnContext(v -> {
          // Runs on the same context
          System.out.println(Thread.currentThread());
          System.out.println("Runs on the original context : " + Vertx.currentContext());
        });
      }
    };
//
    thread.start();

输出

	Running with context : io.vertx.core.impl.EventLoopContext@7112b8fe
	Thread[vert.x-eventloop-thread-0,5,main]
	Thread[Thread-4,5,main]
	Current context : null
	Thread[vert.x-eventloop-thread-0,5,main]
	Runs on the original context : io.vertx.core.impl.EventLoopContext@7112b8fe

通过上面的例子可以看到 thread.run最外层的线程和context都发生了变化

**blocking**

    vertx.runOnContext(v -> {

      // On the event loop
      System.out.println("Calling blocking block from " + Thread.currentThread());

      Handler<Future<String>> blockingCodeHandler = future -> {
        // Non event loop
        System.out.println("Computing with " + Thread.currentThread());
        future.complete("some result");
      };

      Handler<AsyncResult<String>> resultHandler = result -> {
        // Back to the event loop
        System.out.println("Got result in " + Thread.currentThread());
      };

      // Execute the blocking code handler and the associated result handler
      vertx.executeBlocking(blockingCodeHandler, resultHandler);
    });

输出

	Calling blocking block from Thread[vert.x-eventloop-thread-0,5,main]
	Computing with Thread[vert.x-worker-thread-0,5,main]
	Got result in Thread[vert.x-eventloop-thread-0,5,main]

下面的两个测试方法中testBlock中的toComplete.thenRun，虽然它的线程没有改变，但是已经不再正确的context上了，所以不会调用context上定义的异常处理函数，这个方法会一直阻塞。

testImmediateCompletion方法，通过使用runOnContext方法可以使toComplete.thenRun重新回到正确的context上。

    @Rule
    public final RunTestOnContext rule = new RunTestOnContext();

    //线程未变，toComplete.thenRun运行在正确的线程上，但是不在正确的context上,所以不会调用context上的异常处理函数，这个方法会一直阻塞
    @Test
    public void testBlock(TestContext context) {
        final Async async = context.async();
        final Vertx vertx = rule.vertx();
        final CompletableFuture<Integer> toComplete = new CompletableFuture<>();
        // delay future completion by 500 ms
        final String threadName = Thread.currentThread().getName();
        System.out.println(threadName);
        toComplete.complete(100);
        vertx.getOrCreateContext().exceptionHandler(throwable -> {
            throwable.printStackTrace();
            async.complete();
        });
        toComplete.thenRun(() -> {
            System.out.println(Thread.currentThread().getName());
            context.assertEquals(Thread.currentThread().getName(), threadName);
            throw new RuntimeException("a");
			//async.complete();
        });
    }

    @Test
    public void testImmediateCompletion(TestContext context) {
        final Async async = context.async();
        final Vertx vertx = rule.vertx();
        final CompletableFuture<Integer> toComplete = new CompletableFuture<>();
        // delay future completion by 500 ms
        final String threadName = Thread.currentThread().getName();
        System.out.println(threadName);
        toComplete.complete(100);
        vertx.getOrCreateContext().exceptionHandler(throwable -> {
            throwable.printStackTrace();
            async.complete();
        });
        final Context currentContext = vertx.getOrCreateContext();
        toComplete.thenRun(() -> {
            currentContext.runOnContext(v -> {
                System.out.println(Thread.currentThread().getName());
                context.assertEquals(Thread.currentThread().getName(), threadName);
                throw new RuntimeException("a");
				//async.complete();
            });
        });
    }


> While getting back onto the correct context may not be critical if you have remained on the event loop thread throughout, it is critical if you are going to invoke subsequent vert.x handlers, update verticle state or anything similar, so it’s a sensible general approach.


当我们通过deployVerticle方法部署一个Verticle的时候，会为这个Verticle分配一个Context

	  public void deployVerticle(String identifier,
	                             DeploymentOptions options,
	                             Handler<AsyncResult<String>> completionHandler) {
	    ContextImpl callingContext = vertx.getOrCreateContext();
	    ClassLoader cl = getClassLoader(options, callingContext);
	    doDeployVerticle(identifier, generateDeploymentID(), options, callingContext, callingContext, cl, completionHandler);
	  }

所有在同一个context中允许的handler都会在同一个eventloop线程中运行。这样如果一个Verticle是唯一一个访问一个临界资源的Verticle，那么这个临界资源置只会被一个线程访问，因此不需要使用同步方法连限制资源的访问


用法

    vertx.runOnContext(v -> {
      //todo
    });

# 实现

	Context
		-ContextInternal
			-ContextImpl
				-EventLoopContext
				-WorkerContext
					-MultiThreadedWorkerContext

根据Context的继承关系，可以看出，有三种类型的Context

- EventLoopContext
- WorkerContext
- MultiThreadedWorkerContext

## Context接口
Context接口定义了一些基本的方法

判断当前线程类型的方法

	/**
	* Is the current thread a worker thread?
	* <p>
	* NOTE! This is not always the same as calling {@link Context#isWorkerContext}. If you are running blocking code
	* from an event loop context, then this will return true but {@link Context#isWorkerContext} will return false.
	*
	* @return true if current thread is a worker thread, false otherwise
	*/
	static boolean isOnWorkerThread() {
		return ContextImpl.isOnWorkerThread();
	}
	
	/**
	* Is the current thread an event thread?
	* <p>
	* NOTE! This is not always the same as calling {@link Context#isEventLoopContext}. If you are running blocking code
	* from an event loop context, then this will return false but {@link Context#isEventLoopContext} will return true.
	*
	* @return true if current thread is a worker thread, false otherwise
	*/
	static boolean isOnEventLoopThread() {
		return ContextImpl.isOnEventLoopThread();
	}
	
	/**
	* Is the current thread a Vert.x thread? That's either a worker thread or an event loop thread
	*
	* @return true if current thread is a Vert.x thread, false otherwise
	*/
	static boolean isOnVertxThread() {
		return ContextImpl.isOnVertxThread();
	}


判断context类型的方法
	
	/**
	* Is the current context an event loop context?
	* <p>
	* NOTE! when running blocking code using {@link io.vertx.core.Vertx#executeBlocking(Handler, Handler)} from a
	* standard (not worker) verticle, the context will still an event loop context and this {@link this#isEventLoopContext()}
	* will return true.
	*
	* @return true if  false otherwise
	*/
	boolean isEventLoopContext();
	
	/**
	* Is the current context a worker context?
	* <p>
	* NOTE! when running blocking code using {@link io.vertx.core.Vertx#executeBlocking(Handler, Handler)} from a
	* standard (not worker) verticle, the context will still an event loop context and this {@link this#isWorkerContext()}
	* will return false.
	*
	* @return true if the current context is a worker context, false otherwise
	*/
	boolean isWorkerContext();
	
	/**
	* Is the current context a multi-threaded worker context?
	*
	* @return true if the current context is a multi-threaded worker context, false otherwise
	*/
	boolean isMultiThreadedWorkerContext();

因为ContextImpl的实现机制，`isOnWorkerThread`，`isWorkerContext`, `isEventLoopContext`， `isOnEventLoopThread`的返回值在不同的情况下有所区别

共享数据的方法

	<T> T get(String key);
	
	void put(String key, Object value);
	
	boolean remove(String key);

异常处理

	Context exceptionHandler(@Nullable Handler<Throwable> handler);
	
	Handler<Throwable> exceptionHandler();

关闭钩子

	void addCloseHook(Closeable hook);
	
	void removeCloseHook(Closeable hook);

Verticle部署相关

	String deploymentID();
	
	JsonObject config();

执行handler

	void runOnContext(Handler<Void> action);
	
	<T> void executeBlocking(Handler<Future<T>> blockingCodeHandler, boolean ordered, Handler<AsyncResult<T>> resultHandler);
	
	<T> void executeBlocking(Handler<Future<T>> blockingCodeHandler, Handler<AsyncResult<T>> resultHandler);

其他

	//返回launcher的参数
	List<String> processArgs();

## ContextInternal
ContextInternal接口增加了一些内部方法，这些方法不会暴露给开发者

  //返回Context使用的Netty EventLoop
  EventLoop nettyEventLoop();

  <T> void executeBlocking(Handler<Future<T>> blockingCodeHandler, TaskQueue queue, Handler<AsyncResult<T>> resultHandler);

  /**
   * Execute the context task and switch on this context if necessary, this also associates the
   * current thread with the current context so {@link Vertx#currentContext()} returns this context.<p/>
   *
   * The caller thread should be the the event loop thread of this context.<p/>
   *
   * Any exception thrown from the {@literal stack} will be reported on this context.
   *
   * @param task the task to execute
   */
  void executeFromIO(ContextTask task);

## ContextImpl
ContextImpl封装了Context的基本逻辑，它有几个抽象方法需要子类实现

	  protected abstract void executeAsync(Handler<Void> task);
	
	  @Override
	  public abstract boolean isEventLoopContext();
	
	  @Override
	  public abstract boolean isMultiThreadedWorkerContext();
	
	  protected abstract void checkCorrectThread();

由于代码较多，详细内容在最后在阅读

## EventLoopContext
最通用的类型，它的executeAsync直接将handler交由Eventloop线程执行

	public void executeAsync(Handler<Void> task) {
		// No metrics, we are on the event loop.
		nettyEventLoop().execute(wrapTask(null, task, true, null));
	}

它的checkCorrectThread方法会检查线程是否是VertxThread，并且线程必须和context的线程相同

	@Override
	protected void checkCorrectThread() {
		Thread current = Thread.currentThread();
		if (!(current instanceof VertxThread)) {
		  throw new IllegalStateException("Expected to be on Vert.x thread, but actually on: " + current);
		} else if (contextThread != null && current != contextThread) {
		  throw new IllegalStateException("Event delivered on unexpected thread " + current + " expected: " + contextThread);
		}
	}

## WorkerContext
使用worker pool线程池运行，所有的任务按顺序执行

它的executeAsync和executeFromIO方法会将任务放在一个队列(TaskQueue)中执行

	@Override
	public void executeAsync(Handler<Void> task) {
		orderedTasks.execute(wrapTask(null, task, true, workerPool.metrics()), workerPool.executor());
	}

	@Override
	public void executeFromIO(ContextTask task) {
		orderedTasks.execute(wrapTask(task, null, true, workerPool.metrics()), workerPool.executor());
	}

## MultiThreadedWorkerContext

使用worker pool线程池运行，所有的任务并发执行

它的executeAsync方法会将任务直接交由工作线程池执行

	@Override
	public void executeAsync(Handler<Void> task) {
		workerPool.executor().execute(wrapTask(null, task, false, workerPool.metrics()));
	}

## ContextImpl

### 构造函数
ContextImpl的构造函数比较简单，设置一个成员变量

- VertxInternal vertx: vertx对象
- WorkerPool internalBlockingPool: 内部的线程池
- WorkerPool workerPool:工作线程池
- String deploymentID：部署ID
- JsonObject config: 配置属性
- ClassLoader tccl: ClassLoader
- EventLoop eventLoop：Eventloop线程
- TaskQueue orderedTasks: 任务队列
- TaskQueue internalOrderedTasks:内部的任务队列
- CloseHooks closeHooks：关闭钩子

	protected ContextImpl(VertxInternal vertx, WorkerPool internalBlockingPool, WorkerPool workerPool, String deploymentID, JsonObject config,
	                    ClassLoader tccl) {
		if (DISABLE_TCCL && !tccl.getClass().getName().equals("sun.misc.Launcher$AppClassLoader")) {
		  log.warn("You have disabled TCCL checks but you have a custom TCCL to set.");
		}
		this.deploymentID = deploymentID;
		this.config = config;
		EventLoopGroup group = vertx.getEventLoopGroup();
		if (group != null) {
		  this.eventLoop = group.next();
		} else {
		  this.eventLoop = null;
		}
		this.tccl = tccl;
		this.owner = vertx;
		this.workerPool = workerPool;
		this.internalBlockingPool = internalBlockingPool;
		this.orderedTasks = new TaskQueue();
		this.internalOrderedTasks = new TaskQueue();
		this.closeHooks = new CloseHooks(log);
	}

### 关闭钩子

	//添加钩子
	public void addCloseHook(Closeable hook) {
		closeHooks.add(hook);
	}
	
	//删除钩子
	public void removeCloseHook(Closeable hook) {
		closeHooks.remove(hook);
	}
	
	//运行钩子
	public void runCloseHooks(Handler<AsyncResult<Void>> completionHandler) {
		closeHooks.run(completionHandler);
		// Now remove context references from threads
		VertxThreadFactory.unsetContext(this);
	}

在运行钩子`closeHooks.run(completionHandler);`之后，会通过`VertxThreadFactory.unsetContext(this)`方法解除Context和线程的绑定关系

	public static synchronized void unsetContext(ContextImpl ctx) {
		for (VertxThread thread: weakMap.keySet()) {
		  if (thread.getContext() == ctx) {
		    thread.setContext(null);
		  }
		}
	}


我们先看一下CloseHooks，CloseHooks内部有三个属性

- Logger log：日志
- boolean closeHooksRun：关闭钩子是否在运行
- Set<Closeable> closeHooks:关闭构造的集合

	private final Logger log;
	private boolean closeHooksRun;
	private Set<Closeable> closeHooks;
	
	CloseHooks(Logger log) {
		this.log = log;
	}

新增、删除钩子的方法很简单，往closeHooks中添加和删除即可

	synchronized void add(Closeable hook) {
		if (closeHooks == null) {
		  // Has to be concurrent as can be removed from non context thread
		  closeHooks = new HashSet<>();
		}
		closeHooks.add(hook);
	}

	synchronized void remove(Closeable hook) {
		if (closeHooks != null) {
		  closeHooks.remove(hook);
		}
	}

在运行关闭钩子之前，首先需要检查钩子是否已经在运行，然后将钩子的集合复制到一个副本里面，用来避免在运行钩子时修改钩子。**对钩子的新增、删除、复制都需要放在一个同步方法中**

    synchronized (this) {
      if (closeHooksRun) {
        // Sanity check
        throw new IllegalStateException("Close hooks already run");
      }
      closeHooksRun = true;
      if (closeHooks != null && !closeHooks.isEmpty()) {
        // Must copy before looping as can be removed during loop otherwise
        copy = new HashSet<>(closeHooks);
      }
    }

如果没有关闭钩子，直接返回成功：

	completionHandler.handle(Future.succeededFuture());

如果有关闭钩子，依次运行

    AtomicInteger count = new AtomicInteger();
    AtomicBoolean failed = new AtomicBoolean();
    for (Closeable hook : copy) {
      Future<Void> a = Future.future();
      a.setHandler(ar -> {
        if (ar.failed()) {
          if (failed.compareAndSet(false, true)) {
            // Only report one failure
            completionHandler.handle(Future.failedFuture(ar.cause()));
          }
        } else {
          if (count.incrementAndGet() == num) {
            // closeHooksRun = true;
            completionHandler.handle(Future.succeededFuture());
          }
        }
      });
      try {
        hook.close(a);
      } catch (Throwable t) {
        log.warn("Failed to run close hooks", t);
        a.tryFail(t);
      }
    }

对于钩子的运行结果，通过两个原子变量来判断：`AtomicInteger count`和` AtomicBoolean failed`
在每个钩子运行完之后，如果钩子运行失败，会通过failed的CAS操作的结果`failed.compareAndSet(false, true)`来判断是否需要将completionHandler标记为失败,**这样可以保证只会有一个失败的钩子来设置completionHandler，避免completionHandler被标记多次**

如果钩子运行成功，会将count加1，并检查count是否等于钩子的总数，如果count等于钩子的总数，则说明所有的钩子都运行成功，直接将completionHandler标记为成功

### 共享变量
共享变量的代码很检查，在Context内部使用一个`Map<String, Object> contextData`来保存变量

	@Override
	@SuppressWarnings("unchecked")
	public <T> T get(String key) {
		return (T) contextData().get(key);
	}
	
	@Override
	public void put(String key, Object value) {
		contextData().put(key, value);
	}
	
	@Override
	public boolean remove(String key) {
		return contextData().remove(key) != null;
	}

	protected synchronized Map<String, Object> contextData() {
		if (contextData == null) {
		  contextData = new ConcurrentHashMap<>();
		}
		return contextData;
	}

### executeBlocking
Context有4种executeBlocking方法

1.内部的executeBlocking，使用一个internalBlockingPool线程池和internalOrderedTasks队列来执行任务，所有的任务都按顺序执行

	// Execute an internal task on the internal blocking ordered executor
	public <T> void executeBlocking(Action<T> action, Handler<AsyncResult<T>> resultHandler) {
		executeBlocking(action, null, resultHandler, internalBlockingPool.executor(), internalOrderedTasks, internalBlockingPool.metrics());
	}

2.使用工作线程池workerPool来执行任务。可以选择顺序执行，或者不按顺序执行,如果按顺序执行，则使用orderedTasks来保证任务的顺序

	@Override
	public <T> void executeBlocking(Handler<Future<T>> blockingCodeHandler, boolean ordered, Handler<AsyncResult<T>> resultHandler) {
		executeBlocking(null, blockingCodeHandler, resultHandler, workerPool.executor(), ordered ? orderedTasks : null, workerPool.metrics());
	}
	
3.使用工作线程池workerPool和orderedTasks队列来执行任务。所有的任务都按顺序执行

	@Override
	public <T> void executeBlocking(Handler<Future<T>> blockingCodeHandler, Handler<AsyncResult<T>> resultHandler) {
		executeBlocking(blockingCodeHandler, true, resultHandler);
	}

4.使用工作线程池workerPool和自定义的任务队列来执行任务，所有的任务按顺序执行

	@Override
	public <T> void executeBlocking(Handler<Future<T>> blockingCodeHandler, TaskQueue queue, Handler<AsyncResult<T>> resultHandler) {
		executeBlocking(null, blockingCodeHandler, resultHandler, workerPool.executor(), queue, workerPool.metrics());
	} 

上面的executeBlocking方法最终都会通过
`
executeBlocking(Action<T> action, Handler<Future<T>> blockingCodeHandler,
      Handler<AsyncResult<T>> resultHandler,
      Executor exec, TaskQueue queue, PoolMetrics metrics)
`
方法来执行

这个方法以及在block已经阅读，不在详细阅读。

需要注意，如果是是不是使用内部的executeBlocking，还需要将执行阻塞方法的线程与上下文绑定`ContextImpl.setContext(this);`

	if (blockingCodeHandler != null) {
		ContextImpl.setContext(this);
		blockingCodeHandler.handle(res);
	} else {
		T result = action.perform();
		res.complete(result);
	}

所以在一个standard (not worker) verticle里面执行executeBlocking方法，isEventLoopContext方法依然返回true,isWorkerContext方法依然返回false。但是它的isOnWorkerThread会返回true,isOnEventLoopThread方法会返回false

因为VertxThread在由VertxThreadFactory创建时就已经指定了是否是工作线程

	//VertxThreadFactory
	VertxThreadFactory(String prefix, BlockedThreadChecker checker, boolean worker, long maxExecTime) {
		this.prefix = prefix;
		this.checker = checker;
		this.worker = worker;
		this.maxExecTime = maxExecTime;
	}

	public Thread newThread(Runnable runnable) {
		VertxThread t = new VertxThread(runnable, prefix + threadCount.getAndIncrement(), worker, maxExecTime);
		//...
	}

	//ContextImpl
	public static boolean isOnWorkerThread() {
		return isOnVertxThread(true);
	}
	
	public static boolean isOnEventLoopThread() {
		return isOnVertxThread(false);
	}
	
	public static boolean isOnVertxThread() {
		Thread t = Thread.currentThread();
		return (t instanceof VertxThread);
	}
	
	private static boolean isOnVertxThread(boolean worker) {
		Thread t = Thread.currentThread();
		if (t instanceof VertxThread) {
		  VertxThread vt = (VertxThread) t;
		  return vt.isWorker() == worker;
		}
		return false;
	}

### runOnContext
runOnContext直接调用子类的executeAsync方法执行

  public void runOnContext(Handler<Void> task) {
    try {
      executeAsync(task);
    } catch (RejectedExecutionException ignore) {
      // Pool is already shut down
    }
  }

前面已经描述EventLoopContext的executeAsync由Eventloop线程执行，WorkerContext会使用工作线程池顺序执行，MultiThreadedWorkerContext会直接交由工作线程池并发执行

### executeFromIO

	// This is called to execute code where the origin is IO (from Netty probably).
	// In such a case we should already be on an event loop thread (as Netty manages the event loops)
	// but check this anyway, then execute directly
	public void executeFromIO(ContextTask task) {
		if (THREAD_CHECKS) {
		  checkCorrectThread();
		}
		// No metrics on this, as we are on the event loop.
		wrapTask(task, null, true, null).run();
	}

### wrapTask
wrapTask将handler包装为一个Runnable对象，它的逻辑很简单，它分为几个部分

1.检查当前线程，EventLoopContext执行任务的线程一定要与EventLoopContext的线程相同

	Thread th = Thread.currentThread();
	if (!(th instanceof VertxThread)) {
		throw new IllegalStateException("Uh oh! Event loop context executing with wrong thread! Expected " + contextThread + " got " + th);
	}
	VertxThread current = (VertxThread) th;
	if (THREAD_CHECKS && checkThread) {
	if (contextThread == null) {
	  contextThread = current;
	} else if (contextThread != current && !contextThread.isWorker()) {
	  throw new IllegalStateException("Uh oh! Event loop context executing with wrong thread! Expected " + contextThread + " got " + current);
	}
	}

2.执行任务的具体逻辑`cTask.run`或者`hTask.handle(null)`，如果运行出现异常，使用异常处理函数处理异常。如果断Context没有自定义异常处理函数，就使用Vertx的异常处理函数，否则直接使用自定义的异常处理函数

	try {
		setContext(current, ContextImpl.this);
		if (cTask != null) {
		  cTask.run();
		} else {
		  hTask.handle(null);
		}
		if (metrics != null) {
		  metrics.end(metric, true);
		}
	} catch (Throwable t) {
		log.error("Unhandled exception", t);
		Handler<Throwable> handler = this.exceptionHandler;
		if (handler == null) {
		  handler = owner.exceptionHandler();
		}
		if (handler != null) {
		  handler.handle(t);
		}
		if (metrics != null) {
		  metrics.end(metric, false);
		}
	}

### 异常处理

	@Override
	public Context exceptionHandler(Handler<Throwable> handler) {
		exceptionHandler = handler;
		return this;
	}
	
	@Override
	public Handler<Throwable> exceptionHandler() {
		return exceptionHandler;
	}

### getInstanceCount
如果没有Deployment对象，返回0，没有DeploymentOptions，返回1；否则返回DeploymentOptions中定义的实例数。

	public void setDeployment(Deployment deployment) {
		this.deployment = deployment;
	}
	
	public Deployment getDeployment() {
		return deployment;
	}

	public int getInstanceCount() {
	// the no verticle case
	if (deployment == null) {
	  return 0;
	}
	
	// the single verticle without an instance flag explicitly defined
	if (deployment.deploymentOptions() == null) {
	  return 1;
	}
	return deployment.deploymentOptions().getInstances();
	}

在使用DeploymentManager部署Context时会调用setDeployment

      context.setDeployment(deployment);
      deployment.addVerticle(new VerticleHolder(verticle, context));