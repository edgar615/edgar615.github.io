---
layout: post
title: 分布式锁实现
date: 2019-11-05
categories:
    - 分布式
comments: true
permalink: distributed-lock.html
---


最近在服务中增加了一个定时任务，定时从数据表中拉取未正常处理的任务处理后标记成处理成功。在单机时问题不大，但是到集群环境下就会出现问题：

实例A随机从数据表中获取一个记录；实例B随机从数据表中获取一个记录。在理想的情况下，实例A从数据表中获取一个记录，处理完成，标记任务；B从剩下的的记录中再挑一个处理。但在真实情况下，如果不做任何处理，可能会出现A和B挑中了同一个记录的情况。

为了解决这个问题，考虑引入分布式锁。

为了确保分布式锁可用，我们至少要确保锁的实现同时满足以下四个条件：

- 互斥性：任意时刻，只能有一个客户端获取锁，不能同时有两个客户端获取到锁。
- 安全性：加锁和解锁必须是同一个客户端，锁只能被持有该锁的客户端删除，不能由其它客户端删除。
- 不会发生死锁死锁：即使获取锁的客户端因为某些原因（如down机等）而未能释放锁，也能保证后续其它客户端可以获取到该锁。
- 容错性：当部分节点（redis节点等）down机时，客户端仍然能够获取锁和释放锁。 

分布式锁一般有三种实现方式：

- 基于数据库的分布式锁
- 基于Redis的分布式锁
- 基于ZooKeeper的分布式锁

# 数据库

## 数据库锁表

我们可以在数据库中创建一张表来控制共享资源。

下面的`distributed_lock`表通过`lock_key`的唯一索引来实现锁，同一个`lock_key`同只能插入一次。于是对锁的竞争就交给了数据库，例如处理同一个订单号的应用把订单号插入表中，数据库保证了只有一个应用能插入成功，其他应用都会插入失败

```
CREATE TABLE `distributed_lock` (
`id` BIGINT ( 20) NOT NULL AUTO_INCREMENT COMMENT '主键',
`lock_key` VARCHAR ( 64 ) NOT NULL COMMENT '锁定的资源',
`lock_value` VARCHAR ( 255 ) NOT NULL COMMENT '锁的客户端标识',
`lockd_at` BIGINT ( 20 ) NOT NULL COMMENT '锁创建时间，单位毫秒',
`expire_at` BIGINT ( 20 ) NOT NULL COMMENT '锁过期时间，单位毫秒',
`update_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '保存数据时间，自动生成',
PRIMARY KEY ( `id` ),
UNIQUE KEY `uidx_lock_key` ( `lock_key` ) USING BTREE,
KEY `idx_expire_at` ( `expire_at` ) USING BTREE 
) ENGINE = INNODB DEFAULT CHARSET = utf8 COMMENT = '分布式锁';
```

获取锁

```
insert into distributed_lock(lock_key, lock_value, locked_at, expire_at) VALUES (?, ?, ?, ?);
```

释放锁

```
delete from distributed_lock where lock_key = ? AND lock_value = ?;
```

删除过期未释放的锁

```
delete from distributed_lock where expire_at < ?;
```

上述的锁实现并没有实现锁的重入，要实现锁重入，我们需要在表中增加一个重入次数`lock_num`

在获取锁失败后检查一下锁是不是自己的，然后增加一下锁的次数

```
insert into distributed_lock(lock_key, lock_value, locked_at, expire_at, lock_num) VALUES (?, ?, ?, ?, ?);

-- 获取失败后
select id from distributed_lock where lock_key='lock_key' and lock_value = 'lock_value'
-- 如果能够获取到记录，说明可以重入，更新锁的次数
update distributed_lock set lock_num = lock_num + 1 where id = id;
```

在释放锁的时候需要检查一下是不是自己的，然后减少一次锁的次数，如果锁的次数=1，直接删除

```
select id, lock_num from distributed_lock where lock_key='lock_key' and lock_value = 'lock_value'
-- 如果获取到数据，说明锁是自己的
if lock_num == 1
delete from distributed_lock where id = id;
else
update distributed_lock set lock_num = lock_num - 1 where id = id;
```

数据库锁能实现一个简单的避免共享资源被多个系统操作的情况。在并发量不是那么恐怖的情况下，数据库锁的性能也不容易出问题，而且由于数据库的数据具有持久化的特性，一般的应用也足够应付。

## 数据库排他锁

```
begin;
-- 获得锁
select * from distributed_lock where lock_key=xxx for update;
-- 处理业务逻辑

