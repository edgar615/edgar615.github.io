---
layout: post
title: Consul - 健康检查
date: 2019-04-03
categories:
    - Consul
comments: true
permalink: consul-health-check.html
---

Consul提供了下面几种健康检查

- Script + Interval： 使用外部程序（脚本）检查，可能会产生一些输出。输出限制为4KB，超时时间30秒。脚本的返回值0，状态为passing；返回值1，状态为warning；其他返回值，状态为failing
  - `enable_local_script_checks`: 支持从本地配置文件设置健康检查，不支持通过HTTP API的方式设置
  - `enable_script_checks`: 支持设置健康检查
- HTTP + Interval： 定时向指定的HTTP地址发送GET请求，如果返回2XX，状态被认为passing；429 Too ManyRequests，状态被认为warning；其他被认为failure。默认超时时间10秒
- TCP + Interval： 定时向指定的IP:PORT建立TCP连接，如果连接建立成功，状态为success，否则状态为critical。默认超时时间10秒
- Time to Live (TTL) ：定时检查Consul保存的给定TTL的最后一个一致状态。这个状态由外部程序通过HTTP端点更新。状态有：pass、warn、fail、update四种
- Docker + Interval ：通过Docker Exec接口访问docker容器进行健康检查，Consul agent 需要能够访问Docker的HTTP API或者unix socket。Consul 使用 `$DOCKER_HOST` 来决定使用哪种Docker API
- gRPC + Interval ：通过gRPC的健康检查协议进行健康检查
- Alias -

# 1. 健康检查定义

- **Script**

```
{
  "check": {
    "id": "mem-util",
    "name": "Memory utilization",
    "args": ["/usr/local/bin/check_mem.py", "-limit", "256MB"],
    "interval": "10s",
    "timeout": "1s"
  }
}
```

- **HTTP**

```
{
  "check": {
    "id": "api",
    "name": "HTTP API on port 5000",
    "http": "https://localhost:5000/health",
    "tls_skip_verify": false,
    "method": "POST",
    "header": {"Content-Type": ["application/json"]},
    "body": "{\"method\":\"health\"}",
    "interval": "10s",
    "timeout": "1s"
  }
}
```

- **TCP**

```
{
  "check": {
    "id": "ssh",
    "name": "SSH TCP on port 22",
    "tcp": "localhost:22",
    "interval": "10s",
    "timeout": "1s"
  }
}
```

- **TTL**

```
{
  "check": {
    "id": "web-app",
    "name": "Web App Status",
    "notes": "Web app does a curl internally every 10 seconds",
    "ttl": "30s"
  }
}
```

- **Docker**

```
{
  "check": {
    "id": "mem-util",
    "name": "Memory utilization",
    "docker_container_id": "f972c95ebf0e",
    "shell": "/bin/bash",
    "args": ["/usr/local/bin/check_mem.py"],
    "interval": "10s"
  }
}
```

- **gRPC**

```
{
  "check": {
    "id": "mem-util",
    "name": "Service health status",
    "grpc": "127.0.0.1:12345",
    "grpc_use_tls": true,
    "interval": "10s"
  }
}
```

也可以仅检查某个service

```
{
  "check": {
    "id": "mem-util",
    "name": "Service health status",
    "grpc": "127.0.0.1:12345/my_service",
    "grpc_use_tls": true,
    "interval": "10s"
  }
}
```

>  可以参考https://edgar615.github.io/grpc-healthcheck.html

- **alias** 

```
{
  "check": {
    "id": "web-alias",
    "alias_service": "web"
  }
}
```

- **其他参数**

`"DeregisterCriticalServiceAfter": "30s"`如果检查到critical已经超过了设置的时间，会自动将关联的服务注销

`"status": "passing"`用来指定状态的初始值

`"ServiceId": "web-app"`：指定关联的服务实例

`"success_before_passing": 3, "failures_before_critical": 3`只有在指定数量的连续检查返回passing/critical后，才可以将检查配置为passing/critical。在达到配置的阈值之前，状态不会转换状态，在HTTP, TCP, gRPC, Docker & Monitor checks下有效，默认0

# 2. HTTP注册

- **查看检查列表**

```
$ curl http://127.0.0.1:8500/v1/agent/checks
{
    "service:web1": {
        "Node": "VM-0-17-centos",
        "CheckID": "service:web1",
        "Name": "Service 'web' check",
        "Status": "passing",
        "Notes": "",
        "Output": "HTTP GET http://localhost:9000/health: 200 OK Output: ",
        "ServiceID": "web1",
        "ServiceName": "web",
        "ServiceTags": [
            "java"
        ],
        "Type": "http",
        "Definition": {},
        "CreateIndex": 0,
        "ModifyIndex": 0
    }
}

```

根据服务ID和服务名过滤

