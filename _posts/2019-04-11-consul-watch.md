---
layout: post
title: Consul - watch
date: 2019-04-11
categories:
    - Consul
comments: true
permalink: consul-watch.html
---

# 1. Watch

Watches是查看指定数据信息的一种方法，比如查看nodes列表、键值对、健康检查。当监控到更新时，可以调用外部处理程序——可以自定义。比如，发现健康状态发生变化可以通知外部系统健康异常。

Watches在调用http api接口时使用阻塞队列。Agent会自动调用合适的API接口来监控数据的变化。

Watches可以作为Agent配置的一部分。在Agent初始化时就运行，并且支持重新载入配置——运行时新添加或删除配置。

在任意情况下，watches的type都必须指定。Watch支持的每一个type需要的不同的参数，一些是必须的一些事非必须的。这些都是通过JSON来设置的。

Watch配置可以指定监控的数据。一旦数据发生变化，可以运行指定的处理程序——可以是任意可执行的程序。

处理程序可以从标准输入中读取输入，也可以读取json数据。数据格式依赖于watch类型。Watch类型与Json格式是想对应的。因为watch是直接调用HTTP API，因此输入数据要格式化。

目前conusl watch支持两种通知方式：**可执行程序**和**Http接口**。

consul watch支持监控的数据类型：

- Key – 监视指定K/V键值对
- Keyprefix – Watch a prefix in the KV store
- Services – 监视服务列表
- nodes – 监控节点列表
- service – 监视服务实例
- checks- 监视健康检查的值
- event – 监视用户事件

从以上可以看出consul提供非常丰富的监听类型，通过这些类型我们可以实时观测到consul整个集群中的变化，从而实现一些特别的需求，比如：服务告警，配置实时更新等功能。

参数

```
-type：监控的数据类型： key, keyprefix, services, nodes, service, checks, event
-key：监控的键值数据的Key,只有type是key时需要
-prefix：监控的键值数据的Keyrefix,只有type是keyprefix时需要
-service：监控的服务的服务名,只有type是service时需要
-tag: 监控的服务的服务tag，可以指定多次,只有type是service时需要
-state: 监控的服务状态,只有type是checkes时需要
-shell: 监控到数据变化后调用的shell
handler_type - 通知类型，支持script和http
args - 配置通知类型为script的，执行命令，是一个数组，第一个元素是命令，后面第2个到第N个元素是命令的参数。
handler – 监控到数据变化后的调用程序。
```

下面我们先通过key来看一下watch的用法

# 2. key

```
$ echo '#!/bin/sh
exec >> /server/logs/my-key-watch
IFS=" "
while read a
do
    echo $a
done' > /server/shell/consul/my-key-handler.sh

$ chmod a+x /server/shell/consul/my-key-handler.sh
```

## 2.1. 通过配置文件创建

```
$ echo '{
  "watches": [
    {
      "type": "key",
      "key": "foo/bar/baz",
      "handler": "/server/shell/consul/my-key-handler.sh"
    }
  ]
}' > /server/data/consul/consul-key-watch.json
```

测试

```
$ cat /server/logs/my-key-watch
null
$ consul kv put foo/bar/baz 1
Success! Data written to: foo/bar/baz
$ cat /server/logs/my-key-watch
null
{"Key":"foo/bar/baz","CreateIndex":17,"ModifyIndex":17,"LockIndex":0,"Flags":0,"Value":"MQ==","Session":""}
$ consul kv put foo/bar/baz 2
Success! Data written to: foo/bar/baz
$ cat /server/logs/my-key-watch
null
{"Key":"foo/bar/baz","CreateIndex":17,"ModifyIndex":17,"LockIndex":0,"Flags":0,"Value":"MQ==","Session":""}
{"Key":"foo/bar/baz","CreateIndex":17,"ModifyIndex":21,"LockIndex":0,"Flags":0,"Value":"Mg==","Session":""}

```

配置文件可以采用另一种写法，通过args可以传递多个参数

```
$ echo '{
  "watches": [
    {
      "type": "key",
      "key": "foo/bar/baz",
	  "handler_type": "script",
      "args": ["/server/shell/consul/my-key-handler.sh"]
    }
  ]
}' > /server/data/consul/consul-key-watch.json
```

## 2.2. 通过命令行创建

```
$ consul watch -type key -key foo/bar/baz /server/shell/consul/my-key-handler.sh &
```