-- 释放锁
commit;
```

InnoDB 引擎在加锁的时候，只有通过索引进行检索的时候才会使用行级锁，否则会使用表级锁。
所以如果我们希望使用行级锁，就要给 order_id 添加索引，而且这个索引一定要创建成唯一索引。

排他锁实现分布式锁无法解决重入和超时阻塞的问题，而且还会存在更严重的问题：MySql 会对查询进行优化，即便在条件中使用了索引字段，但是否使用索引来检索数据是由 MySQL 通过判断不同执行计划的代价来决定的，如果 MySQL 认为全表扫效率更高，比如对一些很小的表，它就不会使用索引，这种情况下 InnoDB 将使用表锁，而不是行锁。如果发生这种情况就悲剧了。

## 乐观锁

数据库乐观锁大多数是基于数据版本(version)的记录机制实现的。即为数据增加一个版本标识version，读取出数据时，将此版本号一同读出，之后更新时，对此版本号加1。在更新过程中，会对版本号进行比较，如果是一致的，没有发生改变，则会成功执行本次操作；如果版本号不一致，则会更新失败。

取出记录
```
select id, resource, state,version from t_resource  where state=1 and id=5780;
```

标记

```
update t_resoure set state=2, version=version + 1 where id=5780 and state=1 and version=26
```

如果上述update语句真正更新影响到了一行数据，那就说明占位成功。如果没有更新影响到一行数据，则说明这个资源已经被别人占位了。

优点：乐观锁的优点比较明显，由于在检测数据冲突时并不依赖数据库本身的锁机制，不会影响请求的性能，当产生并发且并发量较小的时候只有少部分请求会失败。

缺点：

-  如果业务场景中的一次业务流程中，多个资源都需要用保证数据一致性，那么如果全部使用基于数据库资源表的乐观锁，就要让每个资源都有一张资源表，这个在实际使用场景中肯定是无法满足的。
-  当应用并发量高的时候，version值在频繁变化，则会导致大量请求失败，影响系统的可用性
-  在高并发如大促、秒杀等活动开展的时候，大量的请求同时请求同一条记录的行锁，会对数据库产生很大的写压力

# redis

在redis中设置一个值表示加了锁，然后释放锁的时候就把这个key删除。

加锁

```
SET resource_name my_random_value NX PX 30000
```

- my_random_value 在所有的客户端和请求锁的请求中必须唯一，因为加锁和解锁必须是同一个客户端
- NX 在指定的 key 不存在时，为 key 设置指定的值。保证了只有第一个请求的客户端能持有锁，而其它客户端在锁被释放之前都无法获得锁，满足互斥性。
- PX 指定key的过期时间，单位毫秒。即使锁的持有者后续发生崩溃而没有解锁，锁也会因为到了过期时间而自动解锁（即key被删除），不会发生死锁。

释放锁

```
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

因为释放锁涉及到两个命令，不是原子性，所以需要使用lua来保证原子性

示例代码：

```
public class RedisLock implements Closeable {

  private static final Logger LOGGER = LoggerFactory.getLogger(RedisLock.class);

  private RedisTemplate<String, String> redisTemplate;

  /**
   * redis key
   */
  private String lockKey;

  /**
   * redis value
   */
  private String lockValue;

  /**
   * 过期时间，单位毫秒
   */
  private int expireMills;

  private static final String RELEASE_LUA_SCRIPT = "if redis.call('get',KEYS[1]) == ARGV[1] then\n"
      + "    return redis.call('del',KEYS[1])\n"
      + "else\n"
      + "    return 0\n"
      + "end";

  public RedisLock(RedisTemplate<String, String> redisTemplate, String lockKey, String lockValue, int expireMills) {
    this.redisTemplate = redisTemplate;
    this.lockKey = lockKey;
    this.lockValue = lockValue;
    this.expireMills = expireMills;
  }

  public boolean acquire() {
    return redisTemplate.execute((RedisCallback<Boolean>) connection -> {
      //序列化key
      byte[] serializeKey = LettuceConverters.toBytes(lockKey);
      //序列化value
      byte[] serializeVal = LettuceConverters.toBytes(lockValue);
      boolean result = connection.stringCommands()
          .set(serializeKey, serializeVal, Expiration.milliseconds(expireMills),
              SetOption.SET_IF_ABSENT);
      LOGGER.info("获取redis锁：" + result);
      return result;
    });
  }

  public boolean release() {
    Boolean result = redisTemplate.execute((RedisCallback<Boolean>) connection -> connection.scriptingCommands()
        .eval(
            RELEASE_LUA_SCRIPT.getBytes(),
            ReturnType.BOOLEAN,
            1,
            LettuceConverters.toBytes(lockKey),
            LettuceConverters.toBytes(lockValue)
        ));
    LOGGER.info("释放redis锁：" + result);
    return result;
  }

  /**
   * 自动释放锁
   */
  @Override
  public void close() throws IOException {
    release();
  }
}
```

