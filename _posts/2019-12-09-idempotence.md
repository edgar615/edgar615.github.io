---
layout: post
title: 幂等
date: 2019-12-09
categories:
    - 架构设计
comments: true
permalink: idempotence.html
---

先看两个场景

**用户下单**

![](/assets/images/posts/idempotence/idempotence-1.png)

用户在下订单的时候，点击确认之后，因为网络原因没反应，就又点击了几次。在这种情况下，如果无法保证该接口的幂等性，那么将会出现重复下单问题。

**用户支付**

![](/assets/images/posts/idempotence/idempotence-2.png)

用户支付订单后，服务端成功处理了第三方支付（支付宝/微信）回调通知，执行了扣库存等业务逻辑，但因为网络原因，第三方没有收到服务端返回的确认，第三方会再次发起回调通知，服务端会收到多笔支付结果。如果无法保证该接口的幂等性，那么将会出现重复支付，多扣库存等问题。

# 1. 什么是幂等

在数学里，幂等有两种主要的定义：- 在某二元运算下，幂等元素是指被自己重复运算(或对于函数是为复合)的结果等于它自己的元素。例如，乘法下唯一两个幂等实数为0和1。即 `s *s = s`- 某一元运算为幂等的时，其作用在任一元素两次后会和其作用一次的结果相同。例如，高斯符号便是幂等的，即`f(f(x)) = f(x)`。

在HTTP/1.1规范中幂等性的定义是：

> A request method is considered “idempotent” if the intended effect on  the server of multiple identical requests with that method is the same  as the effect for a single such request. Of the request methods defined  by this specification, PUT, DELETE, and safe request methods are  idempotent.

HTTP的幂等性指的是一次和多次请求某一个资源应该具有相同的副作用。如通过PUT接口将数据的Status置为1，无论是第一次执行还是多次执行，获取到的结果应该是相同的，即执行完成之后Status =1。

在HTTP规范中定义GET,PUT和DELETE方法应该具有幂等性。(不会对系统产生副作用，不代表每次请求都是相同的结果)

在我们的实际业务上，**幂等性是指用户对于同一操作发起的一次请求或者多次请求的结果是一致的，不会因为多次点击而产生了副作用。**

# 2. 为什么需要幂等性

- 在App中下订单的时候，点击确认之后，没反应，就又点击了几次。在这种情况下，如果无法保证该接口的幂等性，那么将会出现重复下单问题。
- 在接收消息的时候，消息推送重复。如果处理消息的接口无法保证幂等，那么重复消费消息产生的影响可能会非常大。例如短信重复发送
- 单个订单请求调用支付接口，当遇到网络或系统故障请求重发，支付服务就会收到一个支付请求多次；
- 一个订单状态更新接口，调用方连续发送了两个消息，一个是已创建，一个是已付款。但是你先接收到已付款，然后又接收到了已创建


在分布式环境中，网络环境更加复杂，因前端操作抖动、网络故障、消息重复、响应速度慢等原因，对接口的重复调用概率会比集中式环境下更大，尤其是重复消息在分布式环境中很难避免。

假如我们在 5 台机器上部署了支付服务，订单服务调用支付服务因为网络超时触发了重试机制，支付服务就会收到一个支付请求两次，而且因为负载均衡算法落在了不同的机器上。

> 在分布式环境中，网络问题很常见，100次请求，都成功。1万次，可能1次是超时会重试。10万，10次。100万，100次。如果有100个请求重复了，你没处理，导致订单扣款2次，100个订单都扣错了；每天被100个用户投诉；一个月被3000个用户投诉。

广义上的RPC，包括客户端对服务端的api调用、后端系统的内网调用、跨机房调用等。一次RPC大体上包括三个步骤：发送请求、执行过程、接收响应。由于网络传输存在不确定性，导致RPC调用存在一个陷阱，即有可能出现第一、第二步都成功、第三步失败的情况，此时RPC的调用方由于接收不到结果，无法判断被调用方是否已经完成过程调用，只能按失败处理。

