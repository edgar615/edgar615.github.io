---
layout: post
title: java HashMap
date: 2019-12-16
categories:
    - java
comments: true
permalink: java-hashmap.html
---

很早看过JDK1.6HashMap源码和死循环问题，后来JDK升级为1.8就没有研究它的优化了，这里重新阅读一下1.8的源码

> 大部分内容参考了美团技术团队的文章

> JDK1.7会产生死循环的原因就是1.7链表新节点采用的是头插法，这样在扩容迁移元素时，会将元素顺序改变，导致两个线程中出现元素的相互指向而形成循环链表
>
> 具体参考这个文章：https://coolshell.cn/articles/9606.html

HashMap：它根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 HashMap最多只允许一条记录的键为null，允许多条记录的值为null。HashMap非线程安全，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。

JDK8对HashMap底层的实现进行了优化，例如引入红黑树的数据结构和扩容的优化等

先看HashMap的存储结构

![](/assets/images/posts/hashmap/hashmap-1.png)

HashMap就是使用哈希表来存储的。但是散列函数不可能是完美的，key分布完全均匀的情况是不存在的，所以碰撞冲突总是难以避免。哈希表为解决冲突，可以采用开放地址法和链地址法等来解决问题，Java中HashMap采用了链地址法。链地址法，简单来说，就是数组加链表的结合。在每个数组元素上都一个链表结构，当数据被Hash后，得到数组下标，把数据放在对应下标元素的链表上。

对于`put`方法，HashMap通过对key计算它的hash值来定位该键值对的存储位置，有时两个key会定位到相同的位置，表示发生了Hash碰撞。当然Hash算法计算结果越分散均匀，Hash碰撞的概率就越小，map的存取效率就会越高。

如果哈希桶数组很大，即使较差的Hash算法也会比较分散，如果哈希桶数组数组很小，即使好的Hash算法也会出现较多碰撞，所以就需要在空间成本和时间成本之间权衡，其实就是在根据实际情况确定哈希桶数组的大小，并在此基础上设计好的hash算法减少Hash碰撞。HashMap通过Hash算法和扩容机制来控制map使得Hash碰撞的概率又小，哈希桶数组（Node[] table）占用空间又少。

# 源码实现

在HashMap内部，K/V数据都是存储为一个内部Node对象，它实现了Map.Entry接口，本质是就是一个映射(键值对)

```
static class Node<K,V> implements Map.Entry<K,V> {
	final int hash;
	final K key;
	V value;
	Node<K,V> next;

	Node(int hash, K key, V value, Node<K,V> next) {
		this.hash = hash;
		this.key = key;
		this.value = value;
		this.next = next;
	}
}
```

# 变量
HashMap有下面几个成员变量

```
/**
 * 哈希桶数组，默认长度16
 */
transient Node<K,V>[] table;

/**
 * Holds cached entrySet(). Note that AbstractMap fields are used
 */
transient Set<Map.Entry<K,V>> entrySet;

/**
 * HashMap中实际存在的键值对数量
 */
transient int size;

/**
 * 记录HashMap内部结构发生变化的次数，主要用于迭代的快速失败。
 * 内部结构发生变化指的是结构发生变化，例如put新键值对，但是某个key对应的value值
 * 被覆盖不属于结构变化。
 */
transient int modCount;

/**
 * 下次扩容的阈值(capacity * load factor).超过这个数目就重新扩容
 */
int threshold;

/**
 * 负载因子，默认为0.75f，在哈希桶数组定义好长度之后，负载因子越大，所能容纳的键值对个数越多。
 */
final float loadFactor;
```
在HashMap中，**哈希桶数组table的长度length大小必须为2的n次方**(一定是合数)，这是一种非常规的设计

> 常规的设计是把桶的大小设计为素数（相对来说素数导致冲突的概率要小于合数）。Hashtable初始化桶大小为11，就是桶大小设计为素数的应用（Hashtable扩容后不能保证还是素数）。HashMap采用这种非常规设计，主要是为了在取模和扩容时做优化，同时为了减少冲突，HashMap定位哈希桶索引位置时，也加入了高位参与运算的过程。

