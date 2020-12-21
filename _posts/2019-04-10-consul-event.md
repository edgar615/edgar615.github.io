---
layout: post
title: Consul - event
date: 2019-04-10
categories:
    - Consul
comments: true
permalink: consul-event.html
---

event 命令提供了一种机制来将自定义用户事件触发到整个数据中心。 这些事件对Consul来说是不透明的，但是它们可以用来构建脚本基础架构来执行自动化部署，重新启动服务或执行其他编排操作。 事件可以通过使用watch来处理。

命令行发布事件

```
$ consul event -h
Usage: consul event [options] [payload]

  Dispatches a custom user event across a datacenter. An event must provide
  a name, but a payload is optional. Events support filtering using
  regular expressions on node name, service, and tag definitions.
```

```
-http-addr：http服务的地址，agent可以链接上来发送命令，如果没有设置，则默认是127.0.0.1:8500。
-token：ACL token，也可以通过环境变量CONSUL_HTTP_TOKEN指定 
-token-file= ACL token的文件，也可以通过环境变量CONSUL_HTTP_TOKEN_FILE指定
-datacenter：数据中心。
-name：事件的名称
-node：一个正则表达式，用来过滤节点
-service：一个正则表达式，用来过滤节点上匹配的服务
-tag：一个正则表达式，用来过滤节点上符合tag的服务，必须和-service一起使用。
```

测试

```
$ consul event -name=web-deploy 1609030
Event ID: 78b4e4f0-f40a-77ef-98af-c1de84713354
```

http发布事件

```
$ curl \
>     --request PUT \
>     --data '1609030' \
>     http://127.0.0.1:8500/v1/event/fire/web-deploy
{
    "ID": "23d97d58-2320-7bd0-4818-ca9c1267f9e5",
    "Name": "web-deploy",
    "Payload": "MTYwOTAzMA==",
    "NodeFilter": "",
    "ServiceFilter": "",
    "TagFilter": "",
    "Version": 1,
    "LTime": 0
}

```

查看事件

```
$ curl -s http://localhost:8500/v1/event/list
[
    {
        "ID": "78b4e4f0-f40a-77ef-98af-c1de84713354",
        "Name": "web-deploy",
        "Payload": "MTYwOTAzMA==",
        "NodeFilter": "",
        "ServiceFilter": "",
        "TagFilter": "",
        "Version": 1,
        "LTime": 2
    },
    {
        "ID": "23d97d58-2320-7bd0-4818-ca9c1267f9e5",
        "Name": "web-deploy",
        "Payload": "MTYwOTAzMA==",
        "NodeFilter": "",
        "ServiceFilter": "",
        "TagFilter": "",
        "Version": 1,
        "LTime": 3
    }
]
$ curl -s http://localhost:8500/v1/event/list?name=web-deploy2
[]

```

接口的详细内容参考官方文档

> https://www.consul.io/api-docs/event