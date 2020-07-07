---
layout: post
title: Rest Api的返回值key=null vs key不返回, 哪个设计更好?
description: 没有定论的一个文章
date: 2019-09-24
categories:
    - Restful
comments: true
permalink: rest-api-exclude-null-fields.html
---

在restful api开发中, 对于返回的json数据, 如果属性值为空，有两种处理方式

选项1

```
{
    "name":"bob",
    "age": null
}
```

选项2

```
{
    "name":"bob"
}
```

> 我最初从节省带宽的角度考虑采用了选项2的方案。在多个项目的对接过程中发现对接人员对选项2经常难理解，而且经常不考虑容错，搜索了一下

**如果接口越公开, 可能的使用者越多, 那么接口越应该保持不变**

- 如果有个字段,有时候返回, 有时候却不返回, 这样的接口稳定性是比较差的
- 在说明文档缺失的情况下, 使用者会产生困惑.然后不得不寻找接口的开发人员咨询,提高了接口支撑成本.
- 如果使用者大多数时候能看到字段, 他会以为字段一直会出现, 然后编写代码的时候可能就不会去判断key是否存在. 当key偶尔不存在导致程序报错时又不好发现和排查.
- 数据key一会出现一会不出现, 会引入不确定性: 如果没有出现, 是因为api改变了吗? 还是因为使用者解析出错了? 还是调用出错了?等等
- 对于开发接口的人来说: 他得写代码把key=null的数据剔除掉(可以使用jackson工具), 但是毕竟引入了新的逻辑. 新逻辑意味着可能的新风险.

在大多数应用中, 网络延迟才是主要考虑因素, 而不是带宽. 出于性能的考虑, 很多api开发人员更喜欢少量"大接口",而不是大量"小接口", 一次拿到很多数据,总体上比不停地调api好一些.
对于大多数应用来说, 数据不一致比存在多余数据更危险.

当然也有一些情况我们需要选择方案2

- 有一些应用会频繁调用api, 对数据大小分毫必争, 多一个字段代价都很大，那么毋庸置疑,为null的字段不返回更好
- 有的接口有100个字段, 但是大多数时候90个字段都是null，这种情况下90个字段都无意义的，此时选择精简节约型也更好一些.
- 部分响应是由类似的东西触发的？fields=foo，bar，其中为所有其他字段返回空值似乎有点违反直觉

# 参考资料

https://blog.csdn.net/yhmybzyfuck/article/details/85943731

https://stackoverflow.com/questions/21188145/is-it-worth-to-exclude-null-fields-from-a-json-server-response-in-a-web-applicat

https://stackoverflow.com/questions/15686995/should-null-values-be-included-in-json-responses-from-a-rest-api

https://stackoverflow.com/questions/11003424/should-json-include-null-values
