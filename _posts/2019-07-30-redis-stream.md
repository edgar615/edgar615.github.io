---
layout: post
title: redis stream消息队列
date: 2019-07-30
categories:
    - redis
comments: true
permalink: redis-stream.html
---

一直在使用kafka做MQ，但对于有些小型项目来说太重，所以考虑找一个轻量一点的MQ。因为现在项目基本都会重度依赖redis，所以就选用了redis的stream。

>在redis推出stream之前，可以通过PUB/SUB或者是BLPOP/BRPOP来实现简易MQ，但是它们在消费了消息之后就在内存里丢失了，而Stream会记住所有历史

![](/assets/images/posts/redis-stream/redis-stream-1.png)

Redis Stream的结构如上图所示，它有一个消息链表，将所有加入的消息都串起来，每个消息都有一个唯一的ID和对应的内容。消息是持久化的，Redis重启后，内容还在。

每个Stream都有唯一的名称，它就是Redis的key。

每个Stream都可以挂多个消费组，每个消费组会有个游标last_delivered_id在Stream数组之上往前移动，表示当前消费组已经消费到哪条消息了。每个消费组都有一个Stream内唯一的名称，消费组不会自动创建，它需要单独的指令xgroup create进行创建，需要指定从Stream的某个消息ID开始消费，这个ID用来初始化last_delivered_id变量。

每个消费组(Consumer Group)的状态都是独立的，相互不受影响。也就是说同一份Stream内部的消息会被每个消费组都消费到。

同一个消费组(Consumer Group)可以挂接多个消费者(Consumer)，这些消费者之间是竞争关系，任意一个消费者读取了消息都会使游标last_delivered_id往前移动。每个消费者者有一个组内唯一名称。

消费者(Consumer)内部会有个状态变量pending_ids，它记录了当前已经被客户端读取的消息，但是还没有ack。如果客户端没有ack，这个变量里面的消息ID会越来越多，一旦某个消息被ack，它就开始减少。这个pending_ids变量在Redis官方被称之为PEL，也就是Pending Entries List，这是一个很核心的数据结构，它用来确保客户端至少消费了消息一次，而不会在网络传输的中途丢失了没处理。

> 和kafka很像，不过Stream不支持分区。如果需要分区，需要提供不同的stream名称，然后客户端对消息进行hash取模将消息发送到不同的stream

# 队列的增删改查

- `XADD` 追加消息，如果stream不存在，先创建一个Stream
- `XDEL` 删除消息
- `XRANGE` 获取消息列表，会自动过滤已经删除的消息
- `XREVRANGE` 倒序查找
- `XLEN` 消息长度
- `DEL` 删除Stream

```
XADD key ID field string [field string ...]
```

- 第一个参数是stream的名字
- 第二个参数是消息的ID，可以自己指定，也可以使用`*`让redis自己分配一个ID
- field string对应了队列中消息的一个键值对，它们是成对出现的，并且可以接多对

```
127.0.0.1:6379> XADD first-stream * name edgar age 32
"1564470138630-0"
127.0.0.1:6379> XADD first-stream * name leo age 30
"1564470177334-0
```
可以看到返回值（消息ID）由两部分组成`timestampInMillis-sequence`，第一部分是一个精确到毫秒的时间戳， 第二部分是自增的整数。时间戳精确到时间范围是毫秒，因此就需要一个自增ID来确保同一毫秒内发送的消息的唯一性。例如`1564470138630-0`，它表示当前的消息在毫米时间戳1564470138630时产生，并且是该毫秒内产生的第0条消息。消息ID可以由服务器自动生成，也可以由客户端自己指定，但是形式必须是`无符号64位整数-无符号64位整数`，而且必须是后面加入的消息的ID要大于前面的消息ID。

