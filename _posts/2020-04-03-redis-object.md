---
layout: post
title: redis对象
date: 2020-04-05
categories:
    - redis
comments: true
permalink: redis-object.html
---

Redis有六种基础数据结构：动态字符串，链表，字典，跳跃表，整数集合和压缩列表。Redis 并没有直接使用这些数据结构来实现键值对数据库， 而是基于这些数据结构创建了一个对象系统， 这个系统包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象这五种类型的对象， 每种对象都用到了**至少一种**基础数据结构。

通过这五种不同类型的对象， Redis 可以在执行命令之前， 根据对象的类型来判断一个对象是否可以执行给定的命令。 使用对象的另一个好处是， 我们可以针对不同的使用场景， 为对象设置多种不同的数据结构实现， 从而优化对象在不同场景下的使用效率。

除此之外， Redis 的对象系统还实现了基于引用计数技术的内存回收机制： 当程序不再使用某个对象的时候， 这个对象所占用的内存就会被自动释放； 另外， Redis 还通过引用计数技术实现了对象共享机制， 这一机制可以在适当的条件下， 通过让多个数据库键共享同一个对象来节约内存。

最后， Redis 的对象带有访问时间记录信息， 该信息可以用于计算数据库键的空转时长， 在服务器启用了 `maxmemory` 功能的情况下， 空转时长较大的那些键可能会优先被服务器删除。

# 1. Redis数据存储的细节

下图是执行set hello world时，所涉及到的数据模型。

![](/assets/images/posts/redis-object/redis-object-25.png)

- **dictEntry：**Redis是Key-Value数据库，因此对每个键值对都会有一个dictEntry，里面存储了指向Key和Value的指针；next指向下一个dictEntry，与本Key-Value无关。

- **Key：**图中右上角可见，Key（“hello”）并不是直接以字符串存储，而是存储在SDS结构中。

- **redisObject：**Value(“world”)既不是直接以字符串存储，也不是像Key一样直接存储在SDS中，而是存储在redisObject中。实际上，**不论Value是5种类型的哪一种，都是通过RedisObject来存储的**；而RedisObject中的type字段指明了Value对象的类型，ptr字段则指向对象所在的地址。不过可以看出，字符串对象虽然经过了RedisObject的包装，但仍然需要通过SDS存储。

实际上，RedisObject除了type和ptr字段以外，还有其它字段图中没有给出，如用于指定对象内部编码的字段。

无论是DictEntry对象，还是RedisObject、SDS对象，都需要内存分配器（如jemalloc）分配内存进行存储。以DictEntry对象为例，有3个指针组成，在64位机器下占24个字节，jemalloc会为它分配32字节大小的内存单元。

# 2. jemalloc

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

# 3. RedisObject 

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

Redis 中的每个对象都由一个 `redisObject` 结构表示， 该结构中和保存数据有关的三个属性分别是 `type` 属性、 `encoding` 属性和 `ptr` 属性：

其中 type 是对象类型，包括REDISSTRING, REDISLIST, REDISHASH, REDISSET 和 REDIS_ZSET。

对于 Redis 数据库保存的键值对来说， 键总是一个字符串对象， 而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种， 因此：

- 当我们称呼一个数据库键为“字符串键”时， 我们指的是“这个数据库键所对应的值为字符串对象”；
- 当我们称呼一个键为“列表键”时， 我们指的是“这个数据库键所对应的值为列表对象”，

对象的 `ptr` 指针指向对象的底层实现数据结构， 而这些数据结构由对象的 `encoding` 属性决定。

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

使用 OBJECT ENCODING 命令可以查看一个数据库键的值对象的编码

```
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> type foo
string
127.0.0.1:6379> object encoding foo
"embstr"
127.0.0.1:6379> set num 1
OK
127.0.0.1:6379> type num
string
127.0.0.1:6379> object encoding num
"int"
```

通过 `encoding` 属性来设定对象所使用的编码， 而不是为特定类型的对象关联一种固定的编码， 极大地提升了 Redis 的灵活性和效率， 因为 Redis 可以根据不同的使用场景来为一个对象设置不同的编码， 从而优化对象在某一场景下的效率。

# 4. 字符串对象

字符串对象的编码可以是 `int` 、 `raw` 或者 `embstr` 。

我们首先来看字符串对象的实现，如下图所示

![](/assets/images/posts/redis-object/redis-object-19.jpeg)

- 如果一个字符串对象保存的是一个字符串值，并且长度大于44字节，那么该字符串对象将使用 SDS 进行保存，并将对象的编码设置为 **raw**，如图的上半部分所示。(而在3.2版本之前，是39字节为分界。)
- 如果字符串的长度小于44字节，那么字符串对象将使用**embstr** 编码方式来保存。
- 如果符串值是整型时，这个值使用**long**整型（8字节）表示

```
127.0.0.1:6379> set num 10086
OK
127.0.0.1:6379> object encoding num
"int"
127.0.0.1:6379> SET story "Long, long, long ago there lived a king ...."
OK
127.0.0.1:6379> strlen story
(integer) 44
127.0.0.1:6379> object encoding story
"embstr"
127.0.0.1:6379> SET story "Long, long, long ago there lived a king ....."
OK
127.0.0.1:6379> object encoding story
"raw"
127.0.0.1:6379> strlen story
(integer) 45

```

