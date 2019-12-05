---
layout: post
title: 归并排序
date: 2019-12-05
categories:
    - 算法
comments: true
permalink: sorting-algorithm-merge-sort.html
---

摘自维基百科

> 归并排序（英语：Merge sort，或mergesort），是创建在归并操作上的一种有效的排序算法，效率为`O(n\log n)`。1945年由约翰·冯·诺伊曼首次提出。该算法是采用分治法（Divide and Conquer）的一个非常典型的应用，且各层分治递归可以同时进行

归并操作（merge），也叫归并算法，指的是将两个已经排序的序列合并成一个序列的操作。归并排序算法依赖归并操作。

归并操作有两种实现方式

**递归法（Top-down）**

1. 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列
2. 设定两个指针，最初位置分别为两个已经排序序列的起始位置
3. 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置
4. 重复步骤3直到某一指针到达序列尾
5. 将另一序列剩下的所有元素直接复制到合并序列尾

单次归并的操作如图所示

![](/assets/images/posts/sorting-algorithm/merge-sort-1.png)

**迭代法（Bottom-up）**

原理如下（假设序列共有n个元素）：

1. 将序列每相邻两个数字进行归并操作，形成`ceil(n/2)`个序列，排序后每个序列包含两/一个元素
2. 若此时序列数不是1个则将上述序列再次归并，形成`ceil(n/4)`个序列，每个序列包含四/三个元素
3. 重复步骤2，直到所有元素排序完毕，即序列数为1

单次归并的操作如图所示

![](/assets/images/posts/sorting-algorithm/merge-sort-2.png)

递归法的代码

```
  public <T extends Comparable<? super T>> void sort(List<T> list) {
    int len = list.size();
    if (len < 2) {
      return;
    }
    List<T> aux = new ArrayList<>(len);
    for (int i = 0; i < len; i ++) {
      aux.add(null);
    }
    mergeSort(list, 0, len - 1, aux);
  }

  private <T extends Comparable<? super T>> void mergeSort(List<T> list, int low, int high,
      List<T> aux) {
    // 终止递归的条件
    if (low == high) {
      return;
    }
    int mid = (int) Math.floor((high - low) / 2) + low;
    mergeSort(list, low, mid, aux);  // 对左半边递归
    mergeSort(list, mid + 1, high, aux);  // 对右半边递归
    merge(list, low, mid, high, aux);  // 单趟合并
  }


  protected  <T extends Comparable<? super T>> void merge(List<T> list, int low, int mid, int high,
      List<T> aux) {
    int i = low;
    int j = mid + 1;
    int k = low;
    while (i <= mid && j <= high) {
      T left = list.get(i);
      T right = list.get(j);
      if (left.compareTo(right) <= 0) {
        aux.set(k, left);
        i ++;
      } else {
        aux.set(k, right);
        j ++;
      }
      k++;
    }
    while (i <= mid) {
      aux.set(k, list.get(i));
      k++;
      i++;
    }
    while (j <= high) {
      aux.set(k, list.get(j));
      k++;
      j++;
    }
    // 将排好序的序列放回源队列
    for (int start = low; start <= high; start ++) {
      list.set(start, aux.get(start));
    }
  }

```

迭代法

```
  public <T extends Comparable<? super T>> void sort(List<T> list) {
    int len = list.size();
    if (len < 2) {
      return;
    }
    List<T> aux = new ArrayList<>(len);
    for (int i = 0; i < len; i ++) {
      aux.add(null);
    }
    for (int subLen = 1; subLen < len; subLen = subLen + subLen) {
      // 相邻归并
      for (int low = 0; low < len; low = low + subLen + subLen) {
        int high = Math.min(low + subLen + subLen - 1, len - 1);
        int mid = (int) Math.floor((subLen + subLen - 1) / 2) + low;
        mid = Math.min(mid, high);
        merge(list, low, mid, high, aux);
      }
    }
  }
```

# 参考资料

