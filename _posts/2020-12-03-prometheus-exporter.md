---
layout: post
title: prometheus数据采集Exporter 
date: 2020-12-03
categories:
    - prometheus
comments: true
permalink: prometheus-exporter.html
---

> 基本复制自爱可生开源社区的《Prometheus 数据采集》，地址忘了

Prometheus 的监控对象各式各样，没有统一标准。为了解决这个问题，Prometheus 制定了一套监控规范，符合这个规范的样本数据可以被 Prometheus 采集并解析样本数据。**Exporter 在 Prometheus 监控系统中是一个采集监控数据并通过 Prometheus 监控规范对外提供数据的组件，针对不同的监控对象可以实现不同的 Exporter，这样就解决了监控对象标准不一的问题。**从广义上说，所有可以向  Prometheus 提供监控样本数据的程序都可以称为 Exporter，Exporter 的实例也就是我们上期所说的"target"。

# 1. 运行方式

Exporter 有两种运行方式

- 集成到应用中

  使用 Prometheus 提供的 Client Library，可以很方便地在应用程序中实现监控代码，这种方式可以将程序内部的运行状态暴露给  Prometheus，适用于需要较多自定义监控指标的项目。目前一些开源项目就增加了对 Prometheus 监控的原生支持，如  Kubernetes，ETCD 等。

- 独立运行

  在很多情况下，对象没法直接提供监控接口，可能原因有：

  \1. 项目发布时间较早，并不支持 Prometheus 监控接口，如 MySQL、Redis；

  \2. 监控对象不能直接提供 HTTP 接口，如监控 Linux 系统状态指标。

  对于上述情况，用户可以选择使用独立运行的 Exporter。除了用户自行实现外，Prometheus 社区也提供了许多独立运行的 Exporter，常见的有 Node  Exporter、MySQL Server  Exporter。更多详情可以到官网了解：https://prometheus.io/docs/instrumenting/exporters/

# 2. 接口数据规范

可以参考prometheus自己的metrics暴露接口

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 3.9794e-05
go_gc_duration_seconds{quantile="0.25"} 0.000119721
go_gc_duration_seconds{quantile="0.5"} 0.00024831
go_gc_duration_seconds{quantile="0.75"} 0.000588213
go_gc_duration_seconds{quantile="1"} 0.001718785
go_gc_duration_seconds_sum 0.007368653
go_gc_duration_seconds_count 17
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 36
# HELP go_memstats_frees_total Total number of frees.
# TYPE go_memstats_frees_total counter
go_memstats_frees_total 520286
# HELP prometheus_http_request_duration_seconds Histogram of latencies for HTTP requests.
# TYPE prometheus_http_request_duration_seconds histogram
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="0.1"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="0.2"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="0.4"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="1"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="3"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="8"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="20"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="60"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="120"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="+Inf"} 1
prometheus_http_request_duration_seconds_sum{handler="/api/v1/label/:name/values"} 0.001304263
prometheus_http_request_duration_seconds_count{handler="/api/v1/label/:name/values"} 1
```

Exporter 通过 HTTP 接口以文本形式向 Prometheus 暴露样本数据，格式简单，没有嵌套，可读性强。每个监控指标对应的数据文本格式如下：

```
# HELP <监控指标名称> <监控指标描述>
# TYPE <监控指标名称> <监控指标类型>
<监控指标名称>{ <标签名称>=<标签值>,<标签名称>=<标签值>...} <样本值1> <时间戳>
<监控指标名称>{ <标签名称>=<标签值>,<标签名称>=<标签值>...} <样本值2> <时间戳>
...
```

- 以 # 开头的行，如果后面跟着"HELP"，Prometheus 将这行解析为监控指标的描述，通常用于描述监控数据的来源
- 以 # 开头的行，如果后面跟着"TYPE"，Prometheus 将这行解析为监控指标的类型，支持的类型有：Counter、Gauge、Histogram、Summary、Untyped。Prometheus 在存储数据时是不区分数据类型的，所以当你在犹豫一个数据类型应该用 Counter 或 Gauge 时，可以试试 Untype
- 以 # 开头的行，如果后面没有跟着"HELP"或"TYPE"，则 Prometheus 将这行视为注释，解析时忽略
- 如果一个监控指标有多条样本数据，那么每条样本数据的标签值组合应该是唯一的
- 每行数据对应一条样本数据
- 时间戳应为采集数据的时间，是可选项，如果 Exporter 没有提供时间戳的话，Prometheus Server 会在拉取到样本数据时将时间戳设置为当前时间
- Summary 和 Histogram 类型的监控指标要求提供两行数据分别表示该监控指标所有样本的和、样本数量，命名格式为：`<监控指标名称>_sum`、`<监控指标名称>_count`

**Summary 的样本数据格式**

- 根据 Exporter 提供的分位点，样本会被计算后拆分成多行数据，每行使用标签"quantile"区分，"quantile"的值包括 Exporter 提供的所有分位点。

- 数据的排列顺序必须是按照标签"quantile"值递增；
- 提供两行数据分别表示该监控指标所有样本的和、样本数量，命名格式为：`<监控指标名称>_sum`、`<监控指标名称>_count`

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 3.9794e-05
go_gc_duration_seconds{quantile="0.25"} 0.000119721
go_gc_duration_seconds{quantile="0.5"} 0.00024831
go_gc_duration_seconds{quantile="0.75"} 0.000588213
go_gc_duration_seconds{quantile="1"} 0.001718785
go_gc_duration_seconds_sum 0.007368653
go_gc_duration_seconds_count 17
```

