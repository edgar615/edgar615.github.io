---
layout: post
title: Plugin实现原理
date: 2021-05-03
categories:
    - 管理
comments: true
permalink: mybatis-plugin.html
---

Mybatis的执行过程

![](/assets/images/posts/mybatis-executor/mybatis-executor-2.png)

我们知道，SQL的具体执行逻辑是由Executor执行的，那么Executor在哪里创建的呢？

跟踪源码可以在Configuration中找到newExecutor方法

```
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  executorType = executorType == null ? defaultExecutorType : executorType;
  executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
  } else {
    executor = new SimpleExecutor(this, transaction);
  }
  if (cacheEnabled) {
    executor = new CachingExecutor(executor);
  }
  executor = (Executor) interceptorChain.pluginAll(executor);
  return executor;
}
```

我们可以看到在创建Executor对象后又通过`  executor = (Executor) interceptorChain.pluginAll(executor);`对它做了包装

interceptorChain.pluginAll的实现很简单，就是迭代拦截器，然后创建Executor的动态代理对象

```
private final List<Interceptor> interceptors = new ArrayList<>();

public Object pluginAll(Object target) {
  for (Interceptor interceptor : interceptors) {
    target = interceptor.plugin(target);
  }
  return target;
}
```

```
default Object plugin(Object target) {
  return Plugin.wrap(target, this);
}
```

```
public static Object wrap(Object target, Interceptor interceptor) {
  Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
  Class<?> type = target.getClass();
  Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
  if (interfaces.length > 0) {
    return Proxy.newProxyInstance(
        type.getClassLoader(),
        interfaces,
        new Plugin(target, interceptor, signatureMap));
  }
  return target;
}
```

Plugin实现了InvocationHandler接口，接下来看一下它的invoke实现

```
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
  	// 获取被拦截的方法签名
    Set<Method> methods = signatureMap.get(method.getDeclaringClass());
    // 如果是被拦截方法，执行代理对象
    if (methods != null && methods.contains(method)) {
      return interceptor.intercept(new Invocation(target, method, args));
    }
    // 直接调用原方法
    return method.invoke(target, args);
  } catch (Exception e) {
    throw ExceptionUtil.unwrapThrowable(e);
  }
}
```

同样的，我们知道Executor的执行实际上又是通过StatementHandler来执行的。在StatementHandler的创建方法中，我们同样可以看到，他们通过`statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);`做了包装。

```
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
  ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
  parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
  return parameterHandler;
}

public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
    ResultHandler resultHandler, BoundSql boundSql) {
  ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
  resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
  return resultSetHandler;
}

public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}
```

所以我们就知道了MyBatis的插件是基于拦截Executor，StatementHandler，ParameterHandler，ResultSetHandler这四大对象来实现的。需要注意的是，虽然我们可以拦截这四大对象，但是并不是这四大对象中的所有方法都能被拦截

![](/assets/images/posts/mybatis-plugin/mybatis-plugin-1.png)