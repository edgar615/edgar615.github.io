---
layout: post
title: equals和hashcode
date: 2019-06-24
categories:
    - java
comments: true
permalink: equals-hashcode.html
---

Object对象中更有两个方法`equals`和`hashCode`，平时对这两个方法虽然有些印象，真正说起来还是有点模糊。整理 一下

在java中任何一个对象都具有equals和hashCode两种方法，因为它们都是在Object类中定义的，equals用来判断两个对象是否相同，如果相同则返回true，如果不相同则返回false。hashCode则是返回一个int数值，在Object类中的默认实现是“将该对象的内部地址转换成一个整数返回”。

# equals

- 自反性 ：`x.equals(x)` 结果应该返回`true`
- 对称性 ： `x.equals(y)` 结果返回`true`当且仅当`y.equals(x)`也应该返回`true`
- 传递性 ： `x.equals(y)` 返回`true`，并且`y.equals(z)` 返回`true`，那么`x.equals(z)` 也应该返回`true`
- 一致性 ：`x.equals(y)`的第一次调用为`true`，那么`x.equals(y)`的第二次，第三次等多次调用也应该为`true`，但是前提条件是在进行比较之前，x和y都没有被修改。 
- 对于任何非空引用值 x，`x.equals(null)`都应返回 false。 
- 这个方法返回true当且仅当x和y指向了同样的对象`(x==y)`，这句话也就是说明了在默认情况下，Object类中的equals方法默认比较的是对象的地址，因为只有是相同的地址才会相等`(x == y)`，如果没有重写equals方法，那么默认就是比较的是地址

**注意：无论何时这个equals方法被重写那么都是有必要去重写hashCode方法，这是因为为了维持hashCode的一种约定，相同的对象必须要有相同的hashCode值**

# hashcode

- 在 Java 应用程序执行期间，在对同一对象多次调用 hashCode 方法时，必须一致地返回相同的整数，前提是将对象进行 equals 比较时所用的信息没有被修改。从某一应用程序的一次执行到同一应用程序的另一次执行，该整数无需保持一致
-  如果根据 equals(Object) 方法，两个对象是相等的，那么对这两个对象中的每个对象调用 hashCode 方法都必须生成相同的整数结果
-  如果根据 equals(java.lang.Object) 方法，两个对象不相等，那么对这两个对象中的任一对象上调用 hashCode 方法不要求一定生成不同的整数结果。但是，程序员应该意识到，为不相等的对象生成不同整数结果可以提高哈希表的性能
-  实际上，由 Object 类定义的 hashCode 方法确实会针对不同的对象返回不同的整数。（这一般是通过将该对象的内部地址转换成一个整数来实现的，但是 JavaTM 编程语言不需要这种实现技巧。） 
   ​     

# equals和hashcode的关系

- 若重写了`equals(Object obj)`方法，则有必要重写hashCode()方法
- 若两个对象``equals(Object obj)`返回true，则hashCode（）有必要也返回相同的int数
- 若两个对象`equals(Object obj)`返回false，则hashCode（）不一定返回不同的int数
- 若两个对象`hashCode（）`返回相同int数，则equals（Object obj）不一定返回true
- 若两个对象`hashCode（）`返回不同int数，则equals（Object obj）一定返回false
- 同一对象在执行期间若已经存储在集合中，则不能修改影响hashCode值的相关信息，否则会导致内存泄露问题。

# 为什么有了equals还要有hashcode方法?

Java中的Collection有两类，一类是List，一类是Set。List内的元素是有序的，元素可以重复。Set元素无序，但元素不可重复。要想保证元素不重复，两个元素是否重复应该依据什么来判断呢？用Object.equals方法。但若每增加一个元素就检查一次，那么当元素很多时，后添加到集合中的元素比较的次数就非常多了。也就是说若集合中已有1000个元素，那么第1001个元素加入集合时，它就要调用1000次equals方法。这显然会大大降低效率(equals方法一般比较复杂)。于是Java采用了哈希表的原理。当Set接收一个元素时根据该对象的内存地址算出hashCode，看它属于哪一个区间，再这个区间里调用equeals方法。**这里需要注意的是：当俩个对象的hashCode值相同的时候，Hashset会将对象保存在同一个位置，但是他们equals返回false，所以实际上这个位置采用链式结构来保存多个对象。** 

> hashmap的实现差不多

上面方法确实提高了效率：

- 通过hashCode可以很快的查到小内存块。
- 通过hashCode比较比equals方法快，当get时先比较hashCode，如果hashCode不同，直接返回false。

但一个面临问题：若两个对象equals相等，但不在一个区间，因为hashCode的值在重写之前是对内存地址计算得出，所以根本没有机会进行比较，会被认为是不同的对象。所以Java对于eqauls方法和hashCode方法是这样规定的：

- 如果两个对象相同，那么它们的hashCode值一定要相同。也告诉我们重写equals方法，一定要重写hashCode方法，也就是说hashCode值要和类中的成员变量挂上钩，对象相同，那么成员变量相同，那么hashCode值也一定相同
- 如果两个对象的hashCode相同，它们并不一定相同，这里的对象相同指的是用eqauls方法比较。


# 测试

对于下面的对象
```
public class Person {

