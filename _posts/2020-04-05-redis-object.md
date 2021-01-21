---
layout: post
title: redis数据结构
date: 2020-04-05
categories:
    - redis
comments: true
permalink: redis-object.html
---

# 1. 数据结构
Redis有六种基础数据结构：动态字符串，链表，字典，跳跃表，整数集合和压缩列表。

## 1.1. 动态字符串
redis 没有直接使用 C 语言传统的字符串表示（以空字符结尾的字符数组，以下简称 C 字符串）， 而是自己构建了一种名为简单动态字符串（simple dynamic string，sds）的抽象类型，并将 sds 用作 redis 的默认字符串表示。

SDS的结构定义在sds.h文件中，SDS的定义在Redis 3.2版本之后有一些改变，由一种数据结构变成了5种数据结构，会根据SDS存储的内容长度来选择不同的结构，以达到节省内存的效果，具体的结构定义，我们看以下代码

```c
// 3.0
struct sdshdr {
    // 记录buf数组中已使用字节的数量，即SDS所保存字符串的长度
    unsigned int len;
    // 记录buf数据中未使用的字节数量
    unsigned int free;
    // 字节数组，用于保存字符串
    char buf[];
};

// 3.2
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

```

3.2版本之后，会根据字符串的长度来选择对应的数据结构

```c
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)  // 32
        return SDS_TYPE_5;
    if (string_size < 1<<8)  // 256
        return SDS_TYPE_8;
    if (string_size < 1<<16)   // 65536 64k
        return SDS_TYPE_16;
    if (string_size < 1ll<<32)  // 4294967296 4G
        return SDS_TYPE_32;
    return SDS_TYPE_64;
}

```

3.2版本的sdshdr8结构如下图所示

![](/assets/images/posts/redis-object/redis-object-1.png)

