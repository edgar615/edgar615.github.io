---
layout: post
title: String面试题
date: 2019-12-25
categories:
    - java
comments: true
permalink: java-string.html
---

了解了[字符串常量池](https://edgar615.github.io/jvm-string-pool)的知识后，对下面string的一些面试题就没多大问题了

# 创建字符串

```
String s1 = new String("123");
String s2 = "123";
System.out.println(s1 == s2);//false
```

```
String s1 = new String("123").intern();
String s2 = "123";
System.out.println(s1 == s2);  // true
```

```
String s1 = new String("123");
s1.intern();
String s2 = "123";
System.out.println(s1 == s2);  // false
```

# 两个双引号的字符串相加

```
String  s1 = "123";
String  s2 = "1"+"23";
System.out.println(s1 == s2);//true
```

```
String  s1 = new String("123").intern();
String  s2 = "1"+"23";
System.out.println(s1 == s2);//true
```

# 两个new String（）的字符串相加

```
String s1 = new String("1")+new String("23");
String s2 = "123";
System.out.println( s1 == s2);// false
```

```
String s1 = new String("1")+new String("23");
s1.intern();
String s2 = "123";
System.out.println( s1 == s2);// true
```

```
String s1 = new String("1")+new String("23");
String s2 = "123";
s1.intern();
System.out.println( s1 == s2);// false
```

# 双引号字符串常量与new String字符串相加

```
String s1 = "1"+new String("23");
String s2 = "123";
System.out.println( s1 == s2);// false
```

```
String s1 = "1"+new String("23");
String s2 = "123";
String s3 = s1.intern();
System.out.println( s3 == s2);// true
```

```
String s1 = "1"+new String("23");
String s3 = s1.intern();
String s2 = "123";
System.out.println( s3 == s2);// true
```

# 双引号字符串常量与一个字符串变量相加

```
String s1 = "1";
String s2 = s1 + "23";
String s3 = "123";
System.out.println( s2 == s3);// false
```

```
String s1 = "1";
String s2 = s1 + "23";
String s3 = "123";
System.out.println( s2.intern() == s3);// true
```

# intern

```
String s1 = new String("1")+new String("23");
s1.intern();
String s2 = "123";
System.out.println( s1 == s2);// true
```

```
String s1 = new String("1")+new String("23");
String s2 = "123";
s1.intern();
System.out.println( s1 == s2);//false
```

```
String h = new String("12") + new String("3");
String h1 = new String("1") + new String("23");
String h3 = h.intern();
String h4 = h1.intern();
String h2 = "123";

System.out.println(h == h1); // false
System.out.println(h3 == h4); // true
System.out.println(h == h3); // true
System.out.println(h3 == h2); // true
```

```
String h = new String("12") + new String("3");
String h1 = new String("1") + new String("23");
String h2 = "123";
String h3 = h.intern();
String h4 = h1.intern();

System.out.println(h == h1); // false
System.out.println(h3 == h4); // true
System.out.println(h == h3); // false
System.out.println(h3 == h2); // true
```

# StringBuilder 的效率一定比 String 更高吗？

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


# String str = new String("Hello World") 创建了几个对象？

这是很常见的一道面试题，大部分的答案都是 2 个，"Hello World" 是一个，另一个是指向字符串的变量 str，其实是不准确的。

因为代码的执行过程和类的加载过程是有区别的，如果只看运行期间，这段代码只创建了 1 个对象，new 只调用了一次，即在堆上创建的 "Hello World" 对象。

而在类加载的过程中，创建了 2 个对象，一个是字符串字面量 "Hello World" 在字符串常量池中所对应的实例，另一个是通过 new String("Hello World") 在堆中创建并初始化，内容与 "Hello World" 相同的实例。

所以在回答这道题的时候，可以先问清楚面试官是在代码执行过程中，还是在类加载过程中。这道题目如果换做是 `String str = new String("Hello World")` 涉及到几个对象，那么答案就是 2 个。

# String、StringBuffer、StringBuilder 有什么区别？

1. String 一旦创建不可变，如果修改即创建新的对象，StringBuffer 和 StringBuilder 可变，修改之后引用不变。
2. String 对象直接拼接效率高，但是如果执行的是间接拼接，效率很低，而 StringBuffer 和 StringBuilder 的效率更高，同时 StringBuilder 的效率高于 StringBuffer。
3. StringBuffer 的方法是线程安全的，StringBuilder 是线程不安全的，在考虑线程安全的情况下，应该使用 StringBuffer。
