---
layout: post
title: Linux管道
date: 2019-09-02
categories:
    - linux
comments: true
permalink: linux-pipeline.html
---

# 1. 管道

管道（Pipeline）的作用是在命令和命令之间，传递数据。比如说一个命令的结果，就可以作为另一个命令的输入。更准确地说，管道在进程间传递数据。

**输入输出流**

每个进程拥有自己的标准输入流、标准输出流、标准错误流。

这几个标准流说起来很复杂，但其实都是文件。

- 标准输入流（用 0 表示）可以作为进程执行的上下文（进程执行可以从输入流中获取数据）。
- 标准输出流（用 1 表示）中写入的结果会被打印到屏幕上。
- 如果进程在执行过程中发生异常，那么异常信息会被记录到标准错误流（用 2 表示）中。

**重定向**

我们执行一个指令，比如`ls -l`，结果会写入标准输出流，进而被打印。这时可以用重定向符将结果重定向到一个文件，比如说`ls -l > out`，这样out文件就会有`ls -l`的结果；而屏幕上也不会再打印`ls -l`的结果。

具体来说`>`符号叫作覆盖重定向；`>>`叫作追加重定向。`>`每次都会把目标文件覆盖，`>>`会在目标文件中追加。

```
[root@VM-0-17-centos ~]# ls1 > out
-bash: ls1: command not found
[root@VM-0-17-centos ~]# cat out
[root@VM-0-17-centos ~]#
```

上述结果并不会存入out文件，因为ls1指令是不存在的。结果会输出到标准错误流中，仍然在屏幕上。这里我们可以把标准错误流也重定向到标准输出流，然后再重定向到文件。

```

[root@VM-0-17-centos ~]# ls1 &> out
[root@VM-0-17-centos ~]# cat out
-bash: ls1: command not found
```

上述的命令等价于

```
ls1 > out 2>&1
```

相当于把ls1的标准输出流重定向到out，因为`ls1 > out`出错了，所以标准错误流被定向到了标准输出流。`&`代表一种引用关系，具体代表的是`ls1 >out`的标准输出流

**管道的作用和分类**

管道（Pipeline）将一个进程的输出流定向到另一个进程的输入流，就像水管一样，作用就是把这两个文件接起来。如果一个进程输出了一个字符 X，那么另一个进程就会获得 X 这个输入。

管道和重定向很像，但是管道是一个连接一个进行计算，重定向是将一个文件的内容定向到另一个文件，这二者经常会结合使用。

Linux 中的管道也是文件，有两种类型的管道：

- 匿名管道（Unnamed Pipeline），这种管道也在文件系统中，但是它只是一个存储节点，不属于任何一个目录。说白了，就是没有路径。
- 命名管道（Named Pipeline），这种管道就是一个文件，有自己的路径。

**FIFO**

管道具有 FIFO（First In First Out），FIFO 和排队场景一样，先排到的先获得。所以先流入管道文件的数据，也会先流出去传递给管道下游的进程。

# 2. 示例

## 2.1. 排序

比如我们用ls，希望按照文件名排序倒序，可以使用匿名管道，将ls的结果传递给sort指令去排序。

```
[root@VM-0-17-centos ~]# ls . | sort -r
out
mysql-install.sh
mysql-install.log
```

## 2.2. 去重

```
[root@VM-0-17-centos ~]# cat a.txt
Apple
Banana
Apple
Banana
[root@VM-0-17-centos ~]# sort a.txt | uniq
Apple
Banana
```

**筛选**

有时候我们想根据正则模式筛选对应的内容。比如说我们想找到项目文件下所有文件名中含有Spring的文件。就可以利用grep指令

```
[root@VM-0-17-centos ~]# find . | grep mysql
./.mysql_history
./mysql-install.sh
./mysql-install.lo
```

**数数行**

查看一个 文件有多少行，就可以使用wc -l指令

```
[root@VM-0-17-centos ~]# wc -l a.txt
4 a.txt
```

查看一个目录，有多少个文件

```
[root@VM-0-17-centos ~]# ls | wc -l
4
```

# 3. 中间结果

管道一个接着一个，是一个计算逻辑。有时候我们想要把中间的结果保存下来，这就需要用到tee指令。tee指令从标准输入流中读取数据到标准输出流。

tee还有一个能力，就是自己利用这个过程把输入流中读取到的数据存到文件中

```
find ./ -i "*.java" | tee JavaList | grep Spring
```

这句指令的意思是从当前目录中找到所有含有 Spring 关键字的 Java 文件。tee 本身不影响指令的执行，但是 tee 会把 find 指令的结果保存到 JavaList 文件中。

tee这个执行就像英文字母中的 T 一样，连通管道两端，下面又开了口。这个开口，在函数式编程里面叫作副作用。

# 4. xargs

`xargs`指令从标准数据流中构造并执行一行行的指令。xargs从输入流获取字符串，然后利用空白、换行符等切割字符串，在这些字符串的基础上构造指令，最后一行行执行这些指令。

```
[root@VM-0-17-centos ~]# ls *.a | xargs -I GG echo "mv GG test_GG"
mv x.a test_x.a
mv y.a test_y.a
mv z.a test_z.a
```

我们用ls找到所有的文件；

-I参数是查找替换符，这里我们用GG替代ls找到的结果；-I GG后面的字符串 GG 会被替换为x.a``x.b或x.z；

echo是一个在命令行打印字符串的指令。使用echo主要是为了安全，帮助我们检查指令是否有错误。

最后去掉 echo，我们就可以将所有`.a`结尾的文件加上前缀`test_`：

```
[root@VM-0-17-centos ~]# ls *.a | xargs -I GG  mv GG test_GG
[root@VM-0-17-centos ~]# ls *.a
test_x.a  test_y.a  test_z.a
[root@VM-0-17-centos ~]#

```

# 5. 管道文件

上面我们花了较长的一段时间讨论匿名管道，用`|`就可以创造和使用。匿名管道也是利用了文件系统的能力，是一种文件结构。匿名管道拥有一个自己的inode，但不属于任何一个文件夹。

还有一种管道叫作命名管道（Named Pipeline）。命名管道是要挂到文件夹中的，因此需要创建。用`mkfifo`指令可以创建一个命名管道，

```
[root@VM-0-17-centos ~]# mkfifo pipe1
[root@VM-0-17-centos ~]# ls -F pipe1
pipe1|
```

命名管道和匿名管道能力类似，可以连接一个输出流到另一个输入流，也是 First In First Out。

当执行cat pipe1的时候，你可以观察到，当前的终端处于等待状态。因为我们cat pipe1的时候pipe1中没有内容。如果这个时候我们再找一个终端去写一点东西到pipe中

```
echo xxxx > pipe1
```

这个时候，cat pipe1就会返回，并打印出xxxx

```
[root@VM-0-17-centos ~]# cat pipe1
xxxx
```



# 6. 参考资料

《重学操作系统 》