---
layout: post
title: Vert.x定时器
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-timer.html
---

# Timer

**一次定时器**

定时定时器在某个延迟后调用事件处理程序，以毫秒表示

示例　 

    long timerID = vertx.setTimer(1000, id -> {
      System.out.println(System.currentTimeMillis() + ": And one second later this is printed");
    });

    System.out.println(System.currentTimeMillis() + ": First this is printed");

输出

	1489715167185: First this is printed
	1489715168187: And one second later this is printed

**周期性定时器**

示例

    long timerID = vertx.setPeriodic(1000, id -> {
      System.out.println(System.currentTimeMillis() + ": And every second this is printed");
    });

    System.out.println(System.currentTimeMillis() + ": First this is printed");

输出

	1489715918135: First this is printed
	1489715919135: And every second this is printed
	1489715920136: And every second this is printed
	1489715921136: And every second this is printed
	1489715922136: And every second this is printed
	1489715923136: And every second this is printed
	1489715924136: And every second this is printed

**取消任务**

    long timerID = vertx.setPeriodic(1000, id -> {
      System.out.println(System.currentTimeMillis() + ": And every second this is printed");
    });

    System.out.println(System.currentTimeMillis() + ": First this is printed");

    vertx.setTimer(3000, id -> {
      vertx.cancelTimer(timerID);
    });

输出

	1489716150895: First this is printed
	1489716151895: And every second this is printed
	1489716152895: And every second this is printed
	1489716153895: And every second this is printed

# 原理

不管是setTimer还是setPeriodic，都是调用的scheduleTimeout方法来实现定时任务

	public long setTimer(long delay, Handler<Long> handler) {
		return scheduleTimeout(getOrCreateContext(), handler, delay, false);
	}

	public long setPeriodic(long delay, Handler<Long> handler) {
		return scheduleTimeout(getOrCreateContext(), handler, delay, true);
	}

scheduleTimeout方法会生产一个自增的定时器ID`long timerId = timeoutCounter.getAndIncrement();`，保存在一个Map对象timeouts里`timeouts.put(timerId, task);`，创建一个定时任务`InternalTimerHandler task`，同时给Vert.x的关闭注册一个关闭任务的钩子函数`context.addCloseHook(task);`

	  private final AtomicLong timeoutCounter = new AtomicLong(0);
	  private final ConcurrentMap<Long, InternalTimerHandler> timeouts = new ConcurrentHashMap<>();
	
	  private long scheduleTimeout(ContextImpl context, Handler<Long> handler, long delay, boolean periodic) {
	    if (delay < 1) {
	      throw new IllegalArgumentException("Cannot schedule a timer with delay < 1 ms");
	    }
	    long timerId = timeoutCounter.getAndIncrement();
	    InternalTimerHandler task = new InternalTimerHandler(timerId, handler, periodic, delay, context);
	    timeouts.put(timerId, task);
	    context.addCloseHook(task);
	    return timerId;
	  }

## InternalTimerHandler
下面看一下定时任务`InternalTimerHandler`的实现.InternalTimerHandler的逻辑很简单，他最终将定时任务委托给EventLoop的定时任务来执行`Runnable toRun = () -> context.runOnContext(this);`。周期性任务：`scheduleAtFixedRate`，定时任务：`schedule`

    InternalTimerHandler(long timerID, Handler<Long> runnable, boolean periodic, long delay, ContextImpl context) {
      this.context = context;
      this.timerID = timerID;
      this.handler = runnable;
      this.periodic = periodic;
      EventLoop el = context.nettyEventLoop();
      Runnable toRun = () -> context.runOnContext(this);
      if (periodic) {
        future = el.scheduleAtFixedRate(toRun, delay, delay, TimeUnit.MILLISECONDS);
      } else {
        future = el.schedule(toRun, delay, TimeUnit.MILLISECONDS);
      }
      metrics.timerCreated(timerID);
    }

