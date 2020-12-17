---
layout: post
title: Consul - G
date: 2019-04-01
categories:
    - Consul
comments: true
permalink: consul-intro.html
---

# 1. Consul简介

Consul提供了下列功能：

**服务发现**

Consul的Client可以提供一个服务（如API或MYSQL），然后其他Client可以通过Consul来发现一个给定服务的提供商。通过DNS或HTTP，应用程序可以很容易找到他们所依赖的服务。

**健康检查**

Consul的客户端可以提供一些健康检查，如一个关联的服务（Web Server返回200 OK），或者一个本地的节点（内存利用率低于90%）。这些信息可以被用来监控集群的健康，并且可以被服务发现组件用来远离不健康的节点

**键/值存储**

应用程序可以通过Consul提供的分层的KEY/VALUE存储数据，来实现各种不同的目的，包括动态配置、功能特征标记、协调、Leader选举等等。Consul提供了一个简单的HTTP API使它易于使用。

**多数据中心**

Consul支持开箱即用的多数据中心.这意味着Consul的用户不需要担心增加额外的抽象层来扩展更多的分区

# 2. 基本架构
Consul是一个分布式、高可用的系统。

为Consul提供服务的每个节点都运行着一个代理(Agent)，运行Agent不需要发现其他服务或者获取key/value数据。Agent负责检查节点上的服务以及节点本身的健康状况

Agent与一个或者多个Consul服务器通信.Server是保持和复制数据的地方。多个Server本身会在它们之间选择出一个领导者。输入Consul用1个Server也可以正常工作，但是仍然建议使用3或5个Server来避免失败导致的数据丢失。每个数据中心都推荐使用一组Server构成的机器

你的基础设施里那些需要发现其他服务或节点的组件可以查询任意一个Server或者Agent.Agent会自动向Server转发查询

每个数据中心都运行着一组Server,当有跨数据中心的服务发现或者配置请求时，本地的Consul服务器会将请求转发到远程数据中心并返回结果

# 3. 快速入门
## 3.1. 安装

将下载的consul_XXX_linux_amd64.zip解压,并将consul的二进制文件放在任何可以被执行的地方.
linux: ~/bin 或者 /usr/local/bin
windows 任意目录，然后通过%PATH%指定

```
wget https://releases.hashicorp.com/consul/1.9.0/consul_1.9.0_linux_amd64.zip
unzip consul_1.9.0_linux_amd64.zip -d /usr/local/bin
```

在终端中输入consul命令来检查

	$ consul
	usage: consul [--version] [--help] <command> [<args>]
	
	Available commands are:
	    agent          Runs a Consul agent
	    configtest     Validate config file
	    event          Fire a new event
	    exec           Executes a command on Consul nodes
	    force-leave    Forces a member of the cluster to enter the "left" state
	    info           Provides debugging information for operators
	    join           Tell Consul agent to join cluster
	    keygen         Generates a new encryption key
	    keyring        Manages gossip layer encryption keys
	    kv             Interact with the key-value store
	    leave          Gracefully leaves the Consul cluster and shuts down
	    lock           Execute a command holding a lock
	    maint          Controls node or service maintenance mode
	    members        Lists the members of a Consul cluster
	    monitor        Stream logs from a Consul agent
	    operator       Provides cluster-level tools for Consul operators
	    reload         Triggers the agent to reload configuration files
	    rtt            Estimates network round trip time between nodes
	    snapshot       Saves, restores and inspects snapshots of Consul server state
	    version        Prints the Consul version
	    watch          Watch for changes in Consul
	
	$ consul version
	Consul v0.7.5
	Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)

## 3.2. Agent

在Consul安装之后，必须运行一个Agent。这个Agent可以用Server模式或者Client模式运行。每个数据中心必须至少有个一个Server，不过一个集群推荐3或5个Server，单个Server在发生失败的情况下会发生数据丢失，因此不推荐使用。

所有其他的Agent以Cient模式运行。一个Client是一个非常轻量级的进程，它可以注册服务，运行健康检查，以及转发查询到服务器。Agent必须运行在集群的每个节点上。

- **启动Agent**

