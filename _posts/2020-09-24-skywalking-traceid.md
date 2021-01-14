---
layout: post
title: Skywalking在代码中获取TraceId
date: 2020-09-24
categories:
	- Skywalking
comments: true
permalink: skywalking-traceId.html
---

Skywalking采用的Java探针的方式，那么如果我们想达到Zipkin那样自定义Span，或者将traceId加到日志中的功能时该怎么做？

> 这种方式的缺点是对代码有侵入

# 1. 打印traceId

添加依赖

```
<dependency>
  <groupId>org.apache.skywalking</groupId>
  <artifactId>apm-toolkit-logback-1.x</artifactId>
  <version>8.3.0</version>
</dependency>
```

修改日志

```
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
	<encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
		<layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
			<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36}  [%tid] %msg%n</pattern>
		</layout>
	</encoder>
</appender>
```

- encoder的class是LayoutWrappingEncoder，不是PatternLayoutEncoder!

- 使用%tid 来占trace-id的位置，默认TID:N/A，当有请求调用时，会显示trace-id

启动后我们可以看启动日志

```
10:21:25.744 [main] INFO  o.s.b.w.e.tomcat.TomcatWebServer  [TID:N/A] Tomcat started on port(s): 9001 (http) with context path ''
```

有请求后我们可以看到日志有了跟踪ID

```
10:22:50.477 [http-nio-9001-exec-9] INFO  c.g.e.s.c.s.SchoolServiceController  [TID:9dea8fc32c6e436e873c5f9ed155b620.75.16105909704540001] Going to call student service to get data!
```

# 2. 将方法加入追踪链路

添加依赖

```
<dependency>
  <groupId>org.apache.skywalking</groupId>
  <artifactId>apm-toolkit-trace</artifactId>
  <version>8.3.0</version>
</dependency>
```

在项目中加入Maven依赖之后，就可以使用`@Trace`来追踪相关方法了。

```
@Override
@Trace
public String callStudentServiceAndGetData(String schoolname) {
	...
}
```

可以看到我们加@Trace注解的方法被追踪了。

![](/assets/images/posts/skywalking-trace/skywalking-trace-1.png)

可以通过`@Trace(operationName = "delegateMethod")`修改端点名称

我们还可以为追踪链路增加其他额外的信息，比如记录参数和返回信息。实现方式：在方法上增加@Tag或者@Tags。

```
@Override
@Trace
@Tags({@Tag(key = "schoolname", value = "arg[0]"),
	  @Tag(key = "result", value = "returnedObj")})
public String callStudentServiceAndGetData(String schoolname) {
	...
}
```


![](/assets/images/posts/skywalking-trace/skywalking-trace-2.png)

# 3. 代码中获取traceId

```
String traceId =  TraceContext.traceId();
```

