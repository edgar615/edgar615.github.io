---
layout: post
title: NIO的Buffer
date: 2020-6-11
categories:
    - netty
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


# 2. Buffer的方法

## 2.1. Allocating a Buffer
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

## 2.2 读取/写入

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

```java
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

## 2.3. 翻转
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

## 2.4. rewind()
rewind()将position设为0，不影响limit，可以重新读取数据。

## 2.5. 释放
布尔函数hasRemaining()会在释放缓冲区时告诉您是否已经达到缓冲区的上界。

```java
for (int i = 0; buffer.hasRemaining( ), i++) {
    myByteArray [i] = buffer.get( );
}
```

一旦缓冲区对象完成填充并释放，它就可以被重新使用了。Clear()函数将缓冲区重置为空状态。它并不改变缓冲区中的任何数据元素，而是仅仅将上界设为容量的值，并把位置设回0

## 2.6. 清除 clear()
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

## 2.7. 压缩 compact()
有时，您可能只想从缓冲区中释放一部分数据，而不是全部，然后重新填充。为了实现这一点，未读的数据元素需要下移以使第一个元素索引为0。尽管重复这样做会效率低下，但这有时非常必要，而API对此为您提供了一个compact()函数。这一缓冲区工具在复制数据时要比您使用get()和put()函数高效得多

*被部分释放的缓冲区*
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
        <td>potision(2)</td>
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

调用<code>buffer.compact();</code>后，缓冲区变为
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
        <td>l</td>
        <td>l</td>
        <td>o</td>
        <td>w</td>
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
        <td>potision(4)</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td>limit(10)、capacity(10)</td>
    </tr>
</table>

您会看到数据元素2-5被复制到0-3位置。位置4和5不受影响，但现在正在或已经超出了当前位置，因此是“死的”。它们可以被之后的put()调用重写。还要注意的是，位置已经被设为被复制的数据元素的数目。也就是说，缓冲区现在被定位在缓冲区中最后一个“存活”元素后插入数据的位置。最后，上界属性被设置为容量的值，因此缓冲区可以被再次填满。

**调用compact()的作用是丢弃已经释放的数据，保留未释放的数据，并使缓冲区对重新填充容量准备就绪**

        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
    </tr>
    <tr>
        <td>slice</td>
        <td></td>
        <td></td>
        <td></td>
        <td>mark(x)、potision(0)</td>
        <td></td>
        <td>limit(2)、capacity(2)</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
    </tr>
</table>

# 3. 字节缓冲区
## 3.1. 字节顺序
大端字节顺序、小端字节顺序

ByteBuffer的字符顺序设定可以随时通过调用以ByteOrder.BIG_ENDIAN或ByteOrder.LITTL_ENDIAN为参数的order()函数来改变。

```java
public final ByteOrder order()
public final ByteBuffer order(ByteOrder bo)
```

如果一个缓冲区被创建为一个ByteBuffer对象的视图，那么`order()`返回的数值就是视图被创建时其创建源头的ByteBuffer的字节顺序设定。视图的字节顺序设定在创建后不能被改变，而且如果原始的字节缓冲区的字节顺序在之后被改变，它也不会受到影响

## 3.2. 直接缓冲区
字节缓冲区跟其他缓冲区类型最明显的不同在于，它们可以成为通道所执行的I/O的源头或目标。

出于这一原因，引入了直接缓冲区的概念。直接缓冲区被用于与通道和固有I/O例程交互。它们通过使用固有代码来告知操作系统直接释放或填充内存区域，对用于通道直接或原始存取的内存区域中的字节元素的存储尽了最大的努力。

直接字节缓冲区通常是I/O操作最好的选择。在设计方面，它们支持JVM可用的最高效I/O机制。非直接字节缓冲区可以被传递给通道，但是这样可能导致性能损耗。通常非直接缓冲不可能成为一个本地I/O操作的目标。如果您向一个通道中传递一个非直接ByteBuffer对象用于写入，通道可能会在每次调用中隐含地进行下面的操作：

    1.创建一个临时的直接ByteBuffer对象。
    2.将非直接缓冲区的内容复制到临时缓冲中。
    3.使用临时缓冲区执行低层次I/O操作。
    4.临时缓冲区对象离开作用域，并最终成为被回收的无用数据。

直接缓冲区是I/O的最佳选择，但可能比创建非直接缓冲区要花费更高的成本。直接缓冲区使用的内存是通过调用本地操作系统方面的代码分配的，绕过了标准JVM堆栈。建立和销毁直接缓冲区会明显比具有堆栈的缓冲区更加破费，这取决于主操作系统以及JVM实现。直接缓冲区的内存区域不受无用存储单元收集支配，因为它们位于标准JVM堆栈之外。

使用直接缓冲区或非直接缓冲区的性能权衡会因JVM，操作系统，以及代码设计而产生巨大差异。通过分配堆栈外的内存，您可以使您的应用程序依赖于JVM未涉及的其它力量。当加入其他的移动部分时，确定您正在达到想要的效果。我以一条旧的软件行业格言建议您：先使其工作，再加快其运行。不要一开始就过多担心优化问题；首先要注重正确性。JVM实现可能会执行缓冲区缓存或其他的优化，这会在不需要您参与许多不必要工作的情况下为您提供所需的性能。

直接ByteBuffer是通过调用具有所需容量的`ByteBuffer.allocateDirect()`函数产生的，就像我们之前所涉及的`allocate()`函数一样。注意用一个`wrap()`函数所创建的被包装的缓冲区总是非直接的。

```java
public static ByteBuffer allocateDirect(int capacity)
public abstract boolean isDirect();
```

## 3.3. 视图缓冲区
视图缓冲区通过已存在的缓冲区对象实例的工厂方法来创建。这种视图对象维护它自己的属性，容量，位置，上界和标记，但是和原来的缓冲区共享数据元素。ByteBuffer类允许创建视图来将byte型缓冲区字节数据映射为其它的原始数据类型。例如，`asLongBuffer()`函数创建一个将八个字节型数据当成一个long型数据来存取的视图缓冲区。

下面列出的每一个工厂方法都在原有的ByteBuffer对象上创建一个视图缓冲区。调用其中的任何一个方法都会创建对应的缓冲区类型，这个缓冲区是基础缓冲区的一个切分，由基础缓冲区的位置和上界决定。新的缓冲区的容量是字节缓冲区中存在的元素数量除以视图类型中组成一个数据类型的字节数。在切分中任一个超过上界的元素对于这个视图缓冲区都是不可见的。视图缓冲区的第一个元素从创建它的ByteBuffer对象的位置开始（`positon()`函数的返回值）。具有能被自然数整除的数据元素个数的视图缓冲区是一种较好的实现

```java
public abstract CharBuffer asCharBuffer();
public abstract DoubleBuffer asDoubleBuffer();
public abstract FloatBuffer asFloatBuffer();
public abstract IntBuffer asIntBuffer();
public abstract LongBuffer asLongBuffer();
public abstract ShortBuffer asShortBuffer();
```

*创建一个ByteBuffer的字符视图*

```java
public class ByteCharView {