使用开发者模式启动一个Agent,这个模式可以非常容易快速地启动一个单节点的Consul环境。当然它并不是用于生产环境下并且它也不会持久任何状态。

	$ consul agent -dev
	
	==> Starting Consul agent...
	==> Starting Consul agent RPC...
	==> Consul agent running!
	           Version: 'v0.7.5'
	           Node ID: 'e46e29fa-6a5b-4b41-ad9a-6559df987baf'
	         Node name: 'csst-ubuntu-server'
	        Datacenter: 'dc1'
	            Server: true (bootstrap: false)
	       Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600, RPC: 8400)
	      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
	    Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false
	             Atlas: <disabled>
	
	==> Log data will now stream in as it occurs:
	
	    2017/03/17 17:17:49 [DEBUG] Using unique ID "e46e29fa-6a5b-4b41-ad9a-6559df987baf" from host as node ID
	    2017/03/17 17:17:49 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:127.0.0.1:8300 Address:127.0.0.1:8300}]
	    2017/03/17 17:17:49 [INFO] raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
	    2017/03/17 17:17:49 [INFO] serf: EventMemberJoin: csst-ubuntu-server 127.0.0.1
	    2017/03/17 17:17:49 [INFO] serf: EventMemberJoin: csst-ubuntu-server.dc1 127.0.0.1
	    2017/03/17 17:17:49 [INFO] consul: Adding LAN server csst-ubuntu-server (Addr: tcp/127.0.0.1:8300) (DC: dc1)
	    2017/03/17 17:17:49 [INFO] consul: Adding WAN server csst-ubuntu-server.dc1 (Addr: tcp/127.0.0.1:8300) (DC: dc1)
	    2017/03/17 17:17:56 [ERR] agent: failed to sync remote state: No cluster leader
	    2017/03/17 17:17:58 [WARN] raft: Heartbeat timeout from "" reached, starting election
	    2017/03/17 17:17:58 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
	    2017/03/17 17:17:58 [DEBUG] raft: Votes needed: 1
	    2017/03/17 17:17:58 [DEBUG] raft: Vote granted from 127.0.0.1:8300 in term 2. Tally: 1
	    2017/03/17 17:17:58 [INFO] raft: Election won. Tally: 1
	    2017/03/17 17:17:58 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
	    2017/03/17 17:17:58 [INFO] consul: cluster leadership acquired
	    2017/03/17 17:17:58 [INFO] consul: New leader elected: csst-ubuntu-server
	    2017/03/17 17:17:58 [DEBUG] consul: reset tombstone GC to index 3
	    2017/03/17 17:17:58 [INFO] consul: member 'csst-ubuntu-server' joined, marking health alive
	    2017/03/17 17:18:00 [INFO] agent: Synced service 'consul'
	    2017/03/17 17:18:00 [DEBUG] agent: Node info in sync

从日志信息中，可以看到我们Agent运行Server模式 `Server: true (bootstrap: false)`，

并且声明集群的Leader 

    2017/03/17 17:17:56 [ERR] agent: failed to sync remote state: No cluster leader
    2017/03/17 17:17:58 [WARN] raft: Heartbeat timeout from "" reached, starting election
    2017/03/17 17:17:58 [INFO] raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
    2017/03/17 17:17:58 [DEBUG] raft: Votes needed: 1
    2017/03/17 17:17:58 [DEBUG] raft: Vote granted from 127.0.0.1:8300 in term 2. Tally: 1
    2017/03/17 17:17:58 [INFO] raft: Election won. Tally: 1
    2017/03/17 17:17:58 [INFO] raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2017/03/17 17:17:58 [INFO] consul: cluster leadership acquired
    2017/03/17 17:17:58 [INFO] consul: New leader elected: csst-ubuntu-server

另外，本地的成员已经被标记为一个健康的集群成员

	    2017/03/17 17:17:58 [INFO] consul: member 'csst-ubuntu-server' joined, marking health alive

- **单机启动**

```
consul agent -server -bootstrap-expect=1 -data-dir=/data/consul-data -ui -bind=192.168.1.221 -client=0.0.0.0
```

- **集群成员**

在终端上输入`consul members`，能看到Consul集群所有的节点

	$ consul members
	
	Node                Address         Status  Type    Build  Protocol  DC
	csst-ubuntu-server  127.0.0.1:8301  alive   server  0.7.5  2         dc1

输出显示了节点的名称、运行地址、健康状态、在集群中的角色、版本信息等等。一些额外元数据可以通过 `-detailed`选项来查看