- `len`：记录当前已使用的字节数（不包括`'\0'`），获取SDS长度的复杂度为O(1)
- `alloc`：记录当前字节数组总共分配的字节数量（不包括`'\0'`）
- `flags`：标记当前字节数组的属性，是`sdshdr8`还是`sdshdr16`等，flags值的定义可以看下面代码
- `buf`：字节数组，用于保存字符串，包括结尾空白字符`'\0'

```c
// flags值定义
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```

> 上面的字节数组的空白处表示未使用空间，是Redis优化的空间策略，给字符串的操作留有余地，保证安全提高效率

**通过SDS的结构可以看出，buf数组的长度=free+len+1（其中1表示字符串结尾的空字符）；所以，一个SDS结构占据的空间为：free所占长度+len所占长度+ buf数组的长度=4+4+free+len+1=free+len+9。**

**SDS与C字符串的区别**

C语言使用长度为N+1的字符数组来表示长度为N的字符串，字符数组的最后一个元素为空字符'\0'，但是这种简单的字符串表示方法并不能满足Redis对于字符串在安全性、效率以及功能方面的要求，

那么使用SDS，会有哪些好处呢

- **常数复杂度获取字符串长度**

C字符串不记录字符串长度，获取长度必须遍历整个字符串，复杂度为O(N)；而SDS结构中本身就有记录字符串长度的len属性，所有复杂度为O(1)。Redis将获取字符串长度所需的复杂度从O(N)降到了O(1)，确保获取字符串长度的工作不会成为Redis的性能瓶颈

- **杜绝缓冲区溢出，减少修改字符串时带来的内存重分配次数**

C字符串不记录自身的长度，每次增长或缩短一个字符串，都要对底层的字符数组进行一次内存重分配操作。如果是拼接append操作之前没有通过内存重分配来扩展底层数据的空间大小，就会产生缓存区溢出；如果是截断trim操作之后没有通过内存重分配来释放不再使用的空间，就会产生内存泄漏

而SDS通过未使用空间解除了字符串长度和底层数据长度的关联，3.0版本是用`free`属性记录未使用空间，3.2版本则是`alloc`属性记录总的分配字节数量。通过未使用空间，SDS实现了**空间预分配**和**惰性空间释放**两种优化的空间分配策略，解决了字符串拼接和截取的空间问题

假设程序中有两个在内存中紧邻着的 字符串 s1 和 s2，其中s1 保存了字符串`redis`，s2 则保存了字符串`MongoDb`：

![](/assets/images/posts/redis-object/redis-object-2.png)

如果我们现在将s1 的内容修改为`redis cluster`，但是又忘了重新为s1 分配足够的空间，这时候就会出现以下问题：

![](/assets/images/posts/redis-object/redis-object-3.png)

我们可以看到，原本s2 中的内容已经被S1的内容给占领了，s2 现在为 cluster，而不是`Mongodb` 

Redis 中SDS 的**空间分配策略**完全杜绝了发生缓冲区溢出的可能性： 当我们需要对一个SDS进行修改的时候，redis会在执行拼接操作之前，预先检查给定SDS空间是否足够，如果不够，会先拓展SDS 的空间，然后再执行拼接操作。

当 SDS 需要被修改，并且要对 SDS 进行空间扩展时，Redis 不仅会为 SDS 分配修改所必须要的空间，还会为 SDS 分配额外的未使用的空间。

- 如果修改后， SDS 的长度(也就是len属性的值)将小于 1MB ，那么 Redis 预分配和 len 属性相同大小的未使用空间。
- 如果修改后， SDS 的长度将大于 1MB ，那么 Redis 会分配 1MB 的未使用空间。

比如说，进行修改后 SDS 的 len 长度为20字节，小于 1MB，那么 Redis 会预先再分配 20 字节的空间， SDS 的 buf数组的实际长度(除去最后一字节)变为 20 + 20 = 40 字节。当 SDS的 len 长度大于 1MB时，则只会再多分配 1MB的空间。

类似的，当 SDS 缩短其保存的字符串长度时，并不会立即释放多出来的字节，而是等待之后使用。

举个例子：

![](/assets/images/posts/redis-object/redis-object-4.png)

![](/assets/images/posts/redis-object/redis-object-5.png)


多次对SDS进行拓展，这时候redis 会将SDS的长度修改为N字节，并且将未使用空间同样修改为N字节，此时如果再次进行修改，因为在上一次修改字符串的时候已经拓展了空间，再次进行修改字符串的时候如果发现空间足够使用，因此无须进行空间拓展 

**通过这种预分配策略，SDS将连续增长N次字符串所需的内存重分配次数从必定N次降低为最多N次**

- **惰性空间释放**

 我们在观察SDS 的结构的时候可以看到里面的free 属性，是用于记录空余空间的。我们除了在拓展字符串的时候会使用到free 来进行记录空余空间以外，在对字符串进行收缩的时候，我们也可以使用free 属性来进行记录剩余空间，这样做的好处就是避免下次对字符串进行再次修改的时候，需要对字符串的空间进行拓展。

然而，我们并不是说不能释放SDS 中空余的空间，SDS 提供了相应的API，让我们可以在有需要的时候，自行释放SDS 的空余空间。通过惰性空间释放，SDS 避免了缩短字符串时所需的内存重分配操作，并未将来可能有的增长操作提供了优化　　　　
- **二进制安全**

C字符串中的字符必须符合某种编码，除了字符串的末尾，字符串里面是不能包含空字符的，否则会被认为是字符串结尾，这些限制了C字符串只能保存文本数据，而不能保存像图片这样的二进制数据

而SDS的API都会以处理二进制的方式来处理存放在`buf`数组里的数据，不会对里面的数据做任何的限制。SDS使用`len`属性的值来判断字符串是否结束，而不是空字符

虽然SDS的API是二进制安全的，但还是像C字符串一样以空字符结尾，目的是为了让保存文本数据的SDS可以重用一部分C字符串的函数

**C字符串与SDS对比**

| C字符串                                | SDS                                          |
| -------------------------------------- | -------------------------------------------- |
| 获取字符串长度复杂度为O(N)             | 获取字符串长度复杂度为O(1)                   |
| API是不安全的，可能会造成缓冲区溢出    | API是安全的，不会造成缓冲区溢出              |
| 修改字符串长度必然会需要执行内存重分配 | 修改字符串长度N次最多会需要执行N次内存重分配 |
| 只能保存文本数据                       | 可以保存文本或二进制数据                     |
| 可以使用所有`<string.h>`库中的函数     | 可以使用一部分`<string.h>`库中的函数         |

## 1.2. 链表
链表是一种比较常见的数据结构了，特点是易于插入和删除、内存利用率高、且可以灵活调整链表长度，但随机访问困难。许多高级编程语言都内置了链表的实现，但是C语言并没有实现链表，所以Redis实现了自己的链表数据结构

链表在Redis中应用的非常广，列表（List）的底层实现就是链表。此外，Redis的发布与订阅、慢查询、监视器等功能也用到了链表

链表上的节点定义如下，adlist.h/listNode

```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
} listNode;
```

链表的定义如下，adlist.h/list

```c
typedef struct list {
    // 链表头节点
    listNode *head;
    // 链表尾节点
    listNode *tail;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    // 链表所包含的节点数量
    unsigned long len;
} list;
```

每个节点listNode可以通过prev和next指针分布指向前一个节点和后一个节点组成双端链表，同时每个链表还会有一个list结构为链表提供表头指针head、表尾指针tail、以及链表长度计数器len，还有三个用于实现多态链表的类型特定函数

- dup：用于复制链表节点所保存的值
- free：用于释放链表节点所保存的值
- match：用于对比链表节点所保存的值和另一个输入值是否相等

链表的结构图如下：

![](/assets/images/posts/redis-object/redis-object-6.png)

**链表的特性**

- 双端链表：带有指向前置节点和后置节点的指针，获取这两个节点的复杂度为O(1)
- 无环：表头节点的`prev`和表尾节点的`next`都指向NULL，对链表的访问以NULL结束
- 链表长度计数器：带有`len`属性，获取链表长度的复杂度为O(1)
- 多态：链表节点使用 `void*`指针保存节点值，可以保存不同类型的值

## 1.3. 字典
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

> 当一个新的键值对要添加到字典中时，会根据键值对的键计算出哈希值和索引值，根据索引值放到对应的哈希表上，即如果索引值为0，则放到`ht[0]`哈希表上。当有两个或多个的键分配到了哈希表数组上的同一个索引时，就发生了**键冲突**的问题，哈希表使用**链地址法**来解决，即使用哈希表节点的`next`指针，将同一个索引上的多个节点连接起来。当哈希表的键值对太多或太少，就需要对哈希表进行扩展和收缩，通过`rehash`(重新散列)来执行

**rehash** 

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

将ht[0]中的数据转移到ht[1]中，在转移的过程中，需要对哈希表节点的数据重新进行哈希值计算

数据转移后的结果

![](/assets/images/posts/redis-object/redis-object-10.png)

- **释放ht[0]**

将ht[0]释放，然后将ht[1]设置成ht[0]，最后为ht[1]分配一个空白哈希表：

![](/assets/images/posts/redis-object/redis-object-11.png)

- ** 渐进式 rehash**

上面我们说到，在进行拓展或者压缩的时候，可以直接将所有的键值对rehash 到ht[1]中，这是因为数据量比较小。在实际开发过程中，**这个rehash 操作并不是一次性、集中式完成的，而是分多次、渐进式地完成的。**

渐进式rehash 的详细步骤：

1. 为ht[1] 分配空间，让字典同时持有ht[0]和ht[1]两个哈希表
2. 在几点钟维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash 开始
3. 在rehash 进行期间，每次对字典执行CRUD操作时，程序除了执行指定的操作以外，还会将ht[0]中的数据rehash 到ht[1]表中，并且将rehashidx加一
4. 当ht[0]中所有数据转移到ht[1]中时，将rehashidx 设置成-1，表示rehash 结束

采用渐进式rehash 的好处在于它采取分而治之的方式，避免了集中式rehash 带来的庞大计算量。

> rehash是以bucket(桶)为基本单位进行渐进式的数据迁移的，每步完成一个bucket的迁移，直至所有数据迁移完毕。一个bucket对应哈希表数组中的一条entry链表。新版本的dictRehash()还加入了一个最大访问空桶数(empty_visits)的限制来进一步减小可能引起阻塞的时间。

**哈希表的扩展与收缩**

当以下条件中的任意一个被满足时， 程序会自动开始对哈希表执行扩展操作：

- 服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 1 ；(哈希表中保存的key数量超过了哈希表的大小)
- 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 5 ；(保存的节点数与哈希表大小的比例超过了安全阈值)

当哈希表的负载因子小于 0.1 时， 程序自动开始对哈希表执行收缩操作。

**渐进式 rehash 执行期间的哈希表操作**

因为在进行渐进式 rehash 的过程中， 字典会同时使用 ht[0] 和 ht[1] 两个哈希表， 所以在渐进式 rehash  进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行： 比如说，  要在字典里面查找一个键的话， 程序会先在 ht[0] 里面进行查找， 如果没找到的话， 就会继续到 ht[1] 里面进行查找， 诸如此类。

另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 ht[1] 里面， 而 ht[0] 则不再进行任何添加操作： 这一措施保证了 ht[0] 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。

**渐进式rehash带来的问题**

渐进式rehash避免了redis阻塞，可以说非常完美，但是由于在rehash时，需要分配一个新的hash表，在rehash期间，同时有两个hash表在使用，会使得redis内存使用量瞬间突增，在Redis 满容状态下由于Rehash会导致大量Key驱逐。

**rehash的时机**

在redis的实现中，没有集中的将原有的key重新rehash到新的槽中，而是分解到各个命令的执行中，以及周期函数中

- **操作辅助rehash** 在redis中每一个增删改查命令中都会判断数据库字典中的哈希表是否正在进行渐进式rehash，如果是则帮助执行一次
- **定时辅助rehash** 虽然redis实现了在读写操作时，辅助服务器进行渐进式rehash操作，但是如果服务器比较空闲，redis数据库将很长时间内都一直使用两个哈希表。所以在redis周期函数中，如果发现有字典正在进行渐进式rehash操作，则会花费1毫秒的时间，帮助一起进行渐进式rehash操作

## 1.4. 跳跃表

[跳表的介绍可以看这篇文章](https://edgar615.github.io/skiplist.html)

## 1.5. 整数集合

整数集合（intset）是Redis用于保存整数值的集合抽象数据结构，可以保存类型为int16_t、int32_t、int64_t的整数值，并且保证集合中不会出现重复元素

整数集合是集合（Set）的底层实现之一，如果一个集合只包含整数值元素，且元素数量不多时，会使用整数集合作为底层实现

整数集合的定义为`inset.h/inset`

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;

```

