---
layout: post
title: java的零拷贝实现
date: 2019-10-31
categories:
    - java
comments: true
permalink: java-zero-copy.html
---

在了解java零拷贝实现之前首先需要了解Linux零拷贝的知识

- [用户空间和内核空间](https://edgar615.github.io/linux-user-kernel-space.html)
- [Linux IO](https://edgar615.github.io/linux-io.html)

在 Java NIO 中的通道（Channel）就相当于操作系统的内核空间（kernel space）的缓冲区。

而缓冲区（Buffer）对应的相当于操作系统的用户空间（user space）中的用户缓冲区（user buffer）：

- 通道（Channel）是全双工的（双向传输），它既可能是读缓冲区（read buffer），也可能是网络缓冲区（socket buffer）。
- 缓冲区（Buffer）分为堆内存（HeapBuffer）和堆外内存（DirectBuffer），这是通过 malloc() 分配出来的用户态内存。

堆外内存（DirectBuffer）在使用后需要应用程序手动回收，而堆内存（HeapBuffer）的数据在 GC 时可能会被自动回收。因此，在使用 HeapBuffer 读写数据时，为了避免缓冲区数据因为 GC 而丢失，NIO 会先把 HeapBuffer 内部的数据拷贝到一个临时的 DirectBuffer 中的本地内存（native memory）。这个拷贝涉及到 `sun.misc.Unsafe.copyMemory()` 的调用，背后的实现原理与 memcpy() 类似。 最后，将临时生成的 DirectBuffer 内部的数据的内存地址传给 I/O 调用函数，这样就避免了再去访问 Java 对象处理 I/O 读写。

# MappedByteBuffer

MappedByteBuffer 是 NIO 基于内存映射（mmap）这种零拷贝方式提供的一种实现，它继承自 ByteBuffer。FileChannel 定义了一个 map() 方法，它可以把一个文件从 position 位置开始的 size 大小的区域映射为内存映像文件。

```
public abstract MappedByteBuffer map(MapMode mode,
									 long position, long size)
	throws IOException;
```

- mode：根据是按只读、读取/写入或专用（写入时拷贝）来映射文件，分别为`FileChannel.MapMode`类中所定义的`READ_ONLY`、`READ_WRITE` 或`PRIVATE`之一
- position：文件中的位置，映射区域从此位置开始；必须为非负数
- size： 要映射的区域大小；必须为非负数且不大于 Integer.MAX_VALUE

所以若想读取文件后半部分内容，需要这样写

```
MappedByteBuffer inputBuffer = new RandomAccessFile(file, "r")
        .getChannel().map(FileChannel.MapMode.READ_ONLY,
            file.length() / 2, file.length()/ 2);
```

若想读取文本后1/8内容，需要这样写
```
map(FileChannel.MapMode.READ_ONLY, f.length()*7/8,f.length()/8)
```

想读取文件所有内容，需要这样写

```
map(FileChannel.MapMode.READ_ONLY, 0,f.length())
```

MappedByteBuffer 相比 ByteBuffer 新增了三个重要的方法：

- fore()：对于处于 READ_WRITE 模式下的缓冲区，把对缓冲区内容的修改强制刷新到本地文件
- load()：将缓冲区的内容载入物理内存中，并返回这个缓冲区的引用
- isLoaded()：如果缓冲区的内容在物理内存中，则返回 true，否则返回 false

写文件数据：打开文件通道 fileChannel 并提供读权限、写权限和数据清空权限，通过 fileChannel 映射到一个可写的内存缓冲区 mappedByteBuffer，将目标数据写入 mappedByteBuffer，通过 force() 方法把缓冲区更改的内容强制写入本地文件。

```
Path path = Paths.get(getClass().getResource(FILE_NAME).getPath());
byte[] bytes = CONTENT.getBytes(Charset.forName(CHARSET));
try (FileChannel fileChannel = FileChannel.open(path, StandardOpenOption.READ,
		StandardOpenOption.WRITE, StandardOpenOption.TRUNCATE_EXISTING)) {
	MappedByteBuffer mappedByteBuffer = fileChannel.map(READ_WRITE, 0, bytes.length);
	if (mappedByteBuffer != null) {
		mappedByteBuffer.put(bytes);
		mappedByteBuffer.force();
	}
} catch (IOException e) {
	e.printStackTrace();
}
```

读文件数据：打开文件通道 fileChannel 并提供只读权限，通过 fileChannel 映射到一个只可读的内存缓冲区 mappedByteBuffer，读取 mappedByteBuffer 中的字节数组即可得到文件数据。

```
Path path = Paths.get(getClass().getResource(FILE_NAME).getPath());
int length = CONTENT.getBytes(Charset.forName(CHARSET)).length;
try (FileChannel fileChannel = FileChannel.open(path, StandardOpenOption.READ)) {
	MappedByteBuffer mappedByteBuffer = fileChannel.map(READ_ONLY, 0, length);
	if (mappedByteBuffer != null) {
		byte[] bytes = new byte[length];
		mappedByteBuffer.get(bytes);
		String content = new String(bytes, StandardCharsets.UTF_8);
	}
} catch (IOException e) {
	e.printStackTrace();
}
```
下面总结一下 MappedByteBuffer 的特点和不足之处：

- MappedByteBuffer 使用是堆外的虚拟内存，因此分配（map）的内存大小不受 JVM 的 -Xmx 参数限制，但是也是有大小限制的。
- 如果当文件超出 Integer.MAX_VALUE 字节限制时，可以通过 position 参数重新 map 文件后面的内容。
- MappedByteBuffer 在处理大文件时性能的确很高，但也存在内存占用、文件关闭不确定等问题，被其打开的文件只有在垃圾回收的才会被关闭，而且这个时间点是不确定的。
- MappedByteBuffer 提供了文件映射内存的 mmap() 方法，也提供了释放映射内存的 unmap() 方法。然而 unmap() 是 FileChannelImpl 中的私有方法，无法直接显示调用。    因此，用户程序需要通过 Java 反射的调用 sun.misc.Cleaner 类的 clean() 方法手动释放映射占用的内存区域。

```
public static void clean(final Object buffer) throws Exception {
    AccessController.doPrivileged((PrivilegedAction<Void>) () -> {
        try {
            Method getCleanerMethod = buffer.getClass().getMethod("cleaner", new Class[0]);
            getCleanerMethod.setAccessible(true);
            Cleaner cleaner = (Cleaner) getCleanerMethod.invoke(buffer, new Object[0]);
            cleaner.clean();
        } catch(Exception e) {
            e.printStackTrace();
        }
    });
}
```

# FileChannel
FileChannel 是一个用于文件读写、映射和操作的通道，同时它在并发环境下是线程安全的。基于 FileInputStream、FileOutputStream 或者 RandomAccessFile 的 getChannel() 方法可以创建并打开一个文件通道。FileChannel 定义了 transferFrom() 和 transferTo() 两个抽象方法，它通过在通道和通道之间建立连接实现数据传输的。

transferTo()：通过 FileChannel 把文件里面的源数据写入一个 WritableByteChannel 的目的通道。

```
public abstract long transferTo(long position, long count,
								WritableByteChannel target)
	throws IOException;
```

transferFrom()：把一个源通道 ReadableByteChannel 中的数据读取到当前 FileChannel 的文件里面。

```
public abstract long transferFrom(ReadableByteChannel src,
								  long position, long count)
	throws IOException;
```

示例代码

```
@Test
public void transferTo() throws Exception {
try (FileChannel fromChannel = new RandomAccessFile(fromFile, "rw").getChannel();
	FileChannel toChannel = new RandomAccessFile(
		toFile, "rw").getChannel()) {
  long position = 0L;
  long offset = fromChannel.size();
  fromChannel.transferTo(position, offset, toChannel);
}
}

@Test
public void transferFrom() throws Exception {
try (FileChannel fromChannel = new RandomAccessFile(
	fromFile, "rw").getChannel();
	FileChannel toChannel = new RandomAccessFile(
		toFile, "rw").getChannel()) {
  long position = 0L;
  long offset = fromChannel.size();
  toChannel.transferFrom(fromChannel, position, offset);
}
}
```
# 参考资料

https://mp.weixin.qq.com/s/mZujKx1bKl1T6gEI1s400Q