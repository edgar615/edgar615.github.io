---
layout: post
title: ThreadLocal对象
date: 2019-11-22
categories:
    -  多线程
comments: true
permalink: threadlocal.html
---

ThreadLocal是什么？我们先看JDK中的描述

> 该类提供了线程局部 (thread-local) 变量。这些变量不同于它们的普通对应物，因为访问某个变量（通过其get 或 set方法）的每个线程都有自己的局部变量，它独立于变量的初始化副本。ThreadLocal实例通常是类中的 private static 字段，它们希望将状态与某一个线程（例如，用户 ID 或事务 ID）相关联。

所以ThreadLocal与线程同步机制不同，线程同步机制是多个线程共享同一个变量，而ThreadLocal是为每一个线程创建一个单独的变量副本，故而每个线程都可以独立地改变自己所拥有的变量副本，而不会影响其他线程所对应的副本。可以说ThreadLocal为多线程环境下变量问题提供了另外一种解决思路。

**ThreadLocal不解决多线程共享变量的问题**

ThreadLocal定义了四个方法：

- get()：返回此线程局部变量的当前线程副本中的值。
- initialValue()：返回此线程局部变量的当前线程的“初始值”。
- remove()：移除此线程局部变量当前线程的值。
- set(T value)：将此线程局部变量的当前线程副本中的值设置为指定值。

这四个方法使用比较简单，并不是文章的重点。

我们先看一下`get`方法

```
public T get() {
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null) {
		ThreadLocalMap.Entry e = map.getEntry(this);
		if (e != null) {
			@SuppressWarnings("unchecked")
			T result = (T)e.value;
			return result;
		}
	}
	return setInitialValue();
}
ThreadLocalMap getMap(Thread t) {
	return t.threadLocals;
}

public class Thread implements Runnable {
	...
    ThreadLocal.ThreadLocalMap threadLocals = null;
	...
}
```

我们可以看到在每个线程中都有一个`ThreadLocalMap`对象，这个对象的key值就是`ThreadLocal`对象自己。通过上面的源码可以看到每个线程的`ThreadLocalMap`默认是null，如果`get`方法判断`ThreadLocalMap`是null，则会通过`setInitialValue`方法调用`createMap`初始化

```
private T setInitialValue() {
	T value = initialValue();
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null)
		map.set(this, value);
	else
		createMap(t, value);
	return value;
}
void createMap(Thread t, T firstValue) {
	t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

`set`方法的过程和get类似，下面我们把关注的重点转移到`ThreadLocalMap`

# 1. ThreadLocalMap
我们先看一下`ThreadLocalMap`的成员变量

```
static class ThreadLocalMap {

	static class Entry extends WeakReference<ThreadLocal<?>> {
		/** The value associated with this ThreadLocal. */
		Object value;

		Entry(ThreadLocal<?> k, Object v) {
			super(k);
			value = v;
		}
	}

	private static final int INITIAL_CAPACITY = 16;

	private Entry[] table;

	private int size = 0;

	private int threshold; // Default to 0
```

- `Entry`类继承了`WeakReference<ThreadLocal<?>>`，即每个`Entry`对象都有一个`ThreadLocal`的弱引用（作为key），这是为了防止内存泄露。一旦线程结束，key变为一个不可达的对象，这个Entry就可以被GC了。
- `INITIAL_CAPACITY ` ThreadLocalMap 的初始容量，必须为2的倍数，默认为16

- `Entry[] table` resized时候需要的table
- `size` table中的entry个数

- `threshold` 扩容数值

再看看它的构造函数

```
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
	table = new Entry[INITIAL_CAPACITY];
	int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
	table[i] = new Entry(firstKey, firstValue);
	size = 1;
	setThreshold(INITIAL_CAPACITY);
}
```
构造函数先先创建一个长度为16的`Entry`数组，然后计算出firstKey（实际是`ThreadLocal`自己）对应的哈希值，然后存储到table中，并设置size和threshold。这里我们需要重点关注计算hash的部分`int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);`

每个ThreadLocal对象在创建的时候都会初始化一个`threadLocalHashCode`常量，

```
private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode =
	new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
	return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

**ThreadLocalMap中解决Hash冲突的方式并非像HashMap一样用链表的方式，而是采用线性探测的方式。所谓线性探测，就是根据初始key的hashcode值确定元素在table数组中的位置，如果发现这个位置上已经有其他key值的元素被占用，则利用固定的算法寻找一定步长的下个位置，依次判断，直至找到能够存放的位置。**

