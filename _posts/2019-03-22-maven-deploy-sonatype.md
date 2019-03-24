---
layout: post
title: 发布Jar包到Maven中央仓库
description: 
date: 2019-03-22
categories:
    - maven
comments: true
permalink: maven-deploy-to-sonatype.html
---

在发布Java包到Maven中央仓库首先需要在[https://issues.sonatype.org/secure/Dashboard.jspa](https://www.iteblog.com/redirect.php?url=aHR0cHM6Ly9pc3N1ZXMuc29uYXR5cGUub3JnL3NlY3VyZS9EYXNoYm9hcmQuanNwYQ==&article=true)网站创建一个工单(Issues)，第一次使用这个网站的时候需要注册自己的帐号（这个帐号和密码需要记住，后面会用到），之后创建自己的Issue，点击导航最上面的`Create`按钮,创建完之后需要等待Sonatype的工作人员审核处理，审核时间还是很快的，我的审核差不多等待了两小时。当Issue的Status变为`RESOLVED`后，就可以进行下一步操作了。

# GPG生成密钥对

参考[linux下gpg的使用](https://edgar615.github.io/linux-gpg.html)

上传到公钥到第三方的key验证库

```
gpg --keyserver hkp://keyserver.ubuntu.com --send-keys 1890095F
gpg --keyserver hkp://pool.sks-keyservers.net --send-keys 1890095F

```

**PS：我忘记上面的哪个起作用了**

# 修改setting.xml

```
<servers>
  <server>
    <id>sonatype-nexus-snapshots</id>
    <username>Sonatype网站的账号</username>
    <password>Sonatype网站的密码</password>
  </server>
  <server>
    <id>sonatype-nexus-staging</id>
    <username>Sonatype网站的账号</username>
    <password>Sonatype网站的密码</password>
  </server>
</servers>
```

# pom.xml增加配置

```
<parent>
        <groupId>org.sonatype.oss</groupId>
        <artifactId>oss-parent</artifactId>
        <version>7</version>
</parent>
```

# 打包发布

```
mvn clean deploy -P sonatype-oss-release
```

访问[https://oss.sonatype.org/#stagingRepositories](https://oss.sonatype.org/#stagingRepositories)找到刚才发布的JAR，分别点击close和release，过几个小时之后就可以在官方仓库中找到了

