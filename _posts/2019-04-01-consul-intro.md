---
layout: post
title: Consul - 介绍
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

下面这张图来源于Consul官网，很好的解释了Consul的工作原理

![](/assets/images/posts/consul/consul-1.png)

首先Consul支持多数据中心，在上图中有两个DataCenter，他们通过Internet互联，同时请注意为了提高通信效率，只有Server节点才加入跨数据中心的通信。

图中的Server是consul服务端高可用集群，Client是consul客户
端。Server之间通过局域网
或广域网通信实现数据一致性。每个Server或Client都是一个consul agent。Consul集群间使用了
GOSSIP协议通信和raft一致性算法。上面这张图涉及到了很多术语：

在单个数据中心中，Consul分为Client和Server两种节点（所有的节点也被称为Agent），consul客户端不保存数据，负责健康检查及转发数据请求到Server。Server节点保存数据，Server之间通过局域网或广域网通信实现数据一致性。每个Server或Client都是一个consul agent。Server节点有一个Leader和多个Follower，Leader节点会将数据同步到Follower，Server的数量推荐是3个或者5个，在Leader挂掉的时候会启动选举机制产生一个新的Leader。

上面这张图涉及到了很多术语：

- **Agent**

agent是一直运行在Consul集群中每个成员上的守护进程。通过运行 consul agent来启动。

agent可以运行在client或者server模式。指定节点作为client或者server是非常简单的，除非有其他agent实例。所有的agent都能运行DNS或者HTTP接口，并负责运行时检查和保持服务同步。

- **Client**

一个Client是一个转发所有RPC到server的代理。这个client是相对无状态的。client唯一执行的后台活动是加入LAN.

- **gossip池**

这有一个最低的资源开销并且仅消耗少量的网络带宽。

- **Server**

一个server是一个有一组扩展功能的代理，这些功能包括参与Raft选举，维护集群状态，响应RPC查询，与其他数据中心交互WANgossip和转发查询给leader或者远程数据中心。

- **DataCenter**

虽然数据中心的定义是显而易见的，但是有一些细微的细节必须考虑。例如，在EC2中，多个可用区域被认为组成一个数据中心？我们定义数据中心为一个私有的，低延迟和高带宽的一个网络环境。这不包括访问公共网络，但是对于我们而言，同一个EC2中的多个可用区域可以被认为是一个数据中心的一部分。

- **Consensus**

在我们的文档中，我们使用Consensus来表明就leader选举和事务的顺序达成一致。由于这些事务都被应用到有限状态机上，Consensus暗示复制状态机的一致性。

- **Gossip**

Consul建立在Serf的基础之上，它提供了一个用于多播目的的完整的gossip协议。Serf提供成员关系，故障检测和事件广播。更多的信息在gossip文档中描述。这足以知道gossip使用基于UDP的随机的点到点通信。

- **LAN Gossip**

它包含所有位于同一个局域网或者数据中心的所有节点。 

- **WAN Gossip**

它只包含Server。这些server主要分布在不同的数据中心并且通常通过因特网或者广域网通信。

在每个数据中心，client和server是混合的。一般建议有3-5台server。这是基于有故障情况下的可用性和性能之间的权衡结果，**因为越多的机器加入达成共识越慢**。然而，并不限制client的数量，它们可以很容易的扩展到数千或者数万台。

同一个数据中心的所有节点都必须加入gossip协议。这意味着gossip协议包含一个给定数据中心的所有节点。这服务于几个目的：

1. 第一，不需要在client上配置server地址。发现都是自动完成的。
2. 第二，检测节点故障的工作不是放在server上，而是分布式的。这是的故障检测相比心跳机制有更高的可扩展性。
3. 第三：它用来作为一个消息层来通知事件，比如leader选举发生时。

每个数据中心的server都是Raft节点集合的一部分。这意味着它们一起工作并选出一个leader，一个有额外工作的server。leader负责处理所有的查询和事务。作为一致性协议的一部分，事务也必须被复制到所有其他的节点。因为这一要求，当一个非leader得server收到一个RPC请求时，它将请求转发给集群leader。