InternalTimerHandler实现了`Handler<Void>, Closeable`两个接口.

Closeable的实现会从`timeouts`中删除任务，并且通过cancel方法将任务取消，cancel方法最终会调用`ScheduledFuture`的取消方法.后面描述

    // Called via Context close hook when Verticle is undeployed
    public void close(Handler<AsyncResult<Void>> completionHandler) {
      VertxImpl.this.timeouts.remove(timerID);
      cancel();
      completionHandler.handle(Future.succeededFuture());
    }

    boolean cancel() {
      metrics.timerEnded(timerID, true);
      return future.cancel(false);
    }

Handler<Void>的实现会执行定时任务`handler.handle(timerID);`，如果任务不是周期性任务，还需要执行以下清理工作，从`timeouts`中删除任务，删除对应的钩子函数`context.removeCloseHook(this);`

    public void handle(Void v) {
      try {
        handler.handle(timerID);
      } finally {
        if (!periodic) {
          // Clean up after it's fired
          cleanupNonPeriodic();
        }
      }
    }

    private void cleanupNonPeriodic() {
      VertxImpl.this.timeouts.remove(timerID);
      metrics.timerEnded(timerID, false);
      ContextImpl context = getContext();
      if (context != null) {
        context.removeCloseHook(this);
      }
    }

## AbstractScheduledEventExecutor
对应AbstractScheduledEventExecutor我们只阅读与定时器有关的内容，其他部分会在EventLoop章节描述

`ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit)`和
`ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) `都会将runnable封装成一个定时任务ScheduledFutureTask，交由`schedule`方法执行

    @Override
    public  ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
        ObjectUtil.checkNotNull(command, "command");
        ObjectUtil.checkNotNull(unit, "unit");
        if (delay < 0) {
            throw new IllegalArgumentException(
                    String.format("delay: %d (expected: >= 0)", delay));
        }
        return schedule(new ScheduledFutureTask<Void>(
                this, command, null, ScheduledFutureTask.deadlineNanos(unit.toNanos(delay))));
    }

    @Override
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
        ObjectUtil.checkNotNull(command, "command");
        ObjectUtil.checkNotNull(unit, "unit");
        if (initialDelay < 0) {
            throw new IllegalArgumentException(
                    String.format("initialDelay: %d (expected: >= 0)", initialDelay));
        }
        if (period <= 0) {
            throw new IllegalArgumentException(
                    String.format("period: %d (expected: > 0)", period));
        }

        return schedule(new ScheduledFutureTask<Void>(
                this, Executors.<Void>callable(command, null),
                ScheduledFutureTask.deadlineNanos(unit.toNanos(initialDelay)), unit.toNanos(period)));
    }

**ScheduledFutureTask**

ScheduledFutureTask有三个属性:

- id 任务的ID，通过一个全局的nextTaskId自增
- deadlineNanos 任务的执行时间
- periodNanos 任务的周期性时间，0表示不重复、大于0表示按固定的周期重复、小于0表示按固定的延时重复

    private static final AtomicLong nextTaskId = new AtomicLong();
    private static final long START_TIME = System.nanoTime();

    private final long id = nextTaskId.getAndIncrement();
    private long deadlineNanos;
    /* 0 - no repeat, >0 - repeat at fixed rate, <0 - repeat with fixed delay */
    private final long periodNanos;

    static long nanoTime() {
        return System.nanoTime() - START_TIME;
    }

    static long deadlineNanos(long delay) {
        return nanoTime() + delay;
    }

创建一次性定时任务时，periodNans=0,deadlineNanos=当前时间+延时时间

	new ScheduledFutureTask<Void>(
                this, command, null, ScheduledFutureTask.deadlineNanos(unit.toNanos(delay)))

创建周期性定时任务时，periodNans=周期,deadlineNanos=当前时间+延时时间

	new ScheduledFutureTask<Void>(
                this, Executors.<Void>callable(command, null),
                ScheduledFutureTask.deadlineNanos(unit.toNanos(initialDelay)), unit.toNanos(period))

