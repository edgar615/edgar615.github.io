---
layout: post
title: redis bitmap
date: 2019-06-12
categories:
    - redis
comments: true
permalink: redis-bitmap.html
---

redis的BitMap本身不是一种数据结构，实际上它就是一个字符串，但是它可以对字符串的位进行操作，可以吧bitmap想象成一个以位为单位的数组，数组的每个单元只能存储0和1，数组的下标在bitmap中叫做偏移量。因为8个bit可以组成一个Byte，所以bitmap本身会极大的节省储存空间

#设置值
setbit key offset value

示例1:假设有20个用户，在今天0,5,11,16,19这5个用户访问了系统，那么初始化结果为：

```
127.0.0.1:6379> setbit user:2017-11-08 0 1
(integer) 0
127.0.0.1:6379> setbit user:2017-11-08 5 1
(integer) 0
127.0.0.1:6379> setbit user:2017-11-08 11 1
(integer) 0
127.0.0.1:6379> setbit user:2017-11-08 16 1
(integer) 0
127.0.0.1:6379> setbit user:2017-11-08 19 1
(integer) 0
```
此时id=50的用户访问了系统

```
127.0.0.1:6379> setbit user:2017-11-08 50 1
(integer) 0
```
注意第一次初始化bitmap的时候，如果偏移量非常大，会造成阻塞

# 获取值

`getbit key offset`

```
127.0.0.1:6379> getbit user:2017-11-08 8
(integer) 0
127.0.0.1:6379> getbit user:2017-11-08 100000
(integer) 0
127.0.0.1:6379> getbit user:2017-11-08 50
(integer) 1
```

# 获取指定范围内值为1的个数
`bitcount key [start] [end]` start和end分别是起始和结束**字节数**

```
127.0.0.1:6379> bitcount user:2017-11-08
(integer) 6
127.0.0.1:6379> bitcount user:2017-11-08 1 3
(integer) 3 //第一个字节到第三个字节，偏移量8到24
```

# bitmap间运算

`bitop op destkey key [key ....]`

bitop是一个复合操作，它可以做多个bitmap的and（交集），or（并集），not（非）, xor(异或)操作，并将结果保存在destkey中

# 计算bitmap中第一个值为targetBit的偏移量

`bitpos key targetBit [start] [end]`

示例：访问网站的最小用户

```
	bitpos user:2017-11-07 1
```

