---
layout: post
title: redis数据结构（1）- 动态字符串
date: 2020-04-02
categories:
    - redis
comments: true
permalink: redis-object-sds.html
---

>  Redis有六种基础数据结构：动态字符串，链表，字典，跳跃表，整数集合和压缩列表。

# 1. 动态字符串

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

**通过SDS的结构可以看出，buf数组的长度=free+len+1（其中1表示字符串结尾的空字符）；所以，一个SDS结构占据的空间为：free所占长度+len所占长度+ buf数组的长度**

# 2. **SDS与C字符串的区别**

C语言使用长度为N+1的字符数组来表示长度为N的字符串，字符数组的最后一个元素为空字符'\0'。

![](/assets/images/posts/redis-object/redis-object-31.png)

但是这种简单的字符串表示方法并不能满足Redis对于字符串在安全性、效率以及功能方面的要求，

那么使用SDS，会有哪些好处呢

## 2.1. **常数复杂度获取字符串长度**

因为 C 字符串并不记录自身的长度信息， 所以为了获取一个 C 字符串的长度， 程序必须遍历整个字符串， 对遇到的每个字符进行计数， 直到遇到代表字符串结尾的空字符为止， 这个操作的复杂度为 O(N) 。

而SDS结构中本身就有记录字符串长度的len属性，所有复杂度为O(1)。Redis将获取字符串长度所需的复杂度从O(N)降到了O(1)，确保获取字符串长度的工作不会成为Redis的性能瓶颈

设置和更新 SDS 长度的工作是由 SDS 的 API 在执行时自动完成的， 使用 SDS 无须进行任何手动修改长度的工作。通过使用 SDS 而不是 C 字符串， Redis 将获取字符串长度所需的复杂度从 O(N) 降低到了 O(1) ， 这确保了获取字符串长度的工作不会成为 Redis 的性能瓶颈。

## 2.2. 杜绝缓冲区溢出

C字符串不记录自身的长度，每次增长或缩短一个字符串，都要对底层的字符数组进行一次内存重分配操作。如果是拼接append操作之前没有通过内存重分配来扩展底层数据的空间大小，就会产生缓存区溢出；如果是截断trim操作之后没有通过内存重分配来释放不再使用的空间，就会产生内存泄漏。

举个例子， `<string.h>/strcat` 函数可以将 `src` 字符串中的内容拼接到 `dest` 字符串的末尾：

```
char *strcat(char *dest, const char *src);
```

因为 C 字符串不记录自身的长度， 所以 `strcat` 假定用户在执行这个函数时， 已经为 `dest` 分配了足够多的内存， 可以容纳 `src` 字符串中的所有内容， 而一旦这个假定不成立时， 就会产生缓冲区溢出。

举个例子， 假设程序里有两个在内存中紧邻着的 C 字符串 `s1` 和 `s2` ， 其中 `s1` 保存了字符串 `"Redis"` ， 而 `s2` 则保存了字符串 `"MongoDB"` ，

![](/assets/images/posts/redis-object/redis-object-2.png)

如果我们现在将s1 的内容修改为`redis cluster`，但是又忘了重新为s1 分配足够的空间，这时候就会出现以下问题：

![](/assets/images/posts/redis-object/redis-object-3.png)

我们可以看到，原本s2 中的内容已经被S1的内容给占领了，s2 现在为 cluster，而不是`Mongodb` 

**SDS通过未使用空间解除了字符串长度和底层数据长度的关联**，3.0版本是用`free`属性记录未使用空间，3.2版本则是`alloc`属性记录总的分配字节数量。通过未使用空间，SDS实现了**空间预分配**和**惰性空间释放**两种优化的空间分配策略，解决了字符串拼接和截取的空间问题

Redis 中SDS 的**空间分配策略**完全杜绝了发生缓冲区溢出的可能性： 当 SDS API 需要对 SDS 进行修改时， API 会先检查 SDS 的空间是否满足修改所需的要求， 如果不满足的话， API 会自动将 SDS 的空间扩展至执行修改所需的大小， 然后才执行实际的修改操作， 所以使用 SDS 既不需要手动修改 SDS 的空间大小， 也不会出现前面所说的缓冲区溢出问题。

SDS 的 API 里面也有一个用于执行拼接操作的 `sdscat` 函数， 它可以将一个 C 字符串拼接到给定 SDS 所保存的字符串的后面， 但是在执行拼接操作之前， `sdscat` 会先检查给定 SDS 的空间是否足够， 如果不够的话， `sdscat` 就会先扩展 SDS 的空间， 然后才执行拼接操作。

比如说， 如果我们执行：

```
sdscat(s, " Cluster");
```

其中 SDS 值 `s` 如下图所示

![](/assets/images/posts/redis-object/redis-object-4.png)

那么 `sdscat` 将在执行拼接操作之前检查 `s` 的长度是否足够， 在发现 `s` 目前的空间不足以拼接 `" Cluster"` 之后， `sdscat` 就会先扩展 `s` 的空间， 然后才执行拼接 `" Cluster"` 的操作， 拼接操作完成之后的 SDS 如下图所示。

