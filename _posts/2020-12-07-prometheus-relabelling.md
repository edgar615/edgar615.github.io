---
layout: post
title: prometheus Relabelling机制
date: 2020-12-07
categories:
    - prometheus
comments: true
permalink: prometheus-relabelling.html
---

> 基本复制自https://mp.weixin.qq.com/s/1exosZYHjpWtu8oCpbKJvw

通过服务发现的方式，我们可以在不重启Prometheus服务的情况下动态的发现需要监控的Target实例信息。

![](/assets/images/posts/prometheus-relabelling/prometheus-sdrelabelling-1.png)

如上图所示，对于线上环境我们可能会划分为:dev, stage,  prod不同的集群。每一个集群运行多个主机节点，每个服务器节点上运行一个Node Exporter实例。Node  Exporter实例会自动注册到Consul中，而Prometheus则根据Consul返回的Node  Exporter实例信息动态的维护Target列表，从而向这些Target轮询监控数据。

然而，如果我们可能还需要：

- 按照不同的环境dev, stage, prod聚合监控数据？
- 对于研发团队而言，我可能只关心dev环境的监控数据，如何处理？
- 如果为每一个团队单独搭建一个Prometheus Server。那么如何让不同团队的Prometheus Server采集不同的环境监控数据？

面对以上这些场景下的需求时，我们实际上是希望Prometheus Server能够按照某些规则（比如标签）从服务发现注册中心返回的Target实例中有选择性的采集某些Exporter实例的监控数据。

# 1. Relabeling机制

> 相关内容在 [prometheus数据采集](https://edgar615.github.io/prometheus-collector.html)有过描述

在Prometheus所有的Target实例中，都包含一些默认的Metadata标签信息。

默认情况下，当Prometheus加载Target实例完成后，这些Target时候都会包含一些默认的标签：

- `__address__：当前Target实例的访问地址<host>:<port>`
- `__scheme__：采集目标服务访问地址的HTTP Scheme，HTTP或者HTTPS`
- `__metrics_path__：采集目标服务访问地址的访问路径`
- `__param_<name>：采集任务目标服务的中包含的请求参数`

上面这些标签将会告诉Prometheus如何从该Target实例中获取监控数据。除了这些默认的标签以外，我们还可以为Target添加自定义的标签，例如，在“基于文件的服务发现”小节中的示例中，我们通过JSON配置文件，为Target实例添加了自定义标签env，如下所示该标签最终也会保存到从该实例采集的样本数据中：

```
node_cpu{cpu="cpu0",env="prod",instance="localhost:9100",job="node",mode="idle"}
```

一般来说，Target以__作为前置的标签是在系统内部使用的，因此这些标签不会被写入到样本数据中。不过这里有一些例外，例如，我们会发现所有通过Prometheus采集的样本数据中都会包含一个名为instance的标签，该标签的内容对应到Target实例的__address__。这里实际上是发生了一次标签的重写处理。

这种发生在采集样本数据之前，对Target实例的标签进行重写的机制在Prometheus被称为Relabeling。

**使用replace/labelmap重写标签**

Relabeling最基本的应用场景就是基于Target实例中包含的metadata标签，动态的添加或者覆盖标签。例如，通过Consul动态发现的服务实例还会包含以下Metadata标签信息：

- `__meta_consul_address：consul地址`
- `__meta_consul_dc：consul服务所在的数据中心`
- `__meta_consulmetadata：服务的metadata`
- `__meta_consul_node：consul服务node节点的信息`
- `__meta_consul_service_address：服务访问地址`
- `__meta_consul_service_id：服务ID`
- `__meta_consul_service_port：服务端口`
- `__meta_consul_service：服务名称`
- `__meta_consul_tags：服务包含的标签信息`

在默认情况下，从Node Exporter实例采集上来的样本数据如下所示

```
node_cpu{cpu="cpu0",instance="localhost:9100",job="node",mode="idle"} 93970.8203125
```

我们希望能有一个额外的标签dc可以表示该样本所属的数据中心：

```
node_cpu{cpu="cpu0",instance="localhost:9100",job="node",mode="idle", dc="dc1"} 93970.8203125
```

在每一个采集任务的配置中可以添加多个relabel_config配置，一个最简单的relabel配置如下：

```
scrape_configs:
  - job_name: node_exporter
    consul_sd_configs:
      - server: localhost:8500
        services:
          - node_exporter
    relabel_configs:
    - source_labels:  ["__meta_consul_dc"]
      target_label: "dc"
```

在这个例子中，通过从Target实例中获取__meta_consul_dc的值，并且重写所有从该实例获取的样本中。

**使用hashmod计算source_labels的Hash值**

> 没有测试通过

当relabel_config设置为hashmod时，Prometheus会根据modulus的值作为系数，计算source_labels值的hash值。例如：

```
scrape_configs
- job_name: 'file_ds'
  relabel_configs:
    - source_labels: [__address__]
      modulus:       4
      target_label:  tmp_hash
      action:        hashmod
  file_sd_configs:
  - files:
    - targets.json
```

根据当前Target实例__address__的值以4作为系数，这样每个Target实例都会包含一个新的标签tmp_hash，并且该值的范围在1~4之间，查看Target实例的标签信息，可以看到如下的结果，每一个Target实例都包含了一个新的tmp_hash值。

利用Hashmod的能力在Target实例级别实现对采集任务的功能分区的:

```
scrape_configs:
  - job_name: some_job
    relabel_configs:
    - source_labels: [__address__]
      modulus:       4
      target_label:  __tmp_hash
      action:        hashmod
    - source_labels: [__tmp_hash]
      regex:         ^1$
      action:        keep
```

这里需要注意的是，如果relabel的操作只是为了产生一个临时变量，以作为下一个relabel操作的输入，那么我们可以使用 `__tmp` 作为标签名的前缀，通过该前缀定义的标签就不会写入到Target或者采集到的样本的标签中。

# 2. 参考资料

https://mp.weixin.qq.com/s/1exosZYHjpWtu8oCpbKJvw