---
layout: post
title: java读写大文件
date: 2019-03-14
categories:
    - java
comments: true
permalink: java-large-file.html
---

# Scanner读取大文件

这种方案将会遍历文件中的所有行——允许对每一行进行处理，而不保持对它的引用

<pre class="line-numbers "><code class="language-java">
FileInputStream inputStream = null;
Scanner sc = null;
try {
  inputStream = new FileInputStream("big_file");
  sc = new Scanner(inputStream, "UTF-8");
  while (sc.hasNextLine()) {
    String line = sc.nextLine();
    System.out.println(line);
  }
  // note that Scanner suppresses exceptions
  if (sc.ioException() != null) {
    throw sc.ioException();
  }
} finally {
  if (inputStream != null) {
    inputStream.close();
  }
  if (sc != null) {
    sc.close();
  }
}
</code></pre>

# 使用BufferedReader读取大文件

<pre class="line-numbers "><code class="language-java">
    File file = new File(filePath);
    BufferedReader reader = null;
    try {
      reader = new BufferedReader(new FileReader(file), 5 * 1024 * 1024);
      String tempString = null;
      while ((tempString = reader.readLine()) != null) {
        System.out.println(tempString);
      }
      reader.close();
    } catch (IOException e) {
      e.printStackTrace();
    } finally {
      if (reader != null) {
        try {
          reader.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
</code></pre>

# NIO内存映射

内存映射文件和标准IO操作最大的不同之处就在于它虽然最终也是要从磁盘读取数据，但是它并不需要将数据读取到OS内核缓冲区，而是直接将进程的用户私有地址空间中的一 部分区域与文件对象建立起映射关系，就好像直接从内存中读、写文件一样，速度当然快了。 

内存映射IO最大的优点可能在于性能，这对于建立高频电子交易系统尤其重要。内存映射文件通常比标准通过正常IO访问文件要快。另一个巨大的优势是内存映 射IO允许加载不能直接访问的潜在巨大文件  。经验表明，内存映射IO在大文件处理方面性能更加优异。尽管它也有不足——增加了页面错误的数目。由于操作系统只将一部分文件加载到内存，如果一个请求 页面没有在内存中，它将导致页面错误。同样它可以被用来在两个进程中共享数据。 

<pre class="line-numbers "><code class="language-java">
	public static void readFile3(String path) {
		File file = new File("e:/test.dat");
    final int BUFFER_SIZE = 0x300000;// 缓冲区大小为3M

    /**
     *
     * map(FileChannel.MapMode mode,long position, long size)
     *
     * mode - 根据是按只读、读取/写入或专用（写入时拷贝）来映射文件，分别为 FileChannel.MapMode 类中所定义的
     * READ_ONLY、READ_WRITE 或 PRIVATE 之一
     *
     * position - 文件中的位置，映射区域从此位置开始；必须为非负数
     *
     * size - 要映射的区域大小；必须为非负数且不大于 Integer.MAX_VALUE
     *
     * 所以若想读取文件后半部分内容，如例子所写；若想读取文本后1/8内容，需要这样写map(FileChannel.MapMode.READ_ONLY,
     * f.length()*7/8,f.length()/8)
     *
     * 想读取文件所有内容，需要这样写map(FileChannel.MapMode.READ_ONLY, 0,f.length())
     *
     */
    MappedByteBuffer inputBuffer = new RandomAccessFile(file, "r")
        .getChannel().map(FileChannel.MapMode.READ_ONLY,
            file.length() / 2, file.length()/ 2);
    
    byte[] dst = new byte[BUFFER_SIZE];// 每次读出3M的内容
    // 一次读取3M，不能超过内存映射的实际大小
    for (int offset = 0; offset < inputBuffer.capacity(); offset += BUFFER_SIZE) {
      if (inputBuffer.capacity() - offset >= BUFFER_SIZE) {
        //说明缓冲区读满
        for (int i = 0; i < BUFFER_SIZE; i++) {
          dst[i] = inputBuffer.get(offset + i);
        }
      } else {
        //说明缓冲区未满，用实际大小
        for (int i = 0; i < inputBuffer.capacity() - offset; i++)
          dst[i] = inputBuffer.get(offset + i);
      }
      int length = (inputBuffer.capacity() % BUFFER_SIZE == 0) ? BUFFER_SIZE
          : inputBuffer.capacity() % BUFFER_SIZE;
      System.out.println(new String(dst, 0, length));
    } 
</code></pre>

# 使用BufferedWriter写大文件

<pre class="line-numbers "><code class="language-java">
File file = new File(filePath);
FileWriter fw = null;
try {
  if (!file.exists()) {
    file.createNewFile();
  }
  fw = new FileWriter(file.getAbsoluteFile());
  BufferedWriter bw = new BufferedWriter(fw);
  bw.write(fileContent);
  bw.close();
} catch (IOException e) {
  e.printStackTrace();
} finally {
  try {
    fw.close();
  } catch (IOException e) {
    e.printStackTrace();
  }
}
</code></pre>