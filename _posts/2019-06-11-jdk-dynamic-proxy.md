---
layout: post
title: JDK动态代理
date: 2019-06-11
categories:
    - java
comments: true
permalink: jdk-dynamic-proxy.html
---

最近为了准备公司内部的知识分享，重新整理了一下java动态代理的知识

# 1. 介绍
首先我们定义了一个接口`IMyService`和实现

<pre class="line-numbers "><code class="language-java">
public interface IMyService {

	void say(String message);
}

public class MyServiceImpl implements IMyService {

	public void say(String message) {
		System.out.println(message);
	}

}
</code></pre>

现在我们要在`say`方法执行前增加一个输出日志，该如何做呢？

最简单的方式是静态代理的实现，新定义一个实现类包装`MyServiceImpl`

<pre class="line-numbers "><code class="language-java">
public class MyServiceProxy implements IMyService {

	private final IMyService myService;
	
	public MyServiceProxy(IMyService myService) {
		this.myService = myService;
	}
	
	public void say(String message) {
		System.out.println("before");
		System.out.println(message);
	}

}
</code></pre>

上述静态代理的实现会有下问题:

- 代理类和被代理类实现了相同的接口，导致代码的重复，如果接口增加一个方法，那么除了被代理类需要实现这个方法外，代理类也要实现这个方法，增加了代码维护的难度。
- 代理对象只服务于一种类型的对象，如果要服务多类型的对象。势必要为每一种对象都进行代理，静态代理在程序规模稍大时就无法胜任了。

# 2. 动态代理
**动态代理**：在程序运行期间根据需要动态创建代理类及其实例来完成具体的功能。动态代理主要分为JDK动态代理和cglib动态代理两大类，对应cglib我只是粗略了解，这里仅介绍JDK动态代理

1.实现InvocationHandler接口，在invoke方法中实现代理逻辑

<pre class="line-numbers "><code class="language-java">
public class MyServiceHandler implements InvocationHandler {

	private Object target;
	
	public MyServiceHandler(Object target) {
		this.target = target;
	}
	
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		System.out.println("before");
		return method.invoke(target, args);
	}

}
</code></pre>

2.创建一个代理对象

通过Proxy的静态方法`newProxyInstance( ClassLoaderloader, Class[] interfaces, InvocationHandler h)`创建一个代理对象

- 第一个参数是指定代理类的类加载器（我们传入当前测试类的类加载器）
- 第二个参数是代理类需要实现的接口（我们传入被代理类实现的接口，这样生成的代理类和被代理类就实现了相同的接口）
- 第三个参数是invocation handler，用来处理方法的调用。这里传入我们自己实现的handler

<pre class="line-numbers "><code class="language-java">
IMyService myService = new MyServiceImpl();
myService.say("Hello");

IMyService myServiceProxy = (IMyService) Proxy.newProxyInstance(
		myService.getClass().getClassLoader(), myService.getClass().getInterfaces(),
		new MyServiceHandler(myService));
</code></pre>

3.测试
<pre class="line-numbers "><code class="language-java">
myServiceProxy.say("proxy");
</code></pre>

# 3. 原理
我们先看一下`Proxy.newProxyInstance(...)`方法

<pre class="line-numbers "><code class="language-java">
public static Object newProxyInstance(ClassLoader loader,
									  Class<?>[] interfaces,
									  InvocationHandler h)
	throws IllegalArgumentException
{
	// ...

	/*
	 * Look up or generate the designated proxy class.
	 */
	Class<?> cl = getProxyClass0(loader, intfs);
	
	// ...
}
</code></pre>

可以看到`Proxy.newProxyInstance(...)`是通过`getProxyClass0(loader, intfs)`方法得到的代理对象，而`getProxyClass0(loader, intfs)`是先从`proxyClassCache`中判断是否有对应的代理对象，如果没有就使用ProxyClassFactory创建代理类。
<pre class="line-numbers "><code class="language-java">
proxyClassCache.get(loader, interfaces);
</code></pre>

`proxyClassCache`的实现
```
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

`WeakCache`是一个二级缓存` key->(subKey->value)`具体的代码就不展开了，通过跟踪代码，我们可以发现如果缓存中不存在`subKey`就会通过`ProxyClassFactory`的`apply`方法创建一个代理对象
<pre class="line-numbers "><code class="language-java">
value = Objects.requireNonNull(valueFactory.apply(key, parameter));
</code></pre>

我们重点看一下`ProxyClassFactory`，首先定义了两个常量
<pre class="line-numbers "><code class="language-java">
// prefix for all proxy class names
// 所有代理类名字的前缀
private static final String proxyClassNamePrefix = "$Proxy";

// next number to use for generation of unique proxy class names
// 用于生成代理类名字的计数器
private static final AtomicLong nextUniqueNumber = new AtomicLong();
</code></pre>

判断代理类的包名

- 验证所有非公共的接口在同一个包内；公共的就无需处理
- 生成包名和类名的逻辑，包名默认是com.sun.proxy，类名默认是$Proxy 加上一个自增的整数值
- 如果被代理类是 non-public proxy interface ，则用和被代理类接口一样的包名

```
String proxyPkg = null;     // package to define proxy class in
int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

/*
 * Record the package of a non-public proxy interface so that the
 * proxy class will be defined in the same package.  Verify that
 * all non-public proxy interfaces are in the same package.
 */
