---
layout: post
title: Class.forName和ClassLoader的区别
date: 2019-12-09
categories:
    - java
comments: true
permalink: class-forName.html
---

对于类的加载有两种方式

- Class.forName("xxxx")
- getClassLoader().loadClass("xxxxx")

对于ClassLoader的方式参考[这里](https://edgar615.github.io/classloader.html)，下面我们的主要看看`Class.forName`

`Class.forName`有两个方法

- forName(String className)
- forName(String name, boolean initialize, ClassLoader loader)

如果参数initialize为true，则加载的类将会被初始化

这两个方法最终都会调用`forName0`方法

```
    /** Called after security check for system loader access checks have been made. */
    private static native Class<?> forName0(String name, boolean initialize, ClassLoader loader, Class<?> caller)
        throws ClassNotFoundException;
```

`forName`调用`forName0`方法时，第二个参数都是true,会对加载的类进行初始化，执行类中的静态代码块，以及对静态变量的赋值等操作。

测试一下

```
public class ClassForName {

    //静态代码块
    static {
        System.out.println("执行了静态代码块");
    }
    //静态变量
    private static String staticFiled = staticMethod();

    //赋值静态变量的静态方法
    public static String staticMethod(){
        System.out.println("执行了静态方法");
        return "给静态字段赋值了";
    }
}
```

```
    try {
      Class.forName("com.github.edgar615.ClassForName");
      System.out.println("###################");
      ClassLoader.getSystemClassLoader().loadClass("com.github.edgar615.ClassForName");
    } catch (ClassNotFoundException e) {
      e.printStackTrace();
    }
```

输出

```
执行了静态代码块
执行了静态方法
###################
```

根据运行结果得出`Class.forName`加载类是将类进了初始化，而ClassLoader的`loadClass`并没有对类进行初始化，只是把类加载到了虚拟机中。

# 参考资料

https://www.cnblogs.com/jimoer/p/9185662.html