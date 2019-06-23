---
layout: post
title: BloomFilter
date: 2019-06-22
categories:
    - 算法
comments: true
permalink: bloom-filter.html
---

假设要你写一个网络爬虫程序（web crawler）。由于网络间的链接错综复杂，爬虫在网络间爬行很可能会形成“环”。为了避免形成“环”，就需要知道爬虫程序已经访问过那些URL。给一个URL，怎样知道爬虫程序是否已经访问过呢？稍微想想，就会有如下几种方案：

1. 将访问过的URL保存到数据库
2. 用HashSet将访问过的URL保存起来。那只需接近O(1)的代价就可以查到一个URL是否被访问过了
3. URL经过MD5或SHA-1等单向哈希后再保存到HashSet或数据库
4. Bit-Map方法：建立一个BitSet，将每个URL经过一个哈希函数映射到某一位。

其中方法1~3都是将访问过的URL完整保存，方法4则只标记URL的一个映射位。以上方法在数据量较小的情况下都能完美解决问题，但是当数据量变得非常庞大时问题就来了。

1. 方法1：数据量变得非常庞大后关系型数据库查询的效率会变得很低
2. 方法2：太消耗内存。随着URL的增多，占用的内存会越来越多。就算只有1亿个URL，每个URL只算50个字符，就需要5GB内存。
3. 方法3：由于字符串经过MD5处理后的信息摘要长度只有128Bit，SHA-1处理后也只有160Bit，因此方法3比方法2节省了好几倍的内存
4. 方法4：消耗内存是相对较少的，但缺点是单一哈希函数发生冲突的概率太高。若要降低冲突发生的概率到1%，就要将BitSet的长度设置为URL个数的100倍。

# Bloom Filter
> 布隆过滤器（Bloom Filter）是1970年由布隆提出的。它实际上是一个很长的二进制向量和一系列随机映射函数。布隆过滤器可以用于检索一个元素是否在一个集合中。它的优点是空间效率和查询时间都远远超过一般的算法，缺点是有一定的误识别率和删除困难

**原理**
当一个元素被加入集合时，通过K个散列函数将这个元素映射成一个位数组中的K个点，把它们置为1。检索时，我们只要看看这些点是不是都是1就（大约）知道集合中有没有它了：如果这些点有任何一个0，则被检元素一定不在；如果都是1，则被检元素很可能在。这就是布隆过滤器的基本思想

BloomFilter的整体算法流程可总结为如下步骤：

1. BloomFilter初始化为m位长度的位向量，每一位均初始化为0
```
00000000000000000000000000000000000000000000000000
```
2. 使用k个相互独立的Hash函数，每个Hash函数将元素映射到{1..m}的范围内
```
hash1(X)=8
hash2(x)=1
hash2(x)=14
```
3. 将第二部hash映射对应的位置为1
```
10000001000001000000000000000000000000000000000000
```
4. 若检查一个元素y是否存在，首先使用k个Hash函数将元素y映射到k位。分别检测每一位是否为0。**若某一位为0，则元素y一定不存在，若全部为1，则有可能存在**

**空间复杂度**

BloomFilter 使用位向量来表示元素，而不存储本身，这样极大压缩了元素的存储空间。其空间复杂度为O(m)，m是位向量的长度。

**时间复杂度**

时间复杂度方面 BloomFilter的时间复杂度仅与Hash函数的个数k有关，即O(k)

**删除元素**

BloomFilter 由于并不存储元素，而是用位的01来表示元素是否存在，并且很有可能一个位时被多个元素同时使用。所以无法通过将某元素对应的位置为0来删除元素。

> Counting BloomFilter通过存储位元素每一位的置为1的数量，使得布隆过滤器可以支持删除操作。但是这样会数倍地增加布隆过滤器的存储空间。

# Guava实现
guava提供了一个`BloomFilter`，我们可以直接使用

```
//参数1：一个 Funnel对象，用于将数据发送给一个接收器（Sink）。
//参数2：一个代表预期插入数量整数。
BloomFilter<String> bloomFilter = BloomFilter.create(Funnels.stringFunnel(Charset.defaultCharset()), 1000);

bloomFilter.put("Hello BloomFilter");
bloomFilter.put(UUID.randomUUID().toString());
boolean mayBeContained = bloomFilter.mightContain("Hello BloomFilter");
System.out.println(mayBeContained);//true
mayBeContained = bloomFilter.mightContain(UUID.randomUUID().toString());
System.out.println(mayBeContained);//false
```

对于一个对象，我们可以通过实现`Funnel`接口来实现`BloomFilter`

```
public class User {

  private final String username;

  public User(String username) {
    this.username = username;
  }

  public String getUsername() {
    return username;
  }

}
```

```
public class UserFunnel implements Funnel<User> {

  @Override
  public void funnel(User user, PrimitiveSink primitiveSink) {
    primitiveSink.putString(user.getUsername(), Charset.defaultCharset());
  }
}
```

测试

```
BloomFilter<User> bloomFilter = BloomFilter.create(new UserFunnel(), 1000);

bloomFilter.put(new User("edgar"));
boolean mayBeContained = bloomFilter.mightContain(new User("edgar"));
System.out.println(mayBeContained);//true
mayBeContained = bloomFilter.mightContain(new User("leo"));
System.out.println(mayBeContained);//false
```

# Redis实现
redis也提供了BloomFilter的实现，需要额外安装
```
 $ git clone git://github.com/RedisLabsModules/rebloom
 $ cd rebloom
 $ make
```

将rebloom加入Redis
```
loadmodule /path/to/redisbloom.so
# 这里和官方文档不一样,文档上写的rebloom.so
```
也可以通过命令直接启动
```
$ redis-server --loadmodule /path/to/redisbloom.so
```
使用
```
 127.0.0.1:6379> BF.ADD bloom mark
 1) (integer) 1
 127.0.0.1:6379> BF.ADD bloom redis
 1) (integer) 1
 127.0.0.1:6379> BF.EXISTS bloom mark
 (integer) 1
 127.0.0.1:6379> BF.EXISTS bloom redis
 (integer) 1
 127.0.0.1:6379> BF.EXISTS bloom nonexist
 (integer) 0
 127.0.0.1:6379> BF.EXISTS bloom que?
 (integer) 0
 127.0.0.1:6379>
 127.0.0.1:6379> BF.MADD bloom elem1 elem2 elem3
 1) (integer) 1
 2) (integer) 1
 3) (integer) 1
 127.0.0.1:6379> BF.MEXISTS bloom elem1 elem2 elem3
 1) (integer) 1
 2) (integer) 1
 3) (integer) 1
```

我们也可以创建自己的BloomFilter

```
 127.0.0.1:6379> BF.RESERVE largebloom 0.0001 1000000
 OK
 127.0.0.1:6379> BF.ADD largebloom mark
 1) (integer) 1
```

# 参考资料

https://juejin.im/post/5c43f695e51d455221611728

http://cyhone.com/2017/02/07/Introduce-to-BloomFilter/

https://redislabs.com/blog/rebloom-bloom-filter-datatype-redis/
