---
layout: post
title: Mapper实现原理
date: 2021-05-01
categories:
    - 管理
comments: true
permalink: mybatis-mapper.html
---

在 MyBatis 中，实现 CustomerMapper 接口与 CustomerMapper.xml 配置文件映射功能的是 binding 模块，其中涉及的核心类如下图所示：

![](/assets/images/posts/mybatis-mapper/mybatis-mapper-1.png)

# 1. MapperRegistry

MapperRegistry 是 MyBatis 初始化过程中构造的一个对象，主要作用就是统一维护 Mapper 接口以及这些 Mapper 的代理对象工厂。

下面我们先来看 MapperRegistry 中的核心字段。

```
// 指向 MyBatis 全局唯一的 Configuration 对象，其中维护了解析之后的全部 MyBatis 配置信息。
private final Configuration config;
// 维护了所有解析到的 Mapper 接口以及 MapperProxyFactory 工厂对象之间的映射关系。
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
```

在 MyBatis 初始化时，会读取全部 Mapper.xml 配置文件，还会扫描全部 Mapper 接口中的注解信息，之后会调用 MapperRegistry.addMapper() 方法填充 knownMappers 集合。在 addMapper() 方法填充 knownMappers 集合之前，MapperRegistry 会先保证传入的 type 参数是一个接口且 knownMappers 集合没有加载过 type 类型，然后才会创建相应的 MapperProxyFactory 工厂并记录到 knownMappers 集合中。

在我们使用 CustomerMapper.find() 方法执行数据库查询的时候，MyBatis 会先从MapperRegistry 中获取 CustomerMapper 接口的代理对象，这里就使用到 MapperRegistry.getMapper()方法，它会拿到前面创建的 MapperProxyFactory 工厂对象，并调用其 newInstance() 方法创建 Mapper 接口的代理对象。

```
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

# 2. MapperProxyFactory

MapperProxyFactory 的核心功能就是创建 Mapper 接口的代理对象。在 MapperRegistry 中会依赖 MapperProxyFactory 的 newInstance() 方法创建代理对象，底层则是通过 JDK 动态代理的方式生成代理对象的，如下代码所示，这里使用的 InvocationHandler 实现是 MapperProxy。

```
protected T newInstance(MapperProxy<T> mapperProxy) {
  //  创建实现了mapperInterface接口的动态代理对象，这里使用的InvocationHandler 实现是MapperProxy
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}
```

# 3. MapperProxy

通过分析 MapperProxyFactory 这个工厂类，我们可以清晰地看到MapperProxy 是生成 Mapper 接口代理对象的关键，它实现了 InvocationHandler 接口。

```
private static final int ALLOWED_MODES = MethodHandles.Lookup.PRIVATE | MethodHandles.Lookup.PROTECTED
    | MethodHandles.Lookup.PACKAGE | MethodHandles.Lookup.PUBLIC;
