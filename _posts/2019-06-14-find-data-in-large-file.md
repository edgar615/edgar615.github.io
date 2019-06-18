---
layout: post
title: 面试题-从大文件中找出一个最大的100个树
date: 2019-06-11
categories:
    - 算法
comments: true
permalink: find-data-in-large-file.html
---

说来惭愧，在十年前上学的时候网上认识的一个大哥（断联系很多年了）给我出了一个类似的题目，这么多年我虽然模糊的知道这么做，却从来没有上手做过。这次整理笔记的时候决定认真的做一下。

这类面试题有

- 有一个100G大小的文件里存的全是数字，并且每个数字见用逗号隔开。现在在这一大堆数字中找出100个最大的数出来
- 有一个1G大小的一个文件，里面每一行是一个词，词的大小不超过16字节，内存限制大小是1M，要求返回频数最高的100个词
- 给定a、b两个文件，各存放50亿个url，每个url各占64字节，内存限制是4G，让你找出a、b文件共同的url?

对于这类题目，思路基本一致：**分解大问题，解决小问题，从局部最优中选择全局最优；**

对于大文件首先我们要进行分解，分解的常用方法是HASH
```
hash(x)%m
```
其中x为字符串/url/ip，m为小问题的数目，比如把一个大文件分解为1000份，m=1000；

然后找到解决问题的辅助数据结构和排序算法，

常用的数据结构有`hash_map`，`Trie树`，`bit map`，`二叉排序树（AVL，SBT，红黑树）`

对于`top K`问题：最大K个用最小堆，最小K个用最大堆，对于大数据排序用户`快速排序`,`堆排序`,`归并排序`,`桶排序`

# 内存映射文件
对于将大文件拆分为小文件，我们还需要考虑内存和I/O瓶颈，比如说我们可定不可能将100G的文件全部读取到内存中，内存映射就是用来处理这类问题的

- 不要复制文件中所有的数据，只需要修改文件中局部的数据。
- 并行/分段处理大文件。 

内存映射文件能让你创建和修改那些因为太大而无法放入内存的文件。有了内存映射文件，你就可以认为文件已经全部读进了内存，然后把它当成一个非常大的数组来访问。这种解决办法能大大简化修改文件的代码。

`fileChannel.map(FileChannel.MapMode mode, long position, long size)`将此通道的文件区域直接映射到内存中。注意，你必须指明，它是从文件的哪个位置开始映射的，映射的范围又有多大；也就是说，它还可以映射一个大 文件的某个小片断

下面的代码演示了如何通过内存映射读取文件内的一段内容

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

未完待续

参考资料

https://www.cnblogs.com/wuxiangli/p/5639011.html

https://blog.csdn.net/fycy2010/article/details/46945641

https://blog.csdn.net/tiankong_/article/details/77234726

https://blog.csdn.net/gongpulin/article/details/81137548

https://www.cnblogs.com/CheeseZH/p/5283390.html

https://blog.51cto.com/13722730/2112462