for (Class<?> intf : interfaces) {
	int flags = intf.getModifiers();
	if (!Modifier.isPublic(flags)) {
		accessFlags = Modifier.FINAL;
		String name = intf.getName();
		int n = name.lastIndexOf('.');
		String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
		if (proxyPkg == null) {
			proxyPkg = pkg;
		} else if (!pkg.equals(proxyPkg)) {
			throw new IllegalArgumentException(
				"non-public interfaces from different packages");
		}
	}
}

if (proxyPkg == null) {
	// if no non-public proxy interfaces, use com.sun.proxy package
	proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
}
```
重点看这里

<pre class="line-numbers "><code class="language-java">
/*
 * Choose a name for the proxy class to generate.
 */
// 生成代理类的名字,如com.sun.proxy.$Proxy0.calss
long num = nextUniqueNumber.getAndIncrement();
String proxyName = proxyPkg + proxyClassNamePrefix + num;

/*
 * Generate the specified proxy class.
 */
// 生成代理类的字节码
byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
	proxyName, interfaces, accessFlags);
try {
// 把代理类加载到JVM中，至此动态代理过程基本结束了
	return defineClass0(loader, proxyName,
	​					proxyClassFile, 0, proxyClassFile.length);
} catch (ClassFormatError e) {
	/*
	 * A ClassFormatError here means that (barring bugs in the
	 * proxy class generation code) there was some other
	 * invalid aspect of the arguments supplied to the proxy
	 * class creation (such as virtual machine limitations
	 * exceeded).
	 */
	throw new IllegalArgumentException(e.toString());
}
</code></pre>

我们将`ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags);`生成的class文件保存到硬盘反编译
<pre class="line-numbers "><code class="language-java">
String path = "D:/$Proxy0.class";
byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0", MyServiceImpl.class.getInterfaces());
FileOutputStream out = null;

try {
​	out = new FileOutputStream(path);
​	out.write(classFile);
​	out.flush();
} catch (Exception e) {
​	e.printStackTrace();
} finally {
​	try {
​		out.close();
​	} catch (IOException e) {
​		e.printStackTrace();
​	}
}
</code></pre>
完整的反编译
<pre class="line-numbers "><code class="language-java">
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

import com.github.edgar615.microservice.IMyService;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements IMyService {
  private static Method m1;
  private static Method m2;
  private static Method m3;
  private static Method m0;

  public $Proxy0(InvocationHandler var1) throws  {
     super(var1);
  }

  public final boolean equals(Object var1) throws  {
​    try {
​      return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
​    } catch (RuntimeException | Error var3) {
​      throw var3;
​    } catch (Throwable var4) {
​      throw new UndeclaredThrowableException(var4);
​    }
  }

  public final String toString() throws  {
​    try {
​      return (String)super.h.invoke(this, m2, (Object[])null);
​    } catch (RuntimeException | Error var2) {
​      throw var2;
​    } catch (Throwable var3) {
​      throw new UndeclaredThrowableException(var3);
​    }
  }

  public final void say(String var1) throws  {
​    try {
​      super.h.invoke(this, m3, new Object[]{var1});
​    } catch (RuntimeException | Error var3) {
​      throw var3;
​    } catch (Throwable var4) {
​      throw new UndeclaredThrowableException(var4);
​    }
  }

  public final int hashCode() throws  {
​    try {
​      return (Integer)super.h.invoke(this, m0, (Object[])null);
​    } catch (RuntimeException | Error var2) {
​      throw var2;
​    } catch (Throwable var3) {
​      throw new UndeclaredThrowableException(var3);
​    }
  }

  static {
​    try {
​      m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
​      m2 = Class.forName("java.lang.Object").getMethod("toString");
​      m3 = Class.forName("com.github.edgar615.microservice.IMyService").getMethod("say", Class.forName("java.lang.String"));
​      m0 = Class.forName("java.lang.Object").getMethod("hashCode");
​    } catch (NoSuchMethodException var2) {
​      throw new NoSuchMethodError(var2.getMessage());
​    } catch (ClassNotFoundException var3) {
​      throw new NoClassDefFoundError(var3.getMessage());
​    }
  }
}
</code></pre>

首先静态代码块将初始化了代理中的所有方法，包括`equals`,`toString`等方法
<pre class="line-numbers "><code class="language-java">
static {
try {
  m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
  m2 = Class.forName("java.lang.Object").getMethod("toString");
  m3 = Class.forName("com.github.edgar615.microservice.IMyService").getMethod("say", Class.forName("java.lang.String"));
  m0 = Class.forName("java.lang.Object").getMethod("hashCode");
} catch (NoSuchMethodException var2) {
  throw new NoSuchMethodError(var2.getMessage());
} catch (ClassNotFoundException var3) {
  throw new NoClassDefFoundError(var3.getMessage());
}
}
</code></pre>

可以发现`add`,`equals`,`toString`方法调用的都是`super.h.invoke`方法
<pre class="line-numbers "><code class="language-java">
public final void say(String var1) throws  {
try {
  super.h.invoke(this, m3, new Object[]{var1});
} catch (RuntimeException | Error var3) {
  throw var3;
} catch (Throwable var4) {
  throw new UndeclaredThrowableException(var4);
}
}
</code></pre>

而`super.h`就是通过构造方法传入的`InvocationHandler`对象
<pre class="line-numbers "><code class="language-java">
public $Proxy0(InvocationHandler var1) throws  {
  super(var1);
}
</code></pre>

参考资料
