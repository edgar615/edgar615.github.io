---
layout: post
title: redis��������
date: 2019-06-12
categories:
    - redis
comments: true
permalink: redis-common-op.html
---

# ͳ�Ƹ���

redis�����ƺ���OMP_OFFLINE��key�ĸ�����

```
src/redis-cli keys "*OMP_OFFLINE*" | wc -l
```

# ����ɾ��
����ɾ�� 0�����ݿ������ƺ���OMP_OFFLINE��key��

```
src/redis-cli -n 0 keys "*OMP_OFFLINE*" | xargs src/redis-cli -n 0 del
```