## 2.3. 通过HTTP通知

```
$ echo '{
    "watches": [
        {
            "type": "key", 
            "key": "foo/bar/baz", 
            "handler_type": "http", 
            "http_handler_config": {
                "path": "http://localhost:9000/watch", 
                "method": "POST", 
                "header": {
                    "x-foo": [
                        "bar", 
                        "baz"
                    ]
                }, 
                "timeout": "10s"
            }
        }
    ]
}' > /server/data/consul/consul-key-watch.json
```

HTTP收到的通知

```
Host=localhost:9000
User-Agent=Go-http-client/1.1
Content-Length=108
Content-Type=application/json
X-Consul-Index=24
X-Foo=bar
X-Foo=baz
Accept-Encoding=gzip
Connection=close

{"Key":"foo/bar/baz","CreateIndex":24,"ModifyIndex":24,"LockIndex":0,"Flags":0,"Value":"MQ==","Session":""}

```

# 3. keyprefix

```
$ echo '{
    "watches": [
        {
            "type": "keyprefix", 
            "key": "foo/", 
            "handler_type": "script", 
            "args": [
                "/server/shell/consul/my-key-handler.sh"
            ]
        }
    ]
}' > /server/data/consul/consul-key-watch.json
```

```
$ consul watch -type keyprefix -prefix foo/ /server/shell/consul/my-key-handler.sh &
```

# 4. services

```
$ echo '{
    "watches": [
        {
            "type": "services", 
            "handler_type": "script", 
            "args": [
                "/server/shell/consul/my-key-handler.sh"
            ]
        }
    ]
}' > /server/data/consul/consul-key-watch.json
```

内部接口`/v1/catalog/services`

```
# curl -s http://localhost:8500/v1/catalog/services
{
    "baidu": [],
    "consul": [],
    "redis": [
        "master"
    ],
    "web": [
        "java"
    ]
}
```

# 5. nodes

```
$ echo '{
    "watches": [
        {
            "type": "nodes", 
            "handler_type": "script", 
            "args": [
                "/server/shell/consul/my-key-handler.sh"
            ]
        }
    ]
}' > /server/data/consul/consul-key-watch.json
```

内部接口`/v1/catalog/nodes`

```
$ curl -s http://localhost:8500/v1/catalog/nodes
[
    {
        "ID": "fce5ac85-f41b-761c-492e-fe42b1ae4102",
        "Node": "VM-0-17-centos",
        "Address": "127.0.0.1",
        "Datacenter": "dc1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "lan_ipv4": "127.0.0.1",
            "wan": "127.0.0.1",
            "wan_ipv4": "127.0.0.1"
        },
        "Meta": {
            "consul-network-segment": ""
        },
        "CreateIndex": 11,
        "ModifyIndex": 13
    }
]

```

# 6. service

```
echo '{
    "watches": [
        {
            "type": "service",
			"service": "web",
			"tag": "java",
            "handler_type": "script", 
            "args": [
                "/server/shell/consul/my-key-handler.sh"
            ]
        }
    ]
}' > /server/data/consul/consul-key-watch.json
```

tag为可以选参数，支持数组`"tag": ["bar", "foo"]`

在命令行下可以同时传入多个`-tag=bar -tag=foo `

内部接口`/v1/health/service`

# 7. checks

```
$ echo '{
    "watches": [
        {
            "type": "checks", 
            "state": "passing", 
            "handler_type": "script", 
            "args": [
                "/server/shell/consul/my-key-handler.sh"
            ]
        }
    ]
}' > /server/data/consul/consul-key-watch.json
```

或者

```
$ echo '{
    "watches": [
        {
            "type": "checks", 
            "service": "web", 
            "handler_type": "script", 
            "args": [
                "/server/shell/consul/my-key-handler.sh"
            ]
        }
    ]
}' > /server/data/consul/consul-key-watch.json
```

内部接口`/v1/health/state/`

# 8. event

```
echo '{
    "watches": [
        {
            "type": "event", 
            "name": "web-deploy", 
            "handler_type": "script", 
            "args": [
                "/server/shell/consul/my-key-handler.sh"
            ]
        }
    ]
}' > /server/data/consul/consul-key-watch.json
```

内部接口`/v1/event/list`

```
$ consul event -name=web-deploy 1609030
Event ID: 78b4e4f0-f40a-77ef-98af-c1de84713354
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
    }
]

```





