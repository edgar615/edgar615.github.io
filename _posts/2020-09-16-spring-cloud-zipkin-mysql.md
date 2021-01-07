---
layout: post
title: Spring Cloud Zipkin（2）-  一些常用的调用跟踪
date: 2020-09-16
categories:
    - Spring
comments: true
permalink: spring-cloud-zipkin-mysql.html
---

前面我们已经了解了通过resttemplate发起远程调用的调用链跟踪，下面简单记录一下我在项目中用到的一些组件。
# 1. MySQL
> 使用brave-instrumentation-mysql的方法我用jdbctemplate没有成功，改用了p6spy成功了，后面再研究原因

## 1.1. brave-instrumentation-mysql

引入依赖

```
<dependency>
  <groupId>io.zipkin.brave</groupId>
  <artifactId>brave-instrumentation-mysql</artifactId>
  <version>5.13.3</version>
</dependency>
```

在jdbcurl中添加拦截器

```
url: jdbc:mysql://localhost:3306/ds0?statementInterceptors=brave.mysql.MySQLStatementInterceptor&zipkinServiceName=mysqlService
```

## 1.2. p6spy

引用依赖

```
<dependency>
  <groupId>com.github.gavlyukovskiy</groupId>
  <artifactId>p6spy-spring-boot-starter</artifactId>
  <version>1.6.2</version>
</dependency>
```

再次测试我们可以看zipkin中多了JDBC的调用信息

![](/assets/images/posts/zipkin/zipkin-6.png)

我们用可以调整跟踪参数

```
# Creates span for every connection and query. Works only with p6spy or datasource-proxy.
decorator.datasource.sleuth.enabled=true
# Specify traces that will be created in zipkin
decorator.datasource.sleuth.include=connection, query, fetch
```

更多信息参考https://github.com/gavlyukovskiy/spring-boot-data-source-decorator

# 2. Redis

# 3. Kafka

# 4. gRPC

# 5. Nginx



# 7. 参考资料

《Spring Cloud 原理与实战 》

```

```