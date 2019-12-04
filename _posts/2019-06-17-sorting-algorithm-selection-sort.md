---
layout: post
title: 选择排序
date: 2019-06-17
categories:
    - 算法
comments: true
permalink: selection-sort.html
---

选择排序是一种简单直观的排序算法，无论什么数据进去都是 O(n²) 的时间复杂度。所以用到它的时候，数据规模越小越好。唯一的好处可能就是不占用额外的内存空间。

### 算法步骤

1. 首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置。
2. 再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。
3. 重复第二步，直到所有元素均排序完毕。

由于选择排序一次能确定一个元素的位置，所以选择排序需要循环size-1次。

下面看一下图示

第一趟

![](/assets/images/posts/sorting-algorithm/selection-sort-1.png)

第二趟

![](/assets/images/posts/sorting-algorithm/selection-sort-2.png)

第三趟

![](/assets/images/posts/sorting-algorithm/selection-sort-2.png)

第四趟

![](/assets/images/posts/sorting-algorithm/selection-sort-4.png)

第五趟

![](/assets/images/posts/sorting-algorithm/selection-sort-5.png)


上代码

```
  private static void sort(int[] array) {
    int len = array.length;
    for (int i = 0; i < len; i ++) {
      int minIndex = i;
      for (int j = i+1; j < len; j ++) {
        if (array[minIndex] > array[j]) {
          minIndex = j;
        }
      }
      if (minIndex != i) {
        int temp = array[minIndex];
        array[minIndex] = array[i];
        array[i] = temp;
      }
    }
  }
```