- `contents`数组：整数集合的每个元素在数组中按值的大小从小到大排序，且不包含重复项
- `length`记录整数集合的元素数量，即contents数组长度
- `encoding`决定contents数组的真正类型，如INTSET_ENC_INT16、INTSET_ENC_INT32、INTSET_ENC_INT64

![](/assets/images/posts/redis-object/redis-object-12.png)

**整数集合的升级**

当想要添加一个新元素到整数集合中时，并且新元素的类型比整数集合现有的所有元素的类型都要长，整数集合需要先进行升级（upgrade），才能将新元素添加到整数集合里面。每次想整数集合中添加新元素都有可能会引起升级，每次升级都需要对底层数组已有的所有元素进行类型转换

升级添加新元素：

- 根据新元素类型，扩展整数集合底层数组的空间大小，并为新元素分配空间
- 把数组现有的元素都转换成新元素的类型，并将转换后的元素放到正确的位置，且要保持数组的有序性
- 添加新元素到底层数组

比如，我们现在有如下的整数集合：

![](/assets/images/posts/redis-object/redis-object-13.png)

我们现在需要插入一个32位的整数，这显然与整数集合不符合，我们将进行编码格式的转换，并为新元素分配空间：

![](/assets/images/posts/redis-object/redis-object-14.png)

