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

要通过SkyWalking将Java应用数据上报至链路追踪控制台，首先需要完成埋点工作。SkyWalking既支持自动埋点（Dubbo、gRPC、JDBC、OkHttp、Spring、Tomcat、Struts、Jedis等），也支持手动埋点（OpenTracing）

# 使用

## 运行Skywalking

首先到官网上下载apache-skywalking-apm-es7-8.3.0.tar.gz，解压后运行`bin`目录下的`startup.sh`脚本即可启动skywalking服务（使用内存存储数据）

SkyWalking控制台服务默认监听8080端口，若有防火墙需要开放该端口。若希望允许远程传输，则还需要开放11800（gRPC）和12800（rest）端口，远程agent将通过该端口传输收集的数据：

正常启动成功后，使用浏览器访问主页如下：

![](/assets/images/posts/skywalking/skywalking-1.png)

## 服务链路追踪

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
