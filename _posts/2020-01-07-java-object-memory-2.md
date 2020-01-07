---
layout: post
title: java对象的内存占用分析
date: 2020-01-07
categories:
    - jvm
comments: true
permalink: java-object-memory-2.html
---

# Java对象内存布局

Java对象的内存布局包括：对象头(Header)，实例数据(Instance Data)和补齐填充(Padding)

> 详细内容查看 https://edgar615.github.io/java-object-memory-2.html

## 对象头

在64位机器上，默认不开启指针压缩`-XX:-UseCompressedOops`的情况下，对象头占用12bytes，开启指针压缩`-XX:+UseCompressedOops`则占用16bytes。

## 实例类型
实例类型的内存占用如下：

| 类型      | 大小                                                         |
| --------- | ------------------------------------------------------------ |
| boolean   | [在数组中占1个字节，单独使用时占4个字节](https://edgar615.github.io/java-boolean.html) |
| byte      | 1                                                            |
| short     | 2                                                            |
| char      | 2                                                            |
| int       | 2                                                            |
| float     | 4                                                            |
| float     | 4                                                            |
| long      | 8                                                            |
| double    | 8                                                            |
| reference | 32位4字节，64位8字节                                         |
| String    | length * 2 + 4                                               |

Java对象占用空间是8字节对齐的，即所有Java对象占用bytes数必须是8的倍数。例如，一个包含两个属性的对象：int和byte，并不是占用17bytes(12+4+1)，而是占用24bytes（对17bytes进行8字节对齐）

对象引用（reference）类型在64位机器上，关闭指针压缩时占用8bytes， 开启时占用4bytes。

## 包装类型

包装类（Boolean/Byte/Short/Character/Integer/Long/Double/Float）占用内存的大小等于对象头大小加上底层基础数据类型的大小。

| 包装类型         | +useCompressedOops | -useCompressedOops |
| ---------------- | ------------------ | ------------------ |
| Byte, Boolean    | 16                 | 24                 |
| Short, Character | 16                 | 24                 |
| Integer, Float   | 16                 | 24                 |
| Long, Double     | 24                 | 24                 |

## 数组

64位机器上，数组对象的对象头占用24  bytes，启用压缩后占用16字节。比普通对象占用内存多是因为需要额外的空间存储数组的长度。基础数据类型数组占用的空间包括数组对象头以及基础数据类型数据占用的内存空间。由于对象数组中存放的是对象的引用，所以对象数组本身的大小=`数组对象头+length  * 引用指针大小`，总大小为对象数组本身大小+存放的数据的大小之和。例如

int[10]: 

- 开启压缩：`16 + 10 * 4 = 56 bytes`；
- 关闭压缩：`24 + 10 * 4 = 64bytes`。

new Integer[3]:

- 开启压缩：`32 + 3 * 16(Integer) = 80 (bytes)`
	- Integer数组本身：16(header) + 3 * 4(Integer reference) = 28(padding) -> 32 (bytes) 
- 关闭压缩：`48 + 3 * 24(Integer) = 120 bytes`
	- Integer数组本身：24(header) + 3 * 8(Integer reference) = 48 bytes;

上面的结果可以通过jol验证

## String

在JDK1.7及以上版本中，String包含2个属性，一个用于存放字符串数据的char[], 一个int类型的hashcode, 

```
public final class String {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0
```

在关闭指针压缩时，一个String本身需要 `16(Header) + 8(char[] reference) + 4(int) = 32 bytes。`
除此之外，一个char[]占用`24 + length * 2 bytes`(8字节对齐), 即一个String占用的内存空间大小为`56 + length * 2 bytes(8字节对齐)`

- 一个空字符串("")的大小应为：`56 + 0 * 2 bytes = 56 bytes`。
- 字符串`"abc"`的大小应为：`56 + 3 * 2 = 62(8字节对齐)->64 (bytes)`
- 字符串`"abcde"`的大小应为：`56 + 5 * 2 = 66->72 (bytes)`
- 字符串"abcde"在开启指针压缩时的大小为：   
	- String本身：`12(Header) + 4(char[] reference) + 4(int hash) = 20(padding) -> 24 (bytes);`
	- 存储数据：`16(char[] header) + 5*2 = 26(padding) -> 32 (bytes)`     
	- 总共：24 + 32 = 56 (bytes)  

### 计算Java对象内存大小有三种方式：

1. AgentSizeOf : 使用jvm代理和 Instrumentation. getObjectSize()
2. UnsafeSizeOf : 使用unsafe
3. ReflectionSizeOf : 通过反射出来Class的成员，通过成员类型进行计算

其他方法
```
Runtime r = Runtime.getRuntime();

// amount of unallocated memory
long free = r.freeMemory();

// total amount of memory available allocated and unallocated.
long total = r.totalMemory();

// total amount of allocated memory
long inuse = r.totalMemory() - r.freeMemory();

// max memory JVM will attempt to use
long max = r.maxMemory();

// suggest now would be a good time for garbage collection
System.gc();
```

# JOL
[JOL](http://openjdk.java.net/projects/code-tools/jol/) (Java Object Layout) 是分析JVM中内存布局的小工具, 通过 Unsafe, JVMTI, 以及 Serviceability Agent (SA) 来解码实际的对象布局,占用,引用。 所以 JOL 比起基于 heap dump, 或者基于规范的其他工具来得准确

测试一下

```
public class MemoryLayoutTest {

    public static void main(String[] args){

        System.out.println(VM.current().details());
        System.out.println(ClassLayout.parseClass(A.class).toPrintable());
    }

    class A{
        long i;
    }

}
```

输出

```
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

com.xxxx.MemoryLayoutTest$A object internals:
 OFFSET  SIZE                                   TYPE DESCRIPTION                               VALUE
      0    12                                        (object header)                           N/A
     12     4   com.xxxx.MemoryLayoutTest A.this$0                                  N/A
     16     8                                   long A.i                                       N/A
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

```

整个对象将占用24个字节的大小。

对象头

```
0    12                                        (object header)                           N/A
```

填充

```
12     4   com.xxxx.MemoryLayoutTest A.this$0                                  N/A
```

字段i

```
16     8                                   long A.i                                       N/A
```




# 参考资料

https://www.mindprod.com/jgloss/sizeof.html

https://segmentfault.com/a/1190000006933272

https://zhuanlan.zhihu.com/p/50984945