---
layout: post
title: Jenkins常见问题
description: 
date: 2019-02-02
categories:
    - Jenkins
comments: true
permalink: jenkins-common.html
---

# 1. 用docker启动jenkins的时候遇到volume权限问题

```
Can not write to /var/jenkins_home/copy_reference_file.log. Wrong volume permissions? touch: cannot touch ‘/var/jenkins_home/copy_reference_file.log’: Permission denied
```

在执行docker run命令的时候增加一个-u参数，如下改进后的命令，
```
docker run -d -v /root/jenkins:/var/jenkins_home -u 0 -P --name jenkins-server jenkins
```

这命令的意思是覆盖容器中内置的帐号，该用外部传入，这里传入0代表的是root帐号Id。这样再启动的时候就应该没问题了。

# 2. 插件地址

有的插件安装失败，手动安装

http://ftp.tsukuba.wide.ad.jp/software/jenkins/plugins

# 3. 防止jenkins部署后杀死进程

```
export BUILD_ID=dontKillMe
```
