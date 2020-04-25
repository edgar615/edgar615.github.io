---
layout: post
title: ClassLoader
date: 2019-07-04
categories:
    - java
comments: true
permalink: classloader.html
---

# ClassLoader
一个Java程序不是管是CS还是BS应用，都是由若干个.class文件组织而成的一个完整的Java应用程序，当程序在运行时，即会调用该程序的一个入口函数来调用系统的相关功能，而这些功能都被封装在不同的class文件当中，所以经常要从这个class文件中要调用另外一个class文件中的方法。
如果另外一个文件不存在的，则会引发系统异常。**而程序在启动的时候，并不会一次性加载程序所要用的所有class文件，而是根据程序的需要，通过Java的类加载机制（ClassLoader）来动态加载某个class文件到内存当中的**，从而只有class文件被载入到了内存之后，才能被其它class所引用。所以ClassLoader就是用来动态加载class文件到内存当中用的。

Java默认提供的三个ClassLoader，分别是：

- 启动类加载器 (BootStrap ClassLoader)
- 扩展类加载器(Extension ClassLoader)
- 应用程序类加载器(Application ClassLoader) 

## BootStrap ClassLoader
启动类加载器，是Java类加载层次中最顶层的类加载器，负责加载JDK中的核心类库，如：rt.jar、resources.jar、charsets.jar等，可以通过属性`sun.boot.class.path`获得类加载器从哪些地方加载了相关的jar或class文件：
```
System.out.println(System.getProperty("sun.boot.class.path"));
```
输出
```
D:\dev\jdk1.8.0_181\jre\lib\resources.jar;D:\dev\jdk1.8.0_181\jre\lib\rt.jar;D:\dev\jdk1.8.0_181\jre\lib\sunrsasign.jar;D:\dev\jdk1.8.0_181\jre\lib\jsse.jar;D:\dev\jdk1.8.0_181\jre\lib\jce.jar;D:\dev\jdk1.8.0_181\jre\lib\charsets.jar;D:\dev\jdk1.8.0_181\jre\lib\jfr.jar;D:\dev\jdk1.8.0_181\jre\classes
```
可以看到都是JAVA_HOME/jre/lib下面的jar包和class

> 我们还可以通过下面方式获取
> ```
>   URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
>   for (int i = 0; i < urls.length; i++) {
>     System.out.println(urls[i].toExternalForm());
>   }
> ```

## Extension ClassLoader
扩展类加载器，负责加载Java的扩展类库，我们可以通过属性`java.ext.dirs`获取类加载器从哪些地方加载了相关的jar或class文件
```
System.out.println(System.getProperty("java.ext.dirs"));
```
输出
```
D:\dev\jdk1.8.0_181\jre\lib\ext;C:\Windows\Sun\Java\lib\ext
```
可以看到都是JAVA_HOME/jre/lib/ext/目录下的类，只要我们将jar包放置这个位置，就会被虚拟机加载

## App ClassLoader
系统类加载器，负责加载应用程序classpath目录下的所有jar和class文件

除了Java默认提供的三个ClassLoader之外，用户还可以根据需要定义自已的ClassLoader，而这些自定义的ClassLoader都必须继承自java.lang.ClassLoader类，也包括Java提供的另外二个ClassLoader（Extension ClassLoader和App ClassLoader）在内，但是Bootstrap ClassLoader不继承自ClassLoader，因为它不是一个普通的Java类，底层由C++编写，已嵌入到了JVM内核当中，当JVM启动后，Bootstrap ClassLoader也随着启动，负责加载完核心类库后，并构造Extension ClassLoader和App ClassLoader类加载器。

这几个ClassLoader的关系如图所示

![](/assets/images/posts/classloader/classloader-1.png)

# 双亲委派模型
ClassLoader使用的是双亲委派模型来搜索类的，每个ClassLoader实例都有一个父类加载器的引用（不是继承的关系，是一个包含的关系），虚拟机内置的类加载器（Bootstrap ClassLoader）本身没有父类加载器，但可以用作其它ClassLoader实例的的父类加载器。当一个ClassLoader实例需要加载某个类时，它会试图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的，首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader 进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常。否则将这个找到的类生成一个类的定义，并将它加载到内存当中，最后返回这个类在内存中的Class实例对象。

