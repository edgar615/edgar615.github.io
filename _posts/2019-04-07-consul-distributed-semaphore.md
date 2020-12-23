---
layout: post
title: Consul - 信号量
date: 2019-04-07
categories:
    - Consul
comments: true
permalink: consul-distributed-semaphore.html
---

信号量是我们在实现并发控制时会经常使用的手段，主要用来限制同时并发线程或进程的数量。

在Leader选举（分布式锁）中我们只用到了K，没有用到V。如果把V也用上的话，就可以实现一个分布式信号量。

- 1.创建session

```
$ curl  -X PUT -d '{"Name": "db-semaphore","TTL": "120s"}' \
>   http://localhost:8500/v1/session/create
{
    "ID": "6a0777cc-1b2b-c7f5-6678-8a3c6a1b2550"
}
```

- 2.创建一个锁，用于初始化信号量

```
$ curl -X PUT -d 'some semaphore' \
> http://localhost:8500/v1/kv/semaphore/key?acquire=6a0777cc-1b2b-c7f5-6678-8a3c6a1b2550
true
```

- 3.使用CAS=0初始化信号量

```
$ curl -X PUT -d '{"Limit": "2","Holders": ["6a0777cc-1b2b-c7f5-6678-8a3c6a1b2550"]}' \
> http://localhost:8500/v1/kv/semaphore/key/.lock?cas=0
true
```

- 4.查看信号量

```
$ curl http://localhost:8500/v1/kv/semaphore/key?recurse
[
    {
        "LockIndex": 1,
        "Key": "semaphore/key",
        "Flags": 0,
        "Value": "c29tZSBzZW1hcGhvcmU=",
        "Session": "6a0777cc-1b2b-c7f5-6678-8a3c6a1b2550",
        "CreateIndex": 240,
        "ModifyIndex": 242
    },
    {
        "LockIndex": 0,
        "Key": "semaphore/key/.lock",
        "Flags": 0,
        "Value": "eyJMaW1pdCI6ICIyIiwiSG9sZGVycyI6IFsiNmEwNzc3Y2MtMWIyYi1jN2Y1LTY2NzgtOGEzYzZhMWIyNTUwIl19",
        "CreateIndex": 243,
        "ModifyIndex": 243
    }
]
```

- 5.查看`semaphore/key/.log` ，如果holders小于limit，可以通过CAS添加一个信号量

base64解码后的值为

```
{"Limit": "2","Holders": ["6a0777cc-1b2b-c7f5-6678-8a3c6a1b2550"]}
```

- 6.同样基于CAS释放信号量

# 参考资料

https://learn.hashicorp.com/tutorials/consul/distributed-semaphore

https://blog.csdn.net/dyc87112/article/details/73739518