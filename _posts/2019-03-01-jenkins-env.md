---
layout: post
title: Jenkins构建完成后自动部署使用参数将应用部署到不同的服务器
description: 
date: 2019-03-01
categories:
    - Jenkins
comments: true
permalink: jenkins-env.html
---

在jenkins根据不通的变量发布执行不同的命令，主要用到其选择参数 

安装插件choice parameter

选择参数化构建过程

![](/assets/images/posts/jenkins-env/deploy_env.png)

增加SSH服务器，在label中输入上一步参数对应的值

![](/assets/images/posts/jenkins-env/ssh.png)

点高级设置参数化发布

![](/assets/images/posts/jenkins-env/publish.png)

发布是选择对应的参数

![](/assets/images/posts/jenkins-env/build.png)