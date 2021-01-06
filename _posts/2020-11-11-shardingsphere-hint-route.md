---
layout: post
title: ShardingSphere（2）- 强制路由
date: 2020-11-02
categories:
    - 分库分表
comments: true
permalink: shardingsphere-hint-route.html
---

从 SQL 执行效果而言，分库分表可以看作是一种路由机制，也就是说把 SQL 语句路由到目标数据库或数据表中并获取数据。

强制路由与一般的分库分表路由不同，它并没有使用任何的分片键和分片策略。我们知道通过解析 SQL 语句提取分片键，并设置分片策略进行分片是  ShardingSphere 对重写 JDBC  规范的实现方式。但是，如果我们没有分片键，是否就只能访问所有的数据库和数据表进行全路由呢？显然，这种处理方式也不大合适。**有时候，我们需要为 SQL 执行开一个“后门”，允许在没有分片键的情况下，同样可以在外部设置目标数据库和表，这就是强制路由的设计理念。**

在 ShardingSphere 中，通过 Hint 机制实现强制路由。在关系型数据库中，Hint 作为一种 SQL 补充语法扮演着非常重要的角色。它允许用户通过相关的语法影响 SQL  的执行方式，改变 SQL 的执行计划，从而对 SQL 进行特殊的优化。很多数据库工具也提供了特殊的 Hint 语法。以 MySQL  为例，比较典型的 Hint 使用方式之一就是对所有索引的强制执行和忽略机制。

MySQL 中的强制索引能够确保所需要执行的 SQL 语句只作用于所指定的索引上，我们可以通过 FORCE INDEX 这一 Hint 语法实现这一目标：

```
SELECT * FROM TABLE1 FORCE INDEX (FIELD1)
```

类似的，IGNORE INDEX 这一 Hint 语法使得原本设置在具体字段上的索引不被使用：

```
SELECT * FROM TABLE1 IGNORE INDEX (FIELD1, FIELD2)
```

对于分片字段非 SQL 决定、而由其他外置条件决定的场景，可使用 SQL Hint 灵活地注入分片字段。

ShardingSphere 使用 ThreadLocal 管理分片键值进行强制路由。 可以通过编程的方式向 HintManager 中添加分片值，**该分片值仅在当前线程内生效。**

Hint 的主要使用场景：

- 分片字段不存在 SQL 和数据库表结构中，而存在于外部业务逻辑。
- 强制在主库进行某些数据操作。

强制路由允许我们在程序运行过程中，强制设置某一次SQL的路由规则。

# 1. 使用

Hint 分片算法需要用户实现 `org.apache.shardingsphere.sharding.api.sharding.hint.HintShardingAlgorithm` 接口。 Apache ShardingSphere 在进行路由时，将会从 HintManager 中获取分片值进行路由操作。

>  网上对这一段的文字都比较简单，下面是我自己摸索的一些结果

```
public class UserIdHintShardingAlgorithm implements HintShardingAlgorithm<Long> {
    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, HintShardingValue<Long> shardingValue) {
        System.out.println("shardingValue=" + shardingValue);
        System.out.println("availableTargetNames=" + availableTargetNames);
        return availableTargetNames;
    }
}
```

修改配置

```
spring:
  shardingsphere:
    sharding:
      tables:
        user:
          actual-data-nodes: ds$->{0..1}.user
          database-strategy:
            hint:
              algorithm-class-name: com.github.edgar615.shaddingsphere.UserIdHintShardingAlgorithm
        user_password:
          actual-data-nodes: ds$->{0..1}.user_password
          database-strategy:
            hint:
              algorithm-class-name: com.github.edgar615.shaddingsphere.UserIdHintShardingAlgorithm
```

执行下面程序后发现ds0,ds1两个数据库中都有相同的数据

```
HintManager hintManager = HintManager.getInstance();
hintManager.addDatabaseShardingValue("user", "ds0");
insertUser(jdbcTemplate, i);
hintManager.close();
```

如果取消user_password表中的hit设置，user_password会进行hash切分

```
spring:
  shardingsphere:
    sharding:
      tables:
        user:
          actual-data-nodes: ds$->{0..1}.user
          database-strategy:
            hint:
              algorithm-class-name: com.github.edgar615.shaddingsphere.UserIdHintShardingAlgorithm
        user_password:
          actual-data-nodes: ds$->{0..1}.user_password
```

