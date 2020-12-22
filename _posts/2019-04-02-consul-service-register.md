---
layout: post
title: Consul - 服务注册
date: 2019-04-02
categories:
    - Consul
comments: true
permalink: consul-service-register.html
---

# 1.  服务定义
```
echo '{
    "service": {
        "name": "web", 
        "tags": [
            "java"
        ], 
        "port": 8080, 
        "check": {
            "args": [
                "curl", 
                "-s", 
                "localhost:9000/health"
            ], 
            "interval": "10s", 
            "status": "passing"
        }
    }
}' > /server/data/consul/web.json
```

服务定义配置文件含义

- `id`服务ID，没有的话j就是name
- `name`服务名
- `tags`服务的tag自定义可以根据这个tag来区分同一个服务名的服务
- `address`服务注册到consul的IP服务发现发现的就是这个IP
- `port`服务注册consul的PORT发现的就是这个PORT
- `enable_tag_override`标签是否允许覆盖
- check  健康检查部分
  - `deregisterCriticalServiceAfter`
  - `args`指定健康检查的命令
  - `interval`健康检查间隔时间每隔10s调用一次

> 更多配置项查看官方文档 https://www.consul.io/docs/discovery/services

# 1. 通过配置目录

```
echo '{
    "service": {
        "id": "web1", 
        "name": "web", 
        "tags": [
            "java"
        ], 
        "port": 9000, 
        "check": {
            "http": "http://localhost:9000/health", 
            "interval": "10s", 
            "status": "passing"
        }
    }
}' > /server/data/consul/web.json

$ consul reload
Configuration reload triggered
```



# 2. 通过命令行

```
$ consul services register /server/tmp/web.json
Registered service: web
$ consul services deregister /server/tmp/web.json
Deregistered service: web1
```



# 3. /agent

Consul提供了2种HTTP API，`/agent/**`和`/catalog/**，/agent仅能访问本地的Agent。不建议使用/agent使用来注册于agent相关的实体

> In the context of Consul, external services run on nodes where you cannot run a local Consul agent. These nodes might be inside your infrastructure (e.g. a mainframe, virtual appliance, or unsupported platform) or outside of it (e.g. a SaaS platform).
>
> Because external services by definition don't have a local Consul agent, you can't register them with that agent or use it for health checking. Instead, you must register them directly with the catalog using the `/catalog/register` endpoint. In contrast to service registration where the object context for the endpoint is a service, the object context for the catalog endpoint is the node. In other words, using the `/catalog/register` endpoint registers an entire node, while the `/agent/service/register` endpoint registers individual services in the context of a node.
>
> https://learn.hashicorp.com/tutorials/consul/service-registration-external-services?in=consul/developer-discovery

> consul会不断地将单个节点的状态与全局目录中的一个节点重新对齐。当这种情况发生时，使用/catalog端点注册的服务将消失。
>
> While the `/catalog` endpoint seems to offer a valid alternative to the `/agent` one when it comes to register services and checks it is not recommended to use it to register agent related entities. The reason behind this is that, thanks to the anti-entropy mechanism Consul will constantly re-align the state of the single nodes with the one of the global catalog. When this happens services that were registered using the `/catalog` endpoint will disappear. The `/catalog` endpoint is the recommended way to register external services because in that case we will register the service as belonging to an `external-node`.
>
> https://learn.hashicorp.com/tutorials/consul/service-registration-health-checks?in=consul/developer-discovery

- 注册

```
curl -X PUT \
  http://127.0.0.1:8500/v1/agent/service/register \
  -H 'content-type: application/json' \
  -d '{
    "name": "web", 
    "tags": [
        "java"
    ], 
    "port": 8080, 
    "check": {
        "args": [
            "curl", 
            "-s", 
            "localhost:9000/health"
        ], 
        "interval": "10s", 
        "status": "passing"
    }
}'
```

- 注销某个服务实例

```
curl -X PUT \
  http://127.0.0.1:8500/v1/agent/service/deregister/web1 \
  -H 'content-type: application/json'
```

- 查看服务列表

```
$curl http://localhost:8500/v1/agent/services
{
    "redis1": {
        "ID": "redis1",
        "Service": "redis",
        "Tags": [],
        "Meta": {},
        "Port": 6379,
        "Address": "",
        "Weights": {
            "Passing": 1,
            "Warning": 1
        },
        "EnableTagOverride": false,
        "Datacenter": "dc1"
    },
    "web1": {
        "ID": "web1",
        "Service": "web",
        "Tags": [
            "java"
        ],
        "Meta": {},
        "Port": 8080,
        "Address": "",
        "Weights": {
            "Passing": 1,
            "Warning": 1
        },
        "EnableTagOverride": false,
        "Datacenter": "dc1"
    }
}
```

- 只查询web的服务

```
$ curl http://localhost:8500/v1/agent/services?filter=Service==web
{
    "web1": {
        "ID": "web1",
        "Service": "web",
        "Tags": [
            "java"
        ],
        "Meta": {},
        "Port": 8080,
        "Address": "",
        "Weights": {
            "Passing": 1,
            "Warning": 1
        },
        "EnableTagOverride": false,
        "Datacenter": "dc1"
    }
}

