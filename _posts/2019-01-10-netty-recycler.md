---
layout: post
title: Netty - 对象池
date: 2019-01-10
categories:
    - netty
comments: true
permalink: netty-recycler.html
---

> 基本复制自《Netty 核心原理剖析与 RPC 实践》

# 1. 使用

Netty为了增强高性能并发能力, 减少通用对象分配的损耗, 也采用了对象池的概念。 当需要某个对象时, 首先从对象池中获取该对象, 当使用完成后, 将对象释放到对象池中, 这样达到重复使用对象的效果。基本使用如下:

```
public class RecycleExample {
    private static final Recycler<User> userRecycler = new Recycler<User>() {
        @Override
        protected User newObject(Handle<User> handle) {
            return new User(handle);
        }
    };

    public static void main(String[] args) {
        User user1 = userRecycler.get(); // 1、从对象池获取 User 对象
        user1.setName("hello"); // 2、设置 User 对象的属性
        user1.recycle(); // 3、回收对象到对象池
        User user2 = userRecycler.get(); // 4、从对象池获取对象
        System.out.println(user2.getName());
        System.out.println(user1 == user2);
    }

    static final class User {
        private String name;
        private Recycler.Handle<User> handle;
        public void setName(String name) {
            this.name = name;
        }
        public String getName() {
            return name;
        }
        public User(Recycler.Handle<User> handle) {
            this.handle = handle;
        }
        public void recycle() {
            handle.recycle(this);
        }
    }
}

```

# 2. Recycler 在 Netty 中的应用

Recycler 在 Netty 里面使用也是非常频繁的，Netty通过RecyclerObjectPool封装了Recycler。

```
private static final class RecyclerObjectPool<T> extends ObjectPool<T> {
    private final Recycler<T> recycler;

    RecyclerObjectPool(final ObjectCreator<T> creator) {
         recycler = new Recycler<T>() {
            @Override
            protected T newObject(Handle<T> handle) {
                return creator.newObject(handle);
            }
        };
    }

    @Override
    public T get() {
        return recycler.get();
    }
}
```

通过调用跟踪，我们可以发现很多类都使用了RecyclerObjectPool的get方法来获取对象。

其中比较常用的有 PooledHeapByteBuf 和 PooledDirectByteBuf，分别对应的堆内存和堆外内存的池化实现。例如我们在使用 PooledDirectByteBuf 的时候，并不是每次都去创建新的对象实例，而是从对象池中获取预先分配好的对象实例，不再使用 PooledDirectByteBuf 时，被回收归还到对象池中。

# 3. 原理

 Recycler一共包含四个核心组件：Stack、WeakOrderQueue、Link、DefaultHandle。

![](/assets/images/posts/netty-recycler/netty-recycler-0.png)

首先我们先看下整个 Recycler 的内部结构中各个组件的关系，可以通过下面这幅图进行描述。

![](/assets/images/posts/netty-recycler/netty-recycler-1.png)

- 在多线程的场景下，Netty 为了避免锁竞争问题，每个线程都会持有各自的对象池, Stack 作为本线程对象池的核心, 通过 FastThreadLocal 来实现每个线程的本地化。
- **本线程** threadA 回收本线程产生的对象时, 会将对象以 **DefaultHandle** 的形式存放在 Stack 中；**其它线程**  threadB 也可以回收 threadA产生的对象，threadB 回收的对象不会立即放回 threadA 的 Stack 中，而是保存在  threadB 内部的一个 **WeakOrderQueue** 中。这些外部线程的 WeakOrderQueue 以链表的方式和 Stack  关联起来。当某个线程从Stack中获取不到对象时会从WeakOrderQueue中获取对象。
- 默认情况下一个线程最多持有 2CPU 个 WeakOrderQueue，也就是说一个线程最多可以帮 2CPU 个外部线程的对象池回收对象。可以通过修改 `io.netty.recycler.maxDelayedQueuesPerThread` 参数的值来修改一个线程最多持有的 WeakOrderQueue 的数量。
- WeakOrderQueue 内部以 Link 来管理对象，回收对象都会被存在 Link 链表中的节点上。每个 Link 存放的对象是有限的，一个 Link 最多存放 16 个对象。 当每个 Link 节点存储满了会创建新的 Link 节点放入链表尾部继续存放。
- DefaultHandle 实例中保存了实际回收的对象，Stack 和 WeakOrderQueue 都使用 DefaultHandle 存储回收的对象。在 Stack 中包含一个 elements 数组，该数组保存的是 DefaultHandle 实例。DefaultHandle 中每个 Link 节点所存储的 16 个对象也是使用 DefaultHandle 表示的。
- 当前线程从对象池中拿对象时, 首先从 Stack 中获取，若没有的话，将尝试从 cursor 指向的 WeakOrderQueue 中回收一个 Link 的对象，如果回收到了就继续从 Stack 中获取对象；如果没有回收到就创建对象。
- 一个对象池中最多存放 4K 个对象 , 可以通过 `io.netty.recycler.maxCapacity` 控制；而 Link 节点中每个 DefaultHandle 数组默认长度 16, 可以通过`io.netty.recycler.linkCapacity` 控制;