第二步，将原有数据他们的数据类型转换为与新数据相同的类型：（重新分配空间后的数据）

![](/assets/images/posts/redis-object/redis-object-15.png)

第三步，将新数据添加到数组中：

![](/assets/images/posts/redis-object/redis-object-16.png)

整数集合的升级策略可以提升整数集合的灵活性，并尽可能的节约内存。另外，整数集合不支持降级，一旦升级，编码就会一直保持升级后的状态

## 1.6. 压缩列表
压缩列表（ziplist）是为了节约内存而设计的，是由一系列特殊编码的连续内存块组成的顺序性（sequential）数据结构，一个压缩列表可以包含多个节点，每个节点可以保存一个字节数组或者一个整数值

压缩列表是列表（List）和散列（Hash）的底层实现之一，一个列表只包含少量列表项，并且每个列表项是小整数值或比较短的字符串，会使用压缩列表作为底层实现（在3.2版本之后是使用quicklist实现）

**压缩列表的构成**

一个压缩列表可以包含多个节点（entry），每个节点可以保存一个字节数组或者一个整数值

![](/assets/images/posts/redis-object/redis-object-17.png)

各部分组成说明如下

- `zlbytes`：记录整个压缩列表占用的内存字节数，在压缩列表内存重分配，或者计算`zlend`的位置时使用
- `zltail`：记录压缩列表表尾节点距离压缩列表的起始地址有多少字节，通过该偏移量，可以不用遍历整个压缩列表就可以确定表尾节点的地址
- `zllen`：记录压缩列表包含的节点数量，但该属性值小于UINT16_MAX（65535）时，该值就是压缩列表的节点数量，否则需要遍历整个压缩列表才能计算出真实的节点数量
- `entryX`：压缩列表的节点
- `zlend`：特殊值0xFF（十进制255），用于标记压缩列表的末端

