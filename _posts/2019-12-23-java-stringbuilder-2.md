---
layout: post
title: String，StringBuilder,StringBuffer比较
date: 2019-12-23
categories:
    - java
comments: true
permalink: java-stringbuilder-2.html
---

先看单线程

```
    String withString = "";
    long t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      withString += "some string";
    }
    System.out.println("strings:" + (System.currentTimeMillis() - t0));

    t0 = System.currentTimeMillis();
    StringBuffer buf = new StringBuffer();
    for (int i = 0; i < 100000; i++) {
      buf.append("some string");
    }
    System.out.println("Buffers : " + (System.currentTimeMillis() - t0));

    t0 = System.currentTimeMillis();
    StringBuilder building = new StringBuilder();
    for (int i = 0; i < 100000; i++) {
      building.append("some string");
    }
    System.out.println("Builder : " + (System.currentTimeMillis() - t0));
```

输出

```
strings:51645
Buffers : 2
Builder : 2
```

可以看到string的很慢，Buffer和Builder差不多。

修改堆内存`-Xmx10240m -Xmn10240m -XX:+PrintGCDetails`，重新测试，buffer比builder慢了一点，string的连接操作伴随了大量的FullGC

> 因为string是不可变对象，拼接操作会伴随着大量char数组的扩容复制操作

```
strings:50391
Buffers : 3
Builder : 1
```

对测试方法做一点改动

```
    long t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      String withString = "some " + "string";
    }
    System.out.println("strings:" + (System.currentTimeMillis() - t0));

    t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      StringBuffer buf = new StringBuffer();
      buf.append("some ").append("string");
    }
    System.out.println("Buffers : " + (System.currentTimeMillis() - t0));

    t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      StringBuilder building = new StringBuilder();
      building.append("some ").append("string");

    }
    System.out.println("Builder : " + (System.currentTimeMillis() - t0));
```

输出

```
strings:3
Buffers : 9
Builder : 8
```

这时发现string的拼接最快，因为String 对象的直接相加，JVM 会自动对其进行优化，也就是说 `"some " + "string"` 在编译期间会自动优化为 "some string"，直接一次性创建完成，所以效率肯定要高于`StringBuffer`和`StringBuilder`的 append 拼接。

但是要注意对于下面这种间接相加的操作，效率要比直接相加低，因为在编译器不会对引用变量进行优化。

```
String str1 = "Hello";
String str2 = "World";
String str3 = str1+str2;
```

测试一下

```
    long t0 = System.currentTimeMillis();
    for (int i = 0; i < 100000; i++) {
      String some = "some ";
      String string = "string";
      String withString = some + string;
    }
    System.out.println("strings:" + (System.currentTimeMillis() - t0));
```

输出

```
strings:17
```

在多线程下比较`StringBuffer`和`StringBuilder`

```
public class StringsPerf {

    public static void main(String[] args) {

        ThreadPoolExecutor executorService = (ThreadPoolExecutor) Executors.newFixedThreadPool(10);
        //With Buffer
        StringBuffer buffer = new StringBuffer();
        for (int i = 0 ; i < 10; i++){
            executorService.execute(new AppendableRunnable(buffer));
        }
        shutdownAndAwaitTermination(executorService);
        System.out.println(" Thread Buffer : "+ AppendableRunnable.time);

        //With Builder
        AppendableRunnable.time = 0;
        executorService = (ThreadPoolExecutor) Executors.newFixedThreadPool(10);
        StringBuilder builder = new StringBuilder();
        for (int i = 0 ; i < 10; i++){
            executorService.execute(new AppendableRunnable(builder));
        }
        shutdownAndAwaitTermination(executorService);
        System.out.println(" Thread Builder: "+ AppendableRunnable.time);

    }

   static void shutdownAndAwaitTermination(ExecutorService pool) {
        pool.shutdown(); // code reduced from Official Javadoc for Executors
        try {
            if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
                pool.shutdownNow();
              if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
                System.err.println("Pool did not terminate");
              }
            }
        } catch (Exception e) {}
    }
}

class AppendableRunnable<T extends Appendable> implements Runnable {

    static long time = 0;
    T appendable;
    public AppendableRunnable(T appendable){
        this.appendable = appendable;
    }

    @Override
    public void run(){
        long t0 = System.currentTimeMillis();
        for (int j = 0 ; j < 10000 ; j++){
            try {
                appendable.append("some string");
            } catch (IOException e) {}
        }
        time+=(System.currentTimeMillis() - t0);
    }
}
```

可以看到`StringBuffer`明显要比`StringBuilder`慢

# 参考资料

https://stackoverflow.com/questions/355089/difference-between-stringbuilder-and-stringbuffer

https://mp.weixin.qq.com/s/dltB1Fzwdezs4PDN8Thx6w