server节点也作为WAN gossip Pool的一部分。这个Pool不同于LAN Pool，因为它是为了优化互联网更高的延迟，并且它只包含其他Consul server节点。这个Pool的目的是为了允许数据中心能够以low-touch的方式发现彼此。这使得一个新的数据中心可以很容易的加入现存的WAN gossip。因为server都运行在这个pool中，它也支持跨数据中心请求。当一个server收到来自另一个数据中心的请求时，它随即转发给正确数据中想一个server。该server再转发给本地leader。这使得数据中心之间只有一个很低的耦合，但是由于故障检测，连接缓存和复用，跨数据中心的请求都是相对快速和可靠的。

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
	Usage: consul [--version] [--help] <command> [<args>]
	
	Available commands are:
	    acl            Interact with Consul's ACLs
	    agent          Runs a Consul agent
	    catalog        Interact with the catalog
	    config         Interact with Consul's Centralized Configurations
	    connect        Interact with Consul Connect
	    debug          Records a debugging archive for operators
	    event          Fire a new event
	    exec           Executes a command on Consul nodes
	    force-leave    Forces a member of the cluster to enter the "left" state
	    info           Provides debugging information for operators.
	    intention      Interact with Connect service intentions
	    join           Tell Consul agent to join cluster
	    keygen         Generates a new encryption key
	    keyring        Manages gossip layer encryption keys
	    kv             Interact with the key-value store
	    leave          Gracefully leaves the Consul cluster and shuts down
	    lock           Execute a command holding a lock
	    login          Login to Consul using an auth method
	    logout         Destroy a Consul token created with login
	    maint          Controls node or service maintenance mode
	    members        Lists the members of a Consul cluster
	    monitor        Stream logs from a Consul agent
	    operator       Provides cluster-level tools for Consul operators
	    reload         Triggers the agent to reload configuration files
	    rtt            Estimates network round trip time between nodes
	    services       Interact with services
	    snapshot       Saves, restores and inspects snapshots of Consul server state
	    tls            Builtin helpers for creating CAs and certificates
	    validate       Validate config files/directories
	    version        Prints the Consul version
	    watch          Watch for changes in Consul
	
	$  consul -v
	Consul v1.9.0
	Revision a417fe510
	Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)


## 3.2. Agent

在Consul安装之后，必须运行一个Agent。这个Agent可以用Server模式或者Client模式运行。每个数据中心必须至少有个一个Server，不过一个集群推荐3或5个Server，单个Server在发生失败的情况下会发生数据丢失，因此不推荐使用。

所有其他的Agent以Cient模式运行。一个Client是一个非常轻量级的进程，它可以注册服务，运行健康检查，以及转发查询到服务器。Agent必须运行在集群的每个节点上。

- **启动Agent**

