---
layout: post
title: ShardingSphere（1）- Get Started
date: 2020-11-01
categories:
    - 分库分表
comments: true
permalink: shardingsphere-getstarted.html
---

# 1. Get Started

- 增加依赖

```
<dependency>
	<groupId>org.apache.shardingsphere</groupId>
	<artifactId>sharding-jdbc-spring-boot-starter</artifactId>
</dependency>
<dependency>
	<groupId>org.antlr</groupId>
	<artifactId>antlr4</artifactId>
</dependency>
```

- 初始化数据源

针对分库场景，我们设计了两个数据库，分别叫 ds0 和 ds1。显然，针对两个数据源，我们就需要初始化两个 DataSource 对象，这两个  DataSource 对象将组成一个 Map 并传递给 ShardingDataSourceFactory 工厂类：

```
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/ds0
        username: root
        password: 123456
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/ds1
        username: root
        password: 123456
```

- 设置分片策略

明确了数据源之后，我们需要设置针对分库的分片策略：

```
spring:
  shardingsphere:
    sharding:
      default-database-strategy:
        inline:
          sharding-column: user_id
          algorithm-expression: ds$->{user_id % 2}
```

在 ShardingSphere 中存在一组 ShardingStrategyConfiguration，这里使用的是基于行表达式的 InlineShardingStrategyConfiguration。
 InlineShardingStrategyConfiguration 包含两个需要设置的参数，一个是指定分片列名称的  shardingColumn，另一个是指定分片算法行表达式的 algorithmExpression。在我们的配置方案中，将基于 user_id 列对 2 的取模值来确定数据应该存储在哪一个数据库中。通过default-database-strategy我们指定了默认分库策略。

- 设置绑定表

所谓绑定表，是指与分片规则一致的一组主表和子表。例如，user  表和 user_password 表中都存在一个 user_id 字段。如果我们在应用过程中按照这个 user_id  字段进行分片，那么这两张表就可以构成互为绑定表关系。

引入绑定表概念的根本原因在于，互为绑定表关系的多表关联查询不会出现笛卡尔积，因此关联查询效率将大大提升。举例说明，如果所执行的为下面这条 SQL：

复制代码

```
SELECT u.password FROM user u JOIN user_password up ON u.user_id=up.user_id WHERE u.user_id in (1, 2);
```

如果我们不显式配置绑定表关系，假设分片键 user_id 将值 1 路由至第 1 片，将数值 2 路由至第 0 片，那么路由后的 SQL 应该为 4 条，它们呈现为笛卡尔积：

复制代码

```
SELECT u.password FROM user0 u JOIN user_password0 up ON u.user_id=up.user_id WHERE u.user_id in (1, 2);
 
SELECT u.password FROM user0 u JOIN user_password1 up ON u.user_id=up.user_id WHERE u.user_id in (1, 2);
 
SELECT u.password FROM user1 u JOIN user_password0 up ON u.user_id=up.user_id WHERE u.user_id in (1, 2);
 
SELECT u.password FROM user1 u JOIN user_password1 up ON u.user_id=up.user_id WHERE u.user_id in (1, 2);
```

然后，在配置绑定表关系后，路由的 SQL 就会减少到 2 条：

复制代码

```
SELECT u.password FROM user0 u JOIN user_password0 up ON u.user_id=up.user_id WHERE u.user_id in (1, 2);
 
SELECT u.password FROM user1 u JOIN user_password1 up ON u.user_id=up.user_id WHERE u.user_id in (1, 2);
```

**请注意，如果想要达到这种效果，互为绑定表的各个表的分片键要完全相同**。在上面的这些 SQL 语句中，我们不难看出，这个需要完全相同的分片键就是 user_id。

```
spring:
  shardingsphere:
    sharding:
      binding-tables:
      - user
      - user_password
```

- 设置表分片规则

配置完与分库操作相关的配置信息后，我们还需要配置表分片规则（必填项）。

表的分片规则包含了用于设置真实数据节点的 actualDataNodes；用于设置分库策略的  databaseShardingStrategyConfig；以及用于设置分布式环境下的自增列生成器的  keyGeneratorConfig。前面已经在 ShardingRuleConfiguration 中设置了默认的  databaseShardingStrategyConfig。

对于 user 表而言，由于存在两个数据源，所以，它所属于的  actual-data-nodes 可以用行表达式 ds$->{0..1}.user 来进行表示，代表在 ds0 和  ds1 中都存在表 user。keyGeneratorConfig的配置在后面的描述。