**压缩列表节点的构成**

每个压缩列表节点可以保存一个字节数字或者一个整数值，结构如下

![](/assets/images/posts/redis-object/redis-object-18.png)

- `previous_entry_ength`：记录压缩列表前一个字节的长度
- `encoding`：节点的encoding保存的是节点的content的内容类型
- `content`：content区域用于保存节点的内容，节点内容类型和长度由encoding决定

# 2. Redis数据存储的细节

下图是执行set hello world时，所涉及到的数据模型。

![](/assets/images/posts/redis-object/redis-object-25.png)

- **dictEntry：**Redis是Key-Value数据库，因此对每个键值对都会有一个dictEntry，里面存储了指向Key和Value的指针；next指向下一个dictEntry，与本Key-Value无关。

- **Key：**图中右上角可见，Key（“hello”）并不是直接以字符串存储，而是存储在SDS结构中。

- **redisObject：**Value(“world”)既不是直接以字符串存储，也不是像Key一样直接存储在SDS中，而是存储在redisObject中。实际上，不论Value是5种类型的哪一种，都是通过RedisObject来存储的；而RedisObject中的type字段指明了Value对象的类型，ptr字段则指向对象所在的地址。不过可以看出，字符串对象虽然经过了RedisObject的包装，但仍然需要通过SDS存储。

  实际上，RedisObject除了type和ptr字段以外，还有其它字段图中没有给出，如用于指定对象内部编码的字段

  **jemalloc：**无论是DictEntry对象，还是RedisObject、SDS对象，都需要内存分配器（如jemalloc）分配内存进行存储。以DictEntry对象为例，有3个指针组成，在64位机器下占24个字节，jemalloc会为它分配32字节大小的内存单元。

## 2.1. jemalloc

Redis在编译时便会指定内存分配器；内存分配器可以是 libc 、jemalloc或者tcmalloc，默认是jemalloc。

jemalloc作为Redis的默认内存分配器，在减小内存碎片方面做的相对比较好。jemalloc在64位系统中，将内存空间划分为小、大、巨大三个范围；每个范围内又划分了许多小的内存块单位；当Redis存储数据时，会选择大小最合适的内存块进行存储。

jemalloc划分的内存单元如下图所示：

![](/assets/images/posts/redis-object/redis-object-26.jpg)

例如，如果需要存储大小为130字节的对象，jemalloc会将其放入160字节的内存单元中。

下面是jemalloc size class categories,左边是用户申请内存范围,右边是实际申请的内存大小
```
1 – 4 size class:4 　　
5 – 8 size class:8 　　
9 – 16 size class:16 　　
17 – 32 size class:32 　　
33 – 48 size class:48 　　
49 – 64 size class:64 　　
65 – 80 size class:80 　　
81 – 96 size class:96 　　
97 – 112 size class:112 　　
113 – 128 size class:128 　　
129 – 192 size class:192 　　
193 – 256 size class:256 　　
257 – 320 size class:320 　　
321 – 384 size class:384 　　
385 – 448 size class:448 　　
449 – 512 size class:512 　　
513 – 768 size class:768 　　
769 – 1024 size class:1024 　　
1025 – 1280 size class:1280 　　
1281 – 1536 size class:1536 　　
1537 – 1792 size class:1792 　　
1793 – 2048 size class:2048 　　
2049 – 2304 size class:2304 　　
2305 – 2560 size class:2560
```

## 2.2. 对象

上面介绍了Redis的主要底层数据结构，包括简单动态字符串（SDS）、链表、字典、跳跃表、整数集合、压缩列表。但是Redis并没有直接使用这些数据结构来构建键值对数据库，而是基于这些数据结构创建了一个对象系统，也就是我们所熟知的可API操作的Redis那些数据类型，如字符串(String)、列表(List)、散列(Hash)、集合(Set)、有序集合(Sorted Set)

根据对象的类型可以判断一个对象是否可以执行给定的命令，也可针对不同的使用场景，对象设置有多种不同的数据结构实现，从而优化对象在不同场景下的使用效率

Redis 的 redisObject 结构的定义如下所示。

