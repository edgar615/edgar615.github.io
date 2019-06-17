---
layout: post
title: 冒泡排序
date: 2019-06-17
categories:
    - 算法
comments: true
permalink: sorting-algorithm-bubble-sort.html
---

冒泡排序（Bubble 
Sort）也是一种简单直观的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢"浮"到数列的顶端。

### 算法步骤

1. 比较相邻的元素。如果第一个比第二个大，就交换他们两个
2. 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对。这步做完后，最后的元素会是最大的数
3. 针对所有的元素重复以上的步骤，除了最后一个
4. 持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。

下面看一下图示

第一趟

![](/assets/images/posts/sorting-algorithm/bubble-sort-1.png)

第二趟

![](/assets/images/posts/sorting-algorithm/bubble-sort-2.png)

第三趟

![](/assets/images/posts/sorting-algorithm/bubble-sort-2.png)

第四趟

![](/assets/images/posts/sorting-algorithm/bubble-sort-4.png)

第五趟

![](/assets/images/posts/sorting-algorithm/bubble-sort-5.png)

通过上面的图已经可以很直观的了解冒泡排序了

> 当输入的数据已经是正序时，速度最快O(n)，当输入的数据是反序时，速度最慢O(n^n)

上代码

```
private static int[] sort(int[] array) {
int len = array.length;
for (int i = 1; i < len; i ++) {
  for (int j = 0; j < len-1; j ++) {
	if (array[j] > array[j+1]) {
	  int temp = array[j];
	  array[j] = array[j+1];
	  array[j+1] = temp;
	}
  }
}
return array;
}
```

仔细观察上面的代码，会发现即使数组在某次循环已经排序完成，依然会继续执行循环。为了优化这种情况，我们可以增加一个标记来表明此次循环是否进行了交换，如果没有交换说明排序已经完成。优化后的代码如下

<pre class="line-numbers "><code class="language-java">
private static int[] sort(int[] array) {
int len = array.length;
for (int i = 1; i < len; i ++) {
  boolean complete = true;
  for (int j = 0; j < len-1; j ++) {
	if (array[j] > array[j+1]) {
	  int temp = array[j];
	  array[j] = array[j+1];
	  array[j+1] = temp;
	  complete = false;
	}
  }
  if (complete) {
	break;
  }
}
return array;
}
</code></pre>