```
spring:
  shardingsphere:
    sharding:
      tables:
        user:
          actual-data-nodes: ds$->{0..1}.user
        user_password:
          actual-data-nodes: ds$->{0..1}.user_password
```

- 测试

```
JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
for (int i = 10; i < 20; i ++) {
	User user = new User();
	user.setUserId(Integer.toUnsignedLong(i));
	user.setNickname("edgar" + i);
	user.setUsername("edgar" + i);
	user.setMail("edgar" + i + "@github.com");
	user.setMobile("180000000" + i);
	SQLBindings sqlBindings = SqlBuilder.insert(user);
	jdbcTemplate.update(sqlBindings.sql(), sqlBindings.bindings().toArray());

	UserPassword userPassword = new UserPassword();
	// 人为将密码的ID设成与用户的奇偶相反
	userPassword.setUserPasswordId(Integer.toUnsignedLong(i + 1));
	userPassword.setUserId(user.getUserId());
	userPassword.setPassword("123456");
	sqlBindings = SqlBuilder.insert(userPassword);
	jdbcTemplate.update(sqlBindings.sql(), sqlBindings.bindings().toArray());

}

SQLBindings sqlBindings = SqlBuilder.countByExample(User.class, Example.create());
int count = jdbcTemplate.queryForObject(sqlBindings.sql(), Integer.class);
System.out.println(count);// 返回10
```

执行完程序后，我们可以发现数据被均匀的插入到了ds0和ds1

```
mysql> select * from ds0.user;
+---------+----------+----------+-------------+--------------------+---------------------+---------------------+
| user_id | username | nickname | mobile      | mail               | created_on          | updated_on          |
+---------+----------+----------+-------------+--------------------+---------------------+---------------------+
|      10 | edgar10  | edgar10  | 18000000010 | edgar10@github.com | 2020-12-30 16:01:26 | 2020-12-30 16:01:26 |
|      12 | edgar12  | edgar12  | 18000000012 | edgar12@github.com | 2020-12-30 16:01:26 | 2020-12-30 16:01:26 |
|      14 | edgar14  | edgar14  | 18000000014 | edgar14@github.com | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
|      16 | edgar16  | edgar16  | 18000000016 | edgar16@github.com | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
|      18 | edgar18  | edgar18  | 18000000018 | edgar18@github.com | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
+---------+----------+----------+-------------+--------------------+---------------------+---------------------+
5 rows in set (0.00 sec)

mysql> select * from ds1.user;
+---------+----------+----------+-------------+--------------------+---------------------+---------------------+
| user_id | username | nickname | mobile      | mail               | created_on          | updated_on          |
+---------+----------+----------+-------------+--------------------+---------------------+---------------------+
|      11 | edgar11  | edgar11  | 18000000011 | edgar11@github.com | 2020-12-30 16:01:26 | 2020-12-30 16:01:26 |
|      13 | edgar13  | edgar13  | 18000000013 | edgar13@github.com | 2020-12-30 16:01:26 | 2020-12-30 16:01:26 |
|      15 | edgar15  | edgar15  | 18000000015 | edgar15@github.com | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
|      17 | edgar17  | edgar17  | 18000000017 | edgar17@github.com | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
|      19 | edgar19  | edgar19  | 18000000019 | edgar19@github.com | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
+---------+----------+----------+-------------+--------------------+---------------------+---------------------+
5 rows in set (0.01 sec)

mysql> select * from ds0.user_password;
+------------------+---------+----------+---------------------+---------------------+
| user_password_id | user_id | password | created_on          | updated_on          |
+------------------+---------+----------+---------------------+---------------------+
|               11 |      10 | 123456   | 2020-12-30 16:01:26 | 2020-12-30 16:01:26 |
|               13 |      12 | 123456   | 2020-12-30 16:01:26 | 2020-12-30 16:01:26 |
|               15 |      14 | 123456   | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
|               17 |      16 | 123456   | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
|               19 |      18 | 123456   | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
+------------------+---------+----------+---------------------+---------------------+
5 rows in set (0.00 sec)

mysql> select * from ds1.user_password;
+------------------+---------+----------+---------------------+---------------------+
| user_password_id | user_id | password | created_on          | updated_on          |
+------------------+---------+----------+---------------------+---------------------+
|               12 |      11 | 123456   | 2020-12-30 16:01:26 | 2020-12-30 16:01:26 |
|               14 |      13 | 123456   | 2020-12-30 16:01:26 | 2020-12-30 16:01:26 |
|               16 |      15 | 123456   | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
|               18 |      17 | 123456   | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
|               20 |      19 | 123456   | 2020-12-30 16:01:27 | 2020-12-30 16:01:27 |
+------------------+---------+----------+---------------------+---------------------+
5 rows in set (0.00 sec)
```



