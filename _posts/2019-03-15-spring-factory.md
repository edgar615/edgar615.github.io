---
layout: post
title: Spring Boot - 自动配置原理
date: 2019-03-15
categories:
    - Spring
comments: true
permalink: spring-factory.html
---

SpringFactoriesLoader 类似JAVA默认的SPI 机制，只不过以服务接口命名的文件是放在 META-INF/spring.factories 文件夹下。

```
public interface MyService {

  void handle();
}
public class MyServiceImpl1 implements MyService {
  @Override
  public void handle() {
    System.out.println("1");
  }
}
public class MyServiceImpl2 implements MyService {
  @Override
  public void handle() {
    System.out.println("2");
  }
}
```

在 META-INF/spring.factories中增加接口声明

```
com.github.edgar615.spring.factory.MyService=\
com.github.edgar615.spring.factory.impl.MyServiceImpl1,\
com.github.edgar615.spring.factory.impl.MyServiceImpl2
```

读取

```
List<MyService> classes = SpringFactoriesLoader.loadFactories(MyService.class, Application.class.getClassLoader());
```

