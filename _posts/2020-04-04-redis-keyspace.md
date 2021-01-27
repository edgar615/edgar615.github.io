---
layout: post
title: redis数据库键空间
date: 2020-04-04
categories:
    - redis
comments: true
permalink: redis-keyspace.html
---

# 1. 键空间

Redis 服务器都有多个 Redis 数据库，每个Redis 数据都有自己独立的键值空间。每个 Redis 数据库使用 dict 保存数据库中所有的键值对。

![](/assets/images/posts/redis-object/redis-object-24.jpeg)

键空间的键也就是数据库的键，每个键都是一个字符串对象，而值对象可能为字符串对象、列表对象、哈希表对象、集合对象和有序集合对象中的一种对象。

除了键空间，Redis 也使用 dict 结构来保存键的过期时间，其键是键空间中的键值，而值是过期时间，如上图所示。

通过过期字典，Redis 可以直接判断一个键是否过期，首先查看该键是否存在于过期字典，如果存在，则比较该键的过期时间和当前服务器时间戳，如果大于，则该键过期，否则未过期。

如果我们在空白的数据库中执行以下命令：

```
redis> SET message "hello world"
OK

redis> RPUSH alphabet "a" "b" "c"
(integer) 3

redis> HSET book name "Redis in Action"
(integer) 1

redis> HSET book author "Josiah L. Carlson"
(integer) 1

redis> HSET book publisher "Manning"
(integer) 1
```

那么在这些命令执行之后， 数据库的键空间将会是下图所展示的样子：

![](/assets/images/posts/redis-object/redis-object-71.png)

- `alphabet` 是一个列表键， 键的名字是一个包含字符串 `"alphabet"` 的字符串对象， 键的值则是一个包含三个元素的列表对象。
- `book` 是一个哈希表键， 键的名字是一个包含字符串 `"book"` 的字符串对象， 键的值则是一个包含三个键值对的哈希表对象。
- `message` 是一个字符串键， 键的名字是一个包含字符串 `"message"` 的字符串对象， 键的值则是一个包含字符串 `"hello world"` 的字符串对象。

因为数据库的键空间是一个字典， 所以所有针对数据库的操作 —— 比如添加一个键值对到数据库， 或者从数据库中删除一个键值对， 又或者在数据库中获取某个键值对， 等等， 实际上都是通过对键空间字典进行操作来实现的，

# 2. 添加新键

添加一个新键值对到数据库， 实际上就是将一个新键值对添加到键空间字典里面， 其中键为字符串对象， 而值则为任意一种类型的 Redis 对象。

```
redis> SET date "2013.12.1"
OK
```

键空间将添加一个新的键值对， 这个新键值对的键是一个包含字符串 `"date"` 的字符串对象， 而键值对的值则是一个包含字符串 `"2013.12.1"` 的字符串对象，如下图所示

![](/assets/images/posts/redis-object/redis-object-72.png)

# 3. 删除键

删除数据库中的一个键， 实际上就是在键空间里面删除键所对应的键值对对象。

```
redis> DEL book
(integer) 1
```

键 `book` 以及它的值将从键空间中被删除，如下图所示

![](/assets/images/posts/redis-object/redis-object-73.png)

# 4. 更新键

对一个数据库键进行更新， 实际上就是对键空间里面键所对应的值对象进行更新， 根据值对象的类型不同， 更新的具体方法也会有所不同。

```
redis> SET message "blah blah"
OK
```

键 `message` 的值对象将从之前包含 `"hello world"` 字符串更新为包含 `"blah blah"` 字符串，如下图所示

![](/assets/images/posts/redis-object/redis-object-74.png)

```
redis> HSET book page 320
(integer) 1
```

那么键空间中 `book` 键的值对象（一个哈希对象）将被更新， 新的键值对 `page` 和 `320` 会被添加到值对象里面，

![](/assets/images/posts/redis-object/redis-object-75.png)

# 5. 对键取值

对一个数据库键进行取值， 实际上就是在键空间中取出键所对应的值对象， 根据值对象的类型不同， 具体的取值方法也会有所不同。

```
redis> GET message
"hello world"
```

GET 命令将首先在键空间中查找键 `message` ， 找到键之后接着取得该键所对应的字符串对象值， 之后再返回值对象所包含的字符串 `"hello world"` ，

![](/assets/images/posts/redis-object/redis-object-76.png)

```
redis> LRANGE alphabet 0 -1
1) "a"
2) "b"
3) "c"
```

LRANGE 命令将首先在键空间中查找键 `alphabet` ， 找到键之后接着取得该键所对应的列表对象值， 之后再返回列表对象中包含的三个字符串对象的值

![](/assets/images/posts/redis-object/redis-object-77.png)

# 6. 参考资料

http://redisbook.com/preview/database/key_space.html