![](/assets/images/posts/classloader/classloader-2.png)

## 为什么要使用双亲委派模型
### 避免重复加载
当父亲已经加载了该类的时候，就没有必要子ClassLoader再加载一次
### 安全因素
如果不使用这种委派模式，那我们就可以随时使用自定义的String来动态替代java核心api中定义的类型，这样会存在非常大的安全隐患，而双亲委托的方式，就可以避免这种情况，因为String已经在启动时就被引导类加载器（Bootstrap ClassLoader）加载，所以用户自定义的ClassLoader永远也无法加载一个自己写的String，除非你改变JDK中ClassLoader搜索类的默认算法。

## 源码实现

双亲委派模型的实现非常简单，实现双亲委派的代码在java.lang.ClassLoader的loadClass()方法之中，如下面的代码所示：

```java
protected Class<?> loadClass(String name, boolean resolve)
	throws ClassNotFoundException
{
	synchronized (getClassLoadingLock(name)) {
		// 检查类是否已加载
		Class<?> c = findLoadedClass(name);
		if (c == null) {
			long t0 = System.nanoTime();
			try {
				if (parent != null) {
					c = parent.loadClass(name, false);
				} else {
					c = findBootstrapClassOrNull(name);
				}
			} catch (ClassNotFoundException e) {
				// 说明父类加载器无法完成加载请求
			}

			if (c == null) {
				// 在父类加载器无法完成加载请求时，使用本类加载器加载
				long t1 = System.nanoTime();
				c = findClass(name);

				// this is the defining class loader; record the stats
				sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
				sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
				sun.misc.PerfCounter.getFindClasses().increment();
			}
		}
		if (resolve) {
			resolveClass(c);
		}
		return c;
	}
}
```

## JVM在搜索类的时候，如何判断两个class相同
JVM在判定两个class是否相同时，不仅要判断两个类名是否相同，而且要判断是否由同一个类加载器实例加载的。只有两者同时满足的情况下，JVM才认为这两个class是相同的。就算两个class是同一份class字节码，如果被两个不同的ClassLoader实例所加载，JVM也会认为它们是两个不同class。

**在一个单虚拟机环境下，标识一个类有两个因素：class的全路径名、该类的ClassLoader。**

```
    // 因为Bootstrap ClassLoader不是一个普通的Java类，这里显示null
    System.out.println(String.class.getClassLoader());
    // sun.misc.Launcher$AppClassLoader@18b4aac2
    System.out.println(ClassLoaderMain.class.getClassLoader());
    // sun.misc.Launcher$ExtClassLoader@6d6f6e28
    System.out.println(ClassLoaderMain.class.getClassLoader().getParent());
    // null
    System.out.println(ClassLoaderMain.class.getClassLoader().getParent().getParent());
```

# 线程上下文加载器
在Thread类里面有一个属性`private ClassLoader contextClassLoader;`我们称为线程上下文加载器

类加载器默认遵循双亲委派准则。而线程上下文加载器就是为了解决当父类加载器加载到的类却依赖子类加载器的情况。

Java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方为这些接口提供实现。常见的 SPI有 JDBC、JCE、JNDI、JAXP 和 JBI 等。这些 SPI 的接口由 Java 核心库来提供，如 JAXP 的 SPI 接口定义包含在 `javax.xml.parsers`包中。这些 SPI 的实现代码很可能是作为 Java 应用所依赖的 jar 包被包含进来，可以通过类路径（CLASSPATH）来找到。

**SPI 的接口是 Java 核心库的一部分，是由引导类加载器来加载的；SPI 实现的 Java 类一般是由系统类加载器来加载的。引导类加载器是无法找到 SPI 的实现类的，因为它只加载 Java 的核心库。它也不能代理给系统类加载器，因为它是系统类加载器的祖先类加载器。也就是说，类加载器的代理模式无法解决这个问题。**

