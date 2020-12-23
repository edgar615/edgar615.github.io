---
layout: post
title: Consul - Session
date: 2019-04-04
categories:
    - Consul
comments: true
permalink: consul-session.html
---

Consul提供session会话机制——可以用于构建分布式锁，session可以绑定到节点、健康检查、KV数据。目的是提供颗粒锁。

 Consulsession代表一个有非常具体的语义——合约，当构建一个session时，需要提供节点名称，健康检查列表，行为， TTL，锁期限。。新建session返回唯一的标识ID。此ID可用于作为与KV操作的锁：互斥机制。

# 1. 设计

 下面一个各部分间的关系图：

![](/assets/images/posts/consul/consul-session-1.png)

Consul提供的合约，发生如下任一情况，session都会被销毁：

​    •节点销毁

​    •任何的健康检查，注销

​    •任何健康检查都要进入Critical状态

​    •会话被显式销毁

​    •TTL到期，如果适用

当session失效后，会被销毁，不能再使用。关联的锁发生什么行为，取决于创建时指定的修饰符。Consul支持release和delete两种处理方式。如果没有指定，默认为release。

- 如果使用release，任何与该session相关的锁都会被释放，并且持有该锁的key的ModifyIndex也会递增。
- 如果使用了delete，持有该锁的KEY将会被删除。

