---
layout: post
title:  多线程（2）-wait notify
date: 2019-06-02
categories:
    - 多线程
comments: true
permalink: java-concurrency-wait-notify.html
---

# 1. 轮询
多线程之间的通信可以通过轮询的方式实现

> 循环执行某个逻辑判断，直到判断条件为true才执行判断体中的逻辑，叫做轮询(Polling)，也可以叫做忙等待。轮询是会浪费一定的CPU资源的。


```
public class BusyWaitRunnable implements Runnable {
    private MySignal signal;

    public BusyWaitRunnable(MySignal signal) {
        this.signal = signal;
    }

    @Override
    public void run() {
        while (signal.hasDataToProcess()) {
            //do something
        }
    }
}
```
忙等待没有对运行等待线程的CPU进行有效的利用，除非平均等待时间非常短。否则，让等待线程进入睡眠或者非运行状态更为明智，直到它接收到它等待的信号。

# 2. wait(),notify()和notifyAll()
Java有一个内建的等待机制来允许线程在等待信号的时候变为非运行状态。java.lang.Object 类定义了三个方法，`wait()`、`notify()`和`notifyAll()`来实现这个等待机制。

一个线程一旦调用了任意对象的`wait()`方法，就会变为非运行状态，直到另一个线程调用了同一个对象的`notify()`方法。为了调用`wait()`或者`notify()`，线程必须先获得那个对象的锁。也就是说，线程必须在同步块里调用`wait()`或者`notify()`

```
public class MyWaitNotify {

    private MonitorObject myMonitorObject  = new MonitorObject();

    public void doWait() throws InterruptedException {
        synchronized (myMonitorObject) {
            myMonitorObject.wait();
        }
        //do something
    }

    public void doNotify() {
        synchronized (myMonitorObject) {
            myMonitorObject.notifyAll();
        }
    }
    
}
```
当一个线程调用一个对象的`notify()`方法，正在等待该对象的所有线程中将有一个线程被唤醒并允许执行（**这个将被唤醒的线程是随机的，不可以指定唤醒哪个线程**）。同时也提供了一个`notifyAll()`方法来唤醒正在等待一个给定对象的所有线程。

> wait()和notify()必须在同步块中调用，一个线程如果没有持有对象锁，将不能调用wait()，notify()或者notifyAll()。否则，会抛出IllegalMonitorStateException异常。

一旦线程调用了wait()方法，它就释放了所持有的监视器对象上的锁。这将允许其他线程也可以调用wait()或者notify()。

一旦一个线程被唤醒，不能立刻就退出wait()的方法调用，直到调用notify()的线程退出了它自己的同步块。换句话说：被唤醒的线程必须重新获得监视器对象的锁，才可以退出wait()的方法调用，因为wait方法调用运行在同步块里面。如果多个线程被notifyAll()唤醒，那么在同一时刻将只有一个线程可以退出wait()方法，因为每个线程在退出wait()前必须获得监视器对象的锁。

# 3. 丢失的信号
但是上面的代码有一个问题：

> notify()和notifyAll()方法不会保存调用它们的方法，因为当这两个方法被调用时，有可能没有线程处于等待状态。通知信号过后便丢弃了。因此，如果一个线程先于被通知线程调用wait()前调用了notify()，等待的线程将错过这个信号。这可能是也可能不是个问题。不过，在某些情况下，这可能使等待线程永远在等待，不再醒来，因为线程错过了唤醒信号。

为了避免丢失信号，必须把它们保存在信号类里。在MyWaitNotify的例子中，通知信号应被存储在MyWaitNotify实例的一个成员变量里

```
public class MyWaitNotify {

    private MonitorObject myMonitorObject  = new MonitorObject();

    boolean wasSignalled = false;

    public void doWait() throws InterruptedException {
        synchronized (myMonitorObject) {
            if (!wasSignalled) {
                myMonitorObject.wait();
            }
            wasSignalled = false;
        }
        //do something
    }

    public void doNotify() {
        synchronized (myMonitorObject) {
            wasSignalled = true;
            myMonitorObject.notifyAll();
        }
    }
    
}
```
为了避免信号丢失， 用一个变量来保存是否被通知过。在notify前，设置自己已经被通知过。在wait后，设置自己没有被通知过，需要等待通知。

# 4. 假唤醒
由于莫名其妙的原因，线程有可能在没有调用过notify()和notifyAll()的情况下醒来。这就是所谓的假唤醒（spurious wakeups）。无端端地醒过来了。

如果在MyWaitNotify的doWait()方法里发生了假唤醒，等待线程即使没有收到正确的信号，也能够执行后续的操作。这可能导致你的应用程序出现严重问题。

为了防止假唤醒，保存信号的成员变量将在一个while循环里接受检查，而不是在if表达式里。这样的一个while循环叫做自旋锁（这种做法要慎重，目前的JVM实现自旋会消耗CPU，如果长时间不调用doNotify方法，doWait方法会一直自旋，CPU会消耗太大）。被唤醒的线程会自旋直到自旋锁(while循环)里的条件变为false

```
public class MyWaitNotify {

    private MonitorObject myMonitorObject  = new MonitorObject();

    boolean wasSignalled = false;

    public void doWait() throws InterruptedException {
        synchronized (myMonitorObject) {
            while (!wasSignalled) {
                myMonitorObject.wait();
            }
            wasSignalled = false;
        }
        //do something
    }

    public void doNotify() {
        synchronized (myMonitorObject) {
            wasSignalled = true;
            myMonitorObject.notifyAll();
        }
    }


}
```

