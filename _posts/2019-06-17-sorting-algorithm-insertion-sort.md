---
layout: post
title: 插入排序
date: 2019-06-17
categories:
    - 算法
comments: true
permalink: insertion-sort.html
---

插入排序的代码实现虽然没有冒泡排序和选择排序那么简单粗暴，但它的原理应该是最容易理解的了，因为只要打过扑克牌的人都应该能够秒懂。插入排序是一种最简单直观的排序算法，它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。

插入排序和冒泡排序一样，也有一种优化算法，叫做拆半插入。

### 算法步骤

1. 将第一待排序序列第一个元素看做一个有序序列，把第二个元素到最后一个元素当成是未排序序列
2. 从头到尾依次扫描未排序序列，将扫描到的每个元素插入有序序列的适当位置。（如果待插入的元素与有序序列中的某个元素相等，则将待插入元素插入到相等元素的后面。）

下面看一下图示

![](/assets/images/posts/sorting-algorithm/insertion-sort-1.png)


上代码

```
  private static void sort(int[] array) {

    int len = array.length;
    for (int i = 1; i < len; i ++) {
      int insertion = array[i];
      // j用来定义有序数组
      int j = i;
      while (j > 0 && array[j-1] > insertion) {
        array[j] = array[j - 1];
        j --;
      }
      if (i != j) {
        array[j] = insertion;
      }
    }
  }
```

由于在排序算法中，前半部分的有序序列已经排好序，所以我们不必按顺序查找插入点，可以直接看着二分查找法来加快找到插入点

```
  private static int binaraySearch(int[] array, int high, int target) {
    int low = 0;
    while (low <= high) {
      int middle = low + (high - low) / 2;
      if (target > array[middle]) {
        low = middle + 1;
      } else {
        high = middle - 1;
      }
    }
    return low;
  }

  private static void sort(int[] array) {

    int len = array.length;
    for (int i = 1; i < len; i++) {
      int insertion = array[i];
      int low = binaraySearch(array, i - 1, insertion);
      System.out.println(low);
      // 插入位置之后的元素依次向后移动
      for (int j = i; j > low; j--) {
        array[j] = array[j - 1];
      }
      array[low] = insertion;
    }
  }
```