该命令输出显示你自己的节点，运行的地址，它的健康状态，它在集群中的角色，以及一些版本信息。另外元数据可以通过 -detailed 选项来查看。

	$ consul members -detailed
	
	Node                Address         Status  Tags
	csst-ubuntu-server  127.0.0.1:8301  alive   build=0.7.5:'21f2d5a,dc=dc1,id=e46e29fa-6a5b-4b41-ad9a-6559df987baf,port=8300,role=consul,vsn=2,vsn_max=3,vsn_min=2

members 命令选项的输出是基于gossip协议，并且其内容保证最终一致性。也就是说，在任何时候你在本地代理看到的内容也许与当前服务器中的状态并不是绝对一致的。如果需要强一致性的状态信息，使用HTTP API向Consul服务器发送请求`http://127.0.0.1:8500/v1/catalog/nodes`	
	$ curl http://127.0.0.1:8500/v1/catalog/nodes
	[
	    {
	        "ID": "e46e29fa-6a5b-4b41-ad9a-6559df987baf",
	        "Node": "csst-ubuntu-server",
	        "Address": "127.0.0.1",
	        "TaggedAddresses": {
	            "lan": "127.0.0.1",
	            "wan": "127.0.0.1"
	        },
	        "Meta": {},
	        "CreateIndex": 4,
	        "ModifyIndex": 5
	    }
	]

另外除了HTTP API，DNS接口也常被用来查询节点信息。但你必须确保你的DNS能够找到Consul代理的DNS服务器，Consul代理的DNS服务器默认运行在8600端口。

	$ dig @127.0.0.1 -p 8600 csst-ubuntu-server.node.consul
	
	; <<>> DiG 9.9.5-3ubuntu0.7-Ubuntu <<>> @127.0.0.1 -p 8600 csst-ubuntu-server.node.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17078
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
	;; WARNING: recursion requested but not available
	
	;; QUESTION SECTION:
	;csst-ubuntu-server.node.consul.	IN	A
	
	;; ANSWER SECTION:
	csst-ubuntu-server.node.consul.	0 IN	A	127.0.0.1
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Fri Mar 17 17:44:30 CST 2017
	;; MSG SIZE  rcvd: 64

- **关闭**

你能够使用 Ctrl-C （中断信号）来优雅地停止代理。停止代理后，你可以看到它脱离集群并且关闭的信息。

为了优雅地离开集群，Consul会通知其他的集群成员自己已经脱离了。如果你强制杀死代理的进程，那么其他的集群成员需要侦测节点是否失效。当一个成员离开，它的服务以及（checks）将从目录中移除。当一个成员失效，它的健康会简单地标记为critical，但它并不会被从目录中移除。Consul将自动尝试重新连接到失效的节点，并允许它在某些网络状况下恢复。

## 3.3. 服务

- **定义服务**

一个服务可以通过提供一个服务定义或者调用HTTP API来注册.

服务定义是最通用的注册服务的方法。

首先，为consul的配置创建一个目录，Consul加载这个配置目录下的所有配件文字，通常在Unix系统中惯例是建立以名为 /etc/consul.d 的目录（ .d 后缀暗示这个目录包含了一些配置文件的集合）。

	$ sudo mkdir /etc/consul.d

下一步，我们创建一个服务定义的配置文件.

	$ echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}' | sudo tee /etc/consul.d/web.json

上面的定义文件定义了一个名称为web，运行在80端口的服务

接着，我们重启Agent，并指定配置文件目录

	$ consul agent -dev -config-dir=/etc/consul.d
		...
	    2017/03/20 11:23:20 [INFO] agent: Synced service 'consul'
	    2017/03/20 11:23:20 [INFO] agent: Synced service 'web'
	    2017/03/20 11:23:20 [DEBUG] agent: Node info in sync
		...


在输出中的`Synced service 'web'`，意味着代理已经从配置文件中装载了该服务定义，并且已经成功注册该服务到服务目录中。如果你想注册多个服务，你可以在Consul配置目录中创建多个服务定义文件。

一旦Agent启动，并且服务已经注册，我们可以通过DNS或HTTP API来查询服务

- **DNS API**

对于DNS API，服务的DNS名称是`NAME.service.consul`，默认所有的DSN名称都是在consul的命名空间下，当然这个也可以配置.`service`的子域名告诉Consul我们我们正在查询服务，`Name`是需要查询的服务名称

