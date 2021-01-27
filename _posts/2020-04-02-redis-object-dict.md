---
layout: post
title: redis数据结构（3）- 字典
date: 2020-04-02
categories:
    - redis
comments: true
permalink: redis-object-dict.html
---

>  Redis有六种基础数据结构：动态字符串，链表，字典，跳跃表，整数集合和压缩列表。

# 1. 字典

字典，又称为符号表（symbol table）、关联数组（associative array）或映射（map），是一种用于保存键值对（key-value pair）的抽象数据结构。字典中的每一个键都是唯一的，可以通过键查找与之关联的值，并对其修改或删除

Redis的键值对存储就是用字典实现的，散列（Hash）的底层实现之一也是字典

Redis的字典底层是使用哈希表实现的，一个哈希表里面可以有多个哈希表节点，每个哈希表节点中保存了字典中的一个键值对

哈希表结构定义，`dict.h/dictht`

```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值，等于size-1
    unsigned long sizemask;
    // 哈希表已有节点的数量
    unsigned long used;
} dictht;

```
哈希表是由数组table组成，table中每个元素都是指向`dict.h/dictEntry`结构的指针，哈希表节点的定义如下

```c
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    // 指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;

```

其中key是我们的键；v是键值，可以是一个指针，也可以是整数或浮点数；next属性是指向下一个哈希表节点的指针，可以让多个哈希值相同的键值对形成链表，解决键冲突问题

**从哈希表节点结构中，可以看出，在redis中解决hash冲突的方式为采用链地址法。key和v分别用于保存键值对的键和值。**

最后就是我们的字典结构，`dict.h/dict`

```c
typedef struct dict {
    // 和类型相关的处理函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表，两个， 一般情况下， 字典只使用 ht[0] 哈希表， ht[1] 哈希表只会在对 ht[0] 哈希表进行 rehash 时使用。
    dictht ht[2];
    // rehash 索引，它记录了 rehash 目前的进度， 如果目前没有在进行 rehash ， 那么它的值为 -1 。
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    // 迭代器数量
    unsigned long iterators; /* number of iterators currently running */
} dict;

```

type属性和privdata属性是针对不同类型的键值对，用于创建多类型的字典，type是指向dictType结构的指针，privdata则保存需要传给类型特定函数的可选参数，关于dictType结构和类型特定函数可以看下面代码

```c
typedef struct dictType {
    // 计算哈希值的行数
    uint64_t (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

```

`dict`的`ht`属性是两个元素的数组，包含两个`dictht`哈希表，一般字典只使用`ht[0]`哈希表，`ht[1]`哈希表会在对`ht[0]`哈希表进行`rehash`（重哈希）的时候使用，即当哈希表的键值对数量超过负载数量过多的时候，会将键值对迁移到`ht[1]`上

`rehashidx`也是跟rehash相关的，rehash的操作不是瞬间完成的，`rehashidx`记录着rehash的进度，如果目前没有在进行rehash，它的值为-1

结合上面的几个结构，我们来看一下**字典的结构图**（没有在进行rehash）

![](/assets/images/posts/redis-object/redis-object-7.png)

# 2. 键冲突

当一个新的键值对要添加到字典中时，会根据键值对的键计算出哈希值和索引值，根据索引值放到对应的哈希表上，即如果索引值为0，则放到`ht[0]`哈希表上。当有两个或多个的键分配到了哈希表数组上的同一个索引时，就发生了**键冲突**的问题，哈希表使用**链地址法**来解决。

每个哈希表节点都有一个 `next` 指针， 多个哈希表节点可以用 `next` 指针构成一个单向链表， 被分配到同一个索引上的多个节点可以用这个单向链表连接起来， 这就解决了键冲突的问题。

举个例子， 假设程序要将键值对 `k2` 和 `v2` 添加到下图所示的哈希表里面， 并且计算得出 `k2` 的索引值为 `2` ， 那么键 `k1` 和 `k2` 将产生冲突， 而解决冲突的办法就是使用 `next` 指针将键 `k2` 和 `k1` 所在的节点连接起来

![](/assets/images/posts/redis-object/redis-object-40.png)

![](/assets/images/posts/redis-object/redis-object-41.png)

因为 `dictEntry` 节点组成的链表没有指向链表表尾的指针， 所以为了速度考虑， **程序总是将新节点添加到链表的表头位置**（复杂度为 O(1)）， 排在其他已有节点的前面。