修改UserIdHintShardingAlgorithm，只返回ds1，此时ds0.user无数据，ds1.user有所有数据，user_password在ds0,ds1中有相同的数据

```
public class UserIdHintShardingAlgorithm implements HintShardingAlgorithm<Long> {
    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, HintShardingValue<Long> shardingValue) {
        return Collections.singleton("ds1");
    }
}
```

通过上面的测试，我们知道了HintShardingAlgorithm的返回值就决定了数据的分片。通过debug，我们发现在对user_password做insert操作的时候并没有调用HintShardingAlgorithm。

> 具体判断逻辑在ShardingStandardRoutingEngine中，后面再研究

如果我们增加一个强制逻辑，user_password都会路由到ds1上

```
hintManager.addDatabaseShardingValue("user_password", "ds0");
```

那么`hintManager.addDatabaseShardingValue("user_password", "ds0");`的作用是什么呢？

通过debug可以发现在`HintShardingStrategy`中调用了`HintShardingAlgorithm`的doSharding方法来指定分片。所以我们可以在

`HintShardingAlgorithm`中通过`HintShardingValue`的值来动态的指定分片。

下面通过代码测试一下，我们将user表中`user_id % 3`等于0的数据指定到ds1,其他到ds0，将user_password表中`user_id % 3`等于0的数据指定到ds0,其他到ds1。

```
public class UserIdHintShardingAlgorithm implements HintShardingAlgorithm<String> {
    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, HintShardingValue<String> shardingValue) {
        List<String> shardingResult = new ArrayList<>();
        for (String targetName : availableTargetNames) {
            for (String value : shardingValue.getValues()) {
                if (targetName.equals(value)) {
                    shardingResult.add(targetName);
                }
            }
        }
        return shardingResult;
    }
}
```

```
for (int i = 10; i < 20; i ++) {
	HintManager hintManager = HintManager.getInstance();
	if (i % 3 == 0) {
		hintManager.addDatabaseShardingValue("user", "ds1");
		hintManager.addDatabaseShardingValue("user_password", "ds0");
	} else {
		hintManager.addDatabaseShardingValue("user", "ds0");
		hintManager.addDatabaseShardingValue("user_password", "ds1");
	}
	insertUser(jdbcTemplate, i);
	hintManager.close();
}
```

# 2. 强制指定表

上面我们已经演示了如何强制指定库，强制指定表的实现与库基本类似

```
public class UserIdTableHintShardingAlgorithm implements HintShardingAlgorithm<Integer> {
    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, HintShardingValue<Integer> shardingValue) {
        List<String> shardingResult = new ArrayList<>();
        for (String targetName : availableTargetNames) {
            for (Integer value : shardingValue.getValues()) {
                shardingResult.add(targetName + value);
            }
        }
        return shardingResult;
    }
}
```

```
spring:
  shardingsphere:
    sharding:
      tables:
        user:
          actual-data-nodes: ds$->{0..1}.user
          database-strategy:
            hint:
              algorithm-class-name: com.github.edgar615.shaddingsphere.UserIdHintShardingAlgorithm
          table-strategy:
            hint:
              algorithm-class-name: com.github.edgar615.shaddingsphere.UserIdTableHintShardingAlgorithm
        user_password:
          actual-data-nodes: ds$->{0..1}.user_password
          database-strategy:
            hint:
              algorithm-class-name: com.github.edgar615.shaddingsphere.UserIdHintShardingAlgorithm
          table-strategy:
            hint:
              algorithm-class-name: com.github.edgar615.shaddingsphere.UserIdTableHintShardingAlgorithm
```

```
HintManager hintManager = HintManager.getInstance();
if (i % 3 == 0) {
	hintManager.addDatabaseShardingValue("user", "ds1");
	hintManager.addDatabaseShardingValue("user_password", "ds0");
	hintManager.addTableShardingValue("user", 0);
	hintManager.addTableShardingValue("user_password", 1);
} else {
	hintManager.addDatabaseShardingValue("user", "ds0");
	hintManager.addDatabaseShardingValue("user_password", "ds1");
	hintManager.addTableShardingValue("user", 1);
	hintManager.addTableShardingValue("user_password", 0);
}
insertUser(jdbcTemplate, i);
hintManager.close();
```