测试一下

```
@SpringBootApplication
public class Application implements CommandLineRunner {

  private static final Logger LOGGER = LoggerFactory.getLogger(Application.class);

  @Autowired
  private RedisTemplate<String, String> redisTemplate;

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }

  @Override
  public void run(String... args) throws Exception {
    try (RedisLock lock = new RedisLock(redisTemplate, "test_key", "test_val", 3000)) {
      //获取锁
      if (lock.acquire()) {
        //模拟执行业务
        Thread.sleep(1000);
        LOGGER.info("获取到锁，执行业务操作耗时1s");
      }
    } catch (Exception e) {
      LOGGER.error(e.getMessage(), e);
    }
  }
}
```



上面代码展示的RedisLock还可以增加重试次数、重试间隔来多次尝试获取锁

缺陷：

- 如果客户端长期阻塞导致锁过期，那么它接下来访问共享资源就不安全了
- 如果采用单机部署模式，会存在单点问题，假如Redis节点宕机了，那么所有客户端就都无法获得锁了，服务变得不可用。

**超时：**如果客户端拿到锁之后设置了超时时长，但是业务执行的时长超过了超时时长，导致客户端还在执行业务但是锁已经被释放，此时其他进程就会拿到锁从而执行相同的业务，此时因为并发导致分布式锁失去了意义。我们可以在获取锁之后启动一个定时任务在key快要过期的时候判断下任务有没有执行完毕，如果还没有那就自动延长过期时间，这样做可以解决并发的问题，但是锁的超时时长也就失去了意义，容易导致其他客户端长时间等待，而且如果发生GC卡顿，也不能完全解决超时的问题。



为了提高可用性，我们可以给这个Redis节点挂一个Slave，当Master节点不可用的时候，系统自动切到Slave上（failover）。但由于Redis的主从复制（replication）是异步的，这可能导致在failover过程中丧失锁的安全性。

1. 客户端1从Master获取了锁
2. Master宕机了，存储锁的key还没有来得及同步到Slave上
3. Slave升级为Master
4. 客户端2从新的Master获取到了对应同一个资源的锁

于是，客户端1和客户端2同时持有了同一个资源的锁。锁的安全性被打破。

基于以上的考虑，redis的作者提出了RedLock的算法：
## Redlock算法
RedLock算法基于N个完全独立的Redis节点（通常情况下N可以设置成5），运行Redlock算法的客户端依次执行下面各个步骤，来完成获取锁的操作：

1. 获取当前时间（毫秒数）
2. 按顺序依次向N个Redis节点执行获取锁的操作。这个获取操作跟前面基于单Redis节点的获取锁的过程相同，包含随机字符串my_random_value，也包含过期时间(比如PX 30000，即锁的有效时间)。为了保证在某个Redis节点不可用的时候算法能够继续运行，这个获取锁的操作还有一个超时时间(time out)，它要远小于锁的有效时间（几十毫秒量级）。客户端在向某个Redis节点获取锁失败以后，应该立即尝试下一个Redis节点。这里的失败，应该包含任何类型的失败，比如该Redis节点不可用，或者该Redis节点上的锁已经被其它客户端持有（注：Redlock原文中这里只提到了Redis节点不可用的情况，但也应该包含其它的失败情况）
3. 计算整个获取锁的过程总共消耗了多长时间，计算方法是用当前时间减去第1步记录的时间。如果客户端从大多数Redis节点（>= N/2+1）成功获取到了锁，并且获取锁总共消耗的时间没有超过锁的有效时间(lock validity time)，那么这时客户端才认为最终获取锁成功；否则，认为最终获取锁失败
4. 如果最终获取锁成功了，那么这个锁的有效时间应该重新计算，它等于最初的锁的有效时间减去第3步计算出来的获取锁消耗的时间
5. 如果最终获取锁失败了（可能由于获取到锁的Redis节点个数少于N/2+1，或者整个获取锁的过程消耗的时间超过了锁的最初有效时间），那么客户端应该立即向所有Redis节点发起释放锁的操作

释放锁的过程比较简单：客户端向所有Redis节点发起释放锁的操作，不管这些节点当时在获取锁的时候成功与否。**即使当时向某个节点获取锁没有成功，在释放锁的时候也不应该漏掉这个节点。**

前面讨论的单Redis节点的分布式锁在failover的时候锁失效的问题，在Redlock中不存在了，但如果有节点发生崩溃重启，还是会对锁的安全性有影响的。具体的影响程度跟Redis对数据的持久化程度有关。

假设一共有5个Redis节点：A, B, C, D, E。设想发生了如下的事件序列：

