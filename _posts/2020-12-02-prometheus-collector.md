---
layout: post
title: prometheus数据采集
date: 2020-12-02
categories:
    - prometheus
comments: true
permalink: prometheus-collector.html
---

> 基本复制自爱可生开源社区的《Prometheus 数据采集》，地址忘了

# 1. 采集数据格式及分类

## 1.1. 采集数据的格式

Prometheus 使用 metric 表示监控度量指标，它由 metric name(度量指标名称) 和 labels(标签对) 组成：

```
<metricname>{<label name=<labelvalue>, ...}
```

metric name 指明了监控度量指标的一般特征，比如 **http_requests_total** 代表收到的 http 请求的总数。metric name 必须由字母、数字、下划线或者冒号组成。冒号是保留给 recording rules 使用的，不应该被直接使用。

Labels 体现了监控度量指标的维度特征，比如 **http_requests_total{method="POST", status="200“}** 代表 POST 响应结果为 200 的请求总数。Prometheus 不仅能很容易地通过增加 label 为一个 metric 增加描述维度，而且还很方便的支持数据查询时的过滤和聚合，比如需要获取所有响应为 200 的请求的总数时，只需要指定 **http_request_total{status="200"}**。

Prometheus 将 metric 随时间流逝产生的一系列值称之为 time series(时间序列)。某个确定的时间点的数据被称为 sample(样本)，它由一个 float64 的浮点值和以毫秒为单位的时间戳组成。

## 1.2. 采集数据的分类

Prometheus 将采集的数据分为 **Counter、Gauge、Histogram、Summary** 四种类型。

**需要注意**的是，这只是一种逻辑分类，Prometheus 内部并没有使用采集的数据的类型信息，而是将它们做为无类型的数据进行处理。这在未来可能会改变。

- **Counter**

Counter 是计数器类型，适合单调递增的场景，比如请求的总数、完成的任务总数、出现的错误总数等。它拥有很好的不相关性，不会因为重启而重置为 0。

- **Gauge**

Gauge 用来表示可增可减的值，比如 CPU 和内存的使用量、IO 大小等。

- **Histogram**

Histogram 是一种累积直方图，它通常用来描述监控项的长尾效应。举个例子：假设使用 Hitogram 来分析 API 调用的响应时间，使用数组 [30ms, 100ms, 300ms, 1s, 3s, 5s, 10s] 将响应时间分为 8 个区间。那么每次采集到响应时间，比如 200ms，那么对应的区间 (0, 30ms], (30ms, 100ms], (100ms, 300ms] 的计数都会加 1。最终以响应时间为横坐标，每个区间的计数值为纵坐标，就能得到 API 调用响应时间的累积直方图。

**Summary**

Summary 和 Histogram 类似，它记录的是监控项的分位数。什么是分位数？举个例子：假设对于一个 http 请求调用了 100 次，得到 100 个响应时间值。将这 100 个时间响应值按照从小到大的顺序排列，那么 0.9 分位数（90% 位置）就代表着第 90 个数。

通过 Histogram 可以近似的计算出百分位数，但是结果并不准确，而 Summary 是在客户端计算的，比 Histogram 更准确。不过，Summary 计算消耗的资源更多，并且计算的指标不能再获取平均数或者关联其他指标，所以它通常独立使用。

# 2. **使用建议**

## 2.1. Metric 的命名

- Metric 名字应该以它所属的领域开头，比如关于进程的 metric 以 process 开头：**process**_cpu_seconds_total。
- Metric名字的结尾应该带有描述性的复数形式的基本单位。如果是总数类的 metric，还可以在结尾加上 **total**，比如：http_requests_total。

## 2.2. Label 的选择

Label 应该用来描述 metric 的典型特征，比如使用 **operation="create|update|delete"** 描述不同类型的 http 请求。需要特别注意：不能将用户 ID、邮件地址这种取值范围非常广泛的值作为 label，否则会显著的增加数据存储量。同时，一个 metric 的 label 数量也不应该过多，单个 metric 的 label 数量尽量保持在 10 个以内。

## 2.3. Histogram 与 Summary 的选择

- 如果需要使用聚合函数，使用 Histogram
- 如果对于观测值的分布有大致的预期，使用 Histogram，否则使用 Summary

##2.4. 应该监测什么？

- 从服务的类型来讲，应该监测所有类型的服务：在线服务、离线服务和批处理任务
- 从单一服务的实现来讲，应该监测服务的关键逻辑，比如关键逻辑执行的总数、失败次数、重试次数等
- 从服务的质量来讲，应该监测服务的请求总数、请求错误率和请求响应时间
- 从系统资源上来讲，应该监测资源的利用率、饱和度和错误

# 3. 数据采集流程

> **target：**采集目标，Prometheus Server 会从这些目标设备上采集监控数据
>
> **sample：** Prometheus Server 从 targets 采集回来的数据样本
>
> **meta label：** 执行 relabel 前，target 的原始标签。可在 Prometheus 的 `/targets` 页面或发送 `GET /api/v1/targets` 请求查看。

![](/assets/images/posts/prometheus-collector/prometheus-collector-1.png)

## 3.1. relabel (targets 标签修改/过滤)

relabel 是 Prometheus 提供的一个针对 target 的功能，relabel 发生 Prometheus Server 从 target 采集数据之前，可以对 target 的标签进行修改或者使用标签进行 target 筛选。注意以下几点：

- Prometheus 在 relabel 步骤默认会为 target 新增一个名为 instance 的标签，并设置成 "__address__" 标签的值；
- 在 relabel 结束后，以 "__" 开头的标签不会被存储到磁盘；
- meta label 会一直保留在内存中，直到 target 被移除。

在 Prometheus 的 targets 页面，可以看到 target 在  relabel 之前的标签，如下图所示，在 relabel 之前，target  的标签有："__address__"，"__metrics_path__"，"__schema__"，"job"。经过 relabel  之后我们最终看到的 targets 的标签为：instance、job。

![](/assets/images/posts/prometheus-collector/prometheus-collector-2.png)

## 3.2. relabel 配置
relabel 的基本配置项：

- source_labels: [<labelname>, ...] #需要进行 relabel 操作的 meta labels
- target_label: <labelname> #relabel 操作的目标标签，当使用 action 为 "replace" 时会把替换的结果写入 target_label
- regex: <regex> #正则表达式，用于在 source_labels 的标签值中提取匹配的内容。默认为"(.*)"
- modulus: <uint64> #用于获取源标签值的哈希的模数
- replacement: <string> #regex 可能匹配到多个内容，replacement 指定要使用哪一个匹配内容进行替换，默认为 "$1"，表示使用第一个匹配的内容
- action: <relabel_action> #定义对 source_labels 进行何种操作，默认为 "replace"

除了使用replace以外，还可以定义action的配置为labelmap。与replace不同的是，labelmap会根据regex的定义去匹配Target实例所有标签的名称，并且以匹配到的内容为新的标签名称，其值作为新标签的值。

## 3.2.1. 示例：replace 修改标签

假如我们希望给 targets 添加一个 "host" 标签，内容取 "__address__" 的 host 部分，可以添加如下 relabel 配置：

```
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    relabel_configs:
    - source_labels: ["__address__"] #我们要替换的 meta label 为"__address__"
      target_label: "host" #给 targets 新增一个名为 "host" 的标签
      regex: "(.*):(.*)" #将匹配的内容分为两部分 groups--> (host):(port)
      replacement: $1 #将匹配的 host 第一个内容设置为新标签的值
      action: replace
```

*"__metrics_path__" 标签保存了 target 提供的 metrics 访问路径，默认情况下"__metrics_path__" 标签在 relabel 之后是会被移除的，但是我们又希望在查询 metrics 时能方便的看到这个采集端的 metrics  访问路径，那么可以使用 replace 将 "__metrics_path__" 标签替换成我们希望的标签，并保留  "__metrics_path__" 的值，配置可以简化如下：

```
- source_labels:  ["__metrics_path__"]    #我们要替换的 meta label 为 "__metrics_path__"
  target_label: "metrics_path"   #给 targets 新增一个名为 "metrics_path" 的标签
```

![](/assets/images/posts/prometheus-collector/prometheus-collector-3.png)

### 3.2.2. 示例：keep/drop 筛选 target

当需要过滤 target 时，可以将 action 项定义为 keep 或 drop。接着上面的例子我们再继续添加如下配置：

```
- source_labels:  ["host"]
  regex: "localhost"  #只保留 host 标签值为 "localhost" 的 targets
  action: keep
```

# 4. scrape 拉取样本

Prometheus 通过 http 从 target 采集所有 metrics 的样本，http 路径可以通过 "metrics_path" 配置，默认为 "/metrics"。请求超时时间的 "scrape_timeout"配置，默认 10s，可根据网络状况作相应调整。标签的合法性也会在这个过程中检查。

## 4.1. honor label 冲突检查

Prometheus 会默认给 metric 添加一些标签，如 "job"、"instance"，或者某些配置项配置了一些特定标签，如果采集回来的时间序列也存在同名的标签，那冲突就产生了。 "honor_labels" 就是用来解决这样的场景的，如果 "honor_labels" 设置为  "true"，那么冲突标签的值会使用采集到的标签值；如果设置为 "false"，采集上来的冲突标签会被重命名：加上 "exported_"  前缀，如 "exported_job"、"exported_instance" 。

## 4.2. metric relabel（metric 标签重写）

metric_relabel 功能、配置和 relabel 类似，区别在于 metric_relabel 针对 sample 的标签，在 config 文件中的配置项为“metric_relabel_configs”。metric_relabel 不支持 Prometheus  自动生成的时间序列，如"up"、"scrape_duration_seconds"、"scrape_samples_scraped"、"scrape_samples_post_metric_relabeling"、"scrape_series_added"等。通常用于过滤掉意义不大、或采集成本过高的时间序列。

## 4.3. save

经过一系列处理后，采集到的数据会被持久化保存。