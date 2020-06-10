---
layout: post
title: NIO的Buffer
description: 
date: 2019-06-10
categories:
    - netty,java
comments: true
permalink: nio-buffer.html
---

> 这是很多年前学习NIO的笔记，来源不记得了

# 1. 简介
Java NIO中的Buffer用于和NIO通道进行交互。 

- 写：Buffer -> Channel 
- 读：Channel -> Buffer

使用Buffer读写数据的步骤：

1. Write data into the Buffer
2. Call buffer.flip()
3. Read data out of the Buffer
4. Call buffer.clear() or buffer.compact()

clear()方法会清空整个缓冲区。

compact()方法只会清除已经读过的数据。任何未读的数据都被移到缓冲区的起始处，新写入的数据将放到缓冲区未读数据的后面。

## 1.1. Buffer类型

Buffer有以下类型

- ByteBuffer
- MappedByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer
- IntBuffer
- LongBuffer
- ShortBuffer

## 1.2. Buffer的四个属性

- capacity 容量

缓冲区能够容纳的数据元素的最大数量。这一容量在缓冲区创建时被设定，并且永远不能被改变。

作为一个内存块，Buffer有一个固定的大小值，你只能往里写capacity个byte、long，char等类型。

一旦Buffer满了，需要将其清空（通过读数据或者清除数据）才能继续写数据往里写数据。

- position 位置

下一个要被读或写的元素的索引。位置会自动由相应的`get()`和`put()`函数更新。

当你写数据到Buffer中时，position表示当前的位置。初始的position值为0.当一个byte、long等数据写到Buffer后， position会向前移动到下一个可插入数据的Buffer单元。position最大可为capacity – 1.

当读取数据时，也是从某个特定位置读。当将Buffer从写模式切换到读模式，position会被重置为0. 当从Buffer的position处读取数据时，position向前移动到下一个可读的位置。

- limit 上界

缓冲区的第一个不能被读或写的元素。或者说，缓冲区中现存元素的计数。

在写模式下，Buffer的limit表示你最多能往Buffer里写多少数据。 写模式下，limit等于Buffer的capacity。

当切换Buffer到读模式时， limit表示你最多能读到多少数据。因此，当切换Buffer到读模式时，limit会被设置成写模式下的position值。换句话说，你能读到之前写入的所有数据（limit被设置成已写数据的数量，这个值在写模式下就是position）

- mark 标记

一个备忘位置。调用`mark()`来设定`mark = postion`。调用`reset()`设定`position = mark`。标记在设定前是未定义的(undefined)。

这四个属性之间总是遵循以下关系： 0 <= mark <= position <= limit <= capacity


## 1.3. Buffer的方法

### 1.3.1. Allocating a Buffer
要想获得一个Buffer对象首先要进行分配

```java
ByteBuffer buf = ByteBuffer.allocate(48);
CharBuffer buf = CharBuffer.allocate(1024);
```

新创建的ByteBuffer格式

<table>
    <tr>
        <td>0</td>
        <td>1</td>
        <td>2</td>
        <td>3</td>
        <td>4</td>
        <td>5</td>
        <td>6</td>
        <td>7</td>
        <td>8</td>
        <td>9</td>
        <td></td>
    </tr>
    <tr>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
    </tr>
    <tr>
        <td>potision(0)、mark(x)</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td>limit(10)、capacity(10)</td>
    </tr>
</table>

### 1.3.2 读取/写入

```java
public abstract byte get();
public abstract byte get(int index);
public abstract ByteBuffer put(byte b);
public abstract ByteBuffer put(int index, byte b);
```

对于`put()`，如果运算会导致位置超出上界，就会抛出BufferOverflowException异常。对于`get()`，如果位置不小于上界，就会抛出BufferUnderflowException异常。绝对存取不会影响缓冲区的位置属性，但是如果您所提供的索引超出范围（负数或不小于上界），也将抛出IndexOutOfBoundsException异常。

```java
buffer.put((byte)'H').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o');
```

执行上面的代码后，buffer的结构如下所示

<table>
    <tr>
        <td>0</td>
        <td>1</td>
        <td>2</td>
        <td>3</td>
        <td>4</td>
        <td>5</td>
        <td>6</td>
        <td>7</td>
        <td>8</td>
        <td>9</td>
        <td></td>
    </tr>
    <tr>
        <td>H</td>
        <td>e</td>
        <td>l</td>
        <td>l</td>
        <td>o</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
    </tr>
    <tr>
        <td>mark(x)</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td>potision(5)</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td>limit(10)、capacity(10)</td>
    </tr>
