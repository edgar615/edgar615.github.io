---
layout: post
title: Consul - Leader选举
date: 2019-04-06
categories:
    - Consul
comments: true
permalink: consul-leader-election.html
---

> 分布式锁的实现和Leader选举基本一样

# 1. Leader选举的步骤

Consul的Leader选举只有两步：

1. Create Session：参与选举的应用分别去创建Session，Session的存活状态关联到健康检查。

2. Acquire KV：多个应用带着创建好的Session去锁定同一个KV，只能有一个应用锁定住，锁定成功的应用就是leader。

![](/assets/images/posts/consul/consul-leader-election-1.jpg)

如上图所示，这里假设App2用Session锁定住了KV，其实就是KV的Session属性设置为了Session2。

**什么时候会触发重新选举呢？**

- Session失效：Session被删除、Session关联的健康检查失败、Session TTL过期等。
- KV被删除：

**那应用们怎么感知这些情况呢？**

应用在选举结束后，应该保持一个到KV的阻塞查询，这个查询会在超时或者KV发生变化的时候返回结果，这时候应用可以根据返回结果判断是否发起新的选举。

```
# 创建session
curl  -X PUT -d '{"Name": "dbservice", "TTL": "120s"}' http://localhost:8500/v1/session/create

# 保持session
curl  -X PUT http://localhost:8500/v1/session/renew/1adeff53-7740-aa78-13ac-e3a91fe6d40d

# 创建KV
curl -X PUT -d 'some data' http://localhost:8500/v1/kv/lead?acquire=1adeff53-7740-aa78-13ac-e3a91fe6d40d

# 释放KV
curl -X PUT http://localhost:8500/v1/kv/lead?release=1adeff53-7740-aa78-13ac-e3a91fe6d40d

# 查看KV
curl -X GET http://localhost:8500/v1/kv/lead

# 阻塞式查询
curl -X GET http://localhost:8500/v1/kv/lead?wait=10s&index=1100
```

# 2. 注意事项

1. 用于选举的Consul KV必须使用锁定的session进行更新，如果通过其它方式更新，KV绑定的Session不会有影响，也就是说KV还是被原来的程序锁定，但是却被其它的程序修改了，这不符合leader的规则。
2. Session 关联的健康检查默认只有当前节点的健康检查，如果应用程序停止，Session并不会失效，所以建议将Session  关联的健康检查包含应用的健康检查；但是如果只有应用的健康检查，服务器停止，应用的健康检查仍可能是健康的，所以Session的健康检查应该把应用程序和Consul 节点的健康检查都纳入进来。
3. 如果程序关闭后很快启动，session关联的健康检查可能不会失败，所以session不会失效，程序启动后如果创建一个新的Session去锁定KV，就不能成功锁定KV，这时候建议将SessionId持久化存储，如果Session还存在，就还是用这个Session 去锁定。
4. lockdelay：这是一个保护机制，但需要考虑好用不用。它可以在session失效后短暂的不允许其它session来锁定KV，这是因为此时应用程序可能还是正常的，session关联的健康检查误报了，应用程序可能还在处理业务，需要一段时间来结束处理。也可以使用0值把这个机制禁止掉。为防止由于网络波动等原因，session的状态被错误的检查为`invalidate` 导致锁被释放。此时如果其它client需要加锁，则需要等待`lock-delay`，才能再次加锁成功。**(主动释放没有这个问题)**
5. 可能leader加载的东西比较多，leader切换比较麻烦，考虑到session关联的健康检查误报的问题，希望leader选举优先上次锁定KV的程序，这样可以提高效率。此时可以在选举程序中增加一些逻辑：如果选举的时候发现上次的leader是当前程序，则立即选举；如果发现上次的leader不是当前程序，则等待两个固定的时间周期再提交选举。

# 3. 参考资料

http://blog.bossma.cn/consul/consul-leader-election-solution/