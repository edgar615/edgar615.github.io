---
layout: post
title: Consul - 集群搭建
date: 2019-04-14
categories:
    - Consul
comments: true
permalink: consul-cluster.html
---

server1

创建用户

```
useradd -M -s /sbin/nologin consul
mkdir -p /server/{data,config}/consul
chown -R consul.consul /server/data/consul
```

创建配置文件

```
cat > /server/config/consul/config.json << EOF
{
  "datacenter": "dc1",
  "bind_addr":"192.168.159.131",
  "log_level": "INFO",
  "node_id":"09d82408-bc4f-49e0-1111-61ef1d4842f7",
  "node_name": "server1",
  "data_dir":"/server/data/consul",
  "server": true,
  "bootstrap_expect": 3,
  "encrypt": "/fNvq869HNZ/qfB6JhR2kW+BSbS9K7a5+4AdYUTnuXs=",
  "ui":true,
  "client_addr":"0.0.0.0",
  "retry_join":["192.168.159.132:8301","192.168.159.133:8301"],
  "ports": {
     "http": 8500,
     "dns": 8600,
     "serf_lan":8301,
     "serf_wan":8302,
     "server":8300,
     "grpc":8400
  }
}
EOF
```

配置服务

```
cat > /lib/systemd/system/consul.service << EOF
[Unit]
Description="consul server"
Requires=network-online.target
After=network-online.target

[Service]
User=consul
Group=consul
ExecStart=consul agent -config-dir=/server/config/consul
ExecReload=consul reload
KillMode=process
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

启动服务

```
systemctl enable consul && systemctl start consul
```

server2和server3的配置基本相同，需要调整bind_addr、 node_id、 node_name、 retry_join几个参数



**参数**

- datacenter 此标志表示代理运行的数据中心。如果未提供，则默认为“dc1”。
   Consul拥有对多个数据中心的一流支持，但它依赖于正确的配置。同一数据中心中的节点应在同一个局域网内。
- primary_datacenter: 这指定了对ACL信息具有权威性的数据中心。必须提供它才能启用ACL。
- bootstrap_expect: Consul将等待指定数量的服务器可用，然后才会引导群集。这允许自动选择初始领导者。
- start_join: 一个字符串数组，指定是其他的consul server agent的地址。这里这样配置，会在启动时，尝试将consul-server2，consul-server3这两个节点加进来，形成一个集群。
- retry_join:  允许start_join时失败时，继续重新连接。重试的时间间隔，可以用retry_interval设置，默认是30s;重试的最大次数，可以用retry_max设置，默认是0，也就是无限次重试。关于retry_interval和retry_max，这里都是用的默认值。
- bind_addr:  内部群集通信绑定的地址。这是群集中所有其他节点都应该可以访问的IP地址。默认情况下，这是“0.0.0.0”，这意味着Consul将绑定到本地计算机上的所有地址，并将第一个可用的私有IPv4地址通告给群集的其余部分。如果有多个私有IPv4地址可用，Consul将在启动时退出并显示错误。如果指定“[::]”，Consul将通告第一个可用的公共IPv6地址。如果有多个可用的公共IPv6地址，Consul将在启动时退出并显示错误。 Consul同时使用TCP和UDP，并且两者使用相同的端口。如果您有防火墙，请务必同时允许这两种协议。
- advertise_addr: 更改我们向群集中其他节点通告的地址。默认情况下，会使用-bind参数指定的地址.
- server: 是否是server agent节点。
- connect.enabled: 是否启动Consul Connect，这里是启用的。
- node_name：节点名称。
- data_dir: agent存储状态的目录。
- enable_script_checks： 是否在此代理上启用执行脚本的健康检查。有安全漏洞，默认值就是false，这里单独提示下。
- enable_local_script_checks: 与enable_script_checks类似，但只有在本地配置文件中定义它们时才启用它们。仍然不允许在HTTP API注册中定义的脚本检查。
- log-file: 将所有Consul Agent日志消息重定向到文件。这里指定的是/opt/consul/log/目录。
- log_rotate_bytes：指定在需要轮换之前应写入日志的字节数。除非指定，否则可以写入日志文件的字节数没有限制
- log_rotate_duration：指定在需要旋转日志之前应写入日志的最长持续时间。除非另有说明，否则日志会每天轮换（24小时。单位可以是"ns", “us” (or “µs”), “ms”, “s”, “m”, “h”， 比如设置值为24h
- encrypt：用于加密Consul Gossip 协议交换的数据。在启动各个server之前，配置成同一个UUID值就行，或者你用命令行consul keygen 命令来生成也可以。
- acl.enabled: 是否启用acl.
- acl.default_policy: “allow”或“deny”;  默认为“allow”，但这将在未来的主要版本中更改。当没有匹配规则时，默认策略控制令牌的行为。在“allow”模式下，ACL是黑名单：允许任何未明确禁止的操作。在“deny”模式下，ACL是白名单：阻止任何未明确允许的操作.
- acl.enable_token_persistence: 可能值为true或者false。值为true时，API使用的令牌集合将被保存到磁盘，并且当代理重新启动时会重新加载。
- acl.tokens.master: 具有全局管理的权限，也就是最大的权限。它允许操作员使用众所周知的令牌密钥ID来引导ACL系统。需要在所有的server agent上设置同一个值，可以设置为一个随机的UUID。这个值权限最大，注意保管好。
- acl.tokens.agent: 用于客户端和服务器执行内部操作.比如catalog api的更新，反熵同步等。
- client_addr: Consul将绑定客户端接口的地址，包括HTTP和DNS服务器
- ui: 启用内置Web UI服务器和对应所需的HTTP路由
- ports.http： 更改默认的http端口。
   2.3.2. 生成两个token，让部门A，B各自管理自己的服务
   首先我们来生成部门A的policy， 意思度所有节点具有写权限（写权限包括读），并且只能写deptA开头的服务。