```
private static final class Stack<T> {

    // 所属的 Recycler
    final Recycler<T> parent;
    // 所属线程的弱引用
    final WeakReference<Thread> threadRef;
    // 异线程回收对象时，其他线程能保存的被回收对象的最大个数
    final AtomicInteger availableSharedCapacity;
    private final int maxDelayedQueues;

	// WeakOrderQueue最大个数，默认最大为 4k
    private final int maxCapacity;
    private final int interval;
    private final int delayedQueueInterval;
    // 存储缓存数据的数组
    DefaultHandle<?>[] elements;
    // 缓存的 DefaultHandle 对象个数
    int size;
    private int handleRecycleCount;
    // WeakOrderQueue 链表的三个重要节点
    private WeakOrderQueue cursor, prev;
    private volatile WeakOrderQueue head;
}
```

## 3.1. 从 Recycler 中回收对象

Recycler#recycle方法已经标记为过期，对象回收的源码入口 DefaultHandle#recycle

```
@Override
public void recycle(Object object) {
    if (object != value) {
        throw new IllegalArgumentException("object does not belong to handle");
    }
	// stack通过构造传入
    Stack<?> stack = this.stack;
    if (lastRecycledId != recycleId || stack == null) {
        throw new IllegalStateException("recycled already");
    }

    stack.push(this);
}
```

可以看到回收对象会向stack中push对象，进入push方法，可以看到

```
void push(DefaultHandle<?> item) {
    Thread currentThread = Thread.currentThread();
    if (threadRef.get() == currentThread) {
        // 同线程回收
        pushNow(item);
    } else {
        // 异线程回收
        pushLater(item, currentThread);
    }
}
```

**同线程对象回收**

```
private void pushNow(DefaultHandle<?> item) {
    if ((item.recycleId | item.lastRecycledId) != 0) {
        throw new IllegalStateException("recycled already");
    }
    item.recycleId = item.lastRecycledId = OWN_THREAD_ID;

    int size = this.size;
    // 如果超过了 Stack 的最大容量，那么对象会被直接丢弃
    // dropHandle 方法控制对象的回收速率，每 8 个对象会有一个被回收到 Stack 中
    if (size >= maxCapacity || dropHandle(item)) {
        return;
    }
    if (size == elements.length) {
        elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
    }
	// 对象存放在栈顶指针指向的位置
    elements[size] = item;
    this.size = size + 1;
}
```

```
boolean dropHandle(DefaultHandle<?> handle) {
    if (!handle.hasBeenRecycled) {
    	// interval就是回收速率，从FastThreadLocal创建Stack的时候传入
    	// 可以通过io.netty.recycler.ratio修改
        if (handleRecycleCount < interval) {
            handleRecycleCount++;
            // Drop the object.
            return true;
        }
        handleRecycleCount = 0;
        handle.hasBeenRecycled = true;
    }
    return false;
}
```

**异线程对象回收**

异线程回收对象时，并不会添加到 Stack 中，而是会与 WeakOrderQueue 直接打交道

```
private void pushLater(DefaultHandle<?> item, Thread thread) {
    if (maxDelayedQueues == 0) {
        // We don't support recycling across threads and should just drop the item on the floor.
        return;
    }

    // 当前线程帮助其他线程回收对象的缓存，从FastThreadLocal中获取
    Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
    // 取出对象绑定的 Stack 对应的WeakOrderQueue
    WeakOrderQueue queue = delayedRecycled.get(this);
    if (queue == null) {
    	// 最多帮助 2*CPU 核数的线程回收线程，从FastThreadLocal创建Stack的时候传入
    	// 可以通过io.netty.recycler.maxDelayedQueuesPerThread修改
        if (delayedRecycled.size() >= maxDelayedQueues) {
            // 通过WeakOrderQueue.DUMMY 表示当前线程无法再帮助该 Stack 回收对象
            delayedRecycled.put(this, WeakOrderQueue.DUMMY);
            return;
        }
        // 新建 WeakOrderQueue
        if ((queue = newWeakOrderQueue(thread)) == null) {
            // drop object
            return;
        }
        // 与stack绑定
        delayedRecycled.put(this, queue);
    } else if (queue == WeakOrderQueue.DUMMY) {
        // drop object
        return;
    }
	// 加入到WeakOrderQueue
    queue.add(item);
}
```

看一下创建WeakOrderQueue的方法