使用开发者模式启动一个Agent,这个模式可以非常容易快速地启动一个单节点的Consul环境。当然它并不是用于生产环境下并且它也不会持久任何状态。

	$  consul agent -dev
	==> Starting Consul agent...
	           Version: '1.9.0'
	           Node ID: '41958888-acfb-fe9e-13ae-a3b8f8534b2a'
	         Node name: 'VM-0-17-centos'
	        Datacenter: 'dc1' (Segment: '<all>')
	            Server: true (Bootstrap: false)
	       Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: 8502, DNS: 8600)
	      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
	           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false, Auto-Encrypt-TLS: false
	
	==> Log data will now stream in as it occurs:
	
	    2020-12-18T10:09:47.368+0800 [INFO]  agent.server.raft: initial configuration: index=1 servers="[{Suffrage:Voter ID:41958888-acfb-fe9e-13ae-a3b8f8534b2a Address:127.0.0.1:8300}]"
	    2020-12-18T10:09:47.368+0800 [INFO]  agent.server.raft: entering follower state: follower="Node at 127.0.0.1:8300 [Follower]" leader=
	    2020-12-18T10:09:47.369+0800 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: VM-0-17-centos.dc1 127.0.0.1
	    2020-12-18T10:09:47.369+0800 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: VM-0-17-centos 127.0.0.1
	    2020-12-18T10:09:47.369+0800 [INFO]  agent.router: Initializing LAN area manager
	    2020-12-18T10:09:47.369+0800 [INFO]  agent.server: Adding LAN server: server="VM-0-17-centos (Addr: tcp/127.0.0.1:8300) (DC: dc1)"
	    2020-12-18T10:09:47.369+0800 [INFO]  agent.server: Handled event for server in area: event=member-join server=VM-0-17-centos.dc1 area=wan
	    2020-12-18T10:09:47.369+0800 [INFO]  agent: Started DNS server: address=127.0.0.1:8600 network=udp
	    2020-12-18T10:09:47.369+0800 [INFO]  agent: Started DNS server: address=127.0.0.1:8600 network=tcp
	    2020-12-18T10:09:47.370+0800 [INFO]  agent: Starting server: address=127.0.0.1:8500 network=tcp protocol=http
	    2020-12-18T10:09:47.370+0800 [WARN]  agent: DEPRECATED Backwards compatibility with pre-1.9 metrics enabled. These metrics will be removed in a future version of Consul. Set `telemetry { disable_compat_1.9 = true }` to disable them.
	    2020-12-18T10:09:47.370+0800 [INFO]  agent: Started gRPC server: address=127.0.0.1:8502 network=tcp
	    2020-12-18T10:09:47.370+0800 [INFO]  agent: started state syncer
	==> Consul agent running!
	    2020-12-18T10:09:47.422+0800 [WARN]  agent.server.raft: heartbeat timeout reached, starting election: last-leader=
	    2020-12-18T10:09:47.422+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 127.0.0.1:8300 [Candidate]" term=2
	    2020-12-18T10:09:47.422+0800 [DEBUG] agent.server.raft: votes: needed=1
	    2020-12-18T10:09:47.422+0800 [DEBUG] agent.server.raft: vote granted: from=41958888-acfb-fe9e-13ae-a3b8f8534b2a term=2 tally=1
	    2020-12-18T10:09:47.422+0800 [INFO]  agent.server.raft: election won: tally=1
	    2020-12-18T10:09:47.422+0800 [INFO]  agent.server.raft: entering leader state: leader="Node at 127.0.0.1:8300 [Leader]"
	    2020-12-18T10:09:47.422+0800 [INFO]  agent.server: cluster leadership acquired
	    2020-12-18T10:09:47.422+0800 [DEBUG] agent.server: Cannot upgrade to new ACLs: leaderMode=0 mode=0 found=true leader=127.0.0.1:8300
	    2020-12-18T10:09:47.422+0800 [INFO]  agent.server: New leader elected: payload=VM-0-17-centos
	    2020-12-18T10:09:47.423+0800 [DEBUG] agent.server.autopilot: autopilot is now running
	    2020-12-18T10:09:47.423+0800 [DEBUG] agent.server.autopilot: state update routine is now running
	    2020-12-18T10:09:47.423+0800 [DEBUG] connect.ca.consul: consul CA provider configured: id=07:80:c8:de:f6:41:86:29:8f:9c:b8:17:d6:48:c2:d5:c5:5c:7f:0c:03:f7:cf:97:5a:a7:c1:68:aa:23:ae:81 is_primary=true
	    2020-12-18T10:09:47.432+0800 [INFO]  agent.server.connect: initialized primary datacenter CA with provider: provider=consul
	    2020-12-18T10:09:47.432+0800 [INFO]  agent.leader: started routine: routine="federation state anti-entropy"
	    2020-12-18T10:09:47.432+0800 [INFO]  agent.leader: started routine: routine="federation state pruning"
	    2020-12-18T10:09:47.432+0800 [INFO]  agent.leader: started routine: routine="intermediate cert renew watch"
	    2020-12-18T10:09:47.432+0800 [INFO]  agent.leader: started routine: routine="CA root pruning"
	    2020-12-18T10:09:47.432+0800 [DEBUG] agent.server: successfully established leadership: duration=10.485092ms
	    2020-12-18T10:09:47.432+0800 [INFO]  agent.server: member joined, marking health alive: member=VM-0-17-centos
	    2020-12-18T10:09:47.433+0800 [INFO]  agent.server: federation state anti-entropy synced
	    2020-12-18T10:09:47.488+0800 [DEBUG] agent: Skipping remote check since it is managed automatically: check=serfHealth
	    2020-12-18T10:09:47.489+0800 [INFO]  agent: Synced node info
	    2020-12-18T10:09:47.489+0800 [DEBUG] agent: Node info in sync

从日志信息中，可以看到我们Agent运行Server模式 `Server: true (bootstrap: false)`，

