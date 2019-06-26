---
layout: post
title: java并发系列-线程
date: 2019-06-24
categories:
    - java
comments: true
permalink: java-concurrency-thread.html
---

这个系列以前的学习笔记，计划陆续整理搬运到这里

> 大部分资料来源与:
> 《java并发变成实战》
> http://tutorials.jenkov.com/java-concurrency/index.html
> http://tutorials.jenkov.com/java-util-concurrent/index.html

# 线程的属性

- ID: 每个线程的独特标识。
- Name: 线程的名称。
- Priority: 线程对象的优先级。优先级别在1-10之间，1是最低级，10是最高级。不建议改变它们的优先级，但是你想的话也是可以的。
- Status: 线程的状态。在Java中，线程只能有这6种中的一种状态： new, runnable, blocked, waiting, time waiting, 或 terminated.

# 线程的生命周期

![](/assets/images/posts/thread/thread-state.png)

## new 新建状态
使用 new 关键字和 Thread 类或其子类建立一个线程对象后，该线程对象就处于新建状态。它保持这个状态直到程序 start() 这个线程。
> 这里的创建仅仅是在JAVA的这种编程语言层面被创建，而在操作系统层面，真正的线程还没有被创建。

```
Thread t = new Thread()
```
## runnable 就绪状态:
当线程对象调用了start()方法之后，该线程就进入就绪状态。就绪状态的线程处于就绪队列中，要等待JVM里线程调度器的调度
> 这时候，线程已经在操作系统层面被创建

```
t.start()
```
## running 运行状态
当CPU空闲时，就绪状态的线程被分得CPU时间片，就可以执行 run()，此时线程便处于运行状态。处于运行状态的线程最为复杂，它可以变为阻塞状态、就绪状态和死亡状态。

## blocked同步阻塞
表示线程阻塞，等待获取锁，如碰到synchronized、lock等关键字等占用临界区的情况，一旦获取到锁就进行RUNNABLE状态继续运行

## waiting无限等待阻塞
表示线程处于无限制等待状态，等待一个特殊的事件来重新唤醒

- 通过wait()方法进行等待的线程等待一个notify()或者notifyAll()方法
- 通过join()方法进行等待的线程等待目标线程运行结束而唤醒

一旦通过相关事件唤醒线程，线程就进入了RUNNABLE状态继续运行。

## timed_waiting有时限等待阻塞
表示线程进入了一个有时限的等待，如sleep(3000)，等待3秒后线程重新进行RUNNABLE状态继续运行。

## terminated终止状态
线程执行完（run()方法执行结束）或者者其他终止条件发生(异常、取消)就会进入终止状态

注意：一旦线程通过start方法启动后就再也不能回到初始NEW状态，线程终止后也不能再回到RUNNABLE状态。

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


# 参考资料

https://segmentfault.com/a/1190000018723860