当哈希表的键值对太多或太少，就需要对哈希表进行扩展和收缩，通过`rehash`(重新散列)来执行

# 3. **rehash** 

随着操作的不断执行， 哈希表保存的键值对会逐渐地增多或者减少， 为了让哈希表的负载因子（load factor）维持在一个合理的范围之内， 当哈希表保存的键值对数量太多或者太少时， 程序需要对哈希表的大小进行相应的扩展或者收缩。

- **目前的哈希表状态：**

我们可以看到，哈希表中的每个节点都已经使用到了，这时候我们需要对哈希表进行拓展。

![](/assets/images/posts/redis-object/redis-object-8.png)

- **为哈希表分配空间**

为字典的` ht[1]` 哈希表分配空间， 这个哈希表的空间大小取决于要执行的操作， 以及 `ht[0] `当前包含的键值对数量 （也即是`ht[0].used` 属性的值）：

- 如果执行的是扩展操作， 那么 `ht[1]` 的大小为第一个大于等于 `ht[0].used * 2` 的 2^n（`2` 的 `n` 次方幂）；
- 如果执行的是收缩操作， 那么 `ht[1]` 的大小为第一个大于等于 `ht[0].used` 的 [2^n]

因此这里我们为ht[1] 分配 空间为8，

![](/assets/images/posts/redis-object/redis-object-9.png)

- **数据转移**

将保存在 `ht[0]` 中的所有键值对 rehash 到 `ht[1]` 上面： rehash 指的是重新计算键的哈希值和索引值， 然后将键值对放置到 `ht[1]` 哈希表的指定位置上。

数据转移后的结果

![](/assets/images/posts/redis-object/redis-object-10.png)

- **释放ht[0]**

当 `ht[0]` 包含的所有键值对都迁移到了 `ht[1]` 之后 （`ht[0]` 变为空表）， 释放 `ht[0]` ， 将 `ht[1]` 设置为 `ht[0]` ， 并在 `ht[1]` 新创建一个空白哈希表， 为下一次 rehash 做准备。

![](/assets/images/posts/redis-object/redis-object-11.png)

# 4. 哈希表的扩展与收缩

当以下条件中的任意一个被满足时， 程序会自动开始对哈希表**执行扩展操作**：

1. 服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 `1` ；
2. 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 `5` ；

其中哈希表的负载因子可以通过公式：

```
# 负载因子 = 哈希表已保存节点数量 / 哈希表大小
load_factor = ht[0].used / ht[0].size
```

计算得出。

比如说， 对于一个大小为 `4` ， 包含 `4` 个键值对的哈希表来说， 这个哈希表的负载因子为：

```
load_factor = 4 / 4 = 1
```

又比如说， 对于一个大小为 `512` ， 包含 `256` 个键值对的哈希表来说， 这个哈希表的负载因子为：

```
load_factor = 256 / 512 = 0.5
```

