---
layout: post
title: 线程
date: 2019-06-25
categories:
    - java,多线程
comments: true
permalink: java-concurrency-thread.html
---

这个系列以前的学习笔记，计划陆续整理搬运到这里

> 大部分资料来源与:
>
> 《java并发变成实战》
>
> http://tutorials.jenkov.com/java-concurrency/index.html
>
> http://tutorials.jenkov.com/java-util-concurrent/index.html

# 线程的属性

- ID: 每个线程的独特标识。
- Name: 线程的名称。
- Priority: 线程对象的优先级。优先级别在1-10之间，1是最低级，10是最高级。不建议改变它们的优先级，但是你想的话也是可以的。
- Status: 线程的状态。在Java中，线程只能有这6种中的一种状态： new, runnable, blocked, waiting, time waiting, 或 terminated.

# 线程的生命周期
## 通用的线程生命周期

![](/assets/images/posts/thread/thread-state.png)

- New：初始态，指的是线程已经被创建，但是还不允许分配 CPU 执行。这个状态属于编程语言特有的，不过这里所谓的被创建，仅仅是在编程语言层面被创建，而在操作系统层面，真正的线程还没有创建。
- Ready：可运行态， 指的是线程可以分配 CPU 执行。在这种状态下，真正的操作系统线程已经被成功创建了，所以可以分配 CPU 执行。
- Running：运行态，当有空闲的 CPU 时，操作系统会将其分配给一个处于可运行状态的线程，被分配到 CPU 的线程的状态就转换成了运行状态。
- Waiting：休眠态，运行状态的线程如果调用一个阻塞的 API（例如以阻塞方式读文件）或者等待某个事件（例如条件变量），那么线程的状态就会转换到休眠状态，同时释放 CPU 使用权，休眠状态的线程永远没有机会获得 CPU 使用权。当等待的事件出现了，线程就会从休眠状态转换到可运行状态。
- Terminated：终止态，线程执行完或者出现异常就会进入终止状态，终止状态的线程不会切换到其他任何状态，进入终止状态也就意味着线程的生命周期结束了。

这五种状态在不同编程语言里会有简化合并。例如，C 语言的 POSIX Threads 规范，就把初始状态和可运行状态合并了；Java 语言里则把可运行状态和运行状态合并了，这两个状态在操作系统调度层面有用，而 JVM 层面不关心这两个状态，因为 JVM 把线程调度交给操作系统处理了。

## Java的线程生命周期
JVM暴露的线程状态和OS底层的线程状态是两个不同层面的事情，在Thread 类下的 State 内部枚举类中所定义的状态：

- NEW：初始化状态
- RUNNABLE：可运行 / 运行状态
- BLOCKED：阻塞状态
- WAITING：无时限等待
- TIMED_WAITING：有时限等待
- TERMINATED：终止状态

![](/assets/images/posts/thread/java-thread-state.png)

在操作系统层面，Java 线程中的 BLOCKED、WAITING、TIMED_WAITING 是一种状态，即前面我们提到的休眠状态。也就是说只要 Java 线程处于这三种状态之一，那么这个线程就永远没有 CPU 的使用权。同时，**java线程中没有RUNNING状态**。

### new 新建状态
使用 new 关键字和 Thread 类或其子类建立一个线程对象后，该线程对象就处于新建状态。它保持这个状态直到程序 start() 这个线程。
> 这里的创建仅仅是在JAVA的这种编程语言层面被创建，而在操作系统层面，真正的线程还没有被创建。

```
Thread t = new Thread()
```

### runnable 可运行 / 运行状态
当线程对象调用了start()方法之后，该线程就进入runnable状态。就绪状态的线程处于就绪队列中，要等待JVM里线程调度器的调度

> 这时候，线程已经在操作系统层面被创建

```
t.start()
```

对 Java 线程状态而言，不存在所谓的running 状态，它的 runnable 状态包含了 running 状态。

> Thread state for a runnable thread.  A thread in the runnable state is executing in the Java virtual machine but it may be waiting for other resources from the operating system such as processor.

javadoc中这样描述：一个线程在JVM中执行就是处于Runnable状态，但是这个线程可能在等待(Processor处理器)资源。