embstr 编码是专门用于保存短字符串的一种优化编码方式，这个编码的组成和 raw 编码一致，都使用 redisObject 结构和 sdshdr 结构来保存字符串，如上图的下半部所示。

`embstr` 编码的字符串对象在执行命令时， 产生的效果和 `raw` 编码的字符串对象执行命令时产生的效果是相同的， 但使用 `embstr` 编码的字符串对象来保存短字符串值有以下好处：

1. `embstr` 编码将创建字符串对象所需的内存分配次数从 `raw` 编码的两次降低为一次。raw 编码会调用两次内存分配来分别创建上述两个结构，而 embstr 则通过一次内存分配来分配一块连续的空间，空间中一次包含两个结构。
2. 释放 `embstr` 编码的字符串对象只需要调用一次内存释放函数， 而释放 `raw` 编码的字符串对象需要调用两次内存释放函数。
3. 因为 `embstr` 编码的字符串对象的所有数据都保存在一块连续的内存里面， 所以这种编码的字符串对象比起 `raw` 编码的字符串对象能够更好地利用缓存带来的优势。

**编码的转换**

`int` 编码的字符串对象和 `embstr` 编码的字符串对象在条件满足的情况下， 会被转换为 `raw` 编码的字符串对象。

对于 `int` 编码的字符串对象来说， 如果我们向对象执行了一些命令， 使得这个对象保存的不再是整数值， 而是一个字符串值， 那么字符串对象的编码将从 `int` 变为 `raw` 。

```
127.0.0.1:6379> object encoding num
"int"
127.0.0.1:6379> append num "test"
(integer) 9
127.0.0.1:6379> object encoding num
"raw"
```

因为 Redis 没有为 `embstr` 编码的字符串对象编写任何相应的修改程序 （只有 `int` 编码的字符串对象和 `raw` 编码的字符串对象有这些程序）， 所以 `embstr` 编码的字符串对象实际上是只读的： 当我们对 `embstr` 编码的字符串对象执行任何修改命令时， 程序会先将对象的编码从 `embstr` 转换成 `raw` ， 然后再执行修改命令； 因为这个原因， `embstr` 编码的字符串对象在执行修改命令之后， 总会变成一个 `raw` 编码的字符串对象。

```
127.0.0.1:6379> object encoding foo
"embstr"
127.0.0.1:6379> append foo "baz"
(integer) 6
127.0.0.1:6379> object encoding foo
"raw"
127.0.0.1:6379> strlen foo
(integer) 6
```

# 5. 列表对象

列表对象的编码可以是 `ziplist` 或者 `linkedlist` 。

如下图所示

![](/assets/images/posts/redis-object/redis-object-20.jpeg)

**编码转换**

当列表对象可以同时满足以下两个条件时，列表对象使用 ziplist 编码：

- 列表对象保存的所有字符串元素的长度都小于 64 字节。
- 列表对象保存的元素数量数量小于 512 个。

不能满足这两个条件的列表对象需要使用 linkedlist 编码。

以上两个条件的上限值是可以修改的。

```
list-max-ziplist-entries 512
list-max-ziplist-value 64
```

对于使用 `ziplist` 编码的列表对象来说， 当使用 `ziplist` 编码所需的两个条件的任意一个不能被满足时， 对象的编码转换操作就会被执行： 原本保存在压缩列表里的所有列表元素都会被转移并保存到双端链表里面， 对象的编码也会从 `ziplist` 变为 `linkedlist` 。

> redis3.2之后已经改成了quicklist实现，所以没有测试

```
127.0.0.1:6379> rpush list "hello" "world"
(integer) 2
127.0.0.1:6379> llen list
(integer) 2
127.0.0.1:6379> object encoding list
"quicklist"
```

# 6.  哈希对象

哈希对象的编码可以使用 ziplist 或 dict。

`ziplist` 编码的哈希对象使用压缩列表作为底层实现， 每当有新的键值对要加入到哈希对象时， 程序会先将保存了键的压缩列表节点推入到压缩列表表尾， 然后再将保存了值的压缩列表节点推入到压缩列表表尾， 因此：

- 保存了同一键值对的两个节点总是紧挨在一起， 保存键的节点在前， 保存值的节点在后；
- 先添加到哈希对象中的键值对会被放在压缩列表的表头方向， 而后来添加到哈希对象中的键值对会被放在压缩列表的表尾方向。

如下图的上半部分所示，该哈希有两个键值对，分别是 `name:Tom` 和 `age:25`。

![](/assets/images/posts/redis-object/redis-object-21.jpeg)

**编码转换**

当哈希对象可以同时满足以下两个条件时，哈希对象使用 ziplist 编码:

- 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节。
- 哈希对象保存的键值对数量小于512个。

不能满足这两个条件的哈希对象需要使用 dict 编码或者转换为 dict 编码。

```
hash-max-zipmap-entries 64
hash-max-zipmap-value 512
```

