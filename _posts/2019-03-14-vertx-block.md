---
layout: post
title: Vert.x阻塞任务
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-block.html
---

# Block

    vertx.executeBlocking(f -> {
      //block method
      try {
        TimeUnit.MILLISECONDS.sleep(2000);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      f.complete("hello");
    }, ar -> {
      System.out.println(System.currentTimeMillis() + ": " + ar.result());
    });
    
    vertx.executeBlocking(f -> {
      //block method
      try {
        TimeUnit.MILLISECONDS.sleep(1500);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      f.complete("hello2");
    }, ar -> {
      System.out.println(System.currentTimeMillis() + ": " + ar.result());
    });
    
    vertx.executeBlocking(f -> {
      //block method
      try {
        TimeUnit.MILLISECONDS.sleep(1000);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      f.complete("hello3");
    }, ar -> {
      System.out.println(System.currentTimeMillis() + ": " + ar.result());
    });

  输出

 1490011527207: hello
1490011528707: hello2
1490011529707: hello3

`executeBlocking(Handler<Future<T>> blockingCodeHandler,                                  Handler<AsyncResult<T>> asyncResultHandler)`  会将任务按顺序执行

我们也可以将任务不按顺序执行

    vertx.executeBlocking(f -> {
      //block method
      try {
        TimeUnit.MILLISECONDS.sleep(2000);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      f.complete("hello");
    }, false, ar -> {
      System.out.println(System.currentTimeMillis() + ": " + ar.result());
    });
    
    vertx.executeBlocking(f -> {
      //block method
      try {
        TimeUnit.MILLISECONDS.sleep(1500);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      f.complete("hello2");
    }, false, ar -> {
      System.out.println(System.currentTimeMillis() + ": " + ar.result());
    });
    
    vertx.executeBlocking(f -> {
      //block method
      try {
        TimeUnit.MILLISECONDS.sleep(1000);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      f.complete("hello3");
    }, false, ar -> {
      System.out.println(System.currentTimeMillis() + ": " + ar.result());
    });

输出

	1490011686474: hello3
	1490011686972: hello2
	1490011687477: hello

使用`executeBlocking(Handler<Future<T>> blockingCodeHandler, boolean ordered, Handler<AsyncResult<T>> asyncResultHandler) `方法可以将阻塞方法不按顺序执行

# 原理
Vert.x在创建VertxImpl对象时，创建了一个线程池 wokerPool

	  private final WorkerPool workerPool;
	
	    ExecutorService workerExec = Executors.newFixedThreadPool(options.getWorkerPoolSize(),
		new VertxThreadFactory("vert.x-worker-thread-", checker, true, options.getMaxWorkerExecuteTime()));
		
	    PoolMetrics workerPoolMetrics = isMetricsEnabled() ? metrics.createMetrics(workerExec, "worker", "vert.x-worker-thread", options.getWorkerPoolSize()) : null;
	
	    workerPool = new WorkerPool(workerExec, workerPoolMetrics);

WorkerPool是一个线程池和度量库的组合

	class WorkerPool {
	
	  private final ExecutorService pool;
	  private final PoolMetrics metrics;
	
	  WorkerPool(ExecutorService pool, PoolMetrics metrics) {
	    this.pool = pool;
	    this.metrics = metrics;
	  }
	
	  ExecutorService executor() {
	    return pool;
	  }
	
	  PoolMetrics metrics() {
	    return metrics;
	  }
	
	  void close() {
	    if (metrics != null) {
	      metrics.close();
	    }
	    pool.shutdownNow();
	  }
	}


​	
`executeBlocking`方法最终会调用ContextImpl的`executeBlocking`方法

	  @Override
	  public <T> void executeBlocking(Handler<Future<T>> blockingCodeHandler, boolean ordered, Handler<AsyncResult<T>> resultHandler) {
	    executeBlocking(null, blockingCodeHandler, resultHandler, workerPool.executor(), ordered ? orderedTasks : null, workerPool.metrics());
	  }

我们看一下executeBlocking的内部实现，它分为两部分组成
第一部分，将阻塞方法封装成一个Runnable，`blockingCodeHandler.handle(res);`在`Future res`完成之后触发回调函数,`runOnContext(v -> res.setHandler(resultHandler));`

	Runnable command = () -> {
		VertxThread current = (VertxThread) Thread.currentThread();
		Object execMetric = null;
		if (metrics != null) {
		  execMetric = metrics.begin(queueMetric);
		}
		if (!DISABLE_TIMINGS) {
		  current.executeStart();
		}
		Future<T> res = Future.future();
		try {
		  if (blockingCodeHandler != null) {
		    ContextImpl.setContext(this);
		    blockingCodeHandler.handle(res);
		  } else {
		    T result = action.perform();
		    res.complete(result);
		  }
		} catch (Throwable e) {
		  res.fail(e);
		} finally {
		  if (!DISABLE_TIMINGS) {
		    current.executeEnd();
		  }
		}
		if (metrics != null) {
		  metrics.end(execMetric, res.succeeded());
		}
		if (resultHandler != null) {
		  runOnContext(v -> res.setHandler(resultHandler));
		}
	      };

第二部分，通过`TaskQueue queue`来判断是否需要顺序执行，是不需要按顺序执行，会将上一步创建的任务放入线程池中执行,如果是需要按顺序执行，则会将任务放到任务队列中执行

      if (queue != null) {
        queue.execute(command, exec);
      } else {
        exec.execute(command);
      }

**TaskQueue**
TaskQueue在内部维护了一个链表

	private final LinkedList<Runnable> tasks = new LinkedList<>();

并创建了一个`Runnable runner`，这个Runnable主要是从队列的头部取到任务执行，直到队列中对任务为空

	  public TaskQueue() {
	    runner = () -> {
	      for (; ; ) {
		final Runnable task;
		synchronized (tasks) {
		  task = tasks.poll();
		  if (task == null) {
		    running = false;
		    return;
		  }
		}
		try {
		  task.run();
		} catch (Throwable t) {
		  log.error("Caught unexpected Throwable", t);
		}
	      }
	    };
	  }

TaskQueue的execute方法比较简单，将任务加入到队尾，然后判断队列中对任务是否在执行，如果没有执行，则将开始执行任务，并将running设为true

	  /**
	   * Run a task.
	   *
	   * @param task the task to run.
	   */
	  public void execute(Runnable task, Executor executor) {
	    synchronized (tasks) {
	      tasks.add(task);
	      if (!running) {
		running = true;
		executor.execute(runner);
	      }
	    }
	  }

# VertxThreadFactory
worker线程池创建线程时会使用VertxThreadFactory来创建新线程

	ExecutorService workerExec = Executors.newFixedThreadPool(options.getWorkerPoolSize(),
		new VertxThreadFactory("vert.x-worker-thread-", checker, true, options.getMaxWorkerExecuteTime()));  

VertxThreadFactory的构造方法接受４个参数：prefix线程池的名称前缀，checker检查阻塞时间的checker，worker标识，maxExecTime最大阻塞时间

	   VertxThreadFactory(String prefix, BlockedThreadChecker checker, boolean worker, long maxExecTime) {
	    this.prefix = prefix;
	    this.checker = checker;
	    this.worker = worker;
	    this.maxExecTime = maxExecTime;
	  }

线程创建之后，会将线程注册到BlockedThreadChecker

	  public Thread newThread(Runnable runnable) {
	    VertxThread t = new VertxThread(runnable, prefix + threadCount.getAndIncrement(), worker, maxExecTime);
	    // Vert.x threads are NOT daemons - we want them to prevent JVM exit so embededd user doesn't
	    // have to explicitly prevent JVM from exiting.
	    if (checker != null) {
	      checker.registerThread(t);
	    }
	    addToMap(t);
	    // I know the default is false anyway, but just to be explicit-  Vert.x threads are NOT daemons
	    // we want to prevent the JVM from exiting until Vert.x instances are closed
	    t.setDaemon(false);
	    return t;
	  }

BlockedThreadChecker再内部维护了一个`private final Map<VertxThread, Object> threads = new WeakHashMap<>();`用于保存上一步创建的线程，同时BlockedThreadChecker的构造方法会创建一个周期性的定时任务，定时检查`threads`中的线程执行实现，如果超过了maxExecTime，则打印警告日志

	BlockedThreadChecker(long interval, long warningExceptionTime) {
	    timer = new Timer("vertx-blocked-thread-checker", true);
	    timer.schedule(new TimerTask() {
	      @Override
	      public void run() {
		synchronized (BlockedThreadChecker.this) {
		  long now = System.nanoTime();
		  for (VertxThread thread : threads.keySet()) {
		    long execStart = thread.startTime();
		    long dur = now - execStart;
		    final long timeLimit = thread.getMaxExecTime();
		    if (execStart != 0 && dur > timeLimit) {
		      final String message = "Thread " + thread + " has been blocked for " + (dur / 1000000) + " ms, time limit is " + (timeLimit / 1000000);
		      if (dur <= warningExceptionTime) {
		        log.warn(message);
		      } else {
		        VertxException stackTrace = new VertxException("Thread blocked");
		        stackTrace.setStackTrace(thread.getStackTrace());
		        log.warn(message, stackTrace);
		      }
		    }
		  }
		}
	      }
	    }, interval, interval);
	  }