```c
typedef struct redisObject {    
    unsigned type:4;    
    unsigned encoding:4;    
    unsigned lru:LRU_BITS;     
    int refcount;    
    void *ptr;
} robj;
```

其中 type 是对象类型，包括REDISSTRING, REDISLIST, REDISHASH, REDISSET 和 REDIS_ZSET。

encoding是指对象使用的数据结构，全集如下

| 类型         | 编码                      | BOJECT ENCODING 命令输出 | 对象                                           |
| ------------ | ------------------------- | ------------------------ | ---------------------------------------------- |
| REDIS_STRING | REDIS_ENCODING_INT        | "int"                    | 使用整数值实现的字符串对象                     |
| REDIS_STRING | REDIS_ENCODING_EMBSTR     | "embstr"                 | 使用embstr编码的简单动态字符串实现的字符串对象 |
| REDIS_STRING | REDIS_ENCODING_RAW        | "raw"                    | 使用简单动态字符串实现的字符串对象             |
| REDIS_LIST   | REDIS_ENCODING_ZIPLIST    | "ziplist"                | 使用压缩列表实现的列表对象                     |
| REDIS_LIST   | REDIS_ENCODING_LINKEDLIST | '"linkedlist'            | 使用双端链表实现的列表对象                     |
| REDIS_HASH   | REDIS_ENCODING_ZIPLIST    | "ziplist"                | 使用压缩列表实现的哈希对象                     |
| REDIS_HASH   | REDIS_ENCODING_HT         | "hashtable"              | 使用字典实现的哈希对象                         |
| REDIS_SET    | REDIS_ENCODING_INTSET     | "intset"                 | 使用整数集合实现的集合对象                     |
| REDIS_SET    | REDIS_ENCODING_HT         | "hashtable"              | 使用字典实现的集合对象                         |
| REDIS_ZSET   | REDIS_ENCODING_ZIPLIST    | "ziplist"                | 使用压缩列表实现的有序集合对象                 |
| REDIS_ZSET   | REDIS_ENCODING_SKIPLIST   | "skiplist"               | 使用跳跃表表实现的有序集合对象                 |

### 2.2.1. 字符串对象

我们首先来看字符串对象的实现，如下图所示

![](/assets/images/posts/redis-object/redis-object-19.jpeg)

如果一个字符串对象保存的是一个字符串值，并且长度大于32字节，那么该字符串对象将使用 SDS 进行保存，并将对象的编码设置为 raw，如图的上半部分所示。
如果字符串的长度小于32字节，那么字符串对象将使用embstr 编码方式来保存。

如果符串值是整型时，这个值使用long整型（8字节）表示

embstr 编码是专门用于保存短字符串的一种优化编码方式，这个编码的组成和 raw 编码一致，都使用 redisObject 结构和 sdshdr 结构来保存字符串，如上图的下半部所示。

但是 raw 编码会调用两次内存分配来分别创建上述两个结构，而 embstr 则通过一次内存分配来分配一块连续的空间，空间中一次包含两个结构。

embstr 只需一次内存分配，而且在同一块连续的内存中，更好的利用缓存带来的优势，但是 embstr 是只读的，不能进行修改，当一个 embstr 编码的字符串对象进行 append 操作时， redis 会现将其转变为 raw 编码再进行操作。

### 2.2.2. 列表对象

列表对象的编码可以是 ziplist 或 linkedlist。如下图所示

![](/assets/images/posts/redis-object/redis-object-20.jpeg)

当列表对象可以同时满足以下两个条件时，列表对象使用 ziplist 编码：

- 列表对象保存的所有字符串元素的长度都小于 64 字节。
- 列表对象保存的元素数量数量小于 512 个。

不能满足这两个条件的列表对象需要使用 linkedlist 编码或者转换为 linkedlist 编码。

```
list-max-ziplist-entries 512
list-max-ziplist-value 64
```

### 2.2.3. 哈希对象

哈希对象的编码可以使用 ziplist 或 dict。

当哈希对象使用压缩队列作为底层实现时，程序将键值对紧挨着插入到压缩队列中，保存键的节点在前，保存值的节点在后。如下图的上半部分所示，该哈希有两个键值对，分别是 `name:Tom` 和 `age:25`。

![](/assets/images/posts/redis-object/redis-object-21.jpeg)

当哈希对象可以同时满足以下两个条件时，哈希对象使用 ziplist 编码:

