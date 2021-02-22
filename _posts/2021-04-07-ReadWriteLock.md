---
layout: post
title: 读写锁
date: 2020-04-07
categories:
    - 多线程
comments: true
permalink: ReadWriteLock.html
---

在没有读写锁之前，我们假设使用普通的 ReentrantLock，那么虽然我们保证了线程安全，但是也浪费了一定的资源，因为如果多个读操作同时进行，其实并没有线程安全问题，我们可以允许让多个读操作并行，以便提高程序效率。
但是写操作不是线程安全的，如果多个线程同时写，或者在写的同时进行读操作，便会造成线程安全问题。

我们的读写锁就解决了这样的问题，它设定了一套规则，既可以保证多个线程同时读的效率，同时又可以保证有写入操作时的线程安全。
整体思路是它有两把锁，第 1 把锁是写锁，获得写锁之后，既可以读数据又可以修改数据，而第 2 把锁是读锁，获得读锁之后，只能查看数据，不能修改数据。读锁可以被多个线程同时持有，所以多个线程可以同时查看数据。
在读的地方合理使用读锁，在写的地方合理使用写锁，灵活控制，可以提高程序的执行效率。

# 1. 读写锁的获取规则

我们在使用读写锁时遵守下面的获取规则：

- 如果有一个线程已经占用了读锁，则此时其他线程如果要申请读锁，可以申请成功。
- 如果有一个线程已经占用了读锁，则此时其他线程如果要申请写锁，则申请写锁的线程会一直等待释放读锁，因为读写不能同时操作。
- 如果有一个线程已经占用了写锁，则此时其他线程如果申请写锁或者读锁，都必须等待之前的线程释放写锁，同样也因为读写不能同时，并且两个线程不应该同时写。

所以我们用一句话总结：要么是一个或多个线程同时有读锁，要么是一个线程有写锁，但是两者不会同时出现。也可以总结为：读读共享、其他都互斥（写写互斥、读写互斥、写读互斥）。

Reader 线程正在读取，Writer 线程正在等待

![](/assets/images/posts/synchronized/synchronized-12.jpg)

Writer 线程正在写入，Reader 线程正在等待

![](/assets/images/posts/synchronized/synchronized-13.jpg)

相比于 ReentrantLock 适用于一般场合，ReadWriteLock 适用于读多写少的情况，合理使用可以进一步提高并发效率。

# 2. 读锁的插队策略

ReentrantReadWriteLock的Sync实现中中定义了两个抽象方法
```
/*
 * Acquires and releases use the same code for fair and
 * nonfair locks, but differ in whether/how they allow barging
 * when queues are non-empty.
 */

/**
 * Returns true if the current thread, when trying to acquire
 * the read lock, and otherwise eligible to do so, should block
 * because of policy for overtaking other waiting threads.
 */
abstract boolean readerShouldBlock();

/**
 * Returns true if the current thread, when trying to acquire
 * the write lock, and otherwise eligible to do so, should block
 * because of policy for overtaking other waiting threads.
 */
abstract boolean writerShouldBlock();
```

FairSync的实现

```
final boolean writerShouldBlock() {
  return hasQueuedPredecessors();
}
final boolean readerShouldBlock() {
  return hasQueuedPredecessors();
}
```

**在公平锁的情况下，只要等待队列中有线程在等待，也就是 hasQueuedPredecessors() 返回 true 的时候，那么 writer 和 reader 都会 block，也就是一律不允许插队，都乖乖去排队，这也符合公平锁的思想。**

NonfairSync的实现

```
final boolean writerShouldBlock() {
  return false; // writers can always barge
}
final boolean readerShouldBlock() {
  /* As a heuristic to avoid indefinite writer starvation,
   * block if the thread that momentarily appears to be head
   * of queue, if one exists, is a waiting writer.  This is
   * only a probabilistic effect since a new reader will not
   * block if there is a waiting writer behind other enabled
   * readers that have not yet drained from the queue.
   */
  return apparentlyFirstQueuedIsExclusive();
}
```

在 writerShouldBlock() 这个方法中始终返回 false，可以看出，对于想获取写锁的线程而言，由于返回值是 false，所以它是随时可以插队的，这就和我们的 ReentrantLock 的设计思想是一样的，但是读锁却不一样。

假设线程 2 和线程 4 正在同时读取，线程 3 想要写入，但是由于线程 2 和线程 4 已经持有读锁了，所以线程 3 就进入等待队列进行等待。此时，线程 5 突然跑过来想要插队获取读锁：

面对这种情况有两种应对策略：

