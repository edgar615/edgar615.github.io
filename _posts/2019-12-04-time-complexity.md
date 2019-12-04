---
layout: post
title: 时间复杂度
date: 2019-12-04
categories:
    - 算法
comments: true
permalink: time-complexity.html
---

算法的时间复杂度，用来度量算法的运行时间，记作: `T(n) = O(f(n))`。它表示随着 输入大小n 的增大，算法执行需要的时间的增长速度可以用` f(n)` 来描述。

# 恒定时间复杂度 O(1)

示例

```
int n = 1000;
System.out.println("Hey - your input is: " + n);
```

我们知道常数项对函数的增长速度影响并不大，所以当 T(n) = c，c 为一个常数的时候，我们说这个算法的时间复杂度为 `O(1)`；如果 `T(n)`不等于一个常数项时，直接将常数项省略。

```
int n = 1000;
System.out.println("Hey - your input is: " + n);
System.out.println("Hmm.. I'm doing more stuff with: " + n);
System.out.println("And more: " + n);
```
上面例子的时间复杂度也是O(1)，O(2), O(3) 甚至 O(1000) 都是相同的意思，我们不在乎算法到底需要多长时间，只在乎它需要恒定的时间。

# 对数时间复杂度 O(log n)

`O(log n)`的时间复杂度略低于`O(1)`,时间复杂度与输入的对数成比例增长。二分查找就是`O(log n)`的算法，每找一次排除一半的可能，256个数据中查找只要找8次就可以找到目标

示例

```
for (int i = 1; i < n; i = i * 2){
    System.out.println("Hey - I'm busy looking at: " + i);
}
```

如果`n=8`算法会执行3次
```
Hey - I'm busy looking at: 1
Hey - I'm busy looking at: 2
Hey - I'm busy looking at: 4
```

# 线性时间复杂度O(n)

`O(n)`的时间复杂度与输入成比例增长。比如常见的遍历算法

示例

```
for (int i = 0; i < n; i++) {
    System.out.println("Hey - I'm busy looking at: " + i);
}
```

与`O(1)`一样，我们不关心运行时的细节。`O（2n+1）`与`O（n）`相同，下面的算法依然是`O(n)`：它们都与输入的大小成正比

```
for (int i = 0; i < n; i++) {
    System.out.println("Hey - I'm busy looking at: " + i);
    System.out.println("Hmm.. Let's have another look at: " + i);
    System.out.println("And another: " + i);
}
```

# O(n log n)

`O(n log n)`时间复杂度与输入的`n log n`成比例增长：

示例

```
for (int i = 1; i <= n; i++){
    for(int j = 1; j < 8; j = j * 2) {
        System.out.println("Hey - I'm busy looking at: " + i + " and " + j);
    }
}
```

# O(n^p)

`O(n^p)`包含二次（n^2）、三次（n^3）、四次（n^4）等函数。`O（n^2）`比`O（n^3）`快

示例

```
for (int i = 1; i <= n; i++) {
    for(int j = 1; j <= n; j++) {
        System.out.println("Hey - I'm busy looking at: " + i + " and " + j);
    }
}
```

# 指数时间复杂度 O(k^n)

`O(k^n)` 的时间复杂度与输入大小成指数的某个因子成比例增长

例如`O（2^n）`算法每增加一个输入就加倍。如果n=2，算法将运行四次；如果n=3，它们将运行八次。

`O（3^n）`算法每增加一个输入就增加三倍，`O（k^n）`算法每增加一个输入就会增加k倍。

示例

```
for (int i = 1; i <= Math.pow(2, n); i++){
    System.out.println("Hey - I'm busy looking at: " + i);
}
```

# 阶乘时间复杂度 O(n!)

`O(n!)`的时间复杂度与输入的阶乘成正比

示例

```
for (int i = 1; i <= factorial(n); i++){
    System.out.println("Hey - I'm busy looking at: " + i);
}
```

# 参考资料

https://www.baeldung.com/java-algorithm-complexity