```
$ curl http://127.0.0.1:8500/v1/agent/checks?filter=ServiceID==web1
{
    "service:web1": {
        "Node": "VM-0-17-centos",
        "CheckID": "service:web1",
        "Name": "Service 'web' check",
        "Status": "critical",
        "Notes": "",
        "Output": "Get \"http://localhost:9000/health\": dial tcp [::1]:9000: connect: connection refused",
        "ServiceID": "web1",
        "ServiceName": "web",
        "ServiceTags": [
            "java"
        ],
        "Type": "http",
        "Definition": {},
        "CreateIndex": 0,
        "ModifyIndex": 0
    }
}

$ curl http://127.0.0.1:8500/v1/agent/checks?filter=ServiceName==web
```

- **注册**

```
$ curl -X PUT \
  http://127.0.0.1:8500/v1/agent/check/register \
  -H 'content-type: application/json' \
  -d '{
    "ID": "service:web1", 
    "Name": "Web health check", 
    "Notes": "Script based health check", 
    "Status": "passing", 
    "DeregisterCriticalServiceAfter": "30s", 
    "ServiceID": "web1", 
    "http": "http://localhost:9000/health", 
    "interval": "5s", 
    "Timeout": "1s"
}'
```

更多的参数示例

```
{
  "ID": "mem",
  "Name": "Memory utilization",
  "Notes": "Ensure we don't oversubscribe memory",
  "DeregisterCriticalServiceAfter": "90m",
  "Args": ["/usr/local/bin/check_mem.py"],
  "DockerContainerID": "f972c95ebf0e",
  "Shell": "/bin/bash",
  "HTTP": "https://example.com",
  "Method": "POST",
  "Header": { "Content-Type": ["application/json"] },
  "Body": "{\"check\":\"mem\"}",
  "TCP": "example.com:22",
  "Interval": "10s",
  "Timeout": "5s",
  "TLSSkipVerify": true
}
```

- **注销**

```
$ curl -X PUT \
>   http://127.0.0.1:8500/v1/agent/check/deregister/service:web1
```

# 3. TTL

先注册一个TTL的健康检查

```
curl -X PUT \
  http://127.0.0.1:8500/v1/agent/check/register \
  -H 'content-type: application/json' \
  -d '{
    "ID": "service:web1", 
    "Name": "Web health check", 
    "Notes": "Script based health check", 
    "Status": "passing", 
    "DeregisterCriticalServiceAfter": "30s", 
    "ServiceID": "web1", 
    "ttl": "10s"
}'
```

- 更新pass状态

```
curl -X PUT http://127.0.0.1:8500/v1/agent/check/pass/service:web1
```

- 更新warn状态

```
curl -X PUT http://127.0.0.1:8500/v1/agent/check/warn/service:web1
```

- 更新失败状态

```
curl -X PUT http://127.0.0.1:8500/v1/agent/check/fail/service:web1
```

- 更新状态

```
curl -X PUT \
	--data '{
	  "Status": "passing",
	  "Output": "curl reported a failure:\n\n..."
	}' \
	http://127.0.0.1:8500/v1/agent/check/update/service:web1
```

# 4. 健康相关的查询接口

- 查看节点的健康检查

```
$ curl http://127.0.0.1:8500/v1/health/node/VM-0-17-centos
[
    {
        "Node": "VM-0-17-centos",
        "CheckID": "serfHealth",
        "Name": "Serf Health Status",
        "Status": "passing",
        "Notes": "",
        "Output": "Agent alive and reachable",
        "ServiceID": "",
        "ServiceName": "",
        "ServiceTags": [],
        "Type": "",
        "Definition": {},
        "CreateIndex": 11,
        "ModifyIndex": 11
    },
    {
        "Node": "VM-0-17-centos",
        "CheckID": "service:web1",
        "Name": "Service 'web' check",
        "Status": "passing",
        "Notes": "",
        "Output": "HTTP GET http://localhost:9000/health: 200 OK Output: ",
        "ServiceID": "web1",
        "ServiceName": "web",
        "ServiceTags": [
            "java"
        ],
        "Type": "http",
        "Definition": {},
        "CreateIndex": 40,
        "ModifyIndex": 48
    }
]

```

- 查看服务的检查检查结果

```
$ curl http://127.0.0.1:8500/v1/health/checks/web
[
    {
        "Node": "VM-0-17-centos",
        "CheckID": "service:web1",
        "Name": "Service 'web' check",
        "Status": "passing",
        "Notes": "",
        "Output": "HTTP GET http://localhost:9000/health: 200 OK Output: ",
        "ServiceID": "web1",
        "ServiceName": "web",
        "ServiceTags": [
            "java"
        ],
        "Type": "http",
        "Definition": {},
        "CreateIndex": 40,
        "ModifyIndex": 48
    }
]

```

- 查看服务列表

