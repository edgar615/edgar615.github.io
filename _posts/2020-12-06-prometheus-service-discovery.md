---
layout: post
title: prometheus服务发现
date: 2020-12-06
categories:
    - prometheus
comments: true
permalink: prometheus-service-discovery.html
---

> 基本复制自https://mp.weixin.qq.com/s/89C3-wdOjXUkgJ1z3xyVJg

前面文章我们使用各类exporter分别对系统、数据库和HTTP服务进行监控指标采集，对于所有监控指标对应的Target的运行状态和资源使用情况，都是用Prometheus的静态配置功能 `static_configs` 来 手动添加主机IP和端口，然后重载服务让Prometheus发现。

对于一组比较少的服务器的测试环境中，这种手动方式添加配置信息是最简单的方法。但是实际生产环境中，对于成百上千的节点组成的大型集群又或者Kubernetes这样的大型集群，很明显，手动方式捉襟见肘了。为此，Prometheus提前已经设计了一套服务发现功能。

Prometheus 服务发现能够自动检测分类，并且能够识别新节点和变更节点。也就是说，可以在容器或者云平台中，自动发现并监控节点或更新节点，动态的进行数据采集和处理。目前 Prometheus 已经支持了很多常见的自动发现服务，比如 `consul` `ec2` `gce` `serverset_sd_config` `openStack` `kubernetes` 等等。

# 1. 为什么要用自动发现

在基于云(IaaS或者CaaS)的基础设施环境中用户可以像使用水、电一样按需使用各种资源（计算、网络、存储）。按需使用就意味着资源的动态性，这些资源可以随着需求规模的变化而变化。例如在AWS中就提供了专门的AutoScall服务，可以根据用户定义的规则动态地创建或者销毁EC2实例，从而使用户部署在AWS上的应用可以自动的适应访问规模的变化。

这种按需的资源使用方式对于监控系统而言就意味着没有了一个固定的监控目标，所有的监控对象(基础设施、应用、服务)都在动态的变化。对于Nagias这类基于Push模式传统监控软件就意味着必须在每一个节点上安装相应的Agent程序，并且通过配置指向中心的Nagias服务，受监控的资源与中心监控服务器之间是一个强耦合的关系，要么直接将Agent构建到基础设施镜像当中，要么使用一些自动化配置管理工具(如Ansible、Chef)动态的配置这些节点。当然实际场景下除了基础设施的监控需求以外，我们还需要监控在云上部署的应用，中间件等等各种各样的服务。要搭建起这样一套中心化的监控系统实施成本和难度是显而易见的。

而对于Prometheus这一类基于Pull模式的监控系统，显然也无法继续使用的static_configs的方式静态的定义监控目标。而对于Prometheus而言其解决方案就是引入一个中间的代理人（服务注册中心），这个代理人掌握着当前所有监控目标的访问信息，Prometheus只需要向这个代理人询问有哪些监控目标控即可， 这种模式被称为服务发现。

![](/assets/images/posts/prometheus-sd/prometheus-sd-1.png)

在不同的场景下，会有不同的东西扮演者代理人（服务发现与注册中心）这一角色。比如在AWS公有云平台或者OpenStack的私有云平台中，由于这些平台自身掌握着所有资源的信息，此时这些云平台自身就扮演了代理人的角色。Prometheus通过使用平台提供的API就可以找到所有需要监控的云主机。在Kubernetes这类容器管理平台中，Kubernetes掌握并管理着所有的容器以及服务信息，那此时Prometheus只需要与Kubernetes打交道就可以找到所有需要监控的容器以及服务对象。Prometheus还可以直接与一些开源的服务发现工具进行集成，例如在微服务架构的应用程序中，经常会使用到例如Consul这样的服务发现注册软件，Promethues也可以与其集成从而动态的发现需要监控的应用服务实例。除了与这些平台级的公有云、私有云、容器云以及专门的服务发现注册中心集成以外，Prometheus还支持基于DNS以及文件的方式动态发现监控目标，从而大大的减少了在云原生，微服务以及云模式下监控实施难度。

# 2. 基于文件的服务发现

在Prometheus支持的众多服务发现的实现方式中，基于文件的服务发现是最通用的方式。这种方式不需要依赖于任何的平台或者第三方服务。对于Prometheus而言也不可能支持所有的平台或者环境。通过基于文件的服务发现方式下，Prometheus会定时从文件中读取最新的Target信息，因此，你可以通过任意的方式将监控Target的信息写入即可。用户可以通过JSON或者YAML格式的文件，定义所有的监控目标。

yaml格式

```
- targets: ['192.168.1.220:9100']
  labels:
    app:   'app1'
    env:   'game1'
    region: 'us-west-2'
- targets: ['192.168.1.221:9100']
  labels:
    app:    'app2'
    env:   'game2'
    region: 'ap-southeast-1'
```

JSON格式