</table>

···java
buffer.put(0,(byte)'M').put((byte)'w');
```

执行上面的代码后，buffer的结构如下所示

<table>
    <tr>
        <td>0</td>
        <td>1</td>
        <td>2</td>
        <td>3</td>
        <td>4</td>
        <td>5</td>
        <td>6</td>
        <td>7</td>
        <td>8</td>
        <td>9</td>
        <td></td>
    </tr>
    <tr>
        <td>M</td>
        <td>e</td>
        <td>l</td>
        <td>l</td>
        <td>o</td>
        <td>w</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
    </tr>
    <tr>
        <td>mark(x)</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td>potision(6)</td>
        <td></td>
        <td></td>
        <td></td>
        <td>limit(10)、capacity(10)</td>
    </tr>
</table>

### 1.3.3. 翻转
我们已经写满了缓冲区，现在我们必须准备将其清空。我们想把这个缓冲区传递给一个通道，以使内容能被全部写出。但如果通道现在在缓冲区上执行get()，那么它将从我们刚刚插入的有用数据之外取出未定义数据。如果我们将位置值重新设为0，通道就会从正确位置开始获取，但是它是怎样知道何时到达我们所插入数据末端的呢？这就是上界属性被引入的目的。上界属性指明了缓冲区有效内容的末端。我们需要将上界属性设置为当前位置，然后将位置重置为0。我们可以人工用下面的代码实现：

```java
buffer.limit(buffer.position()).position(0);
```

API提供了`flip()`函数将一个能够继续添加数据元素的填充状态的缓冲区翻转成一个准备读出元素的释放状态（从写模式切换到读模式）。
调用`flip()`方法会将position设为0，limit设为之前写的position。

翻转后，buffer变为

<table>
    <tr>
        <td>0</td>
        <td>1</td>
        <td>2</td>
        <td>3</td>
        <td>4</td>
        <td>5</td>
        <td>6</td>
        <td>7</td>
        <td>8</td>
        <td>9</td>
        <td></td>
    </tr>
    <tr>
        <td>M</td>
        <td>e</td>
        <td>l</td>
        <td>l</td>
        <td>o</td>
        <td>w</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
    </tr>
    <tr>
        <td>mark(x)、potision(0)</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td>limit(6)</td>
        <td></td>
        <td></td>
        <td></td>
        <td>capacity(10)</td>
    </tr>
</table>

### 1.3.4. rewind()
rewind()将position设为0，不影响limit，可以重新读取数据。

### 1.3.5. 释放
布尔函数hasRemaining()会在释放缓冲区时告诉您是否已经达到缓冲区的上界。

```java
for (int i = 0; buffer.hasRemaining( ), i++) {
    myByteArray [i] = buffer.get( );
}
```

一旦缓冲区对象完成填充并释放，它就可以被重新使用了。Clear()函数将缓冲区重置为空状态。它并不改变缓冲区中的任何数据元素，而是仅仅将上界设为容量的值，并把位置设回0

### 1.3.6. 清除 clear()
clear()方法会清空整个缓冲区。

*填充和释放缓冲区*

```java
public class BufferFillDrain {

    private static int index = 0;
    private static String[] strings = {"A random string value",
            "The product of an infinite number of monkeys",
            "Hey hey we're the Monkees",
            "Opening act for the Monkees: Jimi Hendrix",
            "'Scuse me while I kiss this fly",
            "Help Me! Help Me!"};

    public static void main(String[] args) {
        CharBuffer buffer = CharBuffer.allocate(100);
        while (fillBuff(buffer)) {
            buffer.flip();
            drainBuffer(buffer);
            buffer.clear();
        }
    }

    private static void drainBuffer(CharBuffer buffer) {
        while (buffer.hasRemaining()) {
            System.out.print(buffer.get());
        }
        System.out.println ("");
    }

    private static boolean fillBuff(CharBuffer buffer) {
        if (index >= strings.length) {
            return false;
        }
        String string = strings[index++];
        for (int i = 0; i < string.length(); i++) {
            buffer.put(string.charAt(i));
        }
        return true;
    }
}
```