![](/assets/images/posts/redis-object/redis-object-5.png)

`sdscat` 不仅对这个 SDS 进行了拼接操作， 它还为 SDS 分配了 `13` 字节的未使用空间

## 2.3. 减少修改字符串时带来的内存重分配次数

因为 C 字符串并不记录自身的长度， 所以对于一个包含了 `N` 个字符的 C 字符串来说， 这个 C 字符串的底层实现总是一个 `N+1` 个字符长的数组（额外的一个字符空间用于保存空字符）。

因为 C 字符串的长度和底层数组的长度之间存在着这种关联性， 所以每次增长或者缩短一个 C 字符串， 程序都总要对保存这个 C 字符串的数组进行一次内存重分配操作：

- 如果程序执行的是增长字符串的操作， 比如拼接操作（append）， 那么在执行这个操作之前， 程序需要先通过内存重分配来扩展底层数组的空间大小 —— 如果忘了这一步就会产生缓冲区溢出。
- 如果程序执行的是缩短字符串的操作， 比如截断操作（trim）， 那么在执行这个操作之后， 程序需要通过内存重分配来释放字符串不再使用的那部分空间 —— 如果忘了这一步就会产生内存泄漏。

举个例子， 如果我们持有一个值为 `"Redis"` 的 C 字符串 `s` ， 那么为了将 `s` 的值改为 `"Redis Cluster"` ， 在执行：

```
strcat(s, " Cluster");
```

之前， 我们需要先使用内存重分配操作， 扩展 `s` 的空间。

之后， 如果我们又打算将 `s` 的值从 `"Redis Cluster"` 改为 `"Redis Cluster Tutorial"` ， 那么在执行：

```
strcat(s, " Tutorial");
```

之前， 我们需要再次使用内存重分配扩展 `s` 的空间， 诸如此类。

因为内存重分配涉及复杂的算法， 并且可能需要执行系统调用， 所以它通常是一个比较耗时的操作：

- 在一般程序中， 如果修改字符串长度的情况不太常出现， 那么每次修改都执行一次内存重分配是可以接受的。
- 但是 Redis 作为数据库， 经常被用于速度要求严苛、数据被频繁修改的场合， 如果每次修改字符串的长度都需要执行一次内存重分配的话， 那么光是执行内存重分配的时间就会占去修改字符串所用时间的一大部分， **如果这种修改频繁地发生的话， 可能还会对性能造成影响。**

为了避免 C 字符串的这种缺陷， SDS 通过未使用空间解除了字符串长度和底层数组长度之间的关联： 在 SDS 中， `buf` 数组的长度不一定就是字符数量加一， 数组里面可以包含未使用的字节， 而这些字节的数量就由 SDS 的 `free` 属性记录。

通过未使用空间， SDS 实现了**空间预分配**和**惰性空间**释放两种优化策略。

**空间预分配**

空间预分配用于优化 SDS 的字符串增长操作： 当 SDS 的 API 对一个 SDS 进行修改， 并且需要对 SDS 进行空间扩展的时候， 程序不仅会为 SDS 分配修改所必须要的空间， 还会为 SDS 分配额外的未使用空间。

其中， 额外分配的未使用空间数量由以下公式决定：

- 如果对 SDS 进行修改之后， SDS 的长度（也即是 `len` 属性的值）将小于 `1 MB` ， 那么程序分配和 `len` 属性同样大小的未使用空间， 这时 SDS `len` 属性的值将和 `free` 属性的值相同。 举个例子， 如果进行修改之后， SDS 的 `len` 将变成 `13` 字节， 那么程序也会分配 `13` 字节的未使用空间， SDS 的 `buf` 数组的实际长度将变成 `13 + 13 + 1 = 27` 字节（额外的一字节用于保存空字符）。
- 如果对 SDS 进行修改之后， SDS 的长度将大于等于 `1 MB` ， 那么程序会分配 `1 MB` 的未使用空间。 举个例子， 如果进行修改之后， SDS 的 `len` 将变成 `30 MB` ， 那么程序会分配 `1 MB` 的未使用空间， SDS 的 `buf` 数组的实际长度将为 `30 MB + 1 MB + 1 byte` 。

通过空间预分配策略， Redis 可以减少连续执行字符串增长操作所需的内存重分配次数。

举个例子， 对于下图所示的 SDS 值 `s` 来说， 如果我们执行：

```
sdscat(s, " Cluster");
```

![](/assets/images/posts/redis-object/redis-object-4.png)

那么 `sdscat` 将执行一次内存重分配操作， 将 SDS 的长度修改为 `13` 字节， 并将 SDS 的未使用空间同样修改为 `13` 字节

![](/assets/images/posts/redis-object/redis-object-5.png)

如果这时， 我们再次对 `s` 执行：

```
sdscat(s, " Tutorial");
```

