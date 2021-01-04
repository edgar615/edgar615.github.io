---
layout: post
title: 修改ubuntu源为阿里源
date: 2020-07-02
categories:
    - linux
comments: true
permalink: ubuntu-source.html
---

```
# 更新软件源为阿里源并用它更新软件
cp /etc/apt/sources.list /etc/apt/sources.list.bak_`date "+%y_%m_%d"`
sed -i 's/http:\/\/.*.ubuntu.com/https:\/\/mirrors.aliyun.com/g' /etc/apt/sources.list
apt update
apt upgrade
```

参考

https://developer.aliyun.com/mirror/