```
static WeakOrderQueue newQueue(Stack<?> stack, Thread thread) {
    ...
    // 将WeakOrderQueue设置到stack内部的队列头部
    stack.setHead(queue);

    return queue;
}
```

在看一下WeakOrderQueue的add方法

```
void add(DefaultHandle<?> handle) {
    handle.lastRecycledId = id;
    ...
    handleRecycleCount = 0;

    Link tail = this.tail;
    int writeIndex;
    // 如果链表尾部的 Link 已经写满，那么再新建一个 Link 追加到链表尾部
    if ((writeIndex = tail.get()) == LINK_CAPACITY) {
        Link link = head.newLink();
        if (link == null) {
            // Drop it.
            return;
        }
        // We allocate a Link so reserve the space
        this.tail = tail = tail.next = link;

        writeIndex = tail.get();
    }
    // 添加对象到 Link 尾部
    tail.elements[writeIndex] = handle;
    // handle 的 stack 属性赋值为 null，在取出对象的时候，handle 的 stack 属性又再次被赋值回来。
    // 如果 Stack 不再使用，期望被 GC 回收，发现 handle 中还持有 Stack 的引用，那么就无法被 GC 回收，从而造成内存泄漏。
    handle.stack = null;
    // we lazy set to ensure that setting stack to null appears before we unnull it in the owning thread;
    // this also means we guarantee visibility of an element in the queue if we see the index updated
    tail.lazySet(writeIndex + 1);
}
```

## 3.2. 从 Recycler 中获取对象

Recycler#get() 方法首先通过 FastThreadLocal 获取当前线程的唯一栈缓存 Stack，然后尝试从栈顶弹出 DefaultHandle 对象实例，如果 Stack 中没有可用的 DefaultHandle 对象实例，那么会调用 newObject 生成一个新的对象，完成 handle 与用户对象和 Stack 的绑定。

```
public final T get() {
    if (maxCapacityPerThread == 0) {
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    // 获取当前线程缓存的 Stack
    Stack<T> stack = threadLocal.get();
    // 从 Stack 中弹出一个 DefaultHandle 对象
    DefaultHandle<T> handle = stack.pop();
    // 没有就new一个
    if (handle == null) {
        handle = stack.newHandle();
        handle.value = newObject(handle);
    }
    return (T) handle.value;
}
```

继续看pop方法

```
DefaultHandle<T> pop() {
    int size = this.size;
    // 如果 elements 数组中没有可用的对象实例，会调用 scavenge 方法从其他线程回收的对象实例中转移一些到 elements 数组当中
    if (size == 0) {
    	// 尝试从其他线程回收的对象中转移一些到 elements 数组当中
        if (!scavenge()) {
            return null;
        }
        size = this.size;
        if (size <= 0) {
            // double check, avoid races
            return null;
        }
    }
    // elements 数组中有可用的对象实例，直接将对象实例弹出
    size --;
    // 将实例从栈顶弹出
    DefaultHandle ret = elements[size];
    elements[size] = null;
	...
    return ret;
}
```

关键是 `scavenge()` 的逻辑，这个方法首先会从 cursor 指针指向的 WeakOrderQueue 节点回收部分对象到 Stack 的 elements 数组中，如果没有回收到数据就会将 cursor 指针移到下一个 WeakOrderQueue，重复执行以上过程直至回到到对象实例为止。

![](/assets/images/posts/netty-recycler/netty-recycler-2.png)

继续查看scavenge方法，最终又调用了scavengeSome方法

```
private boolean scavengeSome() {
    WeakOrderQueue prev;
    // cursor 指针指向上一次对 WeakorderQueueu 列表的浏览位置，每一次都从上一次的位置继续，这是一种 FIFO 的处理策略
    WeakOrderQueue cursor = this.cursor;
    // 如果 cursor 指针为 null, 则是第一次从 WeakorderQueueu 链表中获取对象，从头结点开始
    if (cursor == null) {
        prev = null;
        cursor = head;
        if (cursor == null) {
            return false;
        }
    } else {
        prev = this.prev;
    }

	// 不断循环从 WeakOrderQueue 链表中找到一个可用的Link
    boolean success = false;
    do {
    	// 从 WeakOrderQueue 中转移数据到 Stack 中，转移成功则跳出
        if (cursor.transfer(this)) {
            success = true;
            break;
        }
        // 获取不到对象可能是因为当前 WeakOrderQueue 没有对象，也有可能是 WeakOrderQueue 所在的线程已经消亡
        WeakOrderQueue next = cursor.getNext();
        // 如果当前处理的 WeakOrderQueue 所在的线程已经消亡，则尽可能的提取里面的数据，之后从列表中删除这个 WeakOrderQueue。
        // WeakOrderQueue是WeakReference<Thread>，当线程消亡后, 通过 cursor.get() 自然变为 null
        if (cursor.get() == null) {
            if (cursor.hasFinalData()) {
            	 // 尽量将该线程对应的 WeakOrderQueue 里面 Link 对应的对象迁移到 Stack 中
                for (;;) {
                    if (cursor.transfer(this)) {
                        success = true;
                    } else {
                        break;
                    }
                }
            }
			// 将已消亡的线程从 WeakOrderQueue 链表中移除
            if (prev != null) {
                cursor.reclaimAllSpaceAndUnlink();
                prev.setNext(next);
            }
        } else {
            prev = cursor;
        }
		// 将 cursor 指针指向下一个 WeakOrderQueue
        cursor = next;

    } while (cursor != null && !success);

    this.prev = prev;
    this.cursor = cursor;
    return success;
}
```

