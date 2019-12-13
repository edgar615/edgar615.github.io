---
layout: post
title: Java的boolean占几个字节
date: 2019-12-13
categories:
    - java
comments: true
permalink: java-boolean.html
---

今天看了一个面试题发现自己也不清楚，记录一下：

> Java中boolean类型占用多少个字节

我们知道java有8种基本类型

- byte：8bit
- short：16bit
- int：32bit
- long：64bit
- float：32bit
- double：64bit
- char：16bit
- boolean：官方文档上说：**This data type represents one bit of information, but its "size" isn't something that's precisely defined.**

虽然Java虚拟机定义了一个boolean类型，但它只为它提供了非常有限的支持。没有Java虚拟机指令专门用于对boolean值的操作。相反，Java编程语言中对boolean值进行操作的表达式被编译为使用Java虚拟机int数据类型的值。

Java虚拟机直接支持boolean数组。它的newarray指令可以创建boolean数组。使用byte数组指令baload和bastore访问和修改类型为boolean的数组。

    在Oracle的Java虚拟机实现中，Java编程语言中的boolean数组被编码为Java虚拟机byte数组，每个布尔元素使用8位。

> 虽然boolean只需要1位即可保存，但由于大部分计算机在分配内存时最小的内存单元是字节：8bit

Java虚拟机使用1表示boolean数组组件的true，0表示false。其中Java编程语言布尔值由编译器映射到Java虚拟机类型int的值，编译器必须使用相同的编码。

测试一把（代码来着参考资料）

```
class LotsOfBooleans
{
    boolean a0, a1, a2, a3, a4, a5, a6, a7, a8, a9, aa, ab, ac, ad, ae, af;
    boolean b0, b1, b2, b3, b4, b5, b6, b7, b8, b9, ba, bb, bc, bd, be, bf;
    boolean c0, c1, c2, c3, c4, c5, c6, c7, c8, c9, ca, cb, cc, cd, ce, cf;
    boolean d0, d1, d2, d3, d4, d5, d6, d7, d8, d9, da, db, dc, dd, de, df;
    boolean e0, e1, e2, e3, e4, e5, e6, e7, e8, e9, ea, eb, ec, ed, ee, ef;
}

class LotsOfInts
{
    int a0, a1, a2, a3, a4, a5, a6, a7, a8, a9, aa, ab, ac, ad, ae, af;
    int b0, b1, b2, b3, b4, b5, b6, b7, b8, b9, ba, bb, bc, bd, be, bf;
    int c0, c1, c2, c3, c4, c5, c6, c7, c8, c9, ca, cb, cc, cd, ce, cf;
    int d0, d1, d2, d3, d4, d5, d6, d7, d8, d9, da, db, dc, dd, de, df;
    int e0, e1, e2, e3, e4, e5, e6, e7, e8, e9, ea, eb, ec, ed, ee, ef;
}


public class Test
{
    private static final int SIZE = 1000000;

    public static void main(String[] args) throws Exception
    {
        LotsOfBooleans[] first = new LotsOfBooleans[SIZE];
        LotsOfInts[] second = new LotsOfInts[SIZE];

        System.gc();
        long startMem = getMemory();

        for (int i=0; i < SIZE; i++)
        {
            first[i] = new LotsOfBooleans();
        }

        System.gc();
        long endMem = getMemory();

        System.out.println ("Size for LotsOfBooleans: " + (endMem-startMem));
        System.out.println ("Average size: " + ((endMem-startMem) / ((double)SIZE)));

        System.gc();
        startMem = getMemory();
        for (int i=0; i < SIZE; i++)
        {
            second[i] = new LotsOfInts();
        }
        System.gc();
        endMem = getMemory();

        System.out.println ("Size for LotsOfInts: " + (endMem-startMem));
        System.out.println ("Average size: " + ((endMem-startMem) / ((double)SIZE)));

        // Make sure nothing gets collected
        long total = 0;
        for (int i=0; i < SIZE; i++)
        {
            total += (first[i].a0 ? 1 : 0) + second[i].a0;
        }
        System.out.println(total);
    }

    private static long getMemory()
    {
        Runtime runtime = Runtime.getRuntime();
        return runtime.totalMemory() - runtime.freeMemory();
    }
}
```

输出

```
Size for LotsOfBooleans: 94317824
Average size: 94.317824
Size for LotsOfInts: 335999984
Average size: 335.999984
```

# 总结

- boolean类型被编译为int类型，等于是说JVM里占用字节和int完全一样，int是4个字节，于是boolean也是4字节
- boolean数组在Oracle的JVM中，编码为byte数组，每个boolean元素占用8位=1字节

# 参考资料

https://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html

https://stackoverflow.com/questions/383551/what-is-the-size-of-a-boolean-variable-in-java