对于我们前面注册的web服务，一个合格的域名称是`web.service.consul`

	$ dig @127.0.0.1 -p 8600 web.service.consul
	
	; <<>> DiG 9.9.5-3ubuntu0.7-Ubuntu <<>> @127.0.0.1 -p 8600 web.service.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47018
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
	;; WARNING: recursion requested but not available
	
	;; QUESTION SECTION:
	;web.service.consul.		IN	A
	
	;; ANSWER SECTION:
	web.service.consul.	0	IN	A	127.0.0.1
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Mon Mar 20 11:32:54 CST 2017
	;; MSG SIZE  rcvd: 52

上面的查询返回了一个带有节点IP地址的A记录，A记录只能包含IP地址

你也可以使用DNS API来获取完整的地址/端口的 SRV 记录：

	$ dig @127.0.0.1 -p 8600 web.service.consul SRV
	
	; <<>> DiG 9.9.5-3ubuntu0.7-Ubuntu <<>> @127.0.0.1 -p 8600 web.service.consul SRV
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48601
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
	;; WARNING: recursion requested but not available
	
	;; QUESTION SECTION:
	;web.service.consul.		IN	SRV
	
	;; ANSWER SECTION:
	web.service.consul.	0	IN	SRV	1 1 80 csst-ubuntu-server.node.dc1.consul.
	
	;; ADDITIONAL SECTION:
	csst-ubuntu-server.node.dc1.consul. 0 IN A	127.0.0.1
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Mon Mar 20 11:36:43 CST 2017
	;; MSG SIZE  rcvd: 100

SRV记录显示了web服务允许在节点csst-ubuntu-server.node.dc1.consul.的80端口上.额外的步伐和A记录返回的内容一样

最后，我们可以使用DNS API通过tags来过滤服务，基于tag查询服务的格式是`TAG.NAME.service.consul`，例如我们向Consul查询所有带`rails`标记的web服务:

	$ dig @127.0.0.1 -p 8600 rails.web.service.consul
	
	; <<>> DiG 9.9.5-3ubuntu0.7-Ubuntu <<>> @127.0.0.1 -p 8600 rails.web.service.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17492
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
	;; WARNING: recursion requested but not available
	
	;; QUESTION SECTION:
	;rails.web.service.consul.	IN	A
	
	;; ANSWER SECTION:
	rails.web.service.consul. 0	IN	A	127.0.0.1
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Mon Mar 20 11:41:44 CST 2017
	;; MSG SIZE  rcvd: 58

如果查询服务未找到对应的服务，会返回AUTHORITY SECTION

	$ dig @127.0.0.1 -p 8600 db.service.consul
	
	; <<>> DiG 9.9.5-3ubuntu0.7-Ubuntu <<>> @127.0.0.1 -p 8600 db.service.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 43380
	;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 0
	;; WARNING: recursion requested but not available
	
	;; QUESTION SECTION:
	;db.service.consul.		IN	A
	
	;; AUTHORITY SECTION:
	consul.			0	IN	SOA	ns.consul. postmaster.consul. 1489981368 3600 600 86400 0
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Mon Mar 20 11:42:48 CST 2017
	;; MSG SIZE  rcvd: 85

- **HTTP API**

HTTP API的格式如下：

	$ curl http://127.0.0.1:8500/v1/catalog/service/web
	[
	    {
	        "ID": "e46e29fa-6a5b-4b41-ad9a-6559df987baf",
	        "Node": "csst-ubuntu-server",
	        "Address": "127.0.0.1",
	        "TaggedAddresses": {
	            "lan": "127.0.0.1",
	            "wan": "127.0.0.1"
	        },
	        "NodeMeta": {},
	        "ServiceID": "web",
	        "ServiceName": "web",
	        "ServiceTags": [
	            "rails"
	        ],
	        "ServiceAddress": "",
	        "ServicePort": 80,
	        "ServiceEnableTagOverride": false,
	        "CreateIndex": 6,
	        "ModifyIndex": 6
	    }
	]