现在的操作系统架构通常都是用时间分片的方式进行抢占式轮转调度。这个时间分片通常是很小的，一个线程一次最多只能在 cpu 上运行比如10-20ms 的时间（此时处于 running 状态），也即大概只有0.01秒这一量级，时间片用后就要被切换下来放入调度队列的末尾等待再次调度（即回到 ready 状态）。

> 如果期间进行了 I/O 的操作还会导致提前释放时间分片，并进入等待队列。又或者是时间分片没有用完就被抢占，这时也是回到 ready 状态。

这一切换的过程称为线程的上下文切换，当然 cpu 不是简单地把线程踢开就完了，还需要把被相应的执行状态保存到内存中以便后续的恢复执行。如果不计切换开销（每次在1ms 以内），相当于1秒内有50-100次切换。事实上时间片经常没用完，线程就因为各种原因被中断，实际发生的切换次数还会更多。

通常，Java的线程状态是服务于监控的，如果线程切换得是如此之快，那么区分 ready 与 running 就没什么太大意义了：当你看到监控上显示是 running 时，对应的线程可能早就被切换下去了，甚至又再次地切换了上来，也许你只能看到 ready 与 running 两个状态在快速地闪烁。

现今主流的 JVM 实现都把 Java 线程一一映射到操作系统底层的线程上，把调度委托给了操作系统，我们在虚拟机层面看到的状态实质是对底层状态的映射及包装。JVM 本身没有做什么实质的调度，把底层的 ready 及 running 状态映射上来也没多大意义，因此，统一成为runnable 状态是不错的选择。

### blocked同步阻塞
表示线程阻塞，等待获取锁，如碰到synchronized、lock等关键字等占用临界区的情况，一旦获取到锁就进行RUNNABLE状态继续运行

### waiting无限等待阻塞
表示线程处于无限制等待状态，等待一个特殊的事件来重新唤醒

- 通过wait()方法进行等待的线程等待一个notify()或者notifyAll()方法
- 通过join()方法进行等待的线程等待目标线程运行结束而唤醒

一旦通过相关事件唤醒线程，线程就进入了RUNNABLE状态继续运行。

### timed_waiting有时限等待阻塞
表示线程进入了一个有时限的等待，如sleep(3000)，等待3秒后线程重新进行RUNNABLE状态继续运行。

### terminated终止状态
线程执行完（run()方法执行结束）或者者其他终止条件发生(异常、取消)就会进入终止状态

注意：一旦线程通过start方法启动后就再也不能回到初始NEW状态，线程终止后也不能再回到RUNNABLE状态。

# 再说RUNNABLE 
我们知道传统的I/O都是阻塞式（blocked）的，相对于CPU来说，磁盘的速度太慢了，如果CPU要等到IO操作完成，很可能时间片都用完了，I/O 操作还没完成，那CPU的利用率就非常低。为了避免这样的问题，OS识别到线程A在执行IO相关指令，对应的线程A就会立马被切换到waiting状态，然后从ready队列中获取一个新的线程B来进行操作。此时A线程就处于所谓的“阻塞”，被放到等待队列中，也即：waiting状态。

而当 I/O 完成时，则用一种叫中断（interrupt）的机制来通知 cpu：也即所谓的“中断驱动（interrupt-driven）”，现代操作系统基本都采用这一机制。cpu 会收到一个来自硬盘的中断信号，并进入中断处理例程，手头正在执行的线程B因此被打断，回到 ready 队列。而先前因 I/O 而waiting 的线程A随着 I/O 的完成也再次回到调度队列，这时 cpu 可能会选择它来执行。

> 时间分片轮转本质上也是由一个定时器定时中断来驱动的，可以使线程从 running 回到 ready 状态。

![](/assets/images/posts/thread/thread-state-2.png)

进行阻塞式 I/O 操作时，Java 的线程状态是什么？

下面是一个测试代码，
```
@Test
public void testInBlockedIOState() throws InterruptedException {
Scanner in = new Scanner(System.in);
// 创建一个名为“输入输出”的线程t
Thread t = new Thread(new Runnable() {
  @Override
  public void run() {
	try {
	  // 命令行中的阻塞读
	  String input = in.nextLine();
	  System.out.println(input);
	} catch (Exception e) {
	  e.printStackTrace();
	} finally {
	  in.close();
	}
  }
}, "输入输出"); // 线程的名字

// 启动
t.start();

// 确保run已经得到执行
TimeUnit.SECONDS.sleep(1);

// 状态为RUNNABLE
Assert.assertEquals(t.getState(), Thread.State.RUNNABLE);
}
```
如果我们打上断点，可以在debug中看到线程状态