**Histogram 的样本数据格式**

- 根据 Exporter 提供的 Bucket 值，样本会被计算后拆分成多行数据，每行使用标签"le"区分，"le"为 Exporter 提供的 Buckets；
- 数据的排列顺序必须是按照标签"le"值递增；
- 必须要有一行数据的标签 le="+Inf"，值为该监控指标的样本总数

```
# HELP prometheus_http_request_duration_seconds Histogram of latencies for HTTP requests.
# TYPE prometheus_http_request_duration_seconds histogram
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="0.1"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="0.2"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="0.4"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="1"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="3"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="8"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="20"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="60"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="120"} 1
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/label/:name/values",le="+Inf"} 1
prometheus_http_request_duration_seconds_sum{handler="/api/v1/label/:name/values"} 0.001304263
prometheus_http_request_duration_seconds_count{handler="/api/v1/label/:name/values"} 1
```

这样的文本格式也有不足之处：

- 文本内容可能过于冗长；
- Prometheus 在解析时不能校验 HELP 和 TYPE 字段是否缺失，如果缺失 HELP 字段，这条样本数据的来源可能就难以判断；如果缺失 TYPE 字段，Prometheus 对这条样本数据的类型就无从得知；
- 相比于 protobuf，Prometheus 使用的文本格式没有做任何压缩处理，解析成本较高

# 3. Exporter实现方式的考量

我们可以在程序一开始就初始化所有的监控指标，这种方案通常接下来会开启一个采样协程去定期采集、更新监控指标的样本数据，最新的样本数据将一直保留在内存中，在接到 Prometheus Server  的请求时，返回内存里的样本数据。这个方案的优点在于，易于控制采样频率；不用担心并发采样可能带来的资源抢占问题。不足之处有：

- 由于样本数据不会被自动清理，当某个已被采样的采集对象失效了，Prometheus Server 依然能拉取到它的样本数据，只是这个数据从监控对象失效时就已经不会再被更新。这就需要 Exporter 自己提供一个对无效监控对象的数据清理机制；
- 由于响应 Prometheus Server 的请求是从内存里取数据，如果 Exporter 的采样协程异常卡住，Prometheus Server 也无法感知，拉取到的数据可能是过期数据；
- Prometheus Server 拉取的数据不是即时采样的，对于某时间点的数据一致性不能保证。

另一种方案是 MySQL Server Exporter 和 Node Exporter 采用的，也是 Prometheus  官方推荐的方案。该方案是在每次接到 Prometheus Server  的请求时，初始化新的监控指标，开启一个采样协程。和方案一不同的是，这些监控指标只在请求期间存活。然后采样协程会去采集所有样本数据并返回给  Prometheus Server。

相比于方案一，方案二的数据是即时拉取的，可以保证时间点的数据一致性；因为监控指标会在每次请求时重新初始化，所以也不会存在失效的样本数据。不过方案二同样有不足之处：

- 当多个拉取请求同时发生时，需要控制并发采集样本的资源消耗；
- 当多个拉取请求同时发生时，在短时间内需要对同一个监控指标读取多次，对于一个变化频率较低的监控指标来说，多次读取意义不大，却增加了对资源的占用。