catalog API返回了指定节点以及指定的服务信息。就像我们马上要看到了健康检测，通常我们的查询只是查询那些健康的实例，这些实例都是通过了健康检测的。这也是DNS在底层做的事情。

	$ curl http://127.0.0.1:8500/v1/catalog/service/web?passing
	[
	    {
	        "ID": "e46e29fa-6a5b-4b41-ad9a-6559df987baf",
	        "Node": "csst-ubuntu-server",
	        "Address": "127.0.0.1",
	        "TaggedAddresses": {
	            "lan": "127.0.0.1",
	            "wan": "127.0.0.1"
	        },
	        "NodeMeta": {},
	        "ServiceID": "web",
	        "ServiceName": "web",
	        "ServiceTags": [
	            "rails"
	        ],
	        "ServiceAddress": "",
	        "ServicePort": 80,
	        "ServiceEnableTagOverride": false,
	        "CreateIndex": 6,
	        "ModifyIndex": 6
	    }
	]

**更新服务**

当配置文件修改后服务定义可以被更新，需要发送 SIGHUP 信号给代理。这可以让代理更新服务而无需停止代理或者让服务查询时服务不可用。

可以选择HTTP API来动态地增加，删除，以及更改服务。

# 4. 集群
集群中的每个节点都必须有一个唯一的名称.Consul默认使用主机名称`hostname`，但是我们也可以手动的通过命令行参数`-node`来修改。

我们可以指定一个绑定地址：Consul将监听这个地址，并且这个地址必须能够被集群中的其他节点访问到。虽然指定绑定地址并不是必需的，但是最好还是提供一个绑定地址。Consul默认会监听系统中所有的IPV4忘了接口，但是如果发现多个私有IP，Consul将会无法启动。因为生产服务器上通常会提供多个网络接口，所以指定一个绑定地址可以确保不会将Consul绑定到一个错误的网络接口上

-server参数表示节点运行在server模式

-bootstrap-expec参数用来暗示Consul服务器：还有会有多少个其他节点加入集群。这个标志的目的是延迟复制日志的引导直到预期的节点成功加入。

-config-dir参数表示配件文件的目录

第一台机器10.11.0.31

	$ consul agent -server -bootstrap-expect=1 -data-dir=/tmp/consul -node=agent-on -bind=10.11.0.31 -config-dir=/etc/consul.d

第二台机器10.4.7.14

	$ consul agent -data-dir=/tmp/consul -node=agent-two -bind=10.4.7.14 -config-dir=/etc/consul.d

现在我们已经有两个Agent在运行：一个Server和一个Client。但是这两个Agent现在还对彼此没有任何感知，它们都为两个单节点的集群。

在两台机器上查看节点，仅包含一个节点

	$ consul members
	Node      Address          Status  Type    Build  Protocol  DC
	agent-on  10.11.0.31:8301  alive   server  0.7.5  2         dc1
	
	$ consul members
	Node       Address         Status  Type    Build  Protocol  DC
	agent-two  10.4.7.14:8301  alive   client  0.7.5  2         dc1

10.4.7.14的日志

    2017/03/20 14:24:50 [INFO] serf: EventMemberJoin: agent-two 10.4.7.14
    2017/03/20 14:24:50 [WARN] manager: No servers available
    2017/03/20 14:24:50 [ERR] agent: failed to sync remote state: No known Consul servers

## 4.1. 加入集群
通过命令告知10.11.0.31加入第二个节点10.4.7.14

	$ consul join 10.4.7.14
	Successfully joined cluster by contacting 1 nodes.

10.11.0.31的日志

    2017/03/20 14:24:23 [INFO] agent.rpc: Accepted client: 127.0.0.1:41612
    2017/03/20 14:25:53 [INFO] agent.rpc: Accepted client: 127.0.0.1:41701
    2017/03/20 14:29:51 [INFO] agent.rpc: Accepted client: 127.0.0.1:41934
    2017/03/20 14:29:51 [INFO] agent: (LAN) joining: [10.4.7.14]
    2017/03/20 14:29:52 [INFO] serf: EventMemberJoin: agent-two 10.4.7.14
    2017/03/20 14:29:52 [INFO] agent: (LAN) joined: 1 Err: <nil>
    2017/03/20 14:29:52 [INFO] consul: member 'agent-two' joined, marking health alive

10.4.7.14的日志

    2017/03/20 14:28:50 [ERR] agent: failed to sync remote state: No known Consul servers
    2017/03/20 14:29:03 [INFO] serf: EventMemberJoin: agent-on 10.11.0.31
    2017/03/20 14:29:03 [INFO] consul: adding server agent-on (Addr: tcp/10.11.0.31:8300) (DC: dc1)
    2017/03/20 14:29:03 [INFO] consul: New leader elected: agent-on
    2017/03/20 14:29:03 [INFO] agent: Synced node info