```
127.0.0.1:6379> XLEN first-stream
(integer) 2
127.0.0.1:6379> XRANGE first-stream - +  # -表示最小值, +表示最大值
1) 1) "1564470138630-0"
   2) 1) "name"
      2) "edgar"
      3) "age"
      4) "32"
2) 1) "1564470177334-0"
   2) 1) "name"
      2) "leo"
      3) "age"
      4) "30"
127.0.0.1:6379> XRANGE first-stream 1564470138630-1 + # 指定最小值
1) 1) "1564470177334-0"
   2) 1) "name"
      2) "leo"
      3) "age"
      4) "30"
127.0.0.1:6379> XRANGE first-stream - 1564470177333-0 # 指定最大值
1) 1) "1564470138630-0"
   2) 1) "name"
      2) "edgar"
      3) "age"
      4) "32"
127.0.0.1:6379> XRANGE first-stream - + COUNT 1 # 指定返回数量
1) 1) "1564470138630-0"
   2) 1) "name"
      2) "edgar"
      3) "age"
      4) "32"
127.0.0.1:6379> XDEL first-stream 1564470138630-0 # 删除消息
(integer) 1
127.0.0.1:6379> XLEN first-stream
(integer) 1
127.0.0.1:6379> XRANGE first-stream - +
1) 1) "1564470177334-0"
   2) 1) "name"
      2) "leo"
      3) "age"
      4) "30"
127.0.0.1:6379> XREVRANGE first-stream + - # 反向查找
1) 1) "1564470177334-0"
   2) 1) "name"
      2) "leo"
      3) "age"
      4) "30"
127.0.0.1:6379> XREVRANGE first-stream - +
(empty list or set)
```

相同的ID会返回错误
```
127.0.0.1:6379> XADD first-stream 1564470138630-0 name edgar age 32
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item
```

查找可以只使用毫秒查找

```
127.0.0.1:6379> XRANGE first-stream 1564470138630 +
1) 1) "1564470177334-0"
   2) 1) "name"
      2) "leo"
      3) "age"
      4) "30"
```

# 独立消费
通过`xread`我们可以在不定义消费组的情况下进行Stream消息的独立消费，当Stream没有新消息时，甚至可以阻塞等待。使用xread时，我们可以完全忽略消费组(Consumer Group)的存在，就好比Stream就是一个普通的列表(list)。

```
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]
```

```
127.0.0.1:6379> XREAD STREAMS first-stream $ 不阻塞
(nil)
127.0.0.1:6379> XREAD BLOCK 1000 STREAMS first-stream $ #阻塞1秒，没有数据返回nil
(nil)
(1.19s)
127.0.0.1:6379> XREAD BLOCK 0 STREAMS first-stream $ #一直阻塞直到有数据
# 在另一个会话添加消息
127.0.0.1:6379> XADD first-stream * name Bob age 23
"1564472130429-0"
# 前面阻塞的会话会返回
127.0.0.1:6379> XREAD BLOCK 0 STREAMS first-stream $
1) 1) "first-stream"
   2) 1) 1) "1564472130429-0"
         2) 1) "name"
            2) "Bob"
            3) "age"
            4) "23"
(76.27s)

```
客户端如果想要使用`xread`进行顺序消费，一定要记住当前消费到哪里了，也就是返回的消息ID。下次继续调用`xread`时，将上次返回的最后一个消息ID作为参数传递进去，就可以继续消费后续的消息。

> 使用 $ 则是代表 最新的消息的ID

如果多个会话开启`xread`，那么都可以进行读到消息

# 消费者组

![](/assets/images/posts/redis-stream/redis-stream-1.png)

> 消费者组的概念和kafka很类似


- `XGROUP` 是用来创建删除和管理消费组的
- `XREADGROUP` 是用来通过消费组从Stream里读消息的
- `XACK` 是用来确认消息已经消费的

```
XGROUP [CREATE key groupname id-or-$] [SETID key groupname id-or-$] [DESTROY key groupname] [DELCONSUMER key groupname consumername] 

XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID [ID ...] 

XACK key group ID [ID ...] 
```

## 创建消费组
`XGROUP CREATE`用来创建消费组，需要传递起始消息ID参数用来初始化last_delivered_id变量。

```
# 从头部开始消费
127.0.0.1:6379> XGROUP CREATE first-stream first-group 0-0 
OK
# 从尾部开始消费
127.0.0.1:6379> XGROUP CREATE first-stream second-group $
OK
```

创建组之前要先创建Stream

通过`XINFO`可以查看消费组的消费情况

```
127.0.0.1:6379> XINFO GROUPS first-stream
1) 1) "name"
   2) "first-group"
   3) "consumers"
   4) (integer) 0
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "0-0"
2) 1) "name"
   2) "second-group"
   3) "consumers"
   4) (integer) 0
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "1564472532326-0"
```

## 消费