`HASH_INCREMENT = 0x61c88647`就是每次增加的步长`1640531527`，根据JAVADOC所说，选择这个数字是为了让冲突概率最小

> The difference between successively generated hash codes - turns implicit sequential thread-local IDs into near-optimally spread multiplicative hash values for power-of-two-sized tables.

**因为`threadLocalHashCode`初始化时依赖一个共享变量`nextHashCode`，所以每个ThreadLocal对象l的`threadLocalHashCode`都不同**

为了便于理解，我们采用一组简单的数据模拟 ThreadLocal.set() 的过程是如何解决 Hash 冲突的。

- threadLocalHashCode = 4，threadLocalHashCode & 15 = 4；此时数据应该放在数组下标为 4 的位置。下标 4 的位置正好没有数据，可以存放。

- threadLocalHashCode = 19，threadLocalHashCode & 15 = 4；但是下标 4 的位置已经有数据了，如果当前需要添加的 Entry 与下标 4 位置已存在的 Entry 两者的 key 相同，那么该位置 Entry 的 value 将被覆盖为新的值。我们假设 key 都是不相同的，所以此时需要向后移动一位，下标 5 的位置没有冲突，可以存放。

- threadLocalHashCode = 33，threadLocalHashCode & 15 = 3；下标 3 的位置已经有数据，向后移一位，下标 4 位置还是有数据，继续向后查找，发现下标 6 没有数据，可以存放。

ThreadLocal.get() 的过程也是类似的，也是根据 threadLocalHashCode 的值定位到数组下标，然后判断当前位置 Entry 对象与待查询 Entry 对象的 key 是否相同，如果不同，继续向下查找。由此可见，**ThreadLocal.set()/get() 方法在数据密集时很容易出现 Hash 冲突，需要 O(n) 时间复杂度解决冲突问题，效率较低。**

我们在看`ThreadLocal.get()`方法调用的`ThreadLocalMap.Entry e = map.getEntry(this);`获取线程内副本变量的代码

```
private Entry getEntry(ThreadLocal<?> key) {
	int i = key.threadLocalHashCode & (table.length - 1);
	Entry e = table[i];
	if (e != null && e.get() == key)
		return e;
	else
		return getEntryAfterMiss(key, i, e);
}
```

首先根据`ThreadLocal`计算在table数组中的位置，如果这个位置存储的正好是当前`ThreadLocal`对象，那么直接返回，否则要通过`getEntryAfterMiss`方法查找下一个位置

```
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
	Entry[] tab = table;
	int len = tab.length;

	while (e != null) {
		ThreadLocal<?> k = e.get();
		if (k == key)
			return e;
		if (k == null)
			expungeStaleEntry(i);
		else
			i = nextIndex(i, len);
		e = tab[i];
	}
	return null;
}
```

`getEntryAfterMiss`通过一个while循环，不停的在table数组中查找`ThreadLocal`对象，

- 如果找到就返回；
- 如果当前位置没有Entry，通过`expungeStaleEntry`方法将键和值为 null 的 Entry 设置为 null 从而使得该 Entry 可被回收。通过这种方式，ThreadLocal 可防止内存泄漏；
- 如果当前位置存在冲突，继续查找下一个位置`i = nextIndex(i, len);`

```
private static int nextIndex(int i, int len) {
	return ((i + 1 < len) ? i + 1 : 0);
}
```

看完`get`方法，再看`set`方法就比较容易理解了
```
private void set(ThreadLocal<?> key, Object value) {

	// We don't use a fast path as with get() because it is at
	// least as common to use set() to create new entries as
	// it is to replace existing ones, in which case, a fast
	// path would fail more often than not.

	Entry[] tab = table;
	int len = tab.length;
	int i = key.threadLocalHashCode & (len-1);

	for (Entry e = tab[i];
		 e != null;
		 e = tab[i = nextIndex(i, len)]) {
		ThreadLocal<?> k = e.get();

		if (k == key) {
			e.value = value;
			return;
		}

		if (k == null) {
			replaceStaleEntry(key, value, i);
			return;
		}
	}

	tab[i] = new Entry(key, value);
	int sz = ++size;
	if (!cleanSomeSlots(i, sz) && sz >= threshold)
		rehash();
}
```
set方法先去根据hash找到对应的的Entry位置i