通常RPC调用方会针对网络失败进行重试。在上述情况下，如果远端代码不具备幂等性，却进行了重试，将导致系统的数据一致性遭到破坏，本该只执行一次的事务被执行了两次。

对于异步消息的消费者来讲，也有同样的问题。在手动ACK的情况下，消息的处理需要接收消息、处理消息、ACK三步，ACK的失败也会导致相同的问题。

为了解决以上问题，就需要保证接口的幂等性，接口的幂等性实际上就是**接口可重复调用，在调用方多次调用的情况下，接口最终得到的结果是一致的**。

有些接口可以天然的实现幂等性：

- **查询操作** 查询一次和查询多次，在数据不变的情况下，查询结果是一样的。select是天然的幂等操作 
- **删除操作** 删除操作也是幂等的，删除一次和多次删除都是把数据删除。(注意可能返回结果不一样，删除的数据不存在，返回0，删除的数据多条，返回结果多个) 

对于数据库操作，有下面三种场景，只有第三种场景需要开发人员使用其他策略保证幂等性：

1. `SELECT col1 FROM tab1 WHER col2=2`，无论执行多少次都不会改变状态，是天然的幂等。
2. `UPDATE  tab1 SET col1=1 WHERE col2=2`，无论执行**成功**多少次**状态**都是一致的，因此也是幂等操作。
3. `UPDATE  tab1 SET col1=col1+1 WHERE col2=2`，每次执行的结果都会发生变化，这种不是幂等的。

> 这也是为什么我们在扣库存、扣余额时候不应该使用第三种方式实现。

**为什么要设计幂等性的服务**

幂等可以使得客户端逻辑处理变得简单，但是却以服务逻辑变得复杂为代价。满足幂等服务的需要在逻辑中至少包含两点：

1. 首先去查询上一次的执行状态，如果没有则认为是第一次请求
2. 在服务改变状态的业务逻辑前，保证防重复提交的逻辑

# 3. 解决方案

##  3.1. 去重表/唯一索引

利用数据库的特性来实现幂等。通常是在表上构建一个唯一索引，那么只要某一个数据构建完毕，后面再次操作也无法成功写入。这种方法适用于在业务中有唯一标识的插入场景中，如微信、支付宝提供的回调接口都会返回一个交易单号，我们可以将交易单号作为唯一索引，防止多次回调触发回调逻辑

![](/assets/images/posts/idempotence/idempotence-3.png)

去重表可以和业务单据表使用同一个，也可以单独做一个去重表

模拟实现：