// 针对 JDK 8 中的特殊处理，该字段指向了 MethodHandles.Lookup 的构造方法。
private static final Constructor<Lookup> lookupConstructor;
// 除了 JDK 8 之外的其他 JDK 版本会使用该字段，该字段指向 MethodHandles.privateLookupIn() 方法。
private static final Method privateLookupInMethod;
// 记录了当前 MapperProxy 关联的 SqlSession 对象。在与当前 MapperProxy 关联的代理对象中，会用该 SqlSession 访问数据库。
private final SqlSession sqlSession;
// Mapper 接口类型，也是当前 MapperProxy 关联的代理对象实现的接口类型。
private final Class<T> mapperInterface;
// 用于缓存 MapperMethodInvoker 对象的集合。methodCache 中的 key 是 Mapper 接口中的方法，value 是该方法对应的 MapperMethodInvoker 对象。
private final Map<Method, MapperMethodInvoker> methodCache;
```

**MapperProxy.invoke() 方法是代理对象执行的入口**，其中会拦截所有非 Object 方法，针对每个被拦截的方法，都会调用 cachedInvoker() 方法获取对应的 MapperMethod 对象，并调用其 invoke() 方法执行代理逻辑以及目标方法。

```
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    } else {
      return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
}
```

在 cachedInvoker() 方法中，首先会查询 methodCache 缓存，如果查询的方法为 default 方法，则会根据当前使用的 JDK 版本，获取对应的 MethodHandle 并封装成 DefaultMethodInvoker 对象写入缓存；如果查询的方法是非  default 方法，则创建 PlainMethodInvoker 对象写入缓存。

```
private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
  try {
    // 尝试从methodCache缓存中查询方法对应的MapperMethodInvoker
    MapperMethodInvoker invoker = methodCache.get(method);
    if (invoker != null) {
      return invoker;
    }

	// 如果方法在缓存中没有对应的MapperMethodInvoker，则进行创建
    return methodCache.computeIfAbsent(method, m -> {
      if (m.isDefault()) {
      // 针对default方法的处理
        try {
          if (privateLookupInMethod == null) {
          	// 在JDK 8中使用的是lookupConstructor字段
            return new DefaultMethodInvoker(getMethodHandleJava8(method));
          } else {
          	// 在JDK 9中使用privateLookupInMethod字段
            return new DefaultMethodInvoker(getMethodHandleJava9(method));
          }
        } catch (IllegalAccessException | InstantiationException | InvocationTargetException
            | NoSuchMethodException e) {
          throw new RuntimeException(e);
        }
      } else {
       // 对于其他方法，会创建MapperMethod并使用PlainMethodInvoker封装
        return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
      }
    });
  } catch (RuntimeException re) {
    Throwable cause = re.getCause();
    throw cause == null ? re : cause;
  }
}
```

在 DefaultMethodInvoker.invoke() 方法中，会通过底层维护的 MethodHandle 完成方法调用，核心实现如下：

```
@Override
public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
  // 首先将MethodHandle绑定到一个实例对象上，然后调用invokeWithArguments()方法执行目标方法
  return methodHandle.bindTo(proxy).invokeWithArguments(args);
}
```

在 PlainMethodInvoker.invoke() 方法中，会通过底层维护的 MapperMethod 完成方法调用，其核心实现如下：

```
@Override
public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
  // 直接执行MapperMethod.execute()方法完成方法调用
  return mapperMethod.execute(sqlSession, args);
}
```

# 4. MapperMethod

通过对 MapperProxy 的分析我们知道，**MapperMethod 是最终执行 SQL 语句的地方，同时也记录了 Mapper 接口中的对应方法**，其核心字段也围绕这两方面的内容展开。

## 4.1. SqlCommand

**MapperMethod 的第一个核心字段是 command（SqlCommand 类型），其中维护了关联 SQL 语句的相关信息**。在 MapperMethod$SqlCommand 这个内部类中，通过 name 字段记录了关联 SQL 语句的唯一标识，通过 type  字段（SqlCommandType 类型）维护了 SQL 语句的操作类型，这里 SQL 语句的操作类型分为  INSERT、UPDATE、DELETE、SELECT 和 FLUSH 五种。

```
public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
  // 获取Mapper接口中对应的方法名称
  final String methodName = method.getName();
  // 获取Mapper接口的类型
  final Class<?> declaringClass = method.getDeclaringClass();
  // 将Mapper接口名称和方法名称拼接起来作为SQL语句唯一标识，
  // 到Configuration这个全局配置对象中查找SQL语句
  // MappedStatement对象就是Mapper.xml配置文件中一条SQL语句解析之后得到的对象
  MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,
      configuration);
  if (ms == null) {
  	// 针对@Flush注解的处理
    if (method.getAnnotation(Flush.class) != null) {
      name = null;
      type = SqlCommandType.FLUSH;
    } else {
      // 没有@Flush注解，会抛出异常
      throw new BindingException("Invalid bound statement (not found): "
          + mapperInterface.getName() + "." + methodName);
    }
  } else {
    // 记录SQL语句唯一标识
    name = ms.getId();
    // 记录SQL语句的操作类型
    type = ms.getSqlCommandType();
    if (type == SqlCommandType.UNKNOWN) {
      throw new BindingException("Unknown execution method for: " + name);
    }
  }
}
```

这里调用的 resolveMappedStatement() 方法不仅会尝试根据 SQL 语句的唯一标识从 Configuration  全局配置对象中查找关联的 MappedStatement 对象，还会尝试顺着 Mapper 接口的继承树进行查找，直至查找成功为止。

```
 private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName,
      Class<?> declaringClass, Configuration configuration) {
      // 将Mapper接口名称和方法名称拼接起来作为SQL语句唯一标识
    String statementId = mapperInterface.getName() + "." + methodName;
    // 检测Configuration中是否包含相应的MappedStatement对象
    if (configuration.hasStatement(statementId)) {
    	// 如果方法就定义在当前接口中，则证明没有对应的SQL语句，返回null
      return configuration.getMappedStatement(statementId);
    } else if (mapperInterface.equals(declaringClass)) {
      return null;
    }
    // 如果当前检查的Mapper接口(mapperInterface)中不是定义该方法的接口(declaringClass)，
    // 则会从mapperInterface开始，沿着继承关系向上查找递归每个接口，
    // 查找该方法对应的MappedStatement对象
    for (Class<?> superInterface : mapperInterface.getInterfaces()) {
      if (declaringClass.isAssignableFrom(superInterface)) {
        MappedStatement ms = resolveMappedStatement(superInterface, methodName,
            declaringClass, configuration);
        if (ms != null) {
          return ms;
        }
      }
    }
    return null;
  }
}
```

## 4.2. MethodSignature

**MapperMethod 的第二个核心字段是 method 字段（MethodSignature 类型），其中维护了 Mapper 接口中方法的相关信息。**

```
// Mapper 接口方法返回值的相关信息
// 用于表示方法返回值是否为 Collection 集合或数组、Map 集合、void、Cursor、Optional 类型。
private final boolean returnsMany;
private final boolean returnsMap;
private final boolean returnsVoid;
private final boolean returnsCursor;
private final boolean returnsOptional;
// 方法返回值的具体类型。
private final Class<?> returnType;
// 如果方法的返回值为 Map 集合，则通过 mapKey 字段记录了作为 key 的列名。mapKey 字段的值是通过解析方法上的 @MapKey 注解得到的。
private final String mapKey;