并且声明集群的Leader 

    2020-12-18T10:09:47.422+0800 [WARN]  agent.server.raft: heartbeat timeout reached, starting election: last-leader=
    2020-12-18T10:09:47.422+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 127.0.0.1:8300 [Candidate]" term=2
    2020-12-18T10:09:47.422+0800 [DEBUG] agent.server.raft: votes: needed=1
    2020-12-18T10:09:47.422+0800 [DEBUG] agent.server.raft: vote granted: from=41958888-acfb-fe9e-13ae-a3b8f8534b2a term=2 tally=1
    2020-12-18T10:09:47.422+0800 [INFO]  agent.server.raft: election won: tally=1
    2020-12-18T10:09:47.422+0800 [INFO]  agent.server.raft: entering leader state: leader="Node at 127.0.0.1:8300 [Leader]"
    2020-12-18T10:09:47.422+0800 [INFO]  agent.server: cluster leadership acquired
    2020-12-18T10:09:47.422+0800 [DEBUG] agent.server: Cannot upgrade to new ACLs: leaderMode=0 mode=0 found=true leader=127.0.0.1:8300
    2020-12-18T10:09:47.422+0800 [INFO]  agent.server: New leader elected: payload=VM-0-17-centos

另外，本地的成员已经被标记为一个健康的集群成员

	2020-12-18T10:09:47.422+0800 [INFO]  agent.server: New leader elected: payload=VM-0-17-centos

通过日志我们还可以看到，Consul开启了4个服务，分别是UDP，TCP，HTTP，GRPC

```
2020-12-18T10:09:47.369+0800 [INFO]  agent: Started DNS server: address=127.0.0.1:8600 network=udp
2020-12-18T10:09:47.369+0800 [INFO]  agent: Started DNS server: address=127.0.0.1:8600 network=tcp
2020-12-18T10:09:47.370+0800 [INFO]  agent: Starting server: address=127.0.0.1:8500 network=tcp protocol=http
2020-12-18T10:09:47.370+0800 [WARN]  agent: DEPRECATED Backwards compatibility with pre-1.9 metrics enabled. These metrics will be removed in a future version of Consul. Set `telemetry { disable_compat_1.9 = true }` to disable them.
2020-12-18T10:09:47.370+0800 [INFO]  agent: Started gRPC server: address=127.0.0.1:8502 network=tcp
```

- **单机启动**

```
consul agent -server -bootstrap-expect=1 -data-dir=/data/consul-data -ui -bind=192.168.1.221 -client=0.0.0.0
```

- **集群成员**

在终端上输入`consul members`，能看到Consul集群所有的节点

	$  consul members
	Node            Address         Status  Type    Build  Protocol  DC   Segment
	VM-0-17-centos  127.0.0.1:8301  alive   server  1.9.0  2         dc1  <all>

输出显示了节点的名称、运行地址、健康状态、在集群中的角色、版本信息等等。一些额外元数据可以通过 `-detailed`选项来查看

该命令输出显示你自己的节点，运行的地址，它的健康状态，它在集群中的角色，以及一些版本信息。另外元数据可以通过 -detailed 选项来查看。

	$ consul members -detailed
	Node            Address         Status  Tags
	VM-0-17-centos  127.0.0.1:8301  alive   acls=0,bootstrap=1,build=1.9.0:a417fe51,dc=dc1,ft_fs=1,ft_si=1,id=def3b026-0416-5b08-b35f-3a41ae38b4ad,port=8300,raft_vsn=3,role=consul,segment=<all>,vsn=2,vsn_max=3,vsn_min=2,wan_join_port=8302

members 命令选项的输出是基于**gossip协议**，并且其内容保证最终一致性。也就是说，在任何时候你在本地代理看到的内容也许与当前服务器中的状态并不是绝对一致的。如果需要强一致性的状态信息，使用HTTP API向Consul服务器发送请求`http://127.0.0.1:8500/v1/catalog/nodes`	

```
$ curl http://127.0.0.1:8500/v1/catalog/nodes
[
    {
        "ID": "def3b026-0416-5b08-b35f-3a41ae38b4ad", 
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
        "CreateIndex": 5, 
        "ModifyIndex": 7
    }
]
```

另外除了HTTP API，DNS接口也常被用来查询节点信息。但你必须确保你的DNS能够找到Consul代理的DNS服务器，Consul代理的DNS服务器默认运行在8600端口。

	$ dig @127.0.0.1 -p 8600 VM-0-17-centos.node.consul
	
	; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.6 <<>> @127.0.0.1 -p 8600 VM-0-17-centos.node.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53144
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2
	;; WARNING: recursion requested but not available
	
	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;VM-0-17-centos.node.consul.    IN      A
	
	;; ANSWER SECTION:
	VM-0-17-centos.node.consul. 0   IN      A       127.0.0.1
	
	;; ADDITIONAL SECTION:
	VM-0-17-centos.node.consul. 0   IN      TXT     "consul-network-segment="
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Fri Dec 18 10:40:15 CST 2020
	;; MSG SIZE  rcvd: 107

