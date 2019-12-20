---
layout: post
title: 桶排序
date: 2019-12-20
categories:
    - 算法
comments: true
permalink: bucket-sort.html
---

桶排序（Bucket sort）或所谓的箱排序，是一个排序算法，工作的原理是将数组分到有限数量的桶里。每个桶再个别排序（有可能再使用别的排序算法或是以递归方式继续使用桶排序进行排序）。桶排序是鸽巢排序的一种归纳结果。当要被排序的数组内的数值是均匀分配的时候，桶排序使用线性时间`O( n )`。但桶排序并不是比较排序，他不受到` O ( n log ⁡ n )下限的影响。 

> 维基百科

桶排序是计数排序的升级版,它利用了函数的映射关系，高效与否的关键就在于这个映射函数的确定。为了使桶排序更加高效，我们需要做到这两点：

- 在额外空间充足的情况下，尽量增大桶的数量
- 使用的映射函数能够将输入的 N 个数据均匀的分配到 K 个桶中

同时，对于桶中元素的排序，选择何种比较排序算法对于性能的影响至关重要。

算法步骤

1. 找到最大元素与最小元素
2. 根据桶的个数计算每个桶的范围
3. 根据数值大小分配到不同的桶
4. 对桶内元素进行排序
5. 按照桶范围依次输出桶内元素。

![](/assets/images/posts/sorting-algorithm/bucket-sort-1.png)

源码

```
  public void sort(int[] array) {
    int max = array[0], min = array[0];
    for (int a : array) {
      if (max < a) {
        max = a;
      }
      if (min > a) {
        min = a;
      }
    }
    //根据最大值与最小值，将不同大小的数分配到不同的桶中
    int bucketCount = (int) Math.floor((max - min) / bucketStep) + 1;
    int[][] buckets = new int[bucketCount][0];
    // 利用映射函数将数据分配到各个桶中
    for (int i = 0; i < array.length; i++) {
      //计算该元素所属范围，放置到对应的桶里面
      int index = (int) Math.floor((array[i] - min) / bucketStep);
      buckets[index] = arrAppend(buckets[index], array[i]);
    }

    int arrIndex = 0;
    for (int[] bucket : buckets) {
      if (bucket.length <= 0) {
        continue;
      }
      // 对每个桶进行排序，这里使用了插入排序
      new InsertSortAlgorithm().sort(bucket);
      for (int value : bucket) {
        array[arrIndex++] = value;
      }
    }
  }

  private int[] arrAppend(int[] arr, int value) {
    arr = Arrays.copyOf(arr, arr.length + 1);
    arr[arr.length - 1] = value;
    return arr;
  }
```



# 参考资料

https://zh.wikipedia.org/wiki/%E6%A1%B6%E6%8E%92%E5%BA%8F

https://www.runoob.com/w3cnote/bucket-sort.html