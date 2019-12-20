---
layout: post
title: 基数排序
date: 2019-12-20
categories:
    - 算法
comments: true
permalink: radix-sort.html
---

**基数排序**（英语：Radix sort）是一种非比较型整数[排序算法，其原理是将整数按位数切割成不同的数字，然后按每个位数分别比较。由于整数也可以表达字符串（比如名字或日期）和特定格式的浮点数，所以基数排序也不是只能使用于整数。

> 维基百科

算法步骤：将所有待比较数值（正整数）统一为同样的数字长度，数字较短的数前面补零。然后，从最低位开始，依次进行一次排序。这样从最低位排序一直到最高位排序完成以后，数列就变成一个有序序列。 

基数排序的方式可以采用LSD（Least significant digital）或MSD（Most significant digital），LSD的排序方式由键值的最右边开始，而MSD则相反，由键值的最左边开始。 

![](/assets/images/posts/sorting-algorithm/radix-sort-1.png)

源码

```
  public void sort(int[] array) {
    int maximumNumber = findMaximumNumberIn(array);

    int numberOfDigits = calculateNumberOfDigitsIn(maximumNumber);

    int placeValue = 1;

    while (numberOfDigits-- > 0) {
      applyCountingSortOn(array, placeValue);
      placeValue *= 10;
    }
  }

  private void applyCountingSortOn(int[] numbers, int placeValue) {
    int range = 10;

    int length = numbers.length;
    int[] frequency = new int[range];
    int[] sortedValues = new int[length];

    for (int i = 0; i < length; i++) {
      int digit = (numbers[i] / placeValue) % range;
      frequency[digit]++;
    }

    for (int i = 1; i < range; i++) {
      frequency[i] += frequency[i - 1];
    }

    for (int i = length - 1; i >= 0; i--) {
      int digit = (numbers[i] / placeValue) % range;
      sortedValues[frequency[digit] - 1] = numbers[i];
      frequency[digit]--;
    }

    System.arraycopy(sortedValues, 0, numbers, 0, length);
  }

  private int calculateNumberOfDigitsIn(int number) {
    return (int) Math.log10(number) + 1;
  }

  private int findMaximumNumberIn(int[] arr) {
    return Arrays.stream(arr).max().getAsInt();
  }
```



# 参考资料

https://www.baeldung.com/java-radix-sort

https://zh.wikipedia.org/wiki/%E5%9F%BA%E6%95%B0%E6%8E%92%E5%BA%8F