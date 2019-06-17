---
layout: post
title: 按顺序处理同一用户的消息
date: 2019-05-11
categories:
    - 消息队列
comments: true
permalink: Processing-messages-in-sequence.html
---

消息队列是应用开发的常用工具, 也是系统解耦的必备利器。保证同一用户的消息按照顺序处理是应用的常见需求，  譬如在微博应用中， 发表微博、删除微博这两个操作必须按序处理，乱序势必造成业务逻辑错误。

如何保证消息处理顺序？以下是常见的几种做法。

**设计一**： 单线程处理。 虽然单线程处理非常简单好用，但是单线程限制了系统吞吐率。

**设计二**： 每个用户一个队列。看起来很美，实际上基本不可行。首先，每个消息队列基本都没啥消息， 其次， 不可能为每个用户安排一个线程。

**设计三**：  静态消息分发。系统设计分发线程和工作线程，每工作线程设置一个线程私有的消息队列，工作线程从私有消息队列取消息，执行业务逻辑。假定工作线程个数为N,  分发线程从全局的消息队列取消息，推消息到第 Hash(用户名）%N个线程私有队列。  静态消息分发一般来说够用，但也存在一些弱点。 首先存在伪冲突，隶属于同一个私有队列的消息按序处理，慢消息将阻塞私有队列中的后续消息，所以无法适用于实时性要求较高的应用。 其次，全局阻塞问题。线程私有队列慢时，分发线程停顿 ，造成全局停顿，几乎无法接受。

![](/assets/images/posts/queue/user_queue1.png)

**设计四**： 动态路由消息队列。 设计一种新的消息队列，支持以下功能：（1）路由。工作线程取消息时，只能取出不在路由表中的消息， 取出消息之后，登记到路由表。（2）ACK。工作线程处理完消息之后， 调用ACK接口，消息队列删除消息，并清理路由表。从动态路由消息队列取消息，即不存在为冲突，也不存在全局阻塞， 恰好能解决问题。

![](/assets/images/posts/queue/user_queue2.png)

设计四的简易实现

```
public class SequentialQueue<E> {
  /**
   * 任务列表
   */
  private final LinkedList<E> tasks = new LinkedList<>();

  /**
   * 消息注册表，用于确保同一个设备只有一个事件在执行
   */
  private final Set<String> registry = new ConcurrentSkipListSet<>();

  private final IdentificationExtractor<E> extractor;

  private final int limit;

  private E takeElement;

  public SequentialQueue(IdentificationExtractor<E> extractor, int limit) {
    this.extractor = extractor;
    this.limit = limit;
  }

  public SequentialQueue(IdentificationExtractor<E> extractor) {
    this(extractor, Integer.MAX_VALUE);
  }

  private E next() {
    E next = null;
    for (E event : tasks) {
      String id = extractor.apply(event);
      if (!registry.contains(id)) {
        next = event;
        break;
      }
    }
    return next;
  }

  public synchronized E dequeue() throws InterruptedException {
    //如果队列为空，或者下一个出队元素为null，阻塞出队
    while (tasks.isEmpty() || takeElement == null) {
      wait();
    }
    //唤醒等待入队的线程，如果队满，说明可能会有入队线程在等待唤醒(不满不会有入队线程等待唤醒)
    if (this.tasks.size() == this.limit) {
      //唤醒入队，入队线程会在出队方法执行完毕并释放锁之后才开始抢占锁
      notifyAll();
    }
    //从队列中删除元素
    E x = takeElement;
    tasks.remove(x);
    //将元素加入注册表
    registry.add(extractor.apply(x));
    //重新计算下一个可以出队的元素
    takeElement = next();
    return x;
  }

  public synchronized void enqueue(E task) throws InterruptedException {
    //如果队满，阻塞入队
    while (this.tasks.size() == this.limit) {
      wait();
    }
    //唤醒等待出队的线程，如果队空或者下一个出队元素为null，说明可能会有出队线程在等待唤醒
    if (tasks.isEmpty() || takeElement == null) {
      //唤醒出队
      notifyAll();
    }
    //入队
    tasks.add(task);
    //计算下一个可以出队的元素
    if (!registry.contains(extractor.apply(task))
        && takeElement == null) {
      takeElement = task;
    }
  }

  public synchronized void complete(E task) {
    if (takeElement == null) {
      notifyAll();
    }
    registry.remove(extractor.apply(task));
    takeElement = next();
  }

  public synchronized int size() {
    return tasks.size();
  }

}
```


PS：在分布式环境下，可以用Redis做注册表