ScheduledFutureTask实现了JDK的Delayed接口，按照deadlineNanos来计算任务的出队顺序

	public long delayNanos() {
		return Math.max(0, deadlineNanos() - nanoTime());
	}
	
	public long delayNanos(long currentTimeNanos) {
		return Math.max(0, deadlineNanos() - (currentTimeNanos - START_TIME));
	}
	
	@Override
	public long getDelay(TimeUnit unit) {
		return unit.convert(delayNanos(), TimeUnit.NANOSECONDS);
	}
	
	@Override
	public int compareTo(Delayed o) {
		if (this == o) {
			return 0;
		}
	
		ScheduledFutureTask<?> that = (ScheduledFutureTask<?>) o;
		long d = deadlineNanos() - that.deadlineNanos();
		if (d < 0) {
			return -1;
		} else if (d > 0) {
			return 1;
		} else if (id < that.id) {
			return -1;
		} else if (id == that.id) {
			throw new Error();
		} else {
			return 1;
		}
	}

run方法用来调用定时任务，一次性任务直接调用任务函数

    if (setUncancellableInternal()) {
        V result = task.call();
        setSuccessInternal(result);
    }

在任务执行之前，需要先将任务标识为不可取消`setUncancellableInternal`，这部分代码在EventLoop中在做阅读

如果是周期性定时任务，在执行完任务函数之后，需要重新计算下一次任务的deadlineNanos，deadlineNanos的计算规则：

- 如果是按固定的周期重复，直接在deadlineNanos上加上周期就得到最新的deadlineNanos
- 如果是按固定的延时重复，在当前时间上加上周期就得到最新的deadlineNanos，**按照约定负数表示延时重复，所以这里是`nanoTime() - p`来计算得到的deadlineNanos** 

在计算完deadlineNanos之后，会将任务加入到一个优先级队列中` PriorityQueue scheduledTaskQueue`

    // check if is done as it may was cancelled
    if (!isCancelled()) {
        task.call();
        if (!executor().isShutdown()) {
            long p = periodNanos;
            if (p > 0) {
                deadlineNanos += p;
            } else {
                deadlineNanos = nanoTime() - p;
            }
            if (!isCancelled()) {
                // scheduledTaskQueue can never be null as we lazy init it before submit the task!
                Queue<ScheduledFutureTask<?>> scheduledTaskQueue =
                        ((AbstractScheduledEventExecutor) executor()).scheduledTaskQueue;
                assert scheduledTaskQueue != null;
                scheduledTaskQueue.add(this);
            }
        }
    }

cancel方法用来取消任务,当任务取消时，他会调用Executor的removeScheduled方法将任务从优先级队列中删除

    public boolean cancel(boolean mayInterruptIfRunning) {
        boolean canceled = super.cancel(mayInterruptIfRunning);
        if (canceled) {
            ((AbstractScheduledEventExecutor) executor()).removeScheduled(this);
        }
        return canceled;
    }

    boolean cancelWithoutRemove(boolean mayInterruptIfRunning) {
        return super.cancel(mayInterruptIfRunning);
    }

    final void removeScheduled(final ScheduledFutureTask<?> task) {
        if (inEventLoop()) {
            scheduledTaskQueue().remove(task);
        } else {
            execute(new Runnable() {
                @Override
                public void run() {
                    removeScheduled(task);
                }
            });
        }
    }

在scheduleXXX方法创建完ScheduledFutureTask之后，会通过schedule方法将任务存入优先级队列

**ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) **

	   <V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
	        if (inEventLoop()) {
	            scheduledTaskQueue().add(task);
	        } else {
	            execute(new Runnable() {
	                @Override
	                public void run() {
	                    scheduledTaskQueue().add(task);
	                }
	            });
	        }
	
	        return task;
	    }