![](/assets/images/posts/thread/java-thread-state-2.png)

**线程调用阻塞式 API 时，Java 的线程状态依然是RUNNABLE**

JVM 并不关心底层的实现细节，什么时间分片也好，什么 IO 时就要切换也好，它并不关心。**一个线程在JVM中执行就是处于Runnable状态，但是这个线程可能在等待(Processor处理器)资源**。Java 线程状态的改变通常只与自身显式引入的机制有关，如果 JVM 中的线程状态发生改变了，通常是自身机制引发的

所以runnable状态实际上对应了OS层面的Ready，Running以及部分的 Waiting 状态

![](/assets/images/posts/thread/java-thread-state-3.png)

# 状态切换

![](/assets/images/posts/thread/java-thread-state-4.png)

## RUNNABLE 与 BLOCKED 的状态转换
只有一种场景会触发这种转换，就是线程等待 synchronized 的隐式锁。synchronized 修饰的方法、代码块同一时刻只允许一个线程执行，其他线程只能等待，这种情况下，等待的线程就会从 RUNNABLE 转换到 BLOCKED 状态。而当等待的线程获得 synchronized 隐式锁时，就又会从 BLOCKED 转换到 RUNNABLE 状态。

在操作系统层面，线程会转换到休眠状态的，但是在 JVM 层面，Java 线程的状态不会发生变化，也就是说 Java 线程的状态会依然保持 RUNNABLE 状态。JVM 层面并不关心操作系统调度相关的状态，因为在 JVM 看来，等待 CPU 使用权（操作系统层面此时处于可执行状态）与等待 I/O（操作系统层面此时处于休眠状态）没有区别，都是在等待某个资源，所以都归入了 RUNNABLE 状态。而我们平时所谓的 Java 在调用阻塞式 API 时，线程会阻塞，指的是操作系统线程的状态，并不是 Java 线程的状态。

## RUNNABLE 与 WAITING 的状态转换

- 获得 synchronized 隐式锁的线程调用了无参数的 Object.wait() 方法
- 调用无参数的 Thread.join() 方法。其中的 join() 是一种线程同步方法，例如有一个线程对象 thread A，当调用 A.join() 的时候，执行这条语句的线程会等待 thread A 执行完，而等待中的这个线程，其状态会从 RUNNABLE 转换到 WAITING。当线程 thread A 执行完，原来等待它的线程又会从 WAITING 状态转换到 RUNNABLE
- 调用 LockSupport.park() 方法。调用 LockSupport.park() 方法，当前线程会阻塞，线程的状态会从 RUNNABLE 转换到 WAITING。调用 LockSupport.unpark(Thread thread) 可唤醒目标线程，目标线程的状态又会从 WAITING 状态转换到 RUNNABLE。

## RUNNABLE 与 TIMED_WAITING 的状态转换
- 调用带超时参数的 Thread.sleep(long millis) 方法；
- 获得 synchronized 隐式锁的线程，调用带超时参数的 Object.wait(long timeout) 方法；
- 调用带超时参数的 Thread.join(long millis) 方法；
- 调用带超时参数的 LockSupport.parkNanos(Object blocker, long deadline) 方法；
- 调用带超时参数的 LockSupport.parkUntil(long deadline) 方法。

## RUNNABLE 到 TERMINATED 状态
线程执行完 run() 方法后，会自动转换到 TERMINATED 状态，当然如果执行 run() 方法的时候异常抛出，也会导致线程终止。我们有可以通过`interrupt() `方法强制中断 run() 方法的执行。`interrupt()` 方法仅仅是通知线程，线程有机会执行一些后续操作，同时也可以无视这个通知。

被 interrupt 的线程，是怎么收到通知的？一种是异常，另一种是主动检测。