一个非常经典的例子就是SQL的驱动管理类——`java.sql.DriverManager`。`java.sql.DriverManager`是Java的标准服务，该类放在rt.jar中，因此是由启动类加载器加载的，但是在应用启动的时候，该驱动类管理是需要加载由不同数据库厂商实现的驱动，但是启动类加载器找不到这些具体的实现类。

线程上下文加载器可以通过java.lang.Thread类的setContextClassLoader()方法进行设置，如果创建线程时候它还没有被设置，就会从父线程中继承一个，**如果在应用程序的全局范围都没有设置过的话，那这个类加载器就是应用程序类加载器。**

有了线程上下文加载器，就可以解决上面的问题——父类加载器需要请求子类加载器完成类加载的动作，这种行为实际上就是打破了双亲委派的加载规则。

我们可以看一下`DriverManager`的源码，有一个静态代码块初始化Driver

```java
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
```

而`loadInitialDrivers`最终是通过`ServiceLoader.load`方法加载驱动

```java
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
```

```java
   public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
```

# 类加载过程

![](/assets/images/posts/classloader/classloader-3.png)

如上图所示

当程序要使用某个类的时候，如果该类还没有被加载到内存中，系统会通过加载、连接和初始化三步来实现对该类的初始化。

## java类什么时候加载

- 遇到 new、getstatic、putstatic 等指令时
- 对类进行反射调用的时候
- 初始化某个类的子类的时候
- 虚拟机启动时会先加载设置的程序主类
- 使用 JDK 1.7 的动态语言支持的时候

> 通俗点说就是：当运行过程中需要这个类的时候。

## 加载

1. 将class文件中的二进制数据数据读入到内存中，
2. 然后将该字节流所代表的静态存储结构转换为方法区中运行的数据结构，
3. 在堆中生成一个代表此类的java.lang.Class对象，作为访问方法区这些数据结构的入口。

## 连接

连接分为三步  验证->准备->解析

1. 验证：确保class文件中字节流包含的信息符合当前虚拟机的要求，字节码校验器会校验生成的字节码是否正确，如果校验失败，我们会得到校验错误。

- 文件格式的验证：基于字节流验证，验证字节流符合当前的Class文件格式的规范，能被当前虚拟机处理。验证通过后，字节流才会进入内存的方法区进行存储。
- 元数据的验证：基于方法区的存储结构验证，对字节码进行语义验证，确保不存在不符合java语言规范的元数据信息。
- 字节码验证：基于方法区的存储结构验证，通过对数据流和控制流的分析，保证被检验类的方法在运行时不会做出危害虚拟机的动作。
- 符号引用验证：基于方法区的存储结构验证，发生在虚拟机将二进制符号转换为直接引用的时候 ，确保能够将符号引用成功的解析为直接引用，其目的是确保解析动作正常执行。换句话说就是对类自身以外的信息进行匹配性校验

2. 准备：为类变量分配内存并设置初始值。

- 这些变量使用的内存都在方法区中分配这时候分配的内存仅包括
- 类变量(静态变量)，实例变量会在对象实例化的时候        
- 随着对象一起分配在堆内存中

3. 解析：将二进制符号的引用替换为直接引用

例如，在com.example.Person类中引用了com.example.Animal类，在编译阶段，Person类并不知道Animal的实际内存地址，因此只能用com.example.Animal来代表Animal真实的内存地址。在解析阶段，JVM可以通过解析该符号引用，来确定com.example.Animal类的真实内存地址（如果该类未被加载过，则先加载）。

## 初始化：

1. 父类静态(静态的成员变量，静态代码块)，
2. 子类静态(子类静态成员变量，子类的静态代码块)
3. 父类非静态(非静态成员变量，构造代码块，构造函数)
4. 子类非静态(子类非静态成员变量，子类构造代码块，子类构造函数)


对于下面的代码，赋值过程分两次，一是上面我们提到的准备阶段，此时的value将会被赋值为0；而value=33这个过程发生在类构造器的<clinit>()方法中。
```
public static int value=33;
```