```

- 查询除web外的服务

```
# curl http://localhost:8500/v1/agent/services?filter=Service!=web
{
    "redis1": {
        "ID": "redis1",
        "Service": "redis",
        "Tags": [],
        "Meta": {},
        "Port": 6379,
        "Address": "",
        "Weights": {
            "Passing": 1,
            "Warning": 1
        },
        "EnableTagOverride": false,
        "Datacenter": "dc1"
    }
}
```

- 查看某个服务实例

```
$ curl http://localhost:8500/v1/agent/service/web1
{
    "ID": "web1",
    "Service": "web",
    "Tags": [
        "java"
    ],
    "Meta": {},
    "Port": 8080,
    "Address": "",
    "Weights": {
        "Passing": 1,
        "Warning": 1
    },
    "EnableTagOverride": false,
    "ContentHash": "bc9ded7c7f01b119",
    "Datacenter": "dc1"
}

```

- 查看服务健康状态

```
# curl http://localhost:8500/v1/agent/health/service/name/web
[
    {
        "AggregatedStatus": "passing",
        "Service": {
            "ID": "web1",
            "Service": "web",
            "Tags": [
                "java"
            ],
            "Meta": {},
            "Port": 8080,
            "Address": "",
            "Weights": {
                "Passing": 1,
                "Warning": 1
            },
            "EnableTagOverride": false,
            "Datacenter": "dc1"
        },
        "Checks": [
            {
                "Node": "VM-0-17-centos",
                "CheckID": "service:web1",
                "Name": "Service 'web' check",
                "Status": "passing",
                "Notes": "",
                "Output": "",
                "ServiceID": "web1",
                "ServiceName": "web",
                "ServiceTags": [
                    "java"
                ],
                "Type": "",
                "Definition": {
                    "Interval": "0s",
                    "Timeout": "0s",
                    "DeregisterCriticalServiceAfter": "0s",
                    "HTTP": "",
                    "Header": null,
                    "Method": "",
                    "Body": "",
                    "TLSSkipVerify": false,
                    "TCP": ""
                },
                "CreateIndex": 0,
                "ModifyIndex": 0
            }
        ]
    }
]

```

使用`?format=text`可以获得简单的文本结果

```
$ curl http://localhost:8500/v1/agent/health/service/name/web?format=text
passing
```

也可以只查某一个服务实例的健康状态

```
$ curl http://localhost:8500/v1/agent/health/service/id/web1
```

# 4. /catalog

/catlog的参数与/agent不同，一个完整的参数示例如下

```
{
  "Datacenter": "dc1",
  "ID": "40e4a748-2192-161a-0510-9bf59fe950b5",
  "Node": "foobar",
  "Address": "192.168.10.10",
  "TaggedAddresses": {
    "lan": "192.168.10.10",
    "wan": "10.0.10.10"
  },
  "NodeMeta": {
    "somekey": "somevalue"
  },
  "Service": {
    "ID": "redis1",
    "Service": "redis",
    "Tags": ["primary", "v1"],
    "Address": "127.0.0.1",
    "TaggedAddresses": {
      "lan": {
        "address": "127.0.0.1",
        "port": 8000
      },
      "wan": {
        "address": "198.18.0.1",
        "port": 80
      }
    },
    "Meta": {
      "redis_version": "4.0"
    },
    "Port": 8000,
    "Namespace": "default"
  },
  "Check": {
    "Node": "foobar",
    "CheckID": "service:redis1",
    "Name": "Redis health check",
    "Notes": "Script based health check",
    "Status": "passing",
    "ServiceID": "redis1",
    "Definition": {
      "TCP": "localhost:8888",
      "Interval": "5s",
      "Timeout": "1s",
      "DeregisterCriticalServiceAfter": "30s"
    },
    "Namespace": "default"
  },
  "SkipNodeUpdate": false
}
```

- ID：36位的UUID字符串
- Node：必填项，节点ID
- Address：必填项，注册地址
- Datacenter：DC
- NodeMeta ：KV元数据
- Service：注册的服务
  - ID：服务ID，如果没有，使用Service作为服务ID
  - Service：服务名称
- Check：注册健康检查，仅支持TCP和HTTP检查



```
$ curl -X PUT \
  http://127.0.0.1:8500/v1/catalog/register \
  -H 'content-type: application/json' \
  -d '{
  "Node": "VM-0-17-centos",
  "Address": "127.0.0.1",
  "Service": {
    "ID": "web1",
    "Service": "web",
    "Tags": ["java"],
    "Address": "127.0.0.1",
    "Port": 9000
  },
  "Check": {
    "Node": "VM-0-17-centos",
    "CheckID": "service:web1",
    "Name": "Web health check",
    "Notes": "Script based health check",
    "Status": "passing",
    "ServiceID": "web1",
    "Definition": {
      "Http": "http://localhost:9000/health",
      "Interval": "5s",
      "Timeout": "1s",
      "DeregisterCriticalServiceAfter": "30s"
    }
  }
}'
```

> 通过这种方式注册的服务，过几秒就被自动删除了
>
> consul会不断地将单个节点的状态与全局目录中的一个节点重新对齐。当这种情况发生时，使用/catalog端点注册的服务将消失。
>
> While the `/catalog` endpoint seems to offer a valid alternative to the `/agent` one when it comes to register services and checks it is not recommended to use it to register agent related entities. The reason behind this is that, thanks to the anti-entropy mechanism Consul will constantly re-align the state of the single nodes with the one of the global catalog. When this happens services that were registered using the `/catalog` endpoint will disappear. The `/catalog` endpoint is the recommended way to register external services because in that case we will register the service as belonging to an `external-node`.
> 
> https://learn.hashicorp.com/tutorials/consul/service-registration-health-checks?in=consul/developer-discovery