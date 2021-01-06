---
layout: post
title: ShardingSphere（3）- 读写分离
date: 2020-11-12
categories:
    - 分库分表
comments: true
permalink: shardingsphere-rw-separation.html
---

目前 ShardingSphere 支持单主库、多从库的主从架构来完成分片环境下的读写分离，暂时不支持多主库的应用场景。

在数据库主从架构中，因为从库一般会有多台，所以当执行一条面向从库的 SQL 语句时，我们需要实现一套负载均衡机制来完成对目标从库的路由。ShardingSphere 默认提供了随机（Random）和轮询（RoundRobin）这两种负载均衡算法来完成这一目标。

另一方面，由于主库和从库之间存在一定的同步时延和数据不一致情况，所以在有些场景下，我们可能更希望从主库中获取最新数据。ShardingSphere 同样考虑到了这方面需求，开发人员可以通过 Hint 机制来实现对主库的强制路由。

# 1. 使用

为了演示一主多从架构，我们初始化了一个主数据源 dsmaster 以及两个从数据源 dsslave0 和 dsslave1：

```
spring:
  shardingsphere:
    datasource:
      names: dsmaster,dsslave1,dsslave2
      dsmaster:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.131:3306/ds0
        username: root
        password: 123456
      dsslave2:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.132:3306/ds0
        username: root
        password: 123456
      dsslave1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.133:3306/ds0
        username: root
        password: 123456
```

配置主从配置，使用随机算法

```
spring:
  shardingsphere:
    masterslave:
      name: rw_test
      master-data-source-name: dsmaster
      slave-data-source-names: dsslave1,dsslave2
      load-balance-algorithm-type: random #轮询是round_robin
```

开启日志

```
spring:
  shardingsphere:
    props:
      sql:
        show: true
```

通过观察日志，可以发现已经实现了读写分离

# 2. 强制读主

通过`hintManager.setMasterRouteOnly();`我们可以实现强制读主的功能

# 3. 与分库分表结合 

我们同样可以在分库分表的基础上添加读写分离功能。这时候，我们需要设置两个主数据源 dsmaster0 和 dsmaster1，然后针对每个主数据源分别设置两个从数据源：

```
spring:
  shardingsphere:
    datasource:
      names: dsmaster0, dsmaster1,dsmaster0-dsslave0,dsmaster0-dsslave1,dsmaster1-dsslave0,dsmaster1-dsslave1
      dsmaster0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.131:3306/ds0
        username: root
        password: 123456
      dsmaster1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.131:3306/ds1
        username: root
        password: 123456
      dsmaster0-dsslave0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.132:3306/ds0
        username: root
        password: 123456
      dsmaster0-dsslave1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.132:3306/ds1
        username: root
        password: 123456
      dsmaster1-dsslave0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.133:3306/ds0
        username: root
        password: 123456
      dsmaster1-dsslave1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.159.133:3306/ds1
        username: root
        password: 123456
```

这时候的库分片策略 default-database-strategy 同样分别指向 dsmaster0 和 dsmaster1 这两个主数据源：

```
spring:
  shardingsphere:
    sharding:
      default-database-strategy:
        inline:
          sharding-column: user_id
          algorithm-expression: dsmaster$->{user_id % 2}
      binding-tables:
        - user
        - user_password
      tables:
        user:
          actual-data-nodes: dsmaster$->{0..1}.user
        user_password:
          actual-data-nodes: dsmaster$->{0..1}.user_password
```

完成这些设置之后，同样需要设置两个主数据源对应的配置项：

```
spring:
  shardingsphere:
    sharding:
      master-slave-rules:
        dsmaster0:
          master-data-source-name: dsmaster0
          slave-data-source-names: dsmaster0-dsslave0,dsmaster0-dsslave1
          load-balance-algorithm-type: round_robin
        dsmaster1:
          master-data-source-name: dsmaster1
          slave-data-source-names: dsmaster1-dsslave0,dsmaster1-dsslave1
          load-balance-algorithm-type: round_robin
```

这样，我们就在分库分表的基础上添加了对读写分离的支持。

# 4. 参考资料

《ShardingSphere 核心原理精讲 》