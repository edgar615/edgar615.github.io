---
layout: post
title: G1优化重复字符串
date: 2020-04-24
categories:
    - jvm
comments: true
permalink: java-g1-string-deduplication.html
---

现代 Java 应用程序有大量的字符串操作，例如，Web 服务 API 调用（JSON、REST、SOAP 等）、外部数据源调用（SQL、从 DB 返回的数据等）以及文本解析和文本创建等。因此，字符串对象很容易就占据了约至少 30％ 的内存。然而，这些 String 对象中的大多数都是重复的，这些字符串的重复浪费了大量内存。因此，优化重复字符串对象浪费的内存是 Java 非常受欢迎的功能之一。在 G1 中，Java 就对此功能做了支持。

G1 GC 算法运行时，它将从内存中删除垃圾对象。它还从内存中删除重复的字符串对象（字符串重复数据删除）。可以通过设置以下 JVM 参数来激活此功能：

```
-XX:+UseG1GC -XX:+UseStringDeduplication
```

> **注意1：**为了使用此功能， 需要在 Java 8 update 20 或更高版本上运行。
>
> **注意2：**为了使用 `-XX:+UseStringDeduplication`，您需要使用 G1 GC 算法。

让我们通过这个程序来验证

```java
public class StringDeduplicationExample {


    public static List<String> myStrings = new ArrayList<>();


    public static void main(String[] args) throws Exception {
        System.gc();
        long startMem = getMemory();
        for (int counter = 0; counter < 200; ++counter) {
            for (int secondCounter = 0; secondCounter < 1000; ++secondCounter) {
                // Add it 1000 times.
                myStrings.add(("Hello World-" + counter));
            }
            System.out.println("Hello World-" + counter + " has been added 1000 times");
        }
        System.gc();
        long endMem = getMemory();
        System.out.println(endMem - startMem);

        // Make sure nothing gets collected
        long total = 0;
        for (int i=0; i < 10000; i++)
        {
            total += i;
        }
        System.out.println(total);
    }

    private static long getMemory() {
        Runtime runtime = Runtime.getRuntime();
        return runtime.totalMemory() - runtime.freeMemory();
    }
}
```

运行`-XX:+UseStringDeduplication`

```
$ java -XX:+UseG1GC -XX:+UseStringDeduplication StringDeduplicationExample
10467552
```

运行`-XX:-UseStringDeduplication`

```
$ java -XX:+UseG1GC -XX:+UseStringDeduplication StringDeduplicationExample
15278912
```

由于使用了 `-XX:+UseStringDeduplication`参数，从应用程序中删除了大量重复字符串，从而大幅度减少内存消耗。因此，你可以利用 `-XX:+UseG1GC-XX:+UseStringDeduplication`来减少重复字符串导致的内存浪费，它会减少应用程序的整体内存占用。

# 参考资料

https://mp.weixin.qq.com/s/0b3w7ZWi6WOY6wqhHaiUkg