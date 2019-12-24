---
layout: post
title: String，StringBuilder,StringBuffer比较
date: 2019-12-23
categories:
    - java
comments: true
permalink: java-stringbuilder-2.html
---

先看单线程

```
    String withString = "";
    long t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      withString += "some string";
    }
    System.out.println("strings:" + (System.currentTimeMillis() - t0));

    t0 = System.currentTimeMillis();
    StringBuffer buf = new StringBuffer();
    for (int i = 0; i < 100000; i++) {
      buf.append("some string");
    }
    System.out.println("Buffers : " + (System.currentTimeMillis() - t0));

    t0 = System.currentTimeMillis();
    StringBuilder building = new StringBuilder();
    for (int i = 0; i < 100000; i++) {
      building.append("some string");
    }
    System.out.println("Builder : " + (System.currentTimeMillis() - t0));
```

输出

```
strings:51645
Buffers : 2
Builder : 2
```

可以看到string的很慢，Buffer和Builder差不多。

修改堆内存`-Xmx10240m -Xmn10240m -XX:+PrintGCDetails`，重新测试，buffer比builder慢了一点，string的连接操作伴随了大量的FullGC

> 因为string是不可变对象，拼接操作会伴随着大量char数组的扩容复制操作

```
strings:50391
Buffers : 3
Builder : 1
```

对测试方法做一点改动

```
    long t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      String withString = "some " + "string";
    }
    System.out.println("strings:" + (System.currentTimeMillis() - t0));

    t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      StringBuffer buf = new StringBuffer();
      buf.append("some ").append("string");
    }
    System.out.println("Buffers : " + (System.currentTimeMillis() - t0));

    t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      StringBuilder building = new StringBuilder();
      building.append("some ").append("string");

    }
    System.out.println("Builder : " + (System.currentTimeMillis() - t0));
```

输出

```
strings:3
Buffers : 9
Builder : 8
```

这时发现string的拼接最快，因为String 对象的直接相加，JVM 会自动对其进行优化，也就是说 `"some " + "string"` 在编译期间会自动优化为 "some string"，直接一次性创建完成，所以效率肯定要高于`StringBuffer`和`StringBuilder`的 append 拼接。

但是要注意对于下面这种间接相加的操作，效率要比直接相加低，因为在编译器不会对引用变量进行优化。

```
String str1 = "Hello";
String str2 = "World";
String str3 = str1+str2;
```

测试一下

```
    long t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      String some = "some ";
      String string = "string";
      String withString = some + string;
    }
    System.out.println("strings:" + (System.currentTimeMillis() - t0));
```

输出

```
strings:17
```

在多线程下比较`StringBuffer`和`StringBuilder`

```
public class StringsPerf {

    public static void main(String[] args) {

        ThreadPoolExecutor executorService = (ThreadPoolExecutor) Executors.newFixedThreadPool(10);
        //With Buffer
        StringBuffer buffer = new StringBuffer();
        for (int i = 0 ; i < 10; i++){
            executorService.execute(new AppendableRunnable(buffer));
        }
        shutdownAndAwaitTermination(executorService);
        System.out.println(" Thread Buffer : "+ AppendableRunnable.time);

        //With Builder
        AppendableRunnable.time = 0;
        executorService = (ThreadPoolExecutor) Executors.newFixedThreadPool(10);
        StringBuilder builder = new StringBuilder();
        for (int i = 0 ; i < 10; i++){
            executorService.execute(new AppendableRunnable(builder));
        }
        shutdownAndAwaitTermination(executorService);
        System.out.println(" Thread Builder: "+ AppendableRunnable.time);

    }

   static void shutdownAndAwaitTermination(ExecutorService pool) {
        pool.shutdown(); // code reduced from Official Javadoc for Executors
        try {
            if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
                pool.shutdownNow();
              if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
                System.err.println("Pool did not terminate");
              }
            }
        } catch (Exception e) {}
    }
}

class AppendableRunnable<T extends Appendable> implements Runnable {

    static long time = 0;
    T appendable;
    public AppendableRunnable(T appendable){
        this.appendable = appendable;
    }

    @Override
    public void run(){
        long t0 = System.currentTimeMillis();
        for (int j = 0 ; j < 10000 ; j++){
            try {
                appendable.append("some string");
            } catch (IOException e) {}
        }
        time+=(System.currentTimeMillis() - t0);
    }
}
```

