---
layout: post
title: Skywalking（1）- 简单使用
date: 2020-09-20
categories:
	- Skywalking
comments: true
permalink: skywalking.html
---

SkyWalking是一款广受欢迎的国产应用性能监控APM（Application Performance Monitoring）产品，主要针对微服务、Cloud Native和容器化（Docker、Kubernetes、Mesos）架构的应用。SkyWalking的核心是一个分布式追踪系统，且目前是Apache基金会的顶级项目。

> SkyWalking 是一个完整的 APM（Application Performance Management）系统，链路追踪只是其中的一部分。

SkyWalking 和 Zipkin 的定位不同，决定了它们不是相同类型的产品。SkyWalking 中提供的组件更加偏向业务应用层面，并没有涉及过多的组件级别的观测；Zipkin 提供了更多组件级别的链路观测，但并没有提供太多的链路分析能力。你可以根据两者的侧重点来选择合适的产品。

要通过SkyWalking将Java应用数据上报至链路追踪控制台，首先需要完成埋点工作。SkyWalking既支持自动埋点（Dubbo、gRPC、JDBC、OkHttp、Spring、Tomcat、Struts、Jedis等），也支持手动埋点（OpenTracing）

# 1. 系统架构
![](/assets/images/posts/skywalking/skywalking-0.jpg)

从中间往上看，首先是 Receiver Cluster，它代表接收器集群，是整个后端服务的接入入口，专门用来收集各个指标，链路信息，相当于我在上一节所讲的链路收集器。

再往后面走是 Aggregator Cluster，代表聚合服务器，它会汇总接收器集群收集到的所有数据，并且最终存储至数据库，并进行对应的告警通知。右侧标明了它支持的多种不同的存储方式，比如常见的 ElasticSearch、MySQL，我们可以根据需要来选择。

图的左上方表示，我们可以使用 CLI 和 GUI，通过 HTTP 的形式向集群服务器发送请求来读取数据。

图的下方，是数据接收的来源，分为 3 种：

- Metrics System：统计系统。目前支持从 Prometheus 中拉取数据到 SkyWalking，也支持程序通过 micrometer 推送数据。
- Agents： 业务探针。指在各个业务系统中集成探针服务来进行链路追踪，也就是链路追踪中的链路采集部分。SkyWalking 支持 Java、Go、.Net、PHP、NodeJS、Python、Nginx LUA 语言的探针，是目前市面上支持探针语言比较全的系统。同时，它还支持通过 gRPC 或者 HTTP 的方式来传递数据。
- Service Mesh：SkyWalking 还支持目前比较新的 Service Mesh 监控技术，通过观测这部分数据同样可以实现链路数据的观测。


# 2. 使用

## 2.1. 运行Skywalking

首先到官网上下载apache-skywalking-apm-es7-8.3.0.tar.gz，解压后运行`bin`目录下的`startup.sh`脚本即可启动skywalking服务（使用内存存储数据）

SkyWalking控制台服务默认监听8080端口，若有防火墙需要开放该端口。若希望允许远程传输，则还需要开放11800（gRPC）和12800（rest）端口，远程agent将通过该端口传输收集的数据：

正常启动成功后，使用浏览器访问主页如下：

![](/assets/images/posts/skywalking/skywalking-1.png)

## 2.2. 服务链路追踪

首先为每个服务创建一个agent目录`/agent/student`、`/agent/school`然后将skywalking里的agent目录下的所有文件拷贝出来，分别粘贴到新建的agent目录中：

```
# The service name in UI
agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}

# Backend service addresses.
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}
```

配置好agent之后，启动程序是需要指定`skywalking-agent.jar`的目录路径：

```
java -javaagent:/agent/student/skywalking-agent.jar -jar student.java
java -javaagent:/agent/school/skywalking-agent.jar -jar school.java
```

如果不想为每个服务都单独拷贝一个agent目录，则可以通过添加JVM启动参数来覆写配置项

```
-javaagent:/agent/skywalking-agent.jar
-Dskywalking.agent.service_name=school
-Dskywalking.collector.backend_service=localhost:11800
```

请求接口后，到SkyWalking的“追踪”页面上，就可以查看到调用链路信息了

![](/assets/images/posts/skywalking/skywalking-2.png)

点击链路上的节点可以查看到对应的详情：

![](/assets/images/posts/skywalking/skywalking-3.png)

可以看到支持jdbc调用跟踪

![](/assets/images/posts/skywalking/skywalking-4.png)

点击拓扑图可以查看服务拓扑图

![](/assets/images/posts/skywalking/skywalking-5.png)

在仪表盘上可以监控端点、服务实例

![](/assets/images/posts/skywalking/skywalking-6.png)

![](/assets/images/posts/skywalking/skywalking-7.png)

可以看到支持redis调用跟踪

![](/assets/images/posts/skywalking/skywalking-8.png)

支持grpc调用跟踪

![](/assets/images/posts/skywalking/skywalking-9.png)

支持Kafka调用跟踪

![](/assets/images/posts/skywalking/skywalking-10.png)
