---
layout: post
title: 分离（1）- 动静分离
date: 2020-05-10
categories:
    - 架构设计
comments: true
permalink: dynamic-static-separation.html
---

# 1. 动静分离

我们的网站简单来说分为 2 种数据资源，一种是**动态的数据**，即 Java 等程序语言实时生成的数据，在网页内容上主要是 HTML  代码，另一种则是**静态资源**，比如图片、css、js、视频等。

一般网站初建，因为流量小、业务简单等原因，都默认将两种数据放到一台服务器上提供服务。访问量大到一定程度之后，就可能出现带宽不足、甚至磁盘高 IO 等问题。这时就可以实施动静分离优化了，将图片等静态资源同步到另一台 WEB  服务器，然后新增绑定一个二级域名，比如 static.domain.com，最后让开发将网页代码中的静态资源替换成这个二级域名即可。

这样一来，图片等静态资源的访问就落到了新增的服务器上，从而分担了大部分访问数据流量和 IO 负载，我们还可以针对性的给静态资源 WEB  做一些优化，比如 JS/CSS/图片压缩、内存缓存、浏览器缓存等等。进一步，我们还可以将静态资源接入 CDN，实现资源就近访问。

![](/assets/images/posts/dynamic-static-separation/dynamic-static-separation-1.png)

动静分离的一种做法是将静态资源部署在nginx上，后台项目部署到应用服务器上，根据一定规则静态资源的请求全部请求nginx服务器，达到动静分离的目标。 

更好的方案是直接将静态资源全部存放在**CDN服务器**上。将项目中的Java,CSS以及img文件都是存放在CDN服务器上，将HTML文件一起存放到CDN上之后，可以将静态资源统一放置在一种服务器上，便于前端进行维护；而且用户在访问静态资源时，可以很好利用CDN的优点——CDN系统能够实时地根据网络流量和各节点的连接、负载状况以及到用户的距离和响应时间等综合信息将用户的请求重新导向离用户最近的服务节点上。

> CDN内容分发：就是将静态资源服务器部署在全国各个服务器节点上去。
>
> 就近原则：用户访问的时候，遵循就近原则，离得越近传输速度越快，离得越远传输的速度相对来说就会慢一些。就像我们访问国外的网站速度比较慢，而  访问国内的网站速度就会快一些。
>
> 参考：https://edgar615.github.io/cdn.html

![](/assets/images/posts/dynamic-static-separation/dynamic-static-separation-2.png)