  private int id;

  private String firstName;

  private String lastName;
...
}
```

## 重写equals不重写hashcode

```

  @Override
  public boolean equals(Object o) {
    if (this == o) {
      return true;
    }
    if (o == null || getClass() != o.getClass()) {
      return false;
    }
    Person person = (Person) o;
    return id == person.id &&
        Objects.equals(firstName, person.firstName) &&
        Objects.equals(lastName, person.lastName);
  }
```
测试
```
    Set<Person> set = new HashSet<>();
    Person person = new Person();
    person.setId(1);
    person.setFirstName("edgar");
    person.setLastName("zhang");
    set.add(person);

    person = new Person();
    person.setId(1);
    person.setFirstName("edgar");
    person.setLastName("zhang");
    set.add(person);

    System.out.println(set.size());//返回2
```

## 重写hashcode不重写equals

```
  @Override
  public int hashCode() {
    return Objects.hash(id, firstName, lastName);
  }
```
测试
```
    Set<Person> set = new HashSet<>();
    Person person = new Person();
    person.setId(1);
    person.setFirstName("edgar");
    person.setLastName("zhang");
    set.add(person);

    person = new Person();
    person.setId(1);
    person.setFirstName("edgar");
    person.setLastName("zhang");
    set.add(person);

    System.out.println(set.size());//返回2
```

## 重写hashcode和equals

```
  @Override
  public boolean equals(Object o) {
    if (this == o) {
      return true;
    }
    if (o == null || getClass() != o.getClass()) {
      return false;
    }
    Person person = (Person) o;
    return id == person.id &&
        Objects.equals(firstName, person.firstName) &&
        Objects.equals(lastName, person.lastName);
  }

  @Override
  public int hashCode() {
    return Objects.hash(id, firstName, lastName);
  }
```
测试
```
    Set<Person> set = new HashSet<>();
    Person person = new Person();
    person.setId(1);
    person.setFirstName("edgar");
    person.setLastName("zhang");
    set.add(person);

    person = new Person();
    person.setId(1);
    person.setFirstName("edgar");
    person.setLastName("zhang");
    set.add(person);

    System.out.println(set.size());//返回1
```

**要想保证元素的唯一性，必须同时重写hashCode和equals才行**

# 内存泄露

```
    Set<Person> set = new HashSet<>();
    Person person = new Person();
    person.setId(1);
    person.setFirstName("edgar");
    person.setLastName("zhang");
    set.add(person);

    person = new Person();
    person.setId(1);
    person.setFirstName("leo");
    person.setLastName("zhang");
    set.add(person);

    System.out.println(set.size());// 返回2


    person.setFirstName("bob");
    set.remove(person);
    System.out.println(set.size());// 返回1
```

从上面的代码我们看出，如果我们将对象的属性值参与了hashCode的运算中，在进行删除的时候，就不能对其属性值进行修改，否则导致该对象长时间不能被释放，造成内存泄露

# String

```
    String s1 = "hello";
    String s2 = "hello";
    String s3 = new String("hello");
    String s4 = new String("hello");
    System.out.println(s1 == s2);//true
    System.out.println(s1 == s3);//false
    System.out.println(s3 == s4);//false

    System.out.println(s1.equals(s2));//true
    System.out.println(s1.equals(s3));//true
    System.out.println(s3.equals(s4));//true

    System.out.println(s1.hashCode() == s2.hashCode());//true
    System.out.println(s1.hashCode() == s3.hashCode());//true
    System.out.println(s3.hashCode() == s4.hashCode());//true
```
String类重写了equals方法和hashCode方法

```
    public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
    
    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
```

# 参考资料

https://blog.csdn.net/lijiecao0226/article/details/24609559

https://blog.csdn.net/qq_21688757/article/details/53067814