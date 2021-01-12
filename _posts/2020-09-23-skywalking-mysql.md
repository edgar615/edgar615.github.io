---
layout: post
title: Skywalking使用MySQL作为存储
date: 2020-09-23
categories:
	- Skywalking
comments: true
permalink: skywalking-mysql.html
---

创建数据库

```
mysql> create database skywalking;
Query OK, 1 row affected (0.02 sec)
```

skywalking会自动创建表

修改`config/applicaiton.yml`文件的storage

```
storage:
  selector: ${SW_STORAGE:mysql}
  mysql:
    properties:
      jdbcUrl: ${SW_JDBC_URL:"jdbc:mysql://localhost:3306/skywalking"}
      dataSource.user: ${SW_DATA_SOURCE_USER:root}
      dataSource.password: ${SW_DATA_SOURCE_PASSWORD:123456}

```

启动后报错

```
Failed to get driver instance for jdbcUrl=jdbc:mysql://localhost:3306/skywalking
```

我们需要把MySQL驱动包上传到oap-libs/目录