> dig的详细用法参考这篇文章
>
> https://edgar615.github.io/dig.html

- **关闭**

你能够使用 Ctrl-C （中断信号）来优雅地停止代理。停止代理后，你可以看到它脱离集群并且关闭的信息。

为了优雅地离开集群，Consul会通知其他的集群成员自己已经脱离了。如果你强制杀死代理的进程，那么其他的集群成员需要侦测节点是否失效。当一个成员离开，它的服务以及（checks）将从目录中移除。当一个成员失效，它的健康会简单地标记为critical，但它并不会被从目录中移除。Consul将自动尝试重新连接到失效的节点，并允许它在某些网络状况下恢复。

# 4. 服务

## 4.1. 定义服务

一个服务可以通过提供一个服务定义或者调用HTTP API来注册.

服务定义是最通用的注册服务的方法。

首先，为consul的配置创建一个目录，Consul加载这个配置目录下的所有配件文字，通常在Unix系统中惯例是建立以名为 /etc/consul.d 的目录（ .d 后缀暗示这个目录包含了一些配置文件的集合）。我们这里采用一个其他的方式

	$ mkdir /server/data/consul

下一步，我们创建一个服务定义的配置文件.

	$ echo '{"service": {"name": "web", "tags": ["java"], "port":8080}}' | tee /server/data/consul/web.json

上面的定义文件定义了一个名称为web，运行在8080端口的服务

接着，我们重启Agent，并指定配置文件目录

	$ consul agent -dev -config-dir=/server/data/consul
		...
	    2020-12-18T10:44:17.738+0800 [DEBUG] agent: Node info in sync
	    2020-12-18T10:44:17.738+0800 [DEBUG] agent: Service in sync: service=web
		...


在输出中的`Synced service 'web'`，意味着代理已经从配置文件中装载了该服务定义，并且已经成功注册该服务到服务目录中。如果你想注册多个服务，你可以在Consul配置目录中创建多个服务定义文件。

一旦Agent启动，并且服务已经注册，我们可以通过DNS或HTTP API来查询服务

## 4.2. DNS API

对于DNS API，服务的DNS名称是`NAME.service.consul`，默认所有的DSN名称都是在consul的命名空间下，当然这个也可以配置.`service`的子域名告诉Consul我们我们正在查询服务，`Name`是需要查询的服务名称

对于我们前面注册的web服务，一个合格的域名称是`web.service.consul`

	$ dig @127.0.0.1 -p 8600 web.service.consul
	
	; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.6 <<>> @127.0.0.1 -p 8600 web.service.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51922
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
	;; WARNING: recursion requested but not available
	
	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;web.service.consul.            IN      A
	
	;; ANSWER SECTION:
	web.service.consul.     0       IN      A       127.0.0.1
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Fri Dec 18 10:45:41 CST 2020
	;; MSG SIZE  rcvd: 63

上面的查询返回了一个带有节点IP地址的A记录，A记录只能包含IP地址

你也可以使用DNS API来获取完整的地址/端口的 SRV 记录：

	$ dig @127.0.0.1 -p 8600 web.service.consul SRV
	
	; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.6 <<>> @127.0.0.1 -p 8600 web.service.consul SRV
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61016
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 3
	;; WARNING: recursion requested but not available
	
	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;web.service.consul.            IN      SRV
	
	;; ANSWER SECTION:
	web.service.consul.     0       IN      SRV     1 1 8080 VM-0-17-centos.node.dc1.consul.
	
	;; ADDITIONAL SECTION:
	VM-0-17-centos.node.dc1.consul. 0 IN    A       127.0.0.1
	VM-0-17-centos.node.dc1.consul. 0 IN    TXT     "consul-network-segment="
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Fri Dec 18 10:46:23 CST 2020
	;; MSG SIZE  rcvd: 149

SRV记录显示了web服务运行在节点VM-0-17-centos.node.dc1.consul.的8080端口上.额外的信息和A记录返回的内容一样

