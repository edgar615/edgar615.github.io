---
layout: post
title: 缓存-Caffeine(part5)
date: 2019-11-17
categories:
    - 缓存
comments: true
permalink: cache-caffeine.html
---

进程内缓存以前用Guava Cache，后来发现了Caffeine，宣称更好的性能。spring也集成了Caffeine而不再集成Guava Cache。简单了看了下源码，学习了一下设计。

Caffeine提供了两个cache：`UnboundedLocalCache`和`BoundedLocalCache`

# UnboundedLocalCache
如果我们没有配置任何的回收策略，则会默认使用`UnboundedLocalCache`。该实现类最低限度的提供了缓存的功能，初始化时提供了一个默认大小为16的`ConcurrentHashMap`用来存储数据，也提供了基本的状态计数器，删除监听器，编写器等。由于没有任何主动的回收策略，`UnboundedLocalCache`的本质就是对Map的操作。

```
  boolean isBounded() {
    return (maximumSize != UNSET_INT)
        || (maximumWeight != UNSET_INT)
        || (expireAfterAccessNanos != UNSET_INT)
        || (expireAfterWriteNanos != UNSET_INT)
        || (expiry != null)
        || (keyStrength != null)
        || (valueStrength != null);
  }

  @Nonnull
  public <K1 extends K, V1 extends V> Cache<K1, V1> build() {
    ...
    Caffeine<K1, V1> self = (Caffeine<K1, V1>) this;
    return isBounded() || refreshes()
        ? new BoundedLocalCache.BoundedLocalManualCache<>(self)
        : new UnboundedLocalCache.UnboundedLocalManualCache<>(self);
  }
```

```
  UnboundedLocalCache(Caffeine<? super K, ? super V> builder, boolean async) {
    this.data = new ConcurrentHashMap<>(builder.getInitialCapacity());
    this.statsCounter = builder.getStatsCounterSupplier().get();
    this.removalListener = builder.getRemovalListener(async);
    this.isRecordingStats = builder.isRecordingStats();
    this.writer = builder.getCacheWriter();
    this.executor = builder.getExecutor();
    this.ticker = builder.getTicker();
  }
```

查询缓存` cache.getIfPresent(1L);`
get的实现很简单，直接通过map获取数据，然后统计命中率

```
  public @Nullable V getIfPresent(Object key, boolean recordStats) {
    V value = data.get(key);

    if (recordStats) {
      if (value == null) {
        statsCounter.recordMisses(1);
      } else {
        statsCounter.recordHits(1);
      }
    }
    return value;
  }
```

更新缓存`cache.put(1L, new User());`
```

  @Override
  public @Nullable V put(K key, V value, boolean notifyWriter) {
    requireNonNull(value);

    // ensures that the removal notification is processed after the removal has completed
    @SuppressWarnings({"unchecked", "rawtypes"})
    V[] oldValue = (V[]) new Object[1];
    if ((writer == CacheWriter.disabledWriter()) || !notifyWriter) {
      oldValue[0] = data.put(key, value);
    } else {
      data.compute(key, (k, v) -> {
        if (value != v) {
          writer.write(key, value);
        }
        oldValue[0] = v;
        return value;
      });
    }

    if (hasRemovalListener() && (oldValue[0] != null) && (oldValue[0] != value)) {
      notifyRemoval(key, oldValue[0], RemovalCause.REPLACED);
    }

    return oldValue[0];
  }
```
对应数据的写入有两种方式：

- 没有自定义的`CacheWriter`，直接通过map的put方法写入，默认是这个
- 有自定义的`CacheWriter`，当写入的值与缓存中的值不等时，通过自定义`CacheWriter`写入，`CacheWriter`后面研究

如果注册了监听器`removalListener`，且缓存中的值被替换，发送一个异步通知

```
  public void notifyRemoval(@Nullable K key, @Nullable V value, RemovalCause cause) {
    requireNonNull(removalListener(), "Notification should be guarded with a check");
    executor.execute(() -> removalListener().onRemoval(key, value, cause));
  }
```

驱逐缓存`cache.invalidate(1L);`

```
  public @Nullable V remove(Object key) {
    @SuppressWarnings("unchecked")
    K castKey = (K) key;
    @SuppressWarnings({"unchecked", "rawtypes"})
    V[] oldValue = (V[]) new Object[1];

    if (writer == CacheWriter.disabledWriter()) {
      oldValue[0] = data.remove(key);
    } else {
      data.computeIfPresent(castKey, (k, v) -> {
        writer.delete(castKey, v, RemovalCause.EXPLICIT);
        oldValue[0] = v;
        return null;
      });
    }

    if (hasRemovalListener() && (oldValue[0] != null)) {
      notifyRemoval(castKey, oldValue[0], RemovalCause.EXPLICIT);
    }

    return oldValue[0];
  }
```