虽然这是一个简单的设计，它可以实现多种使用模式。 默认情况下，使用[gossip based failure detector](https://www.consul.io/docs/internals/gossip.html)作为相关的健康检查。 该故障检测器允许Consul检测持有锁的节点何时发生故障并自动释放锁。 这种能力为consul锁提供了**活力** ;

 那就是在失败的情况下，系统可以继续取得进展。 然而，由于没有完美的故障检测器，因此即使锁所有者仍然存在，也可能存在一个假阳性（检测到故障），导致锁释放。 这意味着我们正在牺牲一些**安全** 。

相反，可以创建一个没有关联的健康检查的session。 这消除了对于安全性的假阳性和交易活动的可能性。 即使现有的所有者失败，您也可以绝对地确定consul不会释放锁定。 由于consulAPI允许强制破坏session，因此允许构建系统，

要求操作员在发生故障的情况下进行干预，同时排除脑裂的可能性。

第三个健康检查机制是sessionTTL。 创建session时，可以指定TTL。 如果TTL间隔到期而不更新，则session已过期，并且触发invalidated。 这种类型的故障检测器也称为心跳失效检测器。 它比基于gossip的故障检测器的可扩展性更低，因为它对服务器造成了更大的负担，

但在某些情况下可能适用。  TTL的合约是它代表invalidated的下限; 也就是说，在达到TTL之前，consul不会过期session，但允许延迟超过TTL的期限。 在session创建，session更新和领导者故障转移时，更新TTL。 当使用TTL时，客户端应该意识到时钟偏移问题：

即时间可能不会像在consul服务器上那样在客户端上以相同的速率进展。 最好设置保守的TTL值，并在TTL之前更新以考虑网络延迟和时间偏移。

最后的细微差别是session可能会提供`lock-delay` 。 这是0到60秒之间的持续时间。 当发生sessioninvalidated时，Consul防止在`lock-delay`时间间隔内重新获取先前保存的任何锁; 这是Google Chubby的灵感来源。 这种延迟的目的是允许潜在的活着的领导者检测到invalidated，

并停止可能导致不一致状态的处理请求。 虽然不是一个防弹方法，但它确实避免了将睡眠状态引入到应用程序逻辑中，并且可以帮助减轻许多问题。 默认情况下是使用15秒延迟，客户端可以通过提供零延迟值来禁用此机制。

# 2. HTTP

- 创建session

```
$ curl -X PUT \
>   http://127.0.0.1:8500/v1/session/create \
>   -H 'content-type: application/json' \
>   -d '{
>     "LockDelay": "15s",
>     "Name": "my-service-lock",
>     "Behavior": "release",
>     "TTL": "30s"
> }'
{
    "ID": "c457f573-1d09-76c6-881b-d892ecdecd35"
}

```

返回session的ID

- 查看所有session

```1
$ curl http://localhost:8500/v1/session/list                                                      [
    {
        "ID": "62666b3e-bab8-f381-a0dc-b9007bf11168",
        "Name": "my-service-lock",
        "Node": "VM-0-17-centos",
        "LockDelay": 15000000000,
        "Behavior": "release",
        "TTL": "30s",
        "NodeChecks": [
            "serfHealth"
        ],
        "ServiceChecks": null,
        "CreateIndex": 3783,
        "ModifyIndex": 3783
    }
]

```

- 查看节点上的session

```
$ curl http://localhost:8500/v1/session/node/VM-0-17-centos
[
    {
        "ID": "62666b3e-bab8-f381-a0dc-b9007bf11168",
        "Name": "my-service-lock",
        "Node": "VM-0-17-centos",
        "LockDelay": 15000000000,
        "Behavior": "release",
        "TTL": "30s",
        "NodeChecks": [
            "serfHealth"
        ],
        "ServiceChecks": null,
        "CreateIndex": 3783,
        "ModifyIndex": 3783
    }
]

```

- 查看session详情

```
$ curl http://localhost:8500/v1/session/info/62666b3e-bab8-f381-a0dc-b9007bf11168
[
    {
        "ID": "62666b3e-bab8-f381-a0dc-b9007bf11168",
        "Name": "my-service-lock",
        "Node": "VM-0-17-centos",
        "LockDelay": 15000000000,
        "Behavior": "release",
        "TTL": "30s",
        "NodeChecks": [
            "serfHealth"
        ],
        "ServiceChecks": null,
        "CreateIndex": 3783,
        "ModifyIndex": 3783
    }
]

```

- 续租session

```
$ curl -X PUT http://localhost:8500/v1/session/renew/e1b10c14-e9ab-21dc-3b7a-1a704068a163
[
    {
        "ID": "e1b10c14-e9ab-21dc-3b7a-1a704068a163",
        "Name": "my-service-lock",
        "Node": "VM-0-17-centos",
        "LockDelay": 15000000000,
        "Behavior": "release",
        "TTL": "30s",
        "NodeChecks": [
            "serfHealth"
        ],
        "ServiceChecks": null,
        "CreateIndex": 3799,
        "ModifyIndex": 3799
    }
]

```

- 删除session

```
$ curl -X PUT http://localhost:8500/v1/session/destroy/e1b10c14-e9ab-21dc-3b7a-1a704068a163
true
```

# 3. KV集成

KV存储和session的集成是使用session的主要场景。必须在使用之前创建一个session，然后使用它的ID。

KV API支持acquire和release操作，acquire操作类似检查-设置（Check-And-Set）操作——只有当锁不存在持有者时才会返回成功。当成功时，某个normal标识会更新，也会递增LockIndex，当然也会更新session的信息。

如果在acquire操作时，与session相关的锁已经持有，那么LockIndex就不会递增，但是key值会更新，这就允许锁的当前持有者无需重新获得锁就可以更新key的内容。

一旦获得锁，所需要经release操作来释放（使用相同的session）。Release操作也类似于检查-设置（Check-And-Set）操作。如果给定的session无效，那么请求会失败。需要特别注意的是，无需经过session的创建者，lock也是可以被释放的。这种设计是允许操作者干预来终止会话，在需要的时候。如上所述，会话无效也将导致所有被持有的锁被释放或删除。当锁被释放时，LockIndex不会变化，但是session会被清空，并且ModifyIndex递增。

```
$ consul kv put -acquire -session=e6c8e941-4c80-1ead-cedc-80d66fb99e64 redis/config/minconns 1
Success! Lock acquired on: redis/config/minconns
$ consul kv get -detailed redis/config/minconns
CreateIndex      3874
Flags            0
Key              redis/config/minconns
LockIndex        1
ModifyIndex      3937
Session          e6c8e941-4c80-1ead-cedc-80d66fb99e64
Value            1

```

等到session到期后，再次查看KV

```
$ consul kv get -detailed redis/config/minconns
CreateIndex      3874
Flags            0
Key              redis/config/minconns
LockIndex        1
ModifyIndex      3940
Session          -
Value            1
```

创建2个session，再次测试

```
$ consul kv put -acquire -session=4ee167e5-32f4-0b21-2f25-d02b861dc06c redis/config/minconns 1
Success! Lock acquired on: redis/config/minconns

# consul kv put -acquire -session=4ee167e5-32f4-0b21-2f25-d02b861dc06c redis/config/minconns 3
Success! Lock acquired on: redis/config/minconns
$ consul kv put -acquire -session=4ee167e5-32f4-0b21-2f25-d02b861dc06c redis/config/minconns 4
Success! Lock acquired on: redis/config/minconns
$ consul kv put -acquire -session=79c92b78-0c0a-b273-2c34-24d538c2c558 redis/config/minconns 2
Error! Did not acquire lock
$ consul kv get -detailed redis/config/minconns
CreateIndex      3874
Flags            0
Key              redis/config/minconns
LockIndex        1
ModifyIndex      4013
Session          4ee167e5-32f4-0b21-2f25-d02b861dc06c
Value            4
$ consul kv put -release -session=79c92b78-0c0a-b273-2c34-24d538c2c558 redis/config/minconns 1
Error! Did not release lock
$ consul kv put redis/config/minconns 10
Success! Data written to: redis/config/minconns
$ consul kv put -release -session=79c92b78-0c0a-b273-2c34-24d538c2c558 redis/config/minconns 1
Error! Did not release lock
$ consul kv get -detailed redis/config/minconns
CreateIndex      3874
Flags            0
Key              redis/config/minconns
LockIndex        0
ModifyIndex      4017
Session          4ee167e5-32f4-0b21-2f25-d02b861dc06c
Value            10
$ consul kv put -release  redis/config/minconns 10
Error! Missing -session (required with -acquire and -release)
$ consul kv put -release -session=4ee167e5-32f4-0b21-2f25-d02b861dc06c redis/config/minconns
Success! Lock released on: redis/config/minconns
$ consul kv get -detailed redis/config/minconns
CreateIndex      3874
Flags            0
Key              redis/config/minconns
LockIndex        0
ModifyIndex      4021
Session          -
Value
$ consul kv get redis/config/minconns

```

创建一个删除行为的session

```
curl -X PUT \
  http://127.0.0.1:8500/v1/session/create \
  -H 'content-type: application/json' \
  -d '{
    "LockDelay": "5s", 
    "Name": "my-service-lock", 
    "Behavior": "delete", 
    "TTL": "30s"
}'
```

等到session过期后KV被自动删除了

```
$ consul kv put -acquire -session=8ccec390-a259-e527-9754-c99810f9cea1 redis/config/minconns 1
Success! Lock acquired on: redis/config/minconns
$ consul kv get -detailed redis/config/minconns
CreateIndex      3874
Flags            0
Key              redis/config/minconns
LockIndex        2
ModifyIndex      4042
Session          8ccec390-a259-e527-9754-c99810f9cea1
Value            1
$ consul kv get -detailed redis/config/minconns
Error! No key exists at: redis/config/minconns
```



# 4. 参考资料

https://www.cnblogs.com/mhc-fly/p/7228495.html