# 2. 广播表

**所谓广播表（BroadCastTable），是指所有分片数据源中都存在的表，也就是说，这种表的表结构和表中的数据在每个数据库中都是完全一样的**。广播表的适用场景比较明确，通常针对数据量不大且需要与海量数据表进行关联查询的应用场景，典型的例子就是每个分片数据库中都应该存在的字典表。

```
spring:
  shardingsphere:
    sharding:
      broadcast-tables:
      - dict
      - dict_item
```

测试

```
JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
Dict dict = new Dict();
dict.setDictId(1L);
dict.setName("user state");
dict.setDictCode("USER_STATE");
SQLBindings sqlBindings = SqlBuilder.insert(dict);
jdbcTemplate.update(sqlBindings.sql(), sqlBindings.bindings().toArray());

sqlBindings = SqlBuilder.countByExample(Dict.class, Example.create());
int count = jdbcTemplate.queryForObject(sqlBindings.sql(), Integer.class);
System.out.println(count);// 返回1
```

```
mysql> select * from ds0.dict;
+---------+------------+------------+---------------+---------------------+---------------------+
| dict_id | name       | dict_code  | default_value | created_on          | updated_on          |
+---------+------------+------------+---------------+---------------------+---------------------+
|       1 | user state | USER_STATE | NULL          | 2020-12-30 16:36:40 | 2020-12-30 16:36:40 |
+---------+------------+------------+---------------+---------------------+---------------------+
1 row in set (0.00 sec)

mysql> select * from ds1.dict;
+---------+------------+------------+---------------+---------------------+---------------------+
| dict_id | name       | dict_code  | default_value | created_on          | updated_on          |
+---------+------------+------------+---------------+---------------------+---------------------+
|       1 | user state | USER_STATE | NULL          | 2020-12-30 16:36:40 | 2020-12-30 16:36:40 |
+---------+------------+------------+---------------+---------------------+---------------------+
1 row in set (0.00 sec)
```

# 3. 主键生成策略

前面介绍表的分片规则时介绍到了主键策略，我们先简单了解一下雪花算法

```
spring:
  shardingsphere:
    sharding:
      tables:
        user:
          actual-data-nodes: ds$->{0..1}.user
          key-generator:
            column: user_id
            type: SNOWFLAKE
            props:
              worker:
                id: 33
        user_password:
          actual-data-nodes: ds$->{0..1}.user_password
          key-generator:
            column: user_password_id
            type: SNOWFLAKE
            props:
              worker:
                id: 33
```

再次测试，我们可以发现ID都使用了雪花算法生成。

# 4. 实现分表

相比分库，分表操作是在同一个数据库中，完成对一张表的拆分工作。所以从数据源上讲，我们只需要定义一个 DataSource 对象，如何使用TableRuleConfiguration中的tableShardingStrategyConfig来设置分片即可。

```
spring:
  shardingsphere:
    datasource:
      names: ds0
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/ds0
        username: root
        password: 123456
    sharding:
      binding-tables:
      - user
      - user_password
      broadcast-tables:
      - dict
      - dict_item
      tables:
        user:
          actual-data-nodes: ds0.user$->{0..1}
          table-strategy:
            inline:
              sharding-column: user_id
              algorithm-expression: user$->{user_id % 2}
        user_password:
          actual-data-nodes: ds0.user_password$->{0..1}
          table-strategy:
            inline:
              sharding-column: user_id
              algorithm-expression: user_password$->{user_id % 2}
```



# 5. 实现分库分表

在完成独立的分库和分表操作之后，我们可以尝试把分库和分表结合起来。

> 实际应用中分表的意义不大，分库即可

下面的配置演示了2个库，每个库又分3个表的实例

```
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/ds0
        username: root
        password: 123456
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/ds1
        username: root
        password: 123456
    sharding:
      default-database-strategy:
        inline:
          sharding-column: user_id
          algorithm-expression: ds$->{user_id % 2}
      binding-tables:
      - user
      - user_password
      broadcast-tables:
      - dict
      - dict_item
      tables:
        user:
          actual-data-nodes: ds$->{0..1}.user$->{0..2}
          table-strategy:
            inline:
              sharding-column: user_id
              algorithm-expression: user$->{user_id % 3}
        user_password:
          actual-data-nodes: ds$->{0..1}.user_password$->{0..2}
          table-strategy:
            inline:
              sharding-column: user_id
              algorithm-expression: user_password$->{user_id % 3}
```

# 6. 参考资料

《ShardingSphere 核心原理精讲》