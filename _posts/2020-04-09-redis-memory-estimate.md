---
layout: post
title: redis内存估算
date: 2020-04-09
categories:
    - redis
comments: true
permalink: redis-memory-estimate.html
---

要估算Redis中的数据占据的内存大小，需要对Redis的内存模型有比较全面的了解，包括前面介绍的hashtable、SDS、RedisObject、各种对象类型的编码方式等。

> 参考[这篇文章](https://edgar615.github.io/redis-object.html)

# 1. debug命令
通过redis的debug命令，可以查看某个key序列化后的长度。

```
debug object myhash
Value at:0x7f005c6920a0 refcount:1 encoding:ziplist serializedlength:36 lru:3341677 lru_seconds_idle:2
```

输出项

- Value at：key的内存地址
- refcount：引用次数
- encoding：编码类型
- serializedlength：序列化长度
- lru_seconds_idle：空闲时间

serializedlength是key序列化后的长度(redis在将key保存为rdb文件时使用了该算法)，并不是key在内存中的真正长度。这就像一个数组在json_encode后的长度与其在内存中的真正长度并不相同。不过，它侧面反应了一个key的长度，可以用于比较两个key的大小。

serializedlength会对字串做一些可能的压缩。如果有些字串的压缩比特别高，那么在比较时会出现问题。

> 更多内容参考 https://www.jianshu.com/p/c885af575f97

> 时间关系，这篇文章的内容没有验证，后续验证

假设有90000个键值对，每个key的长度是7个字节，每个value的长度也是7个字节（且key和value都不是整数）。

下面来估算这90000个键值对所占用的空间。

在估算占据空间之前，首先可以判定字符串类型使用的编码方式：embstr。90000个键值对占据的内存空间主要可以分为两部分：一部分是90000个dictEntry占据的空间；一部分是键值对所需要的bucket空间（KEY是SDS，VALUE是RedisObject）。

每个dictEntry占据的空间包括：

- 一个dictEntry，24字节，jemalloc会分配32字节的内存块。
- 一个key，7字节，所以SDS(key)需要7+9=16个字节，jemalloc会分配16字节的内存块。
- 一个RedisObject，16字节，jemalloc会分配16字节的内存块。
- 一个value，7字节，所以SDS(value)需要7+9=16个字节，jemalloc会分配16字节的内存块。
- 综上，一个dictEntry需要32+16+16+16=80个字节。

bucket空间：bucket数组的大小为大于90000的最小的2^n，是131072，每个bucket元素为8字节（因为64位系统中指针大小为8字节）。

因此，可以估算出这90000个键值对占据的内存大小为：90000*80 + 131072*8 = 8248576。

下面写个程序在Redis中验证一下：

```
public class RedisTest {


　　public static Jedis jedis = new Jedis("localhost", 6379);


　　public static void main(String[] args) throws Exception{

　　　　Long m1 = Long.valueOf(getMemory());

　　　　insertData();

　　　　Long m2 = Long.valueOf(getMemory());

　　　　System.out.println(m2 - m1);

　　}


　　public static void insertData(){

　　　　for(int i = 10000; i < 100000; i++){

　　　　　　jedis.set("aa" + i, "aa" + i); //key和value长度都是7字节，且不是整数

　　　　}

　　}


　　public static String getMemory(){

　　　　String memoryAllLine = jedis.info("memory");

　　　　String usedMemoryLine = memoryAllLine.split("\r\n")[1];

　　　　String memory = usedMemoryLine.substring(usedMemoryLine.indexOf(':') + 1);

　　　　return memory;

　　}

}
```

运行结果：8247552

理论值与结果值误差在万分之1.2，对于计算需要多少内存来说，这个精度已经足够了。之所以会存在误差，是因为在我们插入90000条数据之前Redis已分配了一定的bucket空间，而这些bucket空间尚未使用。

作为对比将key和value的长度由7字节增加到8字节，则对应的SDS变为17个字节，jemalloc会分配32个字节，因此每个dictEntry占用的字节数也由80字节变为112字节。此时估算这90000个键值对占据内存大小为：90000*112  + 131072*8 = 11128576。

在Redis中验证代码如下（只修改插入数据的代码）：

```
public static void insertData(){

　　for(int i = 10000; i < 100000; i++){

　　　　jedis.set("aaa" + i, "aaa" + i); //key和value长度都是8字节，且不是整数

　　}

}
```

运行结果：11128576；估算准确。

对于字符串类型之外的其它类型，对内存占用的估算方法是类似的，需要结合具体类型的编码方式来确定。

# 参考资料

https://mp.weixin.qq.com/s/4wpsg8BDwGVADWb3WpSzpA
