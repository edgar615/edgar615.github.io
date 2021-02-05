---
layout: post
title: JVM内存与GC（10）- 逃逸分析
date: 2020-01-10
categories:
    - jvm
comments: true
permalink: java-escape.html
---

**Java对象实例和数组元素并不一定都是在堆上分配内存的，在满足特定条件时，它们可以在（虚拟机）栈上分配内存。**

这是因为Java JIT（just-in-time）编译器进行的两项优化，分别称作**逃逸分析（escape analysis）**和**标量替换（scalar replacement）**。

![](/assets/images/posts/java-escape/java-escape-1.png)

> 在编译程序优化理论中，逃逸分析是一种确定指针动态范围的方法——分析在程序的哪些地方可以访问到指针。当一个变量（或对象）在子程序中被分配时，一个指向变量的指针可能逃逸到其它执行线程中，或是返回到调用者子程序。
>
> 如果一个子程序分配一个对象并返回一个该对象的指针，该对象可能在程序中被访问到的地方无法确定——这样指针就成功“逃逸”了。如果指针存储在全局变量或者其它数据结构中，因为全局变量是可以在当前子程序之外访问的，此时指针也发生了逃逸。
>
> 逃逸分析确定某个指针可以存储的所有地方，以及确定能否保证指针的生命周期只在当前进程或线程中。

简单来讲，JVM中的逃逸分析可以通过分析对象引用的使用范围（即动态作用域），来决定对象是否要在堆上分配内存，也可以做一些其他方面的优化。

以下的例子说明了一种对象逃逸的可能性。

```java
  static StringBuilder getStringBuilder1(String a, String b) {
    StringBuilder builder = new StringBuilder(a);
    builder.append(b);
    return builder;   // builder通过方法返回值逃逸到外部
  }

  static String getStringBuilder2(String a, String b) {
    StringBuilder builder = new StringBuilder(a);
    builder.append(b);
    return builder.toString();  // builder范围维持在方法内部，未逃逸
  }

```

以JDK 1.8为例，可以通过设置JVM参数`-XX:+DoEscapeAnalysis`、`-XX:-DoEscapeAnalysis`来开启或关闭逃逸分析（默认当然是开启的）。下面先写一个没有对象逃逸的例子。

```java
public class EscapeAnalysisTest {
  public static void main(String[] args) throws Exception {
    long start = System.currentTimeMillis();
    for (int i = 0; i < 5000000; i++) {
      allocate();
    }
    System.out.println((System.currentTimeMillis() - start) + " ms");
    Thread.sleep(600000);
  }

  static void allocate() {
    MyObject myObject = new MyObject(2019, 2019.0);
  }

  static class MyObject {
    int a;
    double b;

    MyObject(int a, double b) {
      this.a = a;
      this.b = b;
    }
  }
}

```

然后通过开启和关闭`DoEscapeAnalysis`开关观察不同。（需要调大堆内存）

- 关闭逃逸分析

  ```
  $ jmap -histo 14552
  
   num     #instances         #bytes  class name
  ----------------------------------------------
     1:       5000000      120000000  com.github.edgar615.EscapeAnalysisTest$MyObject
     2:           432       20916424  [I
     3:          3055         440472  [C
     4:          2334          56016  java.lang.String
     5:           479          54768  java.lang.Class
  ```

- 开启逃逸分析

```
jmap -histo 9112

 num     #instances         #bytes  class name
----------------------------------------------
   1:           425       27733952  [I
   2:        125353        3008472  com.github.edgar615.EscapeAnalysisTest$MyObject
   3:          3055         440464  [C
   4:          2334          56016  java.lang.String
   5:           479          54768  java.lang.Class


```

可见，关闭逃逸分析之后，堆上有5000000个MyObject实例，而开启逃逸分析之后，就只剩下125353个实例了，不管是实例数还是内存占用都只有原来的2%不到。

但是，**逃逸分析只是栈上内存分配的前提，接下来还需要进行标量替换才能真正实现。**

**所谓标量，就是指JVM中无法再细分的数据，比如int、long、reference等**。相对地，能够再细分的数据叫做聚合量。仍然考虑上面的例子，MyObject就是一个聚合量，因为它由两个标量a、b组成。通过逃逸分析，JVM会发现myObject没有逃逸出allocate()方法的作用域，标量替换过程就会将myObject直接拆解成a和b，也就是变成了

```java
  static void allocate() {
    int a = 2019;
    double b = 2019.0;
  }

```

可见，对象的分配完全被消灭了，而int、double都是基本数据类型，直接在栈上分配就可以了。所以，**在对象不逃逸出作用域并且能够分解为纯标量表示时**，对象就可以在栈上分配

JVM提供了参数`-XX:+EliminateAllocations`来开启标量替换，默认仍然是开启的。显然，如果把它关掉的话，就相当于禁止了栈上内存分配，只有逃逸分析是无法发挥作用的。

除了标量替换之外，通过逃逸分析还能实现锁消除

# 参考资料

https://www.jianshu.com/p/8377e09971b8