# 5. 为什么wait，notify要在同步块代码中使用

我们先看下面的代码

```
public class MyWaitNotify {

    private MonitorObject myMonitorObject  = new MonitorObject();

    public void doWait() throws InterruptedException {
		while (count <= 0) {
			myMonitorObject.wait();
		}
		count --;//这里是非原子性
        //do something
    }

    public void doNotify() {
        count ++;
		myMonitorObject.notifyAll();
	
    }

}
```

假设当前`count=0`，`doWait`方法检查到`count <= 0`，进入循环执行`wait`方法之前发生了线程上下文切换，此时`doNotify方法`执行`notifyAll`方法，由于`myMonitorObject`尚未调用`wait`，此时发出的notify就丢失了。这就是所谓的lost wake up问题。所以需要将wait、notify放在同步块中使用

# 6. 为什么 wait/notify/notifyAll 被定义在 Object 类中，而 sleep 定义在 Thread 类中？

因为 Java 中每个对象都有一把称之为 monitor 监视器的锁，由于每个对象都可以上锁，这就要求在对象头中有一个用来保存锁信息的位置。这个锁是对象级别的，而非线程级别的，wait/notify/notifyAll 也都是锁级别的操作，它们的锁属于对象，所以把它们定义在 Object 类中是最合适，因为 Object 类是所有对象的父类。

因为如果把 wait/notify/notifyAll 方法定义在 Thread 类中，会带来很大的局限性，比如一个线程可能持有多把锁，以便实现相互配合的复杂逻辑，假设此时 wait 方法定义在 Thread 类中，如何实现让一个线程持有多把锁呢？又如何明确线程等待的是哪把锁呢？既然我们是让当前线程去等待某个对象的锁，自然应该通过操作对象来实现，而不是操作线程。

# 7. wait/notify 和 sleep 方法的异同？
wait/notify 和 sleep 方法的异同，主要对比 wait 和 sleep 方法，我们先说相同点：

- 它们都可以让线程阻塞。
- 它们都可以响应 interrupt 中断：在等待的过程中如果收到中断信号，都可以进行响应，并抛出 InterruptedException 异常。

但是它们也有很多的不同点：

- wait 方法必须在 synchronized 保护的代码中使用，而 sleep 方法并没有这个要求。
- 在同步代码中执行 sleep 方法时，并不会释放 monitor 锁，但执行 wait 方法时会主动释放 monitor 锁。
- sleep 方法中会要求必须定义一个时间，时间到期后会主动恢复，而对于没有参数的 wait 方法而言，意味着永久等待，直到被中断或被唤醒才能恢复，它并不会主动恢复。
- wait/notify 是 Object 类的方法，而 sleep 是 Thread 类的方法。

# 8. 小结

1. wait()、notify/notifyAll() 方法是Object的本地final方法，无法被重写。
2. wait()使当前线程阻塞，前提是 必须先获得锁，一般配合synchronized 关键字使用，即，一般在synchronized 同步代码块里使用 wait()、notify/notifyAll() 方法。
3. 由于 wait()、notify/notifyAll() 在synchronized 代码块执行，说明当前线程一定是获取了锁的。
**当线程执行wait()方法时候，会释放当前的锁，然后让出CPU，进入等待状态。只有当 notify/notifyAll() 被执行时候，才会唤醒一个或多个正处于等待状态的线程，然后继续往下执行，直到执行完synchronized 代码块的代码或是中途遇到wait() ，再次释放锁。也就是说，notify/notifyAll() 的执行只是唤醒沉睡的线程，而不会立即释放锁，锁的释放要看代码块的具体执行情况。所以在编程中，尽量在使用了notify/notifyAll() 后立即退出临界区，以唤醒其他线程 **
4. wait() 需要被try catch包围，中断也可以使wait等待的线程唤醒。
5. notify 和wait 的顺序不能错，如果A线程先执行notify方法，B线程在执行wait方法，那么B线程是无法被唤醒的。
6. notify 和 notifyAll的区别：notify方法只唤醒一个等待（对象的）线程并使该线程开始执行。所以如果有多个线程等待一个对象，这个方法只会唤醒其中一个线程，选择哪个线程取决于操作系统对多线程管理的实现。notifyAll 会唤醒所有等待(对象的)线程，尽管哪一个线程将会第一个处理取决于操作系统的实现。如果当前情况下有多个线程需要被唤醒，推荐使用notifyAll 方法。比如在生产者-消费者里面的使用，每次都需要唤醒所有的消费者或是生产者，以判断程序是否可以继续往下执行。
7. 在多线程中要测试某个条件的变化，使用if 还是while？要注意，notify唤醒沉睡的线程后，线程会接着上次的执行继续往下执行。所以在进行条件判断时候，可以先把 wait 语句忽略不计来进行考虑，显然，要确保程序一定要执行，并且要保证程序直到满足一定的条件再执行，要使用while来执行，以确保条件满足和一定执行。
8. 在wait()/notify()机制中，不要使用全局对象，字符串常量等。应该使用对应唯一的对象

# 9. 参考资料

http://ifeve.com/thread-signaling/