- 只有当线程 A 处于 WAITING、TIMED_WAITING 状态时，如果其他线程调用线程 A 的` interrupt()`方法，会使线程 A 返回到 RUNNABLE 状态(中间可能存在BLOCKED状态)，同时线程 A 的代码会触发 InterruptedException 异常。上面我们提到转换到 WAITING、TIMED_WAITING 状态的触发条件，都是调用了类似 `wait()`、`join()`、`sleep()` 这样的方法，我们看这些方法的签名，发现都会 `throws InterruptedException` 这个异常。这个异常的触发条件就是：其他线程调用了该线程的 interrupt() 方法。
- 当线程 A 处于 RUNNABLE 状态时，并且阻塞在` java.nio.channels.InterruptibleChannel` 上时，如果其他线程调用线程 A 的 `interrupt()`方法，线程 A 会触发`java.nio.channels.ClosedByInterruptException`这个异常；而阻塞在` java.nio.channels.Selector`上时，如果其他线程调用线程 A 的 `interrupt()`方法，线程 A 的` java.nio.channels.Selector`会立即返回。

上面这两种情况属于被中断的线程通过异常的方式获得了通知。还有一种是主动检测，如果线程处于 RUNNABLE 状态，并且没有阻塞在某个 I/O 操作上，例如中断计算圆周率的线程 A，这时就得依赖线程 A 主动检测中断状态了。如果其他线程调用线程 A 的 interrupt() 方法，那么线程 A 可以通过 isInterrupted() 方法，检测是不是自己被中断了。

# 常用方法
## sleep()
暂停阻塞等待一段时间，时间过了就继续。

## yield()
暂停当前正在执行的线程对象，让别的就绪状态线程运行（切换），也会等待阻塞一段时间，但是时间不是由客户控制了

## wait()
也是阻塞和等待，但是需要notify来唤醒。

## notify()、notifyAll()
唤醒线程

##  join()
在一个线程中调用other.join(),将等待other执行完后才继续本线程

- join() 暂停当前线程终止，等待该线程终止
- join(long millis) 等待该线程终止的时间最长为 millis 毫秒。
- join(long millis, int nanos) 等待该线程终止的时间最长为 millis 毫秒 + nanos 纳秒。

测试示例

```
public class DataSourcesLoader implements Runnable {


	/**
     * Main method of the class
	 */
	@Override
	public void run() {
		
		// Writes a messsage
		System.out.printf("Begining data sources loading: %s\n",new Date());
		// Sleeps four seconds
		try {
			TimeUnit.SECONDS.sleep(4);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		// Writes a message
		System.out.printf("Data sources loading has finished: %s\n",new Date());
	}
}
```

```
public class NetworkConnectionsLoader implements Runnable {


	/**
     * Main method of the class
	 */
	@Override
	public void run() {
		// Writes a message
		System.out.printf("Begining network connections loading: %s\n",new Date());
		// Sleep six seconds
		try {
			TimeUnit.SECONDS.sleep(6);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		// Writes a message
		System.out.printf("Network connections loading has finished: %s\n",new Date());
	}
}
```

```
    public static void main(String[] args) {

        // Creates and starts a DataSourceLoader runnable object
        DataSourcesLoader dsLoader = new DataSourcesLoader();
        Thread thread1 = new Thread(dsLoader,"DataSourceThread");
        thread1.start();

        // Creates and starts a NetworkConnectionsLoader runnable object
        NetworkConnectionsLoader ncLoader = new NetworkConnectionsLoader();
        Thread thread2 = new Thread(ncLoader,"NetworkConnectionLoader");
        thread2.start();

        // Wait for the finalization of the two threads
        try {
            thread1.join();
            thread2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // Waits a message
        System.out.printf("Main: Configuration has been loaded: %s\n",new Date());
    }
```
输出
```
Begining network connections loading: Tue Jun 25 13:44:18 CST 2019
Begining data sources loading: Tue Jun 25 13:44:18 CST 2019
Data sources loading has finished: Tue Jun 25 13:44:22 CST 2019
Network connections loading has finished: Tue Jun 25 13:44:24 CST 2019
Main: Configuration has been loaded: Tue Jun 25 13:44:24 CST 2019
```

## interrupt()
`Thread.interrupt()`方法仅仅是通知线程，线程有机会执行一些后续操作，同时也可以无视这个通知，这个方法通过修改了调用线程的中断状态来告知那个线程，说它被中断了，线程可以通过isInterrupted() 方法，检测是不是自己被中断。