- 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节。
- 哈希对象保存的键值对数量小于512个。

不能满足这两个条件的哈希对象需要使用 dict 编码或者转换为 dict 编码。

```
hash-max-zipmap-entries 64
hash-max-zipmap-value 512
```

### 2.2.4. 集合对象

集合对象的编码可以使用 intset 或者 dict。

intset 编码的集合对象使用整数集合最为底层实现，所有元素都被保存在整数集合里边。

而使用 dict 进行编码时，字典的每一个键都是一个字符串对象，每个字符串对象就是一个集合元素，而字典的值全部都被设置为NULL。如下图所示

![](/assets/images/posts/redis-object/redis-object-22.jpeg)

当集合对象可以同时满足以下两个条件时，对象使用 intset 编码:

- 集合对象保存的所有元素都是整数值。
- 集合对象保存的元素数量不超过512个。

否则使用 dict 进行编码。


### 2.2.5. 有序集合对象

有序集合的编码可以为 ziplist 或者 skiplist。

有序集合使用 ziplist 编码时，每个集合元素使用两个紧挨在一起的压缩列表节点表示，前一个节点是元素的值，第二个节点是元素的分值，也就是排序比较的数值。

压缩列表内的集合元素按照分值从小到大进行排序，如下图上半部分所示。

有序集合使用 skiplist 编码时使用 zset 结构作为底层实现，一个 zet 结构同时包含一个字典和一个跳跃表。

其中，跳跃表按照分值从小到大保存所有元素，每个跳跃表节点保存一个元素，其score值是元素的分值。而字典则创建一个一个从成员到分值的映射，字典的键是集合成员的值，字典的值是集合成员的分值。通过字典可以在O(1)复杂度查找给定成员的分值。如下图所示。

跳跃表和字典中的集合元素值对象都是共享的，所以不会额外消耗内存。

![](/assets/images/posts/redis-object/redis-object-23.jpeg)

当有序集合对象可以同时满足以下两个条件时，对象使用 ziplist 编码：

- 有序集合保存的元素数量少于128个；
- 有序集合保存的所有元素的长度都小于64字节。

否则使用 skiplist 编码。

### 2.2.6. 数据库键空间

Redis 服务器都有多个 Redis 数据库，每个Redis 数据都有自己独立的键值空间。每个 Redis 数据库使用 dict 保存数据库中所有的键值对。

![](/assets/images/posts/redis-object/redis-object-24.jpeg)

键空间的键也就是数据库的键，每个键都是一个字符串对象，而值对象可能为字符串对象、列表对象、哈希表对象、集合对象和有序集合对象中的一种对象。

除了键空间，Redis 也使用 dict 结构来保存键的过期时间，其键是键空间中的键值，而值是过期时间，如上图所示。

通过过期字典，Redis 可以直接判断一个键是否过期，首先查看该键是否存在于过期字典，如果存在，则比较该键的过期时间和当前服务器时间戳，如果大于，则该键过期，否则未过期。

### 2.2.7. refcount与共享对象
refcount记录的是该对象被引用的次数，类型为整型。refcount的作用，主要在于对象的引用计数和内存回收：


- 当创建新对象时，refcount初始化为1；
- 当有新程序使用该对象时，refcount加1；
- 当对象不再被一个新程序使用时，refcount减1；
- 当refcount变为0时，对象占用的内存会被释放。

Redis中被多次使用的对象(refcount>1)称为共享对象。Redis为了节省内存，当有一些对象重复出现时，新的程序不会创建新的对象，而是仍然使用原来的对象。这个被重复使用的对象，就是共享对象。目前共享对象仅支持整数值的字符串对象。

**共享对象的具体实现**

Redis的共享对象目前只支持整数值的字符串对象。之所以如此，实际上是对内存和CPU（时间）的平衡：共享对象虽然会降低内存消耗，但是判断两个对象是否相等却需要消耗额外的时间。

- 对于整数值，判断操作复杂度为O(1)；
- 对于普通字符串，判断复杂度为O(n)；
- 而对于哈希、列表、集合和有序集合，判断的复杂度为O(n^2)。

虽然共享对象只能是整数值的字符串对象，但是5种类型都可能使用共享对象（如哈希、列表等的元素可以使用）。