>  java中，对于初始化阶段，有且只有以下五种情况才会对要求类立刻初始化：
>  
>  - 使用new关键字实例化对象、访问或者设置一个类的静态字段（被final修饰、编译器优化时已经放入常量池的例外）、调用类方法，都会初始化该静态字段或者静态方法所在的类；
>  - 初始化类的时候，如果其父类没有被初始化过，则要先触发其父类初始化；
>  - 使用java.lang.reflect包的方法进行反射调用的时候，如果类没有被初始化，则要先初始化；
>  - 虚拟机启动时，用户会先初始化要执行的主类（含有main）；
>  - jdk 1.7后，如果java.lang.invoke.MethodHandle的实例最后对应的解析结果是 REF_getStatic、REF_putStatic、REF_invokeStatic方法句柄，并且这个方法所在类没有初始化，则先初始化；

# 自定义ClassLoader

Java中提供的默认ClassLoader，只加载指定目录下的jar和class，如果我们想加载其它位置的类或jar时(如：网络上的一个class文件)，通过动态加载到内存之后，要调用这个类中的方法实现我的业务逻辑。在这样的情况下，默认的ClassLoader就不能满足我们的需求了，所以需要定义自己的ClassLoader。自定义的步骤如下：

- 编写一个类继承自ClassLoader抽象类。
- 复写它的findClass()方法。
- 在findClass()方法中调用defineClass()。

> 一般可以继承`URLClassLoader`，因为这个类已经帮我们实现了大部分工作。

> classloader主要有几个方法
>
> **1. defineclass**
>
> defineclass主要负责将byte字节流解析成JVM能够识别的Class对象。比如通过网络获取字节流，然后转化为Class对象。
>
> defineclass生成的Class对象还没有resolve，只会对象真正实例化时才会resolve（resolve是解析连接的过程，经过这一步Class对象才能真正被访问到）
>
> **2. findclass**
>
> defineclass和findclass方法通常是一起使用的，我们通过直接覆盖classloader父类的findclass方法来实现自定义规则。
>
> 如果想在类加载到JVM时就被连接，那么亦可以在findclass中再次调用resolveclass方法来进行连接。
>
> **3. loadclass**
>
> loadclass按照双亲委派模型进行加载，加载不到才会调用findclass。如果我们要在运行时加载一个类，直接调用`this.getClass().getClassLoader().loadClass()`方法就可以完成类加载了。

示例

```
public class DiskClassLoader extends ClassLoader {

  private String mLibPath;

  public DiskClassLoader(String path) {
    mLibPath = path;
  }

  @Override
  protected Class<?> findClass(String name) throws ClassNotFoundException {
    String fileName = getFileName(name);

    File file = new File(mLibPath, fileName);

    try {
      FileInputStream is = new FileInputStream(file);

      ByteArrayOutputStream bos = new ByteArrayOutputStream();
      int len = 0;
      try {
        while ((len = is.read()) != -1) {
          bos.write(len);
        }
      } catch (IOException e) {
        e.printStackTrace();
      }

      byte[] data = bos.toByteArray();
      is.close();
      bos.close();

      return defineClass(name, data, 0, data.length);

    } catch (IOException e) {
      e.printStackTrace();
    }

    return super.findClass(name);
  }

  //获取要加载 的class文件名
  private String getFileName(String name) {
    int index = name.lastIndexOf('.');
    if (index == -1) {
      return name + ".class";
    } else {
      return name.substring(index + 1) + ".class";
    }
  }
}
```

测试

```
DiskClassLoader diskLoader = new DiskClassLoader(
"D:\\lib");
Class clazz = diskLoader.loadClass("classloader.CustomClassLoaderSimple");
Object obj = clazz.newInstance();
Method method = clazz.getDeclaredMethod("display", null);
method.invoke(obj);
```

# Java应不应该加载动态类

Java有一个痛处，就是修改一个类，必须要重启一遍，才会重新加载，非常费时，如果使用动态的类加载就可以节省很多时间。

但是这是不好的，Java的优势正是基于共享对象的机制，达到信息的高度共享，对象一旦被创建就可以被重复利用。