```
127.0.0.1:6379> XREADGROUP GROUP first-group fg1 STREAMS first-stream >
1) 1) "first-stream"
   2) 1) 1) "1564470138630-0"
         2) 1) "name"
            2) "edgar"
            3) "age"
            4) "32"
      2) 1) "1564470177334-0"
         2) 1) "name"
            2) "leo"
            3) "age"
            4) "30"
      3) 1) "1564472130429-0"
         2) 1) "name"
            2) "Bob"
            3) "age"
            4) "23"
      4) 1) "1564472235221-0"
         2) 1) "name"
            2) "Jeniffer"
            3) "age"
            4) "29"
      5) 1) "1564472532326-0"
         2) 1) "name"
            2) "John"
            3) "age"
            4) "18"
127.0.0.1:6379> XREADGROUP GROUP first-group fg1 STREAMS first-stream >
(nil)
```

> XREADGROUP 中，最后一个参数 > 代表消费者希望Redis只给自己没有发布过的消息。如果使用具体的ID，例如0，则是从那个ID之后的 消息。

在另一个会话中执行下面的命令，发现没有消息返回

```
127.0.0.1:6379> XREADGROUP GROUP first-group fg1 STREAMS first-stream >
(nil)
127.0.0.1:6379> XREADGROUP GROUP first-group fg2 STREAMS first-stream >
(nil)
```

再次查看`XINFO`

```
127.0.0.1:6379> XINFO GROUPS first-stream
1) 1) "name"
   2) "first-group"
   3) "consumers"
   4) (integer) 1
   5) "pending"
   6) (integer) 5
   7) "last-delivered-id"
   8) "1564472532326-0"
2) 1) "name"
   2) "second-group"
   3) "consumers"
   4) (integer) 0
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "1564472532326-0"
```

可以看到`first-group`有一个消费者，有5条消息正在处理但没有ACK

如果我们在两个会话中使用下面的命令，可以看到每个消费者都可以读到消息
```
XREADGROUP GROUP first-group fg1 BLOCK 0 STREAMS first-stream >
XREADGROUP GROUP first-group fg2 BLOCK 0 STREAMS first-stream >
```

## ACK
```
127.0.0.1:6379> XACK first-stream first-group 1564470138630-0
(integer) 1
127.0.0.1:6379> XINFO GROUPS first-stream
1) 1) "name"
   2) "first-group"
   3) "consumers"
   4) (integer) 1
   5) "pending"
   6) (integer) 4
   7) "last-delivered-id"
   8) "1564472532326-0"
2) 1) "name"
   2) "second-group"
   3) "consumers"
   4) (integer) 0
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "1564472532326-0"
```
在ack后，我们可以看到`first-group`的`pending`变成了4

# 定长Stream

要是消息积累太多，Stream的链表会变得很长，可能会导致内存溢出。
`XADD`的指令提供一个定长长度maxlen，就可以将老的消息干掉，确保最多不超过指定长度。
```
127.0.0.1:6379> XADD first-stream MAXLEN 3 * name edgar age 32
"1564474430917-0"
127.0.0.1:6379> XLEN first-stream
(integer) 3
```
有可以通过`XTRIM`现在stream的长度
```
XTRIM key MAXLEN [~] count 
```

> ~表示允许stream的长度比指定的长度长一点点

# Java实现
`lettuce`实现了stream，简单使用了下
## 添加消息

```
    RedisClient redisClient = RedisClient.create("redis://tabao@192.168.1.204:6379/0");
    StatefulRedisConnection<String, String> connection = redisClient.connect();
    RedisStreamCommands<String, String> streamCommands = connection.sync();

    Map<String, String> body =  new HashMap<>();
    body.put("name", "shelly");
    body.put("age", "26");
    String messageId = streamCommands.xadd("first-stream", body);
    System.out.println(messageId);

    connection.close();
    redisClient.shutdown();
```

## 创建消费组

```
streamCommands.xgroupCreate(StreamOffset.latest("first-stream"), "lettuce-group");
```

## 消费

```
    List<StreamMessage<String, String>> messages = streamCommands.xreadgroup(
        Consumer.from("lettuce-group", "lg1"),
        XReadArgs.Builder.block(10000),
        StreamOffset.lastConsumed("first-stream"));

    if(messages.size() == 1) { // a message was read
      System.out.println(messages.get(0).getId());
      System.out.println(messages.get(0).getBody());
    } else { // no message was read

    }
```

# 参考资料

https://mp.weixin.qq.com/s/UUhP_I2wCqUeZV2SaUJm5A