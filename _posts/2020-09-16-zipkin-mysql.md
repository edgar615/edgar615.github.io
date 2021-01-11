---
layout: post
title: 使用MySQL存储Zipkin数据
date: 2020-09-16
categories:
	- Zipkin
comments: true
permalink: zipkin-mysql.html
---

 默认情况下，**Zipkin Server** 都会将跟踪信息存储在内存中，每次重启 **Zipkin Server** 都会使得之前收集的跟踪信息丢失，而且当有大量跟踪信息时我们的内存存储也会成为瓶颈。

所有正常情况下我们都需要将跟踪信息对接到外部存储组件（比如 **MySQL**、**Elasticsearch**）中去。本文首先介绍如何使用 **MySQL** 实现 **Zipkin** 的数据持久化。

首先创建一个名为 **zipkin** 的数据库，如何到[Github](https://github.com/openzipkin/zipkin/tree/master/zipkin-storage/mysql-v1)上下载MySQL脚本，执行创建表结构

使用jar包运行zipkin

```
java -jar zipkin-server-2.12.9-exec.jar --STORAGE_TYPE=mysql --MYSQL_HOST=localhost --MYSQL_TCP_PORT=3306  --MYSQL_USER=root --MYSQL_PASS=123456 --MYSQL_DB=zipkin
```

使用docker运行

```
docker run --name zipkin -d -p 9411:9411 -e STORAGE_TYPE=mysql -e  MYSQL_HOST=localhost -e MYSQL_TCP_PORT=3306 -e MYSQL_USER=root -e  MYSQL_PASS=123456 -e MYSQL_DB=zipkin openzipkin/zipkin
```

也可以使用docker-compose.yml启动

```
version: '2'

services:
  zipkin:
    image: openzipkin/zipkin
    container_name: zipkin
    environment:
      - STORAGE_TYPE=mysql
      - MYSQL_HOST=192.168.60.1
      - MYSQL_TCP_PORT=3306
      - MYSQL_USER=root
      - MYSQL_PASS=hangge1234
      - MYSQL_DB=zipkin
      #- RABBIT_ADDRESSES=192.168.60.133:5672
      #- RABBIT_USER=hangge
      #- RABBIT_PASSWORD=123
    ports:
      - 9411:9411
```

> 我使用本地的MySQL测试，发现zipkin连接不上MySQL，报下面错误，看issue可能是mariadb-client的版本不对
>
> Caused by: java.lang.NoClassDefFoundError: Could not initialize class org.mariadb.jdbc.internal.com.send.authentication.SendGssApiAuthPacket
>         at jdk.internal.reflect.GeneratedConstructorAccessor26.newInstance(Unknown Source) ~[?:?]
>         at jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(Unknown Source) ~[?:?]
>         at java.lang.reflect.Constructor.newInstanceWithCaller(Unknown Source) ~[?:?]
>         at java.lang.reflect.Constructor.newInstance(Unknown Source) ~[?:?]

最终我使用官方提供的[docker-compose.yml](https://github.com/openzipkin-attic/docker-zipkin/blob/master/docker-compose.yml)测试成功