那么这次 sdscat 将不需要执行内存重分配： 因为未使用空间里面的 13 字节足以保存 9 字节的 " Tutorial" ， 执行 sdscat 之后的 SDS

![](/assets/images/posts/redis-object/redis-object-33.png)

在扩展 SDS 空间之前， SDS API 会先检查未使用空间是否足够， 如果足够的话， API 就会直接使用未使用空间， 而无须执行内存重分配。

**通过这种预分配策略， SDS 将连续增长 `N` 次字符串所需的内存重分配次数从必定 `N` 次降低为最多 `N` 次。**

**惰性空间释放**

 我们在观察SDS 的结构的时候可以看到里面的free 属性，是用于记录空余空间的。我们除了在拓展字符串的时候会使用到free 来进行记录空余空间以外，在对字符串进行收缩的时候，我们也可以使用free 属性来进行记录剩余空间，这样做的好处就是避免下次对字符串进行再次修改的时候，需要对字符串的空间进行拓展。

然而，我们并不是说不能释放SDS 中空余的空间，SDS 提供了相应的API，让我们可以在有需要的时候，自行释放SDS 的空余空间。通过惰性空间释放，SDS 避免了缩短字符串时所需的内存重分配操作，并未将来可能有的增长操作提供了优化　　

举个例子， `sdstrim` 函数接受一个 SDS 和一个 C 字符串作为参数， 从 SDS 左右两端分别移除所有在 C 字符串中出现过的字符。

比如对于下图所示的 SDS 值 `s` 来说， 执行：

```
sdstrim(s, "XY");   // 移除 SDS 字符串中的所有 'X' 和 'Y'
```
![](/assets/images/posts/redis-object/redis-object-34.png)

会将 SDS 修改成下图所示的样子。

![](/assets/images/posts/redis-object/redis-object-35.png)

注意执行 `sdstrim` 之后的 SDS 并没有释放多出来的 `8` 字节空间， 而是将这 `8` 字节空间作为未使用空间保留在了 SDS 里面， 如果将来要对 SDS 进行增长操作的话， 这些未使用空间就可能会派上用场。

举个例子， 如果现在对 `s` 执行：

```
sdscat(s, " Redis");
```

那么完成这次 `sdscat` 操作将不需要执行内存重分配： 因为 SDS 里面预留的 `8` 字节空间已经足以拼接 `6` 个字节长的 `" Redis"` ， 如下图所示。

![](/assets/images/posts/redis-object/redis-object-36.png)

通过惰性空间释放策略， SDS 避免了缩短字符串时所需的内存重分配操作， 并为将来可能有的增长操作提供了优化。

与此同时， SDS 也提供了相应的 API ， 让我们可以在有需要时， 真正地释放 SDS 里面的未使用空间， 所以不用担心惰性空间释放策略会造成内存浪费。

## 2.4. **二进制安全**

C 字符串中的字符必须符合某种编码（比如 ASCII）， 并且除了字符串的末尾之外， 字符串里面不能包含空字符， 否则最先被程序读入的空字符将被误认为是字符串结尾 —— 这些限制使得 C 字符串只能保存文本数据， 而不能保存像图片、音频、视频、压缩文件这样的二进制数据。

举个例子， 如果有一种使用空字符来分割多个单词的特殊数据格式， 如下图 所示， 那么这种格式就不能使用 C 字符串来保存， 因为 C 字符串所用的函数只会识别出其中的 `"Redis"` ， 而忽略之后的 `"Cluster"` 。

![](/assets/images/posts/redis-object/redis-object-37.png)

虽然数据库一般用于保存文本数据， 但使用数据库来保存二进制数据的场景也不少见， 因此， 为了确保 Redis 可以适用于各种不同的使用场景， SDS 的 API 都是二进制安全的（binary-safe）： 所有 SDS API 都会以处理二进制的方式来处理 SDS 存放在 `buf` 数组里的数据， 程序不会对其中的数据做任何限制、过滤、或者假设 —— 数据在写入时是什么样的， 它被读取时就是什么样。

SDS使用`len`属性的值来判断字符串是否结束，而不是空字符。

![](/assets/images/posts/redis-object/redis-object-38.png)

通过使用二进制安全的 SDS ， 而不是 C 字符串， 使得 Redis 不仅可以保存文本数据， 还可以保存任意格式的二进制数据。

## 2.5.  小结

| C字符串                                | SDS                                          |
| -------------------------------------- | -------------------------------------------- |
| 获取字符串长度复杂度为O(N)             | 获取字符串长度复杂度为O(1)                   |
| API是不安全的，可能会造成缓冲区溢出    | API是安全的，不会造成缓冲区溢出              |
| 修改字符串长度必然会需要执行内存重分配 | 修改字符串长度N次最多会需要执行N次内存重分配 |
| 只能保存文本数据                       | 可以保存文本或二进制数据                     |

# 3. 参考资料

http://redisbook.com/preview/sds/different_between_sds_and_c_string.html