```
127.0.0.1:6379> HSET book name "Mastering C++ in 21 days"
(integer) 1
127.0.0.1:6379> object encoding book
"ziplist"
127.0.0.1:6379> HSET book long_long_long_long_long_long_long_long_long_long_long_description "content"
(integer) 1
127.0.0.1:6379> object encoding book
"hashtable"
```

除了键的长度太大会引起编码转换之外， 值的长度太大也会引起编码转换

```
127.0.0.1:6379> HSET blah greeting "hello world"
(integer) 1
127.0.0.1:6379> object encoding blah
"ziplist"
127.0.0.1:6379> HSET blah story "many string ... many string ... many string ... many string ... many"
(integer) 1
127.0.0.1:6379> object encoding blah
"hashtable"
```

包含的键值对数量过多也会引起编码转换

```
127.0.0.1:6379> EVAL "for i=1, 512 do redis.call('HSET', KEYS[1], i, i) end" 1 "numbers"
(nil)
127.0.0.1:6379> HLEN numbers
(integer) 512
127.0.0.1:6379> OBJECT ENCODING numbers
"ziplist"
127.0.0.1:6379> HMSET numbers "key" "value"
OK
127.0.0.1:6379> HLEN numbers
(integer) 513
127.0.0.1:6379> OBJECT ENCODING numbers
"hashtable"
```

# 7. 集合对象

集合对象的编码可以使用 intset 或者 dict。

intset 编码的集合对象使用整数集合最为底层实现，所有元素都被保存在整数集合里边。

而使用 dict 进行编码时，字典的每一个键都是一个字符串对象，每个字符串对象就是一个集合元素，而字典的值全部都被设置为NULL。如下图所示

![](/assets/images/posts/redis-object/redis-object-22.jpeg)

**编码转换**

当集合对象可以同时满足以下两个条件时，对象使用 intset 编码:

- 集合对象保存的所有元素都是整数值。
- 集合对象保存的元素数量不超过512个。

否则使用 dict 进行编码。

```
127.0.0.1:6379> SADD numbers 1 3 5
(integer) 3
127.0.0.1:6379> OBJECT ENCODING numbers
"intset"
127.0.0.1:6379> SADD numbers "seven"
(integer) 1
127.0.0.1:6379> OBJECT ENCODING numbers
"hashtable"
```

```
127.0.0.1:6379> EVAL "for i=1, 512 do redis.call('SADD', KEYS[1], i) end" 1 integers
(nil)
127.0.0.1:6379> scard integers
(integer) 512
127.0.0.1:6379> OBJECT ENCODING integers
"intset"
127.0.0.1:6379> SADD integers 10086
(integer) 1
127.0.0.1:6379> scard integers
(integer) 513
127.0.0.1:6379> OBJECT ENCODING integers
"hashtable"
```

# 8. 有序集合对象

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

**编码转换**

```
127.0.0.1:6379> EVAL "for i=1, 128 do redis.call('ZADD', KEYS[1], i, i) end" 1 numbers
(nil)
127.0.0.1:6379> zcard numbers
(integer) 128
127.0.0.1:6379> OBJECT ENCODING numbers
"ziplist"
127.0.0.1:6379> ZADD numbers 3.14 pi
(integer) 1
127.0.0.1:6379> zcard numbers
(integer) 129
127.0.0.1:6379> OBJECT ENCODING numbers
"skiplist"
```

```
127.0.0.1:6379> ZADD blah 1.0 www
(integer) 1
127.0.0.1:6379> OBJECT ENCODING blah
"ziplist"
127.0.0.1:6379> ZADD blah 2.0 oooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo
(integer) 1
127.0.0.1:6379> OBJECT ENCODING blah
"skiplist"
```

# 9. 数据库键空间

Redis 服务器都有多个 Redis 数据库，每个Redis 数据都有自己独立的键值空间。每个 Redis 数据库使用 dict 保存数据库中所有的键值对。

![](/assets/images/posts/redis-object/redis-object-24.jpeg)

键空间的键也就是数据库的键，每个键都是一个字符串对象，而值对象可能为字符串对象、列表对象、哈希表对象、集合对象和有序集合对象中的一种对象。

除了键空间，Redis 也使用 dict 结构来保存键的过期时间，其键是键空间中的键值，而值是过期时间，如上图所示。

通过过期字典，Redis 可以直接判断一个键是否过期，首先查看该键是否存在于过期字典，如果存在，则比较该键的过期时间和当前服务器时间戳，如果大于，则该键过期，否则未过期。

# 10. refcount与共享对象

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

# 11. 参考资料

https://juejin.im/post/5d71d3bee51d453b5f1a04f1

https://mp.weixin.qq.com/s/fO0yoHGqtFH5lpu6688h2w

https://mp.weixin.qq.com/s/4wpsg8BDwGVADWb3WpSzpA

http://redisbook.com/preview/object/object.html

http://redisbook.com/preview/object/string.html

http://redisbook.com/preview/object/list.html

http://redisbook.com/preview/object/hash.html

http://redisbook.com/preview/object/sorted_set.html

《Redis核心技术与实战》