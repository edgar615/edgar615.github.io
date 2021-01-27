---
layout: post
title: redis数据结构（2）- 链表
date: 2020-04-02
categories:
    - redis
comments: true
permalink: redis-object-linklist.html
---

>  Redis有六种基础数据结构：动态字符串，链表，字典，跳跃表，整数集合和压缩列表。

# 1. 链表

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

多个 `listNode` 可以通过 `prev` 和 `next` 指针组成双端链表

![](/assets/images/posts/redis-object/redis-object-39.png)

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

Redis 的链表实现的特性可以总结如下：

- 双端： 链表节点带有 `prev` 和 `next` 指针， 获取某个节点的前置节点和后置节点的复杂度都是 O(1) 。
- 无环： 表头节点的 `prev` 指针和表尾节点的 `next` 指针都指向 `NULL` ， 对链表的访问以 `NULL` 为终点。
- 带表头指针和表尾指针： 通过 `list` 结构的 `head` 指针和 `tail` 指针， 程序获取链表的表头节点和表尾节点的复杂度为 O(1) 。
- 带链表长度计数器： 程序使用 `list` 结构的 `len` 属性来对 `list` 持有的链表节点进行计数， 程序获取链表中节点数量的复杂度为 O(1) 。
- 多态： 链表节点使用 `void*` 指针来保存节点值， 并且可以通过 `list` 结构的 `dup` 、 `free` 、 `match` 三个属性为节点值设置类型特定函数， 所以链表可以用于保存各种不同类型的值。

# 2. 参考资料

http://redisbook.com/preview/adlist/implementation.html