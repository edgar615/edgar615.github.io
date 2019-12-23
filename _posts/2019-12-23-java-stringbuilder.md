---
layout: post
title: java StringBuidler为什么线程不安全
date: 2019-12-23
categories:
    - java
comments: true
permalink: java-stringbuild.html
---

我们都知道StringBuilder不是线程安全的，StringBuffer是线程安全的，现在看一下原因

`StringBuilder`继承自`AbstractStringBuilder`，有两个变量

```
/**
 * 字符串的存储数组
 */
char[] value;

/**
 * 字符串的长度
 */
int count;
```

`StringBuilder`构造的时候默认容量是16（也可以自定义）

```
public StringBuilder() {
	super(16);
}
public StringBuilder(int capacity) {
	super(capacity);
}
```

如果是使用字符串构造，会将容量在字符串的基础上增加16


```
public StringBuilder(String str) {
	super(str.length() + 16);
	append(str);
}
```

我们看一下`append`方法

```
public AbstractStringBuilder append(String str) {
	if (str == null)
		return appendNull();
	int len = str.length();
	// 扩容
	ensureCapacityInternal(count + len);
	// 将str填充到char数组中
	str.getChars(0, len, value, count);
	// 修改count
	count += len;
	return this;
}
```

首先我们可以直观的看到修改count的代码`count += len;`不是原子操作，对于`count=1`的`StringBuilder`，如果线程A和线程B并发添加一个字符串的时候， 在`count += len;`发生CPU切换，会导致count最终不是3，而是2，造成数据丢失

测试一番

```
StringBuilder stringBuilder = new StringBuilder();
for (int i = 0; i < 10; i++){
	new Thread(new Runnable() {
		@Override
		public void run() {
			for (int j = 0; j < 1000; j++){
				stringBuilder.append(j);
			}
		}
	}).start();
}

Thread.sleep(100);
System.out.println(stringBuilder.length() == 10 * 1000);//false
```
**扩容**

我们在回过头来看扩容的代码

> JDK里很多线程不安全的类都是因为扩容引起的

`ensureCapacityInternal`方法调用`newCapacity`来计算新数组长度

```
private void ensureCapacityInternal(int minimumCapacity) {
	// overflow-conscious code
	if (minimumCapacity - value.length > 0) {
		value = Arrays.copyOf(value,
				newCapacity(minimumCapacity));
	}
}

private int newCapacity(int minCapacity) {
	// overflow-conscious code
	int newCapacity = (value.length << 1) + 2;
	if (newCapacity - minCapacity < 0) {
		newCapacity = minCapacity;
	}
	return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
		? hugeCapacity(minCapacity)
		: newCapacity;
}
```

通过代码我们可以看到，每次扩容的长度=`原长度*2 +2`，最大不超过`Integer.MAX_VALUE - 8`，超过最大值会抛出异常`OutOfMemoryError: Java heap space`

> 对于为什么要+2，在网上没有找到原因

测试代码

```
StringBuilder stringBuilder = new StringBuilder();
System.out.println(stringBuilder.length() + ":" + stringBuilder.capacity());
System.out.println(stringBuilder.capacity());
for (int i = 0; i < 17; i ++) {
	stringBuilder.append("0");
	System.out.println(stringBuilder.length() + ":" + stringBuilder.capacity());
}
```

输出

```
0:16
1:16
2:16
3:16
4:16
5:16
6:16
7:16
8:16
9:16
10:16
11:16
12:16
13:16
14:16
15:16
16:16
17:34
```

测试最大值

```
StringBuilder stringBuilder = new StringBuilder();
for (int i = 0; i < Integer.MAX_VALUE; i ++) {
	stringBuilder.append(i);
}
```

输出

```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3332)
	at java.lang.AbstractStringBuilder.ensureCapacityInternal(AbstractStringBuilder.java:124)
	at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:674)
	at java.lang.StringBuilder.append(StringBuilder.java:208)
```

**追加字符串**
`ensureCapacityInternal`完成扩容后通过`str.getChars(0, len, value, count)`来将字符串追加到新数组中，最终是通过`System.arraycopy`将`String`内部的`char[]`数组复制到`StringBuilder`内部的`char[]`数组。

```
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
	if (srcBegin < 0) {
		throw new StringIndexOutOfBoundsException(srcBegin);
	}
	if (srcEnd > value.length) {
		throw new StringIndexOutOfBoundsException(srcEnd);
	}
	if (srcBegin > srcEnd) {
		throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
	}
	System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}
```

对于`count=16`的`StringBuilder`，线程A和线程B并发添加一个字符串的时候，按下面的逻辑执行

1. 线程A执行扩容`ensureCapacityInternal(count + len)`，数组长度17，count=16
2. 线程B执行扩容`ensureCapacityInternal(count + len)`，数组长度17，count=16
3. 线程B执行完剩余逻辑，数组长度17，count=17
4. 线程A执行copy方法`System.arraycopy(str, 0, value, 17, 1)`，这时会抛出异常`ArrayIndexOutOfBoundsException`

用下面的方法模拟了一下

```
char[] array = new char[]{'0'};
char[] dst = new char[17];
System.arraycopy(array, 0, dst, 17, 1);
```

> **注意：这里的ensureCapacityInternal扩容并不是直接将容量加一，这里只是为了说明问题**

为了验证猜想在参考`StringBuilder`封装了一个Append对象用来测试多线程下的`ArrayIndexOutOfBoundsException`

```
public class Append {

  private char[] value;

  private int count = 16;

  public Append() {
    this.value = new char[count];
  }

  public void append(String str) {
    value = Arrays.copyOf(value, value.length + str.length());
    str.getChars(0, str.length(), value, count);
    count += str.length();
  }

  public static void main(String[] args) throws InterruptedException {
    Append append = new Append();
    for (int i = 0; i < 10; i++){
      new Thread(new Runnable() {
        @Override
        public void run() {
          for (int j = 0; j < 1000; j++){
            append.append("0000000000000000");
          }
        }
      }).start();
    }
    TimeUnit.SECONDS.sleep(5);
  }
}
```

运行后发现抛出了ArrayIndexOutOfBoundsException

总结

- StringBuilder在多线程环境下会出现数据丢失和ArrayIndexOutOfBoundsException的问题
- 默认容量为16（根据字符串构建为字符串长度+16），新容量扩为 大小：变成2倍+2
- 数组的最大长度是`Integer.MAX_VALUE - 8`
- 前预估好字符串的长度，进一步减少扩容带来的额外开销。