可以看到`StringBuffer`明显要比`StringBuilder`慢

# 高频面试题

> 下面的内容都是拷贝自参考资料

## StringBuilder 的效率一定比 String 更高吗？

我们通常会说 StringBuilder 效率要比 String 高，严谨一点这句话不完全对，虽然大部分情况下使用 StringBuilder 效率更高，但在某些特定情况下不一定是这样，比如下面这段代码：

```
String str = "Hello"+"World";
StringBuilder stringBuilder = new StringBuilder("Hello");
stringBuilder.append("World");
```

此时，使用 String 创建 "HelloWorld" 的效率要高于使用 StringBuilder 创建 "HelloWorld"，这是为什么呢？

因为 String 对象的直接相加，JVM 会自动对其进行优化，也就是说 "Hello"+"World" 在编译期间会自动优化为 "HelloWorld"，直接一次性创建完成，所以效率肯定要高于 StringBuffer 的 append 拼接。

但是需要注意的是如果是这样的代码：

```
String str1 = "Hello";
String str2 = "World";
String str3 = str1+str2;
```

对于这种间接相加的操作，效率要比直接相加低，因为在编译器不会对引用变量进行优化。

## 下面代码的运行结果是？

```
String str1 = "Hello World";
String str2 = "Hello"+" World";
System.out.println(str1 == str2);
```

true，因为 "Hello"+" World" 在编译期间会被 JVM 自动优化成 "Hello World"，是一个字符串常量，所以和 str1 引用相同。

## 下面代码的运行结果是？

```
String str1 = "Hello World";
String str2 = "Hello";
String str3 = str2 + " World";
System.out.println(str1 == str3);
```

false，JVM 只有在 String 对象直接拼接的时候才会进行优化，如果是对变量进行拼接则不会优化，所以 str2 + " World" 并不会直接优化成字符串常量 "Hello World"，同时这种间接拼接的结果是存放在堆内存中的，所以 str1 和 str3 的引用肯定不同。

## String str = new String("Hello World") 创建了几个对象？

这是很常见的一道面试题，大部分的答案都是 2 个，"Hello World" 是一个，另一个是指向字符串的变量 str，其实是不准确的。

因为代码的执行过程和类的加载过程是有区别的，如果只看运行期间，这段代码只创建了 1 个对象，new 只调用了一次，即在堆上创建的 "Hello World" 对象。

而在类加载的过程中，创建了 2 个对象，一个是字符串字面量 "Hello World" 在字符串常量池中所对应的实例，另一个是通过 new String("Hello World") 在堆中创建并初始化，内容与 "Hello World" 相同的实例。

所以在回答这道题的时候，可以先问清楚面试官是在代码执行过程中，还是在类加载过程中。这道题目如果换做是 `String str = new String("Hello World")` 涉及到几个对象，那么答案就是 2 个。

## String、StringBuffer、StringBuilder 有什么区别？

1. String 一旦创建不可变，如果修改即创建新的对象，StringBuffer 和 StringBuilder 可变，修改之后引用不变。
2. String 对象直接拼接效率高，但是如果执行的是间接拼接，效率很低，而 StringBuffer 和 StringBuilder 的效率更高，同时 StringBuilder 的效率高于 StringBuffer。
3. StringBuffer 的方法是线程安全的，StringBuilder 是线程不安全的，在考虑线程安全的情况下，应该使用 StringBuffer。

## 下面代码的运行结果是？

```
public static void main(String[] args) {
    String str = "Hello";
    test(str);
    System.out.println(str);
}
public static void test(String str){
    str+="World";
}
```

Hello，因为 String 是不可变的，传入 test 方法的参数相当于 str 的一个副本，所以方法内只是修改了副本，str 本身的值没有发生变化。

## 下面代码的运行结果是？

```
public static void main(String[] args) {
  StringBuffer str = new StringBuffer("Hello");
  test(str);
  System.out.println(str);
}
public static void test(StringBuffer str){
  str.append(" World");
}
```

Hello World，因为 StringBuffer 是可变类型，传入 test 方法的参数就是 str 的引用，所以方法内修改的就是 str 本身。

# 参考资料

https://stackoverflow.com/questions/355089/difference-between-stringbuilder-and-stringbuffer

https://mp.weixin.qq.com/s/dltB1Fzwdezs4PDN8Thx6w