根据 BGSAVE 命令或 BGREWRITEAOF 命令是否正在执行， 服务器执行扩展操作所需的负载因子并不相同， 这是因为在执行 BGSAVE 命令或 BGREWRITEAOF 命令的过程中， Redis 需要创建当前服务器进程的子进程， 而大多数操作系统都采用写时复制（[copy-on-write](http://en.wikipedia.org/wiki/Copy-on-write)）技术来优化子进程的使用效率， 所以在子进程存在期间， 服务器会提高执行扩展操作所需的负载因子， 从而尽可能地避免在子进程存在期间进行哈希表扩展操作， 这可以避免不必要的内存写入操作， 最大限度地节约内存。

另一方面， **当哈希表的负载因子小于 `0.1` 时， 程序自动开始对哈希表执行收缩操作。**

# 5. 渐进式 rehash

上面我们说到，在进行拓展或者压缩的时候，可以直接将所有的键值对rehash 到ht[1]中，这是因为数据量比较小。在实际开发过程中，**这个rehash 操作并不是一次性、集中式完成的，而是分多次、渐进式地完成的。**

这样做的原因在于， 如果 `ht[0]` 里只保存着四个键值对， 那么服务器可以在瞬间就将这些键值对全部 rehash 到 `ht[1]` ； 但是， 如果哈希表里保存的键值对数量不是四个， 而是四百万、四千万甚至四亿个键值对， 那么要一次性将这些键值对全部 rehash 到 `ht[1]` 的话， 庞大的计算量可能会导致服务器在一段时间内停止服务。

因此， 为了避免 rehash 对服务器性能造成影响， 服务器不是一次性将 `ht[0]` 里面的所有键值对全部 rehash 到 `ht[1]` ， 而是分多次、渐进式地将 `ht[0]` 里面的键值对慢慢地 rehash 到 `ht[1]` 。

以下是哈希表渐进式 rehash 的详细步骤：

1. 为 `ht[1]` 分配空间， 让字典同时持有 `ht[0]` 和 `ht[1]` 两个哈希表。
2. 在字典中维持一个索引计数器变量 `rehashidx` ， 并将它的值设置为 `0` ， 表示 rehash 工作正式开始。
3. 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 `ht[0]` 哈希表在 `rehashidx` 索引上的所有键值对 rehash 到 `ht[1]` ， 当 rehash 工作完成之后， 程序将 `rehashidx` 属性的值增一。
4. 随着字典操作的不断执行， 最终在某个时间点上， `ht[0]` 的所有键值对都会被 rehash 至 `ht[1]` ， 这时程序将 `rehashidx` 属性的值设为 `-1` ， 表示 rehash 操作已完成。

采用渐进式rehash 的好处在于它采取分而治之的方式，避免了集中式rehash 带来的庞大计算量。

> rehash是以bucket(桶)为基本单位进行渐进式的数据迁移的，每步完成一个bucket的迁移，直至所有数据迁移完毕。一个bucket对应哈希表数组中的一条entry链表。新版本的dictRehash()还加入了一个最大访问空桶数(empty_visits)的限制来进一步减小可能引起阻塞的时间。

下面用一个例子展示了一次完整的渐进式 rehash 过程， 注意观察在整个 rehash 过程中， 字典的 `rehashidx` 属性是如何变化的

- **准备开始rehash**

![](/assets/images/posts/redis-object/redis-object-42.png)

- **rehash索引0上的键值对**

![](/assets/images/posts/redis-object/redis-object-43.png)

- **rehash索引1上的键值对**

![](/assets/images/posts/redis-object/redis-object-44.png)

**rehash索引2上的键值对**

![](/assets/images/posts/redis-object/redis-object-45.png)

- **rehash索引3上的键值对**

![](/assets/images/posts/redis-object/redis-object-46.png)

- **rehash完成**

![](/assets/images/posts/redis-object/redis-object-47.png)

**渐进式 rehash 执行期间的哈希表操作**

因为在进行渐进式 rehash 的过程中， 字典会同时使用 `ht[0]` 和 `ht[1]` 两个哈希表， 所以在渐进式 rehash 进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行： 比如说， 要在字典里面查找一个键的话， 程序会先在 `ht[0]` 里面进行查找， 如果没找到的话， 就会继续到 `ht[1]` 里面进行查找， 诸如此类。

另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 `ht[1]` 里面， 而 `ht[0]` 则不再进行任何添加操作： 这一措施保证了 `ht[0]` 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。

**渐进式rehash带来的问题**

渐进式rehash避免了redis阻塞，可以说非常完美，但是由于在rehash时，需要分配一个新的hash表，在rehash期间，同时有两个hash表在使用，**会使得redis内存使用量瞬间突增，在Redis 满容状态下由于Rehash会导致大量Key驱逐。**

**rehash的时机**

在redis的实现中，没有集中的将原有的key重新rehash到新的槽中，而是分解到各个命令的执行中，以及周期函数中

- **操作辅助rehash** 在redis中每一个增删改查命令中都会判断数据库字典中的哈希表是否正在进行渐进式rehash，如果是则帮助执行一次
- **定时辅助rehash** 虽然redis实现了在读写操作时，辅助服务器进行渐进式rehash操作，但是如果服务器比较空闲，redis数据库将很长时间内都一直使用两个哈希表。所以在redis周期函数中，如果发现有字典正在进行渐进式rehash操作，则会花费1毫秒的时间，帮助一起进行渐进式rehash操作

# 6. 参考资料

https://juejin.im/post/5d71d3bee51d453b5f1a04f1

http://redisbook.com/preview/dict/collision_resolution.html

http://redisbook.com/preview/dict/rehashing.html

http://redisbook.com/preview/dict/incremental_rehashing.html