// Mapper 接口方法的参数列表相关
// 记录了 Mapper 接口方法的参数列表中 ResultHandler 类型参数的位置。
private final Integer resultHandlerIndex;
// 记录了 Mapper 接口方法的参数列表中 RowBounds 类型参数的位置。
private final Integer rowBoundsIndex;
// 用来解析方法参数列表的工具类。
private final ParamNameResolver paramNameResolver;
```

在上述字段中，需要着重讲解的是 **ParamNameResolver** 这个解析方法参数列表的工具类。

在 ParamNameResolver 中有一个 names 字段（SortedMap<Integer,  String>类型）记录了各个参数在参数列表中的位置以及参数名称，其中 key 是参数在参数列表中的位置索引，value  为参数的名称。我们可以通过 @Param 注解指定一个参数名称，如果没有特别指定，则默认使用参数列表中的变量名称作为其名称，这与  ParamNameResolver 的 useActualParamName 字段相关。useActualParamName 是一个全局配置。

```
public static final String GENERIC_NAME_PREFIX = "param";

private final boolean useActualParamName;

/**
 * <p>
 * The key is the index and the value is the name of the parameter.<br />
 * The name is obtained from {@link Param} if specified. When {@link Param} is not specified,
 * the parameter index is used. Note that this index could be different from the actual index
 * when the method has special parameters (i.e. {@link RowBounds} or {@link ResultHandler}).
 * </p>
 * <ul>
 * <li>aMethod(@Param("M") int a, @Param("N") int b) -&gt; {{0, "M"}, {1, "N"}}</li>
 * <li>aMethod(int a, int b) -&gt; {{0, "0"}, {1, "1"}}</li>
 * <li>aMethod(int a, RowBounds rb, int b) -&gt; {{0, "0"}, {2, "1"}}</li>
 * </ul>
 */