```
[
  {
    "targets": [ "192.168.1.221:29090"],
    "labels": {
      "app": "app1",
      "env": "game1",
      "region": "us-west-2"
    }
  },
  {
    "targets": [ "192.168.1.222:29090" ],
    "labels": {
      "app": "app2",
      "env": "game2",
      "region": "ap-southeast-1"
    }
  }
]
```

同时还可以通过为这些实例添加一些额外的标签信息，例如使用env标签标示当前节点所在的环境，这样从这些实例中采集到的样本信息将包含这些标签信息，从而可以通过该标签按照环境对数据进行统计。

在Prometheus配置文件中添加以下内容：

```
  - job_name: 'file_sd_test'
    scrape_interval: 10s
    file_sd_configs:
    - files:
       - /data/prometheus/static_conf/*.yml
       - /data/prometheus/static_conf/*.json
```

这里定义了一个基于file_sd_configs的监控采集test任务，其中模式的任务名称为file_sd_test。在yml文件中可以使用yaml标签覆盖默认的job名称，然后重载Prometheus服务。

Prometheus默认每5m重新读取一次文件内容，当需要修改时，可以通过refresh_interval进行设置，例如：

```
  - job_name: 'file_sd_test'
    scrape_interval: 10s
    file_sd_configs:
    - refresh_interval: 30s # 30s重载配置文件
      files:
      - /data/prometheus/static_conf/*.yml
      - /data/prometheus/static_conf/*.json
```

通过这种方式，Prometheus会自动的周期性读取文件中的内容。当文件中定义的内容发生变化时，不需要对Prometheus进行任何的重启操作。

这种通用的方式可以衍生了很多不同的玩法，比如与自动化配置管理工具(Ansible)结合、与Cron  Job结合等等。对于一些Prometheus还不支持的云环境，比如国内的阿里云、腾讯云等也可以使用这种方式通过一些自定义程序与平台进行交互自动生成监控Target文件，从而实现对这些云环境中基础设施的自动化监控支持。

# 3. 基于DNS的发现

> 受限于环境，没有测试过

对于一些环境，可能基于文件与consul服务发现已经无法满足的时候，我们可能就需要DNS来做服务发现了。在互联网架构中，我们使用主机节点或者Kubernetes集群通常是不对外暴露IP的，这就要求我们在一个内部局域网或者专用的网络中部署DNS服务器，使用DNS服务来完成内部网络中的域名解析工作。这个时候我们就可以使用Prometheus的DNS服务发现，Prometheus的DNS服务发现有俩种方法，第一种是使用DNA A记录来做自动发现，第二种方法是DNS SRV，第一种显然没有没有SRV资源记录更为便捷

**DNA A**，dnsmasq的配置如下

```
# 验证 test1 DNS记录
nslookup test1.example.com
Server:		127.0.0.53
Address:	127.0.0.53#53

Non-authoritative answer:
Name:	test1.example.com
Address: 192.168.1.221
# 验证 test2 DNS记录
nslookup test2.example.com
Server:		127.0.0.53
Address:	127.0.0.53#53

Non-authoritative answer:
Name:	test2.example.com
Address: 192.168.1.222
```

Prometheus配置

```
  # 基于DNS A记录发现
  - job_name: 'DNS-A'                  # job 名称
    metrics_path: "/metrics"            # 路径
    dns_sd_configs:
    - names: ["test1.example.com", "test2.example.com"]   # A记录
      type: A                                   # 解析类型
      port: 29100                     # 端口
```

**DNS SRV**

```
# 配置dns解析
cat /etc/dnsmasq.d/localdomain.conf
address=/test1.example.com/192.168.1.221
address=/test2.example.com/192.168.1.222
# 添加 SRV 记录
cat /etc/dnsmasq.conf
srv-host =_prometheus._tcp.example.com,test1.example.com,29100
srv-host =_prometheus._tcp.example.com,test2.example.com,29100

# 验证srv服务是否正确，192.168.1.123 是内部DNS服务器，
dig @192.168.1.123 +noall +answer SRV _prometheus._tcp.example.com
output...
_prometheus._tcp.example.com. 0	IN	SRV	0 0 9100 test1.example.com.
_prometheus._tcp.example.com. 0	IN	SRV	0 0 9100 test2.example.com.
```

Prometheus配置

```
  - job_name: 'DNS-SRV'                # 名称
    metrics_path: "/metrics"            # 获取数据的路径
    dns_sd_configs:                     # 配置使用DNS解析
    - names: ['_prometheus._tcp.example.com']   # 配置SRV对应的解析地址
```

# 4. 基于Consul的发现

在Prometheus配置文件中添加以下内容

> 使用spring boot项目测试

```
  - job_name: 'school_service_exporter'
	consul_sd_configs:
	- server: 192.168.159.131:8500
	  #token: '${CONSUL_HTTP_TOKEN}'
	  services: ['school-service']
```

![](/assets/images/posts/prometheus-sd/prometheus-sd-2.png)

# 5. 参考资料

https://mp.weixin.qq.com/s/89C3-wdOjXUkgJ1z3xyVJg