1. 客户端1成功锁住了A, B, C，获取锁成功（但D和E没有锁住）
2. 节点C崩溃重启了，但客户端1在C上加的锁没有持久化下来，丢失了
3. 节点C重启后，客户端2锁住了C, D, E，获取锁成功。

这样，客户端1和客户端2同时获得了锁（针对同一资源）

在默认情况下，Redis的AOF持久化方式是每秒写一次磁盘（即执行fsync），因此最坏情况下可能丢失1秒的数据。为了尽可能不丢数据，Redis允许设置成每次修改数据都进行fsync，但这会降低性能。当然，即使执行了fsync也仍然有可能丢失数据（这取决于系统而不是Redis的实现）。所以，上面分析的由于节点重启引发的锁失效问题，总是有可能出现的。为了应对这一问题，antirez又提出了延迟重启(delayed restarts)的概念。也就是说，一个节点崩溃后，先不立即重启它，而是等待一段时间再重启，这段时间应该大于锁的有效时间(lock validity time)。这样的话，这个节点在重启前所参与的锁都会过期，它在重启后就不会对现有的锁造成影响。

为什么释放锁的时候即使当时向某个节点获取锁没有成功，在释放锁的时候也不应该漏掉这个节点？

设想这样一种情况，客户端发给某个Redis节点的获取锁的请求成功到达了该Redis节点，这个节点也成功执行了SET操作，但是它返回给客户端的响应包却丢失了。这在客户端看来，获取锁的请求由于超时而失败了，但在Redis这边看来，加锁已经成功了。因此，释放锁的时候，客户端也应该对当时获取锁失败的那些Redis节点同样发起请求。实际上，这种情况在异步通信模型中是有可能发生的：客户端向服务器通信是正常的，但反方向却是有问题的。

**Redlock依然没有解决这个问题：如果客户端长期阻塞导致锁过期，那么它接下来访问共享资源就不安全**

# zookeeper
zookeeper的分布式锁主要是通过临时节点实现

1. 客户端尝试创建一个znode节点，比如/lock。那么第一个客户端就创建成功了，相当于拿到了锁；而其它的客户端会创建失败（znode已存在），获取锁失败。
2. 持有锁的客户端访问共享资源完成后，将znode删掉，这样其它客户端接下来就能来获取锁了。
3. znode应该被创建成ephemeral的。这是znode的一个特性，它保证如果创建znode的那个客户端崩溃了，那么相应的znode会被自动删除。这保证了锁一定会被释放。
4. 当客户端试图创建/lock的时候，发现它已经存在了，这时候创建失败，但客户端不一定就此对外宣告获取锁失败。客户端可以进入一种等待状态，等待当/lock节点被删除的时候，ZooKeeper通过watch机制通知它，这样它就可以继续完成创建操作（获取锁）

每个客户端都与ZooKeeper的某台服务器维护着一个Session，这个Session依赖定期的心跳(heartbeat)来维持。如果ZooKeeper长时间收不到客户端的心跳（这个时间称为Sesion的过期时间），那么它就认为Session过期了，通过这个Session所创建的所有的ephemeral类型的znode节点都会被自动删除。

但是，如果客户端发生GC卡顿，也会导致session过期，引起锁冲突

更多内容参考[zookeeper相关文章](https://edgar615.github.io/zookeeper.html)

# 哪些地方可以用的分布式锁

如果不同的系统或是同一个系统的不同主机之间共享了一个或一组资源，那么访问这些资源的时候，往往需要互斥来防止彼此干扰来保证一致性，在这种情况下，便需要使用到分布式锁。

- 定时从数据表中拉取未正常处理的任务处理，且任务不能被重复执行
- 库存对商品加锁防止超卖
- 比较敏感的数据比如金额修改，同一时间只能有一个人操作

# 锁粒度优化

用分布式锁来防止库存超卖，但是是每秒上千订单的高并发场景，如何对分布式锁进行优化？

参考ConcurrentHashMap的设计，将核心数据拆分后分段加锁

1. 将库存拆分为N(如20)个段
2. 将请求随机分配到某个库存段
3. 对库存段加锁判断库存是否足够，如果不足立即释放锁并尝试对下一个库存段加锁
4. 处理业务，释放锁

# 参考资料

https://mp.weixin.qq.com/s/0WsfrweMVCamK7co6Ki8aw

https://mp.weixin.qq.com/s/1bPLk_VZhZ0QYNZS8LkviA

https://mp.weixin.qq.com/s/1HvQJaUKHcAqSa224efNmw

https://mp.weixin.qq.com/s/d_IgPDFWyPLHRcT2_yunMQ

https://mp.weixin.qq.com/s/RLeujAj5rwZGNYMD0uLbrg