```
CREATE TABLE `idempotent_check` (
`id` BIGINT ( 20 ) NOT NULL AUTO_INCREMENT COMMENT '主键',
`business_type` TINYINT UNSIGNED ( 4 ) NOT NULL COMMENT '业务类型',
`business_id` BIGINT ( 20 ) NOT NULL COMMENT '业务ID',
`update_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '保存数据时间，自动生成',
PRIMARY KEY ( `id` ),
UNIQUE KEY `uidx_business` ( `business_id`,`business_type` ) USING BTREE 
) ENGINE = INNODB DEFAULT CHARSET = utf8 COMMENT = '业务去重表';
```

1. 查询订单是否正在处理

```
select * from idempotent_check where business_id = 交易单号 and business_type = 支付
```
如果查询到记录，说明订单正在处理，直接返回，如果未查询到记录进入下一步

2. 插入去重表

```
try{
    insert into idempotent_check (business_type,business_id) values ('支付',交易单号);
}catch(Exception e){
    回滚本地事务，或者执行其他业务逻辑;
    
}
```

3. 执行业务代码

缺点：在高并发下存在数据库瓶颈。可以根据业务进行分库分表策略，采用一些路由算法去进行分流。应该保证通过路由算法，业务的唯一标识每次都落到同一个数据库分片上，这样就由单台数据库幂等变成多库的幂等。

##  3.2. 状态机控制

通过状态标识的变更，保证业务中每个流程只会在对应的状态下执行，如果标识已经进入下一个状态，这时候来了上一个状态的操作就不允许变更状态，保证了业务的幂等性。

状态标识经常用在业务流程较长，修改数据较多的场景里。最经典的例子就是订单系统，假如一个订单要经历 创建订单 -> 订单支付\取消 -> 账户计算 -> 通知商户 这四个步骤。那么就有可能一笔订单支付完成后去账户里扣除对应的余额，消耗对应的优惠卷。但是由于网络等原因返回了错误信息，这时候就会重试再次去进行账户计算步骤造成数据错误。所以为了保证整个订单流程的幂等性，可以在订单信息中增加一个状态标识，一旦完成了一个步骤就修改对应的状态标识。比如订单支付成功后，就把订单标识为修改为支付成功，现在再次调用订单支付或者取消接口，会先判断订单状态标识，如果是已经支付过或者取消订单，就不会再次支付了。

```
if (order.state == '等待支付') {
	//支付逻辑
} else {
	throw err('已支付')
}
```

但是在高并发场景下，MySQL的MVCC机制会导致不同事务看不到已提交的状态，导致幂等失效。所以我们需要结合去重表或者乐观锁来实现幂等。

## 3.3. 悲观锁 

获取数据的时候加锁获取 
```
select * from table_xxx where id='xxx' for update; 
```
注意：id字段一定是主键或者唯一索引，不然是锁表

悲观锁使用时一般伴随事务一起使用，数据锁定时间可能会很长，根据实际情况选用 

##  3.4. 乐观锁
乐观锁只是在更新数据那一刻锁表，其他时间不锁表，所以相对于悲观锁，效率更高。乐观锁的实现方式多种多样可以通过version或者其他状态条件：

1. 通过版本号
```
update account set available_amount=#availableAmount#,version=version+1 where id=#id# and version=#version# 
```
2. 通过条件限制
```
update account set available_mount=#availableAmount# where id=#id# and available_mount >= #subAmount#
update t_order set status = '已支付' where order_id = trade_no where status = '等待支付';
```

![](/assets/images/posts/idempotence/idempotence-5.png)

注意：乐观锁的更新操作，最好用主键或者唯一索引来更新，否则更新时会锁表。

## 3.5. token机制

每次操作都生成一个唯一 Token 凭证，服务器通过这个唯一凭证保证同样的操作不会被执行两次。这个 Token 除了字面形式上的唯一字符串，也可以是多个标志的组合，甚至可以是时间段标识等等。

1. 数据提交前要向服务的申请token，token放到redis或jvm内存，token要有有效时间
2. 提交后后台验证token，再将 token 标记为已执行（或者删除token）
3. 执行业务逻辑

![](/assets/images/posts/idempotence/idempotence-4.png)

注意：在redis里如果用select+delete来校验token，存在并发问题，所以建议使用删除操作来判断token，删除成功代表token校验通过。

**这个从严格上来说我认为是防重，应该和幂等区分**

## 3.6. 分布式锁 
对于插入数据来说，如果是分布是系统，构建全局唯一索引比较困难。这时候可以引入分布式锁，通过第三方的系统(redis或zookeeper)，在业务系统插入数据或者更新数据，获取分布式锁，然后做操作，之后释放锁。

这样其实是把多线程并发的锁的思路，引入到多个系统，也就是分布式系统中得解决思路。 

注意：某个长流程处理过程要求不能并发执行，可以在流程执行之前根据某个标志(用户ID+后缀等)获取分布式锁，其他流程执行时获取锁就会失败，也就是同一时间该流程只能有一个能执行成功，执行完成后，释放分布式锁(分布式锁要第三方系统提供) 

# 4. 误区

对于前面描述的乐观锁、token机制等，我们看到在重试的过程中返回了失败。实际上这违法了幂等的原则：**用户对于同一操作发起的一次请求或者多次请求的结果是一致的**，针对这种情况，我们需要增加一个流水日志表，记录每一次操作的结果

![](/assets/images/posts/idempotence/idempotence-6.png)

# 5. 参考资料

https://blog.csdn.net/jks456/article/details/71453053

http://geyifan.cn/2016/06/02/idempotent-solutions/