就目前的实现来说，Redis服务器在初始化时，会创建10000个字符串对象，值分别是0~9999的整数值；当Redis需要使用值为0~9999的字符串对象时，可以直接使用这些共享对象。这样在系统存储了大量数值下，也能一定程度上节省内存并且提高性能，这个参数值 n 的设置需要修改源代码中的一行宏定义 REDIS_SHARED_INTEGERS，该值 默认是 10000，可以根据自己的需要进行修改，修改后重新编译就可以了。

共享对象的引用次数可以通过object refcount命令查看，

# 3. 复杂度

**数据结构的时间复杂度**

![](/assets/images/posts/redis-object/redis-object-27.jpg)

**不同操作的复杂度**

合类型的操作类型很多，有读写单个集合元素的，例如 HGET、HSET，也有操作多个元素的，例如 SADD，还有对整个集合进行遍历操作的，例如 SMEMBERS。这么多操作，它们的复杂度也各不相同。而复杂度的高低又是我们选择集合类型的重要依据

- **单元素操作**

是指每一种集合类型对单个数据实现的增删改查操作。例如，Hash 类型的 HGET、HSET 和 HDEL，Set 类型的 SADD、SREM、SRANDMEMBER 等。这些操作的复杂度由集合采用的数据结构决定，例如，HGET、HSET 和 HDEL 是对哈希表做操作，所以它们的复杂度都是 O(1)；Set 类型用哈希表作为底层数据结构时，它的 SADD、SREM、SRANDMEMBER 复杂度也是 O(1)。

集合类型支持同时对多个元素进行增删改查，例如 Hash 类型的 HMGET 和 HMSET，Set 类型的 SADD 也支持同时增加多个元素。此时，这些操作的复杂度，就是由单个元素操作复杂度和元素个数决定的。例如，HMSET 增加 M 个元素时，复杂度就从 O(1) 变成 O(M) 了。

- **范围操作**

范围操作，是指集合类型中的遍历操作，可以返回集合中的所有数据，比如 Hash 类型的 HGETALL 和 Set 类型的 SMEMBERS，或者返回一个范围内的部分数据，比如 List 类型的 LRANGE 和 ZSet 类型的 ZRANGE。这类操作的复杂度一般是 O(N)，比较耗时，**我们应该尽量避免。**

Redis 从 2.8 版本开始提供了 SCAN 系列操作（包括 HSCAN，SSCAN 和 ZSCAN），这类操作实现了渐进式遍历，每次只返回有限数量的数据。这样一来，相比于 HGETALL、SMEMBERS 这类操作来说，就避免了一次性返回所有元素而导致的 Redis 阻塞。

- **统计操作**

统计操作，是指集合类型对集合中所有元素个数的记录，例如 LLEN 和 SCARD。这类操作复杂度只有 O(1)，这是因为当集合类型采用压缩列表、双向链表、整数数组这些数据结构时，这些结构中专门记录了元素的个数统计，因此可以高效地完成相关操作。

- **例外情况**

例外情况，是指某些数据结构的特殊记录，例如压缩列表和双向链表都会记录表头和表尾的偏移量。这样一来，对于 List 类型的 LPOP、RPOP、LPUSH、RPUSH 这四个操作来说，它们是在列表的头尾增删元素，这就可以通过偏移量直接定位，所以它们的复杂度也只有 O(1)，可以实现快速操作。

Redis 之所以能快速操作键值对，一方面是因为 O(1) 复杂度的哈希表被广泛使用，包括 String、Hash 和 Set，它们的操作复杂度基本由哈希表决定，另一方面，Sorted Set 也采用了 O(logN) 复杂度的跳表。不过，集合类型的范围操作，因为要遍历底层数据结构，复杂度通常是 O(N)。可以用 SCAN 来代替，避免在 Redis 内部产生费时的全集合遍历操作。当然，我们不能忘了复杂度较高的 List 类型，它的两种底层实现结构：双向链表和压缩列表的操作复杂度都是 O(N)。因此，我的建议是：因地制宜地使用 List 类型。例如，既然它的 POP/PUSH 效率很高，那么就将它主要用于 FIFO 队列场景，而不是作为一个可以随机读写的集合。

# 3. 参考资料

https://juejin.im/post/5d71d3bee51d453b5f1a04f1

https://cloud.tencent.com/developer/article/1056182

https://cloud.tencent.com/developer/article/1056184

https://mp.weixin.qq.com/s/fO0yoHGqtFH5lpu6688h2w

https://mp.weixin.qq.com/s/4wpsg8BDwGVADWb3WpSzpA

《Redis核心技术与实战》