当线程被阻塞的时候，比如被Object.wait, Thread.join和Thread.sleep三种方法之一阻塞时；调用它的interrput()方法，会产生InterruptedException异常。

> 后面准备专门用一篇文章详细讲解

## setDaemon()

Java有一种特别的线程叫做守护线程。这种线程的优先级非常低，通常在程序里没有其他线程运行时才会执行它。当守护线程是程序里唯一在运行的线程时，JVM会结束守护线程并终止程序。

根据这些特点，守护线程通常用于在同一程序里给普通线程（也叫使用者线程）提供服务。它们通常无限循环的等待服务请求或执行线程任务。它们不能做重要的任务，因为我们不知道什么时候会被分配到CPU时间片，并且只要没有其他线程在运行，它们可能随时被终止。JAVA中最典型的这种类型代表就是垃圾回收器。

- setDaemon(boolean on)  将该线程标记为守护线程或用户线程。

```
public class WriterTask implements Runnable {

	Deque<Event> deque;
	
	public WriterTask (Deque<Event> deque){
		this.deque=deque;
	}
	
	@Override
	public void run() {
		
		// Writes 100 events
		for (int i=1; i<100; i++) {
			// Creates and initializes the Event objects 
			Event event=new Event();
			event.setDate(new Date());
			event.setEvent(String.format("The thread %s has generated an event",Thread.currentThread().getId()));
			
			// Add to the data structure
			deque.addFirst(event);
			try {
				// Sleeps during one second
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```

```
public class CleanerTask implements Runnable {
    private Deque<Event> deque;

    public CleanerTask(Deque<Event> deque) {
        this.deque = deque;
    }

    @Override
    public void run() {
        while (true) {
            Date date = new Date();
            clean(date);
        }
    }

    private void clean(Date date) {
        long difference;
        boolean delete;

        if (deque.size()==0) {
            return;
        }

        delete=false;
        do {
            Event e = deque.getLast();
            difference = date.getTime() - e.getDate().getTime();
            if (difference > 10000) {
                System.out.printf("Cleaner: %s\n",e.getEvent());
                deque.removeLast();
                delete=true;
            }
        } while (difference > 10000);
        if (delete){
            System.out.printf("Cleaner: Size of the queue: %d\n",deque.size());
        }
    }
}
```

```
public static void main(String[] args) {

    Deque<Event> deque=new ArrayDeque<Event>();

    // Creates the three WriterTask and starts them
    WriterTask writer=new WriterTask(deque);
    for (int i=0; i<3; i++){
        Thread thread=new Thread(writer);
        thread.start();
    }

    // Creates a cleaner task and starts them
    CleanerTask cleaner=new CleanerTask(deque);
    Thread cleanerThread = new Thread(cleaner);
    cleanerThread.setDaemon(true);
    cleanerThread.start();
}
```

## setUncaughtExceptionHandler

`setUncaughtExceptionHandler(Thread.UncaughtExceptionHandler eh)`设置该线程由于未捕获到异常而突然终止时调用的处理程序。

```
public class ExceptionHandler implements Thread.UncaughtExceptionHandler {


	@Override
	public void uncaughtException(Thread t, Throwable e) {
		System.out.printf("An exception has been captured\n");
		System.out.printf("Thread: %s\n",t.getId());
		System.out.printf("Exception: %s: %s\n",e.getClass().getName(),e.getMessage());
		System.out.printf("Stack Trace: \n");
		e.printStackTrace(System.out);
		System.out.printf("Thread status: %s\n",t.getState());
	}

}
```

```
public static void main(String[] args) {
    Task task=new Task();
    Thread thread=new Thread(task);
    thread.setUncaughtExceptionHandler(new ExceptionHandler());
    thread.start();
    try {
        thread.join();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.printf("Thread has finished\n");

}
```

# 从JVM角度理解线程

