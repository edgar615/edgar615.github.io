---
layout: post
title: Executor实现原理
date: 2021-05-02
categories:
    - 管理
comments: true
permalink: mybatis-executor.html
---

在Mapper的实现原理中我们了解到mapper的方法最终是通过MapperMethod#execute执行的。他调用了SqlSession的一系列方法。而SqlSession实际上是通过Executor来真正用来执行Sql语句。

```
@Override
public int update(String statement, Object parameter) {
  try {
    dirty = true;
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.update(ms, wrapCollection(parameter));
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

我们先来看一下Executor的类图关系：

![](/assets/images/posts/mybatis-executor/mybatis-executor-1.png)

这里面其实用到了模板方法模式。顶层接口Executor定义了一系列规范，而在抽象类BaseExecutor中将一些固定不变的方法进行了封装，并定义了一下抽象方法待子类实现。

**BaseExecutor**是一个抽象类，除了下面的四个方法是抽象方法，其余所有方法都是一些如获取缓存，事务提交，获取事务等公共操作，所以就直接被实现了。

- doFlushStatements()：刷新Statement对象
- doQuery()：执行查询语句并返回List
- doQueryCursor()：执行查询语句并返回Cursor对象
- doUpdate()：执行更新操作

配置文件中的setting标签内有一个属性defaultExecutorType，有三种执行类型：SIMPLE，REUSE，BATCH。如果不配置则默认就是SIMPLE。这三种类型就是对应了BaseExecutor的三个子类：**SimpleExecutor**，**ReuseExecutor**和**BatchExecutor**。

我们通过SimpleExecutor看一下update的实现

```
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  clearLocalCache();
  // 调用子类的doUpdate实现
  return doUpdate(ms, parameter);
}
```

```
@Override
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
  Statement stmt = null;
  try {
  	// 获取配置文件
    Configuration configuration = ms.getConfiguration();
    // 获取StatementHandler对象
    StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
    // 初始化Statement，设置参数
    stmt = prepareStatement(handler, ms.getStatementLog());
    // 调用StatementHandler执行更新
    return handler.update(stmt);
  } finally {
  	// 关闭Statement
    closeStatement(stmt);
  }
}
```

**prepareStatement**

```
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  Connection connection = getConnection(statementLog);
  stmt = handler.prepare(connection, transaction.getTimeout());
  // 参数设置
  handler.parameterize(stmt);
  return stmt;
}
```

看一下handler#update方法

```
@Override
public int update(Statement statement) throws SQLException {
  // SQL
  String sql = boundSql.getSql();
  // 参数
  Object parameterObject = boundSql.getParameterObject();
  // KeyGenerator用于返回数据库生成的自增主键值
  KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
  int rows;
  if (keyGenerator instanceof Jdbc3KeyGenerator) {
  	// Jdbc3KeyGenerator：用于处理数据库支持自增主键的情况，如MySQL的auto_increment。
  	// 执行SQL
    statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
    rows = statement.getUpdateCount();
    keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
  } else if (keyGenerator instanceof SelectKeyGenerator) {
  	// SelectKeyGenerator：用于处理数据库不支持自增主键的情况，比如Oracle的sequence序列。
    statement.execute(sql);
    rows = statement.getUpdateCount();
    keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
  } else {
  	// NoKeyGenerator：空实现，不需要处理主键。
    statement.execute(sql);
    rows = statement.getUpdateCount();
  }
  return rows;
}
```

doQuery方法类似

**ReuseExecutor**相比较于SimpleExecutor做了一点优化，那就是将Statement对象进行了缓存处理，不会每次都创建Statement对象，这样做的话减少了SQL预编译和创建对象的开销。

```
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  BoundSql boundSql = handler.getBoundSql();
  String sql = boundSql.getSql();
  if (hasStatementFor(sql)) {
    stmt = getStatement(sql);
    applyTransactionTimeout(stmt);
  } else {
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 放入map
    putStatement(sql, stmt);
  }
  handler.parameterize(stmt);
  return stmt;
}

private boolean hasStatementFor(String sql) {
  try {
  	// 从map中读取
    Statement statement = statementMap.get(sql);
    return statement != null && !statement.getConnection().isClosed();
  } catch (SQLException e) {
    return false;
  }
}
```

因为**ReuseExecutor**对Statement对象进行了缓存处理，所以在SQL执行完后，并没有调用close方法关闭Statement

```
@Override
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
  Configuration configuration = ms.getConfiguration();
  StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
  Statement stmt = prepareStatement(handler, ms.getStatementLog());
  return handler.update(stmt);
}
```

总结一下下Mybatis的执行过程

![](/assets/images/posts/mybatis-executor/mybatis-executor-2.png)