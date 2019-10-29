---
layout: post
title: new一个对象有哪两个过程？
date: 2019-10-29
categories:
    - java
comments: true
permalink: java-new-object.html
---

Java在new一个对象的时候，会先查看对象所属的类有没有被加载到内存，如果没有的话，就会先通过类的全限定名来加载。加载并初始化类完成后，再进行对象的创建工作。如果是第一次使用该类，new一个对象就可以分为两个过程：加载并初始化类和创建对象。

# 类加载过程

在[classloader](https://https://edgar615.github.io/classloader.html)的文章中已经描述了类加载过程：加载>验证>准备>解析>初始化（先父后子）

# 创建对象

1. 在堆区分配对象需要的内存：分配的内存包括本类和父类的所有实例变量，但不包括任何静态变量
2. 对所有实例变量赋默认值：将方法区内对实例变量的定义拷贝一份到堆区，然后赋默认值
3. 执行实例初始化代码：初始化顺序是先初始化父类再初始化子类，初始化时先执行实例代码块然后是构造方法
4. 如果有类似于`Child c = new Child()`形式的c引用的话，在栈区定义Child类型引用变量c，然后将堆区对象的地址赋值给它

需要注意的是，每个子类对象持有父类对象的引用，可在内部通过super关键字来调用父类对象，但在外部不可访问

通过实例引用调用实例方法的时候，先从方法区中对象的实际类型信息找，找不到的话再去父类类型信息中找。

如果继承的层次比较深，要调用的方法位于比较上层的父类，则调用的效率是比较低的，因为每次调用都要经过很多次查找。这时候大多系统会采用一种称为虚方法表的方法来优化调用的效率。

所谓虚方法表，就是在类加载的时候，为每个类创建一个表，这个表包括该类的对象所有动态绑定的方法及其地址，包括父类的方法，但一个方法只有一条记录，子类重写了父类方法后只会保留子类的。当通过对象动态绑定方法的时候，只需要查找这个表就可以了，而不需要挨个查找每个父类。

# 参考资料

https://mp.weixin.qq.com/s/kRVQGz2ZWnRShiFVePwiRw