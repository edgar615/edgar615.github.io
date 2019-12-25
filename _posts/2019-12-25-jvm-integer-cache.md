---
layout: post
title: JVM整数类型的缓存
date: 2019-12-25
categories:
    - jvm
comments: true
permalink: jvm-integer-cache.html
---

先看下面的代码

```
Integer integer1 = 127;
Integer integer2 = 127;
System.out.println(integer1 == integer2);

Integer integer3 = 128;
Integer integer4 = 128;
System.out.println(integer3 == integer4);

Integer integer5 = -128;
Integer integer6 = -128;
System.out.println(integer5 == integer6);

Integer integer7 = -129;
Integer integer8 = -129;
System.out.println(integer7 == integer8);
```

输出：

```
true
false
true
false
```

# 原理

在 JAVA 语言中有8中基本类型和一种比较特殊的类型String。这些类型为了使他们在运行过程中速度更快，更节省内存，都提供了一种常量池的概念。常量池就类似一个JAVA系统级别提供的缓存。

8种基本类型的常量池都是系统协调的，Integer 默认缓存 -128 ~ 127 区间的值，Long 和 Short 也是缓存了这个区间的值，Byte 只能表示 -127 ~ 128 范围的值，全部缓存了，Character 缓存了 0 ~ 127 的值。Float 和 Double 没有缓存的意义。

> Integer 可通过设置可以通过`-XX:AutoBoxCacheMax=<size>`参数去修改这个值，在JVM初始化的时候，会将`java.lang.Integer.IntegerCache.high`写入sun.misc.VM class系统私有配置文件中，并加载。其他基本类型无法修改。

Java 编译器把原始类型自动转换为封装类的过程称为自动装箱（autoboxing），这相当于调用 valueOf 方法

对于`Integer i = 1;`反编译后

```
// int型常量1进栈(const、push：将相应类型的常量放入栈顶)
0: iconst_1
// 调用类方法(static)
1: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
// 栈顶ref对象存入第1局部变量
4: astore_1
```

```
public static Integer valueOf(int i) {
	if (i >= IntegerCache.low && i <= IntegerCache.high)
		return IntegerCache.cache[i + (-IntegerCache.low)];
	return new Integer(i);
}
```
可以看到，如果`i`介于`IntegerCache`的[low, high]之间，就会直接返回缓存中的数据

从IntegerCache的实现可以看出，IntegerCache在初始化的时候就从系统私有变量中读取了` java.lang.Integer.IntegerCache.high`用来设置缓存的最大值，然后迭代初始化缓存。

**因此在调优的时候可以根据实际情况把AutoBoxCacheMax的值设置的大一点。**

```
private static class IntegerCache {
	static final int low = -128;
	static final int high;
	static final Integer cache[];

	static {
		// high value may be configured by property
		int h = 127;
		String integerCacheHighPropValue =
			sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
		if (integerCacheHighPropValue != null) {
			try {
				int i = parseInt(integerCacheHighPropValue);
				i = Math.max(i, 127);
				// Maximum array size is Integer.MAX_VALUE
				h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
			} catch( NumberFormatException nfe) {
				// If the property cannot be parsed into an int, ignore it.
			}
		}
		high = h;

		cache = new Integer[(high - low) + 1];
		int j = low;
		for(int k = 0; k < cache.length; k++)
			cache[k] = new Integer(j++);

		// range [-128, 127] must be interned (JLS7 5.1.7)
		assert IntegerCache.high >= 127;
	}

	private IntegerCache() {}
}
```

LongCache

```
private static class LongCache {
	private LongCache(){}

	static final Long cache[] = new Long[-(-128) + 127 + 1];

	static {
		for(int i = 0; i < cache.length; i++)
			cache[i] = new Long(i - 128);
	}
}
```

ShortCache

```
private static class ShortCache {
	private ShortCache(){}

	static final Short cache[] = new Short[-(-128) + 127 + 1];

	static {
		for(int i = 0; i < cache.length; i++)
			cache[i] = new Short((short)(i - 128));
	}
}
```

ByteCache

```
private static class ByteCache {
	private ByteCache(){}

	static final Byte cache[] = new Byte[-(-128) + 127 + 1];

	static {
		for(int i = 0; i < cache.length; i++)
			cache[i] = new Byte((byte)(i - 128));
	}
}
```

CharacterCache

```
private static class CharacterCache {
	private CharacterCache(){}

	static final Character cache[] = new Character[127 + 1];

	static {
		for (int i = 0; i < cache.length; i++)
			cache[i] = new Character((char)i);
	}
}
```




# 参考资料

https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html