```
$ curl http://127.0.0.1:8500/v1/health/service/web
[
    {
        "Node": {
            "ID": "03698478-2fe7-5bb0-0c4c-be84c2a90b9c",
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
        },
        "Service": {
            "ID": "web1",
            "Service": "web",
            "Tags": [
                "java"
            ],
            "Address": "",
            "Meta": null,
            "Port": 9000,
            "Weights": {
                "Passing": 1,
                "Warning": 1
            },
            "EnableTagOverride": false,
            "Proxy": {
                "MeshGateway": {},
                "Expose": {}
            },
            "Connect": {},
            "CreateIndex": 38,
            "ModifyIndex": 38
        },
        "Checks": [
            {
                "Node": "VM-0-17-centos",
                "CheckID": "serfHealth",
                "Name": "Serf Health Status",
                "Status": "passing",
                "Notes": "",
                "Output": "Agent alive and reachable",
                "ServiceID": "",
                "ServiceName": "",
                "ServiceTags": [],
                "Type": "",
                "Definition": {},
                "CreateIndex": 11,
                "ModifyIndex": 11
            },
            {
                "Node": "VM-0-17-centos",
                "CheckID": "service:web1",
                "Name": "Service 'web' check",
                "Status": "passing",
                "Notes": "",
                "Output": "HTTP GET http://localhost:9000/health: 200 OK Output: ",
                "ServiceID": "web1",
                "ServiceName": "web",
                "ServiceTags": [
                    "java"
                ],
                "Type": "http",
                "Definition": {},
                "CreateIndex": 40,
                "ModifyIndex": 48
            }
        ]
    }
]

```

根据状态过滤

```
$ curl http://127.0.0.1:8500/v1/health/service/web?filter=Checks.Status==passing
```

- 根据状态查询

```
# curl http://127.0.0.1:8500/v1/health/state/any
[
    {
        "Node": "VM-0-17-centos",
        "CheckID": "serfHealth",
        "Name": "Serf Health Status",
        "Status": "passing",
        "Notes": "",
        "Output": "Agent alive and reachable",
        "ServiceID": "",
        "ServiceName": "",
        "ServiceTags": [],
        "Type": "",
        "Definition": {},
        "CreateIndex": 11,
        "ModifyIndex": 11
    },
    {
        "Node": "VM-0-17-centos",
        "CheckID": "service:web1",
        "Name": "Service 'web' check",
        "Status": "passing",
        "Notes": "",
        "Output": "HTTP GET http://localhost:9000/health: 200 OK Output: ",
        "ServiceID": "web1",
        "ServiceName": "web",
        "ServiceTags": [
            "java"
        ],
        "Type": "http",
        "Definition": {},
        "CreateIndex": 40,
        "ModifyIndex": 48
    }
]

$ curl http://127.0.0.1:8500/v1/health/state/fail
[]

```

# 5. 健康检查能不能支持故障转移？

Consul的数据同步也是强一致性的，服务的注册信息会在Server节点之间同步，相比ZK、etcd，服务的信息还是持久化保存的，即使服务部署不可用了，仍旧可以查询到这个服务部署。但是业务服务的可用状态是由注册到的Agent来维护的，Agent如果不能正常工作了，则无法确定服务的真实状态，并且Consul是相当稳定了，Agent挂掉的情况下大概率服务器的状态也可能是不好的，此时屏蔽掉此节点上的服务是合理的。Consul也确实是这样设计的，DNS接口会自动屏蔽挂掉节点上的服务，HTTP API也认为挂掉节点上的服务不是passing的。

鉴于Consul健康检查的这种机制，同时避免单点故障，**所有的业务服务应该部署多份，并注册到不同的Consul节点**。

上边提到健康检查是由服务注册到的Agent来处理的，那么如果这个Agent挂掉了，会不会有别的Agent来接管健康检查呢？答案是**否定的**。

从问题产生的原因来看，在应用于生产环境之前，肯定需要对各种场景进行测试，没有问题才会上线，所以显而易见的问题可以屏蔽掉；如果是新版本Consul的BUG导致的，此时需要降级；如果这个BUG是偶发的，那么只需要将Consul重新拉起来就可以了，这样比较简单；如果是硬件、网络或者操作系统故障，那么节点上服务的可用性也很难保障，不需要别的Agent接管健康检查。

从实现上看，选择哪个节点是个问题，这需要实时或准实时同步各个节点的负载状态，而且由于业务服务运行状态多变，即使当时选择出了负载比较轻松的节点，无法保证某个时段任务又变得繁重，可能造成新的更大范围的崩溃。如果原来的节点还要启动起来，那么接管的健康检查是否还要撤销，如果要，需要记录服务们最初注册的节点，然后有一个监听机制来触发，如果不要，通过服务发现就会获取到很多冗余的信息，并且随着时间推移，这种数据会越来越多，系统变的无序。

从实际应用看，节点上的服务可能既要被发现，又要发现别的服务，如果节点挂掉了，仅提供被发现的功能实际上服务还是不可用的。当然发现别的服务也可以不使用本机节点，可以通过访问一个Nginx实现的若干Consul节点的负载均衡来实现，这无疑又引入了新的技术栈。

如果不是上边提到的问题，或者你可以通过一些方式解决这些问题，健康检查接管的实现也必然是比较复杂的，因为分布式系统的状态同步是比较复杂的。同时不要忘了服务部署了多份，挂掉一个不应该影响系统的快速恢复，所以没必要去做这个接管。

# 6. 参考资料

http://blog.didispace.com/consul-service-discovery-exp/