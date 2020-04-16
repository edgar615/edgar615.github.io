---
layout: post
title: Vert.x Locker
date: 2019-03-14
categories:
    - Vert.x
comments: true
permalink: vertx-lock.html
---

# Lock

在eventloop中使用同步锁会阻塞eventloop线程，降低吞吐量，因此Vert.x的sharedData提供了异步锁的实现

示例　 

    SharedData sharedData = vertx.sharedData();
    sharedData.getLock("mylock", res -> {
      if (res.succeeded()) {
        // Got the lock!
        Lock lock = res.result();
        System.out.println("op1:" + System.currentTimeMillis());
        // 5 seconds later we release the lock so someone else can get it

        vertx.setTimer(5000, tid -> lock.release());

      } else {
        // Something went wrong
      }
    });

    sharedData.getLock("mylock", res -> {
      if (res.succeeded()) {
        // Got the lock!
        Lock lock = res.result();
        System.out.println("op2:" + System.currentTimeMillis());
        // 5 seconds later we release the lock so someone else can get it

        vertx.setTimer(5000, tid -> lock.release());

      } else {
        // Something went wrong
      }
    });

    sharedData.getLockWithTimeout("mylock", 1000, res -> {
      if (res.succeeded()) {
        // Got the lock!
        Lock lock = res.result();
        System.out.println("op3:" + System.currentTimeMillis());
        // 5 seconds later we release the lock so someone else can get it

        vertx.setTimer(5000, tid -> lock.release());

      } else {
        // Something went wrong
        System.out.println("error");
      }
    });

输出

	op1:1501123285127
	error
	op2:1501123290128

可以看到op2比op1晚了5秒钟才执行，op3在指定的时间没有取得锁，返回失败

## 实现

getLock直接调用getLockWithTimeout，默认的超时时间为10秒，在单机模式下，会通过getLocalLock方法来获取锁，在集群模式下，会通过clusterManager.getLockWithTimeout来获取锁。

	  @Override
	  public void getLock(String name, Handler<AsyncResult<Lock>> resultHandler) {
	    Objects.requireNonNull(name, "name");
	    Objects.requireNonNull(resultHandler, "resultHandler");
	    getLockWithTimeout(name, DEFAULT_LOCK_TIMEOUT, resultHandler);
	  }
	
	  @Override
	  public void getLockWithTimeout(String name, long timeout, Handler<AsyncResult<Lock>> resultHandler) {
	    Objects.requireNonNull(name, "name");
	    Objects.requireNonNull(resultHandler, "resultHandler");
	    Arguments.require(timeout >= 0, "timeout must be >= 0");
	    if (clusterManager == null) {
	      getLocalLock(name, timeout, resultHandler);
	    } else {
	      clusterManager.getLockWithTimeout(name, timeout, resultHandler);
	    }
	  }

我们现在只阅读getLocalLock方法，集群的锁以后在阅读。

在SharedData的内部使用一个map来存放所有的lock

	private final ConcurrentMap<String, AsynchronousLock> localLocks = new ConcurrentHashMap<>();

getLocalLock方法从这个map中取到相应的AsynchronousLock

	  private void getLocalLock(String name, long timeout, Handler<AsyncResult<Lock>> resultHandler) {
	    AsynchronousLock lock = localLocks.computeIfAbsent(name, n -> new AsynchronousLock(vertx));
	    lock.acquire(timeout, resultHandler);
	  }

### Lock接口

异步互斥锁的接口，它只定义了一个release方法，用来释放锁

public interface Lock {

  /**
   * Release the lock. Once the lock is released another will be able to obtain the lock.
   */
  void release();
}

###  AsynchronousLock
AsynchronousLock是Lock的具体实现类，它在内部使用一个队列来保存对锁的请求。owned属性用来标识锁是否被获取

	  private final Vertx vertx;
	  private final Queue<LockWaiter> waiters = new LinkedList<>();
	  private boolean owned;

在前面getLocalLock方法获取到AsynchronousLock对象之后，会调用这个对象的acquire方法来获取锁。

	  public void acquire(long timeout, Handler<AsyncResult<Lock>> resultHandler) {
	    Context context = vertx.getOrCreateContext();
	    doAcquire(context, timeout, resultHandler);
	  }
	
acquire调用doAcquire方法，这个方法是一个同步方法。如果`owned==false`标明这个锁还没有被获取，可以获取这个锁。然后通过lockAcquired方法回调resultHandler;如果未获取到锁，则加入等待队列

	  public void doAcquire(Context context, long timeout, Handler<AsyncResult<Lock>> resultHandler) {
	    synchronized (this) {
	      if (!owned) {
	        // We now have the lock
	        owned = true;
	        lockAcquired(context, resultHandler);
	      } else {
	        waiters.add(new LockWaiter(this, context, timeout, resultHandler));
	      }
	    }
	  }
	
	  private void lockAcquired(Context context, Handler<AsyncResult<Lock>> resultHandler) {
	    context.runOnContext(v -> resultHandler.handle(Future.succeededFuture(this)));
	  }

**这里使用context.runOnContext来获取锁是因为获取锁的请求可能来自不同的线程，不同的Context，所以release可能是由与doAcquire不同的线程和context来触发，需要使用context.runOnContext使请求回到原来的context上**

**release方法**
release方法每次从等待队列中取出一个请求，然后尝试获取锁。如果队列中没有数据，将owned标识为false

	  public synchronized void release() {
	    LockWaiter waiter = pollWaiters();
	    if (waiter != null) {
	      waiter.acquire(this);
	    } else {
	      owned = false;
	    }
	  }

### LockWaiter
LockWaiter的实现比较简单，在它的构造方法中创建一个延时任务检查获取锁的请求是否已经超时，如果是超时的请求直接返回失败

    LockWaiter(AsynchronousLock lock, Context context, long timeout, Handler<AsyncResult<Lock>> resultHandler) {
      this.lock = lock;
      this.context = context;
      this.resultHandler = resultHandler;
      if (timeout != Long.MAX_VALUE) {
        context.owner().setTimer(timeout, tid -> timedOut());
      }
    }

    void timedOut() {
      synchronized (lock) {
        if (!acquired) {
          timedOut = true;
          context.runOnContext(v -> resultHandler.handle(Future.failedFuture(new VertxException("Timed out waiting to get lock"))));
        }
      }
    }

acquire方法直接通过lock的lockAcquired方法来获取锁。等待队列中存在数据就已经说明owned=true，所以不需要再将owned设置为true，只需要在队列为空时将owned修改为false即可

    void acquire(AsynchronousLock lock) {
      acquired = true;
      lock.lockAcquired(context, resultHandler);
    }


**所有的lock都是由localLocks这个map保存，但是sharedData并未提供这个map的清理方法，如果lock的键值有很多时，对内存占用会一直得不到释放**