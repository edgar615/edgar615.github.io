---
layout: post
title: 灰度发布（金丝雀发布）
date: 2021-01-16
categories:
    - 架构设计
comments: true
permalink: canary-deploy.html
---

# 1. 灰度发布

互联网产品需要快速迭代开发上线，又要保证质量，保证刚上线的系统，一旦出现问题可以很快控制影响面，就需要设计一套灰度发布系统。

灰度发布系统的作用，可以根据配置，将用户的流量导到新上线的系统上，来快速验证新的功能，而一旦出现问题，也可以马上的修复，简单的说，就是一套A/B Test系统。

灰度发布允许带着bug上线，只要bug不是致命的，当然这个bug是不知道的情况下，如果知道就要很快的改掉

AB Test 就是一种灰度发布方式，让一部分用户继续用A，一部分用户开始用B，如果用户对B没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到B上面来。灰度发布可以保证整体系统的稳定，在初始灰度的时候就可以发现、调整问题，以保证其影响度。

**`灰度发布结构图`**

![](/assets/images/posts/canary-deploy/canary-deploy-1.png)

**A/B Testing**

A/B测试是用来测试应用功能表现的方法，例如可用性、受欢迎程度、可见性等等。 A/B测试通常用在应用的前端上，不过当然需要后端来支持。

![](/assets/images/posts/canary-deploy/canary-deploy-2.png)

A/B 测试与蓝绿发布的区别在于， A/B 测试目的在于通过科学的实验设计、采样样本代表性、流量分割与小流量测试等方式来获得具有代表性的实验结论，并确信该结论在推广到全部流量可信；蓝绿发布的目的是安全稳定地发布新版本应用，并在必要时回滚。

蓝绿发布和金丝雀是发布策略，目标是确保新上线的系统稳定，关注的是新系统的BUG、隐患。
 A/B测试是效果测试，同一时间有多个版本的服务对外服务，这些服务都是经过足够测试，达到了上线标准的服务，有差异但是没有新旧之分（它们上线时可能采用了蓝绿发布的方式）。

# 2. 设计实现

![](/assets/images/posts/canary-deploy/canary-deploy-3.png)

其中分为几个部分：

1. 接入层，接入客户端请求，根据下发的配置将符合条件的请求转发到新旧系统上．
2. 配置管理后台，这个后台可以配置不同的转发策略给接入层．
3. 新旧两种处理客户端请求的业务服务器．

关于接入策略的设计上，从协议层来说，需要从一开始就设计是根据哪些参数来进行转发的，而且这些参数最好跟具体的协议体内容分开，这样减少接入层对协议的解析．举个例子，如果客户端的请求是走HTTP协议的，那么将这些参数放在HEADER部分就好了，接入层不需要去具体解析body部分的数据就拿到了转发策略需要的参数．当然，放在HEADER中的数据，因为没有了加密性，又是需要考虑的另一个问题．

# 3. 灰度的策略

灰度必须要有灰度策略，灰度策略常见的方式有以下几种

- 基于Request Header进行流量切分
- 基于Cookie进行流量切分
- 基于请求参数进行流量切分

举例：根据请求中携带的用户uid进行取模，灰度的范围是百分之一，那么uid取模的范围就是100，模是0访问新版服务，模是1~99的访问老版服务。