- **允许插队** 由于现在有线程在读，而线程 5 又不会特别增加它们读的负担，因为线程们可以共用这把锁，所以第一种策略就是让线程 5 直接加入到线程 2 和线程 4 一起去读取。这种策略看上去增加了效率，但是有一个严重的问题，那就是如果想要读取的线程不停地增加，比如线程 6，那么线程  6 也可以插队，这就会导致读锁长时间内不会被释放，导致线程 3 长时间内拿不到写锁，也就是那个需要拿到写锁的线程会陷入“饥饿”状态，它将在长时间内得不到执行。

- **不允许插队** 这种策略认为由于线程 3 已经提前等待了，所以虽然线程 5 如果直接插队成功，可以提高效率，但是我们依然让线程 5 去排队等待：按照这种策略线程 5 会被放入等待队列中，并且排在线程 3 的后面，让线程 3 优先于线程 5 执行，这样可以避免“饥饿”状态，这对于程序的健壮性是很有好处的，直到线程 3 运行完毕，线程 5 才有机会运行，这样谁都不会等待太久的时间。

**所以我们可以看出，即便是非公平锁，只要等待队列的头结点是尝试获取写锁的线程，那么读锁依然是不能插队的，目的是避免“饥饿”。**

验证代码

```
public class ReadLockJumpQueue {

    private static final ReentrantReadWriteLock reentrantReadWriteLock = new ReentrantReadWriteLock();
    private static final ReentrantReadWriteLock.ReadLock readLock = reentrantReadWriteLock.readLock();
    private static final ReentrantReadWriteLock.WriteLock writeLock = reentrantReadWriteLock.writeLock();

    private static void read() {
        readLock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "得到读锁，正在读取");
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            readLock.unlock();
            System.out.println(Thread.currentThread().getName() + "释放读锁");
        }
    }

    private static void write() {
        writeLock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "得到写锁，正在写入");
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            writeLock.unlock();
            System.out.println(Thread.currentThread().getName() + "释放写锁");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> read(), "Thread-2").start();
        new Thread(() -> read(), "Thread-4").start();
        new Thread(() -> write(), "Thread-3").start();
        new Thread(() -> read(), "Thread-5").start();
    }
}
```

# 3. 锁降级

锁降级的目的是保证本线程中对data修改了之后，在释放写锁之后数据仍然是一致的，即其他线程是不能获取写锁的。

> 锁降级是为了让当前线程感知到数据的变化。

ReentrantReadWriteLock的javadoc中有一段示例

```
public class CachedData {
 
    Object data;
    volatile boolean cacheValid;
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 
    void processCachedData() {
        rwl.readLock().lock();
        if (!cacheValid) {
            //在获取写锁之前，必须首先释放读锁。
            rwl.readLock().unlock();
            rwl.writeLock().lock();
            try {
                //这里需要再次判断数据的有效性,因为在我们释放读锁和获取写锁的空隙之内，可能有其他线程修改了数据。
                if (!cacheValid) {
                    data = new Object();
                    cacheValid = true;
                }
                //在不释放写锁的情况下，直接获取读锁，这就是读写锁的降级。
                rwl.readLock().lock();
            } finally {
                //释放了写锁，但是依然持有读锁
                rwl.writeLock().unlock();
            }
        }
 
        try {
            System.out.println(data);
        } finally {
            //释放读锁
            rwl.readLock().unlock();
        }
    }
}
```

如果先释放写锁，再释放读锁，可能在获取之前会有其他线程获取到写锁，阻塞读锁的获取，当前线程就无法感知数据的变化了，所以要先持有写锁保证数据无变化，在获取读锁，然后释放写锁

**不支持锁升级**

```
final static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 
public static void main(String[] args) {
    upgrade();
}
 
public static void upgrade() {
    rwl.readLock().lock();
    System.out.println("获取到了读锁");
    rwl.writeLock().lock();
    System.out.println("成功升级");
}
```

这段代码会打印出“获取到了读锁”，但是却不会打印出“成功升级”，因为 ReentrantReadWriteLock 不支持读锁升级到写锁。

我们知道读写锁的特点是如果线程都申请读锁，是可以多个线程同时持有的，可是如果是写锁，只能有一个线程持有，并且不可能存在读锁和写锁同时持有的情况。

正是因为不可能有读锁和写锁同时持有的情况，所以升级写锁的过程中，需要等到所有的读锁都释放，此时才能进行升级。

假设线程 A 和 B 都想升级到写锁，那么对于线程 A 而言，它需要等待其他所有线程，包括线程 B 在内释放读锁。而线程 B 也需要等待所有的线程，包括线程 A 释放读锁。这就是一种非常典型的死锁的情况。谁都愿不愿意率先释放掉自己手中的锁。