- 如果i位置的Entry是null的话，让i指向new出来的Entry
- 如果i位置的Entry不为空，判断Entry的key是不是和传入的key相同，如果相同覆盖value返回。
- 如果Entry的key是空的话，进行替换过期Entry的操作（大概就是继续找到和key相同key的Entry设置值，并和i进行swap操作）如果Entry的key不等于传入的key，循环重复此操作。直到下一个Entry为空停止。


> 如果entry里对应的key为null的话，表明此entry为staled entry，就将其替换为当前的key和value：

set方法创建一个新的Entry的时候，并会进行启发式的垃圾清理，用于清理无用的Entry。主要通过`cleanSomeSlots`方法进行清理（清理的时机通常为添加新元素或另一个无用的元素被回收时）。

只要没有清理任何的 stale entries 并且 size 达到阈值的时候（即 table 已满，所有元素都可用），都会触发rehashing：

```
private void rehash() {
	expungeStaleEntries();

	// Use lower threshold for doubling to avoid hysteresis
	if (size >= threshold - threshold / 4)
		resize();
}
```

**rehash 操作会执行一次全表的扫描清理工作，并在 size 大于等于 threshold 的四分之三时进行 resize**

```
private void rehash() {
	expungeStaleEntries();

	// Use lower threshold for doubling to avoid hysteresis
	if (size >= threshold - threshold / 4)
		resize();
}

/**
 * Double the capacity of the table.
 */
private void resize() {
	Entry[] oldTab = table;
	int oldLen = oldTab.length;
	int newLen = oldLen * 2;
	Entry[] newTab = new Entry[newLen];
	int count = 0;

	for (int j = 0; j < oldLen; ++j) {
		Entry e = oldTab[j];
		if (e != null) {
			ThreadLocal<?> k = e.get();
			if (k == null) {
				e.value = null; // Help the GC
			} else {
				int h = k.threadLocalHashCode & (newLen - 1);
				while (newTab[h] != null)
					h = nextIndex(h, newLen);
				newTab[h] = e;
				count++;
			}
		}
	}

	setThreshold(newLen);
	size = count;
	table = newTab;
}

private void setThreshold(int len) {
	threshold = len * 2 / 3;
}
```

注意在`resize`到时候，又将threshold设成了长度的`2/3`，因此 `ThreadLocalMap` 的实际 load factor 为 `3/4 * 2/3 = 0.5`。

remove 方法的思想类似，就不看了。

根据上面的分析和网上的资料，画了一个图

![](/assets/images/posts/threadlocal/threadlocal-1.png)

# 2. key为什么用弱引用
下面我们分两种情况讨论：

- key 使用强引用：引用的ThreadLocal的对象被回收了，但是ThreadLocalMap还持有ThreadLocal的强引用，如果没有手动删除，ThreadLocal不会被回收，导致Entry内存泄漏。
- key 使用弱引用：引用的ThreadLocal的对象被回收了，由于ThreadLocalMap持有ThreadLocal的弱引用，即使没有手动删除，ThreadLocal也会被回收。value在下一次ThreadLocalMap调用set,get，remove的时候会被清除。

比较两种情况，我们可以发现：由于ThreadLocalMap的生命周期跟Thread一样长，如果都没有手动删除对应key，都会导致内存泄漏，但是使用弱引用可以多一层保障：弱引用ThreadLocal不会内存泄漏，对应的value在下一次ThreadLocalMap调用set,get,remove的时候会被清除。


# 3. ThreadLocal的内存泄漏问题

ThreadLocalMap中的key是ThreadLocal对象，然后ThreadLocal对象时被WeakReference包装的，这样当没有强引用指向该ThreadLocal对象之后，或者说Map中的ThreadLocal对象被判定为弱引用可达时，就会在垃圾收集中被回收掉。

在ThreadLocalMap中，只有key是弱引用，value仍然是一个强引用。当某一条线程中的ThreadLocal使用完毕，没有强引用指向它的时候，这个key指向的对象就会被垃圾收集器回收，从而这个key就变成了null；然而，此时value和value指向的对象之间仍然是强引用关系，只要这种关系不解除，value指向的对象永远不会被垃圾收集器回收，从而导致内存泄漏！

不过每次操作set、get、remove操作时，ThreadLocal都会将key为null的Entry删除，从而避免内存泄漏。同时value与线程同生命周期，线程死亡的时候， value也 被 GC 回收。但是如果一个线程运行周期较长，而且将一个大对象放入LocalThreadMap后便不再调用set、get、remove方法，此时该仍然可能会导致内存泄漏。