如果动态加载一个对象到jvm，很难在jvm中平滑过渡。在理论上可以替换原对象然后修改所有指向该对象的引用，但是它违反了jvm的设计原则，。

对象的引用关系只有对象的创建者和持有和使用，jvm不可以干预对象的引用关系。

## 特例

对象不能被动态替换的原因是有很多引用指向它，很多变量保存着它的状态，如果对象不会被引用所指，没有人持有对象的状态，则可以动态地替换对象。

JSP就是这么做的，一旦JSP页面被修改，我们就可以热加载对应生成的servlet实例，这个实例是单例的，并且是无状态的，没有引用指向它，利用这个原理，很多热部署的方案也出现了。

# 一个特别的问题

> 这个章节的内容抄自你假笨大神的[文章](https://mp.weixin.qq.com/s/uyj9UDxR3DfWD3ZCzJrJLQ) 

同一个类加载器对象是否可以加载同一个类文件多次并且得到多个Class对象而都可以被java层使用吗？

## 正常的类加载

从JVM实现角度来说一下。在JVM里有一个数据结构叫做SystemDictonary，这个结构主要就是用来检索我们常说的类信息，这些类信息对应的结构是klass，对SystemDictonary的理解，可以认为就是一个Hashtable，key是类加载器对象+类的名字，value是指向klass的地址。这样当我们任意一个类加载器去正常加载类的时候，就会到这个SystemDictonary中去查找，看是否有这么一个klass可以返回，如果有就返回它，否则就会去创建一个新的并放到结构里。因此在正常情况下不可能出现同一个类加载器加载同一个类多次的情况。

## 特例

```java
    public static void main(String args[]) throws Throwable {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(null);
        String filePath = "Test.class";
        byte[] buffer = getFileContent(filePath);
        Class<?> c1 = unsafe.defineAnonymousClass(ClassTest.class, buffer, null);
        Class<?> c2 = unsafe.defineAnonymousClass(ClassTest.class, buffer, null);
        System.out.println(c1 == c2);
    }
```

输出为

```
false
class com.github.edgar615.nef.user.launcher.Test/500977346
class com.github.edgar615.nef.user.launcher.Test/20132171
```

这也就是说c1和c2其实是两个不同的对象，因为我们的类文件都是一样的，也就是字节码里的类名也是完全一样的，因此在jvm里的类对象的名字其实也都是一样的，但是hashCode不同

另外你无法通过java层面的其他api，比如Class.forName来获取到这种class，所以你要保存好这个得到的Class对象才能后面继续使用它。

## defineAnonymousClass

defineAnonymousClass这个方法比较特别，从名字上也看得出，是创建了一个匿名的类，不过这种匿名的概念和我们理解的匿名是不太一样的。这种类的创建通常会有一个宿主类，也就是第一个参数指定的类，这样一来，这个创建的类会使用这个宿主类的定义类加载器来加载这个类，**最关键的一点是这个类被创建之后并不会丢到上述的SystemDictonary里，也就是说我们通过正常的类查找，比如Class.forName等api是无法去查到这个类是否被定义过的**。因此过度使用这种api来创建这种类在一定程度上会带来一定的内存泄露。

那有人就要问了，看不到啥好处，为啥要提供这种api，这么做有什么意义，大家可以去了解下JSR292。jvm通过InvokeDynamic可以支持动态类型语言，这样一来其实我们可以提供一个类模板，在运行的时候加载一个类的时候先动态替换掉常量池中的某些内容，这样一来，同一个类文件，我们通过加载多次，并且传入不同的一些cpPatches，也就是defineAnonymousClass的第三个参数，  这样就能做到运行时产生不同的效果。

主要是因为原来的JVM类加载机制是不允许这种情况发生的，因为我们对同一个名字的类只能被同一个类加载器加载一次，因而为了能支持动态语言的特性，提供类似的api来达到这种效果。

# 参考资料

https://blog.csdn.net/a724888/article/details/81456439

https://mp.weixin.qq.com/s/uyj9UDxR3DfWD3ZCzJrJLQ