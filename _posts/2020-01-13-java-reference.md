---
layout: post
title: Java reference
date: 2020-01-13
categories:
    - JVM
comments: true
permalink: java-reference.html
---

Java中一共有四种引用类型, 强引用(StrongReference), 软引用(SoftReference), 弱引用(WeakReference), 虚引用(PhantomReference); 

**强引用(Strong Reference)**

代码中普遍存在的，`Object obj = new Object()` 所创建的引用，只要强引用存在，垃圾收集器就永远不会回收被引用对象。当内存不足的时候，jvm 就会抛出 OutOfMemory 错误，而不会回收强引用的对象。

**软引用(Sofe Reference)**

有用但并非必须的对象，可用`SoftReference`类来实现软引用，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行二次回收。如果这次回收还没有足够的内存，才会抛出内存异常。

在该对象被垃圾回收器回收之前，通过`SoftReference`类的get()方法可以获得该Java对象的强引用，一旦该对象被回收之后，get()就会返回null。

**弱引用(Weak Reference)**

被弱引用关联的对象只能生存到下一次垃圾收集发生之前，JDK提供了`WeakReference`类来实现弱引用。

**虚引用(Phantom Reference)**

也称为幽灵引用或幻影引用，是最弱的一种引用关系，JDK提供了`PhantomReference`类来实现虚引用。一个持有虚引用的对象和几乎没有引用是一样的，随时可能被垃圾回收器所回收。虚引用的get方法总是会返回null。为了追踪垃圾回收过程，虚引用必须和引用队列一起使用。

```java
Object obj = new Object();
ReferenceQueue<Object> refQueue =new ReferenceQueue<>();
PhantomReference<Object> phanRef =new PhantomReference<>(obj, refQueue);

Object objg = phanRef.get();
//这里拿到的是null
System.out.println(objg);
//让obj变成垃圾
obj=null;
System.gc();
Thread.sleep(3000);
//gc后会将phanRef加入到refQueue中
Reference<? extends Object> phanRefP = refQueue.remove();
//这里输出true
System.out.println(phanRefP==phanRef);
```

从以上代码中可以看到，虚引用能够在指向对象不可达时得到一个'通知'（其实所有继承References的类都有这个功能），需要注意的是GC完成后，phanRef.referent依然指向之前创建Object，也就是说Object对象一直没被回收！**对于虚引用来说，从refQueue.remove();得到引用对象后，可以调用clear方法强行解除引用和对象之间的关系，使得对象下次可以GC时可以被回收掉。*8


# 参考资料