再次运行 `consul members`,两个Agent都感知到另一个节点的存在

	$ consul members
	Node       Address          Status  Type    Build  Protocol  DC
	agent-on   10.11.0.31:8301  alive   server  0.7.5  2         dc1
	agent-two  10.4.7.14:8301   alive   client  0.7.5  2         dc

记住：**为了加入一个集群，一个Consul代理只需要知道一个现有的成员。在加入指定的集群后，各个代理会互相传播完整的成员信息。**

## 4.2. 启动时自动加入
理想情况下，每当有一个新节点加入到数据中心时，它应该自动加入到Consul集群，而无需人为干预。

你可以在启动的时候使用-join参数指定其他已知的Agent或者设置start_join参数

	$ consul agent -data-dir=/tmp/consul -node=agent-two -bind=10.4.7.14 -config-dir=/etc/consul.d -join=10.11.0.31

## 4.3. 查询节点
和查询服务一样，可以通过DNS或HTTP API来查询节点

对于DNS API，查询名称的结构是`NAME.node.consul`或者`NAME.node.DATACENTER.consul`，如果数据中心被移除，Consul仅查询本地数据中心

例如在agent-one上，我们可以看出agent-two的节点信息

	$ dig @127.0.0.1 -p 8600 agent-two.node.consul
	
	; <<>> DiG 9.9.5-3ubuntu0.7-Ubuntu <<>> @127.0.0.1 -p 8600 agent-two.node.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 33227
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
	;; WARNING: recursion requested but not available
	
	;; QUESTION SECTION:
	;agent-two.node.consul.		IN	A
	
	;; ANSWER SECTION:
	agent-two.node.consul.	0	IN	A	10.4.7.14
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Mon Mar 20 14:49:05 CST 2017
	;; MSG SIZE  rcvd: 55

这种查找节点的能力对于系统管理任务而言是非常有用的。例如知道了节点的地址，我们可以使用ssh登录到该节点并且可以非常容易地使得该节点成为Consul集群中的一部分并且查询它。

## 4.4. 离开集群

为了离开指定的集群，你可以优雅地退出一个Agent（使用 Ctrl-C）或者强制杀死Agent进程。优雅地离开可以使得节点转换成离开状态；其它情况下，其它的节点检测这个节点将失败。

**优雅的离开**

	$ consul members
	Node       Address          Status  Type    Build  Protocol  DC
	agent-on   10.11.0.31:8301  alive   server  0.7.5  2         dc1
	agent-two  10.4.7.14:8301   left    client  0.7.5  2         dc1

**杀死进程**	

	$ consul members
	Node       Address          Status  Type    Build  Protocol  DC
	agent-on   10.11.0.31:8301  alive   server  0.7.5  2         dc1
	agent-two  10.4.7.14:8301   failed  client  0.7.5  2         dc1

# 5. 健康检查

可以通过一个定义检查或者通过调研HTTP API来注册一个监控检查

## 5.1. 定义检查
就像服务一样，使用定义是一个最为常用的方法来设置健康检查

	$ echo '{"check": {"name": "ping", "script": "ping -c1 baidu.com >/dev/null", "interval": "30s"}}' | sudo tee /etc/consul.d/ping.json
	
	$ echo '{"service": {"name": "web", "tags": ["rails"], "port": 80, "check": {"script": "curl localhost >/dev/null 2>&1", "interval": "10s"}}}'  | sudo tee /etc/consul.d/web.json


第一个定义增加了一个主机级别的检测，名为"ping"。该检测每30秒间隔运行一次，调用命令 ping -c1 baidu.com。在一个基于脚本的健康检测中，该检测使用启动Consul进程的用户来启动该检测。如果检测命令返回一个非0的返回码，那么该节点将被标记为不健康。这就是任何基于 脚本 的健康检测的契约。

第二个命令修改名为 web 的服务，增加了一个检测，该检测每10秒用curl发送一个请求来验证该web服务是否可用。就像基于主机的健康检测，如果脚本返回一个非0的返回码，那该服务将被标记为不健康。

重启第二个Agent，或者使用`consul reload`, 或者发送一个`SIGHUP`

	$ consul reload
	Configuration reload triggered