> 所以 出现内存泄露的前提必须是持有 value 的线程一直存活 ，这在使用线程池时是很正常的，在这种情况下 value 一直不会被 GC，因为线程对象与 value 之间维护的是强引用。而且后续线程执行的业务一直没有调用 ThreadLocal 的 get 或 set 方法，导致不会主动去删除 key 为 null 的 value 对象 ，在满足这两个条件下 value 对象一直常驻内存，所以存在内存泄露的可能性。

正确的处理方式是：每次使用完 ThreadLocal，都调用它的 remove() 方法清除数据 ，这样才能从根源上避免内存泄漏问题。

# 4. ThreadLocal存放在哪里

在Java中，栈内存归属于单个线程，每个线程都会有一个栈内存，其存储的变量只能在其所属线程中可见，即栈内存可以理解成线程的私有内存。而堆内存中的对象对所有线程可见。堆内存中的对象可以被所有线程访问。

那么是不是说ThreadLocal的实例以及其值存放在栈上呢？

其实不是，因为ThreadLocal实例实际上也是被其创建的类持有（更顶端应该是被线程持有）。而ThreadLocal的值其实也是被线程实例持有。

它们都是位于堆上，只是通过一些技巧将可见性修改成了线程可见。

# 5. InheritableThreadLocal

使用`InheritableThreadLocal`可以实现多个线程访问`ThreadLocal`的值。

如下，我们在主线程中创建一个InheritableThreadLocal的实例，然后在子线程中得到这个InheritableThreadLocal实例设置的值。

```
  final ThreadLocal threadLocal = new InheritableThreadLocal();
  threadLocal.set("droidyue.com");
  Thread t = new Thread() {
	@Override
	public void run() {
	  super.run();
	  System.out.println(threadLocal.get());
	}
  };
  t.start();
```

# 6. 使用场景

- 实现单个线程单例以及单个线程上下文信息存储，比如交易id等
- 实现线程安全，非线程安全的对象使用ThreadLocal之后就会变得线程安全，因为每个线程都会有一个对应的实例
- 承载一些线程相关的数据，避免在方法中来回传递参数

# 7. 散列算法-魔数0x61c88647

前面提过：`HASH_INCREMENT = 0x61c88647`就是每次增加的步长`1640531527`，根据JAVADOC所说，选择这个数字是为了让冲突概率最小

> The difference between successively generated hash codes - turns implicit sequential thread-local IDs into near-optimally spread multiplicative hash values for power-of-two-sized tables.

魔数0x61c88647的选取和斐波那契散列有关，**0x61c88647**对应的十进制为1640531527。而斐波那契散列的乘数可以用`(long) ((1L << 31) * (Math.sqrt(5) - 1));` 如果把这个值给转为带符号的int，则会得到-1640531527。也就是说
 `(long) ((1L << 31) * (Math.sqrt(5) - 1));`得到的结果就是1640531527，也就是魔数**0x61c88647**

`Math.sqrt(5) - 1)`约等于0.618，也就是说0x61c88647理解为一个黄金分割数乘以2的32次方。它可以保证`nextHashCode`生成的哈希值，均匀的分布在2的幂次方上，且小于2的32次方

测试一下

```
public class ThreadLocalHashCodeTest {

    private static AtomicInteger nextHashCode =
            new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    public static void main(String[] args){
        for (int i = 0; i < 16; i++) {
            System.out.print(nextHashCode() & 15);
            System.out.print(" ");
        }
        System.out.println();
        for (int i = 0; i < 32; i++) {
            System.out.print(nextHashCode() & 31);
            System.out.print(" ");
        }
        System.out.println();
        for (int i = 0; i < 64; i++) {
            System.out.print(nextHashCode() & 63);
            System.out.print(" ");
        }
    }
}
```

输出

```
0 7 14 5 12 3 10 1 8 15 6 13 4 11 2 9 
16 23 30 5 12 19 26 1 8 15 22 29 4 11 18 25 0 7 14 21 28 3 10 17 24 31 6 13 20 27 2 9 
16 23 30 37 44 51 58 1 8 15 22 29 36 43 50 57 0 7 14 21 28 35 42 49 56 63 6 13 20 27 34 41 48 55 62 5 12 19 26 33 40 47 54 61 4 11 18 25 32 39 46 53 60 3 10 17 24 31 38 45 52 59 2 9 
```

可以看到元素索引值完美的散列在数组当中，并没有出现冲突