    public static void main(String[] args) {
        ByteBuffer byteBuffer = ByteBuffer.allocate(7).order(ByteOrder.BIG_ENDIAN);
        CharBuffer charBuffer = byteBuffer.asCharBuffer();

        byteBuffer.put(0, (byte) 0);
        byteBuffer.put(1, (byte) 'H');
        byteBuffer.put(2, (byte) 0);
        byteBuffer.put(3, (byte) 'i');
        byteBuffer.put(4, (byte) 0);
        byteBuffer.put(5, (byte) '!');
        byteBuffer.put(6, (byte) 0);

        println(byteBuffer);
        println(charBuffer);

    }

    private static void println(Buffer buffer) {
        System.out.println("pos=" + buffer.position() + ", limit=" + buffer.limit() + ", capacity=" + buffer.capacity() + ": '" + buffer.toString() + "'");
    }
}
```

输出：

```
pos=0, limit=7, capacity=7: 'java.nio.HeapByteBuffer[pos=0 lim=7 cap=7]'
pos=0, limit=3, capacity=3: 'Hi!'
```

## 3.4. 数据元素视图

```java
public abstract class ByteBuffer extends Buffer implements Comparable {
    public abstract char getChar();

    public abstract char getChar(int index);

    public abstract short getShort();

    public abstract short getShort(int index);

    public abstract int getInt();

    public abstract int getInt(int index);

    public abstract long getLong();

    public abstract long getLong(int index);

    public abstract float getFloat();

    public abstract float getFloat(int index);

    public abstract double getDouble();

    public abstract double getDouble(int index);

    public abstract ByteBuffer putChar(char value);

    public abstract ByteBuffer putChar(int index, char value);

    public abstract ByteBuffer putShort(short value);

    public abstract ByteBuffer putShort(int index, short value);

    public abstract ByteBuffer putInt(int value);

    public abstract ByteBuffer putInt(int index, int value);

    public abstract ByteBuffer putLong(long value);

    public abstract ByteBuffer putLong(int index, long value);

    public abstract ByteBuffer putFloat(float value);

    public abstract ByteBuffer putFloat(int index, float value);

    public abstract ByteBuffer putDouble(double value);

    public abstract ByteBuffer putDouble(int index, double value);
}
```

这些函数从当前位置开始存取ByteBuffer的字节数据，就好像一个数据元素被存储在那里一样。根据这个缓冲区的当前的有效的字节顺序，这些字节数据会被排列或打乱成需要的原始数据类型。比如说，如果`getInt()`函数被调用，从当前的位置开始的四个字节会被包装成一个int类型的变量然后作为函数的返回值返回。

包含一些数据的ByteBuffer

<table>
    <tr>
        <td></td>
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
        <td>07</td>
        <td>3B</td>
        <td>C5</td>
        <td>31</td>
        <td>5E</td>
        <td>94</td>
        <td>D6</td>
        <td>04</td>
        <td></td>
        <td></td>
        <td></td>
    </tr>
    <tr>
        <td></td>
        <td>mark(x)</td>
        <td>potision(1)</td>
        <td></td>
        <td></td>
        <td></td>
        <td>limit(5)</td>
        <td></td>
        <td></td>
        <td></td>
        <td></td>
        <td>capacity(10)</td>
    </tr>
</table>

`int value = buffer.getInt( );`

会返回一个由缓冲区中位置1-4的byte数据值组成的int型变量的值。实际的返回值取决于缓冲区的当前的比特排序（byte-order）设置。更具体的写法是：
`int value = buffer.order (ByteOrder.BIG_ENDIAN).getInt( ); `这将会返回值`0x3BC5315E`，而`int value = buffer.order (ByteOrder.LITTLE_ENDIAN).getInt( ); `返回值`0x5E31C53B`。

## 3.5. 内存映射缓冲区
映射缓冲区是带有存储在文件，通过内存映射来存取数据元素的字节缓冲区。映射缓冲区通常是直接存取内存的，只能通过FileChannel类创建。映射缓冲区的用法和直接缓冲区类似，但是MappedByteBuffer对象可以处理独立于文件存取形式的的许多特定字符