最后，我们可以使用DNS API通过tags来过滤服务，基于tag查询服务的格式是`TAG.NAME.service.consul`，例如我们向Consul查询所有带`java`标记的web服务:

	$ dig @127.0.0.1 -p 8600 java.web.service.consul SRV
	
	; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.6 <<>> @127.0.0.1 -p 8600 java.web.service.consul SRV
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56507
	;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 3
	;; WARNING: recursion requested but not available
	
	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;java.web.service.consul.       IN      SRV
	
	;; ANSWER SECTION:
	java.web.service.consul. 0      IN      SRV     1 1 8080 VM-0-17-centos.node.dc1.consul.
	
	;; ADDITIONAL SECTION:
	VM-0-17-centos.node.dc1.consul. 0 IN    A       127.0.0.1
	VM-0-17-centos.node.dc1.consul. 0 IN    TXT     "consul-network-segment="
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Fri Dec 18 10:47:44 CST 2020
	;; MSG SIZE  rcvd: 154

如果查询服务未找到对应的服务，会返回AUTHORITY SECTION

	$ dig @127.0.0.1 -p 8600 redis.service.consul SRV
	
	; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.6 <<>> @127.0.0.1 -p 8600 redis.service.consul SRV
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 2308
	;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
	;; WARNING: recursion requested but not available
	
	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;redis.service.consul.          IN      SRV
	
	;; AUTHORITY SECTION:
	consul.                 0       IN      SOA     ns.consul. hostmaster.consul. 1608259702 3600 600 86400 0
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Fri Dec 18 10:48:22 CST 2020
	;; MSG SIZE  rcvd: 99

## 4.3. HTTP API

查看所有服务

```
$ curl -s http://127.0.0.1:8500/v1/catalog/services
{
    "consul": [],
    "web": [
        "java"
    ]
}

```

查看某个服务

	$ curl -s http://127.0.0.1:8500/v1/catalog/service/web
	[
	    {
	        "ID": "b0e0d63f-1a0e-acf4-63b8-1651e9446656",
	        "Node": "VM-0-17-centos",
	        "Address": "127.0.0.1",
	        "Datacenter": "dc1",
	        "TaggedAddresses": {
	            "lan": "127.0.0.1",
	            "lan_ipv4": "127.0.0.1",
	            "wan": "127.0.0.1",
	            "wan_ipv4": "127.0.0.1"
	        },
	        "NodeMeta": {
	            "consul-network-segment": ""
	        },
	        "ServiceKind": "",
	        "ServiceID": "web",
	        "ServiceName": "web",
	        "ServiceTags": [
	            "java"
	        ],
	        "ServiceAddress": "",
	        "ServiceWeights": {
	            "Passing": 1,
	            "Warning": 1
	        },
	        "ServiceMeta": {},
	        "ServicePort": 8080,
	        "ServiceEnableTagOverride": false,
	        "ServiceProxy": {
	            "MeshGateway": {},
	            "Expose": {}
	        },
	        "ServiceConnect": {},
	        "CreateIndex": 9,
	        "ModifyIndex": 9
	    }
	]

catalog API返回了指定节点以及指定的服务信息。

通过`/v1/health/service`指定passing=true，表示返回时过滤掉一些不健康的节点。这也是DNS在底层做的事情。

	$ curl -s http://127.0.0.1:8500/v1/health/service/web?passing
	[
	    {
	        "Node": {
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
	        },
	        "Service": {
	            "ID": "web",
	            "Service": "web",
	            "Tags": [
	                "java"
	            ],
	            "Address": "",
	            "Meta": null,
	            "Port": 8080,
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
	            "CreateIndex": 4225,
	            "ModifyIndex": 4225
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
	                "CheckID": "service:web",
	                "Name": "Service 'web' check",
	                "Status": "passing",
	                "Notes": "",
	                "Output": "",
	                "ServiceID": "web",
	                "ServiceName": "web",
	                "ServiceTags": [
	                    "java"
	                ],
	                "Type": "script",
	                "Definition": {},
	                "CreateIndex": 4225,
	                "ModifyIndex": 4231
	            }
	        ]
	    }
	] 

## 4.4. 更新服务

当配置文件修改后服务定义可以被更新，需要发送 SIGHUP 信号给代理。这可以让代理更新服务而无需停止代理或者让服务查询时服务不可用。

```
$ consul reload
Configuration reload triggered
```

可以看到多了一个redis的服务

```
$ curl -s http://127.0.0.1:8500/v1/catalog/services
{
    "consul": [],
    "redis": [
        "master"
    ],
    "web": [
        "java"
    ]
}

```