查看日志

	==> Caught signal: hangup
	==> Reloading configuration...
	    2017/03/20 15:20:00 [INFO] agent: Synced service 'web'
	    2017/03/20 15:20:00 [INFO] agent: Synced check 'ping'
	    2017/03/20 15:20:05 [WARN] agent: Check 'service:web' is now critical
	    2017/03/20 15:20:15 [WARN] agent: Check 'service:web' is now critical

前面的几行指出该Agent已经同步了新的定义。后面的几行指出了被检测的 web 服务被标记为危险。这是因为我们还没有实际运行一个web服务器，所以这个curl测试标记为失败了。

## 5.2. 检查健康状态
现在我们已经增加了一些健康检查，我们可以使用HTTP API来审查它们。首先，我们可以使用命令寻找任何失败的检测，这个命令可以在任何节点上运行

	$ curl http://127.0.0.1:8500/v1/health/state/critical
	[{"Node":"agent-two","CheckID":"service:web","Name":"Service 'web' check","Status":"critical","Notes":"","Output":"","ServiceID":"web","ServiceName":"web","CreateIndex":373,"ModifyIndex":378}]

另外，通过DNS查询web服务，Consul不会放任何结果，因为这个服务是不健康的

	$ dig @127.0.0.1 -p 8600 web.service.consul
	
	; <<>> DiG 9.9.5-3ubuntu0.7-Ubuntu <<>> @127.0.0.1 -p 8600 web.service.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 56242
	;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 0
	;; WARNING: recursion requested but not available
	
	;; QUESTION SECTION:
	;web.service.consul.		IN	A
	
	;; AUTHORITY SECTION:
	consul.			0	IN	SOA	ns.consul. postmaster.consul. 1489994995 3600 600 86400 0
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Mon Mar 20 15:29:55 CST 2017
	;; MSG SIZE  rcvd: 86

# 6. 键/值存储

Consul提供了非常容易使用的键/值对存储。它能被用于存储动态配置信息，帮助服务协作，建构leader选举机制，以及开发者可以想到的建构任何其它的东西。

有两种与Consul的KV存储交互的方式：HTTP API或 Consul KV命令行

**命令行**
查询`redis/config/minconns`的值

	$ consul kv get redis/config/minconns
	Error! No key exists at: redis/config/minconns

插入数据

	$ consul kv put redis/config/minconns 1
	Success! Data written to: redis/config/minconns
	$ consul kv put redis/config/maxconns 25
	Success! Data written to: redis/config/maxconns
	$ consul kv put -flags=42 redis/config/users/admin abcd1234
	Success! Data written to: redis/config/users/admin

再次查询

	$ consul kv get redis/config/minconns
	1

Consul同时维护了K/V对应的元数据，可以通过`-detailed`获取

	$ consul kv get -detailed redis/config/minconns
	CreateIndex      502
	Flags            0
	Key              redis/config/minconns
	LockIndex        0
	ModifyIndex      514
	Session          -
	Value            1

对于`redis/config/users/admin`，我们设置了一个42的flag属性，所有键都支持设置64位整数标记值。这个不是由Consul内部使用，但它可以用于客户端添加任何有意义的元数据

可以使用`recurse`选项查询所有的K/V，结果将按字典顺序返回

	$ consul kv get -recurse
	redis/config/maxconns:25
	redis/config/minconns:1
	redis/config/users/admin:abcd1234

`delete`用于删除K/V
	

	$ consul kv delete redis/config/minconns
	Success! Deleted key: redis/config/minconns

使用`recurse`选项可以递归删除某个前缀

	$ consul kv delete -recurse redis
	Success! Deleted keys with prefix: redis 

`put`用于更新

	$ consul kv put foo bar
	Success! Data written to: foo
	$ consul kv get foo
	bar
	$ consul kv put foo zip
	Success! Data written to: foo
	$ consul kv get foo
	zip

Consul提供了通过CAS操作来对K/V的原子更新，使用`-cas`选项

	$ consul kv get -detailed foo
	CreateIndex      571
	Flags            0
	Key              foo
	LockIndex        0
	ModifyIndex      573
	Session          -
	Value            zip
	$ consul kv put -cas -modify-index=573 foo bar
	Success! Data written to: foo
	$ consul kv put -cas -modify-index=573 foo bar
	Error! Did not write to foo: CAS failed

# 7. WEB UI

	consul agent -client 0.0.0.0 -ui

http://<IP地址>:8500/ui