在回过头来看scavenge方法，可以发现每次 Stack 从 WeakOrderQueue 链表回收其中的一个 Link 节点所存储的对象。

```
private boolean scavenge() {
    // 尝试从 WeakOrderQueue 中转移数据到 Stack 中
    if (scavengeSome()) {
        return true;
    }

    // 如果转移失败，会把 cursor 指针重置到 head 节点
    prev = null;
    cursor = head;
    return false;
}
```

WeakOrderQueue 中有对象且线程没有消亡的情况下，Netty 是怎么回收对象的？

每次回收时 Netty 会回收一个 Link 的对象。一个 Link 内部有两个指针，读指针和写指针，读指针指向上次回收的位置，而写指针指向 Link 的尾端，这两个指针中间的对象就是可回收对象。Netty 会先统计出 Link 内部可回收对象的数量，如果超出 Stack 剩余容量，会先把 Stack 扩容。然后依次将对象从 Link  转移到 Stack。转移的时候为了防止 Stack 扩张太快，Netty 会谨慎地回收从未被回收过的对象，具体来说，每 8  个从未被回收过的对象中只会选择一个进行回收。这主要是为了防止应用程序因为某些原因创建了大量一次性对象而使对象池过度扩张。

```
boolean transfer(Stack<?> dst) {
    Link head = this.head.link;
    // WeakOrderQueue中整个Link链为空, 则直接退出
    if (head == null) {
        return false;
    }
	// 说明 head 已经被读取完了，需要将 head 指向下一个 Link
    if (head.readIndex == LINK_CAPACITY) {
        if (head.next == null) {
            return false;
        }
        head = head.next;
        this.head.relink(head);
    }
	// 获取当前 Link 可读的下标
    final int srcStart = head.readIndex;
    // 获取当前 Link 可写的下标
    int srcEnd = head.get();
    // 总共可读长度
    final int srcSize = srcEnd - srcStart;
    if (srcSize == 0) {
        return false;
    }
	// 获取 Stack 的栈顶位置
    final int dstSize = dst.size;
    final int expectedCapacity = dstSize + srcSize;
	// 计算回收后 Stack 的栈顶位置
    if (expectedCapacity > dst.elements.length) {
    	// 扩容
        final int actualCapacity = dst.increaseCapacity(expectedCapacity);
        srcEnd = min(srcStart + actualCapacity - dstSize, srcEnd);
    }

    if (srcStart != srcEnd) {
        final DefaultHandle[] srcElems = head.elements;
        final DefaultHandle[] dstElems = dst.elements;
        int newDstSize = dstSize;
        for (int i = srcStart; i < srcEnd; i++) {
            DefaultHandle<?> element = srcElems[i];
            if (element.recycleId == 0) {
                element.recycleId = element.lastRecycledId;
            } else if (element.recycleId != element.lastRecycledId) {
                throw new IllegalStateException("recycled already");
            }
            srcElems[i] = null;
			// 为了防止 Stack 扩张太快, 实际每 8 个初次回收的对象中只回收 1 个，7 个都被丢弃了
            if (dst.dropHandle(element)) {
                // Drop the object.
                continue;
            }
            element.stack = dst;
            dstElems[newDstSize ++] = element;
        }

        if (srcEnd == LINK_CAPACITY && head.next != null) {
            // Add capacity back as the Link is GCed.
            this.head.relink(head.next);
        }

        head.readIndex = srcEnd;
        if (dst.size == newDstSize) {
            return false;
        }
        dst.size = newDstSize;
        return true;
    } else {
        // The destination stack is full already.
        return false;
    }
}
```

**那么是如何判断消亡的线程内还有数据呢？**答案很简单，只要看 WeakOrderQueue 中 tail 节点的 Link 的读指针是不是指向 Link 的末端就行：

```
boolean hasFinalData() {
    return tail.readIndex != tail.get();
}
```

# 4. 参考资料

《Netty 核心原理剖析与 RPC 实践》