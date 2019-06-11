---
layout: post
title: JDK动态代理-拦截器的实现
date: 2019-06-11
categories:
    - java
comments: true
permalink: interceptor.html
---

拦截器是我们平时接触比较多的一个模式，像`struts`,`spring`,`mybatis`都提供了拦截器模式，不过实现各不相同。

在公司的项目中需要对数据权限进行拦截，以前通过`mybatis`可以通过`mybatis`自带的拦截器修改SQL来实现。而现在的项目是基于`JdbcTemplate`实现的JDBC组件，虽然可以使用AOP来实现拦截，但本着折腾的精神，我借鉴`mybatis`拦截器的思路，基于动态代理实现了自己的通用拦截器

首先拦截器主要做两件事

1. 判断如何处理方法调用
2. 执行必要的操作并返回结果.

我们可以把InvocationHandler的任务分成两个部分：

1. 决定一个给定的方法做什么
2. 执行第一步的决定。


首先我们定义了一个接口`Interceptor`用来决定一个给定的方法做什么

<pre class="line-numbers "><code class="language-java">
@FunctionalInterface
public interface Interceptor {

  /**
   * 执行拦截器
   *
   * @param invocation 被拦截的方法
   * @return 返回值
   * @throws Throwable 异常
   */
  Object intercept(Invocation invocation) throws Throwable;
}
</code></pre>

仿造mybatis，定义一个注解用来定义要拦截的方法

最简单的方式是静态代理的实现，新定义一个实现类包装`MyServiceImpl`

<pre class="line-numbers "><code class="language-java">
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Signature {

  /**
   * 类
   *
   * @return 类
   */
  Class<?> type();

  /**
   * 方法名
   *
   * @return 方法名
   */
  String method();

  /**
   * 参数类型
   *
   * @return 参数类型
   */
  Class<?>[] args();
}
</code></pre>


`Invocation`类用来执行代理类
<pre class="line-numbers "><code class="language-java">
public class Invocation {

  /**
   * 目标对象
   */
  private final Object target;
  /**
   * 方法
   */
  private final Method method;
  /**
   * 参数
   */
  private final Object[] args;

  private Invocation(Object target, Method method, Object[] args) {
    this.target = target;
    this.method = method;
    this.args = args;
  }

  /**
   * 创建Invocation对象
   *
   * @param target 目标对象
   * @param method 方法
   * @param args 参数
   * @return Invocation
   */
  public static Invocation create(Object target, Method method, Object[] args) {
    return new Invocation(target, method, args);
  }

  public Object target() {
    return target;
  }

  public Method method() {
    return method;
  }

  public Object[] args() {
    return args;
  }

  /**
   * 执行方法
   *
   * @return Object
   * @throws Throwable 反射异常
   */
  public Object proceed() throws Throwable {
    try {
      return method.invoke(target, args);
    } catch (InvocationTargetException e) {
      if (e.getCause() != null) {
        throw e.getCause();
      } else {
        throw e;
      }
    }

  }

}
</code></pre>

接下来我们看看最核心的代理类的实现，这个类的构造方法接收两个参数，

- 要被拦截的对象
- 拦截器

<pre class="line-numbers "><code class="language-java">
public class ObjectInterceptedJdkProxy implements InvocationHandler {

  private final Object target;

  private final Interceptor interceptor;

  private ObjectInterceptedJdkProxy(Object target, Interceptor interceptor) {
    this.target = target;
    this.interceptor = interceptor;
  }

  public static Object create(Object target, Interceptor interceptor) {
    Class[] interfaces = ReflectUtils.extractAllInterfaces(target);
    if (interfaces.length == 0) {
      return target;
    }
    // 封装的一个创建代理类的工具类
    return ObjectProxyBuilder
        .createProxy(new ObjectInterceptedJdkProxy(target, interceptor), interfaces);
  }

}
</code></pre>

增加拦截器的实现方法
```
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Invocation invocation = Invocation.create(target, method, args);
    // 通过checkProxy方法判断拦截器是不是对应方法的拦截器
    if (checkProxy(invocation)) {
      return interceptor.intercept(invocation);
    }
    return invocation.proceed();
  }

  public boolean checkProxy(Invocation invocation) {
    Signature signature = interceptor.getClass().getAnnotation(Signature.class);
    if (!ReflectUtils.isSubClassOrInterfaceOf(target.getClass(), signature.type())) {
      return false;
    }
    if (invocation.method().isDefault()) {
      return false;
    }
    String methodName = invocation.method().getName();
    if (!methodName.equals(signature.method())) {
      return false;
    }
    if (invocation.args() == null && signature.args().length == 0) {
      return true;
    } else if (signature.args().length != invocation.args().length) {
      return false;
    }
    for (int i = 0; i < signature.args().length; i++) {
      Object invocationArg = invocation.args()[i];
      if (invocationArg != null && !ReflectUtils
          .isSubClassOrInterfaceOf(invocationArg.getClass(), signature.args()[i])) {
        return false;
      }
    }
    return true;
  }
```

至此一个简单的拦截器就实现了，我就不贴测试代码了