private final SortedMap<Integer, String> names;

private boolean hasParamAnnotation;
```

如果我们将 useActualParamName 配置为 false，ParamNameResolver  会使用参数的下标索引作为其名称。另外，names 集合会跳过 RowBounds 类型以及 ResultHandler  类型的参数，如果使用下标索引作为参数名称，在 names 集合中就会出现 KV 不一致的场景。上面的注释已经做了说明。

完成 names 集合的初始化之后，我们再来看如何从 names 集合中查询参数名称，该部分逻辑在  ParamNameResolver.getNamedParams() 方法，其中会将 Mapper 接口方法的实参与 names  集合中记录的参数名称相关联，其核心逻辑如下：

完成 names 集合的初始化之后，我们再来看如何从 names 集合中查询参数名称，该部分逻辑在  ParamNameResolver.getNamedParams() 方法，其中会将 Mapper 接口方法的实参与 names  集合中记录的参数名称相关联，其核心逻辑如下：

复制代码

```
public Object getNamedParams(Object[] args) {
    // 获取方法中非特殊类型(RowBounds类型和ResultHandler类型)的参数个数
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
        return null; // 方法没有非特殊类型参数，返回null即可
    } else if (!hasParamAnnotation && paramCount == 1) {
        // 方法参数列表中没有使用@Param注解，且只有一个非特殊类型参数
        Object value = args[names.firstKey()];
        return wrapToMapIfCollection(value, useActualParamName ? names.get(0) : null);
    } else {
        // 处理存在@Param注解或是存在多个非特殊类型参数的场景
        // param集合用于记录了参数名称与实参之间的映射关系
        // 这里的ParamMap继承了HashMap，与HashMap的唯一不同是：
        // 向ParamMap中添加已经存在的key时，会直接抛出异常，而不是覆盖原有的Key
        final Map<String, Object> param = new ParamMap<>();
        int i = 0;
        for (Map.Entry<Integer, String> entry : names.entrySet()) {
            // 将参数名称与实参的映射保存到param集合中
            param.put(entry.getValue(), args[entry.getKey()]);
            // 同时，为参数创建"param+索引"格式的默认参数名称，具体格式为：param1, param2等，
            // 将"param+索引"的默认参数名称与实参的映射关系也保存到param集合中
            final String genericParamName = GENERIC_NAME_PREFIX + (i + 1);
            if (!names.containsValue(genericParamName)) {
                param.put(genericParamName, args[entry.getKey()]);
            }
            i++;
        }
        return param;
    }
}
```

了解了 ParamNameResolver 的核心功能之后，我们回到 MethodSignature  继续分析，在其构造函数中会解析方法中的返回值、参数列表等信息，并初始化前面介绍的核心字段，这里也会使用到前面介绍的  ParamNameResolver 工具类。

```
public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
  Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
  if (resolvedReturnType instanceof Class<?>) {
    this.returnType = (Class<?>) resolvedReturnType;
  } else if (resolvedReturnType instanceof ParameterizedType) {
    this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
  } else {
    this.returnType = method.getReturnType();
  }
  // 根据返回值类型，初始化returnsVoid、returnsMany、returnsCursor、
  // returnsMap、returnsOptional这五个与方法返回值类型相关的字段
  this.returnsVoid = void.class.equals(this.returnType);
  this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
  this.returnsCursor = Cursor.class.equals(this.returnType);
  this.returnsOptional = Optional.class.equals(this.returnType);
  this.mapKey = getMapKey(method);
  this.returnsMap = this.mapKey != null;
  // 解析方法中RowBounds类型参数以及ResultHandler类型参数的下标索引位置，
  // 初始化rowBoundsIndex和resultHandlerIndex字段
  this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
  this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
  // 创建ParamNameResolver工具对象，在创建ParamNameResolver对象的时候，
  // 会解析方法的参数列表信息
  this.paramNameResolver = new ParamNameResolver(configuration, method);
}
```

在初始化过程中，我们看到会调用 getUniqueParamIndex() 方法查找目标类型参数的下标索引位置，其核心原理就是遍历方法的参数列表，逐个匹配参数的类型是否为目标类型，如果匹配成功，则会返回当前参数的下标索引。

## 4.3. execute() 方法

execute() 方法是 MapperMethod 中最核心的方法之一。**execute() 方法会根据要执行的 SQL 语句的具体类型执行 SqlSession 的相应方法完成数据库操作。**

```
public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  switch (command.getType()) {
    case INSERT: {
      // 通过ParamNameResolver.getNamedParams()方法将方法的实参与参数的名称关联起来
      Object param = method.convertArgsToSqlCommandParam(args);
      // 通过SqlSession.insert()方法执行INSERT语句，
      // 在rowCountResult()方法中，会根据方法的返回值类型对结果进行转换
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    case SELECT:
      if (method.returnsVoid() && method.hasResultHandler()) {
      	// 如果方法返回值为void，且参数中包含了ResultHandler类型的实参，则查询的结果集将会由ResultHandler对象进行处理
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
      	// executeForMany()方法处理返回值为集合或数组的场景
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
        if (method.returnsOptional()
            && (result == null || !method.getReturnType().equals(result.getClass()))) {
          result = Optional.ofNullable(result);
        }
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName()
        + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
  }
  return result;
}
```

**在 execute() 方法中，对于 INSERT、UPDATE、DELETE 三类 SQL 语句的返回结果，都会通过 rowCountResult() 方法处理**。我们知道，上述三种类型的 SQL 语句的执行结果是一个数字，多数场景中代表了 SQL 语句影响的数据行数（注意，这个返回值的具体含义根据 MySQL  的配置有所变化），rowCountResult() 方法会将这个 int 值转换成 Mapper 接口方法的返回值，具体规则如下：

- Mapper 方法返回值为 void，则忽略 SQL 语句的 int 返回值，直接返回 null；
- Mapper 方法返回值为 int 或 Integer 类型，则将 SQL 语句返回的 int 值直接返回；
- Mapper 方法返回值为 long 或 Long 类型，则将 SQL 语句返回的 int 值转换成 long 类型之后返回；
- Mapper 方法返回值为 boolean 或 Boolean 类型，则将 SQL 语句返回的 int 值与 0 比较大小，并将比较结果返回。

```
private Object rowCountResult(int rowCount) {
  final Object result;
  if (method.returnsVoid()) {
    result = null;
  } else if (Integer.class.equals(method.getReturnType()) || Integer.TYPE.equals(method.getReturnType())) {
    result = rowCount;
  } else if (Long.class.equals(method.getReturnType()) || Long.TYPE.equals(method.getReturnType())) {
    result = (long) rowCount;
  } else if (Boolean.class.equals(method.getReturnType()) || Boolean.TYPE.equals(method.getReturnType())) {
    result = rowCount > 0;
  } else {
    throw new BindingException("Mapper method '" + command.getName() + "' has an unsupported return type: " + method.getReturnType());
  }
  return result;
}
```

接下来看 execute() 方法**针对 SELECT 语句查询到的结果集的处理**。

- 如果在方法参数列表中有 ResultHandler 类型的参数存在，则会使用  executeWithResultHandler() 方法完成查询，底层依赖的是 SqlSession.select()  方法，结果集将会交由传入的 ResultHandler 对象进行处理。
- 如果方法返回值为集合类型或是数组类型，则会调用 executeForMany() 方法，底层依赖 SqlSession.selectList() 方法进行查询，并将得到的 List 转换成目标集合类型。
- 如果方法返回值为 Map 类型，则会调用 executeForMap() 方法，底层依赖 SqlSession.selectMap() 方法完成查询，并将结果集映射成 Map 集合。
- 针对 Cursor 以及 Optional返回值的处理，也是依赖的 SqlSession 的相关方法完成查询的，这里不再展开。

# 5. 参考资料

《深入剖析 MyBatis 核心原理》