> 这部分全部来自 [聊聊JVM（五）从JVM角度理解线程](https://blog.csdn.net/ITer_ZC/article/details/41825395) 看完之后对线程有了更深的了解，所以在这里备份

我们知道JVM主要是用C++实现的，JVM定义的Thread的类继承结构如下:

Class hierarchy
 - Thread
   - NamedThread
     - VMThread
     - ConcurrentGCThread
     - WorkerThread
       - GangWorker
       - GCTaskThread
   - JavaThread
   - WatcherThread

另外还有一个重要的类**OSThread**不在这个继承关系里，它以组合的方式被Thread类所使用

![](/assets/images/posts/thread/jvm-thread-1.png)

这些类构成了JVM的线程模型，其中最主要的是下面几个类：

- java.lang.Thread: 这个是Java语言里的线程类，由这个Java类创建的instance都会 1:1 映射到一个操作系统的osthread
- JavaThread: JVM中C++定义的类，一个JavaThread的instance代表了在JVM中的java.lang.Thread的instance, 它维护了线程的状态，并且维护一个指针指向java.lang.Thread创建的对象(oop)。它同时还维护了一个指针指向对应的OSThread，来获取底层操作系统创建的osthread的状态
- OSThread: JVM中C++定义的类，代表了JVM中对底层操作系统的osthread的抽象，它维护着实际操作系统创建的线程句柄handle，可以获取底层osthread的状态
- VMThread: JVM中C++定义的类，这个类和用户创建的线程无关，是JVM本身用来进行虚拟机操作的线程，比如GC。

有两种方式可以让用户在JVM中创建线程

1. new java.lang.Thread().start()
2. 使用JNI将一个native thread attach到JVM中

针对 new java.lang.Thread().start()这种方式，只有调用start()方法的时候，才会真正的在JVM中去创建线程，主要的生命周期步骤有：

1. 创建对应的JavaThread的instance
2. 创建对应的OSThread的instance
3. 创建实际的底层操作系统的native thread
4. 准备相应的JVM状态，比如ThreadLocal存储空间分配等
5. 底层的native thread开始运行，调用java.lang.Thread生成的Object的run()方法
6. 当java.lang.Thread生成的Object的run()方法执行完毕返回后,或者抛出异常终止后，终止native thread
7. 释放JVM相关的thread的资源，清除对应的JavaThread和OSThread

针对JNI将一个native thread attach到JVM中，主要的步骤有：

1. 通过JNI call AttachCurrentThread申请连接到执行的JVM实例
2. JVM创建相应的JavaThread和OSThread对象
3. 创建相应的java.lang.Thread的对象
4. 一旦java.lang.Thread的Object创建之后，JNI就可以调用Java代码了
5. 当通过JNI call DetachCurrentThread之后，JNI就从JVM实例中断开连接
6. JVM清除相应的JavaThread, OSThread, java.lang.Thread对象

从JVM的角度来看待线程状态的状态有以下几种:

![](/assets/images/posts/thread/jvm-thread-2.png)

其中主要的状态是这5种:

- _thread_new: 新创建的线程
- _thread_in_Java: 在运行Java代码
- _thread_in_vm: 在运行JVM本身的代码
- _thread_in_native: 在运行native代码
- _thread_blocked: 线程被阻塞了，包括等待一个锁，等待一个条件，sleep，执行一个阻塞的IO等

从OSThread的角度，JVM还定义了一些线程状态给外部使用，比如用jstack输出的线程堆栈信息中线程的状态:

![](/assets/images/posts/thread/jvm-thread-3.png)

比较常见有:

- Runnable: 可以运行或者正在运行的
- MONITOR_WAIT: 等待锁
- OBJECT_WAIT: 执行了Object.wait()之后在条件队列中等待的
- SLEEPING: 执行了Thread.sleep()的

从JavaThread的角度，JVM定义了一些针对Java Thread对象的状态，基本类似，多了一个TIMED_WAITING的状态，用来表示定时阻塞的状态

![](/assets/images/posts/thread/jvm-thread-4.png)

最后来看一下JVM内部的VM Threads，主要由几类:

- VMThread: 执行JVM本身的操作
- Periodic task thread: JVM内部执行定时任务的线程
- GC threads: GC相关的线程，比如单线程/多线程的GC收集器使用的线程
- Compiler threads: JIT用来动态编译的线程
- Signal dispatcher thread: Java解释器Interceptor用来辅助safepoint操作的线程

# 参考资料

https://segmentfault.com/a/1190000018723860

https://mp.weixin.qq.com/s/U5FH139XmfH1XTRf-_WJ_w
