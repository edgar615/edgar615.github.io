---
layout: post
title: JDK动态代理-一道Spring面试题
date: 2019-06-11
categories:
    - java
comments: true
permalink: spring-transaction-interview.html
---

前段时间看到了一个面试题，考察的是动态代理和事务的机制，正好在整理这部分知识，记录一下，后面可以问别人，哈哈

首先我们定义了一个接口`IMyService`和实现

<pre class="line-numbers "><code class="language-java">
@Service
public class MyServiceImpl implements IMyService {

  @Autowired
  private UserDao userDao;

  @Override
  @Transactional(propagation = Propagation.REQUIRED)
  public void method1() {
    User user = new User();
    user.setUsername("method1");
    userDao.insert(user);
    method2();
  }

  @Override
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void method2() {
    User user = new User();
    user.setUsername("method2");
    userDao.insert(user);
    throw new RuntimeException("oops");
  }
}
</code></pre>

问题：执行下面的方法后，数据库插入了几条记录？

<pre class="line-numbers "><code class="language-java">
@Service
public class TestManager {

  @Autowired
  private IMyService myService;

  public void test() {
    myService.method1();
  }
}
</code></pre>

答案是0，因为`TestManager`调用`myService.method1()`是通过动态代理生成的代理对象执行，但是`method1`调用`method2`时属于方法内部直接调用，不会调用代理类，`method2`方法上的`@Transactional`也就失效了，在`method1`抛出异常后会将两次插入的数据都回滚掉