可以选择HTTP API来动态地增加，删除，以及更改服务。

> 在服务注册章节详细描述

# 5. 集群
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

# 6. 健康检查

可以通过一个定义检查或者通过调研HTTP API来注册一个监控检查

## 6.1. 定义检查
我们需要使用参数`enable_script_checks`或`enable_local_script_checks`来开启健康检查

```
$ consul agent -dev -config-dir=/server/data/consul -enable-script-checks
```

就像服务一样，使用定义是一个最为常用的方法来设置健康检查

	$ echo '{
	    "service": {
	        "name": "baidu", 
	        "check": {
	            "args": [
	                "ping", 
	                "-c1", 
	                "baidu.com", 
	                ">/dev/null"
	            ], 
	            "interval": "30s"
	        }
	    }
	}' > /server/data/consul/ping.json
	
	$ echo '{
	    "service": {
	        "name": "web", 
	        "tags": [
	            "java"
	        ], 
	        "port": 8080, 
	        "check": {
	            "args": [
	                "curl", 
	                "-si", 
	                "localhost:8080/health"
	            ], 
	            "interval": "10s"
	        }
	    }
	}' > /server/data/consul/web.json


第一个定义增加了一个主机级别的检测，名为"ping"。该检测每30秒间隔运行一次，调用命令 ping -c1 baidu.com。在一个基于脚本的健康检测中，该检测使用启动Consul进程的用户来启动该检测。如果检测命令返回一个非0的返回码，那么该节点将被标记为不健康。这就是任何基于 脚本 的健康检测的契约。

第二个命令修改名为 web 的服务，增加了一个检测，该检测每10秒用curl发送一个请求来验证该web服务是否可用。就像基于主机的健康检测，如果脚本返回一个非0的返回码，那该服务将被标记为不健康。

重启第二个Agent，或者使用`consul reload`, 或者发送一个`SIGHUP`

	$ consul reload
	Configuration reload triggered

查看日志

	[WARN]  agent: Check is now critical: check=service:web


日志指出了被检测的 web 服务被标记为危险。这是因为我们还没有实际运行一个web服务器，所以这个curl测试标记为失败了。

## 5.2. 检查健康状态
现在我们已经增加了一些健康检查，我们可以使用HTTP API来审查它们。首先，我们可以使用命令寻找任何失败的检测，这个命令可以在任何节点上运行

	$ curl http://127.0.0.1:8500/v1/health/state/critical
	[
	    {
	        "Node": "VM-0-17-centos",
	        "CheckID": "service:baidu",
	        "Name": "Service 'baidu' check",
	        "Status": "critical",
	        "Notes": "",
	        "Output": "",
	        "ServiceID": "baidu",
	        "ServiceName": "baidu",
	        "ServiceTags": [],
	        "Type": "script",
	        "Definition": {},
	        "CreateIndex": 26,
	        "ModifyIndex": 26
	    },
	    {
	        "Node": "VM-0-17-centos",
	        "CheckID": "service:web",
	        "Name": "Service 'web' check",
	        "Status": "critical",
	        "Notes": "",
	        "Output": "  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current\n                                 Dload  Upload   Total   Spent    Left  Speed\n\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0curl: (7) Failed connect to localhost:8080; Connection refused\n",
	        "ServiceID": "web",
	        "ServiceName": "web",
	        "ServiceTags": [
	            "java"
	        ],
	        "Type": "script",
	        "Definition": {},
	        "CreateIndex": 14,
	        "ModifyIndex": 25
	    }
	]


另外，通过DNS查询web服务，Consul不会放任何结果，因为这个服务是不健康的

	$ dig @127.0.0.1 -p 8600 web.service.consul
	
	; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.6 <<>> @127.0.0.1 -p 8600 web.service.consul
	; (1 server found)
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 50774
	;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
	;; WARNING: recursion requested but not available
	
	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;web.service.consul.            IN      A
	
	;; AUTHORITY SECTION:
	consul.                 0       IN      SOA     ns.consul. hostmaster.consul. 1608262882 3600 600 86400 0
	
	;; Query time: 0 msec
	;; SERVER: 127.0.0.1#8600(127.0.0.1)
	;; WHEN: Fri Dec 18 11:41:22 CST 2020
	;; MSG SIZE  rcvd: 97

# 7. WEB UI

	consul agent -client 0.0.0.0 -ui

http://<IP地址>:8500/ui