驱逐缓存和更新相似，也有两种方式来驱逐

- 没有自定义的`CacheWriter`，直接通过map的remove方法写入，默认是这个
- 有自定义的`CacheWriter`，缓存中有值时，通过自定义`CacheWriter`驱逐

如果注册了监听器`removalListener`，且缓存中的值被替换，发送一个异步通知

# BoundedLocalCache
这个cache是我们平时用到最多的缓存，只要我们指定了驱逐策略（最大容量、过期时间等）就会使用这个缓存。这个缓存要比`UnboundedLocalCache`复杂得多。

首先我们来看看默认值

```
  // CPU核数
  static final int NCPU = Runtime.getRuntime().availableProcessors();
  // write buffer的初始值
  static final int WRITE_BUFFER_MIN = 4;
  // write buffer的最大值，ceilingPowerOfTwo(NCPU)用来取cpu核数最接近的2的幂数
  static final int WRITE_BUFFER_MAX = 128 * ceilingPowerOfTwo(NCPU);
  /** The number of attempts to insert into the write buffer before yielding. */
  static final int WRITE_BUFFER_RETRIES = 100;
  /** The maximum weighted capacity of the map. */
  static final long MAXIMUM_CAPACITY = Long.MAX_VALUE - Integer.MAX_VALUE;
  /** The percent of the maximum weighted capacity dedicated to the main space. */
  // Protected区+Probation区占用比例，剩下的属于Eden区
  static final double PERCENT_MAIN = 0.99d;
  // Protected区占用比例
  static final double PERCENT_MAIN_PROTECTED = 0.80d;
  /** The maximum time window between entry updates before the expiration must be reordered. */
  static final long EXPIRE_WRITE_TOLERANCE = TimeUnit.SECONDS.toNanos(1);
  /** The maximum duration before an entry expires. */
  static final long MAXIMUM_EXPIRY = (Long.MAX_VALUE >> 1); // 150 years
```

变量

```
nodeFactory = NodeFactory.newFactory(builder, isAsync);
```
根据缓存的配置会加载不同的NodeFactory，它被用来根据不同的配置将KV转化为一个`Node`对象存储在ConcurrentHashMap中

查询缓存
```
  public @Nullable V getIfPresent(Object key, boolean recordStats) {
    Node<K, V> node = data.get(nodeFactory.newLookupKey(key));
    if (node == null) {
      if (recordStats) {
        statsCounter().recordMisses(1);
      }
      return null;
    }
    long now = expirationTicker().read();
    if (hasExpired(node, now)) {
      if (recordStats) {
        statsCounter().recordMisses(1);
      }
      scheduleDrainBuffers();
      return null;
    }

    @SuppressWarnings("unchecked")
    K castedKey = (K) key;
    V value = node.getValue();

    if (!isComputingAsync(node)) {
      setVariableTime(node, expireAfterRead(node, castedKey, value, expiry(), now));
      setAccessTime(node, now);
    }
    afterRead(node, now, recordStats);
    return value;
  }
```
根据key找到数据后，需要检查数据是否过期，如果会通过`scheduleDrainBuffers`异步执行一个任务`PerformCleanupTask`来清理过期任务。最后会通过`afterRead方法`

更新缓存、删除缓存部分比`UnboundedLocalCache`要复杂的多，具体的细节我没有仔细研究，注意要注意在更新后会触发一个`afterWrite`方法来对数据做一些关键处理

## 数据淘汰策略

在caffeine所有的数据都在ConcurrentHashMap中，在caffeine中有三个记录引用的**LRU**队列:

- Eden队列:在caffeine中规定只能为缓存容量的%1,如果size=100,那这个队列的有效大小就等于1。这个队列中记录的是新到的数据，防止突发流量由于之前没有访问频率，而导致被淘汰。比如有一部新剧上线，在最开始其实是没有访问频率的，防止上线之后被其他缓存淘汰出去，而加入这个区域。伊甸区，最舒服最安逸的区域，在这里很难被其他数据淘汰。
- Probation队列:叫做缓刑队列，在这个队列就代表你的数据相对比较冷，马上就要被淘汰了。这个有效大小为size减去eden减去protected。
- Protected队列:在这个队列中，可以稍微放心一下了，你暂时不会被淘汰，但是别急，如果Probation队列没有数据了或者Protected数据满了，你也将会被面临淘汰的尴尬局面。当然想要变成这个队列，需要把Probation访问一次之后，就会提升为Protected队列。这个有效大小为(size减去eden) X 80% 如果size =100，就会是79。



# PerformCleanupTask
# 参考资料

https://mp.weixin.qq.com/s/wnPrE4MglmCFxyAwtTh_5A