---
layout: post
title: 堆溢出后其他线程还能不能运行
date: 2020-04-09
categories:
    - 多线程
comments: true
permalink: thread-oom.html
---

个美团面试题：“一个线程OOM后，其他线程还能运行吗？”

> OOM又分很多类型；比如：
>
> 堆溢出（“java.lang.OutOfMemoryError: Java heap space”）
>
> 永久带溢出（“java.lang.OutOfMemoryError:Permgen space”）
>
> 不能创建线程（“java.lang.OutOfMemoryError:Unable to create new native thread”）等很多种情况。

测试一把

```java
public class JvmThread {

    public static void main(String[] args) {
        new Thread(() -> {
            List<byte[]> list = new ArrayList<byte[]>();
            while (true) {
                System.out.println(new Date().toString() + Thread.currentThread() + "==");
                byte[] b = new byte[1024 * 1024 * 1];
                list.add(b);
                try {
                    Thread.sleep(1000);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        // 线程二
        new Thread(() -> {
            while (true) {
                System.out.println(new Date().toString() + Thread.currentThread() + "==");
                try {
                    Thread.sleep(1000);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```

启动参数设置为`-Xms16m -Xmx32m`,输出结果

```
Wed Apr 08 15:03:13 CST 2020Thread[Thread-1,5,main]==
Wed Apr 08 15:03:13 CST 2020Thread[Thread-0,5,main]==
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at com.github.edgar615.sentinel.JvmThread.lambda$main$0(JvmThread.java:14)
	at com.github.edgar615.sentinel.JvmThread$$Lambda$1/2046562095.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)
Wed Apr 08 15:03:14 CST 2020Thread[Thread-1,5,main]==
```

**说明其他线程还在运行**

观察堆和线程

![](/assets/images/posts/thread-oom/thread-oom-2.png)

![](/assets/images/posts/thread-oom/thread-oom-2.png)

当一个线程抛出OOM异常后，它所占据的内存资源会全部被释放掉，从而不会影响其他线程的运行！

总结：其实发生OOM的线程一般情况下会死亡，也就是会被终结掉，该线程持有的对象占用的heap都会被gc了，释放内存。因为发生OOM之前要进行gc，就算其他线程能够正常工作，也会因为频繁gc产生较大的影响。

> 如果是栈溢出，结论也是一样的

# 参考资料

https://mp.weixin.qq.com/s/HkKcZxLtvL2T3hMylPwN0Q