默认的负载因子0.75是对空间和时间效率的一个平衡选择，建议大家不要修改，除非在时间和空间比较特殊的情况下，如果内存空间很多而又对时间效率要求很高，可以降低负载因子Load factor的值；相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子loadFactor的值，这个值可以大于1。

即使负载因子和Hash算法设计的再合理，也免不了会出现链表过长的情况，一旦出现链表过长，则会严重影响HashMap的性能。于是，在JDK1.8版本中，对数据结构做了进一步的优化，引入了红黑树。而当链表长度太长（默认超过8）时，链表就转换为红黑树，利用红黑树快速增删改查的特点提高HashMap的性能

# 确定哈希桶数组索引位置

hash计算并不是key的hashCode，而是将 key 的 hashCode 高位数据移位到低位进行异或运算，这样一些计算出来的哈希值主要差异在高位时的数据，就不会因 HashMap 里哈希寻址时被忽略容量以上的高位，那么即可有效避免此类情况下的哈希碰撞

```
static final int hash(Object key) {
	int h;
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

`(h = k.hashCode()) ^ (h >>> 16)`，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。

仔细观察get/put方法，可以发现确定索引位置还有一步取模计算

```
tab[(n - 1) & hash]
```

这个方法非常巧妙，它通过`h & (table.length -1)`来得到该对象的保存位，而HashMap底层数组的长度总是2的n次方，这是HashMap在速度上的优化。**当length总是2的n次方时**，`h&(length-1)`运算等价于对length取模，也就是`h%length`，但是`&`比`%`具有更高的效率。这也是为什么哈希桶数组的长度必须是2的n次方

![](/assets/images/posts/hashmap/hashmap-2.png)

# put方法
先看下面的流程图再看源码会更清晰一点

![](/assets/images/posts/hashmap/hashmap-3.png)

put方法最终调用的是puVal方法

```
public V put(K key, V value) {
	return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
			   boolean evict) {
	Node<K,V>[] tab; Node<K,V> p; int n, i;
	// 如果原table是空的或者未存储任何元素则需要先初始化进行tab的初始化
	// HashMap使用了Lazy策略，table数组只会在第一次调用put()函数时进行初始化，这是一种防止内存浪费的做法，像ArrayList也是Lazy的，它在第一次调用add()时才会初始化内部的数组。
	if ((tab = table) == null || (n = tab.length) == 0)
		n = (tab = resize()).length;
	// 通过取模计算确定元素在table数组中的索引，当对应位置为null时，将新元素放入数组中
	if ((p = tab[i = (n - 1) & hash]) == null)
		tab[i] = newNode(hash, key, value, null);
	// 若对应位置不为空时说明哈希冲突
	else {
		Node<K,V> e; K k;
		// 节点key存在，直接替换节点
		if (p.hash == hash &&
			((k = p.key) == key || (key != null && key.equals(k))))
			e = p;
		// 当p为红黑树的节点时，向树内插入节点
		else if (p instanceof TreeNode)
			e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
		// 当p是链表时，向链表插入节点
		else {
			for (int binCount = 0; ; ++binCount) {
				// 找到尾节点插入
				if ((e = p.next) == null) {
					p.next = newNode(hash, key, value, null);
					// 判断链表长度是否达到树化阈值8，达到就对链表进行树化
					if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
						treeifyBin(tab, hash);
					break;
				}
				// key已经存在直接覆盖value
				if (e.hash == hash &&
					((k = e.key) == key || (key != null && key.equals(k))))
					break;
				p = e;
			}
		}
		// 覆盖操作
		if (e != null) { // existing mapping for key
			V oldValue = e.value;
			if (!onlyIfAbsent || oldValue == null)
				e.value = value;
			 // 回调已允许LinkedHashMap后置操作
			afterNodeAccess(e);
			return oldValue;
		}
	}
	// 更新修改次数
	++modCount;
	// 检查数组是否需要进行扩容
	if (++size > threshold)
		resize();
	// 回调已允许LinkedHashMap后置操作
	afterNodeInsertion(evict);
	return null;
}
```

# resize
HashMap通过负载因子（Load Factor）乘以table数组的长度来计算出临界值，算法：`threshold = load_factor * capacity`。比如，HashMap的默认初始容量为16（`capacity = 16`），默认负载因子为0.75（`load_factor = 0.75`），那么临界值就为`threshold = 0.75 * 16 = 12`，只要Entry的数量大于12，就会触发扩容操作。

扩容方法更复杂

```
final Node<K,V>[] resize() {
	Node<K,V>[] oldTab = table;
	int oldCap = (oldTab == null) ? 0 : oldTab.length;
	// 获取扩容的阈值（容量*负载系数）
	int oldThr = threshold;
	// 定义并初始化新数组的长度和新阈值
	int newCap, newThr = 0;
	// 断是初始化数组还是扩容，等于或小于0表示需要初始化数组，大于0表示需要扩容数组。若  if(oldCap > 0)=true 表示需扩容而非初始化
	if (oldCap > 0) {
		// 判断数组长度是否已经是最大，MAXIMUM_CAPACITY =（2^30），如果已经最大， 将阈值设置为最大，不在扩容
		if (oldCap >= MAXIMUM_CAPACITY) {
			threshold = Integer.MAX_VALUE;
			return oldTab;
		}
		// 数组长度扩为两倍
		else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
				 oldCap >= DEFAULT_INITIAL_CAPACITY)
			//新阈值扩为两倍
			newThr = oldThr << 1; // double threshold
	}
	// 初始化数组
	else if (oldThr > 0) // initial capacity was placed in threshold
		// 说明调用的是HashMap的有参构造函数，因为无参构造函数并没有对threshold进行初始化
		newCap = oldThr;
	else {               // zero initial threshold signifies using defaults
		// 说明调用的是HashMap的无参构造函数
		newCap = DEFAULT_INITIAL_CAPACITY;
		newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
	}
	// 当阈值为0时需重新计算，公式：容量（newCap）*负载系数（loadFactor）
	if (newThr == 0) {
		float ft = (float)newCap * loadFactor;
		newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
				  (int)ft : Integer.MAX_VALUE);
	}
	// 更新阈值
	threshold = newThr;
	// 将新数组赋值给底层数组
	@SuppressWarnings({"rawtypes","unchecked"})
		Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
	table = newTab;
	// 迁移数据
	if (oldTab != null) {
		for (int j = 0; j < oldCap; ++j) {
			Node<K,V> e;
			if ((e = oldTab[j]) != null) {
				oldTab[j] = null;
				// 判断数组内此下标中是否只存储了一个元素，是的话直接转移
				if (e.next == null)
					newTab[e.hash & (newCap - 1)] = e;
				// 红黑树
				else if (e instanceof TreeNode)
					((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
				// 链表
				else { // preserve order
					Node<K,V> loHead = null, loTail = null;
					Node<K,V> hiHead = null, hiTail = null;
					Node<K,V> next;
					do {
						next = e.next;
						if ((e.hash & oldCap) == 0) {
							if (loTail == null)
								loHead = e;
							else
								loTail.next = e;
							loTail = e;
						}
						else {
							if (hiTail == null)
								hiHead = e;
							else
								hiTail.next = e;
							hiTail = e;
						}
					} while ((e = next) != null);
					if (loTail != null) {
						loTail.next = null;
						newTab[j] = loHead;
					}
					if (hiTail != null) {
						hiTail.next = null;
						newTab[j + oldCap] = hiHead;
					}
				}
			}
		}
	}
	return newTab;
}
```

JDK1.8对链表的扩容进行了优化

table数组的大小约束对于整个HashMap都至关重要，为了防止传入一个不是2次幂的整数，必须要有所防范。`tableSizeFor()`函数会尝试修正一个整数，并转换为离该整数最近的2次幂。

```
static final int tableSizeFor(int cap) {
	int n = cap - 1;
	n |= n >>> 1;
	n |= n >>> 2;
	n |= n >>> 4;
	n |= n >>> 8;
	n |= n >>> 16;
	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

![](/assets/images/posts/hashmap/hashmap-7.jpg)

对于取模计算`index = (table.length - 1) & hash`，由于数组的大小永远是一个2次幂，在扩容之后，一个元素的新索引要么是在原位置，要么就是在原位置加上扩容前的容量。这个方法的巧妙之处全在于&运算，之前提到过&运算只会关注n - 1（n = 数组长度）的有效位，当扩容之后，n的有效位相比之前会多增加一位（n会变成之前的二倍，所以确保数组长度永远是2次幂很重要），然后只需要判断hash在新增的有效位的位置是0还是1就可以算出新的索引位置，如果是0，那么索引没有发生变化，如果是1，索引就为原索引加上扩容前的容量。

![](/assets/images/posts/hashmap/hashmap-8.jpg)

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：

![](/assets/images/posts/hashmap/hashmap-6.png)

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点。有一点注意区别，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置。


# get方法
弄清楚put和resize方法后，get方法就很清晰了，这里不在贴代码了

# 线程不安全
JDK1.8解决了并发情况下的死循环问题，但是依然会存在线程不安全的问题

如果线程A和线程B同时进行put操作，刚好这两条不同的数据hash值一样，并且该位置数据为null，所以这线程A、B都会进入下面的代码中。

```
	if ((p = tab[i = (n - 1) & hash]) == null)
		tab[i] = newNode(hash, key, value, null);
```

假设一种情况，线程A进入后还未进行数据插入时挂起，而线程B正常执行，从而正常插入数据，然后线程A获取CPU时间片，此时线程A不用再进行hash判断了，问题出现：线程A会把线程B插入的数据给覆盖，发生线程不安全。

# 为什么转换红黑树的阈值是8

因为在元素放置过程中，如果一个对象哈希冲突，都被放置到同一个桶里，则会形成一个链表，而链表查询是线性的，在查找的效率上只有`O(n)`，而我们大部分的操作都是在进行查找，在`hashCode()`设计的不是非常良好的情况下，碰撞冲突可能会频繁发生，链表也会变得越来越长，这个效率是非常差的。

而且在现实世界，构造哈希冲突的数据并不是非常复杂的事情，恶意代码就可以利用这些数据大量与服务器端交互，导致服务器端 CPU 大量占用，这就构成了哈希碰撞拒绝服务攻击，国内一线互联网公司就发生过类似攻击事件

所以为了保证数据安全及相关操作效率当链表长度达到8就转成红黑树，而当长度降到6就转成普通链表。这样查找红黑树所需的时间就只有`O(log n)`了

当hashCode离散性很好的时候，树型bin用到的概率非常小，因为数据均匀分布在每个bin中，几乎不会有bin中链表长度会达到阈值。但是在随机hashCode下，离散性可能会变差，然而JDK又不能阻止用户实现这种不好的hash算法，因此就可能导致不均匀的数据分布。不过理想情况下随机hashCode算法下所有bin中节点的分布频率会遵循泊松分布，我们可以看到，一个bin中链表长度达到8个元素的概率为0.00000006，几乎是不可能事件。所以，之所以选择8，是根据概率统计决定的。由此可见，发展30年的Java每一项改动和优化都是非常严谨和科学的。

>```
>*
>* Because TreeNodes are about twice the size of regular nodes, we
>* use them only when bins contain enough nodes to warrant use
>* (see TREEIFY_THRESHOLD). And when they become too small (due to
>* removal or resizing) they are converted back to plain bins.  In
>* usages with well-distributed user hashCodes, tree bins are
>* rarely used.  Ideally, under random hashCodes, the frequency of
>* nodes in bins follows a Poisson distribution
>* (http://en.wikipedia.org/wiki/Poisson_distribution) with a
>* parameter of about 0.5 on average for the default resizing
>* threshold of 0.75, although with a large variance because of
>* resizing granularity. Ignoring variance, the expected
>* occurrences of list size k are (exp(-0.5) * pow(0.5, k) /
>* factorial(k)). The first values are:
>*
>* 0:    0.60653066
>* 1:    0.30326533
>* 2:    0.07581633
>* 3:    0.01263606
>* 4:    0.00157952
>* 5:    0.00015795
>* 6:    0.00001316
>* 7:    0.00000094
>* 8:    0.00000006
>* more: less than 1 in ten million
>```

# 参考资料

https://zhuanlan.zhihu.com/p/21673805

https://mp.weixin.qq.com/s/AxL1yP9CcOtREwxnDHGaUg

https://mp.weixin.qq.com/s/dGcjjHipK-oIxKZuTQeAqw

https://mp.weixin.qq.com/s/Tf5SsWJHKsVTbqdSuG-6EQ

https://sylvanassun.github.io/2018/03/16/2018-03-16-map_family/